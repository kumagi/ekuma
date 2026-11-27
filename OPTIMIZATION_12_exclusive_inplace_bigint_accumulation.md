# Optimization #12: Exclusive In-Place BigInt Accumulation (`ObjExclusive`), Stack-Product Fused `OpMulAdd`/`OpMulAccum`, and Compiler Non-Aliasing Analysis

## 1. Summary & Measured Impact

- **Overall Rank**: #8 of 13
- **Suite-Wide Impact**: **+2.55 nats** across all 74 microbenchmarks (**1.035x** step geometric-mean lift in Stage 10)
- **Targeted Subset Impact**: **+2.77 nats** across BigInt arithmetic, numeric multiply-accumulate, and `array_push` recovery
- **Key Benchmark Improvements (Stage 9 → Stage 10 → Final 3-Round Median)**:

| Benchmark | Phase 1 Ekuma | Stage 10 Ekuma | Final Ekuma | Phase 2 Speedup | Overall vs Baseline |
| --- | ---: | ---: | ---: | ---: | ---: |
| `bigint256_arith` (258-bit `sum += a * a; a += 1n`) | 266.9 µs | 73.5 µs | 70.4 µs | **3.79x** (+1.33 nats) | **6.79x** |
| `bigint64_arith` (128-bit `sum += a * a; a += 1n`) | 51.5 µs | 25.9 µs | 24.9 µs | **2.07x** (+0.73 nats) | **4.81x** |
| `float_arith` (fused `s += x * y` in `execFast`) | 8.12 µs | 6.82 µs | 6.82 µs | **1.19x** (+0.17 nats) | **2.23x** |
| `int_arith` (fused `s += x * y` in `execFast`) | 5.97 µs | 5.29 µs | 5.10 µs | **1.17x** (+0.16 nats) | **2.41x** |
| `bigint32_arith` (fused `int64` `OpMulAccum`) | 10.6 µs | 9.93 µs | 9.97 µs | **1.06x** (+0.06 nats) | **3.68x** |

---

## 2. Root Cause Analysis

Even after Optimization #5 (80-byte `BigIntObject` and ≤ 512-bit direct `math/bits` limb arithmetic), `bigint256_arith` (267 µs) and `bigint64_arith` (51.5 µs) still allocated **three heap objects per loop iteration**:

1. **Throwaway intermediate product allocation (`a * a`)**: Computing `sum += a * a` first heap-allocated a 112–160-byte `BigIntObject` for the product `a * a`, immediately added it into `sum`, and threw the product away — generating massive GC allocation and scanning pressure.
2. **Throwaway accumulator allocations (`sum += ...` and `a += 1n`)**: Because JavaScript BigInts are immutable values, every `sum += prod` and every `a += 1n` allocated a brand-new 112–160-byte `BigIntObject` even when the compiler could prove that the local variables `sum` and `a` had **no other references** inside the loop!
3. **Separate `OpMul` + `OpAdd` dispatch for numeric `s += x * y`**: Even for `NumberObject` and `BigInt64Object`, compiling `local += b * c` as two instructions (`OpMul` into a temp register + `OpAdd` into `local`) consumed an extra instruction fetch and dispatch cycle on `int_arith`, `float_arith`, and `bigint32_arith`.

---

## 3. Prerequisites & Dependency Order

- **Depends on Optimization #5** (`BigIntObject`, `unpackBigInt`, `addWords`/`subWords`/`cmpWords`/`mulWords`) and **Optimization #9** (`execQuiet` loop so heap BigInt loops execute without bouncing back to `execFast`).
- Uses the exact same `ObjExclusive` flag and `OpSeal` sealing mechanism already introduced for strings in **Optimization #1**.

---

## 4. Target Files Inventory

| File | Role |
| --- | --- |
| `internal/bytecode/bytecode.go` | Add three opcodes: `OpAccum` (`a += b`, `a` is exclusive accumulator), `OpMulAdd` (`a += b * c`, non-exclusive), and `OpMulAccum` (`a += b * c`, `a` is exclusive accumulator). |
| `internal/bytecode/operands.go` | Register operand layouts for `OpAccum`, `OpMulAdd`, and `OpMulAccum` (`FieldA` written as result/accumulator; `FieldB`, `FieldC` read). |
| `internal/compiler/compiler.go` | Add `accumRegs []int32` on `fn` and call `f.settleAccums()` right after `f.settleAppends()`. |
| `internal/compiler/expr.go` | In compound `+=` on a local variable (`!stringValued`): emit `OpMulAccum` (when `canAccum(local)`) or `OpMulAdd` for `local += b * c`, and `OpAccum` for `local += rhs` when `canAccum(local)`. |
| `internal/compiler/appends.go` | Implement `canAccum`, `isNonAliasingBigIntRead`, `escapingAccumReads`, and `settleAccums` (proving via worklist CFG dataflow whether each candidate accumulator register can safely hold an `ObjExclusive` BigInt, rewriting refused registers back to `OpAdd`/`OpMulAdd` and inserting `OpSeal` only at true escaping reads — zero `OpSeal` inside numeric loops!). |
| `internal/storage/bigint.go` | Define `ExclusiveBigIntWords = 9`, `exclusiveBigInt` (embedding `BigIntObject` + `[9]big.Word`), `NewExclusiveBigInt(words, neg)`, and `(*Object).UpdateExclusiveBigInt(words, neg)` (mutating `o.mag` and `o.Integer` in place with zero heap allocations). |
| `internal/engine/bigint.go` | Seal direct-return operands in `addSubBigInts` (`a.O.Share()` when `b == 0n`, `b.O.Share()` when `a == 0n`); add `makeBigIntInto(dst, words, neg, exclusive)`, `accumBigInts(a, b)`, and `mulAddBigInts(s, x, y, exclusive)` computing the product on the Go stack (`[8]big.Word`) and updating an `ObjExclusive` accumulator in place! |
| `internal/engine/interp.go` | Update `OpSeal` in `execFast` and slow `exec` to clear `ObjExclusive` on both `KindString` and `KindBigInt`; add `OpAccum`, `OpMulAdd`, and `OpMulAccum` fast paths in `execFast` (`NumberObject` and `BigInt64Object`) and slow `exec` (routing heap BigInts to `accumBigInts`/`mulAddBigInts` and entering `execQuiet`). |
| `internal/engine/quiet_exec.go` | Handle `OpAccum`, `OpMulAdd`, and `OpMulAccum` directly inside `execQuiet` for numbers, `BigInt64Object`, and heap BigInts; decline `Array.prototype.push`/`pop` and `Extremum`. |
| `internal/engine/legacy.go` | Ensure `seal` / escape helpers also clear `ObjExclusive` on `KindBigInt` objects. |

---

## 5. Detailed Design Specification (No Code)

### 5.1 Exclusive Mutable BigInt Backing Store (`internal/storage/bigint.go`)

1. Reuse the existing `ObjExclusive` object flag (`o.flags & ObjExclusive != 0`) for `KindBigInt` objects. An `ObjExclusive` BigInt is guaranteed to have **at most one** live reference — the single local register being accumulated — and is always backed by `exclusiveBigInt` (`BigIntObject` + fixed `mag [ExclusiveBigIntWords]big.Word` where `ExclusiveBigIntWords = 9`, supporting up to 576 bits in a single 152 B → 160 B allocation).
2. **`NewExclusiveBigInt(words []big.Word, neg bool) Value`**: Allocates one `exclusiveBigInt`, copies `words` into `o.mag[:n]`, calls `o.Integer.SetBits(o.mag[:n])`, applies `Neg` if `neg`, sets `o.flags = ObjExclusive`, and returns `o.InitBigInt()`.
3. **`(*Object).UpdateExclusiveBigInt(words []big.Word, neg bool)`**: Precondition: `o.flags&ObjExclusive != 0` and `len(words) <= ExclusiveBigIntWords`. Downcasts `o` to `*exclusiveBigInt`, copies `words` into `o.mag[:n]`, updates `o.Integer.SetBits(o.mag[:n])` and sign (`Neg` if `neg`), and leaves `o` in place — **zero heap allocations, zero GC write barriers**!
4. **Sealing (`OpSeal` & escape points)**: Calling `o.Share()` clears `ObjExclusive`. Once cleared, `o` is permanently immutable and behaves identically to any ordinary `BigIntObject`. Update `OpSeal` in `execFast`, slow `exec`, `execQuiet`, and `legacy.go` to check `o.Kind == KindString || o.Kind == KindBigInt`.

### 5.2 Stack-Product Multiply-Accumulate & In-Place Update (`internal/engine/bigint.go`)

1. **Maintain the canonical `BigInt64Object` invariant**: If the computed result has `len(words) == 0` or fits in signed `int64` (`len(words) == 1` and `!neg && w <= MaxInt64` or `neg && w <= 1<<63`), **always** return `BigIntFromInt64(...)` — NEVER store an `int64`-range value inside an `exclusiveBigInt`!
2. **Maintain `MaxBigIntResultBits` before mutation**: Always validate `cx.checkBigIntBits(bitLen)` *before* calling `UpdateExclusiveBigInt` so a `RangeError` leaves the accumulator untouched!
3. **`makeBigIntInto(dst storage.Value, words []big.Word, neg bool, exclusive bool) storage.Value`**:
   - Handle `len(words) == 0` and `int64`-fit first (returning `BigIntFromInt64`).
   - Check `MaxBigIntResultBits` when `int64(len(words))*64 > cx.MaxBigIntResultBits`.
   - If `exclusive && len(words) <= storage.ExclusiveBigIntWords`:
     - If `dst.O != nil && dst.O.Kind == storage.KindBigInt && dst.O.Flags()&storage.ObjExclusive != 0`: call `dst.O.UpdateExclusiveBigInt(words, neg)` and return `dst` (**zero allocations!**).
     - Otherwise return `storage.NewExclusiveBigInt(words, neg)` (1 allocation on the very first transition into `> 64` bits, reused for every subsequent iteration!).
   - Otherwise fall back to `cx.makeBigInt(words, neg)`.
4. **`accumBigInts(a, b storage.Value) storage.Value`**:
   - Fast zero shortcuts: if `b == 0n`, return `a` directly; if `a == 0n`, seal `b` (`if b.O.Flags()&ObjExclusive != 0 { b.O.Share() }`) before returning `b` so two registers can never alias the same exclusive BigInt!
   - Unpack `a` and `b` via `unpackBigInt` into stack `[1]big.Word` buffers. If both fit in `bigIntInlineWords` (`8` words): compute sum/difference into stack `[9]big.Word` via `addWords`/`subWords` and commit via `makeBigIntInto(a, sum, neg, true)`.
   - On overflow beyond `bigIntInlineWords`: seal `a` if exclusive and fall back to `math/big`.
5. **`mulAddBigInts(s, x, y storage.Value, exclusive bool) storage.Value`**:
   - Unpack `x` and `y` via `unpackBigInt`; if `x == 0n || y == 0n`, return `s` (sealing `s` if `!exclusive`).
   - If `len(xw)+len(yw) <= bigIntInlineWords` (product ≤ 512 bits) and `len(sw) <= bigIntInlineWords`:
     - Compute `p = x * y` directly into a **stack** `[bigIntInlineWords]big.Word` buffer via `mulWords` (**zero product allocation!**).
     - Compute `s + p` directly into a **stack** `[bigIntInlineWords+1]big.Word` buffer via `addWords`/`subWords`.
     - Commit via `makeBigIntInto(s, res, resNeg, exclusive)` — **ZERO heap allocations for the entire `sum += a * a` operation!**
   - Fallback for huge operands: seal `s` if exclusive, multiply with `mulBigInts`, add with `addBigInts`.
6. **Zero-operand sealing in `addSubBigInts`**: Whenever `addSubBigInts` returns `a` or `b` directly (e.g., `a + 0n` or `0n + b`), call `.Share()` if `ObjExclusive` is set so an exclusive BigInt can never leak into another register.

### 5.3 Compiler Non-Aliasing Analysis (`internal/compiler/`)

1. **Emission (`internal/compiler/expr.go`)**: When compiling compound `local += rhs` for a non-captured, non-parameter, non-string-valued local variable (`f.canAccum(local)`):
   - If `rhs` is a binary multiplication `left * right` (whose evaluation cannot overwrite `local` — verify `local` is safe or save to temp if needed; remember in JS `local += b * c` evaluates `local` *before* `b * c`, so only emit `OpMulAccum` / `OpMulAdd` when `b` and `c` are side-effect-free expressions that cannot mutate `local`, such as identifiers, numeric/BigInt literals, or pure subexpressions!): compile `b` and `c` into registers and emit `OpMulAccum(local, rb, rc)` (or `OpMulAdd` if `!canAccum(local)`), recording `local` in `f.accumRegs`.
   - Otherwise compile `rhs` into `rb` and emit `OpAccum(local, rb)`, recording `local` in `f.accumRegs`.
2. **CFG Worklist Verification (`settleAccums` in `internal/compiler/appends.go`)**:
   - Mirror `settleAppends`, tracking which PCs can hold an `ObjExclusive` BigInt in candidate register `r` (flowing forward from `OpAccum`/`OpMulAccum` and killed by any overwrite of `r`).
   - Classify every read of `r` at a live exclusive PC:
     - **Non-aliasing BigInt reads** (allowed with **no** `OpSeal`): arithmetic/bitwise/comparison operands (`OpAdd`, `OpSub`, `OpMul`, `OpDiv`, `OpMod`, `OpAccum`, `OpMulAdd`, `OpMulAccum`, `OpLt`, `OpLe`, `OpGt`, `OpGe`, `OpLoop*`, `OpEq`, `OpSEq`, etc.) that only inspect value bits and never copy the `*Object` pointer to another register or heap slot!
     - **Escaping reads** (`OpReturn`, `OpMove`, `OpSetGlobal`, `OpSetProp`, `OpSetIndex`, `OpCall` arguments, etc.): insert `OpSeal(r)` immediately before the escaping read (only if `r` is not inside a tight non-escaping loop where sealing would execute on every iteration; if sealing would be required on a back-edge or unsupported control flow, refuse `r`!).
   - For any refused register `r`: rewrite every `OpAccum(r, b)` back to `OpAdd(r, r, b)` and every `OpMulAccum(r, b, c)` back to `OpMulAdd(r, b, c)`.
   - **Zero overhead for non-BigInt numbers**: In numeric loops (`int_arith`, `float_arith`), `OpMulAccum` and `OpAccum` run directly in `execFast` with zero `OpSeal` instructions inside the loop!

### 5.4 Interpreter Dispatch (`interp.go` & `quiet_exec.go`)

1. **`execFast`**:
   - `OpAccum`: handle `NumberObject + NumberObject` (`SetNumber`) and `BigInt64Object + BigInt64Object` (`Add64` + `SetBigInt64`).
   - `OpMulAdd` / `OpMulAccum`: handle 3× `NumberObject` (`s.N + x.N*y.N` — note: avoid Go FMA contraction if needed by converting `float64(x.N * y.N)` so ECMAScript IEEE-754 double-rounding stays 100% spec-exact!) and 3× `BigInt64Object` (inlined 3-line `bits.Mul64` overflow check + `Add64` + `SetBigInt64`).
2. **Slow `exec` & `execQuiet`**:
   - For `KindBigInt` operands: dispatch `OpAccum` to `cx.accumBigInts`, `OpMulAdd` to `cx.mulAddBigInts(..., false)`, and `OpMulAccum` to `cx.mulAddBigInts(..., true)`.
   - Enter `execQuiet` from slow `exec` after heap BigInt `OpAccum`/`OpMulAdd`/`OpMulAccum` so the entire loop (`bigint256_arith` and `bigint64_arith`) executes 100% inside `execQuiet` with **zero bounces and zero heap allocations per iteration**!

---

## 6. Architectural Invariants & White-Box Test Updates

- **`TestResultConvention` (`internal/bytecode/operands_test.go`)**: Passes automatically because `OpAccum`, `OpMulAdd`, and `OpMulAccum` write only `FieldA`.
- **`TestExecFastPathsCallNothing`**: Inline `bits.Mul64` in `OpMulAdd`/`OpMulAccum` just like `OpMul` so zero Go `CALL` instructions appear in `execFast`.
- **IEEE-754 Double Rounding**: Ensure `s.N + float64(x.N * y.N)` does not get contracted into a hardware FMA instruction (`VFMADD`), preserving exact ECMAScript `a + (b * c)` rounding semantics.
- **Exception & Aliasing Safety**: Verify `TestBigIntBitwiseSizeLimit` and BigInt fuzz tests pass with `MaxBigIntResultBits` lowered.

---

## 7. Known Pitfalls & Failed Approaches to Avoid

1. **Pitfall #1 — Emitting `OpMulAccum` when `b` or `c` can mutate `local`**: In JavaScript, `a += (a = 10, 2) * 3` must use the *old* value of `a` (evaluated before the RHS). Only fuse `local += b * c` into `OpMulAdd`/`OpMulAccum` when `b` and `c` are proven side-effect-free (cannot assign to `local` or invoke getters/functions)!
2. **Pitfall #2 — Leaking an `ObjExclusive` BigInt through `0n + x` or `x + 0n`**: If `addSubBigInts` returns `a` or `b` directly without sealing when the other operand is `0n`, two registers can end up pointing to the same `ObjExclusive` `exclusiveBigInt`, causing silent aliasing corruption on the next `OpAccum`. Always call `.Share()` on any returned operand in non-exclusive paths.
3. **Pitfall #3 — Storing a `<= 64`-bit result back into `exclusiveBigInt`**: If `sum` shrinks back into `int64` range (e.g., adding a negative BigInt), `makeBigIntInto` MUST return `BigIntFromInt64` rather than updating the heap `exclusiveBigInt`, preserving `IsStrictlyEqual`'s unique-representation invariant!

---

## 8. `unsafe` Usage & Pure-Safe-Go Alternative

- **`unsafe` used**: Reuses the single existing embedded-header downcast pattern in `internal/storage/bigint.go` (`(*exclusiveBigInt)(unsafe.Pointer(o))`); **zero** new `unsafe` patterns are introduced. All compiler analysis, bytecode dispatch, and stack-buffer limb arithmetic are 100% safe Go.

---

## 9. Verification & Reproduction Protocol

1. Run `go test ./internal/bytecode ./internal/compiler ./internal/engine ./internal/builtin`.
2. Run `go test -tags ekumavalidate -run 'TestExecFastPathsCallNothing|TestRawStorage|TestBigInt' ./internal/engine`.
3. Run `go test -race ./internal/engine ./internal/compiler`.
4. Run `_bench` microbenchmarks (`-suite Micro -engine '^ekuma$' -rounds 1`) and verify: `bigint256_arith` plunges from ~267 µs to ~70 µs (≥ 3.5x vs Phase 1, ≥ 6.5x vs baseline), `bigint64_arith` drops from ~51 µs to ~25 µs (~2x vs Phase 1, ~4.8x vs baseline), and `int_arith`/`float_arith` improve ~15–20%.
