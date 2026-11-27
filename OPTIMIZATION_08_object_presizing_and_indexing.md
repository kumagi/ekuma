# Optimization #8: Post-Compilation Object Presizing (`presizeNewObjects`) and Plain-Object Element Append Fast Paths

## 1. Summary & Measured Impact

- **Rank**: #8 of 8
- **Targeted Subset Impact**: **+0.54 nats** directly on `prop_create` (contributing to Stage 6A's **+1.98 nats** / 1.027x suite-wide lift alongside Optimization #7; also reduces slot reallocations across object-factory patterns in larger workloads)
- **Key Benchmark Improvement (Stage 5 → Stage 6A, and Baseline → Final 3-Round Median)**:

| Benchmark | Baseline Ekuma | Stage 5 Ekuma | Stage 6A Ekuma | Final 3-Round Ekuma | Stage Speedup | Overall vs Baseline |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| `prop_create` (`obj = {}; obj.a..j = 1..10; for (i=0..9) obj[i] = i`) | 1272 ns | 1358 ns | 790 ns | 802 ns | **1.72x** (+0.54 nats) | **1.59x** |

---

## 2. Root Cause Analysis

Factory functions and object-building loops commonly initialize an empty object literal `obj = {}` and immediately populate a sequence of named properties (`obj.a = 1; obj.b = 2; ...; obj.j = 10;`) and/or dense integer properties (`for (i = 0; i < 10; i++) obj[i] = i;`). Baseline Ekuma suffered two bottlenecks on this pattern:

1. **Zero-capacity `OpNewObject` caused repeated slot reallocations and `execFast` exits**: `OpNewObject` already accepts an inline slot-capacity hint in operand `C` (`storage.NewObjectWithStorage(int(i.C))`), which Ekuma used for literal properties like `{a: 1, b: 2}`. However, for `obj = {}`, the compiler emitted `OpNewObject obj, 0, 0` (`C = 0`). Consequently, adding 10 named properties afterwards missed `execFast`'s `CacheAdd` check (`SlotCount() < SlotRoom()`) three times — at property 1 (0 → 4), property 5 (4 → 8), and property 9 (8 → 16) — causing 3 slow-path exits, 3 separate `[]Value` heap allocations, and 2 slice copies per object.
2. **No `execFast` index-append path for plain objects (`ClassObject`)**: In `execFast`, `OpSetIndex` and `OpSetIndexInt` only supported appending a new dense integer element (`j == ElemCount() && j < ElemRoom()`) when `o.Kind == storage.KindArray`. Every `obj[i] = i` on a plain `{}` object therefore exited `execFast` into slow `setIndex`, even when `obj` already had room in `o.elems` and inherited no index properties from `Object.prototype`.

---

## 3. Prerequisites & Dependency Order

- **Plain-object index append depends on Optimization #2** (`execFast` leaf interpreter loop).
- **Compiler `presizeNewObjects()` is completely independent** and works even without `execFast` (though pairing it with `execFast`'s `CacheAdd` eliminates all slow-path exits too).

---

## 4. Target Files Inventory

| File | Role |
| --- | --- |
| `internal/compiler/appends.go` | Implement `(*fn).presizeNewObjects()` post-compilation bytecode peephole pass. |
| `internal/compiler/compiler.go` | Invoke `f.presizeNewObjects()` inside `(*fn).finish()` right after `f.settleAppends()`. |
| `internal/engine/interp.go` | Add plain `KindObject` (`ClassObject`) element-append fast paths to both `OpSetIndex`/`OpSetIndexStrict` and `OpSetIndexInt`/`OpSetIndexIntStrict` in `execFast`. |
| `internal/engine/rawaccess_test.go` | Increment `ExtendElems` (+2) and `SetElem` (+2) call counts in `rawStoreCalls`. |
| `internal/engine/protowalk_test.go` | Increment `agent.execFast` `Shape.Proto` read count (+2). |

---

## 5. Detailed Design Specification (No Code)

### 5.1 Post-Compilation `presizeNewObjects` Pass (`internal/compiler/appends.go`, `compiler.go`)

1. In `(*fn).finish()` (`internal/compiler/compiler.go`), call `f.presizeNewObjects()` immediately after `f.settleAppends()`.
2. Implement `(*fn).presizeNewObjects()` with zero heap allocations:
   - Set `const maxPresizeSlots = 16` (matching `storage.MaxInlineObjectValues`, the largest inline `objectWith[[17]Value]` single-allocation size class in `storage.NewObjectWithStorage`).
   - Iterate `pc` over `code := f.f.Code`. Skip any instruction unless `code[pc].Op == bytecode.OpNewObject && code[pc].C == 0`.
   - Track the object's destination register `reg := code[pc].A`, and advance `k := pc + 1`.
   - **Follow immediate move**: Because `objectLit` often emits `OpNewObject tmp, 0, 0` followed by `OpMove obj, tmp, 0` when assigning to a local variable (`obj = {}`), check `if k < len(code) && code[k].Op == bytecode.OpMove && code[k].B == reg`, and if so update `reg = code[k].A` and `k++`.
   - Maintain a stack-allocated array `var seen [maxPresizeSlots]int32` and `count := 0`.
   - Scan forward while `k < len(code) && count < maxPresizeSlots`:
     - **`OpSetProp` / `OpSetPropStrict`**:
       - If `in.A != reg`, stop scanning (store targets a different object).
       - Deduplicate by property name `f.f.Keys[in.B]` against `seen[:count]` (two distinct `OpSetProp` sites can target the same key name, e.g. `obj.a = 1; obj.a = 2;`, which occupies only 1 slot).
       - If new, record `seen[count] = in.B; count++`.
     - **Pure value-preparation instructions**: allow `OpLoadK`, `OpLoadInt`, `OpLoadTrue`, `OpLoadFalse`, `OpLoadNull`, `OpLoadUndef`, `OpMove`, and `OpGetUpval` to pass through **only when `in.A != reg`** (if `in.A == reg`, the object register was overwritten — stop scanning).
     - **Any other instruction** (calls, jumps, property reads, arithmetic, etc.): stop scanning immediately.
   - If `count > 0`, write `code[pc].C = int32(count)`.
3. **Why this transformation is 100% semantics-preserving**: `OpNewObject`'s operand `C` is strictly a backing-storage capacity hint (`o.InitSlots(slots[:0])`). The created object still has `len(slots) == 0`, the exact same empty root shape `Shapes.Object`, and zero observable behavioral difference even if control flow exits early before all `count` properties are assigned. For `prop_create`, `count == 10` allocates a single inline `objectWith[[11]Value]` and allows all 10 `obj.a = 1 ... obj.j = 10` assignments to hit `execFast`'s `CacheAdd` with **zero reallocations and zero slow-path exits**.

### 5.2 Plain-Object Index Append in `execFast` (`internal/engine/interp.go`)

1. In `execFast`'s `case bytecode.OpSetIndex, bytecode.OpSetIndexStrict:` — immediately as an `else if` branch after the `o.Kind == storage.KindArray` append block:
   - Preconditions (all must hold):
     - `o.Kind == storage.KindObject`
     - `o.Class == storage.ClassObject` (excludes proxies, typed arrays, arguments objects, exotic objects, boxed primitives)
     - `o.Flags() == 0` (guarantees extensible, not prototype, not sparse, not exotic)
     - `float64(j) == k.N` (exact integer index)
     - `j == o.Raw().ElemCount() && j < o.Raw().ElemRoom()` (note: because `zeroTail.elems` has `cap == 0`, `j < ElemRoom()` with `j >= 0` automatically proves `o` already owns its own `objectTail`, and plain `ClassObject` never sets `SharedElems`!)
   - Prototype check:
     - `p := o.Shape().Proto`
     - Require `p == ag.active.realm.Intrinsics[storage.IntrinsicObjectPrototype] && p.Raw().ElemCount() == 0 && p.Flags()&storage.ObjSparse == 0` (guarantees `Object.prototype` has no inherited index setter or non-writable index property!).
   - Mutation:
     - Read `x := *execRegAt(regs, regsBase, int(i.C))`.
     - Call `o.Raw().ExtendElems(j + 1)` and `o.Raw().SetElem(j, x)` (plain objects have no `.length` field to update!).
     - `continue`.
2. Add the exact symmetric `else if` branch in `case bytecode.OpSetIndexInt, bytecode.OpSetIndexIntStrict:` (dropping the redundant `float64(j) == k.N` check since `j := int(i.B)` is already a non-negative integer literal), keeping `OpSetIndex` and `OpSetIndexInt` strictly in sync per the file's documented invariant.

---

## 6. Architectural Invariants & White-Box Test Updates

- **`internal/engine/rawaccess_test.go`**: Increment `ExtendElems` (+2) and `SetElem` (+2) in `rawStoreCalls` for `agent.execFast`.
- **`internal/engine/protowalk_test.go`**: Increment `agent.execFast` `Shape.Proto` read count by +2 (one for `OpSetIndex`, one for `OpSetIndexInt`).
- **`TestExecFastPathsCallNothing/compiled`**: No new functions or methods are called — all helpers (`Shape`, `Raw`, `ElemCount`, `ElemRoom`, `ExtendElems`, `SetElem`, `Flags`) are already allowlisted and inlined.

---

## 7. Known Pitfalls & Failed Approaches to Avoid

1. **Pitfall #1 — Opcode naming**: Ekuma's integer-literal load opcode is `bytecode.OpLoadInt` (not `OpLoadNum`). Include `OpLoadInt` in the benign value-prep list of `presizeNewObjects`; otherwise `obj.a = 1; obj.b = 2;` stops immediately on `OpLoadInt` and `count` stays 0!
2. **Pitfall #2 — Forgetting `o.Class == storage.ClassObject` or `o.Flags() == 0` in `execFast`**: Plain objects with `Object.preventExtensions(obj)` have `ObjNotExtensible` set in `Flags()`; objects with custom getters/setters on index keys or sparse indices have `ObjSparse`; proxies/lazy objects have `ObjExotic` and non-`ClassObject` classes. Checking `o.Kind == KindObject && o.Class == ClassObject && o.Flags() == 0` rejects every unsafe object in two byte comparisons.
3. **Pitfall #3 — Forgetting `p.Raw().ElemCount() == 0 && p.Flags()&ObjSparse == 0` on `Object.prototype`**: If user code defines an accessor `Object.defineProperty(Object.prototype, 0, {set: ...})`, assigning `obj[0] = 0` must invoke the inherited setter instead of appending an own element. The prototype check guarantees complete ECMAScript compliance.
4. **Pitfall #4 — Do NOT call `o.Raw().SetLength(...)` on `KindObject`**: Unlike `KindArray`, ordinary objects do not maintain a magic `length` field in `Object.length`.

---

## 8. `unsafe` Usage & Pure-Safe-Go Alternative

- **`unsafe` used**: **Zero** — 100% pure safe Go.

---

## 9. Verification & Reproduction Protocol

1. Run `go test ./internal/compiler ./internal/engine` — specifically confirm `TestRawAccess`, `TestPrototypeWalks`, `TestExecFastPathsCallNothing`, and object-property / prototype-setter / `preventExtensions` tests pass.
2. Run `go test -tags ekumavalidate ./internal/engine` and `go test -race ./internal/engine`.
3. Run `_bench` microbenchmarks (`-suite Micro -engine '^ekuma$' -rounds 1`) and confirm `prop_create` drops from ~1,270–1,350 ns down to **~790–850 ns (1.55x–1.72x faster)**.
