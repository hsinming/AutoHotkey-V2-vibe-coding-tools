---
name: get-ahk-ui-context
description: >
  Retrieves best-practice context for AHK v2 GUI construction, window and
  control interaction, hotkey/input handling, and screen graphics operations.
  Must be consulted before writing, modifying, or reviewing any AutoHotkey v2
  code that involves Gui(), windows, controls, hotkeys, hotstrings, Send, Click,
  pixel/image search, multi-monitor, or screen overlays. Trigger this skill
  whenever the user mentions GUI windows, dialogs, buttons, hotkeys, keyboard
  input, mouse clicks, window detection, CoordMode, PixelSearch, ImageSearch,
  tray popups, or screen drawing — even for casual requests like "make an AHK
  GUI" or "bind a hotkey in AHK". Always load the relevant reference module(s)
  before generating any code.
modeSlugs:
  - ahk-code
  - ahk-ask
  - ahk-orchestrator
  - ahk-architect
  - ahk-debug
---

# AHK v2 UI Context

This skill is the **primary knowledge owner** for AHK v2 user-interface, input,
and screen-graphics work.

---

## How to Use This Skill

Follow this decision tree every time a task triggers this skill. Do **not** skip to code generation before completing the loading steps.

### Step 1 — Map the task to targeted sections via Section Navigator

Do **not** load entire Module files. Use the **Section Navigator** below to find
the exact heading to grep and the number of lines to read. Start with
`## API QUICK-REFERENCE` + `## AHK V2 CONSTRAINTS` for most tasks; descend into
TIER sections only when a runnable code example is needed.

A single task may touch multiple domains — load matching sections from all relevant modules before writing any code.

### Step 2 — Load cross-skill dependencies

Always load these in order before writing GUI or event-binding code:

1. `Module_Instructions` → core v2 validation checklist (get-ahk-core-context) — full file, 431 lines
2. `Module_Classes` → required for all GUI encapsulation (get-ahk-core-context)
3. `Module_DynamicProperties` → `.Bind(this)` patterns for callbacks (get-ahk-core-context)
4. `Module_Errors` → TargetError / try-catch guards (get-ahk-logic-context) *(if window/control ops present)*
5. `Module_Validation` → form input validation (get-ahk-logic-context) *(if GUI input present)*

### Step 3 — Apply Universal Critical Rules

Read the **Universal Critical Rules** section below and confirm every rule applies to the code you are about to write.

### Step 4 — Generate code

Write code only after completing Steps 1–3. Each module's TIER system determines the appropriate complexity level — match the tier to the task.

### Step 5 — Self-check before output

- [ ] Every GUI class uses `.Bind(this)` on all event callbacks
- [ ] `CoordMode` is declared before any coordinate-dependent call
- [ ] No `#If` / `#IfWin` — only `#HotIf`
- [ ] `ImageSearch` / `PixelSearch` result checked as Boolean, not `ErrorLevel`
- [ ] All window/control ops wrapped in `try/catch` with `TargetError` guard
- [ ] HWNDs stored as integers, not strings

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

### GUI Windows & Controls (Module_GUI.md — 1176 lines)

| Task | Grep for heading | ~Lines |
|------|-----------------|--------|
| All GUI API: Gui constructor, Add methods, control methods, ListView, TreeView | `## API QUICK-REFERENCE` | 75 |
| `Gui()` constructor + window-level methods (`Show`, `Hide`, `Destroy`, `Submit`) | `### Gui Constructor and Window Methods` | 15 |
| Control add methods (`Add("Edit", …)`, `Add("Button", …)`, …) | `### Gui Control Add Methods` | 21 |
| Control object methods + properties (`.Value`, `.Focus()`, `.OnEvent()`) | `### Control Object Methods and Properties` | 10 |
| ListView methods (`Add`, `Delete`, `Modify`, `GetNext`, column setup) | `### ListView Control Methods` | 11 |
| TreeView methods (`Add`, `Delete`, `Modify`, `GetSelection`) | `### TreeView Control Methods` | 11 |
| v1 → v2 breaking changes (GUI) | `## V1 → V2 BREAKING CHANGES` | 15 |
| Constraints: class encapsulation, `.Bind(this)` callbacks, v1 g-label removal | `## AHK V2 CONSTRAINTS` | 22 |
| **Working example** — basic GUI creation | `## TIER 1 — Basic GUI Creation` | 26 |
| **Working example** — controls, events, ListView, TreeView, multi-window | `## TIER 2 — Controls, Event Handling, ListView, TreeView, Multi-Window` | 322 |
| **Working example** — layout + positioning management | `## TIER 3 — Layout and Positioning Management` | 172 |
| **Working example** — mathematical layout system | `## TIER 4 — Mathematical Layout System` | 167 |
| **Working example** — GuiForm mathematical positioning (TIER 5+) | `## TIER 5 — GuiForm Mathematical Positioning` | 138 |
| **Working example** — advanced layout compositions | `## TIER 6 — Advanced Layout Compositions` | 204 |
| Anti-patterns to avoid | `## ANTI-PATTERNS` | 17 |

---

### Window & Control Interaction (Module_WindowAndControl.md — 823 lines)

| Task | Grep for heading | ~Lines |
|------|-----------------|--------|
| All window/control API: state, metadata, mutation, coordinate, control data | `## API QUICK-REFERENCE` | 91 |
| Window state: `WinExist`, `WinWait`, `WinActive`, `WinGetProcessName` | `### Window State Functions` | 17 |
| Window metadata: `WinGetTitle`, `WinGetPos`, `WinGetClass`, `WinGetPID` | `### Window Metadata Functions` | 13 |
| Window mutation: `WinMove`, `WinSetAlwaysOnTop`, `WinSetTransparent` | `### Window Mutation Functions` | 9 |
| Title matching modes (`ahk_id`, `ahk_class`, `ahk_exe`, `ahk_group`) | `### Title Matching` | 5 |
| `GroupAdd` / `GroupActivate` window groups | `### Window Groups` | 8 |
| `CoordMode` — screen, window, client coordinate systems | `### Coordinate Mode` | 5 |
| `ControlClick`, `ControlSend`, `ControlFocus` | `### Control Interaction` | 11 |
| `ControlGetText`, `ControlGetPos`, `ControlGetValue` | `### Control Data` | 12 |
| `SetWinEventHook` via DllCall | `### Hook and Callback (DllCall)` | 9 |
| v1 → v2 breaking changes | `## V1 → V2 BREAKING CHANGES` | 12 |
| Constraints: TargetError guards, HWND as integer, CoordMode declaration | `## AHK V2 CONSTRAINTS` | 26 |
| **Working example** — basic state checks + safety guards | `## TIER 1 — Basic State Checks and Safety Guards` | 64 |
| **Working example** — window metadata queries | `## TIER 2 — Window Metadata Queries` | 67 |
| **Working example** — CoordMode + basic control commands | `## TIER 3 — Coordinate System Declaration and Basic Control Commands` | 73 |
| **Working example** — advanced control data interaction | `## TIER 4 — Advanced Control Data Interaction` | 97 |
| **Working example** — window groups, batch lifecycle, OOP wrapper | `## TIER 5 — Window Groups, Batch Lifecycle, and OOP Wrapper` | 235 |
| **Working example** — SetWinEventHook via DllCall | `## TIER 6 — System-Level Window Hooks via DllCall + SetWinEventHook` | 127 |
| Anti-patterns to avoid | `## ANTI-PATTERNS` | 12 |

---

### Hotkeys, Hotstrings & Input (Module_InputAndHotkeys.md — 506 lines)

| Task | Grep for heading | ~Lines |
|------|-----------------|--------|
| All input/hotkey API: Send functions, Click, Hotkey(), InputHook | `## API QUICK-REFERENCE` | 65 |
| `Send` / `SendInput` / `SendEvent` / `SendPlay` differences | `### Send Functions` | 10 |
| `Click` / `MouseMove` / `MouseClick` / `MouseClickDrag` | `### Click and Mouse` | 6 |
| `Hotkey()` function: dynamic binding, enable/disable | `### Hotkey Control` | 8 |
| `GetKeyState()` / `KeyWait()` | `### Key State and Wait` | 6 |
| `InputHook` object: constructor, `.Start()`, `.Stop()`, `.OnEnd` | `### InputHook Object` | 11 |
| Built-in variables (`A_PriorKey`, `A_TimeSinceThisHotkey`, …) | `### Built-in Variables` | 9 |
| v1 → v2 breaking changes | `## V1 → V2 BREAKING CHANGES` | 15 |
| Constraints: `#HotIf` only, `Hotkey()` needs callback not string-label | `## AHK V2 CONSTRAINTS` | 27 |
| **Working example** — static hotkeys, hotstrings, Send fundamentals | `## TIER 1 — Static Hotkeys, Hotstrings, and Send Fundamentals` | 62 |
| **Working example** — dynamic hotkeys with `Hotkey()` | `## TIER 2 — Dynamic Hotkeys with Hotkey()` | 30 |
| **Working example** — key state queries + wait operations | `## TIER 3 — Key State Queries and Wait Operations` | 32 |
| **Working example** — context-sensitive routing with `#HotIf` + `HotIf()` | `## TIER 4 — Context-Sensitive Routing with #HotIf and HotIf()` | 70 |
| **Working example** — InputHook + custom modifier chords | `## TIER 5 — InputHook and Custom Modifier Chords` | 94 |
| **Working example** — class-encapsulated hotkey management | `## TIER 6 — Class-Encapsulated Hotkey Management` | 76 |
| Anti-patterns to avoid | `## ANTI-PATTERNS` | 15 |

---

### Graphics & Screen (Module_GraphicsAndScreen.md — 708 lines)

| Task | Grep for heading | ~Lines |
|------|-----------------|--------|
| All graphics API: CoordMode, pixel, ImageSearch, monitor, overlay | `## API QUICK-REFERENCE` | 49 |
| `CoordMode` modes + `A_ScreenWidth` / `A_ScreenHeight` | `### CoordMode and Screen Dimensions` | 8 |
| `MouseMove` / `Click` with coordinate modes | `### Mouse and Click` | 7 |
| `PixelGetColor` / `PixelSearch` | `### Pixel Operations` | 6 |
| `ImageSearch` API + image file options | `### Image Search` | 5 |
| `WinGetPos` / `WinGetClientPos` window position | `### Window Position` | 6 |
| `MonitorGet` / `MonitorGetCount` / `MonitorGetPrimary` | `### Monitor Functions` | 8 |
| `WinSetTransparent` / overlay window creation | `### Overlay / Window Transparency` | 7 |
| v1 → v2 breaking changes | `## V1 → V2 BREAKING CHANGES` | 12 |
| Constraints: `ImageSearch` returns Boolean not ErrorLevel, CoordMode required | `## AHK V2 CONSTRAINTS` | 30 |
| **Working example** — basic screen information | `## TIER 1 — Basic Screen Information` | 45 |
| **Working example** — pixel and color detection | `## TIER 2 — Pixel and Color Detection` | 59 |
| **Working example** — ImageSearch fundamentals | `## TIER 3 — ImageSearch Fundamentals` | 75 |
| **Working example** — multi-monitor + work-area calculation | `## TIER 4 — Multi-Monitor and Work-Area Calculation` | 120 |
| **Working example** — dynamic resolution adaptation | `## TIER 5 — Dynamic Resolution Adaptation` | 162 |
| **Working example** — visual debug overlays | `## TIER 6 — Visual Debug Overlays` | 127 |
| Anti-patterns to avoid | `## ANTI-PATTERNS` | 12 |

---

## Module Index

### Primary Modules (owned by this skill)

| Module | Topic | When to load |
|--------|-------|--------------|
| `Module_GUI` | `Gui()` constructor, control layout, GuiForm mathematical positioning system, event binding, complexity tiers (TIER 1–5+) | Any GUI window, dialog, or control creation |
| `Module_WindowAndControl` | Window detection (`ahk_id`, `ahk_class`, `ahk_exe`), control manipulation, HWND handling, CoordMode, TargetError guards, SetWinEventHook | Window management, control interaction, system monitoring |
| `Module_InputAndHotkeys` | Static/dynamic hotkeys, `#HotIf`, `Hotkey()` function, hotstrings, `Send`/`SendInput`/`SendEvent`, `GetKeyState`, `KeyWait`, input hooks | Hotkeys, hotstrings, keyboard/mouse input, context-aware bindings |
| `Module_GraphicsAndScreen` | `CoordMode`, `PixelGetColor`, `PixelSearch`, `ImageSearch`, multi-monitor, DPI-aware scaling, transparent debug overlays | Pixel/image recognition, screen coordinates, multi-monitor, visual overlays |

### Cross-Skill References

| Module | Reason to load | Path |
|--------|---------------|------|
| `Module_Instructions` | Core validation checklist — **always load first** | `../get-ahk-core-context/references/Module_Instructions.md` |
| `Module_Classes` | All GUI code must be encapsulated in classes — OOP rules apply | `../get-ahk-core-context/references/Module_Classes.md` |
| `Module_DynamicProperties` | Fat arrow callbacks and `.Bind(this)` patterns for event handlers | `../get-ahk-core-context/references/Module_DynamicProperties.md` |
| `Module_Errors` | Error handling for window/control operations and `TargetError` guards | `../get-ahk-logic-context/references/Module_Errors.md` |
| `Module_Validation` | GUI form input validation, `ValidationBuilder` patterns | `../get-ahk-logic-context/references/Module_Validation.md` |

---

## Universal Critical Rules

### GUI
- Always use `Gui()` constructor — never legacy v1 GUI syntax
- ALL GUI code must be encapsulated in classes
- Use the tier system: select the appropriate complexity tier before writing any layout code
- Apply the GuiForm mathematical positioning system for explicit coordinate control (TIER 5+)
- All event callbacks must use `.Bind(this)` for correct object context

### Window & Control
- Always store HWNDs as integers, never as strings
- Declare `CoordMode` explicitly before any coordinate-dependent operation
- Use `ahk_id`, `ahk_class`, `ahk_exe`, or `ahk_group` for reliable window identification
- Wrap all window/control operations in `try/catch` with `TargetError` guards

### Input & Hotkeys
- Multi-line hotkey/hotstring actions always use function block braces `{}`
- Use `#HotIf` exclusively — `#If` and `#IfWin` are v1 legacy; never use them
- `Hotkey()` function requires a callback function object, never a string label
- `Send()` defaults to `SendInput` in v2; use `SendEvent()` only when key delays are required
- Key names in `GetKeyState()` and `KeyWait()` must be quoted strings

### Graphics & Screen
- `ImageSearch` and `PixelSearch` return Boolean in v2 — never check `ErrorLevel`
- Always set `CoordMode` before any pixel/image/mouse coordinate operation
- Apply tolerance tuning for `PixelSearch` in varied display environments
- Scale coordinates with DPI-aware math for multi-monitor setups

---

## Reference Files Location

- `references/Module_GUI.md`
- `references/Module_WindowAndControl.md`
- `references/Module_InputAndHotkeys.md`
- `references/Module_GraphicsAndScreen.md`
