# Module_Validation.md
<!-- DOMAIN: Defensive Programming and Parameter Validation -->
<!-- SCOPE: Custom error class hierarchies, try/catch control flow structure, and validator base-class inheritance belong in Module_Errors.md and Module_Classes.md respectively — not here. -->
<!-- TRIGGERS: validate, guard clause, type check, IsSet, is Integer, is Float, is Number, TypeError, ValueError, HasMethod, HasProp, duck typing, fail early, input validation, assert, precondition, defensive programming, sanitize input, fluent validator, parameter guard, pre-flight check, interface validation, type assertion, ?? operator, constructor guard, __New validation, unset parameter, optional parameter default, RegExMatch format check -->
<!-- CONSTRAINTS: `x is String` always returns false for string primitives — use `Type(x) = "String"` for string detection. Always supply the -1 stack offset in every throw call: `throw TypeError("msg", -1)`. Place all guard clauses at the very top of the function before any logic executes. Duck-type external objects with HasMethod()/HasProp() — never assert Type() against a class name string. -->
<!-- CROSS-REF: Module_Errors.md, Module_Classes.md, Module_Instructions.md -->
<!-- VERSION: AHK v2.0+ -->

## V1 → V2 BREAKING CHANGES

| v1 pattern (LLM commonly writes) | v2 correct form | Consequence |
|----------------------------------|-----------------|-------------|
| `if val is String` as string type check | `if Type(val) = "String"` | `is String` always returns false for string primitives — every string passes the guard silently |
| `throw Error("bad type")` for type violations | `throw TypeError("expected X, got " Type(x), -1)` | Wrong error class; callers cannot `catch TypeError` specifically; missing -1 offset reports throw-line not call-site |
| `if param = ""` to detect unset optional param | `if !IsSet(param)` | Unset variable and empty string are distinct states — empty-string comparison silently treats unset param as valid |
| `Type(obj) = "MyClass"` for capability check | `HasMethod(obj, "MethodName")` | Rejects all valid subclasses and polymorphic implementors — MethodError is raised at the call site, not the validation point |
| `this.Prop := val` before guard block in `__New` | Complete guard block first, then all property assignments | Partially-initialised object can be captured by caller reference before guard throws — corrupt state escapes |
| Guard clauses placed after the first logic statement | Move all guards to function top before any logic line | Logic executes on invalid inputs; internal state may be corrupted before the guard fires |
| `try { obj.Method() } catch MethodError` as capability probe | `HasMethod(obj, "Method")` pre-flight check | O(n) exception unwind on miss path vs O(1) hash lookup; hides the probe intent and complicates diagnostics |

## API QUICK-REFERENCE

### Type Detection Operators
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `is` operator | `value is Integer` / `value is Float` / `value is Number` / `value is ClassName` | Covers Integer, Float, Number, Object, and class names (including full inheritance chain); does NOT reliably detect String — `is String` always false for string primitives |
| `Type()` | `Type(value)` → `"Integer"`, `"Float"`, `"String"`, `"Object"`, class name | Only reliable string type detection: `Type(x) = "String"`; also use for class-name debugging, not for capability checks |
| `IsObject()` | `IsObject(value)` → true/false | True for Array, Map, class instances, and all reference types; false for Integer, Float, String scalars |
| `IsSet()` | `IsSet(param)` → true/false | False only when an optional parameter was omitted by the caller; true even for empty string or 0 — not a blank check |
| `??` operator | `value ?? fallback` | Returns left operand if it is SET (any value including `""` or `0`); returns fallback only when left is genuinely unset; use AFTER guards, not inside them |

### Object Capability Probes
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `HasMethod()` | `HasMethod(obj, methodName)` → true/false | O(1) hash lookup; use before calling any method on an externally-supplied object; preferred over try/catch MethodError |
| `HasProp()` | `HasProp(obj, propName)` → true/false | O(1); use before reading any property on an externally-supplied object; prevents PropertyError |

### Error Constructors (Validation Context)
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `TypeError()` | `throw TypeError(message, stackOffset)` | Use for type mismatches — wrong type passed; always supply `-1` as stackOffset so AHK v2 reports the caller's line |
| `ValueError()` | `throw ValueError(message, stackOffset)` | Use for constraint violations — correct type but invalid value/range; always supply `-1` |
| `OSError()` | `throw OSError(code?)` | Wraps a Win32 error code; `code` is an optional integer (defaults to `A_LastError`); does NOT accept a string message — use `Error(message, -1)` for custom pre-flight failure messages |
| `TargetError()` | `throw TargetError(message, stackOffset)` | Use for window/process pre-flight failures (WinExist false, ProcessExist false) |

### Environment Pre-Flight Functions
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `FileExist()` | `FileExist(path)` → attribute string or `""` | Returns attribute string (e.g. `"A"`) if file exists, `""` if not — not a boolean; use `!= ""` for truthiness |
| `DirExist()` | `DirExist(path)` → attribute string or `""` | Same return semantics as FileExist; use `!= ""` for truthiness |
| `DirCreate()` | `DirCreate(path)` | Creates directory and all parent directories if they do not already exist; throws `OSError` on failure |
| `WinExist()` | `WinExist(winTitle)` → HWND or `0` | Returns window handle if window exists, `0` if not; call before WinActivate / SendInput |
| `ProcessExist()` | `ProcessExist(processName)` → PID or `0` | Returns PID if process is running, `0` if not; call before ProcessClose / window interaction |

### ValidationBuilder / FormValidator Methods
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `ValidationBuilder.For()` | `ValidationBuilder.For(value, fieldName?)` → instance | Static factory; entry point for every fluent chain |
| `.Required()` | `.Required()` → this | Pushes error if String value is blank after Trim; returns `this` for chaining |
| `.IsInteger()` | `.IsInteger()` → this | Pushes error if value is not Integer |
| `.IsString()` | `.IsString()` → this | Pushes error if Type(value) ≠ "String" |
| `.InRange()` | `.InRange(minVal, maxVal)` → this | Pushes error if Integer value is outside [minVal, maxVal] |
| `.MinLength()` | `.MinLength(n)` → this | Pushes error if String length < n |
| `.MaxLength()` | `.MaxLength(n)` → this | Pushes error if String length > n |
| `.Matches()` | `.Matches(pattern, description?)` → this | Pushes error if String does not match RegEx pattern |
| `.Satisfies()` | `.Satisfies(predFn, description?)` → this | Pushes error if callable predFn returns false; validates predFn with HasMethod() first |
| `.Validate()` | `.Validate()` → true or throws ValueError | Terminal — throws ValueError combining all collected errors; returns true if none |
| `.GetErrors()` | `.GetErrors()` → Array | Terminal — returns error Array without throwing; use with FormValidator |
| `.HasErrors()` | `.HasErrors()` → true/false | Non-throwing status check |
| `FormValidator.Add()` | `.Add(builder)` → this | Register a ValidationBuilder with the form aggregator |
| `.GetAllErrors()` | `.GetAllErrors()` → Array | Collect errors across all registered builders |
| `.IsValid()` | `.IsValid()` → true/false | True only if every registered builder has zero errors |

## AHK V2 CONSTRAINTS

- `x is String` is syntactically valid but always returns false for string primitives — AHK v2's `is` operator has no special handling for the String type — use `Type(x) = "String"` for string detection without exception.
- `is` works correctly for `Integer`, `Float`, `Number`, `Object`, and class names (including full inheritance chain) — use it for all numeric checks and class membership.
- `IsSet(param)` and `param = ""` are not equivalent — `IsSet()` returns false only when the caller omitted the argument entirely; empty string, 0, and false all return true from `IsSet()` — never substitute empty-string comparison for an unset check.
- `??` is a default resolver, not a guard — `param ?? fallback` silently swallows an unset state without throwing — use `!IsSet(param)` inside the guard block first, then use `??` on the assignment line that follows.
- Always supply `-1` as the stack offset in every `throw` call — without it, AHK v2 reports the throw line inside the guard helper rather than the caller's line, making error dialogs misleading — `throw TypeError("msg", -1)`.
- Throw `TypeError` for type mismatches, `ValueError` for constraint violations (correct type, wrong value), `TargetError` for window/process failures — for custom pre-flight failure messages where no Win32 error code applies, throw `Error("msg", -1)`; `OSError(code?)` only accepts a numeric OS error code and is intended for wrapping actual Win32 failures.
- Every thrown error message must include the actual value and the expected type or range — a message like `"bad type"` is not actionable; `"x must be Integer, got Float (value: 3.14)"` is.
- Place every guard clause at the very top of the function before the first logic statement — guard clauses placed after logic allow corrupted internal state to accumulate before the throw fires.
- In `__New`, complete the full guard block before the first `this.Prop :=` assignment — a `TypeError` thrown mid-assignment leaves the object partially initialised; `__New`-level guards prevent a corrupt object from ever existing.
- Use `HasMethod(obj, "Method")` and `HasProp(obj, "Prop")` for external object validation — never compare `Type(obj)` to a class name string; that breaks all valid subclasses and polymorphic implementors.
- Reserve guard clauses for public and external API boundaries — over-guarding private internal helpers adds noise without safety benefit.

Safe-access priority order for validation selection:
  1. `is` / `Type()` / `IsSet()` — O(1) scalar checks; always first
  2. `StrLen()` / `Trim()` — O(n) but near-zero on short strings; after type check
  3. `HasMethod()` / `HasProp()` — O(1) hash lookup; for object parameters
  4. `RegExMatch()` — O(n) pattern match; only after type and length screening pass
  5. `FileExist()` / `WinExist()` / `ProcessExist()` — one OS syscall each; last, after all cheaper checks

## TIER 1 — Type and Null Fundamentals
> METHODS COVERED: `is` operator · `Type()` · `IsSet()` · `IsObject()` · `??` operator · `StrLen()` · `RegExMatch()`

TIER 1 covers the foundational building blocks of every validation strategy in AHK v2: using the `is` operator for numeric types, `Type()` for string detection, `IsSet()` for optional parameter presence checks, the `??` nullish coalescing operator for compact default merging, and `IsObject()` to distinguish scalars from objects. Mastering these primitives is a prerequisite for every higher tier — guards in TIER 2 through TIER 6 are assembled from exactly these calls. Use `IsSet()` to detect and reject unset states; use `??` to resolve defaults after guards have already passed.
```ahk
; --- AHK v2 TYPE DETECTION REFERENCE ---
;
; Goal                            Correct AHK v2 syntax
; -------------------------       ---------------------------------------------------------
; Is an integer?                  value is Integer
; Is a float?                     value is Float
; Is integer or float?            value is Number
; Is a string?                    Type(value) = "String"   ; ✗ 'is String' always false
; Is any object?                  IsObject(value)
; Is a specific class?            value is MyClass
; Is subclass instance?           value is BaseClass       ; checks full chain
; Is parameter unset?             !IsSet(param)
; Has a method?                   HasMethod(obj, "Name")
; Has a property?                 HasProp(obj, "Name")

; --- TIER 1: Type Detection and Unset Checks ---

; NUMERIC TYPE CHECKS — use 'is' operator, not Type() string comparison
CheckNumericTypes(val) {
    if val is Integer        ; ✓ 'is' is the correct operator for integer detection
        return "integer: " val
    if val is Float          ; ✓ 'is' is the correct operator for float detection
        return "float: " val
    if val is Number         ; ✓ covers both Integer and Float
        return "number (generic): " val
    return "not a numeric scalar"
}

; STRING TYPE CHECK — 'is String' always returns false for string primitives; use Type()
CheckStringType(val) {
    if Type(val) = "String"  ; ✓ only reliable string type check in AHK v2
        return "string, length: " StrLen(val)
    ; if val is String       ; ✗ always returns false for string primitives in AHK v2
    return "not a string, type: " Type(val)
}

; OBJECT vs SCALAR DISTINCTION
CheckObjectOrScalar(val) {
    if IsObject(val)         ; ✓ true for Array, Map, class instances, etc.
        return "object, type: " Type(val)
    return "scalar, type: " Type(val)
}

; CLASS INSTANCE CHECK — 'is' checks full inheritance chain
class Shape {}
class Circle extends Shape {}

CheckClassMembership(val) {
    if val is Circle         ; true only for Circle instances
        return "Circle"
    if val is Shape          ; true for Circle too — checks ancestor chain
        return "Shape (or subclass)"
    return "unrelated type: " Type(val)
}

; OPTIONAL PARAMETER UNSET CHECK — IsSet() is the only correct method
ProcessData(data, options?) {
    if !IsSet(options)       ; ✓ IsSet() detects unset optional params
        options := Map("mode", "default")
    ; if options = ""        ; ✗ unset variable ≠ empty string — silently passes unset
    MsgBox("Processing with " options.Count " option(s)")
}

; COMBINED FIRST-LINE GUARD — type + null in sequence
Compute(a, b, label?) {
    if !(a is Number)
        throw TypeError("a must be Number, got " Type(a), -1)
    if !(b is Number)
        throw TypeError("b must be Number, got " Type(b), -1)
    if IsSet(label) && !(Type(label) = "String")
        throw TypeError("label must be String when provided, got " Type(label), -1)

    resolvedLabel := IsSet(label) ? label : "result"
    MsgBox(resolvedLabel ": " (a + b))
}

; NULLISH COALESCING (??) — compact default merging after guards pass
; ?? returns the left operand if it is SET (any value); right operand if left is UNSET.
; ✓ Use IsSet() in guards to reject bad states; use ?? to apply defaults cleanly.

ApplyDefaults(title?, timeout?, items?) {
    ; Guards: reject wrong types if the optionals ARE provided
    if IsSet(title)   && !(Type(title) = "String")
        throw TypeError("title must be String when provided, got " Type(title), -1)
    if IsSet(timeout) && !(timeout is Integer)
        throw TypeError("timeout must be Integer when provided, got " Type(timeout), -1)
    if IsSet(items)   && !(items is Array)
        throw TypeError("items must be Array when provided, got " Type(items), -1)

    ; Defaults: ?? resolves unset optionals compactly after all guards passed
    resolvedTitle   := title   ?? "Untitled"   ; ✓ "Untitled" only if title was omitted
    resolvedTimeout := timeout ?? 3000         ; ✓ 3000 ms if timeout was omitted
    resolvedItems   := items   ?? []           ; ✓ empty array if items was omitted

    ; if title = ""   ; ✗ "" is SET, not UNSET — ?? would return "" here, not "Untitled"
    ; if !IsSet(title) ; ✓ correct guard form when you need to REJECT unset

    MsgBox(resolvedTitle " | " resolvedTimeout " ms | "
        resolvedItems.Length " item(s)")
}

; CHAINED ?? — first set value in a priority chain wins
GetDisplayName(override?, saved?, fallback := "Anonymous") {
    return override ?? saved ?? fallback   ; ✓ first initialised value wins
}

; QUICK USAGE DEMONSTRATION
MsgBox(CheckNumericTypes(42))          ; → "integer: 42"
MsgBox(CheckNumericTypes(3.14))        ; → "float: 3.14"
MsgBox(CheckStringType("hello"))       ; → "string, length: 5"
MsgBox(CheckObjectOrScalar(Map()))     ; → "object, type: Map"
MsgBox(CheckObjectOrScalar(99))        ; → "scalar, type: Integer"
ProcessData("payload")                 ; options defaults via IsSet() path
Compute(10, 20)                        ; label is unset — uses default
Compute(10, 20, "sum")                 ; label provided and validated
ApplyDefaults()                        ; all defaults resolved via ??
ApplyDefaults("Report", 5000)          ; title and timeout provided; items defaults
MsgBox(GetDisplayName(, , "Guest"))    ; → "Guest" (first two unset)
```

## TIER 2 — Guard Clauses
> METHODS COVERED: `throw TypeError()` · `throw ValueError()` · `IsSet()` · `Type()` · `is` operator · `StrLen()` · `Trim()` · `.Length` (Array)

TIER 2 assembles TIER 1 primitives into complete guard clause blocks placed at the top of every function that accepts external input. The key discipline is separating `TypeError` (wrong type passed) from `ValueError` (right type, wrong value or constraint), and providing descriptive messages that include both the actual value and the expected contract so callers can diagnose failures without reading source code. `__New` is an API boundary — every constructor must contain a complete guard block before the first property assignment.
```ahk
; --- TIER 2: Guard Clauses — Function-top Parameter Defence ---

; GUARD HIERARCHY: TypeError for type violations, ValueError for constraint violations
SetVolume(level, channel := "master") {
    ; Guard 1: type check — wrong type is always TypeError
    if !(level is Integer)
        throw TypeError("level must be Integer, got " Type(level), -1)

    ; Guard 2: constraint check — correct type but invalid value is ValueError
    if level < 0 || level > 100
        throw ValueError("level must be 0-100, got " level, -1)

    ; Guard 3: string type guard (Type(), not 'is')
    if !(Type(channel) = "String")
        throw TypeError("channel must be String, got " Type(channel), -1)

    ; Guard 4: string constraint — empty channel name is invalid
    if StrLen(Trim(channel)) = 0
        throw ValueError("channel must not be blank", -1)

    ; All inputs verified — safe to proceed
    MsgBox("Setting '" channel "' volume to " level "%")
}

; OPTIONAL PARAMETER GUARDS — always check IsSet() before checking type
RenderWidget(content, color?, fontSize?) {
    ; Required parameter checks first
    if !(Type(content) = "String")
        throw TypeError("content must be String, got " Type(content), -1)
    if StrLen(Trim(content)) = 0
        throw ValueError("content must not be blank", -1)

    ; Optional: check IsSet() BEFORE checking the type of the param
    if IsSet(color) && !(Type(color) = "String")
        throw TypeError("color must be String when provided, got " Type(color), -1)

    if IsSet(fontSize) && !(fontSize is Integer)
        throw TypeError("fontSize must be Integer when provided, got " Type(fontSize), -1)

    if IsSet(fontSize) && fontSize < 1
        throw ValueError("fontSize must be >= 1 when provided, got " fontSize, -1)

    ; Resolve defaults after all guards pass
    resolvedColor    := IsSet(color)    ? color    : "black"
    resolvedFontSize := IsSet(fontSize) ? fontSize : 12
    MsgBox("Rendering: '" content "' [" resolvedColor ", " resolvedFontSize "px]")
}

; ARRAY PARAMETER GUARD — type, class, and length checks stacked
ProcessBatch(items) {
    if !IsObject(items)
        throw TypeError("items must be an Array, got " Type(items), -1)
    if !(items is Array)
        throw TypeError("items must be Array (not Map or other Object), got "
            Type(items), -1)
    if items.Length = 0
        throw ValueError("items array must not be empty", -1)
    if items.Length > 1000
        throw ValueError("items exceeds max batch size of 1000 (got "
            items.Length ")", -1)

    for item in items
        MsgBox("Processing: " item)
}

; ERROR MESSAGE QUALITY — always include actual vs expected in the message
BadGuard(x) {
    if !(x is Integer)
        throw TypeError("bad type")             ; ✗ unhelpful — no context
}

GoodGuard(x) {
    if !(x is Integer)
        throw TypeError(                        ; ✓ includes actual value and type
            "x must be Integer, got " Type(x) " (value: " x ")", -1)
}

; THE -1 STACK OFFSET — required for AHK v2 to report caller line in error dialog
; throw TypeError("msg")      ; ✗ reports error at the throw line (inside guard)
; throw TypeError("msg", -1)  ; ✓ reports error at the caller's line

; USAGE
try {
    SetVolume(150)              ; triggers ValueError: out of range
} catch ValueError as e {
    MsgBox("Caught: " e.Message)
}

try {
    SetVolume(80, "")           ; triggers ValueError: blank channel
} catch ValueError as e {
    MsgBox("Caught: " e.Message)
}

; CLASS CONSTRUCTOR GUARD — type assertions in __New prevent partially-constructed objects
; __New is an API boundary: validate every external parameter before writing any state.
class UserRecord {
    Name  := ""
    Age   := 0
    Email := ""

    __New(name, age, email?) {
        ; === CONSTRUCTOR GUARD BLOCK (before any property assignment) ===
        if !(Type(name) = "String") || StrLen(Trim(name)) = 0
            throw TypeError("name must be a non-empty String, got "
                Type(name), -1)

        if !(age is Integer)
            throw TypeError("age must be Integer, got " Type(age), -1)
        if age < 0 || age > 150
            throw ValueError("age must be 0–150, got " age, -1)

        if IsSet(email) && !(Type(email) = "String")
            throw TypeError("email must be String when provided, got "
                Type(email), -1)
        if IsSet(email) && !RegExMatch(email, "i)^[a-z0-9._%+\-]+@[a-z0-9.\-]+\.[a-z]{2,}$")
            throw ValueError("email format invalid: '" email "'", -1)
        ; === END GUARD BLOCK ===

        ; Property assignment only after all guards pass — object is always coherent
        this.Name  := name
        this.Age   := age
        this.Email := email ?? ""   ; ?? applies default compactly after guards
    }

    Describe() => this.Name " (age " this.Age ")"
        . (StrLen(this.Email) > 0 ? " <" this.Email ">" : "")
}

try {
    u := UserRecord("", 25)         ; throws TypeError: blank name
} catch TypeError as e {
    MsgBox("Rejected: " e.Message)
}

try {
    u := UserRecord("Alice", 200)   ; throws ValueError: age out of range
} catch ValueError as e {
    MsgBox("Rejected: " e.Message)
}

valid := UserRecord("Bob", 30, "bob@example.com")
MsgBox(valid.Describe())            ; → "Bob (age 30) <bob@example.com>"
```

## TIER 3 — String and Business Logic Validation
> METHODS COVERED: `RegExMatch()` · `StrLen()` · `Trim()` · `throw TypeError()` · `throw ValueError()` · `.Push()` · `.Length`

TIER 3 extends guard clauses into domain-specific rules: email formats, password strength, filename safety, and numeric ranges with multiple constraints. This tier introduces the collect-all error pattern for single-field multi-rule validation, where every rule is checked even after an earlier rule fails, and all errors are reported together in a single `ValueError`.
```ahk
; --- TIER 3: String and Business Logic Validation ---

; EMAIL FORMAT VALIDATION
ValidateEmail(email) {
    if !(Type(email) = "String")
        throw TypeError("email must be String, got " Type(email), -1)
    if StrLen(Trim(email)) = 0
        throw ValueError("email must not be blank", -1)
    ; RFC-5321 simplified pattern — tighten the local-part rules as needed
    if !RegExMatch(email, "i)^[a-z0-9._%+\-]+@[a-z0-9.\-]+\.[a-z]{2,}$")
        throw ValueError("email format invalid: '" email "'", -1)
    return true
}

; PASSWORD STRENGTH — collect all failures, report at once (single field)
ValidatePassword(pw) {
    if !(Type(pw) = "String")
        throw TypeError("password must be String, got " Type(pw), -1)

    errors := []
    if StrLen(pw) < 8
        errors.Push("must be at least 8 characters")
    if !RegExMatch(pw, "[A-Z]")
        errors.Push("must contain at least one uppercase letter")
    if !RegExMatch(pw, "[0-9]")
        errors.Push("must contain at least one digit")
    if !RegExMatch(pw, "[^a-zA-Z0-9]")
        errors.Push("must contain at least one special character")

    if errors.Length > 0 {
        msg := ""
        for i, e in errors
            msg .= (i > 1 ? "`n" : "") "- " e
        throw ValueError("Password requirements not met:`n" msg, -1)
    }
    return true
}

; FILENAME SAFETY — reject Windows-illegal characters
ValidateFilename(name) {
    if !(Type(name) = "String")
        throw TypeError("filename must be String, got " Type(name), -1)
    if StrLen(Trim(name)) = 0
        throw ValueError("filename must not be blank", -1)
    ; Windows illegal characters: \ / : * ? " < > |
    if RegExMatch(name, '[\\/:*?"<>|]')
        throw ValueError("filename contains an illegal character: '" name "'", -1)
    if StrLen(name) > 255
        throw ValueError("filename exceeds 255-character limit (got "
            StrLen(name) ")", -1)
    return true
}

; NETWORK PORT — typed integer with range and privilege warning
ValidatePort(port) {
    if !(port is Integer)
        throw TypeError("port must be Integer, got " Type(port), -1)
    if port < 1 || port > 65535
        throw ValueError("port must be 1-65535, got " port, -1)
    if port < 1024
        MsgBox("Note: port " port " is a privileged port (< 1024)")
    return true
}

; URL SLUG — alphanumeric + hyphen, constrained length
ValidateSlug(slug) {
    if !(Type(slug) = "String")
        throw TypeError("slug must be String, got " Type(slug), -1)
    if StrLen(slug) < 1 || StrLen(slug) > 60
        throw ValueError("slug length must be 1-60, got " StrLen(slug), -1)
    if !RegExMatch(slug, "^[a-z0-9][a-z0-9\-]*[a-z0-9]$")
        throw ValueError("slug must be lowercase alphanumeric with hyphens: '"
            slug "'", -1)
    return true
}

; USAGE WITH STRUCTURED ERROR HANDLING
try {
    ValidateEmail("not-valid")      ; throws ValueError
} catch ValueError as e {
    MsgBox("Email error: " e.Message)
}

try {
    ValidatePassword("weak")        ; throws ValueError with all failures listed
} catch ValueError as e {
    MsgBox(e.Message)
}

try {
    ValidateFilename("file?.txt")   ; throws ValueError: illegal '?'
} catch ValueError as e {
    MsgBox("Filename error: " e.Message)
}
```

## TIER 4 — Duck-Type Object and Interface Validation
> METHODS COVERED: `HasMethod()` · `HasProp()` · `IsObject()` · `throw TypeError()`

TIER 4 covers validation of object parameters whose class identity is unknown or intentionally polymorphic. Duck-type validation via `HasMethod()` and `HasProp()` checks that an object fulfils a behavioural contract rather than asserting a class name, which breaks when valid subclasses or unrelated implementors are passed. This tier prevents `MethodError` and `PropertyError` at call sites without sacrificing polymorphic flexibility.
```ahk
; --- TIER 4: Object and Interface Validation (Duck Typing) ---

; SINGLE CAPABILITY PROBE — prevent MethodError before calling
SafeProcess(processor) {
    ; ✓ check capability, not class identity — accepts any object with Process()
    if !HasMethod(processor, "Process")
        throw TypeError(
            "processor requires a Process() method; got " Type(processor), -1)
    processor.Process()
}

BadCapabilityCheck(obj) {
    if Type(obj) != "MyProcessor"   ; ✗ breaks subclasses and polymorphic callers
        throw TypeError("must be MyProcessor")
}

; MULTI-CAPABILITY INTERFACE VALIDATOR
; Validates that an object satisfies all required methods and properties
EnsureInterface(obj, requiredMethods, requiredProps) {
    if !IsObject(obj)
        throw TypeError("obj must be an Object, got " Type(obj), -1)
    for method in requiredMethods {
        if !HasMethod(obj, method)
            throw TypeError("missing required method: '" method
                "' on " Type(obj), -1)
    }
    for prop in requiredProps {
        if !HasProp(obj, prop)
            throw TypeError("missing required property: '" prop
                "' on " Type(obj), -1)
    }
    return true
}

; CALLABLE VALIDATION — verify a value can be invoked as a function
EnsureCallable(handler, name := "handler") {
    if !IsObject(handler)
        throw TypeError(name " must be a callable object, got " Type(handler), -1)
    ; AHK v2 callable objects expose a Call() method (Func, BoundFunc, arrow fns)
    if !HasMethod(handler, "Call")
        throw TypeError(name " must be callable (missing Call method); got "
            Type(handler), -1)
    return true
}

; ITERABLE VALIDATION — verify an object can be consumed by for-in
EnsureIterable(collection, name := "collection") {
    if !IsObject(collection)
        throw TypeError(name " must be an Object, got " Type(collection), -1)
    ; Array, Map, and any custom iterable expose __Enum for for-in
    if !HasMethod(collection, "__Enum")
        throw TypeError(name " must be iterable (missing __Enum method); got "
            Type(collection), -1)
    return true
}

; CONCRETE USAGE — two unrelated classes that satisfy the same interface
class TextRenderer {
    Name  := "TextRenderer"
    Process() => MsgBox("Rendering text")
    Reset()   => MsgBox("Resetting text renderer")
}

class ImageRenderer {
    Name  := "ImageRenderer"
    Process() => MsgBox("Rendering image")
    Reset()   => MsgBox("Resetting image renderer")
}

; Both classes pass SafeProcess — duck typing is polymorphic by design
SafeProcess(TextRenderer())     ; ✓ passes
SafeProcess(ImageRenderer())    ; ✓ passes

; Interface check with method + property requirements
EnsureInterface(
    TextRenderer(),
    ["Process", "Reset"],       ; required methods
    ["Name"]                    ; required properties
)                               ; ✓ passes for both renderers

; Callable check on arrow function
OnSubmit(callback) {
    EnsureCallable(callback, "callback")
    callback()
}

OnSubmit(() => MsgBox("Form submitted"))    ; ✓ arrow function is callable

; PROPERTY EXISTENCE BEFORE READ — prevent PropertyError
SafeGetName(obj) {
    if !HasProp(obj, "Name")
        throw TypeError("obj must have a 'Name' property; got " Type(obj), -1)
    return obj.Name
}

MsgBox(SafeGetName(TextRenderer()))     ; → "TextRenderer"
```

## TIER 5 — State and External Environment Validation
> METHODS COVERED: `FileExist()` · `DirExist()` · `DirCreate()` · `WinExist()` · `WinActivate()` · `ProcessExist()` · `ProcessClose()` · `FileOpen()` · `FileAppend()` · `FileCopy()` · `SplitPath()` · `throw Error()` · `throw TargetError()`

TIER 5 covers pre-flight checks against external state that AHK v2 cannot guarantee through type or capability probing alone: file existence, directory presence, window availability, and process state. Skipping these checks causes AHK v2 to throw `OSError`, `TargetError`, or return silent failures from file operations, making post-hoc diagnosis difficult. Pre-flight guards make every failure explicit and attributable before any destructive or irreversible action is attempted.
```ahk
; --- TIER 5: State and External Environment Pre-flight Checks ---

; FILE PRE-FLIGHT — prevent OSError on missing file
SafeReadFile(path) {
    if !(Type(path) = "String") || StrLen(Trim(path)) = 0
        throw ValueError("path must be a non-empty String", -1)
    if !FileExist(path)                         ; ✓ pre-flight: verify before open
        throw Error("file not found: '" path "'", -1)
    ; File confirmed to exist — FileOpen will not throw unexpectedly
    f       := FileOpen(path, "r", "UTF-8")
    content := f.Read()
    f.Close()
    return content
}

; DIRECTORY WRITE GUARD — create directory if absent, then verify
SafeWriteToDir(dirPath, filename, content) {
    if !(Type(dirPath) = "String") || StrLen(Trim(dirPath)) = 0
        throw ValueError("dirPath must be a non-empty String", -1)
    if !(Type(filename) = "String") || StrLen(Trim(filename)) = 0
        throw ValueError("filename must be a non-empty String", -1)

    if !DirExist(dirPath) {
        DirCreate(dirPath)
        if !DirExist(dirPath)                   ; verify creation succeeded
            throw Error("could not create directory: '" dirPath "'", -1)
    }
    FileAppend(content, dirPath "\" filename, "UTF-8")
}

; WINDOW PRE-FLIGHT — prevent TargetError on missing window
SafeSendKeys(winTitle, keys) {
    if !(Type(winTitle) = "String") || StrLen(Trim(winTitle)) = 0
        throw ValueError("winTitle must be a non-empty String", -1)
    if !(Type(keys) = "String")
        throw TypeError("keys must be String, got " Type(keys), -1)
    if !WinExist(winTitle)                      ; ✓ pre-flight: verify window
        throw TargetError("window not found: '" winTitle "'", -1)
    WinActivate(winTitle)
    SendInput(keys)
}

; PROCESS PRE-FLIGHT — verify running before interacting, verify close after
SafeCloseProcess(processName) {
    if !(Type(processName) = "String") || StrLen(Trim(processName)) = 0
        throw ValueError("processName must be a non-empty String", -1)
    pid := ProcessExist(processName)
    if !pid
        throw TargetError("process not running: '" processName "'", -1)
    ProcessClose(processName)
    ; Post-flight: confirm the process actually closed
    if ProcessExist(processName)
        throw Error("process did not terminate: '" processName "'", -1)
    return "Closed PID " pid
}

; COMPOSITE PRE-FLIGHT — multi-condition guard before destructive file operation
SafeReplaceFile(sourcePath, destPath) {
    if !FileExist(sourcePath)
        throw Error("source file missing: '" sourcePath "'", -1)
    if !(Type(destPath) = "String") || StrLen(Trim(destPath)) = 0
        throw ValueError("destPath must be a non-empty String", -1)

    ; Warn if destination exists — policy decision left to caller
    if FileExist(destPath)
        MsgBox("Warning: destination will be overwritten: '" destPath "'")

    FileCopy(sourcePath, destPath, true)

    ; Post-flight: FileCopy can fail silently — verify the result
    if !FileExist(destPath)
        throw Error("FileCopy silently failed: '" destPath "'", -1)
    return true
}

; USAGE
try {
    SafeReadFile("C:\missing.txt")          ; throws Error
} catch Error as e {
    MsgBox("IO error: " e.Message)
}

try {
    SafeSendKeys("Notepad", "Hello")        ; throws TargetError if not open
} catch TargetError as e {
    MsgBox("Window error: " e.Message)
}
```

### Performance Notes

**Guard ordering principle — cheapest check first eliminates wrong inputs earliest.**

Operation cost tiers:
- **O(1):** `IsSet()`, `is` operator, `Type()`, `StrLen()`, `IsObject()`, `HasMethod()`, `HasProp()` — always first in guard stacks
- **O(n):** `RegExMatch()`, `StrReplace()`, `SubStr()` on long strings — only after O(1) type and length screening passes
- **O(syscall):** `FileExist()`, `WinExist()`, `ProcessExist()` — one OS round-trip each; last in guard stacks; cache results across iterations

**HasMethod / HasProp are O(1)** — always prefer over `try { obj.Method() } catch MethodError {}`, which is expensive on the miss path. The try/catch form also hides the probe intent and complicates reading.

**Environment calls: one OS syscall per unique path** — cache `DirExist()` results in a `Map()` for batch operations. Calling `DirExist()` inside a tight loop over thousands of paths with the same parent directory turns O(1) into O(n) syscalls.

**Static pattern constants** — store regex strings as class-level static properties to avoid repeated string allocation in hot validation paths. Access via `ClassName.property`; the string is allocated once at class load time.

**ValidationBuilder lifecycle** — build chains once and call `.Validate()` once at submit time. Do not rebuild a `ValidationBuilder` on every keystroke in a live form; store the builder instance and add rules once during form initialisation.

**FormValidator scale** — `GetAllErrors()` iterates builders once in O(n) total error count with no nested scanning; the aggregation pattern remains performant even for forms with many fields.
```ahk
; --- PERFORMANCE GUIDELINES ---

; GUARD ORDERING PRINCIPLE: cheapest check first eliminates wrong inputs earliest
; O(1): IsSet(), 'is' operator, Type(), StrLen(), IsObject(), HasMethod()
; O(n): RegExMatch(), StrReplace(), SubStr() on long strings
; O(syscall): FileExist(), WinExist(), ProcessExist() — one OS round-trip each

EfficientGuard(val, pattern) {
    if !(Type(val) = "String")      ; ✓ O(1) type check — eliminates non-strings first
        throw TypeError("val must be String", -1)
    if StrLen(val) = 0              ; ✓ O(1) length — eliminates blanks before regex
        throw ValueError("val must not be empty", -1)
    if StrLen(val) > 500            ; ✓ O(1) upper bound — skip expensive regex on huge input
        throw ValueError("val exceeds maximum length of 500", -1)
    if !RegExMatch(val, pattern)    ; ✓ O(n) regex — only reached when all O(1) checks pass
        throw ValueError("val does not match required format", -1)
    return true
}

; HASMETHOD / HASPROP ARE O(1) — always prefer over try/catch for capability checks
; try { obj.Method() } catch MethodError {} is expensive on the miss path;
; HasMethod() probe is a hash lookup and costs nothing when the method exists.
EfficientCapabilityCheck(obj) {
    if !HasMethod(obj, "Process")   ; ✓ O(1) probe
        throw TypeError("missing Process()", -1)
    ; NOT: try { obj.Process() } catch MethodError  ; ✗ slower on miss
    obj.Process()
}

; ENVIRONMENT CALLS: ONE OS SYSCALL PER UNIQUE PATH — cache results for batch work
SafeBatchWrite(paths, content) {
    dirCache := Map()                           ; cache DirExist() results per directory
    for path in paths {
        SplitPath(path,, &dir)
        if !dirCache.Has(dir)
            dirCache[dir] := DirExist(dir)      ; ✓ one OS call per unique directory
        if !dirCache[dir]
            throw Error("directory missing: '" dir "'", -1)
        FileAppend(content, path, "UTF-8")
    }
}

; STATIC PATTERN CONSTANTS — store regex strings as class-level constants to avoid
; repeated string allocation in hot validation paths
class Patterns {
    static Email    := "i)^[a-z0-9._%+\-]+@[a-z0-9.\-]+\.[a-z]{2,}$"
    static Filename := '^[^\\/:*?"<>|]+$'
    static Phone    := "^\+?[0-9\s\-]{7,15}$"
    static Slug     := "^[a-z0-9][a-z0-9\-]*[a-z0-9]$"
}

; Access via ClassName.property — static members belong to the class scope
ValidateEmailFast(email) {
    if !RegExMatch(email, Patterns.Email)   ; ✓ static constant, no per-call alloc
        throw ValueError("invalid email: '" email "'", -1)
}

; VALIDATIONBUILDER LIFECYCLE: build chains once, call Validate() once at submit
; Do NOT rebuild a ValidationBuilder on every keystroke in a live form;
; store the builder instance and add rules once during form initialization.

; FORMVALIDATOR SCALE: for forms with many fields, FormValidator stays O(n) in
; total error count — GetAllErrors() iterates builders once, no nested scanning.
```

## TIER 6 — Fluent Validator Framework
> METHODS COVERED: `ValidationBuilder.For()` · `.Required()` · `.IsInteger()` · `.IsString()` · `.InRange()` · `.MinLength()` · `.MaxLength()` · `.Matches()` · `.Satisfies()` · `.Validate()` · `.GetErrors()` · `.HasErrors()` · `FormValidator()` · `.Add()` · `.GetAllErrors()` · `.IsValid()`

TIER 6 provides a production-ready `ValidationBuilder` class implementing a fluent interface for accumulating validation errors without throwing on the first failure, and a `FormValidator` aggregator for GUI form scenarios where users require seeing all field errors and reporting them together in a single combined error — the pattern that users require to fix all form issues in one round trip.
```ahk
; --- TIER 6: Fluent Validator Framework ---

; VALIDATIONBUILDER — non-throwing chainable rule accumulator
class ValidationBuilder {
    _errors := []
    _value  := ""
    _field  := "value"

    ; Static factory — entry point for every fluent chain
    static For(value, fieldName := "value") {
        vb        := ValidationBuilder()
        vb._value := value
        vb._field := fieldName
        return vb
    }

    ; CHAIN RULE: String must be non-blank
    Required() {
        if Type(this._value) = "String" && StrLen(Trim(this._value)) = 0
            this._errors.Push(this._field ": is required")
        return this
    }

    ; CHAIN RULE: Must be Integer type
    IsInteger() {
        if !(this._value is Integer)
            this._errors.Push(this._field
                ": must be Integer (got " Type(this._value) ")")
        return this
    }

    ; CHAIN RULE: Must be String type
    IsString() {
        if !(Type(this._value) = "String")
            this._errors.Push(this._field
                ": must be String (got " Type(this._value) ")")
        return this
    }

    ; CHAIN RULE: Integer must be within [minVal, maxVal]
    InRange(minVal, maxVal) {
        if (this._value is Integer)
            && (this._value < minVal || this._value > maxVal)
            this._errors.Push(this._field
                ": must be " minVal "-" maxVal " (got " this._value ")")
        return this
    }

    ; CHAIN RULE: String must be at least n characters
    MinLength(n) {
        if Type(this._value) = "String" && StrLen(this._value) < n
            this._errors.Push(this._field
                ": must be >= " n " chars (got " StrLen(this._value) ")")
        return this
    }

    ; CHAIN RULE: String must be at most n characters
    MaxLength(n) {
        if Type(this._value) = "String" && StrLen(this._value) > n
            this._errors.Push(this._field
                ": must be <= " n " chars (got " StrLen(this._value) ")")
        return this
    }

    ; CHAIN RULE: String must match a regex pattern
    Matches(pattern, description := "valid format") {
        if Type(this._value) = "String" && !RegExMatch(this._value, pattern)
            this._errors.Push(this._field ": must be " description)
        return this
    }

    ; CHAIN RULE: Value must satisfy a custom predicate (duck-typed callable)
    Satisfies(predFn, description := "custom rule") {
        EnsureCallable(predFn, "predFn")    ; reuse TIER 4 helper
        if !predFn(this._value)
            this._errors.Push(this._field ": must satisfy " description)
        return this
    }

    ; TERMINAL: throw ValueError with all collected errors (fail after full scan)
    Validate() {
        if this._errors.Length = 0
            return true
        msg := ""
        for i, err in this._errors
            msg .= (i > 1 ? "`n" : "") "- " err
        throw ValueError(msg, -1)
    }

    ; TERMINAL: return errors without throwing (for FormValidator aggregation)
    GetErrors() => this._errors
    HasErrors()  => this._errors.Length > 0
}

; FORMVALIDATOR — aggregates multiple ValidationBuilder instances for form validation
class FormValidator {
    _builders := []

    ; Register a builder with this form validator
    Add(builder) {
        this._builders.Push(builder)
        return this
    }

    ; Collect all errors across every registered builder
    GetAllErrors() {
        all := []
        for b in this._builders
            for err in b.GetErrors()
                all.Push(err)
        return all
    }

    ; True only if every registered builder has zero errors
    IsValid() {
        for b in this._builders
            if b.HasErrors()
                return false
        return true
    }

    ; Throw a single combined ValueError listing all field failures
    Validate() {
        errors := this.GetAllErrors()
        if errors.Length = 0
            return true
        msg := ""
        for i, e in errors
            msg .= (i > 1 ? "`n" : "") "- " e
        throw ValueError("Form validation failed:`n" msg, -1)
    }
}

; USAGE — GUI registration form with three fields, all errors shown together
SubmitRegistrationForm(username, email, age) {
    form := FormValidator()

    ; Build each field chain — errors accumulate without throwing
    usernameBuilder := ValidationBuilder.For(username, "username")
        .IsString()
        .Required()
        .MinLength(3)
        .MaxLength(20)
        .Matches("^\w+$", "alphanumeric or underscore only")
    form.Add(usernameBuilder)

    emailBuilder := ValidationBuilder.For(email, "email")
        .IsString()
        .Required()
        .Matches("i)^[a-z0-9._%+\-]+@[a-z0-9.\-]+\.[a-z]{2,}$", "valid email address")
    form.Add(emailBuilder)

    ageBuilder := ValidationBuilder.For(age, "age")
        .IsInteger()
        .InRange(13, 120)
    form.Add(ageBuilder)

    ; All three chains evaluated; one throw reports every failure at once
    try {
        form.Validate()
        MsgBox("Registration accepted for: " username)
    } catch ValueError as e {
        MsgBox("Please fix the following issues:`n" e.Message)
    }
}

; Test: all three fields fail — user sees all three errors simultaneously
SubmitRegistrationForm("AB", "not-an-email", 200)

; Test: single field fails
SubmitRegistrationForm("valid_user", "user@example.com", 200)

; HELPER referenced by Satisfies() — copy from TIER 4 into shared scope
EnsureCallable(handler, name := "handler") {
    if !IsObject(handler)
        throw TypeError(name " must be a callable object, got " Type(handler), -1)
    if !HasMethod(handler, "Call")
        throw TypeError(name " must be callable (missing Call method)", -1)
    return true
}
```

## ANTI-PATTERNS

| Pattern | Wrong | Correct | LLM Common Cause |
|---------|-------|---------|------------------|
| String type detection with `is` | `if val is String` | `if Type(val) = "String"` | `is` works for Integer/Float/Number/ClassName — LLMs assume it covers all types; `is String` is syntactically valid but always false |
| Unset vs empty-string conflation | `if param = ""` to detect missing param | `if !IsSet(param)` | Most language training data treats missing and empty as equivalent states; AHK v2 treats them as distinct |
| Base error class for typed violations | `throw Error("bad type")` | `throw TypeError("expected Integer, got " Type(x), -1)` | AHK v1 had only the base `Error` class; LLMs default to it and omit the -1 stack offset |
| Class-name assertion for capability check | `if Type(obj) = "MyClass"` | `if !HasMethod(obj, "Process")` | Static-typed language habits; LLMs assert concrete types by default, breaking all polymorphic callers |
| `??` as an unset guard substitute | `param ?? throw TypeError(...)` | `if !IsSet(param) throw TypeError(...)` then `val := param ?? fallback` | Conflation of nullish coalescing (JS/Kotlin) with null guard idioms — `??` never throws, it silently swallows the unset state |
| Property assignment before guard block in `__New` | `this.X := x` before type checks | Complete guard block first, then all property assignments | OOP training from other languages defaults to eager assignment; AHK v2 has no constructor rollback mechanism |
| Missing -1 stack offset in throw | `throw TypeError("msg")` | `throw TypeError("msg", -1)` | AHK-specific stack depth parameter absent in all other language training data; LLMs generate the two-argument form inconsistently |
| Inline Type() string comparison for numeric types | `if Type(x) = "Integer"` | `if x is Integer` | Confusion between the two type-check mechanisms — `Type()` = "String" is correct for strings but `is` is preferred for numeric and class checks |

## SEE ALSO

> This module does NOT cover: custom error class hierarchies, try/catch control flow structure, and re-throw patterns → see Module_Errors.md
> This module does NOT cover: reusable validator base classes with `__New`/`__Delete` lifecycle, class inheritance for validator types, or prototype-level method sharing → see Module_Classes.md
> This module does NOT cover: AHK v2 core syntax rules, coding standards, and cognitive tier escalation logic → see Module_Instructions.md

- `Module_Errors.md` — try/catch patterns, typed error catching (`catch TypeError as e`), custom error class hierarchies, and structured error re-throw strategies that complement validation guards.
- `Module_Classes.md` — building reusable validator base classes with `__New` lifecycle management, class inheritance for specialised validator types, and prototype-level method sharing across `ValidationBuilder` descendants.
- `Module_Instructions.md` — core AHK v2 syntax rules, coding standards, and cognitive tier escalation logic that governs when TIER 1 type checks vs TIER 6 fluent frameworks are appropriate for a given scenario.