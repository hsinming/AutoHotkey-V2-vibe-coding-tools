# Module_Classes.md
<!-- DOMAIN: Classes and OOP -->
<!-- SCOPE: Prototype chain manipulation, ObjSetBase(), raw ObjPtr arithmetic, and GUI control subclassing are not covered — see Module_ClassPrototyping.md and Module_GUI.md -->
<!-- TRIGGERS: class, extends, __New, __Delete, __Get, __Set, __Call, __Enum, __Item, super, static, DefineProp, HasProp, HasMethod, ObjPtr, ObjFromPtrAddRef, "create object", "object-oriented", "instantiate", "constructor", "destructor", "method chaining", "factory pattern", "observer pattern", "resource cleanup", "weak reference", "dispose", "inheritance", "polymorphism", "mixin" -->
<!-- CONSTRAINTS: Instantiate classes with ClassName() — never `new ClassName()` (NameError). Always call .Bind(this) on any method passed to SetTimer/OnEvent/Hotkey. Use super.__New(args*) not super() in derived constructors. Never rely on __Delete() timing alone — pair it with an explicit dispose() called in a try/finally block. Meta-functions require three-param signatures: __Get(name,params), __Set(name,params,value), __Call(name,params) — two-param v1 signatures silently never fire. -->
<!-- CROSS-REF: Module_ClassPrototyping.md, Module_Objects.md, Module_GUI.md, Module_Errors.md, Module_Arrays.md -->
<!-- VERSION: AHK v2.0+ -->

## V1 → V2 BREAKING CHANGES

| v1 pattern (LLM commonly writes) | v2 correct form | Consequence |
|----------------------------------|-----------------|-------------|
| `new ClassName()` | `ClassName()` | NameError at runtime — `new` is not a keyword in AHK v2; it is parsed as a call to an undefined function named `new` |
| `super()` in derived `__New` | `super.__New(args*)` | Parent constructor never runs — base class fields are uninitialized; accessing them throws UnsetError |
| Raw method reference: `SetTimer(this.Update, 1000)` | `SetTimer(this.Update.Bind(this), 1000)` | `this` inside the callback resolves to the wrong object or throws MethodError |
| `__Get(name)` / `__Set(name, val)` two-param v1 signatures | `__Get(name, params)` / `__Set(name, params, value)` three-param v2 | Meta-function silently never fires — wrong parameter arity makes AHK v2 skip the intercept without error |
| `{key: val}` object literal as data container | `Map("key", val)` | Object literals lack `.Has()`, `.Delete()`, `.Clear()` — silent data loss on any key-existence check |
| Static member access via instance: `this.StaticProp` | `ClassName.StaticProp` | Reads or shadows the per-instance copy instead of the shared class-level value; mutations via `this` do not propagate to the class |
| Multi-line fat-arrow callback: `(x) => { return x*2 }` | Named top-level function or traditional method | AHK v2 fat arrows accept a single expression only — block bodies are a syntax error |

## API QUICK-REFERENCE

### Class Meta-Functions

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `__New()` | `__New(params*)` | — (void) | Any user-thrown error | Constructor; called automatically on `ClassName(args)`; derived classes must call `super.__New()` |
| `__Delete()` | `__Delete()` | — (void) | Should not throw — exceptions are silently swallowed by GC | Destructor; GC timing is non-deterministic — always pair with explicit `dispose()` |
| `__Get()` | `__Get(name, params)` | Value | PropertyError if unhandled | Intercepts undefined property reads; `params` is Array of bracket params; must throw if name is unrecognised |
| `__Set()` | `__Set(name, params, value)` | — (void) | Any user-thrown error | Intercepts undefined property writes; `params` is Array of bracket params; must store value or it is silently discarded |
| `__Call()` | `__Call(name, params)` | Value | MethodError if unhandled | Intercepts undefined method calls; `params` is Array of arguments; must throw MethodError for unknown names |
| `__Enum()` | `__Enum(numberOfVars)` | Enumerator object | TypeError if return is not callable | Called by `for` loop; `numberOfVars` is 1 or 2; delegate to an inner collection's `__Enum()` when wrapping |
| `__Item` | `__Item[key] { get; set }` | Value (get) / — (set) | PropertyError / any | Parameterised bracket-access: `obj["key"]` dispatches here; fat-arrow shorthand supported |

### Instance and Object Methods

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `.HasProp()` | `.HasProp(name)` | True/False | — | True if instance or any prototype owns the named property |
| `.HasMethod()` | `.HasMethod(name?)` | True/False | — | Omit name to test if object itself is callable |
| `.DefineProp()` | `.DefineProp(name, desc)` | Self (the object) | ValueError if desc is invalid | `desc` must be `{value:}`, `{get:}`, `{set:}`, or `{call:}` — not a Map |
| `.DeleteProp()` | `.DeleteProp(name)` | Removed value | — | Removes an own property; inherited prototype properties are unaffected |
| `.GetOwnPropDesc()` | `.GetOwnPropDesc(name)` | Descriptor object | PropertyError if absent | Returns `{value:}`, `{get:}`, `{set:}`, or `{call:}` descriptor |
| `.Bind()` | `method.Bind(thisArg, params*)` | BoundFunc | — | Returns bound callable with fixed `this`; mandatory for all timer/event/hotkey callbacks |
| `.__Class` | `obj.__Class` | String | — | Class name of the object; useful for pool key lookup and debug logging |
| `super.__New()` | `super.__New(args*)` | — (void) | Same as base constructor | Calls base-class constructor from a derived `__New`; must be called before using base fields |
| `super.method()` | `super.methodName(args*)` | Base return value | MethodError if not found in base | Calls base-class implementation from any override, not just `__New` |

### Object Utility Functions

| Function | Signature | Returns | Throws | Notes |
|----------|-----------|---------|--------|-------|
| `ObjPtr()` | `ObjPtr(obj)` | Integer (address) | — | Does NOT increment ref count — pointer only; object may be GC'd while pointer exists |
| `ObjPtrAddRef()` | `ObjPtrAddRef(obj)` | Integer (address) | — | Retrieves pointer AND increments ref count; use when storing address long-term |
| `ObjFromPtrAddRef()` | `ObjFromPtrAddRef(ptr)` | Object reference | Crash/indeterminate if object already freed | Reconstructs object reference from pointer; always call inside `try/catch` to detect dead objects |
| `ObjRelease()` | `ObjRelease(ptr)` | New ref count | — | Decrements ref count; use to balance a manual `ObjAddRef` or `ObjPtrAddRef` |
| `IsSet()` | `IsSet(varOrProp)` | True/False | — | Returns true if variable/property has a value; use instead of comparing to `unset` |

### Map (preferred class state container)

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `Map()` | `Map(k, v, ...)` | Map | — | Preferred key-value container for class state; never use `{k: v}` object literals for data |
| `.Has()` | `.Has(key)` | True/False | — | Existence check — always use before direct `[key]` access |
| `.Get()` | `.Get(key, default)` | Value or default | UnsetItemError if key absent and no default given | `default` must be a concrete value — passing `unset` removes the default and makes Get throw |
| `.Clear()` | `.Clear()` | — (void) | — | Remove all entries; call in `dispose()` or `__Delete()` |
| `.Delete()` | `.Delete(key)` | Removed value | — | Remove a single key; returns the removed value |
| `.Count` | `.Count` | Integer | — | Number of key-value pairs currently stored |

## AHK V2 CONSTRAINTS

- ✗ `new ClassName()` — NameError at runtime; `new` is not a keyword in AHK v2
- ✓ `ClassName(args*)` — bare call without `new` is the only valid instantiation form

- ✗ `SetTimer(this.Method, 1000)` — `this` is absent or wrong inside the callback
- ✓ `SetTimer(this.Method.Bind(this), 1000)` — `.Bind(this)` is mandatory for any timer, GUI event, hotkey handler, or `OnMessage` callback that references instance state; store the BoundFunc as an instance field in `__New` to allow cancellation

- ✗ `super()` in a derived `__New` — base constructor is never executed
- ✓ `super.__New(args*)` — explicit method call on the base class prototype; call before any access to inherited fields

- ✗ Relying on `__Delete()` alone for resource release — GC timing is non-deterministic; resources will leak on exception paths
- ✓ Always provide a named `dispose()` containing all cleanup logic; call `dispose()` explicitly in a `try/finally` block; let `__Delete()` delegate to `dispose()` as a GC safety net only

- ✗ `__Get(name)` / `__Set(name, val)` two-parameter meta-function signatures
- ✓ `__Get(name, params)` / `__Set(name, params, value)` — the `params` Array argument is required in v2 even if unused; omitting it silently prevents the meta-function from firing

- ✗ `this.ClassName.StaticMethod()` or `this.StaticProp` for class-level members
- ✓ `ClassName.StaticMethod()` / `ClassName.StaticProp` — static members belong to the class object, not to any instance; accessing via `this` creates or reads a per-instance shadow

- ✗ `map.Get(key, unset)` as a "safe optional return" — passing `unset` removes the default argument, making Get throw UnsetItemError on absent keys
- ✓ `if map.Has(key) { return map[key] }` — use an explicit `.Has()` guard when the absent case should return nothing

- ✗ `{key: val}` object literal for class state or component spec Maps — `.Has()`, `.Delete()`, `.Clear()` do not exist on plain objects
- ✓ `Map("key", val)` — use Map for any key-value storage that requires existence checks; reserve `{}` object literals for DefineProp descriptor objects only

**Method naming contracts — enforced at review time:**
- **`Get*` / `Fetch*` / `Read*` method prefix → side-effect-free** — must not mutate `this.*` properties, write to global state, or trigger I/O. Violation severity: **Major**.
  - ✗ `GetStatus()` that sets `this.lastChecked := A_Now` — mutation inside a read-prefix method
  - ✓ `GetStatus()` that returns `this.status` — pure read; safe to call in any context including loops
- **`Check*` / `Is*` / `Has*` method prefix → pure boolean predicate** — must return `true`/`false` without modifying any `this.*` property or external state.
  - ✗ `CheckSession()` that resets `this.hitCount := 0` — command masquerading as a predicate
  - ✓ `IsSessionExpired()` + separate `ResetSession()` — predicate and mutation are distinct methods
- **`Set*` / `Reset*` / `Clear*` / `Update*` / `Flush*` prefix → mutation declared** — any method that changes `this.*` properties or external state must carry one of these verb prefixes.

Safe-access priority order for class state Maps:
  1. `.Get(key, default)` — optional key, one-line resolution, never throws when default is a concrete value
  2. `.Has(key)` — when present/absent branches differ significantly in logic
  3. `.Default` property — when the entire Map should share one universal fallback value
  4. `try/catch` — only when the exception itself carries diagnostic information needed at the call site

Unset variable handling: use `IsSet(var)` before reading a variable or property that may not have been assigned; compare with `!= unset` only in contexts where `IsSet()` cannot be applied.

Resource lifecycle: every object that opens a file handle, COM reference, timer, or network socket must provide a `dispose()` method called in `try/finally`; `__Delete()` must delegate to `dispose()` as a backup, not as the primary cleanup path.

## AGENT QA CHECKLIST

- [ ] Did I use `ClassName()` (no `new`) for every class instantiation?
- [ ] Did I call `.Bind(this)` on every method reference passed to `SetTimer`, `OnEvent`, `Hotkey`, or `OnMessage`?
- [ ] Did I use `super.__New(args*)` (not `super()`) in every derived class constructor?
- [ ] Did I implement an explicit `dispose()` method and wrap every resource acquisition in `try/finally { obj.dispose() }` instead of relying on `__Delete()` timing?
- [ ] Did I use three-parameter signatures for all meta-functions: `__Get(name, params)`, `__Set(name, params, value)`, `__Call(name, params)`?

## RUNTIME ERROR MAPPING

| Exception Class | Trigger Condition | Detection Code | Fix |
|----------------|-------------------|----------------|-----|
| `NameError` | `new ClassName()` — `new` parsed as undefined function call | `e.Message` contains "new" | Remove the `new` keyword; use `ClassName(args*)` directly |
| `MethodError` / wrong `this` | Passing a raw method reference `this.Method` to `SetTimer` or `OnEvent` without `.Bind(this)` | Callback runs but `this.*` properties are wrong or missing; `e.Message` may mention undefined property | Add `.Bind(this)` — store the BoundFunc as `this.timerCallback := this.Method.Bind(this)` in `__New` |
| `UnsetError` / base fields missing | Derived `__New` accesses `this.baseField` before calling `super.__New()` | `e.Message` references field name; `e.What` shows derived constructor | Move `super.__New(args*)` to the first line of the derived `__New` before any `this.*` access |

## TIER 1 — Class Fundamentals: Constructors, Properties, and Static Members
> METHODS COVERED: `__New` · `static` · property `get`/`set` · fat-arrow property · parameterised property · `.Bind()` · `SetTimer`

A class is a prototype-based blueprint: `class Name { }` at the top level, instantiated by calling `Name(args)` — never with `new`. Instance fields declared at class scope (outside any method) are per-instance defaults. Static fields declared with the `static` keyword are shared across all instances and accessed via `ClassName.Field`. Properties use `get`/`set` accessor blocks; single-expression accessors use the fat-arrow shorthand. Parameterised properties (`prop[key]`) let bracket-notation route to custom getter/setter logic. Any method passed as a callback — to `SetTimer`, `OnEvent`, or `Hotkey` — must be bound with `.Bind(this)`, stored as an instance field in `__New`, and that stored BoundFunc used for both starting and cancelling the timer.

```ahk
; ✓ Instantiate without "new" — ClassName() is the only valid form in AHK v2
class Animal {
    name := ""
    age  := 0

    __New(name, age) {
        this.name := name
        this.age  := age
    }

    speak() {
        MsgBox("Generic animal sound from " this.name)
    }

    getInfo() {
        return this.name " is " this.age " years old"
    }
}

; ✓ Static members belong to the class, not to any instance — access via ClassName.Prop
class MathUtils {
    static PI      := 3.14159
    static version := "1.0"

    static calculateArea(radius) {
        return MathUtils.PI * radius * radius
    }

    static getVersion() {
        return MathUtils.version
    }
}

animal  := Animal("Buddy", 5)
area    := MathUtils.calculateArea(10)
version := MathUtils.getVersion()

; ✗ "new" keyword is not valid in AHK v2 — NameError at runtime
; animal := new Animal("Buddy", 5)    ; → NameError

; ✓ Full get/set accessor blocks — setter enforces invariants at assignment time
class Person {
    _name := ""
    _age  := 0

    name {
        get {
            return this._name
        }
        set {
            if Type(value) != "String" || value = ""
                throw ValueError("Name must be a non-empty string")
            this._name := value
        }
    }

    ; ✓ Fat-arrow shorthand for simple get/set — single expression only, no block body
    age {
        get => this._age
        set => this._age := Max(0, Integer(value))
    }

    ; ✓ Fat-arrow read-only computed property — no set block needed
    displayName => this._name " (" this._age " years old)"

    ; ✓ Parameterised property — bracket key selects which sub-value to read/write
    phoneNumber[type] {
        get {
            return this.phoneNumbers.Get(type, "")
        }
        set {
            if !this.HasProp("phoneNumbers")
                this.phoneNumbers := Map()
            this.phoneNumbers[type] := value
        }
    }

    __New(name, age) {
        this.name := name    ; Calls the setter — validation runs here
        this.age  := age     ; Calls the setter — clamps negative values
    }
}

person := Person("John", 25)
person.phoneNumber["mobile"] := "555-1234"
displayText := person.displayName

; ✓ .Bind(this) stored as instance field — mandatory for every method-as-callback
class TooltipTimer {
    static Config := Map(
        "interval",    1000,
        "startDelay",  0,
        "initialText", "Timer started",
        "format",      "Time elapsed: {1} seconds"
    )

    __New() {
        this.state         := Map("seconds", 0, "isActive", true)
        ; ✓ BoundFunc stored in __New so the same object cancels and starts the timer
        this.timerCallback := this.UpdateDisplay.Bind(this)
        this.Start()
    }

    Start() {
        ToolTip(TooltipTimer.Config["initialText"])
        SetTimer(this.timerCallback, TooltipTimer.Config["interval"])
    }

    UpdateDisplay() {
        this.state["seconds"]++
        ToolTip(Format(TooltipTimer.Config["format"], this.state["seconds"]))
    }

    __Delete() {
        if this.state["isActive"] {
            SetTimer(this.timerCallback, 0)
            this.state["isActive"] := false
            ToolTip()
        }
    }
}

; ✗ Raw method reference without .Bind — "this" inside UpdateDisplay will be wrong
; SetTimer(this.UpdateDisplay, 1000)    ; → wrong "this" context, MethodError likely

timer := TooltipTimer()
```

## TIER 2 — Inheritance and Polymorphism
> METHODS COVERED: `extends` · `super.__New()` · `super.method()` · abstract method pattern

A derived class inherits all methods and properties of its base using `class Child extends Parent`. The derived `__New` must call `super.__New(args*)` explicitly and as the first statement — AHK v2 does not auto-chain constructors, and accessing inherited fields before the super call causes UnsetError. `super.method()` dispatches to the base-class implementation from inside any override, not just the constructor. Declaring a base method that always throws creates an enforced abstract interface: derived classes that omit the implementation receive a descriptive error at call time rather than silent incorrect behaviour.

```ahk
; ✓ Base class defines shared state and enforces interface via explicit throw
class Vehicle {
    make  := ""
    model := ""
    year  := 0

    __New(make, model, year) {
        this.make  := make
        this.model := model
        this.year  := year
    }

    getDescription() {
        return this.year " " this.make " " this.model
    }

    start() {
        MsgBox("Starting " this.getDescription())
    }

    ; ✓ Abstract method pattern — force derived classes to implement by throwing in base
    getMaxSpeed() {
        throw Error("getMaxSpeed must be implemented by derived class")
    }
}

; ✓ Derived class calls super.__New() first — base fields initialized before this.* access
class Car extends Vehicle {
    doors    := 4
    fuelType := "gasoline"

    __New(make, model, year, doors, fuelType) {
        super.__New(make, model, year)  ; Base fields initialised first — must precede all this.* writes
        this.doors    := doors
        this.fuelType := fuelType
    }

    ; ✓ Override calls super.start() then adds Car-specific behaviour
    start() {
        super.start()
        MsgBox("Car engine started")
    }

    getMaxSpeed() {
        return 120  ; mph
    }
}

class Motorcycle extends Vehicle {
    hasSidecar := false

    __New(make, model, year, hasSidecar := false) {
        super.__New(make, model, year)
        this.hasSidecar := hasSidecar
    }

    start() {
        super.start()
        MsgBox("Motorcycle engine roared to life")
    }

    getMaxSpeed() {
        return 180  ; mph
    }
}

; ✓ Polymorphism: for-loop calls start()/getMaxSpeed() on each concrete type
vehicles := [
    Car("Toyota", "Camry", 2023, 4, "hybrid"),
    Motorcycle("Harley", "Sportster", 2023, false)
]

for vehicle in vehicles {
    vehicle.start()
    speed := vehicle.getMaxSpeed()
    MsgBox(vehicle.getDescription() " - Max Speed: " speed " mph")
}

; ✗ super() is not a constructor call in AHK v2 — super.__New(args*) is the only valid form
; super()    ; → runtime error: calls base class constructor on a new object, not this; base fields of this remain uninitialized
```

## TIER 3 — Meta-Functions and Fluent Interfaces
> METHODS COVERED: `__Get` · `__Set` · `__Call` · `__Enum` · `__Item` · `.DefineProp()` · `.HasProp()` · `.Has()` · `.Get()` · `RegExMatch()`

Meta-functions intercept property/method access at the prototype level. `__Get(name, params)` fires on reads of undefined properties; `__Set(name, params, value)` fires on writes; `__Call(name, params)` fires on calls of undefined methods. The three-parameter v2 signatures are mandatory — two-parameter v1 signatures silently never fire. `params` is an Array of bracket-notation parameters. `__Enum(n)` is called by `for` loops and must return an enumerator matching the variable count. `__Item` defines bracket-access via a parameterised property. The fluent-interface pattern requires every builder method to `return this`.

```ahk
; ✓ Meta-functions intercept undefined access — always handle recognised names or throw a typed error
class ConfigManager {
    _settings := Map()
    _defaults := Map("theme", "dark", "language", "en", "autoSave", true)

    __New() {
        for key, value in this._defaults
            this._settings[key] := value
    }

    ; ✓ __Get(name, params) — v2 requires params argument even if unused
    __Get(name, params) {
        if this._settings.Has(name)
            return this._settings[name]

        if name = "configCount"
            return this._settings.Count
        if name = "allKeys" {
            allKeys := []
            for k in this._settings
                allKeys.Push(k)
            return allKeys
        }

        throw PropertyError("Unknown configuration: " name)
    }

    ; ✓ __Set(name, params, value) — three-parameter v2 signature is mandatory
    __Set(name, params, value) {
        if !RegExMatch(name, "^[a-zA-Z][a-zA-Z0-9_]*$")
            throw ValueError("Invalid setting name: " name)

        this._settings[name] := value
        ; ✓ DefineProp descriptor uses object literal {value:} — correct: this is a descriptor, not a data Map
        this.DefineProp(name, {value: value})
    }

    ; ✓ __Call(name, params) — params is an Array of call arguments
    __Call(name, params) {
        if RegExMatch(name, "^get(.+)$", &match) {
            settingName := StrLower(SubStr(match[1], 1, 1)) SubStr(match[1], 2)
            return this._settings.Get(settingName, "")
        }

        if RegExMatch(name, "^set(.+)$", &match) && params.Length = 1 {
            settingName := StrLower(SubStr(match[1], 1, 1)) SubStr(match[1], 2)
            this._settings[settingName] := params[1]
            return this
        }

        throw MethodError("Unknown method: " name)
    }

    ; ✓ __Enum enables for-in iteration — delegate to the inner Map's own enumerator
    __Enum(numberOfVars) {
        return this._settings.__Enum(numberOfVars)
    }

    ; ✓ __Item defines bracket-access: config["key"] dispatches get/set here
    __Item[key] {
        get => this._settings[key]
        set => this._settings[key] := value
    }
}

config := ConfigManager()
config.newSetting := "value"    ; __Set fires
value  := config.newSetting     ; __Get fires
config.setTheme("light")        ; __Call fires
theme  := config.getTheme()     ; __Call fires
config["debug"] := true         ; __Item set fires

for key, value in config
    MsgBox(key ": " value)

; ✗ v1 two-parameter meta-function signatures — interceptors silently never fire in v2
; __Get(name)         ; → wrong arity, AHK v2 skips this meta-function entirely
; __Set(name, value)  ; → wrong arity, AHK v2 skips this meta-function entirely

; ✓ Fluent interface: every builder method returns this, enabling left-to-right dot-chaining
class QueryBuilder {
    _table      := ""
    _columns    := []
    _conditions := []
    _orderBy    := []
    _limit      := 0

    from(table) {
        this._table := table
        return this
    }

    select(columns*) {
        for column in columns
            this._columns.Push(column)
        return this
    }

    where(condition) {
        this._conditions.Push(condition)
        return this
    }

    orderBy(column, direction := "ASC") {
        this._orderBy.Push(column " " direction)
        return this
    }

    limit(count) {
        this._limit := count
        return this
    }

    build() {
        if this._table = ""
            throw ValueError("Table name is required")

        query  := "SELECT "
        query .= this._columns.Length > 0 ?
            this._joinArray(this._columns, ", ") : "*"
        query .= " FROM " this._table

        if this._conditions.Length > 0
            query .= " WHERE " this._joinArray(this._conditions, " AND ")

        if this._orderBy.Length > 0
            query .= " ORDER BY " this._joinArray(this._orderBy, ", ")

        if this._limit > 0
            query .= " LIMIT " this._limit

        return query
    }

    _joinArray(array, separator) {
        result := ""
        for i, item in array {
            if i > 1
                result .= separator
            result .= item
        }
        return result
    }
}

; ✓ All builder methods return this — chain resolves left-to-right without temporaries
query := QueryBuilder()
    .from("users")
    .select("id", "name", "email")
    .where("active = 1")
    .where("age >= 18")
    .orderBy("name")
    .limit(10)
    .build()

MsgBox(query)
```

## TIER 4 — Nested Classes and Factory Patterns
> METHODS COVERED: nested class access via `Outer.Inner()` · `static` factory methods · `Map()` for component specs · `.Push()` · `.Has()` · `switch`

Nested class declarations (`class Inner { }` inside `class Outer { }`) are accessed as `Outer.Inner()`. They do not capture any implicit reference to the outer class — all shared state must be reached through `Outer.StaticProp`. Factory methods are static methods that centralise object creation and apply shared configuration before returning instances. Component specification Maps must use `Map()` not object literals so `.Has()` checks remain reliable for optional keys.

```ahk
class UIComponentFactory {
    static theme := "default"

    static setTheme(newTheme) {
        UIComponentFactory.theme := newTheme
    }

    ; ✓ Nested class — accessed as UIComponentFactory.Button(), not Button()
    class Button {
        text   := ""
        x      := 0
        y      := 0
        width  := 100
        height := 30

        __New(text, x, y, width := 100, height := 30) {
            this.text   := text
            this.x      := x
            this.y      := y
            this.width  := width
            this.height := height
        }

        render(gui) {
            options := "x" this.x " y" this.y " w" this.width " h" this.height
            return gui.AddButton(options, this.text)
        }

        ; ✓ Access outer class config via ClassName.StaticProp — never via this
        getThemedColor() {
            switch UIComponentFactory.theme {
                case "dark":  return 0x333333
                case "light": return 0xFFFFFF
                default:      return 0xF0F0F0
            }
        }
    }

    class TextInput {
        placeholder := ""
        x           := 0
        y           := 0
        width       := 200
        height      := 25

        __New(placeholder, x, y, width := 200, height := 25) {
            this.placeholder := placeholder
            this.x           := x
            this.y           := y
            this.width       := width
            this.height      := height
        }

        render(gui) {
            options := "x" this.x " y" this.y " w" this.width " h" this.height
            return gui.AddEdit(options, this.placeholder)
        }
    }

    static createButton(text, x, y, width?, height?) {
        return UIComponentFactory.Button(text, x, y, width?, height?)
    }

    static createTextInput(placeholder, x, y, width?, height?) {
        return UIComponentFactory.TextInput(placeholder, x, y, width?, height?)
    }

    ; ✓ Factory reads component specs from Map — .Has() reliable for optional keys
    static createForm(components) {
        result := []
        for component in components {
            switch component["type"] {  ; <!-- CONVERTED: component.type → component["type"] — Map keys require bracket access, not dot notation -->
                case "button":
                    w := component.Has("width")  ? component["width"]  : unset
                    h := component.Has("height") ? component["height"] : unset
                    result.Push(UIComponentFactory.createButton(
                        component["text"], component["x"], component["y"], w?, h?))
                case "input":
                    w := component.Has("width")  ? component["width"]  : unset
                    h := component.Has("height") ? component["height"] : unset
                    result.Push(UIComponentFactory.createTextInput(
                        component["placeholder"], component["x"], component["y"], w?, h?))
            }
        }
        return result
    }
}

UIComponentFactory.setTheme("dark")

; ✓ Component specs as Map() — .Has() remains reliable for optional width/height keys
formSpec := [
    Map("type", "input",  "placeholder", "Enter name",  "x", 10, "y", 10),            ; <!-- CONVERTED: {type:...,placeholder:...,x:...,y:...} → Map() — object literals violate key-value storage constraint -->
    Map("type", "input",  "placeholder", "Enter email", "x", 10, "y", 40),            ; <!-- CONVERTED: same -->
    Map("type", "button", "text",        "Submit",      "x", 10, "y", 70, "width", 80) ; <!-- CONVERTED: same -->
]

components := UIComponentFactory.createForm(formSpec)

; ✗ Object literals as data containers — .Has() / .Delete() / .Clear() unavailable
; formSpec := [{type: "input", placeholder: "Enter name", x: 10, y: 10}]  ; → no .Has("width") possible
```

## TIER 5 — Resource Management and Lifecycle
> METHODS COVERED: `__Delete` · `dispose()` · `.HasMethod()` · `.HasProp()` · `Map.Clear()` · `FileDelete()` · `FileAppend()` · `A_Temp` · `A_TickCount`

`__Delete()` is the AHK v2 destructor, but its execution time is non-deterministic — the garbage collector may defer it or skip it on abnormal exit. The correct pattern is: implement a named `dispose()` containing all cleanup logic, wrap every resource acquisition in `try/finally { resource.dispose() }`, and let `__Delete()` delegate to `dispose()` as a safety net only. The `ResourceManager.use()` static utility encodes this RAII discipline as a reusable pattern. Fat-arrow syntax accepts only a single expression — any block-body callback must be a named top-level function.

```ahk
; ✓ Explicit dispose() + __Delete() delegation — deterministic cleanup on all code paths
class DatabaseConnection {
    _connectionString := ""
    _isConnected      := false
    _queryCache       := Map()
    _tempFiles        := []

    __New(connectionString) {
        this._connectionString := connectionString
        this.connect()
    }

    connect() {
        if this._isConnected
            return

        try {
            this._isConnected := true
            MsgBox("Connected to database")
        } catch as err {
            throw Error("Failed to connect: " err.Message)
        }
    }

    query(sql, params*) {
        if !this._isConnected
            throw Error("Not connected to database")

        cacheKey := sql
        if this._queryCache.Has(cacheKey)
            return this._queryCache[cacheKey]

        result := this._executeQuery(sql, params*)
        this._queryCache[cacheKey] := result
        return result
    }

    _executeQuery(sql, params*) {
        return ["result1", "result2", "result3"]
    }

    createTempFile() {
        tempFile := A_Temp "\db_temp_" A_TickCount ".tmp"
        FileAppend("temp data", tempFile)
        this._tempFiles.Push(tempFile)
        return tempFile
    }

    ; ✓ dispose() is idempotent — safe to call multiple times without double-free
    dispose() {
        if !this._isConnected
            return

        this._queryCache.Clear()

        for tempFile in this._tempFiles {
            try {
                FileDelete(tempFile)
            } catch {
                ; Continue cleanup even if one file deletion fails
            }
        }
        this._tempFiles   := []
        this._isConnected := false
        MsgBox("Database connection closed")
    }

    ; ✓ __Delete delegates to dispose() — GC safety net, not the primary cleanup path
    __Delete() {
        this.dispose()
    }
}

; ✓ RAII helper — callback receives resource; finally block always calls dispose()
class ResourceManager {
    static use(resource, callback) {
        try {
            return callback(resource)
        } finally {
            if resource.HasMethod("dispose")
                resource.dispose()
        }
    }
}

; ✓ Explicit try/finally — dispose() runs even when query throws
db := DatabaseConnection("server=localhost;database=test")
try {
    results  := db.query("SELECT * FROM users")
    tempFile := db.createTempFile()
} finally {
    db.dispose()
}

; ✓ Named function replaces block-body fat-arrow — block bodies are invalid AHK v2 syntax
_runDbQuery(db) {                 ; <!-- CONVERTED: (db) => { results := db.query(...); return results } invalid AHK v2 — fat-arrow requires single expression; replaced with named function -->
    return db.query("SELECT * FROM users")
}
ResourceManager.use(DatabaseConnection("server=localhost;database=test"), _runDbQuery)

; ✗ Multi-line fat-arrow body is a syntax error in AHK v2
; ResourceManager.use(conn, (db) => {    ; → syntax error — block body not valid after =>
;     results := db.query("SELECT * FROM users")
;     return results
; })
```

## TIER 6 — Observer Pattern and Weak References
> METHODS COVERED: `ObjPtr()` · `ObjFromPtrAddRef()` · `.Clone()` · `.Clear()` · `.Delete()` · `RemoveAt()` · `IsSet()` · `OutputDebug()`

Circular references prevent garbage collection: if Model holds a strong reference to View and View holds a strong reference to Model, neither is freed when both go out of scope. The weak-reference pattern stores `ObjPtr(target)` — an integer, not a reference — instead of the object itself. On each `emit()`, call `ObjFromPtrAddRef(ptr)` inside `try/catch` to probe liveness: if the object was collected the call produces indeterminate results, the catch fires, and the dead listener is removed. Cloning the listener list before iteration allows safe removal during emit without invalidating the iterator.

```ahk
; ✓ EventEmitter with weak-reference subscriber tracking — prevents circular reference leaks
class EventEmitter {
    _listeners := Map()
    _weakRefs  := Map()

    ; ✓ Pass target object to store weak reference — emitter does not own the subscriber
    on(event, callback, target := unset) {
        if !this._listeners.Has(event)
            this._listeners[event] := []

        listener := {callback: callback}

        if IsSet(target) {
            listener.targetPtr := ObjPtr(target)   ; Integer pointer — not a reference; does not prevent GC
            listener.weakRef   := true

            if !this._weakRefs.Has(listener.targetPtr)
                this._weakRefs[listener.targetPtr] := []
            this._weakRefs[listener.targetPtr].Push(listener)
        }

        this._listeners[event].Push(listener)
        return this
    }

    off(event, callback := unset) {
        if !this._listeners.Has(event)
            return this

        listeners := this._listeners[event]
        if !IsSet(callback) {
            for listener in listeners
                this._cleanupWeakRef(listener)
            this._listeners[event] := []
        } else {
            for i, listener in listeners {
                if listener.callback = callback {
                    this._cleanupWeakRef(listener)
                    listeners.RemoveAt(i)
                    break
                }
            }
        }
        return this
    }

    ; ✓ Clone listener list before iterating — safe removal of dead entries during emit
    emit(event, args*) {
        if !this._listeners.Has(event)
            return this

        listeners := this._listeners[event].Clone()

        for i, listener in listeners {
            if listener.HasProp("weakRef") {
                try {
                    ObjFromPtrAddRef(listener.targetPtr)
                } catch {
                    ; Object was collected — remove the dead listener and skip
                    this._removeDeadListener(event, listener)
                    continue
                }
            }

            try {
                listener.callback(args*)
            } catch as err {
                OutputDebug("Event listener error: " err.Message)
            }
        }
        return this
    }

    _cleanupWeakRef(listener) {
        if !listener.HasProp("weakRef")
            return

        if this._weakRefs.Has(listener.targetPtr) {
            refs := this._weakRefs[listener.targetPtr]
            for i, ref in refs {
                if ref = listener {
                    refs.RemoveAt(i)
                    break
                }
            }
            if refs.Length = 0
                this._weakRefs.Delete(listener.targetPtr)
        }
    }

    _removeDeadListener(event, deadListener) {
        if !this._listeners.Has(event)
            return

        listeners := this._listeners[event]
        for i, listener in listeners {
            if listener = deadListener {
                listeners.RemoveAt(i)
                this._cleanupWeakRef(listener)
                break
            }
        }
    }

    __Delete() {
        for event, listeners in this._listeners {
            for listener in listeners
                this._cleanupWeakRef(listener)
        }
        this._listeners.Clear()
        this._weakRefs.Clear()
    }
}

; ✓ Model extends EventEmitter — broadcasts changes to all registered subscribers
class Model extends EventEmitter {
    _data := Map()

    set(key, value) {
        ; ✓ Has() guard before access — returns unset cleanly when key is absent
        oldValue := this._data.Has(key) ? this._data[key] : unset  ; <!-- CONVERTED: Map.Get(key, unset) passes no default — throws UnsetItemError on absent key; replaced with Has() guard -->
        this._data[key] := value
        this.emit("change", {key: key, value: value, oldValue: oldValue})
    }

    get(key) {
        if this._data.Has(key)
            return this._data[key]
        ; Implicit: returns unset (no value) if key is absent
    }
}

; ✓ View registers with weak reference — Model does not hold a strong ref back to View
class View {
    model := unset

    __New(model) {
        this.model := model
        ; ✓ Passing "this" as third arg stores ObjPtr(this) — not a strong reference; GC can still collect View
        this.model.on("change", this.onModelChange.Bind(this), this)
    }

    onModelChange(data) {
        MsgBox("View updated: " data.key " = " data.value)
    }

    __Delete() {
        MsgBox("View destroyed")
    }
}

; No memory leak: when view goes out of scope the weak reference allows GC to collect it
model := Model()
view  := View(model)
model.set("name", "John")   ; View receives change notification
view  := unset              ; View is GC'd; emitter cleans up dead listener on next emit

; ✗ Omitting the target arg stores no weak reference — strong ref creates circular ref
; this.model.on("change", this.onModelChange.Bind(this))  ; → strong ref → circular ref → neither object frees
```

### Performance Notes

**Object pooling** eliminates repeated allocation for frequently created short-lived objects. `ObjectPool` maintains per-type queues (`_pools` Map keyed by `.__Class`). `acquire()` pops from the pool or creates a new instance with `%className%(params*)` (dynamic class call by name string). `release()` calls `cleanup()` if present, then pushes back. Pool objects must implement `reset(params*)` and `cleanup()` rather than relying on `__New`/`__Delete` to avoid lifecycle interference. Pooling is beneficial when object allocation is measurably hot (>10k instances/sec); profile before introducing it.

**Lazy initialisation** defers expensive computation to first access. A property getter checks `_initialized` and caches the result — subsequent reads return O(1). An `invalidate()` method clears the flag to force recomputation. Strictly better than computing in `__New` when the property may never be accessed.

**Copy-on-write** defers Array copying until mutation. `clone()` shares the backing `_data` Array reference and sets `_isShared := true` on both copies. The first write calls `_ensureUnique()` which calls `.Clone()` (O(n) once) then clears the flag — reads remain O(1); subsequent writes are O(1).

**General class performance rules:**
- Prefer `Map` over nested object literals for class state — Map lookup is O(1) with reliable `.Has()`.
- Cache `.Bind(this)` results as instance fields in `__New` — never re-bind on every timer fire or event.
- Avoid calling `DefineProp()` inside hot loops — it modifies the prototype object on every call.
- Initialise static Maps and lookup tables once at class scope, not reconstructed per-instance in `__New`.
- Prefer `ObjPtr()` over storing the full object reference when implementing subscriber registries — pointer storage is O(1) and never inflates ref counts.

## DROP-IN RECIPES

```ahk
; MakeWeakCallback — creates a timer/event callback that does not prevent GC of the target
; ✓ Stores ObjPtr(target) — fires method only if target is still alive; cleans up timer on death
; ✓ Returns a BoundFunc ready for SetTimer/OnEvent/Hotkey without holding a strong reference
MakeWeakCallback(target, methodName, timerRef := unset) {
    if !(target is Object)
        throw TypeError("MakeWeakCallback: target must be an object", -1)
    if !(methodName is String) || methodName = ""
        throw TypeError("MakeWeakCallback: methodName must be a non-empty string", -1)
    if !target.HasMethod(methodName)
        throw MethodError("MakeWeakCallback: method '" methodName "' not found on target", -1)

    ptr := ObjPtr(target)  ; Integer — does not prevent GC

    _weakDispatch(args*) {
        try {
            obj := ObjFromPtrAddRef(ptr)
        } catch {
            ; Target was collected — cancel the timer if a reference was provided
            if IsSet(timerRef) && IsSet(%timerRef%)
                SetTimer(%timerRef%, 0)
            return
        }
        obj.%methodName%(args*)
    }

    return _weakDispatch
}
; Call site:
;   cb := MakeWeakCallback(this, "UpdateDisplay", &timerRef)
;   timerRef := cb
;   SetTimer(cb, 1000)

; ─────────────────────────────────────────────────────────────────────────────

; Disposable — base class that provides RAII lifecycle for any resource-holding class
; ✓ Subclass overrides _onDispose() — no need to re-implement try/finally or idempotency guard
; ✓ __Delete delegates to dispose() as a GC safety net — explicit dispose() is still required
class Disposable {
    _disposed := false

    ; ✓ dispose() is idempotent — safe to call multiple times
    dispose() {
        if this._disposed
            return
        this._disposed := true
        try {
            this._onDispose()
        } catch as err {
            ; Log but do not rethrow — dispose must not throw into GC finalizer
            OutputDebug("Disposable._onDispose error: " err.Message)
        }
    }

    ; ✓ Override this method in subclasses — never override dispose() directly
    _onDispose() {
        ; Base implementation is a no-op — subclasses add resource teardown here
    }

    ; ✓ GC safety net — delegates to dispose() so cleanup always runs
    __Delete() {
        this.dispose()
    }

    ; ✓ Static RAII entry point — callback receives the resource; finally always calls dispose()
    static use(resource, callback) {
        if !(resource is Disposable)
            throw TypeError("Disposable.use: resource must extend Disposable", -1)
        try {
            return callback(resource)
        } finally {
            resource.dispose()
        }
    }
}

; Usage — subclass Disposable and override _onDispose() for deterministic cleanup:
class TempFileWriter extends Disposable {
    _path := ""

    __New(path) {
        this._path := path
        FileAppend("", path)  ; Create file
    }

    write(text) {
        if this._disposed
            throw Error("TempFileWriter: already disposed")
        FileAppend(text, this._path)
    }

    _onDispose() {
        if FileExist(this._path) != ""
            FileDelete(this._path)
    }
}
; Call site: Disposable.use(TempFileWriter(A_Temp "\work.tmp"), (w) => w.write("data"))
```

## ANTI-PATTERNS

| Pattern | Wrong | Correct | LLM Common Cause |
|---------|-------|---------|------------------|
| Class instantiation with `new` | `obj := new ClassName()` | `obj := ClassName()` | AHK v1 and almost all OOP language training data (Java, C#, JS, Python) use `new` |
| Object literal as data container | `state := {x: 0, y: 0}` | `state := Map("x", 0, "y", 0)` | AHK v1 training data — object literals were the only key-value container pre-v2 |
| Raw method reference as callback | `SetTimer(this.Update, 1000)` | `SetTimer(this.Update.Bind(this), 1000)` | Global-function paradigm habit — free functions do not need context binding; LLMs miss the instance-method distinction |
| `super()` as constructor call | `super(args*)` in `__New` | `super.__New(args*)` | Python/Java training data where `super().__init__()` is idiomatic |
| Static member access via `this` | `this.StaticProp := val` | `ClassName.StaticProp := val` | Python (`cls.prop`) / Java (`this.field` for both) training data conflation between instance and class members |
| Multi-line fat-arrow callback | `(x) => { return x * 2 }` | Named top-level function | JavaScript training data — JS allows block bodies after `=>`; AHK v2 requires single expression |
| `Map.Get(key, unset)` as safe optional return | `return map.Get(k, unset)` | `if map.Has(k) { return map[k] }` | Mistaken belief that `unset` is a valid default sentinel — it removes the default argument entirely |
| Two-param meta-function v1 signatures | `__Get(name)` / `__Set(name, val)` | `__Get(name, params)` / `__Set(name, params, value)` | AHK v1 training data — v1 meta-functions used two parameters; the silent failure in v2 makes this hard to detect |
| `Get*` method mutating `this.*` | `GetStatus()` that sets `this.lastChecked := A_Now` | Split into `GetStatus()` (pure read) + `RecordCheckTime()` (mutation) | OOP training data co-locates reads and audit logging; the naming contract is absent from most training examples |
| `Check*` / `Is*` method that modifies state | `CheckSession()` that resets `this.hitCount := 0` | `IsSessionExpired()` (predicate) + `ResetSession()` (explicit mutation) | Command-query confusion is common in training data; "check and reset atomically" is a known anti-pattern that LLMs reproduce |

## SEE ALSO

> This module does NOT cover: prototype chain manipulation, `ObjSetBase()`, `__mro`, runtime prototype injection → see Module_ClassPrototyping.md
> This module does NOT cover: GUI control subclassing, `Gui()` constructor patterns, control event binding specifics → see Module_GUI.md
> This module does NOT cover: custom exception class design, `Error` subclassing for domain-specific errors → see Module_Errors.md
> This module does NOT cover: raw object creation with `{}`, property enumeration, `OwnProps()`, and own-vs-inherited property semantics → see Module_Objects.md

- `Module_ClassPrototyping.md` — direct prototype manipulation with `ObjSetBase()`, `__mro`, and runtime class graph surgery beyond what `extends` provides.
- `Module_Objects.md` — raw object creation with `{}`, property enumeration via `OwnProps()`, `GetOwnPropDesc()`, and the distinction between own properties and inherited prototype properties.
- `Module_GUI.md` — connecting class methods to GUI events, `Gui.AddButton().OnEvent()`, and managing GUI lifetimes inside class destructors.
- `Module_Errors.md` — defining custom exception hierarchies by subclassing `Error`, `ValueError`, `TypeError`, and structuring `try/catch/finally` inside class methods.
- `Module_Arrays.md` — Array methods (`.Push()`, `.Pop()`, `.RemoveAt()`, `.Clone()`) used extensively inside class state containers.