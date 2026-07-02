# Calcula3DS

A **scientific calculator** for the Nintendo 3DS with full graphing support,
built on [CalculaThreeDS](https://github.com/LiquidFenrir/CalculaThreeDS) by LiquidFenrir.

Adds: **graph mode**, **dark/light theme**, scientific functions (fact nCr nPr mod logn
floor ceil round pol rec), deg/rad switching, complex display toggle, and bug fixes.

---

## Features

### Scientific Calculator

Pretty-printed equation editor - fractions, exponents, roots render like a textbook.

**Keyboard pages**

| Page | Buttons |
|---|---|
| Basic | ( ) ^ / (fraction) del . 0-9 + - * = |
| Functions | sin cos tan asin acos atan exp abs sqrt ln cosh sinh tanh log conj acosh asinh atanh |
| Variables | a-n, x-z, ans, i, pi, > (store to variable) |
| More | fact nCr nPr mod logn / floor ceil round pol rec / cplx deg |

**More page functions**

| Button | Syntax | What it does |
|---|---|---|
| fact | fact(n) | n! for integer n >= 0 |
| nCr | nCr(n,r) | Binomial coefficient |
| nPr | nPr(n,r) | Permutations |
| mod | mod(a,b) | Modulo (same sign as b) |
| logn | logn(base,x) | Log to arbitrary base |
| floor | floor(x) | Round down |
| ceil | ceil(x) | Round up |
| round | round(x) | Round to nearest |
| pol | pol(x,y) | Cartesian to polar - result shown as r>theta |
| rec | rec(r,t) | Polar to Cartesian - result shown as x+yi |
| deg | -- | Toggle DEG / RAD angle mode |
| cplx | -- | Toggle a+bi / r>theta display |

Angle mode indicator: shows deg or rad in the gap bar.
Complex mode indicator: shows pol when polar display is active.

Number types: real, complex, pi, e, i, ans, variables a-z.
Memory: 12-entry history. Scroll top screen; press A to paste entry back.

---

### Graph mode

Press SELECT to switch to the f(x) graph plotter.

- sin(x), x^2-3, sqrt(1-x^2), tan(x), e^x, etc.
- Asymptotes render as gaps - no false vertical lines
- Pan: circle pad | Zoom: L/R | Reset: X | Grid: Y
- Parse errors shown in red, never crashes on bad input
- Correct redraw after HOME menu or sleep
- Note: graph mode always evaluates in radians. A rad-only warning
 is shown on screen when the calculator is in DEG mode.

---

### Dark / light theme

- Defaults to dark mode
- Toggle: hold START then press SELECT
- Theme change immediately repaints all screens including equation history
- All colours in source/theme.cpp

---

## Controls

| Input | Action |
|---|---|
| SELECT | Toggle calculator / graph mode |
| START tap | Quit |
| Hold START + SELECT | Toggle dark / light theme |

**Calculator**

| Input | Action |
|---|---|
| Touch keyboard | Enter expression |
| A | Calculate |
| B | Delete |
| Y | Clear |
| L / R | Switch keyboard page |
| D-pad left/right | Move cursor |
| X | Switch focus: keyboard / memory |
| Circle pad | Scroll |

**Graph mode**

| Input | Action |
|---|---|
| Touch f(x) box | Open system keyboard |
| Circle pad | Pan |
| R / L | Zoom in / out |
| X | Reset view |
| Y | Toggle grid |

---

## Building

| Tool | Notes |
|---|---|
| [devkitPro](https://devkitpro.org/wiki/Getting_Started) | Select 3DS Development group |
| devkitARM, libctru, citro2d/citro3d | Included in devkitPro |
| [makerom v0.19+](https://github.com/3DSGuy/Project_CTR/releases) | CIA only - place exe in devkitPro\tools\bin |
| [bannertool](https://github.com/diasurgical/bannertool/releases) | CIA only - place exe in devkitPro\tools\bin |

 git clone --recurse-submodules https://github.com/Zushikina-kun/Calcula3DS-App.git
 cd Calcula3DS-App/CalculaThreeDS
 make # out/CalculaThreeDS.3dsx
 make cia # out/CalculaThreeDS.cia

Windows env (run before make):
 set DEVKITPRO=/opt/devkitpro
 set DEVKITARM=/opt/devkitpro/devkitARM
 add devkitARM/bin, tools/bin, msys2/usr/bin to PATH

---

## Project structure

 Calcula3DS-App/
 +-- CalculaThreeDS/
 | +-- source/
 | | +-- main.cpp
 | | +-- theme.h/.cpp (dual-theme palette)
 | | +-- calcmode.h/.cpp (DEG/RAD angle mode)
 | | +-- cplxmode.h/.cpp (RECT/POLAR complex display)
 | | +-- graph_mode.h/.cpp (f(x) graph mode)
 | | +-- expr_parser.h/.cpp (parser for graph mode)
 | | +-- ui_text.h/.cpp (text rendering for graph mode)
 | | +-- sleep_hook.h/.cpp (APT sleep/resume hook)
 | | +-- keyboard.h/.cpp (calculator keyboard and input)
 | | +-- equation.h/.cpp (equation renderer and evaluator)
 | | +-- number.h/.cpp (complex number display)
 | | +-- text.h/.cpp (glyph atlas)
 | +-- cia/
 | | +-- app.rsf (makerom v0.19 config)
 | | +-- banner.png (256x128 banner)
 | | +-- banner.wav (silent WAV)
 | +-- Makefile
 +-- INTEGRATION_NOTES.md
 +-- README.md

---

## What changed from the original

| File | Change |
|---|---|
| theme.h/.cpp | NEW - runtime dual-theme palette |
| calcmode.h/.cpp | NEW - DEG/RAD angle mode |
| cplxmode.h/.cpp | NEW - RECT/POLAR complex display |
| graph_mode.h/.cpp | NEW - f(x) graph mode |
| expr_parser.h/.cpp | NEW - parser/evaluator for graph mode |
| ui_text.h/.cpp | NEW - text rendering for graph mode |
| sleep_hook.h/.cpp | NEW - APT lifecycle hook |
| main.cpp | SELECT/theme/sleep-hook wiring; dark default; kb.invalidate() on theme toggle |
| keyboard.h/.cpp | Theme colours; More page; mode indicators; Keyboard::invalidate() |
| equation.cpp | 10 new functions; trig deg/rad conversion; bug fixes |
| number.cpp | Polar display via CplxMode |
| text.cpp | All More-page labels in menu atlas |
| graph_mode.cpp | Theme colours; rad-only warning; removed dead py_to_y |
| Makefile | Fixed define; make cia target |
| cia/app.rsf | Rewritten for makerom v0.19 |

**Bug fixes**

| Where | Fix |
|---|---|
| equation.cpp | Missing return on empty RPN - silent crash |
| equation.cpp | .real() on plain double from acos/asin/atan (ARM gcc 16) |
| graph_mode.cpp | px_to_x / y_to_py float -> double |
| sleep_hook.h | volatile bool -> std::atomic<bool> |
| keyboard.cpp | Hint drawn after dim overlay |
| keyboard.cpp | Removed dead YES_TINT/NO_TINT code set once and never read |
| main.cpp | Theme toggle did not repaint equation/memory textures (kb.invalidate()) |
| Makefile | Wrong compiler define |

---

## Known limitations

- One function at a time in graph mode
- Graph mode evaluates in radians only (warned on screen when in DEG mode)
- No root-finding, derivative, or integral overlays

---

## Credits

Original CalculaThreeDS by [LiquidFenrir](https://github.com/LiquidFenrir/CalculaThreeDS).
Graph mode, theme system, scientific expansion, and fixes by [Zushikina-kun](https://github.com/Zushikina-kun).
