# Autoclicker Sequence Workflow — Guide for Implementing Agents

This folder breaks [`../PLAN.md`](../PLAN.md) into **6 sequential, self-contained
tickets**. Complete them strictly in numerical order (`01` → `06`). Every ticket
leaves the repository compiling cleanly with zero warnings and all tests passing.

## Ticket order

| Ticket | Phase | Summary |
|---|---|---|
| [`01-sequence-model-and-json.md`](01-sequence-model-and-json.md) | Phase 1a | `src/Sequence.h/.cpp` data model (`SeqNode`, `JitterConfig`), helpers, versioned JSON, CTest setup + `tests/test_sequence.cpp` |
| [`02-pointer-sink-and-click-worker.md`](02-pointer-sink-and-click-worker.md) | Phase 1b | `src/PointerSink.h`, recursive `ClickWorker` rewrite (incl. `[min, max]` tick jitter), 1-step `MainWindow` adapter + `tests/test_click_worker.cpp` |
| [`03-multi-target-mouse-watcher.md`](03-multi-target-mouse-watcher.md) | Phase 1c | Multi-target `MouseWatcher` (`QList<QPoint>` + configurable jitter tolerance), `StopReason` wiring in `MainWindow` (completes Phase 1) |
| [`04-multi-step-editor-ui.md`](04-multi-step-editor-ui.md) | Phase 2 | Multi-step `QTreeWidget` editor UI, Add/Re-pick/Remove/Move, per-step fields, `maxLoops`, Jitter UI controls, resizable window |
| [`05-sub-loops-ui.md`](05-sub-loops-ui.md) | Phase 3 | Sub-loop Group/Ungroup UI (`ExtendedSelection`), loop iteration editing, active-step highlight |
| [`06-persistence-cli-and-polish.md`](06-persistence-cli-and-polish.md) | Phase 4 | Auto-save `sequence.json`, multi-monitor warning, `--test-sequence` CLI, German `.ts` translations, `README.md` |

---

## Global rules & C++ / Qt 6 guardrails (READ BEFORE EVERY TICKET)

1. **Always read existing files before editing.** This codebase is small, clean,
   modern C++20 + Qt 6 (`Widgets`, `DBus`, `LinguistTools`, `Test`) with
   `-Wall -Wextra` enabled in `CMakeLists.txt`. Match the existing style:
   4-space indentation, `m_` member prefix, `QStringLiteral(...)` for non-translated
   literals, `tr(...)` for all user-facing UI strings.
2. **Zero compiler warnings allowed (`-Wall -Wextra`).**
   - Keep constructor member initializer lists in the **exact declaration order**
     shown in the `.h` file (`-Wreorder`).
   - Watch signed vs unsigned comparisons (`int`, `qsizetype`, `quint64`). Note
     that in Qt 6, `QList::size()` returns `qsizetype` (signed 64-bit), not `int`.
   - Handle all enum values explicitly in `switch` statements.
3. **NEVER use `<cassert>` / `assert()` in tests.**
   `CMakeLists.txt` defaults `CMAKE_BUILD_TYPE` to `Release`, which defines
   `-DNDEBUG` and strips `assert()` out completely! Always use `Qt6::Test`
   (`QVERIFY`, `QCOMPARE`).
4. **`CMAKE_AUTOMOC` rule for test `.cpp` files:**
   Whenever you define a `QObject` subclass (such as a `QTest` class) inside a
   `tests/test_*.cpp` file without a separate header, put
   `#include "test_<name>.moc"` on the **very last line** of that `.cpp` file.
   Without it, linking will fail with `undefined reference to vtable`.
5. **NEVER `install(...)` test targets or helper static libraries.**
   `packaging/build-appimage.sh` runs `DESTDIR="$APPDIR" cmake --install` and
   expects only `autoclicker` to be installed. Gate all tests inside
   `if(BUILD_TESTING) ... endif()` after `include(CTest)`.
6. **NEVER run `--test-click`, `--test-move`, or `--test-sequence --live` in an
   unattended workflow.** Those flags open `/dev/uinput` and physically move and
   click the host machine's real cursor! Use `ctest --test-dir build --output-on-failure`,
   `QT_QPA_PLATFORM=offscreen ./build/autoclicker --screenshot ...`, and (in
   Ticket 06) `--test-sequence <file.json>` (which defaults to dry-run).
7. **Wayland / KWin rules:**
   - Do **not** hide and re-show `MainWindow` when opening `TargetPicker` — on
     Wayland, KWin will place the window in a new position (see comment in
     `MainWindow.cpp`).
   - Do **not** use `MainWindow::geometry()` or `frameGeometry()` to check
     whether targets overlap `MainWindow` — Wayland does not expose global
     top-level window coordinates to Qt clients (`topLeft()` is `(0, 0)`).
   - Always preserve the **watch first, then click** startup sequence
     (`m_watcher->start(...)` → wait for `MouseWatcher::ready` →
     `m_worker->start(...)`).
8. **UI typography convention:**
   Existing strings in `MainWindow.cpp` and `i18n/autoclicker_de.ts` use real
   Unicode characters: ellipsis `…` (`U+2026`), en-dash `–` (`U+2013`), and
   multiplication sign `×` (`U+00D7`). Stay consistent with them.

## Standard build & test commands

Run these after your changes in every ticket:

```bash
cmake -B build
cmake --build build -j
ctest --test-dir build --output-on-failure
```
