# Optimization #10: Frameless Leaf Function Calls, Surplus-Argument Inline Frames, `OpReturn`/`OpGetArgsIndexInt` Fusion, and Cached `ProtoElemsDirty` Guard

## 1. Summary & Measured Impact

- **Overall Rank**: #6 of 13
- **Suite-Wide Impact**: **+3.28 nats** across all 74 microbenchmarks (**1.045x** step geometric-mean lift in Stage 7)
- **Targeted Subset Impact**: **+3.80 nats** across function-call, `arguments`, and `array_push` benchmarks
- **Key Benchmark Improvements (Phase 1 → Stage 7 → Final 3-Round Median)**:

| Benchmark | Phase 1 Ekuma | Stage 7 Ekuma | Final Ekuma | Stage 7 Speedup | Overall vs Baseline |
| --- | ---: | ---: | ---: | ---: | ---: |
| `func_call` (tiny local function call in loop) | 103.7 ns | 34.3 ns | 35.9 ns | **3.02x** (+1.11 nats) | **3.59x** |
| `func_closure_call` (closure incrementing upvalue) | 105.1 ns | 34.9 ns | 33.8 ns | **3.01x** (+1.10 nats) | **4.36x** |
| `global_func_call` (global leaf function call) | 98.3 ns | 38.6 ns | 38.5 ns | **2.54x** (+0.93 nats) | **3.36x** |
| `arguments_read` (sloppy `arguments[i]` with surplus args) | 288.1 ns | 215.3 ns | 199.0 ns | **1.34x** (+0.29 nats) | **1.66x** |
| `arguments_strict_read` (strict `arguments[i]`) | 287.1 ns | 217.6 ns | 200.5 ns | **1.32x** (+0.28 nats) | **1.65x** |
| `array_push` (dense `PackedNums` push loop) | 14.1 µs | 12.9 µs | 13.5 µs | **1.10x** (+0.09 nats) | **2.20x** |

---

## 2. Root Cause Analysis

Three distinct overheads dominated function calls, variadic `arguments` reads, and array writes after Phase 1:

1. **Full frame push/pop for trivial leaf functions**: Even after Phase 1's lean frames, calling a 1-instruction leaf function (`function f(a) { return 1; }`, `function id(a) { return a; }`, or a closure `function() { x++; }`) still wrote a 128-byte `frame` entry, switched `fp`/`codeBase`/`regsBase`/`ip`, initialized local registers, executed the body, restored the caller's `frame` and pointers, and zeroed/popped the frame — costing ~70 ns per call even when the body itself did 1 ns of work.
2. **Surplus-argument calls exited `execFast`**: When a function referencing `arguments` (`c.NeedsExtra == true`) was called with more arguments than formal parameters (`argc > c.Nparams`), `execFast`'s `OpCall` declined and bounced to slow `exec` to lay out the extra-arguments region — making `arguments_read` and `arguments_strict_read` bounce out of `execFast` on every call.
3. **Eight-load prototype-chain element guard on every indexed write and `push`**: Every `OpSetIndex`, `OpSetIndexInt`, and inlined `Array.prototype.push` in `execFast` walked `%Array.prototype%` and `%Object.prototype%` checking `Shape`, `Proto`, `ElemCount()`, `NumCount()`, and `SparseFlag` (8 pointer/memory loads per write) to verify that no prototype held indexed properties — even though in 99.99% of JS programs **no prototype object ever gains an indexed element**.

---

## 3. Prerequisites & Dependency Order

- **Depends on Optimization #2** (`execFast` leaf interpreter loop) and **Optimization #4** (lean inline call frames and 2-instruction peek-fusion).
- Independent of Phase 2's `execQuiet` (`09`), `BigInt` accumulation (`12`), and `RegExp`/`Date` kernels (`11`).

---

## 4. Target Files Inventory

| File | Role |
| --- | --- |
| `internal/storage/space.go` | Add `ProtoElemsDirty bool` field right next to `ProtoEpoch` on `Space`. |
| `internal/storage/object.go` | Add `(*Object).touchProtoElems()` (sets `sp.ProtoElemsDirty = true` when `o.flags&ObjIsProto != 0`); call it in `SetProto` when mutating a prototype's own prototype chain. |
| `internal/storage/shape.go` | Set `sp.ProtoElemsDirty = true` in `NewShape` / `rebuildShape` when an object becomes a prototype while already possessing elements or sparse indices. |
| `internal/storage/sparseindex.go`, `internal/storage/property.go`, `internal/storage/valuestorage.go`, `internal/storage/array.go` | Call `touchProtoElems()` on every indexed element or sparse property write to a prototype object; fast-path `(*RealmPart).ArrayProtoPlain()` when `!r.ProtoElemsDirty`. |
| `internal/storage/funcproto.go` | Add `LeafKind` (`LeafNone`, `LeafReturnConst`, `LeafReturnParam`, `LeafStepUpval`), `LeafIndex`, `LeafDelta`, `LeafVal` on `FuncProto`; classify eligible functions in `Prepare()`. |
| `internal/engine/interp.go` | Add frameless leaf execution in `execFast` `OpCall`/`OpCallUndef`, surplus-argument frame placement (`argc > c.Nparams && c.NeedsExtra`), `OpReturn` + `OpAdd` fusion (on `OpCallUndef`), `OpGetArgsIndexInt` + `OpAdd` fusion, and `!ag.ProtoElemsDirty` single-bit checks in `OpSetIndex`, `OpSetIndexInt`, and inlined `push`. |
| `internal/engine/validate_on.go` | Allowlist `OpCallUndef` and `OpGetArgsIndexInt` in `canFuse` for `OpAdd`. |
| `internal/engine/protowalk_test.go` | Reduce `"agent.execFast"` prototype-read count from `8` to `3` (matching the 5 eliminated prototype walks). |

---

## 5. Detailed Design Specification (No Code)

### 5.1 Single-Bit `ProtoElemsDirty` Invariant (`internal/storage/`)

1. Add `ProtoElemsDirty bool` on `storage.Space` immediately adjacent to `ProtoEpoch`. Initially `false`.
2. Invariant: While `sp.ProtoElemsDirty == false`, **no object in `sp` that has ever served as a prototype** (`ObjIsProto`) has any dense elements (`len(elems) > 0`), packed numeric elements (`len(nums) > 0`), or sparse index properties (`SparseFlag`), nor has any prototype had its own prototype changed to an object containing indexed properties.
3. Add `func (o *Object) touchProtoElems()` in `internal/storage/object.go`: if `o.flags&ObjIsProto != 0 && o.shape != nil`, set `o.shape.Space.ProtoElemsDirty = true`.
4. Wire `touchProtoElems()` (or direct `ProtoElemsDirty = true`) at every mutation point:
   - Whenever an object is first marked `ObjIsProto` (`NewShape` / `rebuildShape` in `shape.go`): if `len(proto.elems) > 0 || len(proto.nums) > 0 || proto.shape.HasSparse()`, set `sp.ProtoElemsDirty = true`.
   - Whenever `SetProto` changes the prototype of an `ObjIsProto` object (or installs a new prototype that has elements/sparse indices), set `sp.ProtoElemsDirty = true`.
   - Whenever a dense element, packed number, or sparse index property is added/defined on any object (`valuestorage.go`, `array.go`, `property.go`, `sparseindex.go`), call `o.touchProtoElems()`.
5. In `(*RealmPart).ArrayProtoPlain()` (`internal/storage/array.go`), check `if !r.ProtoElemsDirty` (alongside `%Array.prototype%`'s prototype still being `%Object.prototype%` and `%Object.prototype%`'s prototype being `nil`) to return `true` immediately without inspecting elements/sparse flags.
6. In `execFast` (`internal/engine/interp.go`), replace the multi-load `%Array.prototype%` / `%Object.prototype%` element-and-sparse checks in `OpSetIndex`, `OpSetIndexInt`, and inlined `Array.prototype.push` with `!ag.ProtoElemsDirty`!

### 5.2 Compile-Time Leaf Function Classification (`internal/storage/funcproto.go`)

1. Define `type LeafKind uint8` with four values: `LeafNone`, `LeafReturnConst`, `LeafReturnParam`, `LeafStepUpval`, and add fields `LeafKind`, `LeafIndex int32`, `LeafDelta float64`, `LeafVal Value` to `FuncProto`.
2. In `(*FuncProto).Prepare()`, classify functions whose `InlineFrames == true && !NeedsExtra && !HasDefaultParams && !UsesThis` (checking exact preconditions so no `arguments`, `this` boxing, TDZ, or default expressions can be skipped!):
   - **CRITICAL — Trailing `OpReturnUndef`**: Every compiled function ends with an implicit trailing `OpReturnUndef` instruction! Therefore, check `len(p.Code) >= 1` / `>= 2` (NOT `== 1` / `== 2`!), matching the prefix instructions that unconditionally return:
   - **`LeafReturnConst`**:
     - `Code[0] == OpReturnUndef` → `LeafVal = Undefined`.
     - `Code[0] == OpLoadK(dst, k)` + `Code[1] == OpReturn(dst)` → `LeafVal = Consts[k]`.
     - `Code[0] == OpLoadInt(dst, v)` + `Code[1] == OpReturn(dst)` → `LeafVal = Number(float64(v))`.
     - `Code[0] == OpLoadNull/True/False/Undef(dst)` + `Code[1] == OpReturn(dst)` → corresponding constant `LeafVal`.
   - **`LeafReturnParam`**:
     - `Code[0] == OpReturn(r)` where `0 <= r < Nparams` → `LeafIndex = r`.
     - `Code[0] == OpMove(dst, r)` + `Code[1] == OpReturn(dst)` where `0 <= r < Nparams` → `LeafIndex = r`.
   - **`LeafStepUpval`**:
     - `Code[0] == OpIncUpval(u)` or `OpDecUpval(u)` followed immediately by `Code[1] == OpReturnUndef` (or returning the stepped value if matched) → `LeafIndex = u`, `LeafDelta = +1` or `-1`, `LeafVal = Undefined` (or return new value if the body returns the incremented upvalue). Ensure `OpIncUpval`/`OpDecUpval` only executes framelessly when the upvalue's current `*Value` is a `NumberObject`!

### 5.3 Frameless Leaf Execution, Surplus-Arg Frames & Fusions (`internal/engine/interp.go`)

1. **Frameless leaf execution in `execFast` (`OpCall` / `OpCallUndef`)**:
   - Check `if kind := c.LeafKind; kind != storage.LeafNone && len(ag.frames) < cap(ag.frames)` (preserving stack-overflow depth checks!).
   - Also check `if !validateExec` OR run normal frame in `validateExec` if step tracing expects callee steps — wait: check `TestExecFastPathsCallNothing` and `validate_on.go`: under `validateExec`, either decline frameless leaf calls (so `checkStep` sees every callee instruction!) or allow them cleanly. Declining frameless leaf execution inside `if !validateExec` guarantees 100% step-by-step validation parity with zero runtime cost in normal builds (`validateExec` is a compile-time `false` constant!).
   - **`LeafReturnConst`**: write `c.LeafVal` to `regs[i.A]`. If `i.Op == OpCallUndef && c.LeafVal.O == NumberObject`, peek-fuse an immediately following `OpAdd` (`n.Op == OpAdd && n.C == i.A && n.A == n.B && sum.O == NumberObject`).
   - **`LeafReturnParam`**: read `v := Undefined`; if `int(c.LeafIndex) < argc`, read `v = regs[first + int(c.LeafIndex)]`; write `v` to `regs[i.A]`, with the same `OpCallUndef + OpAdd` peek-fusion when `v.O == NumberObject`.
   - **`LeafStepUpval`**: read `up := f.Fn.Upvals[c.LeafIndex].Loc`; if `up.O != storage.NumberObject`, break to normal call (for BigInt/string/coercion); otherwise compute `nv := up.N + c.LeafDelta`, call `storage.SetNumber(up, nv)`, write return value (`Undefined` or `nv`) to `regs[i.A]`, and `continue` — **zero frame push, zero frame pop, zero register reload**!
2. **Surplus-argument inline frames (`argc > c.Nparams && c.NeedsExtra`)**:
   - Previously `execFast` required `argc <= c.Nparams`. Extend `execFast` so that when `argc > c.Nparams && c.NeedsExtra` (and enough stack capacity exists for `argBase + argc + c.MaxRegs`), place the callee's register window at `base = argBase + argc` (so the caller's argument slots `[argBase .. argBase+argc)` stay untouched as the `extraBase` backing region!), copy formal parameters `0..c.Nparams-1` into `[base .. base+c.Nparams)`, zero `[base+c.Nparams .. base+c.MaxRegs)`, and record `extraBase = argBase + c.Nparams` and `argc` on the new `frame`!
3. **`OpReturn` + `OpAdd` and `OpGetArgsIndexInt` + `OpAdd` peek-fusion**:
   - In `execFast` `OpReturn`: when returning a `NumberObject` value `v` to a caller whose call instruction (`execInstrAt(fp, codeBase, fr.pc-1)`) was `OpCallUndef` and whose next instruction `n` is `OpAdd` (`n.C == ret && n.A == n.B && sum.O == NumberObject`), add `v.N` directly into `sum` and advance the caller's `ip`!
   - In `execFast` `OpGetArgsIndexInt`: after loading an unmaterialized argument `v`, if `v.O == NumberObject` and the next instruction is `OpAdd` accumulating `i.A` into `sum`, fuse the addition directly!
   - Register `OpCallUndef` and `OpGetArgsIndexInt` in `canFuse` for `OpAdd` in `internal/engine/validate_on.go`.

---

## 6. Architectural Invariants & White-Box Test Updates

- **`internal/engine/protowalk_test.go`**: Update `"agent.execFast"` under prototype reads from `8` to `3`.
- **`internal/engine/validate_on.go`**: Add `OpCallUndef` and `OpGetArgsIndexInt` to `canFuse` for `OpAdd`.
- **`TestExecFastPathsCallNothing`**: All frameless leaf and surplus-arg paths remain 100% call-free.

---

## 7. Known Pitfalls & Failed Approaches to Avoid

1. **Pitfall #1 — Checking `len(p.Code) == 2` in `FuncProto.Prepare()` misses 100% of leaf functions**: Every Ekuma function appends a trailing `OpReturnUndef` instruction, so `function f(a) { return 1; }` compiles to **3** instructions (`OpLoadInt`, `OpReturn`, `OpReturnUndef`). Always check `len(p.Code) >= 1` / `>= 2` and inspect the leading instructions up to the first unconditional `OpReturn`/`OpReturnUndef`.
2. **Pitfall #2 — Forgetting `len(ag.frames) < cap(ag.frames)` on frameless leaf calls**: Even though a leaf function cannot recurse, checking frame-stack room preserves exact call-depth limits if a leaf function is invoked at maximum recursion depth.
3. **Pitfall #3 — Missing a prototype mutation site for `ProtoElemsDirty`**: Remember that an object can receive indexed elements *before* being passed to `Object.setPrototypeOf` or `NewShape`. Setting `ProtoElemsDirty = true` both on element writes to existing `ObjIsProto` objects AND when an object with elements/sparse indices newly becomes a prototype guarantees soundness across all Test262 prototype-element tests.

---

## 8. `unsafe` Usage & Pure-Safe-Go Alternative

- **`unsafe` used**: **Zero** new `unsafe` operations — 100% safe Go building on `execFast`'s existing instruction/register helpers.

---

## 9. Verification & Reproduction Protocol

1. Run `go test ./internal/storage ./internal/engine ./internal/builtin`.
2. Run `go test -tags ekumavalidate -run 'TestExecFastPathsCallNothing|TestPrototypeWalks|TestRawStorage' ./internal/engine`.
3. Run `_bench` microbenchmarks (`-suite Micro -engine '^ekuma$' -rounds 1`) and confirm: `func_call` drops from ~104 ns to ~34–36 ns (~3x), `func_closure_call` from ~105 ns to ~34 ns (~3x), `global_func_call` from ~98 ns to ~38 ns (~2.5x), and `arguments_read`/`arguments_strict_read` drop from ~288 ns to ~200–215 ns (~1.35x).
