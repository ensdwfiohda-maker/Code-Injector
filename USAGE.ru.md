# Гайд по использованию — CodeInjector

[English](USAGE.md) | **Русский** · ← назад к [README](README.ru.md)

Пошагово: сборка, настройка, инжект, проверка, разбор ошибок.

> Используйте только в своих процессах или с явного разрешения. См. дисклеймер в [README](README.ru.md).

## Содержание

1. [Что нужно](#1-что-нужно)
2. [Сборка](#2-сборка)
3. [Раскладка файлов](#3-раскладка-файлов)
4. [Конфиг](#4-конфиг)
5. [Запуск](#5-запуск)
6. [Как выглядит успех](#6-как-выглядит-успех)
7. [Демо-DLL и правило 15 секунд](#7-демо-dll-и-правило-15-секунд)
8. [Независимая проверка](#8-независимая-проверка)
9. [Своя DLL](#9-своя-dll)
10. [Разбор ошибок](#10-разбор-ошибок)
11. [FAQ](#11-faq)

---

## 1. Что нужно

- Windows x64.
- Компилятор C++17: **MinGW-w64** (`g++`) или **MSVC** (x64).
- **Запущенный x64-процесс**, который вам принадлежит — `notepad.exe` идеален для опытов.
- Права администратора — **только** если сама цель запущена elevated.
- Вся цепочка x64: инжектор, DLL, цель.

## 2. Сборка

### MinGW-w64

```bash
# инжектор (static → не нужны libstdc++/libgcc DLL)
g++ -std=c++17 -O2 -Wall -Wextra -m64 -static -o CodeInjector.exe CodeInjector.cpp

# демо-DLL (static → внутри цели не нужен рантайм MinGW)
g++ -std=c++17 -O2 -m64 -shared -static -o Inject.dll Inject.cpp -luser32
```

`-static` для DLL важен: зависимости ищутся по путям **самого процесса-цели**, а он обычно не знает, где ваш MinGW `bin`. Динамически слинкованная демо-DLL часто падает с `LoadLibraryA returned NULL`.

### MSVC

Из **x64 Native Tools Command Prompt for VS**:

```bat
cl /nologo /std:c++17 /EHsc /O2 /W4 /MT CodeInjector.cpp /Fe:CodeInjector.exe
cl /nologo /std:c++17 /EHsc /O2 /MT /LD Inject.cpp /Fe:Inject.dll /link user32.lib
```

`/MT` — статический CRT, чтобы цели не требовался VC++ Redistributable.

### CMake (опционально)

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

## 3. Раскладка файлов

Все три файла — в одной папке, путь **короткий и только ASCII**:

```
C:\Tools\CodeInjector\
├── CodeInjector.exe
├── Inject.dll            ← имя захардкожено, именно такое
└── CodeInjectcfg.cfg     ← опционально
```

Две причины не выделываться с путём:

- он передаётся в `LoadLibraryA` (**ANSI**), поэтому не-ASCII символы вроде `C:\Users\Вася\...` могут сломать загрузку;
- папка exe читается в буфер `MAX_PATH`, так что пути длиннее 260 символов обрезаются.

## 4. Конфиг

`CodeInjectcfg.cfg`, простой `key=value`:

```ini
# комментарий через '#'
; комментарий через ';'

target = notepad.exe
```

| Правило | Поведение |
|---|---|
| Комментарии | строка, где первый непробельный символ `#` или `;`, игнорируется |
| Пустые строки | игнорируются |
| Пробелы | обрезаются вокруг ключа и значения |
| Регистр ключа | не важен (`Target`, `TARGET` работают) |
| Дубликаты | побеждает **первое** вхождение |
| Поддерживаемые ключи | только `target` |
| Значение | имя exe (`notepad.exe`) **или** десятичный PID (`12345`) |
| Строки без `=` | игнорируются |

Нет файла / нет ключа / пусто → интерактивный запрос. Путь к DLL через конфиг не задаётся.

## 5. Запуск

Сначала запустите цель, потом `CodeInjector.exe` (двойным кликом или из терминала).

**Сценарий A — цель из конфига**

```ini
target = notepad.exe
```

**Сценарий B — конкретный PID из конфига**

```ini
target = 4242
```

Удобно, когда экземпляров несколько и всегда нужен один и тот же. PID смотрите в Диспетчере задач → Подробности.

**Сценарий C — без конфига, интерактивно**

```
[>] Enter process name (e.g. notepad.exe) or numeric PID: notepad.exe
```

**Сценарий D — несколько экземпляров**

```
[!] Multiple instances found:
  [0]  notepad.exe  PID=12345
  [1]  notepad.exe  PID=23456
[>] Enter index [0..1]: 1
[+] Selected: notepad.exe  PID=23456
```

**Сценарий E — цель elevated**

ПКМ по `CodeInjector.exe` → *Запуск от имени администратора*. Иначе получите `OpenProcess failed: err=5`.

**Коды возврата:** `0` — успех, `1` — любая ошибка. Учтите финальный `Press Enter to exit...`: программа всегда ждёт нажатия перед выходом.

## 6. Как выглядит успех

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

Напечатанный `HMODULE` усечён до 32 бит (`GetExitCodeThread` отдаёт `DWORD`) — это особенность лога, а не признак ошибки.

## 7. Демо-DLL и правило 15 секунд

`Inject.cpp` — **минимальный пример**, а не шаблон для боевого кода:

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

`DllMain` выполняется под **loader lock**. MessageBox блокирует удалённый поток, пока вы не нажмёте OK:

| Ваши действия | Вывод инжектора | DLL загружена? |
|---|---|---|
| OK в течение 15 с | `[+] SUCCESS.` | да |
| OK позже 15 с | `timed out` → `[!] FAILED.` | **да** |
| OK не нажали | `timed out` → `[!] FAILED.` | да, окно висит |
| Цель — служба в session 0 | `timed out` → `[!] FAILED.` | окно не видно |

То есть таймаут с этой демо-DLL обычно значит «вы не успели», а не «инжект не удался». Пока окно висит, другие потоки цели тоже могут подтормаживать — ещё одна причина не делать так в реальном коде.

**Повторный инжект в тот же процесс:** `LoadLibrary` лишь увеличивает счётчик ссылок уже загруженного модуля, `DLL_PROCESS_ATTACH` **повторно не вызывается**. Увидите `SUCCESS`, но **нового MessageBox не будет**. Перезапустите цель, чтобы увидеть окно снова.

## 8. Независимая проверка

Не полагайтесь только на лог:

```bat
:: в каких процессах загружена Inject.dll?
tasklist /m Inject.dll
```

Или **Process Explorer**: найдите цель → `View` → `Lower Pane` → DLLs → ищите `Inject.dll`. Для неблокирующего варианта ниже подойдёт **DebugView** — там будет виден его `OutputDebugString`.

## 9. Своя DLL

Правила для `DllMain`:

- ❌ Нельзя: `LoadLibrary` / `GetProcAddress` на другие DLL, инициализация COM, ожидание потоков, диалоги, сеть, файловый I/O (кроме простейшего лога).
- ✅ Можно: сохранить `hModule`, вызвать `DisableThreadLibraryCalls`, создать рабочий поток, быстро вернуть `TRUE`.
- Собирайте **x64** и **статически** (`-static` / `/MT`).
- Назовите результат **`Inject.dll`** и положите рядом с exe.

Демо не вызывает `DisableThreadLibraryCalls`, поэтому получает ещё и `DLL_THREAD_ATTACH` / `DLL_THREAD_DETACH` на каждый поток цели. Здесь это безвредно, но в реальном коде вызов лучше добавить.

Неблокирующий вариант демо — инжектор сразу рапортует `SUCCESS`, окно появляется асинхронно:

```cpp
// Inject.cpp — неблокирующий вариант
#include <windows.h>

static DWORD WINAPI Worker(LPVOID)
{
    MessageBoxA(nullptr, "Inject successful", "CodeInjector", MB_OK | MB_ICONINFORMATION);
    return 0;   // DLL остаётся загруженной до выхода процесса
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

Рабочий поток стартует только после возврата из `DllMain` и освобождения loader lock — именно поэтому такой паттерн безопасен.

## 10. Разбор ошибок

| Сообщение | Причина | Что делать |
|---|---|---|
| `Inject.dll not found` | DLL не рядом с exe / не то имя | положить `Inject.dll` в папку инжектора |
| `CreateToolhelp32Snapshot failed` | перечисление процессов блокирует защитное ПО | повторить, запустить от администратора, тестировать в обычной десктоп-сессии |
| `OpenProcess failed: err=5` | нет прав, цель elevated, protected process | запустить от администратора; PPL-процессы не инъектируются вообще |
| `No process named "..." found` | опечатка, нет `.exe`, не запущен | проверить в Диспетчере задач или указать PID |
| `Invalid input` / `Index N out of range` | неверный индекс в меню | ввести число из показанного диапазона |
| `PID value out of range` | число не влезает в `stoul` | указать корректный PID |
| `VirtualAllocEx failed` / `WriteProcessMemory failed` | цель завершается, лимиты памяти, ограничения | убедиться, что PID жив и это обычный user-mode процесс |
| `CreateRemoteThread failed: err=5` | AV/EDR/античит, mitigation-политики, protected process | тестировать на своём приложении; исключение в AV — **только для своей папки сборки** |
| `LoadLibraryA returned NULL` | не та битность, нет зависимости, путь недоступен цели, не-ASCII путь, `DllMain` вернул `FALSE` | пересобрать x64 + static, ASCII-путь, проверить ACL, посмотреть Dependencies / Process Monitor |
| `Remote thread timed out after 15 s` | блокирующий `DllMain` (тот самый MessageBox!) | нажать OK быстрее или взять неблокирующий вариант |
| `SUCCESS`, но MessageBox не появился | DLL уже загружена в этот процесс | перезапустить цель |
| AV помечает exe как `HackTool`/`Riskware` | ожидаемо для инжектора | исключение только для тестовой папки, не отключать защиту целиком |

## 11. FAQ

**Всегда нужен администратор?**
Только для elevated-целей (и части системных процессов). Своё обычное приложение инъектируется без повышения прав.

**Можно в 32-битный процесс?**
Нет. Проект только x64 и битность заранее не проверяет — обычно получите `LoadLibraryA returned NULL`.

**Можно другое имя или путь DLL?**
Пока нет: жёстко `Inject.dll` рядом с exe. Ключ `dll=` есть в планах.

**Можно выгрузить DLL обратно?**
Инжектор этого не умеет. Модуль остаётся до выхода процесса (либо DLL выгружает себя сама через `FreeLibraryAndExitThread`).

**Почему в конце ждёт Enter?**
Чтобы окно не исчезало при запуске двойным кликом. Флаг `--no-pause` для скриптов — в планах.

**Таймаут — это провал?**
Не обязательно. Инжектор прекращает проверку через 15 с, а загрузка может завершиться позже. Проверяйте через `tasklist /m Inject.dll`.

**Имя моей цели состоит из цифр — можно выбрать её по имени?**
Нет. Значение только из цифр всегда трактуется как PID. Указывайте настоящий PID.