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

### Step 1 — Decision Table: load modules matched to the task

| If the task involves… | Load this module |
|-----------------------|-----------------|
| **Any** AHK v2 task (always required) | `references/Module_Instructions.md` |
| **Any** AHK v2 task (always required) | `references/ahk_formatting_spec.md` |
| Function definitions, parameters, ByRef, closures | `references/Module_Functions.md` |
| Object design, property descriptors, `DefineProp()` | `references/Module_Objects.md` |
| Class definitions, inheritance, `__New` / `__Delete` | `references/Module_Classes.md` |
| Dynamic class generation, runtime prototype modification | `references/Module_ClassPrototyping.md` |
| Array operations, collection processing, list iteration | `references/Module_Arrays.md` |
| Key-value storage, choosing Map vs Array, nested structures | `references/Module_DataStructures.md` |
| Arrow functions, lambdas, dynamic properties, `.Bind(this)` callbacks | `references/Module_DynamicProperties.md` |

### Step 2 — Cross-skill loading order

Load in this sequence; conditional modules are marked with their trigger condition:

1. `references/Module_Instructions.md` — always first
2. `references/ahk_formatting_spec.md` — always second
3. `references/Module_Functions.md` — *if function/parameter design is present*
4. `references/Module_Objects.md` — *if object hierarchy or `DefineProp()` is present*
5. `references/Module_Classes.md` — *if class definitions or inheritance is present*
6. `references/Module_ClassPrototyping.md` — *if runtime prototype modification is present*
7. `references/Module_Arrays.md` — *if array or collection operations are present*
8. `references/Module_DataStructures.md` — *if Map/Array selection or nested structures are present*
9. `references/Module_DynamicProperties.md` — *if arrow functions, closures, or `.Bind(this)` callbacks are present*

For cross-skill references (modules owned by sibling skills), use:
- UI / window / control ops → `../get-ahk-ui-context/references/Module_GUI.md` *(if window/control ops present)*
- Logic / async / error ops → `../get-ahk-logic-context/references/Module_Errors.md` *(if try/catch or timers present)*
- System / DLL / COM ops → `../get-ahk-system-context/references/Module_DllCallAndMemory.md` *(if DllCall or COM present)*

### Step 3 — Apply Universal Critical Rules

Before writing any code, confirm each rule in the checklist below applies to the current task. Treat any violation as a blocking error — do not proceed to generation until all applicable rules are confirmed.

### Step 4 — Generate code

Write code guided by the TIER system in `references/Module_Instructions.md`. Match code complexity to the appropriate tier; do not introduce patterns above the tier required by the task.

### Step 5 — Pre-output self-check

- [ ] No `new ClassName()` — instantiation is `ClassName()` without `new`
- [ ] No `{key: value}` object literals used as data containers — use `Map("key", value)`
- [ ] Every class method callback uses `.Bind(this)` — bare method references lose `this` context
- [ ] Arrays indexed from `1`, not `0` — `arr[1]` is the first element
- [ ] No space between function name and parenthesis — `Fn(x)` not `Fn (x)`
- [ ] 4-space indentation, CRLF line endings, opening brace on same line (K&R / OTB)
- [ ] Resource cleanup (`FileClose`, `CallbackFree`, COM teardown) is in `__Delete()` or a `finally` block

---

## Module Index

| Module | Topic | Load when task involves… |
|--------|-------|--------------------------|
| `Module_Instructions` | Core role, engineering principles, thinking tiers, syntax validation checklist | **Always** — load first |
| `ahk_formatting_spec` | Indentation (4-space), CRLF line endings, K&R brace style, naming conventions, comment rules | **Always** — load alongside `Module_Instructions` |
| `Module_Functions` | Function declarations, parameters, scoping, ByRef, return values | Function definitions, parameter handling, closures |
| `Module_Objects` | OOP fundamentals, property descriptors, object hierarchy, `DefineProp()` | Object design, property manipulation |
| `Module_Classes` | Class design, inheritance, lifecycle (`__New`, `__Delete`), meta-programming | Class definitions, inheritance, `__New`/`__Delete` |
| `Module_ClassPrototyping` | Runtime class creation, descriptor objects, prototype modification | Dynamic class generation, `DefineProp()` advanced use |
| `Module_Arrays` | Array creation, manipulation, traversal, sorting | Array operations, collection processing, list iteration |
| `Module_DataStructures` | Array vs Map selection, iteration, nested structures | Key-value storage, ordered lists, choosing container types |
| `Module_DynamicProperties` | Fat arrow functions, closures, `__Get`/`__Set`, functional patterns | Arrow syntax, lambdas, dynamic properties, `.Bind(this)` callbacks |

### Quick Selection by Task Type

| Task | Load these modules |
|------|--------------------|
| Write or review any AHK code | `references/Module_Instructions.md` + `references/ahk_formatting_spec.md` |
| Define a class | + `references/Module_Classes.md` |
| Class with inheritance or meta-functions | + `references/Module_Classes.md` + `references/Module_Objects.md` |
| Dynamic/computed properties, closures | + `references/Module_DynamicProperties.md` |
| Runtime prototype modification | + `references/Module_ClassPrototyping.md` |
| Array/collection operations | + `references/Module_Arrays.md` |
| Choosing Map vs Array, nested structures | + `references/Module_DataStructures.md` |
| Function design, ByRef, parameters | + `references/Module_Functions.md` |
| Callback with `.Bind()` or SetTimer | + `references/Module_DynamicProperties.md` |

---

## Universal Critical Rules

These rules apply regardless of which domain module is active.
Violations in any generated code are unconditional errors.

### Syntax
- No space between function name and parenthesis: `Fn(x)` ✅ — `Fn (x)` ❌
- Classes instantiated without `new`: `MyClass()` ✅ — `new MyClass()` ❌
- Use `Map()` for key-value storage — never `{key: value}` object literals for data
- Curly braces `{}` are for function/class/control-flow bodies only
- Arrays are 1-based: first element is `array[1]`, not `array[0]`

### Formatting (always enforced)
- 4 spaces per indent level — never tabs
- CRLF line endings throughout
- K&R / OTB brace style — opening brace on the same line
- One space after `;` for comments — at least one space before inline `;`

### OOP
- Instance members: `this.property` — Static members: `ClassName.property`
- Event handlers and callbacks must use `.Bind(this)` for correct object context
- Resource cleanup goes in `__Delete()` meta-function
- `super.Method()` for base class access in derived classes

### Thinking Tiers (from `Module_Instructions`)
- **Thinking**: Standard tasks — plan → validate → write → internal_validation
- **Ultrathink**: Complex OOP, prototyping, closures — compare 3+ architectures,
  simulate edge cases, justify all design decisions

---

## Cross-Skill Dependencies

This skill is depended upon by all other AHK skills. It has no upstream
cross-skill dependencies.

| Referencing Skill | Modules borrowed |
|-------------------|-----------------|
| `get-ahk-review-context` | `references/Module_Instructions.md` |
| `get-ahk-logic-context` | `references/Module_Instructions.md`, `references/Module_DynamicProperties.md` |
| `get-ahk-ui-context` | `references/Module_Instructions.md`, `references/Module_Classes.md`, `references/Module_DynamicProperties.md` |
| `get-ahk-system-context` | `references/Module_Instructions.md` |

---

## Reference Files Location

All reference modules are stored in the `references/` subdirectory alongside this SKILL.md file.
Use your agent's file-reading tool to load them by constructing the path relative
to this file's location:

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

**To locate the reference files:** resolve the directory of this SKILL.md file, then
load any module as `references/<filename>`. No hardcoded absolute paths are
required — the skill works regardless of where it is installed.
