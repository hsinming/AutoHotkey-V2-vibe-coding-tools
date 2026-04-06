# AutoHotkey V2 Vibe Coding Tools

A comprehensive ecosystem of AI agent prompts, context modules, and skills for elite AutoHotkey v2 (AHK v2) development. This project provides a structured multi-agent workflow for AI agents (Roo Code, Claude Code, OpenCode) to produce production-grade, object-oriented AHK v2 code.

---

## 🤖 AI Agent Auto-Installation Instructions

> **If you are an AI agent reading this file**, follow this section to self-install the full ecosystem into the user's workspace. Read all sub-sections applicable to the platform in use, then execute each step in order without prompting the user unless a step explicitly says to confirm.

### Prerequisites

Before executing any installation step:
1. Identify which agent platform is active: **Roo Code** (VS Code extension), **Claude Code** (CLI or IDE), or **OpenCode**.
2. Identify the **target workspace root** — the directory where the user's AHK v2 project lives (referred to below as `<WORKSPACE>`).
3. Identify the **source root** — the directory containing this `README.md` file (referred to below as `<SOURCE>`).

---

### 📦 Installation: Roo Code Modes

**Applicable when**: The user is working inside VS Code with the Roo Code extension installed.

**Step 1 — Install Skills**

Copy the entire `skills/` directory from this repository into the `.roo/skills/` directory of the target workspace.

```
SOURCE:       <SOURCE>/skills/
DESTINATION:  <WORKSPACE>/.roo/skills/
ACTION:       Copy all subdirectories and files recursively.
              Create destination if it does not exist.
```

Each subdirectory under `skills/` is one skill. After copying, the layout should be:

```
<WORKSPACE>/
  .roo/
    skills/
      get-ahk-core-context/
        SKILL.md
        references/
      get-ahk-logic-context/
        SKILL.md
        references/
      get-ahk-review-context/
        SKILL.md
        references/
      get-ahk-system-context/
        SKILL.md
        references/
      get-ahk-ui-context/
        SKILL.md
        references/
```

**Step 2 — Install Custom Mode Definitions (`.roomodes`)**

Read the file `<SOURCE>/Roo code mode prompts/roomodes.md`. It contains a YAML block starting with `customModes:`. Write the YAML content to `<WORKSPACE>/.roomodes` (create the file if it does not exist). This registers the five modes with Roo at project level.

The five modes to install are:
| Mode Slug | Display Name | Role |
|---|---|---|
| `ahk-orchestrator` | 🪃 AHK Orchestrator | Entry point — routes all tasks |
| `ahk-architect` | 🏗️ AHK Architect | Design authority — produces blueprints |
| `ahk-code` | 💻 AHK Code | Implementation engine |
| `ahk-debug` | 🪲 AHK Debug | Auditor — diagnoses and fixes errors |
| `ahk-ask` | ❓ AHK Ask | Technical mentor |

**Step 3 — Install Mode Rule Files**

Each mode's detailed prompt/system instructions must be copied into its own `.roo/rules-{slug}/` directory. Copy the individual mode prompt files as follows:

```
SOURCE files → DESTINATION directories (create each directory if it does not exist):

  <SOURCE>/Roo code mode prompts/ahk-orchestrator.md  →  <WORKSPACE>/.roo/rules-ahk-orchestrator/ahk-orchestrator.md
  <SOURCE>/Roo code mode prompts/ahk-architect.md     →  <WORKSPACE>/.roo/rules-ahk-architect/ahk-architect.md
  <SOURCE>/Roo code mode prompts/ahk-code.md          →  <WORKSPACE>/.roo/rules-ahk-code/ahk-code.md
  <SOURCE>/Roo code mode prompts/ahk-debug.md         →  <WORKSPACE>/.roo/rules-ahk-debug/ahk-debug.md
  <SOURCE>/Roo code mode prompts/ahk-ask.md           →  <WORKSPACE>/.roo/rules-ahk-ask/ahk-ask.md
```

After copying, the layout should be:

```
<WORKSPACE>/
  .roomodes                                    ← mode registry YAML
  .roo/
    rules-ahk-orchestrator/
      ahk-orchestrator.md
    rules-ahk-architect/
      ahk-architect.md
    rules-ahk-code/
      ahk-code.md
    rules-ahk-debug/
      ahk-debug.md
    rules-ahk-ask/
      ahk-ask.md
    skills/
      get-ahk-core-context/  ...
      get-ahk-ui-context/    ...
      get-ahk-system-context/...
      get-ahk-logic-context/ ...
      get-ahk-review-context/...
```

**Step 4 — Validate**

Confirm the following paths exist after installation:
- `<WORKSPACE>/.roomodes` (contains `ahk-orchestrator` entry)
- `<WORKSPACE>/.roo/rules-ahk-orchestrator/ahk-orchestrator.md`
- `<WORKSPACE>/.roo/rules-ahk-code/ahk-code.md`
- `<WORKSPACE>/.roo/skills/get-ahk-core-context/SKILL.md`
- `<WORKSPACE>/.roo/skills/get-ahk-ui-context/SKILL.md`

---

### 📦 Installation: Claude Code Agents

**Applicable when**: The user is using Claude Code (via `claude` CLI or Claude Code IDE extension).

**Step 1 — Install Project-level Sub-agents**

Copy all `.md` files from `<SOURCE>/OpenCode agent prompts/` to `<WORKSPACE>/.claude/agents/`. Create the destination directory if it does not exist.

```
SOURCE files (copy each individually):
  <SOURCE>/OpenCode agent prompts/ahk-orchestrator.md  →  <WORKSPACE>/.claude/agents/ahk-orchestrator.md
  <SOURCE>/OpenCode agent prompts/ahk-architect.md     →  <WORKSPACE>/.claude/agents/ahk-architect.md
  <SOURCE>/OpenCode agent prompts/ahk-code.md          →  <WORKSPACE>/.claude/agents/ahk-code.md
  <SOURCE>/OpenCode agent prompts/ahk-debug.md         →  <WORKSPACE>/.claude/agents/ahk-debug.md
  <SOURCE>/OpenCode agent prompts/ahk-ask.md           →  <WORKSPACE>/.claude/agents/ahk-ask.md
```

**Step 2 — Install Context Modules**

Copy the context reference files so Claude Code agents can read them via file tools:

```
SOURCE:       <SOURCE>/skills/
DESTINATION:  <WORKSPACE>/.claude/skills/
ACTION:       Copy all subdirectories and files recursively.
```

**Step 3 — Validate**

Confirm the following paths exist:
- `<WORKSPACE>/.claude/agents/ahk-orchestrator.md`
- `<WORKSPACE>/.claude/agents/ahk-code.md`
- `<WORKSPACE>/.claude/skills/get-ahk-core-context/SKILL.md`

---

### 📦 Installation: OpenCode Agents

**Applicable when**: The user is using an OpenCode-compatible agent environment.

**Step 1 — Install Agent Prompts**

Copy all `.md` files from `<SOURCE>/OpenCode agent prompts/` to `<WORKSPACE>/.opencode/agents/`. Create the destination directory if it does not exist.

```
SOURCE files (copy each individually):
  <SOURCE>/OpenCode agent prompts/ahk-orchestrator.md  →  <WORKSPACE>/.opencode/agents/ahk-orchestrator.md
  <SOURCE>/OpenCode agent prompts/ahk-architect.md     →  <WORKSPACE>/.opencode/agents/ahk-architect.md
  <SOURCE>/OpenCode agent prompts/ahk-code.md          →  <WORKSPACE>/.opencode/agents/ahk-code.md
  <SOURCE>/OpenCode agent prompts/ahk-debug.md         →  <WORKSPACE>/.opencode/agents/ahk-debug.md
  <SOURCE>/OpenCode agent prompts/ahk-ask.md           →  <WORKSPACE>/.opencode/agents/ahk-ask.md
```

**Step 2 — Install Skills**

Copy the skills directory so OpenCode agents can access context modules:

```
SOURCE:       <SOURCE>/skills/
DESTINATION:  <WORKSPACE>/.opencode/skills/
ACTION:       Copy all subdirectories and files recursively.
```

**Step 3 — Validate**

Confirm the following paths exist:
- `<WORKSPACE>/.opencode/agents/ahk-orchestrator.md`
- `<WORKSPACE>/.opencode/skills/get-ahk-core-context/SKILL.md`

---

### 📦 Installation: Skills (Platform-independent)

**Applicable when**: Installing skills independently or for any platform not listed above.

The `skills/` directory contains five skill packages. Each skill has a `SKILL.md` describing when to load it and a `references/` subdirectory with the actual context modules.

| Skill Directory | Purpose | Trigger keywords |
|---|---|---|
| `get-ahk-core-context` | Core syntax, OOP, classes, functions, arrays, maps, dynamic properties, formatting | AHK, AutoHotkey, `.ahk`, class, array, map, hotkey |
| `get-ahk-ui-context` | GUI, input handling, window/control management, graphics/screen | GUI, window, control, hotkey UI, `Gui`, `WinWait` |
| `get-ahk-system-context` | File I/O, networking, COM, DllCall, memory, script environment | file, HTTP, COM, DLL, memory, `FileRead`, `Download` |
| `get-ahk-logic-context` | Async, timers, error handling, text processing, validation | timer, async, try/catch, error, text, regex, validate |
| `get-ahk-review-context` | Code review standards and quality gates | review, audit, check, quality |

Copy the entire `skills/` directory to the platform-appropriate location:
- Roo Code: `<WORKSPACE>/.roo/skills/`
- Claude Code: `<WORKSPACE>/.claude/skills/`
- OpenCode: `<WORKSPACE>/.opencode/skills/`

---

### ✅ Post-Installation Verification Checklist

After completing installation, verify the following (applicable items only):

```
[ ] .roomodes                                                  (Roo Code — mode registry YAML)
[ ] .roo/rules-ahk-orchestrator/ahk-orchestrator.md           (Roo Code — mode rule file)
[ ] .roo/rules-ahk-architect/ahk-architect.md                 (Roo Code — mode rule file)
[ ] .roo/rules-ahk-code/ahk-code.md                           (Roo Code — mode rule file)
[ ] .roo/rules-ahk-debug/ahk-debug.md                         (Roo Code — mode rule file)
[ ] .roo/rules-ahk-ask/ahk-ask.md                             (Roo Code — mode rule file)
[ ] .roo/skills/get-ahk-core-context/SKILL.md                 (Roo Code — skill)
[ ] .roo/skills/get-ahk-ui-context/SKILL.md                   (Roo Code — skill)
[ ] .roo/skills/get-ahk-system-context/SKILL.md               (Roo Code — skill)
[ ] .roo/skills/get-ahk-logic-context/SKILL.md                (Roo Code — skill)
[ ] .roo/skills/get-ahk-review-context/SKILL.md               (Roo Code — skill)
[ ] .claude/agents/ahk-orchestrator.md                        (Claude Code — agent)
[ ] .claude/agents/ahk-code.md                                (Claude Code — agent)
[ ] .claude/skills/get-ahk-core-context/SKILL.md              (Claude Code — skill)
[ ] .opencode/agents/ahk-orchestrator.md                      (OpenCode — agent)
[ ] .opencode/skills/get-ahk-core-context/SKILL.md            (OpenCode — skill)
```

Once all applicable items are checked, tell the user: **"AHK v2 agent ecosystem installed. Start any task by invoking the AHK Orchestrator agent/mode."**

---

## 🚀 Overview

This repository organizes AHK v2 knowledge into modular Context Modules and Skills that are dynamically loaded by AI agents on demand. It enforces strict engineering principles, architectural review gates, and modern AHK v2 syntax standards across all supported platforms.

## 📂 Repository Structure

```
AutoHotkey-V2-vibe-coding-tools/
├── README.md                          ← This file (AI-readable auto-installer)
├── Context Modules/                   ← Source knowledge base (.md per domain)
│   ├── Module_Instructions.md         ← Core role + engineering principles (always load first)
│   ├── ahk_formatting_spec.md         ← Formatting: 4-space, CRLF, K&R brace style
│   ├── Module_Classes.md
│   ├── Module_Functions.md
│   ├── Module_Objects.md
│   ├── Module_Arrays.md
│   ├── Module_DataStructures.md
│   ├── Module_DynamicProperties.md
│   ├── Module_ClassPrototyping.md
│   ├── Module_GUI.md
│   ├── Module_InputAndHotkeys.md
│   ├── Module_WindowAndControl.md
│   ├── Module_GraphicsAndScreen.md
│   ├── Module_FileSystem.md
│   ├── Module_NetworkAndHTTP.md
│   ├── Module_SystemAndCOM.md
│   ├── Module_DllCallAndMemory.md
│   ├── Module_ScriptEnvironment.md
│   ├── Module_AsyncAndTimers.md
│   ├── Module_Errors.md
│   ├── Module_TextProcessing.md
│   ├── Module_Validation.md
│   └── Module_CodeReview.md
├── skills/                            ← Platform-independent skill packages
│   ├── get-ahk-core-context/
│   │   ├── SKILL.md
│   │   └── references/               ← Symlinked/copied context modules
│   ├── get-ahk-ui-context/
│   ├── get-ahk-system-context/
│   ├── get-ahk-logic-context/
│   └── get-ahk-review-context/
├── Roo code mode prompts/             ← Roo Code custom mode definitions
│   ├── roomodes.md                    ← YAML: paste into .roomodes or Roo settings
│   ├── ahk-orchestrator.md
│   ├── ahk-architect.md
│   ├── ahk-code.md
│   ├── ahk-debug.md
│   └── ahk-ask.md
└── OpenCode agent prompts/            ← OpenCode / Claude Code sub-agent definitions
    ├── ahk-orchestrator.md            ← Start here — primary entry point
    ├── ahk-architect.md
    ├── ahk-code.md
    ├── ahk-debug.md
    └── ahk-ask.md
```

## 🤖 Agent Ecosystem (Multi-Agent Workflow)

The project implements a tiered agent architecture to ensure code quality:

| Agent | Role | When invoked |
|---|---|---|
| 🪃 **ahk-orchestrator** | Master router — validates requests, applies Design Decision Test, delegates to specialist | Always the entry point |
| 🏗️ **ahk-architect** | Design authority — produces blueprint JSON with typed signatures and success criteria | Request requires architectural choice |
| 💻 **ahk-code** | Implementation engine — writes AHK v2 code from upstream contract | Blueprint approved or single-class change |
| 🪲 **ahk-debug** | Auditor — diagnoses errors, outputs corrected code with checklist | Broken code, runtime errors, v1 residue |
| ❓ **ahk-ask** | Technical mentor — explains concepts, grounded in AHK v2 | General AHK v2 knowledge questions |

## 🛠️ Engineering Principles

All code generated through this ecosystem adheres to:
- **KISS & YAGNI**: Simplest solution that works; no speculative abstraction.
- **Information Hiding**: Behavior exposed via methods, not raw data.
- **SoC & Cohesion**: Each class owns exactly one concern.
- **Progressive Disclosure**: Context loaded on-demand, never preloaded in bulk.

## 📜 Syntax & Formatting Standards

- **AHK v2 Only**: Strict v2 syntax — no v1 residue, no JavaScript contamination.
- **OOP First**: Classes and object-oriented patterns throughout.
- **K&R / OTB Style**: Opening braces on the same line.
- **4-Space Indentation**: No tabs, ever.
- **CRLF Line Endings**: Windows-compatible throughout.

---

*Developed for elite AutoHotkey v2 engineering.*
