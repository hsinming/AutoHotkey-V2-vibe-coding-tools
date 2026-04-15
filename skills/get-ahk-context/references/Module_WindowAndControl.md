# Module_WindowAndControl.md
<!-- DOMAIN: Window and Control Interaction -->
<!-- SCOPE: Pixel/image searching, async I/O, and IStream COM streaming are not covered — see Module_GraphicsAndScreen.md and Module_SystemAndCOM.md -->
<!-- TRIGGERS: WinExist, WinActivate, WinClose, WinWait, WinGetPos, WinGetList, WinMove, WinHide, WinShow, ControlClick, ControlSend, ControlGetText, ControlGetItems, ControlChooseIndex, ControlGetHwnd, GroupAdd, GroupActivate, SetWinEventHook, CallbackCreate, CoordMode, ahk_id, ahk_class, ahk_exe, HWND, "window handle", "check if window is open", "click button in background", "detect when window opens", "send keys to background window", "batch minimize windows", "passive window monitoring", "window group cycling" -->
<!-- CONSTRAINTS: HWND is Integer — never compare or store as string; ControlChooseIndex/ControlGetItems are 1-based (index 0 invalid); always declare CoordMode before any mouse-coordinate-sensitive call (MouseMove, MouseClick, Click, MouseGetPos); guard every Win*/Control* call with WinExist() pre-check or try/catch TargetError; SetWinEventHook callbacks require ObjBindMethod + CallbackFree in __Delete(). -->
<!-- CROSS-REF: Module_Errors.md, Module_Classes.md, Module_SystemAndCOM.md, Module_GraphicsAndScreen.md -->
<!-- VERSION: AHK v2.0+ -->

## V1 → V2 BREAKING CHANGES

| v1 pattern (LLM commonly writes) | v2 correct form | Consequence |
|----------------------------------|-----------------|-------------|
| `WinActivate, ahk_id %hwnd%` (bare command + `%var%` interpolation) | `WinActivate("ahk_id " . hwnd)` | Parse error in AHK v2 — bare command syntax and `%var%` interpolation are both illegal |
| `if hwnd = "0"` (string comparison against HWND) | `if !hwnd` or `if (hwnd = 0)` | Silent mismatch on 64-bit handles — Integer HWND never equals the string `"0"` |
| `ControlClick, Button1, WinTitle` (bare command style) | `ControlClick("Button1", winTitle)` | Parse error — all built-in functions require parentheses in AHK v2 |
| `ControlChooseIndex("CB1", 0, winTitle)` (0-based index) | `ControlChooseIndex(1, "CB1", winTitle)` | Index 0 is invalid — ControlChooseIndex and ControlGetItems Arrays are 1-based |
| `CallbackCreate(this._Dispatch, "Fast", 7)` (unbound method ref) | `CallbackCreate(ObjBindMethod(this, "_Dispatch"), "Fast", 7)` | Wrong `this` context in callback — method receives garbage object reference or crashes |
| `WinExist("Title")` used as bare boolean without capturing return | `hwnd := WinExist("Title")` then `if hwnd` | HWND loss — the Integer return value carries the window handle needed for all follow-on ops |
| Missing TargetError handler on Win*/Control* calls | Wrap in `try { … } catch TargetError as e { … }` | Unhandled throw terminates the script thread — windows are ephemeral and can disappear between AHK statements |

## API QUICK-REFERENCE

### Window State Functions

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `WinExist()` | `WinExist(WinTitle?, WinText?, ExcludeTitle?, ExcludeText?)` | Integer HWND or 0 | — | Dual-purpose: existence check + handle capture in one call |
| `WinActive()` | `WinActive(WinTitle?, ...)` | Integer HWND or 0 | — | Returns HWND if matched window is the current foreground; 0 otherwise |
| `WinActivate()` | `WinActivate(WinTitle?)` | — | TargetError | Brings window to foreground; throws if not found |
| `WinClose()` | `WinClose(WinTitle?, WinText?, SecondsToWait?)` | — | TargetError | Sends close signal; throws if not found |
| `WinWait()` | `WinWait(WinTitle?, WinText?, Timeout?)` | Integer HWND or 0 | — | Blocks until window appears; 0 on timeout |
| `WinWaitActive()` | `WinWaitActive(WinTitle?, WinText?, Timeout?)` | Integer HWND or 0 | — | Blocks until window is active; 0 on timeout |
| `WinWaitClose()` | `WinWaitClose(WinTitle?, WinText?, Timeout?)` | 1 (closed) or 0 (timeout) | — | Returns immediately with 1 if no matching window exists; never throws TargetError |
| `WinGetMinMax()` | `WinGetMinMax(WinTitle?)` | Integer: -1/0/1 | TargetError | -1 = minimized, 0 = normal, 1 = maximized |
| `WinMinimize()` | `WinMinimize(WinTitle?)` | — | TargetError | Minimizes the target window |
| `WinMaximize()` | `WinMaximize(WinTitle?)` | — | TargetError | Maximizes the target window |
| `WinRestore()` | `WinRestore(WinTitle?)` | — | TargetError | Restores a minimized or maximized window to normal state |
| `WinHide()` | `WinHide(WinTitle?)` | — | TargetError | Hides the window from taskbar and screen without closing |
| `WinShow()` | `WinShow(WinTitle?)` | — | TargetError | Makes a hidden window visible again |

### Window Metadata Functions

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `WinGetTitle()` | `WinGetTitle(WinTitle?)` | String | TargetError | Returns window title string |
| `WinGetClass()` | `WinGetClass(WinTitle?)` | String | TargetError | Returns class name; stable across app versions and locales |
| `WinGetProcessName()` | `WinGetProcessName(WinTitle?)` | String | TargetError | Returns executable filename (e.g., `"notepad.exe"`) |
| `WinGetPID()` | `WinGetPID(WinTitle?)` | Integer PID | TargetError | Returns Integer process ID |
| `WinGetID()` | `WinGetID(WinTitle?)` | Integer HWND | TargetError | Returns HWND of first matching window; prefer `WinExist()` when existence check is also needed |
| `WinGetPos()` | `WinGetPos(&x, &y, &w, &h, WinTitle?)` | — (fills refs) | TargetError | Fills output variables by reference; throws if window absent |
| `WinGetList()` | `WinGetList(WinTitle?)` | 1-based Array of Integer HWNDs | — | Returns Array for all matching windows; empty Array if none found |
| `WinGetStyle()` | `WinGetStyle(WinTitle?)` | Integer bitmask | TargetError | Returns window style bits; combine with `&` to test individual flags |
| `WinGetExStyle()` | `WinGetExStyle(WinTitle?)` | Integer bitmask | TargetError | Returns extended style bits (WS_EX_* constants) |

### Window Mutation Functions

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `WinMove()` | `WinMove(x, y, w?, h?, WinTitle?)` | — | TargetError | Repositions and/or resizes; omit w/h to move without resizing |
| `WinSetAlwaysOnTop()` | `WinSetAlwaysOnTop(value, WinTitle?)` | — | TargetError | `1` = pin on top, `0` = clear, `-1` = toggle |
| `WinSetTransparent()` | `WinSetTransparent(N\|"Off", WinTitle?)` | — | TargetError | 0–255 alpha; use `"Off"` (not 255) to fully restore opacity |
| `WinSetTitle()` | `WinSetTitle(newTitle, WinTitle?)` | — | TargetError | Changes the window's visible title string |
| `WinRedraw()` | `WinRedraw(WinTitle?)` | — | TargetError | Forces a repaint; use after layered or transparency changes |

### Title Matching

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `SetTitleMatchMode()` | `SetTitleMatchMode(mode)` | — | — | `1` = starts-with, `2` = contains (default), `3` = exact, `"RegEx"` = regex; global flag — set once at startup |

### Window Groups

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `GroupAdd()` | `GroupAdd(groupName, WinTitle?)` | — | — | Adds a title criterion to a named group; re-evaluated live at each activation |
| `GroupActivate()` | `GroupActivate(groupName, mode?)` | — | — | Activates the next window in the group; `"R"` reverses direction |
| `GroupDeactivate()` | `GroupDeactivate(groupName, mode?)` | — | — | Activates the next window NOT in the group |
| `GroupClose()` | `GroupClose(groupName, mode?)` | — | — | Closes the active window if it is a group member |

### Coordinate Mode

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `CoordMode()` | `CoordMode(targetType, relativeTo?)` | — | — | Sets coordinate origin for mouse/pixel functions; does NOT affect ControlClick or ControlGetPos |

### Mouse Input Functions

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `MouseClick()` | `MouseClick(button?, x?, y?, count?, speed?, d?, r?)` | — | — | Requires CoordMode declaration for screen-absolute coordinates |
| `MouseMove()` | `MouseMove(x, y, speed?, r?)` | — | — | Requires CoordMode; r = relative movement from current position |
| `Click()` | `Click(options?)` | — | — | Shorthand for MouseClick; parses options string; also needs CoordMode |
| `MouseGetPos()` | `MouseGetPos(&outX?, &outY?, &outWin?, &outCtrl?, flag?)` | — (fills refs) | — | Captures cursor position; coordinates obey current CoordMode |

### Control Interaction

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `ControlClick()` | `ControlClick(ctrl?, WinTitle?, WinText?, button?, count?, options?)` | — | TargetError | Clicks without activating window; `"NA"` option suppresses activation |
| `ControlSend()` | `ControlSend(keys, ctrl?, WinTitle?)` | — | TargetError | Sends keystrokes with special-key interpretation; window need not be active |
| `ControlSendText()` | `ControlSendText(text, ctrl?, WinTitle?)` | — | TargetError | Sends literal text verbatim; no special-key interpretation; safe for passwords |
| `ControlGetPos()` | `ControlGetPos(&x, &y, &w, &h, ctrl?, WinTitle?)` | — (fills refs) | TargetError, OSError | Coordinates always relative to target window's client area; CoordMode has no effect |
| `ControlGetHwnd()` | `ControlGetHwnd(ctrl?, WinTitle?)` | Integer HWND | TargetError | Returns HWND of the specified child control |
| `ControlGetFocus()` | `ControlGetFocus(WinTitle?)` | Integer HWND or 0 | TargetError, OSError | Returns HWND of currently focused control; returns 0 if no control has focus |
| `ControlFocus()` | `ControlFocus(ctrl?, WinTitle?)` | — | TargetError | Moves keyboard focus to the specified control |

### Control Data

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `ControlGetText()` | `ControlGetText(ctrl?, WinTitle?)` | String | TargetError | Reads text content from Edit or Static control |
| `ControlSetText()` | `ControlSetText(newText, ctrl?, WinTitle?)` | — | TargetError | Writes content directly; does not trigger onChange events |
| `ControlChooseIndex()` | `ControlChooseIndex(n, ctrl?, WinTitle?)` | — | TargetError | Selects item at 1-based index in ListBox/ComboBox; 0 is invalid |
| `ControlChooseString()` | `ControlChooseString(str, ctrl?, WinTitle?)` | — | TargetError | Selects first item containing `str` (partial match) |
| `ControlGetIndex()` | `ControlGetIndex(ctrl?, WinTitle?)` | Integer (1-based; 0 = none selected) | TargetError | Returns selected index; 0 means nothing is selected |
| `ControlGetItems()` | `ControlGetItems(ctrl?, WinTitle?)` | 1-based Array of Strings | TargetError | Returns all item strings in ListBox/ComboBox |
| `ControlGetChecked()` | `ControlGetChecked(ctrl?, WinTitle?)` | 1 (checked) or 0 (unchecked) | TargetError | For Button/Checkbox controls |
| `ControlSetChecked()` | `ControlSetChecked(value, ctrl?, WinTitle?)` | — | TargetError | Sets checkbox state: `1` = check, `0` = uncheck, `-1` = toggle |
| `ControlAddItem()` | `ControlAddItem(str, ctrl?, WinTitle?)` | — | TargetError | Appends a new item to a ListBox or ComboBox |
| `ControlDeleteItem()` | `ControlDeleteItem(n, ctrl?, WinTitle?)` | — | TargetError | Removes item at 1-based index from a ListBox or ComboBox |
| `ControlFindItem()` | `ControlFindItem(str, ctrl?, WinTitle?)` | Integer (1-based) or 0 | TargetError | Returns 1-based index of first item matching `str` exactly; 0 if not found |

### Hook and Callback (DllCall)

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `DllCall("SetWinEventHook")` | `DllCall("SetWinEventHook", "UInt", eventMin, "UInt", eventMax, "Ptr", 0, "Ptr", callback, "UInt", pid, "UInt", tid, "UInt", flags, "Ptr")` | Ptr (hook handle) or 0 | — | Returns 0 on failure; must be freed with UnhookWinEvent |
| `DllCall("UnhookWinEvent")` | `DllCall("UnhookWinEvent", "Ptr", hHook)` | — | — | Releases the hook handle; always call before CallbackFree |
| `CallbackCreate()` | `CallbackCreate(func, options?, paramCount?)` | Integer (machine-code address) | — | Converts an AHK Func/BoundFunc to a machine-code address for DllCall |
| `CallbackFree()` | `CallbackFree(address)` | — | — | Releases the machine-code address; always call after UnhookWinEvent |
| `ObjBindMethod()` | `ObjBindMethod(obj, method, args*)` | BoundFunc | — | Binds `this` to a method reference; required before passing a method to CallbackCreate |

## AHK V2 CONSTRAINTS

- HWND values returned by `WinExist()`, `WinGetID()`, `WinGetList()`, and `ControlGetHwnd()` are Integer — never compare with string literals (`= "0"`), never interpolate with `%hwnd%`; use `if !hwnd` or `if (hwnd = 0)` for falsy checks — string comparison silently fails on 64-bit handle values.
- CoordMode must be explicitly declared at the top of every function or block that calls a mouse-coordinate-sensitive function (`MouseMove`, `MouseClick`, `Click`, `MouseGetPos`) — AHK v2 defaults to the active window's client area; `ControlClick` and `ControlGetPos` are not affected by CoordMode and always use the target window's client area.
- Guard every `Win*` and `Control*` call with a `WinExist()` pre-check or `try/catch TargetError` — windows are ephemeral OS objects that can close between any two consecutive AHK v2 statements; v2 throws `TargetError` rather than returning silently — an unguarded call will crash the script thread.
- `ControlChooseIndex()` and array indexing into `ControlGetItems()` results are strictly 1-based — index 1 selects the first item; passing 0 is invalid and will produce no selection or an unexpected result — this is a frequent off-by-one error for developers from zero-indexed languages.
- `SetTitleMatchMode()` is a global flag — set it once at script startup, document its value, and never change it silently inside utility functions; changing it mid-script invalidates all title-matching assumptions made by other functions without any runtime warning.
- `GroupAdd()` defines criteria, not a snapshot — the group is re-evaluated live each time `GroupActivate()` or `WinGetList("ahk_group …")` runs; windows that open or close after `GroupAdd()` is called are automatically included or excluded.
- `SetWinEventHook` callbacks require `ObjBindMethod(this, "MethodName")` — not a bare `this.Method` reference — to preserve the correct `this` context; `CallbackCreate` must specify the exact WINEVENTPROC parameter count (`7`); both the hook handle and the callback address must be freed in `__Delete()` via `UnhookWinEvent` + `CallbackFree` — omitting either causes a handle or memory leak that persists until OS restart.

Safe-access priority order for window operations:
  1. `hwnd := WinExist(winTitle)` then `if hwnd` — single call captures HWND and confirms existence; use for all simple guards
  2. `WinWait(title,, timeout)` — when the window may not yet exist and the script should block; returns 0 on timeout
  3. `try { Win*(…) } catch TargetError` — when the window is expected to exist but race conditions are possible, or when the error message carries diagnostic information
  4. `SetWinEventHook` — only when the task is event-driven ("react when window X appears") and polling is explicitly unacceptable; zero CPU at idle, sub-millisecond latency

- ✗ `WinActivate, ahk_id %hwnd%` — parse error in AHK v2
- ✓ `WinActivate("ahk_id " . hwnd)` — correct function-call syntax with string concatenation
- ✗ `if hwnd = "0"` — Integer ≠ string; always false on 64-bit systems
- ✓ `if !hwnd` — correct falsy test for Integer 0
- ✗ `MouseClick("Left", x, y)` with no prior CoordMode — uses active window's client area by default; screen-absolute coordinates silently misfire
- ✓ `CoordMode("Mouse", "Screen")` declared first for absolute screen coordinates, then `MouseClick("Left", x, y)`
- ✗ `ControlChooseIndex(0, "CB1", title)` — 0 is not a valid index
- ✓ `ControlChooseIndex(1, "CB1", title)` — first item is index 1

## AGENT QA CHECKLIST

- [ ] Did I use `if !hwnd` or `if (hwnd = 0)` instead of `if hwnd = "0"` for every HWND falsy check?
- [ ] Did I declare `CoordMode("Mouse", "Screen")` (or the appropriate mode) before every `MouseClick`, `MouseMove`, `Click`, or `MouseGetPos` call?
- [ ] Did I guard every `Win*` and `Control*` call with either a `WinExist()` pre-check or `try/catch TargetError`?
- [ ] Did I use 1-based indexing for `ControlChooseIndex()` and all `ControlGetItems()` Array access?

## RUNTIME ERROR MAPPING

| Exception Class | Trigger Condition | Detection Code | Fix |
|----------------|-------------------|----------------|-----|
| `TargetError` | `Win*` or `Control*` called when the target window/control has closed (race condition) | `e.Message` contains the window title or "Target" | Wrap in `try/catch TargetError`; re-check existence with `WinExist()` before retrying |
| Runtime crash (method call on Integer 0) | Calling `.Read()` or `WinGetTitle()` using a raw 0 HWND without checking `WinExist()` return | No exception — AHK v2 crashes the thread; absent `if !hwnd` guard | Add `if !hwnd { throw TargetError(…) }` immediately after every `WinExist()` or `WinWait()` call |
| Memory/handle leak (silent, no exception) | `CallbackCreate()` address freed without calling `DllCall("UnhookWinEvent")` first, or `UnhookWinEvent` called without `CallbackFree()` | No runtime error; OS handle table grows; detectable only via Task Manager | Always sequence: `DllCall("UnhookWinEvent",…)` → `CallbackFree(cb)` inside `__Delete()`; zero out both fields afterward |

## TIER 1 — Basic State Checks and Safety Guards
> METHODS COVERED: WinExist · WinActive · WinActivate · WinWaitActive · WinClose · WinWaitClose · WinWait · WinGetMinMax · WinRestore

TIER 1 covers the fundamental entry point for all window work: checking existence, activating, closing, and waiting for windows. Every operation at this tier requires a TargetError guard because any of these functions will throw if the window does not exist when called. In AHK v2, `WinExist()` serves dual purpose — it returns the HWND (Integer) of the matched window for immediate reuse, or 0 if not found.

```ahk
; ============================================================
; TIER 1 — Basic state checks, activation, close, and waiting
; ============================================================

; ✓ WinExist() returns HWND as Integer (non-zero = found); capture it for reuse
hwnd := WinExist("ahk_exe notepad.exe")
if hwnd {
    MsgBox("Notepad HWND: " . hwnd)
} else {
    MsgBox("Notepad is not running.")
}

; ✓ Use the returned HWND for subsequent operations via ahk_id concatenation
hwnd := WinExist("ahk_exe notepad.exe")
if hwnd {
    WinActivate("ahk_id " . hwnd)
    ; WinWaitActive blocks until the window reports as active (up to 3 s)
    if !WinWaitActive("ahk_id " . hwnd,, 3)
        MsgBox("Activation timed out.")
}

; ✗ WRONG — v1-style interpolation is a parse error in AHK v2
; WinActivate, ahk_id %hwnd%

; ✓ CORRECT — string concatenation
WinActivate("ahk_id " . hwnd)

; ✓ WinClose with TargetError guard and close-wait
CloseWindow(winTitle) {
    try {
        WinClose(winTitle)
        WinWaitClose(winTitle,, 5)      ; wait up to 5 s for the window to disappear
    } catch TargetError as e {
        MsgBox("Close failed: " . e.Message)
    }
}
CloseWindow("ahk_exe calc.exe")

; ✓ WinWait blocks until the window appears (returns HWND or 0 on timeout)
hwnd := WinWait("ahk_exe setup.exe",, 30)   ; wait up to 30 s
if !hwnd {
    MsgBox("Setup window never appeared.")
    return
}

; ✓ WinActive returns HWND if the matching window is currently the foreground window
if WinActive("ahk_exe notepad.exe")
    MsgBox("Notepad is the active window.")

; ✓ Check minimized state (WinGetMinMax: -1=min, 0=normal, 1=max)
hwnd := WinExist("ahk_exe notepad.exe")
if hwnd {
    state := WinGetMinMax("ahk_id " . hwnd)
    if (state = -1)
        WinRestore("ahk_id " . hwnd)   ; un-minimize before activating
    WinActivate("ahk_id " . hwnd)
}
```

## TIER 2 — Window Metadata Queries
> METHODS COVERED: WinGetTitle · WinGetClass · WinGetProcessName · WinGetPID · WinGetID · WinGetPos · WinGetList · WinGetStyle · WinGetExStyle · SetTitleMatchMode

TIER 2 extracts structured metadata from an existing window: title, class name, process name, PID, position, size, style bitmasks, and a list of all matching HWNDs. These functions always require a valid target and will throw `TargetError` if the window disappears between the existence check and the query call. Using `"ahk_id " . hwnd` for every query avoids title-matching ambiguity and locale-dependency. `WinGetList()` returns a 1-based Array of Integer HWNDs for every window that matches the given criteria.

```ahk
; ============================================================
; TIER 2 — Window metadata: title, class, process, pos, list
; ============================================================

QueryWindowInfo(winTitle) {
    hwnd := WinExist(winTitle)
    if !hwnd {
        MsgBox("Window not found: " . winTitle)
        return
    }
    idStr := "ahk_id " . hwnd

    ; ✓ All queries target HWND directly for locale-independence
    title   := WinGetTitle(idStr)
    class   := WinGetClass(idStr)
    proc    := WinGetProcessName(idStr)
    pid     := WinGetPID(idStr)

    ; ✓ WinGetPos fills &-prefixed output variables (passed by reference)
    WinGetPos(&x, &y, &w, &h, idStr)

    MsgBox(
        "HWND:    " . hwnd    . "`n"
      . "Title:   " . title   . "`n"
      . "Class:   " . class   . "`n"
      . "Process: " . proc    . "`n"
      . "PID:     " . pid     . "`n"
      . "Pos:     " . x . "," . y . "  Size: " . w . "x" . h
    )
}
QueryWindowInfo("ahk_exe notepad.exe")

; ✓ WinGetList — enumerate all matching windows; returns 1-based Array of Integer HWNDs
ListAllNotepadWindows() {
    winList := WinGetList("ahk_exe notepad.exe")
    if (winList.Length = 0) {
        MsgBox("No Notepad windows found.")
        return
    }
    output := "Notepad windows (" . winList.Length . "):`n"
    for hwnd in winList {                           ; ✓ for-in iterates 1-based Array
        title := WinGetTitle("ahk_id " . hwnd)
        output .= "  HWND=" . hwnd . " | " . title . "`n"
    }
    MsgBox(output)
}
ListAllNotepadWindows()

; ✓ WinGetList with no filter — all visible top-level windows
AllWindows() {
    return WinGetList()         ; returns full HWND Array of all top-level windows
}

; ✓ WinGetID returns the HWND of the first matching window
; Use WinExist() when you also need an existence check; WinGetID throws TargetError if absent
hwnd := WinExist("ahk_exe notepad.exe")   ; preferred — existence check + HWND in one call
; hwnd := WinGetID("ahk_exe notepad.exe") ; ✓ valid alternative; throws TargetError if gone

; ✓ WinGetStyle / WinGetExStyle — read style bitmasks for conditional logic
WS_CAPTION    := 0xC00000   ; standard title bar + border
WS_EX_TOPMOST := 0x0008    ; extended style: always-on-top flag
hwnd := WinExist("ahk_exe notepad.exe")
if hwnd {
    style   := WinGetStyle("ahk_id " . hwnd)
    exStyle := WinGetExStyle("ahk_id " . hwnd)
    if (style & WS_CAPTION)
        OutputDebug("Has title bar.")
    if (exStyle & WS_EX_TOPMOST)
        OutputDebug("Window is always-on-top.")
}

; ✓ Combine title text and ahk_exe for precise matching
hwnd := WinExist("Untitled ahk_exe notepad.exe")

; ✓ Use SetTitleMatchMode before scripts that rely on partial matching
SetTitleMatchMode(2)            ; 2 = match anywhere in title
hwnd := WinExist("notepad")     ; matches "Untitled - Notepad", etc.
SetTitleMatchMode(1)            ; restore to mode 1 (starts-with) after the block
```

## TIER 3 — Coordinate System Declaration and Basic Control Commands
> METHODS COVERED: CoordMode · MouseClick · MouseMove · MouseGetPos · Click · ControlClick · ControlSend · ControlSendText · ControlGetPos · ControlGetHwnd · ControlGetFocus · ControlFocus

TIER 3 introduces `CoordMode` — mandatory before any mouse-coordinate-sensitive call such as `MouseClick`, `Click`, or `MouseMove` — and the primary background-window control functions: `ControlClick`, `ControlSend`, and `ControlSendText`. Note that `CoordMode` does not affect `ControlClick` or `ControlGetPos`; both always operate relative to the target window's client area regardless of CoordMode settings. Controls are identified by their ClassNN descriptor (e.g., `"Edit1"`, `"Button3"`), by their visible text, or by a raw control HWND obtained from `ControlGetHwnd()`. Background-window control commands operate without activating or stealing focus from the target window, making them suitable for automation of concurrent processes.

```ahk
; ============================================================
; TIER 3 — CoordMode declaration + ControlClick / ControlSend
; ============================================================

; ✓ ControlClick targets controls by name/HWND — CoordMode is not involved
BackgroundClick(winTitle, ctrlNN) {
    ControlClick(ctrlNN, winTitle)      ; click center of the named control
}

; ✓ CoordMode options — relevant for MouseMove, MouseClick, Click, MouseGetPos
SetupCoordModes() {
    CoordMode("Mouse",   "Screen")  ; absolute desktop coordinates (multi-monitor aware)
    CoordMode("Mouse",   "Window")  ; relative to window including title bar
    CoordMode("Mouse",   "Client")  ; relative to client area (excludes title/borders)
}

; ✓ ControlClick — full signature with button, count, and options
ClickControlPrecisely(winTitle) {
    ; ControlClick(Control, WinTitle, WinText, WhichButton, ClickCount, Options)
    ControlClick("Button1", winTitle,, "Left", 1, "NA")  ; NA = no window activation
}

; ✓ ControlClick with "Right" for context menus
RightClickControl(winTitle, ctrlNN) {
    ControlClick(ctrlNN, winTitle,, "Right")
}

; ✓ ControlSend — send keystrokes directly to a control (window need not be active)
SendToNotepad() {
    hwnd := WinExist("ahk_exe notepad.exe")
    if !hwnd
        return
    idStr := "ahk_id " . hwnd
    ControlSend("Hello from AHK v2{Enter}", "Edit1", idStr)
    ControlSend("^a",                        "Edit1", idStr)  ; Ctrl+A to select all
}
SendToNotepad()

; ✓ ControlSendText — literal text; no special-key interpretation; safe for passwords/email
; Use ControlSend when {Enter}, ^a, etc. must be interpreted; ControlSendText for raw text
ControlSendText("user@example.com", "Edit1", "ahk_exe browser.exe")

; ✓ ControlGetPos — position of a control within the window
; coordinates are always relative to the target window's client area; CoordMode has no effect
GetControlBounds(winTitle, ctrlNN) {
    ControlGetPos(&x, &y, &w, &h, ctrlNN, winTitle)
    return {x: x, y: y, w: w, h: h}
}

; ✓ ControlGetHwnd — get the Integer HWND of a child control
GetEditHwnd(winTitle) {
    return ControlGetHwnd("Edit1", winTitle)         ; returns Integer HWND
}

; ✓ ControlGetFocus — returns Integer HWND of the currently focused control
GetFocusedControl(winTitle) {
    return ControlGetFocus(winTitle)                 ; returns Integer HWND of focused control
}

; ✓ ControlFocus — programmatically move focus without mouse
FocusSearchBox(winTitle) {
    ControlFocus("Edit1", winTitle)
}

; ✓ CoordMode is required before mouse-coordinate functions, not before ControlClick
; ✗ WRONG — MouseClick without CoordMode uses active window client area by default
; MouseClick("Left", x, y)    ; → may silently misfire if screen coords are intended
; ✓ CORRECT — declare CoordMode first when absolute screen coordinates are needed
CoordMode("Mouse", "Screen")
MouseClick("Left", 500, 300)        ; click at screen-absolute (500, 300)
MouseMove(600, 400)                  ; move cursor to (600, 400) without clicking
MouseGetPos(&curX, &curY)            ; capture current cursor position after move
Click("500 300")                     ; ✓ Click() shorthand — also requires CoordMode
```

## TIER 4 — Advanced Control Data Interaction
> METHODS COVERED: ControlGetText · ControlSetText · ControlChooseIndex · ControlChooseString · ControlGetIndex · ControlGetItems · ControlGetChecked · ControlSetChecked · ControlAddItem · ControlDeleteItem · ControlFindItem

TIER 4 covers reading and writing data stored in controls: text from Edit or Static controls, selected indices and item lists from ListBox and ComboBox controls, checkbox state from Button controls, and item add/remove/find operations. `ControlGetItems()` returns an AHK v2 Array using 1-based indexing — a common source of off-by-one errors for developers accustomed to zero-indexed languages. `ControlChooseIndex()` is also 1-based. All functions in this tier can throw `TargetError` if the control or window is absent.

```ahk
; ============================================================
; TIER 4 — ControlGetText, ControlSetText, ListBox/ComboBox ops
; ============================================================

; ✓ Read text content from an Edit or Static control
ReadEditContent(winTitle) {
    try {
        text := ControlGetText("Edit1", winTitle)
        return text
    } catch TargetError as e {
        MsgBox("Control not found: " . e.Message)
        return ""
    }
}

; ✓ ControlSetText — set content directly (bypasses keyboard events)
; ✓ ControlSend  — inject keystrokes (triggers onChange events in some apps)
WriteToEdit(winTitle, content) {
    try {
        ControlSetText(content, "Edit1", winTitle)      ; direct write, no events
        ; ControlSend(content, "Edit1", winTitle)        ; use this when app needs events
    } catch TargetError as e {
        MsgBox("Write failed: " . e.Message)
    }
}

; ✓ ControlChooseIndex — 1-based; index 1 selects the FIRST item
SelectDropdownByIndex(winTitle, oneBasedIndex) {
    ; ✗ WRONG: ControlChooseIndex("ComboBox1", 0, winTitle) — 0 is invalid, 1-based only
    ; → silent no-op or unexpected result
    ; ✓ CORRECT:
    ControlChooseIndex(oneBasedIndex, "ComboBox1", winTitle)
}

; ✓ ControlChooseString — selects first item containing the string
SelectDropdownByName(winTitle, partialName) {
    ControlChooseString(partialName, "ComboBox1", winTitle)
}

; ✓ ControlGetIndex — returns currently selected 1-based index (0 = nothing selected)
GetSelectedIndex(winTitle) {
    idx := ControlGetIndex("ComboBox1", winTitle)
    if (idx = 0)
        MsgBox("Nothing selected.")
    return idx
}

; ✓ ControlGetItems — returns Array (1-based) of all items in ListBox or ComboBox
EnumerateListBox(winTitle) {
    try {
        items := ControlGetItems("ListBox1", winTitle)  ; returns Array, 1-indexed
        output := "Items (" . items.Length . "):`n"
        for i, item in items {                          ; ✓ 1-based for-in iteration
            output .= "  [" . i . "] " . item . "`n"
        }
        MsgBox(output)
        return items
    } catch TargetError as e {
        MsgBox("ListBox not found: " . e.Message)
        return []
    }
}

; ✓ ControlGetChecked / ControlSetChecked for checkbox controls
ReadCheckbox(winTitle) {
    state := ControlGetChecked("Button1", winTitle)     ; 1 = checked, 0 = unchecked
    return state
}
SetCheckbox(winTitle, checked) {
    ControlSetChecked(checked ? 1 : 0, "Button1", winTitle)
}

; ✓ ControlAddItem — append a new item to a ListBox or ComboBox
AddOption(winTitle, newItem) {
    ControlAddItem(newItem, "ComboBox1", winTitle)
}

; ✓ ControlDeleteItem — remove item at 1-based index (1 = first item)
RemoveOption(winTitle, oneBasedIndex) {
    ; ✗ WRONG: ControlDeleteItem(0, "ComboBox1", winTitle)  → 0 invalid; same 1-based rule
    ControlDeleteItem(oneBasedIndex, "ComboBox1", winTitle)
}

; ✓ ControlFindItem — returns 1-based index of exact match, or 0 if not found
FindAndSelectItem(winTitle, targetName) {
    idx := ControlFindItem(targetName, "ListBox1", winTitle)
    if (idx > 0)
        ControlChooseIndex(idx, "ListBox1", winTitle)   ; ✓ reuse returned 1-based index
    else
        MsgBox("Item not found: " . targetName)
}

; ✓ Full ListBox selection workflow with validation
SafeSelectItem(winTitle, listBoxNN, targetName) {
    hwnd := WinExist(winTitle)
    if !hwnd {
        MsgBox("Window not found.")
        return false
    }
    items := ControlGetItems(listBoxNN, "ahk_id " . hwnd)
    for i, item in items {
        if (item = targetName) {
            ControlChooseIndex(i, listBoxNN, "ahk_id " . hwnd)   ; ✓ 1-based index
            return true
        }
    }
    MsgBox("Item not found: " . targetName)
    return false
}
SafeSelectItem("My App", "ListBox1", "Option B")
```

## TIER 5 — Window Groups, Batch Lifecycle, and OOP Wrapper
> METHODS COVERED: GroupAdd · GroupActivate · GroupDeactivate · GroupClose · WinHide · WinShow · WinMinimize · WinMaximize · WinSetAlwaysOnTop · WinSetTransparent · WinSetTitle · WinRedraw · WinMove · WinGetList · WinActivate · WinWaitActive · WinClose · WinWaitClose · WinGetPos · ControlSend · ControlGetText

TIER 5 covers two complementary advanced patterns: batch group management via `GroupAdd`/`WinGetList`, and the `WindowWrapper` OOP class that encapsulates TIER 1–4 operations behind a safe, HWND-anchored interface. `GroupAdd` defines named window sets by title criteria — not by fixed HWND snapshots — so the group is re-evaluated live each time it is activated. The `WindowWrapper` class demonstrates proper `__New`/`__Delete` lifecycle management for a single window reference.

```ahk
; ============================================================
; TIER 5 — GroupAdd, batch Hide/Show/Minimize, WinSetAlwaysOnTop
; ============================================================

; ✓ Define a window group — called once at startup or on demand
SetupEditorGroup() {
    GroupAdd("Editors", "ahk_exe notepad.exe")
    GroupAdd("Editors", "ahk_exe notepad++.exe")
    GroupAdd("Editors", "ahk_exe code.exe")          ; VS Code
    ; Groups are evaluated live — no snapshot of current windows is taken here
}
SetupEditorGroup()

; ✓ GroupActivate — cycle through group members (activates next match each call)
; "R" reverses direction; omit for forward cycling
^!Tab:: GroupActivate("Editors")        ; Ctrl+Alt+Tab cycles forward
^!+Tab:: GroupActivate("Editors", "R")  ; Ctrl+Alt+Shift+Tab cycles backward

; ✓ GroupDeactivate — activate next window NOT in the group
^!`:: GroupDeactivate("Editors")        ; switch to a non-editor window

; ✓ GroupClose — close the active window if it is a group member
^!w:: GroupClose("Editors")

; ✓ Batch hide all windows in a group
HideGroup(groupName) {
    winList := WinGetList("ahk_group " . groupName)
    if (winList.Length = 0)
        return
    for hwnd in winList
        WinHide("ahk_id " . hwnd)
}
HideGroup("Editors")

; ✓ Batch show and restore all windows in a group
ShowGroup(groupName) {
    winList := WinGetList("ahk_group " . groupName)
    for hwnd in winList {
        WinShow("ahk_id " . hwnd)
        WinRestore("ahk_id " . hwnd)        ; un-minimize if needed
    }
}
ShowGroup("Editors")

; ✓ Batch minimize
MinimizeGroup(groupName) {
    for hwnd in WinGetList("ahk_group " . groupName)
        WinMinimize("ahk_id " . hwnd)
}

; ✓ WinSetAlwaysOnTop — 1 = pin on top, 0 = unpin, -1 = toggle
PinGroupOnTop(groupName, state := 1) {
    for hwnd in WinGetList("ahk_group " . groupName)
        WinSetAlwaysOnTop(state, "ahk_id " . hwnd)
}

; ✓ WinSetTransparent — 0-255 alpha; "Off" to fully restore (not 255)
SetGroupTransparency(groupName, alpha) {
    for hwnd in WinGetList("ahk_group " . groupName)
        WinSetTransparent(alpha, "ahk_id " . hwnd)
}
SetGroupTransparency("Editors", 200)        ; slight transparency while working

; ✓ WinSetTitle — rename visible title (useful for tagging windows by state)
TagWindow(hwnd, label) {
    WinSetTitle(label, "ahk_id " . hwnd)
}

; ✓ WinRedraw — force repaint after transparency or layered changes
ForceRepaint(hwnd) {
    WinRedraw("ahk_id " . hwnd)
}

; ✓ WinGetList with ahk_exe for on-demand batch ops without pre-defined group
CloseAllInstancesOfApp(exeName) {
    winList := WinGetList("ahk_exe " . exeName)
    for hwnd in winList {
        try WinClose("ahk_id " . hwnd)
        catch TargetError
            continue                        ; window closed itself before we reached it
    }
}
CloseAllInstancesOfApp("notepad.exe")

; ✓ WinMove batch layout — tile group windows across the left half of screen
TileGroupLeft(groupName) {
    winList := WinGetList("ahk_group " . groupName)
    count   := winList.Length
    if !count
        return
    screenH := A_ScreenHeight // count
    screenW := A_ScreenWidth  // 2
    for i, hwnd in winList
        WinMove(0, (i - 1) * screenH, screenW, screenH, "ahk_id " . hwnd)
}
TileGroupLeft("Editors")
```

```ahk
; ============================================================
; WindowWrapper — HWND-safe OOP interface for a single window
; Usage: win := WindowWrapper("ahk_exe notepad.exe")
;        win := WindowWrapper(someIntegerHwnd)
; ============================================================
class WindowWrapper {
    _hwnd := 0

    __New(titleOrHwnd) {
        ; ✓ Accept either a WinTitle string or a raw Integer HWND
        if (Type(titleOrHwnd) = "Integer") {
            if !WinExist("ahk_id " . titleOrHwnd)
                throw TargetError("HWND not found: " . titleOrHwnd,, titleOrHwnd)
            this._hwnd := titleOrHwnd
        } else {
            hwnd := WinExist(titleOrHwnd)
            if !hwnd
                throw TargetError("Window not found: " . titleOrHwnd,, titleOrHwnd)
            this._hwnd := hwnd
        }
    }

    ; ✓ Read-only property — single-expression shorthand is valid here
    hwnd {
        get { return this._hwnd }
    }

    title {
        get { return WinGetTitle("ahk_id " . this._hwnd) }
    }

    ; ✓ Returns true if the window still exists
    Exists() {
        return WinExist("ahk_id " . this._hwnd) != 0
    }

    ; ✓ Activate and wait up to `timeout` seconds
    Activate(timeout := 3) {
        WinActivate("ahk_id " . this._hwnd)
        WinWaitActive("ahk_id " . this._hwnd,, timeout)
    }

    ; ✓ Close and wait for the window to disappear
    Close(waitSeconds := 3) {
        WinClose("ahk_id " . this._hwnd)
        WinWaitClose("ahk_id " . this._hwnd,, waitSeconds)
    }

    ; ✓ Populate caller-supplied variables via output refs (&)
    GetPos(&x, &y, &w, &h) {
        WinGetPos(&x, &y, &w, &h, "ahk_id " . this._hwnd)
    }

    Move(x, y, w, h) {
        WinMove(x, y, w, h, "ahk_id " . this._hwnd)
    }

    ; ✓ Send keys directly to a named control without activating the window
    SendToControl(ctrlNN, keys) {
        ControlSend(keys, ctrlNN, "ahk_id " . this._hwnd)
    }

    ; ✓ Read text content from a control
    ReadControl(ctrlNN) {
        return ControlGetText(ctrlNN, "ahk_id " . this._hwnd)
    }
}

; ✗ WRONG — "new" keyword is not valid AHK v2 syntax
; win := new WindowWrapper("Calculator")    ; → parse error

; ✓ CORRECT instantiation:
win := WindowWrapper("Calculator")
MsgBox("Title: " . win.title)
```

## TIER 6 — System-Level Window Hooks via DllCall + SetWinEventHook
> METHODS COVERED: ObjBindMethod · CallbackCreate · CallbackFree · DllCall("SetWinEventHook") · DllCall("UnhookWinEvent") · WinGetTitle

TIER 6 replaces polling loops with passive Windows event subscriptions using `SetWinEventHook` via `DllCall`. The OS calls the registered `WINEVENTPROC` callback whenever a matching window event occurs — creation, destruction, or focus change — with zero CPU overhead at idle. `CallbackCreate` converts an AHK v2 function into a machine-code address that Windows can call directly. All hooks and callback addresses must be explicitly freed in `__Delete()` to prevent handle and memory leaks. The `WinEventMonitor` class below encapsulates this pattern with proper OOP lifecycle.

```ahk
; ============================================================
; TIER 6 — SetWinEventHook passive monitoring, WinEventMonitor
; ============================================================

; ---- Windows API constants ----
; (define at script top or inside the class as static properties)
; EVENT_SYSTEM_FOREGROUND := 0x0003   focus moved to a different window
; EVENT_OBJECT_CREATE     := 0x8000   a window or UI object was created
; EVENT_OBJECT_DESTROY    := 0x8001   a window or UI object was destroyed
; WINEVENT_OUTOFCONTEXT   := 0x0000   callback runs in this script's thread
; OBJID_WINDOW            := 0        idObject value that means "the window itself"

; ============================================================
; WinEventMonitor — OOP wrapper for SetWinEventHook lifecycle
; ============================================================
class WinEventMonitor {

    ; ✓ Access static members via ClassName.property, not this.property
    static EVENT_SYSTEM_FOREGROUND := 0x0003
    static EVENT_OBJECT_CREATE     := 0x8000
    static EVENT_OBJECT_DESTROY    := 0x8001
    static WINEVENT_OUTOFCONTEXT   := 0x0000
    static OBJID_WINDOW            := 0

    _hHook    := 0
    _callback := 0
    _handler  := 0          ; Func or BoundFunc called with (event, hwnd)

    ; eventMin/eventMax — range of EVENT_* constants to subscribe to
    ; handler — a Func/BoundFunc receiving (event, hwnd) for each matched event
    ; pid / tid — filter to a specific process/thread; 0 = all
    __New(eventMin, eventMax, handler, pid := 0, tid := 0) {
        if !IsObject(handler)
            throw TypeError("handler must be a Func or BoundFunc",, handler)

        this._handler := handler

        ; ✓ ObjBindMethod properly binds 'this' so _Dispatch receives correct context
        boundDispatch   := ObjBindMethod(this, "_Dispatch")
        this._callback  := CallbackCreate(boundDispatch, "Fast", 7)

        this._hHook := DllCall("SetWinEventHook",
            "UInt", eventMin,             ; lowest event ID in range
            "UInt", eventMax,             ; highest event ID in range
            "Ptr",  0,                    ; hmodWinEventProc: 0 for out-of-context
            "Ptr",  this._callback,       ; address from CallbackCreate
            "UInt", pid,                  ; process filter (0 = all)
            "UInt", tid,                  ; thread filter  (0 = all)
            "UInt", WinEventMonitor.WINEVENT_OUTOFCONTEXT,
            "Ptr")                        ; return type: Ptr (HWINEVENTHOOK handle)

        if !this._hHook {
            CallbackFree(this._callback)
            this._callback := 0
            throw Error("SetWinEventHook failed — insufficient privileges or bad range")
        }
    }

    ; ✓ _Dispatch is the raw WINEVENTPROC; filter noise before calling user handler
    ; Parameters match the WINEVENTPROC signature exactly (7 args from Windows)
    _Dispatch(hHook, event, hwnd, idObj, idChild, dwThread, dwTime) {
        ; Only forward events for actual windows (OBJID_WINDOW = 0), not UI sub-objects
        if (idObj != WinEventMonitor.OBJID_WINDOW)
            return
        if (hwnd = 0)
            return
        this._handler.Call(event, hwnd)
    }

    ; ✓ __Delete releases the hook handle and callback memory
    ; Called automatically when the last reference to this object is cleared
    __Delete() {
        if this._hHook {
            DllCall("UnhookWinEvent", "Ptr", this._hHook)
            this._hHook := 0
        }
        if this._callback {
            CallbackFree(this._callback)
            this._callback := 0
        }
    }
}

; ============================================================
; Usage example — monitor foreground focus + window lifecycle
; ============================================================
OnWindowEvent(event, hwnd) {
    title := ""
    try title := WinGetTitle("ahk_id " . hwnd)

    switch event {
        case 0x0003: OutputDebug("FOCUS   hwnd=" . hwnd . " | " . title)
        case 0x8000: OutputDebug("CREATED hwnd=" . hwnd . " | " . title)
        case 0x8001: OutputDebug("DESTROYED hwnd=" . hwnd)
    }
}

; ✓ Subscribe to foreground focus through window destruction
g_Monitor := WinEventMonitor(
    WinEventMonitor.EVENT_SYSTEM_FOREGROUND,    ; eventMin
    WinEventMonitor.EVENT_OBJECT_DESTROY,       ; eventMax
    OnWindowEvent                               ; handler Func
)

; ✓ OnExit cleanup — named function avoids multi-statement arrow syntax restriction
CleanupMonitor(*) {
    global g_Monitor
    g_Monitor := ""     ; decrement ref count to 0 → triggers __Delete() automatically
}
OnExit(CleanupMonitor)

; ✗ WRONG — using a class method directly without ObjBindMethod loses 'this' context
; bad := CallbackCreate(this._Dispatch, "Fast", 7)    ; → wrong 'this'; garbage object

; ✗ WRONG — forgetting CallbackFree causes a memory leak that persists until OS restart
; DllCall("UnhookWinEvent", "Ptr", hHook)   ; → without CallbackFree — incomplete cleanup

; ✗ WRONG — a multi-statement arrow function is not valid AHK v2 syntax
; OnExit((*)=> { DllCall("UnhookWinEvent",...) CallbackFree(...) })  ; → parse error
; ✓ CORRECT — named cleanup function (see CleanupMonitor above)
```

### Performance Notes

Window and control operations range from O(1) HWND-direct lookups to O(n) title scans across all open windows. The dominant performance mistake is replacing the OS event system with polling loops — `SetWinEventHook` (TIER 6) consumes zero CPU at idle and should be preferred whenever the task is "react when window X appears or disappears."

**HWND caching:** Resolve a title string to an HWND once with `WinExist()`, then compose `"ahk_id " . hwnd` and reuse it for all subsequent operations. Each title-based `Win*` call performs a full O(n) scan of all top-level windows; HWND-direct calls are O(1) message dispatches. In a loop over multiple windows, cache the `WinGetList()` result and iterate the Array — do not call `WinGetList()` on every iteration.

**ControlGetItems caching:** Call `ControlGetItems()` once and store the returned Array locally. If the same item list is needed across multiple operations in sequence, iterating the cached Array is O(1) per item; calling `ControlGetItems()` repeatedly inside a loop adds a cross-process call on every iteration.

**ControlClick vs MouseClick:** Prefer `ControlClick` over `MouseClick` for background-window automation. `MouseClick` requires the window to be in the foreground, which forces it to the front and disrupts the user. `ControlClick` sends a `WM_LBUTTONDOWN` message directly to the control regardless of z-order — O(1) message post with no activation overhead.

```ahk
; ============================================================
; PERFORMANCE — HWND caching, O(1) lookup, avoiding polling
; ============================================================

; ✗ WRONG — resolves title string anew for every call (O(n) scan each time)
WrongRepeatQuery() {
    WinActivate("ahk_exe notepad.exe")           ; title scan #1
    WinGetTitle("ahk_exe notepad.exe")           ; title scan #2
    WinGetPos(&x,&y,&w,&h,"ahk_exe notepad.exe") ; title scan #3
}

; ✓ CORRECT — resolve once, reuse HWND for O(1) subsequent ops
CachedHwndOps() {
    hwnd := WinExist("ahk_exe notepad.exe")      ; one title scan
    if !hwnd
        return
    idStr := "ahk_id " . hwnd                   ; compose string once
    WinActivate(idStr)
    title := WinGetTitle(idStr)
    WinGetPos(&x, &y, &w, &h, idStr)
}

; ✓ Cache ControlGetItems result; avoid per-iteration calls
ProcessListItems(winTitle) {
    hwnd  := WinExist(winTitle)
    if !hwnd
        return
    items := ControlGetItems("ListBox1", "ahk_id " . hwnd)  ; one call
    ; ✓ iterate the cached Array — ControlGetItems is NOT called again per loop
    for i, item in items {
        if InStr(item, "Error")
            MsgBox("Problem item at index " . i . ": " . item)
    }
}

; ✗ WRONG — burns CPU checking WinExist every 100 ms; ~600 scans/minute wasted
PollBadPattern() {
    while !WinExist("Setup Wizard") {
        Sleep(100)    ; → high CPU for no benefit; misses fast windows
    }
}

; ✓ CORRECT — WinWait for a single target; SetWinEventHook for multi-target or recurring
GoodWaitPattern() {
    hwnd := WinWait("Setup Wizard",, 60)    ; OS-native wait; CPU idles until event fires
    if !hwnd
        MsgBox("Setup never appeared within 60 seconds.")
}

; ✓ Use WinGetList once and cache the Array for batch processing
BatchProcessOnce(exeName) {
    winList := WinGetList("ahk_exe " . exeName)     ; one enumeration pass
    for hwnd in winList {                            ; iterate cached Array
        WinMinimize("ahk_id " . hwnd)               ; O(1) per window via HWND
    }
}

; ✓ Prefer ControlClick over MouseClick for background-window automation
; MouseClick requires foreground activation → forces window to front → disturbs user
; ControlClick works on any window regardless of z-order → O(1) message post
```

## DROP-IN RECIPES

```ahk
; WaitAndActivate — wait for a window, restore if minimized, activate, return HWND
; ✓ Validates inputs; uses OS-native WinWait (no polling); throws with diagnostic message on failure
WaitAndActivate(winTitle, timeout := 10) {
    if !(winTitle is String) || winTitle = ""
        throw TypeError("WaitAndActivate: winTitle must be a non-empty string", -1)
    if !(timeout is Number) || timeout <= 0
        throw ValueError("WaitAndActivate: timeout must be a positive number", -1)

    hwnd := WinWait(winTitle,, timeout)
    if !hwnd
        throw TargetError("WaitAndActivate: window never appeared — " . winTitle, -1, winTitle)

    idStr := "ahk_id " . hwnd

    ; ✓ Restore minimized window before activating — WinActivate on minimized fails silently
    if (WinGetMinMax(idStr) = -1)
        WinRestore(idStr)

    WinActivate(idStr)

    ; ✓ WinWaitActive confirms the activation completed; guards against rapid focus stealing
    if !WinWaitActive(idStr,, 3)
        throw TargetError("WaitAndActivate: activation timed out — " . winTitle, -1, winTitle)

    return hwnd
}
; Call site: hwnd := WaitAndActivate("ahk_exe notepad.exe", 15)


; SafeSelectListItem — find and select a ListBox/ComboBox item by exact name
; ✓ Handles window-not-found, control-not-found, and item-not-found as distinct errors
; ✓ Uses ControlFindItem for O(1) exact match; falls back to linear scan for partial match
SafeSelectListItem(winTitle, ctrlNN, targetName, allowPartial := false) {
    if !(winTitle is String) || winTitle = ""
        throw TypeError("SafeSelectListItem: winTitle must be a non-empty string", -1)
    if !(ctrlNN is String) || ctrlNN = ""
        throw TypeError("SafeSelectListItem: ctrlNN must be a non-empty string", -1)
    if !(targetName is String)
        throw TypeError("SafeSelectListItem: targetName must be a string", -1)

    hwnd := WinExist(winTitle)
    if !hwnd
        throw TargetError("SafeSelectListItem: window not found — " . winTitle, -1, winTitle)

    idStr := "ahk_id " . hwnd

    try {
        ; ✓ ControlFindItem: exact match, returns 1-based index or 0
        idx := ControlFindItem(targetName, ctrlNN, idStr)
        if (idx > 0) {
            ControlChooseIndex(idx, ctrlNN, idStr)
            return idx
        }

        ; ✓ Partial match fallback — only when explicitly requested
        if allowPartial {
            items := ControlGetItems(ctrlNN, idStr)
            for i, item in items {
                if InStr(item, targetName) {
                    ControlChooseIndex(i, ctrlNN, idStr)
                    return i
                }
            }
        }
    } catch TargetError as e {
        throw TargetError("SafeSelectListItem: control error — " . e.Message, -1, ctrlNN)
    }

    throw Error("SafeSelectListItem: item not found — '" . targetName . "' in " . ctrlNN, -1)
}
; Call site: idx := SafeSelectListItem("My App", "ComboBox1", "Option B")
; Call site: idx := SafeSelectListItem("My App", "ListBox1", "partial", true)


; BatchApplyToGroup — apply a window operation to every window matching a WinTitle criterion
; ✓ Captures WinGetList snapshot before loop; handles TargetError per-window (ephemeral close)
; ✓ op is a Func/BoundFunc receiving a single "ahk_id N" string argument
BatchApplyToGroup(winTitle, op) {
    if !(winTitle is String) || winTitle = ""
        throw TypeError("BatchApplyToGroup: winTitle must be a non-empty string", -1)
    if !IsObject(op) || !(op is Func)
        throw TypeError("BatchApplyToGroup: op must be a Func or BoundFunc", -1)

    winList := WinGetList(winTitle)
    if (winList.Length = 0)
        return 0

    successCount := 0
    for hwnd in winList {
        try {
            op.Call("ahk_id " . hwnd)
            successCount++
        } catch TargetError
            continue    ; window closed between snapshot and operation — skip silently
    }
    return successCount
}
; Call site: BatchApplyToGroup("ahk_exe notepad.exe", WinMinimize)
; Call site: BatchApplyToGroup("ahk_group Editors", WinClose)
```

## ANTI-PATTERNS

| Pattern | Wrong | Correct | LLM Common Cause |
|---------|-------|---------|------------------|
| v1 ahk_id interpolation | `WinActivate("ahk_id %hwnd%")` | `WinActivate("ahk_id " . hwnd)` | AHK v1 training data — `%var%` was standard string interpolation |
| HWND compared as string | `if hwnd = "0"` | `if !hwnd` or `if (hwnd = 0)` | Cross-language habit — many languages stringify identifiers; Integer ≠ string in AHK v2 |
| Missing CoordMode for mouse functions | `MouseClick("Left", x, y)` with no CoordMode | `CoordMode("Mouse", "Screen")` then `MouseClick("Left", x, y)` | AHK v2 defaults to active window client area; omitting CoordMode when screen-absolute coordinates are intended causes silent miscoordination |
| 0-based control index | `ControlChooseIndex(0, "CB1", title)` | `ControlChooseIndex(1, "CB1", title)` | Dominant zero-based indexing habit from C, Python, and JavaScript training data |
| Polling instead of WinWait/hook | `while !WinExist("X") Sleep 100` | `WinWait("X",, timeout)` for single wait; `SetWinEventHook` for recurring | AHK v1 lacked a clean wait primitive; polling was the idiomatic pattern in mixed-version training data |
| Unbound method ref to CallbackCreate | `CallbackCreate(this._Dispatch, "Fast", 7)` | `CallbackCreate(ObjBindMethod(this, "_Dispatch"), "Fast", 7)` | Missing AHK v2-specific `ObjBindMethod` knowledge; cross-language habit of passing method references directly |
| Skipping CallbackFree | `DllCall("UnhookWinEvent", "Ptr", hHook)` alone | `DllCall("UnhookWinEvent",…)` then `CallbackFree(cb)` in `__Delete()` | AHK v1 had no `CallbackCreate`; the hook/callback pairing requirement is an AHK v2-specific resource contract |

## SEE ALSO

> This module does NOT cover: pixel searching, image searching, and screen capture → see Module_GraphicsAndScreen.md
> This module does NOT cover: IStream COM interface, async window messaging, and WM_* message pump patterns → see Module_SystemAndCOM.md
> This module does NOT cover: try/catch TargetError structure and error-object field access → see Module_Errors.md
> This module does NOT cover: OOP class design principles beyond WindowWrapper and WinEventMonitor → see Module_Classes.md

- `Module_Errors.md` — `try/catch TargetError` patterns for `WinClose`/`ControlClick` failure; error-object field access (`e.Message`, `e.Extra`); and error-type hierarchy for distinguishing `TargetError` from `OSError`.
- `Module_Classes.md` — `__New`/`__Delete` lifecycle design, dynamic property patterns, and inheritance for extending `WindowWrapper` or `WinEventMonitor` with domain-specific behaviour.
- `Module_SystemAndCOM.md` — `IStream` COM interface, async I/O via Windows message pump, and `WM_*` message-based window communication beyond the scope of `ControlSend`.
- `Module_GraphicsAndScreen.md` — `PixelSearch`, `ImageSearch`, `PixelGetColor`, and multi-monitor coordinate systems for screen-content-based window targeting.
- `Module_DllCallAndMemory.md` — `DllCall` type-string reference, `Buffer` objects for passing structures to Win32 API, and `CallbackCreate` parameter-count rules for non-WINEVENTPROC signatures.