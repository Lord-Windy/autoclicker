# Plan: Multi-Location Sequence Clicking with Sub-Loops

Goal: extend the autoclicker from a single target to a **sequence of click
locations**, each with its own label, delay, and repeat count, **sub-loops**
(e.g. "run steps 3–5 twenty-five times, then continue with step 6, then start
over at step 1"), plus an optional **pixel jitter** mode (randomly shifting
1–2 px in X/Y and holding that offset for a random `[min, max]` number of
ticks/clicks). Primary use case: automating click-heavy game grinding to
relieve RSI.

## What exists today (and what it means for this feature)

| Component | Today | Impact |
|---|---|---|
| `MainWindow` | State machine (Idle→Countdown→Clicking), **one** `m_target`, fixed-size window, `poll()` assumes click limit | Heavy rework — becomes a resizable sequence editor, reports `StopReason` & loop/step progress |
| `ClickWorker` | Thread that clicks one point at a fixed rate (`maxClicks` limit) | Rewrite — becomes a recursive `Sequence` interpreter against a `PointerSink` interface |
| `MouseWatcher` | KWin script: cursor >3px from **the single target** = user moved → stop | **Must change** — embed the set of all step positions; keep `start(QPoint)` overload |
| `TargetPicker` | Full-screen crosshair overlay, emits `targetSelected(QPoint)` | **No changes to `TargetPicker`** — `MainWindow` dispatches between *Add step* and *Re-pick step* |
| `VirtualPointer` | uinput `moveTo(pos, desktop)` + `click(hold)` | Implement `PointerSink` interface so tests can swap in a `FakePointer`; uinput logic unchanged |
| `src/main.cpp` | `--test-click`, `--test-move`, `--screenshot` | Keep `--test-move` & `--test-click` working; adapt `--screenshot` in Phase 2; add `--test-sequence` in Phase 4 |
| Settings | Only saves `maxClicks` in `QSettings` | Save sequence as versioned JSON (`sequence.json` in `AppConfigLocation`) + window geometry in `QSettings` |

## 1. Data model & JSON schema

Semantic note: **per-step repeat ≠ sub-loop**. If steps 3–5 each repeat 25×,
you get `3,3,3…4,4,4…5,5,5…`. A sub-loop is `3,4,5,3,4,5…×25` — a *group*
repeat. The model needs both:

```cpp
// src/Sequence.h (new)
struct ClickStep {
    QPoint pos;
    int delayAfterMs = 200;   // step period / delay after click before the next click (ms, >= 0)
    int repeat = 1;           // click this spot N times (>= 1 in UI)
    QString label;            // optional user description, e.g. "Craft", "Confirm"
    // future-proof, cheap to add later:
    // enum class Button { Left, Right } button = Button::Left;
};

struct SeqNode {
    enum class Type { Step, Loop };
    Type type = Type::Step;         // always default-initialized
    ClickStep step;                 // valid when Type::Step
    int iterations = 1;             // valid when Type::Loop (>= 1 in UI)
    QList<SeqNode> children;        // valid when Type::Loop

    static SeqNode makeStep(const ClickStep &s);
    static SeqNode makeLoop(int iterations, QList<SeqNode> children);
};

using Sequence = QList<SeqNode>;

struct JitterConfig {
    static constexpr int MaxOffsetPx = 2; // shifts 1–2 px in X/Y
    bool enabled = false;
    int minTicks = 4;                     // minimum clicks before re-rolling offset (>= 1)
    int maxTicks = 12;                    // maximum clicks before re-rolling offset (>= minTicks)
};

struct SequenceDocument {
    int version = 1;
    quint64 maxLoops = 0;           // 0 = unlimited
    quint64 maxClicks = 0;          // 0 = unlimited
    JitterConfig jitter;
    Sequence nodes;
};
```

- The **main loop** runs `nodes` repeatedly until `maxLoops` or `maxClicks` is
  reached (`0` = unlimited for each) or the user moves the mouse.
- A **sub-loop** is a `Loop` node: "steps 3–5 ×25, then step 6" becomes
  `[1, 2, Loop×25(3,4,5), 6]`.
- **Leaf step numbering:** Leaf `Step` nodes are numbered `1..N` sequentially in
  pre-order traversal across the whole tree (both in the UI `#` column and in
  progress reporting `Step k/N`). `Loop` header rows display `Loop ×N` without
  consuming a step number, so grouping steps 3–5 keeps step 6 as `#6`.
- Design the engine **recursively from day one** (nesting a loop inside a loop
  just works), even though the UI restricts grouping to one level of nesting
  initially. That keeps Phase 3 purely UI work.
- **JSON schema:** Always serialize a versioned root JSON object
  (`{ "version": 1, "maxLoops": 0, "maxClicks": 0, "jitter": { ... }, "nodes": [...] }`)
  rather than a bare array so auto-save, `--test-sequence file.json`, and future
  named profiles share the exact same format.

## 2. The four hard parts

### a. `MouseWatcher` — multi-target & jitter safety (most important change)

Today the KWin script embeds a single `tx, ty`; any position >3px from it
means "user moved, stop." With the pointer jumping between targets, the
worker's own moves would instantly false-trigger this.

**Why target-set checking works cleanly on KWin + uinput:**
`VirtualPointer::moveTo` uses `EV_ABS` (`INPUT_PROP_POINTER`) and sends both X
and Y axes before `SYN_REPORT` (first `<1 px` off-center, then exact). KWin
applies absolute motion directly without intermediate interpolated points, and
`workspace.cursorPosChanged` is edge-triggered on movement.

**Fix:**
- Add `bool MouseWatcher::start(const QList<QPoint> &targets, int tolerance = DefaultTolerance)`
  (`DefaultTolerance = 3`) and keep a convenience overload
  `bool start(QPoint target, int tolerance = DefaultTolerance)` (so
  `src/main.cpp`'s `--test-move` call `watcher->start({-1000000, -1000000})`
  keeps compiling).
- **Jitter tolerance coordination:** When pixel jitter (`±2 px`) is enabled,
  uinput→Wayland coordinate rounding (`±1–2 px`) plus a `2 px` jitter offset
  can reach `~3.5–4 px` from the base target and would false-trip a `3 px`
  watcher! Pass `tolerance = DefaultTolerance + JitterConfig::MaxOffsetPx`
  (`5 px`) whenever jitter is enabled.
- Deduplicate `targets` before generating a JS array
  `var targets = [[x1,y1],[x2,y2],…];`.
- Movement counts as user input only when the cursor is outside tolerance of
  **every** target (`!nearAnyTarget(p)` where near means
  `Math.abs(p.x - t[0]) <= tol && Math.abs(p.y - t[1]) <= tol`).

### b. `ClickWorker` — from rate loop to interpreter

Introduce a minimal `PointerSink` interface (`moveTo`, `click`) implemented by
`VirtualPointer` and by `FakePointer` in unit tests. Replace
`ClickWorker::run(...)` with a recursive walker over `Sequence`:

```cpp
// Pending deadline from the previous click's delayAfterMs (nullopt before the 1st click)
// Step (repeated step.repeat times):
//   1. If nextActionDeadline is set: interruptible wait_until(*nextActionDeadline);
//      if stopRequested -> StopReason::Stopped, return false
//   2. Update currentStepIndex; stepStart = Clock::now()
//   3. hold = clamp(step.delayAfterMs / 4, 5ms, 50ms)
//   4. pointer.moveTo(step.pos, desktop); pointer.click(hold); totalClicks++
//   5. If maxClicks > 0 && totalClicks >= maxClicks -> StopReason::ClickLimitReached, return false
//   6. nextActionDeadline = stepStart + milliseconds(step.delayAfterMs)
// Loop:
//   for (int i = 0; i < node.iterations; ++i)
//       if (!execNodes(node.children)) return false;
// Main loop:
//   for (quint64 loop = 0; maxLoops == 0 || loop < maxLoops; ++loop) {
//       quint64 clicksBefore = totalClicks;
//       if (!execNodes(sequence)) break;
//       if (totalClicks == clicksBefore) { stopReason = EmptySequence; break; } // prevent 100% CPU spin!
//   }
//   if (!stopRequested && stopReason == None) stopReason = LoopLimitReached;
```

Key properties of this design:
- **Exact timing at all speeds:** Measuring `nextActionDeadline` from
  `stepStart` with `hold = clamp(delayAfterMs / 4, 5ms, 50ms)` means `20ms`
  achieves an exact `20ms` period (50 CPS, matching v1.0), `200ms` achieves an
  exact `200ms` period (5 CPS), `2000ms` waits 2s after the click starts, and
  `delayAfterMs = 0` still holds the button for `5ms` so games do not ignore
  the click.
- **No stall after the final click:** Because `nextActionDeadline` is waited
  *before* the next click, hitting `maxClicks` or finishing the final step of
  `maxLoops` exits immediately instead of appearing stuck on a trailing
  cooldown delay, while `currentStepIndex` remains on step `k` throughout step
  `k`'s post-click wait.
- **Zero-work spin guard:** If a sequence has empty loops (`Loop×25([])`) or
  `repeat = 0` steps from hand-edited JSON, `totalClicks == clicksBefore`
  breaks immediately instead of spinning a CPU core at 100% when
  `maxLoops == 0`.
- **Pixel jitter with tick dwell:** When `jitter.enabled` is true, each leaf
  step tracks `{ QPoint offset; int ticksRemaining; }`. Whenever
  `ticksRemaining == 0`, pick a new non-zero `(dx, dy)` in `[-2, +2] × [-2, +2]`
  (`!= previous offset`, anchored to `step.pos` so it **never** cumulative-walks
  off the target!) and roll `ticksRemaining` uniformly in
  `[jitter.minTicks, jitter.maxTicks]`. Click `step.pos + offset` and decrement
  `ticksRemaining`.
- **Single progress & stop-reason mechanism:** Keep `ClickWorker` free of
  `QObject` (pure C++ atomics/mutex snapshot for `Progress` and `StopReason`),
  and read both from `MainWindow::poll()` (`m_pollTimer`), replacing the old
  hardcoded `"Click limit reached"` string with accurate messages for
  `ClickLimitReached` vs `LoopLimitReached`.

### c. UI — step list editor & sub-loops

Replace the Target/Rate form with a `QTreeWidget` (columns: `#`, `Label`, `X`,
`Y`, `Delay`, `Repeats`, with indented child rows for loop groups) plus
controls:

- **Add step** → sets `PendingPick::AddStep` and opens `TargetPicker`. Inserts
  after the current selection (including inside a selected loop) or appends at
  the end.
- **Re-pick** → sets `PendingPick::UpdateSelectedStep` and opens `TargetPicker`
  to update the selected step's `QPoint` in place.
- **Remove / Move up / Move down** (moves within the item's parent list).
- **Group into loop** (Phase 3) → `QTreeWidget` uses `ExtendedSelection`;
  enabled only when ≥1 **contiguous top-level `Step` rows** are selected
  (cleanly enforcing 1-level UI nesting). Wraps them in a `Loop` node (`×N`).
- **Ungroup** (Phase 3) → enabled when a `Loop` row is selected; splices its
  children back into the top-level list at the loop's position.
- **Details editor row/panel** for the selected item:
  - When a `Step` is selected: `Label` (`QLineEdit`), `Delay` (`QSpinBox`,
    `0–3600000 ms`, suffix `" ms"`), `Repeats` (`QSpinBox`, `1–100000`, suffix
    `" ×"`).
  - When a `Loop` is selected: `Iterations` (`QSpinBox`, `1–100000`, suffix
    `" ×"`).
- **Stop after:** `N loops` **and/or** `N clicks` (`0 = unlimited` each; first
  limit reached wins).
- **Jitter option:** Checkbox `Move 1–2 px every` + coupled `minTicks` /
  `maxTicks` spinboxes (`1–100000` ticks, keeping `minTicks <= maxTicks`).
- **Window sizing:** Remove `setFixedSize(sizeHint())`; make `MainWindow`
  resizable with a sensible minimum size. Disable all editor widgets whenever
  `m_state != State::Idle`.

### d. Persistence & monitor safety

- Save the active `SequenceDocument` as clean, human-readable JSON to
  `QStandardPaths::writableLocation(QStandardPaths::AppConfigLocation) + "/sequence.json"`
  on close, and load on start. Save window size in `QSettings`.
- Note: v1.0 intentionally did not save the single target; for multi-step RSI
  grind macros, sequences must survive restarts, so this is a deliberate
  product change (update `README.md` accordingly).
- **Multi-monitor check:** Check every step's `pos` against individual screen
  geometries (`std::any_of(screens, [&](QScreen *s) { return s->geometry().contains(pos); })`,
  not `desktopGeometry().contains(pos)`, which misses dead corners on
  mixed-resolution dual monitors). Show a **non-modal** status-bar warning on
  startup (so startup and `--screenshot` never block on a modal dialog) and
  confirm/warn if off-screen targets remain when clicking Start.

## 3. Phased implementation & verification

Each phase corresponds to self-contained tickets under [`tickets/`](tickets/)
and must build cleanly with `-Wall -Wextra` and pass `ctest`.

### Phase 1 — Engine & CTest foundation, invisible in the UI (Tickets 01–03)

1. **Ticket 01:** `src/Sequence.h/.cpp` (structs, traversal/validation helpers,
   versioned JSON serialization) + `include(CTest)` in `CMakeLists.txt` +
   `tests/test_sequence.cpp`.
2. **Ticket 02:** `src/PointerSink.h` + rewrite `ClickWorker` as recursive
   interpreter with `FakePointer` unit tests (`tests/test_click_worker.cpp`
   covering repeats, nested loops, `maxClicks`, `maxLoops`, empty loop /
   `repeat=0` spin guard, and mid-wait `stop()` cancellation). Temporarily
   adapt `MainWindow`'s single-target UI to run a 1-step `Sequence`.
3. **Ticket 03:** `MouseWatcher::start(const QList<QPoint>&)` + `start(QPoint)`
   overload, deduplicated JS target array, and `MainWindow` `StopReason`
   handling.

**Done when:** `ctest` passes, the app behaves identically to v1.0.0, and
`--test-click`, `--test-move`, and `--screenshot` still work.

### Phase 2 — Multi-step UI (Ticket 04)

Flat step list in `QTreeWidget`, Add/Re-pick/Remove/Move Up/Move Down, per-step
label/delay/repeat editor, `Stop after: N loops / N clicks`, status-bar
progress (`Loop 3 · Step 5/8` + `Clicks: 1234`), resizable window, updated
`--screenshot`.

**Done when:** `ctest` passes, multi-step sequences can be built and run in the
UI, and `QT_QPA_PLATFORM=offscreen ./build/autoclicker --screenshot ...`
succeeds.

### Phase 3 — Sub-loops UI (Ticket 05)

`ExtendedSelection`, Group into loop `×N` on contiguous top-level steps,
Ungroup on loop rows, loop iteration count editing, adding/moving/removing
steps inside a loop, pre-order `1..N` leaf numbering, and active row highlight
while running.

**Done when:** `ctest` passes and steps 3–5 can be grouped into `Loop ×25`,
ungrouped, edited, and executed with accurate step progress.

### Phase 4 — Persistence, CLI dry-run, i18n & docs (Ticket 06)

1. Auto-save/load `sequence.json` + window size, plus non-modal off-screen
   coordinate warnings.
2. Add `--test-sequence <file.json> [--live]` to `src/main.cpp`:
   - Default (**dry-run**): runs `ClickWorker` (or sequence walker) against a
     logging `PointerSink` with zero delays / no `/dev/uinput` / no KWin,
     printing planned clicks and totals — safe for automated workflows and CI.
   - `--live`: runs with `MouseWatcher` + `VirtualPointer` so moving the real
     mouse still aborts safely.
3. Run `cmake --build build --target update_translations` and complete all
   German translations in `i18n/autoclicker_de.ts`.
4. Update `README.md`.

*(Optional future stretch goals after Phase 4: delay-only wait steps,
right-click per step, named profile picker, KWin `registerShortcut` panic
hotkey.)*

## 4. Decisions

| Decision | Choice |
|---|---|
| Nesting depth in UI | One level now (covers the main scenario), recursive engine; deeper UI nesting can be unlocked later |
| Delay semantics | **After** the click, measured start-to-start (`hold = clamp(delayAfterMs / 4, 5ms, 50ms)`, remaining wait interruptible via `nextActionDeadline`; skipped after the final click when a limit stops execution) |
| Progress reporting | Pure C++ thread-safe snapshot + `StopReason` on `ClickWorker`, polled by `MainWindow`'s existing `m_pollTimer` (keeps `ClickWorker` free of `QObject` for unit tests) |
| Live-editing while running | Disabled — moving the mouse stops everything anyway, so restart is cheap |
| Save behavior | Auto-save `sequence.json` in `AppConfigLocation` (Phase 4); named profiles as a future stretch goal |
| Jitter semantics | Anchored `±1–2 px` offset around `step.pos` (never cumulative drift), held for a random `[minTicks, maxTicks]` clicks per leaf step; bumps `MouseWatcher` tolerance from `3 px` to `5 px` |
| `--test-sequence` safety | Dry-run trace by default; requires explicit `--live` (with `MouseWatcher` active) to move/click the real pointer |

## Risks / gotchas specific to this codebase

- **Don't break stop-on-mouse-move:** `beginClicking()`'s "watch first, then
  click" ordering stays — `MouseWatcher` must receive **all** reachable step
  coordinates and emit `ready()` before `ClickWorker::start()` runs.
- **Always-on-top window on Wayland:** `MainWindow` uses a KWin script for
  `keepAbove = true`. On Wayland, Qt clients cannot query their own global
  screen coordinates (`QWidget::geometry().topLeft()` is `(0, 0)`), so do **not**
  attempt to compare `MainWindow::frameGeometry()` against target coordinates;
  instead disable all editor controls while `!idle` and note in docs to keep
  the window clear of target spots. Also do not hide/re-show `MainWindow` when
  picking targets (KWin would reposition the window — see `MainWindow.cpp:280`).
- **CMake defaults to `Release` (`-DNDEBUG`):** Never use `<cassert>` `assert()`
  in unit tests — it compiles to nothing in `Release` builds. Always use
  `Qt6::Test` (`QVERIFY` / `QCOMPARE`).
- **AppImage packaging (`packaging/build-appimage.sh`):** Gate unit tests behind
  `if(BUILD_TESTING)` and **never** pass test executables to `install(TARGETS ...)`,
  so `cmake --install` only installs `autoclicker`.
- **Status-bar minimum widths:** Pre-compute `setMinimumWidth(...)` using
  `fontMetrics().horizontalAdvance(...)` on the status-bar progress and click
  labels so numbers changing rapidly do not jitter the status bar layout.
- **Translations:** Keep all user-visible strings wrapped in `tr(...)` using
  the existing typography (`…`, `–`, `×`) and update `i18n/autoclicker_de.ts`
  via `update_translations` in Phase 4.
