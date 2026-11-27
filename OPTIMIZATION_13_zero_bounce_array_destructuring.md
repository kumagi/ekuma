# Optimization #13: Zero-Bounce Array Destructuring (`OpGetIterator`, `OpIterElision`, `OpIterRest`, `NewArraySlice`)

## 1. Summary & Measured Impact

- **Overall Rank**: #12 of 13
- **Suite-Wide Impact**: **+1.05 nats** across all 74 microbenchmarks (**1.014x** step geometric-mean lift in Stage 8)
- **Targeted Subset Impact**: **+1.03 nats** across the 3 destructuring benchmarks and `array_for_of`
- **Key Benchmark Improvements (Stage 7 → Stage 8 → Final 3-Round Median)**:

| Benchmark | Stage 7 Ekuma | Stage 8 Ekuma | Final Ekuma | Stage 8 Speedup | Overall vs Baseline |
| --- | ---: | ---: | ---: | ---: | ---: |
| `local_destruct` (`[a, , b, ...rest] = arr` into locals) | 241.4 ns | 168.6 ns | 177.0 ns | **1.43x** (+0.36 nats) | **1.41x** |
| `global_destruct` (array destructuring into globals) | 255.7 ns | 189.5 ns | 194.6 ns | **1.35x** (+0.30 nats) | **1.38x** |
| `global_destruct_strict` (strict global destructuring) | 253.7 ns | 192.0 ns | 191.9 ns | **1.32x** (+0.28 nats) | **1.38x** |
| `array_for_of` (plain array `for..of` iteration) | 883.5 ns | 808.6 ns | 837.8 ns | **1.09x** (+0.09 nats) | **1.46x** |

---

## 2. Root Cause Analysis

Array destructuring patterns such as `[a, , b, ...rest] = arr` suffered three severe bytecode and interpreter inefficiencies:

1. **16-instruction elision sequence exited `execFast`**: Every skipped hole (`, ,`) in a destructuring pattern compiled to a 16-instruction generic iterator-step sequence including `OpTypeof` and branch checks, forcing an exit from `execFast` into slow `exec` just to advance the array index by 1!
2. **Element-by-element rest-element loop**: A rest pattern (`...rest`) compiled into an explicit bytecode loop calling `OpIterNext` + `OpArrayPush` for every remaining element — allocating a growable array, looping through bytecode dispatch per element, and never taking advantage of bulk slice copying when the iterator was a plain array cursor (`regs[iter] == Number(index)`).
3. **`OpGetIterator` and numeric `OpIterClose` always exited `execFast`**: Even for a plain dense array whose `[Symbol.iterator]` and `%ArrayIteratorPrototype%.next` were unmodified, `OpGetIterator` exited `execFast` to call `getIterator`, and `OpIterClose` exited `execFast` even when the iterator was an unmaterialized numeric cursor (`NumberObject`, where closing is a guaranteed no-op!).

---

## 3. Prerequisites & Dependency Order

- **Depends on Optimization #2** (`execFast` leaf interpreter loop) and **Optimization #4** (`execFast` `OpIterNext`/`OpIterClose` for plain numeric array cursors).
- Works cleanly with **Optimization #3** (`PackedNums` arrays via `NewArraySlice`).

---

## 4. Target Files Inventory

| File | Role |
| --- | --- |
| `internal/storage/arrayiteration.go` | Add `IterRef, NextRef *Value` on `arrayIterCache` populated when `c.found == true` so `execFast` can verify unmodified `%Array.prototype%[Symbol.iterator]` and `%ArrayIteratorPrototype%.next` in 2 pointer loads without calling a function. |
| `internal/storage/array.go` | Add `(*RealmPart).NewArraySlice(o *Object, from uint32) (*Object, bool)` copying dense elements `[from .. length)` in one allocation (`NewTailObjectWithStorage` for small slices, chunk-polled copy for large `PackedNums`, `InitElems` for proper storage ownership). |
| `internal/bytecode/bytecode.go` | Add `OpIterElision` (advance iterator without storing value) and `OpIterRest` (collect remaining iterator values into a new array and mark iterator done). |
| `internal/bytecode/operands.go` | Register operand encodings for `OpIterElision` and `OpIterRest` (`OpIterRest` writes both `FieldA` (result array) and `FieldB` (iterator register cleared to done)). |
| `internal/bytecode/operands_test.go` | Add `"OpIterRest": true` to `alsoOthers` in `TestResultConvention`. |
| `internal/compiler/pattern.go` | Replace the 16-instruction elision sequence with a single `OpIterElision`; replace the rest-element bytecode loop with a single `OpIterRest` (writing directly into the target local register when `!prepared`). |
| `internal/engine/interp.go` | Add `OpGetIterator` (using `arrayIterCache.IterRef`/`NextRef` + `ProtoEpoch`), `OpIterElision`, and numeric `OpIterClose` fast paths to `execFast`; implement full `OpIterElision` and `OpIterRest` (with `InterruptFlag` check + `NewArraySlice` fast path + generic fallback) in slow `exec`. |
| `internal/engine/rawaccess_test.go` | Raise `Object.PushElement` count in `agent.exec` from `2` to `4` (for generic `OpIterRest` fallback). |
| `internal/engine/testdata/rawusers.txt` | Allowlist `arrayIterCache.IterRef:ref` and `arrayIterCache.NextRef:ref` for `agent.execFast`. |

---

## 5. Detailed Design Specification (No Code)

### 5.1 Zero-Call `OpGetIterator` Cache (`internal/storage/arrayiteration.go`, `internal/engine/interp.go`)

1. On `storage.arrayIterCache`, add `IterRef *Value` (pointing directly at `%Array.prototype%`'s `[Symbol.iterator]` slot) and `NextRef *Value` (pointing directly at `%ArrayIteratorPrototype%`'s `next` slot), populated whenever `PlainArrayIteration` verifies `c.found == true` at `c.epoch == r.ProtoEpoch`.
2. In `execFast` `case bytecode.OpGetIterator:`:
   - Verify `o := regs[i.B].O` is non-nil, `o.Kind == storage.KindArray`, `o.Shape() == ag.active.realm.Shapes.Array`, `o.Flags() == 0`.
   - Verify the realm's `arrayIterCache` has `c.found && c.epoch == ag.ProtoEpoch && c.IterRef.O == IntrinsicArrayPrototypeValues && c.NextRef.O == IntrinsicArrayIteratorPrototypeNext` (and `validateExec` runs `checkPlainArrayIteration`!).
   - Store the unmaterialized numeric cursor `storage.SetNumber(&regs[i.A], 0)` and `regs[i.A+1] = Value{O: o}`, and `continue` — **zero function calls, zero iterator object allocation**!

### 5.2 Dedicated `OpIterElision` & `OpIterRest` Bytecodes (`internal/bytecode/`, `internal/compiler/pattern.go`)

1. **`OpIterElision(iter)`**:
   - In `internal/compiler/pattern.go`, replace the 16-instruction elision sequence with a single `OpIterElision(iter, 0, 0)`.
   - In `execFast`: if `regs[i.A].O == storage.NumberObject` (plain array cursor at index `j`), check `o := regs[i.A+1].O`, `!ag.ProtoElemsDirty`, `o.Shape() == Shapes.Array`, and `o.Flags() == 0`. If `j < o.Length()` and either `j < o.Raw().NumCount()` or (`j < o.Raw().ElemCount()` && `!o.Raw().Elem(j).IsHole()`), simply increment `storage.SetNumber(&regs[i.A], float64(j+1))` and `continue`! If `j >= o.Length()`, mark done (`regs[i.A] = Undefined`, `regs[i.A+1] = Undefined`) and `continue`! Otherwise decline to slow `exec` (which steps the generic iterator and discards the value).
2. **`OpIterRest(dst, iter)`**:
   - In `internal/compiler/pattern.go`, replace the manual rest-element loop with a single `OpIterRest(dst, iter, 0)` (emitting directly into the local variable's register when `!prepared` so no trailing `OpMove` is needed).
   - In slow `exec`:
     - **CRITICAL**: Check `if ag.InterruptFlag.Load()` at the very start of `OpIterRest` and poll before returning!
     - If `regs[i.B].O == storage.NumberObject` (plain array cursor at index `j`) and `!ag.ProtoElemsDirty`: update frame PC when `o.Length() > j + storage.PollChunk`, call `a, ok := ag.active.NewArraySlice(o, j)`. On `ok`: mark iterator done (`regs[i.B] = Undefined`, `regs[i.B+1] = Undefined`), store `regs[i.A] = Value{O: a}`, and `continue`!
     - Otherwise fall back to the generic iterator-draining loop pushing into a new array (`o.PushElement(v)`), marking the iterator done on completion (and leaving it open if an exception is thrown during stepping).
3. **`NewArraySlice(o *Object, from uint32) (*Object, bool)` (`internal/storage/array.go`)**:
   - If `from >= o.length`, return `r.NewArray(nil), true`.
   - Check `o.Kind == KindArray && o.shape == r.Shapes.Array && o.flags == 0`.
   - For `PackedNums`: if `rem <= MaxInlineObjectValues` (≤ 8 elements), allocate one `NewTailObjectWithStorage(rem)` with `InitElems(values)` and fill `Value{O: NumberObject, N: x}` (single allocation, zero slice header alloc!); for larger slices, allocate `make([]float64, rem)` with `PollChunk` polling and return `r.NewPackedNumArray(nums)`.
   - For ordinary dense `elems`: verify `!containsHole(o.elems[from:n])`, use `NewTailObjectWithStorage(rem)` + `InitElems` for small slices or `appendPolled` + `NewArray` for large slices.
4. **Numeric `OpIterClose` in `execFast`**:
   - If `regs[i.A].O == storage.NumberObject` or `regs[i.A].O == nil`, clear `regs[i.A] = Value{}` and `regs[i.A+1] = Value{}` and `continue` directly inside `execFast`!

---

## 6. Architectural Invariants & White-Box Test Updates

- **`internal/bytecode/operands_test.go`**: Add `"OpIterRest": true` to `alsoOthers` because `OpIterRest` writes both `FieldA` (new array) and `FieldB` (clearing iterator state).
- **`internal/engine/rawaccess_test.go`**: Update `Object.PushElement` in `agent.exec` from `2` to `4`.
- **`internal/engine/testdata/rawusers.txt`**: Add `arrayIterCache.IterRef:ref arrayIterCache.NextRef:ref` to `agent.execFast`.
- **`TestValueStorageOwnership`**: Always initialize `NewTailObjectWithStorage` slices in `NewArraySlice` via `a.InitElems(values)` (never direct `a.elems = values`).
- **`TestUnregisteredShortcutInterruption`**: Always check `ag.InterruptFlag.Load()` at the entry of `OpIterRest` and poll across chunks larger than `PollChunk`.

---

## 7. Known Pitfalls & Failed Approaches to Avoid

1. **Pitfall #1 — Writing `a.elems = values` directly in `NewArraySlice`**: Triggers `TestValueStorageOwnership` failure. Always use `a.InitElems(values)`.
2. **Pitfall #2 — Omitting `InterruptFlag` check in `OpIterRest` for small arrays**: A tight JS loop executing `[...smallArr]` can run forever without polling if `OpIterRest` only polls for `rem > PollChunk`. Checking `if ag.InterruptFlag.Load()` at entry prevents uninterruptible loops and satisfies `TestUnregisteredShortcutInterruption`.
3. **Pitfall #3 — Directly writing `regs[i.A].N = ...` in slow `exec`**: Violates `TestRawStorage`'s `Value.N:write` count for `agent.exec`. Always use `storage.SetNumber(&regs[i.A], ...)`.

---

## 8. `unsafe` Usage & Pure-Safe-Go Alternative

- **`unsafe` used**: **Zero** — 100% pure safe Go.

---

## 9. Verification & Reproduction Protocol

1. Run `go test ./internal/bytecode ./internal/compiler ./internal/engine`.
2. Run `go test -tags ekumavalidate -run 'TestExecFastPathsCallNothing|TestRawStorage|TestValueStorageOwnership|TestUnregisteredShortcutInterruption|TestResultConvention' ./internal/engine ./internal/bytecode`.
3. Run `_bench` microbenchmarks (`-suite Micro -engine '^ekuma$' -rounds 1`) and verify: `local_destruct` drops from ~241 ns to ~169–177 ns (~1.4x), `global_destruct` and `global_destruct_strict` drop from ~255 ns to ~190–195 ns (~1.35x), and `array_for_of` improves ~9%.
