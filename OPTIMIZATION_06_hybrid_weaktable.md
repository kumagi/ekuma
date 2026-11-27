# Optimization #6: Hybrid Ephemeron `WeakTable` (Inline Key Slot + Table-Side Fibonacci Hash Table for Primitives) and Weak Collection Kernels

## 1. Summary & Measured Impact

- **Rank**: #6 of 8
- **Suite-Wide Impact**: **+1.73 nats** across all 74 microbenchmarks (**1.024x** step geometric-mean lift in Stage 2)
- **Targeted Subset Impact**: **+2.10 nats** across the two WeakMap benchmarks
- **Key Benchmark Improvements (Stage 1B → Stage 2)**:

| Benchmark | Stage 1B Ekuma | Stage 2 Ekuma | Stage Speedup | Overall vs Baseline |
| --- | ---: | ---: | ---: | ---: |
| `weak_map_delete` (1,000 fresh keys inserted & deleted per WeakMap) | 897.5 µs | 247.7 µs | **3.62x** (+1.29 nats) | **3.41x** |
| `weak_map_set` (1,000 shared keys inserted into many ephemeral WeakMaps) | 317.8 µs | 141.0 µs | **2.25x** (+0.81 nats) | **2.43x** |

---

## 2. Root Cause Analysis

Implementing ECMAScript `WeakMap` ephemeron semantics in Go (entries must vanish when *either* the key or the `WeakMap` becomes unreachable, and a value referencing its own key must not prevent collection) exposed two conflicting workload patterns:

1. **Single-table key overhead (`weak_map_delete`)**: Baseline Ekuma stored every weak entry in a Go `map[weak.Pointer[WeakTable]]Value` inside a 136-byte `ObjectMeta` attached to the key, protected by a `sync.Mutex`, and registered every key on `WeakTable.keys` with `weak.Make` + `runtime.AddCleanup`. Even when a key belonged to only **one** `WeakMap` and held a primitive value, inserting it paid for: `ObjectMeta` allocation + `weakEntries` allocation + Go `map` allocation + `weak.Make(w)` + `WeakTable.keys` mutex lock + `runtime.AddCleanup` registration.
2. **Multi-table key thrashing (`weak_map_set`)**: Conversely, `weak_map_set` creates 1,000 persistent key objects once and inserts all 1,000 keys into *hundreds of short-lived `new WeakMap()` instances* between GCs (mapping each key to an integer `i`). Storing entries on the **key** causes 1,000 scattered per-key maps (or slices) to accumulate hundreds of dead `WeakTable` entries between GC cycles, causing severe L3 cache thrashing — and if a flat slice is used on the key, linear scanning degrades catastrophically to $O(M^2)$ (**2,034 µs — 6.3x slower!**).
3. **Native call overhead**: `WeakMap.prototype.{set,get,has,delete}` and `WeakSet.prototype.{add,has,delete}` lacked quiet kernels, paying full native frame setup on every call.

---

## 3. Prerequisites & Dependency Order

- **None** — completely independent of `execFast` and can be implemented in any order.

---

## 4. Target Files Inventory

| File | Role |
| --- | --- |
| `internal/storage/object.go` | Add compact `weakTail` (embedding `objectTail` + `weakEntries` in one allocation); add `(*Object).weak()` and `(*Object).ensureWeak()`; update `ExtState`, `SetExtState`, and `ensureMeta` to handle `*weakEntries` stored directly in `objectTail.Ext`. |
| `internal/storage/weaktable.go` | Redesign `WeakTable` into a 3-tier hybrid ephemeron structure: inline key-side `firstTable`/`first` slot, table-side open-addressed Fibonacci-hashed `prims []weakPrimEntry` for multi-table primitive values, and mutex-guarded key-side `extra.rest` + lazy `weakTableKeys` cleanup for non-primitive values. |
| `internal/quiet/collectionkernels.go` | Add `WeakMapData`, `WeakSetData`, `WeakMap{Set,Get,Has,Delete}Kernel`, and `WeakSet{Add,Has,Delete}Kernel`. |
| `internal/builtin/weak.go` | Register all 7 WeakMap/WeakSet kernels with `Contract: shortContract`. |
| `internal/engine/storage_test.go` | Add `Object.ensureWeak` to `objectStorageExceptions` (count: 1). |
| `internal/engine/testdata/kernel-probes.js` | Add `"WeakSet": () => new WeakSet,` to `operands` so `TestKernels` exercises `WeakSet` receivers. |

---

## 5. Detailed Design Specification (No Code)

### 5.1 Compact `weakTail` Allocation (`internal/storage/object.go`)

1. Define `type weakTail struct { objectTail; weak weakEntries }` so a plain object with no tail yet gets both its `objectTail` and its `weakEntries` in **one** heap allocation instead of allocating an `objectTail` plus a 136-byte `ObjectMeta` plus a `weakEntries`.
2. Allow `objectTail.Ext` to hold `*weakEntries` directly when the object has no other internal state (`Ext == nil`):
   - `ExtState()`: add `case *weakEntries: return nil`.
   - `SetExtState(v)`: if `o.Ext` is `*weakEntries`, ignore `v == nil`; otherwise promote via `o.ownTail().Ext = &ObjectMeta{weak: w, Ext: v}`.
   - `ensureMeta()`: if `t.Ext` is `*weakEntries`, wrap it in `&ObjectMeta{weak: w}` and store on `t.Ext`.
   - `weak() *weakEntries`: fast check `if w, ok := o.Ext.(*weakEntries)` -> return `w`; else if `m, ok := o.Ext.(*ObjectMeta)` -> return `m.weak`; else `nil`.
   - `ensureWeak() *weakEntries`:
     - If `o.Ext` is already `*weakEntries`, return it.
     - If `o.Ext` is `*ObjectMeta`, initialize `m.weak` if nil and return it.
     - If `o.objectTail == &zeroTail`: allocate `wt := &weakTail{}`, set `wt.Ext = &wt.weak`, assign `o.objectTail = &wt.objectTail`, and return `&wt.weak` (**1 allocation**!).
     - If `o.Ext == nil` (already has its own tail): allocate `w := &weakEntries{}`, assign `o.ownTail().Ext = w`, return `w`.
     - Otherwise (already has non-nil `Ext`): call `o.ensureMeta()`, allocate `m.weak`, return it.
   - **CRITICAL**: Always write through `o.ownTail().Ext = ...` (never `o.Ext = ...`) to satisfy `TestSharedTailWrites`!

### 5.2 Three-Tier Hybrid Ephemeron `WeakTable` (`internal/storage/weaktable.go`)

Design `weakEntries` (32 bytes: `firstTable weak.Pointer[WeakTable]`, `first Value`, `extra *weakExtra`) and `WeakTable` (`self weak.Pointer[WeakTable]`, `prims []weakPrimEntry`, `primsCount int`, `keys *weakTableKeys`):

1. **Key invariant — `isWeakPrimitive(v Value) bool`**:
   - Returns true iff `v.O == nil || v.O.Kind <= KindTrue || v.O == BigInt64Object` (i.e., `undefined`, `null`, `number`, `boolean`, or `int64` BigInt — values holding **zero** collectable heap references!).
   - **Why this is mathematically ephemeron-safe**: A primitive value can never reference its key (or any heap object), so storing `(weak.Pointer[Object](key), primitiveValue)` on the `WeakTable` can **never** form an ephemeron reference cycle. When the `WeakTable` dies, its `prims` slice dies automatically with zero cleanup callbacks; when the key dies, its `weak.Pointer[Object]` zeroes automatically and dead slots are reclaimed on rehash!

2. **Tier 1 — Inline single-table slot (`w.firstTable` / `w.first`)**:
   - On the first `t.Set(k, v)` for a key `k`: `t.self` is lazily initialized (`weak.Make(t)`), and `w.firstTable = table; w.first = v`.
   - If `isWeakPrimitive(v)` is true: **zero `weakExtra`, zero `weak.Make(k)`, zero `sync.Mutex`, zero `weakTableKeys`, zero `runtime.AddCleanup`!**
   - If `!isWeakPrimitive(v)` (value holds a heap object): allocate `ex := w.ensureExtra()`, set `ex.shared = true` *strictly before* publishing `ex.self` (`weak.Make(w)`) into `t.addKey(ex.self)` (which lazily allocates `t.keys` and registers `runtime.AddCleanup(t, (*weakTableKeys).clear, keys)`), and set `ex.firstTracked = true`.
   - **Lock-free fast path**: Because only the single JS agent goroutine ever transitions `ex.shared` from `false` to `true`, and does so *before* any background cleanup goroutine can ever learn about `w`, `Get`/`Has`/`Set`/`Delete` can test `if ex == nil || !ex.shared` with **zero mutex locking** whenever no cleanup can race on `w`! (Passes `go test -race` 100% cleanly!)

3. **Tier 2 — Table-side open-addressed Fibonacci hash table (`t.prims`) for multi-table primitive values**:
   - When `t.Set(k, v)` encounters a key whose `w.firstTable` is already occupied by *another live* `WeakTable` (`w.firstTable.Value() != nil`) **and** `isWeakPrimitive(v)` is true:
     - Ensure `ex := w.ensureExtra()` has a cached `ex.keyWeak = weak.Make(k.O)` and `ex.keyHash = newWeakKeyHash()` (a sequential `atomic.Uint64` ID mixed by Fibonacci constant `0x9e3779b97f4a7c15`, shifted right 32, clamped nonzero so `0` cleanly distinguishes never-used empty buckets from tombstones!).
     - Store `{key: ex.keyWeak, val: v, hash: ex.keyHash}` in `t.prims` (agent-only, zero mutex!):
       - Initial power-of-two capacity `initPrimCap = 16`.
       - Linear probing `(h & mask)`, stepping `(i + 1) & mask`.
       - Load factor threshold: grow/rehash when `t.primsCount * 10 >= len(t.prims) * 7` (70%). During `growPrims()`, prune dead entries (`e.key.Value() == nil`) and tombstones (`e.key == zero && e.hash != 0`); double capacity only if `live * 10 >= newCap * 5` (otherwise compact in-place at the same capacity!).
       - On `Delete`: if found in `t.prims`, clear `e.key = zero; e.val = Undefined` and leave `e.hash = 1` (tombstone marker) so probe chains stay unbroken.
     - Note: if a key previously had a non-primitive value in `w.extra.rest[table]` and is overwritten with a primitive (or vice versa), make sure the old location is deleted/updated so each `(table, key)` pair lives in at most one tier!

4. **Tier 3 — Key-side map (`ex.rest map[weak.Pointer[WeakTable]]weakVal`) for multi-table non-primitive values**:
   - When `w.firstTable` belongs to another live table and `!isWeakPrimitive(v)`:
     - Ensure `ex.shared = true` before calling `t.addKey(ex.self)`.
     - Lock `ex.mu`, delete any stale primitive entry in `t.prims`, store `ex.rest[table] = weakVal{val: v, tracked: true}`, and periodically sweep dead tables from `ex.rest` (`e.table.Value() == nil`) using an adaptive `ex.sweepAt` threshold (`max(minWeakSweep, 2*len(ex.rest))`).
   - Also keep `runtime.KeepAlive(t)` after reading `t.self` (and `runtime.KeepAlive(w)` in `weakTableKeys.clear`) so the GC cannot prematurely collect `t` while its weak pointer is being compared!

5. **Dead-slot reuse on `w.firstTable`**:
   - When `t.Set(k, v)` sees `w.firstTable` is non-zero but `w.firstTable.Value() == nil` (the previous owner `WeakTable` was already collected), reclaim `w.firstTable` directly for `t` (clearing `ex.firstTracked` if needed).

### 5.3 WeakMap & WeakSet Quiet Kernels (`internal/quiet/collectionkernels.go`, `internal/builtin/weak.go`)

1. In `internal/quiet/collectionkernels.go`:
   - Add `WeakMapData(this storage.Value) *storage.WeakTable` (checks `this.IsObject() && this.O.Class == storage.ClassWeakMap`) and `WeakSetData` (`ClassWeakSet`).
   - Implement 7 kernels — each validates receiver via `WeakMapData`/`WeakSetData` and key via `q.Realm.CanBeHeldWeakly(k)` (declining with `(Undefined, false)` if receiver is invalid, or for `Set`/`Add` if key is not weakly holdable so the slow path throws `TypeError`; note for `Get`/`Has`/`Delete`, non-holdable keys do **not** throw — they return `Undefined`/`False` directly!):
     - `WeakMapGetKernel` (`Method1`): returns `(v, true)` or `(Undefined, true)`.
     - `WeakMapHasKernel` (`Method1`): returns `(Bool(t.Has(k)), true)`.
     - `WeakMapDeleteKernel` (`Method1`): returns `(Bool(t.Delete(k)), true)`.
     - `WeakMapSetKernel` (`Method2`): calls `t.Set(k, v)` and returns `(this, true)` (only when `CanBeHeldWeakly(k)`!).
     - `WeakSetHasKernel` (`Method1`): returns `(Bool(t.Has(k)), true)`.
     - `WeakSetDeleteKernel` (`Method1`): returns `(Bool(t.Delete(k)), true)`.
     - `WeakSetAddKernel` (`Method1`): calls `t.Set(v, True)` and returns `(this, true)` (only when `CanBeHeldWeakly(v)`!).
2. In `internal/builtin/weak.go`, register all 7 with `engine.RegisterNativeWithKernels(..., engine.Kernels{..., Contract: shortContract})`.
3. In `internal/engine/testdata/kernel-probes.js`, add `"WeakSet": () => new WeakSet,` right after `"WeakMap": () => new WeakMap,`.

---

## 6. Architectural Invariants & White-Box Test Updates

- **`internal/engine/storage_test.go`**: Add `"Object.ensureWeak": {count: 1, reason: "gives the object a tail of its own with inline weak entries"}` to `objectStorageExceptions`.
- **`TestSharedTailWrites`**: Verify both `SetExtState` and `ensureWeak` write `Ext` exclusively through `o.ownTail().Ext`.
- **`TestKernels`**: Requires `Contract: shortContract` on all 7 registered natives and `"WeakSet"` in `kernel-probes.js`.

---

## 7. Known Pitfalls & Failed Approaches to Avoid

1. **Pitfall #1 — Replacing key-side `map` with a flat `[]weakEntry` slice caused a 6.3x regression on `weak_map_set`**: In our first experiment (`stage2_weakmap_only`), a flat slice on `weakEntries` improved `weak_map_delete` (931 µs → 273 µs) but degraded `weak_map_set` from **324 µs to 2,034 µs** because 1,000 keys accumulated hundreds of dead `WeakTable` entries between GCs and scanned them linearly on every `Set`. Table-side `prims` completely eliminates this pathology.
2. **Pitfall #2 — Storing non-primitive values in `t.prims` breaks ephemeron collection**: Never place objects, strings (that may be reachable/large), symbols, or heap BigInts in `t.prims`! A value pointing back to its key inside `t.prims` would keep the key alive forever via the `WeakTable`, failing `weak_test.go` ephemeron cycle tests. Restrict `isWeakPrimitive` strictly to nil / singletons / `BigInt64Object`.
3. **Pitfall #3 — Data race on `ex.shared` transition**: `ex.shared = true` MUST be written by the agent goroutine *before* `t.addKey(ex.self)` publishes `ex.self` to `weakTableKeys` (and `weakTableKeys.clear` runs on Go's cleanup goroutine). Once `ex.shared` becomes `true`, it never transitions back to `false`, and all subsequent accesses to `w.firstTable`/`w.first`/`ex.rest` lock `ex.mu`.
4. **Pitfall #4 — `Delete` / `Set` tier consistency**: When overwriting an existing key in `t` with a value of the opposite primitiveness (primitive → object or object → primitive), always clear the entry from the old tier (`t.deletePrim` or `delete(ex.rest, table)`) so `Get` cannot return a stale value.
5. **Pitfall #5 — Missing `runtime.KeepAlive(t)`**: Always call `runtime.KeepAlive(t)` after comparing `t.self` against `w.firstTable` or `ex.rest`; otherwise Go's GC may collect `t` mid-method and zero `t.self`!

---

## 8. `unsafe` Usage & Pure-Safe-Go Alternative

- **`unsafe` used**: **Zero** — 100% pure safe Go (using Go 1.24+'s standard `weak` package and `runtime.AddCleanup`).

---

## 9. Verification & Reproduction Protocol

1. Run race-detector weak collection tests: `go test -race -run Weak .` and `go test -race ./internal/engine ./internal/builtin`.
2. Run `go test -run 'TestKernels|TestSharedTailWrites|TestValueStorageOwnership' ./internal/engine` and `go test -tags ekumavalidate ./internal/engine`.
3. Run `_bench` microbenchmarks (`-suite Micro -engine '^ekuma$' -rounds 1`) and verify: `weak_map_delete` drops from ~900 µs to ~240–275 µs (≥ 3.2x faster) and `weak_map_set` drops from ~320 µs to ~130–145 µs (≥ 2.2x faster).
