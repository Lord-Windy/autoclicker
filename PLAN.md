# Plan: Multi-Location Sequence Clicking with Sub-Loops

Goal: extend the autoclicker from a single target to a **sequence of click
locations**, each with its own delay and repeat count, plus **sub-loops**
(e.g. "run steps 3–5 twenty-five times, then continue with step 6, then start
over at step 1"). Primary use case: automating click-heavy game grinding to
relieve RSI.

## What exists today (and what it means for this feature)

| Component | Today | Impact |
|---|---|---|
| `MainWindow` | State machine (Idle→Countdown→Clicking), **one** `m_target`, fixed-size window | Heavy rework — becomes a sequence editor |
| `ClickWorker` | Thread that clicks one point at a fixed rate (`maxClicks` limit) | Rewrite — becomes a sequence interpreter |
| `MouseWatcher` | KWin script: cursor >3px from **the single target** = user moved → stop | **Must change** — the safety mechanism fundamentally assumes one target |
| `TargetPicker` | Full-screen crosshair overlay, emits `targetSelected(QPoint)` | **No changes needed** — just call `pick()` once per step |
| `VirtualPointer` | uinput `moveTo(pos)` + `click(hold)` | No changes |
| Settings | Only saves `maxClicks` | Extend — save the sequence as JSON |

## 1. Data model

Semantic note: **per-step repeat ≠ sub-loop**. If steps 3–5 each repeat 25×,
you get `3,3,3…4,4,4…5,5,5…`. A sub-loop is `3,4,5,3,4,5…×25` — a *group*
repeat. The model needs both:

```cpp
// src/Sequence.h (new)
struct ClickStep {
    QPoint pos;
    int delayAfterMs = 200;   // wait after clicking before the next step
    int repeat = 1;           // click this spot N times (uses its own delay between)
    // future-proof, cheap to add now or later:
    // enum Button { Left, Right } button = Left;
};

struct SeqNode {
    enum class Type { Step, Loop };
    Type type;
    ClickStep step;                 // valid when Type::Step
    int iterations = 1;             // valid when Type::Loop
    QList<SeqNode> children;        // valid when Type::Loop
};

using Sequence = QList<SeqNode>;   // the whole program
```

- The **main loop** is implicit: run the whole sequence forever (or N times).
- A **sub-loop** is just a `Loop` node: "steps 3–5 ×25, then step 6" becomes
  `[1, 2, Loop×25(3,4,5), 6]`.
- Design the engine **recursively from day one** (nesting a loop inside a
  loop just works), even if the UI only allows one level of nesting
  initially. That keeps phase 3 UI-only.

## 2. The four hard parts

### a. `MouseWatcher` — multi-target safety (most important change)

Today the KWin script embeds a single `tx, ty`; any position >3px from it
means "user moved, stop." With the pointer jumping between targets, the
worker's own moves would instantly false-trigger this.

**Fix:** embed the *set* of all step positions into the script template
(generate a JS array `var targets = [[x1,y1],[x2,y2],…]`). Movement counts
as user input only when the cursor is >tolerance from **every** target. The
worker only ever moves to positions in the set, so it can never trip itself;
a human grabbing the mouse produces events at positions outside all circles
within milliseconds. `MouseWatcher::start(QList<QPoint> targets)` instead of
`start(QPoint)`.

### b. `ClickWorker` — from rate loop to interpreter

Replace `run(target, desktop, rate, maxClicks)` with a recursive walker over
the `Sequence`:

```cpp
bool execNodes(const QList<SeqNode> &nodes);  // false = stop requested
// Step:  interruptible-wait(delayAfterMs) → moveTo → click → count++
// Loop:  for (i < iterations) execNodes(children)
// Main:  while (!stopped && loopsLeft--) execNodes(m_sequence);
```

- Reuse the existing mutex/condvar pattern for **interruptible** waits — a
  25-iteration sub-loop with 2s delays must still stop instantly when the
  mouse moves.
- The fixed-rate/scheduled-catch-up logic and live `setRate()` go away
  (delays are per-step now; disable editing while running).
- Add progress reporting back to the UI (thread-safe callback or
  `QMetaObject::invokeMethod`): total clicks, current loop iteration,
  current step index.

### c. UI — step list editor

Replace the Target/Rate form with a `QTreeWidget` (columns: `#`, `X`, `Y`,
`Delay`, `Repeats`, indented rows for loop groups) plus buttons:

- **Add step** → opens the existing `TargetPicker` (fully reusable as-is)
- **Edit / Remove / Move up / Move down**
- **Group into loop ×N** (on selected contiguous steps) / **Ungroup**
- A small details form (or double-click dialog) for delay/repeat of the
  selected step
- **Stop after:** switch from "N clicks" to "N loops **or** N clicks"
  (0 = unlimited each)

Also: `setFixedSize(sizeHint())` has to go — the window becomes resizable
with a sensible minimum. The single-target workflow remains the degenerate
case (a 1-step sequence), so nothing is lost.

### d. Persistence

Save the sequence as JSON (`QJsonDocument`) under a QSettings key on close,
load on start. Note: the README today *intentionally* doesn't save the
target — for grind macros the sequence should survive restarts, so this
changes deliberately. Coordinates depend on the monitor layout; show a
warning if loaded positions fall outside the current desktop.

## 3. Phased implementation

### Phase 1 — Engine, invisible to the user

Validates the risky parts with zero UI risk.

1. New `src/Sequence.h/.cpp`: structs + JSON (de)serialization.
2. Rewrite `ClickWorker` as the recursive interpreter.
3. `MouseWatcher::start(QList<QPoint>)` with the target-set script.
4. `MainWindow` keeps its single-target UI but wraps it as a 1-step sequence.

**Done when the app behaves exactly like today** and `--test-click` still
works.

### Phase 2 — Multi-step UI

Step list, add/edit/remove/reorder via picker, per-step delay & repeat,
progress display ("Loop 3 · Step 5/8 · Clicks: 1234"), resizable window,
stop-after-loops/clicks.

### Phase 3 — Sub-loops

Group/ungroup UI in the tree (engine already supports it from phase 1).

### Phase 4 — Persistence & polish

JSON save/load, then optional niceties in priority order:

- delay-only "wait" steps (game cooldowns)
- right-click per step
- named profiles (save/load multiple macros)
- panic hotkey via KWin's `registerShortcut` in the watcher script
- update the German translation (`update_translations`)
- README/docs/screenshots

### Testing

The interpreter is pure logic — factor it so it can run against a fake
pointer and add a small CTest target for the edge cases (empty group,
`repeat=0`, iteration counting, stop mid-wait). Also extend the `--test-*`
pattern in `main.cpp` with `--test-sequence file.json` so a macro can be
dry-run from a terminal.

## 4. Decisions

| Decision | Choice |
|---|---|
| Nesting depth in UI | One level now (covers the main scenario), recursive engine; enable deeper nesting later if ever needed |
| Delay semantics | **After** the click ("click, wait 2s for animation, next step"), ms granularity |
| Live-editing while running | Disabled for MVP — moving the mouse stops everything anyway, so restart is cheap |
| Save behavior | Auto-save last sequence (phase 4); named profiles as a stretch goal |

## Risks / gotchas specific to this codebase

- **Don't break the stop-on-mouse-move flow** — it's the whole safety story.
  The watcher-set change (2a) is the piece to get right; test with long
  delays between steps.
- The status-bar labels use pre-computed minimum widths for fixed sizing —
  the new loop/step label needs one too.
- New UI strings need
  `cmake --build build --target update_translations` + German translations
  to keep i18n complete.
- `beginClicking()`'s "watch first, then click" ordering stays — the watcher
  must know all targets before the first uinput move.
