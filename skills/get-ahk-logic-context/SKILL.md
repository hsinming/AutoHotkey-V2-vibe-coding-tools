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

## How to Use This Skill

### Step 1 — Select modules based on the task

| If the task involves… | Load this module |
|-----------------------|-----------------|
| `SetTimer`, one-off timers, background tasks, delayed execution, non-blocking loops | `references/Module_AsyncAndTimers.md` |
| `try/catch`, error classes, crash diagnosis, v1→v2 error migration | `references/Module_Errors.md` |
| Input validation, guard clauses, type/constraint checking, defensive programming | `references/Module_Validation.md` |
| Strings, regex, escape sequences, text parsing, concatenation, quote selection | `references/Module_TextProcessing.md` |

Load **all** rows that match — tasks often span multiple modules simultaneously.

### Step 2 — Load cross-skill dependencies in order

1. **Always load first:** `../get-ahk-core-context/references/Module_Instructions.md` — core v2 validation checklist; must be present before any other module.
2. **Load matching primary modules** identified in Step 1.
3. **Conditional:** `../get-ahk-core-context/references/Module_DynamicProperties.md` — *load if timer callbacks, async patterns, or any fat-arrow / closure code is present.*

### Step 3 — Apply Universal Critical Rules

Read every rule in the **Universal Critical Rules** section below and confirm it applies to the code you are about to generate before proceeding.

### Step 4 — Generate code

Produce AHK v2 code that follows the TIER progression defined in the loaded module(s) — start from the lowest TIER that covers the required functionality and work upward only as complexity demands.

### Step 5 — Pre-output self-check

Verify each item before delivering output:

- [ ] Class methods passed to `SetTimer` use `.Bind(this)` or `ObjBindMethod` — never a bare method reference
- [ ] No blocking `Sleep` inside GUI event handlers or critical sections
- [ ] Assignment uses `:=` — never bare `=` (v1 legacy)
- [ ] No `%variable%` percent-sign syntax inside expressions
- [ ] Error class matches the failure domain: `TypeError`, `ValueError`, `OSError`, or `TargetError`
- [ ] Guard clauses are at the very top of every function that accepts external input — before any logic
- [ ] All escape sequences use backtick `` ` `` — never backslash `\`

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
| `Module_Instructions` | Core validation checklist — **always load first** | `../get-ahk-core-context/references/Module_Instructions.md` |
| `Module_DynamicProperties` | Fat arrow callbacks and closures — essential for timer and async callback patterns | `../get-ahk-core-context/references/Module_DynamicProperties.md` |

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
   → `../get-ahk-core-context/references/Module_Instructions.md`
2. **Load the relevant primary module(s)** from this skill (see Module Index above)
3. **Load `Module_DynamicProperties`** from `get-ahk-core-context` when writing
   timer callbacks, async patterns, or any closure/fat-arrow code
   → `../get-ahk-core-context/references/Module_DynamicProperties.md`
4. Apply all critical rules before generating code

---

## Reference Files Location

Primary modules in `references/` (same directory as this SKILL.md):

- `references/Module_AsyncAndTimers.md`
- `references/Module_Errors.md`
- `references/Module_Validation.md`
- `references/Module_TextProcessing.md`