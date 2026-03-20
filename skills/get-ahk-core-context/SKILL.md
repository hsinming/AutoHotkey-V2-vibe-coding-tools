---
name: get-ahk-core-context
description: >
  Retrieves best-practice context for AHK v2 core syntax, functions, OOP,
  data structures, dynamic properties, and code formatting. Must be consulted
  before writing, modifying, or reviewing any AutoHotkey v2 code. Trigger this
  skill whenever the user mentions AHK, AutoHotkey, .ahk files, hotkeys,
  scripts, classes, arrays, maps, objects, arrow functions, closures, or dynamic
  properties — even for casual requests like "write me an AHK script" or "fix
  this AHK code". Always load the relevant reference module(s) before generating
  any code.
modeSlugs:
  - ahk-orchestrator
  - ahk-code
  - ahk-architect
  - ahk-debug
  - ahk-ask
---

# AHK v2 Core Context

This skill is the **primary knowledge owner** for AHK v2 core syntax, OOP
patterns, data structures, dynamic properties, and code formatting standards.
All other skills depend on `Module_Instructions` from this skill — load it first.

---

## Module Index

| Module | Topic | When to load |
|--------|-------|--------------|
| `Module_Instructions` | Core role, engineering principles, thinking tiers, syntax validation checklist | **Always** — load first for every AHK task |
| `ahk_formatting_spec` | Indentation (4-space), CRLF line endings, K&R brace style, naming conventions, comment rules | **Always** — load alongside `Module_Instructions` for any code generation or review |
| `Module_Functions` | Function declarations, parameters, scoping, ByRef, return values | Function definitions, parameter handling |
| `Module_Objects` | OOP fundamentals, property descriptors, object hierarchy, `DefineProp()` | Object design, property manipulation |
| `Module_Classes` | Class design, inheritance, lifecycle (`__New`, `__Delete`), meta-programming | Class definitions, inheritance patterns |
| `Module_ClassPrototyping` | Runtime class creation, descriptor objects, prototype modification | Dynamic class generation, `DefineProp()` advanced use |
| `Module_Arrays` | Array creation, manipulation, traversal, sorting | Array operations, collection processing |
| `Module_DataStructures` | Array vs Map selection, iteration, nested structures | Key-value storage, ordered lists |
| `Module_DynamicProperties` | Fat arrow functions, closures, `__Get`/`__Set`, functional patterns | Arrow syntax, lambdas, dynamic properties, `.Bind(this)` callback patterns |

---

## Universal Critical Rules

These rules apply regardless of which module is active.

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
- **Ultrathink**: Complex OOP, prototyping, closures — compare 3+ architectures, simulate edge cases, justify all design decisions

---

## Cross-Skill Dependencies

This skill is depended upon by all other AHK skills. It has no upstream
cross-skill dependencies.

| Referencing Skill | Modules borrowed |
|-------------------|-----------------|
| `get-ahk-review-context` | `Module_Instructions` |
| `get-ahk-logic-context` | `Module_Instructions`, `Module_DynamicProperties` |
| `get-ahk-ui-context` | `Module_Instructions`, `Module_Classes`, `Module_DynamicProperties` |
| `get-ahk-system-context` | `Module_Instructions` |

---

## Loading Order

1. **Always load first**: `Module_Instructions` + `ahk_formatting_spec`
2. **Load topic modules** based on the task (see Module Index above)
3. **Cross-check** references noted inside each module
4. Apply all critical rules before generating code

---

## Reference Files Location

All modules are in `.roo/skills/get-ahk-core-context/references/`:

- `.roo/skills/get-ahk-core-context/references/Module_Instructions.md`
- `.roo/skills/get-ahk-core-context/references/ahk_formatting_spec.md`
- `.roo/skills/get-ahk-core-context/references/Module_Functions.md`
- `.roo/skills/get-ahk-core-context/references/Module_Objects.md`
- `.roo/skills/get-ahk-core-context/references/Module_Classes.md`
- `.roo/skills/get-ahk-core-context/references/Module_ClassPrototyping.md`
- `.roo/skills/get-ahk-core-context/references/Module_Arrays.md`
- `.roo/skills/get-ahk-core-context/references/Module_DataStructures.md`
- `.roo/skills/get-ahk-core-context/references/Module_DynamicProperties.md`