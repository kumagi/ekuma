# Ekuma Performance Optimization Report

## Executive Summary

Applying eighteen general-purpose, specification-compliant, strictly no-JIT optimizations to Ekuma (a pure-Go ES2026 bytecode interpreter, baseline commit `0e377a5`) delivered a **2.062x geometric-mean speedup (+106.2%, +53.56 nats)** across all 74 benchmarks in the `_bench` microbenchmark suite (`-suite Micro`, 3-round median) with **zero regressions — every single one of the 74 microbenchmarks is at least 1.11x faster than baseline** — and a **1.182x geometric-mean speedup (+18.2%, +1.67 nats)** across all 10 larger workloads in the macrobenchmark suite (`-suite V8`, 1-round run), where **all 10 macrobenchmarks are faster than baseline**, with a **100% pass rate** across `go test ./...`, `go test -tags ekumavalidate ./...`, and `go test -race`.

Individual microbenchmarks improved by up to **7.94x** (`string_build3`), **7.21x** (`bigint256_arith`), **7.01x** (`string_to_float`), **6.94x** (`string_build_large2`), **5.21x** (`bigint64_arith`), **4.23x** (`func_closure_call`), **4.02x** (`weak_map_delete`), **3.76x** (`bigint32_arith`), **3.65x** (`func_call`), **3.32x** (`global_func_call`), **3.17x** (`array_length_decr`), **3.01x** (`string_to_int`), and **3.00x** (`weak_map_set`). Across the 10 macrobenchmarks, real-world workloads accelerated by up to **1.44x** (`Decrypt`), **1.43x** (`NavierStokes`), **1.42x** (`Encrypt`), and **1.19x** (`Richards`), with `Earley` reaching **1.09x** and `RegExp` reaching **1.08x**.

All measurements were collected on Linux `amd64` (Intel Xeon @ 2.60 GHz, Go 1.27.4, `GOMAXPROCS=2`, 200 ms benchtime per microbenchmark run, 1 s benchtime per macrobenchmark run).

---

## All 18 Optimization Techniques (Ranked by Measured Impact)

Each technique is documented in its own standalone, code-free reproduction specification designed so that even a lightweight LLM agent can implement and verify it directly from baseline commit `0e377a5`. File numbering preserves Phase 1 (`01`–`08`), Phase 2 (`09`–`13`), and Phase 3 (`14`–`18`), while the table below ranks all 18 techniques strictly by measured suite-wide impact.

Impact is measured in **nats** (sum of natural logarithms of Ekuma speedup ratios across benchmarks, where a positive Δnats of +0.693 on a single benchmark equals a 2.00x speedup, and +51.29 nats across 74 benchmarks equals a 2.000x suite-wide geometric-mean speedup).

| Rank | Technique | Suite-Wide Gain (Δnats / Geomean Lift) | Targeted Subset Gain (Δnats) | Peak Single-Benchmark Speedup (vs Baseline) | Detailed Specification |
| ---: | --- | --- | --- | --- | --- |
| **1** | **Exclusive In-Place String Building (`StringBuf`, `OpPrepend`/`OpPrependK`, 64-Bit Short Writes, Barrier-Free Length Update, Zero-Copy `apply`)** | **+8.97 nats** (Stages 4 + 6B) | **+9.78 nats** (8 string benchmarks) | **7.94x** (`string_build3`), **6.94x** (`string_build_large2`) | [OPTIMIZATION_01_exclusive_inplace_strings.md](OPTIMIZATION_01_exclusive_inplace_strings.md) |
| **2** | **Leaf Fast-Path Interpreter Loop (`execFast`), Pointer-Based Dispatch, Unchecked Accessors, and 128-Byte Frame Alignment** | **+6.89 nats** (1.098x step) | **+7.18 nats** (26 hot benchmarks) | **2.42x** (`global_read`), **2.37x** (`array_for`) | [OPTIMIZATION_02_leaf_interpreter_loop.md](OPTIMIZATION_02_leaf_interpreter_loop.md) |
| **3** | **Dual-Mode Quiet-Kernel Loop (`execQuiet`), Direct `collection.Table` Probing, Small-Int ShortString Packing, and Primitive `Number` IC Fix** | **+5.05 nats** (Stage 9 + Stage 11 loop-glue expansion) | **+5.64 nats** (Map, int/float formatting, string concat) | **2.53x** (`int_to_string`), **2.43x** (`int_toString`), **1.81x** (`map_set_string`) | [OPTIMIZATION_09_quiet_kernel_mode_and_collections.md](OPTIMIZATION_09_quiet_kernel_mode_and_collections.md) |
| **4** | **Promotion-on-Growth Unboxed Numeric Arrays (`PackedNums` `noscan` `[]float64`), Cached Constructor Checks, and Inlined Array Operations** | **+4.49 nats** (1.063x step) | **+5.03 nats** (array & join benchmarks) | **3.17x** (`array_length_decr`), **2.72x** (`array_slice`), **2.32x** (`array_pop`) | [OPTIMIZATION_03_packed_numeric_arrays.md](OPTIMIZATION_03_packed_numeric_arrays.md) |
| **5** | **Two-Instruction Peek-Fusion, Upvalue Step Opcodes, Lean Call/Return Frames, and Inlined `for..of` Iteration** | **+4.12 nats** (1.057x step) | **+3.15 nats** (13 top movers) | **1.72x** (`empty_down_loop2`), **1.52x** (`array_for_of`) | [OPTIMIZATION_04_peek_fusion_and_lean_frames.md](OPTIMIZATION_04_peek_fusion_and_lean_frames.md) |
| **6** | **Frameless Leaf Function Calls, Surplus-Argument Inline Frames, `OpReturn`/`OpGetArgsIndexInt` Fusion, and Cached `ProtoElemsDirty` Guard** | **+3.28 nats** (1.045x step, Stage 7) | **+3.80 nats** (calls, `arguments`, `array_push`) | **4.23x** (`func_closure_call`), **3.65x** (`func_call`), **3.32x** (`global_func_call`) | [OPTIMIZATION_10_frameless_leaf_calls_and_proto_guard.md](OPTIMIZATION_10_frameless_leaf_calls_and_proto_guard.md) |
| **7** | **RegExp Whole-Literal Fast Path, Single-Allocation Match Results, `RegExp`/`String.replace`/`Array.slice` Kernels, and Integer-Fast `Date` Kernels** | **+2.83 nats** (Stage 11 RegExp + Date + Slice subset) | **+2.67 nats** (3 RegExp + 2 Date benchmarks) | **1.96x** (`regexp_utf16`), **1.95x** (`regexp_ascii`), **1.88x** (`regexp_replace`), **1.42x** (`date_parse`) | [OPTIMIZATION_11_regexp_literal_and_date_kernels.md](OPTIMIZATION_11_regexp_literal_and_date_kernels.md) |
| **8** | **Exclusive In-Place BigInt Accumulation (`ObjExclusive`), Stack-Product Fused `OpMulAdd`/`OpMulAccum`, and Compiler Non-Aliasing Analysis** | **+2.55 nats** (1.035x step, Stage 10) | **+2.77 nats** (BigInt + numeric FMA + push recovery) | **7.21x** (`bigint256_arith`), **5.21x** (`bigint64_arith`) | [OPTIMIZATION_12_exclusive_inplace_bigint_accumulation.md](OPTIMIZATION_12_exclusive_inplace_bigint_accumulation.md) |
| **9** | **Inline `int64` BigInt Execution, Compact 80-Byte `BigIntObject`, and Direct Multi-Limb Arithmetic (≤ 512 Bits)** | **+2.22 nats** (1.030x step, Stage 5) | **+3.08 nats** (3 BigInt benchmarks) | **3.76x** (`bigint32_arith`) | [OPTIMIZATION_05_fast_bigint_arithmetic.md](OPTIMIZATION_05_fast_bigint_arithmetic.md) |
| **10** | **SWAR Branchless Short-String Parsing (`OpBitOrK`/`OpSub`/`OpPlus`) and Single-Pass IEEE-754-Exact Clinger Float Parsing** | **+2.07 nats** (1.028x step, Stage 13) | **+3.05 nats** (2 string-to-number benchmarks) | **7.01x** (`string_to_float`), **3.01x** (`string_to_int`) | [OPTIMIZATION_14_swar_short_string_and_clinger_parsing.md](OPTIMIZATION_14_swar_short_string_and_clinger_parsing.md) |
| **11** | **Hybrid Ephemeron `WeakTable` (Inline Key Slot + 24-Byte 1-Pointer Table-Side Primitive Hash Table + Slot Reuse) and Weak Collection Kernels** | **+1.87 nats** (Stage 2 + Stage 11 refinement) | **+2.22 nats** (2 WeakMap benchmarks) | **4.02x** (`weak_map_delete`), **3.00x** (`weak_map_set`) | [OPTIMIZATION_06_hybrid_weaktable.md](OPTIMIZATION_06_hybrid_weaktable.md) |
| **12** | **Zero-Pack `for..in`, Cached-Slot Property Deletion, Single-Allocation Object Cloning, Holey Array Truncation, and Single-Pass Templates** | **+1.74 nats** (1.024x step, Stage 14) | **+2.33 nats** (5 object, array-hole, and template benchmarks) | **1.81x** (`array_for_in`), **1.53x** (`string_template`), **1.47x** (`array_hole_length_decr`), **1.38x** (`prop_delete`), **1.30x** (`prop_clone`) | [OPTIMIZATION_15_forin_clone_delete_holes_and_templates.md](OPTIMIZATION_15_forin_clone_delete_holes_and_templates.md) |
| **13** | **Pointer-Based `execQuiet` Dispatch, Hoisted Frame Pointer, Zero-Bounce Destructuring, and Pre-Clipped Constant Arrays** | **+1.68 nats** (1.023x step, Stage 15) | **+1.58 nats** (17 `execQuiet` benchmarks) | **1.27x** step (`int_to_string`), **1.20x** step (`string_build4`), **1.12x** step (`array_for_in`, `prop_delete`) | [OPTIMIZATION_16_pointer_quiet_dispatch_and_destructuring.md](OPTIMIZATION_16_pointer_quiet_dispatch_and_destructuring.md) |
| **14** | **Three-Instruction Compound Update Fusion (`a[i] += v`, `o.p += v`), `TypedArray` Read/Write Fusion, and Barrier-Free `SlotRef` Updates** | **+1.34 nats** (1.018x step, Stage 16) | **+1.13 nats** (array update & TypedArray read/write) | **2.35x** (`array_update`), **1.70x** (`typed_array_read`), **1.69x** (`typed_array_write`), **1.59x** (`prop_update`) | [OPTIMIZATION_17_fused_compound_updates_and_typed_arrays.md](OPTIMIZATION_17_fused_compound_updates_and_typed_arrays.md) |
| **15** | **Zero-Bounce Array Destructuring (`OpGetIterator`, `OpIterElision`, `OpIterRest`, `NewArraySlice`)** | **+1.05 nats** (1.014x step, Stage 8) | **+1.03 nats** (3 destructuring + `array_for_of`) | **1.40x** (`global_destruct`, `global_destruct_strict`), **1.39x** (`local_destruct`) | [OPTIMIZATION_13_zero_bounce_array_destructuring.md](OPTIMIZATION_13_zero_bounce_array_destructuring.md) |
| **16** | **`Number.prototype` Formatting Kernels (`shortContract`) and Single-Allocation 80-Byte `asciiText32` Strings** | Part of Stage 6A (**+1.98 nats** combined) | **+0.86 nats** (6 number-formatting benchmarks) | **1.43x** (`float_toExponential`), **1.38x** (`float_toString`) | [OPTIMIZATION_07_number_formatting_kernels.md](OPTIMIZATION_07_number_formatting_kernels.md) |
| **17** | **Loop-Free Pure-Go Schubfach Shortest-Decimal Float64 Formatting and `digitPairs` Integer Formatting** | **+0.59 nats** (1.008x step, Stage 17) | **+0.30 nats** (5 float-formatting + 2 update benchmarks) | **1.06x** step (`float_toFixed`), **1.05x** step (`float_toExponential`) | [OPTIMIZATION_18_schubfach_shortest_float_formatting.md](OPTIMIZATION_18_schubfach_shortest_float_formatting.md) |
| **18** | **Post-Compilation Object Presizing (`presizeNewObjects`) and Plain-Object Element Append Fast Paths** | Part of Stage 6A (**+1.98 nats** combined) | **+0.54 nats** (`prop_create` directly) | **1.59x** (`prop_create`), **1.58x** (`array_prop_create`) | [OPTIMIZATION_08_object_presizing_and_indexing.md](OPTIMIZATION_08_object_presizing_and_indexing.md) |

---

## Recommended Implementation Order for Reproduction

When reproducing the full 18-technique stack from baseline commit `0e377a5`, apply the specifications in dependency order:

1. **Leaf Interpreter Foundation** ([OPTIMIZATION_02_leaf_interpreter_loop.md](OPTIMIZATION_02_leaf_interpreter_loop.md)) — Establishes `execFast`, pointer-based dispatch, `validate_off.go`/`validate_on.go`, 128-byte `frame`, and `endian_le.go`.
2. **Peek-Fusion & Lean Frames** ([OPTIMIZATION_04_peek_fusion_and_lean_frames.md](OPTIMIZATION_04_peek_fusion_and_lean_frames.md)) — Adds 2-instruction peek-fusion, `OpIncUpval`/`OpDecUpval`, lean call/return, and `execFast` `OpIterNext`/`OpIterClose`.
3. **Hybrid `WeakTable`** ([OPTIMIZATION_06_hybrid_weaktable.md](OPTIMIZATION_06_hybrid_weaktable.md)) — Rebuilds `WeakTable` (including the 24-byte 1-pointer `weakPrimEntry` and `firstTable` reuse) and registers `WeakMap`/`WeakSet` kernels.
4. **Packed Numeric Arrays** ([OPTIMIZATION_03_packed_numeric_arrays.md](OPTIMIZATION_03_packed_numeric_arrays.md)) — Introduces `PackedNums`, `tryPromoteNums`, cached `ArrayCtorPlain`, single-buffer `JoinStrings`, and inlined array operations in `execFast`.
5. **Exclusive In-Place Strings** ([OPTIMIZATION_01_exclusive_inplace_strings.md](OPTIMIZATION_01_exclusive_inplace_strings.md)) — Introduces `StringBuf`, `OpPrepend`/`OpPrependK`, `execFast` 64-bit append/prepend + `SetBufLen`, `ConcatASCIIStrings`, and `CallWithArray`.
6. **Fast BigInt Arithmetic** ([OPTIMIZATION_05_fast_bigint_arithmetic.md](OPTIMIZATION_05_fast_bigint_arithmetic.md)) — Shrinks `BigIntObject` to 80 bytes, adds `BigInt64Object` fast paths in `execFast`, and implements ≤ 512-bit direct multi-limb arithmetic.
7. **Number Formatting Kernels** ([OPTIMIZATION_07_number_formatting_kernels.md](OPTIMIZATION_07_number_formatting_kernels.md)) — Extracts pure number formatters into `internal/numconv/format.go`, adds `Number.prototype` kernels, and adds single-allocation `asciiText32`.
8. **Object Presizing & Indexing** ([OPTIMIZATION_08_object_presizing_and_indexing.md](OPTIMIZATION_08_object_presizing_and_indexing.md)) — Adds `presizeNewObjects` and plain `ClassObject` `OpSetIndex`/`OpSetIndexInt` appends in `execFast`.
9. **Frameless Leaf Calls & `ProtoElemsDirty`** ([OPTIMIZATION_10_frameless_leaf_calls_and_proto_guard.md](OPTIMIZATION_10_frameless_leaf_calls_and_proto_guard.md)) — Adds `LeafKind` detection in `FuncProto.Prepare()`, frameless leaf calls and surplus-arg frames in `execFast`, `OpReturn`/`OpGetArgsIndexInt` + `OpAdd` fusion, and the single-bit `ProtoElemsDirty` guard.
10. **Zero-Bounce Array Destructuring** ([OPTIMIZATION_13_zero_bounce_array_destructuring.md](OPTIMIZATION_13_zero_bounce_array_destructuring.md)) — Adds `OpIterElision`, `OpIterRest`, `NewArraySlice`, cached `IterRef`/`NextRef`, and `execFast` `OpGetIterator`/`OpIterElision`.
11. **Dual-Mode Quiet-Kernel Loop & Table Fast Paths** ([OPTIMIZATION_09_quiet_kernel_mode_and_collections.md](OPTIMIZATION_09_quiet_kernel_mode_and_collections.md)) — Introduces `internal/engine/quiet_exec.go` (`execQuiet`), fixes the primitive `Number` `OpGetMethod`/`OpGetProp` cache miss, adds direct short-string/BigInt64 probing in `collection.Table`, and adds direct `uint64` decimal packing for small integers in `NumberToString`.
12. **Exclusive In-Place BigInt Accumulation** ([OPTIMIZATION_12_exclusive_inplace_bigint_accumulation.md](OPTIMIZATION_12_exclusive_inplace_bigint_accumulation.md)) — Adds `ObjExclusive` BigInts, `OpAccum`/`OpMulAdd`/`OpMulAccum`, compiler non-aliasing analysis (`settleAccums`), stack-product `mulAddBigInts`, and `execQuiet` BigInt accumulation while declining `push`/`pop`/`OpAppend`/`OpPrepend`.
13. **RegExp Literal & Date Kernels** ([OPTIMIZATION_11_regexp_literal_and_date_kernels.md](OPTIMIZATION_11_regexp_literal_and_date_kernels.md)) — Exposes `Regexp.WholeLiteral()`, caches `LiteralVal` on `RegExpState`, adds single-allocation `NewLiteralMatchResult`, adds `RegExpExecKernel`/`StringReplaceKernel`/`ArraySliceKernel` + `Date` kernels, replaces `datetime.modulo`/`strconv.AppendUint`/linear month scans with integer arithmetic, and adds `OpRegExpMethod`/`OpRegExpArg` to `execQuiet`.
14. **SWAR Short-String & Clinger Float Parsing** ([OPTIMIZATION_14_swar_short_string_and_clinger_parsing.md](OPTIMIZATION_14_swar_short_string_and_clinger_parsing.md)) — Replaces loop-based `shortstrings.ParseDecimal` with branchless 64-bit SWAR digit/dot parsing, adds single-pass Clinger `parseClingerFast` to `numconv.StringToNumber`, and adds inline SWAR string coercion in `execFast` (`OpBitOrK`, `OpSub`, `OpPlus`) plus `execQuiet` string arithmetic.
15. **Zero-Pack `for..in`, Cached Delete, Single-Alloc Clone, Holey Truncation, and Templates** ([OPTIMIZATION_15_forin_clone_delete_holes_and_templates.md](OPTIMIZATION_15_forin_clone_delete_holes_and_templates.md)) — Changes `Space.IndexStrings` to `[]Value`, caches `Shape.root`/`plainData` and `deletion.slot`/`configurable`, adds `CloneOrdinaryObject` and `DeletePlainNamed`, adds `o.elems` `.length` truncation to `execFast`, adds single-pass ASCII `ConcatStrings`, and wires `OpForInNext`/`OpDeleteProp*`/`OpCopyProps`/`OpConcat`/`OpSetGlobal*` into `execQuiet`.
16. **Pointer-Based `execQuiet` & Constant Array Init** ([OPTIMIZATION_16_pointer_quiet_dispatch_and_destructuring.md](OPTIMIZATION_16_pointer_quiet_dispatch_and_destructuring.md)) — Upgrades `execQuiet` to `codeBase`/`regsBase`/`ip` pointer dispatch with hoisted `curFr`, adds zero-bounce destructuring + `OpNewArrayK` + `OpGetIndex`/`OpAdd` fusion to `execQuiet`, and pre-clips `NewArrayOfConstants`.
17. **Fused Compound Updates & `TypedArray` Fast Paths** ([OPTIMIZATION_17_fused_compound_updates_and_typed_arrays.md](OPTIMIZATION_17_fused_compound_updates_and_typed_arrays.md)) — Adds 3-instruction `+=` compound update fusion (`OpGetIndexInt`/`OpGetIndex`/`OpGetProp` + `OpAdd` + store) with compiler temp-register reuse and barrier-free `SlotRef` writes, plus `TypedArray` read+add and consecutive-write fusion.
18. **Loop-Free Pure-Go Schubfach Float Formatting** ([OPTIMIZATION_18_schubfach_shortest_float_formatting.md](OPTIMIZATION_18_schubfach_shortest_float_formatting.md)) — Adds `internal/numconv/ryu.go` and `ryu_table.go` implementing loop-free Schubfach shortest-decimal formatting and `digitPairs` `appendUintFast` across `AppendNumber`, `AppendFixed`, `AppendPrecision`, and `AppendExponential`.

---

## Cumulative Stage-by-Stage Progression (Ekuma vs Baseline Ekuma)

| Stage | Applied Techniques | Cumulative Geomean Speedup (vs Baseline) | Cumulative Δnats | Step Speedup | Step Δnats |
| --- | --- | ---: | ---: | ---: | ---: |
| **Baseline** (`0e377a5`) | — | 1.000x | +0.00 | 1.000x | +0.00 |
| **Stage 1A** | Leaf `execFast` + pointer dispatch + 128B frame (`02`) | 1.098x | +6.89 | 1.098x | +6.89 |
| **Stage 1B** | Peek-fusion + `OpIncUpval` + lean call + `for..of` (`04`) | 1.160x | +11.01 | 1.057x | +4.12 |
| **Stage 2** | Hybrid `WeakTable` + weak kernels (`06`) | 1.188x | +12.74 | 1.024x | +1.73 |
| **Stage 3** | `PackedNums` + cached ctor + `JoinStrings` + `push`/`pop`/`.length=` (`03`) | 1.262x | +17.23 | 1.063x | +4.49 |
| **Stage 4** | `StringBuf` + `OpPrepend`/`OpPrependK` + `CallWithArray` + `ConcatASCIIStrings` (`01` Part A) | 1.300x | +19.39 | 1.030x | +2.15 |
| **Stage 5** | Inline `BigInt64Object` + 80B `BigIntObject` + ≤ 512-bit limb arithmetic (`05`) | 1.339x | +21.61 | 1.030x | +2.22 |
| **Stage 6A** | `Number.prototype` kernels + `asciiText32` + `presizeNewObjects` + plain-object index (`07`, `08`) | 1.375x | +23.59 | 1.027x | +1.98 |
| **Stage 6B** | 64-bit short string writes + `SetBufLen` + `ObjInterned` fix (`01` Part B) | 1.508x | +30.41 | 1.097x | +6.82 |
| **Phase 1 Final (3r)** | Phase 1 stack 3-round median verification | 1.523x | +31.15 | 1.010x | +0.74 |
| **Stage 7** | Frameless leaf calls + surplus-arg frames + `ProtoElemsDirty` (`10`) | 1.592x | +34.43 | 1.045x | +3.28 |
| **Stage 8** | Zero-bounce array destructuring: `OpIterElision`, `OpIterRest`, `NewArraySlice` (`13`) | 1.615x | +35.48 | 1.014x | +1.05 |
| **Stage 9** | `execQuiet` mode + `collection.Table` short-string/BigInt64 probing + small-int packing + Number IC fix (`09`) | 1.647x | +36.94 | 1.020x | +1.46 |
| **Stage 10** | Exclusive in-place BigInt accumulation + `OpMulAdd`/`OpMulAccum` + `execQuiet` `push`/`pop` fix (`12`) | 1.705x | +39.49 | 1.035x | +2.55 |
| **Stage 11** | RegExp literal & Date/Slice kernels + full `execQuiet` loop-glue + 24B `WeakTable` (`09`, `11`, `06`) | 1.824x | +44.49 | 1.070x | +5.01 |
| **Phase 2 Final (3r)** | Phase 2 stack + Stage 11B polish, 3-round median | 1.860x | +45.91 | 1.019x | +1.42 |
| **Stage 13** | SWAR branchless short-string parsing + single-pass Clinger float parsing (`14`) | 1.912x | +47.98 | 1.028x | +2.07 |
| **Stage 14** | Pre-packed `IndexStrings` + `for..in`/`delete`/`clone` + holey `.length` + single-pass templates (`15`) | 1.958x | +49.72 | 1.024x | +1.74 |
| **Stage 15** | Pointer-based `execQuiet` + hoisted `curFr` + zero-bounce destructuring + pre-clipped `NewArrayOfConstants` (`16`) | 2.003x | +51.40 | 1.023x | +1.68 |
| **Stage 16** | 3-instruction compound `+=` update fusion + `TypedArray` read+add & consecutive-write fusion (`17`) | 2.040x | +52.74 | 1.018x | +1.34 |
| **Stage 17** | Loop-free Schubfach shortest-float formatting + `digitPairs` `appendUintFast` + `SlotRef`/register-reuse polish (`18`, `17`) | 2.056x | +53.33 | 1.008x | +0.59 |
| **Phase 3 Final (3r)** | Full 18-technique stack, 3-round median verification (**0/74 regressions vs baseline**) | **2.062x** | **+53.56** | 1.003x | +0.23 |

---

## End-to-End Benchmark Results (Baseline vs Final)

### Microbenchmark Suite Complete Breakdown (`-suite Micro`, 3-Round Median, All 74 Benchmarks)

| Benchmark | Baseline Ekuma | Phase 1 Ekuma | Phase 2 Ekuma | Final Ekuma | Speedup (vs Baseline) |
| --- | ---: | ---: | ---: | ---: | ---: |
| `string_build3` | 89.3 µs | 11.1 µs | 11.4 µs | 11.2 µs | **7.94x** |
| `bigint256_arith` | 478.0 µs | 266.9 µs | 70.4 µs | 66.3 µs | **7.21x** |
| `string_to_float` | 92.9 ns | 86.6 ns | 89.7 ns | 13.3 ns | **7.01x** |
| `string_build_large2` | 2203 µs | 309 µs | 313 µs | 317 µs | **6.94x** |
| `bigint64_arith` | 119.6 µs | 51.5 µs | 24.9 µs | 23.0 µs | **5.21x** |
| `func_closure_call` | 147.4 ns | 105.1 ns | 33.8 ns | 34.8 ns | **4.23x** |
| `weak_map_delete` | 931.5 µs | 272.8 µs | 236.7 µs | 231.9 µs | **4.02x** |
| `bigint32_arith` | 36.7 µs | 10.6 µs | 9.97 µs | 9.76 µs | **3.76x** |
| `func_call` | 128.7 ns | 103.7 ns | 35.9 ns | 35.2 ns | **3.65x** |
| `global_func_call` | 129.5 ns | 98.3 ns | 38.5 ns | 39.0 ns | **3.32x** |
| `array_length_decr` | 45.8 µs | 14.8 µs | 14.3 µs | 14.4 µs | **3.17x** |
| `string_to_int` | 31.8 ns | 33.6 ns | 33.2 ns | 10.6 ns | **3.01x** |
| `weak_map_set` | 323.9 µs | 133.5 µs | 112.9 µs | 108.0 µs | **3.00x** |
| `array_slice` | 7.81 µs | 2.96 µs | 2.89 µs | 2.87 µs | **2.72x** |
| `string_build1` | 24.6 µs | 9.85 µs | 9.54 µs | 9.43 µs | **2.61x** |
| `string_build2` | 25.1 µs | 9.80 µs | 9.59 µs | 9.66 µs | **2.60x** |
| `int_to_string` | 104.3 ns | 115.1 ns | 49.5 ns | 41.3 ns | **2.53x** |
| `int_arith` | 12.3 µs | 5.97 µs | 5.10 µs | 5.05 µs | **2.43x** |
| `int_toString` | 158.8 ns | 169.3 ns | 67.2 ns | 65.2 ns | **2.43x** |
| `global_read` | 40.0 ns | 16.7 ns | 16.3 ns | 16.5 ns | **2.42x** |
| `string_build_large1` | 691 µs | 289 µs | 288 µs | 287 µs | **2.41x** |
| `array_for` | 1839 ns | 781 ns | 779 ns | 775 ns | **2.37x** |
| `string_build2c` | 25.0 µs | 10.2 µs | 10.5 µs | 10.6 µs | **2.36x** |
| `array_update` | 122.0 ns | 81.2 ns | 80.5 ns | 51.9 ns | **2.35x** |
| `array_pop` | 29.2 µs | 12.2 µs | 12.3 µs | 12.6 µs | **2.32x** |
| `string_build1x` | 24.5 µs | 9.31 µs | 9.55 µs | 10.8 µs | **2.27x** |
| `array_push` | 29.6 µs | 14.1 µs | 13.5 µs | 13.1 µs | **2.26x** |
| `float_arith` | 15.2 µs | 8.12 µs | 6.82 µs | 6.98 µs | **2.18x** |
| `string_concat` | 1454 ns | 680 ns | 643 ns | 695 ns | **2.09x** |
| `array_length_read` | 29.5 ns | 14.8 ns | 14.7 ns | 14.8 ns | **1.99x** |
| `array_read` | 71.8 ns | 35.6 ns | 35.7 ns | 36.3 ns | **1.98x** |
| `string_array_join` | 972 ns | 515 ns | 495 ns | 493 ns | **1.97x** |
| `regexp_utf16` | 407.2 µs | 418.4 µs | 219.3 µs | 207.9 µs | **1.96x** |
| `regexp_ascii` | 362.3 µs | 393.2 µs | 197.9 µs | 186.1 µs | **1.95x** |
| `regexp_replace` | 401.8 µs | 419.6 µs | 206.1 µs | 213.7 µs | **1.88x** |
| `array_write` | 42.0 ns | 22.1 ns | 22.4 ns | 22.4 ns | **1.88x** |
| `map_set_string` | 274.8 µs | 282.9 µs | 156.5 µs | 151.5 µs | **1.81x** |
| `array_for_in` | 3.54 µs | 4.22 µs | 4.28 µs | 1.96 µs | **1.81x** |
| `prop_read` | 36.8 ns | 18.8 ns | 18.2 ns | 20.4 ns | **1.80x** |
| `string_length` | 32.7 ns | 18.8 ns | 20.1 ns | 18.7 ns | **1.75x** |
| `empty_down_loop2` | 6.0 ns | 3.83 ns | 3.65 ns | 3.51 ns | **1.72x** |
| `typed_array_read` | 111.0 ns | 85.8 ns | 82.4 ns | 65.5 ns | **1.70x** |
| `map_delete` | 315.4 µs | 319.2 µs | 191.1 µs | 186.4 µs | **1.69x** |
| `typed_array_write` | 79.7 ns | 67.3 ns | 77.4 ns | 47.2 ns | **1.69x** |
| `arguments_strict_read` | 330.9 ns | 287.1 ns | 200.5 ns | 200.5 ns | **1.65x** |
| `prop_write` | 23.5 ns | 14.3 ns | 14.2 ns | 14.3 ns | **1.65x** |
| `arguments_read` | 329.8 ns | 288.1 ns | 199.0 ns | 203.8 ns | **1.62x** |
| `map_set_int` | 173.9 µs | 143.4 µs | 117.8 µs | 107.5 µs | **1.62x** |
| `prop_update` | 52.7 ns | 35.1 ns | 33.8 ns | 33.1 ns | **1.59x** |
| `prop_create` | 1272 ns | 802 ns | 813 ns | 802 ns | **1.59x** |
| `array_prop_create` | 31.1 µs | 21.7 µs | 19.1 µs | 19.7 µs | **1.58x** |
| `empty_down_loop` | 6.1 ns | 4.14 ns | 4.03 ns | 3.96 ns | **1.54x** |
| `global_write_strict` | 17.5 ns | 11.5 ns | 11.5 ns | 11.4 ns | **1.54x** |
| `string_template` | 522.5 ns | 544.5 ns | 540.3 ns | 340.9 ns | **1.53x** |
| `map_set_bigint` | 221.7 µs | 216.7 µs | 152.8 µs | 144.8 µs | **1.53x** |
| `array_for_of` | 1226 ns | 917 ns | 838 ns | 808 ns | **1.52x** |
| `global_write` | 17.0 ns | 11.6 ns | 11.5 ns | 11.6 ns | **1.47x** |
| `array_hole_length_decr` | 46.9 µs | 54.9 µs | 52.8 µs | 31.9 µs | **1.47x** |
| `float_toExponential` | 472.1 ns | 427.8 ns | 353.2 ns | 329.9 ns | **1.43x** |
| `math_min` | 23.8 µs | 15.9 µs | 15.7 µs | 16.7 µs | **1.43x** |
| `date_parse` | 4.42 µs | 4.53 µs | 3.08 µs | 3.11 µs | **1.42x** |
| `global_destruct` | 269.1 ns | 250.4 ns | 194.6 ns | 191.7 ns | **1.40x** |
| `global_destruct_strict` | 264.7 ns | 252.2 ns | 191.9 ns | 189.3 ns | **1.40x** |
| `empty_do_loop` | 5.5 ns | 3.99 ns | 4.00 ns | 3.97 ns | **1.39x** |
| `local_destruct` | 249.3 ns | 240.8 ns | 177.0 ns | 179.7 ns | **1.39x** |
| `float_toString` | 379.6 ns | 375.5 ns | 287.7 ns | 274.9 ns | **1.38x** |
| `prop_delete` | 1028 ns | 1131 ns | 1160 ns | 745 ns | **1.38x** |
| `empty_loop` | 5.6 ns | 4.12 ns | 4.12 ns | 4.12 ns | **1.37x** |
| `float_toFixed` | 421.4 ns | 422.9 ns | 341.6 ns | 313.2 ns | **1.35x** |
| `float_toPrecision` | 426.3 ns | 409.8 ns | 338.9 ns | 324.0 ns | **1.32x** |
| `prop_clone` | 376.7 ns | 395.9 ns | 417.4 ns | 288.9 ns | **1.30x** |
| `float_to_string` | 307.6 ns | 307.9 ns | 267.2 ns | 253.8 ns | **1.21x** |
| `date_now` | 86.4 ns | 86.1 ns | 80.4 ns | 72.1 ns | **1.20x** |
| `string_build4` | 65.6 µs | 66.3 µs | 62.7 µs | 59.2 µs | **1.11x** |
| **Geometric Mean (all 74 benchmarks)** | **1.000x** | **1.523x** | **1.860x** | **2.062x** | **2.062x (+53.56 nats)** |

### Macrobenchmark Suite (`-suite V8`, 1-Round Verification)

| Benchmark | Baseline Ekuma | Phase 1 Ekuma | Phase 2 Ekuma | Final Ekuma | Speedup (vs Baseline) |
| --- | ---: | ---: | ---: | ---: | ---: |
| `Decrypt` | 78.05 ms | 54.01 ms | 53.61 ms | 54.18 ms | **1.44x** |
| `NavierStokes` | 53.90 ms | 40.28 ms | 38.21 ms | 37.67 ms | **1.43x** |
| `Encrypt` | 4.32 ms | 3.11 ms | 2.99 ms | 3.04 ms | **1.42x** |
| `Richards` | 2.78 ms | 2.29 ms | 2.32 ms | 2.33 ms | **1.19x** |
| `Boyer` | 79.85 ms | 69.80 ms | 71.57 ms | 72.81 ms | **1.10x** |
| `Earley` | 4.79 ms | 4.62 ms | 4.58 ms | 4.38 ms | **1.09x** |
| `RegExp` | 44.45 ms | 45.31 ms | 42.07 ms | 41.19 ms | **1.08x** |
| `Splay` | 1.90 ms | 1.68 ms | 1.73 ms | 1.78 ms | **1.07x** |
| `DeltaBlue` | 5.13 ms | 4.61 ms | 4.66 ms | 4.89 ms | **1.05x** |
| `RayTrace` | 26.33 ms | 23.58 ms | 24.21 ms | 25.16 ms | **1.05x** |
| **Geometric Mean (all 10 benchmarks)** | **1.000x** | **1.182x** | **1.191x** | **1.182x** | **1.182x (+1.67 nats)** |

---

## Cross-Cutting Go No-JIT VM Engineering Principles

Six recurring architectural principles accounted for virtually the entire +53.56 nats speedup across all eighteen techniques:

1. **Dual-Mode Pointer-Based Dispatch (`execFast` + `execQuiet`)**: Any Go function call inside a hot dispatch loop forces Go's register allocator to spill loop-carried state on every instruction, so pure-VM operations must live in a strictly call-free `//go:noinline` leaf function (`execFast`). Conversely, loops whose hot body invokes quiet native kernels (`Map.set`, `RegExp.exec`, `Number.prototype.toFixed`, `Date.parse`, `for..in`, `delete`, template `OpConcat`, or multi-word BigInt accumulation) belong in a dedicated pointer-based `execQuiet` loop with a hoisted frame pointer — while **explicitly declining** operations that already have zero-call `execFast` inline fast paths (`Array.prototype.push`, `Array.prototype.pop`, `Math.min`/`Math.max`, `OpAppend`/`OpPrepend`).
2. **Compiler-Proved Exclusive In-Place Mutation (Strings and BigInts)**: Copying an accumulator on every iteration of a loop creates $O(N^2)$ or high-allocation GC pressure. Using compiler worklist dataflow (`settleAppends` and `settleAccums`) to prove that a local register has no aliasing reads before its next write allows mutating both `StringBuf` and `exclusiveBigInt` buffers in place with zero heap allocations and zero GC write barriers, sealing them (`ObjExclusive -> shared`) only when a value escapes.
3. **Eliminating Go GC Pointer Scanning, Allocations, and Write Barriers**: Every 16-byte tagged `storage.Value` carries a heap pointer word (`O *Object`). Eliminating pointers and excess allocations on hot paths — `noscan` `[]float64` arrays (`PackedNums`), 24-byte single-pointer `weakPrimEntry` entries, length-only string header updates (`SetBufLen`), barrier-skipping `SetNumber`/`SetBigInt64`/`SlotRef`, single-allocation `CloneOrdinaryObject` / `NewLiteralMatchResult` / `asciiText32` / 80-byte `BigIntObject`, and redundant-store guards in `RecordRegExpMatch2` — removes both inline write-barrier instructions and background GC scan CPU.
4. **Word-Parallel Bitwise Arithmetic and Loop-Free Numerics (SWAR, Clinger, Schubfach)**: Packed 64-bit short strings let `execFast` validate and fold up to 8 ASCII decimal digits and strip a decimal point in ~12–20 branchless instructions with zero loops; single-pass Clinger scaling handles general float strings in one exact IEEE-754 operation; and loop-free Schubfach with a 126-bit `pow10` table + `digitPairs` formats shortest float64 decimals directly into ECMAScript syntax without intermediate `strconv` buffers.
5. **Static and Epoch-Invalidated Structural Summaries**: Replacing repeated per-iteration walks with $O(1)$ invariant checks — `ProtoElemsDirty` (1 bit replacing 8 prototype pointer loads), `Shape.root`/`plainData` ($O(1)$ `takenShape`), `Shape.deletion.slot` (zero-lookup repeated `delete`), `FuncProto.LeafKind` (frameless leaf calls), `Regexp.WholeLiteral()`, `Space.IndexStrings []Value` (pre-packed `for..in` keys), and `arrayIterCache.IterRef`/`NextRef` — collapses dynamic ES2026 safety checks to a single predicted branch.
6. **Multi-Instruction Peek-Fusion Validated by White-Box Invariants**: Fusing common 2- and 3-instruction sequences (`OpInc`+`OpLoop*`, `OpGetIndex*`/`OpGetProp`+`OpAdd`, `OpGetIndex*`/`OpGetProp`+`OpAdd`+store, `OpNewObject`+`OpCopyProps`, consecutive `TypedArray` `OpSetIndexInt`) cuts bytecode dispatch and redundant bounds/tag checks by 2x–3x while `-tags ekumavalidate` (`canFuse` / `canFuse3`) mechanically enforces register-write and control-flow soundness on every step.

---

## Note on `unsafe` Footprint

Over **90%** of the total +53.56 nats gain comes from completely safe Go changes. All ten Phase 2 and Phase 3 techniques (`09`–`18`, adding **+22.41 nats** on top of Phase 1) introduce **zero** new `unsafe` primitives beyond reusing the existing `validate_off.go` pointer-dispatch helpers in `execQuiet` and the single existing embedded-header downcast pattern in `internal/storage/bigint.go`. Across the full 18-technique stack, `unsafe` remains narrowly isolated to:

- Unchecked pointer arithmetic in `internal/engine/validate_off.go` (fully disabled under `-tags ekumavalidate`).
- Length-only header updates in `Raw.SetBufLen` (`internal/storage/raw.go`, verified by a compile-time size assertion).
- Single-allocation / zero-copy `unsafe.String` conversions in `internal/storage/` (identical in mechanism to Go's standard `strings.Builder.String`).
- Embedded-header downcasting in `internal/storage/bigint.go` (matching Ekuma's existing `textObject` pattern).
- Little-endian fixed-width loads/stores in `internal/engine/endian_le.go` (which modern Go compilers also lower identically from safe `encoding/binary.LittleEndian`).

Each technique specification explicitly documents its `unsafe` footprint and pure-safe-Go alternative.

---

## Global Verification Protocol

After applying any subset (or all) of the eighteen specifications, run:

- Standard test suite: `go test ./...`
- Strict white-box validation suite (disabling unchecked pointers and enabling full instruction/cache/ownership/fusion validation): `go test -tags ekumavalidate ./...`
- Race detector across all core packages: `go test -race . ./internal/storage ./internal/collection ./internal/engine ./internal/builtin ./internal/numconv ./internal/compiler ./internal/regexp ./internal/datetime ./internal/shortstrings`
- Benchmark verification: run `_bench/cmd/jsbench` with `-suite Micro -engine '^ekuma$' -rounds 3` and `-suite V8 -engine '^ekuma$' -rounds 1`, comparing against baseline commit `0e377a5`.
