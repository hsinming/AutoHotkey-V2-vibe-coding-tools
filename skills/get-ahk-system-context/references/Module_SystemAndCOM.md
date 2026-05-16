# Module_SystemAndCOM.md
<!-- DOMAIN: System and COM -->
<!-- SCOPE: IStream/async COM interfaces, named-pipe IPC, Win32 memory-level OS access, and low-level DLL calls are not covered — see Module_DllCallAndMemory.md -->
<!-- TRIGGERS: Run, RunWait, A_Clipboard, ClipWait, RegRead, RegWrite, RegDelete, ComObject, ComObjActive, ComObjGet, ComObjConnect, ComObjArray, ProcessExist, WinWaitActive, "start application", "clipboard text", "write registry", "read excel cell", "WMI query", "kill process", "COM automation", "exit code", "zombie process" -->
<!-- CONSTRAINTS: COM teardown is order-critical — call .Quit()/.Close() on the host application BEFORE assigning "" to any reference; reversing the order leaves zombie processes. Wrap every COM method call in try/catch — unhandled COM exceptions are immediate non-resumable runtime errors that crash the script. Never call .Quit() on a ComObjActive() instance; that session was not created by the script. -->
<!-- CROSS-REF: Module_DllCallAndMemory.md, Module_WindowAndControl.md, Module_Errors.md, Module_FileSystem.md, Module_Classes.md -->
<!-- VERSION: AHK v2.0+ -->

## V1 → V2 BREAKING CHANGES

| v1 pattern (LLM commonly writes) | v2 correct form | Consequence |
|----------------------------------|-----------------|-------------|
| `Clipboard` built-in variable | `A_Clipboard` | `Clipboard` is undefined in v2 — NameError at runtime |
| `ComObjCreate("Excel.Application")` | `ComObject("Excel.Application")` | `ComObjCreate` is removed in v2 — NameError at runtime |
| `RegRead, OutputVar, KeyName, ValueName` | `val := RegRead(KeyName, ValueName?)` | v1 command syntax is a parse error in v2; OutputVar assignment is silently lost |
| `ErrorLevel` after RunWait | `exitCode := RunWait(...)` (return value) | ErrorLevel is deprecated in v2 and always returns 0 — exit code is silently discarded |
| `Run, %path%, , Hide` without parentheses | `Run(path, , "Hide")` | v1 command syntax is a parse error in v2 |
| `WinActivate` immediately after `Run` | `Run(...)` then `WinWaitActive(title,, timeout)` | Race condition — window may not exist yet; WinActivate on an absent window is silently ignored |
| `Sleep, 200` after clipboard copy | `A_Clipboard := "" ; Send("^c") ; ClipWait(timeout)` | Sleep duration is environment-dependent; ClipWait waits for the population event and confirms success via return value |

## API QUICK-REFERENCE

### Clipboard
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `A_Clipboard` | Built-in variable | Text-only read/write; clear to `""` before copying so ClipWait detects the new content, not leftover data |
| `ClipWait()` | `ClipWait(Timeout?, WaitForAnyData?)` | Returns 1 if clipboard populated within timeout, 0 on timeout — always check the return value |
| `ClipboardAll()` | `ClipboardAll(Data?, Size?)` | Captures all clipboard formats as a binary snapshot; pass no args to snapshot the current clipboard; restore by assigning to `A_Clipboard` |

### Process and Execution
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `Run()` | `Run(Target, WorkingDir?, Options?, &OutputVarPID?)` | Non-blocking; use `&OutputVarPID` to capture PID for subsequent `WinWait` |
| `RunWait()` | `RunWait(Target, WorkingDir?, Options?, &OutputVarPID?)` | Blocks until process exits; **return value is the exit code** — do not use ErrorLevel |
| `ProcessExist()` | `ProcessExist(PIDOrName?)` | Returns PID if running, 0 otherwise; call before Run to prevent duplicate instances |
| `ProcessWait()` | `ProcessWait(PIDOrName, Timeout?)` | Blocks until process appears; returns PID on success, 0 on timeout |
| `ProcessWaitClose()` | `ProcessWaitClose(PIDOrName, Timeout?)` | Blocks until process closes; returns 0 when closed, non-zero on timeout |
| `ProcessClose()` | `ProcessClose(PIDOrName)` | Forcibly terminates process; returns PID that was closed |
| `A_ComSpec` | Built-in variable | Full path to cmd.exe — use instead of hardcoded `"cmd.exe"` for portability |
| `A_IsAdmin` | Built-in variable | 1 if running as administrator, 0 otherwise; check before requiring elevated operations |

### Window
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `WinExist()` | `WinExist(WinTitle?, WinText?, ExcludeTitle?, ExcludeText?)` | Returns HWND if window found, 0 otherwise; use before `WinActivate` to avoid acting on absent windows |
| `WinActivate()` | `WinActivate(WinTitle?, WinText?, ExcludeTitle?, ExcludeText?)` | Always follow with `WinWaitActive()` to confirm focus transfer |
| `WinWait()` | `WinWait(WinTitle?, WinText?, Timeout?, ExcludeTitle?, ExcludeText?)` | Returns HWND on success, 0 on timeout — **always pass a timeout to prevent indefinite hang** |
| `WinWaitActive()` | `WinWaitActive(WinTitle?, WinText?, Timeout?, ExcludeTitle?, ExcludeText?)` | Returns HWND when window gains focus, 0 on timeout |
| `WinWaitClose()` | `WinWaitClose(WinTitle?, WinText?, Timeout?, ExcludeTitle?, ExcludeText?)` | Returns 1 when window closes; 0 on timeout |

### Registry
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `RegRead()` | `RegRead(KeyName, ValueName?)` | Returns value string; throws `OSError` on missing key or insufficient permissions — always wrap in try |
| `RegWrite()` | `RegWrite(Value, ValueType, KeyName, ValueName?)` | ValueType: `"REG_SZ"`, `"REG_DWORD"`, `"REG_BINARY"`, `"REG_EXPAND_SZ"`, `"REG_MULTI_SZ"` |
| `RegDelete()` | `RegDelete(KeyName, ValueName?)` | Deletes a single value; throws if value does not exist — use bare catch to suppress when absence is acceptable |
| `RegDeleteKey()` | `RegDeleteKey(KeyName)` | Deletes entire key and all sub-keys and values |

### COM Functions
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `ComObject()` | `ComObject(CLSID, IID?)` | Creates a new COM instance — replaces removed v1 `ComObjCreate()`; always wrap in try |
| `ComObjActive()` | `ComObjActive(CLSID)` | Attaches to an already-running COM instance — **never call `.Quit()` on the result** |
| `ComObjGet()` | `ComObjGet(Name)` | WMI moniker access (e.g., `"winmgmts:..."`); always wrap in try — WMI permissions vary by environment |
| `ComObjConnect()` | `ComObjConnect(ComObj, PrefixOrSink?)` | Pass a class instance as `PrefixOrSink` to sink events; omit the second argument to disconnect |
| `ComObjArray()` | `ComObjArray(VarType, Count1, Count2?)` | Creates a VARIANT SafeArray for bulk COM transfers; indexing is **0-based** — unlike all AHK v2 arrays |
| `ComValue()` | `ComValue(VarType, Value)` | Wraps an AHK value as a specific COM VARIANT type (e.g., VT_BSTR = 8, VT_VARIANT = 12) |

## AHK V2 CONSTRAINTS

- Use `A_Clipboard` exclusively — the v1 `Clipboard` variable does not exist in v2 and causes a NameError on access.
- Clear `A_Clipboard := ""` before triggering a copy operation, then call `ClipWait(timeout)` and check its return value before reading `A_Clipboard` — skipping the clear means ClipWait may detect leftover content and return immediately with stale data.
- Always pass a timeout to `WinWait`, `WinWaitActive`, `ProcessWait`, and `ProcessWaitClose` — omitting the timeout parameter causes an indefinite script hang when the target never appears.
- Wrap all `RegRead`, `RegWrite`, and `RegDelete` calls in try/catch — these functions throw `OSError` on missing keys or insufficient permissions, crashing the script if unhandled.
- Wrap **every** COM method call in try/catch — COM exceptions propagate as runtime errors that immediately crash the script; there is no "soft failure" for COM errors. This includes `ComObject()`/`ComObjActive()` creation, `.Workbooks.Open()`, `.Range()`, and every property access.
- COM teardown sequence is strict and non-negotiable: **(1)** call `.Quit()` or `.Close()` on the host application object, **(2)** then assign `""` to all references — reversing the order drops the AHK reference before the host application is told to exit, leaving a zombie process (e.g., a hidden `Excel.exe` that blocks file access and consumes memory).
- Never call `.Quit()` on a `ComObjActive()` instance — the script did not create that session and has no authority to terminate it; only release the reference with `ref := ""`.
- Use `ComObjArray()` for bulk COM data transfers instead of iterative writes — single-cell iteration crosses the COM boundary on every call (O(n) overhead); a single SafeArray assignment crosses once (O(1)). Note that `ComObjArray` indices are **0-based**, unlike all other AHK v2 collections.
- Retrieve process exit codes from `RunWait()`'s return value — `ErrorLevel` is deprecated in v2 and always returns 0, silently discarding the exit code.
- Use localization-independent numeric constants for Office COM (e.g., `-4135` for `xlCalculationManual`) rather than string names, which fail on non-English Office installations.

Safe-access priority order for COM and system operations:
  1. `ProcessExist()` / `WinExist()` before attaching — validates presence without exception overhead for predictable absent-process cases
  2. `try { ref := ComObject(...) } catch as err` — COM creation failure is exceptional; the catch block receives the full error context for logging
  3. `try { comRef.Method() } catch as err` — wrap every individual COM method call; COM exceptions are non-resumable and carry diagnostic detail
  4. `finally { if ref { ref.Quit() } ; ref := "" }` — teardown always runs even on exception paths, preventing zombie processes

## TIER 1 — Clipboard and Application Launch Fundamentals
> METHODS COVERED: A_Clipboard · ClipWait · ClipboardAll · Run · WinWaitActive · Send

Tier 1 covers the two most common OS integration primitives: clipboard-mediated text transfer and launching external applications. Both require explicit sequencing — `A_Clipboard` must be cleared before copying so `ClipWait` detects new content, and a launched application must be confirmed active via `WinWaitActive` before any input is sent.
```ahk
; ✓ Clear first so ClipWait detects the NEW copy, not leftover clipboard content
A_Clipboard := ""
Send("^c")
if ClipWait(2) {                            ; ✓ 2-second timeout; return value 1 = success, 0 = timed out
    modifiedText := StrUpper(A_Clipboard)
    A_Clipboard := modifiedText
    Send("^v")
} else {
    MsgBox("Clipboard operation timed out.")
}

; ✗ v1 variable — NameError in AHK v2
; text := Clipboard                         ; → NameError: 'Clipboard' is undefined

; ✓ ClipboardAll snapshots all formats (rich text, images) for save-and-restore workflows
savedClip := ClipboardAll()                 ; capture all clipboard formats
; ... do work that overwrites A_Clipboard ...
A_Clipboard := savedClip                    ; restore original content
savedClip := ""                             ; release snapshot memory

; ✓ WinWaitActive with timeout prevents race condition between Run and Send
targetApp := '"C:\Windows\notepad.exe"'
Run(targetApp)
if WinWaitActive("ahk_exe notepad.exe",, 5) {   ; ✓ 5-second timeout guards against launch failure
    Send("Application loaded successfully.")
}

; ✗ Immediate WinActivate — window may not exist yet when this line executes
; Run(targetApp)
; WinActivate("ahk_exe notepad.exe")        ; → silently ignored if window not yet open
```

## TIER 2 — Registry Operations and Process Exit Codes
> METHODS COVERED: RegWrite · RegDelete · RegRead · RunWait · A_ComSpec · A_ScriptFullPath

Tier 2 covers OS mutation operations: modifying the Windows Registry and capturing the exit codes of spawned processes. Registry operations require try/catch because key absence and permission failures are common across different user environments. Exit codes must be captured from `RunWait()`'s return value — not from the deprecated `ErrorLevel`.
```ahk
; ✓ Registry operations wrapped in try-catch — permissions vary by OS configuration
ManageStartupReg(enable) {
    keyName := "HKCU\Software\Microsoft\Windows\CurrentVersion\Run"
    valName := "MyAHKScript"

    if (enable) {
        try {
            RegWrite('"' . A_ScriptFullPath . '"', "REG_SZ", keyName, valName)
        } catch as err {
            MsgBox("Registry write failed: " . err.Message)
        }
    } else {
        try {
            RegDelete(keyName, valName)
        } catch {
            ; ✓ Suppress — key may not exist; that is the intended end state
        }
    }
}

; ✓ RunWait returns exit code directly — ErrorLevel is deprecated and returns 0 in v2
ExecuteCommand(cmd) {
    ; ✗ v1 command syntax — parse error in v2
    ; RunWait, %cmd%, , Hide

    ; ✓ Functional syntax; A_ComSpec resolves to cmd.exe path portably
    exitCode := RunWait(A_ComSpec . ' /c "' . cmd . '"',, "Hide")
    return exitCode
}

; ✓ Reading a registry value with fallback via catch
GetRegistryValue(keyName, valueName, default := "") {
    try {
        return RegRead(keyName, valueName)
    } catch {
        return default              ; ✓ return caller-supplied default on missing key
    }
}
```

## TIER 3 — Process and Window State Validation
> METHODS COVERED: ProcessExist · ProcessWait · ProcessClose · WinExist · WinActivate · WinWait · WinWaitActive

Tier 3 covers querying and validating system states before acting on them. `ProcessExist()` guards against duplicate launches, and all Wait functions require explicit timeout parameters to prevent indefinite script hangs when a target never appears. `WinExist()` should precede `WinActivate()` to avoid acting on absent windows.
```ahk
; ✓ ProcessExist check prevents duplicate launches; WinWait with timeout prevents hang
EnsureProcessRunning(exeName) {
    pid := ProcessExist(exeName)

    if (!pid) {
        Run(exeName, , , &newPid)           ; ✓ capture PID via ByRef output parameter
        ; ✓ 10-second timeout — fails fast instead of hanging on headless or crashed process
        if (!WinWait("ahk_pid " . newPid,, 10)) {
            throw Error("Process window failed to appear within timeout.")
        }
        pid := newPid
    }

    return pid
}

; ✓ WinExist before WinActivate avoids acting on a window that may have closed
ValidateWindowState(winTitle) {
    if (WinExist(winTitle)) {
        WinActivate(winTitle)
        ; ✓ 3-second timeout prevents indefinite hang if focus transfer fails
        if (WinWaitActive(winTitle,, 3)) {
            return true
        }
    }
    return false
}

; ✗ WinWait without timeout — hangs forever if process is headless or has already crashed
; WinWait("ahk_pid " . pid)               ; → indefinite hang with no recovery path
```

## TIER 4 — WMI Data Queries and Map Transformation
> METHODS COVERED: ComObjGet · .ExecQuery · Map · Map.Has

Tier 4 covers system data extraction via Windows Management Instrumentation (WMI). WMI is accessed through the COM layer using `ComObjGet()` with a moniker string. The returned COM collection is enumerable with `for-in`, and results should be transformed into an AHK v2 `Map` for safe downstream access. WMI permissions vary by OS configuration — always wrap in try/catch.
```ahk
; ✓ Full WMI connection in try-catch — impersonationLevel required for service/hardware queries
GetRunningServices() {
    servicesMap := Map()
    try {
        ; ✓ WMI moniker via ComObjGet — impersonationLevel required for Win32_Service queries
        wmi := ComObjGet("winmgmts:{impersonationLevel=impersonate}!\\.\root\cimv2")

        ; ✓ WQL SELECT limits columns — SELECT * is expensive on large WMI collections
        query := wmi.ExecQuery("SELECT Name, DisplayName FROM Win32_Service WHERE State='Running'")

        ; ✓ for-in enumerates COM collections — AHK's COM layer handles the enumeration protocol
        for service in query {
            servicesMap[service.Name] := service.DisplayName
        }
    } catch as err {
        MsgBox("WMI Query Failed: " . err.Message)
    }

    return servicesMap
}

; ✓ Map.Has() for safe key lookup after WMI population — never index without checking
running := GetRunningServices()
if (running.Has("Themes")) {
    ; Themes service confirmed running
}

; ✗ Direct map index without .Has() — UnsetItemError if key absent
; name := running["Spooler"]              ; → UnsetItemError if Spooler not in result
```

## TIER 5 — COM Application Automation and Lifecycle Management
> METHODS COVERED: ComObject · ComObjActive · .Workbooks.Add · .ActiveSheet · .Cells · .Range · .Value · .Quit · .Close · .AutoFit · .Font.Bold · .Visible · .Columns · ExportArrayToExcel

Tier 5 covers the two foundational COM lifecycle patterns and direct Excel automation. The teardown sequence is the single most critical correctness concern in this domain — the order of operations is non-negotiable. Pattern A applies when the script creates the COM instance (`ComObject`); Pattern B applies when the script attaches to an existing instance (`ComObjActive`). These patterns differ in one critical way: Pattern A calls `.Quit()`, Pattern B never does.
```ahk
; =============================================================================
; Pattern A: Script-owned instance (ComObject)
; Sequence: call .Quit()/.Close() FIRST, then assign "" — reversing the order
; drops the AHK reference before the host application has been told to exit,
; leaving a zombie process (e.g., a hidden Excel.exe in Task Manager that
; blocks file access and persists until manual termination).
; =============================================================================

OwnedComLifecycle() {
    xlApp := ""
    wb    := ""
    try {
        xlApp := ComObject("Excel.Application")     ; ✓ v2: ComObject() — ComObjCreate() is removed
        xlApp.Visible := false
        wb := xlApp.Workbooks.Add()

        ; ✓ Every COM method call wrapped — COM errors are non-resumable runtime exceptions
        try {
            wb.ActiveSheet.Range("A1").Value := "Hello"
        } catch as comErr {
            MsgBox("COM method failed: " . comErr.Message)
        }

        try wb.Close(0)                  ; Step 1: close the workbook (0 = do not save)
    } catch as initErr {
        MsgBox("COM init failed: " . initErr.Message)
    } finally {
        if (xlApp) {
            try xlApp.Quit()             ; Step 2: instruct Excel to exit — MUST precede reference drop
        }
        wb    := ""                      ; Step 3: drop child references to release COM ref-count
        xlApp := ""                      ; Step 4: drop root reference last
    }
}

; =============================================================================
; Pattern B: Attached instance (ComObjActive)
; The script did NOT create this session — it must NOT call .Quit().
; Only perform work, then release the reference. The user's session must not
; be terminated by a script that merely attached to it.
; =============================================================================

AttachedComLifecycle() {
    xlApp := ""
    try {
        xlApp := ComObjActive("Excel.Application")  ; attach to an already-open Excel

        ; ✓ Wrap every COM call — exceptions still crash immediately if unhandled
        try {
            xlApp.ActiveSheet.Range("A1").Value := "Updated"
        } catch as comErr {
            MsgBox("COM call failed: " . comErr.Message)
        }

        ; ✗ Wrong: xlApp.Quit() — terminates the user's running Excel session
    } catch as attachErr {
        MsgBox("No running Excel instance found: " . attachErr.Message)
    } finally {
        xlApp := ""                      ; ✓ Release reference only — do NOT call Quit()
    }
}

; =============================================================================
; Full automation example: export an AHK v2 Array to Excel column A
; =============================================================================

ExportArrayToExcel(dataArray) {
    if (!IsObject(dataArray) || Type(dataArray) != "Array") {
        throw Error("Input must be an array.")
    }

    xlApp := ""
    wb    := ""
    sheet := ""

    try {
        xlApp := ComObject("Excel.Application")
        xlApp.Visible := true

        try {
            wb    := xlApp.Workbooks.Add()
            sheet := wb.ActiveSheet
        } catch as initErr {
            throw Error("Failed to create workbook: " . initErr.Message)
        }

        ; ✗ v1 loop syntax — parse error in v2
        ; Loop % dataArray.Length()

        ; ✓ AHK v2 for-loop provides index and value in one expression
        for index, value in dataArray {
            try {
                ; ✓ Excel Cells() uses 1-based row/column — matches AHK v2 array index
                sheet.Cells(index, 1).Value := value
            } catch as cellErr {
                MsgBox("Cell write failed at row " . index . ": " . cellErr.Message)
                break
            }
        }

        ; ✓ Each formatting call wrapped independently — any can throw without the others failing
        try sheet.Columns("A:A").EntireColumn.AutoFit()
        try sheet.Range("A1").Font.Bold := true

    } catch as comErr {
        MsgBox("COM operation failed: " . comErr.Message)
    } finally {
        ; ✓ Correct teardown sequence: application exit BEFORE dropping references
        ; ✗ Wrong: xlApp := "" before xlApp.Quit() — leaves Excel.exe as a zombie process
        if (xlApp) {
            try xlApp.Quit()             ; Step 1: instruct Excel to exit
        }
        sheet := ""                      ; Step 2: drop child references first
        wb    := ""
        xlApp := ""                      ; Step 3: drop root reference last
    }
}
```

### Performance Notes

COM interaction is fundamentally cross-process — every property read and method call crosses a process boundary and incurs marshalling overhead. For large datasets, the cost difference between iterative and bulk approaches is not constant-factor; it is architectural.

**Iterative cell writes (O(n) COM boundary crossings):** Each `sheet.Cells(i, 1).Value := val` crosses the COM boundary once per row. For 1,000 rows this is 1,000 boundary crossings; for 10,000 rows it is 10,000. This is the correct approach only for small datasets where error-per-row handling is required.

**Bulk SafeArray write (O(1) COM boundary crossings):** Build a `ComObjArray` in AHK memory and assign the entire array to a `Range.Value` in a single call. The COM boundary is crossed once regardless of dataset size. Note that `ComObjArray` indices are **0-based** — unlike all other AHK v2 collections.

**ScreenUpdating and Calculation:** Disabling these during bulk updates prevents Excel from recalculating and re-rendering after every cell write. Always restore both before returning — leaving them disabled makes the workbook appear frozen to the user.
```ahk
; ✓ Bulk SafeArray write — single COM boundary crossing for entire dataset
WriteToExcelFast(dataMap) {
    xlApp := ComObject("Excel.Application")
    ; ✓ Disable ScreenUpdating and Calculation for bulk-write performance
    xlApp.ScreenUpdating := false
    xlApp.Calculation := -4135           ; xlCalculationManual (locale-independent numeric constant)

    wb    := xlApp.Workbooks.Add()
    sheet := wb.ActiveSheet

    ; ✓ ComObjArray(12, rows, cols) — 12 = VT_VARIANT; note 0-based indexing
    safeArray := ComObjArray(12, dataMap.Count, 2)

    rowIndex := 0
    for key, val in dataMap {
        safeArray[rowIndex, 0] := key    ; ✓ ComObjArray is 0-based — unlike AHK v2 Arrays
        safeArray[rowIndex, 1] := val
        rowIndex++
    }

    ; ✓ Single COM boundary crossing writes the entire array
    endCell := "B" . dataMap.Count
    sheet.Range("A1:" . endCell).Value := safeArray

    ; ✓ Always restore application state before returning — omission leaves Excel frozen
    xlApp.Calculation := -4105           ; xlCalculationAutomatic
    xlApp.ScreenUpdating := true
    xlApp.Visible := true
}
```

## TIER 6 — COM Event Sinking and Production Wrapper Classes
> METHODS COVERED: ComObjConnect · ComObjArray · .WorkbookBeforeClose · .SheetSelectionChange · .Workbooks.Open · .Sheets · ToolTip · SetTimer

Tier 6 integrates event-driven COM automation and production-grade wrapper classes. `ComObjConnect()` binds a script class instance as an event sink — AHK routes incoming COM events to class methods whose names match the COM event names exactly. The `ExcelAutomation` wrapper class demonstrates the full lifecycle pattern as a reusable component with `__New` and `__Delete` meta-functions managing resource acquisition and teardown.
```ahk
; =============================================================================
; ExcelEventListener — COM event sinking via ComObjConnect
; AHK routes events from xlApp to methods on this class instance by name match
; =============================================================================

class ExcelEventListener {
    xlApp       := ""
    boundEvents := false

    __New() {
        this.xlApp := ComObject("Excel.Application")
        this.xlApp.Visible := true
        this.xlApp.Workbooks.Add()

        ; ✓ Pass 'this' as event sink — AHK routes COM events to matching method names
        ComObjConnect(this.xlApp, this)
        this.boundEvents := true
    }

    ; ✓ Method name must exactly match the COM event name — AHK does not tolerate case mismatch
    WorkbookBeforeClose(wb, &cancel) {
        result := MsgBox("Are you sure you want to close " . wb.Name . "?", "Confirm", "YesNo")
        if (result == "No") {
            ; ✓ COM ByRef event parameters are assigned directly — no special wrapper needed
            cancel := true
        }
    }

    SheetSelectionChange(sheet, target) {
        ToolTip("Selected: " . target.Address)
        SetTimer(() => ToolTip(), -2000)    ; ✓ negative interval = one-shot timer; clears tooltip after 2 s
    }

    __Delete() {
        if (this.boundEvents && this.xlApp) {
            ; ✓ Disconnect events before releasing — prevents callbacks into a destroyed object
            ComObjConnect(this.xlApp)       ; omit second arg to disconnect all events
        }
        if (this.xlApp) {
            try this.xlApp.Quit()
        }
        this.xlApp := ""
    }
}

; =============================================================================
; ExcelAutomation — production wrapper with full lifecycle encapsulation
; Encapsulates ComObject creation, workbook management, and guaranteed teardown
; in __New/__Delete meta-functions so callers cannot accidentally leak resources
; =============================================================================

class ExcelAutomation {
    xlApp      := ""
    xlWorkbook := ""

    __New(visible := true) {
        try {
            this.xlApp := ComObject("Excel.Application")
            this.xlApp.Visible := visible
        } catch as err {
            throw Error("Failed to initialize Excel COM object: " . err.Message)
        }
    }

    OpenWorkbook(filePath) {
        if (!FileExist(filePath)) {
            throw Error("Workbook path does not exist: " . filePath)
        }
        ; ✓ Workbooks.Open wrapped separately — locked or corrupt files throw here, not in __New
        try {
            this.xlWorkbook := this.xlApp.Workbooks.Open(filePath)
        } catch as openErr {
            throw Error("Failed to open workbook: " . openErr.Message)
        }
        return this.xlWorkbook
    }

    ReadCell(sheetName, cellRange) {
        if (!this.xlWorkbook) {
            throw Error("No workbook is currently open.")
        }
        try {
            sheet := this.xlWorkbook.Sheets(sheetName)
            return sheet.Range(cellRange).Value
        } catch {
            return ""               ; ✓ Return empty string on COM failure — caller decides how to handle
        }
    }

    __Delete() {
        if (this.xlWorkbook) {
            try this.xlWorkbook.Close(0)   ; 0 = do not save changes
        }
        if (this.xlApp) {
            try this.xlApp.Quit()
        }
        ; ✓ Explicitly clear COM references — releases ref-count, prevents zombie Excel.exe
        this.xlWorkbook := ""
        this.xlApp      := ""
    }
}
```

## ANTI-PATTERNS

| Pattern | Wrong | Correct | LLM Common Cause |
|---------|-------|---------|------------------|
| v1 clipboard variable | `text := Clipboard` | `text := A_Clipboard` | AHK v1 training data; `Clipboard` was a valid built-in variable |
| Clipboard race condition | `Send("^c") ; Sleep 200` | `A_Clipboard := "" ; Send("^c") ; ClipWait(2)` | Sleep-based polling is the default pattern in most language training data; ClipWait is AHK-specific |
| Immediate WinActivate after Run | `Run(path) ; WinActivate(title)` | `Run(path) ; WinWaitActive(title,, 5)` | v1 scripts often omit the wait; cross-language models do not know about window appearance latency |
| Wrong COM teardown order | `xlApp := "" ; xlApp.Quit()` | `xlApp.Quit() ; xlApp := ""` | Reference-before-teardown is harmless in GC languages; v2 is reference-counted and order-sensitive |
| Quit on attached instance | `xlApp := ComObjActive(...) ; xlApp.Quit()` | `xlApp := ComObjActive(...) ; xlApp := ""` | Models conflate owned and attached COM lifecycles; the distinction is AHK-specific |
| Unguarded COM method call | `sheet.Range("A1").Value := x` | `try { sheet.Range("A1").Value := x } catch as e { ... }` | COM error handling is unusual compared to most language training data; models omit try by default |
| v1 ComObjCreate | `ComObjCreate("Excel.Application")` | `ComObject("Excel.Application")` | v1 function name dominates mixed-version AHK training data; `ComObjCreate` is removed in v2 |
| v1 RegRead command syntax | `RegRead, val, HKCU\..., Name` | `val := RegRead("HKCU\...", "Name")` | v1 command syntax is the dominant pattern in AHK training data |
| ErrorLevel for exit code | `RunWait(cmd) ; code := ErrorLevel` | `code := RunWait(cmd)` | ErrorLevel was the v1 convention; models do not know it is deprecated and always 0 in v2 |
| Iterative COM writes for bulk data | `for i, v in arr { sheet.Cells(i,1).Value := v }` | Build `ComObjArray` then `sheet.Range(...).Value := safeArray` | Models default to iterative patterns without knowing the O(n) COM boundary cost |

## SEE ALSO

> This module does NOT cover: IStream COM interface, named-pipe IPC, Win32 memory-level OS access, or DllCall — see Module_DllCallAndMemory.md
> This module does NOT cover: advanced window and control interaction beyond WinActivate/WinWait (SendMessage, PostMessage, ControlClick) — see Module_WindowAndControl.md
> This module does NOT cover: try/catch error class hierarchies and custom OSError handling patterns — see Module_Errors.md
> This module does NOT cover: FileRead, FileOpen, and file-system path operations — see Module_FileSystem.md
> This module does NOT cover: wrapper class design, prototype patterns, and __New/__Delete meta-function conventions — see Module_Classes.md

- `Module_DllCallAndMemory.md` — Win32 API calls via DllCall, Buffer/memory management, IStream COM interface, and low-level OS access.
- `Module_WindowAndControl.md` — SendMessage, PostMessage, ControlClick, and fine-grained control manipulation beyond basic window activation.
- `Module_Errors.md` — try/catch patterns, custom Error class hierarchies, and OSError handling for registry and COM failures.
- `Module_FileSystem.md` — FileRead, FileOpen, FileAppend, SplitPath, and path validation for file-based operations that complement system integration.
- `Module_Classes.md` — wrapper class design, `__New`/`__Delete` meta-function conventions, and prototype patterns underlying the `ExcelAutomation` class above.