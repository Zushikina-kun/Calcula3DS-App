# Calcula3DS

A **scientific calculator** for the Nintendo 3DS with full graphing support,
built on [CalculaThreeDS](https://github.com/LiquidFenrir/CalculaThreeDS) by LiquidFenrir.

Adds: **graph mode**, **dark/light theme**, scientific functions (fact, nCr, nPr,
mod, logn, floor, ceil, round, pol, rec), **deg/rad switching**, **complex display toggle**,
and numerous stability fixes for Old 3DS.

---

## Version history

| Version | Changes |
|---|---|
| v1.0.3 | **Startup crash fixed.** Removed APT_SetAppCpuTimeLimit, explicit heap sizes set. |
| v1.0.2 | Added -fno-exceptions, null sprite check, strtod in parser. |
| v1.0.1 | Replaced all .at() with .find() to prevent exception-based heap corruption. |
| v1.0.0 | Initial release (crashes on startup on Old 3DS). |

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

**More page**

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

**Angle mode indicator:** shows deg or rad in the gap bar at all times.
**Complex mode indicator:** shows pol in the gap bar when polar display is active.

Number types: real, complex, pi, e, i, ans, named variables a-z.
Memory: 12-entry history. Scroll top screen; press A to paste entry back.

---

### Graph mode

Press SELECT to switch to the f(x) graph plotter.

- sin(x), x^2-3, sqrt(1-x^2), tan(x), e^x, etc.
- Asymptotes render as gaps - no false vertical lines
- Pan: circle pad | Zoom: L/R | Reset: X | Grid: Y
- Parse errors shown in red, never crashes on bad input
- Note: graph mode evaluates in radians. A rad-only warning shows when in DEG mode.

---

### Dark / light theme

- Defaults to dark mode
- Toggle: hold START then press SELECT
- Theme change repaints all screens including equation history

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

---

## Key bug fixes (all versions)

| Where | Fix |
|---|---|
| main.cpp | APT_SetAppCpuTimeLimit caused svcBreak startup crash on Old 3DS - removed |
| main.cpp | Heap sizes not set - now 16MB + 8MB explicit |
| keyboard.cpp | menu.at() crash on More page - replaced with find() |
| equation.cpp, number.cpp | equ.at() crash on unknown chars - replaced with find() |
| Makefile | Added -fno-exceptions (exception unwinding corrupts heap on Old 3DS) |
| expr_parser.cpp | try/catch replaced with strtod |
| equation.cpp | Missing return on empty RPN |
| main.cpp | Theme toggle did not repaint equation/memory textures |

---

## Known limitations

- One function at a time in graph mode
- Graph mode evaluates in radians only (DEG mode is calculator-only)
- No root-finding, derivative, or integral overlays

---

## Credits

Original CalculaThreeDS by [LiquidFenrir](https://github.com/LiquidFenrir/CalculaThreeDS).
Graph mode, theme system, scientific expansion, and fixes by [Zushikina-kun](https://github.com/Zushikina-kun).
