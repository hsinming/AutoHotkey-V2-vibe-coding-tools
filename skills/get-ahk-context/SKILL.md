---
name: get-ahk-context
description: >
  Provides targeted AHK v2 knowledge context to any AHK coding agent — including
  ahk-orchestrator, ahk-architect, ahk-code, ahk-debug, ahk-ask, and any future
  ahk-* agent mode. Must be consulted before executing any AHK v2 task: code
  generation, architectural design, debugging, explanation, or routing
  decisions. Trigger this skill whenever an AHK agent task contains
  topic_keywords, a delegation_payload, or any reference to AHK v2 concepts
  (GUI, classes, hotkeys, timers, files, COM, DllCall, regex, validation, etc.).
  Always load Module_Instructions.md first; retrieve all other modules
  selectively using keyword-based search — never preload entire files. Even for
  simple AHK tasks, always trigger this skill to ensure correct, idiomatic AHK
  v2 output.
modeSlugs:
  - ahk-orchestrator
  - ahk-architect
  - ahk-code
  - ahk-debug
  - ahk-ask
---

# AHK v2 Agent Context Retrieval

This skill implements **Progressive Disclosure** for AHK v2 knowledge injection.

- `Module_Instructions.md` — always loaded in full
- All other modules — loaded **selectively** via keyword search (grep / rg)
- Never dump an entire module file into context

All reference files live in `references/` relative to this SKILL.md.

---

## How to Use This Skill

### Step 1 — Decision Table: Task → Module

Load every module whose task description matches the current task. Multiple matches are expected and correct — load all of them.

| If the task involves… | Load this module |
|---|---|
| Storing or iterating lists, sorting, filtering, 1-based indexing | `references/Module_Arrays.md` |
| Timers, background tasks, async patterns, SetTimer, Sleep | `references/Module_AsyncAndTimers.md` |
| Defining classes, inheritance, `__New`/`__Delete`, static members | `references/Module_Classes.md` |
| Runtime class modification, `DefineProp`, property descriptors, meta-functions | `references/Module_ClassPrototyping.md` |
| Code quality review, auditing, finding bugs, best-practices check | `references/Module_CodeReview.md` |
| Key-value storage, Map, Dict, nested collections, iteration | `references/Module_DataStructures.md` |
| DllCall, Buffer, Win32 API, structs, NumGet/NumPut, memory | `references/Module_DllCallAndMemory.md` |
| Fat-arrow functions, closures, dynamic properties, `__Get`/`__Set`/`__Call` | `references/Module_DynamicProperties.md` |
| Error handling, try/catch, exception types, v1→v2 migration issues | `references/Module_Errors.md` |
| File read/write, directory ops, FileOpen, path handling, INI | `references/Module_FileSystem.md` |
| Code style, indentation, brace placement, formatting conventions | `references/ahk_formatting_spec.md` |
| Functions, parameters, ByRef, scope, closures, static variables | `references/Module_Functions.md` |
| ImageSearch, PixelSearch, screen coordinates, DPI, monitors | `references/Module_GraphicsAndScreen.md` |
| GUI windows, controls, layouts, dialogs, OnEvent, Gui() | `references/Module_GUI.md` |
| Hotkeys, hotstrings, Send, Click, keyboard/mouse input, `#HotIf` | `references/Module_InputAndHotkeys.md` |
| HTTP requests, WinHttp, REST APIs, JSON, download, network | `references/Module_NetworkAndHTTP.md` |
| Object properties, `DefineProp`, `HasProp`, `HasMethod`, callbacks | `references/Module_Objects.md` |
| `#Requires`, `#SingleInstance`, `#Include`, UAC, tray, Ahk2Exe | `references/Module_ScriptEnvironment.md` |
| Run, clipboard, registry, COM, Excel automation, WMI, processes | `references/Module_SystemAndCOM.md` |
| Strings, regex, `Format()`, `StrReplace()`, `StrSplit()`, escape sequences | `references/Module_TextProcessing.md` |
| Input validation, type guards, `IsSet()`, defensive patterns, `??` operator | `references/Module_Validation.md` |
| `WinExist`, `WinActivate`, `ControlClick`, HWND, window groups, hooks | `references/Module_WindowAndControl.md` |

### Step 2 — Cross-Skill Dependency Loading Order

Load in this sequence. Conditional entries are marked with an asterisk.

1. `references/Module_Instructions.md` — **always first**, unconditionally
2. `references/ahk_formatting_spec.md` — **always** when the agent outputs AHK code
3. `references/Module_Errors.md` — **always** alongside any other module (error handling pervades all domains)
4. `references/Module_Validation.md` — *if input validation, type-checking, or defensive patterns are present*
5. `references/Module_Objects.md` — *if classes, callbacks, or `DefineProp` are present*
6. `references/Module_Classes.md` → then `references/Module_ClassPrototyping.md` — *if OOP or runtime class modification is present; load in this order*
7. `references/Module_DataStructures.md` → then `references/Module_Arrays.md` — *if collection types are present; load DataStructures first for Map/Array choice guidance*
8. `references/Module_GUI.md` → then `references/Module_WindowAndControl.md` — *if GUI or window interaction is present; load GUI first*
9. `references/Module_DllCallAndMemory.md` → then `references/Module_SystemAndCOM.md` — *if low-level OS calls or COM are present; load DllCall first*
10. `references/Module_AsyncAndTimers.md` — *if background tasks, SetTimer, or any non-blocking pattern is present*
11. Domain-specific modules (FileSystem, NetworkAndHTTP, InputAndHotkeys, GraphicsAndScreen, TextProcessing, Functions, ScriptEnvironment, DynamicProperties) — *load only those matched in Step 1*

### Step 3 — Apply Universal Critical Rules

After loading all modules, re-read the **AHK V2 CONSTRAINTS** section of every loaded module and confirm each rule is internalised before writing a single line of code.

> **Confirm action:** Mentally run the pre-output checklist in Step 5 against your planned implementation before proceeding.

### Step 4 — Generate Code

Write the implementation using the TIER system as a complexity guide: TIER 1–2 patterns cover the common case; reach for TIER 3–6 patterns only when the task genuinely requires it. Never introduce a higher-tier pattern where a lower-tier one is sufficient.

> **Reference:** Every module's `## TIER N —` sections contain runnable, annotated examples. Use `> METHODS COVERED:` declarations to locate the right tier for each API.

### Step 5 — Pre-Output Self-Check

Before delivering any code, verify every item:

- [ ] `ClassName()` used for instantiation — **never** `new ClassName()`
- [ ] Every class method passed as a callback uses `.Bind(this)` — bare method references lose `this`
- [ ] Key-value data uses `Map("key", val)` — **never** object literal `{key: val}`
- [ ] Map values accessed with bracket notation `m["key"]` or `m.Get("key", default)` — **never** dot notation `m.key`
- [ ] Fat-arrow functions contain a **single expression only** — no `{ }` block body, no multi-statement body
- [ ] `Buffer(size)` used for raw memory — **never** `VarSetCapacity` (removed in v2)
- [ ] All arrays indexed from **1**, not 0 — `arr[1]` is the first element; `arr[0]` silently returns unset

---

## Step 1 — Always Load: `Module_Instructions.md`

Read the full file before doing anything else.

```bash
cat references/Module_Instructions.md
```

This file contains:
- Engineering principles (KISS, SoC, DIP, Two Hats Rule)
- Thinking tiers (Thinking / Ultrathink)
- The authoritative keyword → module routing table (`<routing_table>`)
- The `<diagnostic_checklist>` all agents must run before finalising code
- Edge-case and fallback rules

---

## Step 2 — Identify Topic Keywords

Collect keywords from all available sources, in priority order:

1. **`topic_keywords[]`** from the upstream `delegation_payload` (most precise)
2. **`task_summary`** or the user's raw request — extract nouns and verbs
3. **Agent type signals** — e.g., if the calling agent is a code-generation mode, always add `"format"` to the keyword set to include `ahk_formatting_spec.md`

Normalise all keywords to lowercase.

---

## Step 3 — Map Keywords to Modules

Cross-reference your keyword set against this routing table.
Load **every** module that matches at least one keyword.

| Keywords (match any) | Module File |
|---|---|
| array, push, pop, length, sort, loop, index, 1-based | `Module_Arrays.md` |
| timer, settimer, sleep, async, delay, periodic, background | `Module_AsyncAndTimers.md` |
| class, inherit, __new, __delete, static, super, extends, lifecycle | `Module_Classes.md` |
| prototype, runtime class, defineprop, descriptor, dynamic class, prototyping | `Module_ClassPrototyping.md` |
| map, key-value, store, dict, array vs map, iteration, nested, collection | `Module_DataStructures.md` |
| dllcall, buffer, win32, api, memory, struct, numget, numput, low-level | `Module_DllCallAndMemory.md` |
| =>, fat arrow, lambda, closure, dynamic property, __get, __set, __call | `Module_DynamicProperties.md` |
| error, wrong, broken, fail, syntax error, runtime error, undefined, not working, v1 to v2 | `Module_Errors.md` |
| file, folder, directory, read, write, save, load, path, txt, ini, fileopen | `Module_FileSystem.md` |
| format, indent, style, comment, brace, crlf, convention, lint, spacing | `ahk_formatting_spec.md` |
| function, return, parameter, optional, byref, scope, global, local, static var | `Module_Functions.md` |
| imagesearch, pixelsearch, pixelgetcolor, screen, monitor, dpi, coordinates | `Module_GraphicsAndScreen.md` |
| gui, window, form, dialog, button, control, layout, position, xm, section, onevent | `Module_GUI.md` |
| hotkey, send, click, mouse, keyboard, bind key, hotstring, #hotif | `Module_InputAndHotkeys.md` |
| http, download, request, api, json, rest, winhttp, post, get, network | `Module_NetworkAndHTTP.md` |
| object, property, descriptor, defineprop, hasprop, hasmethod, bound, bind, callback | `Module_Objects.md` |
| #requires, #singleinstance, #include, admin, uac, tray, compile, ahk2exe | `Module_ScriptEnvironment.md` |
| run, cmd, clipboard, registry, com, excel, wmi, system, process, winactivate | `Module_SystemAndCOM.md` |
| string, text, escape, quote, regex, pattern, match, replace, split, join, `n | `Module_TextProcessing.md` |
| validate, type check, isset, typeerror, duck typing, assert, defensive, ?? | `Module_Validation.md` |
| winexist, winactivate, controlclick, controlsend, hwnd, groupadd, hook, focus | `Module_WindowAndControl.md` |
| review, audit, evaluate, critique, code review, check my script, find bugs, quality check, what's wrong, improve, best practices | `Module_CodeReview.md` |

**Special rules:**
- Code-generating agents (any agent that outputs AHK code): always add `ahk_formatting_spec.md`
- Multiple-keyword overlap: load all matched modules; do not consolidate
- Unrecognised keyword with no match: note `[no module matched for: <keyword>]` and continue

---

## Step 4 — Retrieve Relevant Sections

For each module identified in Step 3, extract only the sections relevant to the
task keywords. Do **not** read the whole file unless it is under 80 lines.

> ⚠️ **`search_files` is prohibited for module retrieval.** It returns fragmented
> line matches that break mid-sentence and miss surrounding context. Use the awk
> semantic chunking method below instead.

### Step 4a — Orient: list all headings

```bash
grep "^#" references/Module_Target.md
```

Scan the heading list to identify which section title(s) most closely match the
task keywords. Note the exact heading text for Step 4b.

### Step 4b — Extract: semantic chunking with awk

Use this single awk command to extract a **complete heading block** — from the
matched heading down to (but not including) the next heading of equal or higher
level. The block is always structurally complete; it never truncates mid-sentence
or mid-example.

```bash
awk -v search="<keyword>" '
BEGIN { search = tolower(search) }
/^#+ / {
    lvl = length($1)
    if (!p && tolower($0) ~ search) {
        p = 1
        target_lvl = lvl
        print
        next
    }
    if (p) {
        if (lvl <= target_lvl) exit
        print
        next
    }
}
p' references/Module_Target.md
```

Replace `<keyword>` with the section title word or task keyword (case-insensitive).

> ⚠️ **The awk script exits after extracting the first matched block.** Run it
> separately for each specific heading identified in Step 4a — one invocation
> per heading, not one invocation per broad keyword.

**Multiple keywords** — run once per keyword and concatenate results:

```bash
for kw in "gui" "onevent" "button"; do
  echo "=== [$kw] ==="
  awk -v search="$kw" '
  BEGIN { search = tolower(search) }
  /^#+ / {
      lvl = length($1)
      if (!p && tolower($0) ~ search) { p=1; target_lvl=lvl; print; next }
      if (p) { if (lvl <= target_lvl) exit; print; next }
  }
  p' references/Module_GUI.md
done
```

### Extraction strategy

1. Run `grep "^#" <file>` to see all headings and choose the best match
2. Run the awk command with the matched heading keyword to extract the full block
3. If the keyword appears in multiple headings, extract each block separately
4. Concatenate all extracted blocks — they form the context for this module

### Context budget per module

| Module | Max lines to inject |
|---|---|
| `Module_Instructions.md` | Full file (always) |
| `Module_GUI.md`, `Module_Classes.md`, `Module_Validation.md` | 200 lines |
| All other modules | 120 lines |
| `ahk_formatting_spec.md` | Full file (188 lines) |

If extracted content slightly exceeds the limit, **do not truncate inside a
code block** (` ```ahk ... ``` `). It is better to exceed the budget by a few
lines than to break code syntax. If it massively exceeds the budget, run the
awk command again with a narrower heading keyword to extract a subsection instead.

---

## Step 5 — Format and Inject Context

Assemble all retrieved content into labelled context blocks **before** beginning
the AHK task. Use this format exactly:

```
╔══ CONTEXT BLOCK ══════════════════════════════════════════╗
║ Source : Module_Instructions.md  [always-loaded]          ║
╚═══════════════════════════════════════════════════════════╝
<full content>

╔══ CONTEXT BLOCK ══════════════════════════════════════════╗
║ Source : Module_GUI.md  [keywords: gui, button, OnEvent]  ║
║ Lines  : 45–98, 210–267                                   ║
╚═══════════════════════════════════════════════════════════╝
<extracted sections>

╔══ CONTEXT BLOCK ══════════════════════════════════════════╗
║ Source : ahk_formatting_spec.md  [code-generation]        ║
║ Lines  : 1–188 (full)                                     ║
╚═══════════════════════════════════════════════════════════╝
<full content>
```

After injecting all context blocks, proceed with the AHK task as instructed by
the calling agent.

---

## Step 6 — Post-Injection Checklist

Before generating any code, confirm all of the following:

- [ ] `Module_Instructions.md` was read in full
- [ ] All modules matched by topic keywords were loaded
- [ ] `ahk_formatting_spec.md` included (if agent produces code)
- [ ] No module injected beyond its line budget
- [ ] Each context block header includes source filename + matched keywords
- [ ] `<diagnostic_checklist>` from `Module_Instructions.md` is in context and
      will be applied before finalising code output

---

## Edge Cases

| Situation | Action |
|---|---|
| `references/` path not accessible | Fall back to `../get-ahk-context/references/` using the same grep strategy |
| Module file missing | Log `[MISSING: filename]` in context header; continue with remaining modules |
| No keywords match any module | Load `Module_Instructions.md` only; note `[no additional modules matched]` |
| Task scope spans > 5 modules | Load all matched modules; apply per-module line budgets strictly |
| Ambiguous keyword (matches multiple modules) | Load all matched modules |
| Calling agent produces no code (routing/explanation only) | Omit `ahk_formatting_spec.md` unless "format" keyword is present |
