# Module_GUI.md
<!-- DOMAIN: GUI -->
<!-- SCOPE: Screen graphics operations (PixelSearch, ImageSearch, GDI+ drawing, screen overlays) and WebView2/IE-based browser controls embedded in a Gui are not covered — see Module_GraphicsAndScreen.md -->
<!-- TRIGGERS: Gui(), AddText(), AddEdit(), AddButton(), AddListView(), AddTreeView(), AddTab3(), AddGroupBox(), Submit(), OnEvent(), .Bind(), GuiFromHwnd(), GuiCtrlFromHwnd(), "make a window", "add a button", "create a form", "gui layout", "position controls", "dialog", "listview rows", "treeview nodes", "resizable window", "modal dialog", "currentY", "padding", "GuiForm", "owned window", "tab control", "status bar", "progress bar", "responsive GUI" -->
<!-- CONSTRAINTS: Never use v1 command syntax (Gui, Add, Show, Destroy) — it causes load-time errors in v2. Store all control references in Map(), never {}. Bind class-method event handlers with .Bind(this) — omitting it causes UnsetError at call time. gui.Close() does not exist — use gui.Hide() to keep the object alive or gui.Destroy() to remove it. Arrow callbacks must be single-expression only — multi-line arrow blocks cause SyntaxError. Re-enable the owner window BEFORE destroying or hiding the owned window — never after. -->
<!-- CROSS-REF: Module_GraphicsAndScreen.md, Module_InputAndHotkeys.md, Module_Classes.md, Module_Errors.md -->
<!-- VERSION: AHK v2.0+ -->

## V1 → V2 BREAKING CHANGES

| v1 pattern (LLM commonly writes) | v2 correct form | Consequence |
|----------------------------------|-----------------|-------------|
| `Gui, Add, Text,, Hello World!` | `gui.AddText(, "Hello World!")` | Load-time error — v1 command syntax is illegal in v2; the comma-based dispatcher no longer exists |
| `Gui, Show` / `Gui, Hide` | `gui.Show()` / `gui.Hide()` | Load-time error — all Gui commands are replaced by object methods on the Gui instance |
| `GuiControl, Disable, ctrlHwnd` | `ctrl.Opt("+Disabled")` or `ctrl.Enabled := false` | GuiControl command does not exist in v2; control state is managed through object properties |
| `GuiControl, , ctrlHwnd, newText` | `ctrl.Value := newText` | Control text cannot be set via the GuiControl command; use the `.Value` property |
| `Gui, Submit, NoHide` | `gui.Submit(false)` | v1 used the sub-command string "NoHide"; v2 uses a boolean — omitting `false` hides the window automatically |
| `Gui, +Owner%hwnd%` | `gui.Opt("+Owner" . hwnd)` | v1 used `%var%` expansion inside option strings; v2 requires explicit string concatenation |
| `new MyGui()` | `MyGui()` | NameError at runtime — `new` keyword does not exist in v2; class instantiation always uses `ClassName()` |
| `this.ctrls := {btn: ctrlRef}` | `this.controls := Map(); this.controls["btn"] := ctrlRef` | Object literals lack `.Has()`, `.Delete()`, and safe-access semantics — dynamic key lookups silently fail or return wrong values |
| `gui.Close()` | `gui.Hide()` or `gui.Destroy()` | MethodError — `.Close()` is not a Gui method in v2; the window remains open and errors can be swallowed silently |
| `Gui, Destroy` | `gui.Destroy()` | v1 command syntax; also note that Destroy is irreversible — use Hide to keep the object alive for reuse |

## API QUICK-REFERENCE

### Gui Constructor and Window Methods

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `Gui()` | `Gui(options?, title?, eventObj?)` | Gui object | — | Never use `new Gui()` — the `new` keyword does not exist in v2 |
| `.Show()` | `.Show(options?)` | — | — | Options: `"w400 h300"`, `"Center"`, `"AutoSize"`, `"Hide"` |
| `.Hide()` | `.Hide()` | — | — | Keeps Gui object alive; preferred over Destroy for toggle patterns |
| `.Destroy()` | `.Destroy()` | — | — | Permanently removes window; object becomes invalid after call |
| `.Submit()` | `.Submit(hide := true)` | Object | — | Collects all v-named control values; pass `false` to suppress auto-hide |
| `.Opt()` | `.Opt(options)` | — | — | Apply/remove window options: `"+Resize"`, `"+Disabled"`, `"+Owner" . hwnd` |
| `.SetFont()` | `.SetFont(options?, fontName?)` | — | — | Applies to controls added after this call: `"s10 bold"`, `"s9 norm cRed"` |
| `.OnEvent()` | `.OnEvent(eventName, callback, addRemove?)` | — | — | Window events: `"Close"`, `"Escape"`, `"Size"`, `"DropFiles"`, `"ContextMenu"` |
| `.Flash()` | `.Flash(blink := true)` | — | — | Blinks the window and its taskbar button once; `false` stops blinking |
| `.Maximize()` | `.Maximize()` | — | — | Unhides and maximizes the window |
| `.Minimize()` | `.Minimize()` | — | — | Unhides and minimizes the window |
| `.Restore()` | `.Restore()` | — | — | Unhides and restores a minimized or maximized window to its previous state |
| `.Move()` | `.Move(x?, y?, w?, h?)` | — | — | Moves/resizes the window itself (not a control); omit params to leave axes unchanged |
| `.GetPos()` | `.GetPos(&x?, &y?, &w?, &h?)` | — | — | Retrieves window position including title bar and border |
| `.GetClientPos()` | `.GetClientPos(&x?, &y?, &w?, &h?)` | — | — | Retrieves client-area position (excludes title bar and borders) |
| `.__Enum()` | `.__Enum(n)` | Enumerator | — | Enables `for ctrl in gui` enumeration of all controls |

### Gui Properties

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `.Hwnd` | `.Hwnd` | Integer | — | Read-only window handle; use for `+Owner` and `WinExist("ahk_id " gui.Hwnd)` |
| `.MarginX` | `.MarginX := n` | Integer | — | Left/right margin in pixels for auto-positioned controls |
| `.MarginY` | `.MarginY := n` | Integer | — | Top/bottom margin in pixels for auto-positioned controls |
| `.BackColor` | `.BackColor := color` | String/Integer | — | Sets window background colour: `"EEAA99"` or `"Default"` to reset |
| `.FocusedCtrl` | `.FocusedCtrl` | GuiControl | — | Read-only; returns the currently focused control object |
| `.MenuBar` | `.MenuBar := menuBarObj` | MenuBar | — | Assign a MenuBar object to attach a menu to the window |
| `.Name` | `.Name := str` | String | — | Arbitrary name for the window; accessible via `Gui[Name]` |
| `.Title` | `.Title := str` | String | — | Gets or sets the window title bar text |
| `.__Item` | `gui[nameOrHwnd]` | GuiControl | — | Retrieves a control by its v-name, ClassNN, or HWND |

### Gui Control Add Methods

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `.Add()` | `.Add(controlType, options?, text?)` | GuiControl | — | Generic form; `gui.AddText()` etc. are sugar wrappers around this |
| `.AddText()` | `.AddText(options?, text?)` | GuiControl | — | Static label; `"xm"` resets left margin |
| `.AddEdit()` | `.AddEdit(options?, text?)` | GuiControl | — | Single or multi-line text input; `"vName"` binds to Submit() |
| `.AddButton()` | `.AddButton(options?, text?)` | GuiControl | — | Push button; `"Default"` = Enter key triggers it |
| `.AddCheckBox()` | `.AddCheckBox(options?, text?)` | GuiControl | — | `.Value` = 1/0/–1 (indeterminate); `"Checked"` to start checked |
| `.AddRadio()` | `.AddRadio(options?, text?)` | GuiControl | — | First in group starts a new group; `.Value` = 1 when selected |
| `.AddListBox()` | `.AddListBox(options?, items?)` | GuiControl | — | Single/multi-select listbox; items passed as Array |
| `.AddDropDownList()` | `.AddDropDownList(options?, items?)` | GuiControl | — | DDL dropdown; items passed as Array |
| `.AddComboBox()` | `.AddComboBox(options?, items?)` | GuiControl | — | Editable dropdown; `.Value` = selected text |
| `.AddSlider()` | `.AddSlider(options?)` | GuiControl | — | `.Value` = current integer position |
| `.AddListView()` | `.AddListView(options?, columns?)` | GuiControl | — | Multi-column list; columns = Array of header strings |
| `.AddTreeView()` | `.AddTreeView(options?)` | GuiControl | — | Hierarchical tree control |
| `.AddGroupBox()` | `.AddGroupBox(options?, text?)` | GuiControl | — | Visual grouping box; inner controls need explicit innerX/innerY offsets |
| `.AddTab3()` | `.AddTab3(options?, tabNames?)` | GuiControl | — | Tabbed panel; tabNames = Array; use `.UseTab(n)` to target a tab |
| `.AddPicture()` | `.AddPicture(options?, filename?)` | GuiControl | — | Image control; supports BMP, GIF, JPG, PNG |
| `.AddStatusBar()` | `.AddStatusBar(options?, text?)` | GuiControl | — | Bottom status bar; exposes `.SetText()`, `.SetParts()`, `.SetIcon()` |
| `.AddProgress()` | `.AddProgress(options?)` | GuiControl | — | Progress bar; `.Value` = current position (0–100 by default) |
| `.AddHotkey()` | `.AddHotkey(options?, default?)` | GuiControl | — | Hotkey input control; `.Value` = hotkey string |

### Control Object Methods

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `.OnEvent()` | `.OnEvent(eventName, callback, addRemove?)` | — | — | Control events: `"Click"`, `"Change"`, `"DoubleClick"`, `"ItemSelect"`, `"Focus"`, `"LoseFocus"` |
| `.Opt()` | `.Opt(options)` | — | — | Add/remove options on existing control: `"+Disabled"`, `"-Visible"`, `"+ReadOnly"` |
| `.Move()` | `.Move(x?, y?, w?, h?)` | — | — | Reposition/resize control; pass `""` to leave an axis unchanged |
| `.GetPos()` | `.GetPos(&x?, &y?, &w?, &h?)` | — | — | Retrieves control position relative to the client area |
| `.Focus()` | `.Focus()` | — | — | Gives keyboard focus to the control |
| `.Redraw()` | `.Redraw()` | — | — | Forces a redraw of the control (useful after bulk value updates) |
| `.SetFont()` | `.SetFont(options?, fontName?)` | — | — | Overrides the window font for this specific control only |

### Control Object Properties

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `.Value` | `.Value` | Varies | — | Get/set current value; type varies by control (string, integer, array) |
| `.Enabled` | `.Enabled := true/false` | Boolean | — | Enable/disable the control |
| `.Visible` | `.Visible := true/false` | Boolean | — | Show/hide control without removing it |
| `.ClassNN` | `.ClassNN` | String | — | Read-only Windows class name + instance: e.g., `"Edit1"`, `"Button3"` |
| `.Focused` | `.Focused` | Boolean | — | Read-only; true if this control currently has keyboard focus |
| `.Hwnd` | `.Hwnd` | Integer | — | Read-only window handle for this control |
| `.Name` | `.Name` | String | — | The v-name assigned via `"vMyName"` option; empty string if none |
| `.Text` | `.Text` | String | — | Get/set the display text (separate from `.Value` for some control types) |
| `.Type` | `.Type` | String | — | Read-only control type string: `"Text"`, `"Edit"`, `"Button"`, etc. |
| `.Gui` | `.Gui` | Gui object | — | Read-only reference to the parent Gui window object |

### ListView Control Methods

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `lv.Add()` | `.Add(options?, col1?, col2?, ...)` | Integer (row#) | — | Adds row; returns 1-based row number; `""` for no options |
| `lv.Delete()` | `.Delete(rowNum?)` | — | — | Deletes specific row (1-based) or all rows if no argument |
| `lv.Modify()` | `.Modify(rowNum, options?, col1?, col2?, ...)` | — | — | Edits row text or options; `""` for options to change text only |
| `lv.ModifyCol()` | `.ModifyCol(col?, options?)` | — | — | Resize/configure column; no args = auto-size all columns to content |
| `lv.GetNext()` | `.GetNext(startRow?, mode?)` | Integer (row#) | — | Finds next selected/checked/focused row; mode: `"Selected"`, `"Focused"`, `"Checked"`; returns 0 at end |
| `lv.GetText()` | `.GetText(row, col?)` | String | — | Gets cell text; row 0 retrieves column header text; col is 1-based; col 0 does not exist |
| `lv.GetCount()` | `.GetCount(mode?)` | Integer | — | Row count; `"Col"` for column count; `"Selected"` for selected row count |

### TreeView Control Methods

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `tv.Add()` | `.Add(text, parentId?, options?)` | Integer (ID) | — | Adds node; parentId `0` = top-level; returns item ID integer |
| `tv.Delete()` | `.Delete(itemId?)` | — | — | Deletes item and all descendants; no arg = delete all |
| `tv.Modify()` | `.Modify(itemId, options?, newText?)` | — | — | Changes options or text: `"Expand"`, `"Select"`, `"Vis"`, `"Bold"` |
| `tv.GetNext()` | `.GetNext(itemId?, mode?)` | Integer (ID) | — | Traverses nodes; `"Full"` = depth-first all nodes; returns 0 at end |
| `tv.GetSelection()` | `.GetSelection()` | Integer (ID) | — | Returns currently selected item ID; 0 if none |
| `tv.GetText()` | `.GetText(itemId)` | String | — | Returns text label of item |
| `tv.GetParent()` | `.GetParent(itemId)` | Integer (ID) | — | Returns parent item ID; 0 if top-level |

### StatusBar Methods

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `sb.SetText()` | `.SetText(text, part?, style?)` | — | — | Sets text of the specified part (1-based, default 1); style: 0=lowered, 1=none, 2=raised |
| `sb.SetParts()` | `.SetParts(width1, width2?, ...)` | Integer | — | Sets part widths in pixels; returns the number of parts created |
| `sb.SetIcon()` | `.SetIcon(filename, iconNum?, partNum?)` | Boolean | — | Assigns an icon to a part (1-based); returns 1 on success |

### Tab3 Methods

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `tab.UseTab()` | `.UseTab(tabName/Index?, exact?)` | — | — | Directs subsequent `gui.Add()` calls into the specified tab; `UseTab()` with no arg exits tab scope |

### Helper Functions

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `GuiFromHwnd()` | `GuiFromHwnd(hwnd, recurse?)` | Gui or "" | — | Returns Gui object from a window handle; `recurse=true` searches child windows |
| `GuiCtrlFromHwnd()` | `GuiCtrlFromHwnd(hwnd)` | GuiControl or "" | — | Returns GuiControl object from a control's HWND; returns "" if not a Gui control |
| `GuiForm()` | `GuiForm(x, y, w, h, extraParams?)` | String | — | User-defined helper; returns `"x{} y{} w{} h{} extras"` position string for `gui.Add()` |

## AHK V2 CONSTRAINTS

- **Class encapsulation is mandatory** — all GUI code must reside inside a class or named function; bare top-level `Gui()` calls at TIER 2+ prevent `.Bind(this)` from resolving the class instance.
- **Map() for all control storage** — `this.controls := Map()` is the only safe container for named control references; `{}` object literals lack `.Has()`, `.Delete()`, and safe-access semantics — using `{}` causes silent data loss on dynamic key lookups.
- **`.Bind(this)` is mandatory for class-method event handlers** — AHK v2 closures do not automatically capture the class instance; omitting `.Bind(this)` leaves `this` undefined inside the handler, producing an UnsetError at call time.
  - ✗ `ctrl.OnEvent("Click", this.HandleClick)` — `this` is undefined inside the handler
  - ✓ `ctrl.OnEvent("Click", this.HandleClick.Bind(this))` — correct instance capture
- **`gui.Close()` does not exist** — calling it throws MethodError; use `gui.Hide()` to keep the window object alive for later `Show()`, or `gui.Destroy()` to permanently remove it.
  - ✗ `gui.Close()` — MethodError at runtime
  - ✓ `gui.Hide()` — hides window, keeps Gui object valid
- **Never place a width on a Section option** — `"Section w300"` causes cumulative horizontal drift in AHK v2's implicit positioning engine; use `"Section"` alone or `"xm Section"` to reset both section and left margin simultaneously.
  - ✗ `gui.AddText("Section w300", "Header")` — cumulative layout drift on every subsequent control
  - ✓ `gui.AddText("xm Section", "Header")` — correct margin reset
- **Always start each section's first control with `xm`** — AHK v2 accumulates the previous control's right edge by default; omitting `xm` after a Section causes every subsequent control to drift rightward.
- **Arrow syntax is single-expression only** — `(*) => expr` is valid; `(*) => { stmt1; stmt2 }` multi-line blocks cause a load-time SyntaxError in AHK v2; use separate named bound methods for multi-statement handlers.
  - ✗ `btn.OnEvent("Click", (*) => { saved := gui.Submit(); MsgBox(saved.Name) })` — SyntaxError
  - ✓ Separate named method + `.Bind(this)` — correct approach for multi-statement handlers
- **`gui.Submit()` hides by default** — `Submit()` without arguments collects v-named control values AND calls `Hide()`; pass `false` to suppress the hide; never call `Hide()` after `Submit()` — it is redundant.
- **Build controls once, guard with `_built`** — calling `CreateControls()` on every `Show()` duplicates controls; set `_built := true` after first build and check before rebuilding.
- **Owner window sequencing is order-sensitive** — call `ownerGui.Opt("+Disabled")` BEFORE showing the owned window; call `ownerGui.Opt("-Disabled")` BEFORE destroying or hiding the owned window — never after; the owner stays permanently disabled if still disabled when the owned window closes.
  - ✗ `ownedGui.Destroy(); ownerGui.Opt("-Disabled")` — owner stays disabled permanently
  - ✓ `ownerGui.Opt("-Disabled"); ownedGui.Destroy()` — correct sequence
- **ListView and TreeView indices are 1-based** — `lv.GetText(1, 1)` is the first data cell; row 0 retrieves column header text; col 0 does not exist; `lv.GetNext()` returns 0 only as an end-of-list sentinel.
  - ✗ `lv.GetText(1, 0)` — col 0 does not exist; runtime produces empty string or error
  - ✓ `lv.GetText(1, 1)` — first column is index 1
- **Forward-loop ListView delete shifts row numbers** — deleting `A_Index` inside a `Loop GetCount()` skips rows because deletion shifts all indices; use a `while (rowNum := lv.GetNext(rowNum, "Selected"))` pattern with `rowNum := 0` reset after each delete.
- **HandleSize must guard against minimized state** — `minMax = -1` means minimized; calling `ctrl.Move()` while minimized causes controls to vanish on restore; always `return` early when `minMax = -1`.

Safe-access priority order for GUI control data:
  1. `ctrl.Value` — direct property read for known-present controls stored in `Map()`
  2. `this.controls.Get("key", "")` — when control existence may vary (dynamic builds)
  3. `this.controls.Has("key")` — when if/else branch logic differs for present vs absent
  4. `try/catch` — only for `lv.GetText()` or `tv.GetText()` on IDs that may have been deleted

Unset variable handling: always verify control references are stored in `Map()` before access; calling methods on a missing `Map()` key throws `UnsetItemError` — guard with `.Has()` or `.Get()` for dynamically built control sets.

Resource lifecycle: `gui.Destroy()` is irreversible — the Gui object becomes invalid after the call. Prefer `gui.Hide()` for toggling visibility to avoid recreating the full control tree on re-open.

## AGENT QA CHECKLIST

- [ ] Did I use `gui.Hide()` or `gui.Destroy()` instead of the non-existent `gui.Close()`?
- [ ] Did I store all control references in `Map()` and use `.Bind(this)` for all class-method event handlers?
- [ ] Are all arrow-function callbacks single-expression only — no `(*) => { stmt1; stmt2 }` multi-line blocks?
- [ ] Did I call `ownerGui.Opt("-Disabled")` BEFORE `ownedGui.Destroy()` or `ownedGui.Hide()` — not after?

## RUNTIME ERROR MAPPING

| Exception Class | Trigger Condition | Detection Code | Fix |
|----------------|-------------------|----------------|-----|
| `MethodError` | Calling `gui.Close()` anywhere in code | `e.Message` contains `"Close"` | Replace with `gui.Hide()` to keep the object alive or `gui.Destroy()` to remove permanently |
| `UnsetError` | Accessing `this.controls["key"]` when the key was never stored, or `.Bind(this)` omitted so `this` is unset inside a handler | `e.Message` contains the missing key name or `"this"` | Guard control reads with `.Has()` or `.Get()`; add `.Bind(this)` to all class-method callbacks |
| `SyntaxError` (load-time) | Multi-line arrow callback `(*) => { stmt1; stmt2 }` used as an event handler | Script fails to load; error line points to the `=>` token | Replace with a separate named method and `.Bind(this)` for any handler requiring more than one statement |

## TIER 1 — Basic GUI Creation
> METHODS COVERED: Gui() · SetFont() · AddText() · AddButton() · OnEvent() · Show() · Hide()

Covers the minimal `Gui()` constructor pattern: creating a window, setting font, handling Close/Escape events, adding one or two controls, and calling `Show()`. Use this tier for single-purpose dialogs, simple notifications, and prototype windows where layout precision is not required. Do not apply LayoutCalculator or Section patterns at this tier — that is over-engineering for 1–2 controls.

```ahk
; ✓ Minimal class-based GUI — TIER 1 pattern; class encapsulation is required even at this tier
SimpleDialog()

class SimpleDialog {
    __New() {
        this.gui := Gui(, "My Dialog")
        this.gui.SetFont("s10")
        this.gui.OnEvent("Close",  (*) => this.gui.Hide())
        this.gui.OnEvent("Escape", (*) => this.gui.Hide())
        this.gui.AddText(, "Hello World!")
        this.gui.AddButton("Default w100", "OK").OnEvent("Click", (*) => this.gui.Hide())
        this.gui.Show()
    }
}

; ✗ v1 command syntax — does not exist in v2; causes load-time error
; Gui, Add, Text,, Hello World!    ; → MethodError (v1 command)
; Gui, Show                        ; → MethodError (v1 command)
; new SimpleDialog()               ; → NameError at runtime — new keyword does not exist in v2
```

## TIER 2 — Controls, Event Handling, ListView, TreeView, Multi-Window
> METHODS COVERED: Gui() · AddEdit() · AddCheckBox() · AddListView() · AddTreeView() · Submit() · OnEvent() · Bind() · Hotkey() · HotIfWinExist() · WinExist() · IsObject() · Opt() · lv.Add() · lv.Delete() · lv.Modify() · lv.ModifyCol() · lv.GetNext() · lv.GetText() · lv.GetCount() · tv.Add() · tv.Delete() · tv.Modify() · tv.GetNext() · tv.GetSelection() · tv.GetText() · tv.GetParent()

Introduces the full class-based GUI template: control references stored in `Map()`, named event handlers with `.Bind(this)`, `Submit()` for reading v-named control values, and hotkey integration for show/hide toggling. Also covers ListView and TreeView CRUD operations, and modal owned-window management with the owner-enable/disable sequence. Use this tier whenever the GUI needs to read user input, respond to multiple controls, or maintain state between interactions.

```ahk
; ✓ Full class template with Map() controls, events, and hotkeys
FormGui()

class FormGui {
    __New() {
        this.gui      := Gui("+Resize", "Input Form")
        this.controls := Map()                          ; ✓ Map() for storage, never {}
        this.gui.SetFont("s10")
        this.gui.OnEvent("Close",  (*) => this.gui.Hide())
        this.gui.OnEvent("Escape", (*) => this.gui.Hide())
        this.CreateControls()
        this.SetupHotkeys()
    }

    CreateControls() {
        this.gui.AddText(, "Full Name:")
        this.controls["nameEdit"]  := this.gui.AddEdit("vUserName w200")
        this.gui.AddText(, "Email:")
        this.controls["emailEdit"] := this.gui.AddEdit("vUserEmail w200")

        this.controls["submitBtn"] := this.gui.AddButton("Default w200", "Submit")
        ; ✓ .Bind(this) is required — omitting it leaves `this` undefined inside the handler
        this.controls["submitBtn"].OnEvent("Click", this.HandleSubmit.Bind(this))
        this.gui.Show()
    }

    HandleSubmit(*) {
        ; Form A — let Submit() hide the window (preferred, concise):
        saved := this.gui.Submit()            ; ✓ hides window AND collects v-named controls
        MsgBox("Name: " saved.UserName "`nEmail: " saved.UserEmail)

        ; Form B — suppress Submit()'s auto-hide and hide manually:
        ; saved := this.gui.Submit(false)     ; false = do NOT hide yet
        ; MsgBox("Name: " saved.UserName "`nEmail: " saved.UserEmail)
        ; this.gui.Hide()
    }

    Toggle(*) {
        if WinExist("ahk_id " this.gui.Hwnd)
            this.gui.Hide()
        else
            this.gui.Show()
    }

    SetupHotkeys() {
        Hotkey("^m", this.Toggle.Bind(this))
        HotIfWinExist("ahk_id " this.gui.Hwnd)
        Hotkey("^Escape", this.Toggle.Bind(this), "On")
        HotIfWinExist()
    }
}

; ✗ Arrow syntax with multi-line block in event handler — INVALID in AHK v2
; this.controls["submitBtn"].OnEvent("Click", (*) => {
;     saved := this.gui.Submit()
;     MsgBox(saved.UserName)
; })                                          ; → SyntaxError (multi-line arrow block)

; === ListView and TreeView CRUD ===

; --- ListView: columns, Add, Delete, Modify, GetNext loop, enumeration ---
class ListViewCrudDemo {
    __New() {
        this.gui      := Gui("+Resize", "ListView CRUD")
        this.controls := Map()
        this.gui.SetFont("s10")
        this.gui.OnEvent("Close",  (*) => this.gui.Hide())
        this.gui.OnEvent("Escape", (*) => this.gui.Hide())
        this.CreateControls()
        this.gui.Show("w520 h360")
    }

    CreateControls() {
        ; ✓ Column headers passed as an Array in the third parameter
        lv := this.gui.AddListView("xm w500 h260 Grid -ReadOnly",
                                    ["Name", "Age", "City"])
        this.controls["lv"] := lv
        lv.Add("", "Alice", "30", "Taipei")      ; ✓ Add() returns new row number (1-based)
        lv.Add("", "Bob",   "25", "Kaohsiung")
        lv.Add("", "Carol", "35", "Tainan")
        lv.ModifyCol()                            ; ✓ auto-size all columns to content

        this.controls["addBtn"] := this.gui.AddButton("xm   w118 h28", "Add Row")
        this.controls["delBtn"] := this.gui.AddButton("x+4  w118 h28", "Delete")
        this.controls["edtBtn"] := this.gui.AddButton("x+4  w118 h28", "Edit")
        this.controls["clrBtn"] := this.gui.AddButton("x+4  w118 h28", "Clear All")
        this.controls["addBtn"].OnEvent("Click", this.AddRow.Bind(this))
        this.controls["delBtn"].OnEvent("Click", this.DeleteRow.Bind(this))
        this.controls["edtBtn"].OnEvent("Click", this.EditRow.Bind(this))
        this.controls["clrBtn"].OnEvent("Click", this.ClearAll.Bind(this))
        this.controls["lv"].OnEvent("DoubleClick", this.HandleDoubleClick.Bind(this))
    }

    AddRow(*) {
        lv     := this.controls["lv"]
        rowNum := lv.Add("", "New Person", "0", "Unknown")   ; ✓ 1-based row number
        lv.Modify(rowNum, "Select Focus Vis")   ; ✓ select it and scroll into view
        lv.ModifyCol()
    }

    DeleteRow(*) {
        lv     := this.controls["lv"]
        rowNum := 0
        ; ✓ GetNext(start, "Selected") finds the next selected row; reset to 0 after each
        ; delete because row numbers shift downward after removal
        while (rowNum := lv.GetNext(rowNum, "Selected")) {
            lv.Delete(rowNum)
            rowNum := 0
        }
    }

    EditRow(*) {
        lv     := this.controls["lv"]
        rowNum := lv.GetNext(0, "Selected")    ; ✓ GetNext(0, ...) = start from row 1
        if !rowNum
            return
        ; ✓ GetText(row, col): both row and col are 1-based integers
        ; ✓ Modify(row, opts, col1, col2, col3): pass "" for opts to change only text
        lv.Modify(rowNum, "",
                  "Edited: " lv.GetText(rowNum, 1),
                  lv.GetText(rowNum, 2),
                  lv.GetText(rowNum, 3))
        lv.ModifyCol()
    }

    ClearAll(*) {
        this.controls["lv"].Delete()             ; ✓ Delete() with no arg removes all rows
    }

    HandleDoubleClick(ctrlObj, rowNum, *) {
        if !rowNum
            return
        MsgBox("Row " rowNum ": " ctrlObj.GetText(rowNum, 1))
    }

    ; ✓ Sequential enumeration: Loop GetCount() gives A_Index = 1..N (1-based)
    EnumerateAll() {
        lv := this.controls["lv"]
        output := ""
        Loop lv.GetCount() {
            output .= A_Index ": "
                    . lv.GetText(A_Index, 1) ", "
                    . lv.GetText(A_Index, 2) "`n"
        }
        return output
    }
}

; --- TreeView: Add with parent IDs, GetNext traversal, Modify, Delete ---
class TreeViewCrudDemo {
    __New() {
        this.gui      := Gui("+Resize", "TreeView CRUD")
        this.controls := Map()
        this.nodeMap  := Map()             ; ✓ Map() to track label → item ID for later use
        this.gui.SetFont("s10")
        this.gui.OnEvent("Close",  (*) => this.gui.Hide())
        this.gui.OnEvent("Escape", (*) => this.gui.Hide())
        this.CreateControls()
        this.gui.Show("w360 h420")
    }

    CreateControls() {
        tv := this.gui.AddTreeView("xm w340 h320")
        this.controls["tv"] := tv

        ; ✓ Add(text, parentItemID, options) — parentItemID 0 = top-level; returns item ID
        fruitsId := tv.Add("Fruits",     0,        "Expand")
        appleId  := tv.Add("Apple",      fruitsId, "")       ; child of Fruits
        tv.Add("Banana",     fruitsId, "")
        vegsId   := tv.Add("Vegetables", 0,        "Expand")
        tv.Add("Carrot",   vegsId, "")
        tv.Add("Broccoli", vegsId, "")

        ; ✓ Store IDs in Map() to retrieve or modify specific nodes later
        this.nodeMap["fruits"] := fruitsId
        this.nodeMap["apple"]  := appleId

        this.controls["addBtn"] := this.gui.AddButton("xm    w162 h28", "Add Child")
        this.controls["delBtn"] := this.gui.AddButton("x+10  w162 h28", "Delete Selected")
        this.controls["addBtn"].OnEvent("Click", this.AddChild.Bind(this))
        this.controls["delBtn"].OnEvent("Click", this.DeleteSelected.Bind(this))
        tv.OnEvent("ItemSelect", this.HandleSelect.Bind(this))
    }

    AddChild(*) {
        tv       := this.controls["tv"]
        parentId := tv.GetSelection()          ; ✓ GetSelection() returns selected item ID
        if !parentId
            return
        newId := tv.Add("New Item", parentId, "Select Vis")  ; ✓ Vis = scroll into view
        tv.Modify(parentId, "Expand")                         ; expand parent to show child
    }

    DeleteSelected(*) {
        tv    := this.controls["tv"]
        selId := tv.GetSelection()
        if !selId
            return
        tv.Delete(selId)                  ; ✓ Delete(itemID) removes item + all descendants
    }

    HandleSelect(ctrlObj, itemId, *) {
        if !itemId
            return
        MsgBox("Selected: " ctrlObj.GetText(itemId))   ; ✓ GetText(itemID) returns label
    }

    ; ✓ Depth-first traversal: GetNext(id, "Full") visits every node; 0 = end of tree
    TraverseAll() {
        tv     := this.controls["tv"]
        itemId := 0
        output := ""
        Loop {
            itemId := tv.GetNext(itemId, "Full")
            if !itemId
                break
            indent := tv.GetParent(itemId) ? "  " : ""
            output .= indent tv.GetText(itemId) "`n"
        }
        return output
    }
}

; ✗ 0-based column index in ListView
; lv.GetText(1, 0)            ; → col 0 does not exist; first column is 1
; ✓ row 0 is valid — retrieves column header text, not a data row:
; lv.GetText(0, 1)            ; → column 1 header text (row 0 = column headers, not a data row)
; ✗ Discarding the item ID returned by tv.Add()
; tv.Add("Child", parentId)   ; → cannot modify or delete later without the ID
; ✗ Forward-loop delete in ListView without resetting the row counter
; Loop lv.GetCount() { lv.Delete(A_Index) }   ; → row numbers shift after each delete;
;                                                  use GetNext while-loop instead

; === Multi-Window Management — Owner / Owned Windows ===
;
; +Owner rules:
;   - childGui.Opt("+Owner" . parentGui.Hwnd) makes childGui owned by parentGui
;   - Owned window: no taskbar button; always on top of its owner
;   - Auto-destroyed when its owner is destroyed (requires same process)
;   - Must call parentGui.Opt("+Disabled") BEFORE showing child (blocks parent input)
;   - Must call parentGui.Opt("-Disabled") BEFORE destroying/hiding child — never after
;     (if owner is still disabled when the owned window closes, owner stays disabled)

MainAppWindow()

class MainAppWindow {
    __New() {
        this.gui    := Gui("+Resize", "Main Window")
        this.dialog := ""            ; ✓ sentinel value — replaced when dialog is open
        this.gui.SetFont("s10")
        this.gui.OnEvent("Close",  (*) => this.gui.Hide())
        this.gui.OnEvent("Escape", (*) => this.gui.Hide())
        this.gui.AddText("xm w360", "Main application window.")
        this.gui.AddButton("xm w180 h30", "Open Settings...").OnEvent("Click",
            this.OpenSettings.Bind(this))
        this.gui.Show("w400 h150")
    }

    OpenSettings(*) {
        if IsObject(this.dialog)   ; ✓ prevent duplicate owned windows
            return
        ; ✓ CRITICAL: disable owner BEFORE showing owned window
        this.gui.Opt("+Disabled")
        this.dialog := OwnedSettingsDialog(this.gui, this.OnSettingsClosed.Bind(this))
    }

    ; Callback invoked by the owned dialog when it finishes
    OnSettingsClosed(*) {
        ; ✓ CRITICAL: re-enable owner BEFORE destroying owned window
        ;   Destroying while owner is still disabled leaves owner permanently disabled
        this.gui.Opt("-Disabled")
        if IsObject(this.dialog) {
            this.dialog.Destroy()
            this.dialog := ""
        }
    }
}

class OwnedSettingsDialog {
    __New(ownerGui, closeCb) {
        this.closeCb := closeCb
        this.gui     := Gui(, "Settings")
        ; ✓ Pass owner's HWND to +Owner option string concatenation
        this.gui.Opt("+Owner" . ownerGui.Hwnd)
        this.gui.SetFont("s10")
        this.gui.OnEvent("Close",  this.HandleClose.Bind(this))
        this.gui.OnEvent("Escape", this.HandleClose.Bind(this))
        this.gui.AddText("xm w280", "Settings for the application.")
        this.gui.AddButton("xm w130 h28 Default", "OK").OnEvent("Click",
            this.HandleClose.Bind(this))
        this.gui.AddButton("x+8 w130 h28", "Cancel").OnEvent("Click",
            this.HandleClose.Bind(this))
        this.gui.Show("w320 h160")
    }

    HandleClose(*) {
        ; ✓ Delegate close logic to owner — do NOT re-enable owner here;
        ;   the owner is responsible for its own Disabled state
        this.closeCb.Call()
    }

    Destroy() {
        this.gui.Destroy()
    }
}

; ✗ Re-enabling owner AFTER destroying the owned window
; ownedGui.Destroy()              ; → destroys while owner still disabled → stuck disabled
; ownerGui.Opt("-Disabled")       ; → too late; must come BEFORE Destroy() or Hide()
;
; ✗ Setting +Owner after the owned window is already shown
; ownedGui.Show()                 ; → must set +Owner before Show(), not after
; ownedGui.Opt("+Owner" ...)      ; → may have no effect once the window is visible
;
; ✗ Using WinExist to guard against duplicate owned windows
; if WinExist("Settings")         ; → unreliable for same-script windows; use an object ref
```

## TIER 3 — Layout and Positioning Management
> METHODS COVERED: AddText() · AddEdit() · AddCheckBox() · AddButton() · Section · xm · ctrl.Move() · OnEvent("Size") · PositionValidator.ValidatePositioning() · PositionValidator.ValidateObjectLiterals() · RegExMatch()

Covers AHK v2's cumulative positioning system: `xm`/`ym` margin resets, Section grouping, multi-column layouts with `x+n` relative offsets, the PositionValidator class for pre-output error detection, and responsive window resizing via `OnEvent("Size")`. Use this tier for any GUI with more than one logical section, more than five controls, or inline multi-column button rows. Always run PositionValidator mentally before outputting code at this tier.

```ahk
; POSITIONING RULES — apply before writing any multi-section GUI:
;
; ✓ Section header always uses xm to reset left margin
;   this.gui.AddText("xm Section", "Section Header")
;
; ✓ First control in each section resets with xm
;   this.controls["ctrl"] := this.gui.AddEdit("xm w200")
;
; ✓ Sibling controls in same row use relative x+n offset
;   this.controls["btn1"] := this.gui.AddButton("xm w100",  "Save")
;   this.controls["btn2"] := this.gui.AddButton("x+10 w100", "Cancel")
;
; ✗ Width on Section tag causes cumulative drift
;   this.gui.AddText("Section w300", "Header")    ; → NEVER put w### on Section
;
; ✗ Missing xm on first control after section header
;   this.controls["ctrl"] := this.gui.AddEdit("w200")  ; → inherits wrong position

MultiSectionGui()

class MultiSectionGui {
    __New() {
        this.gui      := Gui("+Resize", "Settings")
        this.controls := Map()
        this.gui.SetFont("s10")
        this.gui.OnEvent("Close",  (*) => this.gui.Hide())
        this.gui.OnEvent("Escape", (*) => this.gui.Hide())
        this.CreateControls()
        this.gui.Show("w380")
    }

    CreateControls() {
        ; --- Section 1: User Info ---
        this.gui.AddText("xm Section", "User Information")         ; ✓ xm Section
        this.controls["nameEdit"]  := this.gui.AddEdit("xm w340 vUserName")
        this.gui.AddText("xm", "Email:")
        this.controls["emailEdit"] := this.gui.AddEdit("xm w340 vUserEmail")

        ; --- Section 2: Preferences ---
        this.gui.AddText("xm Section", "Preferences")              ; ✓ xm resets
        this.controls["notifyChk"]   := this.gui.AddCheckBox("xm", "Enable Notifications")
        this.controls["autoSaveChk"] := this.gui.AddCheckBox("xm", "Auto-save")

        ; --- Section 3: Actions (multi-column row) ---
        this.gui.AddText("xm Section", "")
        this.controls["saveBtn"]   := this.gui.AddButton("xm w160",    "Save")
        this.controls["cancelBtn"] := this.gui.AddButton("x+20 w160",  "Cancel") ; ✓ x+n
        this.controls["saveBtn"].OnEvent("Click",   this.HandleSave.Bind(this))
        this.controls["cancelBtn"].OnEvent("Click", (*) => this.gui.Hide())
    }

    HandleSave(*) {
        saved := this.gui.Submit(false)   ; ✓ false = keep window visible while saving
        MsgBox("Saved: " saved.UserName)
    }
}

; --- PositionValidator: run this mentally before outputting any TIER 3+ code ---
class PositionValidator {
    static ValidatePositioning(guiCode) {
        errors := []
        if RegExMatch(guiCode, "Section\s+w\d+")
            errors.Push("CRITICAL: Never use 'Section w###' — use 'Section' alone")
        sectionCount := 0
        xmResetCount := 0
        pos := 1
        while (pos := RegExMatch(guiCode, "Section", &m, pos)) {
            sectionCount++
            pos := m.Pos + m.Len
        }
        pos := 1
        while (pos := RegExMatch(guiCode, "\bxm\b", &m, pos)) {
            xmResetCount++
            pos := m.Pos + m.Len
        }
        if (sectionCount > 0 && xmResetCount < sectionCount)
            errors.Push("WARNING: Sections without xm reset may cause positioning drift")
        return errors
    }

    static ValidateObjectLiterals(code) {
        errors := []
        if RegExMatch(code, "=>\s*\{[^}]*\n[^}]*\}")
            errors.Push("CRITICAL: Never use arrow syntax (=>) with multi-line blocks")
        return errors
    }
}

; === Responsive Window — gui.OnEvent("Size", Handler) ===
;
; Size event callback signature (class method form):
;   HandleSize(guiObj, minMax, width, height)
;     guiObj:  the Gui object that was resized
;     minMax:  0 = normal/restored, 1 = maximized, -1 = minimized
;     width, height: new client-area size (title bar and borders excluded)
;
; Use ctrl.Move(x, y, w, h) to reposition or resize controls.
; Pass "" for any axis you want to leave unchanged.
; Always check (minMax = -1) first and return early — moving controls while the
; window is minimized causes them to disappear on restore.

ResizableLogGui()

class ResizableLogGui {
    __New() {
        this.pad      := 10
        this.gui      := Gui("+Resize +MinSize300x200", "Resizable Log Window")
        this.controls := Map()
        this.gui.SetFont("s10")
        this.gui.OnEvent("Close",  (*) => this.gui.Hide())
        this.gui.OnEvent("Escape", (*) => this.gui.Hide())
        ; ✓ Register Size handler with .Bind(this) before Show()
        this.gui.OnEvent("Size", this.HandleSize.Bind(this))
        this.CreateControls()
        this.gui.Show("w600 h400")
    }

    CreateControls() {
        pad := this.pad
        ; Log area — occupies most of the window; stretches both dimensions on resize
        this.controls["log"] := this.gui.AddEdit(
            "xm w580 h330 ReadOnly -Wrap", "")
        ; Status text — docked to bottom, stretches horizontally
        this.controls["status"] := this.gui.AddText(
            "xm w580 h20 +Border", "Ready")
        ; Clear button — anchored to bottom-right corner on resize
        this.controls["clearBtn"] := this.gui.AddButton(
            "xm w80 h28", "Clear")
        this.controls["clearBtn"].OnEvent("Click",
            (*) => (this.controls["log"].Value := ""))
    }

    ; ✓ Multi-statement resize handler must be a named bound method
    HandleSize(guiObj, minMax, width, height) {
        if (minMax = -1)         ; minimized — skip layout recalculation
            return
        pad     := this.pad
        ; Log area fills available space, leaving rows for status bar and button
        logH    := height - pad * 5 - 20 - 28
        logW    := width  - pad * 2
        statusY := pad + logH + pad
        btnX    := width - pad - 80
        btnY    := statusY + 20 + pad
        ; ✓ Move(x, y, w, h): all four axes explicit; pass "" to leave one unchanged
        this.controls["log"].Move(pad, pad, logW, logH)
        this.controls["status"].Move(pad, statusY, logW, 20)
        this.controls["clearBtn"].Move(btnX, btnY, 80, 28)
    }
}

; ✗ Arrow syntax for multi-statement Size handler — INVALID in AHK v2
; this.gui.OnEvent("Size", (g, mm, w, h) => {   ; → SyntaxError (multi-line arrow block)
;     this.controls["log"].Move(...)
;     this.controls["clearBtn"].Move(...)
; })
;
; ✗ Not checking minMax = -1 before moving controls
; HandleSize(guiObj, minMax, width, height) {
;     this.controls["log"].Move(pad, pad, width - 20, height - 80)  ; → controls vanish on restore
; }
;
; ✗ Using hard-coded sizes instead of computing from the width/height parameters
; HandleSize(guiObj, minMax, width, height) {
;     this.controls["log"].Move(10, 10, 580, 330)  ; → ignores new width/height entirely
; }
```

## TIER 4 — Mathematical Layout System
> METHODS COVERED: LayoutCalculator.Calculate() · LayoutCalculator.Validate() · LayoutCalculator.FormatReport() · LayoutAwareGui.__New() · AddElement() · ParseDimension() · CalculateLayout() · CreateControls() · ShowLayoutReport() · Show() · MsgBox()

Introduces the `LayoutCalculator` static class and `LayoutAwareGui` base class for data-driven, mathematically validated GUI construction. Every element's x, y, width, and height is computed from window dimensions and padding; overlap and boundary checks run automatically; a layout report can be displayed on demand. Use this tier for forms with dynamic element lists, settings panels requiring pixel-perfect alignment, or any scenario where positions must be auditable and reproducible.

```ahk
; LayoutCalculator — static utility for coordinate calculation and validation
class LayoutCalculator {
    static Calculate(windowWidth, windowHeight, padding, elements) {
        result   := Map()
        currentY := padding

        for element in elements {
            elementResult := Map()
            elementResult["x"]            := padding
            elementResult["xReason"]      := "Reset to margin (xm) — prevents drift"
            elementResult["y"]            := currentY
            elementResult["yReason"]      := "Accumulated from previous elements + padding"
            elementResult["width"]        := element.Has("width")
                                             ? element["width"]
                                             : windowWidth - (padding * 2)
            elementResult["height"]       := element["height"]
            elementResult["widthReason"]  := element.Has("width")
                                             ? "Specified width"
                                             : "Full width minus left+right padding"
            elementResult["heightReason"] := "Specified height"
            elementResult["ahkPosition"]  := "xm"
            result[element["id"]] := elementResult
            currentY += element["height"] + (padding // 2)
        }
        return result
    }

    static Validate(layout, windowWidth, windowHeight) {
        overlapCheck  := "No overlaps detected"
        boundaryCheck := "All elements within boundaries"
        for elementId, element in layout {
            if (element["x"] + element["width"] > windowWidth)
                boundaryCheck := "Element " elementId " exceeds right boundary"
            if (element["y"] + element["height"] > windowHeight)
                boundaryCheck := "Element " elementId " exceeds bottom boundary"
        }
        return Map("overlap", overlapCheck, "boundary", boundaryCheck)
    }

    static FormatReport(layout, validation) {
        sep    := "=================================================="
        report := "GUI Layout Analysis`n" sep "`n`n"
        for elementId, element in layout {
            report .= "Element: " elementId "`n"
            report .= "  X:      " element["x"]      " px (" element["xReason"]      ")`n"
            report .= "  Y:      " element["y"]      " px (" element["yReason"]      ")`n"
            report .= "  Width:  " element["width"]  " px (" element["widthReason"]  ")`n"
            report .= "  Height: " element["height"] " px (" element["heightReason"] ")`n`n"
        }
        report .= "Validation:`n"
        report .= "  Overlap:  " validation["overlap"]  "`n"
        report .= "  Boundary: " validation["boundary"] "`n"
        return report
    }
}

; LayoutAwareGui — base class; extend this for data-driven GUIs
class LayoutAwareGui {
    __New(title := "Layout GUI", width := 400, height := 300, padding := 10) {
        this.windowWidth  := width
        this.windowHeight := height
        this.padding      := padding
        this.elements     := []
        this.layout       := Map()
        this.controls     := Map()
        this._built       := false
        this.gui          := Gui("+Resize", title)
        this.gui.MarginX  := 0
        this.gui.MarginY  := 0
        this.gui.OnEvent("Close",  (*) => this.gui.Hide())
        this.gui.OnEvent("Escape", (*) => this.gui.Hide())
    }

    AddElement(id, type, options := "", text := "") {
        width  := this.ParseDimension(options, "w", this.windowWidth - (this.padding * 2))
        height := this.ParseDimension(options, "h", 25)
        this.elements.Push(Map(
            "id",      id,
            "type",    type,
            "width",   width,
            "height",  height,
            "options", options,
            "text",    text
        ))
        return this
    }

    ParseDimension(options, prefix, defaultVal) {
        if RegExMatch(options, prefix "(\d+)", &match)
            return Integer(match[1])
        return defaultVal
    }

    CalculateLayout() {
        this.layout := LayoutCalculator.Calculate(
            this.windowWidth, this.windowHeight, this.padding, this.elements
        )
        return this
    }

    CreateControls() {
        for element in this.elements {
            if !this.layout.Has(element["id"])
                continue
            pos     := this.layout[element["id"]]
            options := "x" pos["x"] " y" pos["y"] " w" pos["width"] " h" pos["height"]
            switch element["type"] {
                case "Text":     this.controls[element["id"]] := this.gui.AddText(options,     element["text"])
                case "Edit":     this.controls[element["id"]] := this.gui.AddEdit(options,     element["text"])
                case "Button":   this.controls[element["id"]] := this.gui.AddButton(options,   element["text"])
                case "CheckBox": this.controls[element["id"]] := this.gui.AddCheckBox(options, element["text"])
            }
        }
        return this
    }

    ShowLayoutReport() {
        validation := LayoutCalculator.Validate(this.layout, this.windowWidth, this.windowHeight)
        report     := LayoutCalculator.FormatReport(this.layout, validation)
        MsgBox(report, "Layout Analysis", "Iconi")
        return this
    }

    Show() {
        if !this._built {
            this.CalculateLayout()
            this.CreateControls()
            this._built := true
        }
        this.gui.Show("w" this.windowWidth " h" this.windowHeight)
        return this
    }
}

; Usage: extend LayoutAwareGui, build elements, wire events, then show
SettingsPanel()

class SettingsPanel extends LayoutAwareGui {
    __New() {
        super.__New("Application Settings", 400, 320, 12)
        this.AddElement("header",    "Text",   "h30",      "User Settings")
            .AddElement("nameLabel", "Text",   "h20",      "Full Name:")
            .AddElement("nameEdit",  "Edit",   "h25",      "")
            .AddElement("emailLbl",  "Text",   "h20",      "Email:")
            .AddElement("emailEdit", "Edit",   "h25",      "")
            .AddElement("saveBtn",   "Button", "w160 h32", "Save Settings")

        ; Build controls first, then wire events, then show — correct order
        this.CalculateLayout()
        this.CreateControls()
        this._built := true
        this.controls["saveBtn"].OnEvent("Click", this.HandleSave.Bind(this))
        this.gui.Show("w" this.windowWidth " h" this.windowHeight)
        this.ShowLayoutReport()    ; ✓ always show report in TIER 4 examples
    }

    HandleSave(*) {
        MsgBox("Settings saved!")
    }
}
```

## TIER 5 — GuiForm Mathematical Positioning
> METHODS COVERED: GuiForm() · gui.Add() · Format() · gui.Show() · gui.Submit() · gui.OnEvent() · gui.SetFont() · gui.Hide()

Introduces the `GuiForm()` helper function and the `pad`/`currentY` coordinate system for explicit, mathematically consistent GUI layouts. Every control position is derived from a single `pad` variable and a sequential `currentY` cursor — no hard-coded coordinates, no implicit AHK position accumulation. Use this tier when pixel-perfect alignment, easily adjustable spacing, or a visually professional result is required without the full overhead of `LayoutAwareGui`.

```ahk
; GuiForm helper — define once at the top of every TIER 5+ GUI function
GuiForm(x, y, w, h, extraParams := "") {
    params := Format("x{} y{} w{} h{}", x, y, w, h)
    return extraParams ? params " " extraParams : params
}

; ✓ TIER 5 layout foundation — single pad, currentY tracking
CreateSettingsGui() {
    gui := Gui("+Resize", "Settings")
    gui.OnEvent("Close",  (*) => gui.Hide())
    gui.OnEvent("Escape", (*) => gui.Hide())

    pad          := 10          ; ✓ one variable controls ALL spacing
    currentY     := pad         ; ✓ Y cursor starts at top margin
    windowWidth  := 500
    contentWidth := windowWidth - (pad * 2)

    gui.SetFont("s12 bold")
    gui.Add("Text", GuiForm(pad, currentY, contentWidth, 25, "+Center"), "Application Settings")
    currentY += 25 + pad * 2    ; ✓ major section break uses pad*2
    gui.SetFont("s9 norm")

    ; --- Inline label + edit row ---
    labelWidth := 80
    gui.Add("Text", GuiForm(pad, currentY, labelWidth, 23), "Name:")
    gui.Add("Edit", GuiForm(pad + labelWidth + pad, currentY,
            contentWidth - labelWidth - pad, 23, "vName"), "")
    currentY += 23 + pad

    gui.Add("Text", GuiForm(pad, currentY, labelWidth, 23), "Email:")
    gui.Add("Edit", GuiForm(pad + labelWidth + pad, currentY,
            contentWidth - labelWidth - pad, 23, "vEmail"), "")
    currentY += 23 + pad

    ; --- CheckBox group ---
    gui.Add("CheckBox", GuiForm(pad, currentY, contentWidth, 23, "vAutoSave"),
            "Auto-save enabled")
    currentY += 23 + pad

    gui.Add("CheckBox", GuiForm(pad, currentY, contentWidth, 23, "vDarkMode"),
            "Dark mode")
    currentY += 23 + pad * 2    ; ✓ section break before buttons

    ; --- OK button ---
    btnW := 100
    btnH := 28
    okBtn := gui.Add("Button", GuiForm(pad, currentY, btnW, btnH, "Default"), "OK")
    okBtn.OnEvent("Click", (*) => gui.Submit())
    currentY += btnH + pad

    ; ✓ window height = currentY + pad (bottom margin)
    gui.Show(Format("w{} h{}", windowWidth, currentY + pad))
    return gui
}

CreateSettingsGui()

; ✗ Hard-coded Y values break when spacing changes
; gui.Add("Text", "x10 y47 w100 h23", "Name:")     ; → brittle, unmaintainable
; gui.Add("Edit", "x10 y74 w380 h23")               ; → updating requires recalculating all
```

## TIER 6 — Advanced Layout Compositions
> METHODS COVERED: GuiForm() · AddListView() · AddGroupBox() · AddCheckBox() · AddButton() · gui.Add() · Format() · gui.Show() · gui.Hide() · gui.Submit() · gui.OnEvent()

Applies the GuiForm system to production-grade layout patterns: multi-column grids with mathematically distributed widths, button rows aligned right/center/distributed, GroupBox controls with consistent inner padding, and a complete validated settings GUI. Use this tier when delivering a finished, professionally spaced GUI that must remain easily maintainable by adjusting a single `pad` value.

```ahk
GuiForm(x, y, w, h, extraParams := "") {
    params := Format("x{} y{} w{} h{}", x, y, w, h)
    return extraParams ? params " " extraParams : params
}

; --- PATTERN 1: Two-column side-by-side controls ---
TwoColumnDemo() {
    gui := Gui(, "Two Columns")
    pad          := 10
    currentY     := pad
    windowWidth  := 650
    contentWidth := windowWidth - (pad * 2)

    leftWidth := (contentWidth - pad) / 2    ; ✓ gap between columns = pad
    rightX    := pad + leftWidth + pad

    gui.Add("ListView", GuiForm(pad,    currentY, leftWidth, 200), ["Source"])
    gui.Add("ListView", GuiForm(rightX, currentY, leftWidth, 200), ["Destination"])
    currentY += 200 + pad

    gui.Show(Format("w{} h{}", windowWidth, currentY + pad))
    return gui
}

; --- PATTERN 2: N equal columns ---
ThreeColumnDemo() {
    gui := Gui(, "Three Columns")
    pad          := 10
    currentY     := pad
    windowWidth  := 650
    contentWidth := windowWidth - (pad * 2)

    numCols   := 3
    totalGaps := pad * (numCols - 1)
    colWidth  := (contentWidth - totalGaps) / numCols

    Loop numCols {
        x := pad + (A_Index - 1) * (colWidth + pad)
        gui.Add("ListView", GuiForm(x, currentY, colWidth, 150), ["Column " A_Index])
    }
    currentY += 150 + pad

    gui.Show(Format("w{} h{}", windowWidth, currentY + pad))
    return gui
}

; --- PATTERN 3: Right-aligned button row ---
RightAlignedButtons(gui, currentY, windowWidth, pad) {
    labels := ["OK", "Cancel", "Apply"]
    btnW   := 100
    btnH   := 30
    totalW := labels.Length * btnW + (labels.Length - 1) * pad
    startX := windowWidth - pad - totalW

    for idx, label in labels {
        x := startX + (idx - 1) * (btnW + pad)
        gui.Add("Button", GuiForm(x, currentY, btnW, btnH), label)
    }
    return currentY + btnH + pad
}

; --- PATTERN 4: Centered button row ---
CenteredButtons(gui, currentY, windowWidth, pad) {
    labels := ["Save", "Cancel"]
    btnW   := 110
    btnH   := 30
    totalW := labels.Length * btnW + (labels.Length - 1) * pad
    startX := (windowWidth - totalW) / 2

    for idx, label in labels {
        x := startX + (idx - 1) * (btnW + pad)
        gui.Add("Button", GuiForm(x, currentY, btnW, btnH), label)
    }
    return currentY + btnH + pad
}

; --- PATTERN 5: Distributed button row (equal widths, full content width) ---
DistributedButtons(gui, currentY, windowWidth, pad) {
    labels       := ["Back", "Next", "Finish"]
    btnH         := 30
    contentWidth := windowWidth - (pad * 2)
    btnW         := (contentWidth - pad * (labels.Length - 1)) / labels.Length

    for idx, label in labels {
        x := pad + (idx - 1) * (btnW + pad)
        gui.Add("Button", GuiForm(x, currentY, btnW, btnH), label)
    }
    return currentY + btnH + pad
}

; --- PATTERN 6: GroupBox with nested inner padding ---
GroupBoxDemo() {
    gui := Gui(, "Group Demo")
    pad          := 10
    currentY     := pad
    windowWidth  := 500
    contentWidth := windowWidth - (pad * 2)

    groupH := 110
    gui.Add("GroupBox", GuiForm(pad, currentY, contentWidth, groupH), "Options")

    innerY     := currentY + 20           ; allow for GroupBox label height
    innerX     := pad * 2
    innerWidth := contentWidth - (pad * 2)

    gui.Add("CheckBox", GuiForm(innerX, innerY, innerWidth, 23, "vAutoSave"),
            "Enable auto-save")
    innerY += 23 + pad

    gui.Add("CheckBox", GuiForm(innerX, innerY, innerWidth, 23, "vNotifications"),
            "Enable notifications")
    innerY += 23 + pad

    gui.Add("CheckBox", GuiForm(innerX, innerY, innerWidth, 23, "vDarkMode"),
            "Dark mode")

    currentY += groupH + pad

    gui.Show(Format("w{} h{}", windowWidth, currentY + pad))
    return gui
}

; --- COMPLETE PRODUCTION GUI: all patterns combined ---
CreateProductionGui() {
    gui := Gui("+Resize", "Production Settings")
    gui.OnEvent("Close",  (*) => gui.Hide())
    gui.OnEvent("Escape", (*) => gui.Hide())

    pad          := 10
    currentY     := pad
    windowWidth  := 500
    contentWidth := windowWidth - (pad * 2)

    ; Title
    gui.SetFont("s12 bold")
    gui.Add("Text", GuiForm(pad, currentY, contentWidth, 25, "+Center"),
            "Application Settings")
    currentY += 25 + pad * 2
    gui.SetFont("s9 norm")

    ; Label + edit pairs
    labelWidth := 80
    for idx, fieldDef in [Map("label", "Name:",  "vname", "vName"),
                           Map("label", "Email:", "vname", "vEmail")] {
        gui.Add("Text", GuiForm(pad, currentY, labelWidth, 23), fieldDef["label"])
        gui.Add("Edit", GuiForm(pad + labelWidth + pad, currentY,
                contentWidth - labelWidth - pad, 23, fieldDef["vname"]), "")
        currentY += 23 + pad
    }

    currentY += pad    ; extra section gap

    ; GroupBox with checkboxes
    groupH := 100
    gui.Add("GroupBox", GuiForm(pad, currentY, contentWidth, groupH), "Preferences")
    innerY     := currentY + 20
    innerX     := pad * 2
    innerWidth := contentWidth - (pad * 2)

    gui.Add("CheckBox", GuiForm(innerX, innerY, innerWidth, 23, "vAutoSave"),
            "Auto-save")
    innerY += 23 + pad
    gui.Add("CheckBox", GuiForm(innerX, innerY, innerWidth, 23, "vDarkMode"),
            "Dark mode")
    currentY += groupH + pad * 2

    ; Right-aligned button row
    labels := ["OK", "Cancel", "Apply"]
    btnW   := 80
    btnH   := 28
    totalW := labels.Length * btnW + (labels.Length - 1) * pad
    startX := windowWidth - pad - totalW

    for idx, label in labels {
        x   := startX + (idx - 1) * (btnW + pad)
        btn := gui.Add("Button", GuiForm(x, currentY, btnW, btnH), label)
        if (label = "OK")
            btn.OnEvent("Click", (*) => gui.Submit())
        else if (label = "Cancel")
            btn.OnEvent("Click", (*) => gui.Hide())     ; ✓ gui.Hide(), not gui.Close()
    }
    currentY += btnH + pad

    ; ✓ window height exactly matches content
    gui.Show(Format("w{} h{}", windowWidth, currentY + pad))
    return gui
}

CreateProductionGui()

; --- LAYOUT VALIDATION AUDIT (run mentally before finalizing any TIER 6 GUI) ---
; Check: single `pad` variable defined — no mixed spacing values
; Check: all control positions derived from pad and currentY
; Check: window height = final currentY + pad
; Check: GuiForm() used for every gui.Add() call
; Check: no gui.Close() calls — use gui.Hide() or gui.Destroy()
; Check: for-loop button rows use (idx - 1) * (btnW + pad) formula
```

### Performance Notes

GUI performance in AHK v2 is most affected by control creation overhead, frequent redraws during Show/Hide cycles, and unnecessary `CalculateLayout()` or `GuiForm` recalculations on every `Show()`. Pre-compute layouts once during `__New()` or the creation function rather than recalculating dynamically. For GUIs that change element lists at runtime, invalidate the layout cache explicitly with a dirty-flag. Prefer `Map()` over repeated `HasProp()` lookups for O(1) control access by key name. For `GuiForm` layouts, pre-compute all derived values (`contentWidth`, column widths, button `startX`) once before the control-creation loop — never recompute per-control.

```ahk
class PerformantGui {
    __New() {
        this.gui          := Gui("+Resize", "Performant GUI")
        this.controls     := Map()    ; ✓ O(1) access by key, never linear search
        this._layoutDirty := true
        this.gui.SetFont("s10")
        this.gui.OnEvent("Close",  (*) => this.gui.Hide())
        this.gui.OnEvent("Escape", (*) => this.gui.Hide())
        this.BuildControls()          ; ✓ build all controls once in __New()
    }

    BuildControls() {
        ; ✓ Create all controls once; never rebuild on Show()
        this.controls["status"] := this.gui.AddText("xm w300 h20",      "Ready")
        this.controls["log"]    := this.gui.AddEdit("xm w300 h100 ReadOnly -Wrap")
        this.controls["clear"]  := this.gui.AddButton("xm w140",         "Clear Log")
        this.controls["close"]  := this.gui.AddButton("x+20 w140",       "Close")
        this.controls["clear"].OnEvent("Click", this.ClearLog.Bind(this))
        this.controls["close"].OnEvent("Click", (*) => this.gui.Hide())
    }

    ; ✓ Update control value directly — no full control rebuild
    UpdateStatus(msg) {
        this.controls["status"].Value := msg
    }

    ; ✓ Append to Edit value without recreating the control
    AppendLog(line) {
        this.controls["log"].Value .= line "`n"
    }

    ClearLog(*) {
        this.controls["log"].Value := ""
    }

    Show(*) => this.gui.Show()    ; ✓ single-expression arrow for simple delegation
}

; ✓ GuiForm performance — pre-compute derived layout values once
EfficientGuiFormLayout() {
    gui := Gui(, "Efficient")
    pad          := 10
    currentY     := pad
    windowWidth  := 500
    contentWidth := windowWidth - (pad * 2)    ; ✓ computed once, reused everywhere

    ; ✓ column widths computed once before the loop
    numCols  := 3
    colWidth := (contentWidth - pad * (numCols - 1)) / numCols

    Loop numCols {
        x := pad + (A_Index - 1) * (colWidth + pad)    ; ✓ formula, not literals
        gui.Add("Edit", GuiForm(x, currentY, colWidth, 23), "Col " A_Index)
    }
    currentY += 23 + pad

    gui.Show(Format("w{} h{}", windowWidth, currentY + pad))
    return gui
}
```

**O(1) vs O(n) summary:**
- `Map()` key lookup → O(1) regardless of control count; `HasProp()` on plain objects → O(n) in worst case
- Pre-computing `contentWidth`, `colWidth`, `startX` once before a loop → O(1) setup; computing inside the loop → O(n) redundant arithmetic
- `CalculateLayout()` in `Show()` on every call → O(n) per display; guard with `_built` flag → O(1) after first build
- `ctrl.Value := newText` to update text → O(1) property write; rebuilding the control entirely → O(n) destroy + re-add overhead

## DROP-IN RECIPES

```ahk
; ModalDialog — complete reusable modal owned dialog with correct owner disable/enable sequence
; ✓ Returns "OK" when confirmed, "" when cancelled — never throws on dismiss
ModalDialog(ownerGui, title, message) {
    if !(ownerGui is Gui)
        throw TypeError("ModalDialog: ownerGui must be a Gui object", -1)
    if (Type(title) != "String") || (Type(message) != "String")
        throw TypeError("ModalDialog: title and message must be strings", -1)

    result := ""
    dlg    := Gui(, title)
    dlg.Opt("+Owner" . ownerGui.Hwnd)
    dlg.SetFont("s10")

    ; ✓ Disable owner BEFORE showing the owned window
    ownerGui.Opt("+Disabled")

    done    := false
    cleanup := () => (done := true, ownerGui.Opt("-Disabled"), dlg.Destroy())

    dlg.AddText("xm w320", message)
    dlg.AddButton("xm w148 h28 Default", "OK").OnEvent("Click", (*) => (
        result := "OK",
        cleanup.Call()
    ))
    dlg.AddButton("x+8 w148 h28", "Cancel").OnEvent("Click", (*) => cleanup.Call())
    dlg.OnEvent("Close",  (*) => cleanup.Call())
    dlg.OnEvent("Escape", (*) => cleanup.Call())
    dlg.Show("w340 AutoSize")

    ; ✓ Spin until dismissed — pumps the message loop without blocking AHK threads
    while !done
        Sleep(50)
    return result
}
; Call site: if ModalDialog(myGui, "Confirm", "Delete this item?") = "OK"  { ... }

; SafeGuiControlGet — safely retrieve a named control's current value from a Map() store
; ✓ Returns the control's value or a caller-specified default — never throws on missing key
SafeGuiControlGet(controls, key, default := "") {
    if !(controls is Map)
        throw TypeError("SafeGuiControlGet: controls must be a Map", -1)
    if (Type(key) != "String") || key = ""
        throw TypeError("SafeGuiControlGet: key must be a non-empty string", -1)
    if !controls.Has(key)
        return default
    try
        return controls[key].Value
    catch as e
        throw Error("SafeGuiControlGet: failed reading '" key "' — " e.Message, -1)
}
; Call site: name := SafeGuiControlGet(this.controls, "nameEdit", "")

; BuildStatusBar — create a multi-part status bar with initial text labels
; ✓ Returns the StatusBar control object for later .SetText() calls
BuildStatusBar(gui, parts*) {
    if !(gui is Gui)
        throw TypeError("BuildStatusBar: gui must be a Gui object", -1)
    sb := gui.AddStatusBar()
    if parts.Length > 0 {
        widths := []
        Loop parts.Length - 1
            widths.Push(150)           ; ✓ all parts except last get fixed width
        sb.SetParts(widths*)           ; last part fills remaining space automatically
        for idx, text in parts
            sb.SetText(text, idx)
    }
    return sb
}
; Call site: sb := BuildStatusBar(myGui, "Ready", "0 items", "v1.0")
```

## ANTI-PATTERNS

| Pattern | Wrong | Correct | LLM Common Cause |
|---------|-------|---------|------------------|
| v1 command syntax | `Gui, Add, Text,, Hello` | `gui.AddText(, "Hello")` | AHK v1 training data dominates; v2 method syntax is under-represented in most training corpora |
| No class encapsulation | `g := Gui(...)` at top level with no class | All GUI code in a class `__New()` method | v1 scripts were procedural; LLM generates bare Gui() calls by default |
| Object literal for control storage | `this.ctrls := {btn: ctrl, edit: edit}` | `this.controls := Map(); this.controls["btn"] := ctrl` | AHK v1 used `{}` as dictionaries; LLM carries this v1 habit into v2 |
| `new` keyword for instantiation | `new MyGui()` | `MyGui()` | All mainstream OOP languages (Java, C#, Python) use `new`; LLM defaults to it |
| `gui.Close()` to dismiss window | `gui.Close()` | `gui.Hide()` or `gui.Destroy()` | JavaScript `window.close()` / HTML dialog `.close()` habit transfers to AHK |
| Multi-line arrow handler | `(*) => { saved := ...; MsgBox(...) }` | Separate named method + `.Bind(this)` | JavaScript/C# lambdas allow multi-statement bodies; LLM assumes same in AHK |
| Hard-coded Y coordinates | `gui.Add("Edit", "x10 y74 w380 h23")` | `currentY` tracking + `GuiForm()` | LLM doesn't know the GuiForm pattern; emulates raw pixel positioning from other toolkits |
| Zero-based column index in ListView | `lv.GetText(1, 0)` | `lv.GetText(1, 1)` (col is 1-based; row 0 retrieves column headers, not a data row) | Dominant 0-based indexing habit from most language training data |
| Width on Section option | `"Section w300"` | `"Section"` or `"xm Section"` | LLM confuses Section with a positioned control that accepts a width dimension |
| Re-enable owner after Destroy | `ownedGui.Destroy(); ownerGui.Opt("-Disabled")` | `ownerGui.Opt("-Disabled")` then `ownedGui.Destroy()` | Linear "close then cleanup" ordering from other modal dialog APIs (WinForms, Qt) |
| Forward-loop ListView delete | `Loop lv.GetCount() { lv.Delete(A_Index) }` | `while (rowNum := lv.GetNext(rowNum, "Selected")) { lv.Delete(rowNum); rowNum := 0 }` | LLM doesn't model row-number shifting after each delete in indexed collections |
| Rebuilding controls on every Show() | `CreateControls()` called in `Show()` each time | Guard with `if !this._built` flag; build once in `__New()` | LLM follows a "setup → display" pattern without considering re-entrant calls |

## SEE ALSO

> This module does NOT cover: PixelSearch, ImageSearch, GDI+ drawing, screen overlays, and graphical screen operations — see Module_GraphicsAndScreen.md
> This module does NOT cover: Hotkey() and HotIf() rules outside the GUI context (global hotkeys, context-sensitivity beyond HotIfWinExist) — see Module_InputAndHotkeys.md
> This module does NOT cover: Class inheritance patterns, meta-functions (__Get/__Set/__Call), and OOP design for GUI base classes beyond LayoutAwareGui — see Module_Classes.md
> This module does NOT cover: try/catch wrapping for GUI creation failures and control-access errors — see Module_Errors.md

- `Module_GraphicsAndScreen.md` — PixelSearch, ImageSearch, GDI+, screen coordinate systems, and overlay window techniques.
- `Module_InputAndHotkeys.md` — Hotkey(), HotIf(), HotIfWinActive/Exist(), Send(), and global keyboard/mouse input handling outside the GUI event system.
- `Module_Classes.md` — Full OOP patterns for extending LayoutAwareGui, meta-function design (__Get/__Set/__Call), and class property declarations.
- `Module_Errors.md` — try/catch patterns for Gui creation failure, catching MethodError from invalid control access, and structured error recovery.
