# Optimization #9: Dual-Mode Quiet-Kernel Loop (`execQuiet`), Direct `collection.Table` Probing, Small-Int ShortString Packing, and Primitive `Number` IC Fix

## 1. Summary & Measured Impact

- **Overall Rank**: #3 of 13
- **Suite-Wide Impact**: **+5.05 nats** across all 74 microbenchmarks (+1.46 nats in Stage 9 + ~3.59 nats from Stage 11 loop-glue expansion across number formatting, string concatenation, `Date.now`, and collection loops)
- **Targeted Subset Impact**: **+5.64 nats** across `Map`, integer/float stringification, and quiet-kernel loops
- **Key Benchmark Improvements (Phase 1 → Final 3-Round Median)**:

| Benchmark | Phase 1 Ekuma | Final Ekuma | Phase 2 Speedup | Overall vs Baseline |
| --- | ---: | ---: | ---: | ---: |
| `int_toString` (`i.toString()`) | 169.3 ns | 67.2 ns | **2.52x** (+0.92 nats) | **2.36x** |
| `int_to_string` (`"" + i` / `i + ""`) | 115.1 ns | 49.5 ns | **2.32x** (+0.84 nats) | **2.10x** |
| `map_set_string` (string-keyed `Map.set`) | 282.9 µs | 156.5 µs | **1.81x** (+0.59 nats) | **1.76x** |
| `map_delete` (string-keyed `Map.set` + `delete`) | 319.2 µs | 191.1 µs | **1.67x** (+0.51 nats) | **1.65x** |
| `map_set_bigint` (`int64` BigInt-keyed `Map.set`) | 216.7 µs | 152.8 µs | **1.42x** (+0.35 nats) | **1.45x** |
| `float_toString` (`x.toString()`) | 375.5 ns | 287.7 ns | **1.31x** (+0.27 nats) | **1.32x** |
| `float_toFixed` (`x.toFixed(10)`) | 422.9 ns | 341.6 ns | **1.24x** (+0.21 nats) | **1.23x** |
| `map_set_int` (integer-keyed `Map.set`) | 143.4 µs | 117.8 µs | **1.22x** (+0.20 nats) | **1.48x** |
| `float_toPrecision` (`x.toPrecision(10)`) | 409.8 ns | 338.9 ns | **1.21x** (+0.19 nats) | **1.26x** |
| `float_toExponential` (`x.toExponential(10)`) | 427.8 ns | 353.2 ns | **1.21x** (+0.19 nats) | **1.34x** |
| `float_to_string` (`"" + x`) | 307.9 ns | 267.2 ns | **1.15x** (+0.14 nats) | **1.15x** |

---

## 2. Root Cause Analysis

Four independent bottlenecks throttled built-in kernel loops, `Map` operations, and number-to-string conversions after Phase 1:

1. **Per-iteration `execFast ↔ exec` bounce on quiet-kernel loops**: Because `execFast` must remain strictly call-free (`TestExecFastPathsCallNothing`), every loop whose body invoked a quiet native kernel (`Map.prototype.set`, `Map.prototype.delete`, `Number.prototype.toFixed`, `Date.now`, `RegExp.prototype.exec`, etc.) exited `execFast` into slow `exec`, ran the kernel through `callMethod` (which constructed a Go slice argument and performed indirect dispatch), and immediately re-entered `execFast` for the loop counter — paying full frame/pointer teardown and re-entry on every single iteration.
2. **100% cache miss on primitive `Number` `OpGetMethod` and `OpGetProp`**: In `execFast`, `OpGetMethod` and `OpGetProp` compared `o.Shape()` against the inline cache entry's `Shape`. For primitive numbers, `o == storage.NumberObject`, whose `Shape()` is `nil`, whereas `getNamed` cached `%Number.prototype%.Shape()` (`CacheOwn`). Consequently, **every single call** to `n.toString()`, `n.toFixed()`, `n.toPrecision()`, and `n.toExponential()` missed the inline cache in `execFast` and bounced to slow `exec`.
3. **Slow string and `BigInt64` hashing/probing in `internal/collection/table.go`**: `Table.find` and `Table.Set` extracted full `wtf8.String` views, hashed byte-by-byte, and called `storage.SameValueZero` on every probe slot — even though short strings (≤ 8 bytes) already carry their exact little-endian zero-padded bytes inside `Value.N` (`storage.Value.PackedShortString()`), and `int64` BigInts already fit in a single 64-bit word (`Value.BigInt64()`).
4. **Allocation-heavy small-integer `NumberToString` and `OpAdd` empty-string coercion**: `RealmPart.NumberToString` only cached integers `0..999` and fell through to `numconv.AppendNumber` + `strconv.AppendInt` + `NewASCIIStringBytes` for `1000..99_999_999` — even though every integer in `[-9_999_999, 99_999_999]` fits in ≤ 8 ASCII bytes and can be packed directly into a zero-allocation `ShortStrings[n]` short-string `Value` in one `uint64` word. Furthermore, `"" + x` and `x + ""` (compiled as `OpAdd` rather than `OpConcatK`) called `addValues` → `toStringValue` → `ag.reload` instead of calling `NumberToString` directly.

---

## 3. Prerequisites & Dependency Order

- **Depends on Optimization #2** (`execFast` leaf interpreter loop) and **Optimization #7** (`Number.prototype` kernels).
- Must be implemented before or alongside **Optimization #11** (`RegExp`/`Date` kernels) and **Optimization #12** (exclusive BigInt accumulation), both of which execute directly inside `execQuiet`.

---

## 4. Target Files Inventory

| File | Role |
| --- | --- |
| `internal/engine/quiet_exec.go` (new file) | Implement `(*agent).execQuiet(cx *ExecutionContext, fp *storage.FuncProto, pc int, regs []storage.Value) int`, the dual-mode interpreter loop for quiet-kernel and non-leaf hot loops. |
| `internal/engine/interp.go` | Translate `storage.NumberObject` to `%Number.prototype%` in `execFast` `OpGetMethod` and `OpGetProp`; handle small-int `OpConcatK` directly in `execFast`; fast-path `Empty + Number` / `Number + Empty` in slow `OpAdd` and `OpConcatK`; remove redundant `ag.reload` after quiet-kernel calls and transition into `execQuiet` (declining `Array.prototype.push`/`pop`, `Extremum`, and `OpAppend`/`OpPrepend`). |
| `internal/storage/stringvalue.go` | Add `(*Value).PackedShortString() (uint64, int, bool)` and direct `uint64` little-endian decimal packing in `RealmPart.NumberToString` for `[-9_999_999, 99_999_999]`. |
| `internal/collection/table.go` | Add `probeShortString` and `probeBigInt64` fast paths in `Table.find` and `Table.Set` with inline no-grow append. |
| `internal/storage/object.go` | Add `(*RealmPart).NewOrdinaryObjectWithRoom(n int) *Object` for zero-raw-access `OpNewObject` inside `execQuiet`. |
| `internal/storage/array.go` | Add `(*RealmPart).SetPlainArrayElement(o *Object, j uint32, v Value) bool` for guarded `OpSetIndex` inside `execQuiet`. |
| `internal/engine/rawaccess_test.go` | Allowlist `"agent.execQuiet": 3` under `"CacheEntry.OwnValue"`. |
| `internal/engine/testdata/rawusers.txt` | Allowlist `PropCache.CacheEntry:ref*3 PropCache.Second:ref*3` for `agent.execQuiet` and `Value.N:lit` / `shapeTable.ShortStrings:ref` for `agent.execFast`. |
| `internal/engine/testdata/storagemembers.txt` | Add `Value.PackedShortString` in sorted order for `TestStorageBorder`. |

---

## 5. Detailed Design Specification (No Code)

### 5.1 Dual-Mode Quiet-Kernel Interpreter (`internal/engine/quiet_exec.go`)

1. **Signature & context boundary**: Define `func (ag *agent) execQuiet(cx *ExecutionContext, fp *storage.FuncProto, pc int, regs []storage.Value) int`. Pass `cx *ExecutionContext` explicitly as a parameter instead of reading `ag.active` inside `execQuiet`, satisfying `TestExecutionContextBoundary`.
2. **Entry condition**: Enter `execQuiet` from slow `exec` (only when `!fp.HeapRegs`) immediately after:
   - Any quiet native kernel call (`k.callMethod` / `k.number1` / `k.number2`) succeeds, **except** `IntrinsicArrayPrototypePush`, `IntrinsicArrayPrototypePop`, and `info.Extremum != ExtremumNone` (which must stay on `execFast`'s zero-call inline paths).
   - `OpConcatK` (`NumberObject + Empty` or `KindString`), `OpAdd` (`Empty + NumberObject`, `NumberObject + Empty`, or `KindString + KindString`), or heap-BigInt `OpAccum`/`OpMulAdd`/`OpMulAccum`.
   - **Never** enter `execQuiet` from `OpAppend`, `OpAppendK`, `OpPrepend`, or `OpPrependK` (those have zero-call byte-copy loops in `execFast`).
3. **Supported instructions inside `execQuiet`**:
   - **Constants & moves**: `OpMove`, `OpLoadK`, `OpLoadInt`, `OpLoadUndef`, `OpLoadNull`, `OpLoadTrue`, `OpLoadFalse`, `OpSeal`, `OpNewObject` (via `cx.NewOrdinaryObjectWithRoom`).
   - **Globals & properties**: `OpGetGlobal*` (cache hit only), `OpGetProp` (with `NumberObject → IntrinsicNumberPrototype` translation and `CacheOwn`/`CacheProto` hit via `e.OwnValue(o)`), `OpGetLength` (`KindArray`, `KindString` via `v.StringLength()`, and `CacheOwn`/`CacheProto`), `OpGetIndex`, `OpGetIndexInt`, and `OpSetIndex`/`OpSetIndexStrict` (via `cx.SetPlainArrayElement`).
   - **Arithmetic & stringification**: `OpAdd` (two numbers, `Empty + Number`, `Number + Empty`, or two strings via `cx.Concat`), `OpAddK`, `OpSub`, `OpSubK`, `OpMul`, `OpMulK`, `OpDiv`, `OpDivK`, `OpMod`, `OpModK` (non-negative exact `int64`), `OpBitAnd`, `OpBitAndK`, `OpBitOr`, `OpBitOrK`, `OpBitXor`, `OpShl`, `OpShr`, `OpUShr` (with `numconv.InInt64Range`), `OpConcatK` (`KindString` or `Number + Empty`), plus `OpAccum`/`OpMulAdd`/`OpMulAccum` (numbers, `BigInt64Object`, and heap BigInts).
   - **Control flow**: `OpJump`, `OpJumpIf`, `OpJumpIfNot`, `OpInc` (fused with `OpLoopLt`/`OpLoopLtK`/`OpLoopLe`/`OpLoopLeK` when `!ag.InterruptFlag.Load()`), `OpDec` (fused with `OpLoopGt`/`OpLoopGtK`/`OpLoopGe`/`OpLoopGeK`), and standalone `OpLoopLt`/`Le`/`Gt`/`Ge` + `*K` variants (all declining when `ag.InterruptFlag.Load()` is set).
   - **RegExp literals**: `OpRegExpMethod` (calling `cx.regExpLiteralMethodFast`, declining on cache miss) and `OpRegExpArg` (calling `cx.lendRegExpLiteral`).
   - **Fused `OpGetMethod` + `OpCall` / `OpCallUndef`**: Resolve the method through the 2-entry `PropCache` (translating `NumberObject → IntrinsicNumberPrototype`), fuse with an immediately following `OpCall` on the same register, check `!ag.InterruptFlag.Load()`, verify `f.Fn.Info.Native.HasFast()` and `info.Realm == cx.realmPart` while declining `push`, `pop`, and `Extremum != ExtremumNone`, and dispatch `k.number1`, `k.number2`, `k.method0`, `k.method1`, or `k.method2` **directly inline** without constructing a `[]Value` slice or calling `callMethod`. Update `ag.frames[len(ag.frames)-1].pc = pc` before calling the kernel so any poll interruption records the exact PC.
4. **Validation invariant (`-tags ekumavalidate`)**: Call `ag.checkStep(fp, pc, regs)` at the top of the `execQuiet` loop. Whenever `execQuiet` declines an instruction and returns `pc - 1`, call `ag.enterChecks()` under `if validateExec` so `execFast` can re-validate that PC without triggering a "continued at same PC" panic. Exception: if `OpGetMethod` succeeded and fused `OpCall` declined, do **not** call `enterChecks()` (track with a `fused` boolean), because `execGetMethod` already stepped and `execFast` will next validate `OpCall`.

### 5.2 Primitive `Number` Inline-Cache Fix & `OpAdd`/`OpConcatK` Fast Paths (`internal/engine/interp.go`)

1. In `execFast` `case bytecode.OpGetMethod:` and `case bytecode.OpGetProp:`, immediately after checking `o != nil`, add: if `o == storage.NumberObject`, reassign `o = ag.active.realm.Intrinsics[storage.IntrinsicNumberPrototype]`. Notice `v` (still holding the primitive number) remains written to `regs[a+1]` as `this` for `OpGetMethod`, and `checkEntryRead` still validates `(v, o, key, e)`.
2. In `execFast` `case bytecode.OpConcatK:`, fast-path `x.O == storage.NumberObject && y.O == ag.active.realm.Strings.Empty.O` when `x.N` is an exact integer in `0..999` (returning `ag.active.realm.Strings.SmallInt(int(i))` or packing a ≤ 8-byte short string directly in `execFast` using `ShortStrings[n]`).
3. In slow `exec` `case bytecode.OpAdd:`, before calling `addValues`, handle `x.O == Empty.O && y.O == NumberObject` and `x.O == NumberObject && y.O == Empty.O` by calling `ag.active.NumberToString(...)` and entering `execQuiet` (skipping `ToPrimitive` and `ag.reload`).
4. In slow `exec` after a successful quiet-kernel call, remove `ag.reload` (quiet kernels cannot execute user JS or reallocate the VM register stack) and immediately invoke `pc = ag.execQuiet(ag.active, fp, pc, regs)` when `!fp.HeapRegs` and `f` is not `Array.prototype.push`/`pop`.

### 5.3 Direct Small-Integer Short-String Packing (`internal/storage/stringvalue.go`)

1. Export `func (v Value) PackedShortString() (bits uint64, n int, ok bool)`: returns `(math.Float64bits(v.N), len(v.O.Text()), true)` when `v.O != nil && v.O.Kind == KindString && len(v.O.Text()) <= shortstrings.MaxBytes`. (This relies on Ekuma's universal invariant in `storage/value.go` that every string of ≤ 8 bytes — including interned constants via `ObjectValue` — stores its zero-padded little-endian bytes in `Value.N`.)
2. In `(*RealmPart).NumberToString(n float64) Value`, after the `0..999` small-int table check, check if `i := int64(n); float64(i) == n && i >= -9_999_999 && i <= 99_999_999`:
   - Pack the ASCII decimal digits directly into a `uint64` word in little-endian order (extracting digits from least significant to most significant, shifting the accumulator left by 8 each step, prepending `'-'` if negative).
   - Return `Value{O: r.Shapes.ShortStrings[length], N: math.Float64frombits(word)}` — **zero heap allocations, zero `strconv`, zero `memmove`**!

### 5.4 Direct Short-String and `BigInt64` Probing in `collection.Table` (`internal/collection/table.go`)

1. Add Fibonacci mix helper for 64-bit keys: multiply the 64-bit word (`bits ^ uint64(n)<<56` for short strings, `uint64(i)` for `BigInt64`) by `0x9e3779b97f4a7c15` and shift/xor to produce a well-distributed `uint64` hash.
2. Add specialized `probeShortString(bits uint64, n int, h uint64)` and `probeBigInt64(v int64, h uint64)` loops that compare the stored entry key directly via `e.key.PackedShortString()` (comparing two registers `eb == bits && en == n`!) or `e.key.BigInt64()` (`ei == v`) instead of calling `storage.SameValueZero` or hashing byte slices.
3. In `Table.find` and `Table.Set`, dispatch immediately on `k.IsNumber()`, `k.PackedShortString()`, and `k.BigInt64()` before falling back to generic `hashKey` / `SameValueZero`. When inserting a new entry into `Set` without triggering a rehash (`len(t.entries) < cap(t.entries)` and load factor below threshold), append directly into `t.entries` and link the bucket slot inline.

---

## 6. Architectural Invariants & White-Box Test Updates

- **`TestExecutionContextBoundary`**: `execQuiet` takes `cx *ExecutionContext` explicitly and never reads `ag.active`.
- **`TestStorageBorder`**: Add `Value.PackedShortString` to `internal/engine/testdata/storagemembers.txt` so `internal/collection` uses only exported border-approved `storage.Value` methods (`IsNumber`, `Number`, `BigInt64`, `PackedShortString`).
- **`TestRawStorage` & `TestUnguardedOperations`**: Update `internal/engine/testdata/rawusers.txt` (`agent.execQuiet` and `agent.execFast`) and `internal/engine/rawaccess_test.go` (`CacheEntry.OwnValue` count `3` in `agent.execQuiet`).
- **`-tags ekumavalidate` step validation**: Every declined return in `execQuiet` must call `ag.enterChecks()` unless `fused == true`.

---

## 7. Known Pitfalls & Failed Approaches to Avoid

1. **Pitfall #1 — Trapping `Array.prototype.push` / `pop` or `OpAppend` / `OpPrepend` inside `execQuiet` causes severe regressions**: In Stage 9, letting `execQuiet` run `ArrayPushKernel` regressed `array_push` from **12.9 µs to 19.6 µs**; in Stage 11B, letting `execQuiet` handle `OpAppend`/`OpPrepend` regressed all seven `string_build*` benchmarks by **2.2x** (`9.5 µs → 22 µs`). **Rule**: `execQuiet` must ONLY execute operations that are NOT zero-call inlined in `execFast`; always decline `push`, `pop`, `Extremum`, and `OpAppend`/`OpPrepend` so control immediately returns to `execFast`.
2. **Pitfall #2 — Forgetting `ag.enterChecks()` on decline in `execQuiet`**: Under `-tags ekumavalidate`, `checkStep` rejects executing the same PC twice without an intervening `enterChecks()`. Always guard every non-fused decline with `if validateExec { ag.enterChecks() }`.
3. **Pitfall #3 — Raw `Value.O`/`Value.N` access inside `internal/collection/table.go`**: `TestStorageBorder` forbids packages outside `engine`/`storage` from reading `Value.O`/`Value.N` directly. Always route through `PackedShortString()`, `BigInt64()`, `IsNumber()`, and `Number()`.

---

## 8. `unsafe` Usage & Pure-Safe-Go Alternative

- **`unsafe` used**: **Zero** — 100% pure safe Go across `quiet_exec.go`, `collection/table.go`, and `storage/stringvalue.go`.

---

## 9. Verification & Reproduction Protocol

1. Run `go test ./internal/collection ./internal/engine ./internal/builtin`.
2. Run `go test -tags ekumavalidate -run 'TestKernels|TestExecutionContextBoundary|TestStorageBorder|TestRawStorage|TestUnguardedOperations|TestExecFastPathsCallNothing' ./internal/engine`.
3. Run `go test -race ./internal/collection ./internal/engine`.
4. Run `_bench` microbenchmarks (`-suite Micro -engine '^ekuma$' -rounds 1`) and verify: `map_set_string` drops from ~283 µs to ~155–170 µs, `map_delete` from ~319 µs to ~190–205 µs, `int_to_string` from ~115 ns to ~50 ns, `int_toString` from ~169 ns to ~67 ns, all five `float_to*` benchmarks improve 15–35%, and `array_push` / `string_build*` remain unregressed.
