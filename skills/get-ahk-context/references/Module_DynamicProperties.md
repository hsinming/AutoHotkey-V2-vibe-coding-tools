# Module_DynamicProperties.md
<!-- DOMAIN: Dynamic Properties — Fat Arrow Functions, Lambdas, Closures, Meta-Functions -->
<!-- SCOPE: Block-body arrow syntax (`=> { }`) and async/concurrent callback scheduling are not covered here — block bodies require AHK v2.1 alpha; scheduling patterns belong in Module_AsyncAndTimers.md. -->
<!-- TRIGGERS: =>, fat arrow, arrow function, lambda, closure, __Get, __Set, __Call, DefineProp, dynamic property, meta-function, functional programming, currying, composition, "short function syntax", "inline callback", "computed property", "property that calculates", "function remembers variables", "factory function" -->
<!-- CONSTRAINTS: Fat arrow functions evaluate exactly one expression — block body `=> { }` is a v2.1 alpha feature; assign a named nested function for any multi-statement logic in v2.0. `__Get(name, params)` and `__Set(name, params, value)` must include the `params` Array parameter — omitting it silently shifts all subsequent arguments, corrupting property reads and writes. Object literals `{}` do NOT invoke `__Set` when setting properties — `__Set` is a class meta-function, not triggered by literal initialization. -->
<!-- CROSS-REF: Module_Functions.md, Module_Classes.md, Module_Errors.md, Module_Arrays.md, Module_Objects.md, Module_AsyncAndTimers.md -->
<!-- VERSION: AHK v2.0+ -->

## V1 → V2 BREAKING CHANGES

| v1 pattern (LLM commonly writes) | v2 correct form | Consequence |
|----------------------------------|-----------------|-------------|
| No `=>` syntax — full `{ return expr }` body written everywhere | `param => expr` for single-expression functions | Verbose but not broken; LLMs over-apply full bodies when fat arrow is available and cleaner |
| `__Get(name)` two-parameter signature | `__Get(name, params)` three-parameter signature | `params` Array received as `name`; actual name argument is silently lost — every property read returns garbage |
| `__Set(name, value)` two-parameter signature | `__Set(name, params, value)` three-parameter signature | `value` receives the `params` Array, not the assigned value — property writes silently store wrong data |
| `ObjBindMethod(obj, "MethodName")` for method binding | `obj.MethodName.Bind(obj)` or pass method reference directly | `ObjBindMethod` is still available in v2 — no breaking change; direct method references or `Func.Bind` are the preferred v2 idioms but the v1 pattern remains valid |
| `Func("FuncName")` to obtain a function reference | Bare name `FuncName` or `FuncName.Bind(args*)` | `Func()` lookup removed in v2; throws a runtime error |
| Assuming closures require an `upVar` or similar keyword | Lexical capture is automatic — no keyword needed | Extra keyword causes a parse error; variables in enclosing scope are captured by reference with no annotation |
| `{}` object literal properties trigger `__Set` | `{}` directly sets own property values — `__Set` is never invoked during literal creation | Validators inside `__Set` are silently bypassed when an object is initialized via `{}` notation |

## API QUICK-REFERENCE

### Fat Arrow Syntax Forms

| Form | Signature | Returns | Throws | Notes |
|------|-----------|---------|--------|-------|
| Single-parameter | `param => expr` | Result of expr | Whatever expr throws | Parentheses optional for exactly one parameter |
| Multi-parameter | `(p1, p2, ...) => expr` | Result of expr | Whatever expr throws | Parentheses required for 0 or 2+ parameters |
| No-parameter | `() => expr` | Result of expr | Whatever expr throws | Empty parens required |
| Named recursive | `FuncName(params) => expr` | Closure object | — | `FuncName` visible inside `expr` for self-reference; named form avoids circular reference that anonymous self-reference creates |
| Fat arrow property | `propName => expr` inside class body | Result of expr (per access) | `PropertyError` on write | Getter-only; re-evaluated on every property access |
| Parameterized property | `propName[key] => expr` inside class body | Result of expr | `PropertyError` on write | `key` is available inside `expr` as a bracket argument |

### Meta-Functions

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `__Get` | `__Get(name, params)` | Property value (user-defined) | Typically `PropertyError` | Invoked when an undefined property is read; `params` is an Array of bracket arguments (empty Array for plain access) |
| `__Set` | `__Set(name, params, value)` | — (return value ignored by AHK) | Typically `ValueError` | Invoked when an undefined property is assigned; `params` Array sits between `name` and `value` — NOT called during `{}` literal construction |
| `__Call` | `__Call(name, params)` | Method result (user-defined) | Typically `MethodError` | Invoked when an undefined method is called; `params` is an Array of all arguments passed |

### DefineProp — Programmatic Property Definition

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `DefineProp()` | `obj.DefineProp(name, {get:fn, set:fn, call:fn})` | `obj` (chainable) | — | Attaches a property descriptor; `get`/`set`/`call` keys are all optional; `Value` key defines a value property |
| `DeleteProp()` | `obj.DeleteProp(name)` | Removed value or blank string | — | Removes an own property from the object; returns the old value |
| `GetOwnPropDesc()` | `obj.GetOwnPropDesc(name)` | Descriptor object | `PropertyError` | Returns `{Get, Set, Call}` for dynamic props or `{Value}` for value props; throws if not an own prop |
| `HasOwnProp()` | `obj.HasOwnProp(name)` | 1 or 0 | — | Tests own-property existence without triggering `__Get` |
| `OwnProps()` | `obj.OwnProps()` | Enumerator | — | Enumerates own properties only; does NOT enumerate properties intercepted by `__Get` |

### Functional Utilities

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `Func.Bind()` | `fn.Bind(args*)` | `BoundFunc` | — | Partial application — returns a new Func with leading arguments pre-bound |
| `Func.Call()` | `fn.Call(args*)` | Whatever fn returns | Whatever fn throws | Explicit call; semantically equivalent to `fn(args*)` |
| `IsSet()` | `IsSet(var)` | 1 or 0 | — | Returns true if variable has been assigned; use for optional parameter detection |
| `IsObject()` | `IsObject(val)` | 1 or 0 | — | Returns true for any AHK object, including closures and Func objects |

### Map — Backing Store Methods Used in Meta-Function Examples

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `Map.Has()` | `m.Has(key)` | 1 or 0 | — | Safe key existence check — use inside `__Get`/`__Set` before accessing the backing Map |
| `Map.Get()` | `m.Get(key, default)` | Value or default | — | Returns default if key absent; never throws — preferred over `m[key]` in meta-function bodies |

### Built-in Functions Used in Examples

| Function | Signature | Returns | Throws | Notes |
|----------|-----------|---------|--------|-------|
| `StrLen()` | `StrLen(str)` | Integer | — | Returns character count of string |
| `FormatTime()` | `FormatTime(timestamp?, format?)` | String | — | Formats date/time; omit timestamp for current system time |
| `Mod()` | `Mod(dividend, divisor)` | Integer or Float | `ZeroDivisionError` | Returns the remainder; use for even/odd and cyclic index checks |
| `IsInteger()` | `IsInteger(val)` | 1 or 0 | — | Returns true if val is an integer-valued number or numeric string |

## AHK V2 CONSTRAINTS

- Fat arrow functions evaluate **exactly one expression** — attempting a block body `param => { stmt1; stmt2 }` is a parse error in v2.0 stable (block bodies are a v2.1 alpha feature); assign a named nested function and reference it by name for any multi-statement logic.
- `__Get(name, params)` and `__Set(name, params, value)` **must include the `params` parameter** — `params` is an Array of bracket arguments (e.g., `obj.prop[key]` passes `[key]`); omitting it shifts all subsequent parameters, causing `name` to receive the Array and `value` to receive wrong data — consequence: every dynamic property read or write silently operates on incorrect arguments.
- `__Get` and `__Set` are **invoked only for properties not defined on the class or its prototype** — own properties and prototype-defined methods bypass the meta-functions entirely — consequence: validators inside `__Set` are silently skipped for any property that was declared in the class body.
- Fat arrow **properties are getter-only** — a bare `propName => expr` in a class body defines no setter; assignment throws a `PropertyError` at runtime — consequence: use `propName { get => expr  set { ... } }` syntax when write access is required.
  - ✗ `version => "2.0"` as a class property, then `obj.version := "3.0"` — `PropertyError: no setter`
  - ✓ `version { get => this._ver  set { this._ver := value } }` — read-write combined block
- Lambdas **stored as object-literal properties and called as methods receive the object as the first argument** — always declare a leading parameter (e.g., `this`) to absorb the implicit argument — consequence: without the parameter, the intended first user-supplied argument is silently consumed as the object, shifting all argument positions.
  - ✗ `{increment: () => ++count}` — object received where no parameter declared; call arguments shift
  - ✓ `{increment: (this) => ++count}` — leading parameter absorbs the implicit object
- Variables captured in closures are captured **by reference, not by value** — the closure sees the current value of the outer variable at call time, not its value at closure-creation time — consequence: closures created inside a loop all share the same loop variable, a classic bug where every closure sees the loop's final value.
- **`{}` object literals do NOT invoke `__Set`** — properties set during literal initialization bypass all meta-function dispatch — consequence: runtime validators registered in `__Set` are silently skipped for properties assigned in the literal itself.
  - ✗ Relying on `__Set` validation when building objects via `{key: val, ...}` notation
  - ✓ Use `DefineProp` descriptors or class instances for properties that require `__Set` validation

Safe-access priority order for dynamic properties:
  1. `obj.HasOwnProp(name)` — check own-property existence without triggering `__Get`
  2. `Map.Has(key)` inside `__Get` — safe lookup in backing store before returning a value or throwing
  3. `obj.DefineProp(name, descriptor)` — when property names and behavior are known at class-definition time
  4. `__Get` / `__Set` — only when the property namespace is genuinely open-ended and unknown at class-definition time

Unset variable handling: use `IsSet(var)` before accessing optional parameters in factory functions, and use `Map.Has(key)` before reading from a backing store inside `__Get` — calling `m[key]` on an absent key throws `UnsetItemError`, which bypasses any user-facing `PropertyError` message in the meta-function.

Resource lifecycle: closures pin all captured variables for their full lifetime — avoid capturing large objects or GUI references in long-lived closures. Named recursive fat arrows (e.g., `Fact(n) => ...`) do not capture their own reference, so they cannot create reference cycles; anonymous self-referencing closures (`fact := (n) => ... fact(n-1)`) do create a cycle that prevents garbage collection.

## AGENT QA CHECKLIST

- [ ] Did I include the `params` Array as the second parameter in all `__Get(name, params)` and `__Set(name, params, value)` signatures?
- [ ] Does every fat arrow property that needs write access use the `{ get => expr  set { ... } }` block form instead of the getter-only `propName => expr` form?
- [ ] For lambdas stored as object-literal properties and called as methods, did I declare a leading `(this)` parameter to absorb the implicit object argument?
- [ ] Did I use the named recursive form `FuncName(p) => expr` instead of anonymous self-reference, to avoid creating a circular reference (closure → variable → closure)?

## RUNTIME ERROR MAPPING

| Exception Class | Trigger Condition | Detection Code | Fix |
|----------------|-------------------|----------------|-----|
| `PropertyError` | Assigning to a getter-only fat arrow property (`propName => expr` with no setter defined) | `e.Message` contains "no setter" | Replace with `propName { get => expr  set { this._backing := value } }` combined block |
| Silent argument shift — no exception thrown | `__Get(name)` or `__Set(name, value)` missing the `params` parameter — `params` Array lands in `name`, actual name is lost | Property reads return unexpected values; writes store wrong data with no error | Add `params` as the second parameter in both signatures: `__Get(name, params)` / `__Set(name, params, value)` |
| `MethodError` | Calling an undefined method on a class that defines `__Get`/`__Set` but not `__Call` | `e.Message` contains the method name | Define `__Call(name, params)` to intercept undefined method calls; throw `MethodError` for truly unknown names |

## TIER 1 — Basic Arrow Function Syntax
> METHODS COVERED: `=>` single-parameter form · `=>` multi-parameter form · `=>` no-parameter form

Fat arrow functions in AHK v2 are single-expression function literals using the `=>` operator. They are values — stored in variables, passed as arguments, returned from functions — not mere syntactic sugar for `return`. Parentheses around parameters are required when there are zero parameters or two or more; a single parameter name may appear bare.

```ahk
; ✓ Multi-parameter form requires parentheses — omitting them is a parse error
multiply := (x, y) => x * y

; ✓ Single-parameter form — parentheses optional but permitted for clarity
square := x => x * x

; ✓ Zero-parameter form — empty parens required; `() =>` is the correct spelling
greet := () => "Hello World"

; ✓ Arrow function is a first-class value: store, pass, call like any Func object
result := multiply(4, 5)   ; 20

; ✗ Block body not supported in v2.0 stable — parse error, not a runtime error
; add := (a, b) => {        ; → ParseError: block body requires v2.1 alpha
;     return a + b
; }

; ✓ Traditional function for multi-statement logic — reference by bare name
Add(a, b) {
    return a + b
}

add := (a, b) => a + b   ; single-expression equivalent

result1 := Add(5, 3)     ; 8 — traditional call
result2 := add(5, 3)     ; 8 — arrow call, identical semantics
```

## TIER 2 — Named Arrow Functions and Recursion
> METHODS COVERED: Named recursive form · Named nested function reference pattern

Anonymous fat arrows cannot call themselves — they hold no name accessible from within their own expression. The named recursive form `FuncName(params) => expr` makes `FuncName` visible inside the expression body, enabling self-reference without capturing the outer variable (which would create a circular reference). For multi-statement logic that cannot collapse to a single expression, define a named nested function in the enclosing scope and reference its name as a value.

```ahk
; ✓ Named recursive form — FuncName is in scope inside the expression; no circular reference
factorial := Fact(n) => n <= 1 ? 1 : n * Fact(n-1)

fibonacci := Fib(n) => n <= 1 ? n : Fib(n-1) + Fib(n-2)

result := factorial(5)   ; 120

; ✗ Anonymous self-reference — closure captures the outer `badFact` variable, creating a
; reference cycle (closure → badFact → closure); the cycle prevents garbage collection
; badFact := (n) => n <= 1 ? 1 : n * badFact(n-1)   ; → reference cycle / memory leak

; ✓ Multi-statement logic: named nested function closes over enclosing scope, referenced by name
ProcessData(input) {
    validated := ValidateInput(input)
    if (!validated)
        throw ValueError("Invalid input")

    result := TransformData(validated)
    LogOperation("Process", input, result)
    return result
}
processData := ProcessData   ; processData holds the function reference; no () here
```

## TIER 3 — Closures and Variable Capture
> METHODS COVERED: Lexical capture · `StrLen()` · Closure factory pattern · Named nested function as closure

Arrow functions and named nested functions in AHK v2 automatically capture variables from their lexical enclosing scope. Captured variables are held by reference — the closure sees the current value of the outer variable at call time, not a snapshot from creation time. When a lambda is stored in an object-literal property and invoked as a method, AHK v2 passes the object as the first positional argument; the lambda must declare a leading parameter to absorb it.

```ahk
; ✓ Four lambdas all capture the same `count` variable by reference
CreateCounter() {
    count := 0
    return {
        increment: (this) => ++count,   ; (this) absorbs the implicit object argument
        decrement: (this) => --count,
        getValue:  (this) => count,
        reset:     (this) => count := 0
    }
}

counter := CreateCounter()
counter.increment()
counter.increment()
value := counter.getValue()   ; 2

; ✗ Missing leading parameter on object-property lambda — first user arg silently consumed
; badIncrement: () => ++count   ; → object received where no parameter declared; call arguments shift

; ✓ Factory pattern — each CreateValidator call captures its own minLength/maxLength copy
CreateValidator(minLength, maxLength) {
    Validate(text) {
        len := StrLen(text)
        return len >= minLength && len <= maxLength
    }
    return Validate
}

emailValidator    := CreateValidator(5, 100)
passwordValidator := CreateValidator(8, 50)

isValidEmail    := emailValidator("user@domain.com")   ; true
isValidPassword := passwordValidator("abc")            ; false — StrLen("abc") = 3 < 8
```

## TIER 4 — Fat Arrow Properties
> METHODS COVERED: Getter fat arrow property · Parameterized property `prop[key]` · `get => / set { }` combined block · `FormatTime()`

Fat arrow properties inside a class body define computed getters: the expression is evaluated fresh on every property access and no backing field is allocated. For read-write properties, use the explicit `{ get => expr  set { ... } }` block syntax. Assignment to a pure fat arrow property — one defined with only `propName => expr` — throws a `PropertyError` at runtime.

```ahk
; ✓ Each fat arrow property is re-evaluated on every access — no cached value
class DataProcessor {
    version => "2.0.1"

    ; ✓ FormatTime called on every read — always returns current timestamp
    timestamp => FormatTime(, "yyyy-MM-dd HH:mm:ss")

    ; ✓ Parameterized property — bracket argument available as `x` inside expr
    squareOf[x] => x * x

    _items := []
    ; ✓ Delegates to Array.Length — computed from backing field on each access
    count => this._items.Length
}

processor := DataProcessor()
MsgBox processor.version        ; "2.0.1"
MsgBox processor.timestamp      ; current date-time string
MsgBox processor.squareOf[5]    ; 25

; ✗ Assigning to a fat arrow property — no setter defined, throws PropertyError
; processor.version := "3.0"    ; → PropertyError: no setter

; ✓ Combined get/set: fat arrow getter with validated traditional setter
class Counter {
    _value := 0

    value {
        get => this._value
        set {
            if (value < 0)
                throw ValueError("Counter cannot be negative")
            this._value := value
        }
    }

    isZero => this._value = 0
}
```

## TIER 5 — Dynamic Properties and Meta-Functions
> METHODS COVERED: `__Get` · `__Set` · `Map()` · `Map.Has()` · `HasOwnProp()` · `PropertyError()` · `ValueError()` · `IsInteger()`

`__Get` and `__Set` intercept reads and writes to properties that are not defined on the class or its prototype — enabling fully open-ended dynamic property bags. Both meta-functions in v2 carry a `params` parameter (an Array of bracket arguments) between the property name and, for `__Set`, the assigned value. Providing only two parameters silently misaligns everything: `name` receives the `params` Array and subsequent parameters receive wrong values.

```ahk
; ✓ Both meta-functions include the required `params` parameter
class DynamicObject {
    _props := Map()

    __Get(name, params) {
        if (this._props.Has(name))
            return this._props[name]
        throw PropertyError("Property '" name "' not found")
    }

    __Set(name, params, value) {
        this._props[name] := value
    }

    HasProperty(name) => this._props.Has(name)
    GetPropertyNames() => [this._props*]
}

obj := DynamicObject()
obj.color := "blue"
obj.size  := "large"
MsgBox obj.color   ; "blue"

; ✗ v1-style two-parameter __Get — `params` Array received as `name`, actual name lost
; __Get(name) { return this._props[name] }   ; → every read operates on the wrong key

; ✓ Advanced: meta-functions with per-key validation
class ConfigManager {
    _config     := Map()
    _validators := Map()

    __Get(key, params) {
        if (!this._config.Has(key))
            throw PropertyError("Configuration key '" key "' not found")
        return this._config[key]
    }

    __Set(key, params, value) {
        if (this._validators.Has(key)) {
            validator := this._validators[key]
            if (!validator(value))
                throw ValueError("Invalid value for '" key "'")
        }
        this._config[key] := value
    }

    SetValidator(key, validatorFunc) {
        this._validators[key] := validatorFunc
    }
}

config := ConfigManager()
config.SetValidator("port", (v) => IsInteger(v) && v > 0 && v <= 65535)
config.port := 8080   ; passes validation — IsInteger(8080) true, 8080 in range
```

### Performance Notes

**`__Get` / `__Set` dispatch overhead.** Every access to an undefined property passes through meta-function dispatch. For hot paths (tight loops over many property reads), prefer a `Map` accessed directly (`m[key]`) over a `DynamicObject` wrapper: direct Map access is O(1) with no function-call overhead; `__Get` adds at minimum one extra call frame per read. If all property names are known at class-definition time, `obj.DefineProp(name, descriptor)` with static descriptors costs nothing at access time.

**Closure allocation.** Each call to a factory function allocates a new closure environment that pins references to the captured variables. Avoid creating closures inside tight loops — create them once outside and reuse. AHK v2 has no JIT or closure inlining.

**Fat arrow property re-evaluation.** `timestamp => FormatTime(...)` calls `FormatTime` on every property read. Cache the result in a local variable when the same property is read multiple times in a single block.

**Composition depth.** `compose(compose(f, g), h)` allocates a new closure at every nesting level. For pipelines of more than three functions, an iterative approach (store functions in an Array, iterate with `for`) avoids deep closure chains and is more memory-efficient.

**Named recursive fat arrows.** `Fact(n) => n <= 1 ? 1 : n * Fact(n-1)` incurs standard function-call overhead per recursive call. AHK v2 has no tail-call optimization — prefer iterative implementations or memoization for large inputs.

## TIER 6 — Functional Programming Patterns
> METHODS COVERED: `compose` pattern · `partial` / `Func.Bind()` pattern · `FunctionalArray.Map` · `FunctionalArray.Filter` · `FunctionalArray.Reduce` · `Mod()` · `IsSet()`

Fat arrows as first-class values enable composition, currying, and higher-order collection processing without naming every intermediate function. Composition chains transforms right-to-left; partial application specializes general functions; a custom `Array` subclass exposes `Map`/`Filter`/`Reduce` as a declarative pipeline API.

```ahk
; ✓ compose returns a new arrow that applies g first, then f — right-to-left application
compose := (f, g) => (x) => f(g(x))

addOne := x => x + 1
double := x => x * 2
square := x => x * x

addThenDouble    := compose(double, addOne)    ; double(addOne(x))
doubleThenSquare := compose(square, double)    ; square(double(x))

result1 := addThenDouble(5)     ; 12  — addOne(5)=6, double(6)=12
result2 := doubleThenSquare(3)  ; 36  — double(3)=6, square(6)=36

; ✗ Argument order reversal — compose(addOne, double) applies double first, not add
; wrongOrder := compose(addOne, double)   ; → addOne(double(x)), semantically reversed

; ✓ Currying — each call returns an arrow awaiting the next argument
curriedAdd := (a) => (b) => (c) => a + b + c

; ✓ Partial application via variadic spread — pre-bind leading arguments
partial := (fn, args*) => (remaining*) => fn(args*, remaining*)

multiply := (a, b, c) => a * b * c

multiplyBy2    := partial(multiply, 2)
multiplyBy2And3 := partial(multiply, 2, 3)

result1 := curriedAdd(1)(2)(3)     ; 6
result2 := multiplyBy2(5, 3)       ; 30
result3 := multiplyBy2And3(4)      ; 24

; ✓ FunctionalArray subclass exposes Map/Filter/Reduce as chainable methods
class FunctionalArray extends Array {
    Map(fn) {
        result := FunctionalArray()
        for item in this
            result.Push(fn(item))
        return result
    }

    Filter(predicate) {
        result := FunctionalArray()
        for item in this
            if predicate(item)
                result.Push(item)
        return result
    }

    Reduce(fn, initial := unset) {
        result := IsSet(initial) ? initial : this[1]
        startIndex := IsSet(initial) ? 1 : 2

        Loop this.Length - startIndex + 1 {
            result := fn(result, this[startIndex + A_Index - 1])
        }
        return result
    }
}

numbers := FunctionalArray(1, 2, 3, 4, 5)
doubled := numbers.Map(x => x * 2)              ; [2, 4, 6, 8, 10]
evens   := numbers.Filter(x => Mod(x, 2) = 0)  ; [2, 4]
sum     := numbers.Reduce((a, b) => a + b)      ; 15
```

## DROP-IN RECIPES

```ahk
; MemoizeFunc — wraps any pure function with Map-based result cache; cache lives in the closure
; ✓ One cache Map per MemoizeFunc call — closures from different calls never share state
; ✓ Key is built from all args joined with NUL separator — handles multi-parameter functions
MemoizeFunc(fn) {
    if !(fn is Func)
        throw TypeError("MemoizeFunc: fn must be a Func or Closure object", -1)
    cache := Map()
    Cached(args*) {
        key := ""
        for arg in args
            key .= String(arg) "`0"
        if cache.Has(key)
            return cache[key]
        result := fn(args*)
        cache[key] := result
        return result
    }
    return Cached
}
; Call site: cachedLookup := MemoizeFunc(ExpensiveQuery)  ;  cachedLookup("id-42") calls ExpensiveQuery once; subsequent calls hit cache

; TypedStore — dynamic property bag with per-key validator functions via __Get/__Set
; ✓ __Get and __Set use correct v2 three-parameter signatures including params
; ✓ Validators are Func objects stored in a Map — one validator per key, optional
class TypedStore {
    _props      := Map()
    _validators := Map()

    __New(validators := unset) {
        if IsSet(validators) {
            if !(validators is Map)
                throw TypeError("TypedStore: validators must be a Map of key → Func", -1)
            this._validators := validators
        }
    }

    __Get(name, params) {
        if !this._props.Has(name)
            throw PropertyError("TypedStore: key '" name "' not found", -1)
        return this._props[name]
    }

    __Set(name, params, value) {
        if this._validators.Has(name) && !this._validators[name](value)
            throw ValueError("TypedStore: value for '" name "' failed validation", -1)
        this._props[name] := value
    }

    ; ✓ Has() uses Map.Has() — does not trigger __Get, safe for existence check
    Has(name) => this._props.Has(name)

    ; ✓ Keys() spreads Map enumerator — returns Array of key strings only
    Keys() => [this._props*]
}
; Call site:
; store := TypedStore(Map("port", v => IsInteger(v) && v > 0 && v <= 65535))
; store.port := 8080   ; passes validator
; store.port := -1     ; ValueError: "port" failed validation
```

## ANTI-PATTERNS

| Pattern | Wrong | Correct | LLM Common Cause |
|---------|-------|---------|------------------|
| Block-body fat arrow | `fn := x => { a := x*2 ; return a }` | Named nested function assigned to variable | JavaScript and Python arrow/lambda variants allow block bodies; AHK v2.0 does not — v2.1 alpha training data leakage |
| `__Get` missing `params` | `__Get(name) { return this._store[name] }` | `__Get(name, params) { return this._store[name] }` | AHK v1 `__Get` took only `name`; v2 inserts `params` between name and return — v1 training data regression |
| `__Set` missing `params` | `__Set(name, value) { this._store[name] := value }` | `__Set(name, params, value) { this._store[name] := value }` | Same v1 regression — `value` silently receives the `params` Array instead of the assigned value |
| Lambda property without leading param | `increment: () => ++count` stored in `{}` | `increment: (this) => ++count` | LLMs model AHK like JavaScript, where arrow functions do not receive an implicit `this`; AHK v2 passes the object as first positional arg |
| Anonymous self-reference | `fact := (n) => n <= 1 ? 1 : n * fact(n-1)` | `fact := Fact(n) => n <= 1 ? 1 : n * Fact(n-1)` | The closure captures the outer `fact` variable, creating a circular reference (closure → fact → closure) that prevents garbage collection — the named form avoids capturing its own reference <!-- CONVERTED: original LLM Common Cause incorrectly stated "fact is unset at the point the expression is evaluated"; the actual consequence is a reference cycle, not an UnsetError, because closures capture by reference and fact is set before the first call --> |
| Assigning to fat arrow property | `obj.version := "3.0"` where `version => expr` | `version { get => expr  set { ... } }` combined block | Python `@property` is read-write by default; LLMs assume AHK fat arrow properties behave identically |

## SEE ALSO

> This module does NOT cover: traditional function syntax, default parameters, variadic `args*` params, and the full `Func` object API → see Module_Functions.md
> This module does NOT cover: full class property declaration, inheritance, prototype chain, and `__New` constructor → see Module_Classes.md
> This module does NOT cover: `try/catch` wrapping for `PropertyError` and `ValueError` raised from `__Get`/`__Set` → see Module_Errors.md
> This module does NOT cover: built-in Array methods (`Push`, `Pop`, `RemoveAt`, `Sort`) and standard iteration patterns → see Module_Arrays.md
> This module does NOT cover: `SetTimer`/`OnMessage` async callback scheduling and timer closure patterns → see Module_AsyncAndTimers.md

- `Module_Functions.md` — traditional function definitions, default parameters, variadic `args*`, and the complete `Func` object API including `Bind()`, `Call()`, `MinParams`, and `MaxParams`.
- `Module_Classes.md` — class property declaration with `{get/set}` blocks, inheritance, `__New` constructor, and prototype-chain OOP patterns that pair with the fat arrow property forms in TIER 4.
- `Module_Errors.md` — `try/catch` patterns for `PropertyError` and `ValueError` thrown from `__Get`/`__Set` validators; custom exception class definitions and error propagation.
- `Module_Arrays.md` — built-in Array methods and iteration; the `FunctionalArray` subclass in TIER 6 extends these primitives rather than replacing them.
- `Module_Objects.md` — plain object literals, `OwnProps()` enumeration, and `obj.DefineProp(name, descriptor)` as a static alternative to `__Get`/`__Set` for property interception when property names are known ahead of time.