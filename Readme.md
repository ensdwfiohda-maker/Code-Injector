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
- 🧹 **Resource discipline** — every handle and remote allocation is freed on all
