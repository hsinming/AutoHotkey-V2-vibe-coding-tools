# Module_CodeReview.md
<!-- DOMAIN: Code Review -->
<!-- SCOPE: AHK v2 syntax rules, API semantics, and domain-specific patterns are not defined here — they are enforced by Module_Instructions.md and the domain modules; this module provides only the six-dimension evaluation framework and severity-graded output format for reviewing submitted scripts. -->
<!-- TRIGGERS: review, audit, "code review", "check my script", "find bugs", "quality check", "what's wrong with", "is this v2 compliant", "best practices check", "is my error handling correct", "why does my script crash", "refactor review", MsgBox(), SetTimer(), ComObject(), .Bind(), .OnEvent() -->
<!-- CONSTRAINTS: `=` used as assignment is always Critical — it is comparison in v2 and silently leaves the variable as ""; never flag MsgBox(), ToolTip(), or Sleep() as needing try/catch — they do not throw; every catch block must surface e.Message or e.Extra — an empty catch body is always Critical. -->
<!-- CROSS-REF: Module_Instructions.md, Module_Errors.md, Module_GUI.md, Module_Classes.md -->
<!-- VERSION: AHK v2.0+ -->

## V1 → V2 BREAKING CHANGES

| v1 pattern (LLM commonly writes) | v2 correct form | Consequence |
|----------------------------------|-----------------|-------------|
| `counter = 0` as assignment | `counter := 0` | `=` is comparison in v2 — variable stays `""` with no runtime error; all downstream logic is silently wrong |
| Label-based hotkey `^F1:: ... return` | `^F1:: expr` or `^F1:: { block }` | v1 label form is a parse error or no-op in v2; hotkey never fires |
| `MsgBox, Title, Text` command form | `MsgBox("Text", "Title")` | v1 command syntax is a load-time parse error in v2 scripts |
| `SetTimer, FuncLabel, 1000` | `SetTimer(FuncRef, 1000)` | Label-based SetTimer raises NameError in v2; timer silently never runs |
| `MyBtn := MyGui.Add("Button", "gMyAction w120", "Run")` g-label | `MyBtn.OnEvent("Click", handler)` | g-labels do not exist in v2 — call is silently ignored |
| `(params) => { stmt1`\n`stmt2 }` multi-statement arrow fn | Named function + `.OnEvent("Click", namedFn)` | Multi-line fat-arrow body is a load-time parse error in v2 |
| `ObjRelease(xlApp)` on a `ComObject()` wrapper | `xlApp := ""` | Double-free risk — `ObjRelease` is only correct for raw pointers from `ComObjQuery()`/`DllCall()` |
| `%dynamicVar%` percent-dereferencing in expressions | `Map()` keyed lookup or direct variable | `%Var%` in expression context behaves differently in v2; pattern is a v1 holdover and should be eliminated |

## API QUICK-REFERENCE

### Severity Classification

| Severity | Symbol | Meaning |
|----------|--------|---------|
| Critical | 🔴 | Causes crashes, silent data loss, or wrong behavior at runtime |
| Major | 🟠 | Violates v2 idioms or creates measurable maintenance risk |
| Minor | 🟡 | Readability or low-risk style concern with no correctness impact |
| Pass | ✅ | No issues found in this dimension — always write this explicitly |

### Six Review Dimensions

| Dimension | Name | Primary Focus | Escalate When |
|-----------|------|---------------|---------------|
| 1 | Modern Syntax & v2 Compliance | `= vs :=`, `#Requires`, `%Var%`, MsgBox form, label hotkeys | Always — syntax errors invalidate all other analysis |
| 2 | Error Handling & Type Safety | try/catch on throwing ops, catch surfacing, type guards | Script uses FileRead, ComObject, RegRead, DllCall |
| 3 | Variable Scope & Architecture | globals, data structures, single-responsibility | Script has globals, pseudo-arrays, or long functions |
| 4 | GUI & Event Binding | `.Bind(this)`, `.OnEvent()`, g-labels, SetTimer | Script uses `Gui()`, `SetTimer()`, or class method callbacks |
| 5 | Performance & Resource Management | loop invariants, O(n²) string concat, COM lifecycle | Script has loops with I/O or COM; uses `.=` inside loops |
| 6 | Readability & Composite | naming conventions, docs, duplicate hotkeys, dead code | Script > 50 lines or > 1 hotkey; explicit full-review request |

### Throwing Operations Reference

| Category | Operations that THROW — wrap in try/catch | Operations that DO NOT THROW — no try/catch |
|----------|-------------------------------------------|----------------------------------------------|
| File I/O | `FileRead(path)`, `FileOpen(path, "r")` | — |
| Registry | `RegRead("HKCU\Key", "Val")` | — |
| Regex | `RegExMatch(str, badPattern)`, `RegExReplace(str, badPat)` | — |
| Process | `ProcessGetPath(pid)` | — |
| Network | `Download(url, dest)` | — |
| DLL | `DllCall("lib\fn", ...)` | — |
| COM | `ComObject("ProgID")` | — |
| UI | — | `MsgBox()`, `ToolTip()`, `Sleep()`, `Send()` |
| Win | `WinGetTitle()` — throws `TargetError` if window not found | — |
| String | — | `StrLen(str)` |

### Report Output Format

Every review must follow this exact structure. Dimensions with no issues must still appear with an explicit pass line — never omit a dimension.
```
AHK v2 Code Review Report

Dimension N — [Name]
- [SEVERITY_EMOJI] Line N — Description. Suggested fix (inline code if ≤ 5 lines).
  OR
✅ No issues found.
✅ Not applicable ([reason]).

Summary
2–3 sentences of overall quality assessment.

Priority Action List
1. [Top issue] — one-line rationale. (🔴 Critical)
...5 items ordered by severity then impact
```

### AHK v2 Functions Referenced in Review Patterns

| Function / Method | Notes relevant to review context |
|-------------------|----------------------------------|
| `FileRead(path)` | Throws — requires try/catch; text-only |
| `FileOpen(path, mode, enc)` | Throws — requires try/catch; handle must be `.Close()`d in finally |
| `ComObject("ProgID")` | Throws on HRESULT failure — requires try/catch |
| `RegRead(key, val)` | Throws if key/value absent — requires try/catch |
| `RegExMatch()` / `RegExReplace()` | Throws on malformed pattern — requires try/catch around dynamic patterns |
| `DllCall()` | Throws if DLL not found or signature wrong |
| `Download()` | Throws on network failure |
| `ObjRelease(ptr)` | Correct only for raw pointers from `ComObjQuery()`/`DllCall()` — never for `ComObject()` wrappers |
| `ComObjQuery()` | Returns raw pointer — caller owns `ObjRelease()` responsibility |
| `Type(val)` | Returns exact type string — use for runtime type checks |
| `IsObject(val)` | Returns true for any object — use to distinguish scalar from object |
| `IsInteger(val)` | Returns true if string contains a valid integer — use before `Integer()` cast |
| `Integer(val)` | Throws if val is not a numeric string — guard with `IsInteger()` first |
| `InputBox()` | Returns an object; access `.Value` for the string; `.Result` for button |
| `.Bind(this)` | Mandatory on any class method used as `.OnEvent()` callback — preserves `this` |
| `SetTimer(fnRef, ms)` | Requires a function reference, not a label string |
| `CoordMode(target, mode)` | Global state change — save `A_CoordModeMouse` and restore after affected block |

## AHK V2 CONSTRAINTS

- The `=` operator is comparison in v2, not assignment — using `=` as assignment is always **Critical**: the variable stays `""` with no parse error and no runtime warning; every downstream expression that reads it operates on an empty string.
- Never flag `MsgBox()`, `ToolTip()`, or `Sleep()` as needing try/catch — they do not throw under normal conditions; adding spurious try/catch around them fabricates a non-existent risk and degrades report credibility.
- Every catch block must reference `e.Message` or `e.Extra` — an empty `catch { }` body is always **Critical** because the failure becomes completely invisible and undiagnosable.
- `.Bind(this)` is mandatory on every class method used as a GUI event callback — omitting it makes `this` undefined inside the handler at runtime with no warning (NameError on first property access).
- Duplicate hotkey registrations are always **Critical**: AHK v2 silently uses only the last definition; the first handler is permanently dead code with no diagnostic.
- Arrow function syntax: `(params) => expr` is valid; `(params) => { multi-line }` is a load-time parse error — flag any multi-statement fat-arrow body as **Critical**.
- COM object cleanup: assign wrapper to `""` for `ComObject()` — do not call `ObjRelease()` on it; reserve `ObjRelease()` only for raw pointers from `ComObjQuery()` or `DllCall()` — mixing these causes double-free.
- Evaluate dimensions in order (1 → 6): syntax errors in Dimension 1 can make downstream observations in Dimensions 2–5 invalid; never reorder or skip.
- Dimensions with no findings must still appear in the report with `✅ No issues found.` or `✅ Not applicable (reason).` — silently omitting a dimension causes the reader to assume it was not evaluated.
- Evaluate only what is present in the submitted script — never flag absent features that were not requested (e.g., no GUI cleanup when there is no GUI); mark those dimensions `✅ Not applicable`.
- Code fragments must be noted at the top of the report; limit Dimension 1 checks to visible code and apply Dimensions 2–6 only to constructs present in the fragment.

Safe-access priority order for review triage:
  1. Dimension 1 (`= vs :=`, `#Requires`) — syntax errors invalidate all other analysis; always start here
  2. Dimension 2 (try/catch on throwing operations) — correctness before architecture
  3. Dimension 4 (`.Bind(this)`, `.OnEvent()`) — Critical runtime failures from missing context binding
  4. Dimensions 3, 5, 6 — architecture, performance, and readability in decreasing urgency

Pair every Critical finding with a concrete resolution path:
- ✗ `global counter = 0` — `=` is comparison; `counter` is never assigned — **Critical**
- ✓ `global counter := 0` — `:=` is the assignment operator in v2

- ✗ `catch { }` — empty body swallows exception; failure is invisible — **Critical**
- ✓ `catch (e) { OutputDebug(e.Message) }` — surface the error for diagnosability

## TIER 1 — Modern Syntax & v2 Compliance
> METHODS COVERED: `#Requires` · `:=` assignment · `MsgBox()` · hotkey function form · `%Var%` removal · `Map()`

Dimension 1 evaluates whether the script uses AHK v2 syntax correctly and avoids patterns inherited from v1. The most common Critical issues are using `=` as an assignment operator (a silent no-op in v2 where `=` is comparison only) and `%Var%` percent-dereferencing. These produce no parse error but leave variables permanently uninitialized.
```ahk
; ================================================================
; TIER 1 REVIEW: Modern Syntax & v2 Compliance Checklist
; ================================================================

; CHECK 1: #Requires directive
; ✗ Wrong — script may be silently run by AHK v1 interpreter
; (no directive at top of file)

; ✓ Correct — interpreter version is enforced at load time
#Requires AutoHotkey v2.0

; ----------------------------------------------------------------
; CHECK 2: Assignment operator
; ✗ Wrong — '=' is comparison in v2; counter is never assigned (stays "")
global counter = 0        ; evaluates as boolean expression, discards result
counter = counter + 1     ; same problem; counter stays ""

; ✓ Correct — ':=' is the assignment operator in v2
global counter := 0
counter := counter + 1

; ----------------------------------------------------------------
; CHECK 3: String literals — must be double-quoted
; ✗ Wrong — bare unquoted string (v1 legacy)
MsgBox Hello World        ; parse error or treated as variable reference

; ✓ Correct
MsgBox("Hello World")

; ----------------------------------------------------------------
; CHECK 4: Dynamic variable references — %Var% is a v1 pattern
; ✗ Wrong — %varName% dereferencing in v2 expression context
result := %dynamicVar%    ; AHK v2 handles this differently; avoid entirely

; ✓ Correct — use Map() for dynamic key lookup or direct variable references
lookup := Map("key1", "val1", "key2", "val2")
result := lookup["key1"]

; ----------------------------------------------------------------
; CHECK 5: Function syntax — label-based pseudo-functions are v1
; ✗ Wrong — label-based hotkey (v1 pattern)
^F1::
    DoSomething()
return

; ✓ Correct — function-form hotkey in v2
^F1:: DoSomething()          ; single statement
^F2:: {                      ; multi-statement block
    DoSomething()
    DoOther()
}

; ----------------------------------------------------------------
; CHECK 6: MsgBox call syntax
; ✗ Wrong — v1 command form (parse error in v2 scripts)
; MsgBox, Title, Text       ; this line intentionally commented — causes load error

; ✓ Correct — v2 function call form
MsgBox("Text", "Title")
```

## TIER 2 — Error Handling & Type Safety
> METHODS COVERED: `FileRead()` · `ComObject()` · `RegRead()` · `InputBox()` · `Type()` · `IsObject()` · `IsInteger()` · `Integer()`

Dimension 2 evaluates whether operations that genuinely throw are wrapped in try/catch and whether catch blocks surface error information rather than silently swallowing exceptions. Type validation for externally-sourced data (file content, registry values, user input) is also evaluated here. Consult the Throwing Operations Reference table in API QUICK-REFERENCE for the definitive list.
```ahk
; ================================================================
; TIER 2 REVIEW: Error Handling & Type Safety Checklist
; ================================================================

; CHECK 1: try/catch coverage for throwing operations
; ✗ Wrong — FileRead throws if file is missing; crash is silent to user
ProcessFile(path) {
    content := FileRead(path)    ; 🔴 Critical: unguarded throwing operation
    return content
}

; ✓ Correct — exception is caught and surfaced
ProcessFile(path) {
    try {
        content := FileRead(path)
        return content
    } catch (e) {
        OutputDebug("ProcessFile failed: " e.Message " | Extra: " e.Extra)
        return ""
    }
}

; ----------------------------------------------------------------
; CHECK 2: catch blocks must not swallow exceptions silently
; ✗ Wrong — exception is caught but never logged or re-thrown
try {
    xlApp := ComObject("Excel.Application")
} catch {
    ; (empty catch body) 🔴 Critical: failure is invisible
}

; ✓ Correct — surface e.Message so the failure is diagnosable
try {
    xlApp := ComObject("Excel.Application")
} catch (e) {
    MsgBox("COM init failed: " e.Message, "Error", 0x10)
}

; ----------------------------------------------------------------
; CHECK 3: Type validation for externally-sourced data
; ✗ Wrong — arithmetic on unvalidated registry value (may be "")
regVal := RegRead("HKCU\App", "Timeout")
result := regVal * 1000         ; 🟠 Major: if regVal is "", result is 0 silently

; ✓ Correct — validate before use; IsInteger() checks numeric string content
regVal := ""
try {
    regVal := RegRead("HKCU\App", "Timeout")
} catch {
    regVal := "30"              ; safe default
}
if IsInteger(regVal) {
    result := Integer(regVal) * 1000
} else {
    result := 30000             ; fallback
}

; ----------------------------------------------------------------
; CHECK 4: Null/empty guards for user input and file-sourced values
; ✗ Wrong — no guard; InStr("", needle) returns 0 but later code may break
userInput := InputBox("Enter pattern:").Value
pos := InStr(userInput, "target")

; ✓ Correct — guard empty input before processing
userInput := InputBox("Enter pattern:", "Search").Value
if (userInput = "") {
    return
}
pos := InStr(userInput, "target")

; ----------------------------------------------------------------
; CHECK 5: Type() and IsObject() for runtime type detection
; ✗ Wrong — typeof does not exist in AHK v2 (runtime NameError)
; if (typeof(obj) == "Map") { ... }    ; invalid

; ✓ Correct — use Type() for exact type string
if (Type(obj) = "Map") {
    ; safe to use Map methods
}
; ✓ Also correct — IsObject() distinguishes scalar from object
if IsObject(obj) {
    ; obj is any object type
}
```

## TIER 3 — Variable Scope & Architecture
> METHODS COVERED: `Map()` · `Array()` · `class static` · `FileAppend()` · `for` enumeration

Dimension 3 evaluates whether the script uses appropriate data structures, avoids implicit global coupling, and applies single-responsibility decomposition. The most common Major issues are pseudo-arrays (sequentially named variables used as lists) and functions that both mutate shared state and trigger UI feedback. Cross-reference `Module_Classes.md` for class static property encapsulation patterns.
```ahk
; ================================================================
; TIER 3 REVIEW: Variable Scope & Architecture Checklist
; ================================================================

; CHECK 1: Avoid implicit global coupling — use class static properties
; ✗ Wrong — global mutable state accessed implicitly by multiple functions
global hitCount := 0
global sessionActive := false

IncrementHit() {
    global hitCount             ; 🟠 Major: implicit global coupling
    hitCount += 1
}

; ✓ Correct — encapsulate shared state in a class
class Session {
    static hitCount := 0
    static isActive := false

    static Increment() {
        Session.hitCount += 1
    }
    static Reset() {
        Session.hitCount := 0
        Session.isActive := false
    }
}
Session.Increment()

; ----------------------------------------------------------------
; CHECK 2: Data structure choice — Map() and Array() over pseudo-arrays
; ✗ Wrong — pseudo-array pattern (v1 legacy, not iterable as a collection)
item1 := "apple"
item2 := "banana"
item3 := "cherry"
; 🟠 Major: cannot loop, cannot measure length, cannot serialize

; ✓ Correct — use Array() for ordered collections
items := ["apple", "banana", "cherry"]
for i, item in items {
    OutputDebug(i ": " item)
}

; ✗ Wrong — parallel scalar variables as key-value storage
configHost := "localhost"
configPort := "5432"
configDB   := "prod"

; ✓ Correct — use Map() for key-value storage
config := Map(
    "host", "localhost",
    "port", "5432",
    "db",   "prod"
)
OutputDebug(config["host"])

; ----------------------------------------------------------------
; CHECK 3: Single-responsibility functions
; ✗ Wrong — one function does three unrelated things
UpdateAndNotify(val) {          ; 🟠 Major: ambiguous name, multiple concerns
    global hitCount
    hitCount += val
    if (hitCount > 10)
        MsgBox("Threshold reached!")
    FileAppend(hitCount "`n", "log.txt")
}

; ✓ Correct — three focused functions with descriptive names
IncrementHitCount(delta) {
    Session.hitCount += delta
}

CheckThreshold() {
    if (Session.hitCount > HIT_THRESHOLD)
        NotifyThresholdReached()
}

NotifyThresholdReached() {
    MsgBox("Threshold reached: " Session.hitCount)
}

AppendHitLog(path) {
    try {
        FileAppend(Session.hitCount "`n", path)
    } catch (e) {
        OutputDebug("AppendHitLog failed: " e.Message)
    }
}

; ----------------------------------------------------------------
; CHECK 4: Named constants for magic values
; ✗ Wrong — magic numbers and strings in logic
if (retryCount > 3)             ; 🟡 Minor: what is 3?
    Sleep(5000)                 ; 🟡 Minor: why 5000?

; ✓ Correct — named constants at scope top
MAX_RETRIES   := 3
RETRY_DELAY_MS := 5000

if (retryCount > MAX_RETRIES)
    Sleep(RETRY_DELAY_MS)
```

## TIER 4 — GUI & Event Binding
> METHODS COVERED: `Gui()` · `.Add()` · `.OnEvent()` · `.Bind()` · `.Show()` · `.Destroy()` · `SetTimer()`

Dimension 4 evaluates whether GUI construction uses object-based patterns, event handlers are correctly bound to object context, and resources are cleaned up on GUI destruction. The most common Critical issue is a missing `.Bind(this)` on class method callbacks, which causes `this` to be undefined at runtime with no warning. Cross-reference `Module_GUI.md` for comprehensive control creation patterns.
```ahk
; ================================================================
; TIER 4 REVIEW: GUI & Event Binding Checklist
; ================================================================

; CHECK 1: Object-based GUI construction — store controls as variables
; ✗ Wrong — controls created but not stored; cannot be referenced later
MyGui := Gui()
MyGui.Add("Button", "w120", "Click Me")   ; 🟠 Major: no reference to the button

; ✓ Correct — store control references for later .OnEvent() binding
MyGui := Gui("+Resize", "My App")
MyBtn := MyGui.Add("Button", "w120", "Click Me")
MyEdit := MyGui.Add("Edit", "w200 vUserInput")

; ----------------------------------------------------------------
; CHECK 2: Event binding — always .Bind(this) for class method callbacks
; ✗ Wrong — bare method reference loses 'this' context at call time
class AppController {
    __New() {
        this.ui := Gui()
        this.btn := this.ui.Add("Button",, "Run")
        this.btn.OnEvent("Click", this.OnClick)   ; 🔴 Critical: this is unbound
    }
    OnClick(ctrl, info) {
        MsgBox(this.status)     ; 'this' is undefined — runtime NameError
    }
}

; ✓ Correct — bind the method to preserve object context
class AppController {
    status := "ready"

    __New() {
        this.ui := Gui()
        this.btn := this.ui.Add("Button",, "Run")
        this.btn.OnEvent("Click", this.OnClick.Bind(this))  ; ✓ context preserved
        this.ui.Show()
    }

    OnClick(ctrl, info) {
        MsgBox(this.status)     ; 'this' correctly refers to AppController instance
    }
}
app := AppController()

; ----------------------------------------------------------------
; CHECK 3: Fat-arrow callbacks — only valid for single-line expressions
; ✗ Wrong — arrow fn with block body is invalid AHK v2 syntax (load-time error)
; MyBtn.OnEvent("Click", (*) => {   ; 🔴 Critical: invalid syntax
;     DoSomething()
;     DoOther()
; })

; ✓ Correct — single-expression arrow fn (no this access needed)
MyBtn.OnEvent("Click", (*) => MsgBox("Clicked"))

; ✓ Correct — multi-statement: define a named nested function
HandleBtnClick(ctrl, info) {
    DoSomething()
    DoOther()
}
MyBtn.OnEvent("Click", HandleBtnClick)

; ----------------------------------------------------------------
; CHECK 4: v1 g-label patterns — do not exist in v2
; ✗ Wrong — g-label syntax (v1 only; parse error or silent fail in v2)
; MyBtn := MyGui.Add("Button", "gMyAction w120", "Run")  ; 🔴 Critical: v1 pattern

; ✓ Correct — use .OnEvent() exclusively in v2
MyBtn.OnEvent("Click", HandleBtnClick)

; ----------------------------------------------------------------
; CHECK 5: SetTimer — function reference, not label name
; ✗ Wrong — SetTimer with label name (v1 pattern; NameError in v2)
; SetTimer, UpdateDisplay, 1000    ; 🔴 Critical: v1 syntax

; ✓ Correct — pass a function reference (or bound method)
UpdateDisplay() {
    ToolTip(A_Now)
}
SetTimer(UpdateDisplay, 1000)

; ----------------------------------------------------------------
; CHECK 6: Resource cleanup on GUI destruction
; ✗ Wrong — OnClose handler fires but GUI object lingers; handle leak possible
MyGui.OnEvent("Close", (*) => ExitApp())    ; 🟠 Major if other cleanup required

; ✓ Correct — explicitly destroy GUI before exit in long-running scripts
MyGui.OnEvent("Close", (*) => (MyGui.Destroy(), ExitApp()))
```

## TIER 5 — Performance & Resource Management
> METHODS COVERED: `FileRead()` · `FileOpen()` · `.WriteLine()` · `.Close()` · `ComObject()` · `ObjRelease()` · `ComObjQuery()` · `CoordMode()` · `A_CoordModeMouse` · `Map.Has()`

Dimension 5 evaluates loop efficiency, string construction patterns, COM object lifecycle, and the correct restoration of global interpreter state. The most common Critical issue is `FileRead()` or COM object creation inside a tight loop when the result is loop-invariant. String concatenation with `.=` inside loops is O(n²) and must be replaced with Array accumulation or direct `FileAppend()` per item.
```ahk
; ================================================================
; TIER 5 REVIEW: Performance & Resource Management Checklist
; ================================================================

; CHECK 1: Loop-invariant operations — hoist above the loop
; ✗ Wrong — file is read on every iteration (100× disk I/O per hotkey press)
^F1:: {
    total := 0
    Loop 100 {
        data := FileRead("config.txt")    ; 🔴 Critical: loop-invariant, hoist it
        total += InStr(data, "hit") ? 1 : 0
    }
    MsgBox(total)
}

; ✓ Correct — read once, reuse the cached value
^F1:: {
    try {
        data := FileRead("config.txt")
    } catch (e) {
        MsgBox("Read failed: " e.Message)
        return
    }
    total := 0
    Loop 100 {
        total += InStr(data, "hit") ? 1 : 0
    }
    MsgBox(total)
}

; ----------------------------------------------------------------
; CHECK 2: String concatenation in loops — O(n²) anti-pattern
; ✗ Wrong — .= inside loop copies the growing string on every iteration
BuildReport(items) {
    report := ""
    for item in items {
        report .= item "`n"    ; 🟠 Major: O(n²) — avoid for large collections
    }
    return report
}

; ✓ Correct — accumulate in Array, join in a single pass after loop
BuildReport(items) {
    parts := []
    for item in items {
        parts.Push(item)
    }
    ; AHK v2 has no built-in Array.Join(); construct manually or write to file
    result := ""
    for part in parts {
        result .= part "`n"
    }
    return result
}

; ✓ Correct alternative for file-write scenarios — FileAppend each part directly
WriteReport(items, path) {
    try {
        f := FileOpen(path, "w", "UTF-8")
        for item in items {
            f.WriteLine(item)
        }
        f.Close()
    } catch (e) {
        OutputDebug("WriteReport failed: " e.Message)
    }
}

; ----------------------------------------------------------------
; CHECK 3: COM object lifecycle — correct release patterns
; ✗ Wrong — ObjRelease() on a ComObject() wrapper (double-free risk)
xlApp := ComObject("Excel.Application")
; ... use xlApp ...
ObjRelease(xlApp)    ; 🔴 Critical: incorrect for ComObject() wrapper

; ✓ Correct — assign to "" to trigger AHK v2 reference-count cleanup
xlApp := ComObject("Excel.Application")
; ... use xlApp ...
xlApp := ""          ; reference count drops to 0; COM object released automatically

; Reserve ObjRelease() only for raw interface pointers from ComObjQuery()/DllCall()
rawPtr := ComObjQuery(xlApp, "{IID-GUID}")
; ... use rawPtr via DllCall ...
ObjRelease(rawPtr)   ; ✓ correct for raw pointer

; ----------------------------------------------------------------
; CHECK 4: Global state directives — restore after affected block
; ✗ Wrong — CoordMode altered globally; affects all subsequent hotkeys
^F2:: {
    CoordMode("Mouse", "Screen")    ; 🟠 Major: leaks into global state
    Click(100, 200)
}

; ✓ Correct — save and restore, or scope the directive to one action
^F2:: {
    prevMode := A_CoordModeMouse    ; save current mode
    CoordMode("Mouse", "Screen")
    Click(100, 200)
    CoordMode("Mouse", prevMode)    ; restore
}

; ----------------------------------------------------------------
; CHECK 5: O(1) lookup — prefer Map() over repeated linear search
; ✗ Wrong — O(n) search on every lookup call
IsValidCommand(cmd) {
    validCmds := ["open", "save", "close", "quit", "refresh"]
    for c in validCmds {
        if (c = cmd)
            return true
    }
    return false
}

; ✓ Correct — O(1) Map lookup after one-time construction
VALID_COMMANDS := Map(
    "open", true, "save", true, "close", true,
    "quit", true, "refresh", true
)
IsValidCommand(cmd) {
    return VALID_COMMANDS.Has(cmd)
}

; ----------------------------------------------------------------
; CHECK 6: Pattern/COM caching — avoid recomputing per-call in hot paths
; ✓ Correct — cache compiled pattern in static property; compute only once
class PatternCache {
    static emailRx := ""

    static GetEmailPattern() {
        if (PatternCache.emailRx = "")
            PatternCache.emailRx := "[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}"
        return PatternCache.emailRx
    }
}

ValidateEmail(addr) {
    return RegExMatch(addr, PatternCache.GetEmailPattern())
}
```

### Performance Notes

**Review depth vs. script size:** For short scripts (&lt;50 lines), a full six-dimension pass is appropriate. For long scripts (200+ lines), triage Critical and Major findings first, then Minor. For 500+ line scripts, spot-check Dimension 1 across 20% of lines plus all hotkey definitions; focus Dimension 3 on class/function decomposition and global count; identify every loop body for Dimension 5 invariant checks. Suggest decomposing into modules if function count exceeds 20.

**Fragment review:** Note "Fragment review" at the top of the report; apply Dimension 1 to all visible code; apply Dimensions 2–6 only to constructs present in the fragment.

**Dimension line ownership:** Avoid re-evaluating the same line under multiple dimensions. Each line has exactly one primary dimension owner — assign a finding to the most specific applicable dimension to prevent inflation of the Priority Action List.

**O(1) vs O(n) tradeoffs:** String concatenation with `.=` inside a loop of n items costs O(n²) total copies — flag as Major for any loop > ~20 iterations. Map-based lookups in hot paths replace O(n) linear scans with O(1) — always prefer `Map.Has()` over iterating an Array to find a value.

**COM creation cost:** `ComObject()` construction involves COM marshalling overhead — never call it inside a frequently-executed loop or hotkey handler body; hoist to module-level or class static property initialized once.

## TIER 6 — Readability, Maintainability & Composite Review
> METHODS COVERED: `WinGetTitle()` · `WinGetPID()` · `OutputDebug()` · `RegExMatch()` · `WinActive()`

Dimension 6 evaluates naming conventions, documentation quality, file structure, dead code, and duplicate hotkey registrations. The Critical issue unique to this tier is duplicate hotkey definitions: AHK v2 silently uses only the last definition, making the conflict completely invisible. Composite review at this tier synthesizes findings from all five previous dimensions into a prioritized action list.
```ahk
; ================================================================
; TIER 6 REVIEW: Readability, Maintainability & Composite Checklist
; ================================================================

; CHECK 1: Naming conventions
; ✗ Wrong — cryptic or mis-cased names
x := 0                          ; 🟡 Minor: non-descriptive single-letter
tmp2 := GetData()               ; 🟡 Minor: non-descriptive with index suffix
class myDataManager { }         ; 🟠 Major: class should be PascalCase
MAX_size := 100                 ; 🟡 Minor: constant should be UPPER_SNAKE_CASE

; ✓ Correct — AHK v2 naming conventions
retryCount := 0                 ; camelCase for variables
rawApiResponse := GetData()     ; camelCase, descriptive
class DataManager { }           ; PascalCase for classes
MAX_RETRY_COUNT := 100          ; UPPER_SNAKE_CASE for constants

; ----------------------------------------------------------------
; CHECK 2: Function documentation — header block for public functions
; ✗ Wrong — no documentation; callers must read implementation
GetWindowInfo() {
    return Map("title", WinGetTitle("A"), "pid", WinGetPID("A"))
}

; ✓ Correct — header describes purpose, params, return shape
; GetWindowInfo()
;   Returns a Map with keys "title" (String) and "pid" (Integer)
;   representing the active window's title and process ID.
;   Throws: nothing — returns empty Map on WinGet failure.
GetWindowInfo() {
    try {
        return Map("title", WinGetTitle("A"), "pid", WinGetPID("A"))
    } catch {
        return Map()
    }
}

; ----------------------------------------------------------------
; CHECK 3: File structure — header block at top of script
; ✓ Correct file header template
;
; ============================================================
; Script:      MyTool.ahk
; Purpose:     Automates clipboard history and window layout
; Author:      [name]
; Version:     1.0.0  2025-01-15  Initial release
;              1.1.0  2025-03-02  Added multi-monitor support
; Dependencies: Module_Classes.md patterns, AHK v2.0.11+
; ============================================================

; ----------------------------------------------------------------
; CHECK 4: Dead code — flag and remove
; ✗ Wrong — large commented-out block, debug MsgBox outside debug mode
; OldProcessData(x) {          ; 🟡 Minor: remove or version-control instead
;     ...
; }
MsgBox("DEBUG: val=" currentVal)    ; 🟡 Minor: debug output left in production

; ✓ Correct — use a debug mode flag
DEBUG_MODE := false
DebugLog(msg) {
    if DEBUG_MODE
        OutputDebug("[DEBUG] " msg)
}

; ----------------------------------------------------------------
; CHECK 5: Duplicate hotkey registrations — Critical silent conflict
; ✗ Wrong — both definitions exist; only the second takes effect; silent conflict
^F1:: MsgBox("Action A")    ; 🔴 Critical: overridden by definition below
; ... many lines later ...
^F1:: MsgBox("Action B")    ; only this one fires; developer is unaware

; ✓ Correct — each key combination registered exactly once
^F1:: {
    ; unified handler — dispatch based on context if needed
    if WinActive("ahk_class Notepad")
        HandleNotepadF1()
    else
        HandleDefaultF1()
}

; ----------------------------------------------------------------
; COMPOSITE: Producing the Priority Action List
; After all six dimensions, rank findings: Critical first, then Major by impact
;
; Example Priority Action List structure:
;
; Priority Action List
; 1. Replace = with := on lines 3, 7, 12 — in v2 '=' is comparison; variables
;    stay uninitialized and all downstream logic is silently wrong. (🔴 Critical)
; 2. Wrap FileRead() in try/catch (line 24) — missing file crashes the script. (🔴)
; 3. Hoist FileRead() above Loop block (line 24) — current code reads disk 100×
;    per hotkey press; hoist and cache the result. (🔴 Critical)
; 4. Rewrite ^F1 hotkey as function form — label-based hotkeys are v1. (🟠 Major)
; 5. Encapsulate 'counter' global into Session.hitCount class property. (🟠 Major)
```

## ANTI-PATTERNS

| Pattern | Wrong | Correct | LLM Common Cause |
|---------|-------|---------|------------------|
| Fabricating throwing operations | `🔴 MsgBox can throw — wrap in try/catch` | `✅ MsgBox does not throw — no try/catch needed` | General "defensive programming" training data causes LLMs to add try/catch to every call, not only genuinely throwing operations |
| `=` treated as Minor style issue | `🟡 Use := for clarity` | `🔴 = is comparison, not assignment — variable stays ""` | AHK v1 training data mixed with v2; LLMs familiar with languages where `=` assigns may not recognize the semantic reversal |
| Missing `.Bind(this)` on callbacks | `this.btn.OnEvent("Click", this.OnClick)` | `this.btn.OnEvent("Click", this.OnClick.Bind(this))` | JavaScript callback training data does not always require explicit binding; cross-language habit suppresses the warning |
| `ObjRelease()` on wrapper | `ObjRelease(xlApp)` after `ComObject()` | `xlApp := ""` for wrapper; `ObjRelease(rawPtr)` only for `ComObjQuery()` result | Manual memory management patterns from C++/C training data; LLMs conflate raw pointer lifecycle with AHK's reference-counted wrappers |
| Fat-arrow with block body | `MyBtn.OnEvent("Click", (*) => { DoA()`\n`DoB() })` | Named function + `MyBtn.OnEvent("Click", namedFn)` | JavaScript arrow function training allows block bodies; AHK v2 restricts fat-arrow to a single expression |
| Skipping "✅ No issues found" | Omit dimension section when it has no findings | `✅ No issues found.` or `✅ Not applicable (no GUI present).` | LLMs tend to omit sections with nothing to say; report completeness is not inherent in generation without an explicit formatting constraint |
| Flagging absent features | `🟠 Missing .Destroy() call` when script has no GUI | `✅ Not applicable — no GUI present` | Pattern-matching on known review items without first checking whether the relevant construct exists in the submitted code |

## SEE ALSO

> This module does NOT cover: AHK v2 syntax rules, idiomatic API usage, or language semantics — those are in Module_Instructions.md and the domain-specific knowledge modules.
> This module does NOT cover: try/catch error class hierarchy, typed exception matching, or `OnError()` global handlers → see Module_Errors.md.
> This module does NOT cover: GUI control creation patterns, layout management, or control property APIs → see Module_GUI.md.
> This module does NOT cover: class inheritance, meta-functions (`__Get`/`__Set`/`__Call`), or prototype chains → see Module_Classes.md.

- `Module_Instructions.md` — master AHK v2 syntax rules, idiomatic patterns, and the baseline against which Dimension 1 compliance is measured; load this module first for any general AHK v2 question.
- `Module_Errors.md` — typed error classes, `OnError()` global handler, and try/catch patterns for Dimension 2 deep-dives; cross-reference when evaluating COM or file I/O error handling.
- `Module_GUI.md` — comprehensive control creation, layout, and `.OnEvent()` binding patterns for Dimension 4 evaluation; cross-reference when the submitted script defines class-based GUI controllers.
- `Module_Classes.md` — class static property encapsulation, inheritance, and meta-function patterns for Dimension 3 architecture evaluation; cross-reference when reviewing scripts with multiple classes or OOP-heavy designs.