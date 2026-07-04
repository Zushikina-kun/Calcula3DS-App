
Complete reference from building CalculaThreeDS. Use before starting any new 3DS project.

---

## Cross-examination: Working 3DS apps vs our app

Cloned and examined three reference apps to understand why ours crashed:

### GraphCalc3DS (flarn2006)
- URL: github.com/flarn2006/GraphCalc3DS
- Library: **sf2d** (NOT citro2d/citro3d - completely different GPU layer)
- Language: C++11 with std::vector, std::string
- CXXFLAGS: -fno-rtti -fno-exceptions -std=gnu++11
- ctru_heap_size: NOT SET (uses auto-split)
- stacksize: NOT SET (uses 32KB default)
- APT_SetAppCpuTimeLimit: NO
- Static doubles: ViewWindow view(-5.0, 5.0, -3.0, 3.0) -- literal values, NOT computed
- Init: sf2d_init() then romfsInit() then font loads then romfsExit()
- Key insight: uses sf2d not citro2d, so -fno-exceptions works fine for it

### Original CalculaThreeDS (LiquidFenrir) - our base
- URL: github.com/LiquidFenrir/CalculaThreeDS
- Library: citro2d/citro3d
- Language: C++17 with std::map, std::vector, std::unique_ptr
- CXXFLAGS: **-fno-rtti only** (exceptions ENABLED - this is the default)
- ctru_heap_size: NOT SET
- stacksize: 512KB
- APT_SetAppCpuTimeLimit: YES (30) - this was our original crash cause
- equation.cpp: static const Number E_VAL(std::exp(1.0)) -- VFP at static init
- Init: gfxInit -> C3D_Init -> C2D_Init -> APT_SetAppCpu -> romfsInit -> sprites
- Key insight: worked originally probably because exceptions handled the VFP issue,
 or was never tested on Old 3DS hardware

### devkitPro gpusprites example (official reference)
- Library: citro2d/citro3d
- ctru_heap_size: NOT SET
- APT_SetAppCpuTimeLimit: NO
- Exceptions: ENABLED (default)
- Init: romfsInit -> gfxInit -> C3D_Init -> C2D_Init -> SpriteSheetLoad -> romfsExit

### Summary comparison table

| App | Lib | Exceptions | ctru_heap | APT_SetCpu | Works |
|---|---|---|---|---|---|
| GraphCalc3DS | sf2d | -fno-exceptions | auto-split | NO | YES |
| Original CalculaThreeDS | citro2d | ENABLED | auto-split | YES | ? |
| devkitPro examples | citro2d | ENABLED | auto-split | NO | YES |
| Our v1.0.1-v1.0.5 | citro2d | -fno-exceptions | explicit | NO | NO - crashes |
| Our v1.0.6 | citro2d | ENABLED | explicit | NO | TBD |

### Key lessons

1. **-fno-exceptions is incompatible with citro2d on Old 3DS.**
 sf2d-based apps can use it fine. citro2d apps should use default exceptions.

2. **-fno-exceptions does NOT remove __eh_globals_init from the binary.**
 It changes how pthread_key_create and other runtime init behaves.
 This subtly breaks citro2d initialization.

3. **The safe .at() -> .find() fix is still correct and needed.**
 Even with exceptions enabled, uncaught exceptions on 3DS cause crashes.
 Use .find() everywhere regardless.

4. **VFP in static initializers is still dangerous on Old 3DS.**
 Deferred E_VAL/PI_VAL/I_VAL via ensure_constants() stays in place.

5. **If porting a working citro2d app: keep its original Makefile flags.**
 Do not add -fno-exceptions if the original did not have it.

---

## 1. THE most important rule: no VFP in static initializers

Static const globals with float/double expressions run BEFORE main().
On Old 3DS VFP state at static init time is unreliable.

BAD:
`"
+=cpp
static const double E = std::exp(1.0); // vldr in _GLOBAL__sub_I -> crash
`"
+=

GOOD - defer to first use inside main():
`"
+=cpp
static double E_val = 0.0;
static bool init_done = false;
void ensure_constants() {
 if(init_done) return;
 E_val = std::exp(1.0); // safe here
 init_done = true;
}
`"
+=

Check for VFP statics:
`"
+=bash
arm-none-eabi-objdump -d app.elf | grep -A15 _GLOBAL__sub_I
# Look for vldr or vstrd instructions
`"
+=

---

## 2. Exception handling: match original app flags

For citro2d/citro3d apps: leave exceptions ENABLED (default, do not add -fno-exceptions)
For sf2d apps: -fno-exceptions is fine

Even with exceptions enabled, never use .at() -- uncaught exceptions still crash.
Always use .find() instead of .at() everywhere.

---

## 3. Set explicit heap sizes before main()
`"
+=cpp
u32 __ctru_heap_size = 16 * 1024 * 1024; // 16 MB
u32 __ctru_linear_heap_size = 8 * 1024 * 1024; // 8 MB
`"
+=

---

## 4. DO NOT use APT_SetAppCpuTimeLimit

Changes kernel committed memory count. Without explicit heap sizes it causes svcBreak.
Only use if you need New3DS syscore AND have explicit heap sizes set first.

---

## 5. Required build flags
`"
+=makefile
CXXFLAGS := \ -fno-rtti -std=gnu++17 # no -fno-exceptions for citro2d apps
CFLAGS += -D__3DS__ # NOT -DARM11 or -D_3DS
`"
+=

---

## 6. Never use throwing STL

| Instead of | Use |
|---|---|
| map.at(key) | auto it = map.find(key); if(it != end()) ... |
| std::stod(s) | strtod(s.c_str(), &endp) |
| vector::at(i) | check i < size() first, then [] |

---

## 7. Correct initialization order in main()
`"
+=cpp
// For citro2d apps, match devkitPro example order:
romfsInit();
C2D_SpriteSheet sprites = C2D_SpriteSheetLoad(path);
romfsExit();
if(!sprites) return 1;
gfxInitDefault();
C3D_Init(C3D_DEFAULT_CMDBUF_SIZE);
C2D_Init(C2D_DEFAULT_MAX_OBJECTS);
C2D_Prepare();
`"
+=
Do NOT call consoleDebugInit(debugDevice_SVC) in production.

---

## 8. Crash dump analysis

Dumps: SD:/luma/dumps/arm11/crash_dump_XXXXXXXX.dmp (588 bytes)
Exception type 1 = Undefined Instruction (most common for app crashes)

PC range -> cause:
 0x00100000 - 0x0014FFFF : your app code -> use arm-none-eabi-addr2line
 0x08000000 - 0x0E000000 : heap area -> VFP static init or svcBreak in allocateHeaps
 0x0BFFEBE3 specifically : both causes land here on Old 3DS

If ALL dumps are identical: crash is before main().
Check: arm-none-eabi-objdump for vldr in _GLOBAL__sub_I, and check heap setup.

---

## 9. makerom v0.19 RSF

- SaveDataSize: 0KB (not bare 0)
- IdealProcessor: 0 (not 1 - that is system core)
- ReleaseKernelMinor: 00 (not 33)
- SystemCallAccess and ServiceAccessControl: required
- DirectSdmc (not SdApplication)

---

## 10. Old 3DS vs New 3DS

| Feature | Old 3DS | New 3DS |
|---|---|---|
| RAM | 64 MB | 256 MB |
| App CPU cores | 1 | 2 |
| VFP at static init | Unreliable | Usually fine |
| -fno-exceptions + citro2d | Breaks | May work |

Always develop and test with Old 3DS constraints.

---

## 11. New project checklist

- [ ] No VFP (float/double computed) in static const globals
- [ ] Check arm-none-eabi-objdump for vldr in _GLOBAL__sub_I
- [ ] Explicit heap sizes at global scope
- [ ] No -fno-exceptions for citro2d apps
- [ ] -D__3DS__ (not -DARM11 or -D_3DS)
- [ ] .find() not .at() everywhere
- [ ] romfsInit before sprite load; null-check result
- [ ] No APT_SetAppCpuTimeLimit unless explicit heaps pre-set
- [ ] No consoleDebugInit(debugDevice_SVC) in production
- [ ] IdealProcessor:0 in app.rsf
- [ ] Test in Citra first, then real hardware
- [ ] Identical crash dumps = pre-main crash


---

## 12. If 3DSX works but CIA crashes - RSF parameters wrong

The CIA and 3DSX are the same code but have different process setup.
If the 3DSX works fine but the CIA crashes instantly:

The kernel is rejecting the CIA process parameters from app.rsf BEFORE your code runs.
The crash dumps will always be identical (same PC, same registers every time).

Required process parameters for a working citro2d homebrew CIA (from cross-examination):

| Parameter | Correct value |
|---|---|
| Priority | 16 (not 0x30) |
| MaxCpu | 0 (let system decide) |
| CanWriteSharedPage | true |
| CanUseNonAlphabetAndNumber | true |
| PermitMainFunctionArgument | true |
| CanShareDeviceMemory | true |
| SpecialMemoryArrange | true |
| ReleaseKernelMinor | 33 |
| Dependency list | required - include dsp, gsp, hid, cfg, ptm, apt |

See CalculaThreeDS/cia/app.rsf for a complete working example.