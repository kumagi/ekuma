# Optimization #7: `Number.prototype` Formatting Kernels and Single-Allocation 80-Byte `asciiText32` Strings

## 1. Summary & Measured Impact

- **Rank**: #7 of 8
- **Targeted Subset Impact**: **+0.86 nats** across the 6 number-formatting microbenchmarks (contributing to Stage 6A's **+1.98 nats** / 1.027x suite-wide lift alongside Optimization #8)
- **Key Benchmark Improvements (Stage 5 → Stage 6A, and Stage 5 → Final 3-Round Median)**:

| Benchmark | Stage 5 Ekuma | Stage 6A Ekuma | Final 3-Round Ekuma | Stage 5 → Final Speedup |
| --- | ---: | ---: | ---: | ---: |
| `int_toString` (`(j).toString()`) | 215.6 ns | 171.4 ns | 169.3 ns | **1.27x** (+0.24 nats) |
| `float_toExponential` (`(x).toExponential(d)`) | 524.6 ns | 445.6 ns | 428.0 ns | **1.23x** (+0.20 nats) |
| `float_toPrecision` (`(x).toPrecision(p)`) | 487.6 ns | 419.7 ns | 410.0 ns | **1.19x** (+0.17 nats) |
| `float_toString` (`(x).toString()`) | 440.2 ns | 381.8 ns | 376.0 ns | **1.17x** (+0.16 nats) |
| `float_toFixed` (`(x).toFixed(d)`) | 488.6 ns | 426.9 ns | 423.0 ns | **1.16x** (+0.14 nats) |
| `float_to_string` (`(x) + ''`) | 319.8 ns | 307.3 ns | 308.0 ns | **1.04x** (+0.04 nats) |

---

## 2. Root Cause Analysis

Comparing `int_to_string` (`(j) + ''`: 112 ns) with `int_toString` (`(j).toString()`: 216 ns), and `float_to_string` (320 ns) with `float_toString` (440 ns), revealed two clear overheads:

1. **~100 ns native call overhead per method invocation**: None of `Number.prototype.toString`, `toFixed`, `toPrecision`, or `toExponential` had quiet kernels registered (`internal/builtin/misc.go`). Every call pushed a native call frame, sliced arguments, dispatched through `invokeNative`, and popped the native frame — even though the common case (primitive `Number` receiver and `undefined` or primitive `Number` argument) executes zero JavaScript and allocates only the output string.
2. **Two heap allocations + deferred ASCII scan for 9–32 byte formatted numbers**: `RealmPart.NumberToString` and `ExecutionContext.NewStringBytes` converted stack `[]byte` buffers via `r.NewString(wtf8.String(text))`. For any formatted number longer than 8 bytes (almost every output of `float_toFixed`, `float_toPrecision`, `float_toExponential`, and large `float_toString`), that performed **two** heap allocations — `runtime.slicebytetostring` (16–32 B string buffer) plus `&textObject{}` (48 B) — and left `ObjASCIIKnown` unset even though formatted numbers are always 100% ASCII.

---

## 3. Prerequisites & Dependency Order

- **None** — completely independent of `execFast` and can be applied in any order.

---

## 4. Target Files Inventory

| File | Role |
| --- | --- |
| `internal/storage/stringvalue.go` | Add `asciiText32` (80-byte combined `textObject` + `[32]byte` struct) and `(*RealmPart).NewASCIIStringBytes(b []byte) Value`; route `NumberToString` through `NewASCIIStringBytes`. |
| `internal/quiet/context.go` | Forward `Context.NewASCIIStringBytes(b []byte)` to `q.Realm.NewASCIIStringBytes(b)`. |
| `internal/numconv/format.go` (new) | Move pure Go number-formatting routines (`AppendFixed`, `AppendPrecision`, `AppendExponential`, `AppendExponentialAuto`, `NumberFormatBufSize = 128`, and helpers `appendHalfUp`, `fixedPoint`, `scaleFraction`, `roundSignificant`, `appendDigits`, `isTie`, `fixExponent`, etc.) out of `internal/builtin/misc.go` so both `builtin` and `quiet` share one implementation with zero duplication. |
| `internal/quiet/numberkernels.go` (new) | Implement `Method0` and `Method1` quiet kernels for `toString`, `toFixed`, `toPrecision`, and `toExponential`. |
| `internal/builtin/misc.go` | Register all 4 `Number.prototype` methods via `RegisterNativeWithKernels(..., Contract: shortContract)`; delegate `numberToFixed`, `numberToPrecision`, and `numberToExponential` to `numconv.Append*` + `cx.NewASCIIStringBytes`; remove duplicated helpers and unused `bytes`/`strconv` imports. |
| `internal/engine/testdata/storagemembers.txt` | Add `RealmPart.NewASCIIStringBytes` in alphabetical order. |

---

## 5. Detailed Design Specification (No Code)

### 5.1 Single-Allocation `asciiText32` & `NewASCIIStringBytes` (`internal/storage/stringvalue.go`)

1. Define `type asciiText32 struct { textObject; buf [32]byte }`. On 64-bit Go, `textObject` (48 B) + `[32]byte` = **80 bytes** — an exact `mallocgc` size class!
2. Implement `(*RealmPart).NewASCIIStringBytes(b []byte) Value` (caller guarantees all bytes in `b` are `< 0x80`):
   - `len(b) == 0`: return `r.Strings.Empty`.
   - `len(b) == 1`: return `r.Strings.Character(b[0])`.
   - `len(b) <= shortstrings.MaxBytes` (8): return `r.ShortString(wtf8.String(b))` (zero allocations).
   - `len(b) <= 32`: allocate a single `&asciiText32{...}` with `Kind: KindString`, `flags: ObjASCIIKnown | ObjASCII`, `shape: r.Shapes.String`, `objectTail: &zeroTail`, copy `b` into `t.buf[:]`, set `t.text = wtf8.String(unsafe.String(&t.buf[0], len(b)))`, and return `Value{O: newText(&t.textObject)}`. **One 80-byte allocation instead of two, with `ObjASCIIKnown|ObjASCII` already populated.**
   - `len(b) > 32`: allocate `&textObject{Kind: KindString, flags: ObjASCIIKnown | ObjASCII, shape: r.Shapes.String, text: wtf8.String(b), objectTail: &zeroTail}`.
3. Simplify `(*RealmPart).NumberToString(n float64) Value` to format into a stack `var buf [32]byte` via `numconv.AppendNumber(buf[:0], n)` and return `r.NewASCIIStringBytes(...)`.
4. Add forwarding method `(q Context) NewASCIIStringBytes(b []byte) storage.Value` in `internal/quiet/context.go`.

### 5.2 Extract Pure Formatting Functions (`internal/numconv/format.go`)

Move the pure number-formatting helpers from `internal/builtin/misc.go` into `internal/numconv/format.go` (zero imports from `storage` or `engine`) and expose four clean entry points alongside `const NumberFormatBufSize = 128`:

1. `AppendFixed(b []byte, n float64, d int) []byte`: formats finite `|n| < 1e21` with `0 <= d <= 100` (converting `-0` to `"0..."` by stripping any leading `'-'` when `n == 0`).
2. `AppendPrecision(b []byte, n float64, p int) []byte`: formats finite `n` with `1 <= p <= 100` (handling `n == 0` via `appendHalfUp(b, 0, 'f', p-1)`, trying fast `appendPrecisionFast`, falling back to `appendHalfUp` with `'e'` + `fixExponent` or `precisionFixed`).
3. `AppendExponentialAuto(b []byte, n float64) []byte`: formats finite `n` when the argument is `undefined` (normalizing `-0` to `+0`, calling `fixExponent(strconv.AppendFloat(b, n, 'e', -1, 64))`).
4. `AppendExponential(b []byte, n float64, d int) []byte`: formats finite `n` with `0 <= d <= 100` (normalizing `-0` to `+0`, trying fast `appendExponentialFast`, falling back to `fixExponent(appendHalfUp(b, n, 'e', d))`).

Update `numberToFixed`, `numberToPrecision`, and `numberToExponential` in `internal/builtin/misc.go` to call these four functions and wrap the output with `cx.NewASCIIStringBytes(...)`. Keep `"math/bits"` imported in `misc.go` (it is still used by `Math.clz32`), and remove unused `"bytes"` and `"strconv"`.

### 5.3 Spec-Exact `Number.prototype` Kernels (`internal/quiet/numberkernels.go`, `internal/builtin/misc.go`)

Define helper `toIntRange(v storage.Value, lo, hi float64) (int, bool)` modeling `ToIntegerOrInfinity` for `undefined` and `Number` inputs:
- `v.IsUndefined()`: returns `(0, lo <= 0 && 0 <= hi)`.
- `!v.IsNumber()`: returns `(0, false)` (declines so slow path invokes `valueOf`/`toString` or throws).
- `f := v.Number()`: if `f != f || f == 0` (NaN or ±0), returns `(0, lo <= 0 && 0 <= hi)` because `ToIntegerOrInfinity(NaN) == +0`! If `math.IsInf(f, 0)`, returns `(0, false)`. Otherwise `i := math.Trunc(f)`; return `(int(i), i >= lo && i <= hi)`.

Implement 8 kernel functions (all decline when `!this.IsNumber()`, letting slow path handle boxed `new Number(...)` or throw `TypeError`), carefully preserving **ECMAScript's subtle exception-vs-non-finite ordering** across the four methods:

1. **`NumberToStringKernel0(q, this)`**: return `q.NumberToString(this.Number()), true`.
2. **`NumberToStringKernel1(q, this, radixVal)`**:
   - If `radixVal.IsUndefined()`, return `q.NumberToString(this.Number()), true`.
   - If `!radixVal.IsNumber()`, decline.
   - If `f` is `NaN` or infinite, or `r := math.Trunc(f)` is `< 2 || > 36`, decline (slow path throws `RangeError`).
   - If `r == 10`, return `q.NumberToString(this.Number()), true`.
   - Otherwise return `q.NewString(numconv.NumberToRadixString(this.Number(), int(r))), true`.
3. **`NumberToFixedKernel0` & `NumberToFixedKernel1(q, this, arg)`**:
   - **Check range BEFORE `NaN`/`Infinity`!** Call `d, ok := toIntRange(arg, 0, 100)`; if `!ok`, decline (e.g., `(NaN).toFixed(-1)` MUST throw `RangeError`!).
   - Then if `n != n || math.IsInf(n, 0) || math.Abs(n) >= 1e21`, return `q.NumberToString(n), true`.
   - Otherwise format into stack `[128]byte` via `numconv.AppendFixed` and return `q.NewASCIIStringBytes(...)`.
4. **`NumberToPrecisionKernel0` & `NumberToPrecisionKernel1(q, this, arg)`**:
   - `Method0` or `arg.IsUndefined()`: return `q.NumberToString(n), true`.
   - If `!arg.IsNumber()`, decline (argument coercion runs *before* the non-finite check!).
   - **Check `NaN`/`Infinity` BEFORE range!** If `n != n || math.IsInf(n, 0)`, return `q.NumberToString(n), true` (e.g., `(NaN).toPrecision(0)` and `(NaN).toPrecision(Infinity)` MUST return `"NaN"`, NOT throw `RangeError`!).
   - Then `p, ok := toIntRange(arg, 1, 100)`; if `!ok`, decline.
   - Format via `numconv.AppendPrecision` and return `q.NewASCIIStringBytes(...)`.
5. **`NumberToExponentialKernel0` & `NumberToExponentialKernel1(q, this, arg)`**:
   - `Method0` or `arg.IsUndefined()`: if `n` is `NaN`/`Inf`, return `q.NumberToString(n), true`; else format via `numconv.AppendExponentialAuto`.
   - If `!arg.IsNumber()`, decline.
   - **Check `NaN`/`Infinity` BEFORE range!** If `n != n || math.IsInf(n, 0)`, return `q.NumberToString(n), true`.
   - Then `d, ok := toIntRange(arg, 0, 100)`; if `!ok`, decline.
   - Format via `numconv.AppendExponential` and return `q.NewASCIIStringBytes(...)`.

Register all four (`nativeNumberToString`, `nativeNumberToFixed`, `nativeNumberToPrecision`, `nativeNumberToExponential`) in `internal/builtin/misc.go` with `Method0`, `Method1`, and `Contract: shortContract`.

---

## 6. Architectural Invariants & White-Box Test Updates

- **`internal/engine/testdata/storagemembers.txt`**: Add `RealmPart.NewASCIIStringBytes` in sorted order (between `RealmPart.KeyOfString` and `RealmPart.NewArray`) so `TestStorageBorder` passes.
- **`TestKernels`**: Automatically probes all four methods against every operand in `kernel-probes.js` (`NaN`, `-0`, `-0.4`, `-1.5`, `Infinity`, `1e300`, `Number object`, `converting object`, `throwing conversion`, `undefined`, etc.) and asserts 100% identical outputs, errors, and hook traces between kernel and slow path!

---

## 7. Known Pitfalls & Failed Approaches to Avoid

1. **Pitfall #1 — Exception ordering mismatch between `toFixed` vs `toPrecision`/`toExponential`**:
   - `(NaN).toFixed(-1)` throws `RangeError` (range check first).
   - `(NaN).toPrecision(0)` and `(NaN).toExponential(-1)` return `"NaN"` (non-finite check first, *after* `ToIntegerOrInfinity` coercion!).
   - `(NaN).toPrecision(convertingObj)` MUST still call `convertingObj.valueOf()` — that is why `!arg.IsNumber()` must decline *before* checking `n != n || math.IsInf(n, 0)`!
2. **Pitfall #2 — `ToIntegerOrInfinity(NaN)` is `0`, not a `RangeError`**: Passing `NaN` to `toFixed(NaN)` or `toExponential(NaN)` behaves as `0` digits (succeeds), while `toPrecision(NaN)` produces `0 < 1` (`RangeError` unless `this` is non-finite). `toIntRange` handles both cases uniformly.
3. **Pitfall #3 — `TestStorageBorder` failure**: Calling `cx.NewASCIIStringBytes(...)` from `internal/builtin/misc.go` resolves to the embedded `storage.RealmPart.NewASCIIStringBytes`, which `TestStorageBorder` checks against `internal/engine/testdata/storagemembers.txt`. Remember to add it there.

---

## 8. `unsafe` Usage & Pure-Safe-Go Alternative

- **`unsafe` used**: Only a single `unsafe.String(&t.buf[0], len(b))` inside `asciiText32` (`internal/storage/stringvalue.go`). All four kernels and `internal/numconv/format.go` are **100% pure safe Go**.
- **Safe-Go alternative**: Replacing the `len(b) <= 32` `asciiText32` branch with `&textObject{flags: ObjASCIIKnown | ObjASCII, text: wtf8.String(b)}` removes all `unsafe` from this technique while keeping ~80% of the kernel speedup (costing one extra small allocation on 9–32 byte formatted numbers).

---

## 9. Verification & Reproduction Protocol

1. Run `go test ./internal/numconv ./internal/quiet ./internal/builtin ./internal/engine` — specifically confirm `TestKernels`, `TestStorageBorder`, and `FuzzNumberWithPrecision` (`internal/builtin/number_test.go`) pass.
2. Run `go test -tags ekumavalidate ./internal/engine`.
3. Run `_bench` microbenchmarks (`-suite Micro -engine '^ekuma$' -rounds 1`) and verify `int_toString`, `float_toString`, `float_toFixed`, `float_toPrecision`, and `float_toExponential` all drop by ~40–70 ns per operation (~1.15x–1.26x faster).
