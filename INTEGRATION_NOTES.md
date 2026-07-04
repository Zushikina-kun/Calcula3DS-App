# Integration notes - Calcula3DS-App

Full record of every change, design decision, and testing checklist.

---

## Status

v1.0.3 - startup crash fixed. make clean && make and make cia pass under devkitARM GCC 16.

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

**Session 4 - Crash fix round 1 (v1.0.1)**
Replaced all std::map::at() with .find() + null check.
Fixed equation.cpp, keyboard.cpp, number.cpp. Added -fno-exceptions.
Replaced try/catch in expr_parser.cpp with strtod.

**Session 5 - Root cause crash fix (v1.0.3)**
Identified true crash cause from Luma3DS dump analysis:
- All 4 dumps (150-153) were identical: PC=0x0BFFEBE3, same registers
- 0x0BFFEBE3 is inside OS_HEAP_AREA (0x08000000-0x0E000000) - NOT app code
- This is svcBreak(USERBREAK_PANIC) called by libctru allocateHeaps.c
- APT_SetAppCpuTimeLimit(30) changes committed memory count in kernel resource limit
- allocateHeaps computes remaining = maxCommit - currentCommit
- With modified currentCommit, remaining was too small
- allocateHeaps panics: svcBreak(USERBREAK_PANIC)
Fix: remove APT_SetAppCpuTimeLimit, set explicit heap sizes (16MB + 8MB)

---

## Complete bug fix log

| File | Bug | Fix |
|---|---|---|
| main.cpp | APT_SetAppCpuTimeLimit caused svcBreak in allocateHeaps (ALL CRASHES) | Removed |
| main.cpp | Heap sizes not set - auto-split dangerous on Old 3DS | __ctru_heap_size=16MB __ctru_linear_heap_size=8MB |
| keyboard.cpp | menu.at() throws on More page draw | menu.find() + continue |
| equation.cpp | equ.at() throws on any unknown char | equ.find() + skip draw |
| number.cpp | equ.at() throws on any unknown char | equ.find() + skip draw |
| expr_parser.cpp | try/catch with -fno-exceptions | strtod |
| Makefile | -fexceptions default | -fno-exceptions added |
| equation.cpp | Missing return on final_rpn.empty() | Fixed |
| equation.cpp | .real() on double from acos/asin/atan | Removed |
| graph_mode.cpp | Dead py_to_y() | Removed |
| keyboard.cpp | Dead YES_TINT/NO_TINT never read | Removed |
| main.cpp | Theme toggle left textures with old colours | kb.invalidate() |
| Makefile | Wrong define -DARM11 -D_3DS | -D__3DS__ |
| cia/app.rsf | Old ExHeader format | Rewritten for makerom v0.19 |

---

## Why the crash dumps all looked identical

Every dump: PC=0x0BFFEBE3, r11=0xEE201A20, sp=0xE311000F, same hex bytes.
This is deterministic because svcBreak is a fixed kernel stub address.
The crash happened before main() on every run, so no app code variation was possible.
v1.0.1 and v1.0.2 fixes did not reach the crash site because they were in app code,
but the crash was in the pre-main startup sequence.

---

## Testing checklist (v1.0.3)

1. App launches without crash on Old 3DS
2. Basic arithmetic works
3. Navigate to More page (L/R) - no crash
4. fact(10)=3628800, nCr(10,3)=120
5. pol(3,4) and rec - correct results
6. DEG/RAD toggle works, indicator shows in gap bar
7. cplx toggle changes display format
8. Graph mode works (SELECT to switch)
9. Theme toggle (hold START + SELECT) repaints all screens
10. 12-entry memory history, scrolling works

---

## Next priorities

1. Hardware test v1.0.3 thoroughly
2. DEG/RAD in graph mode (currently always radians)
3. Multiple simultaneous graph functions
4. Statistics mode (mean, std dev, variance)
5. Unit conversion page
6. Table mode
7. Base-N mode
