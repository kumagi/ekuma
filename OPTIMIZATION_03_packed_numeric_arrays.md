# Optimization #3: Promotion-on-Growth Unboxed Numeric Arrays (`PackedNums`), Cached Constructor Checks, Single-Buffer `JoinStrings`, and Inlined Array Operations

## 1. Summary & Measured Impact

- **Rank**: #3 of 8
- **Suite-Wide Impact**: **+4.49 nats** across all 74 microbenchmarks (**1.063x** step geometric-mean lift in Stage 3)
- **Targeted Subset Impact**: **+5.03 nats** across array and string-join benchmarks; also drove a **1.34x** speedup on the macrobenchmark `NavierStokes`
- **Key Benchmark Improvements (Stage 2 → Stage 3)**:

| Benchmark | Stage 2 Ekuma | Stage 3 Ekuma | Stage Speedup | Overall vs Baseline |
| --- | ---: | ---: | ---: | ---: |
| `array_length_decr` (1,000 decrements of `tab.length = i`) | 45.0 µs | 13.9 µs | **3.25x** (+1.18 nats) | **3.10x** |
| `array_slice` (`ref.slice()` of 1,000-element numeric array) | 7.58 µs | 2.87 µs | **2.64x** (+0.97 nats) | **2.64x** |
| `array_pop` (500 `tab.pop()` calls) | 31.7 µs | 12.4 µs | **2.56x** (+0.94 nats) | **2.39x** |
| `array_push` (500 `tab.push(i)` calls) | 28.2 µs | 14.0 µs | **2.02x** (+0.70 nats) | **2.10x** |
| `string_array_join` (`tab.join("")` of 50 strings) | 922 ns | 523 ns | **1.76x** (+0.57 nats) | **1.89x** |
| `array_write` (dense numeric writes) | 30.6 ns | 22.0 ns | **1.39x** (+0.33 nats) | **1.90x** |
| `array_prop_create` (1,000 `tab[i] = i` appends) | 28.9 µs | 21.7 µs | **1.33x** (+0.29 nats) | **1.43x** |
| `array_read` (dense numeric reads) | 41.2 ns | 36.0 ns | **1.15x** (+0.14 nats) | **2.02x** |

---

## 2. Root Cause Analysis

1. **GC pointer scanning & write barriers on `[]storage.Value`**: Every element in baseline `objectTail.elems []Value` is 16 bytes (`N float64` + `O *Object`). Because `Value` contains a pointer, Go allocates `[]Value` in a GC-scanned span and triggers `gcWriteBarrier2` / `wbBufFlush1` on every store, `slice` copy, and `clear()` during length truncation or `pop`. Even a pure array of numbers paid 2x memory bandwidth plus full write-barrier and GC scan overhead.
2. **Repeated prototype/species lookups in `ArrayCtorPlain`**: Every `arr.slice()` walked `Array.prototype.constructor` and `Array[Symbol.species]` via shape lookups.
3. **Native call frame overhead on `push`, `pop`, and `a.length = n`**: Single-argument `tab.push(x)`, zero-argument `tab.pop()`, and `tab.length = i` all exited `execFast` on every iteration of tight loops.
4. **Intermediate allocations in `JoinStrings`**: Joining string arrays used incremental builder writes and redundant checks instead of pre-sizing one exact `[]byte` buffer when all elements are plain non-reversed strings.

---

## 3. Prerequisites & Dependency Order

- **Depends on Optimization #2** (`execFast` leaf interpreter loop) for the inlined `PackedNums` index access, `push`, `pop`, and `.length = n` fast paths.
- The `PackedNums` storage representation, `ArrayCtorPlain` cache, and `JoinStrings` improvements in `internal/storage/` work independently.

---

## 4. Target Files Inventory

| File | Role |
| --- | --- |
| `internal/storage/object.go` | Add `PackedNums = SharedElems << 1` constant on `Object.Pinned` and `nums []float64` field on `objectTail`. |
| `internal/storage/valuestorage.go` | Add `EnsureUnpacked`, `unpackNums`, `ReserveNums`, `tryPromoteNums`; update `AppendElem`, `ReserveElems`, `ElemsStorage`, `elemsOversized`, and `reallocElems`. |
| `internal/storage/raw.go` | Add `Raw.IsPackedNums`, `NumCount`, `NumRoom`, `Num`, `SetNum`, `ExtendNums`; unpack in `Raw.Elems()`. |
| `internal/storage/realmpart.go` | Add `arrayCtorCache` struct (`protoShape`, `ctorShape`, `ctorSlot`, `speciesSlot`) to `RealmPart`. |
| `internal/storage/array.go` | Add `NewPackedNumArray`; cache `ArrayCtorPlain`; support `PackedNums` across `PlainElems`, `isDense`, `elemsMovable`, `Element`, `SetElement`, `ElementList`, `AppendElements`, `PrependElements`, `RemoveFirstElement`, `RemoveLastElement`, `CopyElementRange`, `CopyDenseElements`, `TruncateElements`, `AppendElementsOf`, `SkipHoles`, `SkipHolesBackward`, `ElemCount`, `ElemRoom`, `ElementAt`, `HasElementAt`, `StoreElement`, `HasElement`, `HasHole`, `JoinStrings` (single-buffer zero-copy `unsafe.String`), and all other array helpers. |
| `internal/storage/property.go`, `propkey.go`, `sparseindex.go`, `validate_on.go` | Ensure property deletion, enumeration, sparse index handling, and `ekumavalidate` invariants handle `PackedNums` cleanly (unpacking when non-numeric or hole-introducing mutations occur). |
| `internal/storage/intrinsic.go`, `internal/builtin/array.go` | Add `IntrinsicArrayPrototypePush` and `IntrinsicArrayPrototypePop` intrinsic IDs and record them during array initialization. |
| `internal/engine/iter.go` | Pre-allocate `PackedNums` via `ReserveNums` in `appendIterable` when spreading a `PackedNums` array into an empty array. |
| `internal/engine/interp.go` | Support `PackedNums` in `execFast` (`OpGetIndex`, `OpGetIndexInt`, `OpSetIndex`, `OpSetIndexInt`, `OpIncGetIndex`, `OpPostIncGetIndex`, `OpArrayPush`, `OpIterNext`); inline `a.length = n` truncation in `OpSetProp`; inline 1-arg `push` and 0-arg `pop` in `OpCall`. |
| `internal/engine/fastpath_test.go`, `rawaccess_test.go`, `protowalk_test.go`, `testdata/rawusers.txt` | Update white-box allowlists and counts. |

---

## 5. Detailed Design Specification (No Code)

### 5.1 Representation & Core Invariant (`internal/storage/object.go`, `valuestorage.go`)

1. Define `const PackedNums = SharedElems << 1` as a bit in `Object.Pinned`, and add `nums []float64` to `objectTail`.
2. **Strict Representation Invariant**:
   - Whenever `o.Pinned & PackedNums != 0`:
     - `o.elems` MUST be empty (`len(o.elems) == 0`).
     - `o.nums` contains ONLY unboxed JavaScript numbers with **zero holes** (every index `0 <= i < len(o.nums)` is a present, writable, enumerable, configurable data property).
     - `o.Kind == KindArray`, `o.flags == 0` (not sparse, not non-extensible, not frozen-length, not exotic).
   - Whenever an operation needs to store a non-number value, create a hole, share elements, or expose `[]Value` directly (`Raw.Elems()`, `PlainElems()`, `ReserveElems()`), call `o.EnsureUnpacked()` -> `//go:noinline o.unpackNums()`, which allocates `make([]Value, len(nums), max(cap(nums), MinElems))`, boxes every `float64` into `Value{N: x, O: NumberObject}`, sets `o.nums = nil`, clears `PackedNums`, and assigns `o.elems`.
   - Keeping `unpackNums` `//go:noinline` keeps `EnsureUnpacked` a single inlineable bit-test on the common non-packed path.

### 5.2 Promotion on Capacity Growth (`tryPromoteNums` in `internal/storage/valuestorage.go`)

1. **Why promotion-on-growth instead of promotion-on-empty**: In Ekuma, the compiler emits `OpNewArray arr, 0, 4` for `[]`, giving every empty array literal an inline 4-element `[]Value` slice inside the object's own single allocation (`PinnedElems` set, `cap(o.elems) == 4`). Short arrays (≤ 4 elements) must stay single-allocation inline arrays to avoid allocation regressions and preserve `TestAllocationsPerOperation`.
2. Implement `tryPromoteNums(v Value) bool`, called from `AppendElem(v)` and `AppendElements(o, vals)` **only when `len(o.elems) == cap(o.elems)`** (the array is about to allocate a new backing array anyway):
   - Require `v.O == NumberObject`, `o.Kind == KindArray`, `o.flags == 0`, `o.Pinned&SharedElems == 0`, and `len(o.elems) <= 16`.
   - Scan `o.elems` and confirm every existing element has `e.O == NumberObject` (no holes, no non-numbers).
   - Allocate a `noscan` slice `nums := make([]float64, n+1, max(2*n, MinElems))`, copy `e.N` for `0..n-1`, set `nums[n] = v.N`, call `o.leaveElems(elems)` (clearing pinned inline slots so old pointers are not retained!), clear `o.elems = nil`, assign `o.ownTail().nums = nums`, set `o.Pinned |= PackedNums`, and return `true`.
3. Also implement `ReserveNums(n int)` for empty plain arrays (and already-packed arrays) so array spread (`[...packedArr]` in `internal/engine/iter.go`'s `appendIterable`) allocates a single `noscan` `[]float64` buffer upfront rather than unpacking to `[]Value` (preserving `TestAllocationsPerOperation/spread_of_an_array`).

### 5.3 Storage-Layer Array Operations (`internal/storage/array.go`)

1. **`CopyElementRange`** (powers `Array.prototype.slice` and `splice`):
   - When the source is `PackedNums`, complete, and has no inherited index properties, allocate a `[]float64` of the exact requested slice length `count`, copy in chunks of `PollChunk` (calling `r.Poll()` between chunks so huge slices remain interruptible!), and return `r.NewPackedNumArray(out)`. Zero write barriers, half the bytes copied.
2. **`TruncateElements`** (powers `a.length = n`):
   - When `PackedNums` and `n < len(o.nums)`: **if `len(o.nums) - int(n) >= PollChunk`, call `p.Poll()` first** (required by `TestArrayTruncationPolls`!). Then simply reslice `o.nums = o.nums[:n]` with **zero `clear()` loop** (no pointers to zero!), update `o.length = n`, and shrink via `reallocElems` only if `elemsOversized()`.
3. **`RemoveLastElement`** (powers `pop`):
   - When `PackedNums` and `len(o.nums) > 0`: read `v := Number(o.nums[n-1])`, reslice `o.nums = o.nums[:n-1]`, update `o.length = uint32(n-1)`, shrink if oversized, and return `v` with zero write barriers.
4. **Cached `ArrayCtorPlain`** (`internal/storage/realmpart.go`, `array.go`):
   - Cache `{protoShape, ctorShape, ctorSlot, speciesSlot}` on `RealmPart.arrayCtorCache`.
   - Fast path: verify `o.shape == r.Shapes.Array`, `cc.protoShape == proto.shape`, `cc.ctorShape == ctor.shape`, `proto.slots[cc.ctorSlot].O == ctor`, and `ctor.slots[cc.speciesSlot].Accessor().Getter == r.Intrinsic(IntrinsicArraySpeciesGetter)`. That reduces every `slice()` constructor check to 5 pointer comparisons while remaining 100% sound if anyone mutates `Array.prototype.constructor` or `Array[Symbol.species]`.
5. **Single-Buffer `JoinStrings`**:
   - When joining elements of a plain array (and separator is a non-reversed string) where every non-nullish element is a non-reversed string: compute exact total byte length in one pass (rejecting if `> MaxStringLength` or any non-string/reversed element is encountered), allocate **one** `[]byte` of that exact length, copy elements + separator directly into it (unpacking short strings via `shortstrings.Bytes` when needed), and return `r.NewString(wtf8.String(unsafe.String(unsafe.SliceData(buf), len(buf))))`.
6. **Audit every remaining method touching `o.elems`**: Update `Element`, `SetElement`, `ElementAt`, `HasElementAt`, `StoreElement`, `HasElement`, `HasHole`, `ElementList`, `CopyDenseElements`, `AppendElements`, `AppendElementsOf`, `PrependElements`, `RemoveFirstElement`, `SkipHoles`, `SkipHolesBackward`, `ElemCount`, `ElemRoom`, `property.go` (deleting an index unpacks or pops), `propkey.go`, `sparseindex.go`, and `validate_on.go`.

### 5.4 Inlined Array Fast Paths in `execFast` (`internal/engine/interp.go`)

1. **Element Read/Write/Append (`OpGetIndex`, `OpGetIndexInt`, `OpSetIndex`, `OpSetIndexInt`, `OpIncGetIndex`, `OpPostIncGetIndex`, `OpArrayPush`, `OpIterNext`)**:
   - Check `uint(j) < uint(o.Raw().NumCount())` before `ElemCount()`.
   - On read: load `x := o.Raw().Num(j)` and store via `storage.SetNumber` (plus existing `+ OpAdd` peek-fusion!).
   - On write: if `x.O == NumberObject`, call `o.Raw().SetNum(j, x.N)` and `continue`; if `x` is not a number, break to slow `exec` (which calls `unpackNums`).
   - On append (`j == o.Length() == o.Raw().NumCount() && j < o.Raw().NumRoom()`): after the standard array-prototype-has-no-elements guard, `ExtendNums(j+1)`, `SetNum(j, x.N)`, `SetLength(o.Length()+1)`.
2. **Inlined `tab.length = n` truncation in `OpSetProp` / `OpSetPropStrict`**:
   - When the inline cache misses on `o.Kind == KindArray && o.Shape() == realm.Shapes.Array && o.Flags() == 0 && fp.Keys[i.B] == ag.Key(KeyLength)`:
   - If RHS is a non-negative `uint32` number `newLen <= o.Length()`, `o.Raw().IsPackedNums()`, `int(newLen) <= o.Raw().NumCount()`, **`o.Raw().NumCount() - int(newLen) < storage.PollChunk`**, and not oversized (`NumRoom() <= 64 || int(newLen)*4 >= NumRoom()`):
   - Call `o.Raw().ExtendNums(int(newLen))`, `o.Raw().SetLength(newLen)`, and `continue`.
3. **Inlined 1-arg `Array.prototype.push` and 0-arg `Array.prototype.pop` in `OpCall`**:
   - Store `IntrinsicArrayPrototypePush` and `IntrinsicArrayPrototypePop` in `internal/builtin/array.go`.
   - In `execFast`'s `OpCall`, when `callee == realm.Intrinsics[IntrinsicArrayPrototypePush]` and `argc == 1` (and receiver `this` is a plain `KindArray` with `Flags() == 0`, `Shape() == Shapes.Array`, and plain prototypes):
     - **CRITICAL capacity-hint guard**: Require `j != storage.MinElems && j+1 < room` (for both `PackedNums` and boxed `[]Value` arrays) so that hitting `MinElems` or exact capacity still falls through to slow `push`, preserving `observePush` / `TestPushCapacityAfterLongRuns`!
     - Append the single argument directly, increment `Length`, store `float64(j+1)` into the return register via `storage.SetNumber`, and `continue`.
   - When `callee == realm.Intrinsics[IntrinsicArrayPrototypePop]` and `argc == 0` (plain `KindArray`, `Flags() == 0`, `Shape() == Shapes.Array`, `PackedNums` or non-shared `[]Value` with `Length() == count > 0` and not oversized):
     - For `PackedNums`: read `x := o.Raw().Num(n-1)`, `ExtendNums(n-1)`, `SetLength(uint32(n-1))`, store `x` via `storage.SetNumber`, and `continue`.
     - For boxed `[]Value`: only if `e := o.Raw().Elem(n-1)` is not a hole, clear `o.Raw().SetElem(n-1, storage.Value{})`, `ExtendElems(n-1)`, `SetLength(uint32(n-1))`, store `e`, and `continue`.

---

## 6. Architectural Invariants & White-Box Test Updates

- **`internal/engine/fastpath_test.go`**: Add `IsPackedNums`, `NumCount`, `NumRoom`, `Num`, `SetNum`, `ExtendNums` to `fastPathMethods`, and `"ag.Key": true` to `fastPathCalls`.
- **`internal/engine/rawaccess_test.go`**: Add `ExtendNums` and `SetNum` to `rawStoreCalls`, add `IsPackedNums`, `NumCount`, `NumRoom`, `Num` to `rawLoadMethods`, and increment `SetLength` count.
- **`internal/engine/protowalk_test.go`**: Increment `agent.execFast` `Shape.Proto` read count for the new `PackedNums` append/push prototype checks.
- **Keep `Value.N:write` count unchanged in `rawusers.txt`**: Always write numeric results using `storage.SetNumber(...)` in `execFast`, never raw `dst.N = ...`.

---

## 7. Known Pitfalls & Failed Approaches to Avoid

1. **Pitfall #1 — Promoting only `cap(elems) == 0` arrays never activates `PackedNums`**: Because `[]` compiles to `OpNewArray arr, 0, 4`, every array literal starts with `cap == 4`. Promoting only empty-cap arrays left `array_slice` at 7.5 µs and `array_length_decr` at 48.4 µs. Using `tryPromoteNums` when `len(o.elems) == cap(o.elems)` immediately dropped them to 2.9 µs and 13.9 µs.
2. **Pitfall #2 — Do NOT register kernels on `Array.prototype.slice` or `Array.prototype.join`**: Doing so breaks `TestUnregisteredShortcutInterruption` (which tests interruption via `ObserveNative` on those methods) and risks missing cyclic `Array.prototype.join` tracking (`joins-of-arrays-of-strings.js`). Putting the optimizations inside `storage.CopyElementRange` (with chunk polling) and `storage.JoinStrings` captures the full speedup while honoring all contracts.
3. **Pitfall #3 — `TestPushCapacityAfterLongRuns` regression**: If inlined `push` in `execFast` consumes the final slot `j + 1 == room` or skips `j == MinElems`, Ekuma's adaptive push-size hint (`observePush`) never observes long push runs. Always require `j != storage.MinElems && j+1 < room` in `execFast`.
4. **Pitfall #4 — `TestArrayTruncationPolls` regression**: Truncating a 3M-element `PackedNums` array to 1M elements in O(1) without polling skips pending interrupts. Guard `execFast` truncation with `NumCount() - int(newLen) < storage.PollChunk`, and call `p.Poll()` in `TruncateElements` when removing `>= PollChunk` elements.
5. **Pitfall #5 — `TestAllocationsPerOperation/spread_of_an_array`**: Ensure `appendIterable` calls `ReserveNums` (not `ReserveElems`) when spreading a `PackedNums` array into an empty array so spread stays at 2 allocations.

---

## 8. `unsafe` Usage & Pure-Safe-Go Alternative

- **`unsafe` used**: Only a single `unsafe.String(unsafe.SliceData(buf), len(buf))` in `JoinStrings` (identical to `strings.Builder.String`). The entire `PackedNums` representation, promotion logic, `ArrayCtorPlain` cache, and all inlined `execFast` array paths are **100% safe Go**.
- **Safe-Go alternative**: Using `strings.Builder` in `JoinStrings` eliminates even that single `unsafe.String` with negligible (< 0.1 nats) difference.

---

## 9. Verification & Reproduction Protocol

1. Run `go test . ./internal/engine ./internal/builtin` — specifically check `TestPushCapacityAfterLongRuns`, `TestAllocationsPerOperation`, `TestArrayTruncationPolls`, `TestUnregisteredShortcutInterruption`, `TestKernels`, `TestExecFastPathsCallNothing`, and `joins-of-arrays-of-strings.js`.
2. Run `go test -tags ekumavalidate ./internal/engine` and `go test -race ./internal/storage ./internal/engine`.
3. Run `_bench` microbenchmarks (`-suite Micro -engine '^ekuma$' -rounds 1`) and verify: `array_length_decr` ≥ 2.8x faster, `array_slice` ≥ 2.4x faster, `array_pop` ≥ 2.2x faster, `array_push` ≥ 1.8x faster, `string_array_join` ≥ 1.6x faster. If `array_slice` does not drop below 3.5 µs, check `tryPromoteNums` activation!
