---
name: get-ahk-review-context
description: >
  Retrieves best-practice context for AHK v2 code review, auditing, and quality
  checks. Must be consulted before reviewing, auditing, or evaluating any
  AutoHotkey v2 code for correctness, style, or v2 compliance. Trigger this
  skill whenever the user mentions review, audit, evaluate, critique, "code
  review", "check my script", "find bugs", "quality check", "is this v2
  compliant", "check syntax", "best practices check", "what's wrong with my
  code", "improve my script", "why does my variable stay empty", "my hotkey
  doesn't fire", "clean code", "over-engineering", "YAGNI", "refactor",
  "naming convention", or "too complex".
modeSlugs:
  - ahk-debug
  - ahk-ask
  - ahk-orchestrator
  - ahk-code
  - ahk-architect
---

# AHK v2 Code Review — Agent Execution Protocol

This skill owns all AHK v2 code review standards. Follow this protocol in
order on every review request. Do not skip steps or reorder them.

---

## How to Use This Skill

### Step 1 — Decision Table: task type → module to load

| Task type | Load module |
|-----------|-------------|
| Any code review / audit / v2 compliance check | `references/Module_CodeReview.md` |
| v2 syntax baseline verification (Dimension 1) | `../get-ahk-core-context/references/Module_Instructions.md` |
| OOP / class / inheritance review | `../get-ahk-core-context/references/Module_Classes.md` |
| Function naming contract enforcement | `../get-ahk-core-context/references/Module_Functions.md` |
| Error handling / throwing operations review | `../get-ahk-logic-context/references/Module_Errors.md` |
| GUI / event binding review (Dimension 4) | `../get-ahk-ui-context/references/Module_GUI.md` |
| DllCall / Buffer / COM security review (Dimension 5) | `../get-ahk-system-context/references/Module_DllCallAndMemory.md` |

### Step 2 — Cross-skill dependency loading order

Load in this sequence; conditional modules require scanning the submitted code first.

1. `../get-ahk-core-context/references/Module_Instructions.md` — **always load first** (authoritative v2 syntax baseline)
2. `references/Module_CodeReview.md` — **always load** (primary review framework: 6 dimensions, severity rules, Clean Code + OE contracts)
3. `../get-ahk-core-context/references/Module_Classes.md` — if `class`, `extends`, `__New`, `static`, or OOP patterns present
4. `../get-ahk-core-context/references/Module_Functions.md` — if `Get*`, `Check*`, `Is*`, or `Set*` function/method names present
5. `../get-ahk-logic-context/references/Module_Errors.md` — if `try`, `catch`, `FileRead`, `ComObject`, `RegRead`, `DllCall`, or `Download` present
6. `../get-ahk-ui-context/references/Module_GUI.md` — if `Gui()`, `.Add()`, `.OnEvent()`, or `SetTimer` present
7. `../get-ahk-system-context/references/Module_DllCallAndMemory.md` — if `DllCall`, `Buffer`, or `ComObjQuery` present

### Step 3 — Apply Universal Critical Rules

After loading all required modules, cross-check the submitted code against the Critical and Major severity rule tables in `references/Module_CodeReview.md` before writing any finding. Confirm: every Critical condition has been evaluated.

### Step 4 — Generate the review report

Produce the six-dimension report defined in `references/Module_CodeReview.md`, assigning each finding the mandatory severity level (🔴 Critical / 🟠 Major / 🟡 Minor) from the severity classification tables — not from personal judgment.

### Step 5 — Pre-output self-check

Before writing the final report, verify every item below:

- [ ] `=` is not used as assignment anywhere in the reviewed code (must be `:=`)
- [ ] No `catch {}` body is empty — every catch block contains a handling action
- [ ] Every `.OnEvent()` callback on a class method calls `.Bind(this)`
- [ ] No fat-arrow function uses a multi-statement body `=> { ... }`
- [ ] No duplicate hotkey definitions exist in the script
- [ ] Every `Get*` / `Fetch*` / `Read*` function/method is side-effect-free
- [ ] Every `Check*` / `Is*` / `Has*` function/method returns a boolean and mutates no state

---

## Step 1 — Load Modules (before reading the submitted code)

### Always load

| Module | Path | Why |
|--------|------|-----|
| `Module_CodeReview` | `references/Module_CodeReview.md` | Primary review framework: 6 dimensions, severity rules, Clean Code + OE contracts |
| `Module_Instructions` | `../get-ahk-core-context/references/Module_Instructions.md` | Authoritative v2 syntax baseline for Dimension 1 compliance |

### Load conditionally — scan the submitted code first

| If the submitted code contains… | Also load |
|---------------------------------|-----------|
| `class`, `extends`, `__New`, `static`, OOP patterns | `../get-ahk-core-context/references/Module_Classes.md` — naming contracts for class methods (`Get*`/`Check*`/`Is*`), `__Delete` lifecycle, `.Bind(this)` |
| Function definitions with `Get*`, `Check*`, `Is*`, `Set*` prefixes | `../get-ahk-core-context/references/Module_Functions.md` — naming contract enforcement, pure vs side-effect separation |
| `try`, `catch`, `FileRead`, `ComObject`, `RegRead`, `DllCall`, `Download` | `../get-ahk-logic-context/references/Module_Errors.md` — throwing operation reference, error class hierarchy |
| `Gui()`, `.Add()`, `.OnEvent()`, `SetTimer` | `../get-ahk-ui-context/references/Module_GUI.md` — event binding patterns for Dimension 4 |
| `DllCall`, `Buffer`, `ComObjQuery` | `../get-ahk-system-context/references/Module_DllCallAndMemory.md` — DllCall signature rules for Dimension 5 security checks |

---

## Step 2 — Write the Pre-Review Declaration

Before analysing a single line, output these four declarations exactly:

```
Review scope:    [Complete script | Fragment — Dim 2–6 limited to visible constructs]
AHK version:     [v2.0 confirmed via #Requires | v2.0 assumed — no directive found]
Script length:   [< 50 lines | 50–200 lines | 200–500 lines | 500+ lines]
Declared intent: [stated goal from user] | [Not declared — reviewing for general quality only]
```

**If any declaration is uncertain, make the most conservative assumption and note it.**
Do not proceed to Step 3 until this block is written.

---

## Step 3 — Execute the Six-Dimension Review

Evaluate dimensions in this fixed order. **Do not reorder.**

| # | Dimension | Primary focus | Stop early if… |
|---|-----------|---------------|----------------|
| 1 | Modern Syntax & v2 Compliance | `= vs :=`, `#Requires`, `%Var%`, MsgBox form, label hotkeys | Critical syntax errors found — note that Dims 2–5 findings may be invalid until Dim 1 is fixed |
| 2 | Error Handling & Type Safety | try/catch on throwing ops, non-empty catch bodies, type guards | — |
| 3 | Variable Scope, Architecture & SE Design Principles | globals, data structures, function length (≤ 40 lines), parameter count (≤ 4), nesting depth (≤ 3) | — |
| 4 | GUI & Event Binding | `.Bind(this)`, `.OnEvent()`, g-label removal, SetTimer refs | Only if GUI constructs are present — otherwise: `✅ Not applicable (no GUI)` |
| 5 | Performance, Security & Resource Management | loop invariants, O(n²) string concat, COM lifecycle, user input in DllCall/RegWrite/RegExMatch paths | — |
| **6** | **Clean Code, Over-Engineering & Composite** | **comment quality, naming contracts, YAGNI, over-abstraction, over-defensive guards, duplicate hotkeys** | **Always run — even for < 20 line scripts** |

### Dimension 6 priority rule

Run Dimension 6 Clean Code + OE checks **before** writing Dimension 3 architectural suggestions whenever the script shows:
- Class or factory with a single concrete subclass or single caller
- Neutral-verb function names (`ProcessData`, `HandleItem`, `DoWork`) on functions that mutate state
- Design patterns (Observer, Strategy, Factory) in scripts under 300 lines

**Reason:** OE findings in Dim 6 eliminate code that should not exist. Generating Dim 3 refactoring suggestions for YAGNI classes inflates the Priority Action List with advice for code that should be deleted.

---

## Step 4 — Severity Classification

Apply these rules on every finding. They are not guidelines — they are mandatory overrides.

### Critical 🔴 — fix before ship

| Condition | Example |
|-----------|---------|
| `=` used as assignment anywhere | `counter = 0` → variable stays `""` silently |
| Empty `catch {}` body | Exception swallowed; failure invisible |
| `.OnEvent()` callback without `.Bind(this)` on class method | `this` is undefined at runtime |
| Duplicate hotkey definition | Second definition silently overrides first |
| Multi-statement fat-arrow body `=> { }` | Load-time parse error |
| User input passed to `DllCall` path / `RegWrite` key without validation | Path traversal or key injection |
| Dynamic `RegExMatch(data, userPattern)` outside try/catch | Unhandled throw on malformed pattern |
| Function or nesting depth exceeds critical threshold | > 80 lines or nesting depth > 4 |

### Major 🟠 — fix in this review cycle

| Condition | Example |
|-----------|---------|
| `Get*`/`Fetch*`/`Read*` function/method with any side effect | `GetUser()` that calls `FileAppend()` |
| `Check*`/`Is*`/`Has*` function/method that mutates state | `CheckSession()` that resets `this.hitCount` |
| Abstraction with zero current callers (YAGNI) | Factory class used in exactly one place |
| Design pattern in script < 300 lines without justification comment | Observer pattern for 2 event types |
| Function > 40 lines or parameter count > 4 | Suggests mixed responsibilities |
| Nesting depth > 3 | Early-return flattening required |
| O(n²) string concat `.=` inside loop > ~20 iterations | — |
| `ObjRelease()` on `ComObject()` wrapper | Double-free risk |
| Magic string appearing in 3+ locations | Extract to named constant |
| Clean Code violations (negative naming, `Get*` side effect, comment restates code) | These are **Major by default** — not Minor — they signal systemic maintenance debt |

### Minor 🟡 — fix if time allows

| Condition |
|-----------|
| Comment restates code in English (single occurrence) |
| Non-descriptive name (`x`, `tmp2`) in non-loop context |
| Magic literal appearing in 1–2 locations only |
| Single defensive guard in an internal helper |
| Single-method class that could be a standalone function |

### Pass ✅ — always write explicitly

Every dimension that has no findings **must** appear in the report with `✅ No issues found.`
or `✅ Not applicable ([reason]).`

**Never silently omit a dimension.** A missing dimension line signals to the reader that
it was not evaluated, not that it passed.

---

## Step 5 — Finding Wording Rules

**Critical findings must:**
- Name the exact line number
- State the concrete runtime consequence
- Provide a fix with inline code (≤ 5 lines)
- Never use: "might", "could", "consider", "perhaps"

```
✓  🔴 Line 7 — `counter = 0` uses `=` (comparison in v2); counter stays "".
      Fix: `counter := 0`

✗  🔴 Line 7 — counter might not be assigned correctly. Consider using :=.
```

**Major findings must:**
- Name what the violation costs (maintenance, runtime, security)
- Name the recommended action

```
✓  🟠 Line 23 — `GetUser()` calls `FileAppend()` (side effect in a Get* function).
      Callers cannot safely repeat this call. Split into `GetUser()` + `LogAccess()`.

✗  🟠 Line 23 — GetUser() is not very clean.
```

**Aggregation rule:** If the same pattern (e.g., comment-restates-code) appears in 5+
locations, report it as one finding with representative line numbers — do not list all
occurrences individually.

---

## Step 6 — Produce the Report

Use this exact structure. Every dimension must appear.

```
AHK v2 Code Review Report
[Pre-review declaration block from Step 2]

Dimension 1 — Modern Syntax & v2 Compliance
- 🔴 / 🟠 / 🟡 Line N — [concrete consequence]. Fix: [inline code]
  OR
✅ No issues found.

Dimension 2 — Error Handling & Type Safety
...

Dimension 3 — Variable Scope, Architecture & SE Design Principles
...

Dimension 4 — GUI & Event Binding
✅ Not applicable (no GUI constructs present).

Dimension 5 — Performance, Security & Resource Management
...

Dimension 6 — Clean Code, Over-Engineering & Composite
...

Summary
2–3 sentences of overall quality assessment.

Priority Action List
1. [Top Critical finding] — one-line rationale. (🔴 Critical)
2. ...up to 5 items, ordered by severity then impact
```

**Priority Action List rules:**
- Maximum 5 items
- Order: Critical first → Major by impact → no Minors unless all Criticals and Majors resolved
- If Dim 6 flagged a class as YAGNI: list it once as "delete class X" — do not also list
  Dim 3 refactoring suggestions for the same class

---

## Quick-Reference: Naming Contracts

These contracts are enforced across `Module_CodeReview`, `Module_Functions`, and
`Module_Classes`. Apply on every review regardless of which module is loaded.

| Prefix | Contract | Violation severity |
|--------|----------|--------------------|
| `Get*` / `Fetch*` / `Read*` | Side-effect-free — no writes, no I/O, no `this.*` mutation | **Major** |
| `Check*` / `Is*` / `Has*` | Pure boolean predicate — no state mutation, returns `true`/`false` only | **Major** |
| `Set*` / `Reset*` / `Clear*` / `Update*` / `Log*` / `Write*` | Mutation declared — any mutating operation must carry one of these prefixes | Major if missing |

---

## Quick-Reference: Throwing vs Non-Throwing Operations

Do not fabricate try/catch requirements. Apply only to genuinely throwing operations.

| Throws → requires try/catch | Does NOT throw → no try/catch |
|-----------------------------|-------------------------------|
| `FileRead()`, `FileOpen()` | `MsgBox()`, `ToolTip()`, `Sleep()` |
| `ComObject()`, `RegRead()` | `Send()`, `StrLen()`, `OutputDebug()` |
| `DllCall()`, `Download()` | `WinActivate()` (returns 0 on miss, does not throw) |
| `RegExMatch()` / `RegExReplace()` with dynamic pattern | Static `RegExMatch()` with literal pattern |
| `WinGetTitle()` (throws `TargetError` if window not found) | — |

---

## Reference Files

```
references/Module_CodeReview.md                              ← primary (always load)
../get-ahk-core-context/references/Module_Instructions.md   ← always load alongside
../get-ahk-core-context/references/Module_Functions.md      ← load if Get*/Check*/Is* function names present
../get-ahk-core-context/references/Module_Classes.md        ← load if class / extends / OOP present
../get-ahk-logic-context/references/Module_Errors.md        ← load if try/catch or throwing ops present
../get-ahk-logic-context/references/Module_AsyncAndTimers.md ← load if SetTimer / async patterns present
../get-ahk-ui-context/references/Module_GUI.md              ← load if Gui() or .OnEvent() present
../get-ahk-system-context/references/Module_DllCallAndMemory.md ← load if DllCall / Buffer / COM present
```
