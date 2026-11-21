# Ticket 01: Sequence Data Model, JSON Serialization & CTest Setup

**Phase:** 1a (Engine foundation — no UI changes)  
**Depends on:** None  
**Read first:** `PLAN.md`, `tickets/00-README.md`, `CMakeLists.txt`

## Goal

Create `src/Sequence.h` and `src/Sequence.cpp` containing the recursive sequence
data model, tree traversal/validation helpers, and versioned JSON
serialization/deserialization. Set up CTest in `CMakeLists.txt` and write unit
tests in `tests/test_sequence.cpp`.

Do **not** modify `MainWindow`, `ClickWorker`, or `MouseWatcher` in this ticket.

---

## Files to create / modify

- **Create:** `src/Sequence.h`
- **Create:** `src/Sequence.cpp`
- **Create:** `tests/test_sequence.cpp`
- **Modify:** `CMakeLists.txt`

---

## Exact specification

### 1. Data structures (`src/Sequence.h`)

Use these exact types and signatures so later tickets connect cleanly:

```cpp
#pragma once

#include <QByteArray>
#include <QList>
#include <QPoint>
#include <QString>

#include <optional>

struct ClickStep
{
    QPoint pos;
    int delayAfterMs = 200; // >= 0
    int repeat = 1;         // >= 0 in engine; UI enforces >= 1
    QString label;

    bool operator==(const ClickStep &other) const = default;
};

struct SeqNode;
using Sequence = QList<SeqNode>;

struct SeqNode
{
    enum class Type { Step, Loop };

    Type type = Type::Step;
    ClickStep step;         // used when type == Type::Step
    int iterations = 1;     // used when type == Type::Loop
    Sequence children;      // used when type == Type::Loop

    static SeqNode makeStep(const ClickStep &step);
    static SeqNode makeLoop(int iterations, Sequence children);
};

bool operator==(const SeqNode &a, const SeqNode &b);

struct JitterConfig
{
    static constexpr int MaxOffsetPx = 2; // random 1..2 px shift in X/Y

    bool enabled = false;
    int minTicks = 4;  // minimum clicks before picking a new offset (>= 1)
    int maxTicks = 12; // maximum clicks before picking a new offset (>= minTicks)

    bool operator==(const JitterConfig &other) const = default;
};

struct SequenceDocument
{
    static constexpr int CurrentVersion = 1;

    int version = CurrentVersion;
    quint64 maxLoops = 0;   // 0 = unlimited
    quint64 maxClicks = 0;  // 0 = unlimited
    JitterConfig jitter;
    Sequence nodes;

    bool operator==(const SequenceDocument &other) const = default;
};

// Total number of Type::Step leaf nodes in pre-order traversal (ignores repeat/iterations;
// used for 1..N UI numbering and "Step k/N" progress display).
int leafStepCount(const Sequence &nodes);

// True if at least one Type::Step with repeat >= 1 is reachable (all enclosing Loops have iterations >= 1).
bool hasExecutableSteps(const Sequence &nodes);

// Unique target coordinates of all reachable executable Type::Step nodes, in first-seen pre-order.
QList<QPoint> collectTargets(const Sequence &nodes);

// Serialize to formatted UTF-8 JSON object.
QByteArray sequenceToJson(const SequenceDocument &doc);

// Parse and validate UTF-8 JSON object. Returns std::nullopt and sets *errorOut (if non-null) on failure.
std::optional<SequenceDocument> sequenceFromJson(const QByteArray &json, QString *errorOut = nullptr);
```

### 2. JSON format (`src/Sequence.cpp`)

Serialize a root `QJsonObject` (`QJsonDocument::Indented`):

```json
{
    "version": 1,
    "maxLoops": 0,
    "maxClicks": 0,
    "jitter": {
        "enabled": true,
        "minTicks": 4,
        "maxTicks": 12
    },
    "nodes": [
        {
            "type": "step",
            "x": 400,
            "y": 300,
            "delayAfterMs": 200,
            "repeat": 1,
            "label": "Craft"
        },
        {
            "type": "loop",
            "iterations": 25,
            "children": [
                {
                    "type": "step",
                    "x": 500,
                    "y": 350,
                    "delayAfterMs": 500,
                    "repeat": 2,
                    "label": ""
                }
            ]
        }
    ]
}
```

**Deserialization validation rules (`sequenceFromJson`):**
- Root must be a JSON object (`QJsonParseError` or non-object → fail with clear message).
- `"version"` must be an integer equal to `1` (reject unsupported future versions or `< 1`).
- `"maxLoops"` and `"maxClicks"` default to `0` if omitted; reject negative numbers or non-numeric values. Note JSON numbers are `double`: check `val.isDouble()` and `val.toInteger() >= 0`, then cast to `quint64`.
- `"jitter"` defaults to `JitterConfig{}` if omitted; if present, must be a JSON object:
  - `"enabled"` defaults to `false` (if present, must be boolean).
  - `"minTicks"` defaults to `4` (must be integer `>= 1`).
  - `"maxTicks"` defaults to `12` (must be integer `>= minTicks`; reject if `maxTicks < minTicks` or `< 1`).
- `"nodes"` must be a JSON array.
- For each node object:
  - `"type"` must be `"step"` or `"loop"` (anything else fails).
  - If `"step"`: require integer-convertible `"x"` and `"y"`; `"delayAfterMs"` defaults to `200` (reject `< 0`); `"repeat"` defaults to `1` (reject `< 0`); `"label"` defaults to `""` (if present, must be a string).
  - If `"loop"`: `"iterations"` defaults to `1` (reject `< 0`); `"children"` must be a JSON array, parsed recursively.
  - Guard against pathological recursion depth (e.g., reject nesting depth `> 32` so malformed JSON cannot stack-overflow).

### 3. CMake & CTest setup (`CMakeLists.txt`)

- Add `src/Sequence.cpp` to `autoclicker` (or better: create an `OBJECT` or `STATIC` library `autoclicker_core` containing `src/Sequence.cpp` — and in Ticket 02 `src/ClickWorker.cpp` — linked by both `autoclicker` and unit tests so files are only compiled once and tests don't need `LayerShellQt` or `Qt6::DBus`).
- Enable CTest cleanly:
  ```cmake
  include(CTest)
  if(BUILD_TESTING)
      find_package(Qt6 REQUIRED COMPONENTS Test)
      add_executable(test_sequence tests/test_sequence.cpp)
      target_compile_options(test_sequence PRIVATE -Wall -Wextra)
      target_include_directories(test_sequence PRIVATE src)
      target_link_libraries(test_sequence PRIVATE autoclicker_core Qt6::Test)
      add_test(NAME test_sequence COMMAND test_sequence)
  endif()
  ```
- **Important:** Do **not** add `autoclicker_core` or `test_sequence` to `install(TARGETS ...)`.

### 4. Unit tests (`tests/test_sequence.cpp`)

Use `QTEST_APPLESS_MAIN(TestSequence)` and verify:
- Empty `SequenceDocument` JSON round-trip.
- Nested sequence (`[Step1, Step2, Loop×25([Step3, Loop×2([Step4])]), Step5]`) round-trip preserves all fields (`pos`, `delayAfterMs`, `repeat`, `label`, `iterations`, `maxLoops`, `maxClicks`, and `jitter` `{ enabled, minTicks, maxTicks }`).
- Omitting `"jitter"` in JSON parses cleanly to default `JitterConfig{}`.
- Invalid `"jitter"` (`minTicks < 1`, `maxTicks < minTicks`, non-object `"jitter"`) returns `std::nullopt`.
- `leafStepCount`: returns `0` for empty/empty-loops, and counts all leaf steps in pre-order (`5` in the nested example above).
- `hasExecutableSteps`:
  - `false` on `[]`, `[Loop×10([])]`, `[Step(repeat=0)]`, `[Loop×0([Step(repeat=1)])]`.
  - `true` when at least one reachable `Step` has `repeat >= 1`.
- `collectTargets`:
  - Deduplicates repeated coordinates while preserving first-seen order.
  - Excludes steps inside `Loop`s with `iterations == 0` or steps with `repeat == 0`.
- Error handling in `sequenceFromJson`:
  - Invalid JSON syntax returns `std::nullopt` and populates `errorOut`.
  - Unsupported `"version": 99` returns `std::nullopt`.
  - Negative `"delayAfterMs"`, negative `"repeat"`, negative `"iterations"`, unknown `"type"`, missing `"x"`/`"y"` all return `std::nullopt`.

---

## Advice & pitfalls for the implementing agent

- **Do NOT `= default` `SeqNode::operator==` inside `struct SeqNode`!** Because `SeqNode` contains `QList<SeqNode> children`, `SeqNode` is still an incomplete type inside its own class body, which can break template instantiation of `QList<SeqNode>::operator==`. Implement `bool operator==(const SeqNode &a, const SeqNode &b)` out-of-line in `src/Sequence.cpp` (comparing `type`, and then `step` when `Type::Step` or `iterations` + `children` when `Type::Loop` — that also avoids comparing irrelevant union-like fields!).
- Remember `#include "test_sequence.moc"` on the very last line of `tests/test_sequence.cpp`.
- Never use `assert()` in tests (`Release` mode defines `-DNDEBUG`); use `QVERIFY` and `QCOMPARE`.

## Definition of Done

```bash
cmake -B build
cmake --build build -j
ctest --test-dir build --output-on-failure
```

All commands succeed with **zero compiler warnings** and `test_sequence` passing.
