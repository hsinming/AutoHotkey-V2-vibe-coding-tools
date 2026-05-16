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

### Step 1 — Map the task to targeted sections via Section Navigator

Do **not** load entire Module files. Use the **Section Navigator** below to find
the exact heading to grep and the number of lines to read. Start with
`## API QUICK-REFERENCE` + `## AHK V2 CONSTRAINTS` for most tasks; descend into
TIER sections only when a runnable code example is needed.

### Step 2 — Load cross-skill dependencies in order

1. **Always** load `Module_Instructions` from the core-context skill first:
   `../get-ahk-core-context/references/Module_Instructions.md` (full file, 431 lines)
2. Load targeted sections from the primary module(s) identified in Step 1.
3. *If file I/O, COM, registry, or network operations are present* — also load targeted sections from `Module_Errors`:
   `../get-ahk-logic-context/references/Module_Errors.md`

### Step 3 — Apply Universal Critical Rules

Read the **Universal Critical Rules** section below and confirm each subsection
applies to the loaded domain(s).

### Step 4 — Generate code

Write code at the complexity tier that matches the task. Read the loaded module's
`## TIER N` section for the target tier before writing the first line.

### Step 5 — Pre-output self-check

- [ ] `Buffer()` used for all raw memory — no raw numeric pointer allocations
- [ ] Every `CallbackCreate()` call is paired with a `CallbackFree()` on cleanup
- [ ] All `FileOpen()` / `FileRead()` calls specify an explicit encoding argument
- [ ] All `FileOpen()` handles are closed in a `finally` block via `.Close()`
- [ ] All COM objects are released (assigned `""` or allowed to fall out of scope)
- [ ] All `WinWait` / `ProcessWait` calls have an explicit timeout argument
- [ ] All network and registry calls are wrapped in `try/catch`

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

### DLL Calls & Memory (Module_DllCallAndMemory.md — 1334 lines)

| Task | Grep for heading | ~Lines |
|------|-----------------|--------|
| All DllCall/memory API overview | `## API QUICK-REFERENCE` | 153 |
| `Buffer()` constructor, `.Ptr`, `.Size` | `### Buffer Object` | 9 |
| `NumPut` / `NumGet` numeric memory I/O | `### Numeric Memory I/O` | 8 |
| `StrPut` / `StrGet` string memory I/O | `### String Memory I/O` | 8 |
| `DllCall()` type strings, return types | `### DllCall` | 13 |
| `OnMessage()` / `PostMessage()` / `SendMessage()` | `### Message Functions` | 7 |
| `CallbackCreate()` / `CallbackFree()` | `### Callback Functions` | 10 |
| Win32 type mapping (`Int`, `Ptr`, `WStr`, `AStr`, …) | `### Win32 Type Mapping` | 34 |
| v1 → v2 breaking changes | `## V1 → V2 BREAKING CHANGES` | 15 |
| Constraints: Buffer rules, type-string pitfalls, encoding selection | `## AHK V2 CONSTRAINTS` | 39 |
| **Working example** — Buffer fundamentals | `## TIER 1 — Buffer Object Fundamentals` | 52 |
| **Working example** — memory read, write, block operations | `## TIER 2 — Memory Read, Write, and Block Operations` | 158 |
| **Working example** — simple DllCall + core Win32 utilities | `## TIER 3 — Simple DllCall and Core Win32 Utilities` | 290 |
| **Working example** — system message passing + interception | `## TIER 4 — System Message Passing and Interception` | 81 |
| **Working example** — struct simulation, heap + global memory | `## TIER 5 — Struct Simulation, Heap and Global Memory` | 386 |
| **Working example** — C-style callbacks with CallbackCreate | `## TIER 6 — C-Style Callbacks with CallbackCreate` | 124 |
| Anti-patterns to avoid | `## ANTI-PATTERNS` | 20 |

---

### File System (Module_FileSystem.md — 420 lines)

| Task | Grep for heading | ~Lines |
|------|-----------------|--------|
| All file/directory API: FileRead, FileOpen, SplitPath, Loop, INI | `## API QUICK-REFERENCE` | 84 |
| `FileRead` / `FileAppend` / `FileDelete` standalone functions | `### Standalone Read/Write Functions` | 8 |
| `FileCopy` / `FileMove` / `FileDelete` mutation functions | `### File Mutation Functions` | 7 |
| `DirCreate` / `DirDelete` / `DirCopy` directory functions | `### Directory Functions` | 8 |
| `SplitPath` / `A_WorkingDir` / `A_ScriptDir` path functions | `### Path Functions` | 5 |
| `FileOpen()` File Object methods (`.Read`, `.Write`, `.Seek`, `.Close`) | `### FileOpen Handle (File Object)` | 19 |
| `IniRead` / `IniWrite` / `IniDelete` INI functions | `### INI Functions` | 7 |
| `Loop Files` / `Loop Read` + built-in loop variables | `### Loop Constructs and Built-in Loop Variables` | 12 |
| v1 → v2 breaking changes | `## V1 → V2 BREAKING CHANGES` | 12 |
| Constraints: encoding, FileAppend-in-loop ban, path validation | `## AHK V2 CONSTRAINTS` | 19 |
| **Working example** — existence checks + simple read/write | `## TIER 1 — Fundamentals: Existence Checks and Simple Read/Write` | 30 |
| **Working example** — copy, move, delete | `## TIER 2 — Core Mutation: Copy, Move, and Delete` | 33 |
| **Working example** — path parsing + directory traversal | `## TIER 3 — Search and Validation: Path Parsing and Directory Traversal` | 31 |
| **Working example** — line parsing + CSV processing | `## TIER 4 — Transformations: Line Parsing and CSV Processing` | 39 |
| **Working example** — INI config + FileOpen lifecycle + wrapper classes | `## TIER 5 — Advanced Operations: INI Config, FileOpen Lifecycle, and Wrapper Classes` | 89 |
| **Working example** — binary I/O: raw streaming + file pointer | `## TIER 6 — Binary I/O: Raw Streaming and File Pointer Manipulation` | 55 |
| Anti-patterns to avoid | `## ANTI-PATTERNS` | 10 |

---

### Network & HTTP (Module_NetworkAndHTTP.md — 1109 lines)

| Task | Grep for heading | ~Lines |
|------|-----------------|--------|
| All HTTP API: Download, WinHttp COM, Jxon, HttpPromise | `## API QUICK-REFERENCE` | 80 |
| `Download()` built-in — simple file fetch | `### Download (built-in)` | 5 |
| WinHttp core request methods (`Open`, `Send`, `ResponseText`, …) | `### WinHttp.WinHttpRequest.5.1 COM Object — Core Request Methods` | 18 |
| WinHttp extended capabilities (timeouts, auth, certificates) | `### WinHttp.WinHttpRequest.5.1 COM Object — Extended Capabilities` | 11 |
| `WaitForResponse` — three blocking modes | `### WaitForResponse — Three Blocking Modes` | 8 |
| WinHttp vs Msxml2 comparison table | `### WinHttp vs Msxml2 — Capability Comparison` | 14 |
| `HttpPromise` class API (async, TIER 5) | `### HttpPromise Class (defined in TIER 5)` | 8 |
| Jxon library — JSON parse/stringify | `### Jxon Community Library (third-party — required for full JSON support)` | 6 |
| v1 → v2 breaking changes | `## V1 → V2 BREAKING CHANGES` | 18 |
| Constraints: no built-in JSON, COM release, try/catch requirement | `## AHK V2 CONSTRAINTS` | 35 |
| **Working example** — basic file download | `## TIER 1 — Basic File Download` | 56 |
| **Working example** — synchronous HTTP GET | `## TIER 2 — Synchronous HTTP GET` | 88 |
| **Working example** — HTTP POST with headers + authentication | `## TIER 3 — HTTP POST with Headers and Authentication` | 143 |
| **Working example** — JSON handling strategies | `## TIER 4 — JSON Handling Strategies` | 111 |
| **Working example** — async HTTP + Promise-style concurrency | `## TIER 5 — Asynchronous HTTP and Promise-Style Concurrency` | 294 |
| **Working example** — full REST API client class | `## TIER 6 — Full REST API Client Class` | 250 |
| Anti-patterns to avoid | `## ANTI-PATTERNS` | 17 |

---

### Script Environment & Lifecycle (Module_ScriptEnvironment.md — 633 lines)

| Task | Grep for heading | ~Lines |
|------|-----------------|--------|
| All script-environment API: directives, A_* variables, tray, Ahk2Exe | `## API QUICK-REFERENCE` | 63 |
| `#Requires`, `#SingleInstance`, `#Include` directive signatures | `### Directives (load-time, not runtime-conditional)` | 8 |
| Built-in `A_*` variables (A_ScriptDir, A_Args, A_IsAdmin, …) | `### Built-in A_* Variables` | 20 |
| `A_TrayMenu` (Menu object) API | `### A_TrayMenu (Menu Object)` | 8 |
| `OnExit()`, `ExitApp()`, `Reload()`, `Pause()` lifecycle functions | `### Standalone Functions` | 13 |
| Ahk2Exe compiler directives (`@Ahk2Exe-SetMainIcon`, …) | `### Ahk2Exe Compiler Directives` | 12 |
| v1 → v2 breaking changes | `## V1 → V2 BREAKING CHANGES` | 13 |
| Constraints: directive order, tray API changes, admin elevation | `## AHK V2 CONSTRAINTS` | 20 |
| **Working example** — script directives + environment variables | `## TIER 1 — Script Directives and Environment Variables` | 85 |
| **Working example** — `#Include` + library system | `## TIER 2 — Include Directives and Library System` | 55 |
| **Working example** — tray menu customization | `## TIER 3 — Tray Menu Customization` | 72 |
| **Working example** — admin elevation patterns | `## TIER 4 — Admin Elevation Patterns` | 64 |
| **Working example** — Ahk2Exe compiler directives (metadata) | `## TIER 5 — Compiler Directives (Ahk2Exe Metadata)` | 125 |
| **Working example** — script lifecycle + OnExit cleanup hooks | `## TIER 6 — Script Lifecycle and OnExit Cleanup Hooks` | 108 |
| Anti-patterns to avoid | `## ANTI-PATTERNS` | 12 |

---

### System Control & COM Automation (Module_SystemAndCOM.md — 542 lines)

| Task | Grep for heading | ~Lines |
|------|-----------------|--------|
| All system/COM API: clipboard, Run, registry, COM functions | `## API QUICK-REFERENCE` | 48 |
| `A_Clipboard`, `OnClipboardChange()`, clipboard read/write | `### Clipboard` | 7 |
| `Run`, `RunWait`, `ProcessExist`, `ProcessWaitClose` | `### Process and Execution` | 12 |
| `WinWait`, `WinExist`, `WinClose`, `WinActivate` | `### Window` | 9 |
| `RegRead`, `RegWrite`, `RegDelete`, `RegDeleteKey` | `### Registry` | 8 |
| `ComObject()`, `ComValue()`, `ComObjQuery()`, `ObjRelease()` | `### COM Functions` | 10 |
| v1 → v2 breaking changes | `## V1 → V2 BREAKING CHANGES` | 12 |
| Constraints: clipboard, COM release, WinWait timeouts, registry safety | `## AHK V2 CONSTRAINTS` | 19 |
| **Working example** — clipboard + application launch | `## TIER 1 — Clipboard and Application Launch Fundamentals` | 37 |
| **Working example** — registry operations + process exit codes | `## TIER 2 — Registry Operations and Process Exit Codes` | 45 |
| **Working example** — process + window state validation | `## TIER 3 — Process and Window State Validation` | 37 |
| **Working example** — WMI data queries + Map transformation | `## TIER 4 — WMI Data Queries and Map Transformation` | 36 |
| **Working example** — COM application automation + lifecycle | `## TIER 5 — COM Application Automation and Lifecycle Management` | 165 |
| **Working example** — COM event sinking + production wrapper classes | `## TIER 6 — COM Event Sinking and Production Wrapper Classes` | 108 |
| Anti-patterns to avoid | `## ANTI-PATTERNS` | 15 |

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
- Use `ComObject("WinHttp.WinHttpRequest.5.1")` for all HTTP — no invented built-ins
- Always release WinHttp COM handles after use
- Never assume a built-in JSON parser exists in AHK v2 — use Jxon or implement one explicitly
- Wrap all network calls in `try/catch`

### Script Environment
- Directive order: `#Requires` → `#SingleInstance` → `#Include` → script body
- No v1 tray commands — use `A_TrayMenu` and the `Menu` object API
- Admin elevation via `if !A_IsAdmin { Run "*RunAs " A_ScriptFullPath; ExitApp }`
- Register cleanup in `OnExit()` for any resource that must be released on script termination

---

## Reference Files Location

- `references/Module_DllCallAndMemory.md`
- `references/Module_SystemAndCOM.md`
- `references/Module_FileSystem.md`
- `references/Module_NetworkAndHTTP.md`
- `references/Module_ScriptEnvironment.md`
