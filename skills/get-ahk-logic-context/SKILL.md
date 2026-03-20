---
name: get-ahk-logic-context
description: >
  Retrieves best-practice context for AHK v2 asynchronous operations, timers,
  error handling, data validation, and text/string processing. Must be consulted
  before writing, modifying, or reviewing any AutoHotkey v2 code that involves
  SetTimer, background tasks, try/catch, error classes, guard clauses, type
  checking, string manipulation, or regex. Trigger this skill whenever the user
  mentions timers, async, delayed execution, error handling, debugging,
  validation, input checking, strings, regex, text parsing, or escape sequences
  — even for casual requests like "add a timer to my AHK script" or "validate
  user input in AHK". Always load the relevant reference module(s) before
  generating any code.
modeSlugs:
  - ahk-orchestrator
  - ahk-architect
  - ahk-code
  - ahk-debug
  - ahk-ask
---

# AHK v2 Logic Context

This skill is the **primary knowledge owner** for AHK v2 program logic:
async/timer patterns, structured error handling, defensive validation,
and text processing.

---

## Module Index

### Primary Modules (owned by this skill)

| Module | Topic | When to load |
|--------|-------|--------------|
| `Module_AsyncAndTimers` | `SetTimer`, one-off timers, `.Bind(this)` for callbacks, event-driven architecture, pseudo-concurrency rules, `Sleep -1` message queue flush | Background tasks, delayed execution, periodic updates, non-blocking loops |
| `Module_Errors` | Error classification, `try/catch`, v1→v2 migration pitfalls, syntax/runtime error diagnosis, structured troubleshooting | Error handling, debugging, crash diagnosis |
| `Module_Validation` | Guard clauses, `TypeError`/`ValueError`/`OSError`/`TargetError`, duck-typing via `HasMethod()`/`HasProp()`, fluent `ValidationBuilder` for GUI forms | Input validation, type/constraint checking, defensive programming |
| `Module_TextProcessing` | String building, backtick escape sequences, `.=` concatenation, regex (PCRE with AHK options), `RTrim`, quote type selection, text validation | String operations, pattern matching, text parsing, escape handling |

### Cross-Skill References

Load these from `get-ahk-core-context` when the task requires them:

| Module | Reason to load | Path |
|--------|---------------|------|
| `Module_Instructions` | Core validation checklist — **always load first** | `.roo/skills/get-ahk-core-context/references/Module_Instructions.md` |
| `Module_DynamicProperties` | Fat arrow callbacks and closures — essential for timer and async callback patterns | `.roo/skills/get-ahk-core-context/references/Module_DynamicProperties.md` |

---

## Universal Critical Rules

### Async & Timers
- Class methods passed to `SetTimer` must always use `.Bind(this)` or `ObjBindMethod` to preserve context
- Never use blocking `Sleep` inside GUI event handlers or critical sections
- Use negative intervals (e.g., `-1000`) for one-off timers instead of `Sleep`
- `Sleep -1` is a special case — it flushes the message queue immediately, it does **not** delay
- Timer callbacks must catch their own exceptions — errors inside timers do not propagate outward

### Error Handling
- Use `:=` for assignment, never `=` (v1 legacy)
- All built-in commands are functions requiring parentheses and quoted string arguments
- Never use `%variable%` percent-sign syntax inside expressions
- Match the error class to the failure domain: `TypeError`, `ValueError`, `OSError`, `TargetError`
- Always wrap external/uncertain operations in `try/catch`

### Validation
- Apply guard clauses at the very top of every function that accepts external input — before any logic
- Use `HasMethod()` and `HasProp()` for duck-type capability checks — never assert a class name
- Fail early: throw the most specific error class available
- Use fluent `ValidationBuilder` patterns for multi-field GUI form validation scenarios

### Text Processing
- Use backtick `` ` `` for all escape sequences — never backslash `\`
- Use `.=` operator for string concatenation in loops
- Use `RTrim()` or a ternary pattern when joining arrays to avoid trailing delimiters
- Regex uses PCRE syntax with AHK-specific options: `i` (case-insensitive), `m` (multiline), `s` (dot-all), `x` (extended)
- Choose quote type to minimise escaping: single quotes preserve literals, double quotes interpret escapes

---

## Loading Order

1. **Load `Module_Instructions`** from `get-ahk-core-context` first
   → `.roo/skills/get-ahk-core-context/references/Module_Instructions.md`
2. **Load the relevant primary module(s)** from this skill (see Module Index above)
3. **Load `Module_DynamicProperties`** from `get-ahk-core-context` when writing
   timer callbacks, async patterns, or any closure/fat-arrow code
   → `.roo/skills/get-ahk-core-context/references/Module_DynamicProperties.md`
4. Apply all critical rules before generating code

---

## Reference Files Location

Primary modules in `.roo/skills/get-ahk-logic-context/references/`:

- `.roo/skills/get-ahk-logic-context/references/Module_AsyncAndTimers.md`
- `.roo/skills/get-ahk-logic-context/references/Module_Errors.md`
- `.roo/skills/get-ahk-logic-context/references/Module_Validation.md`
- `.roo/skills/get-ahk-logic-context/references/Module_TextProcessing.md`