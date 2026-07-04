
Lessons learned building CalculaThreeDS. A reference for future 3DS homebrew projects.
---

## 1. Always set explicit heap sizes

NEVER rely on libctru auto-split. Declare at global scope BEFORE main():
`"
+=cpp
u32 __ctru_heap_size = 16 * 1024 * 1024; // 16 MB regular heap (malloc/new)
u32 __ctru_linear_heap_size = 8 * 1024 * 1024; // 8 MB linear heap (GPU/DMA)
`"
+=
Why: if both are 0 (default), libctru auto-splits remaining memory.
APT_SetAppCpuTimeLimit() changes committed memory count, making remaining smaller.
allocateHeaps then calls svcBreak(USERBREAK_PANIC) -> crash before main() even runs.
This was the root cause of every CalculaThreeDS v1.0.0 through v1.0.2 crash.

Memory budget on Old 3DS (64MB total, ~40-48MB usable per app):

| Allocation | Recommended size |
|---|---|
| Stack (__stacksize__) | 512 KB |
| Code + data | ~350-500 KB typical |
| Regular heap (__ctru_heap_size) | 16 MB |
| Linear heap (__ctru_linear_heap_size) | 8 MB |
| OS/system overhead | ~16-20 MB |
| Total used | ~35-40 MB - safe for Old 3DS |

---

## 2. DO NOT call APT_SetAppCpuTimeLimit without explicit heap sizes

APT_SetAppCpuTimeLimit(30) is intended to grant New3DS syscore (extra CPU core) access.
Side effect on ALL 3DS models: modifies kernel committed memory count.
Without explicit heap sizes this causes allocateHeaps to panic.

Safe pattern if you need it:
`"
+=cpp
// At global scope - BEFORE main:
u32 __ctru_heap_size = 16 * 1024 * 1024;
u32 __ctru_linear_heap_size = 8 * 1024 * 1024;

// Then inside main():
APT_SetAppCpuTimeLimit(30); // safe now - heaps already committed at known sizes
`"
+=
For a simple app that does not need New3DS CPU speed, remove it entirely.

---

## 3. Build flags
`"
+=makefile
CXXFLAGS := $(CFLAGS) \ -fno-rtti -fno-exceptions -std=gnu++17
CFLAGS += -D__3DS__ # NOT -DARM11 or -D_3DS (those cause GCC warnings)
`"
+=

---

## 4. Never use throwing STL

With -fno-exceptions, throws corrupt the heap on Old 3DS.
Luma reports this as Undefined Instruction at a malloc-internal address.

| Instead of | Use |
|---|---|
| map.at(key) | auto it = map.find(key); if(it != end) ... |
| std::stod(s) | strtod(s.c_str(), &end) |
| std::stoi(s) | strtol(s.c_str(), &end, 10) |
| vector::at(i) | check i < size() first, then [] |
| try/catch | return error codes |

---

## 5. Luma3DS crash dump quick reference

Dumps saved to: SD:/luma/dumps/arm11/crash_dump_XXXXXXXX.dmp (588 bytes)

Key offsets in the dump:

| Offset | Type | Value |
|---|---|---|
| 0x00 | u32 | Magic 0xDEADC0DE |
| 0x04 | u32 | Magic 0xDEADCAFE |
| 0x08 | u8 | Processor (3=ARM11) |
| 0x09 | u8 | Core number |
| 0x0A | u8 | Exception: 0=FIQ 1=Undefined 2=Prefetch 3=DataAbort 4=IRQ |
| 0x50 | u32 x23 | Registers: r0-r12, sp, lr, pc, cpsr, dfsr, ifsr, far, fpexc, fpinst, fpinst2 |

PC address diagnosis:

| PC range | Meaning |
|---|---|
| 0x00100000 - 0x0014FFFF | Your app code - use arm-none-eabi-addr2line to find source line |
| 0x08000000 - 0x0E000000 | Heap area - memory setup issue or bad function pointer call |
| 0x0BFFEBE3 specifically | svcBreak(USERBREAK_PANIC) in allocateHeaps - heap init failed |
| Anything else | Corrupted PC - look at LR and the instruction stream in dump |

If all crash dumps are byte-for-byte identical: crash happens before main(), not in app code.
This means memory setup, global constructors, or the EH runtime is the culprit.

Decode a dump with PowerShell:
`"
+=powershell
 = [System.IO.File]::ReadAllBytes(path)
 = [0xA]
 = [BitConverter]::ToUInt32(, 0x50 + 15*4)
Write-Host (Exception:  PC: 0x{0:X8} -f )
`"
+=

Map PC to source line:
`"
+=bash
arm-none-eabi-addr2line -e out/MyApp.elf 0xYOUR_PC_VALUE
`"
+=

If addr2line returns ??:0, the PC is not in your code (kernel/heap/system area).

---

## 6. makerom v0.19 RSF format changes

Old ExHeader: block is completely removed in makerom v0.19.
All fields moved to AccessControlInfo and SystemControlInfo.

New requirements in v0.19:
- SaveDataSize must be a string: 0KB (not bare integer 0)
- SystemCallAccess is now required (mapping of SVC names to numbers)
- ServiceAccessControl is now required (list of service names like APT:U, fs:USER)
- FileSystemAccess names changed: SdApplication removed, use DirectSdmc

See CalculaThreeDS/cia/app.rsf for a complete working example.

Tools for CIA build (place both exe in devkitPro/tools/bin/):
- bannertool: github.com/diasurgical/bannertool/releases -> windows-x86_64/bannertool.exe
- makerom: github.com/3DSGuy/Project_CTR/releases -> makerom-vX.X.X-win_x86_64.zip

---

## 7. citro2d gotchas

- C2D_SpriteSheetLoad() returns NULL on failure. Always check before using.
- C2D_TargetClear() must be inside C3D_FrameBegin/End block.
- Off-screen render targets: redraw only on content change, not every frame.
- VRAM total: 6 MB. C3D_TexInitVRAM and sprite sheets both use VRAM.
- romfsInit()/romfsExit() bracket every file load. Do not leave mounted.

---

## 8. Old 3DS vs New 3DS

| Feature | Old 3DS | New 3DS |
|---|---|---|
| RAM | 64 MB | 256 MB |
| App CPU cores | 1 (core 0) | 2 (core 0 + syscore) |
| CPU speed | 268 MHz | 268 or 804 MHz |
| VRAM | 6 MB | 6 MB (same) |
| APT_SetAppCpuTimeLimit | Dangerous without explicit heaps | Grants syscore |

Develop with Old 3DS constraints. Code that works on New3DS may crash on Old3DS.

---

## 9. New project checklist

- [ ] __stacksize__, __ctru_heap_size, __ctru_linear_heap_size set explicitly at global scope
- [ ] -fno-exceptions -fno-rtti in CXXFLAGS
- [ ] -D__3DS__ (not -DARM11 or -D_3DS)
- [ ] .find() instead of .at() everywhere
- [ ] strtod/strtol instead of std::stod/stoi
- [ ] Null-check C2D_SpriteSheetLoad result
- [ ] APT_SetAppCpuTimeLimit only after explicit heap sizes, or removed entirely
- [ ] romfsInit/Exit around all file loads
- [ ] app.rsf uses makerom v0.19 format (SaveDataSize: 0KB, SystemCallAccess required)
- [ ] Test in Citra before hardware
- [ ] PC in 0x08000000+ on crash = heap setup failure, not app code
