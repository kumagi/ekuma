# Optimization #4: Two-Instruction Peek-Fusion, Upvalue Step Opcodes, Lean Call/Return Frames, and Fast `for..of` Iteration

## 1. Summary & Measured Impact

- **Rank**: #4 of 8
- **Suite-Wide Impact**: **+4.12 nats** across all 74 microbenchmarks (**1.057x** step geometric-mean lift in Stage 1B)
- **Key Benchmark Improvements (Stage 1A → Stage 1B)**:

| Benchmark | Stage 1A Ekuma | Stage 1B Ekuma | Stage Speedup | Overall vs Baseline |
| --- | ---: | ---: | ---: | ---: |
| `array_for_of` (`for (var x of arr)`) | 1412 ns | 884 ns | **1.60x** (+0.47 nats) | **1.34x** |
| `empty_down_loop2` (`for (j = n; j --> 0;)`) | 5.5 ns | 3.7 ns | **1.49x** (+0.40 nats) | **1.58x** |
| `array_length_read` (`sum += tab.length`) | 20.4 ns | 14.6 ns | **1.40x** (+0.33 nats) | **2.00x** |
| `string_length` (`sum += str.length`) | 24.4 ns | 18.4 ns | **1.32x** (+0.28 nats) | **1.74x** |
| `array_read` (`sum += tab[0]`) | 49.0 ns | 37.8 ns | **1.30x** (+0.26 nats) | **2.02x** |
| `func_closure_call` (`sum++` on captured var) | 131.8 ns | 103.4 ns | **1.27x** (+0.24 nats) | **1.40x** |
| `prop_read` (`sum += obj.a`) | 23.2 ns | 18.4 ns | **1.26x** (+0.23 nats) | **1.96x** |
| `int_arith` (`sum += i * i`) | 7.71 µs | 6.16 µs | **1.25x** (+0.22 nats) | **2.06x** |
| `global_read` (`sum += global_var0`) | 20.0 ns | 17.0 ns | **1.17x** (+0.16 nats) | **2.40x** |
| `float_arith` (`sum += a * a`) | 9.68 µs | 8.25 µs | **1.17x** (+0.16 nats) | **1.87x** |
| `local_destruct` | 275 ns | 239 ns | **1.15x** (+0.14 nats) | 1.15x |
| `array_for` | 920 ns | 809 ns | **1.14x** (+0.13 nats) | **2.35x** |

---

## 2. Root Cause Analysis

1. **Indirect-jump dispatch tax in Go**: Go has no computed-goto extension, so every bytecode instruction pays a switch bounds check, jump-table lookup, indirect branch, and loop-head spill/reload. Extremely common JavaScript idioms (`i++; if (i < n) loop`, `sum += obj.a`, `sum += arr[k]`, `sum += i * i`) executed two full dispatch cycles and wrote/re-read a temporary register between the two instructions.
2. **Three-instruction upvalue increments**: Mutating a captured variable (`sum++` inside a closure) compiled to `OpGetUpval` + `OpInc` + `OpSetUpval` (plus TDZ checks for lexical bindings), paying 3 dispatches and 2 upvalue pointer dereferences per call.
3. **Unnecessary frame stores on `OpCall` / `OpReturn`**: Every fast-path JS call zeroed `fr.newTarget`, `fr.gen`, and `fr.scopes` on push (even though they were already `nil` on clean pops) and updated `ag.sp` on every single push/pop even while staying inside `execFast`.
4. **`for..of` iterator steps exited `execFast`**: `OpIterNext` and `OpIterClose` had no fast path in `execFast`, forcing every iteration of a `for..of` loop over a plain array to pay a full `execFast` → slow `exec` → `execFast` round-trip.

---

## 3. Prerequisites & Dependency Order

- **Depends on Optimization #2** (`execFast` leaf interpreter loop and `FuncProto.Prepare()` trailing spare instruction slot, which makes peeking at `ip` always in-bounds without a length check).

---

## 4. Target Files Inventory

| File | Role |
| --- | --- |
| `internal/bytecode/bytecode.go` | Declare `OpIncUpval` and `OpDecUpval`. |
| `internal/bytecode/operands.go` | Register operand descriptors for `OpIncUpval` and `OpDecUpval` (`A: OperandUpval`, `B: OperandKey` for TDZ error message). |
| `internal/compiler/expr.go` | Emit `OpIncUpval` / `OpDecUpval` for prefix/postfix statement-context `++`/`--` on mutable non-const upvalues (both `var` and `let`). |
| `internal/engine/interp.go` | Add 2-instruction peek-fusion in `execFast`; add `OpIncUpval`/`OpDecUpval` fast and slow handlers; defer `ag.sp` sync to `execFast` exit and move `newTarget`/`scopes` clearing to pop; add `OpIterNext` and `OpIterClose` fast paths in `execFast`. |
| `internal/engine/validate_on.go` | Allow fused instruction pairs in `checkStep` / `canFuse` under `-tags ekumavalidate`. |

---

## 5. Detailed Design Specification (No Code)

### 5.1 Two-Instruction Peek-Fusion in `execFast` (`internal/engine/interp.go`)

Because `ip` has already been advanced to point at the *next* instruction before the `switch i.Op` runs, peeking at `n := ip` is a single pointer dereference with zero extra bounds checks (guaranteed in-bounds by the spare trailing slot from Optimization #2). All fused patterns preserve **exact** register-level semantics (they still write the first instruction's destination register in case later code reads it, and only consume `n` when the second instruction's fast-path preconditions also hold):

1. **Loop-counter step + conditional backward loop jump**:
   - After `OpInc` (when `i.A == i.B` and operand is `NumberObject`): compute `x = v.N + 1`, store to `i.A`, then peek at `n := ip`.
     - If `n.Op` is `OpLoopLt` or `OpLoopLe` and `n.B == i.A`: load `y := *execRegAt(..., n.C)`. If `y.O == NumberObject`, compare `x < y.N` (or `<=`). If true and `!ag.InterruptFlag.Load()`, jump directly (`ip = execInstrAt(fp, codeBase, int(n.A))`) and `continue`. If false, advance `ip = execNextInstr(ip)` (consuming the non-taken loop branch) and `continue`.
     - If `n.Op` is `OpLoopLtK` or `OpLoopLeK` and `n.B == i.A`: compare `x` against `execConstAt(fp.Consts, n.C).N` in the same way.
   - After `OpDec` (when `i.A == i.B` and operand is `NumberObject`): symmetric fusion with `OpLoopGt`, `OpLoopGe`, `OpLoopGtK`, `OpLoopGeK`.
   - After `OpPostDec` (when operand is `NumberObject`): store old value `old := v.N` into `i.A` and `old - 1` into `i.B`, then fuse with `OpLoopGtK` when `n.B == i.A` (directly accelerating `for (j = n; j --> 0;)` in `empty_down_loop2`!).

2. **Value producer + numeric `OpAdd` accumulation (`sum += produced`)**:
   - After any of the following fast-path instructions produces a value `v` (or numeric value `l`/`p`) into register `dest`:
     - `OpGetGlobal` / `OpGetGlobalStrict` / `OpGetGlobalOrUndef` / `OpGetGlobalOrUndefStrict`
     - `OpGetProp` (both `CacheOwn` and `CacheProto` hits)
     - `OpGetLength` (both `KindArray` and short/ASCII `KindString`)
     - `OpGetIndex` and `OpGetIndexInt` (both boxed `[]Value` and `PackedNums` hits)
     - `OpMul` (numeric product `p`)
     - `OpIterNext` (fast array iterator step)
   - Peek at `n := ip`: if `n.Op == bytecode.OpAdd && n.C == dest && n.A == n.B` and the produced value is `NumberObject`:
     - Load `sum := execRegAt(regs, regsBase, int(n.A))`.
     - If `sum.O == storage.NumberObject`, store `sum.N + producedNum` into `sum` via `storage.SetNumber`, advance `ip = execNextInstr(ip)`, and `continue`.
   - **Edge case**: if `n.A == dest` (`x = produced; x += x`), `dest` was already updated with the produced value before reading `sum`, so `sum.N + producedNum` equals `2 * producedNum` — 100% semantically identical.

3. **Validation build compatibility (`internal/engine/validate_on.go`)**:
   - When `-tags ekumavalidate` is active, `checkStep` verifies instruction-by-instruction progress. Because fused pairs skip the intermediate `checkStep` call on `n`, ensure `checkStep` accepts advancing past fused pairs cleanly (or gate fusion cleanly with `canFuse` if your validation harness checks every PC).

### 5.2 Upvalue Step Opcodes: `OpIncUpval` and `OpDecUpval` (`internal/bytecode/`, `internal/compiler/expr.go`, `internal/engine/interp.go`)

1. Define `OpIncUpval` and `OpDecUpval`:
   - `A`: `OperandUpval` (index into `cl.Fn.Upvals`).
   - `B`: `OperandKey` (variable name key index, used only if the slow path must throw a TDZ `ReferenceError` for an uninitialized `let` binding).
   - `C`: unused (`0`).
2. In `internal/compiler/expr.go`, when compiling an update expression (`++x`, `x++`, `--x`, `x--`) whose result is discarded (`dst < 0`, i.e., statement context like `sum++;` inside a closure) and `x` resolves to a mutable non-const upvalue (`!b.isConst` and not in a `with` / direct-`eval` dynamic scope), emit `OpIncUpval` or `OpDecUpval` with the upvalue index and name key.
3. In `execFast`:
   - Load `u := execUpvalAt(cl.Fn.Upvals, i.A)` and `p := u.P`.
   - If `p.O == storage.NumberObject`: write `p.N + 1` (for `OpIncUpval`) or `p.N - 1` (for `OpDecUpval`) directly into `*p` via `storage.SetNumber(p, ...)` and `continue`! (Note: because `p.O == NumberObject`, it cannot be the TDZ sentinel `tdz`, so a single pointer equality check handles BOTH type checking and TDZ checking simultaneously!)
4. In slow `exec`:
   - Check for `p.O == tdz.O` (throw `ReferenceError` using `fp.Keys[i.B]`), then compute `*p = ag.active.step(*p, +1 or -1)`.

### 5.3 Lean `OpCall` and `OpReturn` in `execFast` (`internal/engine/interp.go`)

1. **Defer `ag.sp` synchronization**:
   - Inside `execFast`, `ag.sp` is never read by any fast-path instruction (every call computes `base` directly from the caller frame's `top()`).
   - Remove `ag.sp = ...` writes from fast-path `OpCall` and `OpReturn`.
   - Instead, synchronize `ag.sp = ag.frames[len(ag.frames)-1].top()` once upon exiting `execFast` back to slow `exec`.
2. **Eliminate redundant zeroing on frame push**:
   - Guarantee that every frame slot above `len(ag.frames)` already has `newTarget == nil`, `gen == nil`, and `scopes == nil` by ensuring slow `OpReturn`, `leaveScopes`, and exception unwinding clear them on pop, and in fast `OpReturn` guarding the clear with `if done.newTarget != nil || done.scopes != nil { done.newTarget, done.scopes = nil, nil }` (predicted not-taken on ordinary calls!).
   - Drop the unconditional `fr.newTarget = nil; fr.gen = nil; fr.scopes = nil` writes from fast `OpCall`, saving 3 pointer writes (and their write-barrier checks!) on every call.

### 5.4 Fast `OpIterNext` and `OpIterClose` in `execFast` (`internal/engine/interp.go`)

1. **Fast `OpIterNext` (`for..of` step)**:
   - Load iterator object `it := execRegAt(regs, regsBase, int(i.B)).O`.
   - Verify the iterator is a built-in array values iterator whose `next` method matches the original `%ArrayIteratorPrototype%.next` intrinsic (confirming `it.Class == ClassIterator`, checking the iterator state pointer `st, ok := it.Ext.(*storage.ArrayIterator)` with `st.Kind == IterValues`, and checking the iterator's cached `next` register matches the intrinsic).
   - Verify the target array `a := st.Target` is non-nil, `a.Kind == KindArray`, `a.Flags() == 0`, and `st.Index < uint32(a.Length())`.
   - If `st.Index < uint32(a.Raw().NumCount())` (once Technique #3 is present): read `val := storage.Number(a.Raw().Num(int(st.Index)))`, increment `st.Index`, write `val` to `i.A`, optionally fuse with following `OpAdd`, and `continue`.
   - Else if `st.Index < uint32(a.Raw().ElemCount())` and `e := a.Raw().Elem(int(st.Index)); e.O != HoleObject`: increment `st.Index`, write `e` to `i.A`, optionally fuse with following `OpAdd`, and `continue`.
   - When `st.Index >= a.Length()`: mark `st.Target = nil` (exhausted), jump to the done label `i.C` (`ip = execInstrAt(fp, codeBase, int(i.C))`), and `continue`.
   - On any exotic array, hole, modified iterator prototype, or non-array iterator, fall through to slow `exec`.
2. **Fast `OpIterClose`**:
   - When the iterator is an exhausted built-in array iterator (or a built-in array iterator with `ag.NoArrayIteratorReturn()` / prototype epoch intact so `%ArrayIteratorPrototype%.return` is known to be absent), simply `continue` without exiting `execFast`.

---

## 6. Architectural Invariants & White-Box Test Updates

- Ensure `TestExecFastPathsCallNothing/compiled` still passes with zero calls in `execFast` (including `OpIterNext` and `OpIterClose` — do not call any helper method that fails to inline!).
- Verify `TestFrameRegisterBoundary` and `TestExecutionContextBoundary` pass after deferring `ag.sp` updates and changing frame-field clearing order.
- Check `go tool objdump -s 'execFast'` to confirm that adding peek-fusion did not inflate loop-head spills.

---

## 7. Known Pitfalls & Failed Approaches to Avoid

1. **Pitfall #1 — Forgetting to keep `done.newTarget` / `done.scopes` clean on every pop path**: If any non-fast pop path (e.g., generator yield, exception unwind, `replaceFrame`, slow `OpReturn`) leaves a non-nil `newTarget`, `gen`, or `scopes` in an unused `ag.frames` slot, a subsequent fast `OpCall` that skips zeroing those fields will inherit stale pointers! Audit every `ag.frames = ag.frames[:n]` site to verify `*cur = frame{}` or explicit nil clearing runs on every pop.
2. **Pitfall #2 — Fusing `OpInc` + `OpLoopLt` without checking `ag.InterruptFlag.Load()`**: Every backward branch MUST still check `!ag.InterruptFlag.Load()` before jumping; if an interrupt is pending, fall through (do NOT advance `ip` past `OpLoopLt`) so slow `exec` runs `OpLoopLt` and polls the interrupt cleanly.
3. **Pitfall #3 — `OpIncUpval` on `const` bindings**: Never emit `OpIncUpval`/`OpDecUpval` for `const` upvalues; `const` assignment must throw a `TypeError` even if the binding holds a number.
4. **Pitfall #4 — Modifying `%ArrayIteratorPrototype%.return` after a `for..of` loop begins**: Fast `OpIterClose` must check that the prototype epoch / no-return invariant still holds; otherwise fall back to slow `exec` to invoke user-defined `return()`.

---

## 8. `unsafe` Usage & Pure-Safe-Go Alternative

- **`unsafe` used**: **Zero new `unsafe`** — 100% of this technique (peek-fusion, `OpIncUpval`/`OpDecUpval`, lean call/return, and `OpIterNext`/`OpIterClose`) is pure safe Go (using the existing `exec*At` accessors from Optimization #2).

---

## 9. Verification & Reproduction Protocol

1. Run `go test ./internal/bytecode ./internal/compiler ./internal/engine` and `go test -tags ekumavalidate ./internal/engine`.
2. Verify specifically: closure TDZ tests (`let x; f(); x = 1`), `const` upvalue mutation tests, `for..of` early `break` with custom `Array.prototype[Symbol.iterator]` / `return` hook tests, and interrupt tests on tight loops (`empty_loop`, `empty_down_loop2`).
3. Run `_bench` microbenchmarks (`-suite Micro -engine '^ekuma$' -rounds 1`) and confirm: `array_for_of` ≥ 1.45x faster than Stage 1A, `empty_down_loop2` ≥ 1.4x faster, `func_closure_call` ≥ 1.2x faster, and overall suite lift of ~1.05x–1.06x (+3.5 to +4.5 nats).
