# Module_InputAndHotkeys.md
<!-- DOMAIN: Input and Hotkeys -->
<!-- SCOPE: Mouse pixel/image search, CoordMode, low-level mouse hook operations, and GUI control event binding (OnEvent) are not covered — see Module_GraphicsAndScreen.md and Module_GUI.md. -->
<!-- TRIGGERS: hotkey, hotstring, Send, Click, KeyWait, GetKeyState, InputHook, HotIf, #HotIf, ObjBindMethod, "bind key", "press key", "type text", "keyboard shortcut", "context-sensitive hotkey", "only works in this app", "wait for keypress", "capture typing", "remap key", "block input" -->
<!-- CONSTRAINTS: Hotkey() callbacks must accept the hotkey name as first parameter — use (*) to silently discard it, never (); every #HotIf block must be closed with a bare #HotIf on its own line — omitting silently applies the condition to all subsequent hotkey definitions; GetKeyState() and KeyWait() require quoted string arguments for all key names and mode flags. -->
<!-- CROSS-REF: Module_GraphicsAndScreen.md, Module_GUI.md, Module_AsyncAndTimers.md, Module_Classes.md, Module_Errors.md -->
<!-- VERSION: AHK v2.0+ -->

## V1 → V2 BREAKING CHANGES

| v1 pattern (LLM commonly writes) | v2 correct form | Consequence |
|----------------------------------|-----------------|-------------|
| `^l::` / `Send("Text")` / `return` | `^l:: { Send("Text") }` | SyntaxError — bare `return` terminates the script in v2, not a hotkey body; braces are mandatory for multi-line hotkeys |
| `#IfWinActive ahk_exe notepad.exe` | `#HotIf WinActive("ahk_exe notepad.exe")` | Hotkey never activates — `#IfWinActive` is an unrecognised v1 directive in v2 |
| `Hotkey, ^c, MyLabel` | `Hotkey("^c", MyFunctionRef)` | RuntimeError — string labels do not exist in v2; a function object is required |
| `Send %myVariable%` | `Send(myVariable)` | SyntaxError — percent-variable syntax is v1 only; v2 requires expression syntax |
| `GetKeyState(CapsLock, T)` | `GetKeyState("CapsLock", "T")` | RuntimeError or silent wrong comparison — unquoted identifiers are variable reads, not string literals |
| `KeyWait, Enter, D` | `KeyWait("Enter", "D")` | SyntaxError — v2 function call syntax requires parentheses and quoted strings |
| `KeyWait("Enter")` then `if !ErrorLevel` | `if !KeyWait("Enter", opts)` | Silent logic error — ErrorLevel was removed in v2; use KeyWait's boolean return value (1 = success, 0 = timeout) |
| `Hotkey("F1", () => Action())` | `Hotkey("F1", (*) => Action())` | RuntimeError — AHK v2 passes the hotkey name as the first argument; a zero-param lambda rejects the call |
| `Hotkey("a", this.Method)` | `Hotkey("a", ObjBindMethod(this, "Method"))` | TypeError — raw method reference loses `this` context; ObjBindMethod preserves it at call time |
| `SetKeyDelay(50)` after `SendMode("Input")` | Switch to `SendMode("Event")` before calling `SetKeyDelay()` | Silent failure — SetKeyDelay has no effect in SendInput mode; delays are silently ignored |

## API QUICK-REFERENCE

### Send Functions
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `Send()` | `Send(keys)` | Sends key sequence; uses current SendMode (default in v2: SendInput) |
| `SendText()` | `SendText(text)` | Sends literal text with no special-character interpretation; safe for strings containing `{`, `+`, `^`, `!` |
| `SendEvent()` | `SendEvent(keys)` | Sends using the legacy SendEvent API; respects SetKeyDelay regardless of the active SendMode |
| `SendMode()` | `SendMode(mode)` | Sets the mode for subsequent `Send()` calls; accepted values: `"Input"`, `"Event"`, `"Play"`, `"InputThenPlay"` |
| `SetKeyDelay()` | `SetKeyDelay(delay, pressDuration?)` | Sets per-key delay and press duration for SendEvent/SendPlay only; silently ignored under SendInput |
| `BlockInput()` | `BlockInput(option)` | Blocks user keyboard/mouse input; option: `"On"`, `"Off"`, `"Send"`, `"Mouse"`, `"SendAndMouse"` |

### Click and Mouse
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `Click()` | `Click(opts?)` | Simulates a mouse click; opts is a space-separated string of coordinates, button, count, `"Down"`/`"Up"` |
| `MouseClick()` | `MouseClick(btn?, x?, y?, count?, speed?, d?, r?)` | Explicit-parameter form; btn: `"Left"`, `"Right"`, `"Middle"`, `"WheelUp"`, etc. |

### Hotkey Control
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `Hotkey()` | `Hotkey(keyName, action?, options?)` | Creates or modifies a dynamic hotkey; action is a function object, `"On"`, `"Off"`, or `"Toggle"` |
| `HotIf()` | `HotIf(criterion?)` | Sets context for **all subsequently-created hotkeys in the current thread** — not a global toggle; call with no args to reset to unconditional |
| `#HotIf` | `#HotIf [expression]` | Directive — applies condition to all hotkeys until the next bare `#HotIf`; expression evaluated per trigger-key press |
| `ObjBindMethod()` | `ObjBindMethod(obj, method, args*)` | Returns a bound function object; preserves `this` context for class method callbacks |

### Key State and Wait
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `GetKeyState()` | `GetKeyState(keyName, mode?)` | mode: `"P"` = physical hardware state, `"T"` = toggle state (CapsLock/NumLock); returns 1 (down/on) or 0 (up/off) |
| `KeyWait()` | `KeyWait(keyName, options?)` | Waits for key state change; returns 1 on success, 0 on timeout; options: `"D"` (wait for down), `"T#"` (timeout), `"L"` (logical state) |

### InputHook Object
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `InputHook()` | `InputHook(options?, endKeys?, matchList?)` | Creates an input capture object; options: `"L#"` (max chars), `"T#"` (timeout seconds), `"V"` (visible to user) |
| `.Start()` | `.Start()` | Begins intercepting keyboard input |
| `.Wait()` | `.Wait(maxSeconds?)` | Blocks until capture ends; returns the EndReason string |
| `.Stop()` | `.Stop()` | Terminates an active capture immediately |
| `.Input` | `.Input` | Property — captured text accumulated so far |
| `.EndReason` | `.EndReason` | Property — why capture ended: `"Timeout"`, `"Max"`, `"EndKey"`, `"Match"`, `"Stopped"` |
| `.KeyOpt()` | `.KeyOpt(keys, options)` | Per-key capture options: `"S"` suppress, `"V"` visible, `"N"` notify |

### Built-in Variables
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `A_PriorHotkey` | `A_PriorHotkey` | Name of the previously activated hotkey |
| `A_TimeSincePriorHotkey` | `A_TimeSincePriorHotkey` | Milliseconds elapsed since the prior hotkey fired; use for double-tap detection |
| `A_SendMode` | `A_SendMode` | Current send mode string; read-only — save before any temporary `SendMode()` override |
| `A_KeyDelay` | `A_KeyDelay` | Current key delay in ms; save before `SetKeyDelay()` for temporary overrides |
| `A_KeyDuration` | `A_KeyDuration` | Current press duration in ms; must be saved and restored alongside `A_KeyDelay` |

### Utility Functions (used in TIER examples)
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `WinActive()` | `WinActive(winTitle?, winText?, exTitle?, exText?)` | Returns the HWND of the active window if it matches; returns 0 otherwise |
| `Sleep()` | `Sleep(ms)` | Pauses script execution for the given number of milliseconds |
| `ToolTip()` | `ToolTip(text?, x?, y?, n?)` | Displays a small floating tooltip; call with no args to hide it |
| `SetTimer()` | `SetTimer(callback?, period?)` | Schedules a callback; negative period = one-shot after `|period|` ms |
| `FormatTime()` | `FormatTime(YYYYMMDDHH24MISS?, format?)` | Formats a timestamp string |
| `StrLen()` | `StrLen(str)` | Returns the character count of a string |
| `Map()` | `Map(key, val, ...)` | Key-value store used in class examples; see Module_DataStructures.md for full reference |
| `.Has()` | `.Has(key)` | Map method — returns 1 if key exists, 0 otherwise |
| `.Delete()` | `.Delete(key)` | Map method — removes a key-value pair and returns the removed value |

## AHK V2 CONSTRAINTS

- Multi-line hotkey bodies must be wrapped in braces `{ }` — bare `return` is a v1 termination pattern that causes a SyntaxError in v2.
- `Hotkey()` callbacks must accept **at least one parameter** (the hotkey name AHK v2 passes automatically) — use `(*)` in arrow functions to silently absorb it; `()` with zero parameters causes a RuntimeError on invocation.
- Every `#HotIf` block must be closed with a **bare `#HotIf`** on its own line — omitting the closing directive silently applies the active condition to all hotkey definitions that follow, producing impossible-to-debug activation failures.
- `GetKeyState()` and `KeyWait()` require **quoted strings** for key names and mode flags — unquoted identifiers are treated as variable reads, not key name literals.
- `SetKeyDelay()` has no effect when the active send mode is `"Input"` (the v2 default) — call `SendMode("Event")` first, then `SetKeyDelay()`, then restore both using saved `A_SendMode` / `A_KeyDelay` / `A_KeyDuration` values.
- `#HotIf` expressions are **evaluated each time the hotkey's trigger key is pressed** — keep them O(1) and free of disk I/O, network calls, and slow computation; a slow expression causes measurable input lag whenever that trigger key is used.
- Use `ObjBindMethod(this, "MethodName")` for class method hotkey callbacks — raw method references (`this.Method`) lose `this` context at call time and cause a TypeError.
- Dynamic hotkeys registered in `__New()` must be unregistered in `__Delete()` — orphaned callbacks hold a reference that prevents garbage collection and leaves phantom hotkeys active after the object is logically destroyed.

Safe-access priority order for hotkey callback definitions:
  1. Named function with explicit `hotkeyName` parameter — clearest intent, easiest to debug
  2. Arrow function with `(*) =>` — for inline one-liners that do not need the hotkey name
  3. `ObjBindMethod(obj, "Method")` — when binding class instance methods to hotkeys
  4. `(hk) => expr` lambda passed to `HotIf()` — only for runtime-dynamic context conditions

Paired prohibition examples:
- ✗ `Hotkey("F1", () => Action())` — RuntimeError: callback receives hotkey name arg; zero params rejects it
- ✓ `Hotkey("F1", (*) => Action())` — variadic `*` silently absorbs the required hotkey name

- ✗ `Hotkey("F1", this.MyMethod)` — TypeError: `this` context lost; method executes as a plain function
- ✓ `Hotkey("F1", ObjBindMethod(this, "MyMethod"))` — bound method retains `this` at call time

- ✗ `#HotIf FileExist("C:\flag.txt")` — disk I/O on every trigger-key press causes input lag
- ✓ Cache state in a variable updated by a timer or a separate hotkey; read only the variable inside `#HotIf`

## TIER 1 — Static Hotkeys, Hotstrings, and Send Fundamentals
> METHODS COVERED: Send · SendText · SendMode · SetKeyDelay · SendEvent · Click · Sleep

Static hotkeys use `::` syntax with braces required for multi-line bodies. Hotstrings use `::trigger::replacement` (single-line) or `::trigger:: { }` (multi-line function block). `Send()` defaults to SendInput mode in v2 — the fastest and most reliable option for standard Windows applications. `SetKeyDelay()` is silently ignored under SendInput; switch to `SendMode("Event")` explicitly when hardware-level key timing is required, and always save and restore all three state variables (`A_SendMode`, `A_KeyDelay`, `A_KeyDuration`) around any temporary override.
```ahk
; ✓ Single-line hotkey — expression syntax; no braces needed for a single statement
^j::Send("Hello World{Enter}")

; ✓ Multi-line hotkey — braces are mandatory; body is a function block in v2
^k:: {
    Send("Line 1{Enter}")
    Sleep(100)
    Click(100, 200)
}

; ✗ v1 bare-return pattern — SyntaxError in v2; braces are required for multi-line
; ^l::
; Send("Text")
; return              ; → SyntaxError: bare return outside a function body

; ✓ Single-line hotstring — auto-replaces "btw" with "by the way" on trigger character
::btw::by the way

; ✓ Multi-line hotstring — function block allows dynamic content like built-in variables
::/time:: {
    Send(A_Hour ":" A_Min)
}

; --- Send Mode Management ---

; ✓ Explicit SendMode at script top — self-documents intent; applies to all Send() calls below
SendMode("Input")   ; Default in v2; stated explicitly for clarity

; ✓ Save all three state variables before override; restore all three after
;   SetKeyDelay() takes two parameters — omitting pressDuration leaves it at its previous value
CompatSend(text) {
    prevMode     := A_SendMode
    prevDelay    := A_KeyDelay
    prevDuration := A_KeyDuration
    SendMode("Event")
    SetKeyDelay(30, 30)         ; 30 ms between keys, 30 ms press duration
    Send(text)
    SetKeyDelay(prevDelay, prevDuration)
    SendMode(prevMode)
}

; ✓ SendPlay for legacy DirectX/game targets that ignore SendInput — always restore afterward
LegacySend(text) {
    SendMode("Play")
    Send(text)
    SendMode("Input")           ; Always restore to default after the call
}

; ✗ SetKeyDelay while in SendInput mode — delay setting is silently ignored
; SendMode("Input")
; SetKeyDelay(50, 50)           ; → Has NO effect when mode is Input; use SendEvent first

; ✗ SendPlay for system-level shortcuts — Win key events are routed to the active window, not the OS
; SendMode("Play")
; Send("#d")                    ; → Will NOT minimize all windows; Win key goes to active window, not OS
```

## TIER 2 — Dynamic Hotkeys with Hotkey()
> METHODS COVERED: Hotkey · SendText

`Hotkey()` creates or modifies hotkeys at runtime and requires a **function object** as its second argument — never a string label. Every callback must accept at least one parameter (the hotkey name AHK v2 passes automatically). Use `(*)` in arrow functions to discard it; use a named parameter to inspect it. Toggling an existing hotkey passes the string `"On"`, `"Off"`, or `"Toggle"` as the second argument instead of a function object.
```ahk
; ✓ Named callback accepts the hotkey name AHK v2 passes as the first argument automatically
CustomAction(hotkeyName) {
    SendText("You pressed " hotkeyName)
}

; ✓ Assign a named function reference to a dynamic hotkey — function object, not a string label
Hotkey("F1", CustomAction)

; ✓ Arrow function with (*) — absorbs the required hotkey name parameter without naming it
Hotkey("F2", (*) => Send("Quick inline action"))

; ✗ Zero-param arrow function — RuntimeError when AHK v2 passes the hotkey name at call time
; Hotkey("F2", () => Send("action"))    ; → RuntimeError: too many parameters

ToggleF1(*) {
    static state := true
    state := !state
    ; ✓ Toggle an existing dynamic hotkey by passing a string instead of a new callback
    Hotkey("F1", state ? "On" : "Off")
}

; ✓ Assign the toggle function — ToggleF1 accepts (*) because the hotkey name is not needed
Hotkey("F3", ToggleF1)
```

## TIER 3 — Key State Queries and Wait Operations
> METHODS COVERED: GetKeyState · KeyWait · A_PriorHotkey · A_TimeSincePriorHotkey

`GetKeyState()` and `KeyWait()` require quoted strings for all key names and mode flags — unquoted identifiers are variable reads, not string literals. `KeyWait()` returns `1` (key reached target state) or `0` (timed out); check the return value directly — `ErrorLevel` does not exist in v2. Default `KeyWait()` with no options waits for the key to be released (key-up); add `"D"` to wait for key-down instead. `A_PriorHotkey` and `A_TimeSincePriorHotkey` together enable double-tap detection without a timer.
```ahk
~Shift:: {
    ; ✓ "T" reads the toggle state (CapsLock on/off); "P" reads physical hardware state
    if GetKeyState("CapsLock", "T") {
        Send("CapsLock is ON while pressing Shift{Enter}")
    }

    ; ✓ No "D" option — default KeyWait waits for the key to be released (key-up)
    KeyWait("Shift")
    Send("Shift released{Enter}")
}

DoubleTapCheck(*) {
    ; ✓ Use return value of KeyWait directly — ErrorLevel was removed in AHK v2
    ; ✓ A_PriorHotkey and A_TimeSincePriorHotkey detect rapid sequential presses without a timer
    if (A_PriorHotkey = "~Ctrl" and A_TimeSincePriorHotkey < 400) {
        Send("Double tapped Ctrl!{Enter}")
    }
}

; ✗ Unquoted key name — treated as a variable read, not a string literal
; if GetKeyState(CapsLock, T)           ; → RuntimeError or silent wrong comparison

; ✗ ErrorLevel after KeyWait — ErrorLevel does not exist in AHK v2
; KeyWait("Enter")
; if !ErrorLevel                        ; → Silent logic error: ErrorLevel is always unset in v2
```

## TIER 4 — Context-Sensitive Routing with #HotIf and HotIf()
> METHODS COVERED: #HotIf · HotIf · WinActive · ToolTip · SetTimer

`#HotIf` is a load-time directive that applies a condition to all hotkey definitions until the next bare `#HotIf`. Its expression is re-evaluated **each time the hotkey's trigger key is pressed** — keep it O(1) and I/O-free. `HotIf()` is the runtime equivalent, setting context for **all subsequently-created hotkeys in the current thread**; always reset it immediately after the desired `Hotkey()` call(s) with `HotIf()` (no args) so subsequent `Hotkey()` calls are unconditional.
```ahk
; --- PATTERN 1: Single-window context ---
; ✓ Bare #HotIf below is REQUIRED — omitting silently applies the condition to all hotkeys that follow
#HotIf WinActive("ahk_exe notepad.exe")
^s:: {
    Send("Saving in Notepad...{Enter}")
}
#HotIf  ; Resets context — all hotkeys defined below this line are unconditional


; --- PATTERN 2: Stacked conditions (AND logic via compound expression) ---
; ✓ Both conditions are evaluated atomically on each trigger-key press — compound expression must remain fast
#HotIf WinActive("ahk_exe chrome.exe") and GetKeyState("Ctrl", "P")
F5:: {
    Send("Ctrl+F5 equivalent in Chrome only{Enter}")
}
#HotIf


; --- PATTERN 3: Custom function for complex application state ---
; ✓ Function is called on each trigger-key press — must be O(1) with no disk I/O
IsEditorReady() {
    ; Reads only an in-memory object property — safe for use inside #HotIf
    return WinActive("ahk_exe code.exe") and AppState.EditMode
}

#HotIf IsEditorReady()
^Enter:: {
    Send("Running code block from editor...{Enter}")
}
#HotIf

; ✗ Slow I/O inside #HotIf — disk read executes on each trigger-key press
; #HotIf FileExist("C:\config\active.flag")   ; → Causes measurable input lag for that hotkey
; ^x::Send("hi")
; #HotIf


; --- PATTERN 4: Runtime-dynamic context switching with HotIf() ---
; HotIf() sets context for all subsequently-created hotkeys in the current thread — not a persistent global toggle
global IsCustomMode := false

F4:: {
    global IsCustomMode := !IsCustomMode
    ; ✓ Arrow function captures the current variable — re-evaluated on each trigger-key press
    HotIf((hk) => IsCustomMode)
    Hotkey("Space", CustomSpaceAction, "On")
    HotIf()  ; Reset immediately — subsequent Hotkey() calls will be unconditional
    ToolTip("Custom mode: " (IsCustomMode ? "ON" : "OFF"))
    SetTimer(() => ToolTip(), -1500)
}

CustomSpaceAction(hk) {
    Send("Custom space mode active!")
}


; --- PATTERN 5: Exclusion logic with NOT ---
; ✓ Hotkey active everywhere EXCEPT the named application
#HotIf not WinActive("ahk_exe teams.exe")
^Space:: {
    Send("Global launcher — suppressed in Teams")
}
#HotIf
```

## TIER 5 — InputHook and Custom Modifier Chords
> METHODS COVERED: InputHook · .Start · .Wait · .Stop · .Input · .EndReason · GetKeyState · KeyWait · SetKeyDelay · SendEvent

Custom modifier chords use the `&` combinator to bind two keys without activating either individually. `InputHook` intercepts the raw keyboard stream, making it the correct tool for capturing arbitrary-length text sequences, modal interfaces, or key-chord sequences that static hotkeys cannot express. Always check `.EndReason` before reading `.Input` to determine whether the capture succeeded or timed out.
```ahk
; ✓ Custom chord — CapsLock acts as modifier; neither key fires its default action
CapsLock & a:: {
    Send("Custom chord executed.")
}

CaptureTextSequence(*) {
    ; ✓ "L5" = max 5 chars, "T3" = 3-second timeout; "{Esc}" terminates early as an end key
    ih := InputHook("L5 T3", "{Esc}")
    ih.Start()
    ih.Wait()

    ; ✓ Check EndReason first — "Max" means length limit reached; "Timeout" means no input
    if (ih.EndReason = "Timeout") {
        Send("Input timed out.")
    } else if (ih.EndReason = "Max") {
        Send("Captured: " ih.Input)
    }
}

Hotkey("^!c", CaptureTextSequence)
```

### Performance Notes

**SendInput vs SendEvent:** `Send()` defaults to SendInput — all keystrokes are buffered into a single OS injection before any user input can interleave. This is fastest and most reliable for standard applications. `SetKeyDelay()` is silently ignored under SendInput. Switch to `SendMode("Event")` only when the target application requires hardware-level key timing (e.g., MMO clients with anti-cheat input scanning), and always restore all three state variables (`A_SendMode`, `A_KeyDelay`, `A_KeyDuration`) afterward.

**Hook overhead (`$` and `~`):** Every hotkey prefixed with `$` or `~` installs a low-level OS keyboard or mouse hook. A small number is negligible, but dozens of globally-scoped `$`- or `~`-prefixed hotkeys cause measurable system-wide input latency — affecting **all** keyboard and mouse input, not just input to the AHK script. Scope hooks with `#HotIf`: inactive `#HotIf` hotkeys do not install a persistent hook, so the overhead disappears completely when the condition is false.

- `$` forces a keyboard hook to prevent the hotkey from being bypassed by `Send()`.
- `~` passes the original keystroke through to the application AND runs the hotkey action.
- Both should be limited to scoped `#HotIf` blocks unless the specific behaviour is explicitly required.

**Polling vs event-driven:** Avoid polling `GetKeyState()` in tight loops — this burns CPU and degrades responsiveness. Prefer `KeyWait()` for single key-state detection and `InputHook` for capturing sequences.
```ahk
; ✓ O(1) event-driven — KeyWait returns 1 (key reached state) or 0 (timeout); no ErrorLevel
WaitEfficient() {
    if KeyWait("Enter", "D T5") {
        Send("Enter pressed within 5 seconds.")
    }
}

; ✗ High-CPU polling loop — burns cycles; KeyWait handles this case natively
WaitInefficient() {
    Loop {
        if GetKeyState("Enter", "P") {
            Send("Enter pressed.")
            break
        }
        Sleep(10)                           ; → Wastes CPU; use KeyWait instead
    }
}

; ✓ Save both Delay and Duration before overriding — SetKeyDelay() takes two parameters
GameSend() {
    oldDelay    := A_KeyDelay
    oldDuration := A_KeyDuration
    SetKeyDelay(50, 50)
    SendEvent("Action")         ; SendEvent respects SetKeyDelay; SendInput does not
    SetKeyDelay(oldDelay, oldDuration)
}

; --- Hook scope examples ---

; ✗ Dozens of global $-prefixed hooks — each adds OS-level interception overhead everywhere
; $^a::Action1()
; $^b::Action2()
; $^c::Action3()    ; → Repeated at scale: measurable system-wide input lag

; ✓ Scope $-prefixed hooks with #HotIf — hook deactivates entirely outside the target context
#HotIf WinActive("ahk_exe myapp.exe")
$^a::Action1()     ; Hook active only inside myapp.exe — zero cost everywhere else
$^b::Action2()
#HotIf

; ✓ Use ~ only when pass-through is explicitly required
~LButton:: {
    ; ~ passes the click to the application AND runs this block
    ; Without ~: click is consumed by AHK and does NOT reach the target window
    HighlightClickPosition()
}

; ✗ ~ on a high-frequency event without scoping — fires on every wheel tick system-wide
; ~WheelUp::LogScroll()         ; → Severe hook overhead on every scroll movement globally
; ✓ Scope it to eliminate the cost when outside the target window
#HotIf WinActive("ahk_exe myapp.exe")
~WheelUp::LogScroll()
#HotIf
```

## TIER 6 — Class-Encapsulated Hotkey Management
> METHODS COVERED: ObjBindMethod · Hotkey (On/Off) · Map · .Has · .Delete · InputHook (integrated) · FormatTime · StrLen

Encapsulating hotkeys in a class prevents global namespace pollution and guarantees lifecycle cleanup. `ObjBindMethod(this, "MethodName")` creates a bound function object that preserves the instance's `this` context — raw `this.Method` references lose context at call time. Store bound callbacks in a `Map` keyed by hotkey name so `__Delete()` can iterate and unregister them all, preventing orphaned callbacks that hold references blocking garbage collection.
```ahk
; --- PATTERN A: Generic HotkeyManager (reusable foundation) ---
class HotkeyManager {
    __New() {
        this.ActiveBinds := Map()
    }

    ; ✓ ObjBindMethod preserves this context — raw this.Method would lose it at call time
    BindKey(keyName, methodName) {
        callback := ObjBindMethod(this, methodName)
        Hotkey(keyName, callback, "On")
        this.ActiveBinds[keyName] := callback
    }

    UnbindKey(keyName) {
        if this.ActiveBinds.Has(keyName) {
            Hotkey(keyName, "Off")
            this.ActiveBinds.Delete(keyName)
        }
    }

    ; ✓ AHK v2 passes the hotkey name as the first parameter — accept it explicitly here
    ExecuteAction(hotkeyName) {
        Send("Action triggered by " hotkeyName "{Enter}")
    }

    ; ✓ __Delete unregisters all hotkeys — prevents orphaned callbacks on object destruction
    __Delete() {
        for keyName, _ in this.ActiveBinds {
            Hotkey(keyName, "Off")
        }
    }
}


; --- PATTERN B: Domain-specific manager with InputHook integration ---
class SnippetManager {
    __New() {
        this.Snippets := Map(
            "sig",  "Best regards,`nJohn Doe",
            "date", FormatTime(A_Now, "yyyy-MM-dd")
        )

        ; ✓ Store the bound callback so __Delete can reference the exact same object to unregister
        this.TriggerBind := ObjBindMethod(this, "ListenForSnippet")
        Hotkey("F8", this.TriggerBind, "On")
    }

    ListenForSnippet(hk) {
        ; ✓ "V" = visible (user sees what they type); T2 = 2-second timeout before giving up
        ih := InputHook("V T2", "{Space}{Enter}")
        ih.Start()
        ih.Wait()

        word := ih.Input
        if this.Snippets.Has(word) {
            ; Erase the typed trigger word plus its terminating character, then insert snippet
            Send("{Backspace " StrLen(word) + 1 "}")
            Send(this.Snippets[word])
        }
    }

    ; ✓ Unregister using the stored bound callback reference — same object Hotkey() registered
    __Delete() {
        Hotkey("F8", this.TriggerBind, "Off")
    }
}

; ✓ Assign to a global variable to keep the instance alive for the script's lifetime
global snippetApp := SnippetManager()
```

## ANTI-PATTERNS

| Pattern | Wrong | Correct | LLM Common Cause |
|---------|-------|---------|------------------|
| Multi-line hotkey without braces | `^l::` / `Send(...)` / `return` | `^l:: { Send(...) }` | AHK v1 allowed bare `return` to end hotkey bodies; v2 requires braces — common v1 regression |
| Legacy context directive | `#IfWinActive ahk_exe notepad.exe` | `#HotIf WinActive("ahk_exe notepad.exe")` | v1 training data — `#IfWinActive` is a v1 directive not recognised in v2 |
| String label as Hotkey callback | `Hotkey("^c", "MyLabel")` | `Hotkey("^c", MyFunctionRef)` | v1 used label strings; v2 requires function objects — high-frequency LLM v1 regression |
| Zero-param arrow callback | `Hotkey("F1", () => Action())` | `Hotkey("F1", (*) => Action())` | Most languages allow zero-param callbacks freely; AHK v2 always passes the hotkey name as first arg |
| Unquoted key names | `GetKeyState(CapsLock, T)` | `GetKeyState("CapsLock", "T")` | v1 allowed bare identifiers as key names; v2 requires string expressions throughout |
| ErrorLevel after KeyWait | `KeyWait("Enter")` / `if !ErrorLevel` | `if !KeyWait("Enter", opts)` | ErrorLevel was AHK v1's global error indicator; removed entirely in v2; use return value |
| SetKeyDelay with SendInput | `SendMode("Input")` + `SetKeyDelay(50)` | Switch to `SendMode("Event")` before `SetKeyDelay()` | Unaware that SetKeyDelay is mode-specific; silently ignored under SendInput — no error is raised |
| Unbound method as callback | `Hotkey("a", this.Method)` | `Hotkey("a", ObjBindMethod(this, "Method"))` | JavaScript/Python-style method reference; AHK v2 loses `this` context without explicit binding |
| Slow I/O inside #HotIf | `#HotIf FileExist("flag.txt")` | Cache result in a variable; read only the variable inside `#HotIf` | Treating `#HotIf` like a conditional block rather than a per-trigger-key-press expression |
| Missing #HotIf reset | `#HotIf WinActive(...)` / hotkeys / *(no closing #HotIf)* | Always close with a bare `#HotIf` | Analogy to unclosed scope in other languages — easy to omit; produces silent context bleed |

## SEE ALSO

> This module does NOT cover: mouse pixel/image search, CoordMode, and screen-coordinate operations — see Module_GraphicsAndScreen.md
> This module does NOT cover: GUI control event binding, button callbacks, and OnEvent handlers — see Module_GUI.md
> This module does NOT cover: SetTimer for delayed callbacks and background task scheduling — see Module_AsyncAndTimers.md
> This module does NOT cover: class method binding, inheritance, and OOP design beyond hotkey managers — see Module_Classes.md
> This module does NOT cover: try/catch patterns for Hotkey() registration failures and InputHook errors — see Module_Errors.md

- `Module_GraphicsAndScreen.md` — PixelSearch, ImageSearch, CoordMode, and all screen-coordinate mouse operations.
- `Module_GUI.md` — Gui control event handlers (OnEvent), button and edit callbacks, window message processing.
- `Module_AsyncAndTimers.md` — SetTimer patterns for deferred execution, hotkey cooldown timers, and periodic callbacks.
- `Module_Classes.md` — Class definition, `__New`/`__Delete` lifecycle, inheritance, and `ObjBindMethod` beyond hotkey use.
- `Module_Errors.md` — try/catch patterns for Hotkey() registration failures, InputHook error handling, and error propagation.