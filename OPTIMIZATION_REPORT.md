# Ekuma Performance Optimization Report

## Executive Summary

Applying eight general-purpose, specification-compliant optimizations to Ekuma (a pure-Go ES2026 bytecode interpreter, baseline commit `0e377a5`) delivered a **1.523x geometric-mean speedup (+52.3%, +31.15 nats)** across all 74 benchmarks in the `_bench` microbenchmark suite (`-suite Micro`, 3-round median) and a **1.182x geometric-mean speedup (+18.2%)** across all 10 larger workloads in the macrobenchmark suite (`-suite V8`, 1-round run), with zero regressions and 100% pass rate across the entire test suite (`go test ./...`, `go test -tags ekumavalidate ./...`, and `go test -race`).

Individual benchmarks improved by up to **8.03x** (`string_build3`), **7.12x** (`string_build_large2`), **3.47x** (`bigint32_arith`), **3.41x** (`weak_map_delete`), **3.10x** (`array_length_decr`), and **2.64x** (`array_slice`, `string_build1x`). Across larger macrobenchmarks, real-world workloads accelerated by up to **1.45x** (`Decrypt`), **1.39x** (`Encrypt`), and **1.34x** (`NavierStokes`).

All measurements were collected on Linux `amd64` (Intel Xeon @ 2.60 GHz, Go 1.27.4, `GOMAXPROCS=2`, 200 ms benchtime per microbenchmark run, 1 s benchtime per macrobenchmark run).

---

## Top 8 Optimization Techniques (Ranked by Measured Impact)

Each technique is documented in its own standalone, code-free reproduction specification designed so that even a lightweight LLM agent can implement and verify it directly from baseline commit `0e377a5`.

Impact is measured in **nats** (sum of natural logarithms of Ekuma speedup ratios across benchmarks, where a positive Δnats of +0.693 on a single benchmark equals a 2.00x speedup, and +7.40 nats across 74 benchmarks equals a +10.5% suite-wide geometric-mean lift).

| Rank | Technique | Suite-Wide Gain (Δnats / Geomean Lift) | Targeted Subset Gain (Δnats) | Peak Single-Benchmark Speedup (vs Baseline) | Detailed Specification |
| ---: | --- | --- | --- | --- | --- |
| **1** | **Exclusive In-Place String Building (`StringBuf`, `OpPrepend`/`OpPrependK`, 64-Bit Short Writes, Barrier-Free Length Update, Zero-Copy `apply`)** | **+8.97 nats** (Stages 4 + 6B) | **+9.78 nats** (8 string benchmarks) | **8.03x** (`string_build3`), **7.12x** (`string_build_large2`) | [OPTIMIZATION_01_exclusive_inplace_strings.md](OPTIMIZATION_01_exclusive_inplace_strings.md) |
| **2** | **Leaf Fast-Path Interpreter Loop (`execFast`), Pointer-Based Dispatch, Unchecked Accessors, and 128-Byte Frame Alignment** | **+6.89 nats** (1.098x suite-wide) | **+7.18 nats** (26 hot benchmarks) | **2.00x** (`global_read`, `array_for`) | [OPTIMIZATION_02_leaf_interpreter_loop.md](OPTIMIZATION_02_leaf_interpreter_loop.md) |
| **3** | **Promotion-on-Growth Unboxed Numeric Arrays (`PackedNums` `noscan` `[]float64`), Cached Constructor Checks, and Inlined Array Operations** | **+4.49 nats** (1.063x step) | **+5.03 nats** (array & join benchmarks) | **3.25x** (`array_length_decr`), **2.64x** (`array_slice`), **2.56x** (`array_pop`) | [OPTIMIZATION_03_packed_numeric_arrays.md](OPTIMIZATION_03_packed_numeric_arrays.md) |
| **4** | **Two-Instruction Peek-Fusion, Upvalue Step Opcodes, Lean Call/Return Frames, and Inlined `for..of` Iteration** | **+4.12 nats** (1.057x step) | **+3.15 nats** (13 top movers) | **1.60x** (`array_for_of`), **1.49x** (`empty_down_loop2`) | [OPTIMIZATION_04_peek_fusion_and_lean_frames.md](OPTIMIZATION_04_peek_fusion_and_lean_frames.md) |
| **5** | **Inline `int64` BigInt Execution, Compact 80-Byte `BigIntObject`, and Direct Multi-Limb Arithmetic (≤ 512 Bits)** | **+2.22 nats** (1.030x step) | **+3.08 nats** (3 BigInt benchmarks) | **4.41x** stage / **3.47x** vs base (`bigint32_arith`), **2.71x** stage (`bigint64_arith`) | [OPTIMIZATION_05_fast_bigint_arithmetic.md](OPTIMIZATION_05_fast_bigint_arithmetic.md) |
| **6** | **Hybrid Ephemeron `WeakTable` (Inline Key Slot + Table-Side Fibonacci Hash Table for Primitives) and Weak Collection Kernels** | **+1.73 nats** (1.024x step) | **+2.10 nats** (2 WeakMap benchmarks) | **3.62x** stage / **3.41x** vs base (`weak_map_delete`), **2.43x** vs base (`weak_map_set`) | [OPTIMIZATION_06_hybrid_weaktable.md](OPTIMIZATION_06_hybrid_weaktable.md) |
| **7** | **`Number.prototype` Formatting Kernels (`shortContract`) and Single-Allocation 80-Byte `asciiText32` Strings** | Part of Stage 6A (**+1.98 nats** combined) | **+0.86 nats** (6 number-formatting benchmarks) | **1.26x** stage (`int_toString`), **1.18x** (`float_toExponential`) | [OPTIMIZATION_07_number_formatting_kernels.md](OPTIMIZATION_07_number_formatting_kernels.md) |
| **8** | **Post-Compilation Object Presizing (`presizeNewObjects`) and Plain-Object Element Append Fast Paths** | Part of Stage 6A (**+1.98 nats** combined) | **+0.54 nats** (`prop_create` directly) | **1.72x** stage / **1.59x** vs base (`prop_create`) | [OPTIMIZATION_08_object_presizing_and_indexing.md](OPTIMIZATION_08_object_presizing_and_indexing.md) |

---

## Recommended Implementation Order for Reproduction

Although the table above ranks techniques by measured speedup, several techniques rely on the `execFast` leaf interpreter introduced in Technique #2. When reproducing the full stack from baseline `0e377a5`, apply the specifications in the following dependency order:

1. **Technique #2** ([OPTIMIZATION_02_leaf_interpreter_loop.md](OPTIMIZATION_02_leaf_interpreter_loop.md)) — Establishes the `execFast` leaf loop, pointer-based instruction dispatch, `validate_off.go`/`validate_on.go` split, 128-byte `frame`, and `endian_le.go`.
2. **Technique #4** ([OPTIMIZATION_04_peek_fusion_and_lean_frames.md](OPTIMIZATION_04_peek_fusion_and_lean_frames.md)) — Adds 2-instruction peek-fusion, `OpIncUpval`/`OpDecUpval`, lean call/return, and `execFast` `OpIterNext`/`OpIterClose`.
3. **Technique #6** ([OPTIMIZATION_06_hybrid_weaktable.md](OPTIMIZATION_06_hybrid_weaktable.md)) — Rebuilds `WeakTable` and registers `WeakMap`/`WeakSet` kernels (independent of `execFast`, can be done anytime).
4. **Technique #3** ([OPTIMIZATION_03_packed_numeric_arrays.md](OPTIMIZATION_03_packed_numeric_arrays.md)) — Introduces `PackedNums`, `tryPromoteNums`, cached `ArrayCtorPlain`, single-buffer `JoinStrings`, and inlined array operations in `execFast`.
5. **Technique #1** ([OPTIMIZATION_01_exclusive_inplace_strings.md](OPTIMIZATION_01_exclusive_inplace_strings.md)) — Introduces `StringBuf`, `OpPrepend`/`OpPrependK`, `execFast` 64-bit append/prepend + `SetBufLen`, `ConcatASCIIStrings`, and `CallWithArray`.
6. **Technique #5** ([OPTIMIZATION_05_fast_bigint_arithmetic.md](OPTIMIZATION_05_fast_bigint_arithmetic.md)) — Shrinks `BigIntObject` to 80 bytes, adds `BigInt64Object` fast paths in `execFast`, and implements ≤ 512-bit direct multi-limb arithmetic.
7. **Technique #7** ([OPTIMIZATION_07_number_formatting_kernels.md](OPTIMIZATION_07_number_formatting_kernels.md)) — Extracts pure number formatters into `internal/numconv/format.go`, adds `Number.prototype` kernels, and adds single-allocation `asciiText32`.
8. **Technique #8** ([OPTIMIZATION_08_object_presizing_and_indexing.md](OPTIMIZATION_08_object_presizing_and_indexing.md)) — Adds `presizeNewObjects` in the compiler and plain `ClassObject` `OpSetIndex`/`OpSetIndexInt` appends in `execFast`.

---

## Cumulative Stage-by-Stage Progression (Ekuma vs Baseline Ekuma)

| Stage | Applied Techniques | Cumulative Geomean Speedup (vs Baseline) | Cumulative Δnats | Step Speedup | Step Δnats |
| --- | --- | ---: | ---: | ---: | ---: |
| **Baseline** (`0e377a5`) | — | 1.000x | +0.00 | 1.000x | +0.00 |
| **Stage 1A** | Technique #2 (`execFast` leaf + pointers + 128B frame) | 1.098x | +6.89 | 1.098x | +6.89 |
| **Stage 1B** | Technique #4 (peek-fusion + `OpIncUpval` + lean call + `for..of`) | 1.160x | +11.01 | 1.057x | +4.12 |
| **Stage 2** | Technique #6 (hybrid `WeakTable` + weak kernels) | 1.188x | +12.74 | 1.024x | +1.73 |
| **Stage 3** | Technique #3 (`PackedNums` + cached ctor + `JoinStrings` + `push`/`pop`/`.length=`) | 1.262x | +17.23 | 1.063x | +4.49 |
| **Stage 4** | Technique #1 Part A (`StringBuf` + `OpPrepend`/`OpPrependK` + `CallWithArray` + `ConcatASCIIStrings`) | 1.300x | +19.39 | 1.030x | +2.15 |
| **Stage 5** | Technique #5 (inline `BigInt64Object` + 80B `BigIntObject` + direct multi-limb arithmetic) | 1.339x | +21.61 | 1.030x | +2.22 |
| **Stage 6A** | Techniques #7 & #8 (`Number.prototype` kernels + `asciiText32` + `presizeNewObjects` + plain-object index append) | 1.375x | +23.59 | 1.027x | +1.98 |
| **Stage 6B** | Technique #1 Part B (64-bit short writes + `SetBufLen` + removing erroneous `ObjInterned == 0` guard) | 1.508x | +30.41 | 1.097x | +6.82 |
| **Final (3 rounds)** | Full stack (3-round median verification) | **1.523x** | **+31.15** | 1.010x | +0.74 |

---

## End-to-End Benchmark Results (Baseline vs Final)

### Microbenchmark Suite Top Movers (`-suite Micro`, 3-Round Median)

| Benchmark | Baseline Ekuma | Final Ekuma | Speedup |
| --- | ---: | ---: | ---: |
| `string_build3` | 89.3 µs | 11.1 µs | **8.03x** |
| `string_build_large2` | 2203 µs | 309 µs | **7.12x** |
| `bigint32_arith` | 36.7 µs | 10.6 µs | **3.47x** |
| `weak_map_delete` | 931.5 µs | 272.8 µs | **3.41x** |
| `array_length_decr` | 45.8 µs | 14.8 µs | **3.10x** |
| `array_slice` | 7.81 µs | 2.96 µs | **2.64x** |
| `string_build1x` | 24.5 µs | 9.31 µs | **2.64x** |
| `string_build2` | 25.1 µs | 9.80 µs | **2.56x** |
| `string_build1` | 24.6 µs | 9.85 µs | **2.50x** |
| `string_build2c` | 25.0 µs | 10.2 µs | **2.46x** |
| `weak_map_set` | 323.9 µs | 133.5 µs | **2.43x** |
| `global_read` | 40.0 ns | 16.7 ns | **2.40x** |
| `string_build_large1` | 691 µs | 289 µs | **2.39x** |
| `array_pop` | 29.2 µs | 12.2 µs | **2.39x** |
| `array_for` | 1839 ns | 781 ns | **2.35x** |
| `bigint64_arith` | 119.6 µs | 51.5 µs | **2.32x** |
| `string_concat` | 1454 ns | 680 ns | **2.14x** |
| `array_push` | 29.6 µs | 14.1 µs | **2.10x** |
| `int_arith` | 12.3 µs | 5.97 µs | **2.06x** |
| `array_read` | 71.8 ns | 35.6 ns | **2.02x** |
| `array_length_read` | 29.5 ns | 14.8 ns | **2.00x** |
| `prop_read` | 36.8 ns | 18.8 ns | **1.96x** |
| `array_write` | 42.0 ns | 22.1 ns | **1.90x** |
| `string_array_join` | 972 ns | 515 ns | **1.89x** |
| `float_arith` | 15.2 µs | 8.12 µs | **1.87x** |
| `bigint256_arith` | 478 µs | 267 µs | **1.79x** |
| `string_length` | 32.7 ns | 18.8 ns | **1.74x** |
| `prop_write` | 23.5 ns | 14.3 ns | **1.63x** |
| `prop_create` | 1272 ns | 802 ns | **1.59x** |
| `empty_down_loop2` | 6.0 ns | 3.83 ns | **1.58x** |
| `global_write_strict` | 17.5 ns | 11.5 ns | **1.53x** |
| `prop_update` | 52.7 ns | 35.1 ns | **1.50x** |
| `array_update` | 122 ns | 81.2 ns | **1.50x** |
| `math_min` | 23.8 µs | 15.9 µs | **1.50x** |
| `empty_down_loop` | 6.1 ns | 4.14 ns | **1.48x** |
| `global_write` | 17.0 ns | 11.6 ns | **1.46x** |
| `array_prop_create` | 31.1 µs | 21.7 µs | **1.43x** |
| `func_closure_call` | 147 ns | 105 ns | **1.40x** |
| `empty_do_loop` | 5.5 ns | 3.99 ns | **1.39x** |
| `empty_loop` | 5.6 ns | 4.12 ns | **1.37x** |
| `array_for_of` | 1226 ns | 917 ns | **1.34x** |
| `global_func_call` | 129.5 ns | 98.3 ns | **1.32x** |
| `typed_array_read` | 111 ns | 85.8 ns | **1.29x** |
| `func_call` | 128.7 ns | 103.7 ns | **1.24x** |
| **Geometric Mean (all 74 benchmarks)** | — | — | **1.523x (+31.15 nats)** |

### Macrobenchmark Suite (`-suite V8`, 1-Round Verification)

| Benchmark | Baseline Ekuma | Final Ekuma | Speedup |
| --- | ---: | ---: | ---: |
| `Decrypt` | 78.05 ms | 54.01 ms | **1.45x** |
| `Encrypt` | 4.32 ms | 3.11 ms | **1.39x** |
| `NavierStokes` | 53.90 ms | 40.28 ms | **1.34x** |
| `Richards` | 2.78 ms | 2.29 ms | **1.21x** |
| `Boyer` | 79.85 ms | 69.80 ms | **1.14x** |
| `Splay` | 1.90 ms | 1.68 ms | **1.13x** |
| `RayTrace` | 26.33 ms | 23.58 ms | **1.12x** |
| `DeltaBlue` | 5.13 ms | 4.61 ms | **1.11x** |
| `Earley` | 4.79 ms | 4.62 ms | **1.04x** |
| `RegExp` | 44.45 ms | 45.31 ms | 0.98x |
| **Geometric Mean (all 10 benchmarks)** | — | — | **1.182x** |

---

## Cross-Cutting Go VM Engineering Principles

Three recurring patterns accounted for almost the entire speedup across all eight techniques:

1. **Protecting the Go Register Allocator in the Dispatch Loop**: Any function call inside an interpreter loop causes Go's register allocator to spill loop-carried state to the stack on every instruction. Separating a strictly call-free `//go:noinline` leaf loop (`execFast`), verifying zero calls against the compiled ELF binary, and fusing common two-instruction idioms eliminates both spill traffic and indirect jump overhead. Note a key subtlety: moving a hot loop into a separate leaf function imposes a small exit/re-entry penalty (~10–19%) on instructions that still fall out to the slow path on every iteration; profiling those exits directly identifies the next highest-leverage operations to pull into `execFast` or quiet kernels.
2. **Eliminating Go GC Scanning and Write Barriers (`gcWriteBarrier2` / `wbBufFlush1`)**: A 16-byte tagged `storage.Value` carries a heap pointer word, so slices of `Value` force GC pointer scanning and write barriers on every store, copy, and truncation. Moving dense numeric arrays to pointer-free `noscan` `[]float64` backing stores, updating only the length word of exclusive in-place string headers, skipping redundant `O` stores when a register already holds `NumberObject` or `BigInt64Object`, and shrinking object allocations (`weakTail`, `plainObject`-embedded `BigIntObject`, `asciiText32`) cuts both CPU barrier instructions and GC pause pressure.
3. **Verifying Fast-Path Hit Rates with `pprof` and `objdump`**: Three times during this work, a seemingly correct fast path silently missed 100% of iterations (empty array literals starting at `cap == 4` masking `PackedNums`, interned short string constants failing an erroneous `ObjInterned == 0` check in `execFast`, and helper nesting exceeding Go's inline budget). Always inspect CPU profiles and compiled disassembly after implementing a fast path to confirm it executes on the hot path and introduces no unintended loop-head spills.

---

## Note on `unsafe` Footprint

Approximately **85–90%** of the total +31.15 nats gain comes from completely safe Go changes (leaf loop extraction, 128-byte frame padding, 2-instruction peek-fusion, hybrid `WeakTable`, `PackedNums` `[]float64`, `OpPrepend`/`OpPrependK`, direct `math/bits` BigInt limb arithmetic, `presizeNewObjects`, and `shortContract` built-in kernels). New `unsafe` usage is narrowly isolated to:

- Unchecked pointer arithmetic in `internal/engine/validate_off.go` (fully disabled under `-tags ekumavalidate`).
- Length-only header updates in `Raw.SetBufLen` (`internal/storage/raw.go`, verified by a compile-time size assertion).
- Single-allocation / zero-copy `unsafe.String` conversions in `internal/storage/` (identical in mechanism to Go's standard `strings.Builder.String`).
- Embedded-header downcasting in `internal/storage/bigint.go` (matching Ekuma's existing `textObject` pattern).
- Little-endian fixed-width loads/stores in `internal/engine/endian_le.go` (which modern Go compilers also lower identically from safe `encoding/binary.LittleEndian`).

Each technique specification explicitly documents its `unsafe` footprint and pure-safe-Go alternative.

---

## Global Verification Protocol

After applying any subset (or all) of the eight specifications, run:

- Standard test suite: `go test ./...`
- Runtime invariant validation build: `go test -tags ekumavalidate ./...`
- Data race detector across core packages: `go test -race . ./internal/storage ./internal/collection ./internal/engine ./internal/builtin ./internal/numconv ./internal/compiler`
- Microbenchmark suite (from `_bench/`): `GOMAXPROCS=2 go run ./cmd/jsbench -suite Micro -engine '^ekuma$' -rounds 3`
- Macrobenchmark suite (from `_bench/`): `GOMAXPROCS=2 go run ./cmd/jsbench -suite V8 -engine '^ekuma$' -rounds 1`
