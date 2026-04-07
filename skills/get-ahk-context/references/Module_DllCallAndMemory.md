# Module_DllCallAndMemory.md
<!-- DOMAIN: DLL Calls and Memory Management -->
<!-- SCOPE: IStream COM interface, async I/O, and GDI/DirectX memory-mapped objects are not covered — see Module_SystemAndCOM.md -->
<!-- TRIGGERS: DllCall, Buffer, NumPut, NumGet, StrPut, StrGet, CallbackCreate, CallbackFree, OnMessage, SendMessage, PostMessage, A_LastError, VarSetCapacity, "call Windows API", "raw memory", "memory allocation", "struct simulation", "Win32 API", "low-level hook", "HANDLE", "HWND", "pointer arithmetic", "enumerate windows" -->
<!-- CONSTRAINTS: Buffer(size, fill) is the only valid memory allocator in AHK v2 — VarSetCapacity is removed and causes NameError at runtime. Every HANDLE, HWND, HMODULE, LPVOID, or pointer-sized DllCall argument and return value must use "Ptr", never "Int" — "Int" silently truncates the upper 32 bits of any address on 64-bit Windows. -->
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

## TIER 2 — Memory Read and Write
> METHODS COVERED: NumPut() · NumGet() · StrPut() · StrGet()

`NumPut` writes typed integer values into a Buffer at a byte offset; `NumGet` reads them back. `StrPut` encodes an AHK string into a Buffer using an explicit encoding, and `StrGet` decodes a Buffer or raw pointer back to an AHK string. Type and encoding specification is mandatory in every call — mismatches silently produce wrong values or corrupt multi-byte string content without throwing any error.
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
; ✗ NumGet(buf, 0) — throws ValueError in v2; Type is mandatory and has no default;
;     always specify the type explicitly

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
```

## TIER 3 — Simple DllCall
> METHODS COVERED: DllCall() · A_LastError · A_PtrSize · GetModuleHandleW · LoadLibraryW · GetProcAddress · FreeLibrary · GetCursorPos · GetForegroundWindow · SetWindowTextW · GetTickCount · MessageBoxW

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

## TIER 5 — Struct Simulation with DllCall
> METHODS COVERED: Buffer() · NumPut() · NumGet() · DllCall() · WinStruct (base class) · RECT (extends WinStruct) · SYSTEMTIME_Struct (extends WinStruct) · GetWindowRect · GetLocalTime · GetWindowPlacement · SetWindowPlacement · GetModuleHandleW · GetProcAddress

Windows APIs that accept or return structs require a correctly-sized, correctly-aligned Buffer with fields accessed at documented byte offsets. AHK v2 has no native struct type, so structs are simulated by computing offsets from the official Windows API documentation and using `NumPut`/`NumGet` with the correct type strings. Wrapping a struct in a class (extending `WinStruct`) centralizes offset knowledge and exposes safe named accessors — eliminating raw magic-number offsets at every call site.
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
```

### Performance Notes

DllCall incurs per-call overhead from type marshalling and AHK's internal name-resolution dispatch. For hot loops calling the same function repeatedly, resolve the function address once via `GetProcAddress` and call by numeric address — this eliminates a `GetProcAddress` lookup on every iteration. Buffer allocation is not free: creating a new `Buffer` inside a loop forces an alloc/free cycle on every iteration; allocate once outside the loop and reuse. Callback stubs are expensive: `CallbackCreate` inside a loop creates a new native stub on every iteration; create once at initialization and store the address. String argument choice matters: use `"Str"` for one-shot arguments (AHK passes the string's own UTF-16 buffer directly); use explicit `Buffer + StrPut` only when the API retains the string pointer beyond the call frame or in async flows.
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

## SEE ALSO

> This module does NOT cover: IStream COM interface, async I/O, and GDI/DirectX memory-mapped objects → see Module_SystemAndCOM.md
> This module does NOT cover: Structured exception handling for DllCall failures, OSError catch patterns, and A_LastError diagnostic workflows → see Module_Errors.md
> This module does NOT cover: OOP class design patterns for struct wrappers beyond the WinStruct base shown in TIER 5 → see Module_Classes.md

- `Module_Errors.md` — try/catch patterns for `OSError` from failed DllCall operations; `A_LastError` diagnostic workflows; structured exception propagation from Win32 API failures.
- `Module_Classes.md` — advanced OOP design for struct wrapper classes; inheritance chains; `__Get` / `__Set` meta-functions for property-mapped struct fields beyond the WinStruct accessor pattern shown in TIER 5.
- `Module_SystemAndCOM.md` — IStream COM interface for streaming I/O; COM object lifecycle; `ComObject`, `ComValue`, and `ComCall` for type-library-bound API access as an alternative to raw DllCall.