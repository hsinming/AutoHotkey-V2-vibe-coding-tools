# Module_CodeReview.md
<!-- DOMAIN: Code Review -->
<!-- SCOPE: AHK v2 syntax rules, API semantics, and domain-specific patterns are not defined here — they are enforced by Module_Instructions.md and the domain modules; this module provides only the six-dimension evaluation framework and severity-graded output format for reviewing submitted scripts. -->
<!-- TRIGGERS: review, audit, "code review", "check my script", "find bugs", "quality check", "what's wrong with", "is this v2 compliant", "best practices check", "is my error handling correct", "why does my script crash", "refactor review", "clean code", "over-engineering", "YAGNI", "too complex", "refactor", "naming convention", "comment quality", "abstraction", "design pattern", MsgBox(), SetTimer(), ComObject(), .Bind(), .OnEvent() -->
<!-- CONSTRAINTS: `=` used as assignment is always Critical — it is comparison in v2 and silently leaves the variable as ""; never flag MsgBox(), ToolTip(), or Sleep() as needing try/catch — they do not throw; every catch block must surface e.Message or e.Extra — an empty catch body is always Critical. Clean Code violations (comments that restate code, negative naming, side-effecting Get* functions) are always Major — they signal systemic maintenance debt, not style preference. Over-Engineering is always Major when an abstraction layer has no current caller or when a design pattern appears in a script under 300 lines without documented justification — flag these before architectural concerns. -->
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
| 3 | Variable Scope & Architecture | globals, data structures, single-responsibility, function length, parameter count, cyclomatic complexity | Script has globals, pseudo-arrays, long functions, or deep nesting |
| 4 | GUI & Event Binding | `.Bind(this)`, `.OnEvent()`, g-labels, SetTimer | Script uses `Gui()`, `SetTimer()`, or class method callbacks |
| 5 | Performance, Security & Resource Management | loop invariants, O(n²) string concat, COM lifecycle, input validation, DllCall injection | Script has loops with I/O or COM; uses `.=` inside loops; passes user input to DllCall/RegWrite/RegExMatch |
| 6 | **Clean Code, Over-Engineering & Composite** | **comment quality, naming contracts, YAGNI, over-abstraction, over-defensive guards, duplicate hotkeys, dead code** | **Always run last — synthesises all dimensions; flag OE and Clean Code violations even in short scripts** |

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

**Clean Code — mandatory constraints:**
- `Get*` / `Fetch*` / `Read*` prefixed functions must be side-effect-free — any function with one of these prefixes that mutates state or triggers I/O is **Major**: callers cannot reason about safety of repeated calls.
- `Check*` / `Is*` / `Has*` prefixed functions must return a boolean and must not mutate state — a `Check*` that modifies a variable is **Major**: it violates the Principle of Least Surprise.
- Comments must explain *why*, not *what* — a comment that merely restates the code in English (e.g., `; add 1 to counter` above `counter += 1`) is always **Minor**; it adds noise without information.
- Negative boolean naming is always **Minor** at minimum: `isNotReady`, `!isDisabled`, `notFound` force double-negation reasoning — always rename to the affirmative form.
- Magic strings (literal registry paths, repeated window class names, status codes) carry the same severity as magic numbers — **Minor** when appearing once, **Major** when the same literal appears in 3+ locations; use named constants.

**Over-Engineering — mandatory constraints:**
- An abstraction layer (class, factory, wrapper function) with exactly zero callers other than its own test is always **Major** — flag as YAGNI; the correct action is deletion, not refactoring.
- Design patterns (Strategy, Observer, Factory, Decorator) in scripts under 300 lines require a documented justification comment; absence of justification is **Major** — the pattern is presumed over-engineered until proven otherwise.
- Defensive type guards inside internal helper functions (functions called only from within the same script, never exposed as a public API) are **Minor** over-engineering when the calling context already guarantees the type — every validation layer adds cognitive load without safety benefit.
- A class with a single method that could be a standalone function is always **Minor** over-engineering; a class with a single method that wraps one AHK built-in call is **Major** — flag both for simplification.

Safe-access priority order for review triage:
  1. Dimension 1 (`= vs :=`, `#Requires`) — syntax errors invalidate all other analysis; always start here
  2. Dimension 2 (try/catch on throwing operations) — correctness before architecture
  3. Dimension 4 (`.Bind(this)`, `.OnEvent()`) — Critical runtime failures from missing context binding
  4. Dimension 6 Clean Code + Over-Engineering checks — run before Dims 3 and 5 when the script shows naming confusion or pattern overuse, because OE findings change the scope of architectural review
  5. Dimensions 3, 5 — architecture, performance in decreasing urgency

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

## TIER 3 — Variable Scope, Architecture & SE Design Principles
> METHODS COVERED: `Map()` · `Array()` · `class static` · `FileAppend()` · `for` enumeration · `StrSplit()` · `RegExMatch()`

Dimension 3 evaluates data structures, implicit global coupling, single-responsibility decomposition, function length, parameter count, and cyclomatic complexity. The most common Major issues are pseudo-arrays and functions that violate the 40-line / 4-nesting-depth boundaries. Cross-reference `Module_Classes.md` for class static property encapsulation patterns.
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

; ----------------------------------------------------------------
; CHECK 5: Function length — single-responsibility upper bound
; ✗ Wrong — function exceeds 40 lines and mixes I/O, logic, and UI
ProcessEverything(path) {       ; 🟠 Major: > 40 lines, 3+ responsibilities
    ; ... reads file, parses content, updates GUI, logs result ...
}

; ✓ Correct — each function ≤ 40 lines with one clearly named responsibility
ReadConfig(path) {              ; I/O only — throws on failure, caller handles
    try {
        return FileRead(path, "UTF-8")
    } catch (e) {
        throw Error("ReadConfig failed: " e.Message)
    }
}

ParseConfigLines(raw) {         ; parsing only — pure function, no I/O
    result := Map()
    for line in StrSplit(raw, "`n") {
        if RegExMatch(line, "^(\w+)=(.+)$", &m)
            result[m[1]] := m[2]
    }
    return result
}

ApplyConfig(cfg) {              ; UI only — reads Map, updates controls
    MyEdit.Value := cfg.Get("host", "localhost")
}

; Agent rule: function body > 40 lines → 🟠 Major; > 80 lines → 🔴 Critical
;             (exception: pure data tables / lookup Maps with no logic)

; ----------------------------------------------------------------
; CHECK 6: Parameter count — more than 4 signals missing abstraction
; ✗ Wrong — positional parameters force callers to remember argument order
SendNotification(to, from, subject, body, cc, bcc) {   ; 🟠 Major: 6 params
    ; caller must know: SendNotification("a","b","c","d","","f")
}

; ✓ Correct — group into a Map parameter object; callers use named keys
SendNotification(params) {
    ; params := Map("to","...","subject","...","body","...")
    to      := params.Get("to", "")
    subject := params.Get("subject", "(no subject)")
    body    := params.Get("body", "")
    ; optional keys absent = safe defaults via .Get()
}

; Agent rule: > 4 parameters → 🟠 Major — suggest Map parameter object
;             > 6 parameters → 🔴 Critical — caller API is unmaintainable

; ----------------------------------------------------------------
; CHECK 7: Cognitive complexity — nesting depth and early-return opportunities
; ✗ Wrong — 4 levels of nesting; reader must hold all conditions simultaneously
ProcessItems(items) {
    for item in items {                     ; depth 1
        if (item.type = "A") {             ; depth 2   🟠 Major: nesting ≥ 4
            if (item.active) {             ; depth 3
                Loop item.count {          ; depth 4
                    if (A_Index > 5) {     ; depth 5 → Critical
                        HandleItem(item)
                    }
                }
            }
        }
    }
}

; ✓ Correct — early return flattens logic; each condition is independently readable
ProcessItems(items) {
    for item in items
        ProcessSingleItem(item)
}

ProcessSingleItem(item) {
    if (item.type != "A" || !item.active)   ; guard: skip ineligible items
        return
    Loop item.count {
        if (A_Index > 5)
            HandleItem(item)
    }
}

; Agent rule: nesting depth > 3 → 🟠 Major — apply early-return flattening
;             nesting depth > 4 → 🔴 Critical — extract inner block to named function
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

## TIER 5 — Performance, Security & Resource Management
> METHODS COVERED: `FileRead()` · `FileOpen()` · `.WriteLine()` · `.Close()` · `ComObject()` · `ObjRelease()` · `ComObjQuery()` · `CoordMode()` · `A_CoordModeMouse` · `Map.Has()` · `DllCall()` · `RegExMatch()` · `RegWrite()` · `InputBox()`

Dimension 5 evaluates loop efficiency, string construction patterns, COM object lifecycle, restoration of global interpreter state, and input sanitisation before external API calls. The most common Critical issues are `FileRead()` or COM object creation inside a tight loop and unvalidated user input passed directly to `DllCall()`, `RegWrite()`, or dynamic `RegExMatch()`.
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

; ----------------------------------------------------------------
; CHECK 7: User input passed to DllCall / file path construction — path traversal
; ✗ Wrong — user-supplied string injected directly into a DllCall path argument
userPath := InputBox("Enter path:").Value
DllCall("DeleteFileW", "Str", userPath)   ; 🔴 Critical: path traversal risk
; → attacker enters "..\..\Windows\System32\kernel32.dll"

; ✓ Correct — validate path before use: regex whitelist + existence check
userPath := InputBox("Enter path:").Value
if !RegExMatch(userPath, "^[A-Za-z]:\\[\w\\.\-]+$") {
    MsgBox("Invalid path format.", "Security Error", 0x10)
    return
}
if !FileExist(userPath) {
    MsgBox("File not found.", "Error", 0x10)
    return
}
DllCall("DeleteFileW", "Str", userPath)   ; ✓ validated before use

; Agent rule: any DllCall/FileRead/FileOpen/FileAppend/FileDelete whose path
;             argument originates from InputBox, RegRead, or IniRead →
;             🔴 Critical if no prior regex whitelist or existence check

; ----------------------------------------------------------------
; CHECK 8: Dynamic RegEx pattern from user input — malformed pattern throws
; ✗ Wrong — user-supplied pattern without try/catch; malformed input crashes
userPattern := InputBox("Enter search pattern:").Value
if RegExMatch(data, userPattern)      ; 🔴 Critical: unhandled throw on bad pattern
    MsgBox("Found")

; ✓ Correct — always wrap dynamic RegExMatch/RegExReplace in try/catch
userPattern := InputBox("Enter search pattern:").Value
try {
    matched := RegExMatch(data, userPattern)
} catch (e) {
    MsgBox("Invalid pattern: " e.Message, "Regex Error", 0x10)
    return
}
if matched
    MsgBox("Found")

; Agent rule: RegExMatch/RegExReplace where the pattern variable was not a
;             compile-time literal string → 🔴 Critical if not in try/catch

; ----------------------------------------------------------------
; CHECK 9: Registry key built from external input — injection risk
; ✗ Wrong — registry subkey path constructed from unvalidated user input
userKey := InputBox("Enter setting name:").Value
RegWrite("value", "REG_SZ", "HKCU\MyApp\" userKey, "Data")   ; 🔴 Critical

; ✓ Correct — whitelist allowed key names before constructing path
ALLOWED_SETTINGS := Map("theme", true, "lang", true, "timeout", true)
userKey := InputBox("Enter setting name:").Value
if !ALLOWED_SETTINGS.Has(userKey) {
    MsgBox("Unknown setting: " userKey, "Error", 0x10)
    return
}
RegWrite("value", "REG_SZ", "HKCU\MyApp\" userKey, "Data")   ; ✓ whitelisted
```

### Performance Notes

**Review depth vs. script size:** For short scripts (&lt;50 lines), a full six-dimension pass is appropriate. For long scripts (200+ lines), triage Critical and Major findings first, then Minor. For 500+ line scripts, spot-check Dimension 1 across 20% of lines plus all hotkey definitions; focus Dimension 3 on class/function decomposition and global count; identify every loop body for Dimension 5 invariant checks. Suggest decomposing into modules if function count exceeds 20.

**Fragment review:** Note "Fragment review" at the top of the report; apply Dimension 1 to all visible code; apply Dimensions 2–6 only to constructs present in the fragment.

**Dimension line ownership:** Avoid re-evaluating the same line under multiple dimensions. Each line has exactly one primary dimension owner — assign a finding to the most specific applicable dimension to prevent inflation of the Priority Action List.

**O(1) vs O(n) tradeoffs:** String concatenation with `.=` inside a loop of n items costs O(n²) total copies — flag as Major for any loop > ~20 iterations. Map-based lookups in hot paths replace O(n) linear scans with O(1) — always prefer `Map.Has()` over iterating an Array to find a value.

**COM creation cost:** `ComObject()` construction involves COM marshalling overhead — never call it inside a frequently-executed loop or hotkey handler body; hoist to module-level or class static property initialized once.

**Over-Engineering review depth:** OE findings in Dimension 6 reduce the scope of architectural review in Dimension 3. If Dim 6 flags a class as YAGNI, do not generate Dim 3 refactoring suggestions for that class — flag it once for deletion and move on. This prevents Priority Action List inflation from reviewing code that should not exist.

**Clean Code review depth:** When a script has more than 5 comment-restates-code violations, do not list each individually — report "pervasive comment noise throughout; see lines X, Y, Z for representative examples" as a single Major finding. Listing all 20 occurrences inflates the report without adding actionable value.

## TIER 6 — Clean Code, Over-Engineering & Composite Review
> METHODS COVERED: `WinGetTitle()` · `WinGetPID()` · `OutputDebug()` · `RegExMatch()` · `WinActive()`

Dimension 6 is the highest-priority readability tier. It evaluates Clean Code principles (comment quality, naming contracts, magic strings, negative naming) and Over-Engineering signals (YAGNI, unnecessary abstraction, over-defensive guards, inappropriate design patterns), then synthesises all five prior dimensions into a prioritised action list. Clean Code and Over-Engineering findings are **Major** by default — they signal systemic maintenance debt, not style preference. The Critical issue unique to this tier is duplicate hotkey definitions: AHK v2 silently uses only the last definition.
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

; ================================================================
; CLEAN CODE CHECKS (always Major unless stated otherwise)
; ================================================================

; ----------------------------------------------------------------
; CHECK 6: Comment quality — explain WHY, not WHAT
; ✗ Wrong — comment restates the code; adds noise, no information
counter := counter + 1          ; 🟡 Minor: "add 1 to counter" says nothing new
items.Push(newItem)             ; 🟡 Minor: "push newItem to items" is redundant

; ✗ Wrong — block comment explains the obvious algorithm step
; Loop through each item in the list and check if it matches
for item in items {
    if (item = target)
        return true
}

; ✓ Correct — comment explains non-obvious intent, constraint, or trade-off
counter := counter + 1          ; debounce: first event always accepted; rest throttled
items.Push(newItem)             ; deferred: actual DB write happens in FlushQueue()

; ✓ Correct — comment explains WHY this approach was chosen over the obvious one
; Linear scan intentional: list always < 10 items; Map overhead not justified here
for item in items {
    if (item = target)
        return true
}

; Agent rule: comment that can be deleted without losing any information →
;             🟡 Minor; comment that actively misleads (stale, wrong) → 🟠 Major

; ----------------------------------------------------------------
; CHECK 7: Naming contracts — Get*/Check*/Is* prefix semantics
; ✗ Wrong — GetUser() writes to a log file; callers cannot safely call it twice
GetUser(id) {
    FileAppend(id "`n", "access.log")   ; 🟠 Major: side effect in a Get* function
    return users.Get(id, "")
}

; ✗ Wrong — CheckThreshold() modifies state; callers expect a pure predicate
CheckThreshold() {
    result := Session.hitCount > MAX_RETRIES
    Session.hitCount := 0           ; 🟠 Major: Check* must not mutate state
    return result
}

; ✓ Correct — Get* reads only; side-effecting operation gets a verb that signals mutation
GetUser(id) {
    return users.Get(id, "")        ; pure read — safe to call multiple times
}

LogAccess(id) {
    FileAppend(id "`n", "access.log")   ; verb "Log" signals mutation
}

IsThresholdExceeded() {
    return Session.hitCount > MAX_RETRIES   ; pure predicate — no state change
}

ResetSession() {
    Session.hitCount := 0           ; verb "Reset" signals mutation explicitly
}

; ----------------------------------------------------------------
; CHECK 8: Negative boolean naming — always rename to affirmative form
; ✗ Wrong — double negation forces mental inversion on every read
if !isNotReady              ; 🟡 Minor: reader must compute "not (not ready)"
    StartProcess()

isDisabled := true
if !isDisabled              ; 🟡 Minor: "not disabled" = "enabled" — use isEnabled

; ✓ Correct — affirmative names read as plain English
if isReady
    StartProcess()

isEnabled := true
if isEnabled
    DoWork()

; ----------------------------------------------------------------
; CHECK 9: Magic strings — same severity as magic numbers
; ✗ Wrong — registry path and window class name appear inline 3+ times
RegWrite("1", "REG_SZ", "HKCU\Software\MyApp", "Enabled")
RegRead("HKCU\Software\MyApp", "Timeout")          ; 🟠 Major: duplicated path
WinActivate("ahk_class Notepad")
WinWait("ahk_class Notepad", , 3)                  ; 🟠 Major: duplicated class

; ✓ Correct — named constants at script top; one authoritative definition
APP_REGISTRY_KEY := "HKCU\Software\MyApp"
NOTEPAD_CLASS    := "ahk_class Notepad"

RegWrite("1", "REG_SZ", APP_REGISTRY_KEY, "Enabled")
RegRead(APP_REGISTRY_KEY, "Timeout")
WinActivate(NOTEPAD_CLASS)
WinWait(NOTEPAD_CLASS, , 3)

; Agent rule: string literal appearing in 3+ locations → 🟠 Major;
;             string literal in 1–2 locations → 🟡 Minor (suggest constant)

; ================================================================
; OVER-ENGINEERING CHECKS (Major unless stated otherwise)
; ================================================================

; ----------------------------------------------------------------
; CHECK 10: YAGNI — abstraction with no current caller
; ✗ Wrong — factory class built for "future extensibility"; only one type ever created
class FileReaderFactory {                   ; 🟠 Major: YAGNI
    static Create(type) {
        return type = "text" ? TextFileReader() : BinaryFileReader()
    }
}
class TextFileReader {
    Read(path) { return FileRead(path, "UTF-8") }
}
; Actual usage in the entire script:
content := FileReaderFactory.Create("text").Read("data.txt")

; ✓ Correct — use the simplest form that satisfies current requirements
content := FileRead("data.txt", "UTF-8")

; ✗ Wrong — extensible plugin architecture for a 150-line hotkey script
class PluginManager {                       ; 🟠 Major: 3 classes to call one function
    static plugins := Map()
    static Register(name, plugin) { PluginManager.plugins[name] := plugin }
    static Execute(name) { PluginManager.plugins[name].Run() }
}

; ✓ Correct — direct Map of function references if dispatch is genuinely needed
handlers := Map("open", OpenHandler, "save", SaveHandler)
handlers[action]()

; Agent rule: class/factory/manager with only one concrete subclass or one caller
;             → 🟠 Major — flag as YAGNI; recommend direct call or Map dispatch

; ----------------------------------------------------------------
; CHECK 11: Over-defensive internal guards
; ✗ Wrong — internal helper already guaranteed to receive an Array by its only caller
BuildSummary(items) {
    if !IsObject(items)             ; 🟡 Minor: caller always passes Array
        throw TypeError("Expected Array")
    if (items.Length = 0)           ; 🟡 Minor: empty array is valid; just returns ""
        return ""
    if (Type(items) != "Array")     ; 🟡 Minor: redundant — IsObject already checked
        throw TypeError("Expected Array, got " Type(items))
    ; ... actual logic ...
}

; ✓ Correct — validate at the public entry point only; internal functions trust callers
ProcessUserCommand(rawInput) {      ; ← public boundary: validate here
    if (Type(rawInput) != "String" || rawInput = "")
        throw ValueError("rawInput must be non-empty String")
    items := ParseCommand(rawInput)
    return BuildSummary(items)      ; internal call — no re-validation needed
}

BuildSummary(items) {               ; internal — trust that caller passed valid Array
    result := ""
    for item in items
        result .= item "`n"
    return result
}

; Agent rule: type guard inside a function called only from within the same script,
;             where the calling context already guarantees the type →
;             🟡 Minor (one guard) or 🟠 Major (3+ redundant guards in one function)

; ----------------------------------------------------------------
; CHECK 12: Inappropriate design patterns for script scale
; ✗ Wrong — Observer pattern for a 200-line utility script with 2 event types
class EventBus {                    ; 🟠 Major: pattern complexity exceeds script size
    static subscribers := Map()
    static Subscribe(event, fn) { ... }
    static Publish(event, data) { ... }
    static Unsubscribe(event, fn) { ... }
}
; Two actual usages:
EventBus.Subscribe("hotkey", HandleHotkey)
EventBus.Publish("hotkey", keyData)

; ✓ Correct — direct function call for 2-subscriber scenarios
HandleHotkey(keyData)

; ✗ Wrong — single-method class wrapping one AHK built-in
class ClipboardHelper {             ; 🟠 Major: class adds zero value
    static Get() { return A_Clipboard }
    static Set(val) { A_Clipboard := val }
}

; ✓ Correct — use A_Clipboard directly; no wrapper needed
A_Clipboard := newValue
current := A_Clipboard

; ================================================================
; REVIEW PROCESS CHECKS
; ================================================================

; ----------------------------------------------------------------
; CHECK 13: Pre-review declaration — run before Dimension 1–6 analysis
;
; Every review report must open with these four declarations:
;
;   Review scope:  [Complete script | Fragment — Dim 2–6 limited to visible constructs]
;   AHK version:   [v2.0 confirmed via #Requires | v2.0 assumed — no directive found]
;   Script length: [< 50 lines | 50–200 lines | 200–500 lines | 500+ lines]
;   Declared intent: [stated goal] | [Not declared — reviewing for general quality only]
;
; Agent rule: omitting any of these four declarations is 🟠 Major —
;             a reader cannot interpret severity in context without them.

; ----------------------------------------------------------------
; CHECK 14: Finding wording norms — severity language must be unambiguous
;
; Critical findings MUST include: (1) concrete runtime consequence,
;                                 (2) specific fix with inline code if ≤ 5 lines.
; Critical findings MUST NOT use: "might", "could", "consider", "perhaps".
;
; ✗ Wrong — vague Critical wording
; 🔴 Line 7 — counter might not be assigned correctly.
;    Consider using := instead of =.
;
; ✓ Correct — concrete Critical wording
; 🔴 Line 7 — `counter = 0` uses `=` which is comparison in v2; counter stays "".
;    Fix: `counter := 0`
;
; Major findings MUST include:   (1) what the violation costs,
;                                (2) recommended action.
; Major findings MUST NOT use:   "might cause issues", "generally better".
;
; ✗ Wrong — vague Major wording
; 🟠 Line 23 — GetUser() is not that clean.
;
; ✓ Correct — concrete Major wording
; 🟠 Line 23 — GetUser() writes to access.log (side effect in a Get* function).
;    Rename to LogAndGetUser() or extract the FileAppend to a separate LogAccess().
;
; Minor findings MAY be brief but must name the specific line and pattern.
; ✗ Wrong: "Some naming issues throughout"
; ✓ Correct: "🟡 Line 45 — isNotReady is negative naming; rename to isReady"

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
| Comment restates code | `; add 1 to counter` above `counter += 1` | `; debounce: first event always accepted` (explains intent) | LLM training data contains abundant "explanatory" comments that describe what, not why; generated comments inherit this habit |
| Get* function with side effect | `GetUser()` that also calls `FileAppend()` | Split into `GetUser()` (pure read) + `LogAccess()` (mutation) | Cross-language training: Python/JS examples often mix logging into retrieval functions without flagging it as a contract violation |
| Negative boolean naming | `isNotReady`, `!isDisabled` in conditionals | `isReady`, `isEnabled` — always affirmative | Negations are common in training data; LLMs do not track the cognitive cost of double-negation for readers |
| Magic string repeated 3+ times | `"HKCU\Software\MyApp"` inline in 4 locations | `APP_REGISTRY_KEY := "HKCU\Software\MyApp"` constant | Magic number rule from training data is well-known; magic string rule is underrepresented, so LLMs apply it inconsistently |
| Over-abstraction / YAGNI | `FileReaderFactory.Create("text").Read(path)` for a single file type | `FileRead(path, "UTF-8")` directly | LLMs default to "extensible" solutions; training data rewards design patterns regardless of whether the problem size justifies them |
| Over-defensive internal guards | 3 type checks inside a private helper whose only caller already guarantees the type | Validate at the public entry point only; trust internal callers | "Defensive programming" training data does not distinguish public API boundaries from internal function calls |
| Single-method class wrapping a built-in | `class ClipboardHelper { static Get() { return A_Clipboard } }` | `A_Clipboard` directly | LLMs generalise OOP "encapsulate everything" patterns from enterprise-scale training data to single-file scripts |
| Function > 40 lines | 80-line function mixing I/O, logic, and UI in one body | ≤ 40 lines per function; one named responsibility | LLMs generate complete end-to-end flows naturally; they do not apply a line-count threshold unless explicitly instructed |
| Parameter count > 4 | `Fn(a, b, c, d, e, f)` positional API | `Fn(params)` where params is `Map(...)` | Positional parameters are the default function signature in most training language examples; Map parameter objects are underrepresented |
| Vague Critical finding wording | `🔴 counter might not be assigned` | `🔴 Line 7 — counter = 0 uses = (comparison); counter stays "". Fix: counter := 0` | LLMs hedge uncertainty with "might/could"; Code Review severity demands declarative language with concrete consequences |
| Unvalidated input in DllCall path | `DllCall("DeleteFileW", "Str", InputBox(...).Value)` | Regex whitelist + `FileExist()` check before `DllCall` | Security validation training data is sparse for scripting language contexts; LLMs default to trusting user-supplied strings |
| Dynamic RegEx without try/catch | `RegExMatch(data, userPattern)` where pattern is from user input | Wrap in `try/catch`; surface `e.Message` | LLMs apply try/catch to static-pattern calls inconsistently; dynamic-pattern risk is rarely demonstrated in training examples |

## SEE ALSO

> This module does NOT cover: AHK v2 syntax rules, idiomatic API usage, or language semantics — those are in Module_Instructions.md and the domain-specific knowledge modules.
> This module does NOT cover: try/catch error class hierarchy, typed exception matching, or `OnError()` global handlers → see Module_Errors.md.
> This module does NOT cover: GUI control creation patterns, layout management, or control property APIs → see Module_GUI.md.
> This module does NOT cover: class inheritance, meta-functions (`__Get`/`__Set`/`__Call`), or prototype chains → see Module_Classes.md.
> This module does NOT cover: general software engineering theory (SOLID, DRY, design pattern catalogues) — only AHK v2 script-scale application of Clean Code and YAGNI principles is defined here.

- `Module_Instructions.md` — master AHK v2 syntax rules, idiomatic patterns, and the baseline against which Dimension 1 compliance is measured; load this module first for any general AHK v2 question.
- `Module_Errors.md` — typed error classes, `OnError()` global handler, and try/catch patterns for Dimension 2 deep-dives; cross-reference when evaluating COM or file I/O error handling.
- `Module_GUI.md` — comprehensive control creation, layout, and `.OnEvent()` binding patterns for Dimension 4 evaluation; cross-reference when the submitted script defines class-based GUI controllers.
- `Module_Classes.md` — class static property encapsulation, inheritance, and meta-function patterns for Dimension 3 architecture evaluation; cross-reference when reviewing scripts with multiple classes or OOP-heavy designs.