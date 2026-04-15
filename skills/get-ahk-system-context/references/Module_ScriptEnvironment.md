# Module_ScriptEnvironment.md
<!-- DOMAIN: Script Environment -->
<!-- SCOPE: GUI window management triggered by tray actions, COM object Release() internals, and DllCall-level process management are not covered — see Module_GUI.md and Module_SystemAndCOM.md. -->
<!-- TRIGGERS: #Requires, #SingleInstance, #Include, #NoTrayIcon, A_TrayMenu, A_IsAdmin, A_IsCompiled, A_ScriptDir, A_ScriptFullPath, A_LineFile, TraySetIcon(), OnExit(), ExitApp(), Reload(), SetWorkingDir(), "tray menu", "tray icon", "admin elevation", "single instance", "script startup", "compiler directive", "compiled script", "script lifecycle", "exit handler", Ahk2Exe, ";@Ahk2Exe-" -->
<!-- CONSTRAINTS: #Requires AutoHotkey v2.0 must be the very first non-comment line — placing it after executable code causes it to be silently ignored. A_TrayMenu is an object — all tray interaction uses OOP methods (.Add(), .Delete(), .Rename()); never the v1 Menu command. Always guard Run("*RunAs ...") with an A_IsAdmin check — omitting the guard causes an infinite restart loop when UAC is denied. OnExit handlers must complete in well under 5 seconds; Windows forcibly terminates the process during logoff/shutdown if the handler blocks. Tray and OnExit callbacks that reference class instance state must be bound with .Bind(this) — AHK v2 does not auto-bind method receivers. -->
<!-- CROSS-REF: Module_GUI.md, Module_Errors.md, Module_SystemAndCOM.md -->
<!-- VERSION: AHK v2.0+ -->

## V1 → V2 BREAKING CHANGES

| v1 pattern (LLM commonly writes) | v2 correct form | Consequence |
|----------------------------------|-----------------|-------------|
| `Menu, Tray, Add, Label, Handler` | `A_TrayMenu.Add("Label", Handler)` | Runtime error — v1 Menu command syntax does not exist in v2; tray customization requires OOP method calls |
| `#SingleInstance` with no argument | `#SingleInstance Force` | In both AHK v1 and v2, omitting the argument defaults to `Prompt` behavior; in production, always specify `Force` explicitly to replace the previous instance without showing a dialog |
| `#Include utils.ahk` (bare relative path) | `#Include %A_ScriptDir%\Lib\utils.ahk` | Path resolves correctly when interpreted but silently fails after Ahk2Exe compilation when the exe is run from a different working directory |
| `Run("*RunAs " A_ScriptFullPath)` with no guard | `if !A_IsAdmin { Run("*RunAs ...") ; ExitApp() }` | Infinite restart loop — if UAC is disabled or the user cancels the prompt the relaunched script immediately tries to elevate again |
| `OnExit(this.Cleanup)` (unbound method) | `OnExit(this.Cleanup.Bind(this))` | `this` is undefined inside the handler — OOP cleanup silently breaks because the event system does not auto-bind method receivers |
| `ExitApp` (no parentheses) | `ExitApp()` | In AHK v2, functions can be called without parentheses; bare `ExitApp` is valid but explicit parentheses are recommended for clarity |
| `MsgBox, text` (command form) | `MsgBox("text")` | Parse error in v2 — the comma-separated command form is invalid; use `MsgBox "text"` or `MsgBox("text")` |
| `A_IsCompiled` assumed false in all paths | Check `if A_IsCompiled` and branch relaunch accordingly | Compiled .exe must relaunch itself; interpreted .ahk must pass the script path to `A_AhkPath` — a single command line fails one context |

## API QUICK-REFERENCE

### Directives (load-time, not runtime-conditional)

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `#Requires` | `#Requires AutoHotkey v2.0` | — | Parse abort | Must be first non-comment line; enforces interpreter version — omitting allows v1 to attempt parsing |
| `#SingleInstance` | `#SingleInstance Force\|Prompt\|Ignore\|Off` | — | — | `Force` silently replaces prior instance; `Prompt` shows dialog; production code always uses `Force`; omitting the argument defaults to `Prompt` |
| `#Include` | `#Include %A_ScriptDir%\Path\File.ahk` or `#Include <LibName>` | — | Load abort on missing file | Load-time only — cannot appear inside if/else; bare relative paths break after compilation |
| `#NoTrayIcon` | `#NoTrayIcon` | — | — | Suppress tray icon entirely; place near top of file before executable code |

### Built-in A_* Variables (Script Properties)

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `A_ScriptDir` | read-only | String | — | Directory of the running script — no trailing backslash; O(1) read |
| `A_ScriptName` | read-only | String | — | Filename only, without directory path (e.g., `"MyScript.ahk"`) |
| `A_ScriptFullPath` | read-only | String | — | Full absolute path including filename — correct in both compiled and interpreted modes |
| `A_WorkingDir` | read/write | String | — | Current working directory — may differ from `A_ScriptDir`; anchor with `SetWorkingDir()` |
| `A_IsCompiled` | read-only | Integer | — | `1` when running as compiled .exe; `0` when running as interpreted .ahk |
| `A_IsAdmin` | read-only | Integer | — | `1` = UAC-elevated; `0` = standard user token |
| `A_AhkPath` | read-only | String | — | Full path to AutoHotkey.exe interpreter — required for relaunch in interpreted mode |
| `A_AhkVersion` | read-only | String | — | Interpreter version string (e.g., `"2.0.11"`) |
| `A_LineFile` | read-only | String | — | Full path of the file containing the currently executing line — useful for library diagnostics |
| `A_IconTip` | read/write | String | — | Tray icon tooltip text; max 127 chars; blank defaults to script name |
| `A_IconHidden` | read/write | Integer | — | `0` = tray visible; `1` = tray hidden; equivalent to `#NoTrayIcon` at runtime |
| `A_TrayMenu` | read-only | Menu object | — | Built-in Menu object for the tray icon — interact via OOP methods only |
| `A_ScriptHwnd` | read-only | Integer | — | HWND of the hidden script message window |
| `A_Temp` | read-only | String | — | System temporary directory |
| `A_AppData` | read-only | String | — | `%APPDATA%` roaming folder |
| `A_MyDocuments` | read-only | String | — | User Documents folder |
| `A_Desktop` | read-only | String | — | User Desktop path |
| `A_Args` | read-only | Array | — | Array of command-line parameters passed to the script |

### A_TrayMenu (Menu Object Methods)

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `.Add()` | `.Add([ItemName, CallbackOrSubmenu, Options])` | — | — | No arguments inserts a separator line; callbacks receive `(ItemName, ItemPos, MyMenu)` |
| `.Delete()` | `.Delete([ItemName])` | — | Error if name not found | No argument removes ALL items; named argument removes one item |
| `.Rename()` | `.Rename(OldName, NewName)` | — | Error if name not found | Rename an existing item in-place |
| `.Default` | `.Default := "ItemName"` | String | — | Item triggered on double-click of the tray icon |

### Standalone Functions

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `TraySetIcon()` | `TraySetIcon(FileName?, IconNumber?, Freeze?)` | — | — | Set tray icon image; use A_ScriptDir-anchored path |
| `SetWorkingDir()` | `SetWorkingDir(DirName)` | — | OSError | Anchor working directory; call at startup with `A_ScriptDir` |
| `OnExit()` | `OnExit(Callback, AddRemove?)` | — | — | Handler signature: `Func(ExitReason, ExitCode)` — return `1` to cancel exit, `0` to allow |
| `ExitApp()` | `ExitApp(ExitCode?)` | — | — | Terminate script; triggers all registered OnExit handlers; `__Delete` is NOT called on local vars |
| `Reload()` | `Reload()` | — | — | Restart script; triggers OnExit handlers with reason `"Reload"` |
| `Run()` | `Run(Target, WorkingDir?, Options?, &OutputVarPID?)` | — | Error on UAC cancel | `"*RunAs"` verb requests UAC elevation — always guard with `A_IsAdmin` check |
| `ProcessClose()` | `ProcessClose(PIDOrName)` | Integer (0 if not found) | — | Terminate a process by PID or name — call inside OnExit to clean up spawned processes |
| `IniRead()` | `IniRead(Filename, Section, Key, Default?)` | String | OSError if key absent and no default | Read one value from an INI file; omitting Default throws if key not found |
| `IniWrite()` | `IniWrite(Value, Filename, Section, Key)` | — | OSError | Write one value to an INI file |
| `Persistent()` | `Persistent(Persist?)` | Boolean | — | Prevent script from auto-exiting when no hotkeys/timers/GUI remain active |

### Ahk2Exe Compiler Directives (comments at runtime)

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `;@Ahk2Exe-SetMainIcon` | `;@Ahk2Exe-SetMainIcon path` | — | — | Embed icon into .exe PE header; ignored entirely at runtime |
| `;@Ahk2Exe-SetVersion` | `;@Ahk2Exe-SetVersion x.y.z` | — | — | Set ProductVersion string in PE headers |
| `;@Ahk2Exe-SetFileVersion` | `;@Ahk2Exe-SetFileVersion x.y.z.w` | — | — | Set FileVersion (4-part integer form) |
| `;@Ahk2Exe-SetProductName` | `;@Ahk2Exe-SetProductName Name` | — | — | Set product name shown in file properties |
| `;@Ahk2Exe-SetCompanyName` | `;@Ahk2Exe-SetCompanyName Name` | — | — | Set company name in PE headers |
| `;@Ahk2Exe-SetCopyright` | `;@Ahk2Exe-SetCopyright text` | — | — | Set copyright string in PE headers |
| `;@Ahk2Exe-SetDescription` | `;@Ahk2Exe-SetDescription text` | — | — | Set file description in PE headers |
| `;@Ahk2Exe-AddResource` | `;@Ahk2Exe-AddResource path, name` | — | — | Bundle an extra file into the compiled .exe as a named resource |

## AHK V2 CONSTRAINTS

- `#Requires AutoHotkey v2.0` must be the very first non-comment line in the script — placing it after any executable statement causes the interpreter to ignore it, allowing v1 or an older v2 build to silently load the script and produce incorrect behavior.
- `#SingleInstance Force` is the explicit and recommended choice — in AHK v2, omitting the argument defaults to `Prompt` (same as in v1), so bare `#SingleInstance` will show a dialog before replacing a prior instance; always specify `Force` explicitly in production to replace silently.
- All `#Include` paths must be anchored to `A_ScriptDir` (`#Include %A_ScriptDir%\Lib\File.ahk`) or use the angle-bracket library shorthand (`#Include <LibName>`) — bare relative paths resolve correctly during development but silently fail when the compiled .exe is launched from a different working directory.
- `#Include` is a load-time directive — it cannot appear inside `if`, `else`, or any conditional block; attempting to do so causes a parse error at load time.
- `A_TrayMenu` is a Menu object — every tray interaction must use OOP method calls (`.Add()`, `.Delete()`, `.Rename()`, `.Default`); the v1 `Menu, Tray, ...` command form does not exist in v2 and produces a runtime error.
  - ✗ `Menu, Tray, Add, Label, MyFunc` — runtime parse error in v2
  - ✓ `A_TrayMenu.Add("Label", MyFunc)` — correct OOP form
- Tray callbacks and OnExit handlers that access class instance state must be bound with `.Bind(this)` — AHK v2's event system invokes callbacks without a receiver, so an unbound method reference has no `this` context and will throw or produce wrong results.
  - ✗ `OnExit(this.Cleanup)` — `this` is undefined when the handler fires
  - ✓ `OnExit(this.Cleanup.Bind(this))` — receiver is captured at registration time
- Always guard `Run("*RunAs ...")` with `if !A_IsAdmin` — without the guard, if UAC is disabled or the user cancels the elevation prompt, the newly launched instance immediately tries to elevate again, creating an infinite restart loop.
  - ✗ `Run("*RunAs " A_ScriptFullPath) ; ExitApp()` — infinite loop when UAC denied
  - ✓ `if !A_IsAdmin { Run("*RunAs ...") ; ExitApp() }` — guard terminates the loop
- Always call `ExitApp()` immediately after the `Run("*RunAs ...")` call — omitting it allows the original non-elevated instance to continue running alongside the new elevated one.
- Ahk2Exe compiler directives (`;@Ahk2Exe-*`) must be placed before the first executable statement — the compiler's linear scan picks them up top-to-bottom; directives after executable code may be missed.
- OnExit handlers must complete quickly, well under 5 seconds — Windows enforces a termination timeout during logoff and shutdown; handlers that block on network calls, heavy serialisation, or UI dialogs will be killed mid-execution.
  - ✗ Heavy database or network operations inside OnExit — process killed by Windows timeout
  - ✓ Fast `FileAppend` or `IniWrite` for logoff/shutdown; heavy saves only for interactive exits
- Note: `ExitApp()` does NOT call `__Delete` on local variables and does NOT run `finally` blocks — resource cleanup that must survive `ExitApp()` belongs in an `OnExit` handler, not a destructor or finally block.
- Static configuration values (app name, version, mutex name, log path) belong in a dedicated `class AppConfig { static ... }` block — scattering them as inline magic strings makes maintenance and refactoring fragile.

Safe-access priority order for script lifecycle operations:
  1. `FileRead()` / `IniRead()` with a default — single-call reads that never throw on missing key
  2. `FileOpen()` / `IniRead()` inside `try/finally` — when partial reads or write-back is needed
  3. `A_IsAdmin` / `A_IsCompiled` check before branching — O(1) reads; cache once if called in tight loops
  4. `try/catch` on `Run("*RunAs ...")` — the caught Error carries the UAC denial message for user feedback

Unset variable handling: `IniRead()` throws `OSError` when the key is absent and no default is provided — always supply a default argument (`IniRead(file, section, key, "")`) or wrap in `try/catch`.

Resource lifecycle: `ExitApp()` bypasses `__Delete` and `finally` blocks — only `OnExit` callbacks are guaranteed to run on all exit paths; place all critical cleanup there.

## AGENT QA CHECKLIST

- [ ] Did I guard every `Run("*RunAs ...")` call with `if !A_IsAdmin` to prevent an infinite restart loop when UAC is denied?
- [ ] Did I bind every class method used as a tray or OnExit callback with `.Bind(this)` so `this` resolves at invocation time?
- [ ] Did I place `#Requires AutoHotkey v2.0` as the very first non-comment line, before all `#Include` directives and executable statements?
- [ ] Did I anchor all `#Include` paths to `A_ScriptDir` (not bare relative paths) so the script works correctly after Ahk2Exe compilation?

## RUNTIME ERROR MAPPING

| Exception Class | Trigger Condition | Detection Code | Fix |
|----------------|-------------------|----------------|-----|
| `Error` (from `Run`) | `Run("*RunAs ...")` fires when UAC is cancelled or disabled — script re-launches infinitely | Process count in Task Manager grows without bound; no explicit error shown | Add `if !A_IsAdmin` guard before `Run`; add `/restart` flag or wrap `Run` in `try/catch` and call `ExitApp()` inside the try block |
| Runtime error — `this` undefined or wrong value | `OnExit(this.Method)` or `A_TrayMenu.Add("X", this.Handler)` without `.Bind(this)` — callback fires without receiver context | `e.Message` references an unset variable or wrong property; handler silently misbehaves | Replace with `OnExit(this.Method.Bind(this))` and `A_TrayMenu.Add("X", this.Handler.Bind(this))` |
| `OSError` | `SetWorkingDir()` called with a path that does not exist at runtime (e.g., a removable drive) | `e.Message` contains the target path and "The system cannot find" | Check `DirExist(path) != ""` before calling `SetWorkingDir()`, or wrap in `try/catch OSError` |

## TIER 1 — Script Directives and Environment Variables
> METHODS COVERED: #Requires · #SingleInstance · #NoTrayIcon · SetWorkingDir() · A_ScriptDir · A_ScriptName · A_ScriptFullPath · A_WorkingDir · A_IsCompiled · A_IsAdmin · A_AhkVersion · A_Temp · A_AppData · A_IconTip

TIER 1 establishes the mandatory script header: version enforcement with `#Requires`, instance control with `#SingleInstance`, and safe reading of the built-in `A_*` path and state variables. Directives are processed at load time and must precede all executable code — omitting or misordering them causes silent runtime failures or allows incompatible interpreters to load the script.

Canonical header order: (1) `#Requires`, (2) `#SingleInstance`, (3) `#Include` directives (shared libraries before local modules), (4) global configuration class or constants block, (5) initialization call (`AppMain()` or equivalent).

```ahk
; ============================================================
; TIER 1 — Script Directives and Environment Variables
; ============================================================
#Requires AutoHotkey v2.0           ; ✓ First directive — enforces AHK v2 interpreter
#SingleInstance Force                ; ✓ Replace previous instance without prompting

; ✗ Omitting #Requires lets AHK v1 attempt to parse v2 syntax silently
; ✗ Placing #Requires after executable code — directive is ignored → silent version mismatch

; ✓ Anchor working directory for predictable relative paths
SetWorkingDir(A_ScriptDir)

; ---- Central constants class — no magic strings scattered through code ---
class AppConfig {
    static Name    := "MyApp"
    static Version := "1.0.0"
    static Mutex   := "Global\MyApp_SingleInstance"
    static LogFile := A_ScriptDir "\Logs\app.log"
}

; ---- Entry point ---
AppMain()

AppMain() {
    info := AppConfig.Name " v" AppConfig.Version
        . "`nScript dir:  " A_ScriptDir
        . "`nWorking dir: " A_WorkingDir
        . "`nCompiled:    " (A_IsCompiled ? "Yes (.exe)" : "No (.ahk)")
        . "`nAdmin:       " (A_IsAdmin    ? "Yes"        : "No")
    MsgBox(info, "Startup Info", 64)
}

; ---- Read core environment variables ----
ShowScriptInfo() {
    details := "Path Variables`n"
        . "  A_ScriptDir:      " A_ScriptDir      "`n"
        . "  A_ScriptName:     " A_ScriptName     "`n"
        . "  A_ScriptFullPath: " A_ScriptFullPath "`n"
        . "  A_WorkingDir:     " A_WorkingDir     "`n"
        . "  A_Temp:           " A_Temp           "`n"
        . "  A_AppData:        " A_AppData        "`n"
        . "`nRuntime State`n"
        . "  A_IsCompiled: " (A_IsCompiled ? "Yes (.exe)" : "No (.ahk)") "`n"
        . "  A_IsAdmin:    " (A_IsAdmin    ? "Yes"        : "No")        "`n"
        . "  A_AhkVersion: " A_AhkVersion
    MsgBox(details, "Script Environment", 64)
}

; ---- A_* variable quick reference (all O(1) built-in reads) ----
; A_ScriptDir        → directory of the running script (no trailing \)
; A_ScriptName       → filename without path  (e.g., MyScript.ahk)
; A_ScriptFullPath   → full absolute path including filename
; A_WorkingDir       → current working dir (may differ from A_ScriptDir)
; A_Temp             → system temporary directory
; A_MyDocuments      → user Documents folder
; A_AppData          → %APPDATA% roaming folder
; A_Desktop          → user Desktop path
; A_AhkPath          → full path to the AutoHotkey.exe interpreter
; A_Args             → Array of command-line parameters

; A_IsCompiled       → 1 (running as compiled .exe) or 0 (.ahk source)
; A_IsAdmin          → 1 (UAC elevated) or 0 (standard user token)
; A_AhkVersion       → interpreter version string (e.g., "2.0.11")
; A_LineFile         → full path of the file containing current line

; A_ScriptHwnd       → HWND of the hidden script message window
; A_IconTip          → tray tooltip text  (writable: A_IconTip := "text")
; A_IconHidden       → 0=tray visible, 1=tray hidden (writable)
; A_TrayMenu         → built-in Menu object for the tray icon

; ---- Version pin variants ----
; #Requires AutoHotkey >=2.0.11    ; minimum version with a specific fix
; #Requires AutoHotkey v2.0        ; any v2.0.x release

; ---- #NoTrayIcon — hide tray icon for background utility scripts ----
; #NoTrayIcon                      ; ✓ place near top; tray icon never appears

ShowScriptInfo()
```

## TIER 2 — Include Directives and Library System
> METHODS COVERED: #Include · A_ScriptDir · A_LineFile

TIER 2 covers the `#Include` system for splitting scripts across multiple files and the three-tier auto-library folder search path that AHK v2 uses when the angle-bracket form (`#Include <LibName>`) is specified. The key discipline is always anchoring paths to `A_ScriptDir` so includes survive compilation and deployment to machines where the working directory differs from the script directory.

```ahk
; ============================================================
; TIER 2 — #Include and the AHK v2 Library System
; ============================================================
#Requires AutoHotkey v2.0
#SingleInstance Force

; ✓ Correct: absolute path anchored to A_ScriptDir — works compiled and uncompiled
#Include %A_ScriptDir%\Lib\StringUtils.ahk
#Include %A_ScriptDir%\Lib\Logger.ahk

; ✓ Standard library shorthand — AHK v2 auto-searches these locations in order:
;     1. %A_ScriptDir%\Lib\
;     2. %A_MyDocuments%\AutoHotkey\Lib\
;     3. <AHK install dir>\Lib\
#Include <DateTimeLib>              ; resolves to Lib\DateTimeLib.ahk automatically

; ✗ Wrong: bare relative path — breaks when compiled exe is run from another dir
; #Include utils.ahk                ; → silent file-not-found at runtime after compilation

; ✗ Wrong: #Include is a load-time directive — cannot be placed inside if/else blocks
; if !A_IsCompiled { #Include debug.ahk }   ; → PARSE ERROR at load time

; ✓ Correct: gate runtime behavior using A_IsCompiled inside the included file itself
;     Inside DevTools.ahk:
;       if A_IsCompiled
;           return                   ; silently skip debug tools when compiled

; ---- A_LineFile: track which included file contains a given line ----
LogFileOrigin() {
    MsgBox("This function lives in: " A_LineFile)
}

; ---- Recommended multi-file project layout ----
; ProjectRoot\
;   Main.ahk                ; only: #Requires + #SingleInstance + #Includes + AppMain()
;   Lib\
;     AppConfig.ahk         ; class AppConfig { static ... }
;     Logger.ahk            ; class Logger { ... }
;     Utils.ahk             ; pure utility functions
;   Assets\
;     app.ico               ; tray and exe icon

; ✓ Include dependency order matters — include base classes before subclasses
;     #Include %A_ScriptDir%\Lib\AppConfig.ahk   ; no dependencies
;     #Include %A_ScriptDir%\Lib\Logger.ahk      ; depends on AppConfig
;     #Include %A_ScriptDir%\Lib\Utils.ahk       ; depends on Logger

LogFileOrigin()
```

## TIER 3 — Tray Menu Customization
> METHODS COVERED: A_TrayMenu.Add() · A_TrayMenu.Delete() · A_TrayMenu.Rename() · A_TrayMenu.Default · A_IconTip · TraySetIcon() · .Bind()

TIER 3 covers AHK v2's object-oriented tray menu API through the built-in `A_TrayMenu` Menu object. All interaction uses method calls (`.Add()`, `.Delete()`, `.Rename()`) rather than the v1 `"Menu, Tray, ..."` command form. Callbacks that require access to class instance state must be bound with `.Bind(this)` — unbound method references lose their object context when invoked by the tray event system.

```ahk
; ============================================================
; TIER 3 — Tray Menu Customization
; ============================================================
#Requires AutoHotkey v2.0
#SingleInstance Force

; ---- Set tray tooltip and icon ----
A_IconTip := "MyApp v1.0"                              ; ✓ tooltip on hover
TraySetIcon(A_ScriptDir "\Assets\app.ico", 1)          ; ✓ custom icon, index 1

; ---- Remove default AHK tray items ----
A_TrayMenu.Delete()                ; ✓ removes ALL items (standard + custom)
; To remove only the standard items, delete them individually by position
; (no DeleteStandard() exists in v2)

; ---- Add custom items ----
; Signature: MenuObject.Add([ItemName, CallbackOrSubmenu, Options])
A_TrayMenu.Add("About",    ShowAbout)
A_TrayMenu.Add("Settings", ShowSettings)
A_TrayMenu.Add()                   ; ✓ empty .Add() inserts a separator line
A_TrayMenu.Add("Exit",     (*) => ExitApp())

; ---- Set double-click default item ----
A_TrayMenu.Default := "About"      ; ✓ triggered when user double-clicks tray icon

; ---- Icon state feedback ----
A_TrayMenu.Add()
A_TrayMenu.Add("Pause", TogglePause)

; ---- Standalone callback functions ----
; ✓ Tray callbacks receive: (ItemName, ItemPos, MyMenu) — use (*) to accept and ignore
ShowAbout(*) {
    MsgBox("MyApp v1.0`nBuilt with AutoHotkey v2", "About", 64)
}

ShowSettings(*) {
    MsgBox("Settings placeholder — open GUI here.", "Settings", 64)
}

TogglePause(*) {
    static paused := false
    paused := !paused
    A_TrayMenu.Rename("Pause", paused ? "Resume" : "Pause")
    A_IconTip := "MyApp — " (paused ? "PAUSED" : "Running")
}

; ---- Class-bound tray callback ----
class AppController {
    _name := ""

    __New(name) {
        this._name := name
        ; ✓ .Bind(this) — ensures 'this' resolves to the instance at call time
        A_TrayMenu.Add("Status", this._ShowStatus.Bind(this))
    }

    _ShowStatus(*) {
        MsgBox("Controller '" this._name "' is running.`nScript: " A_ScriptName)
    }
}

; ✗ Wrong: unbound method — 'this' is undefined when the tray fires the callback
; A_TrayMenu.Add("Status", app._ShowStatus)    ; → runtime error inside _ShowStatus

app := AppController("MainController")
```

## TIER 4 — Admin Elevation Patterns
> METHODS COVERED: A_IsAdmin · A_IsCompiled · A_ScriptFullPath · A_AhkPath · A_Args · Run() · ExitApp()

TIER 4 covers the standard admin-elevation pattern for scripts that require elevated privileges: check `A_IsAdmin` first, then re-launch with the `"*RunAs"` verb and immediately exit the current non-elevated instance. The `A_IsAdmin` guard is mandatory — without it, a UAC denial or disabled UAC silently relaunches the script in an infinite loop. Compiled and interpreted scripts require different relaunch command lines because the compiled .exe IS the interpreter, while the interpreted form must pass the .ahk file path to `AutoHotkey.exe`.

```ahk
; ============================================================
; TIER 4 — Admin Elevation Patterns
; ============================================================
#Requires AutoHotkey v2.0
#SingleInstance Force

; ---- Standard elevation guard — call near top of AppMain() ----
EnsureAdmin() {
    if A_IsAdmin        ; ✓ Already elevated — return and continue normally
        return

    ; ✓ Build correct command line for compiled vs interpreted context
    try {
        if A_IsCompiled {
            ; ✓ Compiled exe: just re-run the .exe itself with RunAs verb
            Run('*RunAs "' A_ScriptFullPath '"')
        } else {
            ; ✓ Interpreted .ahk: pass interpreter path + script path
            Run('*RunAs "' A_AhkPath '" "' A_ScriptFullPath '"')
        }
    } catch Error as e {
        ; User cancelled UAC prompt or system cannot grant elevation
        MsgBox("Admin rights are required.`n`nError: " e.Message,
               "Elevation Failed", 16)
    }
    ExitApp()           ; ✓ Always exit after requesting elevation — prevents duplicate
}

; ---- Elevation with forwarded command-line arguments ----
EnsureAdminWithArgs() {
    if A_IsAdmin
        return

    try {
        args := A_Args.Length > 0 ? " " A_Args[1] : ""    ; forward first arg if present
        cmd  := A_IsCompiled
            ? '*RunAs "' A_ScriptFullPath '"' args
            : '*RunAs "' A_AhkPath '" "' A_ScriptFullPath '"' args
        Run(cmd)
    } catch Error as e {
        MsgBox("Elevation failed: " e.Message, , 16)
    }
    ExitApp()
}

; ✗ Wrong: no A_IsAdmin check before RunAs — infinite loop if UAC is denied
; Run("*RunAs " A_ScriptFullPath)    ; → infinite restart loop
; ExitApp()

; ---- Informational: report current privilege level ----
ReportPrivilege() {
    level := A_IsAdmin ? "Administrator (elevated)" : "Standard User (not elevated)"
    MsgBox("Running as: " level, "Privilege Level", 64)
}

EnsureAdmin()
ReportPrivilege()
```

## TIER 5 — Compiler Directives (Ahk2Exe Metadata)
> METHODS COVERED: ;@Ahk2Exe-SetMainIcon · ;@Ahk2Exe-SetVersion · ;@Ahk2Exe-SetFileVersion · ;@Ahk2Exe-SetProductName · ;@Ahk2Exe-SetCompanyName · ;@Ahk2Exe-SetCopyright · ;@Ahk2Exe-SetDescription · ;@Ahk2Exe-AddResource · A_IsCompiled

TIER 5 covers Ahk2Exe compiler directives, which are syntactically AHK comments at runtime but are parsed and acted upon by the Ahk2Exe compiler when producing a compiled .exe. They embed version information, icons, company metadata, and additional resource files directly into the PE (Portable Executable) headers of the output binary. Directives must be placed before the first executable statement to guarantee reliable pickup by the compiler's linear scan.

```ahk
; ============================================================
; TIER 5 — Ahk2Exe Compiler Directives
; ============================================================
; These lines are normal AHK comments at runtime (ignored completely).
; Ahk2Exe reads them during compilation to configure the output .exe.
; ✓ Place ALL directives before the first executable line.
; ============================================================

;@Ahk2Exe-SetMainIcon    %A_ScriptDir%\Assets\app.ico
;@Ahk2Exe-SetVersion     1.2.0
;@Ahk2Exe-SetFileVersion 1.2.0.0
;@Ahk2Exe-SetProductName MyApp
;@Ahk2Exe-SetCompanyName Acme Corp
;@Ahk2Exe-SetCopyright   Copyright 2025 Acme Corp
;@Ahk2Exe-SetDescription Automation Tool for Windows

; Embed an additional resource file (accessible at runtime via DllCall/LoadLibrary)
;@Ahk2Exe-AddResource    %A_ScriptDir%\Assets\splash.png, SPLASH_IMAGE

; ---- Executable code begins here ----
#Requires AutoHotkey v2.0
#SingleInstance Force

; ---- Runtime: detect compiled vs interpreted context ----
ReportBuildInfo() {
    if A_IsCompiled {
        info := "Running as compiled .exe"
            . "`nExe path:  " A_ScriptFullPath
            . "`nExe dir:   " A_ScriptDir
    } else {
        info := "Running as interpreted .ahk"
            . "`nAHK path:    " A_AhkPath
            . "`nAHK version: " A_AhkVersion
            . "`nScript:      " A_ScriptFullPath
    }
    MsgBox(info, "Build Info", 64)
}

; ---- Path resolution is identical in both modes ----
; ✓ A_ScriptDir resolves correctly whether running as .ahk or compiled .exe
GetResourcePath(filename) {
    return A_ScriptDir "\" filename

    ; ✗ Wrong: hardcoded path breaks on any machine other than the developer's
    ; return "C:\Dev\MyApp\Assets\" filename    ; → FileNotFound on any other machine
}

ReportBuildInfo()
MsgBox("Resource base: " GetResourcePath("app.ico"))
```

## TIER 6 — Script Lifecycle and OnExit Cleanup Hooks
> METHODS COVERED: OnExit() · ExitApp() · Reload() · ProcessClose() · IniRead() · IniWrite() · Persistent() · .Bind()

TIER 6 covers the complete script lifecycle via `OnExit()` callbacks, which fire on every exit path: close button, `ExitApp()`, `Reload()`, system logoff, and shutdown. OnExit handlers receive an ExitReason string and an exit code; returning `1` cancels the exit, while returning `0` (or omitting the return) allows it to proceed. Class-based cleanup must bind the handler with `.Bind(this)` to preserve instance context. Cleanup must be fast — Windows can forcibly terminate the process if the OnExit handler blocks for too long during system shutdown.

```ahk
; ============================================================
; TIER 6 — Script Lifecycle and OnExit Cleanup Hooks
; ============================================================
#Requires AutoHotkey v2.0
#SingleInstance Force

; ---- ExitReason values delivered to OnExit callbacks ----
; "Close"    → user clicked X on the script's hidden window or WM_CLOSE received
; "Error"    → unhandled runtime error caused exit (only in non-persistent scripts)
; "Logoff"   → Windows user logoff
; "Shutdown" → Windows system shutdown
; "Single"   → #SingleInstance replaced this instance
; "Reload"   → Reload() was called
; "Menu"     → user chose "Exit" from the default tray menu or main window menu
; "Exit"     → ExitApp() was called explicitly

; ✓ Persistent() prevents the script from auto-exiting when no hotkeys/timers remain
Persistent(true)

; ---- Class-based lifecycle with cleanup ----
class AppLifecycle {
    _configFile := ""
    _childPid   := 0
    _settings   := Map()

    __New(configFile) {
        this._configFile := configFile
        this._settings   := Map("theme", "dark", "volume", 80)

        ; ✓ Bind to preserve 'this' inside the OnExit callback
        OnExit(this._OnExit.Bind(this))

        ; Spawn a child process and track its PID
        childPid := 0
        Run("notepad.exe", , , &childPid)
        this._childPid := childPid
    }

    ; ✓ OnExit signature: Func(ExitReason, ExitCode) → return 1 to cancel, 0 to allow
    _OnExit(exitReason, exitCode) {
        ; ✓ Fast path for forced exits — no dialogs during shutdown
        if exitReason = "Logoff" || exitReason = "Shutdown" {
            this._SaveSettingsFast()
            this._KillChildren()
            return 0                ; allow exit immediately — Windows is not waiting
        }

        ; Confirmation dialog only for interactive close
        if exitReason = "Close" {
            result := MsgBox("Save settings and exit?", "Confirm Exit", 3 + 32)
            if result = "Cancel"
                return 1            ; ✓ Return 1 to cancel exit
        }

        this._SaveSettings()
        this._KillChildren()
        return 0                    ; ✓ Return 0 to allow exit to proceed
    }

    _SaveSettings() {
        try {
            IniWrite(this._settings["theme"],  this._configFile, "UI",    "Theme")
            IniWrite(this._settings["volume"], this._configFile, "Audio", "Volume")
        } catch Error as e {
            ; Settings save failed — log and continue; do not block exit
            FileAppend("Save failed: " e.Message "`n", A_Temp "\myapp_error.log")
        }
    }

    _SaveSettingsFast() {
        ; ✓ Single FileAppend is fast enough for logoff/shutdown window
        try FileAppend("exit=" A_Now "`n", A_Temp "\myapp_exit.log")
    }

    _KillChildren() {
        if this._childPid {
            try ProcessClose(this._childPid)
        }
    }
}

; ---- Functional (non-class) OnExit approach ----
OnExit(HandleExitGlobal)

HandleExitGlobal(reason, code) {
    ; ✓ Minimal fast cleanup suitable for simple scripts
    FileAppend("Script exited. Reason: " reason " Code: " code "`n",
               A_Temp "\simple_exit.log")
    return 0
}

; ---- Multiple OnExit handlers are allowed ----
; OnExit(HandlerA)     ; called first
; OnExit(HandlerB)     ; called second
; If a handler returns non-zero, exit is cancelled AND no subsequent handlers are called

; ✗ Wrong: relying solely on a hotkey for cleanup — misses logoff/shutdown
; ^q:: ExitApp()       ; → will not fire during Windows shutdown

; ✓ Correct: OnExit covers all exit paths including system shutdown

app := AppLifecycle(A_ScriptDir "\config.ini")
MsgBox("Running. Close this dialog to test OnExit cancel/confirm.", , 64)
ExitApp(0)
```

### Performance Notes

Script environment operations are startup-time, not hot-path. The primary performance discipline is: read file-based configuration once and cache it; build the tray menu once at startup; keep OnExit handlers fast enough to complete before Windows terminates the process during logoff or shutdown; prefer `A_*` built-in variables (O(1) reads) over repeated file or registry queries inside frequently-called functions.

```ahk
; ============================================================
; PERFORMANCE GUIDELINES — Script Environment
; ============================================================

; ---- 1. Cache IniRead results — file I/O on every call is wasteful ----
class Config {
    static _cache := Map()

    ; ✓ O(1) map lookup after first read; O(n) file I/O only on cache miss
    static Get(section, key, default := "") {
        cacheKey := section "." key
        if !Config._cache.Has(cacheKey) {
            Config._cache[cacheKey] := IniRead(
                A_ScriptDir "\config.ini", section, key, default)
        }
        return Config._cache[cacheKey]
    }

    ; ✓ Invalidate a single entry when the value is written
    static Set(section, key, value) {
        IniWrite(value, A_ScriptDir "\config.ini", section, key)
        Config._cache[section "." key] := value    ; ✓ sync cache on write
    }
}

; ---- 2. Build tray menu once at startup — not inside loops or hotkeys ----
InitTrayMenu() {
    ; ✓ Build once; O(1) item access thereafter
    A_TrayMenu.Delete()
    A_TrayMenu.Add("Open",  (*) => MsgBox("Open"))
    A_TrayMenu.Add("Exit",  (*) => ExitApp())
}
InitTrayMenu()                      ; single call at startup

; ✗ Wrong: rebuilding inside a SetTimer callback fires repeatedly
; SetTimer(() => InitTrayMenu(), 1000)   ; → wasteful — menu never changes

; ---- 3. Cache A_IsAdmin and A_IsCompiled — read-once booleans ----
IsAdmin    := A_IsAdmin             ; ✓ O(1) read; avoid re-evaluating in tight loops
IsCompiled := A_IsCompiled

CheckPrivilege() {
    ; ✓ Use cached value — no syscall on repeated checks
    return IsAdmin
}

; ---- 4. OnExit handlers must be fast — Windows timeout is ~5 seconds ----
HandleExit(reason, code) {
    ; ✓ Single FileAppend is O(1) and completes in microseconds
    try FileAppend(reason "`n", A_Temp "\exit.log")
    ; ✗ Wrong: complex serialisation or network calls in OnExit may time out
    ; SaveToDatabase()   ; → can block; Windows kills the process if it takes too long
    return 0
}
OnExit(HandleExit)

; ---- 5. Prefer A_* variables over repeated DllCall for environment data ----
; ✓ A_ScriptDir, A_IsAdmin, A_IsCompiled are O(1) built-in reads
; ✗ Wrong: manually calling GetModuleFileName via DllCall to get script path
;     when A_ScriptFullPath already provides it — unnecessary syscall overhead

MsgBox("Config: theme = " Config.Get("UI", "Theme", "light"))
```

## DROP-IN RECIPES

```ahk
; ============================================================
; RECIPE 1 — SafeElevate()
; ✓ Complete admin-elevation pattern with compiled/interpreted branch,
;   argument forwarding, UAC-cancel error surface, and guaranteed exit.
;   Drop in near the top of AppMain(); never needs modification.
; ============================================================
SafeElevate(showErrorOnCancel := true) {
    if !(showErrorOnCancel is Integer)
        throw TypeError("SafeElevate: showErrorOnCancel must be Integer (0/1)", -1)
    if A_IsAdmin        ; ✓ Already elevated — nothing to do
        return

    try {
        ; ✓ Forward all original command-line arguments to the elevated instance
        argStr := ""
        for arg in A_Args
            argStr .= ' "' arg '"'

        if A_IsCompiled
            Run('*RunAs "' A_ScriptFullPath '"' argStr)
        else
            Run('*RunAs "' A_AhkPath '" "' A_ScriptFullPath '"' argStr)
    } catch Error as e {
        if showErrorOnCancel
            MsgBox("This script requires administrator rights.`n`nDetail: " e.Message,
                   "Elevation Required", 16)
    }
    ExitApp()   ; ✓ Always exit — prevents non-elevated instance from continuing
}
; Call site: SafeElevate()   ; near top of AppMain(), before any privileged operations

; ============================================================
; RECIPE 2 — BuildTrayMenu(items)
; ✓ Build a complete tray menu from a Map of label→callback pairs.
;   Accepts "" as a label to insert a separator. Validates all inputs.
;   Replaces the entire tray menu atomically — safe to call repeatedly.
; ============================================================
BuildTrayMenu(items, tooltip := "", iconPath := "") {
    if !(items is Map)
        throw TypeError("BuildTrayMenu: items must be a Map", -1)
    if Type(tooltip) != "String"
        throw TypeError("BuildTrayMenu: tooltip must be a String", -1)

    A_TrayMenu.Delete()     ; ✓ Remove all existing items before rebuilding

    for label, callback in items {
        if label = ""
            A_TrayMenu.Add()            ; ✓ empty label → separator line
        else if callback is Func || callback is BoundFunc || callback is Closure
            A_TrayMenu.Add(label, callback)
        else
            throw TypeError("BuildTrayMenu: callback for '" label "' must be a Func", -1)
    }

    if tooltip != ""
        A_IconTip := tooltip

    if iconPath != "" && FileExist(iconPath) != ""
        TraySetIcon(iconPath, 1)
}
; Call site:
; BuildTrayMenu(Map(
;     "Open",  (*) => ShowMainWindow(),
;     "",      "",
;     "Exit",  (*) => ExitApp()
; ), "MyApp v1.0", A_ScriptDir "\Assets\app.ico")

; ============================================================
; RECIPE 3 — CachedIni class
; ✓ Read-through cache for IniRead/IniWrite.
;   First Get() reads from disk; subsequent calls return cached value.
;   Set() writes to disk and updates cache atomically.
;   Flush() clears cache (e.g., after external INI modification).
; ============================================================
class CachedIni {
    static _cache := Map()
    static _file  := ""

    static Init(iniPath) {
        if !(iniPath is String) || iniPath = ""
            throw TypeError("CachedIni.Init: iniPath must be a non-empty string", -1)
        CachedIni._file := iniPath
    }

    ; ✓ Returns cached value on hit; reads from disk on miss; returns default if key absent
    static Get(section, key, default := "") {
        if CachedIni._file = ""
            throw Error("CachedIni.Get: call CachedIni.Init(path) before Get()", -1)
        k := section "§" key
        if !CachedIni._cache.Has(k)
            CachedIni._cache[k] := IniRead(CachedIni._file, section, key, default)
        return CachedIni._cache[k]
    }

    ; ✓ Writes to disk and updates cache — no stale reads after Set()
    static Set(section, key, value) {
        if CachedIni._file = ""
            throw Error("CachedIni.Set: call CachedIni.Init(path) before Set()", -1)
        IniWrite(value, CachedIni._file, section, key)
        CachedIni._cache[section "§" key] := value
    }

    ; ✓ Clear all cached entries — call after external writes to the INI file
    static Flush() {
        CachedIni._cache := Map()
    }
}
; Call site:
; CachedIni.Init(A_ScriptDir "\config.ini")
; theme := CachedIni.Get("UI", "Theme", "light")
; CachedIni.Set("UI", "Theme", "dark")
```

## ANTI-PATTERNS

| Pattern | Wrong | Correct | LLM Common Cause |
|---------|-------|---------|------------------|
| Omitting `Force` from `#SingleInstance` | `#SingleInstance` | `#SingleInstance Force` | In both AHK v1 and v2, the bare directive defaults to `Prompt` behavior; always specify `Force` explicitly to replace the prior instance without a dialog |
| Bare relative path in `#Include` | `#Include utils.ahk` | `#Include %A_ScriptDir%\Lib\utils.ahk` | Most language training data uses relative imports; AHK compilation changes the working directory, breaking all relative paths |
| `RunAs` without `A_IsAdmin` guard | `Run("*RunAs " A_ScriptFullPath) ; ExitApp()` | `if !A_IsAdmin { Run("*RunAs ...") ; ExitApp() }` | LLM generates elevation as a single statement, unaware that UAC denial causes the relaunched script to retry elevation infinitely |
| v1 Menu command for tray | `Menu, Tray, Add, Label, Handler` | `A_TrayMenu.Add("Label", Handler)` | AHK v1 training data pervasively uses the command form; v2 replaced it entirely with an OOP API |
| Unbound class method as callback | `OnExit(this.Cleanup)` or `A_TrayMenu.Add("X", this.Handler)` | `OnExit(this.Cleanup.Bind(this))` | LLM is trained on languages (Python, JS) where method references auto-bind; AHK v2 requires explicit `.Bind(this)` |
| Slow operations in `OnExit` | `SaveToServer()` or `HeavySerialise()` inside handler with no timeout path | Fast `FileAppend` for `"Logoff"`/`"Shutdown"`; heavy saves only for interactive exits | LLM unaware of the Windows ~5-second logoff/shutdown termination timeout; models assume cleanup time is unbounded |
| Placing `#Requires` after executable code | Any executable statement before `#Requires AutoHotkey v2.0` | `#Requires` as the absolute first non-comment line | LLM inserts version checks inline rather than at file top, not knowing the interpreter reads `#Requires` only when it appears first |

## SEE ALSO

> This module does NOT cover: GUI window management triggered by tray actions, or the script's hidden-window visible/shown state — see Module_GUI.md
> This module does NOT cover: try/catch error handling patterns for RunAs elevation failure and OnExit handler errors — see Module_Errors.md
> This module does NOT cover: COM object `.Release()` internals, `ObjRelease()` patterns, and DllCall-level process management inside OnExit — see Module_SystemAndCOM.md

- `Module_GUI.md` — tray items that open GUI windows, Gui() lifecycle, and managing the script's visible state via tray icon interactions.
- `Module_Errors.md` — try/catch patterns for RunAs elevation failure, startup mutex errors, and file-based configuration loading inside OnExit handlers.
- `Module_SystemAndCOM.md` — COM object lifecycle (`.Release()`, `ObjRelease()`), DllCall-level process and memory management, and advanced resource cleanup strategies for OnExit hooks that own system handles.