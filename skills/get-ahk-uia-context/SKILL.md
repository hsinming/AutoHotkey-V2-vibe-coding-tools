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

Follow this protocol every time a task triggers this skill. Do **not** skip to code generation before completing the loading steps.

### Step 1 — Identify task type and load targeted sections via Section Navigator

Do **not** load entire Module files. Use the **Section Navigator** below to find the exact heading to grep and the number of lines to read. Start with the **Fast Path** rows; descend into TIER sections only when you need a runnable code example.

A native-UI task almost always requires both a discovery section (UIAElements) and an interaction section (UIAPatterns). A browser task needs UIABrowser. Load only the sections that match your task.

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

## Section Navigator

> **How to use:**
> 1. Find the row matching your task below.
> 2. `grep -n "^## heading text"` the target file to get the current line number.
> 3. `Read(file, offset=<line>, limit=<n>)` — read only that section.
>
> **Fast Path first:** the `## API QUICK-REFERENCE` and `## AHK V2 CONSTRAINTS` sections
> cover ~80 % of tasks in ~100–150 lines. Load a TIER section only when you need
> a complete, runnable code example.

---

### Fast Path — API signatures + constraints (no full-file read needed)

| Task | File | Grep for heading | ~Lines |
|------|------|-----------------|--------|
| All pattern API signatures (Click, InvokePattern, ValuePattern, …) | `Module_UIAPatterns.md` | `## API QUICK-REFERENCE` | 100 |
| All element-discovery API (FindFirst, ElementExist, WaitElement, …) | `Module_UIAElements.md` | `## API QUICK-REFERENCE` | 151 |
| All browser API signatures (Navigate, GetCurrentDocumentElement, …) | `Module_UIABrowser.md` | `## API QUICK-REFERENCE` | 77 |
| Constraints & pitfalls — patterns | `Module_UIAPatterns.md` | `## AHK V2 CONSTRAINTS` | 31 |
| Constraints & pitfalls — elements | `Module_UIAElements.md` | `## AHK V2 CONSTRAINTS` | 49 |
| Constraints & pitfalls — browser | `Module_UIABrowser.md` | `## AHK V2 CONSTRAINTS` | 39 |
| v1 → v2 breaking changes (patterns) | `Module_UIAPatterns.md` | `## V1 → V2 BREAKING CHANGES` | 12 |
| v1 → v2 breaking changes (elements) | `Module_UIAElements.md` | `## V1 → V2 BREAKING CHANGES` | 14 |
| v1 → v2 breaking changes (browser) | `Module_UIABrowser.md` | `## V1 → V2 BREAKING CHANGES` | 13 |
| QA checklist before submitting UIA code | `Module_UIAPatterns.md` | `## AGENT QA CHECKLIST` | 7 |
| Runtime error → probable cause mapping | `Module_UIAPatterns.md` | `## RUNTIME ERROR MAPPING` | 8 |

---

### Native UI — Element Discovery & Waiting

| Task | File | Grep for heading | ~Lines |
|------|------|-----------------|--------|
| Get root from HWND (`UIA.ElementFromHandle`) | `Module_UIAElements.md` | `### Root & Window Acquisition` | 8 |
| `FindFirst` / `FindElement` — throwing variants | `Module_UIAElements.md` | `### Element Discovery (Throwing Variants)` | 7 |
| `ElementExist` / `WaitElement` — safe non-throwing variants | `Module_UIAElements.md` | `### Element Discovery (Safe Non-Throwing Variants)` | 7 |
| Navigate tree by path (`GetChildAt`, `GetParent`, …) | `Module_UIAElements.md` | `### Element Navigation & Path Traversal` | 7 |
| Read element properties (Name, Value, AutomationId, Type, …) | `Module_UIAElements.md` | `### Element Properties` | 27 |
| Debug element tree (`Dump`, `GetUIATreeAsString`) | `Module_UIAElements.md` | `### Element Actions & Debugging` | 10 |
| Tree scope constants (`UIA.TreeScope.*`) | `Module_UIAElements.md` | `### Tree Scope Constants` | 11 |
| ControlType integer constants (`UIA.Type.*`) | `Module_UIAElements.md` | `### ControlType Integer Constants (UIA.Type.*)` | 39 |
| Staleness rules + re-fetch patterns (full rules) | `Module_UIAElements.md` | `## AHK V2 CONSTRAINTS` | 49 |
| **Working example** — root acquisition + safe discovery | `Module_UIAElements.md` | `## TIER 1 — Root Acquisition and Safe Element Discovery` | 47 |
| **Working example** — nested FindFirst + preflight validation | `Module_UIAElements.md` | `## TIER 2 — Nested FindFirst and Preflight Validation` | 26 |
| **Working example** — WaitElement for async elements | `Module_UIAElements.md` | `## TIER 3 — WaitElement for Async Elements` | 36 |
| **Working example** — multi-window HWND resolution + UIA.Filter | `Module_UIAElements.md` | `## TIER 5 — Multi-Window HWND Resolution and UIA.Filter` | 53 |

---

### Native UI — Element Interaction (click, type, toggle, expand, scroll, grid)

| Task | File | Grep for heading | ~Lines |
|------|------|-----------------|--------|
| `element.Click()` smart-action (API signature) | `Module_UIAPatterns.md` | `### element.Click() — Universal Smart Action` | 4 |
| Check which patterns an element supports | `Module_UIAPatterns.md` | `### Pattern Availability Checks (on element)` | 13 |
| `InvokePattern.Invoke()` (API) | `Module_UIAPatterns.md` | `### InvokePattern` | 4 |
| `ValuePattern.SetValue()` / `.Value` (API) | `Module_UIAPatterns.md` | `### ValuePattern` | 7 |
| `TogglePattern.Toggle()` / `.ToggleState` (API) | `Module_UIAPatterns.md` | `### TogglePattern` | 5 |
| `ExpandCollapsePattern.Expand()` / `.Collapse()` (API) | `Module_UIAPatterns.md` | `### ExpandCollapsePattern` | 6 |
| `SelectionItemPattern.Select()` / `.IsSelected` (API) | `Module_UIAPatterns.md` | `### SelectionItemPattern` | 7 |
| Slider / spin box / progress bar (`RangeValuePattern`) | `Module_UIAPatterns.md` | `### RangeValuePattern (Slider, SpinBox, ProgressBar)` | 7 |
| `ScrollItemPattern.ScrollIntoView()` (API) | `Module_UIAPatterns.md` | `### ScrollItemPattern` | 4 |
| Scroll a container (`ScrollPattern`) | `Module_UIAPatterns.md` | `### ScrollPattern (on container)` | 6 |
| Read DataGrid cell value (`GridItemPattern`) | `Module_UIAPatterns.md` | `### GridItemPattern (on cell within DataGrid)` | 6 |
| Window-level operations (`WindowPattern`) | `Module_UIAPatterns.md` | `### WindowPattern` | 6 |
| BM_CLICK / PostMessage fallback (API) | `Module_UIAPatterns.md` | `### NativeWindowHandle (WM_* Fallback)` | 4 |
| `UIA.Pattern.*` integer constants | `Module_UIAPatterns.md` | `### Pattern Constants (UIA.Pattern.*)` | 17 |
| **Working example** — Click smart-action + InvokePattern | `Module_UIAPatterns.md` | `## TIER 1 — element.Click() Smart Action and InvokePattern` | 47 |
| **Working example** — ValuePattern text input + RangeValuePattern | `Module_UIAPatterns.md` | `## TIER 2 — ValuePattern Text Input and RangeValuePattern` | 65 |
| **Working example** — NativeWindowHandle BM_CLICK fallback | `Module_UIAPatterns.md` | `## TIER 3 — NativeWindowHandle Fallback (BM_CLICK / WM_*)` | 47 |
| **Working example** — Toggle, ExpandCollapse, SelectionItem | `Module_UIAPatterns.md` | `## TIER 4 — Toggle, ExpandCollapse, and SelectionItem Patterns` | 69 |
| **Working example** — Scroll, Grid, Window patterns | `Module_UIAPatterns.md` | `## TIER 5 — Scroll, Grid, and Window Patterns` | 78 |

---

### Native UI — DataGridView Traversal

For DataGridView tasks load **both** files: UIAElements for row discovery, UIAPatterns for reading cell values.

| Task | File | Grep for heading | ~Lines |
|------|------|-----------------|--------|
| **Working example** — traverse rows + match columns by Name | `Module_UIAElements.md` | `## TIER 4 — DataGridView Traversal by Name Pattern` | 51 |
| Read a cell's value via GridItemPattern | `Module_UIAPatterns.md` | `### GridItemPattern (on cell within DataGrid)` | 6 |
| SelectionItemPattern for grid row selection | `Module_UIAPatterns.md` | `### SelectionItemPattern` | 7 |

---

### Native UI — Drop-in Recipes & Anti-patterns

| Task | File | Grep for heading | ~Lines |
|------|------|-----------------|--------|
| Copy-paste working snippets (complete adapter patterns) | `Module_UIAElements.md` | `## DROP-IN RECIPES` | 61 |
| Copy-paste working snippets (pattern interactions) | `Module_UIAPatterns.md` | `## DROP-IN RECIPES` | 59 |
| Anti-patterns to avoid | `Module_UIAElements.md` | `## ANTI-PATTERNS` | 12 |
| Anti-patterns to avoid | `Module_UIAPatterns.md` | `## ANTI-PATTERNS` | 12 |

---

### Browser Automation (UIA_Browser)

| Task | File | Grep for heading | ~Lines |
|------|------|-----------------|--------|
| Browser instance properties + constructor signatures | `Module_UIABrowser.md` | `### Constructors` | 5 |
| `GetCurrentDocumentElement()` lifecycle rules (API + return type) | `Module_UIABrowser.md` | `### Document & Navigation` | 11 |
| Tab management API (`GetTab`, `TabExist`, `SelectTab`, `GetTabs`) | `Module_UIABrowser.md` | `### Tab Management` | 10 |
| JavaScript interop (`JSExecute`, `JSReturnThroughClipboard`) | `Module_UIABrowser.md` | `### JavaScript Interop` | 11 |
| Content extraction (`GetCurrentURL`, text) | `Module_UIABrowser.md` | `### Content Extraction` | 5 |
| Alert / dialog handling | `Module_UIABrowser.md` | `### Alerts & Dialogs` | 5 |
| Browser constraints + stale-document rules (full) | `Module_UIABrowser.md` | `## AHK V2 CONSTRAINTS` | 39 |
| **Working example** — constructor + auto-detection + proxy | `Module_UIABrowser.md` | `## TIER 1 — Constructor, Auto-Detection, and BrowserElement Proxy` | 37 |
| **Working example** — document lifecycle + page load sync | `Module_UIABrowser.md` | `## TIER 2 — Document Element Lifecycle and Page Load Synchronisation` | 52 |
| **Working example** — navigate + URL management + GetCurrentURL | `Module_UIABrowser.md` | `## TIER 3 — Navigation Controls, URL Management, and GetCurrentURL` | 42 |
| **Working example** — tab management (open, switch, close) | `Module_UIABrowser.md` | `## TIER 4 — Tab Management` | 59 |
| **Working example** — JSExecute + querySelector interaction | `Module_UIABrowser.md` | `## TIER 5 — JavaScript Interop and querySelector Interaction` | 67 |
| Copy-paste browser automation snippets | `Module_UIABrowser.md` | `## DROP-IN RECIPES` | 45 |
| Browser anti-patterns | `Module_UIABrowser.md` | `## ANTI-PATTERNS` | 12 |

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

These rules are inlined here so they are available at zero cost — no Module read needed for the common cases.

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
