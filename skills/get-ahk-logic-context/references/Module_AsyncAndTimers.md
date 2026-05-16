# Module_AsyncAndTimers.md
<!-- DOMAIN: Async Operations and Timers -->
<!-- SCOPE: True multithreading, COM-based async, and inter-process communication are not covered — see Module_SystemAndCOM.md -->
<!-- TRIGGERS: SetTimer, Sleep, Critical, ObjBindMethod, "timer", "delay", "interval", "background task", "debounce", "throttle", "polling", "async", "run every N seconds", "one-off delay", "re-entrancy", "race condition", "shared state corruption", "class timer fails" -->
<!-- CONSTRAINTS: Class methods passed to SetTimer must be stored as bound references via .Bind(this) — a fresh .Bind() at cancel time creates a different object AHK cannot match, leaving the timer alive. Sleep -1 is NOT a delay; it immediately flushes the message queue. Timer callback exceptions never propagate to the caller and must be caught internally. Critical queues re-fired threads; only an IsRunning boolean guard prevents true callback re-entry. -->
<!-- CROSS-REF: Module_Classes.md, Module_GUI.md, Module_Errors.md, Module_SystemAndCOM.md -->
<!-- VERSION: AHK v2.0+ -->

## V1 → V2 BREAKING CHANGES

| v1 pattern (LLM commonly writes) | v2 correct form | Consequence |
|----------------------------------|-----------------|-------------|
| `SetTimer(this.Method, 1000)` unbound | `SetTimer(this.Method.Bind(this), 1000)` | TypeError at callback time — `this` is unset inside the unbound call |
| `Sleep(5000)` in a GUI or hotkey handler | `SetTimer(Callback, -5000)` | Freezes GUI, blocks all hotkeys and timer events for the full duration |
| `() => { return x }` multi-line fat arrow | `() { return x }` multi-line function body | Syntax error — v2 fat arrows must be single-expression with no braces |
| `Sleep -1` used as a minimal delay | Restructure with `SetTimer` or `Sleep(10)` | Sleep -1 flushes the message queue immediately — it is not a 1 ms pause |
| No `__Delete` cleanup for class timers | Call `SetTimer(this.BoundRef, 0)` in `__Delete` | Timer holds a reference to the object, preventing garbage collection — permanent leak |
| `SetTimer(Func, "1000")` string period | `SetTimer(Func, 1000)` integer period | TypeError — Period must be an Integer; string coercion does not apply |
| Loop closure `() => DoWork(i)` for timer | `DoWork.Bind(i)` at schedule time | Closure captures the variable reference — all callbacks fire with the final loop value |

## API QUICK-REFERENCE

### SetTimer

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `SetTimer()` | `SetTimer(Function, Period, Priority)` | — | TypeError if Period is non-integer or Function is empty string | Positive period = repeating; negative = one-off auto-delete; 0 = cancel. Default 250 ms if new. |
| `SetTimer(, 0)` | `SetTimer(, 0)` inside callback | — | — | Omit Function to self-cancel the currently running timer; invalid if called from outside a timer context |
| `Priority` | Integer parameter of SetTimer | — | — | Controls interrupt precedence; range -2147483648 to 2147483647; default 0 |

### Sleep

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `Sleep()` | `Sleep(N)` | — | — | Blocks current thread for N ms. **-1 = flush message queue immediately, NOT a delay.** |

### Critical

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `Critical` | `Critical` | Integer (prev A_IsCritical) | — | Makes current thread non-interruptible; queued events fire after release — nothing is discarded |
| `Critical "Off"` | `Critical "Off"` | Integer (prev A_IsCritical) | — | Release critical section; interruption occurs ~5 ms after call, not immediately |
| `Critical N` | `Critical N` (positive integer) | Integer (prev A_IsCritical) | — | Enable Critical with explicit message-check interval N ms; default is 16 ms when on |
| `A_IsCritical` | Built-in variable (read) | Integer | — | 0 if not critical; otherwise the message-check interval. Save before entering Critical in nestable functions |

### Binding Utilities

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `.Bind()` | `Func.Bind(arg1, arg2, ...)` | BoundFunc object | — | Locks args at bind time; each call produces a **distinct** object — store the reference, do not call again at cancel time |
| `ObjBindMethod()` | `ObjBindMethod(Obj, MethodName, args*)` | BoundFunc object | — | Binds by method name string; useful for dynamic dispatch; each call also produces a distinct object |

### Time and Measurement

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `FormatTime()` | `FormatTime(Timestamp?, Format?)` | String | ValueError on invalid timestamp | Human-readable time output; use instead of `A_Now` (which returns raw YYYYMMDDHH24MISS) |
| `A_TickCount` | Built-in variable (read) | Integer (ms) | — | Milliseconds since system start; use for elapsed-time measurements; ~10 ms granularity |

### Supporting Built-ins Used in Examples

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `ToolTip()` | `ToolTip(Text?, X?, Y?, Which?)` | — | — | See Module_GUI.md |
| `FileExist()` | `FileExist(Path)` | Attribute string or "" | — | See Module_FileSystem.md |
| `FileRead()` | `FileRead(Filename, Options?)` | String | OSError | See Module_FileSystem.md |
| `Map.Has()` | `.Has(Key)` | Integer (1 or 0) | — | See Module_Objects.md |
| `Array.Push()` | `.Push(Value*)` | — | — | See Module_Arrays.md |
| `Array.Length` | `.Length` | Integer | — | See Module_Arrays.md |
| `Min()` | `Min(Value*)` | Number | — | Built-in math — no module required |

## AHK V2 CONSTRAINTS

- Class methods passed to `SetTimer` must be stored as a bound reference in a class property — `this.BoundRef := this.Method.Bind(this)` — and that same stored reference used for both `SetTimer(this.BoundRef, interval)` and `SetTimer(this.BoundRef, 0)`; a fresh `.Bind(this)` call at cancel time creates a different object that AHK cannot match, leaving the timer permanently active.
- Always cancel class timers in `__Delete` — `__Delete` timing is non-deterministic in v2, and a live timer holds a strong reference to the object, preventing garbage collection indefinitely.
- `Sleep -1` is NOT a delay — it immediately processes the script's message queue; using it as a minimal pause introduces unexpected callback execution mid-function at an uncontrolled point.
- Timer callback exceptions do not propagate to any caller — every callback that can throw must contain its own `try/catch` block; uncaught exceptions inside a timer are silently discarded.
- `Critical` buffers interrupting threads rather than discarding them — every queued hotkey and timer event fires in sequence after `Critical "Off"` is called; it does NOT prevent a timer from re-firing on schedule.
- `Critical` does NOT prevent callback re-entry — a second firing of the same timer during a Critical block will execute immediately after `Critical "Off"`; an `IsRunning` boolean guard released in `finally` is the correct re-entry prevention mechanism.
- Bracket `Critical` tightly around only the atomic read-modify-write statements; holding `Critical` over `Sleep`, `FileRead`, or any long I/O starves all other threads and makes the script unresponsive.
- Save `A_IsCritical` before entering `Critical` in any function that may be called from an already-critical context; restore it on exit to avoid inadvertently releasing an outer Critical block.
- Negative-period timers (`SetTimer(cb, -ms)`) fire exactly once and self-delete — do not call `SetTimer(ref, 0)` to cancel a one-shot timer that has already fired; the timer entry no longer exists.
- `SetTimer` period is rounded up to the nearest OS clock tick (typically 10–15.6 ms); requesting sub-10 ms periods does not achieve sub-10 ms cadence.

Safe-access priority order for timer dispatch:
  1. `SetTimer(cb, -ms)` — one-off delayed execution; fires once, self-deletes, no cleanup required
  2. `SetTimer(cb, ms)` with `IsRunning` guard — repeating execution; guard must be released in `finally`
  3. `Critical` around shared-state atomic blocks — only when two pseudo-threads mutate the same variables
  4. `try/catch` inside the callback — only when exception details carry diagnostic value

Pairing rules:
- ✗ `SetTimer(this.OnTick, 1000)` — TypeError: `this` is unset at callback time
- ✓ `SetTimer(this.OnTick.Bind(this), 1000)` — context preserved via stored BoundFunc
- ✗ `Sleep(5000)` in GUI callback — blocks GUI thread and all hotkeys for 5 s
- ✓ `SetTimer(Callback, -5000)` — deferred execution without blocking
- ✗ Rely on `Critical` alone to prevent re-entry — only delays the second firing until after `Critical "Off"`
- ✓ `IsRunning` boolean guard released in `finally` — prevents true callback re-entry

Unset variable handling: always verify `IsSet(this.BoundRef)` or that the bound reference property is initialized before calling `SetTimer`; calling `SetTimer` with an uninitialized variable raises TypeError.

Resource lifecycle: every class-based timer that holds external resources (file handles, COM objects) must call `Stop()` in `__Delete`, and `Stop()` must guard against double-cancellation with an `IsRunning` check.

## AGENT QA CHECKLIST

- [ ] Did I store the BoundFunc in a class property and use **that same stored reference** for both the start call and the cancel call (`SetTimer(this.BoundRef, 0)`)?
- [ ] Does every timer callback contain its own `try/catch` block — timer exceptions do not propagate and are silently discarded without one?
- [ ] Did I release the `IsRunning` re-entrancy guard in a `finally` block, not only at the normal end of the callback?
- [ ] Did I use a negative period (`SetTimer(cb, -ms)`) for one-shot delayed work instead of `Sleep`?

## RUNTIME ERROR MAPPING

| Exception Class | Trigger Condition | Detection Code | Fix |
|----------------|-------------------|----------------|-----|
| `TypeError` | Calling `SetTimer(this.Method, 1000)` without `.Bind(this)` — `this` is unset at callback time | `e.Message` contains "unset" or "this" | Store bound method: `this.BoundTick := this.Method.Bind(this)` then pass `this.BoundTick` |
| Silent discard (no exception surfaced) | Uncaught exception thrown inside a timer callback | No visible error — callback stops silently; add `OutputDebug(e.Message)` in catch | Wrap entire callback body in `try { ... } catch as e { OutputDebug("Timer error: " e.Message) }` |
| Silent leak (no exception) | `SetTimer(this.Method.Bind(this), 0)` at cancel time — fresh `.Bind()` creates a new object AHK cannot match | Timer continues firing; object is never garbage-collected; `KeyHistory` shows timer count not decreasing | Store the BoundFunc once in `__New`: `this.BoundRef := this.Method.Bind(this)` and cancel with `SetTimer(this.BoundRef, 0)` |

## TIER 1 — Basic Timer Creation and One-Off Delays
> METHODS COVERED: SetTimer · FormatTime

AHK v2 timers run as cooperative pseudo-threads: the OS fires the callback at the specified interval, AHK suspends the current execution context, runs the callback to completion, and then resumes. Positive period repeats indefinitely; negative period fires exactly once and auto-deletes; period 0 cancels. When `Function` is omitted inside a callback, `SetTimer(, 0)` self-cancels the currently running timer. Default period is 250 ms when omitted and the timer is new.

```ahk
; ✓ FormatTime produces human-readable time — A_Now returns raw YYYYMMDDHH24MISS, not display text
ShowTime() {
    ToolTip("Current time: " FormatTime(, "HH:mm:ss"))
}
SetTimer(ShowTime, 1000)

; ✓ Negative period fires exactly once after 3 s then auto-deletes — no manual cancel needed
HideToolTip() {
    ToolTip()
}
SetTimer(HideToolTip, -3000)

; ✓ SetTimer(, 0) with Function omitted self-cancels the current timer from inside its own callback
CountdownTick() {
    static count := 5
    ToolTip(count--)
    if (count < 0) {
        SetTimer(, 0)   ; Omitting Function cancels the current timer
        ToolTip()
    }
}
SetTimer(CountdownTick, 1000)

; ✗ Sleep blocks the entire thread — GUI freezes and hotkeys stop responding during the wait
; Sleep(3000)   ; → GUI unresponsive for 3 s; all queued events delayed until Sleep returns

; ✗ Sleep -1 is not a minimal delay — it flushes the message queue immediately
; Sleep(-1)     ; → NOT a 1 ms pause; triggers all pending callbacks at this instant
```

## TIER 2 — Timer Lifecycle and Class Integration
> METHODS COVERED: SetTimer · .Bind · ObjBindMethod

Class-based timers require three invariants: (1) store the bound method in a property so the same object reference is passed to both the start and cancel calls; (2) use a liveness boolean to prevent redundant Start/Stop calls; (3) always cancel in `__Delete`. A fresh `.Bind(this)` at cancel time creates a distinct BoundFunc object that AHK cannot match against the registered timer — the timer survives and holds the object alive indefinitely.

```ahk
; ✓ Canonical lifecycle class: stored bound ref, liveness guard, __Delete teardown
class AsyncWorker {
    __New() {
        this.IsRunning := false
        this.TickCount := 0
        ; ✓ Store the bound method — same reference used for both start and stop
        this.BoundTick := this.OnTick.Bind(this)
    }

    Start(intervalMs := 1000) {
        if (this.IsRunning)
            return
        this.IsRunning := true
        SetTimer(this.BoundTick, intervalMs)
    }

    Stop() {
        if (!this.IsRunning)
            return
        this.IsRunning := false
        SetTimer(this.BoundTick, 0)   ; ✓ Same stored reference — AHK matches and cancels it
    }

    OnTick() {
        this.TickCount++
        ; Background work here
    }

    __Delete() {
        this.Stop()   ; ✓ Ensures the timer cannot outlive the object
    }
}

; ✗ Fresh .Bind() at cancel time — AHK sees a different object; timer is never cancelled
; SetTimer(this.OnTick.Bind(this), 0)   ; → timer persists; memory leak

; ✓ Dynamic period mutation: calling SetTimer with the same callback resets and updates its period
class PollingService {
    __New() {
        this.Poller := this.CheckStatus.Bind(this)
        this.CurrentInterval := 1000
        SetTimer(this.Poller, this.CurrentInterval)
    }

    ; Accelerate the timer
    SpeedUp() {
        this.CurrentInterval := 200
        ; ✓ Calling SetTimer again with the same callback resets and updates the period
        SetTimer(this.Poller, this.CurrentInterval)
    }

    ; Pause the timer completely
    Pause() {
        SetTimer(this.Poller, 0)   ; ✓ Period 0 cancels without discarding the stored reference
    }

    CheckStatus() {
        ; Polling logic
    }
}
```

## TIER 3 — State Tracking, Re-Entrancy Guards, and Critical Sections
> METHODS COVERED: Critical · A_IsCritical · SetTimer · Map.Has

AHK v2's single-threaded interruption model means two pseudo-threads can interleave reads and writes on the same global variables or object properties. `Critical` atomizes multi-statement operations by preventing other threads from interrupting during the block — but it does NOT prevent a timer from re-firing on its own schedule. A boolean `IsRunning` guard is the correct mechanism to prevent a slow callback from being interrupted and re-started by its own next scheduled firing. AHK v2 provides no built-in API to query a running timer's period — timer state must be tracked explicitly in a `Map`.

```ahk
; ===========================================================================
; Pattern 1 — Critical for atomic shared-state mutation
; ===========================================================================

global g_Counter := 0

; ✗ Race condition: Thread A reads, gets interrupted, Thread B increments, Thread A overwrites
IncrementAndLog() {
    local snapshot := g_Counter         ; Thread A reads g_Counter = 5
    ; <<< Timer fires here, Thread B runs, increments g_Counter to 6 >>>
    ; Thread A resumes — snapshot is still 5, but g_Counter is 6
    ; Thread A writes snapshot + 1 = 6, silently discarding Thread B's increment
    g_Counter := snapshot + 1
    FileAppend(g_Counter "`n", "log.txt")
}

; ✓ Critical makes the read-modify-write atomic — no thread can interrupt between the two lines
IncrementAndLogSafe() {
    Critical                      ; ✓ Block interruptions for this thread
    local snapshot := g_Counter
    g_Counter := snapshot + 1
    Critical "Off"                ; ✓ Release as soon as the atomic block ends
    FileAppend(g_Counter "`n", "log.txt")   ; File I/O outside Critical — safe to interrupt here
}

; ===========================================================================
; Pattern 2 — IsRunning re-entrancy guard for slow callbacks
; ===========================================================================

; ✓ IsRunning prevents a second instance from running before the first finishes
class FileWatcher {
    __New(path) {
        this.Path      := path
        this.IsRunning := false          ; ✓ Re-entrancy guard flag
        this.Worker    := this.Poll.Bind(this)
        SetTimer(this.Worker, 500)
    }

    Poll() {
        ; ✓ Re-entrancy guard: abort if previous execution has not finished
        if (this.IsRunning)
            return
        this.IsRunning := true
        try {
            ; Simulate work that may take longer than 500 ms
            if FileExist(this.Path) {
                content := FileRead(this.Path)
                ; ... process content ...
            }
        } finally {
            ; ✓ Always release the guard, even if an exception occurs
            this.IsRunning := false
        }
    }

    __Delete() {
        SetTimer(this.Worker, 0)
    }
}

; ✗ No guard — if Poll() takes > 500 ms, a second thread starts concurrently
; → unpredictable interleaved mutations of this.Path, content, and shared state

; ===========================================================================
; Pattern 3 — Map-based timer registry for multi-timer state tracking
; ===========================================================================

; ✓ TimerManager: explicit Map tracks what AHK v2 provides no built-in API to query
class TimerManager {
    __New() {
        this.ActiveTimers := Map()
    }

    RegisterTask(taskName, callback, interval) {
        if (this.ActiveTimers.Has(taskName))
            throw Error("Task already exists: " taskName)

        this.ActiveTimers[taskName] := Map(
            "Callback",  callback,
            "Interval",  interval,
            "IsRunning", true
        )
        SetTimer(callback, interval)
    }

    IsTaskRunning(taskName) {
        if (this.ActiveTimers.Has(taskName))
            return this.ActiveTimers[taskName]["IsRunning"]
        return false
    }

    StopTask(taskName) {
        if (this.ActiveTimers.Has(taskName) && this.ActiveTimers[taskName]["IsRunning"]) {
            SetTimer(this.ActiveTimers[taskName]["Callback"], 0)
            this.ActiveTimers[taskName]["IsRunning"] := false
        }
    }
}
```

## TIER 4 — Higher-Order Timing: Debounce and Throttle
> METHODS COVERED: SetTimer · .Bind

Debouncing wraps a callback so it executes only after a quiet period with no new calls — each new call resets the pending one-shot timer. The AHK v2 implementation uses nested function closure semantics: `CreateDebounce` returns a `Debouncer` closure that correctly captures its own `timeoutCallback` and `waitMs` per instance. Resetting is achieved by calling `SetTimer` with the same callback and a new negative period; AHK replaces the pending one-shot entry without creating a duplicate.

```ahk
; ✓ CreateDebounce: each call to the returned Debouncer resets the one-shot timer
; Each call to CreateDebounce returns a new closure instance, correctly capturing
; its own timeoutCallback and waitMs via AHK v2 nested function closure semantics.
CreateDebounce(callback, waitMs) {
    timeoutCallback := () => callback()   ; ✓ Single-expression fat arrow — no braces

    Debouncer() {
        ; ✓ Negative period fires once after quiet period; repeated calls keep resetting it
        SetTimer(timeoutCallback, -waitMs)
    }

    return Debouncer
}

; Usage: DebouncedSave fires once, 1 second after the last call in a rapid series
OnTypingComplete() {
    ; Save draft logic
}

DebouncedSave := CreateDebounce(OnTypingComplete, 1000)

; ✓ Calling this rapidly results in one execution, 1 second after the final call
DebouncedSave()
DebouncedSave()
DebouncedSave()

; ✗ Multi-line fat-arrow with braces — syntax error in AHK v2
; timeoutCallback := () => {   ; → SyntaxError: fat arrows must be single-expression
;     return callback()
; }

; ✓ Correct multi-line form: drop the arrow, use a block function body
; timeoutCallback() {
;     return callback()
; }
```

## TIER 5 — Parameterized Callbacks and Advanced Binding
> METHODS COVERED: SetTimer · .Bind · Critical · A_IsCritical

Passing arguments to a `SetTimer` callback requires `.Bind()` — closures in loops capture the variable reference, not its value at bind time, so `.Bind(arg)` is the only safe form for loop-scheduled callbacks. When a callback must mutate shared state that another timer also touches, wrap the atomic statements in `Critical` / `Critical "Off"` and save `A_IsCritical` before entry to correctly support nested invocations without inadvertently releasing an outer Critical block.

```ahk
; ✓ NotificationSystem: .Bind(this, message) locks both context and argument at scheduling time
class NotificationSystem {
    __New() {
        this.Queue := []
    }

    ; Schedule a notification with specific data
    ScheduleAlert(message, delayMs) {
        ; ✓ Bind 'this' context AND the 'message' argument to the callback
        boundAlert := this.ShowAlert.Bind(this, message)

        ; ✓ Execute once after delay — argument value locked at bind time, not fire time
        SetTimer(boundAlert, -delayMs)
    }

    ShowAlert(message) {
        ; ✓ 'this' is preserved, 'message' is injected via Bind
        this.Queue.Push(message)
        ; Display logic
    }
}

; ✗ Arrow closure in a loop captures the loop variable reference, not its value at schedule time
; for i, msg in messages
;     SetTimer(() => ShowAlert(msg), -1000 * i)   ; → all callbacks fire with the last value of msg

; ✓ .Bind() locks the argument value at scheduling time — each callback fires with its own msg
; for i, msg in messages
;     SetTimer(ShowAlert.Bind(msg), -(1000 * i))

; -----------------------------------------------------------------------
; Advanced Critical: save and restore A_IsCritical for nestable functions
; -----------------------------------------------------------------------

; ✓ Pattern 1: Save and restore Critical state — safe when called from within an existing Critical block
ProtectedUpdate(sharedMap, key, value) {
    prevCritical := A_IsCritical    ; ✓ Save current Critical state before entry
    Critical                        ; Begin atomic block

    sharedMap[key] := value         ; Multi-step mutation that must be atomic
    sharedMap["LastModified"] := A_TickCount

    Critical prevCritical           ; ✓ Restore previous Critical state — not unconditional "Off"
}

; ✓ Pattern 2: Guard a GUI multi-control update so no timer sees a half-updated state
UpdateDashboard(ctrl1, ctrl2, newVal1, newVal2) {
    Critical
    ctrl1.Value := newVal1
    ctrl2.Value := newVal2
    Critical "Off"
}

; ✗ Wrapping entire functions including I/O and Sleep inside Critical
; BadPattern() {
;     Critical
;     Sleep(500)           ; → blocks message queue 500 ms; all other threads starved
;     FileRead("huge.txt") ; → long I/O holds Critical far too long
;     Critical "Off"
; }
```

## TIER 6 — Non-Blocking State Machines and Chunked Processing
> METHODS COVERED: SetTimer · .Bind · Min

Sequential multi-phase tasks that would normally require `Sleep` between phases can be expressed as a state machine: each phase sets the next state integer and schedules the same handler via a one-shot negative-period timer. The script remains fully responsive between phases because no thread is ever blocked. Chunked processing applies the same pattern to large datasets — processing a fixed-size batch per tick and self-cancelling when the dataset is exhausted.

```ahk
; ✓ AsyncSequence: state machine replaces Sleep-based sequential execution entirely
class AsyncSequence {
    __New() {
        this.State     := 0
        this.Processor := this.Step.Bind(this)
    }

    Start() {
        this.State := 1
        SetTimer(this.Processor, -10)   ; ✓ Start immediately; negative = one-shot
    }

    Step() {
        if (this.State == 1) {
            ; Do phase 1 work
            this.State := 2
            SetTimer(this.Processor, -500)    ; ✓ Schedule phase 2 in 500 ms — no blocking
        }
        else if (this.State == 2) {
            ; Do phase 2 work
            this.State := 3
            SetTimer(this.Processor, -1000)   ; ✓ Schedule phase 3 in 1 s — no blocking
        }
        else if (this.State == 3) {
            ; Finish sequence — no further timer scheduled; sequence complete
            this.State := 0
        }
    }
}

; ✗ Sleep-based sequential execution blocks the entire thread between phases
; Phase1()
; Sleep(500)    ; → GUI unresponsive for 500 ms; hotkeys silenced
; Phase2()
; Sleep(1000)   ; → GUI unresponsive for another 1 s

; ✓ HeavyProcessor: chunked processing yields between batches — script remains responsive
class HeavyProcessor {
    __New(largeDataSet) {
        this.Data      := largeDataSet
        this.Index     := 1
        this.ChunkSize := 100
        this.Worker    := this.ProcessChunk.Bind(this)
    }

    Start() {
        SetTimer(this.Worker, 10)   ; ✓ Fast cadence; each tick processes one chunk only
    }

    ProcessChunk() {
        endIndex := Min(this.Index + this.ChunkSize - 1, this.Data.Length)

        Loop (endIndex - this.Index + 1) {
            ; Process this.Data[this.Index]
            this.Index++
        }

        if (this.Index > this.Data.Length) {
            SetTimer(this.Worker, 0)   ; ✓ All chunks processed — self-cancel
        }
    }
}
```

### Performance Notes

`SetTimer` period is rounded up to the nearest OS clock resolution — typically 10 ms or 15.6 ms depending on hardware and power settings. Requesting a 1 ms timer does not achieve 1 ms cadence; periods below the resolution floor behave identically to that floor. Avoid designing timing logic that depends on sub-10 ms precision.

Avoid tight loops with `Sleep` for batch processing. Instead, slice the dataset using a fast timer (10–15 ms cadence) and process a fixed chunk per tick. This yields control to the OS and other threads between chunks, keeping the script responsive. Calibrate chunk size so each tick completes well under the timer period — profiling the slowest chunk is required, not optional.

For timer state lookups in a multi-timer registry, prefer `Map.Has(key)` O(1) lookup over iterating an Array to locate a callback reference. Storing callbacks in a `Map` keyed by name or ID eliminates linear scans as the registry grows.

`Critical` carries measurable overhead: every thread that tries to interrupt a Critical block is queued and must be dispatched in sequence after `Critical "Off"`. Holding `Critical` over any I/O, `Sleep`, or long computation multiplies this cost across all waiting threads. Profile before widening a Critical section — the atomic block should contain only the minimum necessary statements.

## DROP-IN RECIPES

```ahk
; SafeTimerCallback — wraps any function to isolate timer exceptions and route them to a handler
; ✓ Prevents silent discard — without this wrapper, all exceptions inside SetTimer vanish
; ✓ Returns a BoundFunc-compatible closure ready for direct use with SetTimer
SafeTimerCallback(fn, onError := "") {
    if !(fn is Func)
        throw TypeError("SafeTimerCallback: fn must be a Func or BoundFunc", -1)

    SafeWrapper() {
        try {
            fn()
        } catch as e {
            if (onError is Func)
                onError(e)
            else
                OutputDebug("Timer exception [" fn.Name "]: " e.Message " @ " e.File ":" e.Line)
        }
    }

    return SafeWrapper
}
; Call site (repeating timer with error isolation):
;   SetTimer(SafeTimerCallback(MyCallback), 1000)
; Call site (one-shot with custom error handler):
;   SetTimer(SafeTimerCallback(MyCallback, (e) => MsgBox("Error: " e.Message)), -5000)


; CreateThrottle — rate-limits a callback so it fires at most once per minIntervalMs
; ✓ Unlike debounce, throttle guarantees the first call fires immediately, then enforces the gap
; ✓ Uses A_TickCount for elapsed measurement — no additional timer registration required
CreateThrottle(callback, minIntervalMs) {
    if !(callback is Func)
        throw TypeError("CreateThrottle: callback must be a Func or BoundFunc", -1)
    if !(minIntervalMs is Integer) || minIntervalMs < 1
        throw ValueError("CreateThrottle: minIntervalMs must be a positive Integer", -1)

    lastFired := 0   ; ✓ Captured in closure — each CreateThrottle call has its own independent state

    Throttled() {
        now := A_TickCount
        ; ✓ A_TickCount wraps at ~49.7 days; subtraction handles wrap-around correctly
        if (now - lastFired >= minIntervalMs) {
            lastFired := now
            callback()
        }
    }

    return Throttled
}
; Call site (fire at most once per 500 ms regardless of how often the timer fires):
;   ThrottledUpdate := CreateThrottle(UpdateDisplay, 500)
;   SetTimer(ThrottledUpdate, 16)   ; ~60 fps polling cadence, but callback capped at 2 Hz
```

## ANTI-PATTERNS

| Pattern | Wrong | Correct | LLM Common Cause |
|---------|-------|---------|------------------|
| Unbound class method in SetTimer | `SetTimer(this.OnTick, 1000)` | `SetTimer(this.OnTick.Bind(this), 1000)` | AHK v1 did not require explicit binding; JS/Python OOP binding semantics differ |
| Blocking delay in GUI or hotkey thread | `Sleep(5000)` | `SetTimer(Callback, -5000)` | AHK v1 used `Sleep` loops as the primary async pattern; event-loop alternative not in training data |
| String period type | `SetTimer(Func, "1000")` | `SetTimer(Func, 1000)` | JS implicit type coercion training data; string/integer conflation |
| Multi-line fat-arrow with braces | `() => { return x }` | `() { return x }` or single-expression `() => x` | JS/Python arrow function training data where braces are permitted |
| Sleep -1 as a minimal pause | `Sleep(-1)` expecting ~1 ms delay | Restructure with `SetTimer` or `Sleep(10)` | Undocumented sentinel value; AHK v1 training data did not cover this special case |
| Missing re-entrancy guard | No `IsRunning` check in a slow callback | `if (this.IsRunning) return` at entry; `finally { this.IsRunning := false }` | AHK thread-interruption model is AHK-specific; LLMs assume async requires explicit concurrency primitives |
| Over-holding Critical | `Critical` wrapping `Sleep` or long file I/O | Bracket only the atomic read-modify-write statements; release immediately after | Analogized to mutex lock-holding from C/Java training data; LLMs do not know AHK threads queue, not block |
| Critical as re-entry prevention | Relying solely on `Critical` to stop a callback running twice | `IsRunning` boolean guard; `Critical` only delays the second firing until after `Critical "Off"` | Confused with mutex semantics; LLMs expect Critical to block, not queue, competing threads |
| Fresh `.Bind()` at cancel time | `SetTimer(this.OnTick.Bind(this), 0)` | Store `this.BoundTick := this.OnTick.Bind(this)` and cancel with the stored reference | Lack of AHK object-identity knowledge; each `.Bind()` produces a distinct BoundFunc object |

## SEE ALSO

> This module does NOT cover: true multithreading, COM-based async, WScript.Shell background processes, or inter-process communication — see Module_SystemAndCOM.md
> This module does NOT cover: class meta-function lifecycle (`__New`, `__Delete`), inheritance, or prototype chains that underpin class-based timer patterns — see Module_Classes.md
> This module does NOT cover: GUI event handler threading, control update safety from timer callbacks, or offloading work from GUI events — see Module_GUI.md
> This module does NOT cover: structured exception handling, custom `Error` subclasses, or exception propagation for internal callback error recovery — see Module_Errors.md

- `Module_Classes.md` — `__New`/`__Delete` lifecycle, OOP binding semantics, and prototype chain rules that underpin every class-based timer pattern in this module.
- `Module_GUI.md` — GUI event handler threading model, control-update safety from background timer callbacks, and offloading heavy processing from GUI events to timer-based state machines.
- `Module_Errors.md` — `try/catch/finally` patterns for exception handling inside timer callbacks, where errors do not propagate to the caller and must be caught internally.
- `Module_SystemAndCOM.md` — COM-based async, WScript.Shell background execution, and true parallelism via external processes for workloads that exceed what single-threaded time-slicing can handle.