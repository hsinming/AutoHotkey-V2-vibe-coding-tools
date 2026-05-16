# Module_Objects.md
<!-- DOMAIN: Objects and OOP -->
<!-- SCOPE: Advanced meta-functions (__Get/__Set/__Call/__Delete/__Enum), mixin patterns, and deep class hierarchies are not covered — see Module_Classes.md. -->
<!-- TRIGGERS: DefineProp, HasProp, HasOwnProp, HasMethod, HasBase, GetMethod, ObjBindMethod, Type(), "create object", "property descriptor", "property validation", "method binding", "computed property", "callback context", "check object type", "read-only property", "prototype extension", "class inheritance", "object clone", class, extends, prototype -->
<!-- CONSTRAINTS: Instantiate classes with ClassName() — never `new ClassName()`; the keyword was removed in v2 and throws NameError. Arrow syntax (=>) is valid only for single-expression DefineProp descriptor bodies; multi-line descriptor bodies require named function references or AHK v2 throws a parse error. ObjBindMethod() or .Bind(this) is required whenever a method is passed as a callback — a raw this.Method reference loses its this context at invocation, causing UnsetError. -->
<!-- CROSS-REF: Module_Classes.md, Module_DataStructures.md, Module_GUI.md, Module_Errors.md, Module_Arrays.md -->
<!-- VERSION: AHK v2.0+ -->

## V1 → V2 BREAKING CHANGES

| v1 pattern (LLM commonly writes) | v2 correct form | Consequence |
|----------------------------------|-----------------|-------------|
| `new ClassName()` | `ClassName()` | NameError at runtime — `new` keyword removed entirely in AHK v2 |
| `{key: val}` as a data container | `Map("key", val)` | Object literals lack `.Has()`, `.Get()`, `.Delete()` — missing keys throw UnsetError with no safe fallback |
| `IsObject(x)` for type checking | `x is Object` or `Type(x)` | `IsObject()` returns non-zero for any object; `is` and `Type()` are the preferred v2 alternatives for finer-grained type discrimination |
| `SetTimer, % this.Method, 1000` | `SetTimer(this.Method.Bind(this), 1000)` | v1 percent-expression syntax removed; unbound method loses `this` context — UnsetError in the callback body |
| `obj.__Set := func` (v1 meta-function assignment) | `obj.DefineProp("prop", {set: func})` | Meta-function assignment removed — `DefineProp()` is the only v2 hook for intercepting property write behavior |
| `(this, value) => { ... multiline ... }` in DefineProp | Named function reference passed to descriptor key | Parse error at the opening brace — arrow syntax cannot open a multi-line block body in AHK v2 |
| `Object().DefineProp(SomeProto, "x", d)` | `ObjDefineProp := Object.Prototype.DefineProp` then `ObjDefineProp(SomeProto, "x", d)` | `DefineProp()` is an instance method; calling it on an anonymous `Object()` with a prototype as the first argument has no effect on that prototype |
| `set => { this._w := value }` inside a class body | `set { this._w := value }` inside a class body | `set =>` block is a parse error in class body context — class body `set` uses brace block, not arrow |

## API QUICK-REFERENCE

### Any (root — every AHK v2 value inherits these)

| Method / Property | Signature | Returns | Throws | Notes |
|-------------------|-----------|---------|--------|-------|
| `.GetMethod()` | `.GetMethod(name, paramCount?)` | Func object | MethodError if not found | Retrieves the implementation function; use HasMethod() first if existence is uncertain |
| `.HasBase()` | `.HasBase(baseObj)` | 1 or 0 | — | Traverses full prototype chain; never throws; equivalent to `HasBase(val, baseObj)` |
| `.HasMethod()` | `.HasMethod(name, paramCount?)` | 1 or 0 | — | Optionally validates arity via MinParams/MaxParams; never throws |
| `.HasProp()` | `.HasProp(name)` | 1 or 0 | — | Checks instance and full prototype chain; never throws |
| `.Base` | `.Base` | Prototype object or `""` | — | Readable on Any; writable on Object instances only; changing base reassigns inherited API |

### Object (instance methods — available on plain objects and class instances)

| Method / Property | Signature | Returns | Throws | Notes |
|-------------------|-----------|---------|--------|-------|
| `.Clone()` | `.Clone()` | Shallow copy of the object | TypeError if derived from unsupported built-in type | Copies references, not sub-objects; dynamic properties are copied without invocation |
| `.DefineProp()` | `.DefineProp(name, descriptor)` | `this` (chainable) | — | Descriptor keys: `get`, `set`, `call`, `value`; not possible to mix `value` with accessor functions |
| `.DeleteProp()` | `.DeleteProp(name)` | Last value of removed property (blank if none) | — | Removes own property only; no error if property absent |
| `.GetOwnPropDesc()` | `.GetOwnPropDesc(name)` | Descriptor object | PropertyError if property not an own property | Returns `{value:...}` for value props, `{get:..., set:...}` for dynamic props |
| `.HasOwnProp()` | `.HasOwnProp(name)` | 1 or 0 | — | Own properties only — does NOT traverse the prototype chain; contrast with `.HasProp()` |
| `.OwnProps()` | `.OwnProps()` | Enumerator | — | Skips call-only properties in two-var mode; use in `for name, val in obj.OwnProps()` |

### Func / BoundFunc

| Method / Property | Signature | Returns | Throws | Notes |
|-------------------|-----------|---------|--------|-------|
| `.Bind()` | `.Bind(args*)` | BoundFunc | — | Pre-fills leading arguments; primary tool for capturing `this` in callbacks |
| `ObjBindMethod()` | `ObjBindMethod(obj, methodName, args*)` | BoundFunc | — | Binds by method name string — safer than `.Bind()` for SetTimer and GUI events when the method may be overridden |

### Type Introspection

| Function / Operator | Signature | Returns | Throws | Notes |
|---------------------|-----------|---------|--------|-------|
| `Type()` | `Type(value)` | Type name string | — | Returns `"Integer"`, `"Float"`, `"String"`, `"Object"`, or the exact class name |
| `is` operator | `expr is TypeOrClass` | 1 or 0 | — | Traverses full prototype chain; works with Integer and Float primitive types and user-defined classes; `x is String` always returns false for string primitives — use `Type(x) = "String"` instead |

### Supporting Functions (referenced in examples — defined in other modules)

| Function | Primary Module | Role in examples |
|----------|---------------|-----------------|
| `SetTimer()` | Module_AsyncAndTimers.md | Registers BoundFunc callbacks |
| `Gui.OnEvent()` | Module_GUI.md | Wires GUI control events to bound methods |
| `FormatTime()` | Module_TextProcessing.md | Used in computed property examples |
| `RegExMatch()` | Module_TextProcessing.md | Used in validation helper examples |
| `StrSplit()` | Module_TextProcessing.md | Used in prototype extension example |

## AHK V2 CONSTRAINTS

- `ClassName()` — never `new ClassName()` — the `new` keyword was removed in AHK v2 and throws NameError; every class instantiation in v2 is a plain function call.
  - ✗ `john := new Person("John", 30)` — NameError, `new` removed in AHK v2
  - ✓ `john := Person("John", 30)` — correct v2 instantiation

- Arrow syntax (`=>`) is valid only for single-expression descriptor bodies — any descriptor body requiring more than one statement must use a named function reference; mixing arrow with a brace block (`=> { ... }`) is a parse error.
  - ✗ `obj.DefineProp("broken", {set: (this, v) => { if v < 0 ... }})` — parse error, arrow + block
  - ✓ Named function reference passed as the `set` descriptor value for multi-line bodies

- Class body property descriptors use `get => expr` / `set { ... }` syntax — do NOT write `set => { ... }` inside a class body; the arrow + brace block form is rejected by the AHK v2 parser.
  - ✗ `Width { set => { this._w := value } }` — parse error inside class body
  - ✓ `Width { set { this._w := value } }` — correct class body setter syntax

- Use `Map()` for key-value data storage, not object literals — object literals lack `.Has()`, `.Get()`, and `.Delete()`, making safe missing-key access impossible.
  - ✗ `data := {theme: "dark"}` then `data["nonexistent"]` — UnsetError with no safe fallback
  - ✓ `data := Map("theme", "dark")` then `data.Get("nonexistent", "")` — always safe

- `ObjBindMethod()` or `.Bind(this)` is required whenever passing a method as a callback — a raw `this.Method` reference passed to `SetTimer` or `OnEvent` loses `this` context at invocation time, causing UnsetError inside the method body.
  - ✗ `SetTimer(this.Tick, 1000)` — `this` lost at callback time → UnsetError
  - ✓ `SetTimer(this.Tick.Bind(this), 1000)` — BoundFunc preserves object context

- `.HasProp(name)` checks the entire prototype chain; `.HasOwnProp(name)` checks only the object's own properties — use the appropriate form; confusing them causes incorrect branch logic for inherited vs. own property detection.
  - ✗ `obj.HasProp("length")` used to check whether the instance itself has a `length` property — returns 1 even when inherited
  - ✓ `obj.HasOwnProp("length")` — returns 1 only for properties defined directly on that instance

- Prototype extension via `DefineProp` on a shared prototype is global — it applies to every instance of that type across the entire script; use judiciously to avoid hidden coupling between unrelated call sites.

- `{}` object literals in v2 bypass `__Set` and property setters — they directly write own property values; do not rely on setter interception from object literal initialization.

Safe-access priority order for object property reads:

1. `.HasProp(name)` before access — when absent vs. present requires different branch logic and the value is inherited or own
2. `.HasOwnProp(name)` before access — when you specifically need to distinguish an own property from an inherited one
3. `Map.Get(key, default)` — for dynamic key-value data; always prefer Map over plain Object for data bags
4. `.GetOwnPropDesc(name)` — only when you need the descriptor itself (getter/setter/value), not the property value
5. `try/catch` on property access — only when the thrown PropertyError or UnsetError carries diagnostic information beyond "key absent"

Unset variable handling: always check `.HasProp()` or `.HasOwnProp()` before accessing a property whose presence is conditional; reading an absent property throws UnsetError with no safe default path.

Resource lifecycle: objects holding COM references, GUI handles, or file handles must release them in `__Delete` or explicitly in a `finally` block — AHK v2 reference-counting GC collects objects when the last reference drops, but `__Delete` timing is not guaranteed when circular references or closures keep objects alive.

## AGENT QA CHECKLIST

- [ ] Did I instantiate every class with `ClassName()` and never `new ClassName()`?
- [ ] Did I use named function references (not arrow + brace) for every multi-line DefineProp descriptor body?
- [ ] Did I call `.Bind(this)` or `ObjBindMethod()` for every method passed to SetTimer, OnEvent, or any callback that fires outside the object's call stack?
- [ ] Did I use `Map()` rather than `{}` object literals for any property whose key set is dynamic or not known at compile time?
- [ ] Did I use `.HasOwnProp()` vs `.HasProp()` correctly — own-only check vs full-chain check?

## RUNTIME ERROR MAPPING

| Exception Class | Trigger Condition | Detection Code | Fix |
|----------------|-------------------|----------------|-----|
| `NameError` | `new ClassName()` — `new` keyword used at instantiation | `e.Message` contains "NameError" at the `new` token | Replace `new ClassName(args)` with `ClassName(args)` throughout |
| `UnsetError` / `PropertyError` | Reading `obj.propName` when the property is absent on the instance and its prototype chain | `e.Message` contains the property name; triggered on any unguarded access | Guard with `obj.HasProp("propName")` before access, or check `obj.HasOwnProp()` for own-only verification |
| `MethodError` | Calling `.GetMethod(name)` when the method does not exist, or invoking an unbound callback where `this` is unset | `e.Message` contains the method name; or `UnsetError` fires inside the callback body | Use `.HasMethod(name)` before `.GetMethod()`; use `.Bind(this)` or `ObjBindMethod()` before registering any callback |

## TIER 1 — Object Fundamentals, Creation, and the Any Root
> METHODS COVERED: Type · HasProp · HasMethod · Map (constructor) · DefineProp (class body __New)

Every value in AHK v2 — strings, integers, functions, class instances — is an object inheriting from the `Any` root class. This tier establishes the object hierarchy, the introspection methods available on every value, and the two primary creation forms: class instantiation (no `new`) and plain object literals for ad-hoc structures. `Type()` and `HasProp()`/`HasMethod()` are available on every value without exception.

```ahk
; ✓ Type() reveals the runtime type of any value — everything is a typed object in AHK v2
greeting := "Hello"
MsgBox(Type(greeting))   ; "String"

number := 42
MsgBox(Type(number))     ; "Integer" — primitives are typed objects, not raw values

; ✓ HasProp/HasMethod from Any are available on every value — never throw, return 0 or 1
obj := {}
MsgBox(obj.HasProp("SomeProperty"))   ; 0
MsgBox(obj.HasMethod("ToString"))     ; 0

; ✓ Class instantiation is a plain function call — 'new' keyword removed in AHK v2
class Person {
    __New(name, age) {
        this.name := name
        this.age  := age
    }

    Greet() {
        MsgBox("Hello, I'm " . this.name)
    }
}

john := Person("John", 30)   ; Correct v2 form — no 'new'
john.Greet()

; ✓ Object literal for ad-hoc structured data with known, fixed properties
person := {
    name: "John",
    age:  30
}

; ✓ Map() for dynamic key-value storage — provides .Has(), .Get(), .Delete()
settings := Map("theme", "dark", "volume", 80)

; ✗ 'new' keyword removed — NameError at runtime
; john := new Person("John", 30)   ; → NameError

; ✗ Object literal for data with unknown or dynamic keys — no safe missing-key access
; data := {theme: "dark"}
; MsgBox(data["nonexistent"])      ; → UnsetError
```

## TIER 2 — Property Descriptors: get, set, call, and DefineProp
> METHODS COVERED: DefineProp · GetOwnPropDesc · OwnProps · DeleteProp · HasOwnProp

Descriptors control exactly how a property behaves when read (`get`), assigned (`set`), or invoked (`call`). `DefineProp()` is the runtime API for any object instance; class body `get`/`set` blocks are the compile-time equivalent with slightly different but compatible syntax. Arrow syntax is valid only for single-expression bodies — multi-line bodies require named function references. `HasOwnProp()` distinguishes own properties from inherited ones; `GetOwnPropDesc()` inspects the descriptor itself.

```ahk
; ✓ Single-line arrow descriptor — valid because the body is one expression
obj := {}
obj.DefineProp("currentTime", {
    get: (*) => FormatTime(, "yyyy-MM-dd HH:mm:ss")
})
MsgBox(obj.currentTime)   ; Always computes fresh on each read

; ✓ Multi-line descriptor body requires a named function reference
;   Arrow + brace block is a parse error — use this pattern instead
SetAge(this, value) {
    if (value < 0 || value > 150)
        throw ValueError("Invalid age: " . value)
    this._age := value
}
obj.DefineProp("age", {
    set: SetAge,
    get: (this) => this._age ?? 0
})
obj.age := 25   ; Validated; out-of-range throws ValueError

; ✓ Full get+set in DefineProp with separate named setter
SetComplexValue(this, value) {
    if (value < 0)
        throw ValueError("Value must be positive")
    this._value := value
}
obj.DefineProp("complex", {
    get: (this) => this._value ?? 0,
    set: SetComplexValue
})

; ✗ Arrow syntax with multi-line block — parse error in AHK v2
; obj.DefineProp("broken", {
;     set: (this, value) => {    ; → Parse error: arrow cannot open a brace block
;         if (value < 0)
;             throw ValueError("Invalid")
;         this._value := value
;     }
; })

; ✓ Call descriptor — transforms a property into a callable method
CalculateOp(this, operation, a, b) {
    switch operation {
        case "add":      return a + b
        case "multiply": return a * b
        default:         throw ValueError("Unknown operation: " . operation)
    }
}
obj.DefineProp("calculate", {call: CalculateOp})
result := obj.calculate("add", 5, 3)   ; Returns 8

; ✓ HasOwnProp checks only the instance — does not traverse the prototype chain
obj.DefineProp("ownProp", {value: 42})
MsgBox(obj.HasOwnProp("ownProp"))   ; 1 — defined directly on obj
MsgBox(obj.HasOwnProp("calculate")) ; 1 — also own; contrast with inherited methods

; ✓ GetOwnPropDesc inspects descriptor type — distinguishes dynamic from value properties
desc := obj.GetOwnPropDesc("age")
MsgBox(desc.HasProp("Value") ? "value property" : "dynamic property")   ; dynamic

; ✓ Class body get/set syntax — different form, semantically equivalent to DefineProp
;   Use 'set { }' NOT 'set => { }' inside a class body
class Rectangle {
    __New(width, height) {
        this._width  := width
        this._height := height
    }

    ; Validated set + simple get — private backing property _width
    Width {
        get => this._width
        set {
            if (value <= 0)
                throw ValueError("Width must be positive")
            this._width := value
        }
    }

    ; Computed read-only property — get-only descriptor
    Area {
        get => this._width * this._height
    }
}

rect := Rectangle(10, 5)
MsgBox("Area: " . rect.Area)   ; 50

; ✓ Class methods are automatically call descriptors — no manual DefineProp needed
class Calculator {
    Add(a, b)      => a + b
    Multiply(a, b) => a * b
}
calc   := Calculator()
result := calc.Add(10, 5)   ; 15

; ✗ 'set => { }' inside class body — parse error, not valid class body syntax
; Width {
;     set => { this._width := value }   ; → Parse error inside class body
; }
```

## TIER 3 — Class Inheritance and Prototype Extension
> METHODS COVERED: extends · super.__New · is operator · HasBase · Clone · String.Prototype.DefineProp

`extends` builds a prototype chain; `super` reaches the parent constructor and methods. The `is` operator traverses the full chain for type checking. `Clone()` produces a shallow copy with the same base. Prototype extension via `DefineProp` on a shared prototype adds a method globally to all instances of that type — the pattern requires extracting `DefineProp` from `Object.Prototype` because primitive prototype objects do not inherit from `Object.Prototype` directly.

```ahk
; ✓ extends + super.__New delegates to the parent constructor without duplication
class Animal {
    __New(name, species) {
        this.name    := name
        this.species := species
    }

    Speak() {
        MsgBox(this.name . " makes a sound")
    }
}

class Dog extends Animal {
    __New(name, breed) {
        super.__New(name, "Dog")   ; Delegate to Animal.__New — sets name and species
        this.breed := breed
    }

    ; Override parent method — super.Speak() still accessible if needed
    Speak() {
        MsgBox(this.name . " barks: Woof!")
    }
}

buddy := Dog("Buddy", "Golden Retriever")
buddy.Speak()   ; "Buddy barks: Woof!"

; ✓ 'is' operator traverses the full prototype chain
if (buddy is Animal)
    MsgBox("Buddy is an Animal")   ; Shows — Dog → Animal is a valid chain

; ✓ HasBase() for programmatic prototype chain inspection
MsgBox(buddy.HasBase(Animal.Prototype))   ; 1

; ✓ Clone() produces a shallow copy with the same base — references copied, not sub-objects
original := Dog("Rex", "Labrador")
cloned   := original.Clone()
MsgBox(cloned.name)                   ; "Rex" — value copied
MsgBox(cloned.HasBase(Dog.Prototype)) ; 1 — same prototype chain as original

; ✓ Prototype extension — String.Prototype is not Object-derived; extract DefineProp first
ReverseStr(this) {
    chars    := StrSplit(this)
    reversed := ""
    Loop chars.Length
        reversed .= chars[chars.Length - A_Index + 1]
    return reversed
}
ObjDefineProp := Object.Prototype.DefineProp
ObjDefineProp(String.Prototype, "Reverse", {call: ReverseStr})

text := "Hello World"
MsgBox("Reversed: " . text.Reverse())   ; "dlroW olleH"

; ✗ Wrong prototype extension syntax — String.Prototype has no DefineProp instance method
; Object().DefineProp(String.Prototype, "Reverse", {call: ReverseStr})   ; → No effect on String.Prototype

; Warning: Prototype extension is global — affects every call site using that type in the script
```

## TIER 4 — BoundFunc and Callback Context Management
> METHODS COVERED: .Bind() · ObjBindMethod · SetTimer · Gui.OnEvent

Raw method references (`this.Method`) passed to `SetTimer` or GUI `OnEvent` lose their `this` binding at invocation time because the callback fires outside the object context. `.Bind(this)` or `ObjBindMethod()` must capture the object before the callback is registered — not at invocation time. Store BoundFuncs as instance properties in `__New`; never call `.Bind()` inside a timer tick or event handler that fires repeatedly.

```ahk
; ✓ .Bind(this) creates a BoundFunc that carries object context into the callback
class Timer {
    __New(name) {
        this.name  := name
        this.count := 0
    }

    Start() {
        ; ✓ Bind captures 'this' — store result once in __New or Start, not repeatedly
        this.boundTick := this.Tick.Bind(this)
        SetTimer(this.boundTick, 1000)
    }

    Tick() {
        this.count++
        MsgBox(this.name . " tick #" . this.count)
    }
}

timer := Timer("MyTimer")
timer.Start()

; ✗ Unbound method — 'this' inside Tick() is undefined at callback time
; SetTimer(this.Tick, 1000)   ; → UnsetError when Tick() accesses this.name

; ✓ GUI event binding — bind every OnEvent handler so class methods access instance state
class SimpleGUI {
    __New() {
        this.CreateGUI()
    }

    CreateGUI() {
        this.gui       := Gui("+Resize", "My Application")
        this.nameEdit  := this.gui.AddEdit("w200")
        this.submitBtn := this.gui.AddButton("w100", "Submit")

        ; ✓ Bind before registering — handler fires with correct instance context
        this.submitBtn.OnEvent("Click", this.OnSubmit.Bind(this))
        this.gui.OnEvent("Close",       this.OnClose.Bind(this))

        this.gui.Show("w300 h150")
    }

    OnSubmit(*) {
        name := this.nameEdit.Value
        if (name.Length > 0) {
            MsgBox("Hello, " . name . "!")
            this.nameEdit.Value := ""
        }
    }

    OnClose(*) {
        ExitApp()
    }
}

app := SimpleGUI()
```

## TIER 5 — Dynamic Properties and Object Composition
> METHODS COVERED: DefineProp · Map.Set · Map.Get · Map.Has · Map (constructor)

`DefineProp` at runtime enables plugin-style architectures and configuration-driven objects where property names are not known at compile time. Composition — assembling objects from focused collaborator instances — scales more flexibly than deep inheritance for complex domains and produces flatter, faster prototype chains. Each collaborator object has a single responsibility, making the system easier to test and profile.

```ahk
; ✓ Runtime property creation with per-property validator via DefineProp
class DynamicObject {
    __New() {
        this.properties := Map()
    }

    AddProperty(name, initialValue := "", validator := "") {
        if (validator != "") {
            SetValidated(this, value) {
                if (validator.Call(value))
                    this.properties.Set(name, value)
                else
                    throw ValueError("Invalid value for " . name . ": " . value)
            }
            this.DefineProp(name, {
                get: (this) => this.properties.Get(name, ""),
                set: SetValidated
            })
        } else {
            this.DefineProp(name, {
                get: (this) => this.properties.Get(name, ""),
                set: (this, value) => this.properties.Set(name, value)
            })
        }

        this.properties.Set(name, initialValue)
    }
}

obj := DynamicObject()
obj.AddProperty("age", 0, (value) => value >= 0 && value <= 150)
obj.age := 25
MsgBox("Age: " . obj.age)

; ✓ Composition — complex objects built from focused single-responsibility collaborators
class Logger {
    __New(level := "INFO") {
        this.level := level
    }

    Log(message, level := "INFO") {
        timestamp := FormatTime(, "yyyy-MM-dd HH:mm:ss")
        MsgBox("[" . timestamp . "] " . level . ": " . message)
    }
}

class Database {
    __New(connectionString) {
        this.connectionString := connectionString
        this.connected        := false
    }

    Connect() {
        this.connected := true
        return true
    }

    Query(sql) {
        if (!this.connected)
            throw Error("Not connected to database")
        return "Result for: " . sql
    }
}

; ✓ DataService composes Logger + Database + Map cache — no inheritance needed
class DataService {
    __New(connectionString, logLevel := "INFO") {
        this.logger   := Logger(logLevel)
        this.database := Database(connectionString)
        this.cache    := Map()
    }

    Initialize() {
        this.logger.Log("Initializing data service")
        return this.database.Connect()
    }

    GetData(query, useCache := true) {
        ; ✓ Map.Has() before Map.Get() — when cache-hit and cache-miss paths differ
        if (useCache && this.cache.Has(query))
            return this.cache.Get(query)

        result := this.database.Query(query)

        if (useCache)
            this.cache.Set(query, result)

        return result
    }
}

service := DataService("server=localhost;db=test")
if (service.Initialize()) {
    result := service.GetData("SELECT * FROM users")
    MsgBox("Query result: " . result)
}
```

## TIER 6 — Validated Objects and Config Patterns
> METHODS COVERED: DefineProp · Map.Get · Map.Set · RegExMatch · ValueError (class)

Combining descriptor-level validation with structured config classes creates a self-enforcing data layer — each property rejects invalid input at assignment time, eliminating repeated defensive checks at every call site that consumes the config. The `set` descriptor becomes the single enforcement point; callers need no guards at all.

```ahk
; ✓ ValidatedConfig uses set descriptors for per-property validation — callers need no guards
class ValidatedConfig {
    __New() {
        this._settings := Map()
        this.SetupValidation()
    }

    SetupValidation() {
        ; Database host — rejects empty string at assignment time
        SetDatabaseHost(this, value) {
            if (value == "")
                throw ValueError("Database host cannot be empty")
            this._settings.Set("DatabaseHost", value)
        }
        this.DefineProp("DatabaseHost", {
            get: (this) => this._settings.Get("DatabaseHost", "localhost"),
            set: SetDatabaseHost
        })

        ; Database port — type and range validated in the descriptor, not at call sites
        SetDatabasePort(this, value) {
            if (!(value is Integer) || value < 1 || value > 65535)
                throw ValueError("Port must be between 1 and 65535")
            this._settings.Set("DatabasePort", value)
        }
        this.DefineProp("DatabasePort", {
            get: (this) => this._settings.Get("DatabasePort", 5432),
            set: SetDatabasePort
        })
    }

    ; Reusable validation helper — callable from any set descriptor in this class
    IsValidEmail(email) {
        return RegExMatch(email, "i)^[^@\s]+@[^@\s]+\.[^@\s]+$") > 0
    }
}

; ✓ Errors surface at assignment boundary — one try/catch, not scattered guards
try {
    config := ValidatedConfig()
    config.DatabaseHost := "production.db.company.com"
    config.DatabasePort := 5432

    MsgBox("Configuration valid - Host: " . config.DatabaseHost)

} catch ValueError as e {
    MsgBox("Configuration error: " . e.Message)
}

; ✗ Defensive checking at every call site — fragile, repeated, easy to miss
; host := GetConfig("host")
; if (host == "")                          ; → Must repeat this guard everywhere
;     throw ValueError("Host empty")
; port := GetConfig("port")
; if (port < 1 || port > 65535)           ; → Duplicated validation logic
;     throw ValueError("Invalid port")
```

### Performance Notes

- **Descriptor get/set is not free.** Each `DefineProp` get/set invokes a closure on every property read or write. For inner-loop hot paths, compute the value once and cache it in a plain backing property; invalidate on `set` rather than recomputing on every `get`.
- **Flat prototype chains.** Deep inheritance chains (5+ levels) increase property lookup time proportionally. Composition with single-level collaborator objects is both faster to look up and easier to profile — prefer it for performance-sensitive code.
- **BoundFunc allocation.** Each `.Bind()` call allocates a new BoundFunc. Create bound callbacks once in `__New` and store them as instance properties; never call `.Bind()` inside event handlers or timer ticks that fire repeatedly.
- **`OwnProps()` enumerator cost.** Enumerating own properties on objects with many `DefineProp` descriptors is O(n). Avoid enumerating inside tight loops; cache the key set in a Map if repeated iteration is required.
- **Map over Object for data.** Map lookup is hash-based O(1). Iterating object own properties via `OwnProps()` has overhead proportional to the number of descriptors. Map is not only semantically correct for key-value data — it is also faster.
- **`Clone()` is shallow.** Cloning a deep object graph requires recursive traversal. For performance-sensitive copy patterns, cache the clone and invalidate on mutation rather than cloning on every read.

## DROP-IN RECIPES

```ahk
; SafeGetProp — read a property from any object with a typed fallback; never throws
; ✓ Handles absent properties at any level — use wherever property existence is conditional
SafeGetProp(obj, name, default := "") {
    if !(obj is Object) && Type(obj) != "String" && !(obj is Integer) && !(obj is Float)
        throw TypeError("SafeGetProp: obj must be an AHK value", -1)
    if Type(name) != "String" || name = ""
        throw TypeError("SafeGetProp: name must be a non-empty string", -1)
    return obj.HasProp(name) ? obj.%name% : default
}
; Call site: theme := SafeGetProp(config, "theme", "light")


; BindAll — bulk-bind a list of method names on an object; stores each as obj.bound_MethodName
; ✓ Eliminates repetitive .Bind(this) calls when wiring multiple GUI events or timers
BindAll(obj, methodNames*) {
    if !(obj is Object)
        throw TypeError("BindAll: obj must be an Object", -1)
    for name in methodNames {
        if Type(name) != "String" || name = ""
            throw TypeError("BindAll: each method name must be a non-empty string", -1)
        if !obj.HasMethod(name)
            throw MethodError("BindAll: method '" . name . "' not found on object", -1)
        obj.DefineProp("bound_" . name, {value: obj.%name%.Bind(obj)})
    }
    return obj
}
; Call site: BindAll(this, "OnClick", "OnClose", "OnResize")
;            this.submitBtn.OnEvent("Click", this.bound_OnClick)


; MakeReadOnly — convert an existing value property to a get-only descriptor; throws on write attempt
; ✓ Seals a property after initialization without rewriting the class — use in __New after setup
MakeReadOnly(obj, propName) {
    if !(obj is Object)
        throw TypeError("MakeReadOnly: obj must be an Object", -1)
    if !(propName is String) || propName = ""
        throw TypeError("MakeReadOnly: propName must be a non-empty string", -1)
    if !obj.HasOwnProp(propName)
        throw PropertyError("MakeReadOnly: '" . propName . "' is not an own property of obj", -1)
    currentVal := obj.%propName%
    obj.DefineProp(propName, {get: (this) => currentVal})
    return obj
}
; Call site: MakeReadOnly(config, "ApiKey")  ; config.ApiKey now throws on assignment
```

## ANTI-PATTERNS

| Pattern | Wrong | Correct | LLM Common Cause |
|---------|-------|---------|------------------|
| `new` keyword for instantiation | `new Person("John", 30)` | `Person("John", 30)` | AHK v1 and every mainstream OOP language require `new`; v2 dropped it silently — strong cross-language training signal |
| Object literal as data container | `{key: val}` for dynamic storage | `Map("key", val)` | AHK v1 used object literals for all key-value data; v2 Map() is the designated replacement with a full safe-access API |
| Arrow + block in DefineProp descriptor | `set: (this, v) => { if v < 0 ... }` | Named function reference for any multi-line body | JS/Python lambda syntax allows multi-line bodies; AHK v2 arrow syntax does not — mixed-language training data causes this |
| Unbound method as callback | `SetTimer(this.Tick, 1000)` | `SetTimer(this.Tick.Bind(this), 1000)` | AHK v1 percent-expression syntax obscured binding mechanics; LLMs trained on v1 omit `.Bind()` entirely |
| `IsObject()` for type checking | `if IsObject(x)` | `if x is Object` or `Type(x) != "Integer"` | `IsObject()` still exists in v2 but `is` and `Type()` offer chain-aware type checking with exact class name resolution |
| Wrong prototype extension site | `Object().DefineProp(String.Prototype, "x", d)` | `ObjDP := Object.Prototype.DefineProp` then `ObjDP(String.Prototype, "x", d)` | `String.Prototype` is not Object-derived — `DefineProp` must be extracted from `Object.Prototype` and called with `String.Prototype` as the explicit receiver |
| `set => { }` block in class body | `Width { set => { this._w := value } }` | `Width { set { this._w := value } }` | Conflating DefineProp arrow descriptor syntax with class body `set` block syntax — two distinct parse contexts with different rules |
| Confusing HasProp with HasOwnProp | `obj.HasProp("method")` to check if the instance defines the method itself | `obj.HasOwnProp("method")` for own-only check | Most documentation examples show `HasProp`; the own-vs-inherited distinction is not prominent in introductory AHK v2 material |

## SEE ALSO

> This module does NOT cover: meta-functions (`__Get`, `__Set`, `__Call`, `__Delete`, `__Enum`), abstract base classes, and mixin composition patterns — see Module_Classes.md.
> This module does NOT cover: Map vs Object selection criteria, nested structure patterns, and serialization — see Module_DataStructures.md.
> This module does NOT cover: try/catch design in object methods, custom exception classes, and error propagation chains — see Module_Errors.md.

- `Module_Classes.md` — advanced class hierarchies, meta-functions (`__Get`/`__Set`/`__Call`/`__Enum`), abstract bases, and mixin patterns.
- `Module_DataStructures.md` — authoritative Map vs Object selection, nested structures, and data serialization.
- `Module_GUI.md` — complete GUI object integration, control event wiring, and GUI lifecycle management.
- `Module_Errors.md` — try/catch patterns for descriptor-level validation, custom exception classes, and propagation.
- `Module_Arrays.md` — Array iteration, transformation, and storage of object collections.