# Module_GraphicsAndScreen.md
<!-- DOMAIN: Graphics and Screen -->
<!-- SCOPE: GDI+ bitmap rendering, screen-to-file capture, DirectX surface access, and GUI overlay event wiring are not covered — see Module_DllCallAndMemory.md and Module_GUI.md. -->
<!-- TRIGGERS: ImageSearch(), PixelSearch(), PixelGetColor(), CoordMode(), MonitorGet(), MonitorGetWorkArea(), MonitorGetCount(), MonitorGetPrimary(), "screen coordinates", "pixel color", "find image on screen", "multi-monitor", "screen resolution", "transparent overlay", "DPI scaling", "work area", "image recognition", "visual automation" -->
<!-- CONSTRAINTS: ImageSearch and PixelSearch return Boolean in AHK v2 — never check ErrorLevel; use `if ImageSearch(&x, &y, ...)` for branching. CoordMode must be set explicitly at the top of every function that performs pixel, mouse, or image operations — it does not propagate between function scopes. ImageSearch option flags (*n tolerance, *TransCOLOR) must be embedded as string prefixes in the ImageFile argument — never passed as extra positional arguments. -->
<!-- CROSS-REF: Module_GUI.md, Module_Errors.md, Module_DllCallAndMemory.md -->
<!-- VERSION: AHK v2.0+ -->

## V1 → V2 BREAKING CHANGES

| v1 pattern (LLM commonly writes) | v2 correct form | Consequence |
|----------------------------------|-----------------|-------------|
| `if ErrorLevel = 0` after `ImageSearch` or `PixelSearch` | `if ImageSearch(&x, &y, ...)` — branch on Boolean return | v1 check is always false in v2; image is silently "never found" even when present |
| `ImageSearch, OutputX, OutputY, x1, y1, x2, y2, img` (legacy command syntax) | `ImageSearch(&foundX, &foundY, x1, y1, x2, y2, imgFile)` | SyntaxError or wrong parameter mapping in AHK v2 |
| `ImageSearch(&x,&y, x1,y1,x2,y2, 20, img)` — tolerance as positional argument | `ImageSearch(&x,&y, x1,y1,x2,y2, "*20 " img)` — tolerance embedded in ImageFile string | Extra argument causes runtime TypeError; tolerance is silently ignored or crashes |
| `PixelGetColor, x, y` — legacy command, sets ErrorLevel, returns decimal int | `color := PixelGetColor(x, y)` — returns hex string `"0xRRGGBB"` | Comparing returned string against an integer literal always fails |
| `CoordMode, Pixel, Screen` — legacy command syntax | `CoordMode("Pixel", "Screen")` | SyntaxError in AHK v2; script aborts |
| `new ScreenOverlay()` / `new ResolutionScaler()` | `ScreenOverlay()` / `ResolutionScaler()` | `new` keyword is AHK v1 OOP; AHK v2 calls `__New()` via `ClassName()` — `new` causes NameError |
| `A_ScreenWidth` / `A_ScreenHeight` as search bounds for any monitor | `MonitorGet(n, &l, &t, &r, &b)` for per-monitor bounds | Returns primary-monitor dimensions only; produces off-screen or clipped region on secondary monitors |

## API QUICK-REFERENCE

### CoordMode and Screen Dimensions

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `CoordMode()` | `CoordMode(TargetType, RelativeTo?)` | Previous setting string (`"Screen"`, `"Window"`, or `"Client"`) | — | TargetType: `"Pixel"`, `"Mouse"`, `"ToolTip"`, `"Caret"`, `"Menu"`. RelativeTo: `"Screen"`, `"Window"`, `"Client"`. Default RelativeTo is `"Screen"`. Must be called in every function that issues coordinate ops — does not propagate between function scopes |
| `A_ScreenWidth` | read-only Integer | Primary monitor pixel width | — | Primary monitor only — not valid for secondary monitors |
| `A_ScreenHeight` | read-only Integer | Primary monitor pixel height | — | Primary monitor only — not valid for secondary monitors |
| `A_ScreenDPI` | read-only Integer | Current DPI as Integer | — | 96 = 100%, 144 = 150%, 192 = 200% |

### Mouse and Click

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `MouseGetPos()` | `MouseGetPos(&OutputX?, &OutputY?, &OutputVarWin?, &Control?)` | — | — | Returns position in coordinates established by `CoordMode("Mouse", ...)` |
| `MouseMove()` | `MouseMove(X, Y, Speed?, Relative?)` | — | — | Moves cursor; respects current CoordMode |
| `Click()` | `Click(Options*)` | — | — | `Click(x, y)`, `Click("Right", x, y)`, `Click(x, y, 2)` for double-click |

### Pixel Operations

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `PixelGetColor()` | `PixelGetColor(X, Y, Mode?)` | Hex string `"0xRRGGBB"` | `OSError` on failure | Mode: `"Alt"` (~10% slower, more reliable for certain windows), `"Slow"` (most reliable, works in some full-screen apps). Requires prior `CoordMode("Pixel", ...)` |
| `PixelSearch()` | `PixelSearch(&OutputX, &OutputY, X1, Y1, X2, Y2, ColorID, Variation?)` | Boolean (`1` = found, `0` = not found) | `OSError` if search could not be conducted | `Variation` 0–255: per-channel tolerance. Blanks output vars when not found. Requires prior `CoordMode("Pixel", ...)` |

### Image Search

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `ImageSearch()` | `ImageSearch(&OutputX, &OutputY, X1, Y1, X2, Y2, ImageFile)` | Boolean (`1` = found, `0` = not found) | `ValueError` if image file invalid or unloadable; `OSError` on internal failure | Option flags embedded as string prefixes in `ImageFile`: `"*n"` = variation 0–255, `"*TransCOLOR"` = treat hex color as transparent, `"*wN *hN"` = scale image. Never pass flags as extra positional arguments |

### Window Position

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `WinGetPos()` | `WinGetPos(&X?, &Y?, &Width?, &Height?, WinTitle?)` | — | `TargetError` if window not found | `"A"` targets active window; omitted output vars are skipped |
| `WinMove()` | `WinMove(X, Y, Width?, Height?, WinTitle?)` | — | `TargetError` if window not found | Repositions and/or resizes window |

### Monitor Functions

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `MonitorGetCount()` | `MonitorGetCount()` | Integer (total monitors) | — | Includes all connected monitors |
| `MonitorGetPrimary()` | `MonitorGetPrimary()` | Integer (1-based index) | — | Index of the primary monitor |
| `MonitorGet()` | `MonitorGet(N, &Left, &Top, &Right, &Bottom)` | — | — | Full monitor bounds including taskbar. N is 1-based |
| `MonitorGetWorkArea()` | `MonitorGetWorkArea(N, &Left, &Top, &Right, &Bottom)` | — | — | Work-area bounds excluding taskbar and reserved screen edges |

### Overlay and Window Transparency

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `WinSetTransColor()` | `WinSetTransColor(Color, WinTitle?)` | — | `TargetError` if window not found | Chroma-key transparency — makes one specific color fully invisible. Append space + alpha (0–255) to also set uniform transparency: `"RRGGBB 150"` |
| `WinSetTransparent()` | `WinSetTransparent(N, WinTitle?)` | — | `TargetError` if window not found | Applies uniform alpha 0–255 to the **entire** window including controls — not interchangeable with `WinSetTransColor` |
| `ObjBindMethod()` | `ObjBindMethod(Obj, Method, Params*)` | `BoundFunc` object | — | Creates a bound callback for `SetTimer` that calls `Obj.Method(Params*)` — use instead of multi-statement arrow functions |

## AHK V2 CONSTRAINTS

- `ImageSearch` and `PixelSearch` return **Boolean** in AHK v2 — use `if ImageSearch(&x, &y, ...)` for branching; never check `ErrorLevel` after these calls — `ErrorLevel` is always zero and produces permanently wrong logic.
  - ✗ `if ErrorLevel = 0` after ImageSearch — silent logic failure; the false branch never runs even when image is absent
  - ✓ `if ImageSearch(&foundX, &foundY, x1, y1, x2, y2, imgFile)` — correct Boolean branch

- `CoordMode` must be called explicitly in every function that uses coordinate operations — AHK v2 does not propagate CoordMode between function scopes; omitting it defaults to active-window-relative mode (`"Client"`), producing wrong coordinates when the active window changes.
  - ✗ Bare `PixelGetColor(x, y)` with no prior `CoordMode` — returns window-relative values, not screen coordinates
  - ✓ `CoordMode("Pixel", "Screen")` as the first line of any screen-automation function

- ImageSearch option flags (`*n` variation, `*TransCOLOR` transparency, `*wN *hN` scaling) must be embedded as a string prefix in the `ImageFile` parameter — they are not separate positional arguments.
  - ✗ `ImageSearch(&x,&y, 0,0,1920,1080, 20, "btn.png")` — TypeError at runtime
  - ✓ `ImageSearch(&x,&y, 0,0,1920,1080, "*20 btn.png")` — correct embedded prefix

- `A_ScreenWidth` and `A_ScreenHeight` return primary-monitor dimensions only — never use them as search region bounds in any multi-monitor context; use `MonitorGet(n, &l, &t, &r, &b)` for per-monitor bounds — using the primary dimensions on a secondary monitor produces an off-screen search region that can never match.
  - ✗ `ImageSearch(&x,&y, 0,0, A_ScreenWidth, A_ScreenHeight, img)` on monitor 2 — off-screen region
  - ✓ `MonitorGet(2, &ml,&mt,&mr,&mb)` then `ImageSearch(&x,&y, ml,mt,mr,mb, img)`

- `PixelGetColor` returns a hex string `"0xRRGGBB"` — compare against matching hex string literals or call `Integer()` to convert for arithmetic; comparing against bare integers silently fails.
  - ✗ `if (PixelGetColor(x, y) = 0xFF0000)` — integer vs string comparison, always false
  - ✓ `if (PixelGetColor(x, y) = "0xFF0000")` or `Integer(PixelGetColor(x, y)) > threshold`

- `WinSetTransColor` removes one specific chroma-key color from an overlay window; `WinSetTransparent` applies uniform alpha to the entire window including all controls — the two are not interchangeable for debug overlay construction.
  - ✗ `WinSetTransparent(200, overlayGui)` — makes all controls semi-transparent, destroying visibility
  - ✓ `WinSetTransColor(bgColor, overlayGui)` — removes only the chroma-key; colored controls remain fully opaque

- Instantiate classes with `ClassName()` — never `new ClassName()` — `new` is AHK v1 syntax and causes NameError in AHK v2.
  - ✗ `overlay := new ScreenOverlay()` — NameError: `new` is not valid in v2
  - ✓ `overlay := ScreenOverlay()` — correct v2 class instantiation

Safe-access priority order for screen operations:
1. `PixelGetColor(x, y)` — O(1) single-point check; use when position is already known
2. `PixelSearch(&x, &y, x1, y1, x2, y2, color, variation)` — O(area) scan; use when position is unknown
3. `ImageSearch(&x, &y, x1, y1, x2, y2, imgFile)` — O(area) template match; use for UI element detection
4. `try/catch` around `ImageSearch` — only when the image file may not exist and the exception message carries diagnostic information worth logging

Unset variable handling: `ImageSearch` and `PixelSearch` blank their output VarRefs when the search fails — access `foundX`/`foundY` only inside the `if` true-branch; accessing them after a failed search yields an unset-variable runtime error.

Resource lifecycle: `ScreenOverlay` Gui objects must be explicitly destroyed via `Destroy()` or `AutoHide()` — AHK v2 does not guarantee `__Delete` timing, so leaving `AlwaysOnTop` transparent windows open without an explicit cleanup path leaks window handles.

## AGENT QA CHECKLIST

- [ ] Did I call `CoordMode("Pixel", "Screen")` as the first line inside every function that issues pixel, image, or mouse operations?
- [ ] Did I branch on the Boolean return of `ImageSearch` / `PixelSearch` rather than checking `ErrorLevel` after the call?
- [ ] Did I embed tolerance (`*n`) and transparency (`*TransCOLOR`) flags in the `ImageFile` string rather than passing them as extra positional arguments?
- [ ] Did I use `MonitorGet(n, &l, &t, &r, &b)` for per-monitor search bounds instead of `A_ScreenWidth` / `A_ScreenHeight`?

## RUNTIME ERROR MAPPING

| Exception Class | Trigger Condition | Detection Code | Fix |
|----------------|-------------------|----------------|-----|
| `ValueError` | `ImageSearch()` called when the image file does not exist or cannot be loaded (bad format, corrupt file) | `e.Message` contains "could not be loaded" or "invalid parameter" | Guard with `if FileExist(imgPath) = ""` before calling; wrap in `try/catch` for format errors |
| `OSError` | `PixelGetColor()` or `PixelSearch()` fails during screen capture — typically caused by a hardware-accelerated or DRM-protected window | `e.Message` describes the internal failure | Retry with `"Alt"` or `"Slow"` mode on `PixelGetColor`; ensure the target window is visible and not minimized |
| Unset variable runtime error | Accessing `foundX` or `foundY` outside the `if ImageSearch(...)` true-branch — output VarRefs are blanked on search failure | `e.Message` contains "unset variable" or references the output var name | Always place coordinate usage inside `if ImageSearch(&fx, &fy, ...) { ... }` true-branch only |

## TIER 1 — Basic Screen Information
> METHODS COVERED: CoordMode · A_ScreenWidth · A_ScreenHeight · MouseGetPos · MouseMove · Click · WinGetPos

Tier 1 covers reading primary monitor dimensions, setting coordinate modes, querying the mouse cursor position, and performing basic click and move operations in AHK v2. `A_ScreenWidth` and `A_ScreenHeight` reflect only the primary monitor. CoordMode must always precede any coordinate-dependent call to ensure screen-relative results — the default mode is active-window-relative (`"Client"`) and will produce wrong coordinates as soon as the active window changes.

```ahk
; ─── Primary monitor dimensions (primary monitor only) ───────────────────────
primaryW := A_ScreenWidth    ; e.g. 1920
primaryH := A_ScreenHeight   ; e.g. 1080

; ─── CoordMode must be set before ANY coordinate operation ───────────────────
; ✓ Screen-relative mode ensures consistent coordinates regardless of active window
CoordMode("Mouse", "Screen")  ; Mouse coords relative to screen (0,0)
CoordMode("Pixel", "Screen")  ; Required before PixelGetColor / PixelSearch / ImageSearch

; ✗ Omitting CoordMode — coordinates default to active-window-relative ("Client")
; MouseGetPos(&x, &y)   ; → returns window-relative coords, not screen coords

; ─── Query current cursor position ───────────────────────────────────────────
CoordMode("Mouse", "Screen")
MouseGetPos(&curX, &curY)
MsgBox("Mouse at screen coords: " curX ", " curY)

; ─── Move mouse to screen position ───────────────────────────────────────────
MouseMove(A_ScreenWidth // 2, A_ScreenHeight // 2)  ; Center of primary monitor
MouseMove(0, 0)                                      ; Top-left corner

; ─── Click at absolute screen coordinates ────────────────────────────────────
; ✓ CoordMode("Mouse","Screen") established above; clicks target screen-absolute coords
CoordMode("Mouse", "Screen")
Click(100, 200)              ; Left-click at (100, 200) screen coords
Click("Right", 300, 400)     ; Right-click
Click(500, 600, 2)           ; Double left-click

; ─── Get active window position and dimensions ───────────────────────────────
WinGetPos(&winX, &winY, &winW, &winH, "A")
MsgBox("Window: x=" winX " y=" winY " w=" winW " h=" winH)

; ─── Window-client-relative mouse position (for app-specific automation) ─────
; ✓ "Client" mode makes coordinates relative to window's client area — useful for
;   targeting controls inside a specific application regardless of window position
CoordMode("Mouse", "Client")
MouseGetPos(&clientX, &clientY)
MsgBox("Client-relative cursor: " clientX ", " clientY)
```

## TIER 2 — Pixel and Color Detection
> METHODS COVERED: PixelGetColor · PixelSearch · CoordMode · Integer()

Tier 2 covers `PixelGetColor` for single-point sampling and `PixelSearch` for regional color scanning. Both require `CoordMode("Pixel","Screen")` for absolute coordinates. `PixelSearch` returns Boolean in AHK v2 — the v1 `ErrorLevel` pattern is silently wrong and must never appear. `Variation` (0–255) controls how much each color channel may deviate from the target before a match is rejected.

```ahk
; ✓ CoordMode must be the first line of any function doing pixel operations
CoordMode("Pixel", "Screen")

; ─── PixelGetColor — returns hex string "0xRRGGBB" ───────────────────────────
color     := PixelGetColor(500, 300)              ; Standard mode
colorAlt  := PixelGetColor(500, 300, "Alt")       ; Alternate method; ~10% slower, more reliable for certain windows
colorSlow := PixelGetColor(500, 300, "Slow")      ; More elaborate method; may succeed in full-screen applications where other modes fail

; ─── Comparing colors — compare hex string, not integer ──────────────────────
; ✓ Compare against matching hex string literal — PixelGetColor returns "0xRRGGBB"
targetColor := "0xFF0000"   ; Pure red
if (color = targetColor)
    MsgBox("Pixel at (500,300) is red")

; ✗ Integer literal comparison always fails — PixelGetColor does not return an integer
; if (color = 0xFF0000)    ; → always false; string vs integer type mismatch

; ✓ Convert to integer only when arithmetic operations are needed
colorInt := Integer(PixelGetColor(500, 300))   ; "0xFF0000" → 16711680

; ─── PixelSearch — returns Boolean (True = found, False = not found) ─────────
; ✓ Variation parameter (0–255): maximum per-channel deviation accepted
if PixelSearch(&foundX, &foundY, 0, 0, A_ScreenWidth, A_ScreenHeight, 0xFF0000, 15) {
    MsgBox("Red pixel found at " foundX ", " foundY)
    Click(foundX, foundY)
} else {
    MsgBox("No red pixel found in search region")
}

; ✗ v1 ErrorLevel pattern — silently never executes the found-branch in v2
; PixelSearch, foundX, foundY, 0, 0, 1920, 1080, 0xFF0000, 5
; if ErrorLevel = 0  { ... }   ; → ErrorLevel is always 0; branch always taken regardless of result

; ─── Constrained region search (faster; reduces false positives) ─────────────
; ✓ Narrow bounds reduce O(area) scan cost significantly
if PixelSearch(&px, &py, 200, 100, 800, 600, 0x00FF00, 10)
    Click(px, py)   ; Click the found green pixel

; ─── Polling loop: wait for pixel to become a target color ───────────────────
CoordMode("Pixel", "Screen")
watchX := 960, watchY := 540, deadline := A_TickCount + 8000
Loop {
    if (PixelGetColor(watchX, watchY) = "0x00FF00") {
        MsgBox("Pixel turned green at " watchX ", " watchY)
        break
    }
    if (A_TickCount > deadline) {
        MsgBox("Timeout: pixel did not change to green")
        break
    }
    Sleep(150)   ; ✓ Yield CPU — never busy-wait without Sleep in a pixel polling loop
}
```

## TIER 3 — ImageSearch Fundamentals
> METHODS COVERED: ImageSearch · FileExist · CoordMode · WinGetPos

Tier 3 covers the `ImageSearch` function: correct Boolean return handling, embedded option-flag syntax for tolerance and transparency, constrained region searches, file validation guards, and timeout-loop patterns. `ImageSearch` returns True when the image is found and places the top-left corner coordinates in the reference output variables. Option flags are prefixed in the `ImageFile` string — never passed as extra positional arguments.

```ahk
; ✓ CoordMode("Pixel","Screen") is required before every ImageSearch call
CoordMode("Pixel", "Screen")
imgPath := A_ScriptDir "\assets\button.png"

; ─── Basic ImageSearch — returns True if found ───────────────────────────────
if ImageSearch(&foundX, &foundY, 0, 0, A_ScreenWidth, A_ScreenHeight, imgPath) {
    MsgBox("Found at: " foundX ", " foundY)
    Click(foundX, foundY)
} else {
    MsgBox("Image not found on primary monitor")
}

; ─── Tolerance (*n) — embedded in the ImageFile string, never as a positional arg ──
; ✗ Tolerance as separate argument — causes runtime TypeError
; ImageSearch(&x, &y, 0, 0, 1920, 1080, 20, imgPath)   ; → TypeError

; ✓ Tolerance flag prefixed in the ImageFile string — correct v2 form
tolerance := 20
if ImageSearch(&foundX, &foundY, 0, 0, A_ScreenWidth, A_ScreenHeight,
               "*" tolerance " " imgPath)
    Click(foundX, foundY)

; ─── Multiple option flags combined in one string ────────────────────────────
; *n = variation  |  *TransCOLOR = treat specified hex color as transparent
; ✓ Multiple flags are space-separated prefixes before the file path
transImg := "*30 *Trans000000 " imgPath    ; 30 variation; black treated as transparent
if ImageSearch(&foundX, &foundY, 0, 0, A_ScreenWidth, A_ScreenHeight, transImg)
    MsgBox("Found with transparent mask at " foundX ", " foundY)

; ─── Constrained region search (preferred: smaller area = faster) ────────────
; ✓ Explicit bounds reduce O(area) scan time; narrow to smallest reliable region
searchX1 := 100, searchY1 := 50, searchX2 := 900, searchY2 := 700
if ImageSearch(&foundX, &foundY, searchX1, searchY1, searchX2, searchY2, imgPath)
    MsgBox("Found in region at: " foundX ", " foundY)

; ─── File-existence guard before searching ───────────────────────────────────
; ✓ Guard prevents ValueError from ImageSearch when path is wrong — validate first
if !FileExist(imgPath)
    throw Error("ImageSearch aborted — file missing: " imgPath)

; ─── Timeout loop: wait up to 10 s for image to appear ──────────────────────
deadline := A_TickCount + 10000
Loop {
    if ImageSearch(&foundX, &foundY, 0, 0, A_ScreenWidth, A_ScreenHeight,
                   "*15 " imgPath) {
        Click(foundX, foundY)
        break
    }
    if (A_TickCount > deadline) {
        MsgBox("Image did not appear within 10 seconds")
        break
    }
    Sleep(300)   ; ✓ Throttle loop — do not spin at 100% CPU waiting for image
}

; ─── ImageSearch scoped to a specific window's bounds ────────────────────────
; ✓ Scoping to the window reduces area and eliminates false positives elsewhere
FindInWindow(imgFile, winTitle, tolerance := 15) {
    CoordMode("Pixel", "Screen")          ; ✓ CoordMode set inside function — does not inherit
    WinGetPos(&wx, &wy, &ww, &wh, winTitle)
    if ImageSearch(&fx, &fy, wx, wy, wx+ww, wy+wh, "*" tolerance " " imgFile)
        return [fx, fy]
    return []   ; Return empty array when not found
}

result := FindInWindow(imgPath, "Notepad")
if result.Length
    Click(result[1], result[2])
```

## TIER 4 — Multi-Monitor and Work-Area Calculation
> METHODS COVERED: MonitorGetCount · MonitorGetPrimary · MonitorGet · MonitorGetWorkArea · WinGetPos · WinMove · MouseGetPos · CoordMode · ImageSearch

Tier 4 covers the `MonitorGet` family of functions for accurate multi-monitor bounds, work-area queries that exclude the taskbar, per-monitor ImageSearch loops, mouse-to-monitor detection, and the `ScreenHelper` utility class that caches all monitor data in a Map array for reuse. `A_ScreenWidth` and `A_ScreenHeight` must not be used as search region bounds on secondary monitors — they return the primary monitor's dimensions only and produce an off-screen search region on any other monitor.

```ahk
; ─── Monitor inventory ────────────────────────────────────────────────────────
monitorCount := MonitorGetCount()    ; Total number of connected monitors
primaryIdx   := MonitorGetPrimary()  ; 1-based index of primary monitor

; ─── Full monitor bounds (includes taskbar / docked panels) ──────────────────
MonitorGet(1, &m1L, &m1T, &m1R, &m1B)
m1Width  := m1R - m1L    ; Pixel width of monitor 1
m1Height := m1B - m1T    ; Pixel height of monitor 1

; ─── Work-area bounds (excludes taskbar and reserved screen edges) ────────────
MonitorGetWorkArea(1, &wa1L, &wa1T, &wa1R, &wa1B)
taskbarH := m1B - wa1B   ; Approximate taskbar height

; ─── Per-monitor ImageSearch loop ────────────────────────────────────────────
; ✗ Only searches primary monitor — produces off-screen region for monitor 2
; ImageSearch(&x, &y, 0, 0, A_ScreenWidth, A_ScreenHeight, imgPath)   ; → wrong on monitor 2

; ✓ Iterate every monitor with its own correct bounds
CoordMode("Pixel", "Screen")
imgPath := A_ScriptDir "\assets\target.png"
found := false
Loop MonitorGetCount() {
    MonitorGet(A_Index, &ml, &mt, &mr, &mb)
    if ImageSearch(&foundX, &foundY, ml, mt, mr, mb, "*20 " imgPath) {
        found := true
        MsgBox("Found on monitor " A_Index " at " foundX ", " foundY)
        break
    }
}
if !found
    MsgBox("Image not found on any connected monitor")

; ─── Center a window on a specific monitor's work area ───────────────────────
; ✓ Uses MonitorGetWorkArea so the window lands in usable space, not behind taskbar
CenterOnMonitor(winTitle, monitorN) {
    MonitorGetWorkArea(monitorN, &wl, &wt, &wr, &wb)
    ww := wr - wl, wh := wb - wt
    WinGetPos(,,&winW, &winH, winTitle)
    newX := wl + ((ww - winW) // 2)
    newY := wt + ((wh - winH) // 2)
    WinMove(newX, newY,,, winTitle)
}
; CenterOnMonitor("MyApp", 2)

; ─── Determine which monitor the mouse is currently on ───────────────────────
CoordMode("Mouse", "Screen")
MouseGetPos(&mx, &my)
activeMonitor := 0
Loop MonitorGetCount() {
    MonitorGet(A_Index, &ml, &mt, &mr, &mb)
    if (mx >= ml && mx < mr && my >= mt && my < mb) {
        activeMonitor := A_Index
        break
    }
}
MsgBox("Mouse is on monitor index: " activeMonitor)

; ─── Cache all monitor data in Map array (call once; reuse everywhere) ────────
; ✓ MonitorGet data is stable at runtime — calling it inside a tight loop wastes syscalls
GetAllMonitors() {
    monitors := []
    Loop MonitorGetCount() {
        MonitorGet(A_Index, &ml, &mt, &mr, &mb)
        MonitorGetWorkArea(A_Index, &wl, &wt, &wr, &wb)
        monitors.Push(Map(
            "index",     A_Index,
            "left",      ml, "top",      mt, "right",      mr, "bottom",      mb,
            "width",     mr - ml,        "height",         mb - mt,
            "workLeft",  wl, "workTop",  wt, "workRight",  wr, "workBottom",  wb,
            "workW",     wr - wl,        "workH",          wb - wt
        ))
    }
    return monitors
}

g_Monitors := GetAllMonitors()   ; ✓ Cached once at script load; reuse across all functions

; ─── ScreenHelper utility class — encapsulates CoordMode + monitor boilerplate ──
class ScreenHelper {
    ; ✓ Set screen-relative CoordMode for both pixel and mouse contexts in one call
    static SetScreenMode() {
        CoordMode("Pixel", "Screen")
        CoordMode("Mouse", "Screen")
    }

    ; ✓ Primary bounds via MonitorGet + MonitorGetPrimary — never A_ScreenWidth directly
    static GetPrimaryBounds(&left, &top, &right, &bottom) {
        MonitorGet(MonitorGetPrimary(), &left, &top, &right, &bottom)
    }

    ; ✓ Returns Array of Maps, one per monitor, with full and work-area bounds
    static GetAllMonitors() {
        monitors := []
        Loop MonitorGetCount() {
            MonitorGet(A_Index, &ml, &mt, &mr, &mb)
            MonitorGetWorkArea(A_Index, &wl, &wt, &wr, &wb)
            monitors.Push(Map(
                "index",      A_Index,
                "left",       ml, "top",       mt, "right",      mr, "bottom",      mb,
                "width",      mr - ml,         "height",         mb - mt,
                "workLeft",   wl, "workTop",   wt, "workRight",  wr, "workBottom",  wb
            ))
        }
        return monitors
    }

    ; ✓ ImageSearch scoped to primary monitor with embedded tolerance option
    static FindOnPrimary(imgFile, &foundX, &foundY, tolerance := 15) {
        ScreenHelper.SetScreenMode()
        MonitorGet(MonitorGetPrimary(), &l, &t, &r, &b)
        return ImageSearch(&foundX, &foundY, l, t, r, b, "*" tolerance " " imgFile)
    }
}
```

## TIER 5 — Dynamic Resolution Adaptation
> METHODS COVERED: ResolutionScaler · WindowRelative · A_ScreenDPI · WinGetPos · ImageSearch · CoordMode · MonitorGet · MouseGetPos

Tier 5 covers the mathematical models for making scripts portable across different screen resolutions and DPI settings. Absolute pixel coordinates break when the target machine has a different resolution. The solution is to express all positions as fractions of the current screen or window dimensions, or to scale from a known reference resolution using computed X and Y scale factors.

```ahk
; ─── Screen-ratio scaler: scale from reference resolution to current ──────────
class ResolutionScaler {
    refW   := 1920
    refH   := 1080
    scaleX := 1.0
    scaleY := 1.0

    __New(refW := 1920, refH := 1080) {
        this.refW   := refW
        this.refH   := refH
        this.scaleX := A_ScreenWidth  / refW   ; Primary monitor only — use MonitorGet for secondary
        this.scaleY := A_ScreenHeight / refH
    }

    ; Map a reference-resolution point to the current screen
    Scale(refX, refY) {
        return [Round(refX * this.scaleX), Round(refY * this.scaleY)]
    }

    ; Scale and left-click — sets its own CoordMode inside
    ScaleClick(refX, refY) {
        coords := this.Scale(refX, refY)
        CoordMode("Mouse", "Screen")   ; ✓ CoordMode set inside method — not inherited
        Click(coords[1], coords[2])
    }

    ; Scale a rectangle region for use with ImageSearch
    ScaleRegion(refX1, refY1, refX2, refY2) {
        return [Round(refX1 * this.scaleX), Round(refY1 * this.scaleY),
                Round(refX2 * this.scaleX), Round(refY2 * this.scaleY)]
    }
}

; ✓ Instantiate with ClassName() — no "new" keyword
scaler := ResolutionScaler()              ; Default ref: 1920×1080
scaler4K := ResolutionScaler(3840, 2160) ; Custom ref resolution
coords := scaler.Scale(960, 540)          ; Maps ref-center to current-center
scaler.ScaleClick(100, 200)              ; Click at scaled position

; ─── Window-fraction coordinate model ────────────────────────────────────────
; ✓ Positions as fractions of the target window's actual dimensions —
;   fracX=0.5, fracY=0.5 → center of window regardless of its current size
class WindowRelative {
    _winTitle := ""

    __New(winTitle) {
        this._winTitle := winTitle
    }

    ; Return [absX, absY] for a fractional position within the window
    FracCoord(fracX, fracY) {
        WinGetPos(&wx, &wy, &ww, &wh, this._winTitle)
        return [wx + Round(ww * fracX), wy + Round(wh * fracY)]
    }

    ; Click at fractional window position
    FracClick(fracX, fracY) {
        coords := this.FracCoord(fracX, fracY)
        CoordMode("Mouse", "Screen")   ; ✓ CoordMode set inside method — not inherited
        Click(coords[1], coords[2])
    }

    ; ImageSearch scoped to window, scaling tolerance by window vs reference size
    FindScaled(imgFile, refWinW := 1280, tolerance := 15) {
        WinGetPos(&wx, &wy, &ww, &wh, this._winTitle)
        CoordMode("Pixel", "Screen")   ; ✓ CoordMode set inside method — not inherited
        ; ✓ Adjust tolerance proportionally if window is smaller than reference
        scaledTol := Max(5, Round(tolerance * (ww / refWinW)))
        return ImageSearch(&foundX, &foundY, wx, wy, wx+ww, wy+wh,
                           "*" scaledTol " " imgFile)
            ? [foundX, foundY]
            : []
    }
}

wr := WindowRelative("Notepad")   ; ✓ ClassName() — no "new"
wr.FracClick(0.5, 0.1)            ; Top-center of Notepad regardless of window size

; ─── DPI-aware coordinate adjustment ─────────────────────────────────────────
; ✓ A_ScreenDPI: current DPI setting (96 = 100%, 144 = 150%, 192 = 200%)
dpiScale := A_ScreenDPI / 96.0
ScaleToDPI(logicalX, logicalY) {
    dpi := A_ScreenDPI / 96.0
    return [Round(logicalX * dpi), Round(logicalY * dpi)]
}

; ─── Per-monitor DPI: ImageSearch region using actual monitor work-area ───────
; ✓ Detects which monitor the mouse is on, then constrains ImageSearch to that monitor
FindOnCurrentMonitor(imgFile, &foundX, &foundY, tolerance := 15) {
    CoordMode("Mouse", "Screen")
    MouseGetPos(&mx, &my)
    CoordMode("Pixel", "Screen")
    Loop MonitorGetCount() {
        MonitorGet(A_Index, &ml, &mt, &mr, &mb)
        if (mx >= ml && mx < mr && my >= mt && my < mb) {
            return ImageSearch(&foundX, &foundY, ml, mt, mr, mb,
                               "*" tolerance " " imgFile)
        }
    }
    return false
}
```

## TIER 6 — Visual Debug Overlays
> METHODS COVERED: Gui() · WinSetTransColor · SetTimer · ObjBindMethod · ImageSearch · CoordMode

Tier 6 covers transparent, click-through GUI overlays that visualize ImageSearch results, scan regions, and found coordinates directly on screen. The overlay uses `WS_EX_TRANSPARENT` (`+E0x20`) for click-through behavior and `WinSetTransColor` for chroma-key transparency. `WinSetTransColor` removes one specific color entirely from the window — the background color becomes invisible while colored controls remain visible. See `Module_GUI.md` for Gui lifecycle and threading rules.

```ahk
; ─── ScreenOverlay class — chroma-keyed transparent debug layer ───────────────
class ScreenOverlay {
    _gui     := ""
    _panels  := []
    _bgColor := "000001"   ; Near-black chroma key — will be made fully transparent

    __New() {
        this._Init()
    }

    _Init() {
        ; ✓ +E0x20 = WS_EX_TRANSPARENT: window passes mouse clicks through to underlying windows
        ; ✓ -Caption +ToolWindow: no title bar, no taskbar button
        this._gui := Gui("+AlwaysOnTop -Caption +ToolWindow +E0x20")
        this._gui.BackColor := this._bgColor
        ; ✓ WinSetTransColor removes only the chroma-key color — controls remain visible
        ; ✗ WinSetTransparent would apply uniform alpha to entire window including all controls
        WinSetTransColor(this._bgColor, this._gui)
        this._panels := []
    }

    ; ✓ Add a solid colored filled rectangle at absolute screen coordinates
    AddFilledRect(x, y, w, h, color := "00FF00") {
        ctrl := this._gui.Add("Text",
            "x" x " y" y " w" w " h" h " Background" color, "")
        this._panels.Push(ctrl)
    }

    ; ✓ Draw a hollow border rectangle using four thin Text bar controls
    MarkRect(x, y, w, h, color := "FF0000", thickness := 3) {
        this._AddBar(x,                 y,                 w,         thickness, color) ; Top
        this._AddBar(x,                 y + h - thickness, w,         thickness, color) ; Bottom
        this._AddBar(x,                 y,                 thickness, h,         color) ; Left
        this._AddBar(x + w - thickness, y,                 thickness, h,         color) ; Right
    }

    ; ✓ Mark a single found coordinate with a crosshair and optional text label
    MarkPoint(x, y, label := "", color := "FFFF00", armLen := 20) {
        half := armLen // 2
        this._AddBar(x - half, y - 1,    armLen, 3,      color)  ; Horizontal arm
        this._AddBar(x - 1,    y - half, 3,      armLen, color)  ; Vertical arm
        if (label != "") {
            lbl := this._gui.Add("Text",
                "x" (x + 6) " y" (y - 10) " w130 h18 Background" color, label)
            this._panels.Push(lbl)
        }
    }

    ; ✓ Show the overlay spanning the full primary monitor
    Show(alpha := 200) {
        WinSetTransColor(this._bgColor " " alpha, this._gui)
        this._gui.Show("x0 y0 w" A_ScreenWidth " h" A_ScreenHeight " NoActivate")
    }

    ; ✓ Show spanning a custom region (e.g., a specific monitor)
    ShowRegion(x, y, w, h, alpha := 200) {
        WinSetTransColor(this._bgColor " " alpha, this._gui)
        this._gui.Show("x" x " y" y " w" w " h" h " NoActivate")
    }

    ; ✓ Destroy and reinitialize — clears all visual elements
    Clear() {
        this._gui.Destroy()
        this._Init()
    }

    ; ✓ Schedule auto-clear after delayMs milliseconds via ObjBindMethod — never arrow function
    AutoHide(delayMs := 2500) {
        cb := ObjBindMethod(this, "Clear")
        SetTimer(cb, -delayMs)
    }

    ; ✓ Resource cleanup — called automatically when object goes out of scope
    __Delete() {
        try this._gui.Destroy()
    }

    _AddBar(x, y, w, h, color) {
        ctrl := this._gui.Add("Text",
            "x" x " y" y " w" w " h" h " Background" color, "")
        this._panels.Push(ctrl)
    }
}

; ─── Usage: visualize ImageSearch result with marked bounding box ─────────────
CoordMode("Pixel", "Screen")
imgPath := A_ScriptDir "\assets\target.png"
overlay := ScreenOverlay()   ; ✓ ClassName() — no "new" keyword

if ImageSearch(&foundX, &foundY, 0, 0, A_ScreenWidth, A_ScreenHeight,
               "*20 " imgPath) {
    ; ✓ Hollow green border 4 px outside the found image top-left
    overlay.MarkRect(foundX - 4, foundY - 4, 88, 88, "00FF00", 3)
    ; ✓ Crosshair + coordinate label at the exact found point
    overlay.MarkPoint(foundX, foundY, foundX "," foundY, "FFFF00")
    overlay.Show(210)
    overlay.AutoHide(3000)   ; ✓ Dissolve after 3 seconds
} else {
    ; ✓ Highlight the full scan region in semi-transparent red to confirm coverage
    overlay.AddFilledRect(0, 0, A_ScreenWidth, A_ScreenHeight, "FF0000")
    overlay.Show(60)         ; Low alpha = subtle tint
    overlay.AutoHide(2000)
    MsgBox("Image not found — scan region shown in red overlay")
}

; ─── Multi-monitor overlay spanning all monitors ─────────────────────────────
; ✓ Compute virtual desktop bounds via MonitorGet — never assume A_ScreenWidth covers all monitors
GetVirtualDesktopBounds(&vl, &vt, &vr, &vb) {
    vl := 99999, vt := 99999, vr := -99999, vb := -99999
    Loop MonitorGetCount() {
        MonitorGet(A_Index, &ml, &mt, &mr, &mb)
        vl := Min(vl, ml), vt := Min(vt, mt)
        vr := Max(vr, mr), vb := Max(vb, mb)
    }
}

GetVirtualDesktopBounds(&vLeft, &vTop, &vRight, &vBottom)
allMonitorOverlay := ScreenOverlay()
allMonitorOverlay.ShowRegion(vLeft, vTop, vRight - vLeft, vBottom - vTop, 40)
allMonitorOverlay.AutoHide(1500)
```

### Performance Notes

Screen and image operations are among the most CPU-intensive AHK v2 calls. `ImageSearch` and `PixelSearch` both scale at O(area) — time is proportional to the pixel count of the search region. Narrowing the search bounds is the highest-leverage optimization available. `PixelGetColor` is O(1) and should always be preferred when the pixel position is already known. `MonitorGet` data is stable for the script's lifetime and should be cached once at startup rather than called inside loops.

Sleep throttling in polling loops is mandatory. A loop calling `ImageSearch` or `PixelSearch` without `Sleep` will saturate a CPU core indefinitely. A minimum of `Sleep(100)` between iterations is recommended; `Sleep(300)` is appropriate for most UI automation waits.

Tolerance (`*n`) has a direct cost: higher values require more comparison work per pixel. `*10` is a good default for clean UI elements; `*20` handles anti-aliasing and minor rendering variation; values above `*50` are significantly slower and should be benchmarked before deployment. `*0` (exact match) is fastest but brittle — any rendering variation breaks it.

AlwaysOnTop transparent overlay windows must be destroyed promptly. Lingering overlays consume resources and can interfere with other windows. Use `AutoHide(delayMs)` with `ObjBindMethod` for guaranteed cleanup via `SetTimer`, with `__Delete()` as a scope-exit fallback.

```ahk
; ─── Narrow the search region — O(area) complexity ───────────────────────────
; ✓ Constrain to the smallest region that reliably contains the target
WinGetPos(&wx, &wy, &ww, &wh, "TargetApp")
if ImageSearch(&x, &y, wx, wy, wx+ww, wy+wh, imgPath)   ; ✓ Window-scoped
    Click(x, y)

; ✗ Wasteful — searching entire primary screen when target is inside one window
; ImageSearch(&x, &y, 0, 0, A_ScreenWidth, A_ScreenHeight, imgPath)   ; → O(1920*1080)

; ─── Cache MonitorGet results — stable for script lifetime ───────────────────
; ✓ Call once at startup; store in a variable and reuse
g_Monitors := GetAllMonitors()
; ✗ Calling MonitorGet inside a tight loop — unnecessary syscall per iteration

; ─── PixelGetColor vs PixelSearch ────────────────────────────────────────────
; ✓ PixelGetColor is O(1) — use when the pixel position is already known
color := PixelGetColor(knownX, knownY)

; ✓ PixelSearch is O(area) — use only when position is unknown
if PixelSearch(&x, &y, 0, 0, 500, 400, 0xFF0000, 5)
    Click(x, y)

; ─── Tolerance trade-off reference ───────────────────────────────────────────
; *0  = exact pixel match  (fastest; brittle — rendering variation breaks it)
; *10 = low tolerance       (good default for clean UI elements)
; *20 = medium tolerance    (handles anti-aliasing and minor rendering differences)
; *50+= high tolerance      (significantly slower; benchmark before deploying)

; ─── Sleep in polling loops — mandatory to prevent 100% CPU usage ─────────────
; ✓ Yield at least 100 ms between scans in any pixel or image polling loop
CoordMode("Pixel", "Screen")
Loop {
    if PixelSearch(&x, &y, 0, 0, A_ScreenWidth, A_ScreenHeight, 0xFF0000, 10)
        break
    Sleep(100)
}

; ─── Destroy overlay Gui promptly — do not leave AlwaysOnTop windows open ─────
overlay := ScreenOverlay()
; ... perform search and mark result ...
overlay.AutoHide(3000)   ; ✓ Guaranteed cleanup via SetTimer + ObjBindMethod
; __Delete() provides fallback cleanup when object goes out of scope
```

## DROP-IN RECIPES

```ahk
; FindImageOnAllMonitors — search every connected monitor in order with optional timeout
; ✓ Returns Map("found", true/false, "x", n, "y", n, "monitor", n) — never throws on not-found
FindImageOnAllMonitors(imgFile, tolerance := 15, maxWaitMs := 0) {
    if !(imgFile is String) || imgFile = ""
        throw TypeError("FindImageOnAllMonitors: imgFile must be a non-empty string", -1)
    if FileExist(imgFile) = ""
        throw Error("FindImageOnAllMonitors: image file not found — " imgFile, -1)

    deadline := (maxWaitMs > 0) ? (A_TickCount + maxWaitMs) : 0
    CoordMode("Pixel", "Screen")   ; ✓ Set inside function — not inherited from caller scope

    Loop {
        Loop MonitorGetCount() {
            MonitorGet(A_Index, &ml, &mt, &mr, &mb)
            if ImageSearch(&fx, &fy, ml, mt, mr, mb, "*" tolerance " " imgFile)
                return Map("found", true, "x", fx, "y", fy, "monitor", A_Index)
        }
        if !deadline || (A_TickCount >= deadline)
            break
        Sleep(200)   ; ✓ Yield CPU between full-screen sweeps
    }
    return Map("found", false, "x", 0, "y", 0, "monitor", 0)
}
; Call site: r := FindImageOnAllMonitors(A_ScriptDir "\btn.png", 20, 5000)
;            if r["found"] { Click(r["x"], r["y"]) }

; WaitForPixelColor — poll a single coordinate until it matches a target color or times out
; ✓ Returns true on match, false on timeout — never throws; Sleep prevents CPU saturation
WaitForPixelColor(x, y, targetColor, timeoutMs := 5000, intervalMs := 150) {
    if !(targetColor is String)
        throw TypeError("WaitForPixelColor: targetColor must be a hex string like ""0xFF0000""", -1)
    if (intervalMs < 50)
        intervalMs := 50   ; ✓ Enforce minimum sleep to prevent CPU starvation

    CoordMode("Pixel", "Screen")   ; ✓ Set inside function — not inherited
    deadline := A_TickCount + timeoutMs

    Loop {
        try {
            current := PixelGetColor(x, y)
            if (current = targetColor)
                return true
        } catch OSError {
            ; ✓ Swallow transient screen-capture failures; retry on next iteration
        }
        if (A_TickCount >= deadline)
            return false
        Sleep(intervalMs)
    }
}
; Call site: if WaitForPixelColor(960, 540, "0x00FF00", 8000) { MsgBox("Pixel is green") }
```

## ANTI-PATTERNS

| Pattern | Wrong | Correct | LLM Common Cause |
|---------|-------|---------|------------------|
| ErrorLevel check after ImageSearch / PixelSearch | `if ErrorLevel = 0` after call | `if ImageSearch(&x, &y, ...)` — branch on Boolean return | AHK v1 training data — v1 used ErrorLevel for both functions; in v2 this branch is always taken, silently masking every failure |
| Tolerance as positional argument | `ImageSearch(&x,&y, 0,0,w,h, 20, img)` | `ImageSearch(&x,&y, 0,0,w,h, "*20 " img)` | Missing v2 API knowledge — the embedded-flag syntax has no equivalent in most languages; LLMs default to positional parameter convention |
| A_ScreenWidth as multi-monitor search bound | `ImageSearch(&x,&y, 0,0, A_ScreenWidth, A_ScreenHeight, img)` for monitor 2 | `MonitorGet(2,&l,&t,&r,&b)` then `ImageSearch(&x,&y, l,t,r,b, img)` | Single-monitor assumption baked into most training data; `A_ScreenWidth` name implies "the screen" which models interpret as universal |
| Omitting CoordMode before pixel operations | Bare `PixelGetColor(x, y)` or `PixelSearch(&x,&y, ...)` with no prior CoordMode | `CoordMode("Pixel","Screen")` then `PixelGetColor(x, y)` | No equivalent concept in most language training data; models treat coordinate functions as self-contained, unaware that AHK v2 scopes coordinate mode per-call-context |
| PixelGetColor result compared to integer | `if (PixelGetColor(x, y) = 0xFF0000)` | `if (PixelGetColor(x, y) = "0xFF0000")` | v1 returned a decimal integer; most language color APIs return integers; LLMs default to integer comparison for color values |
| Instantiating overlay class with `new` | `new ScreenOverlay()` | `ScreenOverlay()` | AHK v1 and most OOP languages (Python, Java, C#, JS) use `new` for object construction; LLMs apply this cross-language habit to v2 |
| Multi-statement arrow function for SetTimer callback | `cb := () => { this.Clear() ... }` (multi-line body) | Named method + `ObjBindMethod(this, "Clear")` | Arrow functions with multi-statement bodies are idiomatic in JavaScript; AHK v2 arrow functions are single-expression only — multi-statement bodies silently parse incorrectly |

## SEE ALSO

> This module does NOT cover: GDI+ bitmap rendering, screen-to-file capture, and DirectX surface access → see Module_DllCallAndMemory.md
> This module does NOT cover: Gui overlay lifecycle management, event wiring, AlwaysOnTop threading rules, and persistent Gui window management beyond basic show-and-hide → see Module_GUI.md
> This module does NOT cover: try/catch patterns for ImageSearch file-not-found errors and I/O error recovery → see Module_Errors.md

- `Module_GUI.md` — Gui construction, event wiring, AlwaysOnTop window management, and threading rules needed when overlay Gui objects require user interaction or persistent behavior beyond `ScreenOverlay.AutoHide()`.
- `Module_Errors.md` — try/catch patterns for wrapping `ImageSearch` when the image file path may not exist at runtime, and structured error propagation from screen-automation functions.
- `Module_DllCallAndMemory.md` — GDI+ bitmap operations, `DllCall("gdi32\BitBlt")` screen capture to HBITMAP, and low-level pixel buffer access for scenarios where `PixelGetColor` throughput is insufficient.
- `Module_InputAndHotkeys.md` — hotkey-triggered screen automation workflows and `Click` / `Send` integration patterns used alongside ImageSearch result coordinates.