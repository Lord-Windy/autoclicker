# Ticket 06: Persistence, Monitor Bounds Warning, `--test-sequence` CLI, German Translations & Docs (Phase 4)

**Phase:** 4 (Persistence, CLI dry-run/live runner, i18n & documentation)  
**Depends on:** Tickets 01–05  
**Read first:** `tickets/00-README.md`, `PLAN.md`, `src/Sequence.h`, `src/MainWindow.cpp`, `src/main.cpp`, `i18n/autoclicker_de.ts`, `README.md`

## Goal

Finish the feature end-to-end:
1. Auto-save and auto-load the sequence (`sequence.json` in `QStandardPaths::AppConfigLocation`) and window size (`QSettings`).
2. Warn non-modally if loaded/current step coordinates lie outside all connected screens (`QGuiApplication::screens()`).
3. Add `--test-sequence <file.json> [--live]` to `src/main.cpp` (defaulting to a fast, safe dry-run trace that needs neither `/dev/uinput` nor KWin).
4. Update `i18n/autoclicker_de.ts` via `update_translations` and translate **every** new UI string into natural German matching the existing file's terminology and tone.
5. Update `README.md` to document multi-location sequences, sub-loops, persistence, and `--test-sequence`.

---

## Files to create / modify

- **Modify:** `src/Sequence.h`, `src/Sequence.cpp` (optional file load/save convenience + screen bounds helper)
- **Modify:** `src/MainWindow.h`, `src/MainWindow.cpp`
- **Modify:** `src/main.cpp`
- **Modify:** `i18n/autoclicker_de.ts`
- **Modify:** `README.md`
- **Modify:** `tests/test_sequence.cpp` (add file round-trip & screen-bounds helper tests)

---

## Exact specification

### 1. Persistence (`MainWindow::loadSettings` / `saveSettings`)

- **Sequence file path:**
  `QStandardPaths::writableLocation(QStandardPaths::AppConfigLocation) + QStringLiteral("/sequence.json")`
  - On `saveSettings()`: create parent directory with `QDir().mkpath(...)` and write `sequenceToJson(sequenceDocument())` atomically using `QSaveFile`!
  - On `loadSettings()`: if `sequence.json` exists, read and parse with `sequenceFromJson(...)`.
    - If valid: call `setSequenceDocument(*doc)`.
    - If corrupt/unreadable: do **not** crash and do **not** pop up a blocking modal dialog on startup — leave sequence empty and show a non-modal warning in `m_status` (e.g. `tr("Saved sequence could not be loaded: %1").arg(err)`).
- **Window geometry in `QSettings`:**
  - Save `size()` under key `"windowSize"` in `QSettings` (remove legacy `"maxClicks"` reliance once migrated, or fall back to `QSettings` `"maxClicks"` on very first launch when `sequence.json` doesn't exist yet!).
- **Important CLI isolation for `--screenshot`:**
  - If `--screenshot` is passed on the command line (or add a constructor flag / skip loading user's `~/.config/autoclicker/sequence.json` when `--screenshot` overrides the sequence), make sure `--screenshot out.png 400 300` renders a deterministic screenshot rather than whatever random sequence was in `~/.config`! (Passing `--screenshot out.png path/to/file.json` is also very handy to support.)

### 2. Multi-monitor coordinate validation

- Helper function (testable with a `QList<QRect>` in `src/Sequence.h`!):
  ```cpp
  QList<QPoint> findOffScreenTargets(const Sequence &nodes, const QList<QRect> &screenGeometries);
  ```
  A target `pos` is on-screen iff **at least one** `QRect` in `screenGeometries` contains `pos` (`std::any_of`).
  **Do NOT check against the united `desktopGeometry()` bounding box!** On dual-monitor setups with different resolutions (e.g. 2560×1440 next to 1920×1080), the united bounding box includes dead off-screen corners that no monitor displays.
- **On startup (`loadSettings`):**
  If `m_pointer->isValid()` and `findOffScreenTargets(...)` is non-empty, display a **non-modal** status bar message:
  `tr("Warning: %1 target(s) are outside the current screen layout").arg(offScreen.size())`.
- **On `onStartButton()`:**
  Re-check `findOffScreenTargets` against current `QGuiApplication::screens()` (monitors may have been unplugged while the app was open). If any reachable target is off-screen, ask via `QMessageBox::question` whether to start anyway (`Yes` / `No`, default `No`).

### 3. `--test-sequence <file.json> [--live]` (`src/main.cpp`)

Add CLI handling before creating `MainWindow`:
- **Usage:** `autoclicker --test-sequence <file.json> [--live]` (exit `2` on missing path, exit `1` on file/JSON error or empty executable sequence).
- **Default mode (dry-run — when `--live` is NOT passed):**
  - Does **not** open `/dev/uinput` and does **not** talk to KWin!
  - Uses a `DryRunPointer : public PointerSink` that prints each click (`#1  moveTo(400, 300)  click(hold=50ms)`) to `stdout` without sleeping.
  - Note: If `doc.maxLoops == 0 && doc.maxClicks == 0` (unlimited) in dry-run mode, run **1 main loop** (`effectiveMaxLoops = 1`) and print a note `(unlimited sequence: dry-running 1 loop)` so `--test-sequence` terminates instead of running forever!
  - Also run dry-run with `skipWaits = true` on `ClickWorker::start(doc.nodes, QRect(0, 0, 1920, 1080), effectiveMaxLoops, doc.maxClicks, doc.jitter, /*skipWaits=*/true)` — that exercises the real `ClickWorker` engine (including `hold` calculation and `doc.jitter` offsets!) while finishing in `< 1ms`!
  - Print summary line at the end: `Total clicks: N, stop reason: ...` and exit `0`.
- **`--live` mode:**
  - Opens `VirtualPointer`, waits `DeviceSettleMs`, starts `MouseWatcher(collectTargets(doc.nodes), doc.jitter.enabled ? MouseWatcher::DefaultTolerance + JitterConfig::MaxOffsetPx : MouseWatcher::DefaultTolerance)`, waits for `MouseWatcher::ready`, runs `ClickWorker` with `doc.jitter` and real delays (`skipWaits = false`), stops immediately on `MouseWatcher::userActivity` (printing `Stopped by mouse movement`), and exits `0`.

### 4. German translations (`i18n/autoclicker_de.ts`)

1. Run:
   ```bash
   cmake --build build --target update_translations
   ```
   *(Notice `CMakeLists.txt` configures `LUPDATE_OPTIONS -no-obsolete -locations none`, which automatically strips obsolete v1.0 single-target strings and adds all new `tr(...)` strings with `type="unfinished"`.)*
2. Edit `i18n/autoclicker_de.ts` to translate **every** `<translation type="unfinished">` entry and remove `type="unfinished"`.
3. Match the existing German terminology and style in `i18n/autoclicker_de.ts`:
   - `Autoklicker`, `Klicks`, `Schleife` / `Schleifen` (for loop/loops), `Schritt` / `Schritte` (for step/steps), `Verzögerung` (for delay), `Wiederholungen` (for repeats), `Beschreibung` / `Bezeichnung` (for label), `Jitter:` / `Versatz:` (`1–2 px bewegen, halten für` … `bis` … `Ticks`)
   - Button infinitives matching `Ziel auswählen` / `Klicken starten`: e.g. `Schritt hinzufügen`, `Neu wählen`, `Entfernen`, `Nach oben`, `Nach unten`, `Als Schleife gruppieren`, `Gruppierung aufheben`
   - Preserve Unicode `…`, `–`, `×`, and `%1` / `%2` / `%3` placeholders!
4. Rebuild (`cmake --build build -j`) and verify `lrelease` compiles `autoclicker_de.qm` with **zero unfinished translations and zero warnings**.

### 5. Documentation (`README.md`)

Update `README.md`:
- Update description & **Features** list (multi-location sequences, per-step label/delay/repeat, sub-loops e.g. steps 3–5 ×25, optional 1–2 px jitter held for a configurable `[min, max]` tick range, stop after N loops or N clicks, automatic sequence saving, stop-on-mouse-movement across all targets).
- Update **Usage** steps (and remove the outdated sentence *"The target is intentionally not saved"* — explain that sequences are saved automatically in `~/.config/autoclicker/sequence.json` and advise keeping the always-on-top window clear of target coordinates).
- Update **Source code** table (`src/Sequence.cpp`, `src/PointerSink.h`, etc.).
- Update **Test and developer options** with `--test-sequence file.json [--live]` and `ctest --test-dir build`.

---

## Advice & pitfalls for the implementing agent

- **Check for any remaining `type="unfinished"` in `i18n/autoclicker_de.ts`** using `grep` before finishing (`grep unfinished i18n/autoclicker_de.ts` must return 0 matches!).
- **Do not run `update_translations` *after* editing `i18n/autoclicker_de.ts` unless you modified C++ source strings again** — if you do modify C++ strings, re-run `update_translations`, re-check `unfinished`, and rebuild.
- **Test `--test-sequence` in dry-run mode** on a sample JSON file under `/tmp/opencode/` (with a 25× sub-loop!) to verify CLI output and exit code `0` without touching `/dev/uinput`.

## Definition of Done

```bash
cmake -B build
cmake --build build --target update_translations
cmake --build build -j
ctest --test-dir build --output-on-failure
./build/autoclicker --test-sequence /tmp/opencode/sample_macro.json
QT_QPA_PLATFORM=offscreen ./build/autoclicker --screenshot /tmp/opencode/final_ui.png 400 300
```

- Zero compiler warnings (`-Wall -Wextra`).
- Zero `"unfinished"` entries in `i18n/autoclicker_de.ts`.
- All CTest unit tests pass (`test_sequence`, `test_click_worker`).
- Dry-run `--test-sequence` prints the expanded sequence and exits `0`.
