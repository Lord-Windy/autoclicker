# Ticket 05: Sub-Loops Group / Ungroup UI & Active Step Highlighting (Phase 3)

**Phase:** 3 (Sub-loops UI — engine support already exists from Ticket 02)  
**Depends on:** Tickets 01–04  
**Read first:** `tickets/00-README.md`, `PLAN.md`, `src/Sequence.h`, `src/MainWindow.h`, `src/MainWindow.cpp`

## Goal

Enable **1-level sub-loops** in `MainWindow`'s sequence tree so users can select
contiguous top-level steps (e.g. steps 3–5) and group them into `Loop ×25`,
edit the loop's iteration count, add/move/remove steps inside an existing loop,
ungroup a loop back into top-level steps, and visually see which leaf step is
currently executing while `State::Clicking`.

---

## Files to create / modify

- **Modify:** `src/MainWindow.h`, `src/MainWindow.cpp`
- **Optional:** Add tree/sequence manipulation helper functions into `src/Sequence.h/.cpp` (with unit tests in `tests/test_sequence.cpp`!) so grouping, ungrouping, moving, and inserting are unit-tested independently of `QTreeWidget`!

---

## Exact specification

### 1. Selection & toolbar buttons (`MainWindow`)

- Switch `m_tree->setSelectionMode(QAbstractItemView::ExtendedSelection)` so the user can Shift-click or Ctrl-click steps 3–5.
- Add two buttons to the step toolbar:
  - **Group into loop** (`tr("Group into loop")`)
  - **Ungroup** (`tr("Ungroup")`)

### 2. Exact enablement & behavior rules (enforcing 1-level UI nesting)

Putting pure sequence-editing helpers in `src/Sequence.h/.cpp` (and testing them in `tests/test_sequence.cpp`) is **strongly recommended**!

- **Group into loop:**
  - **Enabled iff:** `m_state == State::Idle` AND ≥1 items are selected AND **every** selected item is a **top-level `SeqNode::Type::Step`** (not a `Loop` row, and not already a child inside a `Loop`) AND their top-level indices form a **contiguous range** `[first..last]` (e.g. `{2, 3, 4}`, not `{1, 4}`).
  - **Action:** Replaces `m_sequence[first..last]` with a single `SeqNode::makeLoop(2, extractedSteps)` (default `2` iterations — or keep focus on the iterations spinbox so the user can immediately type `25`), expands the new `Loop` item in `m_tree`, and selects the new `Loop` row.
  - Notice how requiring *top-level `Step` rows only* guarantees the UI can never create nested loops (`Loop` inside `Loop`), cleanly enforcing Decision 1 without error popups!

- **Ungroup:**
  - **Enabled iff:** `m_state == State::Idle` AND exactly **one** item is selected AND it is a top-level `SeqNode::Type::Loop`.
  - **Action:** Replaces that `Loop` node at index `i` with its `children` spliced directly into `m_sequence` at index `i..i + children.size() - 1`, and selects those ungrouped rows.

- **Add step / Re-pick / Remove / Move up / Move down with loops & multi-selection:**
  - **Add step:**
    - If a single top-level `Loop` row is selected → append the new `Step` at the end of that `Loop`'s `children` (and expand the loop).
    - If a single child `Step` inside a `Loop` is selected → insert the new `Step` right after that child *inside the same `Loop`*.
    - If a single top-level `Step` is selected → insert at top level right after it.
    - Otherwise (nothing or multiple items selected) → append at the end of top-level `m_sequence`.
  - **Re-pick:** Enabled only when **exactly one** `Step` row (top-level or inside a loop) is selected.
  - **Remove:** Works for 1 or more selected items! Watch out when removing both a `Loop` and one of its children in a multi-selection, or removing the **last remaining child** inside a `Loop`:
    - If removing children leaves a `Loop` with `children.isEmpty()`, automatically remove the now-empty `Loop` wrapper too so empty dead loops aren't left behind in the UI!
  - **Move up / Move down:** Enabled when **exactly one** item is selected (either a top-level `Step`/`Loop`, or a child `Step` inside a `Loop`) and it has a previous / next sibling within its **same parent list**.

### 3. Details panel for `Loop` vs `Step`

Use a `QStackedWidget` (or show/hide widgets) in the details editor area:
- **No selection or multi-selection (>1 rows):** Disable detail inputs (or show a subtle hint).
- **Single `Step` selected:** Show/enable `Label`, `Delay (ms)`, `Repeats (×)` (same as Ticket 04).
- **Single `Loop` selected:** Show/enable `Loop iterations:` `QSpinBox` (`setRange(1, 100'000)`, `setSuffix(tr(" ×"))`), updating `loopNode.iterations` and refreshing the `Loop` row's text (`tr("Loop ×%1").arg(iterations)` in `Label` column and `"%1 ×"` in `Repeats` column).

### 4. Pre-order `1..N` leaf numbering & active-step highlight

- **Column 0 (`#`):**
  - Every `Type::Step` row (whether top-level or indented inside a `Loop`) displays its sequential pre-order number `1, 2, 3, …, N`!
  - Every `Type::Loop` header row displays `"↻"` (or empty `""`) in column 0 so loop headers never consume a step number.
  - Always keep `Loop` items expanded (`item->setExpanded(true)`) when populating the tree.
- **Active step highlight while `State::Clicking`:**
  - In `MainWindow::poll()`, read `m_worker->progress().currentStep` (`1..totalSteps`) and visually indicate that leaf row in `m_tree` (e.g. `m_tree->setCurrentItem(activeLeafItem)` while `m_tree` is disabled, or bold font / subtle highlight marker `"▶ %1"` in column `#`, clearing it in `stopClicking()`).

---

## Advice & pitfalls for the implementing agent

- **Put grouping/ungrouping/removal logic into pure functions on `Sequence` in `src/Sequence.h/.cpp` and test them in `tests/test_sequence.cpp`!** Manipulation of indices on multi-select delete or group/ungroup is trivial in pure `QList<SeqNode>` functions and 10× easier to test with `QCOMPARE` than through `QTreeWidgetItem` pointers.
- Remember `QSignalBlocker` on `m_tree` and on the loop `iterations` spinbox whenever syncing UI state from `m_sequence`.
- Update `main.cpp` `--screenshot` (optional enhancement): if no extra args or `X Y` are passed, keep Ticket 04's behavior so existing invocations keep working.

## Definition of Done

```bash
cmake -B build
cmake --build build -j
ctest --test-dir build --output-on-failure
QT_QPA_PLATFORM=offscreen ./build/autoclicker --screenshot /tmp/opencode/phase3.png 400 300
```

All commands exit `0` with **zero compiler warnings**, and `test_sequence` covers group/ungroup/auto-prune-empty-loop edge cases.
