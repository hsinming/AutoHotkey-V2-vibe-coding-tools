---
name: get-ahk-system-context
description: >
  Retrieves best-practice context for AHK v2 low-level OS interaction, memory
  operations, file system, networking, COM automation, and script
  environment/lifecycle management. Must be consulted before writing, modifying,
  or reviewing any AutoHotkey v2 code that involves DllCall, Buffer, Win32 API,
  registry, COM objects, file I/O, HTTP requests, JSON, tray icons, script
  directives, or exit/cleanup handling. Trigger this skill whenever the user
  mentions DLL calls, memory, Windows API, registry, clipboard, COM, Excel
  automation, WMI, file read/write, HTTP, network requests, JSON, script
  compilation, tray menu, admin elevation, or OnExit — even for casual requests
  like "read a file in AHK" or "call a Windows API". Always load the relevant
  reference module(s) before generating code.
modeSlugs:
  - ahk-orchestrator
  - ahk-architect
  - ahk-code
  - ahk-debug
  - ahk-ask
---

# AHK v2 System Context

This skill is the **primary knowledge owner** for AHK v2 code that interacts
with the operating system, external resources, and the script's own lifecycle.

---

## How to Use This Skill

Follow this decision flow every time a task triggers this skill.

**Step 1 — Map the task to primary module(s)**

| Task involves… | Load this module |
|----------------|-----------------|
| DllCall, Buffer, Win32 API, raw memory, native callbacks | `references/Module_DllCallAndMemory.md` |
| Run, processes, clipboard, registry, COM, Excel, WMI | `references/Module_SystemAndCOM.md` |
| FileOpen, FileRead, file write, directory traversal, paths | `references/Module_FileSystem.md` |
| HTTP requests, WinHttp, JSON, REST, file downloads | `references/Module_NetworkAndHTTP.md` |
| `#Requires`, `#SingleInstance`, tray menu, OnExit, Ahk2Exe | `references/Module_ScriptEnvironment.md` |

Multiple rows may match — load all that apply. Do not generate code from memory without loading the relevant module first.

**Step 2 — Load cross-skill dependencies in order**

1. **Always** load `Module_Instructions` from the core-context skill first — it is the v1→v2 regression gate that all other modules assume has been read:
   `../get-ahk-core-context/references/Module_Instructions.md`
2. Load the primary module(s) identified in Step 1.
3. *If file I/O, COM, registry, or network operations are present* — also load `Module_Errors` from the logic-context skill:
   `../get-ahk-logic-context/references/Module_Errors.md`

**Step 3 — Apply Universal Critical Rules**

Read the **Universal Critical Rules** section below and confirm each subsection applies to the loaded domain(s). The rules in that section are non-negotiable minimums; domain-specific constraints inside each module take precedence when they are stricter.

**Step 4 — Generate code against the appropriate TIER**

Write code at the complexity tier that matches the task (TIER 1 = basic creation/access → TIER 6 = advanced patterns). Read the loaded module's `## TIER N` section for the target tier before writing the first line. Every method you call must appear in that module's `## API QUICK-REFERENCE` table first.

**Step 5 — Pre-output self-check**

Before returning code to the user, confirm every applicable item:

- [ ] `Buffer()` used for all raw memory — no raw numeric pointer allocations
- [ ] Every `CallbackCreate()` call is paired with a `CallbackFree()` on cleanup
- [ ] All `FileOpen()` / `FileRead()` calls specify an explicit encoding argument
- [ ] All `FileOpen()` handles are closed in a `finally` block via `.Close()`
- [ ] All COM objects are released (assigned `""` or allowed to fall out of scope)
- [ ] All `WinWait` / `ProcessWait` calls have an explicit timeout argument
- [ ] All network and registry calls are wrapped in `try/catch`

---

## Module Index

### Primary Modules (owned by this skill)

| Module | Topic | When to load |
|--------|-------|--------------|
| `Module_DllCallAndMemory` | DllCall type signatures, Buffer allocation, NumPut/NumGet, Win32 struct simulation, OnMessage, CallbackCreate | DLL calls, raw memory, Win32 API, native callbacks |
| `Module_SystemAndCOM` | Process execution, clipboard, registry, COM automation (Excel, WMI), RunWait | System control, COM objects, registry, inter-process |
| `Module_FileSystem` | FileOpen/FileRead/FileWrite, encoding, SplitPath, DirExist, directory traversal | File and directory I/O |
| `Module_NetworkAndHTTP` | WinHttp COM requests, file downloads, JSON parsing/serialization, REST clients | HTTP calls, API integration, JSON handling |
| `Module_ScriptEnvironment` | Startup directives (`#Requires`, `#SingleInstance`), `#Include`, tray menu, admin elevation, Ahk2Exe metadata, OnExit cleanup | Script setup, lifecycle, compilation, tray |

### Cross-Skill References

Load these from other skills when the task requires them:

| Module | Reason to load | Path |
|--------|---------------|------|
| `Module_Instructions` | Core validation checklist — **always load first** | `../get-ahk-core-context/references/Module_Instructions.md` |
| `Module_Errors` | Error handling for file I/O, COM, registry, and network operations | `../get-ahk-logic-context/references/Module_Errors.md` |

---

## Universal Critical Rules

### Low-Level / DllCall
- Always specify DllCall type strings explicitly (e.g., `"Int"`, `"Ptr"`, `"AStr"`)
- Use `Buffer()` for all raw memory allocation — never raw numeric pointers
- Every `CallbackCreate()` handle must be freed with `CallbackFree()` when no longer needed
- Match encoding to API: `"WStr"` for Unicode Win32 functions, `"AStr"` for ANSI

### System & COM
- Use `A_Clipboard` exclusively — never legacy v1 clipboard variables
- Always assign COM objects to `""` or let them fall out of scope to release the reference count
- Always set timeouts on `WinWait` and `ProcessWait` to prevent infinite hangs
- Wrap all registry and WMI calls in `try/catch` for permission safety
- Retrieve process exit codes via `RunWait` return value, not `ErrorLevel`

### File System
- Always specify encoding explicitly in `FileOpen()` and `FileRead()`
- Never use `FileAppend` inside a loop — use a `FileOpen` File Object instead
- Always close File Objects via `.Close()` or a `__Delete` wrapper
- Use `SplitPath` for path parsing — never manual string splits or regex
- Validate existence with `FileExist()` / `DirExist()` before destructive operations

### Network & HTTP
- Use `ComObject("Msxml2.ServerXMLHTTP.6.0")` for all HTTP — no invented built-ins
- Always release WinHttp COM handles after use
- Never assume a built-in JSON parser exists in AHK v2 — implement or include one explicitly
- Wrap all network calls in `try/catch`

### Script Environment
- Directive order: `#Requires` → `#SingleInstance` → `#Include` → script body
- No v1 tray commands — use `A_TrayMenu` and the `Menu` object API
- Admin elevation via `if !A_IsAdmin { Run "*RunAs " A_ScriptFullPath; ExitApp }`
- Register cleanup in `OnExit()` for any resource that must be released on script termination

---

## Loading Order

1. **Load `Module_Instructions`** from `get-ahk-core-context` first
   → `../get-ahk-core-context/references/Module_Instructions.md`
2. **Load the relevant primary module(s)** from this skill (see Module Index above)
3. **Load `Module_Errors`** from `get-ahk-logic-context` for any operation involving
   file I/O, COM, registry, or network
   → `../get-ahk-logic-context/references/Module_Errors.md`
4. Apply all critical rules before generating code

---

## Reference Files Location

Primary modules live in `references/` alongside this SKILL.md:

- `references/Module_DllCallAndMemory.md`
- `references/Module_SystemAndCOM.md`
- `references/Module_FileSystem.md`
- `references/Module_NetworkAndHTTP.md`
- `references/Module_ScriptEnvironment.md`