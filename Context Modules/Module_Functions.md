# Module_Functions.md
<!-- DOMAIN: Functions — named function declaration, parameterization, variable scoping, and encapsulation in AHK v2 -->
<!-- SCOPE: OOP method definitions, class-based encapsulation, and DLL/COM callback function pointers are not covered — see Module_Classes.md and Module_DllCallAndMemory.md -->
<!-- TRIGGERS: function, IsSet(), ByRef, &, variadic, static, global, "define function", "declare function", "return value", "optional param", "by reference", "variable scope", "closure", "nested function", "multi-return", "output param", "unset param", "spread args" -->
<!-- CONSTRAINTS: No space may appear between a function name and its `(` at any call site — AHK v2 parses `Name (...)` as a variable dereference, not a function call. `=>` fat-arrow is single-expression only and must never be combined with `{ }` braces. All function-body variables are local by default; writing to an outer global requires explicit `global VarName` before first assignment. ByRef `&` must appear at both the definition and the call site — omitting it at the call site throws TypeError, not a silent copy. -->
<!-- CROSS-REF: Module_Classes.md, Module_Errors.md, Module_Arrays.md, Module_Objects.md, Module_DllCallAndMemory.md -->
<!-- VERSION: AHK v2.0+ -->

## V1 → V2 BREAKING CHANGES

| v1 pattern (LLM commonly writes) | v2 correct form | Consequence |
|----------------------------------|-----------------|-------------|
| `ByRef varName` keyword in parameter definition | `&varName` prefix on parameter | Load-time parse error — the `ByRef` keyword does not exist in v2 |
| Implicit global write inside function (no declaration) | `global VarName` declared before first assignment in function body | Silent local shadow — outer variable is unchanged, no error raised; hardest scope bug to diagnose |
| `FuncName (args)` with space before `(` | `FuncName(args)` — name and `(` immediately adjacent | v2 parser treats it as a variable dereference followed by a parenthesized expression — silent wrong behavior or NameError |
| No unset-param syntax; `if param == ""` to detect absent arg | `param?` in signature + `IsSet(param)` guard before any access | Without `?`, the parameter is required; if truly optional and accessed without guard, throws UnsetError |
| Variadic via legacy `args` Object without `*` syntax | `args*` suffix on last parameter in definition; `arr*` spread at call site | v1 variadic used a different object structure; v2 collects into a typed 1-based Array via `*` |
| `return` on one line, expression on next line (continuation style) | `return expression` on a single line | Bare `return` exits immediately returning ""; value on next line is dead code — silent wrong return |
| `f := (a) => { return a + 1 }` fat-arrow with brace block | `f(a) { return a + 1 }` — named function with `{ }` body | AHK v2 grammar forbids `=> { }`; load-time parse failure with no deferred error |

## API QUICK-REFERENCE

### Parameter Modifier Syntax
| Modifier | Form | Notes |
|----------|------|-------|
| Default value | `param := expr` | Caller may omit; default expression is evaluated at call time when the argument is absent |
| Unset / optional | `param?` | Caller may omit entirely; **must** guard with `IsSet(param)` before any access or throws UnsetError |
| Variadic collector | `param*` | Collects all extra arguments into a typed 1-based Array; must be the last parameter in the list |
| ByRef output | `&param` | Allows function to write back into caller's variable; `&` required at both definition and every call site |

### Parameter Order Rule
| Position | Modifier | Example |
|----------|----------|---------|
| 1st | Required (no modifier) | `req` |
| 2nd | Default value | `opt := 0` |
| 3rd | Unset optional | `maybe?` |
| Last | Variadic | `rest*` |

Any other order causes a load-time parse error.

### Scope Declaration Keywords
| Keyword | Form | Notes |
|---------|------|-------|
| `global` | `global VarName` | Required before first **write** to an outer global inside a function body; declare at top |
| `static` | `static VarName := init` | Persists across calls; initializer runs exactly once on first invocation — never use for values that must reset per call |
| `local` | `local VarName` | Redundant (local is the default) but legal for clarity |

### IsSet
| Function | Signature | Notes |
|----------|-----------|-------|
| `IsSet()` | `IsSet(var)` | Returns 1 if `var` has a value; 0 if unset — mandatory guard before accessing any `?` parameter |

### Func Object (function references)
| Property / Constructor | Signature | Notes |
|-----------------------|-----------|-------|
| `Func()` | `Func(name)` | Retrieve a Func reference object by name string |
| `.Name` | `.Name` | The function's name as declared |
| `.MinParams` | `.MinParams` | Minimum number of arguments required |
| `.MaxParams` | `.MaxParams` | Number of formally-declared parameters for user-defined functions (excluding the variadic collector); for variadic functions, parameters above this count overflow into the collector |
| `.IsVariadic` | `.IsVariadic` | 1 if the function is variadic, 0 otherwise; use this — not MaxParams — to detect when there is no fixed upper bound on parameter count |
| `.IsBuiltIn` | `.IsBuiltIn` | 1 for built-in functions; 0 for user-defined |

### Built-in Utility Functions Used in Examples
| Function | Signature | Notes |
|----------|-----------|-------|
| `MsgBox()` | `MsgBox(text)` | Used throughout as output demonstration — not a Functions-domain API |
| `IsSet()` | `IsSet(var)` | See IsSet section above |
| `StrSplit()` | `StrSplit(str, delims?, omit?, maxParts?)` | Returns a typed 1-based Array of substrings |
| `Integer()` | `Integer(val)` | Converts a value to an integer; throws ValueError on non-numeric input |
| `Float()` | `Float(val)` | Converts a value to a floating-point number |
| `Trim()` | `Trim(str, chars?)` | Removes leading/trailing characters (space by default) |
| `Mod()` | `Mod(dividend, divisor)` | Returns remainder; works with floats |
| `Map()` | `Map(key, val, ...)` | Creates a Map object — used for multi-return and Func-reference containers |
| `FileAppend()` | `FileAppend(text, path, encoding?)` | Appends text to file; encoding should always be explicit |
| `A_Now` | Built-in var | Current local date-time in YYYYMMDDHH24MISS format |

## AHK V2 CONSTRAINTS

- **No space between function name and `(`** — `MyFunc (x)` is parsed as a variable dereference followed by a parenthesized expression, not a function call; this produces a silent logic error or a NameError depending on context. This rule applies at every call site: definitions, nested calls, and method chains — zero whitespace between name and `(` without exception.

- **`=>` fat-arrow is single-expression only** — never pair `=>` with `{ }` braces. For any multi-statement logic, define a standard named function with `FuncName(params) { body }`. Combining `=>` with `{ }` is a hard AHK v2 grammar constraint; the parser fails at load time with no deferred error.

- **All function-body variables are local by default** — writing to an outer global without `global VarName` declared first creates a silent local shadow. The outer variable appears unchanged and no error is raised, making this the most common source of scope bugs. Declare `global VarName` at the top of the function body, before the first access.

- **ByRef `&` must appear at both the definition and every call site** — omitting `&` at the call site throws a TypeError at runtime rather than silently passing a copy. AHK v2 requires explicit opt-in from the caller; it does not fall back to copy semantics.

- **`return` and its value must be on the same line** — a bare `return` immediately exits the function returning an empty string; any expression on a subsequent line becomes dead code. There is no implicit return of the last expression.

- **`?` unset parameters must be guarded with `IsSet()` before any access** — touching an unset parameter without a guard throws an UnsetError. Always place `if !IsSet(param)` before the first use of any `?`-declared parameter.

- **`static` variables initialize exactly once** — the initializer (`:= value`) executes only on the very first invocation of the function. Never use `static` for values that must reset per call; use a regular local variable instead.

- **Parameter order is mandatory: required → default → unset `?` → variadic `*`** — placing a required parameter after a defaulted or optional one is a load-time parse error.

Safe-access priority order for optional / absent parameters:
  1. `:=` default value — caller omits the argument; the known safe default fills in automatically, no guard needed
  2. `param?` + `IsSet(param)` guard — when absent vs present requires different branch logic in the body
  3. `&param := default` — optional ByRef output parameter; caller may omit with a defined fallback
  4. `try/catch` around the call site — only when the exception message itself carries diagnostic information needed by the caller

Paired prohibitions and their positive alternatives:
- ✗ `MyFunc (x)` — parsed as variable dereference, not a function call; silent wrong behavior
- ✓ `MyFunc(x)` — name and `(` adjacent is the only valid call form

- ✗ `f := (a) => { return a + 1 }` — load-time parse failure
- ✓ `f(a) { return a + 1 }` — named function with `{ }` body for any multi-statement logic

- ✗ Assigning to outer global name inside function without `global` declaration — silent local shadow
- ✓ `global gVar` declared at the top of the function body before first write

- ✗ `Swap(x, y)` when Swap defines `(&a, &b)` — throws TypeError at call site
- ✓ `Swap(&x, &y)` — `&` mirrored at every call site

- ✗ `return` on one line, computed value on the next — bare return exits; value is dead code
- ✓ `return computedValue` — return and value on the same line

## TIER 1 — Basic Declaration and Calling
> METHODS COVERED: named function definition · `return` · `MsgBox()` · `StrSplit()` · `IsSet()`

A named function in AHK v2 consists of a name immediately adjacent to `(` (zero whitespace), a parameter list, a brace-enclosed body, and `return expr` on a single line. Functions defined at file scope are hoisted — call order relative to definition does not matter. A function with no explicit `return`, or with a bare `return`, yields an empty string, not `0` or `undefined`. The declaration template below shows all parameter modifier forms together; each form is covered in depth in TIER 3.
```ahk
; ============================================================
; TIER 1 — Basic Declaration and Calling
; ============================================================

; ---- STANDARD NAMED FUNCTION TEMPLATE ----
; ✓ Shows all parameter modifier positions — FunctionName immediately touches (
FunctionName(requiredParam, optionalParam := "default", unsetParam?, extras*) {
    ; Declare any globals this function writes at the very top
    ; global gSomeGlobal

    ; ✓ IsSet() guard is mandatory before any access to a ? parameter
    if !IsSet(unsetParam)
        unsetParam := ""

    ; ✓ static initializer runs once; value persists across all subsequent calls
    static callCount := 0
    callCount += 1

    ; ... body logic ...

    return requiredParam   ; ✓ return and value on same line
}

; ---- ByRef OUTPUT PARAMETER TEMPLATE ----
; ✓ & prefix on both definition and call site — caller opts in explicitly
ParsePair(str, &outA, &outB) {
    parts := StrSplit(str, ",")
    outA := parts[1]
    outB := parts[2]
}
ParsePair("10,20", &valA, &valB)   ; ✓ & mirrored at call site

; ---- BASIC DECLARATION AND CALL ----
; ✓ Name touches ( with no whitespace — the only valid call form in AHK v2
Square(n) {
    return n * n           ; ✓ return + value on same line
}

result := Square(7)        ; ✓ correct call — no space before (
MsgBox(result)             ; 49

; ✗ Space before ( causes silent parse failure — treated as variable dereference
; result2 := Square (7)    ; ✗ → NameError or wrong evaluation, never a function call

; ---- MULTIPLE PARAMETERS with MULTIPLE RETURN PATHS ----
; ✓ Each return branch carries its value on the same line — no bare returns
Clamp(value, lo, hi) {
    if value < lo
        return lo
    if value > hi
        return hi
    return value
}
MsgBox(Clamp(15, 0, 10))   ; 10

; ---- MISSING RETURN yields empty string "" — not 0 or undefined ----
; ✓ Knowing the default return value prevents silent type errors in callers
NoReturn() {
    x := 1 + 1
}
val := NoReturn()
MsgBox(val == "" ? "empty" : val)   ; empty

; ---- RETURN VALUE IS ANY EXPRESSION ----
; ✓ Concatenation, arithmetic, or nested call results are all valid return expressions
FullName(first, last) {
    return first " " last
}
MsgBox(FullName("Ada", "Lovelace"))   ; Ada Lovelace

; ---- HOISTING: call order does not matter ----
; ✓ Functions at file scope are hoisted — can be called before their definition line
MsgBox(Double(5))   ; 10  — defined below, still callable

Double(x) {
    return x * 2
}
```

## TIER 2 — Variable Scope: Local, Global, Static
> METHODS COVERED: `global` declaration · `static` declaration · `local` declaration

Every variable inside a function is local by default — it does not share state with any outer variable of the same name. Writing to an outer global requires `global VarName` declared before the first assignment inside the function body. Static variables persist across calls but are scoped to the function; their initializer runs exactly once on first invocation. Independent functions each have their own independent static state even if they declare statics with the same name.
```ahk
; ============================================================
; TIER 2 — Variable Scope: Local, Global, Static
; ============================================================

; ---- LOCAL (default) ----
; ✓ Inner x is a completely separate variable — outer x is untouched without any declaration
LocalDemo() {
    x := 99
    return x
}
x := 0
LocalDemo()
MsgBox(x)   ; 0 — outer x untouched; inner x is an independent local

; ---- GLOBAL ----
; ✓ global declaration before first write grants write access to the outer variable
gCounter := 0

IncrementGlobal() {
    global gCounter        ; ✓ must appear before any access to gCounter
    gCounter += 1
}
IncrementGlobal()
IncrementGlobal()
MsgBox(gCounter)           ; 2

; ✗ Assigning without global declaration creates a silent local shadow — no error raised
IncrementShadow() {
    gCounter += 1          ; ✗ operates on a local "gCounter", not the outer one
}
IncrementShadow()
MsgBox(gCounter)           ; still 2 — outer unchanged; local shadow is silently discarded

; ---- STATIC ----
; ✓ := initializer runs exactly once on first call; retained value survives between calls
CountCalls() {
    static n := 0          ; := 0 runs only the first time CountCalls() is invoked
    n += 1
    return n
}
MsgBox(CountCalls())   ; 1
MsgBox(CountCalls())   ; 2
MsgBox(CountCalls())   ; 3

; ✓ Each function has its own independent static — same name does not conflict
OtherCounter() {
    static n := 0
    n += 1
    return n
}
MsgBox(OtherCounter())   ; 1  — completely unrelated to CountCalls()

; ---- LOCAL declaration (explicit) ----
; ✓ Explicit local is redundant but legal — useful for documentation of intent
Explicit() {
    local result := "defined locally"
    return result
}
```

## TIER 3 — Parameter Modifiers: Default, Unset, Variadic
> METHODS COVERED: `:=` default value · `?` unset marker · `*` variadic collector · `arr*` spread · `IsSet()` · `StrSplit()` · `Trim()`

AHK v2 provides three parameter modifier syntaxes with a mandatory positional order: required parameters first, then defaulted (`:=`), then unset (`?`), then the variadic collector (`*`) last. The `?` modifier marks a parameter as optionally omittable but does not provide a fallback — `IsSet()` must guard every access to a `?` parameter. The `*` modifier at the call site (`arr*`) spreads an Array into individual arguments with zero copy overhead.
```ahk
; ============================================================
; TIER 3 — Parameter Modifiers: Default, Unset, Variadic
; ============================================================

; ---- DEFAULT VALUE with := ----
; ✓ := provides a fallback value — no guard needed, caller may simply omit the argument
Repeat(str, times := 1) {
    result := ""
    loop times
        result .= str
    return result
}
MsgBox(Repeat("ha"))        ; ha      — times defaults to 1
MsgBox(Repeat("ha", 3))     ; hahaha

; ---- UNSET PARAMETER with ? ----
; ✓ ? marks the parameter as truly omittable — IsSet() guard is mandatory before any access
Describe(name, age?) {
    if !IsSet(age)
        return name " (age unknown)"
    return name " age " age
}
MsgBox(Describe("Alice"))        ; Alice (age unknown)
MsgBox(Describe("Bob", 30))      ; Bob age 30

; ✗ Accessing unset param without IsSet guard throws UnsetError at runtime — never omit the guard
; BadDescribe(name, age?) {
;     return name " " age        ; ✗ if age not passed → UnsetError
; }

; ---- VARIADIC PARAMETER with * ----
; ✓ * collects all trailing arguments into a typed 1-based Array; must be the last parameter
Sum(nums*) {
    total := 0
    for n in nums
        total += n
    return total
}
MsgBox(Sum(1, 2, 3, 4, 5))   ; 15
MsgBox(Sum())                 ; 0  — nums is an empty Array when no arguments are passed

; ✓ Fixed + variadic: fixed params are resolved first, the rest collected into the Array
Prefix(label, values*) {
    out := label ": "
    for v in values
        out .= v " "
    return Trim(out)
}
MsgBox(Prefix("nums", 10, 20, 30))   ; nums: 10 20 30

; ---- VARIADIC SPREAD at CALL SITE ----
; ✓ arr* unpacks an Array into individual arguments — zero-copy forwarding, no intermediate Array
args := [7, 8, 9]
MsgBox(Sum(args*))   ; 24  — array contents spread into variadic params

; ---- PARAMETER ORDER RULE ----
; ✓ required → default → unset(?) → variadic(*) — any other order is a load-time parse error
Combined(req, opt := 0, maybe?, rest*) {
    result := req + opt
    if IsSet(maybe)
        result += maybe
    for v in rest
        result += v
    return result
}
MsgBox(Combined(10))               ; 10
MsgBox(Combined(10, 5))            ; 15
MsgBox(Combined(10, 5, 3))         ; 18
MsgBox(Combined(10, 5, 3, 1, 2))   ; 21
```

## TIER 4 — ByRef Parameters: Output Params and In-Place Mutation
> METHODS COVERED: `&` ByRef definition · `&` ByRef call site · `StrSplit()` · `Integer()` · `Float()`

ByRef allows a function to write back into the caller's variable. Both the function definition and every call site must carry `&`; omitting `&` at the call site throws a TypeError at runtime — it does not silently pass a copy. ByRef is the primary AHK v2 strategy for output parameters (returning a status flag AND populating result data simultaneously) and for in-place reassignment of caller-owned variables.
```ahk
; ============================================================
; TIER 4 — ByRef Parameters: Output Params and In-Place Mutation
; ============================================================

; ---- BASIC ByRef: & on definition and call site ----
; ✓ & on both sides — function writes directly into the caller's x and y variables
Swap(&a, &b) {
    temp := a
    a    := b
    b    := temp
}

x := 10, y := 20
Swap(&x, &y)                ; ✓ & at call site — required, not optional
MsgBox(x " / " y)           ; 20 / 10

; ✗ Missing & at call site throws TypeError — AHK v2 requires explicit caller opt-in
; Swap(x, y)                ; ✗ → TypeError — x and y are NOT swapped

; ---- OUTPUT PARAMETER pattern ----
; ✓ Use when a function needs to return a status flag AND populate result data simultaneously
TryParseInt(str, &out) {
    if str ~= "^\d+$" {
        out := Integer(str)
        return true
    }
    out := 0
    return false
}

if TryParseInt("42", &parsed)
    MsgBox("Parsed: " parsed)   ; Parsed: 42

if !TryParseInt("abc", &parsed)
    MsgBox("Failed, out=" parsed)   ; Failed, out=0

; ---- MULTIPLE OUTPUT PARAMETERS ----
; ✓ Each & param is an independent write-back channel into the caller's scope
ParseCoord(str, &outX, &outY) {
    parts := StrSplit(str, ",")
    outX  := Float(parts[1])
    outY  := Float(parts[2])
}

ParseCoord("3.5,7.2", &px, &py)
MsgBox(px " / " py)   ; 3.5 / 7.2

; ---- IN-PLACE MUTATION of a complex object ----
; ✓ ByRef on an Array reference lets the function reassign the caller's variable itself
DoubleAll(&arr) {
    newArr := []
    for v in arr
        newArr.Push(v * 2)
    arr := newArr   ; ✓ reassigns the caller's variable to the new Array
}

data := [1, 2, 3]
DoubleAll(&data)
MsgBox(data[1] " " data[2] " " data[3])   ; 2 4 6

; ---- ByRef with default value ----
; ✓ Optional output params: default lets callers omit them without TypeError
GetBounds(arr, &outMin := 0, &outMax := 0) {
    outMin := arr[1]
    outMax := arr[1]
    for v in arr {
        if v < outMin
            outMin := v
        if v > outMax
            outMax := v
    }
}

GetBounds([3, 1, 4, 1, 5, 9], &mn, &mx)
MsgBox(mn " to " mx)   ; 1 to 9
```

## TIER 5 — Nested Functions: Lexical Scope and Closures
> METHODS COVERED: nested function declaration · lexical capture · closure factory pattern · `Map()` (Func reference container) · `FileAppend()` · `StrSplit()` · static memoization · `MsgBox()`

AHK v2 nested (inner) functions have lexical scope: they can read and write the enclosing function's local variables by name without any special syntax. An outer function may return a reference to an inner function, creating a closure where each invocation of the outer function produces an independent inner Func with its own captured copy of the enclosing locals. Nested functions are invisible outside the enclosing function, keeping the public API surface small and preventing name collisions.
```ahk
; ============================================================
; TIER 5 — Nested Functions: Lexical Scope and Closures
; ============================================================

; ---- NESTED HELPER: scope isolation ----
; ✓ Validate() is only visible inside ProcessPositive() — reduces public name pollution
ProcessPositive(data) {
    Validate(item) {
        return (item is Number) && (item > 0)
    }

    result := []
    for item in data {
        if Validate(item)
            result.Push(item)
    }
    return result
}

clean := ProcessPositive([3, -1, 0, 7, 2])
MsgBox(clean[1] " " clean[2] " " clean[3])   ; 3 7 2

; ---- LEXICAL CAPTURE: inner function reads enclosing function's local ----
; ✓ Each call to BuildPrefixer captures a distinct 'prefix' — independent closures
BuildPrefixer(prefix) {
    Apply(value) {
        return prefix ": " value   ; 'prefix' is captured from BuildPrefixer's scope
    }
    return Apply   ; ✓ return a Func reference — Apply survives beyond BuildPrefixer's call
}

errFmt := BuildPrefixer("ERROR")
wrnFmt := BuildPrefixer("WARN")
MsgBox(errFmt("file missing"))   ; ERROR: file missing
MsgBox(wrnFmt("low memory"))     ; WARN: low memory

; ---- CLOSURE FACTORY: each outer call produces an independent closure ----
; ✓ n is captured per-call of MakeAdder — Add5 and Add10 hold independent captured values
MakeAdder(n) {
    Add(x) {
        return x + n   ; n is captured per-call of MakeAdder — independent per instance
    }
    return Add
}

Add5  := MakeAdder(5)
Add10 := MakeAdder(10)
MsgBox(Add5(3))    ; 8
MsgBox(Add10(3))   ; 13

; ---- NESTED WRITE: inner function modifies enclosing function's local ----
; ✓ Accumulate and GetTotal share 'total' from BuildAccumulator's local scope
BuildAccumulator() {
    total := 0

    Accumulate(n) {
        total += n       ; ✓ writes to outer local 'total' directly
    }

    GetTotal() {
        return total     ; ✓ reads outer local 'total' directly
    }

    return Map("add", Accumulate, "get", GetTotal)
}

acc := BuildAccumulator()
acc["add"](10)
acc["add"](25)
MsgBox(acc["get"]())   ; 35

; ✗ Nested functions are NOT accessible outside the enclosing function
; Validate("x")   ; ✗ → NameError — Validate only exists inside ProcessPositive()
```

### Performance Notes

**Static memoization — O(1) cache vs O(n) recomputation:** Use `static cache := Map()` inside a pure function to avoid recomputing results for previously seen inputs. The `static` initializer runs once; all subsequent cache hits are O(1) Map lookups. This pattern is safe **only for pure functions** — functions with side effects must not be memoized via static state, as the cached result will be returned on repeat calls without re-executing the side effect.

**Early-return guard clauses:** Short-circuit before expensive work by checking trivial or degenerate cases at the top of the function. Each guard `return` costs nothing compared to the expensive path; the expensive logic is only reached when necessary.

**ByRef for large string primitives:** Primitive values (strings, numbers) are copied on function call. For large strings, passing via `&param` eliminates the copy overhead. Objects (Map, Array) are reference types — passing without ByRef already copies only the reference pointer, not the object content. Use ByRef on objects only when the function needs to **reassign** the caller's variable binding, not merely mutate its content.

**Variadic spread (`arr*`) is zero-copy:** Spreading an Array into a variadic call (`fn(arr*)`) does not create an intermediate copy. Prefer it over manually passing individual elements when the array size is dynamic or unknown.

**Prefer built-in methods over custom AHK loops:** `StrSplit()`, `RegExMatch()`, `Sort()` are implemented in native C and outperform equivalent AHK-script loops for the vast majority of input sizes. Reach for them before writing manual character-iteration loops.

**Static constant tables:** Initialize lookup Maps with `static m := Map(...)` — the Map is constructed once on the first call and reused at O(1) cost for all subsequent lookups.
```ahk
; ============================================================
; TIER 5 — Performance Patterns
; ============================================================

; ---- STATIC MEMOIZATION: O(1) cache lookup vs O(n) recomputation ----
; ✓ cache persists across calls — Fibonacci(30) is computed once, then O(1) on all repeats
Fibonacci(n) {
    static cache := Map()
    if cache.Has(n)
        return cache[n]         ; ✓ O(1) cache hit — no recomputation
    if n <= 1 {
        cache[n] := n
        return n
    }
    result := Fibonacci(n - 1) + Fibonacci(n - 2)
    cache[n] := result
    return result
}
MsgBox(Fibonacci(30))   ; fast — result is cached after first call

; ---- EARLY RETURN: short-circuit guard clauses ----
; ✓ Guard clauses eliminate the expensive path for trivial inputs at zero cost
ProcessLargeArray(arr) {
    if arr.Length == 0          ; guard: bail immediately on empty input
        return []
    if arr.Length == 1          ; guard: trivial case avoids full processing
        return [arr[1]]
    ; ... expensive sort / transform only reached for non-trivial inputs ...
    return arr
}

; ---- ByRef FOR LARGE STRINGS: avoids copying primitive values ----
; ✓ ByRef: no copy overhead — the reference pointer is passed, not the content
SumValues(&bigMap) {
    total := 0
    for , v in bigMap
        total += v
    return total
}

; ✗ Without ByRef — entire string/primitive is copied on each call for primitive types
; SumValuesCopy(bigMap) { ... }   ; ✗ full copy of primitive value on every call

; ---- VARIADIC SPREAD: zero-copy argument forwarding ----
; ✓ segments* expands the Array contents without creating any intermediate copy
Log(level, parts*) {
    line := level ": "
    for p in parts
        line .= p
    FileAppend(line "`n", "app.log")
}

segments := ["user=", "alice", " action=", "login"]
Log("INFO", segments*)   ; ✓ spreads without building a new intermediate array

; ---- PREFER BUILT-IN METHODS over custom AHK loops ----
; ✓ StrSplit is native C — faster than manual AHK character loops for most inputs
words := StrSplit("one,two,three", ",")

; ✗ Avoid manual character-by-character parsing when a native built-in exists
; ManualSplit(str, delim) { ... }   ; ✗ slower than StrSplit for virtually all real inputs

; ---- STATIC CONSTANT TABLES: initialize once, read O(1) per call ----
; ✓ Map is constructed once on first call — all subsequent lookups are O(1)
GetDayName(n) {
    static days := Map(
        1, "Mon", 2, "Tue", 3, "Wed",
        4, "Thu", 5, "Fri", 6, "Sat", 7, "Sun"
    )
    return days.Has(n) ? days[n] : "Unknown"
}
MsgBox(GetDayName(3))   ; Wed
```

## TIER 6 — Multi-Return and Pure vs Side-Effect Separation
> METHODS COVERED: `Map()` multi-return · Array multi-return · `StrSplit()` · `FileAppend()` · `A_Now` · `Mod()` · `MsgBox()` · pure function pattern · side-effect wrapper pattern

Returning more than one value uses either a Map (preferred for 3+ named fields — consumer uses self-documenting key access) or an Array (best for exactly 2–3 positional values). The complementary architectural discipline is separating pure functions (deterministic, no external interaction, safely memoizable) from side-effect functions (explicitly named, isolated, returning a success flag). Pure cores can be independently tested and statically cached; side-effect wrappers cannot.
```ahk
; ============================================================
; TIER 6 — Multi-Return and Pure vs Side-Effect Separation
; ============================================================

; ---- MULTI-RETURN via Map (named fields — preferred for 3+ values) ----
; ✓ Map keys give consumers self-documenting field names at the access site
AnalyzeList(arr) {
    if arr.Length == 0
        return Map("count", 0, "sum", 0, "mean", 0, "min", "", "max", "")

    mn := arr[1], mx := arr[1], total := 0
    for v in arr {
        total += v
        if v < mn
            mn := v
        if v > mx
            mx := v
    }
    return Map(
        "count", arr.Length,
        "sum",   total,
        "mean",  total / arr.Length,
        "min",   mn,
        "max",   mx
    )
}

stats := AnalyzeList([4, 7, 2, 9, 1])
MsgBox("count=" stats["count"] " mean=" stats["mean"])   ; count=5 mean=4.6

; ---- MULTI-RETURN via Array (positional — best for exactly 2–3 values) ----
; ✓ Array index access is appropriate when position is the natural, obvious contract
SplitFullName(fullName) {
    parts := StrSplit(fullName, " ", , 2)
    return [parts[1], parts[2]]
}

name := SplitFullName("Grace Hopper")
MsgBox(name[1] " / " name[2])   ; Grace / Hopper

; ---- PURE FUNCTION: no side effects, deterministic ----
; ✓ Same input ALWAYS produces same output — safe to memoize via static cache
PureScale(value, factor) {
    return value * factor
}

; ✓ Pure — builds a new Array without mutating input; caller's arr is unchanged
PureFilter(arr, predFn) {
    result := []
    for v in arr {
        if predFn(v)
            result.Push(v)
    }
    return result
}

evens := PureFilter([1, 2, 3, 4, 5, 6], (n) => Mod(n, 2) == 0)
MsgBox(evens[1] " " evens[2] " " evens[3])   ; 2 4 6

; ---- SIDE-EFFECT FUNCTION: explicitly named and isolated ----
; ✓ Side effects are documented, isolated in one place, and return a success flag
SaveRecord(&record, filePath) {
    ; Side effects: stamps record with timestamp, appends to disk
    record["savedAt"] := A_Now
    try {
        FileAppend(record["data"] "`n", filePath)
        return true
    } catch OSError as e {
        return false
    }
}

; ---- SEPARATION PATTERN: pure core + thin side-effect wrapper ----
; ✓ FormatLogLine is pure — independently testable and cacheable
FormatLogLine(level, msg) {
    return "[" A_Now "] " level ": " msg
}

; ✓ WriteLog is the side-effect wrapper — all external interaction is isolated here
WriteLog(level, msg, logPath) {
    line := FormatLogLine(level, msg)   ; ✓ pure computation isolated
    FileAppend(line "`n", logPath)       ; ✓ side effect isolated here
}

; ---- COMBINING Map multi-return with ByRef (TIER 4+6 composite) ----
; ✓ Functions that both mutate and report combine ByRef output with Map return value
NormalizeAndReport(arr, &outModified) {
    total   := 0
    outModified := false
    for v in arr
        total += v
    if total == 0
        return Map("status", "zero-sum", "result", arr)
    outModified := true
    result := []
    for v in arr
        result.Push(v / total)
    return Map("status", "ok", "result", result)
}

raw := [1, 2, 3, 4]
info := NormalizeAndReport(raw, &wasChanged)
MsgBox("modified=" wasChanged " status=" info["status"])
; modified=1 status=ok
```

## ANTI-PATTERNS

| Pattern | Wrong | Correct | LLM Common Cause |
|---------|-------|---------|------------------|
| Space before `(` at call site | `MyFunc (x)` | `MyFunc(x)` | Most languages allow optional whitespace before call `()`; AHK v2 parses the space as a variable dereference instead |
| Fat-arrow combined with brace block | `f := (a) => { return a + 1 }` | `f(a) { return a + 1 }` | JavaScript/Python training — both allow `=>` or lambda with `{}` bodies; AHK v2 grammar forbids the combination |
| Silent global shadow | Assigning to outer var name without `global` declaration | `global gVar` at top of function before first write | AHK v1 allowed implicit global access inside functions; v2 removed this — LLMs trained on v1 omit the declaration |
| Missing `&` at ByRef call site | `Swap(x, y)` when Swap defines `(&a, &b)` | `Swap(&x, &y)` | AHK v1 used `ByRef` keyword only in the definition; callers had no opt-in marker — LLMs mirror that one-sided pattern |
| `return` split across lines | `return` on one line, value expression on next | `return computedValue` on one line | C/Python training — both allow expression continuation after `return`; AHK v2 bare `return` exits immediately |
| Unguarded `?` parameter access | `return name " " age` when `age?` may be unset | `if !IsSet(age)` guard before first use of `age` | v1 had no `?` unset syntax; LLMs default to treating optional as "may be empty string" rather than "genuinely absent" |
| `static` for per-call reset values | `static result := ""` that must clear each invocation | `local result := ""` — regular local variable | Confusion between static (persists) and local (resets); both look like `var := init` on paper, only semantics differ |

## SEE ALSO

> This module does NOT cover: OOP method definitions, class properties, and class-based encapsulation → see Module_Classes.md
> This module does NOT cover: DLL callback function pointers and COM method references → see Module_DllCallAndMemory.md and Module_SystemAndCOM.md
> This module does NOT cover: try/catch error handling patterns and custom error classes inside function bodies → see Module_Errors.md
> This module does NOT cover: consuming and iterating Array/Map return values from multi-return functions → see Module_Arrays.md and Module_Objects.md

- `Module_Classes.md` — method definitions, property accessors (`get`/`set`), and class-based encapsulation that build on the named function patterns in this module.
- `Module_Errors.md` — try/catch patterns for function body error handling, I/O exception recovery, and custom error class definitions.
- `Module_Arrays.md` — consuming, iterating, and transforming the Array return values produced by multi-return functions and variadic collectors.
- `Module_Objects.md` — Map-based multi-return consumption, key validation, and structured data patterns for function return values.
- `Module_DllCallAndMemory.md` — DllCall function pointers, callback registration, and COM method binding that interact with AHK v2 function references.