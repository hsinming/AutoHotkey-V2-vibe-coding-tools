---
name: get-ahk-review-context
description: >
  Retrieves best-practice context for AHK v2 code review, auditing, and quality
  checks. Must be consulted before reviewing, auditing, or evaluating any
  AutoHotkey v2 code for correctness, style, or v2 compliance. Trigger this
  skill whenever the user mentions review, audit, evaluate, critique, "code
  review", "check my script", "find bugs", "quality check", "is this v2
  compliant", "check syntax", "best practices check", "what's wrong with my
  code", "improve my script", "why does my variable stay empty", or "my hotkey
  doesn't fire".
modeSlugs:
  - ahk-debug
  - ahk-ask
  - ahk-orchestrator
  - ahk-code
  - ahk-architect
---

# AHK v2 Review Context

This skill is the **primary knowledge owner** for AHK v2 code review standards,
severity classification, and v2 compliance auditing.

---

## Module Index

### Primary Module (owned by this skill)

| Module | Topic | When to load |
|--------|-------|--------------|
| `Module_CodeReview` | Multi-tier review framework (Tier 1–5+): v2 compliance, OOP structure, error handling, GUI patterns, performance; severity classification (Critical / Major / Minor) | Any code review, audit, quality check, or "what's wrong" request |

### Cross-Skill References

Load these alongside `Module_CodeReview` for complete review coverage:

| Module | Reason to load | Path |
|--------|---------------|------|
| `Module_Instructions` | Core engineering principles and syntax rules that define what "correct" means | `.roo/skills/get-ahk-core-context/references/Module_Instructions.md` |
| `Module_Errors` | Error handling review — empty catch blocks, wrong error classes, missing try/catch | `.roo/skills/get-ahk-logic-context/references/Module_Errors.md` |

---

## Review Tiers at a Glance

`Module_CodeReview` organises review work into tiers — load the module for full
checklists and examples:

| Tier | Focus |
|------|-------|
| 1 | Modern syntax & v2 compliance (assignment operators, percent syntax, directives) |
| 2 | OOP structure (class encapsulation, `__New`/`__Delete`, static vs instance) |
| 3 | Error handling (try/catch coverage, error class matching, empty catch) |
| 4 | GUI patterns (event binding, `.Bind(this)`, control layout) |
| 5+ | Performance, architecture, layering violations |

## Severity Rules (always apply)

- **Critical** — crashes, silent data loss, or permanent uninitialized state
  (e.g., `=` used as assignment is a silent no-op; empty `catch {}` swallows failures completely)
- **Major** — violates v2 idioms or creates measurable maintenance risk
- **Minor** — low-risk readability concern; style preferences alone never qualify as Major

---

## Loading Order

1. **Load `Module_Instructions`** from `get-ahk-core-context` first
   → `.roo/skills/get-ahk-core-context/references/Module_Instructions.md`
2. **Load `Module_CodeReview`** from this skill
3. **Load `Module_Errors`** from `get-ahk-logic-context` for any review involving
   error handling patterns
   → `.roo/skills/get-ahk-logic-context/references/Module_Errors.md`
4. Load domain-specific modules from other skills if the code under review
   involves GUI, system calls, or async patterns

---

## Reference Files Location

Primary module in `.roo/skills/get-ahk-review-context/references/`:

- `.roo/skills/get-ahk-review-context/references/Module_CodeReview.md`