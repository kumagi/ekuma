# Optimization #1: Exclusive In-Place String Building, Prepending, 64-Bit Short Writes, and Zero-Copy Concatenation

## 1. Summary & Measured Impact

- **Rank**: #1 of 8
- **Suite-Wide Impact**: **+8.97 nats** across all 74 microbenchmarks (Stage 4: +2.15 nats; Stage 6B: +6.82 nats)
- **Targeted Subset Impact**: **+9.78 nats** across the 8 string-building and concatenation benchmarks (measured from Stage 3 to Final 3-round median)
- **Key Benchmark Improvements (Baseline → Final 3-Round Median)**:

| Benchmark | Baseline Ekuma | Final Ekuma | Speedup vs Baseline | Stage 3 → Final Speedup |
| --- | ---: | ---: | ---: | ---: |
| `string_build3` (repeated prepend `"x" + r`, 1,000 iters) | 89.3 µs | 11.1 µs | **8.03x** | 7.76x (+2.05 nats) |
| `string_build_large2` (repeated prepend `"abcdef" + r`, 20,000 iters) | 2203 µs | 309 µs | **7.12x** | 6.67x (+1.90 nats) |
| `string_build1x` (`r = r + "x"`, 1,000 iters) | 24.5 µs | 9.31 µs | **2.64x** | 2.85x (+1.05 nats) |
| `string_build2` (parameter `r += "x"`, 1,000 iters) | 25.1 µs | 9.80 µs | **2.56x** | 2.89x (+1.06 nats) |
| `string_build1` (local `r += "x"`, 1,000 iters) | 24.6 µs | 9.85 µs | **2.50x** | 2.82x (+1.04 nats) |
| `string_build2c` (`r += "xy"`, 1,000 iters) | 25.0 µs | 10.2 µs | **2.46x** | 2.74x (+1.01 nats) |
| `string_build_large1` (`r += "abcdef"`, 20,000 iters) | 691 µs | 289 µs | **2.39x** | 2.49x (+0.91 nats) |
| `string_concat` (`String.prototype.concat.apply("", tab)`) | 1454 ns | 680 ns | **2.14x** | 2.07x (+0.73 nats) |

---

## 2. Root Cause Analysis

Baseline Ekuma already possessed an exclusive append optimization (`OpAppend` / `OpAppendK` + `OpSeal` + `ObjExclusive` + `*strings.Builder` / `*ReversedBuilder`), but four compounding bottlenecks limited its performance:

1. **No exclusive prepend bytecode**: Repeated prepending (`r = "x" + r`) compiled to generic `OpAdd` rather than an exclusive in-place prepend instruction. Even though `ReversedBuilder` avoided quadratic byte copying, every single iteration still allocated a brand-new `textObject` header and ran `ClearBuilder` on the previous head.
2. **Every append/prepend exited the fast interpreter loop**: Because buffered strings stored an opaque `*strings.Builder`, the leaf fast loop (`execFast`) could not inspect capacity or append bytes directly. Every iteration of `string_build1..large2` exited `execFast`, entered slow `exec`, dispatched to `ExecutionContext.Append` or `Concat`, called `appendInPlace`, copied bytes via `strings.Builder.Write`, and called `ExtendExclusive` (which triggered a `string` pointer write barrier even when the underlying slice did not reallocate).
3. **Late buffer promotion on exclusive loops**: `Concat` waited until `len >= MinBufferedLength` (32 bytes) *and* `ConcatCount >= BufferAfter` (3). Because strings up to 8 bytes stay packed as short strings with `ConcatCount == 0`, single-character append loops ran 23 separate heap allocations and `memoizedConcat` cache lookups (lengths 9 through 31) before creating a builder.
4. **Extra copying in `String.prototype.concat.apply("", tab)`**: `Function.prototype.apply` cloned the dense array `tab` into a temporary Go slice via `CreateListFromArrayLike`, and `stringConcat` concatenated arguments incrementally instead of computing the exact total ASCII length and allocating once.

---

## 3. Prerequisites & Dependency Order

- **Depends on Optimization #2** (`execFast` leaf interpreter loop and `writeU64LE` in `internal/engine/endian_le.go` / `endian_generic.go`) for the inline `OpAppend`/`OpAppendK`/`OpPrepend`/`OpPrependK` fast paths.
- The `StringBuf`, `OpPrepend`/`OpPrependK`, `ConcatASCIIStrings`, and `CallWithArray` pieces work independently even without `execFast`.

---

## 4. Target Files Inventory

| File | Role |
| --- | --- |
| `internal/bytecode/bytecode.go` | Declare `OpPrepend` and `OpPrependK` opcodes. |
| `internal/bytecode/operands.go` | Register operand metadata for `OpPrepend` and `OpPrependK`. |
| `internal/compiler/appends.go` | Add `emitPrepend`; extend `settleAppends` and `unsealedReads` to recognize `OpPrepend` and `OpPrependK`. |
| `internal/compiler/expr.go` | Recognize `local = value + local` and emit `emitPrepend` when `appends(name, local, value)` holds. |
| `internal/storage/concat.go` | Replace `*strings.Builder` with `*StringBuf`; add `WriteReversed`, `WriteReversedBytes`, `ExtendHeadExclusive`, and `RealmPart.ConcatASCIIStrings`. |
| `internal/storage/object.go` | Add compile-time assertion verifying `wtf8.String` occupies exactly 2 pointer-sized words (`ptr`, `len`). |
| `internal/storage/raw.go` | Add `Raw.TailBuf()`, `Raw.HeadBuf()`, and `Raw.SetBufLen(n int)`. |
| `internal/quiet/array.go` | Forward `Context.ConcatASCIIStrings` to `RealmPart.ConcatASCIIStrings`. |
| `internal/engine/concat.go` | Migrate builder calls to `*storage.StringBuf`; add `Prepend`, `prependInPlace`, short-ASCII fast path in `Concat` for `HeadBuilder`, and eager exclusive buffering at `BufferAfter` with initial capacity ≥ 64. |
| `internal/engine/interp.go` | Implement `OpAppend`/`OpAppendK` and `OpPrepend`/`OpPrependK` fast paths in `execFast` (with 64-bit `writeU64LE` / `bits.ReverseBytes64` and `SetBufLen`) and slow-path handlers in `exec`. |
| `internal/engine/runtime.go` | Add `ExecutionContext.CallWithArray` to invoke a callable directly using a plain array's backing `[]Value` slice without cloning. |
| `internal/builtin/builtin.go` | Use `CallWithArray` in `functionApply` when the argument list is a plain array. |
| `internal/builtin/string.go` | Use `ConcatASCIIStrings` as a fast path in `stringConcat`. |
| `internal/engine/fastpath_test.go`, `internal/engine/rawaccess_test.go`, `internal/engine/testdata/rawusers.txt` | Update white-box allowlists and call counts. |

---

## 5. Detailed Design Specification (No Code)

### 5.1 Replace `*strings.Builder` with `storage.StringBuf` (`internal/storage/concat.go`)

1. Define a struct `StringBuf` containing a single exported byte slice field `Buf []byte`.
2. Implement small, inlineable methods on `*StringBuf`:
   - `Len() int`: returns `len(b.Buf)`.
   - `Room() int`: returns `cap(b.Buf)`.
   - `Extend(n int) []byte`: reslices `b.Buf` to `b.Buf[:n]` and returns `b.Buf`.
   - `Grow(n int)`: if `cap(b.Buf) - len(b.Buf) < n`, grows `b.Buf` via `slices.Grow(b.Buf, n)`.
   - `WriteString(s string)`: appends `s...` to `b.Buf`.
   - `Write(p []byte)`: appends `p...` to `b.Buf`.
   - `String() string`: returns `unsafe.String(unsafe.SliceData(b.Buf), len(b.Buf))` (zero-copy view, identical to `strings.Builder.String`).
3. Change `ReversedBuilder` to hold `Sb StringBuf` as its **first** field (offset 0).
4. Update `TailBuilder(o *Object) *StringBuf`, `ExtendExclusive(sb *StringBuf, ascii bool)`, and `NewBufferedString` to accept `*StringBuf` instead of `*strings.Builder`.
5. Optimize `WriteReversed(sb *StringBuf, s wtf8.String)` and add `WriteReversedBytes(sb *StringBuf, p []byte)`:
   - Instead of writing one byte at a time via `WriteByte`, call `sb.Grow(len(s))`, record `old := len(sb.Buf)`, reslice `dst := sb.Buf[:old+len(s)]`, and write `dst[old+i] = s[len(s)-1-i]` in a single pass with no intermediate buffer.
6. Add `(*Object).ExtendHeadExclusive(rb *ReversedBuilder, ascii bool)` mirroring `ExtendExclusive`:
   - Under `ValidateExec`, assert `o.flags&(ObjExclusive|ObjInterned|ObjReversed|ObjASCIIKnown) == ObjExclusive|ObjReversed|ObjASCIIKnown`, `o.length == 0`, and `o.stringBuilder() == unsafe.Pointer(rb)`.
   - Update `o.SetText(wtf8.String(rb.Sb.String()))`. If `ObjASCII` was set and `!ascii`, clear `ObjASCII`. Otherwise, if `o.Meta()` has a cached non-zero UTF-16 `Length`, increment it by `wtf8.Len` of the newly added slice tail `s[oldLen:]`.

### 5.2 Add `OpPrepend` and `OpPrependK` Opcodes (`internal/bytecode/`, `internal/compiler/`)

1. In `internal/bytecode/bytecode.go`, add two opcodes right next to `OpAppend` / `OpAppendK`:
   - `OpPrepend`: `a = b + a`, where `a` is an exclusive register (operand A: `OperandReg`), `b` is a register (operand B: `OperandReg`), and `c` is unused (0).
   - `OpPrependK`: `a = Consts[b] + a`, where `a` is a register (operand A: `OperandReg`), `b` is a constant index (operand B: `OperandConst`), and `c` is an `OperandFlag` (`0` = exclusive in-place prepend allowed; `1` = refused/shared, must behave like non-exclusive `Concat(Consts[b], a)`).
2. In `internal/bytecode/operands.go`, register both opcodes with accurate read/write operand kinds (note that both read *and* write register `A`, and `OpPrepend` also reads register `B`).
3. In `internal/compiler/appends.go`:
   - Add `emitPrepend(local int32, value ast.Expr, dst int32)`: if `value` is `*ast.StringLit`, emit `OpPrependK local, constStr(s.Value), 0`; otherwise evaluate `value` into a temporary operand `r`, emit `OpPrepend local, r, 0`, and free the temporary. If `dst >= 0 && dst != local`, emit `OpSeal local` followed by `OpMove dst, local`.
   - Update `settleAppends()`: when register `in.A` is refused (unsealed read detected), rewrite `OpPrepend` into `OpAdd A: in.A, B: in.B, C: in.A` (note argument order `B + A`!), and rewrite `OpPrependK` by setting `in.C = 1` (since there is no `OpPrependConcatK` opcode).
   - Update `unsealedReads()`: treat `OpPrepend` and `OpPrependK` (when `in.C == 0`) identically to `OpAppend` / `OpAppendK` as producing an exclusive string in `in.A`, and refuse if `OpPrepend` has `in.B == r` while `r` is already unsealed (`r = r + r`).
4. In `internal/compiler/expr.go`:
   - In assignment compilation (`local = left + right` where `right` is an identifier matching `local` and `left` is not `local`), verify the left expression does not mutate or read `local` before `right` is read, check `f.appends(name, local, bin.Left)` (ensuring `left` is provably string-valued and `local` has no hidden reads), and call `f.emitPrepend(local, bin.Left, dst)`. (Remember: if `left` can read `local`, `f.seal` runs before evaluating `left` or `settleAppends` automatically refuses it safely!)

### 5.3 Raw Accessors & Barrier-Free Length Update (`internal/storage/raw.go`, `internal/storage/object.go`)

1. In `internal/storage/object.go`, add a compile-time layout assertion next to the existing `textObject` assertions verifying `unsafe.Sizeof(wtf8.String("")) == 2 * unsafe.Sizeof(uintptr(0))`.
2. In `internal/storage/raw.go`, add three minimal, guaranteed-inlineable methods on `Raw`:
   - `TailBuf() *StringBuf`: casts `(*textObject)(unsafe.Pointer(r.o)).builder` directly to `*StringBuf` (do NOT call `r.o.stringBuilder()` or `r.o.Text()` inside this method — see Pitfall #1 below!). Caller verifies `sb.Len() == len(ao.Text())`.
   - `HeadBuf() *StringBuf`: loads `p := (*textObject)(unsafe.Pointer(r.o)).builder`; if non-nil, returns `&(*ReversedBuilder)(p).Sb`, else `nil`. Caller verifies `rb.Len() == len(ao.Text())`.
   - `SetBufLen(n int)`: updates only the second pointer-sized word (the `len` field) of `(*textObject)(unsafe.Pointer(r.o)).text` to `n`. This is strictly valid only when `old == len(ao.Text())` and `old + n <= sb.Room()` (meaning `sb.Buf` did not reallocate, so the string data pointer in word 0 is already identical to `unsafe.SliceData(sb.Buf)`). Because no pointer word is written, Go emits a single `MOVQ` with zero `gcWriteBarrier2` and zero `unsafe.String` runtime checks.

### 5.4 Engine `Prepend`, Short-ASCII `HeadBuilder`, and Eager Exclusive Buffering (`internal/engine/concat.go`)

1. In `ExecutionContext.Concat(a, b)`: inside the `b.O.Flags()&ObjReversed != 0` / `HeadBuilder` fast path, add a second branch for packed short ASCII strings (`shortstrings.IsASCII(math.Float64bits(a.N))` and `n := a.StringByteLength() <= shortstrings.MaxBytes`) that unpacks `shortstrings.Bytes(data)[:n]` and calls `WriteReversedBytes(&rb.Sb, bytes[:n])`, matching the existing `TailBuilder` fast path.
2. Add `ExecutionContext.Prepend(a, b)` and `prependInPlace(a, b, rb)` mirroring `Append` / `appendInPlace`:
   - If `a.O.Flags()&ObjExclusive != 0` and `rb := HeadBuilder(a.O)` is non-nil, attempt `prependInPlace` (handling `ObjASCIIKnown|ObjASCII` without `ObjReversed`, packed short ASCII via `WriteReversedBytes`, and general WTF-8 after checking `!storage.JoinsPair(y, false, x, true)`).
   - **Eager exclusive buffering** (in both `Append` and `Prepend`): when `a.O` is not yet exclusive/reversed and its directional `ConcatCount` has reached `storage.BufferAfter` (3), and both operands are non-empty ASCII strings whose combined length is `<= MaxStringLength`, immediately allocate a new `StringBuf` (or `ReversedBuilder`) with `Grow(max(64, n))` — bypassing the 32-byte `MinBufferedLength` threshold and avoiding the 32→64 growth step — populate it, mark the resulting string `ObjExclusive`, and return it.
   - Otherwise fall back to `cx.Concat(b, a)` and mark the result `ObjExclusive` if it is a newly created head of a builder.

### 5.5 Inline 64-Bit Append & Prepend in `execFast` (`internal/engine/interp.go`)

1. **In `execFast`, add `case bytecode.OpAppend, bytecode.OpAppendK:`**:
   - Load `ao := execRegAt(regs, regsBase, int(i.A)).O`. Require `ao != nil`, `ao.Kind == KindString`, and `ao.Flags()&(ObjExclusive|ObjASCIIKnown|ObjASCII|ObjReversed) == ObjExclusive|ObjASCIIKnown|ObjASCII`.
   - Load `sb := ao.Raw().TailBuf()`. Require `sb != nil`.
   - Load `b` from `fp.Consts[i.B]` (`OpAppendK`) or register `i.B` (`OpAppend`). Require `bo := b.O` is non-nil with `bo.Kind == KindString`.
   - Compute `n := len(bo.Text())` and `old := sb.Len()`. Require `old == len(ao.Text())`, `old + n <= sb.Room()`, and `old + n <= storage.MaxStringLength`.
   - **Short-string path (checked FIRST)**: if `n <= shortstrings.MaxBytes` (**CRITICAL: do NOT require `bo.Flags()&ObjInterned == 0` — see Pitfall #2!**):
     - Extract `data := math.Float64bits(b.N)`. Require `shortstrings.IsASCII(data)`.
     - Call `dst := sb.Extend(old + n)`.
     - If `old + 8 <= sb.Room()`: execute `writeU64LE(dst[old:old+8], data)` — a single 64-bit `MOVQ` with zero byte loop and zero per-byte bounds checks. (Writing unused padding bytes up to `old+8` within the exclusive buffer's capacity is invisible past `len(sb.Buf)`.)
     - Else (near the end of buffer capacity): copy `n` bytes via a simple shift loop.
     - Call `ao.Raw().SetBufLen(old + n)` and `continue`.
   - **Longer ASCII path**: else if `bo.Flags()&(ObjASCIIKnown|ObjASCII|ObjReversed) == ObjASCIIKnown|ObjASCII && n <= 16`:
     - Copy `y := bo.Text()` byte-by-byte into `dst[old+k]`, call `ao.Raw().SetBufLen(old + n)`, and `continue`.
   - On any failed condition (buffer full / reallocation needed, non-ASCII, non-exclusive, non-string), fall through to exit `execFast` into slow `exec`.

2. **In `execFast`, add `case bytecode.OpPrepend, bytecode.OpPrependK:`**:
   - If `i.Op == OpPrependK && i.C != 0`, break immediately to slow `exec`.
   - Same checks as `OpAppend`, except require `ObjReversed` to be **set** in `ao.Flags()` and use `rb := ao.Raw().HeadBuf()`.
   - **Short-string path**: if `n <= shortstrings.MaxBytes` and `shortstrings.IsASCII(data)`:
     - Call `dst := rb.Extend(old + n)`.
     - If `old + 8 <= rb.Room()`: compute `rev := bits.ReverseBytes64(data) >> (8 * (8 - uint(n)))` (compiles to a single `BSWAPQ` + `SHRQ` on `amd64`) and write `writeU64LE(dst[old:old+8], rev)`. Note that for `n == 0`, shifting a `uint64` by 64 yields 0 safely.
     - Else: write bytes in reverse order from `k = n - 1` down to `0`.
     - Call `ao.Raw().SetBufLen(old + n)` and `continue`.
   - **Longer ASCII path**: copy `y[n-1-k]` for `k = 0..n-1`, call `ao.Raw().SetBufLen(old + n)`, and `continue`.

3. **In slow `exec`**:
   - Add `case bytecode.OpPrepend:` calling `ag.active.Prepend(x, y)` when both are strings, else `addValues(y, x)` + `reload`.
   - Add `case bytecode.OpPrependK:` calling `Prepend(x, y)` when `i.C == 0` or `Concat(y, x)` when `i.C != 0` for string `x`, else `addValues(y, x)` + `reload`.

4. **Also fix `OpGetLength` in `execFast`**: when checking short ASCII strings for `.length`, check `len(o.Text()) <= shortstrings.MaxBytes && shortstrings.IsASCII(math.Float64bits(v.N))` without an `ObjInterned == 0` restriction.

### 5.6 Zero-Copy `Function.prototype.apply` & `ConcatASCIIStrings` (`internal/engine/runtime.go`, `internal/builtin/`, `internal/storage/concat.go`)

1. **`CallWithArray(f *storage.Object, this, arr storage.Value) (storage.Value, bool)`** in `internal/engine/runtime.go`:
   - Verify `arr.IsObject()`, call `elems, ok := arr.O.PlainElems()`. If `!ok`, return `(Undefined, false)` so `functionApply` falls back to `CreateListFromArrayLike`.
   - Otherwise invoke `cx.Call(f, this, elems...)` directly (passing the existing dense `[]Value` slice without allocating or copying a second slice) and return `(res, true)`.
   - In `internal/builtin/builtin.go` (`functionApply`): before calling `CreateListFromArrayLike`, try `if v, ok := cx.CallWithArray(f, thisArg, argArray); ok { return v }`.
2. **`RealmPart.ConcatASCIIStrings(first Value, rest []Value) (Value, bool)`** in `internal/storage/concat.go` (forwarded through `internal/quiet/array.go`):
   - First pass over `first` and every element of `rest`: confirm every value is a `KindString`, non-reversed, and ASCII (either `ObjASCIIKnown|ObjASCII` set, or `len(Text()) <= shortstrings.MaxBytes && shortstrings.IsASCII(math.Float64bits(v.N))`). Accumulate total byte length `total`, rejecting if `total > MaxStringLength` or any element fails (returning `false` to fall back to the general path).
   - If `total == 0`, return `r.Strings.Empty, true`; if `total <= shortstrings.MaxBytes`, pack into a short string.
   - Otherwise allocate **one** `buf := make([]byte, total)`, copy all segments into `buf` (unpacking short strings via `shortstrings.Bytes` when `ObjInterned == 0`, or copying `o.Text()` directly), and return `r.NewString(wtf8.String(unsafe.String(unsafe.SliceData(buf), total)))` with `ObjASCIIKnown|ObjASCII` set.
   - In `internal/builtin/string.go` (`stringConcat`): when `this` is already a string and `len(args) > 0`, try `cx.ConcatASCIIStrings(this, args)` before allocating a parts slice or calling `ToStringValue` in a loop.

---

## 6. Architectural Invariants & White-Box Test Updates

- **`internal/engine/fastpath_test.go`**:
  - Add `"bits.ReverseBytes64": true` to `fastPathCalls`.
  - Add `"TailBuf": true`, `"HeadBuf": true`, `"SetBufLen": true`, `"Len": true`, `"Room": true`, `"Extend": true` to `fastPathMethods`.
- **`internal/engine/rawaccess_test.go`**:
  - Add `"TailBuf": {"agent.execFast": 1}`, `"HeadBuf": {"agent.execFast": 1}`, and `"SetBufLen": {"agent.execFast": 4}` to `rawStoreCalls`.
- **`internal/engine/testdata/rawusers.txt`**:
  - Add `ExecutionContext.Prepend ReversedBuilder.Sb:ref*3` and `ExecutionContext.prependInPlace ReversedBuilder.Sb:ref*3` under `[stores]`.

---

## 7. Known Pitfalls & Failed Approaches to Avoid

1. **Pitfall #1 — Helper nesting breaks Go's inline budget**: Initially `Raw.TailBuf()` and `Raw.HeadBuf()` called `r.o.stringBuilder()` and `len(r.o.Text())`. That exceeded Go's inlining threshold inside `execFast` and failed `TestExecFastPathsCallNothing/compiled` with non-inlined calls to `storage.Raw.TailBuf` and `storage.Raw.HeadBuf`. Keep `TailBuf()` and `HeadBuf()` as trivial single-pointer casts on `(*textObject)(unsafe.Pointer(r.o)).builder` and perform `old == len(ao.Text())` in `execFast`.
2. **Pitfall #2 — Erroneous `bo.Flags()&storage.ObjInterned == 0` guard causes 100% fast-path misses**: String literal constants in `fp.Consts` (such as `"x"`, `"xy"`, `"abcdef"`) are interned (`ObjInterned` set) and do *not* have `ObjASCIIKnown` set. However, by Ekuma's value invariant (`internal/storage/value.go`), **every** string with `len(Text()) <= shortstrings.MaxBytes` — interned or not — stores its packed bytes in `Value.N`. Adding `bo.Flags()&ObjInterned == 0` caused all seven `string_build*` benchmarks to miss `execFast` on 100% of iterations (a hidden **+6.82 nats** loss until caught via `pprof`). Never gate reading `b.N` on `ObjInterned == 0`.
3. **Pitfall #3 — Byte-by-byte copy loops cause register reloads inside the loop**: Before switching short strings to `writeU64LE`, Go used `AX` as a temporary inside the byte-copy loop and emitted a stack reload of `ag` (`MOVQ 0x170(SP), AX`) on every byte copied, plus `unsafe.String` checks and `gcWriteBarrier2` in `SetBufText`. Replacing the loop with `writeU64LE` + `SetBufLen` eliminated all three costs.
4. **Pitfall #4 — `OpPrependK` refusal encoding**: Unlike `OpPrepend` (which rewrites to `OpAdd` on refusal), `OpPrependK` stays `OpPrependK` with `C = 1` when refused. Both `execFast` and slow `exec` must check `i.C != 0` on `OpPrependK`, and `unsealedReads` must treat `OpPrependK` as exclusive only when `C == 0`.
5. **Pitfall #5 — Do NOT use `copy(...)` inside `execFast`**: Go lowers variable-length `copy` to `runtime.memmove`, which fails `TestExecFastPathsCallNothing/compiled`. Use `writeU64LE` for `n <= 8` and explicit small loops for `n <= 16`.

---

## 8. `unsafe` Usage & Pure-Safe-Go Alternative

- **`unsafe` used**:
  1. `unsafe.String` in `StringBuf.String()` and `ConcatASCIIStrings` (identical to Go's standard `strings.Builder.String`).
  2. `(*textObject)(unsafe.Pointer(r.o)).builder` in `Raw.TailBuf` / `Raw.HeadBuf` (identical to baseline `Object.stringBuilder()`).
  3. `Raw.SetBufLen(n)` writing word 1 (`len`) of `textObject.text` to skip `gcWriteBarrier2`.
  4. `writeU64LE` in `internal/engine/endian_le.go`.
- **Safe-Go alternative**: Replacing `writeU64LE` with `encoding/binary.LittleEndian.PutUint64` produces identical `MOVQ` machine code on `amd64`/`arm64` (0 nats loss). Replacing `SetBufLen` with `SetText(wtf8.String(sb.String()))` costs ~0.5–1.0 nats on `string_build*` due to the write barrier check, while preserving >85% of the technique's gain.

---

## 9. Verification & Reproduction Protocol

1. Run targeted unit and white-box tests: `go test ./internal/bytecode ./internal/compiler ./internal/builtin ./internal/engine` and `go test -run 'TestAppendedStringsToGo|TestAllocationsPerOperation' .`.
2. Run runtime invariant validation: `go test -tags ekumavalidate ./internal/engine`.
3. Verify `execFast` compiled cleanly with zero calls: `TestExecFastPathsCallNothing/compiled` must pass.
4. **Mandatory fast-path hit verification**: Profile `string_build_large1` (`go test -run=^$ -bench='BenchmarkMicro/string_build_large1/ekuma$' -benchtime=1s -cpuprofile=...` inside `_bench/`) and run `go tool pprof -top`. Confirm `ExecutionContext.Append` and `appendInPlace` account for **< 5%** of CPU samples (almost all time must sit inside `(*agent).execFast` and occasional buffer-growth `memmove`).
5. **Expected benchmark signals (vs baseline)**: `string_build3` ≥ 6x faster, `string_build_large2` ≥ 5x faster, `string_build1..2` ≥ 2.3x faster, `string_build_large1` ≥ 2.1x faster, `string_concat` ≥ 1.9x faster.
