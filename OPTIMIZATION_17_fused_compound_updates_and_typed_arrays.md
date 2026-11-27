# Optimization #17: Three-Instruction Compound Update Fusion (`a[i] += v`, `o.p += v`), `TypedArray` Read/Write Fusion, and Barrier-Free `SlotRef` Updates

## 1. Summary & Measured Impact

- **Overall Rank**: #14 of 18 (Stage 16 + Stage 17 compound-update polish)
- **Suite-Wide Impact**: **+1.34 nats** across all 74 microbenchmarks (**1.018x** step geometric-mean lift in Stage 16, plus **+0.13 nats** targeted polish on `array_update`/`prop_update` in Stage 17)
- **Targeted Subset Impact**: **+1.13 nats** across `array_update`, `typed_array_read`, and `typed_array_write` in Stage 16
- **Key Benchmark Improvements (Stage 15 → Stage 16 → Final 3-Round Median)**:

| Benchmark | Baseline Ekuma | Stage 15 Ekuma | Stage 16 Ekuma | Final Ekuma (3r) | Stage 16 Speedup | Overall vs Baseline |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| `typed_array_write` (consecutive `Int32Array` stores) | 79.7 ns | 80.0 ns | 46.3 ns | 47.2 ns | **1.73x** (+0.55 nats) | **1.69x** |
| `array_update` (`tab[0] += j` compound array updates) | 122.0 ns | 79.8 ns | 53.7 ns | 51.9 ns | **1.49x** (+0.40 nats) | **2.35x** |
| `typed_array_read` (`sum += tab[k]` `Int32Array` loads) | 111.0 ns | 84.6 ns | 70.2 ns | 65.5 ns | **1.21x** (+0.19 nats) | **1.70x** |
| `prop_update` (`obj.a += j` compound property updates) | 52.7 ns | 34.0 ns | 36.1 ns (33.4 in S17) | 33.1 ns | **1.03x** (after S17 polish) | **1.59x** |

---

## 2. Root Cause Analysis

CPU profiles of `array_update`, `prop_update`, `typed_array_read`, and `typed_array_write` showed all four spending 94%–97% of total CPU inside `execFast`:

1. **Triplicated dispatch and validation on compound updates (`a[i] += v`, `o.p += v`)**: Every `tab[0] += j` executed three separate `execFast` instructions — `OpGetIndexInt`, `OpAdd`, `OpSetIndexInt` — checking `o != nil`, bounds (`NumCount`/`ElemCount`), and `NumberObject` tags twice while paying an intermediate `SetNumber` write to a temporary register (`24.2%` of total CPU in `array_update` was `storage.SetNumber` alone!). Worse, `compiler/expr.go` allocated a *second* temporary register (`v := f.newReg()`) for the result of `compoundOp`, forcing two dead register writes per update.
2. **Write-barrier overhead on in-place numeric slot updates**: Updating a numeric property slot via `o.Raw().SetSlot(slot, storage.Number(res))` wrote a full 16-byte `storage.Value` containing a pointer word (`NumberObject`), triggering Go's GC write barrier even when the slot already held `NumberObject`.
3. **Missing `OpAdd` peek-fusion and slow type dispatch on `TypedArray` reads**: While `OpGetIndexInt` and `OpGetIndex` on `Array` fused with a following `OpAdd`, the `TypedArray` branches did not — and `OpGetIndexInt` reloaded `regs[i.B].O` a second time and placed `element.Int32` fifth in its `switch t.Type`.
4. **Repeated type assertions and buffer bounds checks on consecutive `TypedArray` writes**: Sequential writes to the same `TypedArray` (`tab[0] = v0; tab[1] = v1; ...`) repeated `o.Ext.(*storage.TypedArray)`, `t.NumberStore`, `!t.Tracking`, `t.Type.Size()`, and buffer-length validation on every element.

---

## 3. Prerequisites & Dependency Order

- **Depends on Optimization #2** (`execFast` pointer dispatch), **Optimization #3** (`PackedNums` arrays), and **Optimization #4** (`execFast` peek-fusion & `validate_on.go` `canFuse`).

---

## 4. Target Files Inventory

| File | Role |
| --- | --- |
| `internal/compiler/expr.go` | In compound `MemberExpr` assignment (`assign`), reuse the `old` temporary register as the result register (`f.emit(compoundOp(n.Op), old, old, r); f.storeTarget(t, old)`) instead of allocating a second temporary `v`. |
| `internal/engine/interp.go` | Add 3-instruction compound `+=` fusion to `OpGetIndexInt` (`PackedNums` and dense `elems`), `OpGetIndex` (`PackedNums`), and `OpGetProp` (`CacheOwn`, using barrier-free `storage.SetNumber(o.Raw().SlotRef(slot), res)` when `c2.Slot == e.Slot`); merge duplicate receiver loads, hoist `Int32` first, and add `OpAdd` fusion in `TypedArray` `OpGetIndex`/`OpGetIndexInt`; add consecutive-store pair fusion for `Int32`/`Uint32`/`Float64` in `TypedArray` `OpSetIndexInt`. |
| `internal/engine/validate_on.go` | Add `canFuse3` and `fits3` for 3-instruction compound update fusion (`OpGetIndexInt`/`OpGetIndex`/`OpGetProp` + `OpAdd` + matching store); allow `OpSetIndexInt*` + `OpSetIndexInt*` (same target register) in `canFuse`. |
| `internal/engine/fastpath_test.go` | Add `"SlotRef": true` to `fastPathMethods`. |
| `internal/engine/rawaccess_test.go` | Update `agent.execFast` counts: `SetElem: 10`, `SetNum: 8`, `SetSlot: 4`, `SlotRef: 1`. |

---

## 5. Detailed Design Specification (No Code)

### 5.1 Compiler Register Reuse on Compound Member Assignments (`internal/compiler/expr.go`)

In `(*fn).assign` (`case *ast.MemberExpr`, `default` branch for compound operators like `+=`, `-=`, `*=`):
- `old := f.newReg()` holds the loaded property value, and `r := f.immediateOperand(n.Value)` evaluates the RHS into registers `>= old+1` or returns an existing local `< old` (never aliasing `old`).
- Emit `f.emit(compoundOp(n.Op), old, old, r)` and `f.storeTarget(t, old)` instead of allocating a fresh `v := f.newReg()`. This eliminates one register slot and guarantees `n.A == i.A` for statement-level compound updates.

### 5.2 Three-Instruction Compound `+=` Fusion in `execFast` (`internal/engine/interp.go`)

1. **`OpGetIndexInt` (`PackedNums` and dense `elems`) & `OpGetIndex` (`PackedNums`)**:
   - After loading numeric element `x` (or checking `e.O == NumberObject` and `!o.Raw().ElemsShared()` for `elems`), inspect `n := ip`.
   - Keep the existing 2-instruction `sum += a[k]` check (`n.Op == OpAdd && n.C == i.A && n.A == n.B`).
   - Otherwise, if `n.Op == OpAdd && n.B == i.A && n.C != i.A` (excluding self-add `a[k] += a[k]` before `i.A` is written) and `n.A != i.B && i.A != i.B` (plus `n.A != i.C && i.A != i.C` for `OpGetIndex`, ensuring neither receiver nor index register is clobbered):
     - Peek `n2 := execNextInstr(ip)`. If `n2` is the matching `OpSetIndexInt*`/`OpSetIndex*` on the same object register (`n2.A == i.B`), same index (`n2.B == i.C`), and source `n2.C == n.A`, and `y := regs[n.C]` has `y.O == NumberObject`:
     - Compute `res := x + y.N`. Write `x` to `i.A` only if `n.A != i.A`, write `SetNumber(regs[n.A], res)`, store `res` directly into `o.Raw().SetNum(...)` (or `SetElem(..., storage.Number(res))`), advance `ip = execNextInstr(n2)`, and `continue`.
2. **`OpGetProp` (`CacheOwn`)**:
   - When `e.Kind == storage.CacheOwn && v.O == storage.NumberObject` and `n := ip` satisfies `n.Op == OpAdd && n.B == a && n.C != a && n.A != i.B && a != i.B` and **`regs[i.B].O == o`** (critical guard: rejects primitive `NumberObject` receivers where `o` was redirected to `%Number.prototype%`!):
     - Peek `n2 := execNextInstr(ip)`. If `(n2.Op == OpSetProp || n2.Op == OpSetPropStrict) && n2.A == i.B && n2.C == n.A`, check store cache `c2 := execCacheAt(fp.Caches, n2.B)` for `c2.Shape == o.Shape() && c2.Kind == storage.CacheOwn` (guaranteeing the target property is own and **writable**, since read-only properties never receive `CacheOwn` in a store cache!).
     - If `y := regs[n.C]` is `NumberObject`, compute `res := v.N + y.N`, run `checkStoreHit` under `validateExec`, write `regs[n.A]`, and if `c2.Slot == e.Slot` execute `storage.SetNumber(o.Raw().SlotRef(int(e.Slot)), res)` (skipping the Go GC write barrier because slot `e.Slot` is already proven to hold `NumberObject`!), else `o.Raw().SetSlot(int(c2.Slot), storage.Number(res))`. Advance `ip = execNextInstr(n2)` and `continue`.

### 5.3 `TypedArray` Read/Write Tightening (`internal/engine/interp.go`)

1. **`OpGetIndexInt` & `OpGetIndex` (`TypedArray`)**:
   - Merge the duplicate `if o := execRegAt(...).O; o != nil` blocks in `OpGetIndexInt`.
   - Test `t.Type == element.Int32` before the `switch t.Type` (and place `Float64`, `Uint32` first inside the switch).
   - Add 2-instruction `OpAdd` peek-fusion (`n.Op == OpAdd && n.C == i.A && n.A == n.B`) immediately after `storage.SetNumber(..., x)`.
2. **`OpSetIndexInt` (`TypedArray`, `Int32`/`Uint32`/`Float64`)**:
   - Immediately after `writeU32LE(b, uint32(u))` (or `writeU64LE(b, math.Float64bits(f))`), peek `n := ip`.
   - If `(n.Op == OpSetIndexInt || n.Op == OpSetIndexIntStrict) && n.A == i.A && uint(n.B) < uint(t.FixedLength)` and `y := regs[n.C]` is a `NumberObject` (plus `InInt64Range(y.N)` for `Int32`/`Uint32`), perform the second write directly using the already-validated `t`, `t.Type`, `t.Offset`, and `t.Data.Data`, and advance `ip = execNextInstr(ip)`. Note: keep this fusion strictly in `OpSetIndexInt` (not `OpSetIndex`).

### 5.4 Three-Instruction Validation (`internal/engine/validate_on.go`)

1. Extend `checkInstr` so that when `second := bytecode.NextPC(code, s.pc)` exists, it checks both 2-instruction `canFuse(code[s.pc], code[second])` and 3-instruction `canFuse3(code[s.pc], code[second], code[third])` via `fits3`.
2. In `canFuse`, allow `OpSetIndexInt`/`OpSetIndexIntStrict` followed by `OpSetIndexInt`/`OpSetIndexIntStrict` with `second.A == first.A`.

---

## 6. Architectural Invariants & White-Box Test Updates

- **Writable-property guarantee**: `OpGetProp`'s `CacheOwn` only proves the property is an own data slot (it could be read-only). Checking `c2.Shape == o.Shape() && c2.Kind == storage.CacheOwn` on the store's own `PropCache` guarantees writability per ES2026.
- **Primitive receiver safety**: Checking `regs[i.B].O == o` prevents `OpGetProp` + `OpSetProp` fusion from ever writing onto `%Number.prototype%` when `regs[i.B]` is a primitive number.
- **Self-add safety**: Checking `n.C != i.A` guarantees `regs[n.C]` is not read before `i.A` is populated.
- **White-box test updates**: Update `fastpath_test.go` (`SlotRef`), `rawaccess_test.go` (`SetElem: 10`, `SetNum: 8`, `SetSlot: 4`, `SlotRef: 1`), and `validate_on.go` (`canFuse` + `canFuse3`/`fits3`).

---

## 7. Known Pitfalls & Failed Approaches to Avoid

1. **Pitfall #1 — Accidentally editing `OpSetIndex` alongside `OpSetIndexInt`**: Because `OpSetIndex` and `OpSetIndexInt` share identical `TypedArray` store blocks, a naive search-and-replace inserts `OpSetIndexInt` fusion into `OpSetIndex`, which `canFuse` rejects under `-tags ekumavalidate`. Apply consecutive `OpSetIndexInt` pair fusion exclusively inside `case bytecode.OpSetIndexInt, bytecode.OpSetIndexIntStrict:`.
2. **Pitfall #2 — Trusting the read cache `e` for the store in `OpGetProp` + `OpSetProp` fusion**: A non-writable own property caches as `CacheOwn` in `OpGetProp` but never in `OpSetProp`. Always validate `c2.Shape == o.Shape() && c2.Kind == storage.CacheOwn` on the store instruction's cache `n2.B`.

---

## 8. `unsafe` Usage & Pure-Safe-Go Alternative

- **`unsafe` used**: **Zero new `unsafe`** (reuses `execNextInstr` / `execRegAt` / `execCacheAt` from Optimization #2).

---

## 9. Verification & Reproduction Protocol

1. Run `go test ./internal/compiler` and `go test -run 'TestExecFastPathsCallNothing|TestRawStorage|TestUnguardedOperations' ./internal/engine`.
2. Run `go test -tags ekumavalidate ./internal/engine`.
3. Run `_bench` microbenchmarks (`-suite Micro -engine '^ekuma$' -rounds 1`) and verify: `array_update` drops from ~80 ns to ~51–54 ns (~2.35x vs baseline), `typed_array_write` drops from ~80 ns to ~46–47 ns (~1.69x vs baseline), and `typed_array_read` drops from ~85 ns to ~65–70 ns (~1.70x vs baseline).
