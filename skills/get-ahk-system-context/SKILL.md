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
| `Module_Instructions` | Core validation checklist — **always load first** | `.roo/skills/get-ahk-core-context/references/Module_Instructions.md` |
| `Module_Errors` | Error handling for file I/O, COM, registry, and network operations | `.roo/skills/get-ahk-logic-context/references/Module_Errors.md` |

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
   → `.roo/skills/get-ahk-core-context/references/Module_Instructions.md`
2. **Load the relevant primary module(s)** from this skill (see Module Index above)
3. **Load `Module_Errors`** from `get-ahk-logic-context` for any operation involving
   file I/O, COM, registry, or network
   → `.roo/skills/get-ahk-logic-context/references/Module_Errors.md`
4. Apply all critical rules before generating code

---

## Reference Files Location

Primary modules in `.roo/skills/get-ahk-system-context/references/`:

- `.roo/skills/get-ahk-system-context/references/Module_DllCallAndMemory.md`
- `.roo/skills/get-ahk-system-context/references/Module_SystemAndCOM.md`
- `.roo/skills/get-ahk-system-context/references/Module_FileSystem.md`
- `.roo/skills/get-ahk-system-context/references/Module_NetworkAndHTTP.md`
- `.roo/skills/get-ahk-system-context/references/Module_ScriptEnvironment.md`