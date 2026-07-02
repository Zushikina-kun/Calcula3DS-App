# Integration notes — Calcula3DS-App

This document covers every change made to the base CalculaThreeDS app,
the reasoning behind each decision, and what to watch for when testing.

---

## Status

**Fully integrated and building.** `make clean && make` passes with zero
warnings under devkitARM on devkitPro. All files have been tested in the
build system; hardware testing is still needed (see checklist below).

---

## Files added to `source/`

All new files are **additive only** — they can be removed without touching
any original file.

| File | Purpose |
|---|---|
| `theme.h / theme.cpp` | Runtime dual-theme palette (dark + light). `extern u32` variables set by `Theme::set(ID)`. All colours in one place — nothing hardcoded inline anywhere in the codebase. |
| `graph_mode.h / graph_mode.cpp` | f(x) plotter. Coordinate transforms, dirty-flag sampling loop, citro2d drawing, system keyboard (swkbd) for function input. |
| `expr_parser.h / expr_parser.cpp` | Self-contained recursive-descent parser and evaluator. Bounded recursion (kMaxDepth=32), bounded node count (256 nodes), no allocation during eval. |
| `ui_text.h / ui_text.cpp` | Thin text rendering wrapper over the existing `TextMap::char_to_sprite->equ` atlas. Additive — does not touch `text.cpp`. |
| `sleep_hook.h / sleep_hook.cpp` | APT lifecycle hook. Uses `std::atomic<bool>` with acquire/release ordering (replaces original `volatile bool` design). |

The Makefile globs `*.cpp`/`*.h` in `source/` — no Makefile changes needed
for these files beyond what's already there.

---

## Files modified

### `main.cpp`

- Includes `graph_mode.h`, `sleep_hook.h`, `theme.h` (replaces `colors.h`)
- `Theme::set(Theme::DARK)` called at startup — dark mode is the default
- SELECT alone toggles calculator ↔ graph mode
- Hold START + press SELECT toggles dark ↔ light theme
- Sleep hook connected: `lifecycle.consume_resume_flag()` → `graph.invalidate()`
- `C2D_TargetClear` colours updated to `Theme::BG`
- `KEY_START` and `KEY_SELECT` masked from both input dispatch paths so
  neither leaks into `kb.handle_buttons` or `graph.handle_buttons`

### `keyboard.cpp`

- `#include "colors.h"` replaced with `#include "theme.h"`
- Every `COLOR_*` reference replaced with the corresponding `Theme::*` name
- `C2D_Color32(0,0,0,255)` cursor → `Theme::CURSOR`
- `C2D_Color32(0,0,0,128)` scroll arrows → `Theme::FG_ARROW`
- `COLOR_BLACK` gap bar fill → `Theme::SURFACE`
- `COLOR_BLUE` button face → `Theme::KEY_FACE`
- Button labels now use a separate `label_tint` at `Theme::KEY_LABEL`
  (previously the label was the same tint as the button face, making it
  invisible in dark mode)
- "select graph" hint drawn **before** the `COLOR_HIDE` overlay so it
  dims correctly when the top screen is selected

### `equation.cpp`

- `#include "colors.h"` replaced with `#include "theme.h"`
- `COLOR_BLACK` text/line tints → `Theme::FG`
- `COLOR_GRAY` temp-paren tint → `Theme::FG_DIM`
- `COLOR_BLUE` selected-paren tint → `Theme::KEY_FACE`
- **Bug fix:** `if(final_rpn.empty()) std::make_pair(...)` was missing
  `return` — the expression was a no-op, so an empty RPN silently continued
  into the evaluator. Fixed to `return std::make_pair(Number{}, true)`.

### `number.cpp`

- `#include "colors.h"` replaced with `#include "theme.h"`
- Answer bar background → `Theme::ANS_BAR_BG`
- Answer bar text tint → `Theme::ANS_BAR_FG`

### `graph_mode.cpp`

- `#include "colors.h"` replaced with `#include "theme.h"`
- All drawing uses `Theme::GRAPH_*` and `Theme::FG*` constants
- Hardcoded `constexpr u32 CURVE_COLOR` removed — now `Theme::GRAPH_CURVE`
- `px_to_x` return type changed `float` → `double` (precision fix)
- `y_to_py` parameter type changed `float` → `double` (precision fix)
- `y_to_py(0.0f)` axis call updated to `y_to_py(0.0)`

### `Makefile`

- `-DARM11 -D_3DS` → `-D__3DS__` (removes compiler warning on every build)
- Added `cia` phony target with `bannertool` / `makerom` guard and
  `cia/app.rsf` RSF configuration

---

## Why a separate expression parser for graph mode

The existing `Equation` class is the pretty-printing editor driven by
`Keyboard`'s private `add_part_at()` and `variables` map. Reusing it for
graph mode would mean either exposing private internals or replicating how
`Keyboard` drives `Equation` — both are regression risks. `Expr` is small,
independently testable, and deliberately mirrors the existing engine's
function names (`ln` = natural log, `log` = log10) so it feels like one app.

**Future unification path:** extract `Equation::calculate`'s
function-dispatch table into something `Expr` can call directly. Out of
scope until the graph side is confirmed stable on hardware.

---

## Why a separate theme system instead of a config file

The 3DS has no writable filesystem accessible without romfsInit/romfsExit
overhead per write, and no standard config format in libctru. A runtime
palette stored in global variables is the lightest possible approach:
- Zero I/O
- Zero allocation
- Toggle is instant (next frame picks up new colours)
- `theme.cpp` is the single place to add colours or new themes

---

## Theme toggle controls

| Input | Action |
|---|---|
| SELECT (alone) | Toggle calculator ↔ graph mode |
| Hold START + press SELECT | Toggle dark ↔ light theme |
| START tap (release before ~800ms) | Quit |

The 800ms threshold means a normal quit tap won't accidentally trigger the
theme toggle, and holding START long enough to toggle won't quit.

---

## Optional: New3DS syscore for the calculation thread

**Skip if you are on an original 3DS or 2DS.** This change only benefits
New3DS hardware.

<details>
<summary>New3DS-only: syscore thread affinity (click to expand)</summary>

`main.cpp` calls `APT_SetAppCpuTimeLimit(30)` at startup. On New3DS this
grants permission to use the syscore (the extra core Old3DS doesn't have).
`keyboard.cpp`'s calculation thread uses affinity `1` so that permission is
never used — the thread competes with the render thread on core 1.

Change in `Keyboard::start_calculating`:
```cpp
// before:
calcThread = threadCreate(calculation_loop, this, 256 * 1024, 31, 1, false);

// after:
s32 calc_core = 1;
bool is_n3ds = false;
if(R_SUCCEEDED(APT_CheckNew3DS(&is_n3ds)) && is_n3ds)
    calc_core = -2; // New3DS syscore
calcThread = threadCreate(calculation_loop, this, 256 * 1024, 31, calc_core, false);
```

Test on both Old3DS and New3DS (or Citra in New3DS mode) before trusting this.

</details>

---

## Testing checklist

1. **Builds clean:** `make clean && make` — zero warnings, zero errors
2. **Runs in Citra** before putting on hardware — citro2d depth ordering
   can differ subtly between emulator and hardware
3. **Theme toggle:** hold START + SELECT, confirm both screens switch colours
   instantly; confirm START tap still quits normally
4. **Mode toggle:** SELECT back and forth several times — confirm the
   calculator's equation and memory are untouched after visiting graph mode
5. **Dark mode rendering:** equation text, cursors, sqrt bars, fraction lines,
   paren highlights all visible against dark background
6. **Graph error states:** type `sin(` — confirm red error box, not crash
7. **Asymptotes:** graph `1/x` and `tan(x)` — gaps at discontinuities, not
   vertical lines
8. **Zoom guard:** zoom in (R) repeatedly — confirm it stops, no
   divide-by-zero or inverted view
9. **Sleep/resume:** press HOME, wait, return — confirm graph screen redraws
   correctly (sleep hook)
10. **CIA build** (if bannertool available): `make cia`, install via FBI,
    confirm it launches from home menu

---

## Next pieces, in priority order

1. Fix anything the hardware test above turns up
2. Real touch-driven tab bar (Standard / Graph) replacing SELECT toggle
3. Degree/radian toggle for graph mode and the calculator
4. Multiple simultaneous functions on one graph (different colours)
5. Axis tick labels with proper numeric formatting
6. Base-N (hex/oct/bin) mode
7. Root-finding / derivative overlay on graph
8. Unified math engine (one evaluator for both editor and graph mode)
