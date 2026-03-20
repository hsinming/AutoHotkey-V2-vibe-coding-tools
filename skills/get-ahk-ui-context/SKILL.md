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
| `Module_Instructions` | Core validation checklist — **always load first** | `.roo/skills/get-ahk-core-context/references/Module_Instructions.md` |
| `Module_Classes` | All GUI code must be encapsulated in classes — OOP rules apply | `.roo/skills/get-ahk-core-context/references/Module_Classes.md` |
| `Module_DynamicProperties` | Fat arrow callbacks and `.Bind(this)` patterns for event handlers | `.roo/skills/get-ahk-core-context/references/Module_DynamicProperties.md` |
| `Module_Errors` | Error handling for window/control operations and `TargetError` guards | `.roo/skills/get-ahk-logic-context/references/Module_Errors.md` |
| `Module_Validation` | GUI form input validation, `ValidationBuilder` patterns | `.roo/skills/get-ahk-logic-context/references/Module_Validation.md` |

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
   → `.roo/skills/get-ahk-core-context/references/Module_Instructions.md`
2. **Load `Module_Classes`** from `get-ahk-core-context` for any GUI work
   → `.roo/skills/get-ahk-core-context/references/Module_Classes.md`
3. **Load `Module_DynamicProperties`** from `get-ahk-core-context` for event binding
   → `.roo/skills/get-ahk-core-context/references/Module_DynamicProperties.md`
4. **Load the relevant primary module(s)** from this skill (see Module Index above)
5. **Load `Module_Errors` / `Module_Validation`** from `get-ahk-logic-context`
   when handling form input or window/control errors
6. Apply all critical rules before generating code

---

## Reference Files Location

Primary modules in `.roo/skills/get-ahk-ui-context/references/`:

- `.roo/skills/get-ahk-ui-context/references/Module_GUI.md`
- `.roo/skills/get-ahk-ui-context/references/Module_WindowAndControl.md`
- `.roo/skills/get-ahk-ui-context/references/Module_InputAndHotkeys.md`
- `.roo/skills/get-ahk-ui-context/references/Module_GraphicsAndScreen.md`