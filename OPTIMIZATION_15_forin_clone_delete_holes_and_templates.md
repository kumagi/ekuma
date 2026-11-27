# Optimization #15: Zero-Pack `for..in`, Cached-Slot Property Deletion, Single-Allocation Object Cloning, Holey Array Truncation, and Single-Pass Templates

## 1. Summary & Measured Impact

- **Overall Rank**: #12 of 18 (Stage 14)
- **Suite-Wide Impact**: **+1.74 nats** across all 74 microbenchmarks (**1.024x** step geometric-mean lift in Stage 14)
- **Targeted Subset Impact**: **+2.33 nats** across the five targeted benchmarks, eliminating every remaining baseline regression across the entire 74-benchmark suite
- **Key Benchmark Improvements (Stage 13 → Stage 14 → Final 3-Round Median)**:

| Benchmark | Baseline Ekuma | Stage 13 Ekuma | Stage 14 Ekuma | Final Ekuma (3r) | Stage 14 Speedup | Overall vs Baseline |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| `array_for_in` (`for (i in arr) sum += arr[i]`) | 3.54 µs | 4.26 µs | 2.21 µs | 1.96 µs | **1.93x** (+0.66 nats) | **1.81x** |
| `array_hole_length_decr` (decrementing `.length` on holey array) | 46.9 µs | 53.2 µs | 32.3 µs | 31.9 µs | **1.65x** (+0.50 nats) | **1.47x** |
| `string_template` (40-part template literal interpolation) | 522.5 ns | 543.5 ns | 344.0 ns | 340.9 ns | **1.58x** (+0.46 nats) | **1.53x** |
| `prop_delete` (cloning and deleting 10 named + 10 indexed properties) | 1.03 µs | 1.16 µs | 787 ns | 745 ns | **1.47x** (+0.39 nats) | **1.38x** |
| `prop_clone` (object spread `{ ...ref }` with 10 slots + 10 elements) | 376.7 ns | 410.9 ns | 294.5 ns | 288.9 ns | **1.40x** (+0.33 nats) | **1.30x** |

---

## 2. Root Cause Analysis

CPU profiling of the last four baseline regressions plus `string_template` uncovered five clean, general-purpose bottlenecks:

1. **`array_for_in` repacked cached strings on every iteration and bounced twice per step**: `Space.IndexStrings` cached `[]*storage.Object`, so every hit in `indexString(i)` called `storage.ObjectValue(o)` -> `shortstrings.Pack(o.Text())` (16.7% of total CPU!). Furthermore, both `OpForInNext` and string-keyed `OpGetIndex` (`r[i]` where `i` is `"0"`..`"99"`) exited `execFast` into slow `exec` with a stack `reload` on every iteration.
2. **`array_hole_length_decr` missed `execFast`'s `.length` truncation fast path**: In `interp.go`, `execFast`'s `OpSetProp` `.length = newLen` fast path only accepted `o.Raw().IsPackedNums()`. Holey arrays store elements in `o.elems` (not `o.nums`), so every single decrement bounced to slow `exec` -> `TruncateElements`.
3. **`prop_clone` did 4 heap allocations per `{ ...ref }` and walked the shape chain every time**: `OpNewObject` allocated a tailless `plainObject` (alloc #1); `OpCopyProps` bounced to slow `exec`, called `takenShape` which walked all 10 `Parent` pointers every clone, allocated `objectTail` via `ownTail()` (alloc #2), `slices.Clone(from.elems)` (alloc #3), and `fillSlots` (alloc #4).
4. **`prop_delete` ran slow `deleteValue` + `ToPropertyKey` + `ToObject` + double `Shape.Lookup` on every property**: Every `delete obj.a` and `delete obj[0]` exited `execFast`, performed full key/object coercion, looked up `key` in `old.Table` once in `DeleteProp` and a second time inside `Space.Remove` even when `old.deletion` already cached the exact target shape!
5. **`string_template` ran two full passes and allocated short-string unpacks via `p.RawString()`**: `OpConcat` always exited `execFast` (and `OpSetGlobal` was missing from `execQuiet`), while `concatSmall` called `p.RawString()` on every non-interned short part instead of reading the already-packed bytes from `p.N`.

---

## 3. Prerequisites & Dependency Order

- **Depends on Optimization #2** (`execFast`), **Optimization #3** (`OpSetProp` `.length` truncation), and **Optimization #9** (`execQuiet`).

---

## 4. Target Files Inventory

| File | Role |
| --- | --- |
| `internal/storage/space.go` | Change `Space.IndexStrings` from `[]*Object` to `[]Value` holding pre-packed index string values. |
| `internal/engine/object.go` | Update `ExecutionContext.indexString` to store `storage.ObjectValue(o)` (preserving `o.SetArrayIndex(i)` while eliminating repeated `shortstrings.Pack`). |
| `internal/storage/shape.go` | Add `root *Shape` and `plainData bool` to `Shape` (initialized in `NewShape`, propagated in `Shape.With`) for $O(1)$ `takenShape`; extend `deletion` with `slot int32` and `configurable bool`. |
| `internal/storage/property.go` | Make `takenShape` $O(1)$ via `s.plainData && s.root == dst.shape`; combine `elems` + `slots` into one slice in `TakeDataProperties`; add `(*RealmPart).CloneOrdinaryObject` (single `tailObjectWith[[24]Value]` / `NewTailObjectWithStorage` / `NewObjectWithStorage` allocation); add `(*Space).DeletePlainNamed` with O(1) `old.deletion` fast hit. |
| `internal/engine/concat.go` | Add single-pass stack-buffer ASCII/short-string fast path at the top of `ConcatStrings`; replace `p.RawString()` in `concatSmall` with direct `p.N` byte extraction. |
| `internal/engine/interp.go` | Extend `execFast` `OpSetProp` `.length = newLen` to handle non-packed `o.elems` (with `o.Meta() == nil` shifted-prefix guard and `PollChunk` / capacity-shrink guards); fuse `OpNewObject` + `OpCopyProps` via `CloneOrdinaryObject` in slow `exec`; enter `execQuiet` from `OpForIn`, `OpForInNext`, `OpDeleteProp*`, `OpConcat`, and fused `OpNewObject`+`OpCopyProps`. |
| `internal/engine/quiet_exec.go` | Add `OpSetGlobal*`/`OpPutGlobal*`, fused `OpNewObject`+`OpCopyProps`, `OpCopyProps`, `OpDeleteProp*`, `OpConcat`, `OpForInNext`, string-keyed `OpGetIndex` (checking `KnownArrayIndex` and canonical non-leading-zero `shortstrings.ParseDecimal`), and backward-jump `InterruptFlag` check in `OpJump`. |
| `internal/engine/validate_on.go` | Register `OpNewObject` + `OpCopyProps` fusion in `canFuse`. |
| `internal/engine/fastpath_test.go` | Add `"Meta": true` to `fastPathMethods`. |
| `internal/engine/rawaccess_test.go` | Update `agent.execFast` counts (`ExtendElems: 7`, `SetElem: 9`, `SetLength: 12`) and add `"agent.execQuiet": 1` under `Object.StoreElement`. |
| `internal/engine/testdata/rawusers.txt` | Add `GlobalCache.Ref:write` to `agent.execQuiet`. |

---

## 5. Detailed Design Specification (No Code)

### 5.1 Pre-Packed `IndexStrings` and `execQuiet` `for..in` (`space.go`, `object.go`, `quiet_exec.go`, `interp.go`)

1. Change `Space.IndexStrings` to `[]storage.Value`. On cache miss in `indexString(i)`, create `v := cx.newIndexString(i)`, materialize `o := v.StringObject()`, tag `o.SetArrayIndex(i)`, and store `t[i] = storage.ObjectValue(o)`. On cache hit (`v.O != nil`), return `v` directly — both `v.N` (packed bytes) and `v.O.KnownArrayIndex()` are already populated with zero repacking.
2. In `execQuiet`, handle `OpForInNext`: inspect `it := regs[i.B].O.ExtState().(*forIn)`; while `it.obj != nil && it.index < it.indices && it.obj.HasElementAt(it.index)`, increment `it.index` and store `cx.indexString(idx)`. Decline cleanly at end-of-array or on holes/prototypes.
3. Extend `execQuiet`'s `OpGetIndex` to accept `KindString` keys via `k.O.KnownArrayIndex()`, falling back to `shortstrings.ParseDecimal` for short strings that have no leading zero (`n == 1 || byte(data) != '0'`, enforcing canonical JS array-index rules so `"00"`/`"01"` decline).
4. **Critical interrupt check**: Update `execQuiet`'s `OpJump` to decline whenever `target < pc && ag.InterruptFlag.Load()`, keeping `for..in` loops over huge arrays interruptible.

### 5.2 Holey Array `.length` Truncation in `execFast` (`interp.go`, `fastpath_test.go`, `rawaccess_test.go`)

1. Inside `execFast`'s `OpSetProp` `.length = newLen` fast path (`o.Kind == KindArray && o.Shape() == Shapes.Array && o.Flags() == 0 && newLen <= o.Length()`), add an `else` branch for non-packed arrays:
   - Require `!o.Raw().ElemsShared()`, `o.Meta() == nil` (guaranteeing no shifted prefix from prior `Array.prototype.shift()`, so `ElemsStorage() == ElemRoom()`), and `(o.Raw().ElemRoom() <= 64 || int(newLen)*4 >= o.Raw().ElemRoom())` (matching `elemsOversized()` so oversized storage still shrinks in `TruncateElements`).
   - If `int(newLen) >= o.Raw().ElemCount()`, simply update `o.Raw().SetLength(newLen)` and `continue`.
   - Else if `n - int(newLen) < storage.PollChunk`, clear `o.Raw().SetElem(j, storage.Value{})` for `j` in `[newLen, n)`, call `o.Raw().ExtendElems(int(newLen))`, call `o.Raw().SetLength(newLen)`, and `continue`.

### 5.3 $O(1)$ `takenShape` and Single-Allocation Object Cloning (`shape.go`, `property.go`, `interp.go`, `quiet_exec.go`)

1. On `storage.Shape`, add `root *Shape` and `plainData bool`. Set `s.root = s; s.plainData = true` in `NewShape`; in `Shape.With(key, flags)`, set `t.root = s.root` and `t.plainData = s.plainData && flags == FlagsData && key.Kind == KindString`.
2. Simplify `takenShape(dst, from)` from an $O(N)$ parent-chain walk to a single $O(1)$ check: `!s.Private && !s.flatBased && s.Count > 0 && s.plainData && s.root == dst.shape`.
3. Add `(*RealmPart).CloneOrdinaryObject(from *Object) (*Object, bool)`: for a plain ordinary object with dense hole-free elements (`len(from.elems) <= PollChunk`) and `s.plainData && s.root == r.Shapes.Object`, allocate object + tail + elements + slots in **one** allocation (`NewObjectWithStorage(ns)` when `ne == 0`, `NewTailObjectWithStorage(ne+ns)` when `ne+ns <= 16`, `tailObjectWith[[24]Value]` with `Pinned = PinnedSlots | PinnedElems` when `16 < ne+ns <= 24`, or `&PropertyObject{}` + one combined `make([]Value, ne+ns)` slice otherwise), clipping `elems` to `[:ne:ne]` and `slots` to `[ne:total:total]` so neither can overwrite the other on growth.
4. In `TakeDataProperties`, when `ne > 0 && ne <= PollChunk && cap(dst.slots) < ns`, allocate one combined `make([]Value, ne+ns)` slice instead of two separate slices.
5. In both slow `exec` and `execQuiet`, peek-fuse `OpNewObject` + `OpCopyProps` (when `!ag.InterruptFlag.Load()` and `next.A == i.A`) to invoke `CloneOrdinaryObject` directly, and register `OpNewObject` + `OpCopyProps` in `canFuse` (`validate_on.go`).

### 5.4 Cached-Slot Property Deletion in `execQuiet` (`shape.go`, `property.go`, `interp.go`, `quiet_exec.go`)

1. Extend `Shape.deletion` with `slot int32` and `configurable bool`, populated in `Shape.Without(key)` from `p.Count - 1` and `p.Flags & FlagConfigurable != 0`.
2. Add `(*Space).DeletePlainNamed(o, key *Object) (deleted, handled bool)`:
   - Reject exotic objects (`o.flags & ObjExotic != 0`).
   - Check `if d := old.deletion; d != nil && d.key == key && d.configurable` **before** calling `old.Lookup(key)` — on repeated deletion sequences, this skips the Go map lookup entirely and executes `o.removeSlot(int(d.slot)); o.shape = d.shape; sp.TouchProto(o)`.
   - On cache miss, call `old.Lookup(key)`; if absent return `(true, true)`; if non-configurable return `(false, true)`; otherwise transition via `old.Without(key)` (reusing `p.Slot` without a second lookup) or fall back to `sp.Remove(o, key)`.
3. Add `OpDeleteProp` / `OpDeletePropStrict` to `execQuiet` (and enter `execQuiet` from slow `exec`): handle numeric indices directly via `o.HasElementAt(idx)` + `o.StoreElement(idx, storage.Hole)`, and handle interned non-index string keys (verified via `k.O.ArrayIndex()`, not `KnownArrayIndex()`) via `ag.DeletePlainNamed`, declining strict non-configurable failures to slow `exec` to throw `TypeError`. Also add `OpSetGlobal*`/`OpPutGlobal*` to `execQuiet` so loops assigning results to globals (`prop_clone`, `string_template`) never bounce out of `execQuiet`.

### 5.5 Single-Pass ASCII Stack-Buffer `ConcatStrings` (`concat.go`, `interp.go`, `quiet_exec.go`)

1. At the entry of `ConcatStrings`, when `len(parts) >= 2 && storage.TailBuilder(parts[0].O) == nil`, run a single pass copying parts directly into a stack `[storage.MaxStackConcatBytes]byte` buffer:
   - Skip empty parts; abort to the general path on `ObjReversed` or `written + nbytes > len(buf)`.
   - For `nbytes == 1`: write `byte(math.Float64bits(p.N))` directly.
   - For `2 <= nbytes <= shortstrings.MaxBytes`: abort if `nbytes >= 3 && !shortstrings.IsASCII(data)` (preventing surrogate joins), else unpack via `shortstrings.Bytes(data)` into `buf`.
   - For `nbytes > shortstrings.MaxBytes`: require `ObjASCIIKnown|ObjASCII` and copy `p.O.Text()`.
   - On success, handle 0/1 non-empty parts, short results (`<= 8` bytes via `ShortString`), or return `cx.memoizedConcat(&buf, written, nextCount)` when below the buffering threshold.
2. In `concatSmall`, replace `p.RawString()` with direct `p.N` extraction for short strings so non-ASCII multi-part concatenations also avoid heap allocations.
3. Add `OpConcat` to `execQuiet` and enter `execQuiet` from slow `exec` on `OpConcat` when `!fp.HeapRegs`.

---

## 6. Architectural Invariants & White-Box Test Updates

- **`TestPushCapacityAfterLongRuns`**: Requiring `o.Meta() == nil` in `execFast`'s non-packed `.length` truncation ensures arrays with a shifted prefix from `shift()` still fall through to `TruncateElements` -> `reallocElems`.
- **`TestUnregisteredShortcuts` (`validate_on.go`)**: Register `OpNewObject` -> `OpCopyProps` (`second.A == first.A`) in `canFuse`.
- **`TestRawStorage` / `TestUnguardedOperations`**: Update `rawaccess_test.go` (`ExtendElems: 7`, `SetElem: 9`, `SetLength: 12`, `Object.StoreElement` in `agent.execQuiet: 1`) and `rawusers.txt` (`GlobalCache.Ref:write` in `agent.execQuiet`).
- **`KnownArrayIndex` vs `ArrayIndex`**: In `OpDeleteProp`, always use `k.O.ArrayIndex()` (not `KnownArrayIndex()`) so interned numeric keys like `"0"` whose index cache bit has not been populated yet are never misrouted to `DeletePlainNamed`.

---

## 7. Known Pitfalls & Failed Approaches to Avoid

1. **Pitfall #1 — Omitting the `o.Meta() == nil` check in `execFast` `.length` truncation**: An array that had 64 elements shifted off a 128-element backing slice has `ElemRoom() == 64` while `ElemsStorage() == 128`; taking the `execFast` path skips `reallocElems` and fails `TestPushCapacityAfterLongRuns`.
2. **Pitfall #2 — Forgetting `InterruptFlag` check in `execQuiet` `OpJump`**: `for..in` loops jump backward using `OpJump` rather than `OpLoopLt`; without checking `target < pc && ag.InterruptFlag.Load()`, a `for..in` loop inside `execQuiet` would become uninterruptible.
3. **Pitfall #3 — Omitting `OpSetGlobal` from `execQuiet`**: Both `prop_clone` and `string_template` store their iteration result into a global variable; without `OpSetGlobal*` in `execQuiet`, each iteration bounces out immediately after `OpNewObject`+`OpCopyProps` or `OpConcat`.

---

## 8. `unsafe` Usage & Pure-Safe-Go Alternative

- **`unsafe` used**: **Zero** — 100% pure safe Go.

---

## 9. Verification & Reproduction Protocol

1. Run `go test -run 'TestExecFastPathsCallNothing|TestPushCapacityAfterLongRuns|TestRawStorage|TestUnguardedOperations' ./internal/engine`.
2. Run `go test -tags ekumavalidate ./internal/engine` (including `TestUnregisteredShortcuts` and `TestArrayTruncationPolls`).
3. Run `_bench` microbenchmarks (`-suite Micro -engine '^ekuma$' -rounds 1`) and verify all five targets improve cleanly with zero regressions against baseline across the suite.
