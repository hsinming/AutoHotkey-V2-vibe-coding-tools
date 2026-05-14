---
name: get-ahk-uia-context
description: >
  Retrieves best-practice context for UI Automation (UIA) in AHK v2: element
  discovery, tree traversal, interaction patterns, staleness handling, and
  browser automation. Must be consulted before writing, modifying, or reviewing
  any AutoHotkey v2 code that uses UIA.ElementFromHandle, FindFirst, FindAll,
  ElementExist, WaitElement, ValuePattern, InvokePattern, or accesses element
  properties (Name, Value, AutomationId, Type). Also triggers for browser
  automation tasks using UIA_Browser, UIA_Chrome, UIA_Edge, UIA_Mozilla,
  Navigate, GetCurrentDocumentElement, WaitPageLoad, GetTab, SelectTab,
  JSExecute, or JSReturnThroughClipboard. Trigger this skill whenever the task
  involves scraping a Windows Forms/WPF/browser UI tree, clicking or reading
  controls via UIA, handling stale element errors, traversing a DataGridView,
  navigating browser tabs, or executing JavaScript via the address bar.
modeSlugs:
  - ahk-code
  - ahk-ask
  - ahk-orchestrator
  - ahk-architect
  - ahk-debug
---

# AHK v2 UIA Context

This skill is the **primary knowledge owner** for UI Automation interaction in this project, covering both native Windows UI trees and browser document automation via UIA_Browser.

---

## How to Use This Skill

Follow this decision tree every time a task triggers this skill. Do **not** skip to code generation before completing the loading steps.

### Step 1 — Identify which domains the task touches

| If the task involves… | Load this module |
|-----------------------|-----------------|
| Element discovery, FindFirst, FindAll, ElementExist, WaitElement, tree traversal, DataGridView traversal, element staleness | `references/Module_UIAElements.md` |
| Clicking, invoking, setting values, reading element text via UIA patterns (InvokePattern, ValuePattern, TogglePattern, etc.) | `references/Module_UIAPatterns.md` |
| Browser automation — Chrome, Edge, Firefox; Navigate, tab management, document element, JSExecute, GetCurrentURL | `references/Module_UIABrowser.md` |

A native-UI task almost always touches both UIAElements and UIAPatterns — load **both**. A browser task almost always touches all three — load **all three**.

### Step 2 — Load cross-skill dependencies

Always load these before writing UIA interaction code:

1. `Module_Instructions` → core v2 validation checklist (get-ahk-core-context)
2. `Module_Classes` → required for all adapter encapsulation (get-ahk-core-context)
3. `Module_Errors` → try/catch patterns for UIA element failures (get-ahk-logic-context)

### Step 3 — Apply Universal Critical Rules

Read the **Universal Critical Rules** section below and confirm every rule applies to the code you are about to write.

### Step 4 — Generate code

Write code only after completing Steps 1–3. Match the TIER level to the complexity of the task.

### Step 5 — Self-check before output

**Native UI (Windows Forms / WPF)**
- [ ] Every public method re-fetches `root` via `UIA.ElementFromHandle(this._hwnd)` at entry — never stored as instance variable
- [ ] `FindFirst` / `FindElement` calls are either inside `try/catch TargetError` or replaced with `ElementExist()` — never guarded with `IsObject()` alone
- [ ] `WaitElement` used instead of `FindFirst` for elements that appear asynchronously
- [ ] All UIA calls are inside `try/catch as err` — never bare
- [ ] HWND obtained via `ahk_id` string; never title strings inside UIA calls
- [ ] No stored element references reused across method boundaries
- [ ] Condition objects use `{}` object literals, NOT `Map()` — UIA library iterates them via `OwnProps()`
- [ ] DataGridView rows iterated via `FindAll({Type: UIA.Type.DataItem}, UIA.TreeScope.Children)`, columns matched by Name substring not position index
- [ ] Pattern availability checked via `element.Is*PatternAvailable` property before accessing pattern object
- [ ] `element.Click()` (smart-action) tried first; `element.InvokePattern.Invoke()` used only when a specific pattern is required
- [ ] `BM_CLICK` fallback uses `element.NativeWindowHandle` and `PostMessage(0x00F5, 0, 0, "ahk_id " hwnd)` — not title strings

**Browser Automation (UIA_Browser)**
- [ ] `GetCurrentDocumentElement()` result stored, then checked with `IsObject(doc)` before any method call — it returns `0` on 3-second timeout and never throws
- [ ] `GetCurrentDocumentElement()` called again after every `Navigate()`, `WaitPageLoad()`, `Back()`, `Forward()`, or tab switch — previous reference is stale
- [ ] `TabExist()` used for conditional tab checks; `GetTab()` used only inside `try/catch TargetError` (it throws on not-found)
- [ ] `WaitPageLoad()` completes before any `JSExecute()` call
- [ ] `WaitTitleChange()` used instead of `WaitPageLoad()` for SPA route changes where Reload button state does not change
- [ ] Stored-callable rule applied: Func/BoundFunc extracted to local variable before `.Call()`

---

## Module Index

| Module | Topic | When to load |
|--------|-------|--------------|
| `Module_UIAElements` | ElementFromHandle, FindFirst, ElementExist, FindAll, WaitElement, tree scope constants, ControlType integers, element staleness, DataGridView traversal | Any element discovery or tree walking against native Windows UI |
| `Module_UIAPatterns` | element.Click() smart-action, InvokePattern, ValuePattern, TogglePattern, ExpandCollapsePattern, SelectionItemPattern, RangeValuePattern, ScrollPattern, NativeWindowHandle, BM_CLICK fallback, pattern-availability guards | Any element interaction (click, type, read value, expand, select) |
| `Module_UIABrowser` | UIA_Browser / UIA_Chrome / UIA_Edge / UIA_Mozilla constructor, GetCurrentDocumentElement lifecycle, Navigate, WaitPageLoad, WaitTitleChange, GetCurrentURL, tab management (GetTab / TabExist / SelectTab / GetTabs), JSExecute, JSReturnThroughClipboard, ClickJSElement, querySelector interaction | Any browser automation task |

### Cross-Skill References

| Module | Reason | Path |
|--------|--------|------|
| `Module_Instructions` | Core v2 validation — always load first | `../get-ahk-core-context/references/Module_Instructions.md` |
| `Module_Classes` | Adapter/class design, `__New`/`__Delete` lifecycle | `../get-ahk-core-context/references/Module_Classes.md` |
| `Module_Errors` | try/catch guards for TargetError and UIA element failures | `../get-ahk-logic-context/references/Module_Errors.md` |

---

## Universal Critical Rules

### Element Staleness (Native UI)
- **Re-fetch `root` at every public method entry** via `UIA.ElementFromHandle(this._hwnd)` — never reuse a stored element across method calls; Windows Forms rebuilds the element tree on any repaint, focus change, or data refresh
- Store `_hwnd` (integer HWND) as the only persistent reference; re-acquire the UIA element on demand at each call site

### Discovery (Native UI)
- **`FindFirst` and `FindElement` throw `TargetError` on not-found** — they do NOT return a falsy value. The only safe non-throwing variants are `ElementExist()` (returns `0`) and `WaitElement()` (returns `0` on timeout). Never guard a `FindFirst` call with `IsObject()` alone
- **Use `ElementExist()` for conditional found/not-found branches**, `WaitElement()` for elements that appear asynchronously (form open, tab switch, async data load), and `try/catch TargetError` only when distinct handling of not-found vs COM error is needed
- Always check `if !el` after `WaitElement()` and `ElementExist()` before calling any method — both return integer `0` on failure
- Condition objects use **`{}` AHK object literals, NOT `Map()`** — the UIA library's `__CreateRawCondition()` iterates them via `OwnProps()`, which silently produces an empty condition for `Map` instances
- Target by `AutomationId` first, then by `Name`, never by position index alone — column positions change across layout variants

### Interaction (Native UI)
- **All UIA calls must be inside `try/catch as err`** — the element tree can change between any two consecutive statements
- **Check pattern availability via `element.Is*PatternAvailable` boolean property** before accessing any pattern object (`element.InvokePattern`, `element.ValuePattern`, etc.) — accessing an unsupported pattern throws a COM Error
- **Use `element.Click()` (empty WhichButton) as the first-choice smart-action** — it tries InvokePattern → TogglePattern → ExpandCollapsePattern → SelectionItemPattern → LegacyIAccessible in sequence and returns the action name; use `element.InvokePattern.Invoke()` only when a specific pattern is explicitly required
- For `ValuePattern.SetValue()`: guard with `element.IsEnabled` AND `!element.ValueIsReadOnly` before calling — read-only controls accept the call silently without changing the value
- **BM_CLICK fallback**: obtain `local hwnd := element.NativeWindowHandle` then `PostMessage(0x00F5, 0, 0, "ahk_id " hwnd)` — never pass a title string or raw integer to the 4th PostMessage argument

### HWND Targeting
- Use `"ahk_id " hwnd` for all window and control identification — never title strings inside UIA calls
- Obtain HWND once via `WinExist("ahk_exe app.exe")` and cache the integer as `this._hwnd`

### Browser Document Lifecycle (UIA_Browser)
- **`GetCurrentDocumentElement()` returns integer `0` on a 3-second internal timeout** — it does NOT throw. Always store the result and check `IsObject(doc)` before calling any method on it; calling any property on integer `0` throws `MethodError`
- **DocumentElement becomes stale after every navigation** — re-call `GetCurrentDocumentElement()` after every `Navigate()`, `WaitPageLoad()`, `Back()`, `Forward()`, `SelectTab()`, or any user-initiated navigation; never reuse a previously stored reference
- **`GetTab()` throws `TargetError` when no matching tab is found** — use `TabExist()` for conditional checks (it returns `0` on not-found); use `GetTab()` only inside `try/catch TargetError`
- **`JSExecute()` navigates the address bar** — never call it while `WaitPageLoad()` is still blocking or during any ongoing navigation; it will cancel the current load
- **`WaitPageLoad()` polls the Reload button state, not DOM readiness** — for SPA route changes where the Reload button does not change state, use `WaitTitleChange()` instead
- Browser automation condition objects follow the same rule as native UIA: `{}` object literals, not `Map()`

### Stored-Callable Rule
- When calling a stored Func/BoundFunc property inside a method, **extract to a local variable first**: `local fn := this._callback` then `fn(args)` — calling `this._callback(args)` directly causes AHK v2 to inject `this` as the first argument silently

---

## Reference Files Location

- `references/Module_UIAElements.md`
- `references/Module_UIAPatterns.md`
- `references/Module_UIABrowser.md`
