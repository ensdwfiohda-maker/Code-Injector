# CodeInjector

**Классический x64 DLL-инжектор на чистом WinAPI.** Один `.cpp`-файл, ноль внешних зависимостей, консольный интерфейс, конфиг + интерактивный режим.

![Platform](https://img.shields.io/badge/platform-Windows%20x64-blue)
![C++](https://img.shields.io/badge/C%2B%2B-17-blue)
![License](https://img.shields.io/badge/license-MIT-green)

Техника: `OpenProcess` → `VirtualAllocEx` → `WriteProcessMemory` → `CreateRemoteThread(LoadLibraryA)` → `GetExitCodeThread`.

---

## ⚠️ Дисклеймер

Проект **образовательный**: разобраться в WinAPI, механике загрузчика DLL и в том, как работают отладчики, профайлеры и системы хуков.

Используйте его только в **своих** процессах или там, где у вас есть явное разрешение на вмешательство. Инструмент не предназначен для обхода античитов, EDR и прочих защит и не содержит ничего для этого. Внедрение в чужие приложения может нарушать их EULA и законодательство. Ответственность за использование — на пользователе.

---

## Как это работает

```
CodeInjector.exe                        target.exe (x64)
──────────────────                      ────────────────
OpenProcess(pid)             ─────────► handle с минимальными правами
VirtualAllocEx               ─────────► буфер под строку пути
WriteProcessMemory           ─────────► "C:\...\Inject.dll\0"
CreateRemoteThread           ─────────► LoadLibraryA(remoteBuf)
                                            └─► DllMain(DLL_PROCESS_ATTACH)
WaitForSingleObject (15 s)   ◄───────── код выхода потока = HMODULE
GetExitCodeThread            ─────────► 0 → провал, != 0 → успех
VirtualFreeEx + CloseHandle  ─────────► очистка на всех ветках
```

Адрес `LoadLibraryA` берётся из собственного процесса: база `kernel32.dll` в Windows общая для процессов одной сессии, поэтому он валиден и в цели.

## Возможности

- 🔎 **Поиск процесса по имени** через Toolhelp32 (регистронезависимо) или **прямой ввод PID**.
- 🧩 **Несколько экземпляров** — выводит список с индексами и даёт выбрать нужный.
- ⚙️ **Конфиг** `CodeInjectcfg.cfg` с ключом `target=`; если его нет — интерактивный запрос.
- 🧹 **Дисциплина ресурсов**: каждый хэндл и удалённая аллокация освобождаются на любой ветке выхода, включая ошибки и таймаут.
- 🩺 **Внятная диагностика**: подсказки про права администратора, битность, зависимости DLL, блокировку со стороны AV.
- 📦 **Один файл**, собирается MinGW-w64 или MSVC, тянет только `kernel32`.

## Требования

| | |
|---|---|
| ОС | Windows x64 (10/11 рекомендуется) |
| Компилятор | C++17: MinGW-w64 (тестировалось на GCC 16) или MSVC 2019+ |
| Битность | **инжектор, `Inject.dll` и целевой процесс — все x64** |
| Права | для elevated-цели инжектор тоже нужно запускать от администратора |

## Сборка

**MinGW-w64** (статическая линковка, чтобы не тащить `libstdc++-6.dll`):

```bash
g++ -std=c++17 -O2 -Wall -Wextra -m64 -static -o CodeInjector.exe CodeInjector.cpp
```

**MSVC** (из *x64 Native Tools Command Prompt*):

```bat
cl /nologo /std:c++17 /EHsc /O2 /W4 CodeInjector.cpp /Fe:CodeInjector.exe
```

**CMake:**

```cmake
cmake_minimum_required(VERSION 3.15)
project(CodeInjector LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 17)
add_executable(CodeInjector CodeInjector.cpp)
```

```bash
cmake -B build -A x64 && cmake --build build --config Release
```

## Раскладка файлов

DLL ищется **строго рядом с exe** и называется **строго `Inject.dll`**:

```
C:\Tools\CodeInjector\
├── CodeInjector.exe
├── Inject.dll            ← ваша DLL (x64!)
└── CodeInjectcfg.cfg     ← опционально
```

## Конфигурация

`CodeInjectcfg.cfg` — простой `key=value`:

```ini
# Комментарии: '#' или ';'
; Пробелы вокруг ключа/значения обрезаются, ключ регистронезависим.
; Побеждает ПЕРВОЕ вхождение ключа.

target = notepad.exe

# Можно и конкретный PID:
# target = 12345
```

Сейчас читается только ключ `target`. Если файла нет, ключ пустой или невалидный — программа спросит цель в консоли. Путь к DLL через конфиг **не** задаётся (см. Roadmap).

## Использование

```bat
:: обычный запуск — цель из конфига или интерактивно
CodeInjector.exe

:: цель запущена от администратора? запускайте инжектор тоже elevated
```

Пример сессии:

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

Если процессов с таким именем несколько:

```
[!] Multiple instances found:
  [0]  notepad.exe  PID=12345
  [1]  notepad.exe  PID=23456
[>] Enter index [0..1]:
```

**Коды возврата:** `0` — успех, `1` — любая ошибка (удобно для скриптов, но учтите финальный `Press Enter to exit`).

## Минимальная тестовая DLL

```cpp
// Inject.cpp
#include <windows.h>

BOOL APIENTRY DllMain(HMODULE hModule, DWORD reason, LPVOID)
{
    if (reason == DLL_PROCESS_ATTACH) {
        DisableThreadLibraryCalls(hModule);
        OutputDebugStringA("[Inject] attached");   // смотреть в DebugView
    }
    return TRUE;
}
```

```bash
g++ -std=c++17 -O2 -m64 -shared -static -o Inject.dll Inject.cpp
```

> В `DllMain` действует loader lock: никаких `LoadLibrary`, ожиданий потоков, сетевых вызовов и `MessageBox`. Всю реальную работу выносите в отдельный поток, созданный из `DllMain`.

## Диагностика

| Сообщение | Вероятная причина | Что делать |
|---|---|---|
| `Inject.dll not found` | DLL не рядом с exe | положить `Inject.dll` в папку инжектора |
| `OpenProcess failed: err=5` | нет прав / цель elevated / protected process | запустить от администратора; PPL-процессы не инъектируются в принципе |
| `No process named "..." found` | опечатка, процесс не запущен, имя без `.exe` | проверить имя в диспетчере задач или указать PID |
| `VirtualAllocEx / WriteProcessMemory failed` | цель умирает, нехватка памяти, ограничения процесса | проверить, что PID живой и это обычный user-mode процесс |
| `CreateRemoteThread failed: err=5` | AV/EDR/античит, политики mitigation, protected process | тестировать на своём процессе; для своей папки сборки завести исключение в AV |
| `LoadLibraryA returned NULL` | не та битность, путь недоступен цели, нет зависимостей, `DllMain` вернул `FALSE` | проверить x64, ASCII-путь, зависимости (Dependencies / Process Monitor), лог `DllMain` |
| `Remote thread timed out after 15 s` | `DllMain` заблокирован | убрать блокирующие операции; DLL при этом может догрузиться позже |
| `PID value out of range` | введено слишком большое число | указать корректный PID |

> Антивирусы часто помечают такие инструменты как `HackTool`/`Riskware` — для инжектора это ожидаемо. Не отключайте защиту целиком: лучше добавьте исключение только для своей тестовой папки.

## Известные ограничения

- **Только x64.** x86 и WOW64-цели не поддерживаются; битность цели перед инжектом не проверяется.
- **ANSI-путь.** Путь передаётся в `LoadLibraryA`, имена процессов конвертируются через `CP_ACP` — держите путь к DLL в ASCII, без кириллицы и пробелов-сюрпризов.
- **HMODULE в логе усечён до 32 бит** (`GetExitCodeThread` отдаёт `DWORD`). Проверка на `NULL` корректна, но напечатанный адрес — не полный.
- **Нет выгрузки.** DLL остаётся в процессе до его завершения.
- **Имя DLL захардкожено** (`Inject.dll` рядом с exe), таймаут фиксирован (15 с).
- **Таймаут ≠ отмена.** Через 15 секунд инжектор сдаётся, но загрузка в цели может завершиться позже.
- **Не работает** против protected/PPL-процессов, процессов с политикой «только Microsoft-signed DLL» и целей под античитом/EDR.
- `SeDebugPrivilege` не включается — часть системных процессов недоступна даже администратору.

## Roadmap

- [ ] Ключи `dll=` и `timeout=` в конфиге
- [ ] CLI-аргументы (`--pid`, `--name`, `--dll`, `--no-pause`) и quiet-режим
- [ ] Включение `SeDebugPrivilege`
- [ ] Проверка битности цели через `IsWow64Process2` с внятным отказом
- [ ] Полный Unicode-путь: `LoadLibraryW` + W-API по всему коду
- [ ] Лог в файл и опциональная выгрузка DLL (`FreeLibrary`)
- [ ] RAII-обёртки для хэндлов и удалённой памяти вместо ручных cleanup-ветвей

## Лицензия

MIT — см. [LICENSE](LICENSE).
