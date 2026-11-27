# Optimization #5: Inline `int64` BigInt Execution, Compact 80-Byte `BigIntObject`, and Direct Multi-Limb Arithmetic (≤ 512 Bits)

## 1. Summary & Measured Impact

- **Rank**: #5 of 8
- **Suite-Wide Impact**: **+2.22 nats** across all 74 microbenchmarks (**1.030x** step geometric-mean lift in Stage 5)
- **Targeted Subset Impact**: **+3.08 nats** across the 3 BigInt arithmetic benchmarks
- **Key Benchmark Improvements (Stage 4 → Stage 5)**:

| Benchmark | Stage 4 Ekuma | Stage 5 Ekuma | Stage Speedup | Overall vs Baseline |
| --- | ---: | ---: | ---: | ---: |
| `bigint32_arith` (32-bit BigInt `sum += a * a; a += 1n`) | 46.3 µs | 10.5 µs | **4.41x** (+1.48 nats) | **3.47x** |
| `bigint64_arith` (64-bit `a*a` + >64-bit `sum`) | 135.1 µs | 49.8 µs | **2.71x** (+1.00 nats) | **2.32x** |
| `bigint256_arith` (124-bit `a`, 248-bit `a*a`, 258-bit `sum`) | 480.0 µs | 262.7 µs | **1.83x** (+0.60 nats) | **1.79x** |

---

## 2. Root Cause Analysis

Four independent inefficiencies slowed down BigInt arithmetic in baseline Ekuma:

1. **`BigInt64Object` exited `execFast` on every operation**: Ekuma already unboxed `int64`-range BigInts into `Value{N: Float64frombits(uint64(i)), O: BigInt64Object}`, but `execFast` only checked `NumberObject` in `OpAdd`, `OpSub`, and `OpMul`. Thus `bigint32_arith` (where 100% of values fit in `int64`) and the `a * a` / `a += 1n` steps of `bigint64_arith` bounced out to slow `exec` on every instruction.
2. **Heap-allocating `big.NewInt(i)` for mixed 64-bit / multi-word operations**: `addBigInts` unconditionally called `a.BigInt(), b.BigInt()`. Whenever one operand was a `BigInt64Object` and the other was a multi-word `BigIntObject` (every single `sum += a * a` in `bigint64_arith` and `a += 1n` in `bigint256_arith`), `Value.BigInt()` allocated a throwaway `*big.Int` on the heap via `big.NewInt(i)`.
3. **Oversized 152-byte `BigIntObject` with `Ext` boxing**: `BigIntObject` embedded `PropertyObject` (120 bytes: 48-byte `Object` + 72-byte `objectTail`) plus `big.Int` (32 bytes) = **152 bytes** before adding magnitude limbs, and stored `&o.Integer` inside `objectTail.Ext` as an `any` interface value. Reading a BigInt chased `o.objectTail -> Ext -> type assertion (*big.Int)`.
4. **256-bit inline cutoff (`4` words) missed 258-bit sums and forced `math/big` overhead**: `bigIntSumWords` and `bigIntProductWords` were capped at `256 / bits.UintSize` (4 words on 64-bit). In `bigint256_arith`, `sum` reaches 258 bits (**5 words**), missing the inline buffer on every single iteration and allocating three separate heap objects (`new(big.Int)`, a `[]big.Word` slice, and a 152-byte `BigIntObject`). Furthermore, slow `exec` bounced back into `execFast` after each individual BigInt instruction (`OpMul`, `OpAdd`, `OpAdd`) — 3 round-trips per loop iteration.

---

## 3. Prerequisites & Dependency Order

- **Depends on Optimization #2** (`execFast` leaf interpreter loop) for the inline `BigInt64Object` paths in `OpAdd`, `OpSub`, and `OpMul`.
- The compact 80-byte `BigIntObject` and direct multi-limb arithmetic in `internal/storage/bigint.go` and `internal/engine/bigint.go` are fully independent of `execFast`.

---

## 4. Target Files Inventory

| File | Role |
| --- | --- |
| `internal/storage/bigint.go` | Embed `plainObject` instead of `PropertyObject` in `BigIntObject` (152 B → 80 B); change `InitBigInt()` to take no arguments; add `(*Object).bigInt() *big.Int`; export `Sub64` and `Mul64`. |
| `internal/storage/value.go` | Add barrier-avoiding `SetBigInt64(dst *Value, i int64)`; update `Value.BigInt()` to call `v.O.bigInt()`. |
| `internal/storage/compare.go` | Update `IsStrictlyEqual` for `KindBigInt` to call `a.O.bigInt().Cmp(b.O.bigInt())`; drop unused `math/big` import. |
| `internal/storage/poll.go` | Update `comparedBytes` for `KindBigInt` to call `v.O.bigInt().BitLen() / 8`; drop unused `math/big` import. |
| `internal/engine/bigint.go` | Raise inline threshold to `bigIntInlineWords = 8` (512 bits); add allocation-free `unpackBigInt`, size-class-matched `makeBigInt`, pure `math/bits` limb helpers (`addWords`, `subWords`, `cmpWords`, `mulWords`), `mulBigInts`, `addSubBigInts`, and `subBigInts`. |
| `internal/engine/runtime.go` | Route `OpSub` and `OpMul` in `ExecutionContext.arith` directly to `subBigInts` and `mulBigInts` before allocating `new(big.Int)` or calling `x.BigInt()`/`y.BigInt()`. |
| `internal/engine/interp.go` | Add `BigInt64Object` fast paths (including `OpMul` + `OpAdd` fusion) in `execFast`; add direct `KindBigInt` dispatch (without `addValues`/`arith`/`reload`) and chained `OpMul` + `OpAdd` + `OpAdd` execution in slow `exec`. |
| `internal/engine/fastpath_test.go` | Allowlist `storage.SetBigInt64`, `storage.Add64`, `storage.Sub64`, and `bits.Mul64` in `fastPathCalls`. |

---

## 5. Detailed Design Specification (No Code)

### 5.1 Compact 80-Byte `BigIntObject` & `SetBigInt64` (`internal/storage/`)

1. In `internal/storage/bigint.go`:
   - Change `BigIntObject` from embedding `PropertyObject` (120 B) to embedding `plainObject` (48 B, whose `Init()` attaches `&zeroTail` and is already allowlisted in `TestValueStorageOwnership`). Total `BigIntObject` header size drops from **152 B to 80 B**.
   - Update `(*BigIntObject).InitBigInt() Value` to take zero parameters, set `o.Kind = KindBigInt`, and return `Value{O: o.Init()}`.
   - Add unexported `(*Object).bigInt() *big.Int` returning `&(*BigIntObject)(unsafe.Pointer(o)).Integer` (a single constant pointer offset, zero `objectTail` dereference, zero interface box, zero type assertion).
   - Export `Sub64(x, y int64) (int64, bool)` and `Mul64(x, y int64) (int64, bool)` (previously unexported `sub64` and `mul64`).
2. Audit all `ExtState().(*big.Int)` callers across the entire codebase (there are exactly three, all inside `internal/storage`):
   - `internal/storage/value.go` (`Value.BigInt()`): replace `v.O.ExtState().(*big.Int)` with `v.O.bigInt()`.
   - `internal/storage/compare.go` (`IsStrictlyEqual`): replace both `ExtState().(*big.Int)` calls with `.bigInt()`.
   - `internal/storage/poll.go` (`comparedBytes`): replace `ExtState().(*big.Int)` with `.bigInt()`.
   - Remove the now-unused `"math/big"` imports from `compare.go` and `poll.go`.
3. In `internal/storage/value.go`, add `SetBigInt64(dst *Value, i int64)`: if `dst.O != BigInt64Object`, set `dst.O = BigInt64Object`; always set `dst.N = math.Float64frombits(uint64(i))`. Mirrors `SetNumber`, skipping the GC write barrier on `dst.O` whenever the register already holds a 64-bit BigInt.

### 5.2 Direct Multi-Limb Arithmetic ≤ 512 Bits (`internal/engine/bigint.go`)

1. Define `const bigIntInlineWords = 8` (8 × 64-bit words = 512 bits, covering 64-, 128-, 256-, and 512-bit operands and products).
2. Guard the direct 1-word-per-`int64` limb path with the compile-time constant `if bits.UintSize == 64` so 32-bit architectures automatically stay on the portable `math/big` fallback with zero dead-code overhead on 64-bit.
3. **`unpackBigInt(v storage.Value, buf *[1]big.Word) (words []big.Word, neg bool)`**:
   - If `v.O == storage.BigInt64Object`: extract `i := int64(math.Float64bits(v.N))`. If `i == 0`, return `(nil, false)`. If `i < 0`, store `big.Word(uint64(-i))` in `buf[0]` (note: in Go two's complement, `uint64(-math.MinInt64)` is cleanly `1 << 63`!) and return `(buf[:1], true)`. Otherwise store `big.Word(uint64(i))` and return `(buf[:1], false)`. **Zero heap allocations.**
   - Otherwise call `bi := v.BigInt()` and return `(bi.Bits(), bi.Sign() < 0)`.
4. **Pure `math/bits` limb arithmetic helpers** (using `bits.Add`, `bits.Sub`, `bits.Mul` on `uint`, which compile to hardware `ADCQ`/`SBBQ`/`MULQ` instructions):
   - `addWords(dst, x, y []big.Word) int` (precondition `len(x) >= len(y)`): propagate carry with `bits.Add`, append final carry word if non-zero, return normalized word length.
   - `cmpWords(x, y []big.Word) int`: compare lengths first, then scan top-down from most-significant word to index 0.
   - `subWords(dst, x, y []big.Word) int` (precondition `x > y`): propagate borrow with `bits.Sub`, strip trailing zero words, return normalized length.
   - `mulWords(dst, x, y []big.Word) int` (precondition `dst` zeroed with `len >= len(x)+len(y)`): schoolbook loop over `i = 0..nx-1`, skipping `x[i] == 0`, inner loop over `j = 0..ny-1` computing `(hi, lo) = bits.Mul(xi, y[j])`, adding `dst[i+j]` (carry `c1`) and `carry` (carry `c2`), storing `dst[i+j] = lo`, and setting `carry = hi + c1 + c2` (mathematically bounded by $2^{64}-1$, guaranteed never to overflow `uint`!). Store final `dst[i+ny] = carry` and strip trailing zero words.
5. **Size-class-matched result packer `(*ExecutionContext).makeBigInt(words []big.Word, neg bool) storage.Value`**:
   - If `len(words) == 0`: return `BigIntFromInt64(0)`.
   - If `len(words) == 1`: check exact `int64` fit — if `!neg && w <= math.MaxInt64`, return `BigIntFromInt64(int64(w))`; if `neg && w <= 1<<63`, return `BigIntFromInt64(int64(-w))`.
   - **Result bit-length limit check**: if `int64(n)*64 > cx.MaxBigIntResultBits`, compute exact `bitLen = (n-1)*64 + bits.Len64(uint64(words[n-1]))` and call `cx.checkBigIntBits(int64(bitLen))`. (In normal execution where `MaxBigIntResultBits == 1<<20`, the outer branch is never taken, keeping the check to 1 predicted comparison while still honoring tight test limits like `TestBigIntBitwiseSizeLimit` with 128 bits.)
   - Allocate the smallest composite struct embedding `bigIntObject` (80 B) plus inline `[N]big.Word` matching Go's `mallocgc` size classes:
     - `n <= 2`: `mag [2]big.Word` → **96 B** exact size class (copy directly without `copy()` call).
     - `n <= 4`: `mag [4]big.Word` → **112 B** exact size class.
     - `n <= bigIntInlineWords + 1` (up to 9 words): `mag [9]big.Word` → **152 B → 160 B** size class.
   - Attach `o.Integer.SetBits(o.mag[:n])` (zero-copy aliasing into the freshly allocated immutable struct), call `o.Integer.Neg(&o.Integer)` if `neg`, and return `o.InitBigInt()`.
6. **Unified `addSubBigInts(a, b storage.Value, sub bool)` & `mulBigInts(a, b storage.Value)`**:
   - Handle zero-operand shortcuts immediately (`a + 0 == a`, `a - 0 == a`, `0 + b == b`, `0 - b == -b`, `x * 0 == 0`).
   - For `addSubBigInts` with `len(aw) <= bigIntInlineWords && len(bw) <= bigIntInlineWords`: compute effective sign `bNegEff := bNeg != sub`; if `aNeg == bNegEff` call `addWords`, else branch on `cmpWords` and call `subWords` (returning `0` on equality).
   - For `mulBigInts` with `len(aw)+len(bw) <= bigIntInlineWords`: compute into stack `[bigIntInlineWords]big.Word` via `mulWords` and pack via `makeBigInt` with sign `aNeg != bNeg`.
   - On operands exceeding `bigIntInlineWords`: poll `if cx.InterruptFlag.Load() { cx.Poll() }` and fall back to `math/big` / `numconv.MulBigInt` with `cx.checkBigIntBits`.
7. In `internal/engine/runtime.go` (`ExecutionContext.arith`), dispatch `OpSub` to `cx.subBigInts(x, y)` and `OpMul` to `cx.mulBigInts(x, y)` *before* calling `x.BigInt()`, `y.BigInt()`, or `new(big.Int)`.

### 5.3 Fast `BigInt64Object` in `execFast` and Chained BigInt Ops in Slow `exec` (`internal/engine/interp.go`)

1. **In `execFast`**:
   - `OpAdd`: add an `else if x.O == storage.BigInt64Object && y.O == storage.BigInt64Object` branch calling `storage.Add64`; on no overflow, call `storage.SetBigInt64(dst, s)` and `continue`.
   - `OpSub`: add an `else if x.O == storage.BigInt64Object && y.O == storage.BigInt64Object` branch calling `storage.Sub64` + `storage.SetBigInt64` and `continue`.
   - `OpMul`: add an `else if x.O == storage.BigInt64Object && y.O == storage.BigInt64Object` branch with the 3-line signed `bits.Mul64` overflow check written inline (see Pitfall #1 below!). On no overflow, write `p` with `storage.SetBigInt64`, then **peek-fuse** a following `OpAdd` (`n.Op == OpAdd && n.C == i.A && n.A == n.B`) when `sum.O == BigInt64Object` and `storage.Add64(sum, p)` does not overflow — writing `s` via `storage.SetBigInt64` and advancing `ip`.
2. **In slow `exec`**:
   - `OpAdd`: right next to the two-string `Concat` fast dispatch, check `x.O.Kind == KindBigInt && y.O.Kind == KindBigInt`, call `ag.active.addBigInts(x, y)` directly, and `continue` (skipping `addValues` and `ag.reload`, since BigInt arithmetic cannot execute user JS or move the stack!).
   - `OpSub`: add a dedicated `case bytecode.OpSub:` checking two `KindBigInt` operands and calling `ag.active.subBigInts(x, y)` directly without `arith` or `reload`.
   - `OpMul`: add a dedicated `case bytecode.OpMul:` checking two `KindBigInt` operands, calling `p := ag.active.mulBigInts(x, y)`, storing `regs[i.A] = p`, and then **chaining** immediately following BigInt `OpAdd` instructions (`next := fp.Code[pc]` with `next.Op == OpAdd && next.C == i.A && next.A == next.B` when `regs[next.A]` is BigInt, and if that succeeded, also a second following `OpAdd` of two BigInts!). Remember to update `ag.frames[len(ag.frames)-1].pc = pc` before each chained call so stack traces on `RangeError` remain exact. This cuts `execFast` exit/re-entry round-trips in `bigint256_arith` from 3 per iteration down to 1.

---

## 6. Architectural Invariants & White-Box Test Updates

- **`internal/engine/fastpath_test.go`**: Add `"storage.SetBigInt64": true`, `"storage.Add64": true`, `"storage.Sub64": true`, and `"bits.Mul64": true` to `fastPathCalls`.
- **`TestValueStorageOwnership` (`internal/engine/storage_test.go`)**: Requires **zero** changes because `plainObject.Init()` is already allowlisted.
- **`rawusers.txt`**: Requires **zero** changes because `SetBigInt64` lives in `internal/storage/value.go` (not `internal/engine`), and `Value.N` reads are not restricted by `rawusers.txt`.

---

## 7. Known Pitfalls & Failed Approaches to Avoid

1. **Pitfall #1 — Calling `storage.Mul64` inside `execFast` fails `TestExecFastPathsCallNothing/compiled`**: While Go cleanly inlined `storage.Add64`, `storage.Sub64`, and `storage.SetBigInt64` into `execFast`, Go declined to inline `storage.Mul64` (mid-stack / budget exceeded with `bits.Mul64` + signed adjustment), causing an ELF `CALL` failure. Write the 3-line signed `bits.Mul64` overflow check directly inside `execFast`'s `OpMul` case; `bits.Mul64` is a compiler intrinsic (`MULQ`) with zero function call.
2. **Pitfall #2 — `TestBigIntBitwiseSizeLimit` with small custom `MaxBigIntResultBits`**: Never skip `cx.checkBigIntBits` for inline results solely because `n <= 8`; tests configure `MaxBigIntResultBits = 128` (2 words). Always check `if int64(n)*64 > cx.MaxBigIntResultBits` before allocating.
3. **Pitfall #3 — `math.MinInt64` edge cases**: Verify `-math.MinInt64` (`+2^63`) does **not** return `BigIntFromInt64` (`!neg && w <= math.MaxInt64` is false for `1 << 63`), whereas `-2^63` (`neg && w <= 1 << 63`) **does** return `BigIntFromInt64(math.MinInt64)` — maintaining Ekuma's canonical single-representation invariant (`IsStrictlyEqual` relies on the fact that an `int64`-range BigInt is *never* a heap `BigIntObject`!).

---

## 8. `unsafe` Usage & Pure-Safe-Go Alternative

- **`unsafe` used**: Only a single line — `&(*BigIntObject)(unsafe.Pointer(o)).Integer` in `internal/storage/bigint.go` (matching Ekuma's existing `textObject` embedded-header pattern). All limb arithmetic, `execFast` `BigInt64Object` execution, and slow-path chaining are **100% safe Go**.
- **Safe-Go alternative**: Keeping `PropertyObject` and `ExtState().(*big.Int)` removes that 1 `unsafe` line at the cost of +72 bytes per heap BigInt and an extra indirection (~0.2–0.3 nats on `bigint64_arith`/`bigint256_arith`, zero impact on `bigint32_arith`).

---

## 9. Verification & Reproduction Protocol

1. Run `go test ./internal/engine ./internal/builtin` and `go test -run 'BigInt|Allocations' .`.
2. Run `go test -tags ekumavalidate ./internal/engine` and `go test -race ./internal/engine`.
3. Verify `TestBigIntBitwiseSizeLimit` and `TestExecFastPathsCallNothing/compiled` pass.
4. Run `_bench` microbenchmarks (`-suite Micro -engine '^ekuma$' -rounds 1`) and confirm: `bigint32_arith` drops below 15 µs (≥ 3x vs baseline), `bigint64_arith` drops below 60 µs (≥ 2x vs baseline), and `bigint256_arith` drops below 300 µs (≥ 1.6x vs baseline).
