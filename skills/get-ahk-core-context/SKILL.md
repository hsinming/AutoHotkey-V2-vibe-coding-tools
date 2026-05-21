---
name: get-ahk-core-context
description: >
  Use when writing, modifying, or reviewing any AutoHotkey v2 code. Triggers:
  AHK, AutoHotkey, .ahk files, hotkeys, scripts, classes, arrays, maps, objects,
  arrow functions, closures, dynamic properties, OOP, data structures, functions,
  parameters — even casual requests like "write me an AHK script" or "fix this
  AHK code".
modeSlugs:
  - ahk-orchestrator
  - ahk-architect
  - ahk-code
  - ahk-debug
  - ahk-ask
---

# AHK v2 Core Context

## Overview

This skill provides reference modules for AHK v2 core syntax, OOP patterns,
data structures, dynamic properties, and code formatting standards.
`Module_Instructions` is the foundational module — load it before every AHK task.

---

## How to Use This Skill

### Step 1 — Map the task to targeted sections via Section Navigator

Do **not** load entire Module files. Use the **Section Navigator** below to find
the exact heading to grep and the number of lines to read. The `## API QUICK-REFERENCE`
and `## AHK V2 CONSTRAINTS` sections cover most tasks; descend into TIER sections
only when you need a runnable code example.

**Exception:** `Module_Instructions.md` and `ahk_formatting_spec.md` have no navigable
headings — load them in full (431 and 188 lines respectively). Always load both first.

### Step 2 — Cross-skill loading order

Load in this sequence; conditional modules are marked with their trigger condition:

1. `references/Module_Instructions.md` — always first (full file, 431 lines)
2. `references/ahk_formatting_spec.md` — always second (full file, 188 lines)
3. Targeted sections from further modules as identified by the Section Navigator

For cross-skill references (modules owned by sibling skills), use:
- UI / window / control ops → `../get-ahk-ui-context/references/Module_GUI.md`
- Logic / async / error ops → `../get-ahk-logic-context/references/Module_Errors.md`
- System / DLL / COM ops → `../get-ahk-system-context/references/Module_DllCallAndMemory.md`

### Step 3 — Apply Universal Critical Rules

Before writing any code, confirm each rule in the checklist below applies to the
current task. Treat any violation as a blocking error.

### Step 4 — Generate code

Write code guided by the TIER system in each module. Match code complexity to the
appropriate tier; do not introduce patterns above the tier required by the task.

### Step 5 — Pre-output self-check

- [ ] No `new ClassName()` — instantiation is `ClassName()` without `new`
- [ ] No `{key: value}` object literals used as data containers — use `Map("key", value)`
- [ ] Every class method callback uses `.Bind(this)` — bare method references lose `this` context
- [ ] Arrays indexed from `1`, not `0` — `arr[1]` is the first element
- [ ] No space between function name and parenthesis — `Fn(x)` not `Fn (x)`
- [ ] 4-space indentation, CRLF line endings, opening brace on same line (K&R / OTB)
- [ ] Resource cleanup (`FileClose`, `CallbackFree`, COM teardown) is in `__Delete()` or a `finally` block

---

## Section Navigator

> **How to use:**
> 1. Find the row matching your task below.
> 2. `grep -n "^## heading text"` the target file to get the current line number.
> 3. `Read(file, offset=<line>, limit=<n>)` — read only that section.
>
> `## API QUICK-REFERENCE` + `## AHK V2 CONSTRAINTS` cover ~80 % of tasks.
> Load a TIER section only when you need a complete, runnable code example.

---

### Functions (Module_Functions.md — 864 lines)

| Task | Grep for heading | ~Lines |
|------|-----------------|--------|
| All function API: parameter syntax, Func object, IsSet | `## API QUICK-REFERENCE` | 75 |
| Parameter modifier keywords (`&`, `*`, default values) | `### Parameter Modifier Syntax` | 9 |
| Parameter ordering rules (required → optional → variadic) | `### Parameter Order Rule` | 11 |
| `local` / `global` / `static` scope keywords | `### Scope Declaration Keywords` | 8 |
| `IsSet()` / `IsSetRef()` and unset parameter checking | `### IsSet / IsSetRef` | 7 |
| `Func` object methods: `.Bind()`, `.Call()`, `.MaxParams` | `### Func Object — Methods` | 9 |
| `Func` object properties: `.MinParams`, `.MaxParams`, `.IsBuiltIn` | `### Func Object — Properties` | 10 |
| `Func()` constructor + factory helpers | `### Func Constructor` | 6 |
| Built-in utility functions referenced in examples | `### Built-in Utility Functions Referenced in Examples` | 13 |
| v1 → v2 breaking changes (functions) | `## V1 → V2 BREAKING CHANGES` | 13 |
| Constraints: scope traps, assignment vs comparison | `## AHK V2 CONSTRAINTS` | 43 |
| QA checklist before submitting code | `## AGENT QA CHECKLIST` | 7 |
| Runtime error → probable cause mapping | `## RUNTIME ERROR MAPPING` | 8 |
| **Working example** — basic declaration and calling | `## TIER 1 — Basic Declaration and Calling` | 85 |
| **Working example** — local, global, static scope | `## TIER 2 — Variable Scope: Local, Global, Static` | 66 |
| **Working example** — default, unset, variadic params | `## TIER 3 — Parameter Modifiers: Default, Unset, Variadic` | 77 |
| **Working example** — ByRef output parameters | `## TIER 4 — ByRef Parameters: Output Params and In-Place Mutation` | 83 |
| **Working example** — lexical closures + nested functions | `## TIER 5 — Nested Functions: Lexical Scope, Closures, and Func Objects` | 190 |
| **Working example** — multi-return + pure/side-effect separation | `## TIER 6 — Multi-Return and Pure vs Side-Effect Separation` | 112 |
| Copy-paste working snippets | `## DROP-IN RECIPES` | 71 |
| Anti-patterns to avoid | `## ANTI-PATTERNS` | 15 |

---

### Objects & Property Descriptors (Module_Objects.md — 663 lines)

| Task | Grep for heading | ~Lines |
|------|-----------------|--------|
| All object API: Any root, Object methods, Func/BoundFunc, type introspection | `## API QUICK-REFERENCE` | 47 |
| `Any` root class methods (`HasProp`, `HasMethod`, `GetOwnPropDesc`) | `### Any (root — every AHK v2 value inherits these)` | 10 |
| `Object` instance methods (`DefineProp`, `OwnProps`, `DeleteProp`) | `### Object (instance methods — available on plain objects and class instances)` | 11 |
| `Func` / `BoundFunc` API | `### Func / BoundFunc` | 7 |
| `Type()`, `IsObject()`, `instanceof` type introspection | `### Type Introspection` | 7 |
| Supporting functions used in examples | `### Supporting Functions (referenced in examples — defined in other modules)` | 10 |
| v1 → v2 breaking changes (objects) | `## V1 → V2 BREAKING CHANGES` | 13 |
| Constraints: DefineProp pitfalls, prototype chain rules | `## AHK V2 CONSTRAINTS` | 42 |
| QA checklist before submitting code | `## AGENT QA CHECKLIST` | 8 |
| Runtime error → probable cause mapping | `## RUNTIME ERROR MAPPING` | 8 |
| **Working example** — object fundamentals + Any root | `## TIER 1 — Object Fundamentals, Creation, and the Any Root` | 50 |
| **Working example** — property descriptors: get/set/call + DefineProp | `## TIER 2 — Property Descriptors: get, set, call, and DefineProp` | 107 |
| **Working example** — inheritance + prototype extension | `## TIER 3 — Class Inheritance and Prototype Extension` | 66 |
| **Working example** — BoundFunc + callback context | `## TIER 4 — BoundFunc and Callback Context Management` | 65 |
| **Working example** — dynamic properties + object composition | `## TIER 5 — Dynamic Properties and Object Composition` | 104 |
| **Working example** — validated objects + config patterns | `## TIER 6 — Validated Objects and Config Patterns` | 64 |
| Copy-paste working snippets | `## DROP-IN RECIPES` | 49 |
| Anti-patterns to avoid | `## ANTI-PATTERNS` | 13 |

---

### Classes (Module_Classes.md — 1047 lines)

| Task | Grep for heading | ~Lines |
|------|-----------------|--------|
| All class API: meta-functions, instance methods, Map for state | `## API QUICK-REFERENCE` | 49 |
| `__New`, `__Delete`, `__Call`, `__Get`, `__Set` signatures | `### Class Meta-Functions` | 12 |
| `OwnProps()`, `HasOwnProp()`, `DeleteProp()` | `### Instance and Object Methods` | 14 |
| Object utility functions (`Clone`, `DefineProp`, …) | `### Object Utility Functions` | 10 |
| Map as class state container (preferred pattern) | `### Map (preferred class state container)` | 11 |
| v1 → v2 breaking changes (classes) | `## V1 → V2 BREAKING CHANGES` | 12 |
| Constraints: `this` in callbacks, `new` removed, static vs instance | `## AHK V2 CONSTRAINTS` | 45 |
| QA checklist before submitting code | `## AGENT QA CHECKLIST` | 8 |
| Runtime error → probable cause mapping | `## RUNTIME ERROR MAPPING` | 8 |
| **Working example** — class fundamentals: constructor, properties, static | `## TIER 1 — Class Fundamentals: Constructors, Properties, and Static Members` | 134 |
| **Working example** — inheritance + polymorphism | `## TIER 2 — Inheritance and Polymorphism` | 88 |
| **Working example** — meta-functions + fluent interfaces | `## TIER 3 — Meta-Functions and Fluent Interfaces` | 164 |
| **Working example** — nested classes + factory patterns | `## TIER 4 — Nested Classes and Factory Patterns` | 109 |
| **Working example** — resource management + lifecycle | `## TIER 5 — Resource Management and Lifecycle` | 113 |
| **Working example** — observer pattern + weak references | `## TIER 6 — Observer Pattern and Weak References` | 167 |
| Copy-paste working snippets | `## DROP-IN RECIPES` | 101 |
| Anti-patterns to avoid | `## ANTI-PATTERNS` | 15 |

---

### Arrays (Module_Arrays.md — 675 lines)

| Task | Grep for heading | ~Lines |
|------|-----------------|--------|
| All array API: built-in methods, utility functions | `## API QUICK-REFERENCE` | 58 |
| Built-in Array methods (`Push`, `Pop`, `RemoveAt`, `Length`, …) | `### Array (built-in)` | 20 |
| Global functions used with arrays (`Sort`, `Join`, …) | `### Global Functions Used with Arrays` | 9 |
| Module utility functions (Map, Filter, Reduce, etc.) | `### Module Utility Functions (defined in this module)` | 27 |
| v1 → v2 breaking changes (arrays) | `## V1 → V2 BREAKING CHANGES` | 12 |
| Constraints: 1-based indexing, iteration safety | `## AHK V2 CONSTRAINTS` | 32 |
| QA checklist before submitting code | `## AGENT QA CHECKLIST` | 7 |
| Runtime error → probable cause mapping | `## RUNTIME ERROR MAPPING` | 8 |
| **Working example** — creation, access, type verification | `## TIER 1 — Fundamentals: Creation, Access, and Type Verification` | 61 |
| **Working example** — mutation: add, remove, clear | `## TIER 2 — Mutation: Add, Remove, and Clear` | 66 |
| **Working example** — search, predicates, type guards | `## TIER 3 — Search, Predicates, and Type Guards` | 49 |
| **Working example** — Map, Filter, Reduce transformations | `## TIER 4 — Transformations: Clone, DeepClone, Map, Filter, Reduce` | 79 |
| **Working example** — sorting + deduplication | `## TIER 5 — Sorting, Deduplication, and Performance Patterns` | 110 |
| **Working example** — set operations (diff, intersect, union) | `## TIER 6 — Set Operations: Difference, Intersection, Union, Without` | 90 |
| Copy-paste working snippets | `## DROP-IN RECIPES` | 60 |
| Anti-patterns to avoid | `## ANTI-PATTERNS` | 12 |

---

### Data Structures — Map vs Array (Module_DataStructures.md — 650 lines)

| Task | Grep for heading | ~Lines |
|------|-----------------|--------|
| Array API summary | `### Array` | 16 |
| Map API summary (`Has`, `Get`, `Set`, `Delete`, CaseSense, `__Enum`) | `### Map` | 15 |
| Map prototype extension methods | `### Prototype Extension (Map)` | 6 |
| v1 → v2 breaking changes (data structures) | `## V1 → V2 BREAKING CHANGES` | 12 |
| Constraints: when to use Map vs Object vs Array | `## AHK V2 CONSTRAINTS` | 33 |
| QA checklist before submitting code | `## AGENT QA CHECKLIST` | 7 |
| Runtime error → probable cause mapping | `## RUNTIME ERROR MAPPING` | 8 |
| **Working example** — choosing Map vs Array; storage fundamentals | `## TIER 1 — Data Storage Fundamentals: Map vs Object Literal; Choosing Array vs Map` | 33 |
| **Working example** — Array construction, mutation, safe access | `## TIER 2 — Array Construction, Mutation, and Safe Access` | 73 |
| **Working example** — Map construction, safe access, CaseSense | `## TIER 3 — Map Construction, Safe Access, Mutation, and CaseSense` | 90 |
| **Working example** — iteration over Array and Map | `## TIER 4 — Iteration Patterns: Array and Map Enumeration` | 59 |
| **Working example** — nested structures, static class Maps, filtering | `## TIER 5 — Advanced Patterns: Nested Structures, Static Class Maps, Filtering` | 84 |
| **Working example** — IndexError / UnsetItemError defensive guards | `## TIER 6 — Error Handling: IndexError, UnsetItemError, Defensive Guards` | 62 |
| Copy-paste working snippets | `## DROP-IN RECIPES` | 61 |
| Anti-patterns to avoid | `## ANTI-PATTERNS` | 11 |

---

### Closures & Dynamic Properties (Module_DynamicProperties.md — 499 lines)

| Task | Grep for heading | ~Lines |
|------|-----------------|--------|
| All API: fat arrow forms, meta-functions, DefineProp, `.Bind()` | `## API QUICK-REFERENCE` | 56 |
| Fat arrow syntax forms (short, returning, expression) | `### Fat Arrow Syntax Forms` | 11 |
| `__Get` / `__Set` meta-functions | `### Meta-Functions` | 8 |
| `DefineProp()` for programmatic property creation | `### DefineProp — Programmatic Property Definition` | 10 |
| Functional utilities (`Bind`, `Partial`, `Curry`) | `### Functional Utilities` | 9 |
| Map backing-store methods used in meta-function examples | `### Map — Backing Store Methods Used in Meta-Function Examples` | 7 |
| Built-in functions used in examples | `### Built-in Functions Used in Examples` | 9 |
| v1 → v2 breaking changes | `## V1 → V2 BREAKING CHANGES` | 12 |
| Constraints: arrow vs named function, capture-by-reference | `## AHK V2 CONSTRAINTS` | 26 |
| QA checklist before submitting code | `## AGENT QA CHECKLIST` | 7 |
| Runtime error → probable cause mapping | `## RUNTIME ERROR MAPPING` | 8 |
| **Working example** — basic arrow function syntax | `## TIER 1 — Basic Arrow Function Syntax` | 34 |
| **Working example** — named arrow functions + recursion | `## TIER 2 — Named Arrow Functions and Recursion` | 30 |
| **Working example** — closures + variable capture | `## TIER 3 — Closures and Variable Capture` | 41 |
| **Working example** — fat arrow properties | `## TIER 4 — Fat Arrow Properties` | 46 |
| **Working example** — dynamic properties + meta-functions | `## TIER 5 — Dynamic Properties and Meta-Functions` | 74 |
| **Working example** — functional programming patterns | `## TIER 6 — Functional Programming Patterns` | 71 |
| Copy-paste working snippets | `## DROP-IN RECIPES` | 63 |
| Anti-patterns to avoid | `## ANTI-PATTERNS` | 11 |

---

### Advanced: Runtime Class Prototyping (Module_ClassPrototyping.md — 538 lines)

| Task | Grep for heading | ~Lines |
|------|-----------------|--------|
| All API: descriptor construction, DefineProp, Func binding, type inspection | `## API QUICK-REFERENCE` | 39 |
| Object descriptor construction + prototype manipulation | `### Object — Descriptor Construction and Prototype Manipulation` | 14 |
| Descriptor object keys (`get`, `set`, `call`, `value`) | `### Descriptor Object Keys` | 9 |
| `Func` binding + introspection helpers | `### Func — Binding and Introspection` | 7 |
| Type inspection utilities | `### Type Inspection` | 7 |
| v1 → v2 breaking changes | `## V1 → V2 BREAKING CHANGES` | 12 |
| Constraints: descriptor pitfalls, prototype mutation order | `## AHK V2 CONSTRAINTS` | 45 |
| QA checklist before submitting code | `## AGENT QA CHECKLIST` | 7 |
| Runtime error → probable cause mapping | `## RUNTIME ERROR MAPPING` | 8 |
| **Working example** — descriptor fundamentals: dynamic call assignment | `## TIER 1 — Descriptor Object Fundamentals: Dynamic Call Assignment` | 31 |
| **Working example** — dynamic getters/setters + validated properties | `## TIER 2 — Dynamic Getters and Setters: Computed and Validated Properties` | 42 |
| **Working example** — existence guards: safe property/method validation | `## TIER 3 — Existence Guards: Safe Property and Method Validation` | 80 |
| **Working example** — closure factories: context-aware properties | `## TIER 4 — Closure Factories: Context-Aware Properties Without State Pollution` | 33 |
| **Working example** — method decorators: non-destructive interception | `## TIER 5 — Method Decorators: Non-Destructive Interception and Augmentation` | 58 |
| **Working example** — runtime class generation (dynamic class factory) | `## TIER 6 — Runtime Class Generation: v2.0-Compatible Dynamic Class Factory` | 64 |
| Copy-paste working snippets | `## DROP-IN RECIPES` | 56 |
| Anti-patterns to avoid | `## ANTI-PATTERNS` | 13 |

---

## Module Index

| Module | Topic | Load when task involves… |
|--------|-------|--------------------------|
| `Module_Instructions` | Core role, engineering principles, thinking tiers, syntax validation checklist | **Always** — load first (full file) |
| `ahk_formatting_spec` | Indentation (4-space), CRLF line endings, K&R brace style, naming conventions, comment rules | **Always** — load alongside `Module_Instructions` (full file) |
| `Module_Functions` | Function declarations, parameters, scoping, ByRef, return values | Function definitions, parameter handling, closures |
| `Module_Objects` | OOP fundamentals, property descriptors, object hierarchy, `DefineProp()` | Object design, property manipulation |
| `Module_Classes` | Class design, inheritance, lifecycle (`__New`, `__Delete`), meta-programming | Class definitions, inheritance, `__New`/`__Delete` |
| `Module_ClassPrototyping` | Runtime class creation, descriptor objects, prototype modification | Dynamic class generation, `DefineProp()` advanced use |
| `Module_Arrays` | Array creation, manipulation, traversal, sorting | Array operations, collection processing, list iteration |
| `Module_DataStructures` | Array vs Map selection, iteration, nested structures | Key-value storage, ordered lists, choosing container types |
| `Module_DynamicProperties` | Fat arrow functions, closures, `__Get`/`__Set`, functional patterns | Arrow syntax, lambdas, dynamic properties, `.Bind(this)` callbacks |

### Cross-Skill Dependencies

This skill is depended upon by all other AHK skills. It has no upstream
cross-skill dependencies.

| Referencing Skill | Modules borrowed |
|-------------------|-----------------|
| `get-ahk-review-context` | `references/Module_Instructions.md` |
| `get-ahk-logic-context` | `references/Module_Instructions.md`, `references/Module_DynamicProperties.md` |
| `get-ahk-ui-context` | `references/Module_Instructions.md`, `references/Module_Classes.md`, `references/Module_DynamicProperties.md` |
| `get-ahk-system-context` | `references/Module_Instructions.md` |

---

## Universal Critical Rules

### Syntax
- No space between function name and parenthesis: `Fn(x)` ✅ — `Fn (x)` ❌
- Classes instantiated without `new`: `MyClass()` ✅ — `new MyClass()` ❌
- Use `Map()` for key-value storage — never `{key: value}` object literals for data
- Curly braces `{}` are for function/class/control-flow bodies only
- Arrays are 1-based: first element is `array[1]`, not `array[0]`
- AHK v2 escape character is backtick (`` ` ``), NOT backslash (`\`):
  - Newline is `` `n `` (never `\n`)
  - Tab is `` `t `` (never `\t`)
  - Double quotes in double-quoted strings are `` `" `` or `""` (never `\"`)
  - Backslash `\` is a literal and does not escape anything

### Platform Tool Constraints
- **NEVER use the built-in `edit` tool on `*.ahk` files**: It has escape, CRLF, and Unicode bugs on AHK files. Use PowerShell/Python `.Replace()` via `bash` or full rewrite via `write` instead.
- **NEVER use LSP query tools on `*.ahk` files** (`lsp_symbols`, `lsp_find_references`, `lsp_goto_definition`, `lsp_prepare_rename`, `lsp_rename`): They crash the LSP server. Use `grep` and `read` instead.

### Formatting (always enforced)
- 4 spaces per indent level — never tabs
- CRLF line endings throughout
- K&R / OTB brace style — opening brace on the same line
- One space after `;` for comments — at least one space before inline `;`

### OOP
- Instance members: `this.property` — Static members: `ClassName.property`
- Event handlers and callbacks must use `.Bind(this)` for correct object context
- Resource cleanup goes in `__Delete()` meta-function
- super.Method() for base class access in derived classes

### Common Traps & Gotchas
- **Stored-callable context injection**: Calling a function or closure stored in an object property or Map element directly using obj.prop(args) treats prop as a method and automatically injects obj as the first argument. To call it without injection, extract it to a local variable: local fn := obj.prop; fn(args) or use obj.prop.Call(args).
- **FileOpen Encoding "RAW" error**: NEVER pass "RAW" as the encoding parameter to FileOpen() — it throws a runtime ValueError. To read raw binary, omit the parameter entirely. To write BOM-free files, use "UTF-8-RAW". To write standard UTF-8 files with BOM, use "UTF-8".
- **UTF-8 with BOM requirement**: AHK v2 source files (*.ahk) **must** be encoded as UTF-8 with BOM. If written BOM-free, non-ASCII characters in comments or literals will cause silent syntax/parse errors during compilation/execution.

---

## Reference Files Location

All reference modules are stored in the `references/` subdirectory alongside this SKILL.md file.

```
<skill-directory>/
  SKILL.md                      ← you are here
  references/
    Module_Instructions.md
    ahk_formatting_spec.md
    Module_Functions.md
    Module_Objects.md
    Module_Classes.md
    Module_ClassPrototyping.md
    Module_Arrays.md
    Module_DataStructures.md
    Module_DynamicProperties.md
```
