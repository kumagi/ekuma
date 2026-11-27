# Optimization #16: Pointer-Based `execQuiet` Dispatch, Hoisted Frame Pointer, Zero-Bounce Destructuring, and Pre-Clipped Constant Arrays

## 1. Summary & Measured Impact

- **Overall Rank**: #13 of 18 (Stage 15)
- **Suite-Wide Impact**: **+1.68 nats** across all 74 microbenchmarks (**1.023x** step geometric-mean lift in Stage 15 — crossing the **2.00x** cumulative baseline threshold)
- **Targeted Subset Impact**: **+1.58 nats** across 17 `execQuiet`-resident benchmarks (number formatting, collections, `for..in`, deletion, RegExp, buffered string building, and multi-word BigInt)
- **Key Benchmark Improvements (Stage 14 → Stage 15 → Final 3-Round Median)**:

| Benchmark | Baseline Ekuma | Stage 14 Ekuma | Stage 15 Ekuma | Final Ekuma (3r) | Stage 15 Speedup | Overall vs Baseline |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| `int_to_string` | 104.3 ns | 51.8 ns | 40.8 ns | 41.3 ns | **1.27x** (+0.24 nats) | **2.53x** |
| `int_toString` | 158.8 ns | 69.5 ns | 56.4 ns | 65.2 ns | **1.23x** (+0.21 nats) | **2.43x** |
| `string_build4` | 65.6 µs | 75.2 µs | 62.7 µs | 59.2 µs | **1.20x** (+0.18 nats) | **1.11x** |
| `array_for_in` | 3.54 µs | 2.21 µs | 1.97 µs | 1.96 µs | **1.12x** (+0.11 nats) | **1.81x** |
| `prop_delete` | 1.03 µs | 787 ns | 704 ns | 745 ns | **1.12x** (+0.11 nats) | **1.38x** |
| `float_to_string` | 307.6 ns | 283.1 ns | 257.0 ns | 253.8 ns | **1.10x** (+0.10 nats) | **1.21x** |
| `float_toPrecision` / `float_toExponential` | 426 / 472 ns | 343 / 379 ns | 317 / 352 ns | 324 / 330 ns | **1.08x** (+0.08 each) | **1.32x / 1.43x** |
| `bigint256_arith` | 478.0 µs | 73.8 µs | 68.4 µs | 66.3 µs | **1.08x** (+0.08 nats) | **7.21x** |
| `map_set_int` / `map_set_string` / `map_set_bigint` | 174 / 275 / 222 µs | 119 / 158 / 151 µs | 112 / 150 / 144 µs | 107 / 152 / 145 µs | **1.05x–1.07x** | **1.62x / 1.81x / 1.53x** |

---

## 2. Root Cause Analysis

As Stages 9–14 moved more than twenty benchmarks into `execQuiet`, CPU profiles showed `execQuiet`'s own flat dispatch loop consuming 17%–38% of total CPU time (`map_set_int` 37.5%, `float_toFixed` 22.6%, `float_to_string` 16.7%):

1. **Bounds-checked slice indexing on every operand**: Unlike `execFast` (which used `codeBase`/`regsBase`/`ip` and `execRegAt`/`execConstAt`/`execCacheAt`/`execGlobalAt`), `execQuiet` still used bounds-checked Go slice indexing (`code[pc]`, `regs[i.A]`, `regs[i.B]`, `regs[i.C]`, `fp.Consts[i.C]`, `fp.Caches[i.C]`, `fp.Globals[i.C]`) on every single instruction and operand.
2. **Repeated `ag.frames[len(ag.frames)-1].pc = pc` reloads**: `execQuiet` never calls user JavaScript and never pushes or pops call frames, yet every kernel call, string concatenation, and BigInt op re-indexed `ag.frames[len(ag.frames)-1]` with a slice length load and bounds check.
3. **Destructuring `OpIterRest` still bounced out of `execFast`**: While Optimization #13 moved `OpGetIterator`, `OpIterElision`, and `OpIterClose` into `execFast`, `OpIterRest` (`...rest`) allocates an array and had to exit to slow `exec` on every iteration of `local_destruct` / `global_destruct`.
4. **Redundant `InitElems` + `slices.Clip` on every constant array literal**: `NewArrayOfConstants` clipped `t.elems` and invoked `a.InitElems(...)` -> `ownTail()` on every allocation, even though `PropertyObject` embeds its own `objectTail`.

---

## 3. Prerequisites & Dependency Order

- **Depends on Optimization #2** (`validate_off.go` / `validate_on.go` pointer-dispatch helpers), **Optimization #9** (`execQuiet`), **Optimization #13** (array destructuring opcodes), and **Optimization #15** (`execQuiet` `OpForInNext` / `OpSetGlobal*`).

---

## 4. Target Files Inventory

| File | Role |
| --- | --- |
| `internal/engine/quiet_exec.go` | Convert `execQuiet` to pointer-based `codeBase`/`regsBase`/`ip` dispatch using `execInstrAt`/`execNextInstr`/`execPC`/`execRegAt`/`execConstAt`/`execCacheAt`/`execGlobalAt`/`execFrameAt`; hoist `curFr`; add `OpRequireCoercible`, `OpNewArrayK`, `OpGetIterator`, `OpIterNext` (with `OpAdd` fusion), `OpIterElision`, `OpIterRest`, `OpIterClose`, and `OpGetIndex`+`OpAdd` peek-fusion. |
| `internal/engine/interp.go` | Enter `execQuiet` from slow `exec`'s `OpIterRest` plain-array `NewArraySlice` fast path when `!fp.HeapRegs`. |
| `internal/storage/array.go` | Pre-clip `t.elems` once via `t.clipElems()` on template initialization in `NewArrayOfConstants`, and initialize `PropertyObject`'s embedded `objectTail{elems: t.elems}` directly in the struct literal. |

---

## 5. Detailed Design Specification (No Code)

### 5.1 Pointer-Based `execQuiet` Dispatch & Hoisted Frame Pointer (`internal/engine/quiet_exec.go`)

1. At the start of `execQuiet`, compute once outside the loop:
   - `codeBase := unsafe.Pointer(unsafe.SliceData(fp.Code))`
   - `regsBase := unsafe.Pointer(unsafe.SliceData(regs))`
   - `curFr := execFrameAt(ag.frames, len(ag.frames)-1)`
   - `ip := execInstrAt(fp, codeBase, pc)`
2. In the dispatch loop:
   - Under `validateExec`, call `ag.checkStep(fp, execPC(codeBase, ip), regs)`.
   - Fetch `i := ip; ip = execNextInstr(ip)`.
   - Replace every `regs[...]`, `fp.Consts[...]`, `fp.Caches[...]`, and `fp.Globals[...]` access with `execRegAt`, `execConstAt`, `execCacheAt`, and `execGlobalAt`. Under `-tags ekumavalidate`, `validate_on.go` maps every helper back to full bounds-checked Go slice indexing.
   - Replace every `ag.frames[len(ag.frames)-1].pc = pc` with `curFr.pc = execPC(codeBase, ip)`.
   - Replace every `return pc - 1` decline with `return execPC(codeBase, i)` (keeping the `if validateExec && !fused { ag.enterChecks() }` guard intact on fused `OpGetMethod`+`OpCall` declines).
   - For `OpJump`, compute `target := execInstrAt(fp, codeBase, int(i.A))` and check `uintptr(unsafe.Pointer(target)) < uintptr(unsafe.Pointer(ip)) && ag.InterruptFlag.Load()` before assigning `ip = target`.
   - For `OpGetIndex`, add `OpAdd` peek-fusion (`n := ip; n.Op == OpAdd && n.C == i.A && n.A == n.B && e.O == NumberObject`), saving an extra dispatch on `sum += r[i]` inside `for..in` loops.

### 5.2 Zero-Bounce Destructuring & Constant Array Literals in `execQuiet` (`quiet_exec.go`, `interp.go`)

1. Add the plain-array destructuring instruction family to `execQuiet`:
   - **`OpRequireCoercible`**: succeed immediately when `regs[i.A].O != nil`, otherwise decline to throw `TypeError`.
   - **`OpGetIterator`**: when `i.C == bytecode.IterSync` and `v.O` is a `KindArray` satisfying `cx.ArrayIterationPlain(v.O)`, store `Number(0)` and `v` directly.
   - **`OpIterNext`**: handle finished (`it.O == nil`) and plain numeric cursor (`it.O == NumberObject`) via `o.ElementAt(j)` (covering both `PackedNums` and dense `elems`), with `OpAdd` peek-fusion for numeric elements.
   - **`OpIterElision`**: advance plain numeric cursors when `o.ElementAt(j)` succeeds, or clear on `j >= o.Length()`.
   - **`OpIterRest`**: check `InterruptFlag` first, handle finished iterators (`NewArray(nil)`), and handle plain numeric cursors via `cx.NewArraySlice(o, j)` (recording `curFr.pc` if `o.Length() > j + PollChunk`), clearing iterator registers on success.
   - **`OpIterClose`**: no-op when `regs[i.A].O == nil`.
   - **`OpNewArrayK`**: invoke `cx.NewArrayOfConstants(fp.Consts[i.B : i.B+i.C])` when already in `execQuiet` (do **not** enter `execQuiet` from standalone `OpNewArrayK` in slow `exec`, preserving `execFast` residency for loops that only allocate array literals).
2. In slow `exec`'s `OpIterRest`, after `NewArraySlice` succeeds, enter `execQuiet` when `!fp.HeapRegs`.

### 5.3 Pre-Clipped Direct Struct Init in `NewArrayOfConstants` (`internal/storage/array.go`)

1. On first execution of an array literal's constant run, create template `t = r.NewArrayCopy(run)` and call `t.clipElems()` once so `cap(t.elems) == len(t.elems)` permanently.
2. On every call, construct `(&PropertyObject{Object: ..., objectTail: objectTail{elems: t.elems}}).Init()` directly — eliminating `slices.Clip`, `InitElems`, `ownTail()`, and an extra write barrier on every constant array literal.

---

## 6. Architectural Invariants & White-Box Test Updates

- **`TestValueStorageOwnership`**: In `NewArrayOfConstants`, always call `t.clipElems()` rather than assigning `t.elems = ...` directly (which triggers `TestValueStorageOwnership`) or calling `t.InitElems(...)` on an already-initialized array (which panics under `-tags ekumavalidate`).
- **`TestRawStorage`**: No new raw storage entries are required; `execQuiet`'s raw field access counts remain unchanged.
- **`execQuiet` Exclusion Invariant**: `OpGetMethod`/`OpCall` in `execQuiet` must continue to decline `Array.prototype.push`, `Array.prototype.pop`, and `Extremum != ExtremumNone` so those stay on `execFast`.

---

## 7. Known Pitfalls & Failed Approaches to Avoid

1. **Pitfall #1 — Calling `t.InitElems(...)` on an existing template array**: `NewArrayCopy` already initializes `t.elems`, so calling `t.InitElems(slices.Clip(t.elems))` panics with `storage: InitElems on an object with elements` under `-tags ekumavalidate`. Use `t.clipElems()`, which is the canonical guarded method for clipping existing element storage.
2. **Pitfall #2 — Entering `execQuiet` unconditionally on `OpNewArrayK` from slow `exec`**: Loops whose only non-leaf instruction is an array literal followed by hot `execFast` arithmetic would be pulled out of `execFast` unnecessarily; keep `OpNewArrayK` inside `execQuiet` only when already executing in quiet mode.

---

## 8. `unsafe` Usage & Pure-Safe-Go Alternative

- **`unsafe` used**: Reuses the existing `validate_off.go` pointer helpers (`execInstrAt`, `execNextInstr`, `execPC`, `execRegAt`, `execConstAt`, `execCacheAt`, `execGlobalAt`, `execFrameAt`), which are automatically replaced by 100% bounds-checked Go slices under `-tags ekumavalidate`.
- **Pure-safe-Go alternative**: Retaining `curFr` hoisting, `OpGetIndex`+`OpAdd` fusion, destructuring ops in `execQuiet`, and `NewArrayOfConstants` requires zero `unsafe`; pointer-based dispatch accounts for ~75% of Stage 15's gain by eliminating repeated slice bounds checks across all `execQuiet` instructions.

---

## 9. Verification & Reproduction Protocol

1. Run `go test -run 'TestExecFastPathsCallNothing|TestRawStorage|TestUnguardedOperations|TestValueStorageOwnership|TestSharedTailWrites' ./internal/engine`.
2. Run `go test -tags ekumavalidate ./internal/engine`.
3. Run `_bench` microbenchmarks (`-suite Micro -engine '^ekuma$' -rounds 1`) and verify broad 5%–27% improvements across `int_to*`, `float_to*`, `map_*`, `weak_map_set`, `array_for_in`, `prop_delete`, `string_build4`, and `bigint256_arith`.
