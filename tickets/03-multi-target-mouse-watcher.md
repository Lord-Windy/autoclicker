# Ticket 03: Multi-Target `MouseWatcher` & Phase 1 Completion

**Phase:** 1c (Completes Phase 1 — app still looks identical to v1.0.0)  
**Depends on:** Ticket 01, Ticket 02  
**Read first:** `tickets/00-README.md`, `src/MouseWatcher.h`, `src/MouseWatcher.cpp`, `src/MainWindow.cpp`, `src/main.cpp`

## Goal

Update `MouseWatcher` so its KWin script accepts a **list of target positions**
and a configurable **pixel tolerance** (default `3` px, bumped to `5` px when
`±2 px` jitter is active so jittered clicks + Wayland rounding never
false-trigger `userActivity`). Wire `MainWindow` to pass `collectTargets(seq)`
to `MouseWatcher` and report accurate `ClickWorker::StopReason` messages in
`MainWindow::poll()`.

---

## Files to create / modify

- **Modify:** `src/MouseWatcher.h`, `src/MouseWatcher.cpp`
- **Modify:** `src/MainWindow.h`, `src/MainWindow.cpp`
- **Optional / Recommended:** Extract the JS target-list formatting helper (or put a small unit test in `tests/test_sequence.cpp` or a new `tests/test_watcher_script.cpp`) to verify script generation.

---

## Exact specification

### 1. `MouseWatcher` (`src/MouseWatcher.h` & `src/MouseWatcher.cpp`)

- Update `MouseWatcher` public methods and constant:
  ```cpp
  static constexpr int DefaultTolerance = 3;

  bool start(const QList<QPoint> &targets, int tolerance = DefaultTolerance);
  bool start(QPoint target, int tolerance = DefaultTolerance)
  {
      return start(QList<QPoint>{target}, tolerance);
  }
  ```
  **Keep the `start(QPoint, int)` overload!** `src/main.cpp`'s `testMove()` calls `watcher->start({-1000000, -1000000})`. With `{int, int}`, C++ will bind cleanly to `start(QPoint, int)` instead of failing overload resolution on `QList<QPoint>`.
  *(Why configurable `tolerance` matters: KWin's `workspace.cursorPos` is floating-point and uinput→Wayland coordinate quantization can already differ by ~1–2 px. When `±2 px` jitter is enabled in Ticket 04, passing `DefaultTolerance + JitterConfig::MaxOffsetPx` (`5 px`) guarantees jittered clicks can never false-trigger `scriptUserMoved`.)*

- Deduplicate `targets` inside `MouseWatcher::start(const QList<QPoint> &targets, int tolerance)` before building the JS array string so a 50-step macro that clicks the same 3 spots only embeds 3 `[x,y]` pairs in the KWin script:
  ```cpp
  QStringList jsPoints;
  QList<QPoint> uniqueTargets;
  for (const QPoint &pt : targets) {
      if (!uniqueTargets.contains(pt)) {
          uniqueTargets.append(pt);
          jsPoints.append(QStringLiteral("[%1,%2]").arg(pt.x()).arg(pt.y()));
      }
  }
  const QString targetsJs = QStringLiteral("[") + jsPoints.join(QLatin1Char(',')) + QStringLiteral("]");
  ```

- Update `ScriptTemplate` in `src/MouseWatcher.cpp`:
  ```javascript
  var svc = "%1", path = "%2", iface = "local.autoclicker.Watcher";
  var targets = %3, tol = %4;
  var first = true, reported = false;
  function fmt(p) { return String(Math.round(p.x)) + "," + String(Math.round(p.y)); }
  function nearAnyTarget(p) {
      for (var i = 0; i < targets.length; ++i) {
          var t = targets[i];
          if (Math.abs(p.x - t[0]) <= tol && Math.abs(p.y - t[1]) <= tol)
              return true;
      }
      return false;
  }
  workspace.cursorPosChanged.connect(function () {
      var p = workspace.cursorPos;
      if (first) {
          first = false;
          callDBus(svc, path, iface, "scriptHomed", fmt(p));
      }
      if (!reported && !nearAnyTarget(p)) {
          reported = true;
          callDBus(svc, path, iface, "scriptUserMoved", fmt(p));
      }
  });
  callDBus(svc, path, iface, "scriptReady", fmt(workspace.cursorPos));
  ```
  **Check the boolean math carefully:**
  - Inside target `i`: **both** `|p.x - t[0]| <= tol` **AND** `|p.y - t[1]| <= tol`.
  - User moved: `!nearAnyTarget(p)` (cursor is outside every target's `±3 px` box).
  - If `targets` is empty `[]`, `nearAnyTarget(p)` returns `false`, so any movement immediately reports user activity.

### 2. `MainWindow` updates (`src/MainWindow.h` & `src/MainWindow.cpp`)

- Build the current `Sequence` in a helper method `Sequence buildCurrentSequence() const` (still a 1-step sequence from `m_target` and `clicksPerSecond()` in Phase 1; Ticket 04 will replace this with the multi-step editor's `m_sequence`).
- In `MainWindow::beginClicking()`:
  Pass `collectTargets(buildCurrentSequence())` to `m_watcher->start(...)` (preserving the "watch first, then click" rule).
- In `MainWindow::poll()`:
  Replace the hardcoded `stopClicking(tr("Click limit reached"))` when `!m_worker->isRunning()` with a switch on `m_worker->stopReason()`:
  - `StopReason::ClickLimitReached` → `tr("Click limit reached")`
  - `StopReason::LoopLimitReached` → `tr("Loop limit reached")`
  - `StopReason::EmptySequence` → `tr("Nothing to run")`
  - `StopReason::Stopped` / `StopReason::None` → `tr("Stopped")`

---

## Advice & pitfalls for the implementing agent

- Watch `QString::arg(...)` chaining! Never chain `.arg(userString).arg(number)` when `userString` could contain `%1`, though here `bus.baseService()`, `ObjectPath`, `targetsJs`, and `tolerance` are safe. Using multi-arg `.arg(bus.baseService(), ObjectPath, targetsJs, QString::number(std::max(0, tolerance)))` is cleaner and faster in Qt 6.
- Do **not** run `./build/autoclicker --test-move` or `--test-click` automatically during verification (they move the real cursor). Verify compilation and run `ctest` + offscreen `--screenshot`.

## Definition of Done

```bash
cmake -B build
cmake --build build -j
ctest --test-dir build --output-on-failure
QT_QPA_PLATFORM=offscreen ./build/autoclicker --screenshot /tmp/opencode/phase1.png 400 300
```

All commands exit `0` with **zero compiler warnings**. Phase 1 is now complete!
