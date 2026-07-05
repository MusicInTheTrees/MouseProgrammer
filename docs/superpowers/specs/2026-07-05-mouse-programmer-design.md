# MouseProgrammer — v1 Design

**Date:** 2026-07-05
**Status:** Approved (design), pending implementation plan

## Purpose

A locally runnable GUI utility for programming and automating mouse input on
Windows. The user builds an ordered **sequence of steps** (clicks, click-and-hold,
right-clicks, moves) and runs it on a loop. This is v1; it is intended as a clean
base to grow toward richer mouse programming later.

## Requirements

Confirmed with the user during brainstorming:

- **Sequence of steps** (not a single looped action). Steps run top to bottom.
- **Step types:** left click, left-click-and-hold, right click, move-only.
- **Target per step:** act at the **current cursor position** OR at a **saved
  coordinate** (chosen per step). A step with a coordinate target moves there
  first — this covers "move + click" and "move + hold" for free.
- **Timing controls:** hold duration (for hold steps), wait-after (pause before
  the next step), and sequence-level repeat count OR run-forever.
- **Global start/stop hotkey** that works when the app is not focused.
- **Save/load sequences to disk** (JSON) so a sequence survives closing the app.
- **Locally runnable GUI.**

## Technology

- **Language:** Python 3.
- **Mouse control + global hotkeys:** `pynput` (clean press/release for
  hold; global hotkeys independent of window focus).
- **GUI:** `tkinter` (standard library, no extra install).
- **Emergency stop:** pyautogui-style failsafe implemented ourselves — moving the
  mouse into the top-left screen corner aborts a run.

## Architecture

Three layers so the engine is testable without a GUI:

1. **Model** (`models.py`) — plain data classes. `Step` and `Sequence`. No I/O,
   no side effects.
2. **Engine** (`engine.py`) — executes a `Sequence` via a mouse controller
   abstraction. Runs on a background thread; knows nothing about tkinter.
3. **GUI** (`app.py`) — tkinter window to edit the sequence and start/stop runs.
4. **Storage** (`storage.py`) — serialize/deserialize a `Sequence` to/from JSON.

### Data model

**Step**
- `type`: one of `left_click`, `left_hold`, `right_click`, `move`
- `target`: `cursor` (act where the mouse is) OR `{x, y}` (move there first)
- `hold_ms`: hold duration in milliseconds (used only by `left_hold`)
- `wait_after_ms`: pause after this step before the next one

**Sequence**
- `steps`: ordered list of `Step`
- `repeat`: a positive integer count OR `forever`

### Mouse controller abstraction

The engine depends on a small controller interface (move, press, release,
click, right-click, get-position). The real implementation wraps `pynput`. A
**fake controller** records calls for tests. This keeps the engine unit-testable
without moving a real mouse.

### Execution flow

Press Start → optional short countdown → engine thread walks the steps top to
bottom, repeating according to `repeat`. For each step:

1. If `target` is a coordinate, move the mouse there.
2. Perform the action (click / press-hold-release / right-click / move-only).
3. Wait `wait_after_ms`.

A **stop flag** is checked between every step and during holds/waits so a run
halts promptly. The engine reports progress (current step index, current loop)
back to the GUI via a callback.

## GUI

- **Step list**: add, edit, delete, move up/down (reorder).
- **Add/edit step panel**: choose type; choose target — a **"Capture position"**
  button grabs the current cursor X/Y after a 2-second countdown, or coordinates
  can be typed manually; set hold and wait times.
- **Loop control**: repeat N times or run forever.
- **Start / Stop** buttons and a live **status line** (e.g. "Running step 2/5,
  loop 3").
- **Save / Load** buttons (JSON file).

## Safety

- **Global start/stop hotkey** (default **F6**) via `pynput`, works when the app
  is not focused.
- **Emergency stop**: move the mouse into the top-left screen corner to abort
  instantly; **Esc** also stops.
- **Pre-run countdown** so the user can clear their hands before automation
  begins.

## Testing

- **Engine** tested against the **fake mouse controller**: verifies step order,
  correct action per type, hold durations, loop/repeat counts, and prompt
  stopping via the stop flag.
- **Model** and **storage**: plain unit tests (construction, JSON round-trip,
  invalid input handling).
- **GUI** is intentionally thin and left to manual testing.

## Project layout

```
MouseProgrammer/
  mouse_programmer/
    __init__.py
    models.py
    engine.py
    storage.py
    app.py
  tests/
    test_engine.py
    test_models.py
    test_storage.py
  main.py
  requirements.txt
  README.md
```

## Out of scope for v1

- Keyboard automation.
- Scroll-wheel actions.
- Conditional/branching logic or image-based triggers.
- Recording live mouse movement paths (only discrete captured coordinates).
- Multi-monitor coordinate niceties beyond raw screen X/Y.
