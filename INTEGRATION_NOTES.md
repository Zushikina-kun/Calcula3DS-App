# Integration notes - Calcula3DS-App

Full record of every change, design decision, and testing checklist.

---

## Status

Fully building. make clean && make and make cia pass under devkitARM GCC 16.
Crash from v1.0.0 identified and fixed. See Session 4 below.

---

## Session history

**Session 1 - Graph mode + theme**
Added graph_mode, expr_parser, ui_text, sleep_hook, theme.h/cpp. Dark default.

**Session 2 - Scientific expansion + CIA**
Added calcmode, cplxmode. More keyboard page. 10 new evaluators.
Trig deg/rad. Polar display. CIA build. makerom v0.19 RSF.

**Session 3 - Audit and cleanup**
Removed dead YES_TINT code. Removed dead py_to_y(). Added Keyboard::invalidate().
Theme toggle now repaints all textures. Rad-only warning in graph mode.
Fixed .gitignore for CIA build artifacts.

**Session 4 - Crash fix (v1.0.0 crash dump analysis)**
Exception type 1 (Undefined Instruction) at PC=0x0BFFEBE3 (newlib malloc internals).
Root cause: std::map::at() throws std::out_of_range, corrupting the heap during
exception unwinding on the 3DS. Three call sites affected:
 1. keyboard.cpp draw(): menu.at(scr.buttons[y][x]) when drawing More page button labels
 2. equation.cpp render_parts(): equ.at(char) when rendering user-entered chars
 3. number.cpp render(): equ.at(char) when rendering the answer bar
All three fixed by replacing .at() with .find() + null check.

---

## Complete bug fix log

| File | Bug | Fix |
|---|---|---|
| keyboard.cpp | menu.at() throws on More page draw - CONFIRMED CRASH | menu.find() + continue |
| equation.cpp | equ.at() throws on any char not in atlas | equ.find() + skip draw |
| number.cpp | equ.at() throws on any char not in atlas | equ.find() + skip draw |
| keyboard.cpp | equ.at() on screen name chars | equ.find() + skip |
| equation.cpp | Missing return on final_rpn.empty() | return make_pair(Number{}, true) |
| equation.cpp | .real() on double from acos/asin/atan | Removed .real() |
| graph_mode.cpp | Dead py_to_y() never called | Removed |
| keyboard.cpp | Dead YES_TINT/NO_TINT set once, never read | Removed |
| main.cpp | Theme toggle left equation/memory textures with old colours | kb.invalidate() |
| sleep_hook.h | volatile bool | std::atomic<bool> acquire/release |
| keyboard.cpp | Hint drawn after dim overlay | Moved before overlay |
| Makefile | Wrong compiler define -DARM11 -D_3DS | -D__3DS__ |
| cia/app.rsf | Old ExHeader format | Rewritten for makerom v0.19 |
| .gitignore | banner.bin icon.icn not excluded | Added |
| cplxmode.h | Unicode angle symbol in comment | Changed to r>theta |

---

## Why .at() crashes on 3DS

std::map::at() throws std::out_of_range when the key is not found.
On the Old 3DS, C++ exception unwinding is unreliable under the 3DS exception handler.
The throw corrupts stack/heap state, which then manifests as an Undefined Instruction
exception inside newlib malloc when the next heap allocation occurs.
The crash dump showed PC inside malloc free-list code with corrupted frame pointer
(r11=0xEE201A20, a VFP instruction encoding, not a valid address).
Rule going forward: never use .at() anywhere in this codebase. Always use .find().

---

## Testing checklist (v1.0.1)

1. make clean && make - zero warnings
2. make cia - cia produced
3. Navigate to More page (L/R) - must not crash
4. Press fact, nCr, nPr, mod, logn - buttons must insert text
5. Calculate nCr(10,3) - must return 120
6. Press deg - indicator switches to DEG
7. Press cplx - display toggles to r>theta on complex results
8. Theme toggle - all screens repaint immediately
9. DEG mode in graph - rad-only warning visible top-right
10. Graph sin( - red error, not crash
11. CIA installs and launches from home menu
12. No crash after extended use (type, calculate, scroll memory)

---

## Next priorities

1. Hardware test the v1.0.1 build thoroughly
2. DEG/RAD mode in graph mode
3. Multiple simultaneous graph functions
4. Statistics mode
5. Unit conversion page
6. Table mode
7. Base-N mode
