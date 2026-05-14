# Module_UIAPatterns.md
<!-- DOMAIN: UI Automation Interaction Patterns (Invoke, Value, Toggle, ExpandCollapse, SelectionItem, RangeValue, Grid, Scroll, Window) -->
<!-- SCOPE: Element discovery and tree traversal (FindFirst, FindAll, WaitElement, ElementExist) are not covered here — see Module_UIAElements.md; browser-specific interaction is in Module_UIABrowser.md -->
<!-- TRIGGERS: InvokePattern, ValuePattern, TogglePattern, ExpandCollapsePattern, SelectionItemPattern, RangeValuePattern, ScrollItemPattern, GridItemPattern, element.Click(), element.Value, SetValue, NativeWindowHandle, BM_CLICK, IsInvokePatternAvailable, "click via UIA", "type into UIA element", "read UIA value", "check checkbox UIA", "expand tree node", "select list item", "scroll into view", "read slider value" -->
<!-- CONSTRAINTS: element.Click() (empty WhichButton) is a valid universal smart-action in UIA.ahk — it tries Invoke→Toggle→ExpandCollapse→SelectionItem→LegacyIAccessible in order; element.InvokePattern.Invoke() is the explicit single-pattern call. Pattern accessor properties throw COM Error when the pattern is unsupported — always wrap in try/catch or check Is*PatternAvailable first. ValuePattern.SetValue() requires IsEnabled=True AND IsReadOnly=False. -->
<!-- CROSS-REF: Module_UIAElements.md, Module_UIABrowser.md, Module_WindowAndControl.md -->
<!-- VERSION: AHK v2.0+ with Lib/UIA.ahk v1.1.3 (Descolada) -->

## V1 → V2 BREAKING CHANGES

| v1 pattern (LLM commonly writes) | v2 correct form | Consequence |
|----------------------------------|-----------------|-------------|
| `element.Invoke()` called directly on the element | `element.InvokePattern.Invoke()` (via pattern accessor) | `Invoke()` lives on `InvokePattern`, not on the element; calling it on the element throws MethodError |
| `element.Click()` assumed to be invalid / "must use GetCurrentPattern" | `element.Click()` (empty WhichButton) IS valid in UIA.ahk — tries Invoke, Toggle, ExpandCollapse, SelectionItem, LegacyIAccessible in order | Existing docs mislead — `element.Click()` is the idiomatic universal smart-action; rejecting it forces unnecessary boilerplate |
| `element.SetValue("text")` directly on element | `element.ValuePattern.SetValue("text")` | SetValue is a ValuePattern method, not a universal element property setter; calling on element throws MethodError |
| Using HWND from `element.hwnd` (lower-case) | `element.NativeWindowHandle` (exact property name) | Wrong property name returns empty — causes silent null-HWND PostMessage that has no effect |
| `PostMessage(0x00F5,,,ctrl,title)` with window title string | Obtain HWND via `element.NativeWindowHandle` then `PostMessage(0x00F5, 0, 0, "ahk_id " hwnd)` | String title matching is unreliable for background windows; HWND is O(1) and unambiguous |
| Checking `element.IsInvokePatternAvailable` (property name from docs) | `element.IsInvokePatternAvailable` (same) OR just `try { element.InvokePattern.Invoke() } catch` | Availability check is correct but try/catch is often simpler; both patterns are valid |
| `element.TogglePattern.CurrentToggleState` (LLM adds "Current" prefix) | `element.TogglePattern.ToggleState` — no "Current" prefix on properties | UIA.ahk __Call strips "Current" prefix on method names but properties use the raw name; ToggleState is the correct property |

## API QUICK-REFERENCE

> Method count exceeds 20 — structured bulleted list format used.

### element.Click() — Universal Smart Action

- **element.Click()** — Signature: `element.Click(WhichButton:="", ClickCount:=1, ...)` | Returns: action name String (`"Invoke"`, `"Toggle"`, `"Expand"`, `"Collapse"`, `"Select"`, `"DoDefaultAction"`) or `0` if nothing worked | Throws: — | Notes: Empty WhichButton tries InvokePattern → TogglePattern → ExpandCollapsePattern → SelectionItemPattern → LegacyIAccessiblePattern.DoDefaultAction() in order; `"left"`/`"right"` uses AHK Click() at element center; Integer WhichButton = sleep N ms after smart-action

### Pattern Availability Checks (on element)

- **element.IsInvokePatternAvailable** — Type: Boolean | Notes: Read before `element.InvokePattern` to avoid COM Error; fastest availability check
- **element.IsValuePatternAvailable** — Type: Boolean | Notes: Read before `element.ValuePattern`; also check `element.ValueIsReadOnly` before SetValue
- **element.IsTogglePatternAvailable** — Type: Boolean | Notes: Read before `element.TogglePattern`
- **element.IsExpandCollapsePatternAvailable** — Type: Boolean | Notes: Read before `element.ExpandCollapsePattern`
- **element.IsSelectionItemPatternAvailable** — Type: Boolean | Notes: Read before `element.SelectionItemPattern`
- **element.IsRangeValuePatternAvailable** — Type: Boolean | Notes: Read before `element.RangeValuePattern`
- **element.IsScrollItemPatternAvailable** — Type: Boolean | Notes: Read before `element.ScrollItemPattern`
- **element.IsGridItemPatternAvailable** — Type: Boolean | Notes: Read before `element.GridItemPattern`
- **element.IsWindowPatternAvailable** — Type: Boolean | Notes: Read before `element.WindowPattern`
- **element.ValueIsReadOnly** — Type: Boolean | Notes: Check before `ValuePattern.SetValue()` — SetValue silently fails or throws on read-only controls

### InvokePattern

- **element.InvokePattern.Invoke()** — Signature: `Invoke()` | Returns: — | Throws: COM Error if not supported | Notes: Fires the element's default action without mouse movement; supports background windows; available on Button, MenuItem, Hyperlink

### ValuePattern

- **element.ValuePattern.SetValue()** — Signature: `SetValue(val)` | Returns: — | Throws: COM Error if not supported or IsReadOnly=True | Notes: Sets text content and triggers TextChanged/Validation events; preferred over ControlSetText for Windows Forms edits
- **element.ValuePattern.Value** — Type: String (get/set) | Notes: Get reads current text; Set calls SetValue internally; single-line edits only — multi-line requires TextPattern
- **element.ValuePattern.IsReadOnly** — Type: Boolean | Notes: Same as element.ValueIsReadOnly; confirm False before SetValue
- **element.Value** (element shortcut) — Type: String (get/set) | Notes: Direct shortcut on the element — reads ValuePattern, then RangeValuePattern, then LegacyIAccessible; setter calls SetValue; throws Error if no pattern supports it

### TogglePattern

- **element.TogglePattern.Toggle()** — Signature: `Toggle()` | Returns: — | Throws: COM Error | Notes: Cycles through toggle states (Off→On→Indeterminate for tristate); available on CheckBox, ToggleButton
- **element.TogglePattern.ToggleState** — Type: Integer | Notes: `UIA.ToggleState.Off=0`, `.On=1`, `.Indeterminate=2`; read AFTER toggling to confirm state changed

### ExpandCollapsePattern

- **element.ExpandCollapsePattern.Expand()** — Signature: `Expand()` | Returns: — | Throws: COM Error | Notes: Available on ComboBox, TreeItem, MenuItem with submenu; may be no-op if already expanded
- **element.ExpandCollapsePattern.Collapse()** — Signature: `Collapse()` | Returns: — | Throws: COM Error | Notes: Collapses the element; no-op if already collapsed or LeafNode
- **element.ExpandCollapsePattern.ExpandCollapseState** — Type: Integer | Notes: `UIA.ExpandCollapseState.Collapsed=0`, `.Expanded=1`, `.PartiallyExpanded=2`, `.LeafNode=3`

### SelectionItemPattern

- **element.SelectionItemPattern.Select()** — Signature: `Select()` | Returns: — | Throws: COM Error | Notes: Selects this item and deselects others in the container; available on ListItem, TabItem, RadioButton, TreeItem
- **element.SelectionItemPattern.AddToSelection()** — Signature: `AddToSelection()` | Returns: — | Throws: COM Error | Notes: Adds to multi-select; throws if CanSelectMultiple=False on the container
- **element.SelectionItemPattern.RemoveFromSelection()** — Signature: `RemoveFromSelection()` | Returns: — | Throws: COM Error | Notes: Deselects item; throws if IsSelectionRequired=True on the container
- **element.SelectionItemPattern.IsSelected** — Type: Boolean | Notes: Read after Select/AddToSelection to confirm the operation succeeded

### RangeValuePattern (Slider, SpinBox, ProgressBar)

- **element.RangeValuePattern.SetValue()** — Signature: `SetValue(val)` | Returns: — | Throws: COM Error | Notes: Sets numeric value; val must be within [Minimum, Maximum]; val is a double
- **element.RangeValuePattern.Value** — Type: Float | Notes: Current numeric value
- **element.RangeValuePattern.Minimum / Maximum** — Type: Float | Notes: Read before SetValue to clamp input; ProgressBar Maximum is typically 100
- **element.RangeValuePattern.SmallChange / LargeChange** — Type: Float | Notes: Step sizes for keyboard navigation increments

### ScrollItemPattern

- **element.ScrollItemPattern.ScrollIntoView()** — Signature: `ScrollIntoView()` | Returns: — | Throws: COM Error | Notes: Scrolls the containing ScrollPattern provider to make this element visible; use before Click() on off-screen items

### ScrollPattern (on container)

- **element.ScrollPattern.Scroll()** — Signature: `Scroll(verticalAmount:=-1, horizontalAmount:=-1)` | Returns: — | Throws: COM Error | Notes: UIA.ScrollAmount constants: NoScroll=-1, LargeDecrement=0, SmallDecrement=1, NoAmount=2, LargeIncrement=3, SmallIncrement=4
- **element.ScrollPattern.SetScrollPercent()** — Signature: `SetScrollPercent(verticalPercent:=-1, horizontalPercent:=-1)` | Returns: — | Throws: COM Error | Notes: -1 = do not scroll that axis; 0.0–100.0 for the other axis
- **element.ScrollPattern.VerticalScrollPercent / HorizontalScrollPercent** — Type: Float | Notes: Current scroll position as percentage

### GridItemPattern (on cell within DataGrid)

- **element.GridItemPattern.Row** — Type: Integer | Notes: Zero-based row index of this cell in its containing grid
- **element.GridItemPattern.Column** — Type: Integer | Notes: Zero-based column index
- **element.GridItemPattern.ContainingGrid** — Type: UIA element | Notes: The parent DataGrid element; use to navigate back to the container

### WindowPattern

- **element.WindowPattern.Close()** — Signature: `Close()` | Returns: — | Throws: COM Error | Notes: Programmatically closes the window; equivalent to clicking the X button
- **element.WindowPattern.SetWindowVisualState()** — Signature: `SetWindowVisualState(state)` | Returns: — | Throws: COM Error | Notes: `UIA.WindowVisualState.Normal=0`, `.Maximized=1`, `.Minimized=2`
- **element.WindowPattern.CanMaximize / CanMinimize** — Type: Boolean | Notes: Check before calling SetWindowVisualState to avoid COM Error

### NativeWindowHandle (WM_* Fallback)

- **element.NativeWindowHandle** — Type: Integer HWND | Notes: HWND of the underlying control; use for `PostMessage`/`SendMessage`/`ControlSend` when UIA patterns are unavailable; compose `"ahk_id " hwnd` for AHK window functions

### Pattern Constants (UIA.Pattern.*)

| Pattern | Constant | Supported On |
|---------|----------|--------------|
| `UIA.Pattern.Invoke` | Invoke | Button, MenuItem, Hyperlink |
| `UIA.Pattern.Value` | Value | Edit, ComboBox (editable) |
| `UIA.Pattern.Toggle` | Toggle | CheckBox, ToggleButton |
| `UIA.Pattern.ExpandCollapse` | ExpandCollapse | ComboBox, TreeItem, MenuItem with submenu |
| `UIA.Pattern.SelectionItem` | SelectionItem | ListItem, TabItem, RadioButton, TreeItem |
| `UIA.Pattern.Selection` | Selection | ListBox, TreeView, TabStrip (container) |
| `UIA.Pattern.RangeValue` | RangeValue | Slider, SpinBox, ProgressBar |
| `UIA.Pattern.ScrollItem` | ScrollItem | Any element inside a scrollable container |
| `UIA.Pattern.Scroll` | Scroll | ScrollViewer, ListBox, DataGrid (container) |
| `UIA.Pattern.GridItem` | GridItem | Cell within a DataGrid |
| `UIA.Pattern.Grid` | Grid | DataGrid, Table (container) |
| `UIA.Pattern.Window` | Window | Top-level and child windows |

## AHK V2 CONSTRAINTS

- `element.Click()` with empty `WhichButton` is the idiomatic smart-action in UIA.ahk — it tries InvokePattern, TogglePattern, ExpandCollapsePattern, SelectionItemPattern, and LegacyIAccessiblePattern.DoDefaultAction() in order and returns the action name. Use `element.InvokePattern.Invoke()` only when you specifically want invoke and nothing else.
  - ✗ `element.Invoke()` — MethodError; Invoke() is on the pattern object, not the element
  - ✓ `element.Click()` — universal smart-action; returns `"Invoke"`, `"Toggle"`, `"Select"` etc. or `0`
  - ✓ `element.InvokePattern.Invoke()` — explicit invoke; throws COM Error if InvokePattern unsupported

- Pattern accessor properties (`element.InvokePattern`, `element.ValuePattern`, etc.) throw a COM Error when the control does not support the pattern. Always guard with `Is*PatternAvailable` check or a `try/catch`.
  - ✗ `element.TogglePattern.Toggle()` without guard — COM Error if CheckBox is disabled or element is wrong type
  - ✓ `if element.IsTogglePatternAvailable` then `element.TogglePattern.Toggle()` — pre-check
  - ✓ `try { element.TogglePattern.Toggle() } catch as err { ... }` — exception handling

- `ValuePattern.SetValue()` requires both `element.IsEnabled = True` AND `ValuePattern.IsReadOnly = False` — check both before calling; a read-only edit control accepts the call silently but the value does not change.
  - ✗ `element.ValuePattern.SetValue("text")` on a grayed-out or read-only field — silent no-op
  - ✓ `if element.IsEnabled && !element.ValueIsReadOnly` then `element.ValuePattern.SetValue("text")`

- `element.NativeWindowHandle` returns an Integer — compose `"ahk_id " hwnd` for window-function calls; the integer alone is valid as the second argument to `PostMessage`.
  - ✗ `PostMessage(0x00F5, 0, 0, element.NativeWindowHandle)` — NativeWindowHandle is Integer; PostMessage 4th arg expects string or control ref
  - ✓ `PostMessage(0x00F5, 0, 0, "ahk_id " element.NativeWindowHandle)` — correct HWND string form

- `element.Value` (the element shorthand) reads ValuePattern → RangeValuePattern → LegacyIAccessible in sequence. It throws `Error` if none of those patterns is available. Always wrap in `try`.
  - ✓ `try { local v := element.Value } catch { v := "" }` — safe read with empty-string fallback

- All pattern method calls that cross a COM boundary (`Invoke()`, `SetValue()`, `Toggle()`, `Select()`) must be inside `try { } catch as err { }` — any of these can throw if the element becomes stale or the application transitions state between the discovery and the interaction.

Safe-access priority order for pattern interaction:
  1. `element.Click()` — when you need the control's default action and don't care which pattern fires
  2. `element.Is*PatternAvailable` check then explicit pattern call — when you need a specific pattern with branch logic
  3. `try { element.*Pattern.Method() } catch as err` — when the exception message carries diagnostic information
  4. `element.NativeWindowHandle` + `PostMessage` / `BM_CLICK` — last resort for controls that expose no UIA patterns

## AGENT QA CHECKLIST

- [ ] Did I use `element.Click()` (smart-action) or a specific `element.*Pattern.*()` call, rather than calling `element.Invoke()` directly on the element?
- [ ] Did I check `IsEnabled` and `ValueIsReadOnly` before calling `ValuePattern.SetValue()`?
- [ ] Did I wrap every pattern method call in `try/catch as err` to handle stale elements and unsupported-pattern COM errors?
- [ ] Did I use `element.NativeWindowHandle` (not `element.hwnd`) when composing `"ahk_id " hwnd` for PostMessage/ControlSend fallbacks?

## RUNTIME ERROR MAPPING

| Exception Class | Trigger Condition | Detection Code | Fix |
|----------------|-------------------|----------------|-----|
| `Error` (COM, code 0x80040201) | Accessing `element.InvokePattern` / any pattern accessor when the control does not support that pattern | `e.Message` contains "pattern" or "not supported" | Check `element.Is*PatternAvailable` before accessing the pattern property, or use `try/catch`; fall back to `element.Click()` or `NativeWindowHandle + PostMessage` |
| `Error` (COM, 0x80131509 UIA_E_INVALIDOPERATION) | Calling `ExpandCollapsePattern.Expand()` or other state-change methods when the element is in the wrong state or disabled | `e.Extra` = "0x80131509" | Check `element.IsEnabled` and `ExpandCollapsePattern.ExpandCollapseState` before calling; wrap in try/catch |
| `Error` (COM, stale element) | Any pattern method call after Windows Forms repainted the window (element reference is stale) | `e.Message` contains "element" or COM HRESULT in `e.Extra` | Never cache pattern objects across method calls; re-acquire element via `UIA.ElementFromHandle` + `ElementExist`, then re-access the pattern |

## TIER 1 — element.Click() Smart Action and InvokePattern
> METHODS COVERED: element.Click · element.InvokePattern.Invoke · element.IsInvokePatternAvailable

TIER 1 covers the two correct ways to trigger a control's default action. `element.Click()` (empty WhichButton) is the idiomatic smart-action that tries multiple patterns in sequence — use it when you want the natural default action. `element.InvokePattern.Invoke()` is the explicit single-pattern call — use it when you specifically need InvokePattern and want a COM Error if it's not available.

```ahk
; ✓ element.Click() — idiomatic smart-action; tries Invoke→Toggle→Expand→Select→LegacyIAccessible
ClickSave() {
    try {
        local root := UIA.ElementFromHandle(this._hwnd)
        local btn := root.ElementExist({AutomationId: "cmdRunSave"})
        if !btn
            return Result.Fail(Result.CODE_UIA_ELEMENT_NOT_FOUND, "Save button not found")

        ; ✓ element.Click() tries InvokePattern first — returns "Invoke" on success
        local action := btn.Click()
        if !action
            return Result.Fail(Result.CODE_UIA_INVOKE_FAIL, "Click() found no applicable pattern")
        return Result.Ok(action)
    } catch as err {
        return Result.Fail(Result.CODE_UIA_ELEMENT_NOT_FOUND, "ClickSave error: " err.Message)
    }
}

; ✓ Explicit InvokePattern — use when you specifically require Invoke (not Toggle or Select)
ClickSaveExplicit() {
    try {
        local root := UIA.ElementFromHandle(this._hwnd)
        local btn := root.ElementExist({AutomationId: "cmdRunSave"})
        if !btn
            return Result.Fail(Result.CODE_UIA_ELEMENT_NOT_FOUND, "Save button not found")

        ; ✓ Is*PatternAvailable check before pattern access — avoids COM Error
        if !btn.IsInvokePatternAvailable
            return Result.Fail(Result.CODE_UIA_INVOKE_FAIL, "InvokePattern not supported")
        btn.InvokePattern.Invoke()
        return Result.Ok(true)
    } catch as err {
        return Result.Fail(Result.CODE_UIA_ELEMENT_NOT_FOUND, "ClickSaveExplicit error: " err.Message)
    }
}

; ✗ WRONG — Invoke() is on InvokePattern, not the element; also btn.Click() IS valid
; btn.Invoke()           ; → MethodError: method not found on IUIAutomationElement
; btn.Click()            ; ← actually CORRECT in UIA.ahk — this is the smart-action
```

## TIER 2 — ValuePattern Text Input and RangeValuePattern
> METHODS COVERED: element.Value · element.ValuePattern.SetValue · element.ValueIsReadOnly · element.RangeValuePattern.SetValue · element.RangeValuePattern.Value

TIER 2 covers reading and writing text in edit controls and numeric values in sliders/spinners. `element.Value` is the shorthand that internally tries ValuePattern then RangeValuePattern then LegacyIAccessible. `ValuePattern.SetValue()` requires the control to be enabled and not read-only — check both before calling.

```ahk
; ✓ Read text — try the element shorthand first; falls back through pattern chain automatically
ReadExamField() {
    try {
        local root := UIA.ElementFromHandle(this._hwnd)
        local examField := root.ElementExist({AutomationId: "EXAM"})
        if !examField
            return Result.Fail(Result.CODE_UIA_ELEMENT_NOT_FOUND, "EXAM field not found")

        ; ✓ element.Value shorthand — tries ValuePattern, RangeValuePattern, LegacyIAccessible
        local text := ""
        try text := examField.Value
        return Result.Ok(text)
    } catch as err {
        return Result.Fail(Result.CODE_UIA_ELEMENT_NOT_FOUND, "ReadExamField error: " err.Message)
    }
}

; ✓ Write text — check IsEnabled and IsReadOnly before SetValue
PasteReport(text) {
    try {
        local root := UIA.ElementFromHandle(this._hwnd)
        local examField := root.ElementExist({AutomationId: "EXAM"})
        if !examField
            return Result.Fail(Result.CODE_UIA_ELEMENT_NOT_FOUND, "EXAM field not found")

        ; ✓ Guard: IsEnabled and not IsReadOnly before SetValue — silent no-op otherwise
        if !examField.IsEnabled
            return Result.Fail(Result.CODE_UIA_INVOKE_FAIL, "EXAM field is disabled")
        if examField.ValueIsReadOnly
            return Result.Fail(Result.CODE_UIA_INVOKE_FAIL, "EXAM field is read-only")

        examField.ValuePattern.SetValue(text)
        return Result.Ok(true)
    } catch as err {
        return Result.Fail(Result.CODE_UIA_ELEMENT_NOT_FOUND, "PasteReport error: " err.Message)
    }
}

; ✓ RangeValuePattern — read and set slider/spinner numeric values
SetSliderValue(root, automationId, targetValue) {
    try {
        local slider := root.ElementExist({AutomationId: automationId})
        if !slider || !slider.IsRangeValuePatternAvailable
            return Result.Fail(Result.CODE_UIA_ELEMENT_NOT_FOUND, automationId " slider not found or unsupported")

        local rvp := slider.RangeValuePattern
        ; ✓ Clamp to valid range before setting — SetValue throws on out-of-range values
        local clamped := Max(rvp.Minimum, Min(rvp.Maximum, targetValue))
        rvp.SetValue(clamped)
        return Result.Ok(rvp.Value)   ; read back to confirm
    } catch as err {
        return Result.Fail(Result.CODE_UIA_INVOKE_FAIL, "SetSliderValue error: " err.Message)
    }
}

; ✗ WRONG — element.SetValue() does not exist on IUIAutomationElement
; examField.SetValue(text)   ; → MethodError
```

## TIER 3 — NativeWindowHandle Fallback (BM_CLICK / WM_*)
> METHODS COVERED: element.NativeWindowHandle · PostMessage · BM_CLICK · InvokePattern fallback chain

TIER 3 covers controls that expose no UIA patterns — owner-drawn buttons and legacy Windows Forms controls. The pattern is: try InvokePattern first, fall back to `BM_CLICK` via `NativeWindowHandle`. This works on background windows without activation.

```ahk
; ✓ Invoke-first then BM_CLICK fallback
ClickButtonFallback(automationId) {
    try {
        local root := UIA.ElementFromHandle(this._hwnd)
        local btn := root.ElementExist({AutomationId: automationId})
        if !btn
            return Result.Fail(Result.CODE_UIA_ELEMENT_NOT_FOUND, automationId " not found")

        ; ✓ Try InvokePattern first — preferred; works without window activation
        if btn.IsInvokePatternAvailable {
            btn.InvokePattern.Invoke()
            return Result.Ok("Invoke")
        }

        ; ✓ BM_CLICK fallback via NativeWindowHandle — for owner-drawn buttons
        local ctrlHwnd := btn.NativeWindowHandle
        if !ctrlHwnd
            return Result.Fail(Result.CODE_UIA_INVOKE_FAIL, "No HWND for " automationId)

        ; ✓ "ahk_id " hwnd — correct form for PostMessage; HWND is Integer
        PostMessage(0x00F5, 0, 0, "ahk_id " ctrlHwnd)   ; 0x00F5 = BM_CLICK
        return Result.Ok("BM_CLICK")
    } catch as err {
        return Result.Fail(Result.CODE_UIA_ELEMENT_NOT_FOUND, "ClickButtonFallback error: " err.Message)
    }
}

; ✓ SendMessage for synchronous WM_* — waits for message to be processed
SendWMCommand(element, wMsg, wParam := 0, lParam := 0) {
    local hwnd := element.NativeWindowHandle
    if !hwnd
        throw Error("SendWMCommand: element has no NativeWindowHandle", -1)
    ; ✓ SendMessage waits for processing; PostMessage is fire-and-forget
    return SendMessage(wMsg, wParam, lParam, "ahk_id " hwnd)
}

; ✗ WRONG — element.hwnd is not the correct property name in UIA.ahk
; local ctrlHwnd := btn.hwnd   ; → empty; property is NativeWindowHandle
; PostMessage(0x00F5, 0, 0, btn.NativeWindowHandle)  ; → PostMessage expects string, not Integer
```

## TIER 4 — Toggle, ExpandCollapse, and SelectionItem Patterns
> METHODS COVERED: element.TogglePattern.Toggle · element.TogglePattern.ToggleState · element.ExpandCollapsePattern.Expand · element.ExpandCollapsePattern.Collapse · element.ExpandCollapsePattern.ExpandCollapseState · element.SelectionItemPattern.Select · element.SelectionItemPattern.IsSelected

TIER 4 covers the three patterns that `element.Click()` tries after InvokePattern. Use them explicitly when you need to verify state after the operation or when `Click()` fires the wrong pattern.

```ahk
; ✓ TogglePattern — check state after toggle to confirm success
SetCheckboxState(root, automationId, desiredOn) {
    try {
        local cb := root.ElementExist({AutomationId: automationId})
        if !cb || !cb.IsTogglePatternAvailable
            return Result.Fail(Result.CODE_UIA_ELEMENT_NOT_FOUND, automationId " checkbox not found")

        local tp := cb.TogglePattern
        ; ✓ Only toggle if current state differs from desired state
        if (tp.ToggleState = UIA.ToggleState.On) != desiredOn
            tp.Toggle()
        ; ✓ Read back state to confirm — Toggle() may be a no-op on disabled controls
        local newState := tp.ToggleState
        if (newState = UIA.ToggleState.On) != desiredOn
            return Result.Fail(Result.CODE_UIA_INVOKE_FAIL, "Toggle state did not change to desired")
        return Result.Ok(newState)
    } catch as err {
        return Result.Fail(Result.CODE_UIA_INVOKE_FAIL, "SetCheckboxState error: " err.Message)
    }
}

; ✓ ExpandCollapsePattern — expand/collapse tree or combo; check state first
ExpandTreeNode(root, automationId) {
    try {
        local node := root.ElementExist({AutomationId: automationId})
        if !node || !node.IsExpandCollapsePatternAvailable
            return Result.Fail(Result.CODE_UIA_ELEMENT_NOT_FOUND, automationId " tree node not found")

        local ecp := node.ExpandCollapsePattern
        ; ✓ Check current state — Expand() on a LeafNode throws 0x80131509
        local state := ecp.ExpandCollapseState
        if state = UIA.ExpandCollapseState.LeafNode
            return Result.Fail(Result.CODE_UIA_INVOKE_FAIL, automationId " is a leaf node — cannot expand")
        if state = UIA.ExpandCollapseState.Collapsed
            ecp.Expand()
        ; state = Expanded or PartiallyExpanded: already open
        return Result.Ok(ecp.ExpandCollapseState)
    } catch as err {
        return Result.Fail(Result.CODE_UIA_INVOKE_FAIL, "ExpandTreeNode error: " err.Message)
    }
}

; ✓ SelectionItemPattern — select list/tab item; confirm IsSelected after
SelectListItem(container, itemName) {
    try {
        local item := container.ElementExist({Name: itemName, Type: UIA.Type.ListItem})
        if !item || !item.IsSelectionItemPatternAvailable
            return Result.Fail(Result.CODE_UIA_ELEMENT_NOT_FOUND, itemName " list item not found")

        item.SelectionItemPattern.Select()
        ; ✓ Read back IsSelected to confirm
        if !item.SelectionItemPattern.IsSelected
            return Result.Fail(Result.CODE_UIA_INVOKE_FAIL, itemName " was not selected after Select()")
        return Result.Ok(true)
    } catch as err {
        return Result.Fail(Result.CODE_UIA_INVOKE_FAIL, "SelectListItem error: " err.Message)
    }
}

; ✗ WRONG — CurrentToggleState does not exist; property is ToggleState
; local state := element.TogglePattern.CurrentToggleState   ; → empty or PropertyError
```

## TIER 5 — Scroll, Grid, and Window Patterns
> METHODS COVERED: element.ScrollItemPattern.ScrollIntoView · element.ScrollPattern.SetScrollPercent · element.GridItemPattern.Row · element.GridItemPattern.Column · element.WindowPattern.Close · element.WindowPattern.SetWindowVisualState

TIER 5 covers patterns needed for scrollable lists, DataGrid cell position reading, and programmatic window management. `ScrollIntoView` must be called before `Click()` on off-screen items — UIA Click attempts to find a clickable point and will fail if the element is scrolled out of view.

```ahk
; ✓ ScrollIntoView before Click on off-screen list item
ClickOffScreenItem(listElement, itemName) {
    try {
        local item := listElement.ElementExist({Name: itemName})
        if !item
            return Result.Fail(Result.CODE_UIA_ELEMENT_NOT_FOUND, itemName " not found")

        ; ✓ ScrollIntoView first — Click() on IsOffscreen=True element may fail silently
        if item.IsOffscreen {
            if item.IsScrollItemPatternAvailable
                item.ScrollItemPattern.ScrollIntoView()
            Sleep 100   ; allow scroll animation to complete
        }
        item.Click()
        return Result.Ok(true)
    } catch as err {
        return Result.Fail(Result.CODE_UIA_INVOKE_FAIL, "ClickOffScreenItem error: " err.Message)
    }
}

; ✓ GridItemPattern — read cell row/column coordinates in DataGrid
ReadCellPosition(cell) {
    if !cell.IsGridItemPatternAvailable
        return Map("row", -1, "col", -1)
    local gip := cell.GridItemPattern
    ; ✓ Row/Column are zero-based integers from GridItemPattern
    return Map("row", gip.Row, "col", gip.Column)
}

; ✓ ScrollPattern — programmatic scroll to 50% vertical on a listbox
ScrollToMiddle(container) {
    try {
        if !container.IsScrollPatternAvailable
            return
        local sp := container.ScrollPattern
        if sp.ScrollVerticallyScrollable
            sp.SetScrollPercent(50, -1)   ; -1 = don't change horizontal
    } catch as err {
        OutputDebug "ScrollToMiddle error: " err.Message
    }
}

; ✓ WindowPattern — close a secondary dialog window
CloseDialog(root, dialogAutomationId) {
    try {
        local dlg := root.ElementExist({AutomationId: dialogAutomationId})
        if !dlg || !dlg.IsWindowPatternAvailable
            return Result.Fail(Result.CODE_UIA_ELEMENT_NOT_FOUND, dialogAutomationId " dialog not found")
        dlg.WindowPattern.Close()
        return Result.Ok(true)
    } catch as err {
        return Result.Fail(Result.CODE_UIA_INVOKE_FAIL, "CloseDialog error: " err.Message)
    }
}

; ✓ WindowPattern — minimize/restore window programmatically
MinimizeWindow(root) {
    try {
        local win := root.ElementExist({Type: UIA.Type.Window})
        if !win || !win.IsWindowPatternAvailable
            return
        if win.WindowPattern.CanMinimize
            win.WindowPattern.SetWindowVisualState(UIA.WindowVisualState.Minimized)
    } catch
        return
}
```

### Performance Notes

Pattern accessor properties (`element.InvokePattern`, `element.ValuePattern`, etc.) are lazy-cached by UIA.ahk via `DefineProp` — the first access calls `GetPattern()` (one COM round trip), subsequent accesses read the cached value at O(1). Cache the pattern object in a local variable when calling multiple methods on the same pattern: `local vp := element.ValuePattern` then `vp.SetValue(...)` and `vp.IsReadOnly` — avoids the DefineProp overhead on each access. `Is*PatternAvailable` reads from the UIA property bag (one COM call per check) — for hot-path code where pattern support is known from context, skip the availability check and use `try/catch` instead. `element.Click()` checks three pattern availability properties internally before falling back — if you know a button uses InvokePattern, `element.InvokePattern.Invoke()` is marginally faster. `ValuePattern.SetValue()` triggers `TextChanged` and validation events synchronously — the call may block until the application processes the event; if the application has slow validators, add a small `Sleep` after SetValue before reading the field back. `ScrollIntoView()` is asynchronous — the scroll animation completes after the call returns; add `Sleep 100–200` before clicking the newly visible element to avoid hitting the wrong position.

## DROP-IN RECIPES

```ahk
; SafeInvoke — clicks a UIA element using the best available pattern, with fallback chain
; ✓ Tries: element.Click() → InvokePattern → BM_CLICK via NativeWindowHandle
SafeInvoke(element) {
    if !(element is Object)
        throw TypeError("SafeInvoke: element must be a UIA element object", -1)
    ; ✓ element.Click() tries Invoke/Toggle/Expand/Select automatically — use first
    local action := element.Click()
    if action
        return action   ; returns "Invoke", "Toggle", "Expand", "Select", etc.
    ; ✓ BM_CLICK fallback — for owner-drawn buttons that expose no UIA patterns
    local hwnd := 0
    try hwnd := element.NativeWindowHandle
    if hwnd {
        PostMessage(0x00F5, 0, 0, "ahk_id " hwnd)
        return "BM_CLICK"
    }
    throw Error("SafeInvoke: element has no applicable pattern and no NativeWindowHandle", -1)
}
; Call site: local action := SafeInvoke(btn)   ; returns the action name used

; SafeSetValue — sets text on an edit control with IsEnabled/IsReadOnly guard + LegacyIAccessible fallback
; ✓ Tries: ValuePattern.SetValue → LegacyIAccessiblePattern.SetValue → ControlSend fallback
SafeSetValue(element, text) {
    if !(element is Object)
        throw TypeError("SafeSetValue: element must be a UIA element object", -1)
    if !(text is String)
        throw TypeError("SafeSetValue: text must be a String", -1)
    if !element.IsEnabled
        throw Error("SafeSetValue: element is disabled", -1)
    ; ✓ ValuePattern path — preferred; triggers TextChanged events
    if element.IsValuePatternAvailable {
        if element.ValueIsReadOnly
            throw Error("SafeSetValue: element is read-only", -1)
        element.ValuePattern.SetValue(text)
        return "ValuePattern"
    }
    ; ✓ LegacyIAccessible fallback — for controls that expose MSAA but not ValuePattern
    if element.IsLegacyIAccessiblePatternAvailable {
        try {
            element.LegacyIAccessiblePattern.SetValue(text)
            return "LegacyIAccessible"
        }
    }
    ; ✓ ControlSend last resort — bypasses validation events; note the trade-off
    local hwnd := 0
    try hwnd := element.NativeWindowHandle
    if hwnd {
        ControlFocus "ahk_id " hwnd
        ControlSetText text, "ahk_id " hwnd
        return "ControlSetText"
    }
    throw Error("SafeSetValue: no supported write method found for this element", -1)
}
; Call site: local method := SafeSetValue(examField, "CT CHEST")
```

## ANTI-PATTERNS

| Pattern | Wrong | Correct | LLM Common Cause |
|---------|-------|---------|------------------|
| Calling Invoke on element | `element.Invoke()` | `element.Click()` (smart-action) or `element.InvokePattern.Invoke()` | Cross-framework confusion with Selenium/Playwright `.click()` method on element objects |
| SetValue directly on element | `element.SetValue("text")` | `element.ValuePattern.SetValue("text")` | Missing UIA pattern model; confusing element property setter with pattern method |
| Wrong HWND property name | `element.hwnd` | `element.NativeWindowHandle` | Property name guessing from WinForms/WPF; `hwnd` is not a UIA.ahk property |
| No pattern availability guard | `element.TogglePattern.Toggle()` without guard | `if element.IsTogglePatternAvailable` check or `try/catch as err` | Assuming all checkboxes support TogglePattern; some expose SelectionItem instead |
| ControlSetText instead of ValuePattern | `ControlSetText("text", "EXAM", title)` | `examField.ValuePattern.SetValue("text")` | Bypasses form validation and TextChanged events; leaves form in inconsistent state |
| SetValue on read-only field | `element.ValuePattern.SetValue(text)` without checking IsReadOnly | Guard with `if !element.ValueIsReadOnly` before SetValue | Unaware of IsReadOnly property; silent failure (value unchanged) is hard to diagnose |
| Click on off-screen element | `item.Click()` when `item.IsOffscreen = True` | `item.ScrollItemPattern.ScrollIntoView()` then `Sleep 100` then `item.Click()` | Assuming UIA Click auto-scrolls; it tries to find a clickable point at the element's current (off-screen) coordinates |

## SEE ALSO

> This module does NOT cover: element discovery (FindFirst, FindAll, ElementExist, WaitElement, tree traversal) → see Module_UIAElements.md
> This module does NOT cover: browser-specific document element access, tab management, JavaScript execution → see Module_UIABrowser.md
> This module does NOT cover: WinExist, ControlSend, ControlGetText, window title matching for non-UIA fallbacks → see Module_WindowAndControl.md

- `Module_UIAElements.md` — element discovery and tree traversal; use to find the element before calling any pattern method
- `Module_UIABrowser.md` — browser automation; document element lifecycle, WaitPageLoad, tab management, JSExecute
- `Module_WindowAndControl.md` — WinExist, ControlSend, ControlGetText, ControlClick as non-UIA fallbacks when `NativeWindowHandle` is available but no UIA pattern exists
