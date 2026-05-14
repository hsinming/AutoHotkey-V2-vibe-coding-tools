# Module_UIAElements.md
<!-- DOMAIN: UI Automation Element Discovery and Tree Traversal -->
<!-- SCOPE: Control interaction patterns (Invoke, ValuePattern, Toggle, ExpandCollapse) and UIA event handlers are not covered — see Module_UIAPatterns.md -->
<!-- TRIGGERS: UIA.ElementFromHandle, FindFirst, FindAll, FindElement, ElementExist, WaitElement, WalkTree, AutomationId, TreeScope, ControlType, DataGridView, element staleness, "find element", "wait for element", "traverse tree", "automation element", "UI tree", "accessibility tree" -->
<!-- CONSTRAINTS: FindFirst and FindElement THROW TargetError when no element is found — never use IsObject() guard without an outer try/catch; use ElementExist() (returns 0 on failure) for safe non-throwing checks. Re-fetch root via UIA.ElementFromHandle() at every public method entry — UIA elements become stale after any Windows Forms repaint. Condition objects use {} object literals (not Map()) because UIA.__CreateRawCondition() calls OwnProps() on them. -->
<!-- CROSS-REF: Module_UIAPatterns.md, Module_UIABrowser.md, Module_Classes.md, Module_Errors.md, Module_WindowAndControl.md -->
<!-- VERSION: AHK v2.0+ with Lib/UIA.ahk v1.1.3 (Descolada w32 UIA wrapper) -->

## V1 → V2 BREAKING CHANGES

| v1 pattern (LLM commonly writes) | v2 correct form | Consequence |
|----------------------------------|-----------------|-------------|
| `new UIA.ElementFromHandle(hwnd)` | `UIA.ElementFromHandle(hwnd)` — no `new` keyword | AHK v2 has no `new` keyword; `new` is treated as an undefined identifier and throws `NameError` at runtime |
| `element.FindFirst(condition)` then `if !result` | `try { element.FindFirst(condition) } catch TargetError` OR `element.ElementExist(condition)` | FindFirst THROWS TargetError on not-found; IsObject/falsy check is dead code before the throw |
| `element.FindFirst({Type: 50029}, 4)` on large DataGridView | `element.FindAll({Type: 50029}, UIA.TreeScope.Children)` (scope=2) | Default scope 4 = Descendants recurses into cells, doubling results and causing multi-second hangs |
| Storing `this._element := root.FindFirst(...)` across method calls | Store `this._hwnd` (Integer) only; re-acquire via `UIA.ElementFromHandle(this._hwnd)` at each entry | Windows Forms rebuilds the element tree on any repaint or data refresh; stored element becomes stale COM reference |
| `catch err` (no `as`) | `catch as err` | AHK v2 requires `as` keyword to bind the error object; `catch err` is a syntax error |
| `UIA.TreeScope.Subtree` to mean scope=4 (all descendants) | `UIA.TreeScope.Descendants` (=4) for all descendants; `UIA.TreeScope.Subtree` (=7) includes the element itself | Subtree=7 includes the root element in results — unexpected extra match at index 1 |
| `{key: "val"}` as Map for data storage | `Map("key", "val")` for key-value data | AHK object literals `{}` work only as UIA condition descriptors (UIA library reads them via OwnProps()); using them as general containers silently loses .Has()/.Delete() |
| Calling `FindFirst`/`FindAll` with `scope=TreeScope.Ancestors` or `TreeScope.Parent` to search upward | Use `element.Parent` or `element.WalkTree("p")` for parent navigation | Microsoft UIA explicitly does not support searching for ancestor elements via Find methods — behavior is undefined or silently empty |
| `UIA.GetRootElement().FindAll(cond, TreeScope.Subtree)` to find all windows | `UIA.GetRootElement().FindAll(cond, UIA.TreeScope.Children)` | Subtree search on the desktop root can iterate thousands of elements — Microsoft recommends searching only direct children of root |

## API QUICK-REFERENCE

> Method count exceeds 20 — structured bulleted list format used.

### Root & Window Acquisition

- **UIA.ElementFromHandle()** — Signature: `UIA.ElementFromHandle(hwnd:="", cacheRequest?, activateChromiumAccessibility:=500)` | Returns: `UIA.IUIAutomationElement` | Throws: `TargetError` if hwnd resolves to no window, `UnsetError` if COM returns null | Notes: hwnd can be Integer or WinTitle string (resolved via WinExist); call at every public method entry
- **UIA.ElementFromWindow()** — Signature: `UIA.ElementFromWindow(WinTitle:="", ...)` | Returns: element | Throws: TargetError | Notes: Alias for ElementFromHandle; accepts WinTitle string directly
- **UIA.ElementFromPoint()** — Signature: `UIA.ElementFromPoint(x?, y?, cacheRequest?, activateChromiumAccessibility:=500)` | Returns: element | Throws: TargetError | Notes: Omit x/y to use current cursor position
- **UIA.GetRootElement()** — Signature: `UIA.GetRootElement(cacheRequest?)` | Returns: desktop root element | Throws: — | Notes: Returns the virtual desktop root; use for top-level window enumeration
- **UIA.GetFocusedElement()** — Signature: `UIA.GetFocusedElement(cacheRequest?)` | Returns: element with keyboard focus | Throws: — | Notes: Returns current focused element across all windows

### Element Discovery (Throwing Variants)

- **element.FindFirst()** — Signature: `element.FindFirst(condition:=UIA.TrueCondition, scope:=4)` | Returns: `UIA.IUIAutomationElement` | Throws: **TargetError** when no element matches | Notes: Default scope 4 = Descendants; ALWAYS wrap in try/catch or replace with ElementExist
- **element.FindAll()** — Signature: `element.FindAll(condition:=UIA.TrueCondition, scope:=4)` | Returns: `Array` of elements (empty `[]` when none match) | Throws: `TypeError` if condition is not an object | Notes: Safe to iterate without guard; check `.Length` before processing
- **element.FindElement()** — Signature: `element.FindElement(condition, scope:=4, index:=1, order:=0, startingElement:=0, cacheRequest:=0)` | Returns: element | Throws: **TargetError** | Notes: Enhanced version supporting nth-match (`index:=2`), reverse order, named params in condition object; ALWAYS wrap in try/catch
- **element.FindElements()** — Signature: `element.FindElements(condition, scope:=4, order:=0, startingElement:=0, cacheRequest:=0)` | Returns: `Array` (empty on no match) | Throws: — | Notes: Enhanced FindAll supporting order and startingElement; safe like FindAll

### Element Discovery (Safe Non-Throwing Variants)

- **element.ElementExist()** — Signature: `element.ElementExist(condition, scope:=4, index:=1, order:=0, startingElement:=0, cacheRequest:=0)` | Returns: element on success, `0` (integer) on not-found | Throws: — | Notes: Preferred over FindFirst when you need a falsy check; wraps FindElement in try/catch internally
- **element.WaitElement()** — Signature: `element.WaitElement(condition, timeOut:=-1, scope:=4, index:=1, order:=0, startingElement:=0, cacheRequest:=0, tick:=20)` | Returns: element on found, `0` on timeout | Throws: — | Notes: timeOut=-1 waits indefinitely; tick=20ms poll interval; use for elements that appear asynchronously
- **element.WaitElementNotExist()** — Signature: `element.WaitElementNotExist(condition, timeout:=-1, scope:=4, ...)` | Returns: `1` if element disappeared, `0` if timed out | Throws: — | Notes: Use after invoking an action to confirm the element was removed
- **element.WaitNotExist()** — Signature: `element.WaitNotExist(timeOut:=-1, tick:=20)` | Returns: `1` if this element disappeared | Throws: — | Notes: Called on the element itself; use after navigating away

### Element Navigation & Path Traversal

- **element.ElementFromPath()** — Signature: `element.ElementFromPath(paths*)` | Returns: element | Throws: `IndexError` | Notes: Accepts numeric path `"3,2"`, type path `"Button3,2"`, UIA encoded path, or condition chain; shorthand via `element[path]`
- **element.WaitElementFromPath()** — Signature: `element.WaitElementFromPath(paths*)` | Returns: element or nothing on timeout | Throws: — | Notes: Append integer timeOut and optional tick as last args; loops until path resolves
- **element.WalkTree()** — Signature: `element.WalkTree(searchPath, filterCondition?)` | Returns: element | Throws: `IndexError`, `ValueError` | Notes: searchPath is comma-separated: `"n"` = nth child, `"+n"` = nth next sibling, `"-n"` = nth prev sibling, `"pn"` = nth parent, `"\"n\""` = normalize
- **element.GetChildren()** — Signature: `element.GetChildren(c?, scope:=2)` | Returns: `Array` | Throws: — | Notes: Alias for FindAll with default Children scope; optionally accepts filter condition

### Element Properties

- **element.Name** — Type: String | Notes: Human-readable label; stable for column headers; may be empty for unlabeled controls
- **element.Value** — Type: String (get/set) | Notes: Reads ValuePattern, then RangeValuePattern, then LegacyIAccessiblePattern; throws Error if none available
- **element.AutomationId** — Type: String | Notes: Developer-assigned ID; most stable identifier across sessions — prefer over Name for fixed controls
- **element.Type** — Type: Integer | Notes: UIA ControlType integer; alias `ControlType`; use `UIA.Type.*` constants (e.g. `UIA.Type.Button = 50000`)
- **element.IsEnabled** — Type: Boolean | Notes: False if control is greyed out or disabled
- **element.IsOffscreen** — Type: Boolean | Notes: True if element is outside visible viewport; used by Exists property
- **element.BoundingRectangle** — Type: Object `{l, t, r, b}` | Notes: Screen coordinates; zero-rect for offscreen elements
- **element.Location** — Type: Object `{x, y, w, h}` | Notes: Convenience alias derived from BoundingRectangle; x=l, y=t, w=r-l, h=b-t
- **element.Exists** — Type: Integer 0/1 | Notes: Returns 0 if BoundingRectangle is zero or IsOffscreen; use to poll for element visibility
- **element.Children** — Type: Array | Notes: Direct children only; equivalent to FindAll(TrueCondition, Children); cached variant: CachedChildren
- **element.Parent** — Type: element | Notes: Returns parent via TreeWalkerTrue.GetParentElement; throws on desktop root
- **element.Length** — Type: Integer | Notes: Count of direct children; calls GetChildren() internally
- **element.WinId** — Type: Integer HWND | Notes: HWND of the containing top-level window; computed once and cached on element
- **element.ControlId** — Type: Integer HWND | Notes: HWND of the specific control hosting the element
- **element.RuntimeId** — Type: String | Notes: String representation of COM RuntimeId; NOT stable across sessions or reboots; for in-session identity comparison only
- **element.SetFocus()** — Signature: `element.SetFocus()` | Returns: — | Throws: COM error | Notes: Sets keyboard focus to this element; may activate parent window
- **element.ClassName** — Type: String | Notes: Implementation-specific class name (e.g. `"WindowsForms10.BUTTON.app.0...."` for WinForms); use `FrameworkId` to identify the UI framework; not always stable across application versions
- **element.FrameworkId** — Type: String | Notes: UI framework name: `"Win32"`, `"WinForm"`, `"WPF"`, `"DirectUI"`, `"Chrome"`, `"Edge"`; use to discriminate Windows Forms controls from WPF when tree structure differs
- **element.HasKeyboardFocus** — Type: Boolean | Notes: True if this specific element currently has keyboard focus; use to confirm focus after `SetFocus()` call
- **element.IsKeyboardFocusable** — Type: Boolean | Notes: True if the control can accept keyboard focus; check before calling `SetFocus()` to avoid silent no-ops
- **element.HelpText** — Type: String | Notes: Tooltip or placeholder text (e.g. "Type text here to search"); useful when Name is empty but HelpText disambiguates controls
- **element.IsContentElement** — Type: Boolean | Notes: True if element is in the Content tree view — conveys actual information to the user; filter with `{IsContentElement: True}` to exclude decorative elements
- **element.IsControlElement** — Type: Boolean | Notes: True if element is in the Control tree view — interactive or structural UI element; most elements are True; layout-only panels are False
- **element.ProcessId** — Type: Integer | Notes: PID of the process owning the element; use to correlate elements with `WinGetPID` results when multiple processes share similar UIA trees

### Element Actions & Debugging

- **element.Click()** — Signature: `element.Click(WhichButton:="", ClickCount:=1, DownOrUp:="", Relative:="", NoActivate:=False, MoveBack?)` | Returns: action name or 0 | Throws: — | Notes: Empty WhichButton tries InvokePattern, Toggle, ExpandCollapse, SelectionItem in order; `"left"`/`"right"` uses AHK Click() at element center
- **element.ControlClick()** — Signature: `element.ControlClick(WhichButton:="left", ClickCount:=1, Options:="", WinTitle?)` | Returns: — | Throws: — | Notes: Uses ControlClick at client-relative coordinates; more reliable than Click() for partially obscured controls
- **element.GetPos()** — Signature: `element.GetPos(relativeTo:="", WinTitle?)` | Returns: `{x, y, w, h}` | Throws: Error on invalid relativeTo | Notes: relativeTo: `"screen"` (default), `"window"`, `"client"`
- **element.Highlight()** — Signature: `element.Highlight(showTime:=unset, color:="Red", d:=2)` | Returns: element | Throws: — | Notes: showTime unset=toggle 2s, 0=permanent, positive=block N ms, negative=non-blocking N ms, `"clear"`=remove; debugging only
- **element.ClearHighlight()** — Signature: `element.ClearHighlight()` | Returns: — | Throws: — | Notes: Alias for `Highlight("clear")`
- **element.Dump()** — Signature: `element.Dump(scope:=1, delimiter:=" ", maxDepth:=-1)` | Returns: String | Throws: — | Notes: scope=1=element only, scope=5=full subtree; returns formatted tree string; use for debugging tree structure
- **element.DumpAll()** — Signature: `element.DumpAll(delimiter:=" ", maxDepth:=-1)` | Returns: String | Throws: — | Notes: Alias for Dump(7); dumps entire subtree from this element

### UIA Static Helpers

- **UIA.Filter()** — Signature: `UIA.Filter(elementArray, function)` | Returns: filtered `Array` | Throws: `TypeError` | Notes: Accepts Array or IUIAutomationElementArray; callback receives `(element)` and returns truthy to keep; matched elements get `.Index` property set
- **UIA.CompareElements()** — Signature: `UIA.CompareElements(el1, el2)` | Returns: Boolean | Throws: — | Notes: True if both reference the same UIA element; use instead of object identity comparison

### UIA Tree Views (Raw / Control / Content)

> Source: Microsoft Docs — UI Automation Tree Overview

The UIA tree is exposed in three filtered views. Most automation work uses the default Raw view (all elements). Understanding the other two is essential when `IsControlElement`/`IsContentElement` filtering affects search results.

| View | Filter | Walker | When to Use |
|------|--------|--------|-------------|
| Raw | None — all elements | `UIA.TreeWalkerTrue` | Default; used by FindFirst/FindAll; includes decorative and layout-only elements |
| Control | `IsControlElement = TRUE` | `ControlViewWalker` | Elements user can interact with plus structural non-interactive elements (toolbars, menus) |
| Content | `IsControlElement = TRUE` AND `IsContentElement = TRUE` | `ContentViewWalker` | Elements conveying actual information to the user (list values, text, inputs) — most useful for screen readers |

Use `{IsControlElement: True}` or `{IsContentElement: True}` as additional condition keys to restrict FindAll results to a specific view.

### Tree Scope Constants

| Constant | Value | Use |
|----------|-------|-----|
| `UIA.TreeScope.Element` | 1 | This element only — for property queries, not searches |
| `UIA.TreeScope.Children` | 2 | Direct children only — use for DataGridView row→cell traversal |
| `UIA.TreeScope.Family` | 3 | Element + Children combined |
| `UIA.TreeScope.Descendants` | 4 | All descendants (default for FindFirst/FindAll/FindElement) |
| `UIA.TreeScope.ElementDescendants` | 5 | Element + all descendants, includes the root element itself |
| `UIA.TreeScope.Subtree` | 7 | Element + all descendants — results include the root element itself |

### ControlType Integer Constants (UIA.Type.*)

> Source: UIAutomationClient.h — UIA_*ControlTypeId named constants (Microsoft Docs)

| Name | Value | Application |
|------|-------|-------------|
| `Button` | 50000 | Clickable buttons (cmdRunSave, cmdRunNext) |
| `Calendar` | 50001 | Date-picker calendar controls |
| `CheckBox` | 50002 | Toggle checkbox controls |
| `ComboBox` | 50003 | Drop-down combo boxes |
| `Edit` | 50004 | Text input fields (EXAM, txtALL) |
| `Hyperlink` | 50005 | Clickable link text |
| `Image` | 50006 | Image or icon controls |
| `ListItem` | 50007 | Individual items in a list |
| `List` | 50008 | List containers |
| `Menu` | 50009 | Context or drop-down menus |
| `MenuBar` | 50010 | Top-level menu bars |
| `MenuItem` | 50011 | Individual menu items |
| `ProgressBar` | 50012 | Progress indicators |
| `RadioButton` | 50013 | Radio button controls |
| `Tab` | 50018 | Tab strip containers |
| `TabItem` | 50019 | Individual tab pages |
| `Text` | 50020 | Static text labels |
| `ToolBar` | 50021 | Toolbar containers |
| `Tree` | 50023 | Tree view containers |
| `TreeItem` | 50024 | Individual tree view nodes |
| `Custom` | 50025 | Windows Forms UserControl panels (KWorkListCtl, KReportCtl) |
| `Group` | 50026 | GroupBox containers |
| `DataGrid` | 50028 | DataGridView containers |
| `DataItem` | 50029 | DataGridView rows and cells |
| `Document` | 50030 | Rich text areas and browser document panes |
| `Window` | 50032 | Top-level or child window controls |
| `Pane` | 50033 | Generic panel containers |
| `Header` | 50034 | Column header bars |
| `HeaderItem` | 50035 | Individual column header cells |
| `Table` | 50036 | Table containers (distinct from DataGrid) |
| `TitleBar` | 50037 | Window title bar area |
| `StatusBar` | 50017 | Status bar at window bottom |

### UIA_Browser Class (Browser Automation Overlay)

- **UIA_Browser() / UIA_Chrome() / UIA_Edge()** — Signature: `UIA_Browser(wTitle:="")` | Returns: browser wrapper instance | Throws: TargetError | Notes: Auto-detects browser type from window title; requires `#include UIA_Browser.ahk`
- **instance.BrowserElement** — Type: element | Notes: Root element of the browser window; proxies FindFirst, FindAll, ElementExist etc. directly
- **instance.GetCurrentDocumentElement()** — Signature: `GetCurrentDocumentElement()` | Returns: document element on success, `0` on timeout | Throws: — | Notes: Returns current page content element; must be called after each navigation completes; ALWAYS check `IsObject(result)` before use
- **instance.Navigate()** — Signature: `Navigate(url, targetTitle:="", waitLoadTimeOut:=-1, sleepAfter:=500)` | Returns: — | Throws: — | Notes: Sets URL and calls WaitPageLoad; sleepAfter adds DOM settlement delay
- **instance.GetCurrentURL()** — Signature: `GetCurrentURL(fromAddressBar:=False)` | Returns: String URL | Throws: — | Notes: Default fetches via COM (browser must be visible); fromAddressBar=True reads Edit element directly
- **instance.WaitPageLoad()** — Signature: `WaitPageLoad(targetTitle:="", timeOut:=-1, sleepAfter:=500, titleMatchMode:="", titleCaseSensitive:=False)` | Returns: — | Throws: — | Notes: Polls for title match; sleepAfter waits for DOM settlement after title change
- **instance.GetAllLinks()** — Signature: `GetAllLinks()` | Returns: Array of link elements | Throws: — | Notes: Finds all Link-type elements in BrowserElement subtree
- **instance.GetTab() / SelectTab()** — Signature: `GetTab(searchPhrase:="", matchMode:=3, caseSense:=True)` | Returns: tab element | Throws: `TargetError` | Notes: matchMode follows SetTitleMatchMode: 1=startsWith, 2=contains, 3=exact, RegEx
- **instance.JSExecute()** — Signature: `JSExecute(js)` | Returns: — | Throws: — | Notes: Executes JavaScript via address bar; use when UIA tree does not expose the needed interaction

## AHK V2 CONSTRAINTS

- `FindFirst()` and `FindElement()` throw `TargetError` when no element matches — they do NOT return `""` or `0`. The only safe non-throwing check variants are `ElementExist()` (returns `0` on failure) and `WaitElement()` (returns `0` on timeout). Every `FindFirst` call must be either inside a `try/catch` block or replaced with `ElementExist`.
  - ✗ `local btn := root.FindFirst({AutomationId: "Save"})` then `if !IsObject(btn)` — dead code; TargetError is thrown before the check runs
  - ✓ `local btn := root.ElementExist({AutomationId: "Save"})` then `if !btn` — ElementExist wraps FindElement in try/catch internally, returns 0 on failure
  - ✓ `try { local btn := root.FindFirst({AutomationId: "Save"}) } catch TargetError as err { ... }` — explicit handling per error type

- `UIA.ElementFromHandle()` also throws `TargetError` (no matching window) or `UnsetError` (COM returned null) — it never returns a falsy value. Always wrap in `try/catch`.
  - ✗ `if !IsObject(UIA.ElementFromHandle(hwnd))` — always false; exception raised before IsObject runs
  - ✓ `try { local root := UIA.ElementFromHandle(this._hwnd) } catch as err { return Result.Fail(...) }`

- `UIA.ElementFromHandle()` must be called at the start of every public adapter method that touches the UI tree — Windows Forms can fully rebuild the element tree on any repaint, focus change, or data refresh; a reference obtained in a prior method call is silently stale.
  - ✗ `this._rootElement := UIA.ElementFromHandle(...)` stored in `__New` — stale after first repaint
  - ✓ `local root := UIA.ElementFromHandle(this._hwnd)` at top of each method — always fresh

- Condition objects use `{}` AHK object literals, NOT `Map()` — the UIA library's `__CreateRawCondition()` method iterates conditions with `OwnProps()`, which does not work on `Map` instances.
  - ✗ `root.FindFirst(Map("AutomationId", "Save"))` — OwnProps() on Map returns nothing; condition is empty
  - ✓ `root.FindFirst({AutomationId: "Save"})` — object literal, OwnProps() enumerates correctly

- `WaitElement()` returns `0` (integer) on timeout — always check `if !result` or `if result = 0` before calling any property on it.
  - ✗ `local el := root.WaitElement({...}, 5000)` then `el.Click()` — MethodError if timeout returns 0
  - ✓ `local el := root.WaitElement({...}, 5000)` then `if !el` guard before `.Click()`

- `FindAll()` with `UIA.TreeScope.Descendants` (=4) on a DataGridView with hundreds of rows recurses into every cell — always use `UIA.TreeScope.Children` (=2) when iterating direct children of a known container.
  - ✗ `grid.FindAll({Type: UIA.Type.DataItem})` — default scope 4 recurses into cell children, returning rows AND cells mixed
  - ✓ `grid.FindAll({Type: UIA.Type.DataItem}, UIA.TreeScope.Children)` — direct row children only

- `FindFirst()` and `FindAll()` do NOT support searching for ancestor elements — `TreeScope.Parent` (=8) and `TreeScope.Ancestors` (=16) are invalid scopes for these methods (Microsoft UIA specification). Use `element.Parent` or `element.WalkTree("p")` for upward navigation.
  - ✗ `root.FindFirst(cond, UIA.TreeScope.Ancestors)` — undefined behavior; may return empty or error
  - ✓ `element.Parent` — returns direct parent via TreeWalker; `element.WalkTree("p,p")` = grandparent

- Searching the desktop root (`UIA.GetRootElement()`) with `TreeScope.Descendants` or `TreeScope.Subtree` can iterate thousands of elements across all open applications — always use `TreeScope.Children` (direct application windows only) when searching from the desktop root (Microsoft Docs warning).
  - ✗ `UIA.GetRootElement().FindAll(cond, UIA.TreeScope.Subtree)` — iterates all UI elements system-wide
  - ✓ `UIA.GetRootElement().FindAll(cond, UIA.TreeScope.Children)` — direct top-level windows only

- `catch as err` — the `as` keyword is mandatory in AHK v2; `catch err` without `as` is a syntax error.

- DataGridView cells in NTUH RIS adapter use the Name pattern `"{ColumnHeader} 資料列 {RowIndex}"` — never rely on position index because the column count differs between layout variants (24 vs 28 columns).

Safe-access priority order for UIA element discovery:
  1. `ElementExist(condition)` — when code only needs to branch on found/not-found, no error context needed
  2. `WaitElement(condition, timeOut)` — when element may not yet exist (async form open, tab switch)
  3. `try { FindFirst(condition) } catch TargetError` — when not-found needs specific handling distinct from COM errors
  4. `try { ... } catch as err { return Result.Fail(..., err.Message) }` — outer catch when element-not-found and COM errors are handled the same way

Unset variable handling: always check `if !el` after `WaitElement()` and `ElementExist()` before calling any method — these return integer `0` on failure, and `0.SomeMethod()` causes a MethodError.

Resource lifecycle: UIA element objects hold COM references and become stale after any UI refresh — store only `this._hwnd` (Integer HWND) across method calls, never store UIA element objects as instance variables.

## AGENT QA CHECKLIST

- [ ] Did I wrap every `FindFirst()` and `FindElement()` call in `try/catch TargetError`, or replace it with `ElementExist()`?
- [ ] Did I re-fetch root via `UIA.ElementFromHandle(this._hwnd)` at the entry of every public method — not once in `__New`?
- [ ] Did I use `WaitElement()` (not `FindFirst`) for any element that appears after a user action, tab switch, or async data load?
- [ ] Did I use `UIA.TreeScope.Children` (=2, not the default =4) when calling `FindAll` on a DataGridView to enumerate direct row children?

## RUNTIME ERROR MAPPING

| Exception Class | Trigger Condition | Detection Code | Fix |
|----------------|-------------------|----------------|-----|
| `TargetError` | `FindFirst()` or `FindElement()` called when target element does not exist, or `UIA.ElementFromHandle()` when window is closed | `e.Message` = "An element matching the condition was not found" or "No matching window found" | Replace FindFirst with `ElementExist()` for safe check, or wrap in `try/catch TargetError as err`; verify window exists with `WinExist()` before ElementFromHandle |
| `MethodError` | Calling `.Value`, `.Name`, or `.Click()` on integer `0` returned by `WaitElement()` or `ElementExist()` when they fail | `e.Message` contains "object of type Integer" | Add `if !el` guard immediately after every `WaitElement()` and `ElementExist()` call before any method access |
| `Error` (COM 0x80131509) | Accessing any property on a stale UIA element reference after Windows Forms repainted the window | `e.Extra` = "0x80131509" or `e.Message` contains "UIA_E_INVALIDOPERATION" | Never store UIA element objects between method calls; store `this._hwnd` only and re-acquire via `UIA.ElementFromHandle` at each entry point |

## TIER 1 — Root Acquisition and Safe Element Discovery
> METHODS COVERED: UIA.ElementFromHandle · ElementExist · FindFirst · try/catch TargetError

TIER 1 covers the two correct patterns for acquiring the root element and finding a single known control. Both `UIA.ElementFromHandle` and `FindFirst` throw `TargetError` on failure — neither returns a falsy value that can be checked with `IsObject()`. Pattern A (`ElementExist`) is preferred for simple branches; Pattern B (`try/catch TargetError`) is required when not-found needs distinct handling from COM errors.

```ahk
; ✓ Pattern A — ElementExist: safe non-throwing check, preferred for simple found/not-found branches
SomeAdapterMethod() {
    ; ✓ ElementFromHandle throws on invalid hwnd — wrap all root acquisition in try/catch
    local root
    try
        root := UIA.ElementFromHandle(this._hwnd)
    catch as err
        return Result.Fail(Result.CODE_UIA_PREFLIGHT_FAIL, "Element tree unavailable: " err.Message)

    ; ✓ ElementExist returns 0 on not-found — if !btn is meaningful here
    local btn := root.ElementExist({AutomationId: "cmdRunSave"})
    if !btn
        return Result.Fail(Result.CODE_UIA_ELEMENT_NOT_FOUND, "Save button not found")
    return Result.Ok(btn)
}

; ✓ Pattern B — try/catch TargetError: when not-found vs COM error need separate responses
SomeAdapterMethodB() {
    try {
        local root := UIA.ElementFromHandle(this._hwnd)
        local btn  := root.FindFirst({AutomationId: "cmdRunSave"})
        return Result.Ok(btn)
    } catch TargetError as err {
        ; ✓ TargetError = element not found — distinct from generic COM errors below
        return Result.Fail(Result.CODE_UIA_ELEMENT_NOT_FOUND, "Save button not found: " err.Message)
    } catch as err {
        return Result.Fail(Result.CODE_UIA_PREFLIGHT_FAIL, "COM error: " err.Message)
    }
}

; ✗ WRONG — IsObject check is dead code; FindFirst throws TargetError before returning
; local btn := root.FindFirst({AutomationId: "cmdRunSave"})
; if !IsObject(btn)    ; → never executes; TargetError already propagated
;     return Result.Fail(...)

; ✗ WRONG — storing root element between calls causes stale COM reference
__New() {
    ; this._rootElement := UIA.ElementFromHandle(this._hwnd)  ; → stale after any repaint
}
```

## TIER 2 — Nested FindFirst and Preflight Validation
> METHODS COVERED: ElementExist · chained container search · preflight sequence

TIER 2 covers finding a container element and then searching within it — the pattern used in `IsReady()` and `OpenPatientList()`. `ElementExist` is used instead of `FindFirst` so that the `if !el` guard is meaningful. The outer `try/catch` remains as the safety net for COM errors (stale element, window disappeared mid-call).

```ahk
; ✓ Chained ElementExist with preflight guards — precise error message per missing element
OpenPatientList() {
    try {
        local root := UIA.ElementFromHandle(this._hwnd)
        ; ✓ ElementExist returns 0 on not-found — the if !workListPane guard is live code
        local workListPane := root.ElementExist({AutomationId: "KWorkListCtl"})
        if !workListPane
            return Result.Fail(Result.CODE_UIA_PREFLIGHT_FAIL, "KWorkListCtl not found")

        local dataGrid := workListPane.ElementExist({AutomationId: "DataGridView1"})
        if !dataGrid
            return Result.Fail(Result.CODE_UIA_PREFLIGHT_FAIL, "DataGridView1 not found")

        return Result.Ok(dataGrid)
    } catch as err {
        return Result.Fail(Result.CODE_UIA_ELEMENT_NOT_FOUND, "OpenPatientList error: " err.Message)
    }
}
```

## TIER 3 — WaitElement for Async Elements
> METHODS COVERED: WaitElement · WaitElementNotExist · timeout handling

TIER 3 covers elements that appear after a user action (double-click opens form, tab switch shows panel). `WaitElement` blocks with a deadline and returns `0` on timeout — always check the return value before any property access.

```ahk
; ✓ WaitElement for async form open — KReportCtl appears after double-click, not immediately
OpenReportEditor(index) {
    try {
        ; ... (invoke or PostMessage to open KReportCtl) ...

        local root := UIA.ElementFromHandle(this._hwnd)
        ; ✓ WaitElement — KReportCtl appears asynchronously; FindFirst would TargetError immediately
        local reportPane := root.WaitElement({AutomationId: "KReportCtl"}, 10000)
        ; ✓ WaitElement returns 0 on timeout — must guard before any property access
        if !reportPane
            return Result.Fail(Result.CODE_TIMEOUT, "KReportCtl did not appear within 10s")

        return Result.Ok(reportPane)
    } catch as err {
        return Result.Fail(Result.CODE_UIA_ELEMENT_NOT_FOUND, "OpenReportEditor error: " err.Message)
    }
}

; ✓ WaitElementNotExist — confirm a dialog closed before the next operation proceeds
WaitForDialogClose(root, timeoutMs := 5000) {
    ; ✓ Returns 1 if element disappeared, 0 if timed out — check the return value
    if !root.WaitElementNotExist({AutomationId: "ConfirmDialog"}, timeoutMs)
        return Result.Fail(Result.CODE_TIMEOUT, "ConfirmDialog still visible after " timeoutMs "ms")
    return Result.Ok()
}

; ✗ WRONG — FindFirst on element that may not yet exist throws TargetError immediately
; local reportPane := root.FindFirst({AutomationId: "KReportCtl"})  ; → TargetError if not loaded yet
```

## TIER 4 — DataGridView Traversal by Name Pattern
> METHODS COVERED: FindAll with TreeScope.Children · ElementExist · Name pattern matching · multi-column row extraction

TIER 4 covers reading all rows from a DataGridView and extracting specific columns by Name pattern. Column positions vary across layout variants (24 vs 28 columns), so columns are matched by Name substring, not index. `UIA.TreeScope.Children` (=2) is mandatory — the default Descendants scope enters cells, producing a mixed row+cell result set.

```ahk
; ✓ Iterate DataGridView rows by DataItem type, extract cells by Name pattern
GetAllPatients() {
    try {
        local root := UIA.ElementFromHandle(this._hwnd)
        local grid := root.ElementExist({AutomationId: "DataGridView1"})
        if !grid
            return Result.Fail(Result.CODE_UIA_ELEMENT_NOT_FOUND, "DataGridView1 not found")

        ; ✓ Scope = Children (=2) — direct row children only, not recursive into cells
        ; ✗ Default Descendants (=4) would recurse into all cells, mixing rows and cells
        local rows := grid.FindAll({Type: UIA.Type.DataItem}, UIA.TreeScope.Children)
        if !rows.Length
            return Result.Ok([])

        local patients := []
        for row in rows {
            ; ✓ Re-fetch cells as direct children of each row
            local cells := row.FindAll({Type: UIA.Type.DataItem}, UIA.TreeScope.Children)
            local patientId := ""
            local examName  := ""

            for cell in cells {
                local cellName := ""
                try cellName := cell.Name
                ; ✓ Name pattern: "{ColumnHeader} 資料列 {RowIndex}" — column position unreliable
                if InStr(cellName, "病歷號")
                    try patientId := cell.Value
                if InStr(cellName, "檢查名稱")
                    try examName := cell.Value
            }

            if patientId != ""
                patients.Push(Map("patientId", patientId, "examName", examName))
        }

        return Result.Ok(patients)
    } catch as err {
        return Result.Fail(Result.CODE_UIA_ELEMENT_NOT_FOUND, "GetAllPatients error: " err.Message)
    }
}

; ✗ WRONG — column index brittle across 24-column vs 28-column layout variants
; local patientId := cells[3].Value   ; → wrong value in alternate layout
```

## TIER 5 — Multi-Window HWND Resolution and UIA.Filter
> METHODS COVERED: WinGetList · UIA.ElementFromHandle per window · UIA.Filter · WalkTree · Dump

TIER 5 covers finding the correct window among multiple instances and filtering element arrays by callback. `UIA.Filter` is O(n) over a pre-fetched array — it does not re-query the tree. `WalkTree` navigates by relative path without performing a subtree search.

```ahk
; ✓ Find a specific window among multiple instances by title substring
FindTargetWindow(exeName, titlePart) {
    local winList := WinGetList("ahk_exe " exeName)
    for hwnd in winList {
        local title := ""
        try title := WinGetTitle("ahk_id " hwnd)
        if InStr(title, titlePart)
            return hwnd
    }
    return 0
}

; ✓ Use per-window HWND for UIA root acquisition
GetRISWindowRoot() {
    local hwnd := FindTargetWindow("KReport.exe", "放射線資訊")
    if !hwnd
        return Result.Fail(Result.CODE_UIA_ELEMENT_NOT_FOUND, "RIS window not found")
    try
        return Result.Ok(UIA.ElementFromHandle(hwnd))
    catch as err
        return Result.Fail(Result.CODE_UIA_PREFLIGHT_FAIL, "ElementFromHandle failed: " err.Message)
}

; ✓ UIA.Filter — keep only enabled buttons from a pre-fetched array (no extra COM queries)
GetEnabledButtons(container) {
    local allButtons := container.FindAll({Type: UIA.Type.Button}, UIA.TreeScope.Children)
    ; ✓ Filter callback returns truthy to keep element; matched elements get .Index property
    return UIA.Filter(allButtons, (el) => el.IsEnabled)
}

; ✓ WalkTree — navigate by relative path without subtree search
NavigateToSibling(element) {
    ; ✓ "p" = parent, "-3" = 3rd previous sibling of parent — no FindFirst needed
    try return element.WalkTree("p,-3")
    catch as err
        return ""
}

; ✓ Dump — inspect tree structure during development; remove from production
DebugDumpGrid(root) {
    local grid := root.ElementExist({AutomationId: "DataGridView1"})
    if grid
        OutputDebug(grid.Dump(7))   ; scope 7 = UIA.TreeScope.Subtree = element + all descendants
}
```


## TIER 6 — Browser Automation
> METHODS COVERED: see Module_UIABrowser.md

Browser-specific UIA automation — `UIA_Browser`, `UIA_Chrome`, `UIA_Edge`, `UIA_Mozilla` — is a separate domain with its own document-ephemerality rules, Chromium accessibility activation, tab management, and JavaScript execution via the address bar. Load `Module_UIABrowser.md` for all browser automation tasks.

Key distinctions that justify a separate module: `GetCurrentDocumentElement()` returns `0` on timeout (never throws), document elements are invalidated after every navigation, `GetTab()` throws while `TabExist()` returns 0, and `WaitPageLoad()` polls the Reload button state (not DOM readiness).

```ahk
; ✓ For browser automation — load Module_UIABrowser.md instead of this section
; This stub exists so agents searching "UIA_Browser" find the cross-reference.
; ✓ Correct include order for browser automation:
; #include <UIA>
; #include <UIA_Browser>
; local browser := UIA_Browser("ahk_exe chrome.exe")   ; auto-detects Chrome/Edge/Mozilla
; local doc     := browser.GetCurrentDocumentElement()  ; ALWAYS check IsObject(doc) after
; ; All UIA element methods on doc: see Module_UIAElements.md
```


## DROP-IN RECIPES

```ahk
; SafeUIAFind — find one element with full preflight, returns element or throws descriptive Error
; ✓ Consolidates ElementFromHandle + ElementExist/WaitElement into one guarded call
SafeUIAFind(hwnd, condition, scope := 4, timeoutMs := 0) {
    if !(hwnd is Integer) || hwnd = 0
        throw TypeError("SafeUIAFind: hwnd must be a non-zero Integer", -1)
    if !(condition is Object)
        throw TypeError("SafeUIAFind: condition must be an AHK object literal {}", -1)
    local root
    try
        root := UIA.ElementFromHandle(hwnd)
    catch as err
        throw Error("SafeUIAFind: window unavailable (hwnd=" hwnd ") — " err.Message, -1)
    ; ✓ WaitElement for async, ElementExist for immediate — both return 0 on failure
    local el := timeoutMs > 0
        ? root.WaitElement(condition, timeoutMs, scope)
        : root.ElementExist(condition, scope)
    if !el {
        local condStr := ""
        for k, v in condition.OwnProps()
            condStr .= k "=" v " "
        throw Error("SafeUIAFind: element not found — {" Trim(condStr) "}", -1)
    }
    return el
}
; Call site: local btn := SafeUIAFind(hwnd, {AutomationId: "cmdRunSave"})
; Call site (async): local pane := SafeUIAFind(hwnd, {AutomationId: "KReportCtl"}, 4, 10000)

; ReadGridRowsByColumn — extract all DataGridView rows as Array of Maps, keyed by column header
; ✓ Uses TreeScope.Children to avoid recursing into cells; matches columns by Name substring
ReadGridRowsByColumn(grid, columnHeaders) {
    if !(grid is Object)
        throw TypeError("ReadGridRowsByColumn: grid must be a UIA element", -1)
    if !(columnHeaders is Array) || columnHeaders.Length = 0
        throw ValueError("ReadGridRowsByColumn: columnHeaders must be a non-empty Array", -1)
    ; ✓ Children scope — direct row children only; Descendants would mix rows and cells
    local rows := grid.FindAll({Type: UIA.Type.DataItem}, UIA.TreeScope.Children)
    local result := []
    for row in rows {
        local cells := row.FindAll({Type: UIA.Type.DataItem}, UIA.TreeScope.Children)
        local rowData := Map()
        for cell in cells {
            local cellName := ""
            try cellName := cell.Name
            for header in columnHeaders {
                if InStr(cellName, header)
                    try rowData[header] := cell.Value
            }
        }
        ; ✓ Only push rows where at least one target column was matched
        if rowData.Count > 0
            result.Push(rowData)
    }
    return result
}
; Call site: local rows := ReadGridRowsByColumn(grid, ["病歷號", "檢查名稱", "檢查日期"])
; Access row: rows[1]["病歷號"]
```

## ANTI-PATTERNS

| Pattern | Wrong | Correct | LLM Common Cause |
|---------|-------|---------|------------------|
| IsObject check after FindFirst | `local el := root.FindFirst(cond)` then `if !IsObject(el)` | `local el := root.ElementExist(cond)` then `if !el` | Confusion with older UIA v1 wrappers or web DOM APIs where find() returns null/undefined instead of throwing |
| Storing UIA element between calls | `this._reportEl := root.FindFirst(...)` in `__New` | Re-fetch in every method: `local root := UIA.ElementFromHandle(this._hwnd)` | Missing understanding that UIA elements are COM live references, not snapshots; DOM node analogy where elements are stable objects |
| FindFirst on async element | `local el := root.FindFirst({AutomationId: "KReportCtl"})` immediately after trigger | `root.WaitElement({AutomationId: "KReportCtl"}, 10000)` | Confusion between synchronous DOM query and asynchronous Windows Forms paint cycle |
| Subtree search on DataGridView | `grid.FindAll({Type: 50029})` (default scope=4) | `grid.FindAll({Type: 50029}, UIA.TreeScope.Children)` | Default scope assumption; Descendants (4) recurses into cells doubling the result set |
| Column access by position | `cells[5].Value` | Find cell where `InStr(cell.Name, "病歷號")` | Assuming fixed column layout; position changes across layout variants with 24 vs 28 columns |
| Map() for condition | `root.FindFirst(Map("AutomationId", "Save"))` | `root.FindFirst({AutomationId: "Save"})` | Correctly applying the general AHK v2 Map() rule to a context where {} object literals are required by the UIA library's OwnProps() iteration |
| `catch err` without `as` | `catch err { MsgBox err.Message }` | `catch as err { MsgBox err.Message }` | AHK v1 syntax; v1 bound the caught object to the catch parameter name without `as` |

## SEE ALSO

> This module does NOT cover: control interaction patterns (Invoke, ValuePattern, Toggle, ExpandCollapse, SelectionItem) → see Module_UIAPatterns.md
> This module does NOT cover: try/catch error class hierarchy, custom Error subclasses, error propagation strategies → see Module_Errors.md
> This module does NOT cover: WinExist, ControlClick fallback, HWND lifecycle, window title matching → see Module_WindowAndControl.md

- `Module_UIAPatterns.md` — InvokePattern, ValuePattern, NativeWindowHandle fallback, BM_CLICK; use after element discovery to actually interact with found elements
- `Module_Errors.md` — try/catch structure, TargetError vs OSError vs TypeError hierarchy, structured retry logic for transient COM failures
- `Module_WindowAndControl.md` — WinExist, WinGetList, HWND management, ControlClick fallback for elements where UIA interaction fails
