# Optimization #14: SWAR Branchless Short-String Parsing and Single-Pass Clinger Float Parsing

## 1. Summary & Measured Impact

- **Overall Rank**: #10 of 18 (Stage 13)
- **Suite-Wide Impact**: **+2.07 nats** across all 74 microbenchmarks (**1.028x** step geometric-mean lift in Stage 13)
- **Targeted Subset Impact**: **+3.05 nats** across the two string-to-number conversion benchmarks
- **Key Benchmark Improvements (Phase 2 Final → Stage 13 → Final 3-Round Median)**:

| Benchmark | Baseline Ekuma | Phase 2 Ekuma | Stage 13 Ekuma | Final Ekuma (3r) | Stage 13 Speedup | Overall vs Baseline |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| `string_to_float` (numeric coercion of decimal float string in subtraction) | 92.9 ns | 89.7 ns | 13.5 ns | 13.3 ns | **6.62x** (+1.89 nats) | **7.01x** |
| `string_to_int` (numeric coercion of integer string via bitwise OR) | 31.8 ns | 33.2 ns | 10.4 ns | 10.6 ns | **3.18x** (+1.16 nats) | **3.01x** |

---

## 2. Root Cause Analysis

Before Stage 13, numeric coercion of string operands in arithmetic and bitwise operations (`s | 0`, `r -= s`, `+s`) had barely improved over baseline commit `0e377a5`:

1. **Every string-to-number coercion exited `execFast`**: In `internal/engine/interp.go`, `execFast` had no string branch for `OpBitOrK`, `OpSub`, or `OpPlus` (in fact `OpPlus` was absent from `execFast` entirely). Every loop iteration bounced `execFast -> exec -> arith -> ToNumeric -> ToNumber -> reload -> execFast`, accounting for ~90% of total CPU time in both benchmarks.
2. **Loop-based byte-by-byte short-string parsing and Go `BigCaller` inline rejection**: `shortstrings.ParseDecimal` scanned packed bytes one at a time in a loop. Worse, Go's compiler applies a strict `BigCaller` inlining penalty inside large functions like `execFast`, refusing to inline multi-statement helpers (~15+ AST nodes) and triggering `TestExecFastPathsCallNothing/compiled` failures if called from `execFast`.
3. **Three-pass slow path in `numconv.StringToNumber`**: For any decimal string containing a decimal point or exponent (e.g., `"12345.6"` or `"-1.5e-3"`), `StringToNumber` failed its digits-only fast check and ran three separate full scans: `trimWhiteSpace`, `isDecimalLiteral`, and `ParseDecimal` (`strconv.ParseFloat`).

---

## 3. Prerequisites & Dependency Order

- **Depends on Optimization #2** (`execFast` call-free leaf interpreter loop) and **Optimization #9** (`execQuiet` dual-mode quiet loop for non-short strings).
- Completely independent of array, BigInt, RegExp, and Date optimizations.

---

## 4. Target Files Inventory

| File | Role |
| --- | --- |
| `internal/shortstrings/shortstrings.go` | Replace the loop in `ParseDecimal` with branchless 64-bit SWAR digit validation and 3-multiply folding; add `DecimalDigits`, `FoldDecimal`, `StripDot`, `Pow10`, `ParseSignedDecimal`, `ParseUnsignedFloat`, and `ParseShortFloat`. |
| `internal/shortstrings/shortstrings_test.go` | Add `TestParseSignedDecimal` and `TestParseShortFloat` covering empty strings, bare signs/dots, multiple dots, `-0` / `-0.0` sign preservation, leading/trailing dots (`.5`, `5.`), max 8-byte inputs, whitespace, hex, and non-ASCII bytes. |
| `internal/numconv/numconv.go` | Add the 23 exact IEEE 754 `float64` powers of ten (`exactPow10`, $10^0$ through $10^{22}$) and `parseClingerFast`, replacing the digits-only prefix in `StringToNumber` with a single-pass Clinger fast path. |
| `internal/numconv/numconv_test.go` | Add `TestStringToNumberClingerExact` verifying bit-for-bit `math.Float64bits` equality against `strconv.ParseFloat` on boundary cases and 10,000 randomized decimal/exponent strings. |
| `internal/engine/conv.go` | Upgrade `ExecutionContext.ToNumber`'s short-string branch from `shortstrings.ParseDecimal` to `shortstrings.ParseShortFloat`. |
| `internal/engine/interp.go` | Inline branchless SWAR short-string parsing directly into `execFast` for `OpBitOrK`, `OpSub`, and `OpPlus`; add direct `KindString` fast paths (bypassing `ToNumeric` and `reload`, entering `execQuiet` when `!fp.HeapRegs`) in slow `exec` for `OpSub`, `OpSubK`, `OpBitOr`, `OpBitOrK`, and `OpPlus`. |
| `internal/engine/quiet_exec.go` | Add `OpPlus` and string-operand branches for `OpSub`, `OpSubK`, `OpBitOr`, and `OpBitOrK` using `shortstrings.ParseShortFloat` and `numconv.StringToNumber`. |
| `internal/engine/fastpath_test.go` | Allowlist `"bits.TrailingZeros64"` and `"shortstrings.Pow10"` in `fastPathCalls`. |

---

## 5. Detailed Design Specification (No Code)

### 5.1 Branchless 64-Bit SWAR Short-String Parsing (`internal/shortstrings/shortstrings.go`)

Ekuma packs every short string (byte length $1 \le n \le 8$) in little-endian order into the 64 bits of `Value.N` (byte 0 in bits 0..7, byte $n-1$ in bits $8(n-1)..8n-1$, upper $8-n$ bytes zero).

1. **`DecimalDigits(data uint64, n int) (uint64, bool)`**:
   - Reject $n = 0$ immediately (this also safely rejects empty strings and any buffered string whose `o.Text()` length is 0).
   - Right-align the $n$ bytes at the top of the word and fill the unused low $8 - n$ bytes with ASCII `'0'` (`0x30`): compute shift $s = 8n$ and `padded = (data << (64 - s)) | (0x3030303030303030 >> s)` (Go shifts by 64 yield 0 when $n = 8$).
   - Subtract `0x3030303030303030` to obtain `v`. Validate that all 8 bytes lie in `'0'..'9'` in a single branchless expression: `((padded + 0x4646464646464646) | v) & 0x8080808080808080 == 0`. For the lowest non-digit byte $b_k$, no borrow/carry enters byte $k$, so $b_k < \text{0x30}$ sets bit 7 of `v`, $\text{0x3A} \le b_k \le \text{0xAF}$ sets bit 7 of `padded + 0x46...`, and $b_k \ge \text{0xB0}$ sets bit 7 of `v`. All 8 bytes in `'0'..'9'` clear bit 7 in both terms with zero inter-byte borrow/carry.
2. **`FoldDecimal(v uint64) uint32`**:
   - Fold the 8 decimal digit bytes (most significant digit in byte 0, least significant in byte 7) to a `uint32` in three parallel multiplications with zero branches:
     - Pair reduction: `v = (v*10 + (v >> 8)) & 0x00FF00FF00FF00FF`.
     - Quad reduction: `v = (v*100 + (v >> 16)) & 0x0000FFFF0000FFFF`.
     - Final reduction: `uint32(v)*10000 + uint32(v >> 32)`.
3. **`StripDot(data uint64, n int) (digits uint64, dn, frac int)`**:
   - Broadcast `'.'` (`0x2E`): `x = data ^ 0x2e2e2e2e2e2e2e2e`.
   - Apply the standard SWAR zero-byte detector: `dots = (x - 0x0101010101010101) & ^x & 0x8080808080808080`. Unused zero bytes XOR with `0x2E` to `0x2E != 0`, so only active `'.'` bytes set bit 7.
   - If `dots == 0`, return `(data, n, 0)`.
   - Otherwise locate the first dot via `tz = bits.TrailingZeros64(dots)` (dot byte index `tz >> 3`), build `mask = (1 << (tz &^ 7)) - 1`, remove the dot byte via `(data & mask) | ((data >> 8) & ^mask)`, and return new length `n - 1` and fractional digit count `n - 1 - (tz >> 3)`. Any second `'.'` remains in `digits` and automatically fails `DecimalDigits` (`0x2E < 0x30`).
4. **`Pow10(k int) float64`**:
   - Return `shortPow10[k&7]` from a static `[8]float64` table (`1` through `1e7`). Because `n <= 8`, the folded integer significand is at most $99{,}999{,}999 < 2^{53}$ and the divisor is at most $10^7 \le 10^{22}$; both are exact in IEEE 754 `float64`, so a single division `float64(mant) / Pow10(frac)` is guaranteed correctly rounded to nearest-even.
5. **High-level wrappers (`ParseDecimal`, `ParseSignedDecimal`, `ParseUnsignedFloat`, `ParseShortFloat`)**:
   - Compose the primitives above. `ParseShortFloat` strips optional leading `'+'`/`'-'` and preserves `-0.0` via runtime negation `if neg { f = -f }`. `ParseSignedDecimal` strips optional leading `'-'` and returns `int32` (`"-0"` yields `0`, matching `ToInt32`).

### 5.2 Single-Pass IEEE-754-Exact Clinger Float Fast Path (`internal/numconv/numconv.go`)

1. Define `exactPow10` holding $10^0$ through $10^{22}$ as exact `float64` values ($5^{22} < 2^{53} < 5^{23}$).
2. In `StringToNumber`, return `0` immediately for `len(str) == 0`, then invoke `parseClingerFast(str)` before falling back to `trimWhiteSpace` / `isDecimalLiteral` / `ParseDecimal`:
   - Require `0 < len(str) <= 32` (bounding work to $O(1)$ so `poll` is never needed on the fast path).
   - Parse optional leading `'+'`/`'-'` (tracking `neg`), integer ASCII digits, optional `'.'` plus fractional digits (counting `frac`), and optional `'e'`/`'E'` plus optional sign and bounded exponent digits (`exp < 1000`).
   - Accumulate `mant` in `uint64`, aborting to the slow path whenever `mant > (MaxSafeInteger - uint64(d)) / 10` so `mant <= 2^53 - 1` is guaranteed exact in `float64`.
   - Require `digits > 0`, valid exponent digits if `'e'`/`'E'` was present, and `i == len(str)` (automatically rejecting whitespace, hex/binary/octal prefixes `0x`/`0b`/`0o`, `Infinity`, and trailing garbage).
   - If `mant == 0`, return `0.0` negated when `neg` is set (preserving `-0.0`).
   - Compute `exp10 = exp - frac`. If `exp10 == 0`, return `±float64(mant)`. If `0 < exp10 <= 22`, return `±(float64(mant) * exactPow10[exp10])`. If `exp10 > 22`, apply Clinger's extended fast path by multiplying `mant` by 10 while `exp10 > 22 && mant <= MaxSafeInteger/10`, then multiply by `exactPow10[exp10]` if `exp10 <= 22`. If `-22 <= exp10 < 0`, return `±(float64(mant) / exactPow10[-exp10])`. Otherwise return false to fall back to `ParseDecimal`.

### 5.3 Inline `execFast` & `execQuiet` Coercion Paths (`internal/engine/interp.go`, `quiet_exec.go`, `conv.go`)

1. **`execFast` (`internal/engine/interp.go`)**:
   - Add inline SWAR unsigned short-string integer parsing in `OpBitOrK` (6 lines) and inline SWAR unsigned short-string float parsing in `OpSub` (sharing one SWAR block for both `number - shortString` and `shortString - number`) and `OpPlus` (adding `OpPlus` to `execFast` for both `NumberObject` and short strings).
   - Keep the SWAR arithmetic inline rather than calling `shortstrings.DecimalDigits`/`FoldDecimal`/`StripDot`, because Go's `BigCaller` heuristic refuses to inline ~30-node callees inside `execFast` (only tiny ~5-node helpers like `shortstrings.Pow10` and `bits.TrailingZeros64` inline there).
2. **Slow `exec` and `execQuiet` (`internal/engine/interp.go`, `quiet_exec.go`)**:
   - Handle `KindString` operands directly in `OpSub`, `OpSubK`, `OpBitOr`, `OpBitOrK`, and `OpPlus` via `shortstrings.ParseShortFloat` (in `execQuiet`) and `numconv.StringToNumber(s.Str(), poll)`, skipping `ToNumeric`, `ToNumber`, and stack `reload` (since `StringToNumber` cannot invoke user JS), and enter `execQuiet` from slow `exec` when `!fp.HeapRegs` so loops coercing longer float strings (`> 8` bytes) stay bounce-free.
3. **`ExecutionContext.ToNumber` (`internal/engine/conv.go`)**:
   - Replace `shortstrings.ParseDecimal` with `shortstrings.ParseShortFloat` so all other numeric coercions (`Number(s)`, math builtins, typed array stores) also parse short signed/fractional strings directly from `Value.N`.

---

## 6. Architectural Invariants & White-Box Test Updates

- **`TestExecFastPathsCallNothing/source` & `/compiled`**: Add `"bits.TrailingZeros64": true` and `"shortstrings.Pow10": true` to `fastPathCalls` in `internal/engine/fastpath_test.go`, and confirm zero non-barrier `CALL` instructions remain in the compiled `(*agent).execFast` ELF disassembly.
- **Buffered-string safety**: Because buffered strings always have `len(o.Text()) == 0` or `len(o.Text()) > shortstrings.MaxBytes`, checking `n > 0 && n <= shortstrings.MaxBytes` guarantees `Value.N` holds packed short-string bytes.
- **IEEE-754 exactness**: Single rounding only (`mant <= MaxSafeInteger` and $|exp10| \le 22$), preserving `-0.0` on `"-0"`, `"-0.0"`, `"-0e5"`.

---

## 7. Known Pitfalls & Failed Approaches to Avoid

1. **Pitfall #1 — Calling helper functions with > 15 AST nodes inside `execFast`**: Even though `shortstrings.DecimalDigits`, `FoldDecimal`, and `StripDot` were well under Go's standard 80-node inlining budget, Go's `BigCaller` heuristic declined to inline them inside `execFast`, failing `TestExecFastPathsCallNothing/compiled`. Writing the 8–12 lines of SWAR bit arithmetic directly in `OpBitOrK`, `OpSub`, and `OpPlus` (calling only `bits.TrailingZeros64` and `shortstrings.Pow10`) produced zero calls and kept `execFast` compact.
2. **Pitfall #2 — Rejecting `"5"` in dot-tracking logic**: If a dot-index variable is initialized to `n` instead of `-1`, single-digit integer strings (`n == 1`) collide with the bare-dot rejection test `dot == 1 && n == 1`. Using SWAR `dots == 0` / `dots != 0` avoids this entirely: `"."` decrements `n` from 1 to 0, which `n > 0` rejects cleanly.
3. **Pitfall #3 — Passing `"-0"` through `OpPlus` as `+0.0`**: While `"-0" | 0` evaluates to `+0` via `ToInt32`, `+"-0"` and `"-0" - 0` must preserve `-0.0`. Handling unsigned short strings in `execFast` and delegating signed floats (`-` prefix) to `ParseShortFloat` / `parseClingerFast` (which preserve `-0.0`) guarantees exact spec compliance.

---

## 8. `unsafe` Usage & Pure-Safe-Go Alternative

- **`unsafe` used**: **Zero** — 100% pure safe Go (`math` and `math/bits` intrinsics only).

---

## 9. Verification & Reproduction Protocol

1. Run `go test ./internal/shortstrings ./internal/numconv`.
2. Run `go test -run TestExecFastPathsCallNothing ./internal/engine` and `go test -tags ekumavalidate ./internal/engine`.
3. Run `_bench` microbenchmarks (`-suite Micro -engine '^ekuma$' -rounds 1`) and verify: `string_to_int` drops from ~33 ns to ~10–11 ns (~3.0x vs baseline), and `string_to_float` drops from ~90 ns to ~13–14 ns (~7.0x vs baseline).
