# Module_ClassPrototyping.md
<!-- DOMAIN: Class Prototyping -->
<!-- SCOPE: Static class definitions using the `class` keyword, standard `extends` inheritance, and __Item / __Enum meta-function patterns are not covered — see Module_Classes.md and Module_Objects.md. -->
<!-- TRIGGERS: DefineProp(), ObjSetBase(), HasProp(), HasMethod(), GetMethod(), GetOwnPropDesc(), DeleteProp(), OwnProps(), "property descriptor", "descriptor object", "dynamic getter", "readonly property", "intercept property assignment", "change method at runtime", "add property to instance", "runtime class generation", "method decorator", "class generator", "prototyping", "metaclass", "mixin" -->
<!-- CONSTRAINTS: Every Get/Set/Call descriptor function must accept the object instance as its first parameter — omitting it silently shifts arguments with no runtime error. DefineProp() returns the target object, not a boolean — it does not throw on success or failure; guard with HasProp()/HasMethod() before calling it to avoid silent overwrites. ObjSetBase() throws TypeError if base type is incompatible — verify type alignment before calling it. -->
<!-- CROSS-REF: Module_Classes.md, Module_Objects.md, Module_Errors.md -->
<!-- VERSION: AHK v2.0+ -->

## V1 → V2 BREAKING CHANGES

| v1 pattern (LLM commonly writes) | v2 correct form | Consequence |
|----------------------------------|-----------------|-------------|
| `new ClassName()` | `ClassName()` | `new` keyword does not exist in AHK v2 — runtime NameError |
| `{Get: MyFunc}` as descriptor literal | `d := Object()` then `d.Get := MyFunc` | Object literal descriptors are technically accepted by DefineProp, but Object() is the explicit, unambiguous convention for descriptor containers to prevent confusion with general data objects |
| `Class(BaseObj)` (v2.1-alpha) | Manual prototype chain via `ObjSetBase()` | v2.0 `Class()` produces an incomplete object with no `Call` method and is not a subclass of Object — unusable as a class factory in stable releases |
| Getter/Setter function with no leading parameter | `GetterFunc(Obj)` / `SetterFunc(Obj, Val)` | AHK v2 always injects the object instance as the first argument — omitting it silently shifts all arguments, causing wrong values or property-not-found errors |
| `(this) => { x := 1 \| return x }` multi-line fat-arrow | Named function block `MyFunc(this) { ... }` | AHK v2 fat-arrow functions are single-expression only — multi-line body is a syntax error |
| Modifying `Array.Prototype` or `String.Prototype` directly | Wrapper object or subclass with `extends` | Pollutes the global namespace; affects every Array/String in the script silently and irreversibly |
| Assuming `DefineProp()` throws on duplicate name | Guard with `HasProp()` / `HasMethod()` before calling | DefineProp silently overwrites any existing descriptor — there is no "already exists" exception |

## API QUICK-REFERENCE

### Object — Descriptor Construction and Prototype Manipulation

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `Object()` | `Object()` | Object | — | Creates a plain Object instance with no own properties — preferred form for descriptor containers; `{}` is equivalent in type but Object() makes the descriptor role explicit |
| `.DefineProp()` | `obj.DefineProp(Name, Descriptor)` | `obj` (target object) | — | Returns the target object for chaining; does NOT throw on duplicate — silently overwrites any existing descriptor |
| `.HasProp()` | `obj.HasProp(Name)` | Integer (1/0) | — | Returns 1 if the property exists on `obj` or any prototype in its chain; does not invoke the getter |
| `.HasMethod()` | `obj.HasMethod(Name, ParamCount?)` | Integer (1/0) | — | Returns 1 if a Call descriptor is resolvable; optional ParamCount validates MinParams/MaxParams compatibility |
| `.HasOwnProp()` | `obj.HasOwnProp(Name)` | Integer (1/0) | — | Returns 1 only for properties defined directly on `obj` — does not walk the prototype chain; also callable as `ObjHasOwnProp(obj, Name)` |
| `.DeleteProp()` | `obj.DeleteProp(Name)` | Any (removed value, or "" if none) | — | Removes an own property; returns the old value; no error if property is absent |
| `.GetOwnPropDesc()` | `obj.GetOwnPropDesc(Name)` | Object (descriptor) | PropertyError | Returns a new Object with Get/Set/Call/Value keys matching the stored descriptor; throws PropertyError if `obj` does not own the named property |
| `.OwnProps()` | `for Name, Value in obj.OwnProps()` | Enumerator | — | Enumerates own properties only; skips Call-only properties in two-variable mode; also callable as `ObjOwnProps(obj)` |
| `ObjSetBase()` | `ObjSetBase(Obj, BaseObj)` | — | TypeError | Sets `obj`'s prototype to `BaseObj`; bypasses meta-functions; throws TypeError if type would be changed (e.g., Array base to non-Array prototype) |

### Descriptor Object Keys

| Key | Expected value | Returns | Throws | Notes |
|-----|----------------|---------|--------|-------|
| `.Get` | `GetFunc(Obj)` | Any | — | Called when the property is read; first param is always the target object |
| `.Set` | `SetFunc(Obj, Value)` | — | Any exception | Called on assignment; second param is the assigned value; throw to reject invalid values |
| `.Call` | `CallFunc(Obj, Args*)` | Any | — | Called when the property is invoked as a method; first param is always the target object |
| `.Value` | Any AHK value | — | — | Simple static value — no function executes; cannot coexist with `.Get`/`.Set` on the same descriptor |

### Func — Binding and Introspection

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `.Bind()` | `FuncRef.Bind(BoundArgs*)` | BoundFunc | — | Returns a new BoundFunc with leading parameters pre-filled; one function object shared across all bound instances |
| `GetMethod()` | `GetMethod(Value, Name?, ParamCount?)` | Function Object | MethodError | Retrieves the implementation function of a method; throws MethodError if method not found or cannot be retrieved without invoking a getter |

### Type Inspection

| Function | Signature | Returns | Throws | Notes |
|----------|-----------|---------|--------|-------|
| `Type()` | `Type(Value)` | String | — | Returns `"Integer"`, `"String"`, `"Float"`, `"Object"`, `"Array"`, `"Map"`, etc.; use for type guards inside Set descriptors |
| `IsObject()` | `IsObject(Value)` | Integer (1/0) | — | Returns 1 for any object reference — preliminary guard before calling `.DefineProp()` or `.ObjSetBase()` |

## AHK V2 CONSTRAINTS

- **Every Get descriptor function must have exactly one parameter: the object instance** — `GetterFunc(Obj)`. AHK v2 injects the target object as the first argument unconditionally; a zero-parameter getter will receive the object and discard it, returning unpredictable results.
  - ✗ `GetterFunc()` with no parameters — object instance silently discarded; returns wrong value
  - ✓ `GetterFunc(Obj)` — receives the target object as its first argument

- **Every Set descriptor function must have exactly two parameters: the object instance and the assigned value** — `SetterFunc(Obj, Value)`. Omitting `Obj` shifts `Value` into the wrong slot with no runtime error.
  - ✗ `SetterFunc(AssignedValue)` — `AssignedValue` receives the object instance; intended value is lost
  - ✓ `SetterFunc(Obj, Value)` — `Obj` receives the instance, `Value` receives the assignment

- **Every Call descriptor function must accept the object instance as its first parameter** — `MethodFunc(Obj, Args*)`. The call `myObj.Prop(x)` internally becomes `Desc.Call(myObj, x)`.
  - ✗ `MethodFunc(x)` — `x` receives the object instance; intended argument is lost
  - ✓ `MethodFunc(Obj, Args*)` — `Obj` receives the instance; subsequent args are caller-supplied

- **`DefineProp()` does not throw on duplicate — it silently overwrites** — guard with `HasProp()` or `HasMethod()` before calling it when overwriting is not the intent.
  - ✗ `obj.DefineProp("Read", desc)` without a guard — destroys existing descriptor silently
  - ✓ `if !obj.HasMethod("Read") { obj.DefineProp("Read", desc) }` — safe non-destructive definition

- **Do not use `Class(BaseObj)` in v2.0 scripts** — it is a v2.1-alpha feature; the resulting object lacks a `.Call` method and cannot act as a class factory. Use manual prototype chain construction with `ObjSetBase()` instead.
  - ✗ `DynamicClass := Class(BaseObj)` — no `.Call` method; cannot be invoked as constructor in v2.0
  - ✓ Manually construct prototype chain with `Object()` + `ObjSetBase()` + a `Call` descriptor

- **Do not modify built-in prototypes (`Array.Prototype`, `Map.Prototype`, etc.) unless explicitly required** — changes propagate globally to every instance of that type in the script, creating invisible coupling and name collision risk with no way to undo.
  - ✗ `Array.Prototype.DefineProp("Sum", d)` — pollutes every Array globally, irreversibly
  - ✓ Subclass with `class MyArray extends Array { }` or use standalone utility functions

- **Fat-arrow functions assigned to descriptors must be single-expression** — `(Obj) => Obj.Name` is valid; `(Obj) => { x := 1 | return x }` is a syntax error. Use a named function block for multi-statement descriptor logic.
  - ✗ `(Obj) => { x := 1 | return x }` — syntax error; multi-line fat-arrow is not valid in AHK v2
  - ✓ Named function block `MyFunc(Obj) { x := 1 | return x }` — full function body supported

- **`GetOwnPropDesc()` throws PropertyError if the property is not an own property of `obj`** — do not call it on inherited properties or non-existent names without a `HasOwnProp()` guard.
  - ✗ `obj.GetOwnPropDesc("InheritedProp")` — PropertyError even if `HasProp()` returns true
  - ✓ `if obj.HasOwnProp("Read") { desc := obj.GetOwnPropDesc("Read") }` — safe inspection

Safe-access priority order for prototype manipulation:
  1. `.HasOwnProp(Name)` — when you need to distinguish own from inherited; required before `GetOwnPropDesc()`
  2. `.HasProp(Name)` — when chain-level existence is sufficient; avoids UnsetError on absent properties
  3. `.HasMethod(Name)` — call-specific guard; confirms a Call descriptor exists before invoking or decorating
  4. `IsObject(Target)` — preliminary guard before calling any Object method; prevents MethodError on non-objects
  5. `try/catch` — only when the DefineProp or ObjSetBase call itself may fail and the exception message carries diagnostic information

Unset variable handling: `DefineProp()` does not throw — the only unset risk is reading a dynamic property before the descriptor function assigns a return value. Use `HasProp()` before reading any property whose descriptor path is conditional.

Resource lifecycle: descriptor closures capture outer variable references and keep those references alive for as long as the descriptor is attached to the object. `DeleteProp()` releases the descriptor and drops its closure reference — call it when objects with closure-backed descriptors are being recycled in high-volume loops.

## AGENT QA CHECKLIST

- [ ] Did I add the object instance as the first parameter in every Get, Set, and Call descriptor function?
- [ ] Did I guard with `HasMethod()` or `HasProp()` before calling `DefineProp()` to prevent silent descriptor overwrites?
- [ ] Am I using `ObjSetBase()` instead of `Class(BaseObj)` for dynamic prototype chain construction in AHK v2.0 stable?
- [ ] Are all fat-arrow functions assigned to descriptors single-expression only — no multi-line `{ }` body?

## RUNTIME ERROR MAPPING

| Exception Class | Trigger Condition | Detection Code | Fix |
|----------------|-------------------|----------------|-----|
| `MethodError` | `GetMethod(obj, "Name")` when the method does not exist on `obj` or any prototype, or when retrieval requires invoking a getter | `e.Message` contains the method name; `e.What` is `"GetMethod"` | Guard with `obj.HasMethod("Name")` before calling `GetMethod()`; note that HasMethod is true if a getter returns a callable, but GetMethod cannot retrieve it — use `try/catch` in that case |
| `PropertyError` | `obj.GetOwnPropDesc("Name")` when `obj` does not own the named property (even if it is inherited) | `e.Message` contains the property name | Check `obj.HasOwnProp("Name")` first; use `obj.HasProp("Name")` if chain-level existence is sufficient |
| `TypeError` | `ObjSetBase(obj, baseObj)` when assigning the new base would change the native type of `obj` (e.g., setting an Array instance's base to a non-Array prototype object) | `e.Message` mentions type incompatibility | Ensure `baseObj` derives from the same built-in prototype as `obj`; use `baseObj.HasBase(Array.Prototype)` to verify before calling ObjSetBase on Array instances |

## TIER 1 — Descriptor Object Fundamentals: Dynamic Call Assignment
> METHODS COVERED: Object() · DefineProp() · .Call (descriptor key)

A Call descriptor turns any function into a method on a specific object instance. The descriptor must be a raw `Object()` — not an object literal — and the function it holds must accept the target object as its first parameter. This tier covers the minimal DefineProp pattern: create descriptor, assign function, attach to instance.

```ahk
CustomMethod(Obj, Prefix) {
    return Prefix . ": " . Obj.Name
}

; Create the object
MyObj := Object()
MyObj.Name := "Alpha"

; ✓ Object() produces a plain Object instance — the preferred form for descriptor containers
PropDesc := Object()
PropDesc.Call := CustomMethod

; ✓ DefineProp attaches the descriptor; MyObj.Greet now resolves as a callable method
MyObj.DefineProp("Greet", PropDesc)

; ✓ Implicitly calls CustomMethod(MyObj, "Status") — AHK injects MyObj as first argument
Result := MyObj.Greet("Status")   ; Result = "Status: Alpha"

; ✗ Avoid {} as the descriptor container — use Object() for explicit descriptor construction
; PropDesc := {Call: CustomMethod}    ; → avoided by convention; technically works but use Object()

; ✗ Calling DefineProp again overwrites without error — guard required to prevent silent loss
; MyObj.DefineProp("Greet", PropDesc)  ; → silently destroys prior descriptor with no exception
```

## TIER 2 — Dynamic Getters and Setters: Computed and Validated Properties
> METHODS COVERED: Object() · DefineProp() · .Get (descriptor key) · .Set (descriptor key) · Type()

Get descriptors execute when a property is read; Set descriptors execute on assignment. Together they create computed, validated, or read-only property behaviour without storing the logic-driven value directly on the object. Both functions must accept the object instance as their first parameter.

```ahk
GetterFunc(Obj) {
    ; ✓ Getter receives Obj as first param — computes and returns value without storing it on Obj
    return Obj._HiddenValue * 2
}

SetterFunc(Obj, AssignedValue) {
    ; ✓ Type() returns a string — != is the correct AHK v2 not-equal operator
    if (Type(AssignedValue) != "Integer") {
        throw TypeError("Only integers allowed.")
    }
    Obj._HiddenValue := AssignedValue
}

MyData := Object()
MyData._HiddenValue := 10

; ✓ Separate descriptor keys for Get and Set on the same property
PropDesc := Object()
PropDesc.Get := GetterFunc
PropDesc.Set := SetterFunc

MyData.DefineProp("Doubled", PropDesc)

; ✓ Triggers GetterFunc(MyData) — returns _HiddenValue * 2 = 20
CurrentVal := MyData.Doubled

; ✓ Triggers SetterFunc(MyData, 50) — validates then stores in _HiddenValue
MyData.Doubled := 50

; ✗ Assigning a non-integer raises TypeError via SetterFunc — expected behaviour
; MyData.Doubled := "String"          ; → TypeError("Only integers allowed.")

; ✗ Omitting the Obj parameter shifts AssignedValue into the wrong slot
; SetterFunc(AssignedValue) { ... }   ; → AssignedValue receives the object instance silently
```

## TIER 3 — Existence Guards: Safe Property and Method Validation
> METHODS COVERED: HasProp() · HasMethod() · HasOwnProp() · GetOwnPropDesc() · GetMethod() · IsObject() · DefineProp() · Object()

Before mutating a prototype or redefining a property, always validate that the target is an object and that the property does not already exist. `HasOwnProp` distinguishes own from inherited; `GetOwnPropDesc` inspects descriptor details before redefining; `GetMethod` retrieves the callable function object for introspection or delegation.

```ahk
SafeDefineMethod(TargetObj, MethodName, FuncRef) {
    ; ✓ IsObject() guards against non-object inputs before any property access
    if !IsObject(TargetObj) {
        throw TypeError("Target must be an object.")
    }

    ; ✓ HasMethod() checks for an existing Call descriptor before overwriting
    if TargetObj.HasMethod(MethodName) {
        return false
    }

    Desc := Object()
    Desc.Call := FuncRef
    TargetObj.DefineProp(MethodName, Desc)
    return true
}

InspectOwnDescriptor(Obj, PropName) {
    ; ✓ HasOwnProp() — distinguishes own from inherited; required before GetOwnPropDesc()
    if !Obj.HasOwnProp(PropName) {
        return "not an own property"   ; HasProp() might be true for inherited — HasOwnProp() is stricter
    }

    ; ✓ GetOwnPropDesc() returns a descriptor Object with Get/Set/Call/Value keys present
    Desc := Obj.GetOwnPropDesc(PropName)

    ; ✓ Determine descriptor kind by inspecting which keys are present
    if Desc.HasProp("Value") {
        return "value property: " . Desc.Value
    }
    if Desc.HasProp("Get") && Desc.HasProp("Set") {
        return "read-write dynamic property"
    }
    if Desc.HasProp("Get") {
        return "read-only dynamic property"
    }
    if Desc.HasProp("Call") {
        return "method descriptor"
    }
    return "unknown descriptor"
}

RetrieveMethodFunc(Obj, MethodName) {
    ; ✓ HasMethod() before GetMethod() — GetMethod throws MethodError if method absent
    if !Obj.HasMethod(MethodName) {
        return ""
    }
    ; ✓ GetMethod() returns the raw function object — useful for delegation or introspection
    return GetMethod(Obj, MethodName)
}

SayHello(Obj) {
    return "Hello from " . (Obj.HasProp("Name") ? Obj.Name : "unknown")
}

TestObj := Object()
TestObj.Name := "Beta"

; ✓ Returns true and defines "Speak" — method was absent
Success := SafeDefineMethod(TestObj, "Speak", SayHello)

; ✓ Returns false without overwriting — method is now present
Success2 := SafeDefineMethod(TestObj, "Speak", SayHello)

; ✓ Retrieves descriptor kind for the value property "Name"
Kind := InspectOwnDescriptor(TestObj, "Name")   ; "value property: Beta"

; ✗ Calling GetOwnPropDesc on an inherited property throws PropertyError
; Desc := TestObj.GetOwnPropDesc("Speak")        ; → PropertyError if Speak is only in prototype

; ✗ Calling DefineProp without HasMethod guard silently overwrites existing methods
; TargetObj.DefineProp(MethodName, Desc)  ; → destroys existing Call descriptor with no warning
```

## TIER 4 — Closure Factories: Context-Aware Properties Without State Pollution
> METHODS COVERED: Object() · DefineProp() · .Get (descriptor key)

Closures let a descriptor function capture outer variables at definition time, keeping captured state isolated from the object instance. The factory function returns an inner function that closes over the factory's parameters. This tier demonstrates injecting read-only context (such as a version string) into a property without storing it as a visible field on the object.

```ahk
; ✓ Factory returns a closure — the inner function captures ReturnValue at call time
CreateConstantGetter(ReturnValue) {
    return ConstantGetter

    ; ✓ Inner function closes over ReturnValue; Obj is the mandatory first param
    ConstantGetter(Obj) {
        return ReturnValue
    }
}

ApplyVersion(TargetObj, VersionStr) {
    Desc := Object()
    ; ✓ Desc.Get holds the closure — each call to ApplyVersion captures its own VersionStr
    Desc.Get := CreateConstantGetter(VersionStr)
    TargetObj.DefineProp("Version", Desc)
}

AppObj := Object()
ApplyVersion(AppObj, "1.0.4")

; ✓ Triggers ConstantGetter(AppObj) — returns "1.0.4" without AppObj._Version existing
CurrentVer := AppObj.Version   ; "1.0.4"

; ✗ Storing state directly on the instance leaks implementation details to callers
; AppObj._Version := "1.0.4"          ; → caller can read and modify _Version directly
```

## TIER 5 — Method Decorators: Non-Destructive Interception and Augmentation
> METHODS COVERED: HasProp() · HasMethod() · GetMethod() · DefineProp() · Object() · .Call (descriptor key) · .Bind()

Decorators wrap an existing function in a new closure that injects pre- or post-execution logic (logging, timing, access control) without touching the original function's source. The decorator returns a new function that calls the original internally, preserving its semantics while extending its behaviour. `GetMethod()` retrieves the original callable before wrapping, enabling fully transparent delegation.

```ahk
WithCallCounter(OriginalMethod) {
    return LoggedMethod

    ; ✓ LoggedMethod captures OriginalMethod from the outer scope via closure
    LoggedMethod(Obj, Args*) {
        ; ✓ HasProp() guard prevents overwriting an existing counter
        if !Obj.HasProp("CallCount") {
            Obj.CallCount := 0
        }

        ; Side-effect: increment counter on every invocation
        Obj.CallCount++

        ; ✓ Forwards all arguments to OriginalMethod — decorator is fully transparent
        return OriginalMethod(Obj, Args*)
    }
}

; ✓ DecorateMethod wraps an existing method non-destructively using GetMethod + DefineProp
DecorateMethod(Obj, MethodName, DecoratorFunc) {
    ; ✓ HasMethod() guard — only decorate if the method exists
    if !Obj.HasMethod(MethodName) {
        throw Error("DecorateMethod: method '" MethodName "' not found on target.")
    }

    ; ✓ GetMethod() retrieves the original callable before overwriting the descriptor
    Original := GetMethod(Obj, MethodName)

    Desc := Object()
    ; ✓ Bind(Original) passes the original function as first arg to the decorator
    Desc.Call := DecoratorFunc.Bind(Original)
    Obj.DefineProp(MethodName, Desc)
}

BaseAction(Obj, Val) {
    return Val * 10
}

Worker := Object()
Desc := Object()
; ✓ Wraps BaseAction — Worker.Process now counts calls without modifying BaseAction
Desc.Call := WithCallCounter(BaseAction)
Worker.DefineProp("Process", Desc)

Worker.Process(5)
Worker.Process(10)
; ✓ Worker.CallCount is now 2

; ✗ Modifying BaseAction directly to add logging destroys the original implementation
; BaseAction(Obj, Val) { Obj.CallCount++ | return Val * 10 }  ; → not reversible, mixes concerns
```

## TIER 6 — Runtime Class Generation: v2.0-Compatible Dynamic Class Factory
> METHODS COVERED: Object() · ObjSetBase() · DefineProp() · HasMethod() · DeleteProp() · OwnProps() · .Call (descriptor key)

Because `Class()` in AHK v2.0 produces an incomplete object with no `Call` method and is not a subclass of Object, a fully functional dynamic class requires manual prototype chain construction. A plain `Object()` acts as the class object; a separate `Object()` acts as its prototype. A custom `Call` descriptor on the class object handles instantiation — replicating the behaviour AHK normally performs when a compiled class is invoked.

```ahk
; ✓ v2.0-compatible runtime class factory — no alpha features required
CreateDynamicClass(ClassName) {
    DynamicClass := Object()
    DynamicClass.Prototype := Object()
    DynamicClass.Prototype.__Class := ClassName

    ; ✓ Call descriptor makes DynamicClass() callable as a constructor
    ConstructorDesc := Object()
    ConstructorDesc.Call := DynamicClass_Call
    DynamicClass.DefineProp("Call", ConstructorDesc)

    return DynamicClass
}

; ✓ Replicates AHK's standard instance creation workflow
DynamicClass_Call(Cls, Args*) {
    Instance := Object()
    ; ✓ ObjSetBase links the instance to the class prototype — enables method lookup
    ObjSetBase(Instance, Cls.Prototype)

    ; ✓ HasMethod() guards — only call if defined; avoids MethodError on bare instances
    if Instance.HasMethod("__Init") {
        Instance.__Init()
    }
    if Instance.HasMethod("__New") {
        Instance.__New(Args*)
    }
    return Instance
}

; Usage: create a new type at runtime
MyDynamicType := CreateDynamicClass("RuntimeEntity")

; ✓ Attach a method to the shared prototype — all instances inherit it
MethodDesc := Object()
MethodDesc.Call := (this) => "I am a " . this.__Class
MyDynamicType.Prototype.DefineProp("Identify", MethodDesc)

; ✓ Instantiate via the Call descriptor — no 'new' keyword
EntityInst := MyDynamicType()
EntityClass := EntityInst.__Class    ; "RuntimeEntity"
EntityId    := EntityInst.Identify() ; "I am a RuntimeEntity"

; ✓ OwnProps enumerates own properties — useful for inspecting what is attached to a prototype
for PropName in MyDynamicType.Prototype.OwnProps() {
    ; PropName iterates: "__Class", "Identify"
}

; ✓ DeleteProp removes a descriptor from the prototype — cleans up when recycling dynamic types
MyDynamicType.Prototype.DeleteProp("Identify")

; ✗ Class(BaseObj) is a v2.1-alpha feature — produces unusable object in v2.0 stable
; DynamicClass := Class(BaseObj)      ; → no .Call method; cannot be invoked as constructor

; ✗ 'new' keyword does not exist in AHK v2
; EntityInst := new MyDynamicType()   ; → runtime NameError
```

### Performance Notes

`DefineProp` carries a one-time overhead at the moment of assignment — not per access. Once attached, property lookup through a descriptor follows the normal prototype chain with no additional cost beyond the function call itself.

**Closure memory cost:** Every closure object allocates a separate activation record capturing its outer variables. Creating a new closure inside a per-instance factory (e.g., inside a loop over thousands of objects) multiplies this allocation. Prefer defining the function statically and binding context via `.Bind()` instead:

```ahk
; ✗ New closure per instance — high memory footprint when called thousands of times
AttachSlowMethod(Obj, Prefix) {
    Desc := Object()
    ; Each call to AttachSlowMethod allocates a new anonymous function object
    Desc.Call := (this) => Prefix . this.Name
    Obj.DefineProp("Read", Desc)
}

; ✓ Static function + Bind — one function object shared; only the BoundFunc is allocated per instance
StaticReader(Prefix, this) {
    return Prefix . this.Name
}

AttachFastMethod(Obj, Prefix) {
    Desc := Object()
    ; ✓ Bind(Prefix) produces a lightweight BoundFunc; StaticReader itself is allocated once
    Desc.Call := StaticReader.Bind(Prefix)
    Obj.DefineProp("Read", Desc)
}
```

**Prototype chain depth:** Each property lookup walks the prototype chain from instance to base. Dynamically constructed chains that are 4+ levels deep degrade lookup to O(N) where N is chain depth. Keep runtime-generated hierarchies shallow; prefer composition (attaching methods to a single shared prototype) over deep inheritance.

**`DeleteProp` for lifecycle cleanup:** Closure-backed descriptors hold strong references to their captured variables. In high-volume object pools, call `DeleteProp()` explicitly before recycling an object rather than relying on garbage collection to release captured closure memory.

**Built-in method preference:** AHK v2's native `HasProp`, `HasMethod`, `HasOwnProp`, and `GetMethod` are implemented in native code and are faster than any custom existence-check loop. Always prefer them over manual property enumeration.

## DROP-IN RECIPES

```ahk
; SafeDefineProp — robust DefineProp with full validation, non-destructive by default
; ✓ Validates target is an object, verifies descriptor has at least one recognised key,
;   and refuses to overwrite an existing property unless Force := true is passed
SafeDefineProp(Obj, PropName, DescObj, Force := false) {
    if !(PropName is String) || PropName = "" {
        throw TypeError("SafeDefineProp: PropName must be a non-empty string", -1)
    }
    if !IsObject(Obj) {
        throw TypeError("SafeDefineProp: Obj must be an object", -1)
    }
    if !IsObject(DescObj) {
        throw TypeError("SafeDefineProp: DescObj must be an Object() descriptor", -1)
    }
    ; ✓ Verify descriptor carries at least one recognised key before attaching
    if !DescObj.HasProp("Get") && !DescObj.HasProp("Set")
    && !DescObj.HasProp("Call") && !DescObj.HasProp("Value") {
        throw ValueError("SafeDefineProp: descriptor must have Get, Set, Call, or Value key", -1)
    }
    ; ✓ Guard against silent overwrite unless caller explicitly opts in
    if !Force && Obj.HasProp(PropName) {
        throw Error("SafeDefineProp: property '" PropName "' already exists; pass Force:=true to overwrite", -1)
    }
    Obj.DefineProp(PropName, DescObj)
    return Obj
}
; Call site: SafeDefineProp(MyObj, "Status", d)  or  SafeDefineProp(MyObj, "Status", d, true)


; CopyOwnDescriptors — copies all own descriptors from one object/prototype to another
; ✓ Uses GetOwnPropDesc + DefineProp to perform a faithful descriptor-level copy;
;   optionally skips names in the ExcludeList array
CopyOwnDescriptors(FromObj, ToObj, ExcludeList := []) {
    if !IsObject(FromObj) || !IsObject(ToObj) {
        throw TypeError("CopyOwnDescriptors: both arguments must be objects", -1)
    }
    Excluded := Map()
    for name in ExcludeList {
        Excluded[name] := true
    }
    ; ✓ OwnProps single-variable mode — iterates names without invoking getters
    for PropName in FromObj.OwnProps() {
        if Excluded.Has(PropName) {
            continue
        }
        ; ✓ GetOwnPropDesc returns a copy of the descriptor — safe to attach directly
        Desc := FromObj.GetOwnPropDesc(PropName)
        ToObj.DefineProp(PropName, Desc)
    }
    return ToObj
}
; Call site: CopyOwnDescriptors(SourceProto, TargetProto, ["__Class"])
```

## ANTI-PATTERNS

| Pattern | Wrong | Correct | LLM Common Cause |
|---------|-------|---------|------------------|
| Object literal as descriptor | `DefineProp("X", {Get: MyFunc})` | `d := Object()` then `d.Get := MyFunc` then `DefineProp("X", d)` | AHK v1 and general OOP training data treat `{}` as universal object constructor; Object() is the explicit convention for descriptor containers in this module series, even though both are accepted by DefineProp |
| Missing `this` parameter in descriptor function | `GetterFunc()` with no params | `GetterFunc(Obj)` where `Obj` is the injected instance | Cross-language habit — Python/JS descriptors use implicit self/this binding; AHK v2 passes it explicitly as the first positional argument |
| Using `Class(BaseObj)` in stable scripts | `DynamicClass := Class(BaseObj)` | Manual prototype chain with `ObjSetBase()` | LLM training data includes v2.1-alpha documentation; model cannot distinguish alpha from stable feature boundaries |
| Multi-line fat-arrow in descriptor | `(Obj) => { x := 1 \| return x }` | Named function block `MyFunc(Obj) { x := 1 \| return x }` | JavaScript and Python both support multi-line lambda/arrow bodies; AHK v2 fat-arrow is single-expression only |
| Assuming `DefineProp` is safe without a guard | `obj.DefineProp(name, desc)` without `HasMethod`/`HasProp` check | `if !obj.HasMethod(name) { obj.DefineProp(name, desc) }` | LLMs trained on JS `Object.defineProperty` assume an options flag (configurable, writable) prevents silent overwrite; AHK v2 has no such flag |
| Using `new` keyword | `new ClassName()` | `ClassName()` | AHK v1 required `new`; dominant OOP training data (Java, C#, JS) reinforces the keyword |
| Calling `GetOwnPropDesc` on inherited property | `obj.GetOwnPropDesc("InheritedMethod")` | `if obj.HasOwnProp(name) { desc := obj.GetOwnPropDesc(name) }` | LLMs conflate `HasProp` (chain) with `HasOwnProp` (own-only); JS `Object.getOwnPropertyDescriptor` also silently returns undefined for inherited names, giving no training signal for the AHK exception |
| Modifying built-in prototypes | `Array.Prototype.DefineProp("Sum", d)` | Wrapper class or standalone function | LLM training data includes JavaScript prototype extension patterns; AHK v2 built-in pollution is global and irreversible |

## SEE ALSO

> This module does NOT cover: static `class` definitions, `extends` inheritance, `super` dispatch, or `__Item` / `__Enum` meta-function patterns → see Module_Classes.md
> This module does NOT cover: base Object API, `ObjGetBase()` traversal, or raw prototype chain inspection → see Module_Objects.md
> This module does NOT cover: try/catch patterns for DefineProp failures, TypeError and ValueError construction, or custom error class hierarchies → see Module_Errors.md

- `Module_Classes.md` — static class definitions, `extends` inheritance, meta-functions (`__New`, `__Delete`, `__Get`, `__Set`), and `super` dispatch.
- `Module_Objects.md` — base Object API (`ObjGetBase`, `ObjSetBase`, `ObjOwnProps`), prototype chain traversal, and raw object construction patterns.
- `Module_Errors.md` — try/catch patterns for runtime DefineProp failures, TypeError and ValueError construction, and custom exception class hierarchies.