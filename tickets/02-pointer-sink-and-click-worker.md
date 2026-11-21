# Ticket 02: `PointerSink` Interface & Recursive `ClickWorker` Interpreter

**Phase:** 1b (Engine rewrite + unit tests; UI remains unchanged)  
**Depends on:** Ticket 01  
**Read first:** `tickets/00-README.md`, `src/Sequence.h`, `src/VirtualPointer.h`, `src/VirtualPointer.cpp`, `src/ClickWorker.h`, `src/ClickWorker.cpp`, `src/MainWindow.h`, `src/MainWindow.cpp`

## Goal

1. Extract a lightweight `PointerSink` interface (`src/PointerSink.h`) implemented by `VirtualPointer` so tests can inject a `FakePointer` without opening `/dev/uinput`.
2. Rewrite `ClickWorker` from a fixed-rate single-point loop into a thread-safe recursive `Sequence` interpreter supporting `ClickStep::repeat`, arbitrarily nested `SeqNode::Type::Loop`, `maxLoops`, `maxClicks`, `JitterConfig` (1–2 px random offset held for `[minTicks, maxTicks]` clicks), `StopReason`, `Progress`, and instant cancellation mid-delay.
3. Keep `MainWindow` compiling and working identically to v1.0.0 by wrapping its single `m_target` + `clicksPerSecond()` into a 1-step `Sequence` (removing live `setRate()` while clicking, per `PLAN.md`).
4. Add thorough CTest unit tests in `tests/test_click_worker.cpp`.

Do **not** change `MouseWatcher` or redesign `MainWindow`'s UI layout in this ticket.

---

## Files to create / modify

- **Create:** `src/PointerSink.h`
- **Modify:** `src/VirtualPointer.h`
- **Modify:** `src/ClickWorker.h`, `src/ClickWorker.cpp`
- **Modify:** `src/MainWindow.cpp` (minimal 1-step sequence adapter)
- **Create:** `tests/test_click_worker.cpp`
- **Modify:** `CMakeLists.txt`

---

## Exact specification

### 1. `src/PointerSink.h` & `VirtualPointer`

```cpp
#pragma once

#include <QPoint>
#include <QRect>

#include <chrono>

class PointerSink
{
public:
    virtual ~PointerSink() = default;
    virtual void moveTo(QPoint pos, const QRect &desktop) = 0;
    virtual void click(std::chrono::microseconds hold) = 0;
};
```

Update `VirtualPointer` in `src/VirtualPointer.h` to inherit `public PointerSink` and mark `moveTo` and `click` with `override`. Do not alter `VirtualPointer.cpp`'s uinput logic.

### 2. `ClickWorker` API (`src/ClickWorker.h`)

Keep `ClickWorker` completely free of `QObject` (pure standard C++ threading + Qt value types `QPoint`/`QRect`/`Sequence`):

```cpp
#pragma once

#include "Sequence.h"

#include <QRect>

#include <atomic>
#include <chrono>
#include <condition_variable>
#include <mutex>
#include <optional>
#include <thread>

class PointerSink;

class ClickWorker
{
public:
    enum class StopReason {
        None,
        Stopped,
        ClickLimitReached,
        LoopLimitReached,
        EmptySequence
    };

    struct Progress {
        quint64 totalClicks = 0;
        quint64 currentLoop = 0; // 1-based main loop counter while running
        int currentStep = 0;     // 1-based pre-order leaf step index (1..totalSteps)
        int totalSteps = 0;      // leafStepCount(sequence)
    };

    explicit ClickWorker(PointerSink &pointer);
    ~ClickWorker();

    // maxLoops: 0 = unlimited. maxClicks: 0 = unlimited.
    void start(Sequence sequence, QRect desktop, quint64 maxLoops, quint64 maxClicks,
               JitterConfig jitter = {}, bool skipWaits = false);
    void stop();

    bool isRunning() const { return m_running; }
    quint64 clickCount() const { return m_count; }
    Progress progress() const;
    StopReason stopReason() const { return m_stopReason; }

private:
    // ...
};
```

### 3. Exact execution & timing algorithm (`src/ClickWorker.cpp`)

Follow this exact timing design so there are no off-by-one stalls, no 50 CPS regressions, and no 100%-CPU busy spins:

- **Pre-order leaf step numbering during execution:**
  Before starting the main loop, compute `totalSteps = leafStepCount(sequence)`.
  At the start of each main-loop pass (`loop = 1, 2, …`), reset a walk counter `int leafIndex = 0;` that increments by `1` each time `execNodes` visits a `SeqNode::Type::Step` (even if `step.repeat == 0`, so step indices always match `leafStepCount`!).
- **Pixel jitter with `[minTicks, maxTicks]` dwell (CRITICAL — read carefully):**
  - Maintain a `std::vector<StepJitterState>` of size `totalSteps` (`struct StepJitterState { QPoint offset {0, 0}; int ticksRemaining = 0; };`) plus a `std::mt19937 rng{std::random_device{}()};` local to `run(...)`.
  - **NEVER accumulate jitter onto the previous position (`pos += delta`)** — a cumulative random walk would drift off the target button and trip `MouseWatcher`! Always compute `step.pos + state.offset` anchored to `step.pos`.
  - When `jitter.enabled` is `false`: click `step.pos` directly.
  - When `jitter.enabled` is `true` (on leaf step `idx = thisStepIndex - 1`):
    1. If `state[idx].ticksRemaining <= 0`:
       - Build or pick from the 24 non-zero offsets `(dx, dy)` in `[-2, +2] × [-2, +2]` (`dx != 0 || dy != 0`, i.e., `1 <= max(|dx|, |dy|) <= 2`), **excluding `state[idx].offset`** so every re-roll visibly moves to a *different* pixel! (If `!desktop.isEmpty() && desktop.contains(step.pos)`, prefer candidates where `desktop.contains(step.pos + candidate)`.)
       - Roll `const int lo = std::max(1, jitter.minTicks); const int hi = std::max(lo, jitter.maxTicks);` using `std::uniform_int_distribution<int>(lo, hi)(rng)` and store in `state[idx].ticksRemaining`.
    2. Use `const QPoint clickPos = step.pos + state[idx].offset;` and decrement `--state[idx].ticksRemaining;`.
- **Pending deadline (`std::optional<Clock::time_point> nextActionDeadline`):**
  - Starts as `std::nullopt` before the very first click of the run.
  - For a `Type::Step`:
    ```cpp
    ++leafIndex;
    const int thisStepIndex = leafIndex;
    for (int r = 0; r < step.repeat; ++r) {
        if (m_stopRequested) {
            m_stopReason = StopReason::Stopped;
            return false;
        }
        if (!skipWaits && nextActionDeadline.has_value()) {
            m_cv.wait_until(lock, *nextActionDeadline, [this] { return m_stopRequested.load(); });
            if (m_stopRequested) {
                m_stopReason = StopReason::Stopped;
                return false;
            }
        }

        m_currentStep = thisStepIndex;
        const auto stepStart = Clock::now();
        const int delayMs = std::max(0, step.delayAfterMs);
        const auto hold = std::chrono::milliseconds(std::clamp(delayMs / 4, 5, 50));
        const QPoint clickPos = resolveClickPos(step.pos, thisStepIndex - 1);

        lock.unlock();
        m_pointer.moveTo(clickPos, desktop);
        m_pointer.click(hold);
        lock.lock();

        const quint64 clicks = ++m_count;
        if (maxClicks > 0 && clicks >= maxClicks) {
            m_stopReason = StopReason::ClickLimitReached;
            return false;
        }
        if (m_stopRequested) {
            m_stopReason = StopReason::Stopped;
            return false;
        }

        const auto now = Clock::now();
        const auto targetNext = stepStart + std::chrono::milliseconds(delayMs);
        // Do not "catch up" if pointer.click() took longer than delayMs
        nextActionDeadline = (now > targetNext) ? now : targetNext;
    }
    return true;
    ```
  - For a `Type::Loop`:
    - Remember the `leafIndex` at the start of the loop body (`const int loopBaseLeaf = leafIndex;`), and at the start of **each iteration** reset `leafIndex = loopBaseLeaf;` so steps inside a sub-loop keep their consistent `1..totalSteps` numbers on iterations `2..N`!
    - Also handle `node.iterations <= 0`: if `node.iterations <= 0`, advance `leafIndex += leafStepCount(node.children)` so subsequent steps after the skipped loop still get their right `1..totalSteps` index!
    - If `m_stopRequested`, set `StopReason::Stopped` and return `false`.
- **Main loop & zero-work spin guard:**
  ```cpp
  if (!hasExecutableSteps(sequence)) {
      m_stopReason = StopReason::EmptySequence;
      m_running = false;
      return;
  }
  for (quint64 loop = 0; maxLoops == 0 || loop < maxLoops; ++loop) {
      if (m_stopRequested) {
          m_stopReason = StopReason::Stopped;
          break;
      }
      m_currentLoop = loop + 1;
      int leafIndex = 0;
      const quint64 clicksBefore = m_count;
      if (!execNodes(sequence, leafIndex))
          break;
      if (m_count == clicksBefore) {
          // Safety net against 100% CPU infinite loop
          m_stopReason = StopReason::EmptySequence;
          break;
      }
  }
  if (m_stopReason == StopReason::None)
      m_stopReason = StopReason::LoopLimitReached;
  m_running = false;
  ```
- Notice why this works so cleanly:
  - Click #1 happens **immediately** (no wait before first click).
  - Step 1's `delayAfterMs` is waited *before* Click #2, while `m_currentStep` still reads `1`.
  - When `maxClicks` or `maxLoops` finishes, execution stops **immediately** after the final click without waiting on a trailing delay.
  - `stop()` wakes `m_cv` and joins `m_thread`; if `m_stopReason == StopReason::None`, set it to `StopReason::Stopped`.

### 4. Temporary 1-step adapter in `MainWindow.cpp`

- Remove the `connect(m_rateSpin, &QSpinBox::valueChanged, ...)` live rate update lambda (and disable `m_rateSpin` when `!idle` in `MainWindow::setState`, alongside `m_perSecond`/`m_perMinute`).
- In `MainWindow::onWatcherReady()`:
  Build a 1-step `Sequence`:
  ```cpp
  ClickStep step;
  step.pos = *m_target;
  step.delayAfterMs = qMax(1, qRound(1000.0 / clicksPerSecond()));
  step.repeat = 1;
  Sequence seq { SeqNode::makeStep(step) };
  m_worker->start(std::move(seq), desktopGeometry(), 0,
                  static_cast<quint64>(m_maxClicks->value()));
  ```

### 5. Unit tests (`tests/test_click_worker.cpp` & `CMakeLists.txt`)

Add `src/ClickWorker.cpp` to `autoclicker_core` in `CMakeLists.txt` and register `test_click_worker` inside `if(BUILD_TESTING)`.

Create a `FakePointer : public PointerSink` in `tests/test_click_worker.cpp` that records `{ QPoint pos; std::chrono::microseconds hold; }` into a mutex-protected `QList` **without sleeping in `click()`**. Test all of the following:
1. **Per-step repeat vs sub-loop order:**
   `[Step(A, repeat=2, delay=5), Loop×3([Step(B, delay=5), Step(C, delay=5)]), Step(D, delay=5)]` with `maxLoops = 1` produces the exact click order `A, A, B, C, B, C, B, C, D` (9 clicks) and `StopReason::LoopLimitReached`.
2. **Nested loop inside a loop:**
   `[Loop×2([Step(A, delay=1), Loop×2([Step(B, delay=1)])])]` with `maxLoops = 1` produces `A, B, B, A, B, B` (6 clicks).
3. **`maxClicks` stops mid-sequence & skips trailing delay:**
   With `delayAfterMs = 5000` (5 seconds!) and `maxClicks = 2` on a 3-step sequence (Wait: step 1 delay is 5ms, step 2 delay is 5000ms), verify it stops in `< 200ms` right after click #2 with `StopReason::ClickLimitReached`!
4. **`maxLoops` skips trailing delay on final step:**
   `[Step(A, delay=5000)]` with `maxLoops = 1, maxClicks = 0` completes 1 click in `< 200ms` with `StopReason::LoopLimitReached`.
5. **Empty sequence / empty loop / `repeat=0` spin guard:**
   `[]`, `[Loop×25([])]`, and `[Step(A, repeat=0)]` with `maxLoops = 0` return immediately in `< 100ms` with `0` clicks and `StopReason::EmptySequence`.
6. **Interruptible mid-wait `stop()`:**
   Start `[Step(A, delay=5000), Step(B, delay=5000)]` with `maxLoops = 1`. Wait until `clickCount() == 1` (poll briefly for up to 200ms), then call `worker.stop()` and verify `stop()` returns in `< 200ms` (not 5000ms!) with `StopReason::Stopped` and `clickCount() == 1`.
7. **Step index numbering across loops:**
   Verify `progress().totalSteps` and `currentStep` stay within `1..totalSteps` even on iteration 2+ of a sub-loop or after a `Loop×0(...)` node.
8. **Jitter offset bounds & `[minTicks, maxTicks]` dwell:**
   Run `[Step(QPoint(500, 500), delay=50)]` against `desktop = QRect(0, 0, 1920, 1080)` with `maxLoops = 0, maxClicks = 40`, `JitterConfig{true, 3, 5}`, and `skipWaits = true`:
   - Verify all 40 recorded `QPoint`s satisfy `1 <= std::max(std::abs(p.x() - 500), std::abs(p.y() - 500)) <= 2` (every click shifted 1–2 px, never unshifted `(500, 500)`, never >2 px away).
   - Measure consecutive run-lengths of identical `QPoint`s across the 40 clicks. Because every re-roll picks a *different* offset from the previous one, **every completed run** (all runs except the final one cut off at click #40) is deterministically guaranteed to have length `>= 3` and `<= 5`, and there must be at least 7 distinct runs!

---

## Advice & pitfalls for the implementing agent

- **`m_mutex` deadlock hazard:** Never hold `m_mutex` while calling `m_pointer.moveTo(...)` or `m_pointer.click(...)` (`VirtualPointer::click` sleeps up to 50ms, and locking across it would block `stop()`!). Always `lock.unlock()` around pointer calls and `lock.lock()` afterwards, just like the original `ClickWorker.cpp` did.
- **`std::atomic` vs `Progress`:** Either make `m_count`, `m_currentLoop`, `m_currentStep`, `m_totalSteps`, and `m_stopReason` `std::atomic` so `progress()` / `stopReason()` are lock-free, or guard `Progress` with a dedicated spin/mutex. Atomics are simplest and cannot contend with `m_mutex`.
- Put `#include "test_click_worker.moc"` on the very last line of `tests/test_click_worker.cpp`.

## Definition of Done

```bash
cmake -B build
cmake --build build -j
ctest --test-dir build --output-on-failure
```

All commands pass in under 1 second with **zero compiler warnings**.
