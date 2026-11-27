# Optimization #2: Leaf Fast-Path Interpreter Extraction (`execFast`), Pointer-Based Dispatch, Unchecked Accessors, and 128-Byte Frame Alignment

## 1. Summary & Measured Impact

- **Rank**: #2 of 8 (and the foundational prerequisite for Techniques #1, #3, #4, #5, and #8)
- **Suite-Wide Impact**: **+6.89 nats** across all 74 microbenchmarks (**1.098x** geometric-mean speedup on its own)
- **Targeted Subset Impact**: **+7.18 nats** across the 26 hot microbenchmarks that stay inside `execFast`
- **Key Benchmark Improvements (Baseline → Stage 1A)**:

| Benchmark | Baseline Ekuma | Stage 1A Ekuma | Speedup |
| --- | ---: | ---: | ---: |
| `global_read` | 40.0 ns | 20.0 ns | **2.00x** (+0.69 nats) |
| `array_for` | 1839 ns | 920 ns | **2.00x** (+0.69 nats) |
| `prop_write` | 23.5 ns | 14.6 ns | **1.61x** (+0.48 nats) |
| `int_arith` | 12.29 µs | 7.71 µs | **1.59x** (+0.47 nats) |
| `prop_read` | 36.8 ns | 23.2 ns | **1.59x** (+0.46 nats) |
| `float_arith` | 15.20 µs | 9.68 µs | **1.57x** (+0.45 nats) |
| `prop_update` | 52.7 ns | 35.0 ns | **1.51x** (+0.41 nats) |
| `array_read` | 71.8 ns | 49.0 ns | **1.46x** (+0.38 nats) |
| `global_write_strict` | 17.5 ns | 12.0 ns | **1.46x** (+0.38 nats) |
| `math_min` | 23.81 µs | 16.31 µs | **1.46x** (+0.38 nats) |
| `array_length_read` | 29.5 ns | 20.4 ns | **1.45x** (+0.37 nats) |
| `array_update` | 122 ns | 85.6 ns | **1.43x** (+0.35 nats) |
| `empty_down_loop` | 6.1 ns | 4.3 ns | **1.42x** (+0.35 nats) |
| `empty_loop` | 5.6 ns | 4.1 ns | **1.38x** (+0.32 nats) |
| `array_write` | 42.0 ns | 30.9 ns | **1.36x** (+0.31 nats) |
| `string_length` | 32.7 ns | 24.4 ns | **1.34x** (+0.30 nats) |
| `typed_array_read` | 111 ns | 88.9 ns | **1.25x** (+0.22 nats) |

---

## 2. Root Cause Analysis

In baseline Ekuma (`internal/engine/interp.go`), the fast-path `switch` and the slow-path `switch` (which invokes dozens of runtime helpers, throws exceptions, grows stacks, and reloads slices) lived inside the same `(*agent).exec` function:

1. **Function-wide register spilling**: Go's register allocator preserves live values across `CALL` sites in the same function by spilling them to the stack frame. Even though the fast-path `switch` made zero calls (enforced by `TestExecFastPathsCallNothing`), the presence of calls in the slow-path `switch` forced Go to spill `ag`, `fp`, `pc`, `regs` (data + len + cap), and `stop` at the loop head on every single bytecode instruction.
2. **Repeated slice bounds checks and base-pointer reloads**: Every instruction indexed `fp.Code[pc]`, `regs[i.A]`, `regs[i.B]`, `regs[i.C]`, `fp.Consts[...]`, `fp.Caches[...]`, `fp.Globals[...]`, `cl.Fn.Upvals[...]`, and `ag.frames[len(ag.frames)-1]`, each emitting a slice length load + compare + conditional branch to `runtime.panicIndex`.
3. **Non-power-of-two `frame` struct size**: Indexing `ag.frames[n]` required an integer multiply (`IMULQ`) instead of a single bit-shift (`SHLQ`).
4. **Slow short-string `.length` and TypedArray endianness**: Packed short ASCII strings lacked `ObjASCIIKnown|ObjASCII` on their shared sentinel objects, causing `OpGetLength` on short strings (`"abcde"`) to exit the fast path. TypedArray reads/writes called `encoding/binary.LittleEndian` methods through slice expressions with extra bounds checks.

---

## 3. Prerequisites & Dependency Order

- **None** — this is the first technique to apply on top of baseline commit `0e377a5`.

---

## 4. Target Files Inventory

| File | Role |
| --- | --- |
| `internal/engine/interp.go` | Extract `//go:noinline (*agent).execFast`; pad `frame` to 128 bytes; switch fast-path loop to `ip`, `codeBase`, `regsBase` and `exec*At` helpers; add short-ASCII string check in `OpGetLength`; use `readU*LE`/`writeU*LE` in TypedArray fast paths. |
| `internal/engine/validate_off.go` | Implement unchecked pointer-arithmetic `exec*At` / `execInstrAt` / `execNextInstr` / `execPC` / `execStackPtr` helpers for standard (`!ekumavalidate`) builds. |
| `internal/engine/validate_on.go` | Implement bounds-checked slice-indexing counterparts for `-tags ekumavalidate` builds. |
| `internal/storage/funcproto.go` | Ensure `FuncProto.Prepare()` reserves at least 1 spare instruction slot (`cap(p.Code) >= len(p.Code) + 1`) so advancing `ip` past the final instruction stays inside the same Go allocation. |
| `internal/engine/endian_le.go` (new) | Build-tagged (`amd64 || arm64 || 386 || arm || loong64 || riscv64 || wasm`) direct little-endian 16/32/64-bit load/store helpers. |
| `internal/engine/endian_generic.go` (new) | Fallback (`!(amd64 || ...)` ) helpers using `encoding/binary.LittleEndian`. |
| `internal/engine/fastpath_test.go` | Point AST/ELF verification at `execFast` and allowlist the new inlineable helpers. |
| `internal/engine/context_boundary_test.go`, `internal/engine/rawaccess_test.go`, `internal/engine/protowalk_test.go`, `internal/engine/testdata/rawusers.txt` | Reattribute fast-path call counts from `agent.exec` to `agent.execFast`. |

---

## 5. Detailed Design Specification (No Code)

### 5.1 Spare Trailing Instruction Slot (`internal/storage/funcproto.go`)

1. In `(*FuncProto).Prepare()`, when initializing caches for a `FuncProto`, check whether `cap(p.Code) == len(p.Code)`.
2. If so, allocate a new `[]bytecode.Instr` of length `len(p.Code)` and capacity `len(p.Code) + 1`, copy `p.Code` into it, and store it back into `p.Code`.
3. **Why this is mandatory for Go GC safety**: Advancing `ip` to `execNextInstr(ip)` immediately after loading `*ip` produces a pointer one slot past the last live instruction (`&p.Code[len(p.Code)]`). Without 1 spare element inside the backing array, that pointer could point to the start of an unrelated heap object in the next span slot, violating Go's pointer safety rules.

### 5.2 Pad `frame` to 128 Bytes (`internal/engine/interp.go`)

1. Inspect `unsafe.Sizeof(frame{})` on 64-bit platforms (baseline is ~112–120 bytes).
2. Add an unnamed byte-array padding field at the end of `type frame struct` so that its total size is exactly 128 bytes on 64-bit builds, and guard it with a compile-time assertion (when `unsafe.Sizeof(uintptr(0)) == 8`, assert `unsafe.Sizeof(frame{}) == 128`).
3. Indexing `ag.frames[n]` now compiles to `SHLQ $7` instead of `IMULQ`.

### 5.3 Dual-Mode Accessor Helpers (`internal/engine/validate_off.go`, `internal/engine/validate_on.go`)

Define the exact same set of 10 tiny, inlineable helper functions in both files:

1. `execInstrAt(fp *storage.FuncProto, base unsafe.Pointer, pc int) *bytecode.Instr`:
   - `validate_off`: returns `(*bytecode.Instr)(unsafe.Add(base, uintptr(pc)*unsafe.Sizeof(bytecode.Instr{})))`.
   - `validate_on`: returns `&fp.Code[pc]`.
2. `execNextInstr(ip *bytecode.Instr) *bytecode.Instr`:
   - Both: advances `ip` by `unsafe.Sizeof(bytecode.Instr{})` via `unsafe.Add` (safe because of the spare slot in `Prepare`).
3. `execPC(base unsafe.Pointer, ip *bytecode.Instr) int`:
   - Both: returns `int((uintptr(unsafe.Pointer(ip)) - uintptr(base)) / unsafe.Sizeof(bytecode.Instr{}))`.
4. `execRegAt(regs []storage.Value, base unsafe.Pointer, idx int) *storage.Value`:
   - `validate_off`: `unsafe.Add(base, uintptr(idx)*unsafe.Sizeof(storage.Value{}))`.
   - `validate_on`: `&regs[idx]`.
5. `execConstAt(consts []storage.Value, idx int32) *storage.Value`:
   - `validate_off`: `unsafe.Add` on `unsafe.SliceData(consts)`.
   - `validate_on`: `&consts[idx]`.
6. `execCacheAt(caches []storage.PropCache, idx int32) *storage.PropCache`:
   - `validate_off`: `unsafe.Add` on `unsafe.SliceData(caches)`.
   - `validate_on`: `&caches[idx]`.
7. `execGlobalAt(globals []storage.GlobalCache, idx int32) *storage.GlobalCache`:
   - `validate_off`: `unsafe.Add` on `unsafe.SliceData(globals)`.
   - `validate_on`: `&globals[idx]`.
8. `execUpvalAt(upvals []*storage.Upvalue, idx int32) *storage.Upvalue`:
   - `validate_off`: `unsafe.Add` on `unsafe.SliceData(upvals)`.
   - `validate_on`: `upvals[idx]`.
9. `execFrameAt(frames []frame, idx int) *frame`:
   - `validate_off`: `unsafe.Add` on `unsafe.SliceData(frames)`.
   - `validate_on`: `&frames[idx]`.
10. `execStackPtr(stack []storage.Value, idx int) unsafe.Pointer`:
    - `validate_off`: `unsafe.Add` on `unsafe.SliceData(stack)`.
    - `validate_on`: `unsafe.Pointer(&stack[idx])`.

### 5.4 Extract `//go:noinline (*agent).execFast` (`internal/engine/interp.go`)

1. Give `execFast` the signature `(ag *agent) execFast(stop int, fp *storage.FuncProto, pc int, regs []storage.Value) (*storage.FuncProto, int)` and annotate it with `//go:noinline`.
   - **Why `//go:noinline` is critical**: If Go inlined `execFast` back into `exec`, all the slow-path `CALL`s in `exec` would once again pollute `execFast`'s register allocation.
2. On entry to `execFast`:
   - Compute `codeBase := unsafe.Pointer(unsafe.SliceData(fp.Code))`, `regsBase := unsafe.Pointer(unsafe.SliceData(regs))`, and `ip := execInstrAt(fp, codeBase, pc)`.
3. Inside the infinite `for` loop:
   - Under `if validateExec`, call `ag.checkStep(fp, execPC(codeBase, ip), slices.Clip(regs))`.
   - Load `i := ip` and advance `ip = execNextInstr(ip)`.
   - Dispatch `switch i.Op` covering all call-free fast-path cases originally in `exec`'s first switch, rewriting every slice index to its corresponding `exec*At` helper.
   - Whenever an `OpCall` or `OpReturn` changes frames inside `execFast`, refresh `fp`, `codeBase`, `ip`, `regsBase` (and `regs` when `validateExec` is true).
   - For backward-jump interrupt checks (`OpJump`), compare `uintptr(unsafe.Pointer(target)) >= uintptr(unsafe.Pointer(ip))` instead of computing `pc` via division.
   - When any case falls out of the `switch` (not handled by a fast path), **return `(fp, execPC(codeBase, i))`** (note: returning the PC of `i`, *before* advancing `ip`, so slow `exec` fetches `i` cleanly!).
4. In `(*agent).exec(stop int)`:
   - At the top of the loop, call `fp, pc = ag.execFast(stop, fp, pc, regs)`.
   - Because `execFast` may have pushed or popped frames before returning, recompute `regs = ag.frames[len(ag.frames)-1].registers(ag.stack)` and `ag.sp = ag.frames[len(ag.frames)-1].top()` when returning to slow `exec`.
   - Fetch `i := fp.Code[pc]; pc++` and execute the slow-path `switch` unchanged.

### 5.5 Short ASCII String `.length` Fast Path (`internal/engine/interp.go`)

1. In `execFast`'s `OpGetLength` case, when `o.Kind == storage.KindString`:
   - Accept either `o.Flags()&(ObjASCIIKnown|ObjASCII) == ObjASCIIKnown|ObjASCII` **or** (`len(o.Text()) <= shortstrings.MaxBytes && shortstrings.IsASCII(math.Float64bits(v.N))`).
   - Store `float64(len(o.Text()))` into register `a` via `storage.SetNumber` and `continue`.

### 5.6 Endian Helpers (`internal/engine/endian_le.go`, `internal/engine/endian_generic.go`)

1. Provide `readU16LE`, `readU32LE`, `readU64LE`, `writeU16LE`, `writeU32LE`, `writeU64LE` operating on `[]byte`.
2. In `endian_le.go`, perform a single bounds-check hint (`_ = b[1]`, `_ = b[3]`, `_ = b[7]`) followed by a typed pointer load/store at `&b[0]`.
3. Replace the `encoding/binary.LittleEndian` calls in `execFast`'s TypedArray `OpGetIndex`/`OpSetIndex`/`OpGetIndexInt`/`OpSetIndexInt` fast paths with these helpers.

---

## 6. Architectural Invariants & White-Box Test Updates

- **`internal/engine/fastpath_test.go`**:
  - Update `TestExecFastPathsCallNothing/source` to inspect the AST of `execFast` (instead of the first switch of `exec`).
  - Update `TestExecFastPathsCallNothing/compiled` to disassemble `(*agent).execFast` and verify zero `CALL` instructions (other than `runtime.gcWriteBarrier*` and unreachable panic stubs).
  - Add `uintptr`, `unsafe.Pointer`, `unsafe.SliceData`, `execInstrAt`, `execNextInstr`, `execPC`, `execRegAt`, `execConstAt`, `execCacheAt`, `execGlobalAt`, `execUpvalAt`, `execFrameAt`, `execStackPtr`, `readU16LE..writeU64LE`, and `shortstrings.IsASCII` to `fastPathCalls`.
- **`internal/engine/rawaccess_test.go`**, **`internal/engine/protowalk_test.go`**, **`internal/engine/context_boundary_test.go`**, **`internal/engine/testdata/rawusers.txt`**:
  - Move the fast-path call counts previously attributed to `agent.exec` onto `agent.execFast`.

---

## 7. Known Pitfalls & Failed Approaches to Avoid

1. **Pitfall #1 — `if validateExec && cond` fails `TestExecFastPathsCallNothing/source`**: The source AST test skips `if` statements only when the condition is the bare identifier `validateExec`. Never combine `validateExec` with `&&`; always nest the extra condition inside `if validateExec { if cond { ... } }`.
2. **Pitfall #2 — Temporary regression on slow-path-heavy benchmarks**: Extracting `execFast` introduces one function call + epilogue boundary whenever an instruction misses `execFast`. In our Stage 1A measurement, benchmarks whose inner loop exited `execFast` on every iteration regressed temporarily by 12–19% (`bigint32_arith`, `string_build1`, `array_for_of`, `int_toString`, `float_toPrecision`). This is expected and completely reversed (turning into massive wins) once those hot operations are brought into `execFast` or `shortContract` kernels in Techniques #1, #4, #5, and #7.
3. **Pitfall #3 — Forgetting to reload `regs` after `execFast` returns**: Because `execFast` handles `OpCall` and `OpReturn` internally, the active frame upon returning to `exec` may differ from the frame passed into `execFast`. Always refresh `regs` (and `ag.sp`) from the current top frame in `ag.frames` upon returning from `execFast`.
4. **Pitfall #4 — Loop-head register spills**: Check `go tool objdump -s 'execFast'` after every edit to `execFast`. Verify the main dispatch block (`JMP 0(R10)(R8*8)`) has at most 1–2 stack spills (`codeBase` / `fp`), not 5+.

---

## 8. `unsafe` Usage & Pure-Safe-Go Alternative

- **`unsafe` used**:
  1. Pointer arithmetic in `internal/engine/validate_off.go` (`unsafe.Add`, `unsafe.SliceData`, `unsafe.Pointer`, `unsafe.Sizeof`) and `ip` pointer comparison in `OpJump`.
  2. Direct little-endian pointer casts in `internal/engine/endian_le.go`.
- **Safe-Go alternative**:
  - Using `validate_on.go`'s slice-indexing helpers for all builds and `encoding/binary.LittleEndian` everywhere removes 100% of `unsafe` from this technique while keeping the `//go:noinline execFast` leaf-loop extraction and 128-byte `frame` padding intact. The pure-safe-Go variant retains an estimated **~60% (+4 nats)** of Stage 1A's gain; the remaining ~2–3 nats come from eliminating per-operand slice bounds checks and advancing `ip` as a pointer.

---

## 9. Verification & Reproduction Protocol

1. Run `go test ./internal/engine ./internal/storage` and `go test -tags ekumavalidate ./internal/engine`.
2. Build a test binary (`go test -c -o /tmp/engine.test ./internal/engine`) and inspect the first 50 lines of `go tool objdump -s 'execFast' /tmp/engine.test`:
   - Confirm `execFast` is a standalone symbol with `LEAQ 0x10(DI), R9` instruction advance and a jump-table dispatch (`JMP 0(R10)(R8*8)`).
   - Confirm at most 1–2 `MOVQ ..., ...(SP)` spills at the dispatch head.
3. Run `_bench` microbenchmarks (`-suite Micro -engine '^ekuma$' -rounds 1`): confirm suite-wide geometric mean improves by ~1.09x–1.10x, with `global_read` and `array_for` near ~2.0x faster and `prop_read`/`prop_write`/`int_arith`/`float_arith` ~1.5x–1.6x faster.
