# Calcula3DS

A scientific calculator with graphing for the Nintendo 3DS, built on
[CalculaThreeDS](https://github.com/LiquidFenrir/CalculaThreeDS) by LiquidFenrir.

This repo adds a **graph mode**, a **dark/light theme system**, and a
collection of bug fixes on top of the original app — all without touching
the original calculator's core logic.

---

## Screenshots

> Default dark theme on the original 3DS blue model.

| Calculator | Graph mode |
|---|---|
| Dark keyboard with blue-slate keys | f(x) plotter on deep-navy background |

---

## Features

### Standard calculator
- Complex number support
- Pretty-printed equations: fractions, exponents, roots, absolute values
- 12-entry scrollable memory history
- Scientific functions: `sin cos tan asin acos atan sinh cosh tanh ln log sqrt abs exp conj`
- Variables: `a-n, x-z, ans, i, pi`

### Graph mode
- Plot any `f(x)`: `sin(x)`, `x^2-3`, `sqrt(1-x^2)`, `1/x`, `tan(x)`, etc.
- Circle pad to pan, L/R to zoom, X to reset, Y to toggle grid
- Asymptotes and domain edges render as gaps — no false vertical lines
- Parse errors shown inline — never crashes on bad input
- Correct redraw after HOME menu / system sleep

### Dark / light theme
- Defaults to **dark mode** — easy on the eyes in any lighting
- Toggle at runtime: **hold START then press SELECT**
- Two full palettes: dark (Material-inspired navy/slate) and light (clean white)
- Every colour centralised in `theme.h` / `theme.cpp` — trivial to customise

---

## Controls

### Both modes
| Input | Action |
|---|---|
| SELECT | Toggle calculator ↔ graph mode |
| START (tap) | Quit |
| Hold START + SELECT | Toggle dark / light theme |

### Calculator mode
| Input | Action |
|---|---|
| Touch keyboard | Enter expression |
| A | Calculate |
| B | Delete character |
| Y | Clear (new equation) |
| L / R | Switch keyboard page (basic / functions / variables) |
| D-pad ← → | Move cursor |
| X | Switch focus between bottom (keyboard) and top (memory) screen |
| Circle pad | Scroll equation or memory list |

### Graph mode
| Input | Action |
|---|---|
| Touch f(x) box | Open system keyboard to type function |
| Circle pad | Pan view |
| R | Zoom in |
| L | Zoom out |
| X | Reset view to −10 … +10 |
| Y | Toggle grid lines |

---

## Requirements

### To run
| Requirement | Details |
|---|---|
| Nintendo 3DS / 2DS (any model) | Tested target: original 3DS |
| Custom Firmware (CFW) | [3ds.hacks.guide](https://3ds.hacks.guide/) — Luma3DS recommended |
| Homebrew Launcher **or** FBI | For installing `.3dsx` or `.cia` |

### To build
| Tool | Version | Notes |
|---|---|---|
| [devkitPro](https://devkitpro.org/wiki/Getting_Started) | Latest | Windows / Linux / macOS |
| devkitARM | Included in `3ds-dev` group | ARM cross-compiler |
| libctru | Included in `3ds-dev` group | 3DS system library |
| citro2d / citro3d | Included in `3ds-dev` group | 2D/3D GPU wrappers |
| [makerom](https://github.com/3DSGuy/Project_CTR/releases) | v0.19.0+ | For CIA builds only |
| [bannertool](https://github.com/Steveice10/bannertool) | Any | For CIA builds only — must build from source or find a pre-built binary |

---

## Building

### 1. Install devkitPro (Windows)

Download the installer from [devkitpro.org](https://devkitpro.org/wiki/Getting_Started).
During setup, select the **3DS Development** group to install devkitARM,
libctru, citro2d, and citro3d.

### 2. Set environment variables

Open PowerShell and run (or add to your system environment permanently):

```powershell
$env:DEVKITPRO = "C:\devkitPro"
$env:DEVKITARM = "C:\devkitPro\devkitARM"
$env:CTRULIB   = "C:\devkitPro\libctru"
$env:PATH     += ";C:\devkitPro\devkitARM\bin;C:\devkitPro\tools\bin;C:\devkitPro\msys2\usr\bin"
```

### 3. Clone with submodule

```bash
git clone --recurse-submodules https://github.com/Zushikina-kun/Calcula3DS-App.git
cd Calcula3DS-App
```

If already cloned without `--recurse-submodules`:

```bash
git submodule update --init --recursive
```

### 4. Build the `.3dsx`

```bash
cd CalculaThreeDS
make
```

Output: `CalculaThreeDS/out/CalculaThreeDS.3dsx`

Clean rebuild:

```bash
make clean && make
```

### 5. Install to 3DS (`.3dsx`)

Copy `CalculaThreeDS.3dsx` to `/3ds/CalculaThreeDS/CalculaThreeDS.3dsx` on
your SD card, then launch via the Homebrew Launcher.

Or install directly over Wi-Fi using `3dslink`:

```bash
3dslink CalculaThreeDS/out/CalculaThreeDS.3dsx
```

### 6. Build a CIA (optional)

A CIA lets you install the app to your home menu via FBI.

**Requirements:**
- `makerom` — download `makerom-vX.X.X-win_x86_64.zip` from
  [Project_CTR releases](https://github.com/3DSGuy/Project_CTR/releases),
  extract `makerom.exe` to `C:\devkitPro\tools\bin\` (already on PATH)
- `bannertool` — build from source at
  [Steveice10/bannertool](https://github.com/Steveice10/bannertool) or find
  a pre-built binary and place it on your PATH

Once both tools are on PATH:

```bash
cd CalculaThreeDS
make cia
```

Output: `CalculaThreeDS/out/CalculaThreeDS.cia`

Install the CIA using [FBI](https://github.com/Steveice10/FBI) on your 3DS.

---

## Project structure

```
Calcula3DS-App/
├── CalculaThreeDS/              # Base app (git submodule — LiquidFenrir)
│   ├── source/
│   │   ├── main.cpp             # Entry point, mode toggle, theme toggle, sleep hook
│   │   ├── theme.h / theme.cpp  # Dual-theme palette system (dark + light)  ← new
│   │   ├── graph_mode.h/.cpp    # f(x) graph mode                            ← new
│   │   ├── expr_parser.h/.cpp   # Recursive-descent expression parser        ← new
│   │   ├── ui_text.h/.cpp       # Text rendering utility for graph mode      ← new
│   │   ├── sleep_hook.h/.cpp    # APT sleep/resume hook                      ← new
│   │   ├── keyboard.h/.cpp      # Calculator keyboard UI and input
│   │   ├── equation.h/.cpp      # Equation renderer and evaluator
│   │   ├── number.h/.cpp        # Complex number display
│   │   └── text.h/.cpp          # Glyph atlas
│   ├── assets/                  # Sprite sheet sources (.t3s / .png)
│   ├── romfs/                   # Runtime assets (built sprite sheet)
│   ├── cia/                     # CIA build support files
│   │   └── app.rsf              # makerom RSF configuration
│   └── Makefile                 # Build system (adds `make cia` target)
├── INTEGRATION_NOTES.md         # Developer notes on the graph mode patch
└── README.md                    # This file
```

---

## What changed from the base app

All changes are additive files dropped into `CalculaThreeDS/source/`.
The submodule is not forked — the Makefile globs `*.cpp`/`*.h` automatically.

### New files
| File | Purpose |
|---|---|
| `theme.h / theme.cpp` | Runtime dual-theme palette; all colours in one place |
| `graph_mode.h/.cpp` | Graph mode: viewport, sampling, drawing, input |
| `expr_parser.h/.cpp` | Self-contained `f(x)` parser and evaluator |
| `ui_text.h/.cpp` | Text rendering for graph mode using existing glyph atlas |
| `sleep_hook.h/.cpp` | APT lifecycle hook (`std::atomic<bool>`, acquire/release) |

### Modified files
| File | Change |
|---|---|
| `main.cpp` | SELECT toggle, theme toggle (hold START + SELECT), sleep hook, dark default |
| `keyboard.cpp` | All colours → `Theme::*`; separate KEY_FACE / KEY_LABEL tints; "select graph" hint |
| `graph_mode.cpp` | All colours → `Theme::*` |
| `equation.cpp` | All colours → `Theme::*`; fixed missing `return` on `final_rpn.empty()` |
| `number.cpp` | All colours → `Theme::*` |
| `Makefile` | `-DARM11 -D_3DS` → `-D__3DS__`; added `make cia` target |

### Bug fixes
| File | Fix |
|---|---|
| `equation.cpp` | Missing `return` on `final_rpn.empty()` — silent crash risk |
| `graph_mode.cpp` | `px_to_x` now returns `double` (was `float`, lost precision before `eval()`) |
| `graph_mode.cpp` | `y_to_py` now takes `double` (was `float`, unnecessary precision loss) |
| `sleep_hook.h/.cpp` | `volatile bool` → `std::atomic<bool>` with acquire/release ordering |
| `keyboard.cpp` | Theme hint drawn before dim overlay (was drawn after — never dimmed) |
| `Makefile` | Compiler warning on every build (wrong define name) |

---

## Customising the theme

All colours live in `CalculaThreeDS/source/theme.cpp`. To add a third theme
or tweak any colour, edit the `set(DARK)` or `set(LIGHT)` block:

```cpp
// example: make the graph curve orange in dark mode
GRAPH_CURVE = C2D_Color32(255, 140, 0, 255);
```

Then `make clean && make`.

---

## Known limitations

- One function at a time — no multiple simultaneous graphs
- No degree/radian toggle (graph mode always uses radians)
- No root-finding, derivative, or integral overlays
- Axis tick labels use the sprite font — spaces render as gaps (cosmetic only)
- CIA build requires `bannertool` which has no pre-built Windows binary in the
  official repo; you must build it from source

---

## Credits

Original CalculaThreeDS by [LiquidFenrir](https://github.com/LiquidFenrir/CalculaThreeDS).  
Graph mode, theme system, and fixes by [Zushikina-kun](https://github.com/Zushikina-kun).
