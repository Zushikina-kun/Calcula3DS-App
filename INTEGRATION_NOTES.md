# Integration notes - Calcula3DS-App

Full record of every change, design decision, and testing checklist.

---

## Status

Fully building. make clean && make and make cia both pass under devkitARM GCC 16.

---

## Session history

Session 1 - Graph mode + theme: added graph_mode, expr_parser, ui_text, sleep_hook,
theme.h/cpp (dark default), wired into main.cpp, added make cia.

Session 2 - Scientific expansion + CIA build:
- Added calcmode.h/cpp (DEG/RAD) and cplxmode.h/cpp (RECT/POLAR)
- More keyboard page implemented (was empty - crash risk)
- 10 new evaluators: fact floor ceil round nCr nPr mod logn pol rec
- CalcMode wired into all trig and inverse trig evaluators
- Polar display in number.cpp via CplxMode::is_polar()
- All new button labels registered in text.cpp menu atlas
- bannertool + makerom downloaded; banner.png and banner.wav generated
- cia/app.rsf rewritten for makerom v0.19
- Fixed ARM gcc 16 error: .real() on plain double in acos/asin/atan

---

## New files

| File | Purpose |
|---|---|
| theme.h/.cpp | Runtime dual-theme palette. All colours in one place. |
| calcmode.h/.cpp | Angle mode - RAD default. to_rad()/from_rad() for all trig and pol/rec. |
| cplxmode.h/.cpp | Complex display - RECT default. is_polar() queried in number.cpp. |
| graph_mode.h/.cpp | f(x) plotter: transforms, dirty-flag sampling, citro2d draw, swkbd. |
| expr_parser.h/.cpp | Recursive-descent parser. kMaxDepth=32, 256 node budget. |
| ui_text.h/.cpp | Thin wrapper over TextMap equ atlas. Does not touch text.cpp. |
| sleep_hook.h/.cpp | APT lifecycle hook using std::atomic<bool> acquire/release. |

---

## Modified files

main.cpp: Theme::set(DARK) at startup. SELECT toggles modes. Hold START+SELECT toggles theme.
START tap < 800ms quits. lifecycle.consume_resume_flag() -> graph.invalidate().

keyboard.cpp: Added calcmode.h + cplxmode.h. toggle_angle_mode() and toggle_cplx_mode()
added. More page: row0=fact nCr nPr mod logn, row1=floor ceil round pol rec,
row2=cplx(col3), row3=deg(col4). Gap bar: deg/rad and pol indicators.
Separate KEY_LABEL tint (labels were invisible in dark mode).

equation.cpp: CalcMode::to_rad() on trig inputs, from_rad() on inverse trig.
Removed .real() from acos/asin/atan (ARM gcc 16 error - these return plain double).
New single-arg: fact, floor, ceil, round.
New two-arg: nCr, nPr, mod, logn, pol, rec.
sci_pol(x,y)=Number(hypot(x,y), from_rad(atan2(y,x)))
sci_rec(r,t)=Number(r*cos(to_rad(t)), r*sin(to_rad(t)))

number.cpp: Added cplxmode.h + calcmode.h. Polar render when is_polar() and imag!=0.
Renders as r>theta. Buffer 64->80 bytes.

text.cpp: New menu atlas entries for fact nCr nPr mod logn floor ceil round deg pol rec cplx.
Composed from existing letter sprites - no new sprites needed.

Makefile: -DARM11 -D_3DS -> -D__3DS__. make cia target added.

cia/app.rsf: Rewritten for makerom v0.19. ExHeader removed, fields in AccessControlInfo/
SystemControlInfo. SaveDataSize: 0KB. SystemCallAccess + ServiceAccessControl added.

---

## Design notes

pol(x,y) packs result as Number(r, theta). CplxMode::POLAR renders as r>theta.
The > (assign arrow) is the angle separator - the atlas has no angle symbol.

expr_parser is separate from Equation because Equation is coupled to Keyboard internals.

---

## Bug fixes

| File | Bug | Fix |
|---|---|---|
| equation.cpp | Missing return on final_rpn.empty() | return make_pair(Number{}, true) |
| equation.cpp | .real() on double from acos/asin/atan | Removed .real() |
| graph_mode.cpp | px_to_x returned float | Changed to double |
| graph_mode.cpp | y_to_py took float | Changed to double |
| sleep_hook.h | volatile bool | std::atomic<bool> acquire/release |
| keyboard.cpp | Hint drawn after dim overlay | Moved before overlay |
| Makefile | Wrong define | -D__3DS__ |
| cia/app.rsf | Old ExHeader format | Rewritten for makerom v0.19 |

---

## Testing checklist

1. make clean && make - zero warnings
2. make cia - cia produced
3. Citra before hardware
4. Theme toggle: hold START+SELECT; START tap quits
5. DEG: sin(90)=1, asin(1)=90
6. RAD: sin(pi/2)=1
7. More page reachable via L/R
8. pol(3,4)=5>53.13 (deg) or 5>0.927 (rad)
9. rec(5,53.13)=3+4i in DEG
10. cplx toggle: sqrt(-1) -> polar, again -> a+bi
11. fact(10)=3628800, nCr(10,3)=120
12. Graph sin( -> red error
13. Graph 1/x and tan(x): gaps at discontinuities
14. HOME -> return -> graph redraws
15. CIA installs and launches

---

## Next priorities

1. Fix anything hardware testing finds
2. DEG/RAD in graph mode
3. Multiple simultaneous graph functions
4. Statistics mode
5. Unit conversion page
6. Table mode - f(x) table on top screen
7. Base-N mode
