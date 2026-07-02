# Integration notes - Calcula3DS-App

Full record of every change, design decision, and testing checklist.

---

## Status

Fully building. make clean && make and make cia both pass under devkitARM GCC 16.
All known bugs fixed. Hardware testing still needed.

---

## Session history

**Session 1 - Graph mode + theme system**
Added graph_mode, expr_parser, ui_text, sleep_hook.
Added theme.h/cpp (dark default). Wired into main.cpp. Added make cia.

**Session 2 - Scientific expansion + deg/rad + complex display + CIA**
Added calcmode.h/cpp and cplxmode.h/cpp.
Implemented More keyboard page (was declared but empty - crash risk).
Added 10 new evaluators: fact floor ceil round nCr nPr mod logn pol rec.
Wired CalcMode into trig/inverse trig. Polar display in number.cpp.
Registered all new labels in text.cpp menu atlas.
Downloaded bannertool+makerom, generated banner assets.
Rewrote cia/app.rsf for makerom v0.19.
Fixed ARM gcc 16 error: .real() on plain double in acos/asin/atan.

**Session 3 - Audit and final fixes**
Removed dead YES_TINT/NO_TINT/TEXT_YES_TINT/TEXT_NO_TINT code from keyboard.cpp.
Removed dead py_to_y() from graph_mode.h and graph_mode.cpp.
Added Keyboard::invalidate() - force repaint of equation/memory textures.
Called kb.invalidate() in main.cpp on theme toggle (bug: textures kept old colours).
Added rad-only warning to graph mode top screen when calculator is in DEG mode.
Fixed cia/app.rsf gitignore: excluded banner.bin, icon.icn.
Fixed cplxmode.h comment: was showing unicode angle symbol.

---

## All source files

| File | Purpose |
|---|---|
| theme.h/.cpp | Runtime dual-theme palette. All colours in one place. |
| calcmode.h/.cpp | Angle mode RAD/DEG. to_rad()/from_rad() used by all trig and pol/rec. |
| cplxmode.h/.cpp | Complex display RECT/POLAR. is_polar() queried in number.cpp at render. |
| graph_mode.h/.cpp | f(x) plotter: transforms, dirty-flag sampling, citro2d draw, swkbd. |
| expr_parser.h/.cpp | Recursive-descent parser. kMaxDepth=32, 256 node budget, no alloc during eval. |
| ui_text.h/.cpp | Thin wrapper over TextMap equ atlas. Does not touch text.cpp. |
| sleep_hook.h/.cpp | APT lifecycle hook using std::atomic<bool> acquire/release. |

---

## Key changes to existing files

**main.cpp**
Theme::set(DARK) at startup. SELECT toggles modes. Hold START+SELECT toggles theme.
kb.invalidate() called on theme toggle so equation/memory textures repaint.
START tap < 800ms quits. lifecycle.consume_resume_flag() -> graph.invalidate().

**keyboard.cpp**
All COLOR_* -> Theme::*. toggle_angle_mode() and toggle_cplx_mode() added.
More page: row0=fact nCr nPr mod logn, row1=floor ceil round pol rec,
row2=cplx(col3), row3=deg(col4).
Gap bar: deg/rad indicator left side; pol appended when polar display active.
Separate KEY_LABEL tint (labels were invisible in dark mode).
Keyboard::invalidate(): sets any_change=true, redo_top=true, redo_bottom=true.
Removed dead YES_TINT/NO_TINT/TEXT_YES_TINT/TEXT_NO_TINT - set once, never read.

**graph_mode.cpp**
All colours -> Theme::GRAPH_*. Removed dead py_to_y() (never called).
Added rad-only warning (top-right corner) when CalcMode::is_degrees() is true.

**equation.cpp**
CalcMode::to_rad() on trig inputs. CalcMode::from_rad() on inverse trig.
Removed .real() from acos/asin/atan (ARM gcc 16 error - plain double return).
New single-arg: fact, floor, ceil, round.
New two-arg: nCr, nPr, mod, logn, pol, rec.
sci_pol(x,y) = Number(hypot(x,y), from_rad(atan2(y,x)))
sci_rec(r,t) = Number(r*cos(to_rad(t)), r*sin(to_rad(t)))

**number.cpp**
Polar render: when CplxMode::is_polar() and imag!=0, renders as r>theta.
Buffer 64->80 bytes.

**text.cpp**
New menu atlas entries: fact nCr nPr mod logn floor ceil round deg pol rec cplx.

**Makefile**
-DARM11 -D_3DS -> -D__3DS__. make cia target added.

**cia/app.rsf**
Rewritten for makerom v0.19. ExHeader removed. SaveDataSize: 0KB.
SystemCallAccess and ServiceAccessControl added (required by v0.19).

---

## Complete bug fix log

| File | Bug | Fix |
|---|---|---|
| equation.cpp | Missing return on final_rpn.empty() | return make_pair(Number{}, true) |
| equation.cpp | .real() on double from acos/asin/atan | Removed .real() |
| graph_mode.cpp | px_to_x returned float | Changed to double |
| graph_mode.cpp | y_to_py took float | Changed to double |
| graph_mode.cpp | Dead py_to_y() never called | Removed |
| keyboard.cpp | Dead YES_TINT etc set but never read | Removed |
| keyboard.cpp | Hint drawn after dim overlay | Moved before overlay |
| main.cpp | Theme toggle left equation/memory textures with old colours | kb.invalidate() added |
| sleep_hook.h | volatile bool | std::atomic<bool> acquire/release |
| Makefile | Wrong compiler define | -D__3DS__ |
| cia/app.rsf | Old ExHeader format | Rewritten for makerom v0.19 |
| cplxmode.h | Unicode angle symbol in comment | Changed to r>theta |
| .gitignore | banner.bin, icon.icn not excluded | Added |

---

## Design notes

pol(x,y) packs result as Number(r, theta) - fits the complex type losslessly.
CplxMode::POLAR renders as r>theta. The > (assign arrow sprite) is the angle separator
since the glyph atlas has no angle symbol.

expr_parser is separate from Equation because Equation is coupled to Keyboard internals.

Keyboard::invalidate() sets any_change + redo_top + redo_bottom so all three
off-screen textures (equation, top memory slot, bottom memory slot) repaint.

---

## Testing checklist

1. make clean && make - zero warnings
2. make cia - cia produced
3. Citra before hardware
4. Theme toggle: screens AND equation texture switch colours immediately
5. DEG: sin(90)=1, asin(1)=90
6. RAD: sin(pi/2)=1
7. Graph in DEG mode: rad-only warning visible top-right
8. More page reachable via L/R, all 10 function buttons + cplx + deg visible
9. pol(3,4) = 5>53.13 (deg) or 5>0.927 (rad)
10. rec(5,53.13) = 3+4i in DEG mode
11. cplx toggle: sqrt(-1) -> polar form; again -> a+bi
12. fact(10)=3628800, nCr(10,3)=120, mod(10,3)=1
13. Graph sin( -> red error, not crash
14. Graph 1/x and tan(x): gaps at discontinuities, no vertical lines
15. HOME -> return -> graph redraws correctly
16. CIA: installs and launches from home menu

---

## Next priorities

1. Fix anything hardware testing finds
2. DEG/RAD mode in graph mode (currently always radians)
3. Multiple simultaneous graph functions
4. Statistics mode (mean, std dev, variance)
5. Unit conversion page
6. Table mode - f(x) table on top screen
7. Base-N (hex/oct/bin) mode
