# Ticket 04: Multi-Step Sequence Editor UI (Phase 2)

**Phase:** 2 (Multi-step flat sequence UI — sub-loop grouping comes in Ticket 05)  
**Depends on:** Tickets 01, 02, 03  
**Read first:** `tickets/00-README.md`, `PLAN.md`, `src/Sequence.h`, `src/ClickWorker.h`, `src/MainWindow.h`, `src/MainWindow.cpp`, `src/TargetPicker.h`, `src/main.cpp`

## Goal

Transform `MainWindow` from a fixed-size single-target form into a **resizable
multi-step sequence editor** backed by `Sequence m_sequence`. Users can add
multiple click locations with `TargetPicker`, re-pick coordinates for an existing
step, edit each step's label / delay / repeat count, reorder or delete steps, set
`Stop after: N loops` and/or `N clicks`, and see live loop/step/click progress in
the status bar.

Do **not** add *Group into loop* / *Ungroup* buttons yet (Ticket 05), though
`populateTree()` should already render `SeqNode::Type::Loop` rows cleanly if one
is loaded via `setSequenceDocument(...)`.

---

## Files to create / modify

- **Modify:** `src/MainWindow.h`, `src/MainWindow.cpp`
- **Modify:** `src/main.cpp` (update `--screenshot` for the new `MainWindow` API)

---

## Exact specification

### 1. Data model ownership in `MainWindow`

- Replace `std::optional<QPoint> m_target`, `m_rateSpin`, `m_perSecond`, `m_perMinute`, `m_selectButton`, `m_targetInline`, `m_targetLabel` with `Sequence m_sequence;` as the **single source of truth**.
- Provide public methods on `MainWindow` for CLI / tests / future persistence:
  ```cpp
  void addStep(QPoint pos);
  void setSequenceDocument(const SequenceDocument &doc);
  SequenceDocument sequenceDocument() const;
  ```
- Update `src/main.cpp`'s `--screenshot` block:
  - `--screenshot <out.png> [X Y]` calls `w.addStep(QPoint(x, y))` if `X Y` are passed (keeping backwards compatibility with existing README screenshot invocation!).

### 2. Window layout & controls (`MainWindow.cpp`)

Remove `setFixedSize(sizeHint())`! Instead set a sensible minimum size (e.g.
`setMinimumSize(520, 420)` and `resize(580, 480)` by default) and enable the
status bar size grip or let window decorations resize cleanly.

Build a clean vertical layout matching the existing spacing constants
(`margin = 12`, `gap = 8`, `groupGap = 16`):

1. **Sequence `QTreeWidget` (`m_tree`, stretch = 1):**
   - Columns:
     0. `#` (pre-order `1..N` leaf step number; right-aligned or centered, compact width)
     1. `Label` (step label, or `tr("Loop ×%1").arg(iterations)` for loop rows; stretches)
     2. `X` (`"%1 px"`)
     3. `Y` (`"%1 px"`)
     4. `Delay` (`"%1 ms"` for steps, empty for loop rows)
     5. `Repeats` (`"%1 ×"` for steps, `"%1 ×"` for loop rows)
   - Configure `m_tree`:
     - `setRootIsDecorated(true)` (ready for sub-loops in Ticket 05, or `false` when flat and `true` when any loop exists — keeping `true` or `setUniformRowHeights(true)` is clean and consistent)
     - `setAlternatingRowColors(true)`
     - `setAllColumnsShowFocus(true)`
     - `setSelectionBehavior(QAbstractItemView::SelectRows)`
     - `setSelectionMode(QAbstractItemView::SingleSelection)` (Ticket 05 upgrades this to `ExtendedSelection`)

2. **Step toolbar row (`QHBoxLayout` of `QPushButton`s):**
   - **Add step** (`tr("Add step")`): sets `m_pickMode = PickMode::AddStep` and calls `m_picker->pick()`. When `TargetPicker::targetSelected(pos)` fires, inserts a new `ClickStep{pos, 200, 1, {}}` right after the currently selected row (or at the end if nothing is selected), refreshes the tree, and selects the newly inserted row.
   - **Re-pick** (`tr("Re-pick")`): enabled only when a single `Step` row is selected. Sets `m_pickMode = PickMode::RepickStep` and calls `m_picker->pick()`. Updates only `step.pos = pos` on that step (preserving its `label`, `delayAfterMs`, `repeat`).
   - **Remove** (`tr("Remove")`): enabled when a row is selected; removes it and selects a neighbor row.
   - **Move up** (`tr("Move up")`) / **Move down** (`tr("Move down")`): enabled when the selected row has a previous / next sibling in its parent list; swaps them and preserves selection.
   - Also connect `TargetPicker::cancelled` to reset `m_pickMode = PickMode::None`.

3. **Selected-step details editor (`QGroupBox` or compact `QHBoxLayout` / `QFormLayout`):**
   Enabled only when a `Step` row is selected (and `m_state == State::Idle`):
   - `Label:` `QLineEdit` (`setPlaceholderText(tr("Optional description"))`)
   - `Delay:` `QSpinBox` (`setRange(0, 3'600'000)`, `setSingleStep(50)`, `setSuffix(tr(" ms"))`, default `200`)
   - `Repeats:` `QSpinBox` (`setRange(1, 100'000)`, `setSuffix(tr(" ×"))`, default `1`)
   - Editing any of these immediately updates the corresponding `SeqNode` in `m_sequence` and refreshes that `QTreeWidgetItem`'s column texts **without resetting selection or triggering recursive `valueChanged` feedback loops** (`QSignalBlocker` when loading selection into the spinboxes/lineedit!).

4. **Stop limits & Jitter options section (`QFormLayout`):**
   - Row 1 (`tr("Stop after:")`): two spinboxes on one clean row:
     - `m_maxLoops` (`QSpinBox`, range `0..1'000'000'000`, `0` = unlimited) + `QLabel(tr("loops"))`
     - `m_maxClicks` (`QSpinBox`, range `0..1'000'000'000`, `0` = unlimited) + `QLabel(tr("clicks (0 = unlimited)"))`
   - Row 2 (`tr("Jitter:")`): pixel jitter checkbox + low/high tick dwell range on one row:
     - `m_jitterCheck` (`QCheckBox(tr("Move 1–2 px, hold for"))`)
     - `m_jitterMinTicks` (`QSpinBox`, range `1..100'000`, default `4`)
     - `QLabel(tr("to"))`
     - `m_jitterMaxTicks` (`QSpinBox`, range `1..100'000`, default `12`)
     - `QLabel(tr("ticks"))`
     - Couple the two spinboxes cleanly (using `QSignalBlocker`): raising `m_jitterMinTicks` above `m_jitterMaxTicks` bumps `m_jitterMaxTicks` to match; lowering `m_jitterMaxTicks` below `m_jitterMinTicks` lowers `m_jitterMinTicks` to match.
     - Enable `m_jitterMinTicks` and `m_jitterMaxTicks` iff `m_state == State::Idle && m_jitterCheck->isChecked()`.
     - Include `JitterConfig` in `sequenceDocument()` and `setSequenceDocument(...)`.

5. **Start / Countdown / Clicking action stack (`m_actionStack`):**
   - Keep the exact existing `m_startButton` + `m_banner` (`CountdownSeconds = 2`, `#8a6d00` amber countdown banner, `#b3261e` red clicking banner)!
   - `m_startButton` is enabled in `State::Idle` iff `hasExecutableSteps(m_sequence) && m_pointer->isValid()`.

6. **Status bar:**
   - Left (`m_status`, stretch 1):
     - On startup with empty sequence (and valid pointer): `tr("No steps added yet")`
     - When ≥1 executable step exists in `State::Idle`: `tr("Ready")` (unless preserving the last stop reason until edited/restarted, matching existing behavior!).
   - Permanent right label 1 (`m_progressLabel`):
     - Pre-compute `setMinimumWidth` using `fontMetrics().horizontalAdvance(tr("Loop %1 · Step %2/%3").arg(88888).arg(888).arg(888))`.
     - While `State::Clicking`: shows `tr("Loop %1 · Step %2/%3").arg(p.currentLoop).arg(p.currentStep).arg(p.totalSteps)`.
     - While `State::Idle` / `State::Countdown`: shows `tr("Steps: %1").arg(leafStepCount(m_sequence))` (or hidden when `0` steps).
   - Permanent right label 2 (`m_clickCount`):
     - Keep `tr("Clicks: %1").arg(count)` with pre-computed minimum width.

### 3. State locking & starting (`MainWindow::setState` / `beginClicking` / `onWatcherReady`)

When `state != State::Idle`, disable **all** editor widgets:
`m_tree`, Add/Re-pick/Remove/MoveUp/MoveDown buttons, step detail fields (`Label`, `Delay`, `Repeats`), `m_maxLoops`, `m_maxClicks`, `m_jitterCheck`, `m_jitterMinTicks`, and `m_jitterMaxTicks`.

In `MainWindow::beginClicking()`, widen `MouseWatcher` tolerance by `JitterConfig::MaxOffsetPx` whenever jitter is enabled so `±2 px` jittered clicks never self-trip mouse-movement detection:
```cpp
const int tolerance = m_jitterCheck->isChecked()
    ? (MouseWatcher::DefaultTolerance + JitterConfig::MaxOffsetPx)
    : MouseWatcher::DefaultTolerance;
if (!m_watcher->start(collectTargets(m_sequence), tolerance)) { ... }
```

When `onWatcherReady()` starts `m_worker`, pass `m_maxLoops`, `m_maxClicks`, and `JitterConfig`:
```cpp
const JitterConfig jitter {
    m_jitterCheck->isChecked(),
    m_jitterMinTicks->value(),
    m_jitterMaxTicks->value()
};
m_worker->start(m_sequence, desktopGeometry(),
                static_cast<quint64>(m_maxLoops->value()),
                static_cast<quint64>(m_maxClicks->value()),
                jitter);
```

---

## Advice & pitfalls for the implementing agent

- **Always use `QSignalBlocker` when populating detail controls from the selected tree item!** Without `QSignalBlocker`, changing tree selection triggers `QSpinBox::valueChanged` / `QLineEdit::textChanged` with stale values and overwrites the newly selected step!
- **Mapping `QTreeWidgetItem*` ↔ `SeqNode`:** Store tree path indices in `item->data(0, Qt::UserRole)` (e.g. top-level index `i`, and in Ticket 05 child index `j`, or rebuild `m_tree` cleanly via `rebuildTree()` while restoring the selected path). A small `struct NodePath { int topIndex = -1; int childIndex = -1; };` makes both flat lists (Ticket 04) and 1-level sub-loops (Ticket 05) dead simple and bug-free!
- Do **not** hide `MainWindow` when calling `m_picker->pick()` (`PickerOverlay` uses `LayerOverlay` above all windows; hiding/showing `MainWindow` makes KWin lose its window position).

## Definition of Done

```bash
cmake -B build
cmake --build build -j
ctest --test-dir build --output-on-failure
QT_QPA_PLATFORM=offscreen ./build/autoclicker --screenshot /tmp/opencode/phase2.png 400 300
```

All commands exit `0` with **zero compiler warnings**, and `/tmp/opencode/phase2.png` renders the new sequence editor cleanly.
