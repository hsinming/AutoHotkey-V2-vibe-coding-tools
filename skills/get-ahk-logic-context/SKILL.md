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

### Step 1 — Map the task to targeted sections via Section Navigator

Do **not** load entire Module files. Use the **Section Navigator** below to find
the exact heading to grep and the number of lines to read. Start with
`## API QUICK-REFERENCE` + `## AHK V2 CONSTRAINTS` for most tasks; descend into
TIER sections only when a runnable code example is needed.

### Step 2 — Load cross-skill dependencies in order

1. **Always load first:** `../get-ahk-core-context/references/Module_Instructions.md` (full file, 431 lines)
2. **Load matching primary module sections** identified via the Section Navigator.
3. **Conditional:** `../get-ahk-core-context/references/Module_DynamicProperties.md` — load targeted sections if timer callbacks, async patterns, or closures are present.

### Step 3 — Apply Universal Critical Rules

Read every rule in the **Universal Critical Rules** section below and confirm it
applies to the code you are about to generate before proceeding.

### Step 4 — Generate code

Produce AHK v2 code following the TIER progression in the loaded module(s) —
start from the lowest TIER that covers the required functionality.

### Step 5 — Pre-output self-check

- [ ] Class methods passed to `SetTimer` use `.Bind(this)` or `ObjBindMethod` — never a bare method reference
- [ ] No blocking `Sleep` inside GUI event handlers or critical sections
- [ ] Assignment uses `:=` — never bare `=` (v1 legacy)
- [ ] No `%variable%` percent-sign syntax inside expressions
- [ ] Error class matches the failure domain: `TypeError`, `ValueError`, `OSError`, or `TargetError`
- [ ] Guard clauses are at the very top of every function that accepts external input — before any logic
- [ ] All escape sequences use backtick `` ` `` — never backslash `\`

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

### Timers & Async (Module_AsyncAndTimers.md — 517 lines)

| Task | Grep for heading | ~Lines |
|------|-----------------|--------|
| All timer/async API: SetTimer, Sleep, Critical, binding utilities | `## API QUICK-REFERENCE` | 45 |
| `SetTimer` signature, interval, period `-1` one-shot | `### SetTimer` | 7 |
| `Sleep` and `Sleep -1` (message-queue flush) | `### Sleep` | 5 |
| `Critical` — blocking the message queue | `### Critical` | 8 |
| `.Bind(this)` / `ObjBindMethod` for callbacks | `### Binding Utilities` | 6 |
| v1 → v2 breaking changes (timers) | `## V1 → V2 BREAKING CHANGES` | 12 |
| Constraints: re-entrancy, Sleep in GUI, one-shot pattern | `## AHK V2 CONSTRAINTS` | 27 |
| **Working example** — basic timer creation + one-off delays | `## TIER 1 — Basic Timer Creation and One-Off Delays` | 36 |
| **Working example** — timer lifecycle + class integration | `## TIER 2 — Timer Lifecycle and Class Integration` | 68 |
| **Working example** — state tracking, re-entrancy guards, Critical | `## TIER 3 — State Tracking, Re-Entrancy Guards, and Critical Sections` | 106 |
| **Working example** — debounce and throttle | `## TIER 4 — Higher-Order Timing: Debounce and Throttle` | 43 |
| **Working example** — parameterized callbacks + advanced binding | `## TIER 5 — Parameterized Callbacks and Advanced Binding` | 78 |
| **Working example** — non-blocking state machines + chunked processing | `## TIER 6 — Non-Blocking State Machines and Chunked Processing` | 70 |
| Anti-patterns to avoid | `## ANTI-PATTERNS` | 14 |

---

### Error Handling (Module_Errors.md — 666 lines)

| Task | Grep for heading | ~Lines |
|------|-----------------|--------|
| All error API: built-in error classes, properties, try/catch/finally | `## API QUICK-REFERENCE` | 51 |
| Built-in error class hierarchy (`Error`, `TypeError`, `ValueError`, `OSError`, `TargetError`) | `### Built-in Error Classes` | 15 |
| Error object properties (`.Message`, `.What`, `.Extra`, `.Stack`) | `### Error Object Properties` | 10 |
| `try/catch/finally`, `throw`, `OnError()` | `### Exception Control Flow` | 9 |
| Which operations throw (vs return falsy) | `### Supporting Functions Used in Error Patterns` | 15 |
| v1 → v2 breaking changes (error handling) | `## V1 → V2 BREAKING CHANGES` | 15 |
| Constraints: throwing vs non-throwing ops, empty catch ban | `## AHK V2 CONSTRAINTS` | 30 |
| **Working example** — syntax, built-in variable, and escaping errors | `## TIER 1 — Syntax, Built-in Variable and Escaping Errors` | 61 |
| **Working example** — scope, control flow, return statement errors | `## TIER 2 — Scope, Control Flow and Return Statement Errors` | 74 |
| **Working example** — logic, operator, event and callback errors | `## TIER 3 — Logic, Operator, Event and Callback Errors` | 70 |
| **Working example** — context, hotkey, automation and path errors | `## TIER 4 — Context, Hotkey, Automation and Path Errors` | 70 |
| **Working example** — try/catch, custom exceptions, OnError | `## TIER 5 — Exception Handling: try/catch, Custom Exceptions and OnError` | 146 |
| **Working example** — version/compatibility + diagnostic patterns | `## TIER 6 — Version, Compatibility and Diagnostic Patterns` | 114 |
| Anti-patterns to avoid | `## ANTI-PATTERNS` | 15 |

---

### Text & String Processing (Module_TextProcessing.md — 436 lines)

| Task | Grep for heading | ~Lines |
|------|-----------------|--------|
| All string API: StrLen, SubStr, StrReplace, Trim, Format, … | `## API QUICK-REFERENCE` | 32 |
| String built-in functions (`StrLen`, `SubStr`, `StrReplace`, `Trim`, `Format`, …) | `### String Functions` | 15 |
| Regex functions (`RegExMatch`, `RegExReplace`) | `### Regex Functions` | 6 |
| `RegExMatch` match object properties | `### RegExMatch Object (matchObj)` | 9 |
| v1 → v2 breaking changes (strings, regex) | `## V1 → V2 BREAKING CHANGES` | 12 |
| Constraints: backtick escapes, `.=` in loops, quote selection | `## AHK V2 CONSTRAINTS` | 16 |
| **Working example** — string building, concatenation, quote strategy | `## TIER 1 — String Building, Concatenation, and Quote Strategy` | 33 |
| **Working example** — backtick escape system (full reference) | `## TIER 2 — Escape Sequences: The Backtick System` | 63 |
| **Working example** — built-in string methods | `## TIER 3 — Built-in String Methods` | 35 |
| **Working example** — regex fundamentals: RegExMatch + RegExReplace | `## TIER 4 — Regex Fundamentals: RegExMatch and RegExReplace` | 62 |
| **Working example** — string validation classes | `## TIER 5 — String Validation Classes` | 43 |
| **Working example** — StringBuilder + EscapeValidator | `## TIER 6 — StringBuilder and EscapeValidator: Advanced String Construction` | 109 |
| Anti-patterns to avoid | `## ANTI-PATTERNS` | 12 |

---

### Input Validation (Module_Validation.md — 959 lines)

| Task | Grep for heading | ~Lines |
|------|-----------------|--------|
| All validation API: type detection, capability probes, ValidationBuilder | `## API QUICK-REFERENCE` | 53 |
| `IsObject()`, `IsInteger()`, `IsFloat()`, `IsAlpha()`, … type operators | `### Type Detection Operators` | 9 |
| `HasMethod()`, `HasProp()` capability probes | `### Object Capability Probes` | 6 |
| `TypeError`, `ValueError`, `OSError`, `TargetError` constructors | `### Error Constructors (Validation Context)` | 8 |
| `FileExist()`, `WinExist()`, `DirExist()` pre-flight functions | `### Environment Pre-Flight Functions` | 9 |
| `ValidationBuilder` / `FormValidator` fluent API | `### ValidationBuilder / FormValidator Methods` | 19 |
| v1 → v2 breaking changes (validation) | `## V1 → V2 BREAKING CHANGES` | 12 |
| Constraints: duck-typing, fail-early, guard-clause placement | `## AHK V2 CONSTRAINTS` | 21 |
| **Working example** — type and null fundamentals | `## TIER 1 — Type and Null Fundamentals` | 125 |
| **Working example** — guard clauses | `## TIER 2 — Guard Clauses` | 150 |
| **Working example** — string + business logic validation | `## TIER 3 — String and Business Logic Validation` | 101 |
| **Working example** — duck-type object + interface validation | `## TIER 4 — Duck-Type Object and Interface Validation` | 103 |
| **Working example** — state + external environment validation | `## TIER 5 — State and External Environment Validation` | 178 |
| **Working example** — fluent ValidationBuilder framework | `## TIER 6 — Fluent Validator Framework` | 187 |
| Anti-patterns to avoid | `## ANTI-PATTERNS` | 13 |

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

---

## Reference Files Location

- `references/Module_AsyncAndTimers.md`
- `references/Module_Errors.md`
- `references/Module_Validation.md`
- `references/Module_TextProcessing.md`
