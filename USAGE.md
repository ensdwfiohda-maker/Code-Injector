# Usage guide — CodeInjector

**English** | [Русский](USAGE.ru.md) · ← back to [README](README.md)

Step-by-step: build, configure, inject, verify, troubleshoot.

> Use only on your own processes or with explicit permission. See the disclaimer in the [README](README.md).

## Contents

1. [Prerequisites](#1-prerequisites)
2. [Build](#2-build)
3. [File layout](#3-file-layout)
4. [Config reference](#4-config-reference)
5. [Running](#5-running)
6. [What success looks like](#6-what-success-looks-like)
7. [The demo DLL and the 15-second rule](#7-the-demo-dll-and-the-15-second-rule)
8. [Independent verification](#8-independent-verification)
9. [Your own DLL](#9-your-own-dll)
10. [Troubleshooting](#10-troubleshooting)
11. [FAQ](#11-faq)

---

## 1. Prerequisites

- Windows x64.
- A C++17 compiler: **MinGW-w64** (`g++`) or **MSVC** (x64 toolchain).
- A **running x64 target** you own — `notepad.exe` is the safest lab rat.
- Administrator rights **only** if the target itself is elevated.
- Everything in the chain must be x64: injector, DLL, target.

## 2. Build

### MinGW-w64

```bash
# injector (static → no libstdc++/libgcc DLLs needed)
g++ -std=c++17 -O2 -Wall -Wextra -m64 -static -o CodeInjector.exe CodeInjector.cpp

# demo DLL (static → no MinGW runtime needed inside the target)
g++ -std=c++17 -O2 -m64 -shared -static -o Inject.dll Inject.cpp -luser32
```

`-static` on the DLL matters: the target resolves dependencies using **its own** search paths, and it usually has no idea where your MinGW `bin` folder is. A dynamically linked demo DLL will often fail with `LoadLibraryA returned NULL`.

### MSVC

Run from **x64 Native Tools Command Prompt for VS**:

```bat
cl /nologo /std:c++17 /EHsc /O2 /W4 /MT CodeInjector.cpp /Fe:CodeInjector.exe
cl /nologo /std:c++17 /EHsc /O2 /MT /LD Inject.cpp /Fe:Inject.dll /link user32.lib
```

`/MT` = static CRT, so the target does not need the VC++ redistributable.

### CMake (optional)

```cmake
cmake_minimum_required(VERSION 3.15)
project(CodeInjector LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)

add_executable(CodeInjector CodeInjector.cpp)
add_library(Inject SHARED Inject.cpp)
set_target_properties(Inject PROPERTIES PREFIX "" OUTPUT_NAME "Inject")
target_link_libraries(Inject PRIVATE user32)

if (MSVC)
  set_property(TARGET CodeInjector Inject PROPERTY MSVC_RUNTIME_LIBRARY "MultiThreaded")
endif()
```

```bash
cmake -B build -A x64
cmake --build build --config Release
```

## 3. File layout

Keep all three files in one folder, with a **short ASCII-only path**:

```
C:\Tools\CodeInjector\
├── CodeInjector.exe
├── Inject.dll            ← name is hardcoded, must be exactly this
└── CodeInjectcfg.cfg     ← optional
```

Two reasons to keep the path boring:

- it is passed to `LoadLibraryA` (**ANSI**), so non-ASCII characters like `C:\Users\Вася\...` can break loading;
- the exe directory is read into a `MAX_PATH` buffer, so paths beyond 260 characters get truncated.

## 4. Config reference

`CodeInjectcfg.cfg`, plain `key=value`:

```ini
# comment with '#'
; comment with ';'

target = notepad.exe
```

| Rule | Behaviour |
|---|---|
| Comments | line whose first non-space char is `#` or `;` is ignored |
| Blank lines | ignored |
| Whitespace | trimmed around key and value |
| Key case | insensitive (`Target`, `TARGET` work) |
| Duplicates | **first** occurrence wins |
| Supported keys | `target` only |
| Value | exe name (`notepad.exe`) **or** decimal PID (`12345`) |
| Lines without `=` | ignored |

Missing file / missing key / empty value → interactive prompt. The DLL path is **not** configurable.

## 5. Running

Start the target first, then launch `CodeInjector.exe` (double-click or from a terminal).

**Scenario A — target from config**

```ini
target = notepad.exe
```

**Scenario B — exact PID from config**

```ini
target = 4242
```

Useful when several instances exist and you always want the same one. Get the PID from Task Manager → Details.

**Scenario C — no config, interactive**

```
[>] Enter process name (e.g. notepad.exe) or numeric PID: notepad.exe
```

**Scenario D — several instances**

```
[!] Multiple instances found:
  [0]  notepad.exe  PID=12345
  [1]  notepad.exe  PID=23456
[>] Enter index [0..1]: 1
[+] Selected: notepad.exe  PID=23456
```

**Scenario E — elevated target**

Right-click `CodeInjector.exe` → *Run as administrator*. Without it you get `OpenProcess failed: err=5`.

**Exit codes:** `0` = success, `1` = any failure. Note the final `Press Enter to exit...` — the program always waits for a keypress before exiting.

## 6. What success looks like

```
=========================================
  CodeInjector  -  x64 DLL Injector
=========================================
[*] Injector, Inject.dll, and target must all be x64.
[*] Elevated target? Run this injector as Administrator.

[*] Exe directory : C:\Tools\CodeInjector
[*] DLL path      : C:\Tools\CodeInjector\Inject.dll
[+] Inject.dll found.

[*] Config target: "notepad.exe"
[*] Searching for process: "notepad.exe" ...
[+] Found: notepad.exe  PID=12345

[*] Injecting "C:\Tools\CodeInjector\Inject.dll"
    into PID=12345 ...

[*] Opening process PID=12345 ...
[+] Process handle obtained.
[+] Remote buffer allocated at 0x000001f2a3b40000
[+] DLL path written to target memory.
[+] LoadLibraryA @ 0x00007ffb1c2d5a10
[+] Remote thread created. Waiting (timeout 15 s) ...
[+] LoadLibraryA returned HMODULE=0x1c2d0000  - injection confirmed!

[+] SUCCESS.
```

The printed `HMODULE` is truncated to 32 bits (`GetExitCodeThread` returns a `DWORD`) — a logging quirk, not a failure.

## 7. The demo DLL and the 15-second rule

`Inject.cpp` is a **minimal example**, not a template for production:

```cpp
BOOL APIENTRY DllMain(HMODULE hModule, DWORD ul_reason_for_call, LPVOID lpReserved)
{
    if (ul_reason_for_call == DLL_PROCESS_ATTACH)
    {
        MessageBoxA(NULL, "Inject successful", "CodeInjector", MB_OK | MB_ICONINFORMATION);
    }
    return TRUE;
}
```

`DllMain` runs under the **loader lock**. A MessageBox there blocks the remote thread until you click OK, so:

| Your action | Injector output | Actually loaded? |
|---|---|---|
| Clicked OK within 15 s | `[+] SUCCESS.` | yes |
| Clicked OK after 15 s | `timed out` → `[!] FAILED.` | **yes** |
| Never clicked | `timed out` → `[!] FAILED.` | yes, box still on screen |
| Target is a session-0 service | `timed out` → `[!] FAILED.` | box is invisible |

So a timeout with this demo usually means "you were too slow", not "injection failed". Other threads of the target may also stall while the box is up — another reason not to do this in real code.

**Second injection into the same process:** `LoadLibrary` only bumps the reference count of an already-loaded module; `DLL_PROCESS_ATTACH` is **not** fired again. You'll see `SUCCESS` with **no new MessageBox**. Restart the target to see the box again.

## 8. Independent verification

Don't trust the log alone:

```bat
:: which processes have Inject.dll loaded?
tasklist /m Inject.dll
```

Or use **Process Explorer** → find the target → `View` → `Lower Pane` → DLLs → look for `Inject.dll`. For the non-blocking variant below, **DebugView** will show its `OutputDebugString` trace.

## 9. Your own DLL

Rules for `DllMain`:

- ❌ No `LoadLibrary` / `GetProcAddress` on other DLLs, no COM init, no waiting on threads, no dialogs, no network or file I/O beyond trivial logging.
- ✅ Save `hModule`, call `DisableThreadLibraryCalls`, create a worker thread, return `TRUE` fast.
- Build **x64** and **statically linked** (`-static` / `/MT`).
- Name the output **`Inject.dll`** and put it next to the exe.

The demo skips `DisableThreadLibraryCalls`, so it also receives `DLL_THREAD_ATTACH` / `DLL_THREAD_DETACH` for every thread of the target. Harmless here, but worth calling in anything real.

Non-blocking version of the demo — the injector reports `SUCCESS` immediately, the box appears asynchronously:

```cpp
// Inject.cpp — non-blocking variant
#include <windows.h>

static DWORD WINAPI Worker(LPVOID)
{
    MessageBoxA(nullptr, "Inject successful", "CodeInjector", MB_OK | MB_ICONINFORMATION);
    return 0;   // DLL stays loaded until the process exits
}

BOOL APIENTRY DllMain(HMODULE hModule, DWORD reason, LPVOID)
{
    if (reason == DLL_PROCESS_ATTACH) {
        DisableThreadLibraryCalls(hModule);
        OutputDebugStringA("[Inject] attached");
        if (HANDLE t = CreateThread(nullptr, 0, Worker, nullptr, 0, nullptr))
            CloseHandle(t);
    }
    return TRUE;
}
```

The worker thread starts only after `DllMain` returns and the loader lock is released — that's exactly why this pattern is safe.

## 10. Troubleshooting

| Message | Cause | Fix |
|---|---|---|
| `Inject.dll not found` | DLL not next to the exe / wrong name | copy `Inject.dll` into the injector's folder |
| `CreateToolhelp32Snapshot failed` | enumeration blocked by security software | retry, run as Administrator, test on a plain desktop session |
| `OpenProcess failed: err=5` | no rights, elevated target, protected process | run as Administrator; PPL processes can't be injected at all |
| `No process named "..." found` | typo, missing `.exe`, not running | check Task Manager, or use a PID |
| `Invalid input` / `Index N out of range` | bad menu index | enter a number from the shown range |
| `PID value out of range` | number too large for `stoul` | enter a valid PID |
| `VirtualAllocEx failed` / `WriteProcessMemory failed` | target exiting, memory limits, restrictions | confirm the PID is alive and is a normal user-mode process |
| `CreateRemoteThread failed: err=5` | AV/EDR/anti-cheat, process mitigations, protected process | test on your own app; add an AV exclusion **only for your build folder** |
| `LoadLibraryA returned NULL` | bitness mismatch, missing dependency, path unreadable by target, non-ASCII path, `DllMain` returned `FALSE` | rebuild x64 + static, use an ASCII path, check ACLs, inspect with Dependencies / Process Monitor |
| `Remote thread timed out after 15 s` | blocking `DllMain` (the demo MessageBox!) | click OK faster, or use the non-blocking variant |
| `SUCCESS` but no MessageBox | DLL already loaded in that process | restart the target |
| AV flags the exe as `HackTool`/`Riskware` | expected for an injector | scope an exclusion to your test folder; don't disable protection globally |

## 11. FAQ

**Do I always need Administrator?**
Only for elevated targets (and some system processes). A normal user-mode app of yours works without elevation.

**Can I inject into a 32-bit process?**
No. This project is x64-only and doesn't check bitness up front — you'll typically get `LoadLibraryA returned NULL`.

**Can I use a different DLL name or path?**
Not currently. It's hardcoded to `Inject.dll` next to the exe. A `dll=` config key is on the roadmap.

**Can I unload the DLL afterwards?**
The injector has no unload feature. The module stays until the process exits (or the DLL unloads itself via `FreeLibraryAndExitThread`).

**Why does the program wait for Enter at the end?**
So the console window doesn't vanish when launched by double-click. A `--no-pause` flag for scripting is planned.

**Is a timeout a failure?**
Not necessarily. The injector stops verifying after 15 s; loading may finish later. Check with `tasklist /m Inject.dll`.

**My target's name is all digits — can I select it by name?**
No. An all-digits value is always treated as a PID. Use the real PID instead.