# Optimization #18: Loop-Free Pure-Go Schubfach Shortest-Decimal Float64 Formatting and `digitPairs` Integer Formatting

## 1. Summary & Measured Impact

- **Overall Rank**: #17 of 18 (Stage 17)
- **Suite-Wide Impact**: **+0.59 nats** across all 74 microbenchmarks (**1.008x** step geometric-mean lift in Stage 17, combined with Stage 17 compound-update polish)
- **Targeted Subset Impact**: **+0.30 nats** across the five float-formatting and two compound-update benchmarks
- **Key Benchmark Improvements (Stage 16 → Stage 17 → Final 3-Round Median)**:

| Benchmark | Baseline Ekuma | Stage 16 Ekuma | Stage 17 Ekuma | Final Ekuma (3r) | Stage 17 Speedup | Overall vs Baseline |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| `float_toFixed` (`Number.prototype.toFixed`) | 421.4 ns | 327.1 ns | 308.6 ns | 313.2 ns | **1.06x** (+0.06 nats) | **1.35x** |
| `float_toExponential` (`Number.prototype.toExponential`) | 472.1 ns | 351.0 ns | 333.4 ns | 329.9 ns | **1.05x** (+0.05 nats) | **1.43x** |
| `float_to_string` (`(x) + ''`) | 307.6 ns | 258.8 ns | 251.8 ns | 253.8 ns | **1.03x** (+0.03 nats) | **1.21x** |
| `float_toString` (`(x).toString()`) | 379.6 ns | 272.4 ns | 268.1 ns | 274.9 ns | **1.02x** (+0.02 nats) | **1.38x** |
| `float_toPrecision` (`Number.prototype.toPrecision`) | 426.3 ns | 326.4 ns | 321.2 ns | 324.0 ns | **1.02x** (+0.02 nats) | **1.32x** |

---

## 2. Root Cause Analysis

CPU profiling of `float_to_string` and `float_toFixed` on Go 1.27 revealed why `strconv.AppendFloat` and `strconv.AppendUint` still consumed 45.8% of `float_to_string` and 10% of `float_toFixed`:

1. **Multi-layer formatting wrappers in `strconv.AppendFloat`**: While Go 1.27's internal shortest-decimal computation (`shortFloat`) itself accounted for only 8.3% of `float_to_string`, the surrounding general-purpose formatting machinery — `setDigits` (16.7%), `formatBase10` (8.3%), `fmtEFG` (8.3%), and `ftoa64` (4.2%) — wrote digits into an intermediate buffer and re-copied them under generic printf rules before `AppendNumber` reformatted out-of-range exponents into ECMAScript syntax.
2. **General-purpose `strconv.AppendUint`/`AppendInt` in `format.go`**: In `appendHalfUpFixed` (`toFixed`) and `appendScientific` (`toPrecision`/`toExponential`), formatting the integer part and exponent still called `strconv.AppendUint` / `strconv.AppendInt` instead of reusing `format.go`'s two-digit `digitPairs` lookup table.
3. **Narrow integer threshold in `AppendNumber`**: `AppendNumber` only fast-pathed exact integers in `(-1e15, 1e15)` rather than the full `(-MaxSafeInteger, MaxSafeInteger)` range (`±(2^53 - 1)`).

---

## 3. Prerequisites & Dependency Order

- **Depends on Optimization #7** (`internal/numconv/format.go`, `digitPairs`, `uint64Pow10`, `appendDigits`) and **Optimization #9** / **Optimization #16** (`execQuiet` float formatting dispatch).

---

## 4. Target Files Inventory

| File | Role |
| --- | --- |
| `internal/numconv/ryu_table.go` | New file holding the 617-entry 126-bit upper-bound power-of-ten table `schubfachPow10` (`[617][2]uint64`) for $k \in [-324, 292]$ (`schubfachKMin = -324`). |
| `internal/numconv/ryu.go` | New file implementing pure-Go loop-free Schubfach (`schubfachRop`, `schubfach`), O(1) binary-search trailing-zero stripping, `appendUintFast` (`digitPairs`), direct ECMAScript `appendSchubfach`, and `appendSchubfachExp`. |
| `internal/numconv/numconv.go` | Replace the `strconv.AppendFloat` + `strconv.AppendInt` body of `AppendNumber` with `appendUintFast` (for exact integers within `±MaxSafeInteger`) and `appendSchubfach`. |
| `internal/numconv/format.go` | Replace `strconv.AppendFloat` + `fixExponent` in `AppendExponentialAuto` with `appendSchubfachExp`; replace `strconv.AppendUint` in `appendHalfUpFixed` and `strconv.AppendInt` in `appendScientific` with `appendUintFast`. |
| `internal/numconv/numconv_test.go` | Add `TestAppendNumberSchubfachExhaustive` verifying bit-for-bit string equality against a `strconv`-based ECMAScript reference across special values, first 2,000 subnormals, normal/subnormal boundary, all powers of 2 ±1 ULP, all powers of 10 ±1 ULP, and 100,000 random `float64` values. |

---

## 5. Detailed Design Specification (No Code)

### 5.1 Precomputed 126-Bit Power-of-Ten Table (`internal/numconv/ryu_table.go`)

For every integer $k \in [-324, 292]$ (617 entries, 9.87 KB read-only static array):
- Let $e = -k$ and $s = \lfloor e \cdot \log_2(10) \rfloor = (e \cdot 1741647) \gg 19$ (mathematically exact for all $e \in [-292, 324]$).
- Compute in exact arbitrary precision $g = \lfloor 10^e \cdot 2^{125 - s} \rfloor + 1$ (always in $[2^{125}+1, 2^{126}]$) and store `{uint64(g >> 64), uint64(g)}` at `schubfachPow10[k - (-324)]`.

### 5.2 Loop-Free Schubfach Shortest-Decimal Algorithm (`internal/numconv/ryu.go`)

1. **`schubfachRop(g1, g0, cp uint64) uint64`**:
   - Compute round-to-odd of $(cp \cdot g) / 2^{127}$ using two `bits.Mul64` calls (`x1, _ := bits.Mul64(g0, cp)`; `y1, y0 := bits.Mul64(g1, cp)`) and one `bits.Add64(y0, x1, 0)` giving `(sum, carry)`.
   - Compute `res := ((y1 + carry) << 1) | (sum >> 63)` and set bit 0 (`res |= 1`) if `sum & ((1 << 63) - 1) != 0`. Because $cp < 2^{60}$ and $g < 2^{126}$, `res < 2^59` never overflows `uint64`.
2. **`schubfach(f float64) (d uint64, e int)`**:
   - Unpack `f > 0` into binary significand $c$ and exponent $q$ (normal: $c = \text{mant} \mid 2^{52}, q = \text{exp} - 1075$; subnormal: $c = \text{mant}, q = -1074$). Let `out = c & 1`, `cb = c << 2`, `cbr = cb + 2`.
   - Regular case (`mant != 0 || exp <= 1`): `cbl = cb - 2`, `k = (q * 315653) >> 20` (exact $\lfloor q \log_{10} 2 \rfloor$ for all $q \in [-1074, 971]$).
   - Irregular case (`mant == 0 && exp > 1`): `cbl = cb - 1`, `k = (q*315653 - 131008) >> 20` (exact $\lfloor q \log_{10} 2 - \log_{10}(4/3) \rfloor$ for all $q \in [-1074, 971]$).
   - Compute $h = q + ((-k \cdot 1741647) \gg 19) + 2$ (always in $[2, 5]$) and look up `(g1, g0) = schubfachPow10[k - schubfachKMin]`.
   - Compute `vb`, `vbl`, `vbr` via `schubfachRop(g1, g0, {cb,cbl,cbr} << h)` and `s = vb >> 2`.
   - **Shorter 10-multiple check (`if s >= 10`)**: compute `sp = s / 10`, `sp10 = sp * 10`, `upin = vbl + out <= sp10 << 2`, `wpin = (sp10 + 10) << 2 + out <= vbr`. If `upin != wpin`, select `sp` (if `upin`) or `sp + 1` (else) with exponent `k + 1`.
   - **Otherwise** test `s` vs `t = s + 1`: `uin = vbl + out <= s << 2`, `win = t << 2 + out <= vbr`. If `uin != win`, select `s` or `t`; on tie (`uin == win`), compare `vb` against `(s + t) << 1` and break exact halfway ties toward even `s & 1 == 0`.
   - **O(1) Binary-Search Trailing-Zero Stripping**: when `d % 10 == 0`, strip factors of $10^8$ in a loop (runs at most twice for up to 16 trailing zeros), then test $10^4$, $100$, and $10$ once each (each constant division compiles to a single magic multiply).

### 5.3 Direct ECMAScript Formatting & `appendUintFast` (`ryu.go`, `numconv.go`, `format.go`)

1. **`appendUintFast(b []byte, u uint64) []byte`**:
   - Handle `u < 10` (1 byte) and `u < 100` (2 bytes from `digitPairs`) inline; for larger `u`, extract two digits per `/100` into a stack `[20]byte` buffer from the back and append once.
2. **`appendSchubfach(dst []byte, n float64) []byte`**:
   - Emit `'-'` and negate if `n < 0`; run `d, e := schubfach(n)`.
   - Compute exact decimal digit count $k \in [1, 17]$ in O(1) via `k := bits.Len64(d)*1233 >> 12; if d >= uint64Pow10[k] { k++ }`, and ECMAScript decimal position `pos := e + k`.
   - Format directly into `dst` with zero intermediate `decimalSlice` or `fmtEFG` copy:
     - $k \le \text{pos} \le 21$: `appendDigits(dst, d, k)` + `pos - k` zeros.
     - $0 < \text{pos} < k$: split `hi := d / uint64Pow10[k-pos]` and `lo := d - hi*pw`, write `appendDigits(hi, pos)` + `'.'` + `appendDigits(lo, k-pos)` directly with zero byte shifting.
     - $-6 < \text{pos} \le 0$: write `"0.0000000"[:2-pos]` + `appendDigits(dst, d, k)`.
     - Otherwise (scientific): write leading digit, optional `'.'` + remaining `k - 1` digits, `'e'`, `'+'`/`'-'`, and `appendUintFast(abs(pos - 1))`.
3. **`appendSchubfachExp`**: Format `d, e := schubfach(n)` directly as `d0[.d1..dk-1]e±exp` for `AppendExponentialAuto` (0-arg `toExponential()`).
4. Wire `appendUintFast` into `AppendNumber` (expanding exact integer range to `(-MaxSafeInteger, MaxSafeInteger)`), `appendHalfUpFixed`, and `appendScientific`.

---

## 6. Architectural Invariants & White-Box Test Updates

- **Strict IEEE 754 shortest round-trip equivalence**: Every finite `float64` (normal and subnormal) must format bit-for-bit identically to ECMA-262 6.1.6.1.20 (`Number::toString`), verified by `TestAppendNumberSchubfachExhaustive` across 100,000+ values including all powers of 2/10 and subnormal boundaries.
- **No white-box engine allowlist changes required**: All changes live inside `internal/numconv`.

---

## 7. Known Pitfalls & Failed Approaches to Avoid

1. **Pitfall #1 — Using `s >= 100` instead of `s >= 10` for the 10-multiple check**: Small subnormals such as `8e-323` (`s = 79`) and `1e-322` (`s = 99`) have $10 \le s < 100$ and require checking `sp10`/`tp10` so they format as `"8e-323"` and `"1e-322"` rather than `"7.9e-323"` and `"9.9e-323"`. Always gate on `s >= 10`.
2. **Pitfall #2 — Stripping `1e8` only once during trailing-zero removal**: Because `d` can have up to 17 digits (up to 16 trailing zeros, e.g., $10^{16}$), a single `1e8` + `1e4` + `100` + `10` cascade strips at most $8 + 4 + 2 + 1 = 15$ zeros, leaving one trailing zero on $10^{16}$. Looping `1e8` (at most 2 iterations) handles all 0..23 trailing zeros in O(1).

---

## 8. `unsafe` Usage & Pure-Safe-Go Alternative

- **`unsafe` used**: **Zero** — 100% pure safe Go (`math` and `math/bits` only).

---

## 9. Verification & Reproduction Protocol

1. Run `go test -v -run 'TestNumberToString|TestAppendNumberSchubfachExhaustive' ./internal/numconv` and `go test ./internal/numconv`.
2. Run `go test ./...` and `go test -tags ekumavalidate ./...`.
3. Run `_bench` microbenchmarks (`-suite Micro -engine '^ekuma$' -rounds 1`) and verify improvements across `float_toFixed`, `float_toExponential`, `float_to_string`, `float_toString`, and `float_toPrecision`.
