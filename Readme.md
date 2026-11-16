# CodeInjector

**English** | [Русский](README.ru.md)

Classic **x64 DLL injector** — pure WinAPI, one translation unit, zero external dependencies.

![Platform](https://img.shields.io/badge/platform-Windows%20x64-blue)
![C++](https://img.shields.io/badge/C%2B%2B-17-blue)
![License](https://img.shields.io/badge/license-MIT-green)

Technique: `OpenProcess` → `VirtualAllocEx` → `WriteProcessMemory` → `CreateRemoteThread(LoadLibraryA)` → `GetExitCodeThread`.

📖 **Full usage guide:** [USAGE.md](USAGE.md) · [русская версия](USAGE.ru.md)

---

## ⚠️ Disclaimer

Educational project: understanding WinAPI, the DLL loader, and how debuggers, profilers and hooking frameworks work.

Use it only on **your own** processes, or where you have explicit permission. It is not designed to bypass anti-cheat, EDR or any other protection, and contains nothing for that purpose. Injecting into third-party software may violate its EULA and local law. You are responsible for how you use it.

## How it works

```
CodeInjector.exe                        target.exe (x64)
──────────────────                      ────────────────
OpenProcess(pid)             ─────────► handle, minimum required rights
VirtualAllocEx               ─────────► buffer for the path string
WriteProcessMemory           ─────────► "C:\...\Inject.dll\0"
CreateRemoteThread           ─────────► LoadLibraryA(remoteBuf)
                                            └─► DllMain(DLL_PROCESS_ATTACH)
WaitForSingleObject (15 s)   ◄───────── thread exit code = HMODULE
GetExitCodeThread            ─────────► 0 → failed, != 0 → loaded
VirtualFreeEx + CloseHandle  ─────────► cleanup on every exit path
```

The address of `LoadLibraryA` is taken from the injector's own process: `kernel32.dll` shares its base across processes of the same session, so the address is valid in the target too.

## Features

- 🔎 Find a process **by name** (case-insensitive, Toolhelp32) or target a **PID** directly.
- 🧩 **Multiple instances** — lists them with indices and lets you pick one.
- ⚙️ **Config file** `CodeInjectcfg.cfg` with `target=`; interactive prompt as fallback.
- 🧹 **Resource discipline** — every handle and remote allocation is freed on all paths, including errors and timeout.
- 🩺 **Readable diagnostics** — hints about admin rights, bitness, missing dependencies, AV blocking.
- 📦 **Single file**, builds with MinGW-w64 or MSVC, links only `kernel32`.
- 🧪 **Demo DLL included** (`Inject.cpp`) that shows a MessageBox on attach.

## Requirements

| | |
|---|---|
| OS | Windows x64 (10/11 recommended) |
| Compiler | C++17 — MinGW-w64 (tested on GCC 16) or MSVC 2019+ |
| Bitness | **injector, `Inject.dll` and target must all be x64** |
| Privileges | elevated target → run the injector as Administrator too |

## Quick start

```bash
# 1) the injector
g++ -std=c++17 -O2 -m64 -static -o CodeInjector.exe CodeInjector.cpp

# 2) the demo DLL (shows "Inject successful" on attach)
g++ -std=c++17 -O2 -m64 -shared -static -o Inject.dll Inject.cpp -luser32

# 3) start your target (e.g. notepad.exe), then run
CodeInjector.exe
```

MSVC and CMake commands, config reference and troubleshooting → **[usage guide](USAGE.md)**.

## Runtime layout

The DLL is looked up **next to the exe** and must be named **exactly `Inject.dll`**:

```
C:\Tools\CodeInjector\
├── CodeInjector.exe
├── Inject.dll            ← your DLL (x64!)
└── CodeInjectcfg.cfg     ← optional
```

## Configuration

```ini
# Comments: '#' or ';'   ·   key is case-insensitive   ·   first match wins
target = notepad.exe
# or a specific PID:
# target = 12345
```

Only `target` is read today. No config / empty value → the program asks for a target interactively.

## About `Inject.cpp` (demo only)

The example DLL calls `MessageBoxA` **inside `DllMain`**. That is intentionally simple, but it blocks the loader lock: the remote thread cannot return until you click **OK**.

- Click OK **within 15 seconds** → injector prints `SUCCESS`.
- Click later (or never) → injector prints `Remote thread timed out` / `FAILED`, **even though the DLL did load**.

For real work, never block in `DllMain` — spawn a worker thread instead. A non-blocking variant of the demo is in the [guide](USAGE.md#9-your-own-dll).

## Known limitations

- **x64 only**; x86 / WOW64 targets unsupported, target bitness is not pre-checked.
- **ANSI path** (`LoadLibraryA`, names converted via `CP_ACP`) — keep the DLL path ASCII-only.
- **HMODULE in the log is truncated to 32 bits** (`GetExitCodeThread` returns `DWORD`); the NULL check is still correct.
- **No unloading** — the DLL stays until the target exits.
- DLL name and the 15 s timeout are **hardcoded**.
- **Timeout ≠ cancel** — loading may still complete after the injector gives up.
- Does not work against protected/PPL processes, "Microsoft-signed DLL only" policies, or anti-cheat/EDR-protected targets.
- `SeDebugPrivilege` is not enabled.

## Roadmap

- [ ] `dll=` and `timeout=` config keys
- [ ] CLI flags (`--pid`, `--name`, `--dll`, `--no-pause`) and quiet mode
- [ ] Enable `SeDebugPrivilege`
- [ ] Target bitness check via `IsWow64Process2`
- [ ] Full Unicode paths (`LoadLibraryW` + W APIs throughout)
- [ ] File logging, optional unload (`FreeLibrary`)
- [ ] RAII wrappers for handles and remote memory

## Repository layout

```
.
├── CodeInjector.cpp      # the injector
├── Inject.cpp            # demo DLL (MessageBox on attach)
├── CodeInjectcfg.cfg     # optional config
├── README.md
├── README.ru.md
├── USAGE.md
├── USAGE.ru.md
└── LICENSE
```

## License

MIT — see [LICENSE](LICENSE).
