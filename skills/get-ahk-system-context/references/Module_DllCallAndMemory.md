# Module_DllCallAndMemory.md
<!-- DOMAIN: DLL Calls and Memory Management -->
<!-- SCOPE: IStream COM interface, async I/O, and GDI/DirectX memory-mapped objects are not covered — see Module_SystemAndCOM.md -->
<!-- TRIGGERS: DllCall, Buffer, NumPut, NumGet, StrPut, StrGet, CallbackCreate, CallbackFree, OnMessage, SendMessage, PostMessage, A_LastError, VarSetCapacity, RtlMoveMemory, RtlZeroMemory, RtlFillMemory, RtlCompareMemory, VirtualAlloc, VirtualFree, HeapAlloc, HeapFree, GlobalAlloc, GlobalFree, FormatMessageW, GetSystemInfo, WideCharToMultiByte, MultiByteToWideChar, CreateMutexW, WaitForSingleObject, "call Windows API", "raw memory", "memory allocation", "memory copy", "struct simulation", "Win32 API", "low-level hook", "HANDLE", "HWND", "pointer arithmetic", "enumerate windows", "virtual memory", "process heap", "clipboard memory", "error code to string" -->
<!-- CONSTRAINTS: Buffer(size, fill) is the only valid memory allocator in AHK v2 — VarSetCapacity is removed and causes NameError at runtime. Every HANDLE, HWND, HMODULE, LPVOID, or pointer-sized DllCall argument and return value must use "Ptr", never "Int" — "Int" silently truncates the upper 32 bits of any address on 64-bit Windows. SIZE_T parameters (RtlMoveMemory, VirtualAlloc, HeapAlloc length fields) require "UPtr", not "UInt" — "UInt" silently clips lengths above 4 GB and produces wrong results on 64-bit. -->
<!-- CROSS-REF: Module_Errors.md, Module_Classes.md, Module_SystemAndCOM.md -->
<!-- VERSION: AHK v2.0+ -->

## V1 → V2 BREAKING CHANGES

| v1 pattern (LLM commonly writes) | v2 correct form | Consequence |
|----------------------------------|-----------------|-------------|
| `VarSetCapacity(v, 64, 0)` | `v := Buffer(64, 0)` | NameError at runtime — VarSetCapacity does not exist in v2 |
| `NumGet(ptr+offset, type)` | `NumGet(ptr, offset, type)` | If ptr is a Buffer object, buf+offset throws TypeError; use NumGet(buf, offset, type) to pass buffer and offset separately |
| `NumPut(val, ptr+offset, type)` | `NumPut(type, val, target, offset)` | Argument order is reversed; in v2 the first argument must be a type string, not a value — passing a numeric value first throws TypeError |
| `&var` (address-of a plain variable) | `buf.Ptr` (Buffer property) | The `&` address-of operator is removed in v2 — only Buffer objects expose a stable `.Ptr` |
| `DllCall("Kernel32\GetModuleHandleW", "Str", name)` — no return type | `DllCall("...", "Str", name, "Ptr")` | Default "Int" return type truncates 64-bit handle addresses on x64, causing silent NULL misreads |
| `"Int", hWnd` for HANDLE or HWND arguments | `"Ptr", hWnd` | "Int" is 32-bit; upper 32 bits of 64-bit handles are silently discarded, causing intermittent failures |
| `(n, w, l) => { stmt1; stmt2 }` passed to CallbackCreate | Define a named function, pass its reference | Multi-statement fat-arrow functions are not supported in AHK v2; callback receives corrupted stack frame |
| `"Int"` for SIZE_T/ULONG_PTR (RtlMoveMemory, VirtualAlloc, HeapAlloc length) | `"UPtr"` | "Int" is signed 32-bit; sizes above 2 GB are misrepresented on x64, and negative or zero-length copies result |
| `DllCall("ntdll\RtlMoveMemory", ...)` return checked for 0 | No return check needed — function is void | RtlMoveMemory/RtlZeroMemory/RtlFillMemory are void; checking return value reads garbage or an unrelated prior value |
| `VirtualAlloc(...)` return type default "Int" | `VirtualAlloc(...)` return type `"Ptr"` | VirtualAlloc returns LPVOID — a pointer; "Int" truncates the base address on x64, making the allocation appear to fail |

## API QUICK-REFERENCE

### Buffer Object
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `Buffer()` | `Buffer(ByteCount)` | Allocates ByteCount bytes; contents **uninitialized** — write before read |
| `Buffer()` | `Buffer(ByteCount, FillByte)` | Allocates and fills every byte with FillByte; use `0` for struct initialization |
| `.Ptr` | Read-only integer | Raw memory address; A_PtrSize bytes wide (4 on x86, 8 on x64) |
| `.Size` | Read/write integer | Byte count; assign a new value to reallocate in-place (data preserved, address may change) |
| Auto-deref | `"Ptr", buf` ≡ `"Ptr", buf.Ptr` | AHK auto-dereferences a Buffer to `.Ptr` in any `"Ptr"`-typed DllCall argument |

### Numeric Memory I/O
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `NumPut()` | `NumPut(Type, Value, Target [, Offset])` | Writes Value at byte Offset into Target Buffer; **type is first argument** |
| `NumGet()` | `NumGet(Source, Offset, Type)` | Reads typed value at byte Offset from Source; **type is third argument** |

> **Argument order asymmetry:** `NumPut(Type, Value, buf, offset)` vs `NumGet(buf, offset, Type)` — the type string is in different positions. This is the single most common error source in memory I/O code.

### String Memory I/O
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `StrPut()` | `StrPut(String, Encoding)` | Returns required byte count (including null terminator) when Target is omitted |
| `StrPut()` | `StrPut(String, Target, Encoding)` | Encodes String into Target Buffer using Encoding |
| `StrGet()` | `StrGet(Source, Encoding)` | Decodes null-terminated string from Source (Buffer or raw address) |
| `StrGet()` | `StrGet(Source, Length, Encoding)` | Decodes exactly Length characters from Source |

### DllCall
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `DllCall()` | `DllCall(Func, [Type, Arg]..., ReturnType)` | Calls native function; Func may be `"DLL\FuncName"` or a cached Ptr address |
| `A_LastError` | Built-in variable | Holds the Win32 error code set by the last DllCall; read immediately after a failed call |
| `A_PtrSize` | Built-in variable | Pointer width in bytes: 4 on x86, 8 on x64; use for struct field offset calculations |

**DllCall string-type decision rule:**
- `"Str"` — one-shot IN parameter; passes the address of the string's own UTF-16 buffer directly; in AHK v2 (Unicode), no temporary conversion is performed
- `"WStr"` — equivalent to `"Str"` in AHK v2 (Unicode-only); use `Buffer + StrPut + "Ptr"` for truly retained pointers
- `Buffer + StrPut + "Ptr"` — when the string buffer must outlive the call (async, OUT params)
- `"AStr"` — only for 8-bit ANSI APIs (e.g., `GetProcAddress` function name argument)

### Message Functions
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `SendMessage()` | `SendMessage(Msg, wParam, lParam [, Control, WinTitle...])` | Synchronous; waits for target to process; returns reply value |
| `PostMessage()` | `PostMessage(Msg, wParam, lParam [, Control, WinTitle...])` | Asynchronous; returns immediately with no reply |
| `OnMessage()` | `OnMessage(MsgNumber, Callback [, MaxThreads])` | Registers Callback to intercept Msg delivered to the script window; MaxThreads defaults to 1 |

### Callback Functions
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `CallbackCreate()` | `CallbackCreate(FuncRef [, Options, ParamCount])` | Returns a Ptr-sized native machine-code stub; Options: `"Fast"` skips inter-thread overhead |
| `CallbackFree()` | `CallbackFree(Address)` | Frees the stub allocated by CallbackCreate; **must be called** — leaks are permanent |

**CallbackCreate "Fast" mode rule:**
- Use `"Fast"` for synchronous callbacks (EnumWindows, LL hooks) — correct and lower overhead
- Omit `"Fast"` for callbacks invoked on a foreign thread or calling AHK built-ins with side effects

### ntdll Memory Operations
These functions are exported from both `ntdll.dll` and `kernel32.dll`; the `ntdll\` prefix is preferred for raw memory operations. All length parameters are `SIZE_T` → `"UPtr"`.

| Function | Signature (C) | AHK DllCall types | Notes |
|----------|---------------|-------------------|-------|
| `RtlMoveMemory` | `VOID RtlMoveMemory(PVOID dst, const VOID* src, SIZE_T len)` | `"Ptr", dst, "Ptr", src, "UPtr", len` | Safe for overlapping regions; equivalent to C `memmove`. No return value. |
| `RtlCopyMemory` | `VOID RtlCopyMemory(PVOID dst, const VOID* src, SIZE_T len)` | `"Ptr", dst, "Ptr", src, "UPtr", len` | Non-overlapping only; equivalent to C `memcpy`. No return value. |
| `RtlFillMemory` | `VOID RtlFillMemory(PVOID dst, SIZE_T len, UCHAR fill)` | `"Ptr", dst, "UPtr", len, "UChar", fill` | Fills `len` bytes with `fill`. Equivalent to C `memset`. No return value. |
| `RtlZeroMemory` | `VOID RtlZeroMemory(PVOID dst, SIZE_T len)` | `"Ptr", dst, "UPtr", len` | Zero-fills `len` bytes. Equivalent to `RtlFillMemory(dst, len, 0)`. No return value. |
| `RtlCompareMemory` | `SIZE_T RtlCompareMemory(const VOID* s1, const VOID* s2, SIZE_T len)` | `"Ptr", s1, "Ptr", s2, "UPtr", len, "UPtr"` | Returns **count of matching bytes** from the start. Equal iff result = len. |

> **Common mistake:** `RtlCompareMemory` returns a match *count*, not a boolean — testing `if !result` only detects a mismatch on the first byte.

### Virtual Memory (kernel32)
| Function | C signature summary | AHK return type | Notes |
|----------|--------------------|-----------------|----|
| `VirtualAlloc` | `LPVOID VirtualAlloc(LPVOID lpAddr, SIZE_T dwSize, DWORD flAllocType, DWORD flProtect)` | `"Ptr"` | Returns base address or 0 on failure; `MEM_COMMIT|MEM_RESERVE = 0x3000` |
| `VirtualFree` | `BOOL VirtualFree(LPVOID lpAddr, SIZE_T dwSize, DWORD dwFreeType)` | `"Int"` | `MEM_RELEASE = 0x8000` requires `dwSize = 0` |
| `VirtualProtect` | `BOOL VirtualProtect(LPVOID lpAddr, SIZE_T dwSize, DWORD flNewProtect, PDWORD lpOldProtect)` | `"Int"` | Changes page protection; pass a 4-byte Buffer for lpOldProtect |
| `VirtualQuery` | `SIZE_T VirtualQuery(LPCVOID lpAddr, PMEMORY_BASIC_INFORMATION lpBuf, SIZE_T dwLen)` | `"UPtr"` | Returns number of bytes written; MEMORY_BASIC_INFORMATION is 48 bytes on x64 |

### Heap Management (kernel32)
| Function | C signature summary | AHK return type | Notes |
|----------|--------------------|-----------------|----|
| `GetProcessHeap` | `HANDLE GetProcessHeap()` | `"Ptr"` | Returns handle to the default process heap; do not call `HeapFree` on objects you did not allocate |
| `HeapAlloc` | `LPVOID HeapAlloc(HANDLE hHeap, DWORD dwFlags, SIZE_T dwBytes)` | `"Ptr"` | Returns address or 0; `HEAP_ZERO_MEMORY = 8` fills with zero |
| `HeapFree` | `BOOL HeapFree(HANDLE hHeap, DWORD dwFlags, LPVOID lpMem)` | `"Int"` | dwFlags must be 0; do not call on a Buffer — Buffer has its own allocator |
| `HeapSize` | `SIZE_T HeapSize(HANDLE hHeap, DWORD dwFlags, LPCVOID lpMem)` | `"UPtr"` | Returns allocation size; `(SIZE_T)-1` on failure |
| `GlobalAlloc` | `HGLOBAL GlobalAlloc(UINT uFlags, SIZE_T dwBytes)` | `"Ptr"` | Required for clipboard data; `GMEM_MOVEABLE = 2`, `GMEM_ZEROINIT = 0x40` |
| `GlobalFree` | `HGLOBAL GlobalFree(HGLOBAL hMem)` | `"Ptr"` | Returns NULL on success; non-NULL indicates failure |
| `GlobalLock` | `LPVOID GlobalLock(HGLOBAL hMem)` | `"Ptr"` | Locks a moveable block; returns usable pointer; must be paired with `GlobalUnlock` |
| `GlobalUnlock` | `BOOL GlobalUnlock(HGLOBAL hMem)` | `"Int"` | Call after every `GlobalLock` — lock count is reference-counted |
| `GlobalSize` | `SIZE_T GlobalSize(HGLOBAL hMem)` | `"UPtr"` | Returns allocation size in bytes |

### Encoding Conversion (kernel32)
| Function | C signature summary | AHK return type | Notes |
|----------|--------------------|-----------------|----|
| `WideCharToMultiByte` | `int WideCharToMultiByte(UINT cp, DWORD flags, LPCWSTR src, int srcLen, LPSTR dst, int dstLen, LPCSTR defChar, LPBOOL usedDefault)` | `"Int"` | Returns bytes written; cp = `65001` for UTF-8, `0` for system ANSI; pass 0 for dstLen to compute required size |
| `MultiByteToWideChar` | `int MultiByteToWideChar(UINT cp, DWORD flags, LPCSTR src, int srcLen, LPWSTR dst, int dstLen)` | `"Int"` | Returns wide chars written; pass 0 for dstLen to compute required size |

### Error Reporting (kernel32)
| Function | C signature summary | AHK return type | Notes |
|----------|--------------------|-----------------|----|
| `GetLastError` | `DWORD GetLastError()` | `"UInt"` | Same value as `A_LastError`; use `A_LastError` directly in AHK v2 instead |
| `FormatMessageW` | `DWORD FormatMessageW(DWORD flags, LPCVOID src, DWORD msgId, DWORD langId, LPWSTR buf, DWORD size, va_list* args)` | `"UInt"` | `FORMAT_MESSAGE_FROM_SYSTEM|ALLOCATE_BUFFER = 0x1300` returns a self-allocated buffer pointer |

### System Information (kernel32)
| Function | C signature summary | AHK return type | Notes |
|----------|--------------------|-----------------|----|
| `GetSystemInfo` | `VOID GetSystemInfo(LPSYSTEM_INFO lpSystemInfo)` | (void) | SYSTEM_INFO is 48 bytes; wProcessorArchitecture at offset 0 (UShort), dwNumberOfProcessors at offset 20 (UInt), dwPageSize at offset 4 (UInt) |
| `GetComputerNameW` | `BOOL GetComputerNameW(LPWSTR lpBuf, LPDWORD nSize)` | `"Int"` | lpBuf = Buffer(MAX_COMPUTERNAME_LENGTH*2+2); pass nSize as `"UIntP"` |
| `GetUserNameW` | `BOOL GetUserNameW(LPWSTR lpBuf, LPDWORD pcbBuf)` | `"Int"` | Same pattern as GetComputerNameW; requires `advapi32\GetUserNameW` |

### Synchronization (kernel32)
| Function | C signature summary | AHK return type | Notes |
|----------|--------------------|-----------------|----|
| `CreateMutexW` | `HANDLE CreateMutexW(LPSECURITY_ATTRIBUTES lpAttr, BOOL bInitialOwner, LPCWSTR lpName)` | `"Ptr"` | Named mutex for cross-process; unnamed for in-process |
| `OpenMutexW` | `HANDLE OpenMutexW(DWORD dwAccess, BOOL bInherit, LPCWSTR lpName)` | `"Ptr"` | Opens existing named mutex; `MUTEX_ALL_ACCESS = 0x1F0001` |
| `WaitForSingleObject` | `DWORD WaitForSingleObject(HANDLE hObj, DWORD dwMs)` | `"UInt"` | `WAIT_OBJECT_0 = 0`, `WAIT_TIMEOUT = 0x102`, `INFINITE = 0xFFFFFFFF` |
| `ReleaseMutex` | `BOOL ReleaseMutex(HANDLE hMutex)` | `"Int"` | Must be called by the owning thread to release |
| `CloseHandle` | `BOOL CloseHandle(HANDLE hObj)` | `"Int"` | Required for every HANDLE returned by Create*/Open* — omitting leaks kernel objects |

### Win32 Type Mapping
Reference this table when reading Windows API documentation to select the correct DllCall type string for each parameter or return value.

| C / Win32 Typedef | AHK v2 DllCall Type | Width | Notes |
|-------------------|--------------------|----|-------|
| `HWND, HANDLE, HHOOK, HMODULE, HDC, HKEY` | `"Ptr"` | 4/8 | All HANDLE types — never `"Int"` |
| `LPVOID, LPCVOID, void*`, any pointer | `"Ptr"` | 4/8 | |
| `WPARAM, UINT_PTR, SIZE_T, ULONG_PTR` | `"UPtr"` | 4/8 | Unsigned pointer-width |
| `LPARAM, LRESULT, INT_PTR, LONG_PTR` | `"Ptr"` | 4/8 | Signed pointer-width |
| `DWORD, UINT, ULONG, COLORREF` | `"UInt"` | 4 | |
| `LONG, INT, BOOL` | `"Int"` | 4 | BOOL is 32-bit; 0=FALSE, non-zero=TRUE |
| `HRESULT` | `"Int"` or `"HRESULT"` | 4 | Use `"HRESULT"` to auto-throw OSError on failure |
| `WORD, USHORT, ATOM, WCHAR` | `"UShort"` | 2 | |
| `SHORT` | `"Short"` | 2 | |
| `BYTE, UCHAR, BOOLEAN` | `"UChar"` | 1 | |
| `CHAR` | `"Char"` | 1 | |
| `LONGLONG, __int64, LARGE_INTEGER` | `"Int64"` | 8 | |
| `ULONGLONG, DWORD64, QWORD` | `"UInt64"` | 8 | |
| `FLOAT` | `"Float"` | 4 | |
| `DOUBLE` | `"Double"` | 8 | |
| `LPWSTR, LPCWSTR, LPTSTR` (Unicode) | `"Str"` or `"WStr"` | — | `"Str"` = one-shot; `"WStr"` = equivalent to `"Str"` in v2 |
| `LPSTR, LPCSTR` (ANSI) | `"AStr"` | — | Only for 8-bit ANSI APIs |
| `LPRECT, LPPOINT, LPSYSTEMTIME` (struct ptr) | `"Ptr"` | 4/8 | Allocate Buffer of correct size, pass directly |

**Quick-reference (most common):**

| | | | |
|-|-|-|-|
| `HWND` → `"Ptr"` | `DWORD` → `"UInt"` | `BOOL` → `"Int"` | `WORD` → `"UShort"` |
| `HANDLE` → `"Ptr"` | `UINT` → `"UInt"` | `LONG` → `"Int"` | `BYTE` → `"UChar"` |
| `HMODULE` → `"Ptr"` | `WPARAM` → `"UPtr"` | `LPWSTR` → `"Str"` | `LPSTR` → `"AStr"` |
| `HDC` → `"Ptr"` | `LPARAM` → `"Ptr"` | `LPCWSTR` → `"Str"` | `FLOAT` → `"Float"` |
| `SIZE_T` → `"UPtr"` | `LRESULT` → `"Ptr"` | `LONGLONG` → `"Int64"` | `DOUBLE` → `"Double"` |

## AHK V2 CONSTRAINTS

- `Buffer(ByteCount, FillByte)` is the **only** valid memory allocator in AHK v2 — `VarSetCapacity` is removed and causes a `NameError` at runtime; all v1 code using `VarSetCapacity` must be migrated before running.
- Use `"Ptr"` for every Windows HANDLE, HWND, HMODULE, LPVOID, and any pointer-sized DllCall argument or return value — `"Int"` is 32-bit and silently truncates the upper 32 bits of 64-bit addresses, causing intermittent failures that are difficult to reproduce in testing.
- Always specify the DllCall return type explicitly when it is not `"Int"` — the default `"Int"` truncates `Ptr`-sized and `UInt`-sized return values on 64-bit without any warning or error.
- `NumPut` argument order is `NumPut(Type, Value, Target [, Offset])` and `NumGet` argument order is `NumGet(Source, Offset, Type)` — the type string is in a different positional slot for each function; swapping them silently writes or reads garbage values.
- Never cache `.Ptr` in a separate variable and use it after the Buffer may have gone out of scope — the address becomes dangling instantly and `NumGet`/`NumPut` on it causes a crash; pass the Buffer object itself across function boundaries.
- Every `CallbackCreate` must be paired with a `CallbackFree` in a cleanup path (typically `__Delete()`) — failing to free leaks a permanent native memory allocation that persists for the lifetime of the AHK process.
- Low-level hook procedures (WH_KEYBOARD_LL, WH_MOUSE_LL) **must** call `CallNextHookEx` on every invocation — omitting the chain call stalls keyboard or mouse input for all other applications on the system.
- `EnumWindows` and `EnumChildWindows` callbacks must return a non-zero value (typically `1`) to continue enumeration — returning `0` terminates iteration after the first window.
- String encoding must be explicit: Windows Unicode APIs require `"UTF-16"` for `StrPut`/`StrGet`; ANSI APIs require `"AStr"` in DllCall, or `"CP0"` (system ANSI codepage) for `StrPut`/`StrGet` — mixing encodings silently corrupts multi-byte string content.
- Struct field byte offsets must be derived from official Windows API documentation — even a one-byte error in offset produces silently wrong values or a crash; never estimate offsets.
- `SIZE_T` parameters (`RtlMoveMemory`, `RtlFillMemory`, `RtlZeroMemory`, `RtlCompareMemory`, `VirtualAlloc`, `HeapAlloc`, `GlobalAlloc` length fields) **must** use `"UPtr"`, not `"UInt"` — `"UInt"` clips the value to 32 bits on a 64-bit process, silently truncating lengths above 4 GB and misrepresenting pointer-arithmetic results.
- `RtlMoveMemory`, `RtlZeroMemory`, `RtlFillMemory` are void functions — do not specify or check a return type; specifying a return type reads an undefined register value.
- Every `HANDLE` obtained via `CreateMutexW`, `OpenMutexW`, `VirtualAlloc`, `HeapAlloc`, or `GlobalAlloc` must be released via the corresponding close/free function — Windows kernel objects are not reference-counted; omitting the close leaks the handle for the lifetime of the process.
- `HeapFree` must not be called on a Buffer object's `.Ptr` — Buffer has its own internal allocator; mixing allocators causes heap corruption and a crash.

Safe-access priority order for DllCall and memory operations:
  1. `DllCall("DLL\Func", types..., "ReturnType")` with explicit return type — prevents silent truncation on first call
  2. Check return value against documented sentinel (0, NULL, -1) then read `A_LastError` — captures Win32 error immediately before any subsequent call can overwrite it
  3. `try/catch OSError` — when AHK throws on non-zero HRESULT or when the error message carries diagnostic information
  4. `NumGet(buf, offset, Type)` with documented offset — safe only when the Buffer object is still in scope; never use on a cached `.Ptr` from an expired Buffer

Paired prohibition/alternative examples:
- ✗ `val := VarSetCapacity(v, 64, 0)` — NameError at runtime
- ✓ `v := Buffer(64, 0)` — canonical v2 allocator

- ✗ `DllCall("GetModuleHandleW", "Str", name)` — default "Int" return truncates 64-bit handle
- ✓ `DllCall("Kernel32\GetModuleHandleW", "Str", name, "Ptr")` — preserves full 64-bit address

- ✗ `(n, w, l) => { stmts }` passed to CallbackCreate — multi-statement fat-arrow not supported
- ✓ Define a named function and pass its reference to CallbackCreate

- ✗ `DllCall("ntdll\RtlMoveMemory", "Ptr", dst, "Ptr", src, "UInt", size)` — "UInt" clips SIZE_T
- ✓ `DllCall("ntdll\RtlMoveMemory", "Ptr", dst, "Ptr", src, "UPtr", size)` — SIZE_T is pointer-width

- ✗ `addr := DllCall("Kernel32\VirtualAlloc", ..., "Int")` — "Int" truncates the returned LPVOID
- ✓ `addr := DllCall("Kernel32\VirtualAlloc", ..., "Ptr")` — preserves full 64-bit base address

## TIER 1 — Buffer Object Fundamentals
> METHODS COVERED: Buffer() · .Ptr · .Size · A_PtrSize

`Buffer` is the AHK v2 replacement for the removed `VarSetCapacity`. A Buffer object tracks both the allocation size (`.Size`) and the raw memory address (`.Ptr`), and its lifetime is automatically managed by AHK's reference counter. Passing a Buffer directly as a `"Ptr"`-typed DllCall argument is idiomatic — AHK auto-dereferences it to `.Ptr` at the call site, so the explicit `.Ptr` suffix is never required when calling DllCall.
```ahk
; ── Basic allocation ─────────────────────────────────────────────────────────────
; ✓ Buffer(size, fillByte) is the canonical v2 form — VarSetCapacity does not exist in v2
buf1 := Buffer(64, 0)          ; 64 bytes, all zeroed  (safe baseline for structs)
; ✗ VarSetCapacity(x, 64)      ; → NameError at runtime — removed in v2

MsgBox(buf1.Size)              ; → 64   (byte count)
MsgBox(buf1.Ptr)               ; → raw memory address as an integer

; ── Allocation without zeroing ───────────────────────────────────────────────────
; ✓ Omit FillByte only when you will write every byte before reading — contents are undefined
buf2 := Buffer(32)             ; FillByte omitted — contents undefined; write before read

; ── Passing Buffer to DllCall — auto-dereference via "Ptr" type ──────────────────
; ✓ Buffer passed directly — AHK auto-dereferences to .Ptr; explicit .Ptr suffix is redundant
pt := Buffer(8, 0)             ; POINT struct placeholder (8 bytes)
DllCall("User32\GetCursorPos", "Ptr", pt)
; DllCall("User32\GetCursorPos", "Ptr", pt.Ptr)   ; valid but redundant

; ── Querying address and size after allocation ───────────────────────────────────
ShowBufInfo(buf) {
    MsgBox("Ptr=" buf.Ptr "  Size=" buf.Size)
}
ShowBufInfo(buf1)

; ── Buffer lifetime is scope-bound — never cache .Ptr across scopes ──────────────
; ✗ Caching .Ptr and returning it — the Buffer is freed on function return, .Ptr is dangling
LeakDemo() {
    local b := Buffer(16, 0xFF)
    return b.Ptr        ; → dangling address — b freed on function return; crash on NumGet
}
; ✓ Pass the Buffer object itself — reference count keeps allocation alive
SafeDemo(b) {
    MsgBox(b.Ptr)
}

; ── Reallocation: two valid forms ────────────────────────────────────────────────
buf3 := Buffer(16, 0)
; ✓ In-place resize — existing data preserved; address may change; new bytes uninitialized
buf3.Size := 32
; ✓ Replace with new zeroed allocation — old allocation released immediately
buf3 := Buffer(32, 0)

; ── A_PtrSize: pointer-width in bytes (4 on x86, 8 on x64) ─────────────────────
; ✓ Use A_PtrSize for any allocation that must hold exactly one pointer-width slot
ptrBuf := Buffer(A_PtrSize, 0)
```

## TIER 2 — Memory Read, Write, and Block Operations
> METHODS COVERED: NumPut() · NumGet() · StrPut() · StrGet() · RtlMoveMemory · RtlCopyMemory · RtlFillMemory · RtlZeroMemory · RtlCompareMemory

`NumPut` writes typed integer values into a Buffer at a byte offset; `NumGet` reads them back. `StrPut` encodes an AHK string into a Buffer using an explicit encoding, and `StrGet` decodes a Buffer or raw pointer back to an AHK string. Type and encoding specification is mandatory in every call — mismatches silently produce wrong values or corrupt multi-byte string content without throwing any error. The `ntdll` block-operation functions (`RtlMoveMemory`, `RtlFillMemory`, `RtlZeroMemory`, `RtlCompareMemory`) provide efficient byte-level operations on entire Buffer regions and are preferred over manual `NumPut`/`NumGet` loops for large-scale memory manipulation.
```ahk
; ── NumPut / NumGet — integer field I/O ──────────────────────────────────────────
; ✓ NumPut(Type, Value, Target, Offset) — type string is FIRST argument
buf := Buffer(24, 0)

NumPut("Int",    -1000,       buf, 0)    ; write Int32  at byte offset 0
NumPut("UInt",   4294967295,  buf, 4)    ; write UInt32 at byte offset 4
NumPut("Short",  -32768,      buf, 8)    ; write Int16  at byte offset 8
NumPut("UShort", 65535,       buf, 10)   ; write UInt16 at byte offset 10
NumPut("Int64",  9007199254,  buf, 12)   ; write Int64  at byte offset 12

; ✓ NumGet(Source, Offset, Type) — type string is THIRD argument
v1 := NumGet(buf, 0,  "Int")    ; → -1000
v2 := NumGet(buf, 4,  "UInt")   ; → 4294967295
v3 := NumGet(buf, 8,  "Short")  ; → -32768
v4 := NumGet(buf, 10, "UShort") ; → 65535
v5 := NumGet(buf, 12, "Int64")  ; → 9007199254

; ── Always specify type in NumGet to document intent clearly ─────────────────────
; ✓ Explicit type — correct and self-documenting on both x86 and x64
ptrVal := NumGet(buf, 0, "UPtr")
; ✗ NumGet(buf, 0) — Type silently defaults to "UPtr"; specify type explicitly
;     to document intent and avoid silently reading at the wrong width

; ── StrPut / StrGet — encoding-safe string I/O ───────────────────────────────────
; ✓ Call StrPut with no Target first to get the required byte count, then allocate
str     := "Hello, 世界"
byteLen := StrPut(str, "UTF-16")         ; compute required buffer size (includes null)
strBuf  := Buffer(byteLen, 0)
StrPut(str, strBuf, "UTF-16")            ; write UTF-16 encoded string into buffer

result := StrGet(strBuf, "UTF-16")       ; → "Hello, 世界"

; ── UTF-8 round-trip ──────────────────────────────────────────────────────────────
; ✓ Same two-step pattern: compute size, allocate, encode
utf8Buf := Buffer(StrPut(str, "UTF-8"), 0)
StrPut(str, utf8Buf, "UTF-8")
result2 := StrGet(utf8Buf, "UTF-8")      ; → "Hello, 世界"

; ── StrGet from a raw pointer with known character count ─────────────────────────
; ✓ Used when a Win32 API returns an LPWSTR with a documented character count
rawPtr    := strBuf.Ptr
charCount := 11                           ; characters to read (excluding null)
partial   := StrGet(rawPtr, charCount, "UTF-16")

; ── Null-terminate a wide string manually ────────────────────────────────────────
wideBuf := Buffer(40, 0)
StrPut("Test", wideBuf, "UTF-16")
NumPut("UShort", 0, wideBuf, 8)          ; explicit wide null at byte offset 8

; ── Writing a pointer value into a buffer field ───────────────────────────────────
; ✓ Use "Ptr" type for pointer-sized fields to ensure 4/8 byte correctness per architecture
pBuf  := Buffer(A_PtrSize * 2, 0)
inner := Buffer(8, 0xAB)
NumPut("Ptr", inner.Ptr, pBuf, 0)                       ; store inner.Ptr at slot 0
readBack := NumGet(pBuf, 0, "Ptr")                       ; retrieve the stored address
MsgBox(readBack = inner.Ptr ? "Ptr match" : "Mismatch") ; → "Ptr match"

; ════════════════════════════════════════════════════════════════════════════════════
; ── ntdll BLOCK OPERATIONS — efficient byte-level memory manipulation ────────────
; ════════════════════════════════════════════════════════════════════════════════════

; ── RtlMoveMemory — copy bytes between two Buffer objects (overlap-safe) ─────────
; VOID RtlMoveMemory(PVOID dst, const VOID* src, SIZE_T len)
; ✓ "UPtr" for SIZE_T — "UInt" clips to 32 bits on x64 and corrupts copies above 4 GB
srcBuf := Buffer(16, 0)
dstBuf := Buffer(16, 0)
NumPut("Int64", 0x0102030405060708, srcBuf, 0)
NumPut("Int64", 0x090A0B0C0D0E0F10, srcBuf, 8)

DllCall("ntdll\RtlMoveMemory",
    "Ptr",  dstBuf,       ; Destination (PVOID)
    "Ptr",  srcBuf,       ; Source      (const VOID*)
    "UPtr", srcBuf.Size)  ; Length      (SIZE_T) — UPtr, not UInt

; ✓ RtlMoveMemory returns void — no return type argument needed
MsgBox(NumGet(dstBuf, 0, "Int64") = 0x0102030405060708 ? "Copy OK" : "Mismatch")

; ── Overlap-safe in-place shift (memmove semantics) ──────────────────────────────
; ✓ RtlMoveMemory handles src/dst overlap correctly — use when regions may overlap
overlapBuf := Buffer(32, 0)
NumPut("Int", 1, overlapBuf, 0)
NumPut("Int", 2, overlapBuf, 4)
NumPut("Int", 3, overlapBuf, 8)

; Shift data from offset 0 to offset 4 (overlapping 8-byte copy)
DllCall("ntdll\RtlMoveMemory",
    "Ptr",  overlapBuf.Ptr + 4,   ; dst = 4 bytes into the same buffer
    "Ptr",  overlapBuf,            ; src = start of buffer
    "UPtr", 12)                    ; copy 12 bytes — overlaps safely
; → overlapBuf[4..15] now contains the original [0..11]

; ── RtlCopyMemory — fast non-overlapping copy (memcpy semantics) ─────────────────
; ✓ Use only when src and dst regions are guaranteed non-overlapping
buf_a := Buffer(64, 0xAA)
buf_b := Buffer(64, 0)
DllCall("ntdll\RtlCopyMemory",
    "Ptr",  buf_b,
    "Ptr",  buf_a,
    "UPtr", 64)
MsgBox(NumGet(buf_b, 0, "UChar") = 0xAA ? "RtlCopyMemory OK" : "Fail")

; ── RtlFillMemory — fill a region with a repeating byte value ────────────────────
; VOID RtlFillMemory(PVOID dst, SIZE_T len, UCHAR fill)
; ✓ Preferred over a NumPut loop for initialization of large buffers
fillBuf := Buffer(256)
DllCall("ntdll\RtlFillMemory",
    "Ptr",   fillBuf,       ; Destination
    "UPtr",  fillBuf.Size,  ; Length (SIZE_T)
    "UChar", 0xFF)          ; Fill byte
MsgBox(NumGet(fillBuf, 0,   "UChar") = 0xFF   ; → true
    && NumGet(fillBuf, 255, "UChar") = 0xFF ? "Fill OK" : "Fail")

; ✗ NumPut loop for large-buffer init — O(n) individual calls vs one C-speed fill
; Loop fillBuf.Size
;     NumPut("UChar", 0xFF, fillBuf, A_Index-1)  ; → very slow at scale

; ── RtlZeroMemory — zero-fill a region (reset a reused Buffer) ───────────────────
; VOID RtlZeroMemory(PVOID dst, SIZE_T len)
; ✓ Re-initializes a reused Buffer to a known-zero state without re-allocating
reuse := Buffer(128, 0xFF)    ; pre-filled for demonstration
MsgBox(NumGet(reuse, 0, "UChar"))   ; → 255

DllCall("ntdll\RtlZeroMemory",
    "Ptr",  reuse,
    "UPtr", reuse.Size)
MsgBox(NumGet(reuse, 0, "UChar"))   ; → 0  (re-zeroed in-place, no reallocation)

; ── RtlCompareMemory — compare two regions, get count of matching bytes ───────────
; SIZE_T RtlCompareMemory(const VOID* s1, const VOID* s2, SIZE_T len) → "UPtr"
; ✓ Returns the number of identical bytes from the start; equal iff result = len
cmp_a := Buffer(8, 0xAB)
cmp_b := Buffer(8, 0xAB)
NumPut("UChar", 0xFF, cmp_b, 4)   ; difference at byte 4

matchCount := DllCall("ntdll\RtlCompareMemory",
    "Ptr",  cmp_a,
    "Ptr",  cmp_b,
    "UPtr", cmp_a.Size,
    "UPtr")                         ; ✓ return "UPtr" — SIZE_T is pointer-width

MsgBox("First " matchCount " bytes match (expected 4)")

; ✓ Full equality test: match count must equal the requested length
if matchCount = cmp_a.Size
    MsgBox("Buffers are identical")
else
    MsgBox("First difference at byte " matchCount)

; ✗ Wrong: testing !matchCount as "not equal" — returns 0 only if the first byte differs
; if !matchCount          ; → only fires when byte[0] differs; misses all other cases
;     MsgBox("Not equal") ; → silent false-positive when byte[0] matches but later bytes differ
```

## TIER 3 — Simple DllCall and Core Win32 Utilities
> METHODS COVERED: DllCall() · A_LastError · A_PtrSize · GetModuleHandleW · LoadLibraryW · GetProcAddress · FreeLibrary · GetCursorPos · GetForegroundWindow · SetWindowTextW · GetTickCount · MessageBoxW · VirtualAlloc · VirtualFree · GetSystemInfo · GetComputerNameW · FormatMessageW · WideCharToMultiByte · MultiByteToWideChar

`DllCall("DLL\FunctionName", Type1, Arg1, ..., ReturnType)` is the gateway to any Windows API. The DLL name may be omitted for functions exported from the four auto-searched system libraries (`user32`, `kernel32`, `comctl32`, `gdi32`). Every argument requires an explicit type string, and the return type must be specified whenever it differs from the default `"Int"` to prevent silent truncation of pointer-sized or unsigned return values.
```ahk
; ── GetTickCount — no arguments, UInt return ─────────────────────────────────────
; ✓ "UInt" return type matches DWORD — correct for a 32-bit unsigned millisecond count
ticks := DllCall("Kernel32\GetTickCount", "UInt")
MsgBox("Milliseconds since boot: " ticks)

; ── MessageBoxW — Ptr + Str arguments, Int return ────────────────────────────────
; ✓ "Ptr" for hWnd (a HANDLE), "Str" auto-converts AHK string to UTF-16 temp buffer
ret := DllCall("User32\MessageBoxW",
    "Ptr",  0,                          ; hWnd = NULL
    "Str",  "Hello from DllCall",       ; lpText (UTF-16)
    "Str",  "DllCall Demo",             ; lpCaption
    "UInt", 0x41,                       ; uType = MB_OK | MB_ICONINFORMATION
    "Int")                              ; return type: IDOK=1, IDCANCEL=2, etc.
MsgBox("Button pressed: " ret)

; ── GetCursorPos — Buffer as struct pointer ───────────────────────────────────────
; ✓ Buffer(8, 0) for POINT { LONG x; LONG y; } — 2 × 4-byte Int fields = 8 bytes
pt := Buffer(8, 0)
ok := DllCall("User32\GetCursorPos", "Ptr", pt, "Int")
if ok {
    x := NumGet(pt, 0, "Int")   ; POINT.x
    y := NumGet(pt, 4, "Int")   ; POINT.y
    MsgBox("Cursor at: " x ", " y)
}

; ── Error handling via A_LastError ───────────────────────────────────────────────
; ✓ Read A_LastError immediately — any subsequent API call can overwrite it
result := DllCall("User32\SetWindowTextW",
    "Ptr", WinExist("A"),
    "Str", "Renamed by DllCall",
    "Int")
if !result
    MsgBox("SetWindowTextW failed. Win32 error: " A_LastError)

; ── GetModuleHandleW — must return "Ptr" to capture full 64-bit address ──────────
; ✓ "Ptr" return type preserves the full 8-byte HMODULE on x64
hMod := DllCall("Kernel32\GetModuleHandleW", "Str", "kernel32.dll", "Ptr")
; ✗ DllCall("Kernel32\GetModuleHandleW", "Str", "kernel32.dll") — defaults to "Int"
;     → truncates upper 32 bits on x64; any handle above 4 GB boundary becomes 0
if hMod = 0
    MsgBox("Module not found. Error: " A_LastError)

; ── Dynamic DLL loading and calling ─────────────────────────────────────────────
; ✓ "Ptr" return for LoadLibraryW — HMODULE is pointer-sized
hLib := DllCall("Kernel32\LoadLibraryW", "Str", "winmm.dll", "Ptr")
if hLib {
    DllCall("winmm\PlaySound",
        "Str",  "SystemAsterisk",
        "Ptr",  0,
        "UInt", 0x00010000,   ; SND_ALIAS
        "Int")
    DllCall("Kernel32\FreeLibrary", "Ptr", hLib, "Int")
}

; ── try/catch wrapper for DllCall — recommended for production code ───────────────
; ✓ Wrap in try/catch OSError for APIs that may fail with system error propagation
SafeDllCall() {
    try {
        result := DllCall("User32\GetForegroundWindow", "Ptr")
        if result = 0
            throw Error("GetForegroundWindow returned NULL", -1)
        return result
    } catch OSError as e {
        MsgBox("OSError: " e.Message " (code " e.Extra ")")
        return 0
    }
}

; ════════════════════════════════════════════════════════════════════════════════════
; ── VIRTUAL MEMORY — VirtualAlloc / VirtualFree ───────────────────────────────────
; VirtualAlloc allocates memory at page granularity (typically 4 KB).
; Use when you need executable memory, large scratch space, or controlled protection.
; ════════════════════════════════════════════════════════════════════════════════════

; LPVOID VirtualAlloc(LPVOID lpAddress, SIZE_T dwSize, DWORD flAllocType, DWORD flProtect)
; ✓ Return type "Ptr" — LPVOID is pointer-sized; "Int" truncates the address on x64
MEM_COMMIT_RESERVE := 0x3000   ; MEM_COMMIT (0x1000) | MEM_RESERVE (0x2000)
PAGE_READWRITE     := 0x04

addr := DllCall("Kernel32\VirtualAlloc",
    "Ptr",  0,                    ; lpAddress = NULL (let OS choose)
    "UPtr", 4096,                 ; dwSize = one page (SIZE_T → UPtr)
    "UInt", MEM_COMMIT_RESERVE,   ; flAllocType
    "UInt", PAGE_READWRITE,       ; flProtect
    "Ptr")                        ; ✓ LPVOID return

if addr = 0
    MsgBox("VirtualAlloc failed: " A_LastError)
else {
    ; ✓ Write and read back using NumPut/NumGet on the raw address
    NumPut("UInt", 0xDEADBEEF, addr, 0)
    val := NumGet(addr, 0, "UInt")
    MsgBox(Format("VirtualAlloc: wrote 0x{:X} at 0x{:X}", val, addr))

    ; BOOL VirtualFree(LPVOID lpAddress, SIZE_T dwSize, DWORD dwFreeType)
    ; ✓ MEM_RELEASE (0x8000) requires dwSize = 0 — any non-zero size causes ERROR_INVALID_PARAMETER
    ok := DllCall("Kernel32\VirtualFree",
        "Ptr",  addr,
        "UPtr", 0,         ; dwSize must be 0 for MEM_RELEASE
        "UInt", 0x8000,    ; MEM_RELEASE
        "Int")
    if !ok
        MsgBox("VirtualFree failed: " A_LastError)
}

; ── VirtualProtect — change protection on an already-committed page ───────────────
; ✓ Pass a 4-byte Buffer for lpflOldProtect (output DWORD)
VirtualAllocAndProtect() {
    local addr := DllCall("Kernel32\VirtualAlloc",
        "Ptr", 0, "UPtr", 4096, "UInt", 0x3000, "UInt", 0x04, "Ptr")
    if !addr
        return

    local oldProt := Buffer(4, 0)    ; PDWORD lpflOldProtect output slot
    ; Change to PAGE_READONLY (0x02)
    DllCall("Kernel32\VirtualProtect",
        "Ptr",  addr,
        "UPtr", 4096,
        "UInt", 0x02,      ; PAGE_READONLY
        "Ptr",  oldProt,   ; receives the previous protection value
        "Int")
    MsgBox("Old protection: 0x" Format("{:X}", NumGet(oldProt, 0, "UInt")))  ; → 0x4

    DllCall("Kernel32\VirtualFree", "Ptr", addr, "UPtr", 0, "UInt", 0x8000, "Int")
}
VirtualAllocAndProtect()

; ════════════════════════════════════════════════════════════════════════════════════
; ── SYSTEM INFORMATION — GetSystemInfo / GetComputerNameW ─────────────────────────
; ════════════════════════════════════════════════════════════════════════════════════

; typedef struct _SYSTEM_INFO {
;   WORD  wProcessorArchitecture;  // offset 0  (UShort)
;   WORD  wReserved;               // offset 2
;   DWORD dwPageSize;              // offset 4  (UInt)
;   LPVOID lpMinimumApplicationAddress; // offset 8  (Ptr)
;   LPVOID lpMaximumApplicationAddress; // offset 8+A_PtrSize (Ptr)
;   DWORD_PTR dwActiveProcessorMask;    // offset 8+A_PtrSize*2 (UPtr)
;   DWORD dwNumberOfProcessors;    // offset 8+A_PtrSize*3 (UInt)
;   DWORD dwProcessorType;         // offset 12+A_PtrSize*3 (UInt)
;   DWORD dwAllocationGranularity; // offset 16+A_PtrSize*3 (UInt)
;   WORD  wProcessorLevel;         // offset 20+A_PtrSize*3 (UShort)
;   WORD  wProcessorRevision;      // offset 22+A_PtrSize*3 (UShort)
; }  SYSTEM_INFO; (total 48 bytes on x64)

; ✓ GetSystemInfo is void — no return type needed; allocate at least 48 bytes
sysInfo := Buffer(48, 0)
DllCall("Kernel32\GetSystemInfo", "Ptr", sysInfo)

; Architecture constants: 0=x86, 9=x64/AMD64, 12=ARM64
arch        := NumGet(sysInfo, 0,                "UShort")
pageSize    := NumGet(sysInfo, 4,                "UInt")
numCPUs     := NumGet(sysInfo, 8 + A_PtrSize*3,  "UInt")   ; dwNumberOfProcessors
granularity := NumGet(sysInfo, 16 + A_PtrSize*3, "UInt")   ; dwAllocationGranularity

archName := (arch = 9) ? "x64" : (arch = 12) ? "ARM64" : (arch = 0) ? "x86" : "Unknown"
MsgBox("Architecture: "  archName
    "`nPage size: "      pageSize    " bytes"
    "`nLogical CPUs: "   numCPUs
    "`nAlloc granularity: " granularity " bytes")

; ── GetComputerNameW — retrieve this machine's NetBIOS name ──────────────────────
; BOOL GetComputerNameW(LPWSTR lpBuffer, LPDWORD nSize)
; MAX_COMPUTERNAME_LENGTH = 15 characters; buffer = (15+1) * 2 bytes for wide chars
MAX_COMPUTERNAME := 16
nameBuf  := Buffer(MAX_COMPUTERNAME * 2, 0)   ; wide-char buffer
nameSize := MAX_COMPUTERNAME                   ; character count (not bytes)

; ✓ "UIntP" passes the address of nameSize (LPDWORD); API writes the actual char count back
ok := DllCall("Kernel32\GetComputerNameW",
    "Ptr",   nameBuf,
    "UIntP", nameSize,    ; IN/OUT: pass count, receive actual length
    "Int")
if ok
    MsgBox("Computer name: " StrGet(nameBuf, "UTF-16"))

; ════════════════════════════════════════════════════════════════════════════════════
; ── FormatMessageW — convert Win32 error code to human-readable string ────────────
; Eliminates the need for a lookup table of error code descriptions.
; ════════════════════════════════════════════════════════════════════════════════════

; FORMAT_MESSAGE_FROM_SYSTEM  = 0x00001000
; FORMAT_MESSAGE_IGNORE_INSERTS = 0x00000200
; Combined flag               = 0x00001200
FormatWin32Error(errCode) {
    local msgBuf := Buffer(512 * 2, 0)   ; 512 wide chars
    local len := DllCall("Kernel32\FormatMessageW",
        "UInt", 0x1200,   ; FORMAT_MESSAGE_FROM_SYSTEM | FORMAT_MESSAGE_IGNORE_INSERTS
        "Ptr",  0,         ; lpSource = NULL (use system message table)
        "UInt", errCode,
        "UInt", 0x0400,   ; MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT) = 0x0400
        "Ptr",  msgBuf,
        "UInt", 256,       ; nSize in wide chars
        "Ptr",  0,         ; va_list = NULL
        "UInt")
    if len = 0
        return "Unknown error 0x" Format("{:08X}", errCode)
    ; Strip trailing \r\n that FormatMessage appends
    local msg := StrGet(msgBuf, "UTF-16")
    return RTrim(msg, "`r`n ")
}

; ✓ Any A_LastError value can be converted to a readable description
result := DllCall("User32\GetForegroundWindow", "Ptr")
if result = 0
    MsgBox("Error: " FormatWin32Error(A_LastError))
else
    MsgBox(FormatWin32Error(0))   ; → "The operation completed successfully."

; ════════════════════════════════════════════════════════════════════════════════════
; ── WideCharToMultiByte / MultiByteToWideChar — explicit encoding conversion ───────
; Use when StrPut/StrGet encoding support is insufficient (e.g., GB2312, Shift-JIS,
; or when you need the Windows codepage flags for best-fit mapping control).
; ════════════════════════════════════════════════════════════════════════════════════

; ── Wide (UTF-16) → UTF-8 using WideCharToMultiByte ──────────────────────────────
; ✓ Two-call pattern: first call with dstLen=0 to get required byte count, then convert
WideToUTF8(wideStr) {
    ; Step 1: compute required byte count (including null terminator)
    local byteCount := DllCall("Kernel32\WideCharToMultiByte",
        "UInt", 65001,      ; CP_UTF8
        "UInt", 0,          ; dwFlags = 0
        "Str",  wideStr,    ; lpWideCharStr (AHK "Str" passes UTF-16 buffer)
        "Int",  -1,         ; cchWideChar = -1 → null-terminated
        "Ptr",  0,          ; lpMultiByteStr = NULL (compute only)
        "Int",  0,          ; cbMultiByte = 0
        "Ptr",  0,          ; lpDefaultChar = NULL
        "Ptr",  0,          ; lpUsedDefaultChar = NULL
        "Int")
    if byteCount = 0
        return ""

    ; Step 2: allocate and convert
    local utf8Buf := Buffer(byteCount, 0)
    DllCall("Kernel32\WideCharToMultiByte",
        "UInt", 65001,
        "UInt", 0,
        "Str",  wideStr,
        "Int",  -1,
        "Ptr",  utf8Buf,
        "Int",  byteCount,
        "Ptr",  0,
        "Ptr",  0,
        "Int")
    ; ✓ StrGet with "UTF-8" recovers the string; byteCount-1 excludes the null
    return StrGet(utf8Buf, byteCount - 1, "UTF-8")
}

; ── UTF-8 bytes → Wide (UTF-16) using MultiByteToWideChar ─────────────────────────
; ✓ Two-call pattern: first call with cchWideChar=0 to get required wide-char count
UTF8ToWide(utf8Buf, byteCount) {
    ; Step 1: compute wide char count
    local wcharCount := DllCall("Kernel32\MultiByteToWideChar",
        "UInt", 65001,      ; CP_UTF8
        "UInt", 0,          ; dwFlags
        "Ptr",  utf8Buf,    ; lpMultiByteStr
        "Int",  byteCount,  ; cbMultiByte
        "Ptr",  0,          ; lpWideCharStr = NULL (compute only)
        "Int",  0,          ; cchWideChar = 0
        "Int")
    if wcharCount = 0
        return ""

    ; Step 2: allocate wide buffer and convert
    local wideBuf := Buffer(wcharCount * 2, 0)   ; 2 bytes per UTF-16 code unit
    DllCall("Kernel32\MultiByteToWideChar",
        "UInt", 65001,
        "UInt", 0,
        "Ptr",  utf8Buf,
        "Int",  byteCount,
        "Ptr",  wideBuf,
        "Int",  wcharCount,
        "Int")
    return StrGet(wideBuf, "UTF-16")
}

; ✓ Round-trip test: AHK string → UTF-8 bytes → AHK string
original   := "Hello, 世界"
asUTF8     := WideToUTF8(original)           ; → byte string
utf8Bytes  := Buffer(StrPut(original, "UTF-8") - 1, 0)
StrPut(original, utf8Bytes, "UTF-8")
recovered  := UTF8ToWide(utf8Bytes, utf8Bytes.Size)
MsgBox(recovered = original ? "Round-trip OK" : "Mismatch")
```

## TIER 4 — System Message Passing and Interception
> METHODS COVERED: SendMessage() · PostMessage() · OnMessage() · NumGet() · StrGet()

AHK v2's built-in `SendMessage` and `PostMessage` communicate with any window via message number, wParam, and lParam. `OnMessage` registers an AHK function to intercept Windows messages delivered to the script's own hidden window, enabling monitoring of system-wide broadcasts — device change events, power state transitions, and custom inter-process communications — without any DllCall overhead.
```ahk
; ── SendMessage — synchronous, waits for target to process and returns reply ──────
; ✓ Returns the LRESULT reply from the target window handler
len := SendMessage(0x000E, 0, 0, "Edit1", "ahk_class Notepad")  ; EM_GETTEXTLENGTH
MsgBox("Notepad text length: " len)

; ── PostMessage — asynchronous, returns immediately (no reply) ────────────────────
; ✓ Use PostMessage when delivery confirmation is not required
PostMessage(0x0010, 0, 0, , "ahk_class Notepad")   ; WM_CLOSE — does not wait

; ── OnMessage — monitor WM_DEVICECHANGE (0x0219) for USB events ──────────────────
; ✓ Register before any device events can fire — handler receives all future events
OnMessage(0x0219, OnDeviceChange)

OnDeviceChange(wParam, lParam, msg, hwnd) {
    if (wParam = 0x8000)       ; DBT_DEVICEARRIVAL — device connected
        MsgBox("A device was connected.")
    else if (wParam = 0x8004)  ; DBT_DEVICEREMOVECOMPLETE — device removed
        MsgBox("A device was disconnected.")
    ; For WM_DEVICECHANGE, no return value is required for monitoring purposes
}

; ── OnMessage — monitor WM_POWERBROADCAST (0x0218) for sleep and wake ────────────
OnMessage(0x0218, OnPowerEvent)

OnPowerEvent(wParam, lParam, msg, hwnd) {
    if (wParam = 4)             ; PBT_APMSUSPEND (0x4)
        MsgBox("System going to sleep.")
    else if (wParam = 0x12)     ; PBT_APMRESUMEAUTOMATIC (0x12)
        MsgBox("System resumed from sleep.")
}

; ── OnMessage — WM_COPYDATA (0x004A) for IPC between AHK scripts ─────────────────
OnMessage(0x004A, OnCopyData)

OnCopyData(wParam, lParam, msg, hwnd) {
    ; COPYDATASTRUCT layout (x86 / x64):
    ;   dwData   : UPtr  at offset 0              (4 / 8 bytes)
    ;   cbData   : UInt  at offset A_PtrSize       (4 bytes)
    ;   <padding>          4 bytes on x64 only, to 8-byte-align lpData
    ;   lpData   : Ptr   at offset A_PtrSize * 2  (4 / 8 bytes)
    ;   Total size: A_PtrSize * 3                  (12 / 24 bytes)
    ; ✓ A_PtrSize-aware offsets make the read correct on both x86 and x64
    cbData  := NumGet(lParam, A_PtrSize,     "UInt")
    lpData  := NumGet(lParam, A_PtrSize * 2, "Ptr")
    if cbData > 0 {
        payload := StrGet(lpData, cbData // 2, "UTF-16")
        MsgBox("Received IPC payload: " payload)
    }
    return 1    ; acknowledge receipt
}

; ── Sending WM_COPYDATA from another script ───────────────────────────────────────
; ✓ Use A_PtrSize * 3 for COPYDATASTRUCT size — matches x86 and x64 field alignment
SendIPC(targetTitle, dataStr) {
    local cds  := Buffer(A_PtrSize * 3, 0)   ; COPYDATASTRUCT (12 bytes x86 / 24 bytes x64)
    local strB := Buffer(StrPut(dataStr, "UTF-16"), 0)
    StrPut(dataStr, strB, "UTF-16")

    NumPut("UPtr", 1,         cds, 0)                    ; dwData = app-defined tag
    NumPut("UInt", strB.Size, cds, A_PtrSize)             ; cbData = byte count
    NumPut("Ptr",  strB.Ptr,  cds, A_PtrSize * 2)         ; lpData = string buffer ptr

    SendMessage(0x004A, 0, cds.Ptr, , targetTitle)
}

; ── Limit OnMessage handler to a single concurrent thread ─────────────────────────
; ✓ Third argument = maxThreads; 1 prevents re-entry during long handlers
OnMessage(0x0200, OnMouseMove, 1)      ; WM_MOUSEMOVE

OnMouseMove(wParam, lParam, msg, hwnd) {
    x := lParam & 0xFFFF
    y := (lParam >> 16) & 0xFFFF
    ToolTip("Mouse: " x ", " y)
}
```

## TIER 5 — Struct Simulation, Heap and Global Memory
> METHODS COVERED: Buffer() · NumPut() · NumGet() · DllCall() · WinStruct (base class) · RECT (extends WinStruct) · SYSTEMTIME_Struct (extends WinStruct) · GetWindowRect · GetLocalTime · GetWindowPlacement · SetWindowPlacement · GetModuleHandleW · GetProcAddress · GetProcessHeap · HeapAlloc · HeapFree · HeapSize · GlobalAlloc · GlobalFree · GlobalLock · GlobalUnlock · GlobalSize · CreateMutexW · WaitForSingleObject · ReleaseMutex · CloseHandle

Windows APIs that accept or return structs require a correctly-sized, correctly-aligned Buffer with fields accessed at documented byte offsets. AHK v2 has no native struct type, so structs are simulated by computing offsets from the official Windows API documentation and using `NumPut`/`NumGet` with the correct type strings. Wrapping a struct in a class (extending `WinStruct`) centralizes offset knowledge and exposes safe named accessors — eliminating raw magic-number offsets at every call site. This tier also covers the process heap and global memory APIs, which are required for COM interop (clipboard, shell), and synchronization primitives for guarding shared state accessed from multiple AHK threads.
```ahk
; ── WinStruct — base class for all C-struct simulations ──────────────────────────
; ✓ Subclasses set a static ByteSize, call super.__New(), and expose field offsets
; as named property accessors using GetInt / GetUShort etc.
class WinStruct {
    _buf := unset                          ; internal Buffer object

    __New(byteSize, fillByte := 0) {
        this._buf := Buffer(byteSize, fillByte)
    }

    ; ✓ Read-only passthrough properties — single-expression get accessor is valid
    Ptr  { get => this._buf.Ptr  }
    Size { get => this._buf.Size }

    ; Generic typed field readers
    GetInt(offset)     { return NumGet(this._buf, offset, "Int")    }
    GetUInt(offset)    { return NumGet(this._buf, offset, "UInt")   }
    GetShort(offset)   { return NumGet(this._buf, offset, "Short")  }
    GetUShort(offset)  { return NumGet(this._buf, offset, "UShort") }
    GetPtr(offset)     { return NumGet(this._buf, offset, "Ptr")    }

    ; Generic typed field writers
    SetInt(offset, v)  { NumPut("Int",    v, this._buf, offset) }
    SetUInt(offset, v) { NumPut("UInt",   v, this._buf, offset) }
    SetShort(offset,v) { NumPut("Short",  v, this._buf, offset) }
    SetPtr(offset, v)  { NumPut("Ptr",    v, this._buf, offset) }

    __Delete() {
        ; Buffer is reference-counted and freed automatically.
        ; Override in subclasses to release external handles or resources.
    }
}

; ✓ Concrete struct: RECT (4 × Int32 = 16 bytes)
; typedef struct tagRECT { LONG left; LONG top; LONG right; LONG bottom; } RECT;
class RECT extends WinStruct {
    static ByteSize := 16

    __New() {
        super.__New(RECT.ByteSize)       ; ✓ ClassName.Property for static access in __New
    }

    Left   { get => this.GetInt(0)  }
    Top    { get => this.GetInt(4)  }
    Right  { get => this.GetInt(8)  }
    Bottom { get => this.GetInt(12) }
    Width  { get => this.Right  - this.Left }
    Height { get => this.Bottom - this.Top  }
}

; ── RECT struct — GetWindowRect ───────────────────────────────────────────────────
; Layout: left=Int@0, top=Int@4, right=Int@8, bottom=Int@12  (total 16 bytes)
rect := Buffer(16, 0)
hwnd := WinExist("A")
if hwnd {
    DllCall("User32\GetWindowRect", "Ptr", hwnd, "Ptr", rect, "Int")
    left   := NumGet(rect, 0,  "Int")   ; RECT.left
    top    := NumGet(rect, 4,  "Int")   ; RECT.top
    right  := NumGet(rect, 8,  "Int")   ; RECT.right
    bottom := NumGet(rect, 12, "Int")   ; RECT.bottom
    MsgBox("Window: " left "," top " → " right "," bottom
        " (" (right-left) "×" (bottom-top) ")")
}

; ── SYSTEMTIME struct — GetLocalTime ─────────────────────────────────────────────
; typedef struct _SYSTEMTIME {
;   WORD Year, Month, DayOfWeek, Day, Hour, Minute, Second, Milliseconds;
; } SYSTEMTIME;
; Layout: 8 × UShort at offsets 0,2,4,6,8,10,12,14  (total 16 bytes)
st := Buffer(16, 0)
DllCall("Kernel32\GetLocalTime", "Ptr", st)
year   := NumGet(st, 0,  "UShort")   ; SYSTEMTIME.wYear
month  := NumGet(st, 2,  "UShort")   ; SYSTEMTIME.wMonth
day    := NumGet(st, 6,  "UShort")   ; SYSTEMTIME.wDay   (index 3, offset 6)
hour   := NumGet(st, 8,  "UShort")   ; SYSTEMTIME.wHour
minute := NumGet(st, 10, "UShort")   ; SYSTEMTIME.wMinute
second := NumGet(st, 12, "UShort")   ; SYSTEMTIME.wSecond
ms     := NumGet(st, 14, "UShort")   ; SYSTEMTIME.wMilliseconds
MsgBox(year "-" Format("{:02}", month) "-" Format("{:02}", day)
    " " Format("{:02}", hour) ":" Format("{:02}", minute)
    ":" Format("{:02}", second) "." Format("{:03}", ms))

; ── OOP wrapper for SYSTEMTIME using WinStruct base ──────────────────────────────
; ✓ Named accessors eliminate magic offsets at every call site
class SYSTEMTIME_Struct extends WinStruct {
    static ByteSize := 16

    __New() {
        super.__New(SYSTEMTIME_Struct.ByteSize)
        DllCall("Kernel32\GetLocalTime", "Ptr", this.Ptr)
    }

    Year         { get => this.GetUShort(0)  }
    Month        { get => this.GetUShort(2)  }
    DayOfWeek    { get => this.GetUShort(4)  }
    Day          { get => this.GetUShort(6)  }
    Hour         { get => this.GetUShort(8)  }
    Minute       { get => this.GetUShort(10) }
    Second       { get => this.GetUShort(12) }
    Milliseconds { get => this.GetUShort(14) }
}

; ✓ Clean named access — no raw offsets at call sites
now := SYSTEMTIME_Struct()
MsgBox(now.Year "-" now.Month "-" now.Day)

; ── WINDOWPLACEMENT struct — save and restore window position ─────────────────────
; typedef struct tagWINDOWPLACEMENT {
;   UINT  length;           // offset 0   (4 bytes)
;   UINT  flags;            // offset 4   (4 bytes)
;   UINT  showCmd;          // offset 8   (4 bytes) SW_SHOWNORMAL=1, SW_SHOWMINIMIZED=2
;   POINT ptMinPosition;    // offset 12  (8 bytes: x@12, y@16)
;   POINT ptMaxPosition;    // offset 20  (8 bytes: x@20, y@24)
;   RECT  rcNormalPosition; // offset 28  (16 bytes: l@28, t@32, r@36, b@40)
; } WINDOWPLACEMENT;        // total 44 bytes

SaveWindowPlacement(hwnd) {
    local wp := Buffer(44, 0)
    ; ✓ length field MUST be set before calling GetWindowPlacement — API reads it to verify
    NumPut("UInt", 44, wp, 0)    ; length
    DllCall("User32\GetWindowPlacement", "Ptr", hwnd, "Ptr", wp, "Int")
    return {
        showCmd  : NumGet(wp, 8,  "UInt"),   ; WINDOWPLACEMENT.showCmd
        normLeft : NumGet(wp, 28, "Int"),    ; rcNormalPosition.left
        normTop  : NumGet(wp, 32, "Int"),    ; rcNormalPosition.top
        normRight: NumGet(wp, 36, "Int"),    ; rcNormalPosition.right
        normBot  : NumGet(wp, 40, "Int")     ; rcNormalPosition.bottom
    }
}

RestoreWindowPlacement(hwnd, saved) {
    local wp := Buffer(44, 0)
    NumPut("UInt", 44,            wp, 0)    ; length
    NumPut("UInt", saved.showCmd, wp, 8)    ; showCmd
    NumPut("Int",  saved.normLeft,  wp, 28)
    NumPut("Int",  saved.normTop,   wp, 32)
    NumPut("Int",  saved.normRight, wp, 36)
    NumPut("Int",  saved.normBot,   wp, 40)
    DllCall("User32\SetWindowPlacement", "Ptr", hwnd, "Ptr", wp, "Int")
}

; Usage:
if hwnd := WinExist("A") {
    saved := SaveWindowPlacement(hwnd)
    MsgBox("showCmd=" saved.showCmd)
    RestoreWindowPlacement(hwnd, saved)
}

; ════════════════════════════════════════════════════════════════════════════════════
; ── PROCESS HEAP — HeapAlloc / HeapFree ──────────────────────────────────────────
; Use the process heap when interoperating with COM components or Win32 APIs that
; expect HeapFree-compatible allocations, or when you need raw heap memory whose
; lifetime you control manually (without AHK's Buffer reference counting).
; ════════════════════════════════════════════════════════════════════════════════════

; ✓ GetProcessHeap() returns the default process heap handle — reuse across calls
hHeap := DllCall("Kernel32\GetProcessHeap", "Ptr")
MsgBox("Process heap handle: 0x" Format("{:X}", hHeap))

; HEAP_ZERO_MEMORY = 8 — fills allocated bytes with zero (equivalent to Buffer(n, 0))
; ✓ "Ptr" return for HeapAlloc — LPVOID is pointer-sized
heapPtr := DllCall("Kernel32\HeapAlloc",
    "Ptr",  hHeap,
    "UInt", 8,        ; HEAP_ZERO_MEMORY
    "UPtr", 128,      ; dwBytes (SIZE_T → UPtr)
    "Ptr")

if heapPtr = 0
    MsgBox("HeapAlloc failed: " A_LastError)
else {
    ; Write data into the heap block using NumPut with the raw address
    NumPut("UInt", 0xCAFEBABE, heapPtr, 0)
    val := NumGet(heapPtr, 0, "UInt")
    MsgBox(Format("HeapAlloc: wrote 0x{:X}", val))

    ; Confirm allocation size via HeapSize
    allocSz := DllCall("Kernel32\HeapSize",
        "Ptr",  hHeap,
        "UInt", 0,
        "Ptr",  heapPtr,
        "UPtr")                    ; SIZE_T return
    MsgBox("HeapSize: " allocSz " bytes (requested 128)")

    ; ✓ Always free with the same heap handle used at allocation
    DllCall("Kernel32\HeapFree", "Ptr", hHeap, "UInt", 0, "Ptr", heapPtr, "Int")
    ; ✗ DO NOT call HeapFree on a Buffer's .Ptr — Buffer uses its own allocator
}

; ════════════════════════════════════════════════════════════════════════════════════
; ── GLOBAL MEMORY — GlobalAlloc / GlobalLock / GlobalUnlock / GlobalFree ─────────
; Required for clipboard operations (CF_UNICODETEXT, CF_HDROP) and some shell APIs
; that document "pass an HGLOBAL" — these expect GlobalAlloc/GlobalFree lifecycle.
; ════════════════════════════════════════════════════════════════════════════════════

; GMEM_MOVEABLE = 0x0002 — required for clipboard; OS may move the block
; GMEM_ZEROINIT = 0x0040 — zero-fills on allocation
; Combined:        0x0042
CopyTextToClipboard(text) {
    local byteSize := StrPut(text, "UTF-16")   ; includes null terminator
    local hGlobal  := DllCall("Kernel32\GlobalAlloc",
        "UInt", 0x0042,       ; GMEM_MOVEABLE | GMEM_ZEROINIT
        "UPtr", byteSize,     ; SIZE_T → UPtr
        "Ptr")                ; HGLOBAL return → Ptr

    if hGlobal = 0
        throw Error("GlobalAlloc failed: " A_LastError, -1)

    ; ✓ Lock the moveable block to get a usable pointer
    local pData := DllCall("Kernel32\GlobalLock", "Ptr", hGlobal, "Ptr")
    if pData = 0 {
        DllCall("Kernel32\GlobalFree", "Ptr", hGlobal, "Ptr")
        throw Error("GlobalLock failed: " A_LastError, -1)
    }

    StrPut(text, pData, "UTF-16")

    ; ✓ Must unlock before passing to clipboard — GlobalUnlock decrements lock count
    DllCall("Kernel32\GlobalUnlock", "Ptr", hGlobal, "Int")

    ; Open clipboard, set data, close (OS takes ownership of hGlobal on SetClipboardData)
    if DllCall("User32\OpenClipboard", "Ptr", 0, "Int") {
        DllCall("User32\EmptyClipboard", "Int")
        DllCall("User32\SetClipboardData",
            "UInt", 13,        ; CF_UNICODETEXT = 13
            "Ptr",  hGlobal,   ; hGlobal ownership transferred to OS — do NOT free
            "Ptr")
        DllCall("User32\CloseClipboard", "Int")
    } else {
        ; ✓ Free only if we did not transfer ownership to the clipboard
        DllCall("Kernel32\GlobalFree", "Ptr", hGlobal, "Ptr")
        throw Error("OpenClipboard failed: " A_LastError, -1)
    }
}

; ✓ Check GlobalSize before writing to verify allocation succeeded
VerifyGlobalAlloc(hGlobal, expected) {
    local sz := DllCall("Kernel32\GlobalSize", "Ptr", hGlobal, "UPtr")
    return sz >= expected
}

CopyTextToClipboard("Hello from AHK v2 via GlobalAlloc!")
MsgBox("Clipboard set — Ctrl+V to paste")

; ════════════════════════════════════════════════════════════════════════════════════
; ── SYNCHRONIZATION — CreateMutexW / WaitForSingleObject / ReleaseMutex ──────────
; Use named mutexes to enforce single-instance execution or to protect shared state
; accessed from multiple SetTimer callbacks or threads.
; ════════════════════════════════════════════════════════════════════════════════════

; MUTEX_ALL_ACCESS  = 0x1F0001
; WAIT_OBJECT_0     = 0x00000000  (mutex acquired successfully)
; WAIT_TIMEOUT      = 0x00000102  (timeout elapsed before acquisition)
; INFINITE          = 0xFFFFFFFF  (wait forever)

; ── Named mutex — single-instance guard ──────────────────────────────────────────
; ✓ "Ptr" return for HANDLE — CreateMutexW returns a HANDLE (pointer-sized)
hMutex := DllCall("Kernel32\CreateMutexW",
    "Ptr",  0,                    ; lpMutexAttributes = NULL (no security descriptor)
    "Int",  0,                    ; bInitialOwner = FALSE
    "Str",  "MyApp_SingleInst",   ; lpName — unique per-app string
    "Ptr")                        ; HANDLE return

if hMutex = 0
    throw Error("CreateMutexW failed: " A_LastError, -1)

; ERROR_ALREADY_EXISTS = 183 — another instance created this mutex before us
if A_LastError = 183 {
    MsgBox("Another instance is already running.")
    DllCall("Kernel32\CloseHandle", "Ptr", hMutex, "Int")
    ExitApp
}

; ── Timed wait before acquiring mutex ownership ───────────────────────────────────
; ✓ 1000 ms timeout — avoids indefinite blocking if the mutex owner hangs
waitResult := DllCall("Kernel32\WaitForSingleObject",
    "Ptr",  hMutex,
    "UInt", 1000,    ; dwMilliseconds — INFINITE (0xFFFFFFFF) waits forever
    "UInt")

if waitResult = 0x00000000 {        ; WAIT_OBJECT_0 — mutex acquired
    ; ── critical section ──
    MsgBox("Mutex acquired. Running exclusive code.")
    ; ── release after critical section ──
    ; ✓ ReleaseMutex must be called by the same thread that called WaitForSingleObject
    DllCall("Kernel32\ReleaseMutex", "Ptr", hMutex, "Int")
} else if waitResult = 0x00000102 { ; WAIT_TIMEOUT
    MsgBox("Mutex wait timed out — another instance holds the lock.")
} else {
    MsgBox("WaitForSingleObject failed: " A_LastError)
}

; ✓ Always close the handle when done — kernel object leak if omitted
DllCall("Kernel32\CloseHandle", "Ptr", hMutex, "Int")
```

### Performance Notes

DllCall incurs per-call overhead from type marshalling and AHK's internal name-resolution dispatch. For hot loops calling the same function repeatedly, resolve the function address once via `GetProcAddress` and call by numeric address — this eliminates a `GetProcAddress` lookup on every iteration. Buffer allocation is not free: creating a new `Buffer` inside a loop forces an alloc/free cycle on every iteration; allocate once outside the loop and reuse. Callback stubs are expensive: `CallbackCreate` inside a loop creates a new native stub on every iteration; create once at initialization and store the address. String argument choice matters: use `"Str"` for one-shot arguments (AHK passes the string's own UTF-16 buffer directly); use explicit `Buffer + StrPut` only when the API retains the string pointer beyond the call frame or in async flows.

For block memory operations: `RtlMoveMemory`/`RtlZeroMemory`/`RtlFillMemory` are implemented in C-speed native code and are dramatically faster than equivalent `NumPut` loops — prefer them for any buffer larger than ~32 bytes. `RtlCompareMemory` short-circuits on the first mismatch, making it O(1) in the common cache-hit case; however it always scans the full `len` bytes in the worst case, so minimize `len` to the meaningful region. `VirtualAlloc` and `HeapAlloc` both add per-call overhead well beyond `Buffer` creation; use them only when the calling convention requires it (clipboard, COM, executable memory), not as a general-purpose replacement for `Buffer`.

`WideCharToMultiByte` and `MultiByteToWideChar` are O(n) in the string length; cache the result when the same string is converted repeatedly in a tight loop. `FormatMessageW` with `FORMAT_MESSAGE_FROM_SYSTEM` performs a message table lookup on every call — cache error strings at the call site rather than calling `FormatMessageW` inside a retry loop.

Mutex acquisition (`WaitForSingleObject`) is a kernel-mode transition costing several microseconds even in the uncontested case; use it to guard shared state across timer callbacks or `OnMessage` handlers, but do not acquire a mutex on every key press in a `#HotIf`-guarded hotkey if the critical section is trivially short.
```ahk
; ── Cache function address for high-frequency DllCall invocations ─────────────────
; ✓ Resolve name once — subsequent calls use the cached Ptr with no lookup overhead
hUser32    := DllCall("Kernel32\GetModuleHandleW", "Str", "user32.dll", "Ptr")
pSetWinPos := DllCall("Kernel32\GetProcAddress",
    "Ptr",  hUser32,
    "AStr", "SetWindowPos",     ; ✓ GetProcAddress requires an ANSI ("AStr") function name
    "Ptr")

; Subsequent calls use the cached Ptr — no name lookup overhead per iteration
; DllCall(pSetWinPos, "Ptr", hwnd, "Ptr", 0,
;         "Int", x, "Int", y, "Int", w, "Int", h, "UInt", flags, "Int")

; ── Reuse Buffer objects across loop iterations ────────────────────────────────────
; ✓ Allocate once outside the loop — zero alloc/free overhead per iteration
pt := Buffer(8, 0)
Loop 1000 {
    DllCall("User32\GetCursorPos", "Ptr", pt)
    x := NumGet(pt, 0, "Int")
    y := NumGet(pt, 4, "Int")
    ; process x, y ...
}

; ✗ Avoid: per-iteration Buffer allocation — 1000 alloc/free cycles
; Loop 1000 {
;     localPt := Buffer(8, 0)     ; → new allocation every iteration
;     DllCall("User32\GetCursorPos", "Ptr", localPt)
; }

; ── Create callbacks once at startup, not inside loops ───────────────────────────
; ✓ Create once, reuse address, free on cleanup
globalCb := CallbackCreate(MyProc, "Fast", 2)
Loop 5 {
    DllCall("User32\EnumWindows", "Ptr", globalCb, "Ptr", 0, "Int")
}
CallbackFree(globalCb)

MyProc(hwnd, lParam) {
    return 1
}

; ✗ Avoid: CallbackCreate inside loop — new stub every iteration; leaks if not freed
; Loop 5 {
;     cb := CallbackCreate(MyProc, "Fast", 2)   ; → new native stub each iteration
;     DllCall("User32\EnumWindows", "Ptr", cb, "Ptr", 0, "Int")
;     CallbackFree(cb)   ; freed here, but alloc overhead remains
; }

; ── Prefer "Str" for one-shot API arguments ───────────────────────────────────────
; ✓ AHK passes the string's own UTF-16 buffer directly — zero boilerplate
DllCall("User32\MessageBoxW", "Ptr", 0, "Str", "Fast", "Str", "Title", "UInt", 0)

; ✗ Unnecessary for non-retained strings — Buffer+StrPut adds boilerplate with no benefit
; tmpBuf := Buffer(StrPut("Fast", "UTF-16"), 0)
; StrPut("Fast", tmpBuf, "UTF-16")
; DllCall("User32\MessageBoxW", "Ptr", 0, "Ptr", tmpBuf.Ptr, "Str", "Title", "UInt", 0)

; ✓ Use explicit Buffer+StrPut only when the API retains the pointer after the call
; (e.g., async operations or OUT parameter buffers that must outlive the DllCall frame)

; ── Prefer RtlMoveMemory over NumPut loop for bulk copy ───────────────────────────
; ✓ Single ntdll call — C-speed; no AHK dispatch overhead per byte
src := Buffer(4096, 0xCC)
dst := Buffer(4096, 0)
DllCall("ntdll\RtlMoveMemory", "Ptr", dst, "Ptr", src, "UPtr", src.Size)

; ✗ Avoid: NumPut loop for large copies — 4096 individual AHK dispatch calls
; Loop 4096
;     NumPut("UChar", NumGet(src, A_Index-1, "UChar"), dst, A_Index-1)

; ── NumGet on raw address vs Buffer — equivalent speed; Buffer adds bounds safety ──
; ✓ Both forms are accepted; prefer the Buffer form in non-performance-critical paths
rawAddr := DllCall("SomeModule\GetPtr", "Ptr")
if rawAddr
    val := NumGet(rawAddr, 0, "UInt")       ; valid — raw integer address is accepted
```

## TIER 6 — C-Style Callbacks with CallbackCreate
> METHODS COVERED: CallbackCreate() · CallbackFree() · EnumWindows · EnumChildWindows · SetWindowsHookExW · UnhookWindowsHookEx · CallNextHookEx · WinGetTitle · WinGetClass · OutputDebug

`CallbackCreate` converts an AHK v2 function reference into a machine-code stub with a native calling convention, returning a `Ptr`-sized address that any Win32 API expecting a function pointer can invoke directly. The `"Fast"` option skips AHK's inter-thread overhead and is correct for synchronous callbacks such as `EnumWindows` and hook procedures. Every address from `CallbackCreate` must be freed with `CallbackFree` — typically in a `__Delete()` meta-function — to prevent a permanent native memory leak that persists for the life of the AHK process.
```ahk
; ── EnumWindows — collect all top-level window titles ────────────────────────────
; BOOL CALLBACK EnumWindowsProc(HWND hwnd, LPARAM lParam) — 2 parameters
titles := []

EnumWindowsProc(hwnd, lParam) {
    local t := WinGetTitle("ahk_id " hwnd)
    if (t != "")
        titles.Push(t)
    ; ✓ return 1 (non-zero) = continue enumeration
    ; ✗ return 0 — stops after the first window, silently missing all remaining windows
    return 1
}

cbAddr := CallbackCreate(EnumWindowsProc, "Fast", 2)
DllCall("User32\EnumWindows", "Ptr", cbAddr, "Ptr", 0, "Int")
; ✓ Always pair CallbackFree with CallbackCreate — leak is permanent if omitted
CallbackFree(cbAddr)
MsgBox("Found " titles.Length " titled windows.")

; ── EnumChildWindows — enumerate children of a specific parent ────────────────────
EnumChildProc(hwnd, lParam) {
    local cls := WinGetClass("ahk_id " hwnd)
    OutputDebug("Child class: " cls)
    return 1    ; ✓ must return 1 to continue iteration
}

if parentHwnd := WinExist("A") {
    cb2 := CallbackCreate(EnumChildProc, "Fast", 2)
    DllCall("User32\EnumChildWindows", "Ptr", parentHwnd, "Ptr", cb2, "Ptr", 0, "Int")
    CallbackFree(cb2)
}

; ── Low-level keyboard hook (WH_KEYBOARD_LL = 13) encapsulated in a class ────────
; LRESULT CALLBACK LowLevelKeyboardProc(int nCode, WPARAM wParam, LPARAM lParam)
; lParam → KBDLLHOOKSTRUCT:
;   vkCode   : UInt at offset 0
;   scanCode : UInt at offset 4
;   flags    : UInt at offset 8
;   time     : UInt at offset 12
;   dwExtraInfo: UPtr at offset 16

class KeyboardHook {
    _hHook   := 0
    _cbAddr  := 0

    __New() {
        ; ✓ .Bind(this) required — bare method reference loses instance context in callback
        this._cbAddr := CallbackCreate(this._Proc.Bind(this), "Fast", 3)
        this._hHook  := DllCall("User32\SetWindowsHookExW",
            "Int",  13,              ; WH_KEYBOARD_LL
            "Ptr",  this._cbAddr,
            "Ptr",  0,               ; hMod = NULL for global LL hook
            "UInt", 0,               ; dwThreadId = 0 (all threads)
            "Ptr")                   ; ✓ return type "Ptr" — HHOOK is pointer-sized
        if !this._hHook
            throw Error("SetWindowsHookExW failed: " A_LastError, -1)
    }

    _Proc(nCode, wParam, lParam) {
        if (nCode >= 0) {
            local vk := NumGet(lParam, 0, "UInt")        ; KBDLLHOOKSTRUCT.vkCode
            if (wParam = 0x0100)                         ; WM_KEYDOWN
                OutputDebug("KeyDown VK=0x" Format("{:X}", vk))
        }
        ; ✓ MUST call CallNextHookEx — omitting stalls keyboard input for ALL applications
        ; ✗ return 0 without chaining — input blocked system-wide
        return DllCall("User32\CallNextHookEx",
            "Ptr", this._hHook,
            "Int", nCode,
            "Ptr", wParam,
            "Ptr", lParam, "Ptr")
    }

    __Delete() {
        if this._hHook {
            DllCall("User32\UnhookWindowsHookEx", "Ptr", this._hHook, "Int")
            this._hHook := 0
        }
        if this._cbAddr {
            ; ✓ Free the native stub in __Delete — pairs every CallbackCreate
            CallbackFree(this._cbAddr)
            this._cbAddr := 0
        }
    }
}

; ✓ Hook is active for as long as the object remains in scope
hook := KeyboardHook()
MsgBox("Hook installed. Press OK to remove.")
hook := unset    ; __Delete() is called — hook uninstalled, callback freed

; ── Caching callbacks in a static class to avoid re-creation ─────────────────────
; ✓ Static class manages the callback address lifetime — avoids re-creation per call
class AppCallbacks {
    static _enumCb := 0

    static Init() {
        ; ✓ Access static property via ClassName.Property, not this.Property
        AppCallbacks._enumCb := CallbackCreate(AppCallbacks._OnEnum, "Fast", 2)
    }

    static _OnEnum(hwnd, lParam) {
        OutputDebug("hwnd=" hwnd)
        return 1    ; ✓ continue enumeration
    }

    static Cleanup() {
        if AppCallbacks._enumCb {
            CallbackFree(AppCallbacks._enumCb)
            AppCallbacks._enumCb := 0
        }
    }
}

AppCallbacks.Init()
DllCall("User32\EnumWindows", "Ptr", AppCallbacks._enumCb, "Ptr", 0, "Int")
AppCallbacks.Cleanup()
```

## ANTI-PATTERNS

| Pattern | Wrong | Correct | LLM Common Cause |
|---------|-------|---------|------------------|
| Buffer allocation | `VarSetCapacity(v, 64, 0)` | `v := Buffer(64, 0)` | AHK v1 training data — VarSetCapacity was the only allocator in v1 |
| HANDLE/HWND type | `"Int", hWnd` in DllCall | `"Ptr", hWnd` | 32-bit coding habit; `"Int"` worked on v1 x86 where pointers were 4 bytes |
| Missing return type | `DllCall("Kernel32\GetModuleHandleW", "Str", name)` | `DllCall("...", "Str", name, "Ptr")` | Default `"Int"` worked in v1 on 32-bit; 64-bit truncation is not immediately visible |
| NumPut/NumGet argument order | `NumPut(val, buf, offset, type)` | `NumPut(type, val, buf, offset)` | v1 NumPut argument order was `(Value, Target, Offset, Type)` — completely reversed |
| EnumWindows callback return | `return 0` or omit return | `return 1` | C/Boolean instinct — `0 = false = stop` seems wrong for "continue" semantics |
| Multi-statement arrow callback | `(n,w,l) => { stmt1; stmt2 }` passed to CallbackCreate | Define a named function, pass its reference | AHK v1 / JavaScript cross-contamination; fat-arrow with block body is not supported in v2 |
| Omit CallbackFree | `CallbackCreate(...)` with no paired free | Store address; call `CallbackFree(addr)` in `__Delete()` | AHK v1 had no CallbackCreate API; the create/free contract has no v1 analogue to anchor it |
| Leak .Ptr across scope | `dangerPtr := buf.Ptr` then use after `buf` goes out of scope | Pass the `Buffer` object itself across function boundaries | C/C++ pointer semantics — raw address appears stable; AHK's reference-count lifetime is invisible |
| Buffer inside loop | `Loop n { local pt := Buffer(8, 0); DllCall(...) }` | Allocate `pt` once before the loop and reuse | Defensive initialization habit; allocation cost is invisible in small loops but compounds at scale |
| SIZE_T as "UInt" | `DllCall("ntdll\RtlMoveMemory", "Ptr", dst, "Ptr", src, "UInt", size)` | `DllCall("ntdll\RtlMoveMemory", "Ptr", dst, "Ptr", src, "UPtr", size)` | "UInt" used for all unsigned integers in C-habit training data; SIZE_T is pointer-width, not fixed 32-bit |
| RtlCompareMemory as boolean | `if !DllCall("ntdll\RtlCompareMemory", ...)` | `if DllCall(..., "UPtr") = bufSize` | memcmp() in C returns 0/non-zero; RtlCompareMemory returns match *count*, not a difference value |
| VirtualAlloc return as "Int" | `addr := DllCall("Kernel32\VirtualAlloc", ..., "Int")` | `addr := DllCall("Kernel32\VirtualAlloc", ..., "Ptr")` | LPVOID looks like a "pointer" but LLMs map it to "Int" by habit from 32-bit API examples |
| HeapFree on Buffer.Ptr | `DllCall("Kernel32\HeapFree", "Ptr", hHeap, "UInt", 0, "Ptr", buf.Ptr)` | Never call HeapFree on a Buffer — Buffer uses its own allocator | C habit: "free everything you no longer need"; AHK Buffer and Win32 heap are independent allocators |
| GlobalFree after SetClipboardData | `DllCall("Kernel32\GlobalFree", "Ptr", hGlobal)` after a successful `SetClipboardData` | Do NOT free — ownership transfers to OS on successful `SetClipboardData` | COM/RAII training data pattern: "caller frees what caller allocates"; clipboard API violates this |
| Unclosed HANDLE | `hMutex := DllCall("Kernel32\CreateMutexW", ...)` with no `CloseHandle` | Always call `DllCall("Kernel32\CloseHandle", "Ptr", hMutex)` in cleanup | AHK variables freed by GC; kernel HANDLEs are not — no analogue in most scripting language training data |

## SEE ALSO

> This module does NOT cover: IStream COM interface, async I/O, and GDI/DirectX memory-mapped objects → see Module_SystemAndCOM.md
> This module does NOT cover: Structured exception handling for DllCall failures, OSError catch patterns, and A_LastError diagnostic workflows → see Module_Errors.md
> This module does NOT cover: OOP class design patterns for struct wrappers beyond the WinStruct base shown in TIER 5 → see Module_Classes.md

- `Module_Errors.md` — try/catch patterns for `OSError` from failed DllCall operations; `A_LastError` diagnostic workflows; structured exception propagation from Win32 API failures.
- `Module_Classes.md` — advanced OOP design for struct wrapper classes; inheritance chains; `__Get` / `__Set` meta-functions for property-mapped struct fields beyond the WinStruct accessor pattern shown in TIER 5.
- `Module_SystemAndCOM.md` — IStream COM interface for streaming I/O; COM object lifecycle; `ComObject`, `ComValue`, and `ComCall` for type-library-bound API access as an alternative to raw DllCall.