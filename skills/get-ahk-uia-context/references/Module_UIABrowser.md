# Module_UIABrowser.md
<!-- DOMAIN: Browser Automation via UI Automation (UIA_Browser wrapper) -->
<!-- SCOPE: General UIA element discovery, tree traversal, and control pattern interaction within page documents are not covered here — see Module_UIAElements.md and Module_UIAPatterns.md -->
<!-- TRIGGERS: UIA_Browser, UIA_Chrome, UIA_Edge, UIA_Mozilla, Navigate, GetCurrentDocumentElement, WaitPageLoad, GetTab, SelectTab, JSExecute, JSReturnThroughClipboard, TabExist, CloseTab, "browser automation", "chrome automation", "edge automation", "navigate URL", "get page content", "click link in browser", "web scraping AHK" -->
<!-- CONSTRAINTS: GetCurrentDocumentElement() returns 0 (not element) on 3-second timeout — always check IsObject() before use; it does NOT throw. DocumentElement becomes stale after every page navigation — re-call GetCurrentDocumentElement() after each Navigate/WaitPageLoad. GetTab() throws TargetError on not-found — use TabExist() for safe checks. -->
<!-- CROSS-REF: Module_UIAElements.md, Module_UIAPatterns.md, Module_Errors.md -->
<!-- VERSION: AHK v2.0+ with Lib/UIA.ahk v1.1.3 + Lib/UIA_Browser.ahk (Descolada) -->

## V1 → V2 BREAKING CHANGES

| v1 pattern (LLM commonly writes) | v2 correct form | Consequence |
|----------------------------------|-----------------|-------------|
| `new UIA_Browser("ahk_exe chrome.exe")` | `UIA_Browser("ahk_exe chrome.exe")` — no `new` | AHK v2 has no `new` keyword; `new` is treated as an undefined identifier and throws `NameError` at runtime |
| `cUIA.DocumentElement.FindFirst(cond)` reused after navigation | Re-call `cUIA.GetCurrentDocumentElement()` after every Navigate/WaitPageLoad | DocumentElement COM reference becomes stale on page change; any property access causes COM error |
| `if !cUIA.GetCurrentDocumentElement()` to check failure | `local doc := cUIA.GetCurrentDocumentElement()` then `if !IsObject(doc)` | GetCurrentDocumentElement() returns integer 0 on 3s timeout; the call itself never throws, but `!call()` evaluates the method call twice |
| `cUIA.GetTab(name)` without try/catch | `cUIA.TabExist(name)` for safe check, or `try cUIA.GetTab(name) catch TargetError` | GetTab() throws TargetError when no matching tab is found |
| `cUIA.GetCurrentURL()` while browser is minimized or behind other windows | Ensure `cUIA.IsBrowserVisible()` first, or use `fromAddressBar:=True` | GetCurrentURL() reads the document element Value which requires the browser to be visible; auto-activates (may steal focus) for non-Mozilla browsers |
| Calling `JSExecute(js)` during an ongoing page load | Wait for `WaitPageLoad()` to complete before any JS call | JSExecute navigates the address bar to `javascript:...`; mid-load navigation cancels the current page load |
| `catch err` without `as` | `catch as err` | AHK v2 syntax error; `as` is mandatory to bind the error object |
| Storing `UIA_Browser` instance and reusing across browser restarts | Re-create the instance after detecting window closure | BrowserId (HWND) and all internal element references become invalid after the browser closes and restarts |

## API QUICK-REFERENCE

> Method count exceeds 20 — structured bulleted list format used.

### Instance Properties (set during init and by GetCurrentMainPaneElement)

- **instance.BrowserId** — Type: Integer HWND | Notes: Window handle from `WinExist(wTitle)`; use with `WinGetTitle("ahk_id " BrowserId)` to check if still alive
- **instance.BrowserType** — Type: String | Notes: Auto-detected: `"Chrome"`, `"Edge"`, `"Mozilla"`, `"Vivaldi"`, `"Brave"`, `"Unknown"`; drives per-browser method dispatch
- **instance.BrowserElement** — Type: UIA element | Notes: Root UIA element of the browser window; acts as proxy target for all element methods called on the instance directly
- **instance.MainPaneElement** — Type: UIA element | Notes: Element containing URL bar, tabs, nav buttons — excludes page content; refreshed by `GetCurrentMainPaneElement()`
- **instance.NavigationBarElement** — Type: UIA element | Notes: Smallest toolbar element containing URL bar and navigation buttons
- **instance.TabBarElement** — Type: UIA element | Notes: Element containing only tab items; used by all GetTab/SelectTab methods
- **instance.URLEditElement** — Type: UIA element | Notes: The address bar Edit element; used by `SetURL()` and `GetCurrentURL(fromAddressBar:=True)`
- **instance.DocumentElement** — Type: UIA element or 0 | Notes: Set by `GetCurrentDocumentElement()`; alias `CurrentDocumentElement`; STALE after any navigation — always re-fetch
- **instance.BackButton** — Type: UIA element | Notes: Cached back navigation button; refreshed by `GetCurrentBackButton()`
- **instance.ReloadButton** — Type: UIA element | Notes: Cached reload button; WaitPageLoad polls its Name/Description against cached init values

### Constructors

- **UIA_Browser()** — Signature: `UIA_Browser(wTitle:="")` | Returns: browser instance (auto-typed) | Throws: `TargetError` if window not found | Notes: Auto-detects Chrome/Edge/Mozilla/Vivaldi/Brave from exe name and window class; rebases instance on the correct subclass prototype; call `#include UIA_Browser.ahk` before use
- **UIA_Chrome() / UIA_Edge() / UIA_Mozilla() / UIA_Vivaldi() / UIA_Brave()** — Signature: same as UIA_Browser | Returns: typed browser instance | Throws: `TargetError` | Notes: Use when auto-detection is unreliable or when a specific subclass behaviour is required

### Document & Navigation

- **instance.GetCurrentDocumentElement()** — Signature: `GetCurrentDocumentElement()` | Returns: UIA element on success, **`0` on 3-second timeout** | Throws: — | Notes: Uses WaitElement(DocumentCondition, 3000) internally; ALWAYS check `IsObject(result)` before use; re-call after every navigation
- **instance.GetCurrentMainPaneElement()** — Signature: `GetCurrentMainPaneElement()` | Returns: MainPaneElement | Throws: `TargetError` if DocumentElement unavailable | Notes: Refreshes MainPaneElement, NavigationBarElement, TabBarElement, URLEditElement, BackButton, ReloadButton; called automatically in constructor
- **instance.Navigate()** — Signature: `Navigate(url, targetTitle:="", waitLoadTimeOut:=-1, sleepAfter:=500)` | Returns: — | Throws: — | Notes: Calls SetURL(url, True) then WaitPageLoad; always follow with `GetCurrentDocumentElement()`
- **instance.WaitPageLoad()** — Signature: `WaitPageLoad(targetTitle:="", timeOut:=-1, sleepAfter:=500, titleMatchMode:="", titleCaseSensitive:=False)` | Returns: title string on success, `False` on timeout | Throws: — | Notes: Polls Reload button Name/Description against values cached at init; sleepAfter adds DOM-settlement delay after button stabilises
- **instance.WaitTitleChange()** — Signature: `WaitTitleChange(targetTitle:="", timeOut:=-1)` | Returns: new title string, or `False` on timeout | Throws: — | Notes: Empty targetTitle waits for any title change; useful for SPA route changes where Reload button state does not change
- **instance.GetCurrentURL()** — Signature: `GetCurrentURL(fromAddressBar:=False)` | Returns: String URL | Throws: — | Notes: fromAddressBar=False reads document element Value (requires visible window; auto-activates non-Mozilla); fromAddressBar=True reads URLEditElement.Value (may lack `https://` prefix; user may have typed partial text)
- **instance.SetURL()** — Signature: `SetURL(newUrl, navigateToNewUrl:=False)` | Returns: — | Throws: — | Notes: Sets ValuePattern then LegacyIAccessible fallback; navigateToNewUrl=True sends Ctrl+Enter; does NOT call WaitPageLoad
- **instance.Back() / Forward() / Reload() / Home()** — Signature: `Back()` etc. | Returns: — | Throws: — | Notes: Clicks the corresponding navigation button via UIA InvokePattern; Home() silently returns if no Home button exists

### Tab Management

- **instance.GetTab()** — Signature: `GetTab(searchPhrase:="", matchMode:=3, caseSense:=True)` | Returns: UIA tab element | Throws: **`TargetError`** when no tab matches | Notes: Empty searchPhrase returns currently selected tab (matched by `SelectionItemIsSelected:1`); Integer searchPhrase returns nth tab by index; ALWAYS use TabExist() for safe checks
- **instance.TabExist()** — Signature: `TabExist(searchPhrase:="", matchMode:=3, caseSense:=True)` | Returns: element on found, `0` on not-found | Throws: — | Notes: Safe wrapper around GetTab() that catches TargetError; preferred over GetTab() in conditional logic
- **instance.GetTabs()** — Signature: `GetTabs(searchPhrase:="", matchMode:=3, caseSense:=True)` | Returns: `Array` of tab elements (empty if none match) | Throws: — | Notes: Empty searchPhrase returns all tabs; uses FindAll/FindElements on TabBarElement
- **instance.GetAllTabNames()** — Signature: `GetAllTabNames()` | Returns: `Array` of Strings | Throws: — | Notes: Maps GetTabs() to `.Name` for each element
- **instance.SelectTab()** — Signature: `SelectTab(tabName, matchMode:=3, caseSense:=True)` | Returns: selected tab element | Throws: **`TargetError`** when tab not found | Notes: Accepts tab element object or name string; Vivaldi uses ControlClick; all others use Click
- **instance.NewTab()** — Signature: `NewTab()` | Returns: — | Throws: — | Notes: Sends Ctrl+T via ControlSend; browser must be focused or have ControlSend target set
- **instance.CloseTab()** — Signature: `CloseTab(tabElementOrName:="", matchMode:=3, caseSense:=True)` | Returns: — | Throws: — | Notes: Empty arg closes current tab by clicking the last child button of the active tab element (the ✕ button)

### Content Extraction

- **instance.GetAllText()** — Signature: `GetAllText()` | Returns: String (newline-joined) | Throws: — | Notes: FindAll on all Text-type elements in BrowserElement; auto-activates if not visible; use for quick page text dump, not for structured data
- **instance.GetAllLinks()** — Signature: `GetAllLinks()` | Returns: `Array` of Hyperlink elements | Throws: — | Notes: FindAll on Hyperlink-type elements in BrowserElement; auto-activates if not visible; read `.Name` for link text, `.Value` may give href

### JavaScript Interop

- **instance.JSExecute()** — Signature: `JSExecute(js)` | Returns: — | Throws: — | Notes: Navigates address bar to `javascript:...` via SetURL; for Mozilla, defaults to Console method (slower) — set `JavascriptExecutionMethod := "Bookmark"` for better performance; fire-and-forget, no return value
- **instance.JSReturnThroughClipboard()** — Signature: `JSReturnThroughClipboard(js)` | Returns: String (JS return value) | Throws: — | Notes: Injects clipboard-copy code alongside the JS; saves and restores clipboard; preferred return-value method; NOT thread-safe
- **instance.JSReturnThroughTitle()** — Signature: `JSReturnThroughTitle(js, timeOut:=500)` | Returns: String or `""` on timeout | Throws: — | Notes: Sets document.title to the return value temporarily; unreliable — use JSReturnThroughClipboard instead
- **instance.JSSetTitle()** — Signature: `JSSetTitle(newTitle)` | Returns: — | Throws: — | Notes: Convenience wrapper — `document.title = newTitle` via JSExecute; useful for WaitTitleChange signalling
- **instance.JSGetElementPos()** — Signature: `JSGetElementPos(selector, useRenderWidgetPos:=False)` | Returns: `{x, y, w, h}` in screen coordinates | Throws: — | Notes: Uses querySelector + getBoundingClientRect + devicePixelRatio; useRenderWidgetPos=True uses Chrome_RenderWidgetHostHWND1 control position (more reliable for Chromium DPI scaling)
- **instance.JSClickElement()** — Signature: `JSClickElement(selector)` | Returns: — | Throws: — | Notes: querySelector + `.click()` — faster than ClickJSElement but unreliable for elements that don't support JS click events
- **instance.ClickJSElement()** — Signature: `ClickJSElement(selector, WhichButton:="", ClickCount:=1, DownOrUp:="", Relative:="", useRenderWidgetPos:=False)` | Returns: — | Throws: — | Notes: JSGetElementPos + AHK Click at element center; reliable for all clickable elements
- **instance.ControlClickJSElement()** — Signature: `ControlClickJSElement(selector, WhichButton?, ClickCount?, Options?, useRenderWidgetPos:=False)` | Returns: — | Throws: — | Notes: JSGetElementPos + ControlClick; use when browser window need not be foreground

### Alerts & Dialogs

- **instance.GetAlertText()** — Signature: `GetAlertText(closeAlert:=True, timeOut:=3000)` | Returns: String text | Throws: — | Notes: Waits up to timeOut ms for JS alert dialog; closes it by default; Edge uses FindFirstWithOptions variant
- **instance.CloseAlert()** — Signature: `CloseAlert()` | Returns: — | Throws: — | Notes: Clicks the OK button on the current JS alert; silent if no alert present

### Visibility & Input

- **instance.IsBrowserVisible()** — Signature: `IsBrowserVisible()` | Returns: True/False | Throws: — | Notes: Tests whether any of the 4 window corners is hit-tested to the browser; use before GetCurrentURL()/GetAllText() to avoid unwanted window activation
- **instance.Send()** — Signature: `Send(text)` | Returns: — | Throws: — | Notes: Focuses browser via WM_SETFOCUS + ControlSend; does not release modifier keys
- **instance.ControlSend()** — Signature: `ControlSend(text, releaseModifiers:=True)` | Returns: — | Throws: — | Notes: Releases active modifier keys (Ctrl/Alt/Shift) before sending, then re-presses; use for reliable hotkey sequences

### Proxy Behaviour (via __Get / __Call)

UIA_Browser delegates unknown property/method calls first to `BrowserElement`, then to `UIA`. This means all UIA element methods are available directly on the instance:

- `browser.FindFirst(cond)` = `browser.BrowserElement.FindFirst(cond)`
- `browser.ElementFromHandle(hwnd)` = `UIA.ElementFromHandle(hwnd)`

## AHK V2 CONSTRAINTS

- `GetCurrentDocumentElement()` returns integer `0` on a 3-second internal timeout — it does NOT throw. Always store the result and check `IsObject(result)` before calling any method on it.
  - ✗ `local doc := cUIA.GetCurrentDocumentElement()` then `doc.FindFirst(cond)` without guard — MethodError if 0
  - ✓ `local doc := cUIA.GetCurrentDocumentElement()` then `if !IsObject(doc)` guard before any access

- `DocumentElement` and `CurrentDocumentElement` properties become stale COM references after any page navigation — re-call `GetCurrentDocumentElement()` after every `Navigate()`, `WaitPageLoad()`, `Back()`, `Forward()`, or user-initiated navigation.
  - ✗ `local doc := cUIA.GetCurrentDocumentElement()` stored before `Navigate(...)` — stale after navigation
  - ✓ `cUIA.Navigate(url)`, then `local doc := cUIA.GetCurrentDocumentElement()` — always fresh

- `GetTab()` throws `TargetError` when no matching tab is found — use `TabExist()` for conditional checks without exception handling.
  - ✗ `local tab := cUIA.GetTab("Settings")` then `if !tab` — TargetError already thrown, check never runs
  - ✓ `local tab := cUIA.TabExist("Settings")` then `if tab` — TabExist returns 0 on not-found

- `JSExecute(js)` navigates the address bar to `javascript:<js>` — do not call while `WaitPageLoad()` is still blocking or during an ongoing load; it will cancel the current navigation.
  - ✗ `cUIA.Navigate(url)` then immediately `cUIA.JSExecute(js)` — cancels the navigation
  - ✓ `cUIA.Navigate(url, targetTitle)` (waits internally) then `cUIA.JSExecute(js)` — safe sequence

- `JSReturnThroughClipboard()` saves and restores the clipboard but is NOT atomic — a concurrent clipboard write between save and restore corrupts the user's clipboard content.
  - ✓ Use only when no concurrent clipboard operations are possible; consider `JSReturnThroughTitle()` for low-contention cases despite its unreliability

- `GetCurrentURL(fromAddressBar:=False)` requires the browser to be visible; for non-Mozilla it auto-activates (steals focus). Check `IsBrowserVisible()` first if focus disruption is unacceptable.
  - ✗ `cUIA.GetCurrentURL()` from a background script — unexpectedly activates browser window
  - ✓ `if cUIA.IsBrowserVisible()` then `cUIA.GetCurrentURL()` else `cUIA.GetCurrentURL(True)`

- `WaitPageLoad()` polls the Reload button's cached Name/Description from init — it does NOT poll DOM readiness. After a SPA route change where the Reload button state doesn't change, use `WaitTitleChange()` instead.

- Condition objects passed to `FindFirst`/`FindAll` via the BrowserElement proxy use `{}` AHK object literals, NOT `Map()` — the UIA library's `__CreateRawCondition()` uses `OwnProps()` on them (same rule as Module_UIAElements.md).

Safe-access priority order for browser element operations:
  1. `TabExist(name)` — safe tab check; never throws; returns 0 on not-found
  2. `WaitPageLoad(title, timeOut)` + `GetCurrentDocumentElement()` — standard post-navigation sequence
  3. `try { GetTab(name) } catch TargetError` — when tab-not-found vs other errors need separate handling
  4. `try { ... } catch as err { ... }` — outer catch for COM errors from stale elements

Unset variable handling: always check `IsObject(doc)` after `GetCurrentDocumentElement()` — returning 0 means the page did not expose a Document element within 3 seconds.

Resource lifecycle: `UIA_Browser` instances hold the BrowserId HWND. After browser restart the HWND is invalid — always verify with `WinExist("ahk_id " cUIA.BrowserId)` before reuse in long-running scripts.

## AGENT QA CHECKLIST

- [ ] Did I call `GetCurrentDocumentElement()` and check `IsObject(doc)` after every `Navigate()`, `Back()`, `Forward()`, or user navigation before accessing any document element?
- [ ] Did I use `TabExist()` (not `GetTab()`) when checking whether a tab exists in conditional logic?
- [ ] Did I call `WaitPageLoad()` before any `JSExecute()` call to ensure the previous navigation is complete?
- [ ] Did I use `WaitTitleChange()` instead of `WaitPageLoad()` for SPA route changes where the Reload button state does not change?

## RUNTIME ERROR MAPPING

| Exception Class | Trigger Condition | Detection Code | Fix |
|----------------|-------------------|----------------|-----|
| `TargetError` | `UIA_Browser(wTitle)` when browser window does not exist, OR `GetCurrentMainPaneElement()` when document element unavailable after init | `e.Message` contains "failed to find the browser" or "unable to find the Document element" | Verify browser is running with `WinExist(wTitle)` before constructing; ensure browser is at least partially visible before `UIA_Browser()` call |
| `TargetError` | `GetTab(name)` / `SelectTab(name)` when no tab matches the search phrase | `e.Message` contains "Tab with name ... was not found" | Replace `GetTab()` with `TabExist()` for conditional checks; wrap `SelectTab()` in `try/catch TargetError` |
| `MethodError` | Calling `.FindFirst()`, `.Name`, or any method on `DocumentElement` that is integer `0` (3s timeout from `GetCurrentDocumentElement()`) | `e.Message` = "This value of type "Integer"..." | Add `if !IsObject(doc)` guard after every `GetCurrentDocumentElement()` call; consider increasing retry loop |

## TIER 1 — Constructor, Auto-Detection, and BrowserElement Proxy
> METHODS COVERED: UIA_Browser · UIA_Chrome · UIA_Edge · UIA_Mozilla · IsBrowserVisible · ControlSend

TIER 1 covers constructing a browser instance and understanding the proxy mechanism. `UIA_Browser()` auto-detects the browser type and rebases the instance onto the correct subclass prototype. The `__Get`/`__Call` metacall system means element methods can be called directly on the instance — `browser.FindFirst(cond)` is identical to `browser.BrowserElement.FindFirst(cond)`.

```ahk
; ✓ Auto-detection: UIA_Browser rebases on UIA_Chrome, UIA_Edge, UIA_Mozilla automatically
local browser
try
    browser := UIA_Browser("ahk_exe chrome.exe")
catch TargetError as err {
    MsgBox "Browser not found: " err.Message
    return
}

; ✓ BrowserType tells which subclass was selected
OutputDebug "BrowserType: " browser.BrowserType   ; "Chrome", "Edge", "Mozilla", etc.
OutputDebug "BrowserId: "   browser.BrowserId      ; Integer HWND

; ✓ Proxy: browser.FindFirst(cond) == browser.BrowserElement.FindFirst(cond)
local urlBar := browser.URLEditElement             ; set by GetCurrentMainPaneElement() at init
local curVal := urlBar.Value                        ; reads Value via ValuePattern

; ✓ Force a specific subclass when auto-detection is unreliable
local edgeBrowser := UIA_Edge("ahk_exe msedge.exe")

; ✓ IsBrowserVisible before operations that require the browser to be in the foreground
if !browser.IsBrowserVisible()
    WinActivate "ahk_id " browser.BrowserId

; ✓ ControlSend releases modifier keys — safer than Send for hotkey sequences
browser.ControlSend("{Ctrl down}r{Ctrl up}")   ; force reload

; ✗ WRONG — no new keyword in AHK v2
; local b := new UIA_Browser("ahk_exe chrome.exe")   ; → NameError at runtime
```

## TIER 2 — Document Element Lifecycle and Page Load Synchronisation
> METHODS COVERED: GetCurrentDocumentElement · Navigate · WaitPageLoad · WaitTitleChange · IsObject guard

TIER 2 covers the core document element lifecycle rule: every navigation invalidates the previous `DocumentElement`. `GetCurrentDocumentElement()` uses `WaitElement(DocumentCondition, 3000)` internally — it returns `0` on timeout, never throws. `WaitPageLoad()` polls the Reload button's state, so it works correctly for full page loads but NOT for SPA route changes.

```ahk
; ✓ Standard navigation sequence: Navigate → WaitPageLoad → re-fetch document
NavigateAndGetDoc(browser, url, expectedTitle := "") {
    ; ✓ Navigate calls SetURL + WaitPageLoad internally
    browser.Navigate(url, expectedTitle, 15000)

    ; ✓ Re-fetch document AFTER load completes — previous reference is now stale
    local doc := browser.GetCurrentDocumentElement()
    ; ✓ GetCurrentDocumentElement returns 0 on 3s timeout — check before use
    if !IsObject(doc)
        return Result.Fail("TIMEOUT", "Document element unavailable after navigation to " url)
    return Result.Ok(doc)
}

; ✓ SPA route change — Reload button state unchanged, use WaitTitleChange instead
NavigateSPA(browser, clickAction, expectedTitle, timeoutMs := 8000) {
    ; Capture current title before action
    local prevTitle := WinGetTitle("ahk_id " browser.BrowserId)
    clickAction.Call()   ; e.g. a lambda that clicks a SPA nav link

    ; ✓ WaitTitleChange detects SPA route changes even when Reload button stays idle
    if !browser.WaitTitleChange(expectedTitle, timeoutMs)
        return Result.Fail("TIMEOUT", "SPA route did not change to: " expectedTitle)

    ; ✓ Still re-fetch document — SPA may have replaced the Document element
    local doc := browser.GetCurrentDocumentElement()
    if !IsObject(doc)
        return Result.Fail("TIMEOUT", "Document element unavailable after SPA navigation")
    return Result.Ok(doc)
}

; ✗ WRONG — caching DocumentElement across navigations
GetSearchResults(browser, query) {
    ; local doc := browser.GetCurrentDocumentElement()   ; ← stale after next Navigate
    browser.Navigate("https://example.com/search?q=" query, "results")
    ; local box := doc.FindFirst({Name: "result"})        ; ← COM error; doc is now stale
}
; ✓ Correct version: call GetCurrentDocumentElement AFTER navigate
GetSearchResults(browser, query) {
    browser.Navigate("https://example.com/search?q=" query, "results")
    local doc := browser.GetCurrentDocumentElement()
    if !IsObject(doc)
        return []
    return doc.FindAll({Type: UIA.Type.ListItem}, UIA.TreeScope.Descendants)
}
```

## TIER 3 — Navigation Controls, URL Management, and GetCurrentURL
> METHODS COVERED: GetCurrentURL · SetURL · Back · Forward · Reload · IsBrowserVisible

TIER 3 covers URL reading/writing and navigation button control. `GetCurrentURL(fromAddressBar:=False)` reads the document element's Value property — the browser must be visible, and non-Mozilla browsers will auto-activate. Use `fromAddressBar:=True` to read the Edit element directly (no activation needed, but no `https://` prefix guarantee).

```ahk
; ✓ GetCurrentURL — check visibility first to avoid unexpected window activation
GetURL(browser) {
    if browser.IsBrowserVisible()
        return browser.GetCurrentURL()              ; reads document element Value
    else
        return browser.GetCurrentURL(True)          ; reads URLEditElement.Value; may lack https://
}

; ✓ SetURL + Navigate separately when you want to set URL without immediate load
browser.SetURL("https://example.com")               ; sets address bar text only
; ... do something else ...
browser.SetURL("https://example.com", True)         ; sets AND navigates (sends Ctrl+Enter)

; ✓ Back/Forward navigation — re-fetch document after each
browser.Back()
browser.WaitPageLoad("", 8000)
local doc := browser.GetCurrentDocumentElement()
if !IsObject(doc)
    return

; ✓ WaitTitleChange after Forward — useful when WaitPageLoad doesn't fire
browser.Forward()
browser.WaitTitleChange("", 5000)    ; waits for any title change
local doc2 := browser.GetCurrentDocumentElement()

; ✓ Reload and wait for page to settle
browser.Reload()
browser.WaitPageLoad("", 10000, 1000)   ; 1000ms sleepAfter for heavier pages
local doc3 := browser.GetCurrentDocumentElement()
if IsObject(doc3)
    OutputDebug "Reloaded URL: " browser.GetCurrentURL()

; ✗ WRONG — GetCurrentURL when browser is hidden and not Mozilla
; local url := browser.GetCurrentURL()   ; → auto-activates Chrome/Edge, steals focus
```

## TIER 4 — Tab Management
> METHODS COVERED: GetTab · TabExist · GetTabs · GetAllTabNames · SelectTab · NewTab · CloseTab

TIER 4 covers all tab operations. The critical asymmetry: `GetTab()` **throws** `TargetError`; `TabExist()` safely returns `0`. Always use `TabExist()` in conditional logic. After `SelectTab()` completes, call `WaitPageLoad()` before accessing the document element of the newly selected tab.

```ahk
; ✓ TabExist for conditional check — safe, never throws
SwitchToTab(browser, tabName) {
    ; ✓ TabExist returns element or 0 — if !tab is live code
    local tab := browser.TabExist(tabName, 2)   ; matchMode=2 = contains anywhere
    if !tab {
        browser.NewTab()
        browser.Navigate("https://example.com/" tabName)
        return
    }
    ; ✓ SelectTab may throw TargetError — wrap when multiple concurrent navigations possible
    try {
        browser.SelectTab(tab)               ; accepts element object directly
        browser.WaitPageLoad("", 5000)       ; wait for tab's page to settle
    } catch TargetError as err {
        return Result.Fail("TAB_ERROR", "SelectTab failed: " err.Message)
    }
}

; ✓ Enumerate all open tabs
ListAllTabs(browser) {
    local names := browser.GetAllTabNames()
    local result := ""
    for i, name in names
        result .= i ": " name "`n"
    return result
}

; ✓ GetTabs with matchMode=2 (substring) — returns array of matching tabs
CloseAllResultTabs(browser, partialName) {
    local tabs := browser.GetTabs(partialName, 2)
    for tab in tabs
        browser.CloseTab(tab)   ; pass element directly — no name lookup
}

; ✓ GetTab with integer index — returns nth tab (1-based)
SelectSecondTab(browser) {
    try {
        browser.SelectTab(browser.GetTab(2))
        browser.WaitPageLoad("", 5000)
    } catch TargetError
        return   ; fewer than 2 tabs open
}

; ✗ WRONG — GetTab throws; if !tab check is dead code
; local tab := browser.GetTab("Settings")
; if !tab                        ; → never runs; TargetError already thrown
;     return

; ✗ WRONG — accessing document without waiting after tab switch
; browser.SelectTab("My Tab")
; local doc := browser.GetCurrentDocumentElement()  ; → may be wrong tab's document
```

## TIER 5 — JavaScript Interop and querySelector Interaction
> METHODS COVERED: JSExecute · JSReturnThroughClipboard · JSReturnThroughTitle · JSGetElementPos · JSClickElement · ClickJSElement · ControlClickJSElement · GetAlertText · CloseAlert

TIER 5 covers JS execution via the address bar and querySelector-based element interaction. `JSExecute` is fire-and-forget; use `JSReturnThroughClipboard` when a return value is needed. Note that all JS methods navigate the address bar, so the page must be fully loaded before calling them.

```ahk
; ✓ Full JS interop sequence — wait for load before any JS call
AutomateWithJS(browser, targetUrl) {
    browser.Navigate(targetUrl, "My Page", 15000)

    ; ✓ Ensure document is ready before JS calls
    local doc := browser.GetCurrentDocumentElement()
    if !IsObject(doc)
        return

    ; ✓ JSExecute — fire-and-forget; no return value
    browser.JSExecute("document.getElementById('cookieBanner').style.display='none';void(0);")

    ; ✓ JSReturnThroughClipboard — preferred return-value method
    local pageTitle := browser.JSReturnThroughClipboard("document.title")
    OutputDebug "Page title via JS: " pageTitle

    ; ✓ ClickJSElement — querySelector + AHK Click; reliable for all elements
    browser.ClickJSElement("#submitButton")

    ; ✓ ControlClickJSElement — no window activation needed
    browser.ControlClickJSElement("button.accept", "left")

    ; ✓ JSClickElement — querySelector + JS .click(); faster but unreliable for some elements
    browser.JSClickElement("#simpleLink")

    ; ✓ JSGetElementPos — get screen coordinates of a querySelector result
    local pos := browser.JSGetElementPos("#mainContent", True)   ; True = use RenderWidget pos for DPI accuracy
    OutputDebug "Element at: x=" pos.x " y=" pos.y " w=" pos.w " h=" pos.h

    return pageTitle
}

; ✓ Handle JS alert dialogs
HandleAlert(browser) {
    ; ✓ GetAlertText waits up to 3s; closes alert by default
    local alertMsg := browser.GetAlertText(True, 3000)
    if alertMsg != ""
        OutputDebug "Alert said: " alertMsg
}

; ✓ Mozilla-specific: set JavascriptExecutionMethod for better JS performance
SetupMozilla(wTitle) {
    local moz := UIA_Mozilla(wTitle)
    ; ✓ "Bookmark" method requires a bookmark with URL "javascript:%s" and keyword "javascript"
    moz.JavascriptExecutionMethod := "Bookmark"   ; faster than Console (default)
    return moz
}

; ✗ WRONG — JSExecute during ongoing page load cancels the navigation
; browser.Navigate(url)   ; starts navigation
; browser.JSExecute(js)   ; → cancels Navigate; page never loads

; ✗ WRONG — JSReturnThroughClipboard during concurrent clipboard use
; A_Clipboard := "important data"
; local val := browser.JSReturnThroughClipboard(js)  ; → "important data" may be lost
```

### Performance Notes

Per Microsoft's official guidance, walking the UIA tree is resource-intensive — use `FindFirst`/`FindAll` on the DocumentElement rather than a TreeWalker. `GetAllText()` and `GetAllLinks()` both call `FindAll` on the entire `BrowserElement` subtree, which includes navigation UI — filter results by checking element positions if only page content is needed. `JSReturnThroughClipboard` is fast for returning a single value but adds a `ClipWait 2` timeout — prefer batching multiple JS queries into one JSON-returning call rather than calling it repeatedly. `WaitPageLoad` polls every 40ms — for known-fast navigations, pass a short `timeOut` to fail quickly rather than waiting indefinitely. `JSGetElementPos` uses `JSReturnThroughClipboard` internally — it incurs the full clipboard round-trip cost; cache the result if the position is needed multiple times. Constructing a new `UIA_Browser` instance is expensive (it calls `GetCurrentMainPaneElement()` which walks the toolbar tree) — create once per script session and reuse. Calling `GetCurrentDocumentElement()` on a loaded page is fast (WaitElement resolves immediately) — calling it on a slow page blocks the thread for up to 3 seconds.

## DROP-IN RECIPES

```ahk
; SafeNavigate — Navigate + WaitPageLoad + re-fetch doc, returns doc element or throws
; ✓ Encapsulates the full navigation + document-acquisition cycle
SafeNavigate(browser, url, expectedTitle := "", timeoutMs := 15000) {
    if !(browser is Object)
        throw TypeError("SafeNavigate: browser must be a UIA_Browser instance", -1)
    if !(url is String) || url = ""
        throw ValueError("SafeNavigate: url must be a non-empty String", -1)
    ; ✓ Navigate calls SetURL + WaitPageLoad; returns False on timeout
    local loaded := browser.Navigate(url, expectedTitle, timeoutMs)
    if loaded = False
        throw Error("SafeNavigate: page load timed out after " timeoutMs "ms (url=" url ")", -1)
    ; ✓ Re-fetch document — Navigate already waited; this should resolve immediately
    local doc := browser.GetCurrentDocumentElement()
    if !IsObject(doc)
        throw Error("SafeNavigate: document element unavailable after load (url=" url ")", -1)
    return doc
}
; Call site: local doc := SafeNavigate(browser, "https://example.com/search", "Search Results")

; SafeSelectTab — switch to named tab + wait + return fresh document element
; ✓ Uses TabExist for safe check, handles SelectTab TargetError, re-fetches document
SafeSelectTab(browser, tabName, matchMode := 2, timeoutMs := 8000) {
    if !(browser is Object)
        throw TypeError("SafeSelectTab: browser must be a UIA_Browser instance", -1)
    if !(tabName is String) || tabName = ""
        throw ValueError("SafeSelectTab: tabName must be a non-empty String", -1)
    local tab := browser.TabExist(tabName, matchMode)
    if !tab
        throw Error("SafeSelectTab: tab '" tabName "' not found (matchMode=" matchMode ")", -1)
    try
        browser.SelectTab(tab)
    catch TargetError as err
        throw Error("SafeSelectTab: SelectTab failed — " err.Message, -1)
    browser.WaitPageLoad("", timeoutMs, 300)
    local doc := browser.GetCurrentDocumentElement()
    if !IsObject(doc)
        throw Error("SafeSelectTab: document element unavailable after tab switch to '" tabName "'", -1)
    return doc
}
; Call site: local doc := SafeSelectTab(browser, "放射線資訊", 2)
```

## ANTI-PATTERNS

| Pattern | Wrong | Correct | LLM Common Cause |
|---------|-------|---------|------------------|
| Caching DocumentElement across navigations | `local doc := cUIA.GetCurrentDocumentElement()` before `Navigate()`, reused after | Re-call `GetCurrentDocumentElement()` after every `Navigate()` / `WaitPageLoad()` | DOM node analogy — browser DOM nodes are stable across navigation; UIA COM references are not |
| IsObject check on GetTab result | `local t := cUIA.GetTab(name)` then `if !IsObject(t)` | `local t := cUIA.TabExist(name)` then `if !t` | Conflating GetTab() (throws) with TabExist() (returns 0); GetTab parallels FindElement not ElementExist |
| JSExecute during page load | `Navigate(url)` immediately followed by `JSExecute(js)` | `Navigate(url, title, timeout)` (blocks until loaded) then `JSExecute(js)` | Assuming JS runs in page context regardless of load state; address-bar navigation cancels the current load |
| GetCurrentURL without visibility check | `cUIA.GetCurrentURL()` in background loop | `if cUIA.IsBrowserVisible()` then `GetCurrentURL()` else `GetCurrentURL(True)` | Unaware of auto-activation side-effect for non-Mozilla browsers |
| WaitPageLoad for SPA route change | `Navigate(spaLink); WaitPageLoad("NewSection")` when only title changes | `clickSpaLink(); WaitTitleChange("NewSection", 8000)` | Assuming all page transitions involve a full load cycle; SPAs don't trigger Reload button state change |
| Repeated JSReturnThroughClipboard calls | `for item in list { val := JSReturnThroughClipboard("get('" item "')") }` | Batch into one JS call returning JSON array, call once | Underestimating clipboard round-trip cost; should batch JS queries like batching DB calls |
| Forgetting sleepAfter in WaitPageLoad | `WaitPageLoad(title, 10000, 0)` for heavy SPA pages | `WaitPageLoad(title, 10000, 800)` | Setting sleepAfter=0 causes the script to proceed before the DOM finishes rendering after the Reload button stabilises |

## SEE ALSO

> This module does NOT cover: UIA element discovery, tree traversal, FindFirst/FindAll/WaitElement within page documents — see Module_UIAElements.md
> This module does NOT cover: control pattern interaction (InvokePattern, ValuePattern, ExpandCollapse) for browser page elements — see Module_UIAPatterns.md
> This module does NOT cover: try/catch error class hierarchy, structured retry logic — see Module_Errors.md

- `Module_UIAElements.md` — element discovery (FindFirst, FindAll, ElementExist, WaitElement), tree scope constants, ControlType integers; use on `doc` objects returned by `GetCurrentDocumentElement()`
- `Module_UIAPatterns.md` — InvokePattern, ValuePattern, SelectionItemPattern; use when UIA element interaction inside the browser document requires explicit pattern calls rather than `element.Click()` / `element.Value :=`
- `Module_Errors.md` — TargetError/MethodError hierarchy, structured retry logic for transient COM failures during browser element access
