# Optimization #11: RegExp Whole-Literal Fast Path, Single-Allocation Match Results, `RegExp`/`String.replace`/`Array.slice` Kernels, and Integer-Fast `Date` Kernels

## 1. Summary & Measured Impact

- **Overall Rank**: #7 of 13
- **Suite-Wide Impact**: **+2.83 nats** attributed to RegExp, Date, and `Array.prototype.slice` (part of Stage 11's **+6.42 nats** combined lift alongside `execQuiet` loop-glue completion)
- **Targeted Subset Impact**: **+2.67 nats** across the 3 RegExp and 2 Date microbenchmarks, plus flipping macrobenchmark `RegExp` from 0.98x to **1.06x** vs baseline
- **Key Benchmark Improvements (Stage 10 → Final 3-Round Median)**:

| Benchmark | Stage 10 Ekuma | Final Ekuma | Stage 11 Speedup | Overall vs Baseline |
| --- | ---: | ---: | ---: | ---: |
| `regexp_replace` (`str.replace(/.../g, rep)`) | 440.6 µs | 206.1 µs | **2.14x** (+0.76 nats) | **1.95x** |
| `regexp_ascii` (`/literal/.exec(asciiStr)`) | 400.7 µs | 197.9 µs | **2.02x** (+0.71 nats) | **1.83x** |
| `regexp_utf16` (`/literal/.exec(utf16Str)`) | 422.7 µs | 219.3 µs | **1.93x** (+0.66 nats) | **1.86x** |
| `date_parse` (`Date.parse`, `toISOString`, `toGMTString`, `toString`) | 4.62 µs | 3.08 µs | **1.50x** (+0.40 nats) | **1.43x** |
| `date_now` (`Date.now()` loop) | 92.2 ns | 80.4 ns | **1.15x** (+0.14 nats) | **1.08x** |
| `array_slice` (`Array.prototype.slice` kernel) | 2.98 µs | 2.89 µs | **1.03x** | **2.70x** |

---

## 2. Root Cause Analysis

CPU profiling revealed that RegExp and Date operations were dominated by native-frame setup, redundant allocations, and floating-point calendar math:

1. **No quiet kernels for `RegExp.prototype.exec` or `String.prototype.replace`**: Even when matching a lent RegExp literal (`OpRegExpMethod` / `OpRegExpArg`) with unmodified `%RegExp.prototype%`, every call built a full `invokeNative` frame (56–67% of total CPU time in `regexp_ascii` and `regexp_utf16`!).
2. **Multi-allocation match results & redundant substring copies**: Even though `internal/regexp/compile.go` already set `prog.whole = true` for exact literal patterns, `builtinExec` ran through the RegExp matcher, converted UTF-16 spans, allocated a substring for the matched text (identical to the literal itself!), and allocated the match result array with separate `SetElement` + `DefineMatchResultProperties` steps. Additionally, `RecordRegExpMatch` triggered two Go GC write barriers (`l.Input = s; l.Str = s`) on every match even when matching the same subject string repeatedly.
3. **16 `invokeNative` frames + floating-point `modulo` + `strconv.AppendUint` in `date_parse`**: None of `Date.parse`, `toISOString`, `toUTCString`/`toGMTString`, or `toString` had quiet kernels (79.5% of CPU was under `invokeNative`). Inside `internal/datetime`, `HourFromTime`/`MinFromTime`/`SecFromTime`/`WeekDay`/`DaysInYear`/`MakeDay` all called floating-point `modulo()` separately (12.8% of CPU!), `pad()` called `strconv.AppendUint` 6–7 times per formatted date (12.8% of CPU!), `matchName` linearly scanned 12 month strings and 7 weekday strings (7.7% of CPU!), and `NewDateObject` performed a shape-map lookup on every `new Date(...)`.

---

## 3. Prerequisites & Dependency Order

- **Depends on Optimization #9** (`execQuiet` dual-mode loop, with `OpRegExpMethod` and `OpRegExpArg` handled directly inside `execQuiet`).
- `datetime` integer-math and formatting speedups are standalone and benefit every caller (`Date`, `Temporal`, and `Intl`).

---

## 4. Target Files Inventory

| File | Role |
| --- | --- |
| `internal/regexp/regexp.go` | Add `(*Regexp).WholeLiteral() (wtf8.String, bool)` exposing pure valid-UTF-8 unanchored literal patterns. |
| `internal/storage/regexpstate.go` | Add `Literal wtf8.String` and `LiteralVal Value` on `RegExpState`; add `(*Object).RegExpLiteral() (wtf8.String, Value, bool)`. |
| `internal/storage/regexpobject.go` | Add single-allocation `(*RealmPart).NewLiteralMatchResult(matched, index, input Value) *Object` and cached `(*RealmPart).DateShape() *Shape`. |
| `internal/storage/shape.go` | Add `date *Shape` field to `shapeTable`. |
| `internal/storage/space.go` | Add `HasHostTimeZone bool` next to `HasHostClock bool`. |
| `internal/engine/regexp.go` | Populate `Literal`/`LiteralVal` in `RegExpInitialize` and `regExpLiteralState`; split out non-reentering `regExpLiteralMethodFast` for `execQuiet`. |
| `internal/engine/clock.go` | Set `ag.HasHostTimeZone = tz != nil` in `SetDefaultTimeZone`. |
| `internal/engine/slots.go` | Use cached `cx.DateShape()` in `NewDateObject` when `proto == IntrinsicDatePrototype`. |
| `internal/datetime/datetime.go` | Add single-remainder integer `timeOfDay(t float64)` and fast `int64` paths in `DaysInYear`, `WeekDay`, `HourFromTime`, `MinFromTime`, `SecFromTime`, and `MakeDay` (`0 <= m < 12`). |
| `internal/datetime/format.go` | Add `digitPairs`-based `append2`/`append3`/`append4`/`append6` and buffer-appending `AppendISOString`, `AppendUTCString`, `AppendString`; rewrite `ISOString`/`UTCString`/`String`/`DateString`/`TimeString` to use stack byte arrays. |
| `internal/datetime/parse.go` | Replace `poll func()` with `type Poller interface { Poll() }` (matching `regexp.Poller` and eliminating closure allocations + `TestQuietContextBoundary` violations); add O(1) 3-byte integer switches `matchWeekday3` and `matchMonth3`; unroll 2-digit and 4-digit paths in `(*scanner).fixed`. |
| `internal/datetime/parse_test.go` | Adapt `TestParsePolled` to a 3-line `pollCounter` implementing `datetime.Poller`. |
| `internal/quiet/clock.go` | Add `DefaultTimeZone() (datetime.TimeZone, bool)` returning `datetime.Local(), true` iff `!q.Realm.HasHostTimeZone`. |
| `internal/quiet/regexp.go` | Add allocation-free, write-barrier-skipping `RecordRegExpMatch2(r, s, start, end)`. |
| `internal/quiet/regexpkernels.go` | Implement `RegExpExecKernel` (whole-literal + ≤ 5-capture general fast paths) and `StringReplaceKernel` (plain-string + whole-literal RegExp + general unmodified RegExp fast paths). |
| `internal/quiet/datekernels.go` | Implement `SetDateTimeZoneNameHook`, `DateParseKernel`, `DateToISOStringKernel`, `DateToUTCStringKernel`, and `DateToStringKernel`. |
| `internal/quiet/arraykernels.go` | Implement `ArraySliceKernel0`, `ArraySliceKernel1`, `ArraySliceKernel2` (declining slices > `storage.PollInterval`). |
| `internal/builtin/regexp.go`, `internal/builtin/string.go`, `internal/builtin/date.go`, `internal/builtin/array.go` | Register all new kernels with `Contract: shortContract`; pass `cx` as `datetime.Poller` in `date.go`; notify `quiet.SetDateTimeZoneNameHook` in `SetDateTimeZoneName`. |
| `internal/engine/protowalk_test.go`, `internal/engine/testdata/rawusers.txt` | Allowlist `"realmPart.dateShape": 1` in `protoCopies`; rename `ExecutionContext.regExpLiteralMethod` to `ExecutionContext.regExpLiteralMethodFast` in `rawusers.txt`. |

---

## 5. Detailed Design Specification (No Code)

### 5.1 RegExp Whole-Literal Detection & Single-Allocation Match Result

1. **`(*Regexp).WholeLiteral()` (`internal/regexp/regexp.go`)**: Return `(p.prefix, true)` iff `p.whole && !p.anchored && p.prefix != "" && utf8.ValidString(string(p.prefix))`. Requiring `utf8.ValidString` guarantees the literal contains no lone surrogates (`0xED 0xA0..BF`), so because WTF-8 is self-synchronizing, a byte-level `strings.Index` search is **100% semantically identical** to UTF-16 ECMAScript matching in both Unicode and non-Unicode modes!
2. **Cached `LiteralVal` (`internal/storage/regexpstate.go`, `internal/engine/regexp.go`)**: Store `Literal` and `LiteralVal` (interned once via `cx.NewString(lit)`) on `RegExpState` at compile/init time. On every match, return `LiteralVal` directly — **zero substring allocation**.
3. **Single-allocation `NewLiteralMatchResult` (`internal/storage/regexpobject.go`)**: Allocate `o, store := NewTailObjectWithStorage(4)` (one single 112-byte heap object!). Set `o.Kind = KindArray`, `o.Class = ClassArray`, `o.shape = r.matchResultShapesFor(r.Shapes.Array).plain`, `o.length = 1`. Populate `store[0] = matched`, `store[1] = index`, `store[2] = input` (`store[3]` is already `Undefined` for `groups`). Attach disjoint full-slice views `o.InitElems(store[:1:1])` and `o.InitSlots(store[1:4:4])` (matching `NewArrayWithRoom` and passing `TestValueStorageOwnership`).
4. **Barrier-skipping `RecordRegExpMatch2` (`internal/quiet/regexp.go`)**: For a single whole-match span `[start, end]`, only assign `l.Input = s` / `l.Str = s` when `l.Input.O != s.O || l.Input.N != s.N` (avoiding 2 GC write barriers per match on repeated searches over the same string!) and write `l.Caps[0], l.Caps[1] = start, end` in place whenever `cap(l.Caps) >= 2`.

### 5.2 `RegExpExecKernel`, `StringReplaceKernel`, and `ArraySliceKernel`

1. **`RegExpExecKernel` (`shortContract`, `internal/quiet/regexpkernels.go`)**:
   - Accept `this.IsObject() && arg.IsString() && arg.StringByteLength() <= 65536`, non-nil `re := r.RegExpMatcher()`, writable numeric `lastIndex` (`q.RegExpLastIndex(r)`), no `FlagHasIndices`, no named groups, and `re.GroupCount() <= 5`.
   - **Whole-literal path**: Convert `lastIndex` to `byteOff` (`0` if `lastIndex == 0`, `int(lastIndex)` if ASCII, or `wtf8.Seek` for non-Unicode non-ASCII when `!pos.Low`; fall back to `q.MatchRegExp` for surrogate-middle Unicode edge cases). Match via `strings.HasPrefix` (if sticky) or `strings.Index` (otherwise). Compute `startUnit` incrementally (`matchByte` for ASCII, or `int(lastIndex) + wtf8.Len(str[byteOff:matchByte])` — scanning only the gap rather than from byte 0!). Update `lastIndex`, call `q.RecordRegExpMatch2`, and return `q.Realm.NewLiteralMatchResult(litVal, Number(startUnit), arg)`.
   - **General ≤ 5-capture path**: Run `q.MatchRegExp`, update `lastIndex` and `RecordRegExpMatch`, and build the result via `NewLiteralMatchResult` (for 0 captures) or single-allocation `NewArrayWithRoom(n+1, 3)` + `DefineMatchResultProperties` (for 1–5 captures).
2. **`StringReplaceKernel` (`shortContract`, `internal/quiet/regexpkernels.go`)**:
   - Accept `this.IsString() && rep.IsString()` with both `<= 65536` bytes and no `'$'` byte in `rep`.
   - **Plain-string `pat`**: Handle `pat.IsString()` via `strings.Index` + stack `[256]byte` assembly (`NewASCIIStringBytes` if both ASCII). Supporting string patterns also ensures `TestKernels` exercises `Method2` within its first 8 operands!
   - **Unmodified RegExp `pat`**: Validate `q.UnmodifiedRegExpMatcher(rx)`, writable numeric `lastIndex`, and `global || !sticky`. Before mutating `lastIndex` or legacy statics, perform an exact worst-case `MaxStringLength` bound check (declining to `replaceUnmodified` if the result could possibly throw `RangeError`).
     - For whole-literal patterns: run a tight `strings.Index` loop into a stack `[512]byte` buffer (growing to heap only if needed), track `lastMatchByte`, reset `lastIndex = 0` once at the end (if global), call `RecordRegExpMatch2` once for the final match, and return `NewASCIIStringBytes` / `NewString`.
     - For general unmodified non-literal patterns (`len(str) <= 16384`): run a tight `q.MatchRegExp` loop writing directly into a stack `[512]byte` buffer without `wtf8.Builder` or `substituteMatch`.
3. **`ArraySliceKernel0/1/2` (`shortContract`, `internal/quiet/arraykernels.go`)**:
   - Validate plain `KindArray`, numeric/undefined `start`/`end`, and `end - start <= storage.PollInterval` (declining huge slices so `arraySlice` handles interruption polling and `TestUnregisteredShortcutInterruption` passes!), then delegate to `q.CopyElementRange(o, start, end)`.

### 5.3 Integer-Fast `datetime` & Host-Safe `Date` Kernels

1. **Integer calendar arithmetic (`internal/datetime/datetime.go`)**: Every valid ECMAScript Date time value lies in `[-8.64e15, +8.64e15]`, which is strictly inside exact `int64` / `2^53` range. Add `timeOfDay(t)` computing `(hour, min, sec, ms)` from a **single** `int64(t) % MsPerDay` remainder, replace `modulo()` calls with direct `int64` remainders in `DaysInYear`, `WeekDay`, `HourFromTime`, `MinFromTime`, `SecFromTime`, and short-circuit `0 <= m < 12` in `MakeDay`.
2. **Table-driven ASCII formatting (`internal/datetime/format.go`)**: Replace `strconv.AppendUint` calls in `pad()` with a 200-byte `digitPairs` lookup table (`append2`, `append3`, `append4`, `append6`), keeping `appendYear` unpadded beyond 4 digits for years ≥ 10,000 (only `ISOString` zero-pads years > 9999 to 6 digits!). Export `AppendISOString`, `AppendUTCString`, and `AppendString` so kernels format directly into stack `[32]`/`[40]`/`[96]byte` buffers and construct a single `asciiText32` or `NewASCIIStringBytes` value.
3. **O(1) month/weekday matching & unrolled `fixed` (`internal/datetime/parse.go`)**: Pack 3-byte ASCII abbreviations into `uint32(s[0])<<16 | uint32(s[1])<<8 | uint32(s[2])` for O(1) switch dispatch in `matchWeekday3`/`matchMonth3`, unroll `n == 2` and `n == 4` in `(*scanner).fixed` using integer arithmetic, and replace `poll func()` with `type Poller interface { Poll() }` so `datetime.Parse` has zero indirect function-value calls (passing `TestQuietContextBoundary`).
4. **Host-hook-safe `Date` kernels (`internal/quiet/datekernels.go`, `internal/builtin/date.go`)**:
   - Track `HasHostTimeZone bool` on `storage.Space` (updated by `SetDefaultTimeZone`) and `hasDateTimeZoneNameHook bool` in `quiet` (updated by `builtin.SetDateTimeZoneName`).
   - `DateParseKernel` (`Method1`, declines `len > 256` or `HasHostTimeZone`), `DateToISOStringKernel` (`Method0`, declines `NaN`), `DateToUTCStringKernel` (`Method0`, handles `toUTCString` and `toGMTString`), and `DateToStringKernel` (`Method0`, declines `HasHostTimeZone` or `hasDateTimeZoneNameHook`). All registered with `shortContract`.
   - Cache `DateShape()` on `RealmPart.Shapes.date` and use it in `ExecutionContext.NewDateObject`.

---

## 6. Architectural Invariants & White-Box Test Updates

- **`TestKernels` (`withHostHooks` & `testPace`)**: Because `HasHostTimeZone` declines `DateParseKernel`/`DateToStringKernel` whenever a custom `TimeZone` is installed, host re-entrancy hooks observe the exact native call frame. Declining inputs exceeding size caps (`65536` for RegExp/Replace, `256` for `Date.parse`, `PollInterval` for `Array.slice`) guarantees `shortContract` (`PaceShort`) matches generic interruption behavior on all `largeOperands`.
- **`TestQuietContextBoundary`**: Replacing `poll func()` with `datetime.Poller` removes the only indirect function-value call reachable from `quiet.DateParseKernel`.
- **`TestPrototypeWalks`**: Add `"realmPart.dateShape": 1` under `protoCopies` in `internal/engine/protowalk_test.go`.
- **`TestRawStorage`**: Rename `ExecutionContext.regExpLiteralMethod` to `ExecutionContext.regExpLiteralMethodFast` in `internal/engine/testdata/rawusers.txt`.

---

## 7. Known Pitfalls & Failed Approaches to Avoid

1. **Pitfall #1 — Padding 5-digit years to 6 digits in `appendYear`**: `Date.prototype.toUTCString` and `toDateString` format year `10000` as `"10000"` (5 digits), NOT `"010000"`! Only `toISOString` pads years > 9999 to 6 digits (`+010000`). Keep `append6` strictly inside `AppendISOString`.
2. **Pitfall #2 — `TestKernels` `Method2` operand coverage (`few = 8`)**: In `kernel_test.go`, `operandLists` only combines the first 8 operands in `kernel-probes.js` for `Method2` (`"RegExp"` is around index 55). Implementing plain-string `pat` support inside `StringReplaceKernel` ensures `Method2` is exercised on `"two units"` while accelerating both string and RegExp replacements.
3. **Pitfall #3 — Registering `ArraySliceKernel` with `polledContract` fails `TestKernels` and `TestUnregisteredShortcutInterruption`**: `PollChunk` is `1 << 20`, which is larger than `large["array"]` (`1.5e5`), so `CopyElementRange` does not poll on `1.5e5` elements. Use `shortContract` and decline `end - start > storage.PollInterval`.
4. **Pitfall #4 — `datetime.ParsePolled(..., func())` triggers `TestQuietContextBoundary`**: Even when `Parse` passes `nil` as the `func()`, static call-graph analysis sees an indirect call to `sc.poll()`. Converting `poll` to an explicit `Poller` interface eliminates both the violation and the `cx.Poll` method-value allocation.

---

## 8. `unsafe` Usage & Pure-Safe-Go Alternative

- **`unsafe` used**: **Zero** — 100% pure safe Go across all RegExp, Date, and Array slice optimizations.

---

## 9. Verification & Reproduction Protocol

1. Run `go test ./internal/regexp ./internal/datetime ./internal/builtin ./internal/engine`.
2. Run `go test -tags ekumavalidate -run 'TestKernels|TestQuietContextBoundary|TestPrototypeWalks|TestRawStorage|TestValueStorageOwnership|TestUnregisteredShortcutInterruption' ./internal/engine`.
3. Run `go test -race ./internal/regexp ./internal/datetime ./internal/engine ./internal/builtin`.
4. Run `_bench` microbenchmarks (`-suite Micro -engine '^ekuma$' -rounds 1`) and verify: `regexp_ascii` drops below 210 µs (~2x), `regexp_utf16` below 230 µs (~1.9x), `regexp_replace` below 215 µs (~2.1x), and `date_parse` drops from ~4.5 µs to ~3.1 µs (~1.45x).
