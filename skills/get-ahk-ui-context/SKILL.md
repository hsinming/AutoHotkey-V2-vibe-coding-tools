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

### Step 1 — Identify which domains the task touches

| If the task involves… | Load this module |
|-----------------------|-----------------|
| GUI windows, dialogs, controls, layout, event binding | `references/Module_GUI.md` |
| Window detection, HWND, control manipulation, WinEventHook | `references/Module_WindowAndControl.md` |
| Hotkeys, hotstrings, `Send`, `GetKeyState`, input hooks | `references/Module_InputAndHotkeys.md` |
| `PixelSearch`, `ImageSearch`, `CoordMode`, multi-monitor, overlays | `references/Module_GraphicsAndScreen.md` |

A single task may touch multiple domains — load **all** matching modules before writing any code.

### Step 2 — Load cross-skill dependencies

Always load these in order before writing GUI or event-binding code:

1. `Module_Instructions` → core v2 validation checklist (get-ahk-core-context)
2. `Module_Classes` → required for all GUI encapsulation (get-ahk-core-context)
3. `Module_DynamicProperties` → `.Bind(this)` patterns for callbacks (get-ahk-core-context)
4. `Module_Errors` → TargetError / try-catch guards (get-ahk-logic-context) *(if window/control ops present)*
5. `Module_Validation` → form input validation (get-ahk-logic-context) *(if GUI input present)*

### Step 3 — Apply Universal Critical Rules

Read the **Universal Critical Rules** section below and confirm every rule applies to the code you are about to write. Do not proceed if any rule is unclear.

### Step 4 — Generate code

Write code only after completing Steps 1–3. Each module's TIER system determines the appropriate complexity level — match the tier to the task, not the other way around.

### Step 5 — Self-check before output

- [ ] Every GUI class uses `.Bind(this)` on all event callbacks
- [ ] `CoordMode` is declared before any coordinate-dependent call
- [ ] No `#If` / `#IfWin` — only `#HotIf`
- [ ] `ImageSearch` / `PixelSearch` result checked as Boolean, not `ErrorLevel`
- [ ] All window/control ops wrapped in `try/catch` with `TargetError` guard
- [ ] HWNDs stored as integers, not strings

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

Load these from other skills when the task requires them:

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

## Loading Order

1. **Load `Module_Instructions`** from `get-ahk-core-context` first
   → `../get-ahk-core-context/references/Module_Instructions.md`
2. **Load `Module_Classes`** from `get-ahk-core-context` for any GUI work
   → `../get-ahk-core-context/references/Module_Classes.md`
3. **Load `Module_DynamicProperties`** from `get-ahk-core-context` for event binding
   → `../get-ahk-core-context/references/Module_DynamicProperties.md`
4. **Load the relevant primary module(s)** from this skill (see Module Index above)
5. **Load `Module_Errors` / `Module_Validation`** from `get-ahk-logic-context`
   when handling form input or window/control errors
6. Apply all critical rules before generating code

---

## Reference Files Location

Primary modules in `references/` (same directory as this SKILL.md):

- `references/Module_GUI.md`
- `references/Module_WindowAndControl.md`
- `references/Module_InputAndHotkeys.md`
- `references/Module_GraphicsAndScreen.md`