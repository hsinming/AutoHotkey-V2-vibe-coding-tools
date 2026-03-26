# Module_NetworkAndHTTP.md
<!-- DOMAIN: Network Requests and HTTP — WinHttp.WinHttpRequest.5.1 COM, file download, JSON handling, REST API clients, Promise-style async -->
<!-- SCOPE: COM automation beyond WinHttp.WinHttpRequest.5.1 (IStream direct manipulation, WebSocket COM, WinInet), OAuth multi-step flows, TLS certificate management via registry, and raw socket I/O are not covered — see Module_SystemAndCOM.md -->
<!-- TRIGGERS: WinHttp.WinHttpRequest, WinHttpRequest, ComObject HTTP, Download(), HttpGet, HttpPost, REST API, JSON parse, JSON stringify, "HTTP request", "fetch URL", "web request", "API call", "async HTTP", "Bearer token", "API client", responseText, setRequestHeader, WaitForResponse, Jxon, "network request", "rate limit", "retry request", ResponseBody, HttpPromise, "Promise async", "concurrent requests", "non-blocking HTTP" -->
<!-- CONSTRAINTS: Use ComObject("WinHttp.WinHttpRequest.5.1") — ComObjCreate() removed in AHK v2. SetTimeouts ResolveTimeout MUST be 0 (non-zero leaks DNS thread handles per Microsoft KB). AHK v2 has NO native JSON — never generate JSON.Parse()/JSON.Stringify(). WinHttp has no readyState — use WaitForResponse() (no-arg, pumps message loop) for blocking-but-responsive, or WaitForResponse(0) in a SetTimer for fully non-blocking. ComObjConnect is incompatible with WinHttp (IUnknown-based events). Storing a BoundFunc that captures `this` as a property of `this` creates a reference cycle — always nullify with `:= unset` after stopping the timer. -->
<!-- CROSS-REF: Module_Errors.md, Module_Classes.md, Module_SystemAndCOM.md -->
<!-- VERSION: AHK v2.0+ -->

## V1 → V2 BREAKING CHANGES

| v1 pattern (LLM commonly writes) | v2 correct form | Consequence |
|----------------------------------|-----------------|-------------|
| `ComObjCreate("WinHttp.WinHttpRequest.5.1")` | `ComObject("WinHttp.WinHttpRequest.5.1")` | MethodError at runtime — `ComObjCreate()` was removed in AHK v2; no HTTP session is created |
| `ComObject("Msxml2.ServerXMLHTTP.6.0")` | `ComObject("WinHttp.WinHttpRequest.5.1")` | Works but uses the MSXML6-stack COM server — WinHttp.WinHttpRequest.5.1 is a native Windows OS component since XP SP1, has `ResponseBody` for binary, `WaitForResponse()` for async, and explicit `SetProxy()`/`SetCredentials()` methods |
| `JSON.Parse(xhr.responseText)` | `Jxon.Load(xhr.responseText)` after `#Include Jxon.ahk` | MethodError — no native JSON parser exists in AHK v2 standard library |
| `JSON.Stringify(obj)` or `JSON.Dump(obj)` | `Jxon.Dump(obj)` or custom `MapToJsonFlat()` | MethodError — same root cause; agent must use community library or manual serializer |
| `data["items"][0]` on Jxon-returned Array | `data["items"][1]` | Runtime `IndexError` — AHK v2 Arrays are 1-based; index 0 throws `IndexError` |
| `xhr.setTimeouts(5000, ...)` — non-zero ResolveTimeout | `xhr.setTimeouts(0, 10000, 30000, 30000)` — zero ResolveTimeout | DNS thread handle leak: every DNS resolution leaks a thread handle; over many requests causes measurable memory growth. Leave ResolveTimeout at 0 — Windows DNS client has its own timeout |
| `xhr.open(...)` then `xhr.send()` with no `xhr.setTimeouts()` | `xhr.setTimeouts(0, 10000, 30000, 30000)` called before `open()` | WinHttp default ReceiveTimeout is 30 seconds — shorter than Msxml2's infinite, but connect/send timeouts vary by build; set explicitly for deterministic behaviour |
| Reading `xhr.responseText` immediately after `send()` without status check | `if (xhr.status == 200) return xhr.responseText` | Silent wrong data or empty string — WinHttp does not throw on HTTP 4xx/5xx; caller must inspect `.status` |
| `ComObjConnect(xhr, sink)` for async events | `SetTimer(pollFn, 50)` + poll `xhr.WaitForResponse(0)` | WinHttp uses `IWinHttpRequestEvent` which derives from `IUnknown` not `IDispatch` — `ComObjConnect` returns `0x80004002 No such interface supported` |
| `xhr.readyState == 4` for async completion check | `xhr.WaitForResponse(0)` returning true | WinHttp has no `readyState` property — accessing it returns empty; use `WaitForResponse(0)` for non-blocking poll |
| `open(false)` + `send()` in GUI event handler | `open(true)` + `WaitForResponse()` | `open(false)` blocks the entire AHK message loop — GUI freezes and hotkeys stop responding; `open(true)` + `WaitForResponse()` pumps the message loop while waiting |
| Storing BoundFunc in `this` that captures `this`: `this.fn := this.Method.Bind(this)` | After `SetTimer(this.fn, 0)`, immediately do `this.fn := unset` | Reference cycle via AHK v2 reference counting — `this → fn → this` prevents garbage collection; BoundFunc and COM object are never freed |
| Multi-statement arrow function as async callback: `(s, b) => { MsgBox(s) ... }` | Define a named function: `HandleResponse(s, b) { MsgBox(s) }` | SyntaxError — AHK v2 arrow functions are single-expression only; callbacks with multiple statements require named functions |

## API QUICK-REFERENCE

### Download (built-in)
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `Download()` | `Download(url, destPath)` | Fully synchronous — blocks AHK thread for entire duration; throws `OSError` on failure (network-level error or disk write error); HTTP 4xx/5xx responses are NOT exceptions — the response body is silently written to the destination file |

### WinHttp.WinHttpRequest.5.1 COM Object — Core Request Methods
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `ComObject()` | `ComObject("WinHttp.WinHttpRequest.5.1")` | Creates HTTP session — native Windows OS component, preferred over `Msxml2.ServerXMLHTTP.6.0` |
| `.setTimeouts()` | `.setTimeouts(resolve, connect, send, receive)` | All values in milliseconds; **resolve must be 0** (non-zero causes DNS thread handle leaks per Microsoft KB); must be called before `open()` |
| `.open()` | `.open(method, url, async)` | `async=false` = blocks AHK message loop (GUI freezes); `async=true` = HTTP I/O on background thread, use `WaitForResponse()` or `WaitForResponse(0)` for completion |
| `.setRequestHeader()` | `.setRequestHeader(name, value)` | Must be called after `open()` and before `send()` |
| `.send()` | `.send(body?)` | Accepts string or empty for GET/HEAD; returns immediately when `async=true` |
| `.abort()` | `.abort()` | Cancels an in-flight async request |
| `.waitForResponse()` | `.waitForResponse()` | **No-arg form**: blocks current AHK code flow but pumps the message loop — other timers and hotkeys remain active. Per official AHK v2 docs: canonical "script remains responsive" pattern. Throws on timeout/network error. |
| `.waitForResponse(0)` | `.waitForResponse(timeoutSecs)` | **Zero-timeout form**: non-blocking poll — returns `true` immediately if response is ready, `false` if still in-flight; use inside `SetTimer` callback for fully non-blocking async |
| `.waitForResponse(n)` | `.waitForResponse(timeoutSecs)` | **Timed form**: blocks up to `n` seconds then throws if no response; unlike no-arg form, has an explicit deadline |
| `.status` | `.status` (read-only) | HTTP status integer (200, 404, etc.); never auto-throws on 4xx/5xx |
| `.responseText` | `.responseText` (read-only) | Response body as string; check `.status` first — returns empty string or error body on HTTP error |
| `.statusText` | `.statusText` (read-only) | HTTP reason phrase (e.g., `"Not Found"`) |
| `.getAllResponseHeaders()` | `.getAllResponseHeaders()` | Returns all response headers as a single CRLF-delimited string |
| `.getResponseHeader()` | `.getResponseHeader(name)` | Returns one named response header value |

### WinHttp.WinHttpRequest.5.1 COM Object — Extended Capabilities
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `.responseBody` | `.responseBody` (read-only) | Response entity body as SafeArray of unsigned bytes — use for binary downloads (images, ZIPs, PDFs); not available on Msxml2 |
| `.responseStream` | `.responseStream` (read-only) | Response entity body as `IStream` COM object — use for large streaming responses |
| `.setProxy()` | `.setProxy(proxySetting, proxyServer?, bypassList?)` | Explicit proxy config: `0`=default, `1`=no proxy, `2`=named proxy; `proxyServer` = `"server:port"` |
| `.setCredentials()` | `.setCredentials(username, password, flags)` | Built-in HTTP or proxy authentication; `flags`: `0`=server, `1`=proxy |
| `.setClientCertificate()` | `.setClientCertificate(clientCertificate)` | Select TLS client certificate; cert string format: `"CURRENT_USER\My\CertSubject"` |
| `.setAutoLogonPolicy()` | `.setAutoLogonPolicy(autoLogonPolicy)` | NTLM/Kerberos auto-logon: `0`=always, `1`=on intranet, `2`=never |
| `.option` | `.option[optionID]` (read/write) | Get or set a WinHTTP option value by numeric ID |

### WaitForResponse — Three Blocking Modes
| Call form | Blocks message loop? | Blocks current code? | Use when |
|-----------|---------------------|---------------------|----------|
| `.waitForResponse()` | No — pumps it | Yes — code waits | GUI script needs a single request; simplest correct async pattern per AHK v2 docs |
| `.waitForResponse(0)` | No | No — returns immediately | Inside `SetTimer` callback for fully non-blocking poll |
| `.waitForResponse(n)` | No — pumps it | Yes — up to n seconds | Timed wait with explicit deadline |
| `open(false)` + `send()` | **Yes — freezes GUI** | Yes | Background scripts with no GUI only |

### WinHttp vs Msxml2 — Capability Comparison
| Aspect | Msxml2.ServerXMLHTTP.6.0 | WinHttp.WinHttpRequest.5.1 |
|--------|--------------------------|---------------------------|
| Part of Windows OS | No — requires MSXML6 package | Yes — built into Windows since XP SP1 |
| Async completion API | `readyState` 0–4 polling via SetTimer | `WaitForResponse()` or `WaitForResponse(0)` |
| onreadystatechange callback | Not available (`Msxml2.XMLHTTP` has it, not ServerXMLHTTP) | Not available |
| Binary response | No native byte array | `.responseBody` SafeArray, `.responseStream` IStream |
| Proxy configuration | WPAD auto-detect only | `.setProxy()` explicit + auto-detect |
| HTTP/proxy authentication | Manual `Authorization` header | `.setCredentials()` built-in |
| TLS client certificates | Not available | `.setClientCertificate()` |
| ComObjConnect events | Not supported | Not supported (IUnknown-based, not IDispatch) |
| Default ReceiveTimeout | Infinite (hang risk) | 30 seconds |
| ResolveTimeout | Safe to set non-zero | Must be 0 — non-zero causes DNS thread handle leaks |

### HttpPromise Class (defined in TIER 5)
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `HttpPromise()` | `HttpPromise(method, url, opts?)` | `opts` Map: `connectMs`, `sendMs`, `receiveMs`, `headers`, `body`, `contentType`; COM object stored in `this._xhr`; SetTimer polling starts in constructor |
| `.then()` | `.then(onFulfilled)` | Register success callback; if already fulfilled, fires immediately; returns `this` for chaining |
| `.catch()` | `.catch(onRejected)` | Register error callback; if already rejected, fires immediately; returns `this` for chaining; **requires AHK v2.0.3+** — `catch` as method name was a parser bug before v2.0.3 |
| `.abort()` | `.abort()` | Stop polling, cancel COM request, call reject handlers; nullifies `_pollFn` to break reference cycle |

### Jxon Community Library (third-party — required for full JSON support)
| Method | Signature | Notes |
|--------|-----------|-------|
| `Jxon.Load()` | `Jxon.Load(jsonString)` | Parses JSON string into AHK v2 Map / Array structure; Arrays are 1-based |
| `Jxon.Dump()` | `Jxon.Dump(obj, indent?)` | Serializes Map / Array to JSON string |

### Path and File Helpers (used in TIER 1)
| Function | Signature | Notes |
|----------|-----------|-------|
| `SplitPath()` | `SplitPath(path, &name?, &dir?, &ext?, &nameNoExt?, &drive?)` | Output vars passed by reference |
| `FileExist()` | `FileExist(path)` | Returns attribute string or `""` — not a boolean; use `!= ""` for truthiness |
| `DirExist()` | `DirExist(path)` | Returns attribute string or `""` |
| `DirCreate()` | `DirCreate(path)` | Creates directory; throws `OSError` if it already exists on some systems |

## AHK V2 CONSTRAINTS

- Use `ComObject("WinHttp.WinHttpRequest.5.1")` for every HTTP session — `ComObjCreate()` was removed in AHK v2 and throws `MethodError` immediately. `Msxml2.ServerXMLHTTP.6.0` also works but is an MSXML6-stack dependency; WinHttp is the native Windows OS component preferred for new scripts.
- **Always set `setTimeouts` ResolveTimeout (first parameter) to `0`** — any non-zero value triggers a thread handle leak on every DNS resolution in WinHTTP per Microsoft documentation. Windows provides its own DNS client timeout. Correct call: `xhr.setTimeouts(0, 10000, 30000, 30000)`.
- AHK v2 ships with no JSON parser — never generate `JSON.Parse()`, `JSON.Stringify()`, `JSON.Load()`, or any similar built-in form; use the `Jxon` community library (`#Include Jxon.ahk`), or use `RegExMatch` named-capture patterns for known single-field extraction from flat JSON only.
- **WinHttp `WaitForResponse()` has three distinct modes** — the no-arg form blocks current code flow but pumps the AHK message loop (GUI and other timers remain active, per official AHK v2 docs); `WaitForResponse(0)` returns true/false immediately for use inside `SetTimer` polling; `open(false)` + `send()` blocks the entire message loop and freezes the GUI.
- Always check `xhr.status` after `send()` before reading `xhr.responseText` — WinHttp does not throw on HTTP 4xx/5xx; accessing `responseText` on a failed request silently returns an empty string or the error body.
- **WinHttp has no `readyState` property** — do not read `xhr.readyState` on a `WinHttp.WinHttpRequest.5.1` object; it does not exist and always returns empty. For async completion use `xhr.waitForResponse(0)`.
- `WinHttp.WinHttpRequest.5.1` does NOT support `ComObjConnect()` event sinks — the event interface `IWinHttpRequestEvent` derives from `IUnknown`, not `IDispatch`; `ComObjConnect` returns `0x80004002`. For async requests: use `open(true)` + `WaitForResponse()` (simplest) or `open(true)` + `SetTimer` + `WaitForResponse(0)` (fully non-blocking).
- **Reference cycle rule**: storing a BoundFunc that captures `this` as a property of `this` (e.g. `this._pollFn := this._Poll.Bind(this)`) creates a cycle that prevents GC under AHK v2 reference counting. Always nullify with `this._pollFn := unset` immediately after calling `SetTimer(this._pollFn, 0)`.
- Store async COM objects in script-scope (global or static) variables — a local-variable COM object goes out of scope when the enclosing function returns, silently cancelling the in-flight request.
- Escape all user-supplied strings with `JsonEscape()` before embedding in JSON payloads — unescaped `"` and `\` produce malformed JSON silently.
- ✗ `xhr := ComObjCreate("WinHttp.WinHttpRequest.5.1")` — MethodError, no session
- ✓ `xhr := ComObject("WinHttp.WinHttpRequest.5.1")` — correct v2 COM instantiation
- ✗ `xhr.setTimeouts(5000, 10000, 30000, 30000)` — non-zero ResolveTimeout causes DNS thread handle leak
- ✓ `xhr.setTimeouts(0, 10000, 30000, 30000)` — ResolveTimeout must be 0
- ✗ `data := JSON.Parse(xhr.responseText)` — MethodError, no native parser
- ✓ `data := Jxon.Load(xhr.responseText)` — community library after `#Include Jxon.ahk`
- ✗ `open(false)` + `send()` in GUI script — AHK message loop blocked, GUI freezes
- ✓ `open(true)` + `WaitForResponse()` — message loop pumped, GUI stays live
- ✗ `return xhr.responseText` immediately after `send()` — silent empty string on 4xx/5xx
- ✓ `if (xhr.status == 200) return xhr.responseText` — always validate status first
- ✗ `ComObjConnect(xhr, sink)` — `0x80004002` error; WinHttp events use IUnknown
- ✓ `open(true)` + `WaitForResponse()` or `SetTimer` + `WaitForResponse(0)` — correct async patterns
- ✗ `if (xhr.readyState == 4)` — readyState does not exist on WinHttp; always returns empty
- ✓ `if xhr.waitForResponse(0)` — WinHttp's non-blocking async completion test
- ✗ `this._pollFn := this.Method.Bind(this)` left set after timer deletion — reference cycle, GC blocked
- ✓ After `SetTimer(this._pollFn, 0)`, immediately `this._pollFn := unset` — cycle broken

Safe-access priority order for network responses:
  1. `Download(url, path)` in `try/catch` — complete file fetches, single-call, auto-close
  2. `ComObject("WinHttp.WinHttpRequest.5.1")` + `open(true)` + `WaitForResponse()` — blocking-but-responsive single request; simplest correct async per AHK v2 docs
  3. `open(true)` + `SetTimer` + `WaitForResponse(0)` polling — fully non-blocking; use when current code flow must return immediately
  4. `SetTimer` + synchronous `open(false)` loop (`ApiPoller`) — periodic polling; no COM lifetime risk, simple to debug

## TIER 1 — Basic File Download
> METHODS COVERED: Download · FileExist · DirExist · DirCreate · SplitPath

`Download()` is AHK v2's built-in single-function file fetcher. It accepts a URL and a local destination path, executes synchronously on the calling thread for its entire duration, and throws an `OSError` on network-level failure (DNS, timeout, refused connection) or disk write failure. HTTP 4xx/5xx responses are not exceptions — the response body is silently saved to the destination path. All TIER 1 usage must wrap `Download()` in `try/catch` and verify the file exists after the call. Never use `Download()` for large files in GUI contexts — the AHK thread blocks completely until the transfer finishes.
```ahk
; ── TIER 1: Download() with error handling ────────────────────────────────────
#Requires AutoHotkey v2.0

DownloadFile(url, destPath) {
    ; ✓ Guard: ensure destination directory exists before attempting write
    SplitPath(destPath, , &destDir)
    if destDir && !DirExist(destDir)
        DirCreate(destDir)

    try {
        Download(url, destPath)

        ; ✓ Verify the file was actually written — Download() may succeed with empty content
        if !FileExist(destPath)
            throw Error("File not found after download", -1, destPath)

        return true

    } catch OSError as e {
        ; Network-level failure (DNS, timeout, refused) or disk write failure
        MsgBox("Network error: " e.Message "`nCode: " e.Number, "Download Failed", 16)
        return false

    } catch Error as e {
        ; Manually thrown Error from file-existence check above
        MsgBox("Download error: " e.Message, "Download Failed", 16)
        return false
    }
}

; ── Usage examples ────────────────────────────────────────────────────────────
; ✓ Explicit destination path, result checked before proceeding
if DownloadFile("https://example.com/data.csv", A_Desktop "\data.csv")
    MsgBox("Saved to desktop")

; ✗ No try/catch, no existence check — silent failure or unhandled exception
; Download("https://example.com/data.csv", "data.csv")   ; → uncaught Error on timeout/network failure; HTTP 4xx/5xx silently saves error page

; ✓ Batch download with polite delay to avoid rate limiting
urls := [
    "https://example.com/file1.txt",
    "https://example.com/file2.txt",
    "https://example.com/file3.txt"
]
for i, url in urls {
    SplitPath(url, &fname)
    DownloadFile(url, A_Temp "\" fname)
    Sleep(500)   ; polite delay between requests
}
```

## TIER 2 — Synchronous HTTP GET
> METHODS COVERED: ComObject · .setTimeouts · .open · .send · .status · .responseText · .statusText · BuildQueryString · UriEncode · StrJoin

A synchronous GET request uses `ComObject("WinHttp.WinHttpRequest.5.1")` with `open()` in blocking mode (`async=false`), then reads `xhr.responseText` after `send()` completes. Use this mode only in non-GUI scripts — `open(false)` blocks the entire AHK message loop and freezes any GUI. For GUI scripts, use `open(true)` + `WaitForResponse()` (TIER 5). Every WinHttp session must follow this initialization sequence: `ComObject()` → `setTimeouts(0,...)` → `open()` → `setRequestHeader()` calls → `send()` → check `.status` → read `.responseText`.
```ahk
; ── TIER 2: Synchronous GET request ──────────────────────────────────────────
#Requires AutoHotkey v2.0

HttpGet(url) {
    ; ✓ Correct AHK v2 COM instantiation — WinHttp is the preferred native Windows HTTP component
    xhr := ComObject("WinHttp.WinHttpRequest.5.1")

    ; ✗ Wrong — ComObjCreate() was removed; produces MethodError immediately
    ; xhr := ComObjCreate("WinHttp.WinHttpRequest.5.1")   ; → MethodError

    ; ✓ ResolveTimeout MUST be 0 — any non-zero value causes DNS thread handle leaks
    ; Params: resolve(must be 0), connect, send, receive (milliseconds)
    xhr.setTimeouts(0, 10000, 30000, 30000)

    ; ✗ Wrong — non-zero ResolveTimeout leaks a thread handle on every DNS lookup
    ; xhr.setTimeouts(5000, 10000, 30000, 30000)   ; → DNS thread handle leak per Microsoft KB

    try {
        ; ✓ false = synchronous (blocking) — use only in non-GUI scripts
        ; ✗ In GUI scripts use open(true) + WaitForResponse() — open(false) freezes GUI
        xhr.open("GET", url, false)
        xhr.send()

        ; ✓ Always check status before reading response body
        if (xhr.status == 200)
            return xhr.responseText

        ; ✗ Wrong — reading responseText without status check silently returns empty on error
        ; return xhr.responseText   ; → empty string on 404/500 with no error signal

        ; Non-200 responses: surface status code to caller for context-specific recovery
        throw Error("HTTP " xhr.status " from " url, -1, xhr.statusText)

    } catch Error as e {
        throw   ; re-throw; let caller handle context-specific recovery
    }
}

; ── URL encoding helper for query parameters ──────────────────────────────────
BuildQueryString(params) {
    ; params is a Map of key → value pairs
    parts := []
    for key, val in params
        parts.Push(UriEncode(key) "=" UriEncode(val))
    return parts.Length ? "?" StrJoin(parts, "&") : ""
}

UriEncode(str) {
    ; ✓ Basic percent-encoding for query parameter values
    encoded := ""
    loop parse str {
        char := A_LoopField
        if RegExMatch(char, "[A-Za-z0-9\-_.~]")
            encoded .= char
        else
            encoded .= Format("%{:02X}", Ord(char))
    }
    return encoded
}

StrJoin(arr, sep) {
    result := ""
    for i, v in arr
        result .= (i > 1 ? sep : "") v
    return result
}

; ── Usage ─────────────────────────────────────────────────────────────────────
try {
    ; Simple GET
    body := HttpGet("https://httpbin.org/get")
    MsgBox(SubStr(body, 1, 300))   ; preview first 300 chars

    ; GET with query params
    qp := Map("q", "autohotkey v2", "limit", "10")
    body2 := HttpGet("https://api.example.com/search" BuildQueryString(qp))
    MsgBox(body2)

} catch Error as e {
    MsgBox("Request failed: " e.Message)
}
```

## TIER 3 — HTTP POST with Headers and Authentication
> METHODS COVERED: .setRequestHeader · .getAllResponseHeaders · .setCredentials · BasicAuthHeader · HttpSend · MapToJsonFlat · JsonEscape · PostJsonWithBearer · StrJoin

POST requests extend the TIER 2 baseline by adding `setRequestHeader()` calls for `Content-Type`, `Authorization`, and any custom headers, then passing a string payload to `send()`. This tier also covers Bearer token injection, Basic Auth construction via `CryptBinaryToStringW`, and the built-in `.setCredentials()` method unique to WinHttp. The `HttpSend()` function in this tier is the general-purpose multi-method sender that all higher tiers build on. The `PUT`/`PATCH`/`DELETE` method variants follow identical patterns with different method strings.
```ahk
; ── TIER 3: POST with headers and auth ───────────────────────────────────────
#Requires AutoHotkey v2.0

; ✓ Encode Base64 credentials for Basic Auth via Windows CryptBinaryToStringW
BasicAuthHeader(username, password) {
    credentials := username ":" password
    ; AHK v2 built-in Base64 via CryptBinaryToStringW
    buf := Buffer(StrPut(credentials, "UTF-8"))
    StrPut(credentials, buf, "UTF-8")
    encoded := ""
    DllCall("Crypt32\CryptBinaryToStringW",
        "Ptr",  buf,
        "UInt", buf.Size - 1,
        "UInt", 0x00000001,   ; CRYPT_STRING_BASE64
        "Ptr",  0,
        "UInt*", &size := 0)
    outBuf := Buffer(size * 2)
    DllCall("Crypt32\CryptBinaryToStringW",
        "Ptr",  buf,
        "UInt", buf.Size - 1,
        "UInt", 0x00000001,
        "Ptr",  outBuf,
        "UInt*", &size)
    encoded := StrGet(outBuf, "UTF-16")
    ; Strip trailing CRLF added by CryptBinaryToStringW
    return "Basic " Trim(encoded, "`r`n ")
}

; ✓ Generic HTTP method sender — baseline for PUT, PATCH, DELETE, POST
HttpSend(method, url, payload := "", headers := Map()) {
    xhr := ComObject("WinHttp.WinHttpRequest.5.1")
    ; ✓ ResolveTimeout = 0 to avoid DNS thread handle leaks
    xhr.setTimeouts(0, 10000, 30000, 30000)

    try {
        xhr.open(method, url, false)

        ; Apply all supplied headers — setRequestHeader must be called after open()
        for hKey, hVal in headers
            xhr.setRequestHeader(hKey, hVal)

        xhr.send(payload)

        return Map(
            "status",  xhr.status,
            "body",    xhr.responseText,
            "headers", xhr.getAllResponseHeaders()
        )

    } catch Error as e {
        throw Error("HttpSend failed [" method " " url "]: " e.Message, -1, e.Extra)
    }
}

; ── WinHttp built-in credential auth (alternative to manual header) ───────────
; ✓ WinHttp.WinHttpRequest.5.1 supports SetCredentials() for server or proxy auth
HttpGetWithCredentials(url, username, password) {
    xhr := ComObject("WinHttp.WinHttpRequest.5.1")
    xhr.setTimeouts(0, 10000, 30000, 30000)
    xhr.open("GET", url, false)

    ; ✓ Built-in credentials: flags 0=server auth, 1=proxy auth
    xhr.setCredentials(username, password, 0)

    xhr.send()
    if (xhr.status == 200)
        return xhr.responseText
    throw Error("HTTP " xhr.status, -1, xhr.statusText)
}

; ── POST with JSON body and Bearer token ─────────────────────────────────────
PostJsonWithBearer(url, token, dataMap) {
    ; ✓ Build JSON payload from flat Map — use Jxon for nested structures
    payload := MapToJsonFlat(dataMap)

    headers := Map(
        "Content-Type",  "application/json",
        "Authorization", "Bearer " token,
        "Accept",        "application/json"
    )

    result := HttpSend("POST", url, payload, headers)

    if (result["status"] >= 400)
        throw Error("API error " result["status"], -1, SubStr(result["body"], 1, 200))

    return result["body"]
}

; ✓ Flat Map → JSON string (single-level strings and numbers only)
; ✗ Do not attempt manual serialization for deeply nested or array-heavy data — use Jxon
MapToJsonFlat(m) {
    parts := []
    for k, v in m {
        escapedKey := JsonEscape(String(k))
        if (Type(v) == "Integer" || Type(v) == "Float")
            parts.Push('"' escapedKey '":' v)
        else
            parts.Push('"' escapedKey '":"' JsonEscape(String(v)) '"')
    }
    return "{" StrJoin(parts, ",") "}"
}

; ✓ Escape helper — always run user input through this before embedding in JSON
JsonEscape(str) {
    str := StrReplace(str, "\",  "\\")
    str := StrReplace(str, '"',  '\"')
    str := StrReplace(str, "`n", "\n")
    str := StrReplace(str, "`r", "\r")
    str := StrReplace(str, "`t", "\t")
    return str
}

StrJoin(arr, sep) {
    result := ""
    for i, v in arr
        result .= (i > 1 ? sep : "") v
    return result
}

; ── PUT and DELETE follow same pattern ───────────────────────────────────────
; result := HttpSend("PUT",    url, payload, headers)
; result := HttpSend("DELETE", url, "",      headers)

; ── Usage ─────────────────────────────────────────────────────────────────────
try {
    token := "eyJhbGciOiJIUzI1NiJ9.example"
    url   := "https://api.example.com/users"
    data  := Map("name", "Alice", "role", "admin", "active", 1)

    response := PostJsonWithBearer(url, token, data)
    MsgBox("Created: " SubStr(response, 1, 200))

} catch Error as e {
    MsgBox("POST failed: " e.Message)
}
```

## TIER 4 — JSON Handling Strategies
> METHODS COVERED: Jxon.Load · Jxon.Dump · RegExMatch · ExtractJsonField · ArrayOfMapsToJson · MapToJsonFlat · JsonEscape · StrJoin

AHK v2 has no built-in JSON parser. This tier documents the three valid approaches: **(A)** community library `Jxon` for full parsing and serialization, **(B)** `RegExMatch` named-capture patterns for single known-field extraction from simple flat JSON, and **(C)** manual `MapToJsonFlat` serialization for outbound payloads. Complex or nested JSON always requires Jxon — `RegExMatch` patterns break silently on nested objects, escaped characters within values, or JSON arrays. Never generate `JSON.Parse()`, `JSON.Stringify()`, or any built-in form; these throw `MethodError` at runtime.
```ahk
; ── TIER 4A: Jxon community library (RECOMMENDED) ────────────────────────────
; Download: https://github.com/cocobelgica/AutoHotkey-JSON
; Place Jxon.ahk next to your script, then:

; #Include Jxon.ahk
;
; ParseWithJxon(jsonString) {
;     data := Jxon.Load(jsonString)     ; returns Map or Array
;
;     ; Accessing object fields
;     name := data["name"]              ; ✓ Map key access
;     age  := data["profile"]["age"]    ; ✓ Nested Map
;
;     ; ✓ Accessing array items — 1-based in AHK v2 Arrays returned by Jxon
;     first := data["items"][1]         ; ✓ 1-based index
;     ; first := data["items"][0]       ; ✗ throws IndexError — AHK Arrays are 1-based
;
;     ; Serializing back to JSON
;     outJson := Jxon.Dump(data)
;     return outJson
; }

; ── TIER 4B: RegExMatch for single-field extraction (no library) ──────────────
; ✓ Only suitable for simple, flat, known-structure JSON responses.
; ✗ Never use for arrays, nested objects, or fields containing escaped characters.

ExtractJsonField(json, fieldName) {
    ; Matches string values: "fieldName": "value"
    pattern := '"' fieldName '"\s*:\s*"(?P<val>[^"\\]*(?:\\.[^"\\]*)*)"'
    if RegExMatch(json, pattern, &m)
        return m["val"]

    ; Matches numeric values: "fieldName": 123 or 123.45
    pattern2 := '"' fieldName '"\s*:\s*(?P<val>-?\d+(?:\.\d+)?)'
    if RegExMatch(json, pattern2, &m2)
        return m2["val"]

    ; Matches boolean/null values
    pattern3 := '"' fieldName '"\s*:\s*(?P<val>true|false|null)'
    if RegExMatch(json, pattern3, &m3)
        return m3["val"]

    return ""   ; field not found
}

; ── TIER 4C: Array serialization ──────────────────────────────────────────────
; ✓ Convert an AHK Array of Maps into a JSON array string
ArrayOfMapsToJson(arr) {
    items := []
    for item in arr {
        ; Each item must be a flat Map
        items.Push(MapToJsonFlat(item))
    }
    return "[" StrJoin(items, ",") "]"
}

MapToJsonFlat(m) {
    parts := []
    for k, v in m {
        escapedKey := JsonEscape(String(k))
        if (Type(v) == "Integer" || Type(v) == "Float")
            parts.Push('"' escapedKey '":' v)
        else
            parts.Push('"' escapedKey '":"' JsonEscape(String(v)) '"')
    }
    return "{" StrJoin(parts, ",") "}"
}

JsonEscape(str) {
    str := StrReplace(str, "\",  "\\")
    str := StrReplace(str, '"',  '\"')
    str := StrReplace(str, "`n", "\n")
    str := StrReplace(str, "`r", "\r")
    str := StrReplace(str, "`t", "\t")
    return str
}

StrJoin(arr, sep) {
    result := ""
    for i, v in arr
        result .= (i > 1 ? sep : "") v
    return result
}

; ── Usage ─────────────────────────────────────────────────────────────────────
; Simulate API response
jsonResponse := '{"id":42,"username":"alice","active":true,"score":98.5}'

id       := ExtractJsonField(jsonResponse, "id")        ; "42"
username := ExtractJsonField(jsonResponse, "username")  ; "alice"
active   := ExtractJsonField(jsonResponse, "active")    ; "true"
MsgBox("User: " username " | ID: " id " | Active: " active)

; ✗ Wrong: this function does not exist in AHK v2 — produces MethodError at runtime
; parsed := JSON.Parse(jsonResponse)   ; → MethodError: "JSON" has no method "Parse"

; ✓ Build outbound JSON array
users := [
    Map("name", "Alice", "role", "admin"),
    Map("name", "Bob",   "role", "viewer")
]
payload := ArrayOfMapsToJson(users)
MsgBox(payload)
; → [{"name":"Alice","role":"admin"},{"name":"Bob","role":"viewer"}]
```

## TIER 5 — Asynchronous HTTP and Promise-Style Concurrency
> METHODS COVERED: .open (async=true) · .send · .waitForResponse · .waitForResponse(0) · .abort · SetTimer · AsyncGetSimple · AsyncGet · PollReady · ApiPoller · .Start · .Stop · .Poll · HttpPromise.__New · HttpPromise.then · HttpPromise.catch · HttpPromise.abort · HttpPromise._Poll · HttpPromise._resolve · HttpPromise._reject

WinHttp async works because `open(true)` moves HTTP I/O onto a Windows thread pool background thread. AHK v2 itself remains single-threaded and cooperative — timers and hotkeys are serviced in the message loop gaps, not in parallel OS threads. Three async patterns exist with different tradeoffs:

**Pattern A (`open(true)` + `WaitForResponse()`):** the official AHK v2 docs pattern. Blocks the current code flow but pumps the message loop — other `SetTimer` callbacks and hotkeys remain active. Simplest correct choice for GUI scripts that need a single request.

**Pattern B (`SetTimer` + `WaitForResponse(0)`):** truly non-blocking — current function returns immediately. Polling is done in a timer callback that fires every 50 ms. Required when current code must not block at all, or for multiple concurrent in-flight requests.

**Pattern C (ApiPoller, `SetTimer` + sync `open(false)`):** periodic polling where a fresh synchronous COM object fires on each interval. Simplest lifetime management — no async state to track.

**Pattern D (HttpPromise):** Promise-style API for concurrent requests. Multiple HttpPromise instances run simultaneously, each managing its own COM object, timer, and callback queue. Critical: storing `this._pollFn := this._Poll.Bind(this)` creates a reference cycle; always set `this._pollFn := unset` after stopping the timer to allow GC.
```ahk
; ── TIER 5A: open(true) + WaitForResponse() — simplest GUI-safe async ─────────
; Per official AHK v2 docs: this pattern keeps the script responsive
#Requires AutoHotkey v2.0

AsyncGetSimple(url) {
    xhr := ComObject("WinHttp.WinHttpRequest.5.1")
    ; ✓ ResolveTimeout = 0 — non-zero causes DNS thread handle leaks
    xhr.setTimeouts(0, 10000, 30000, 30000)
    xhr.open("GET", url, true)   ; true = async; HTTP I/O on background thread
    xhr.send()

    ; ✓ WaitForResponse() no-arg: blocks current code but pumps message loop
    ;   GUI events, hotkeys, and other SetTimer callbacks still fire during wait
    xhr.waitForResponse()

    ; ✗ open(false) + send() — entire message loop blocked, GUI frozen
    ; xhr.open("GET", url, false)   ; → GUI freezes until response

    if (xhr.status == 200)
        return xhr.responseText
    throw Error("HTTP " xhr.status, -1, xhr.statusText)
}

; ── TIER 5B: SetTimer + WaitForResponse(0) — fully non-blocking ───────────────
; ✓ Script-scope variables keep the COM object alive for the async duration
; ✗ Wrong: local variable inside AsyncGet() — GC'd when function returns
global g_xhr    := unset
global g_pollFn := unset

AsyncGet(url, onComplete) {
    global g_xhr, g_pollFn

    ; ✓ Correct: WinHttp.WinHttpRequest.5.1 — native Windows HTTP
    g_xhr := ComObject("WinHttp.WinHttpRequest.5.1")
    g_xhr.setTimeouts(0, 10000, 30000, 30000)

    g_xhr.open("GET", url, true)
    g_xhr.send()

    ; ✗ Wrong: ComObjConnect(g_xhr, sink) — 0x80004002; WinHttp uses IUnknown not IDispatch
    ; ✗ Wrong: if (g_xhr.readyState == 4) — readyState does not exist on WinHttp
    ; ✓ Correct: poll WaitForResponse(0) — returns true when response ready, false if not
    g_pollFn := PollReady.Bind(onComplete)
    SetTimer(g_pollFn, 50)
}

PollReady(callback) {
    global g_xhr, g_pollFn

    ; ✓ WaitForResponse(0) = non-blocking: true if done, false if in-flight
    if !g_xhr.waitForResponse(0)
        return   ; still in progress — timer fires again in 50 ms

    SetTimer(g_pollFn, 0)
    status := g_xhr.status
    body   := (status == 200) ? g_xhr.responseText : ""
    callback.Call(status, body)
}

AbortAsync() {
    global g_xhr, g_pollFn
    if IsSet(g_pollFn)
        SetTimer(g_pollFn, 0)
    ; ✓ WinHttp has no readyState — just call abort() directly
    ; ✗ Wrong: if IsSet(g_xhr) && (g_xhr.readyState != 0)  ; readyState not on WinHttp
    if IsSet(g_xhr)
        g_xhr.abort()
}

; ── TIER 5C: SetTimer periodic polling (RECOMMENDED for periodic checks) ───────
class ApiPoller {
    __New(url, intervalMs, onData) {
        this.url        := url
        this.intervalMs := intervalMs
        this.onData     := onData
        this.timerFn    := this.Poll.Bind(this)
    }

    Start() {
        SetTimer(this.timerFn, this.intervalMs)
    }

    Stop() {
        SetTimer(this.timerFn, 0)
    }

    Poll() {
        try {
            ; ✓ Fresh synchronous COM object per poll — no shared state between polls
            xhr := ComObject("WinHttp.WinHttpRequest.5.1")
            xhr.setTimeouts(0, 5000, 10000, 10000)
            xhr.open("GET", this.url, false)
            xhr.send()
            if (xhr.status == 200)
                this.onData.Call(xhr.responseText)
        } catch Error {
            ; Silent fail on polling interval — log if needed
        }
    }
}

; ── TIER 5D: HttpPromise — Promise-style API for concurrent requests ───────────
; Requires AHK v2.0.3+ for .catch() method name (keyword-as-method fixed in v2.0.3)
class HttpPromise {
    static PENDING   := 0
    static FULFILLED := 1
    static REJECTED  := 2

    __New(method, url, opts := Map()) {
        this._state   := HttpPromise.PENDING
        this._value   := unset
        this._reason  := unset
        this._thens   := []
        this._catches := []

        ; ✓ COM object stored in this — keeps request alive for async duration
        this._xhr := ComObject("WinHttp.WinHttpRequest.5.1")
        ; ✓ ResolveTimeout = 0 — non-zero causes DNS thread handle leaks
        this._xhr.setTimeouts(0, opts.Get("connectMs", 10000),
                                 opts.Get("sendMs",    30000),
                                 opts.Get("receiveMs", 30000))

        this._xhr.open(method, url, true)
        if opts.Has("headers")
            for k, v in opts["headers"]
                this._xhr.setRequestHeader(k, v)

        payload := opts.Get("body", "")
        if (payload != "")
            this._xhr.setRequestHeader("Content-Type",
                opts.Get("contentType", "application/json; charset=UTF-8"))

        this._xhr.send(payload)

        ; ✗ Wrong: do NOT store BoundFunc in this without later nullifying
        ;   this._pollFn := this._Poll.Bind(this)  ; → reference cycle: this → fn → this
        ; ✓ Correct: store it, then nullify after SetTimer deletion to break the cycle
        this._pollFn := this._Poll.Bind(this)
        SetTimer(this._pollFn, 50)
    }

    _Poll() {
        if !this._xhr.waitForResponse(0)
            return   ; still in-flight — timer fires again in 50 ms

        ; ✓ Stop timer and immediately break reference cycle before doing anything else
        SetTimer(this._pollFn, 0)
        this._pollFn := unset   ; cycle broken: this no longer holds a ref to the BoundFunc

        status := this._xhr.status
        body   := this._xhr.responseText

        if (status >= 200 && status < 300)
            this._resolve(Map("status", status, "body", body,
                              "headers", this._xhr.getAllResponseHeaders()))
        else
            this._reject(Error("HTTP " status, -1, SubStr(body, 1, 300)))
    }

    _resolve(value) {
        if (this._state != HttpPromise.PENDING)
            return
        this._state := HttpPromise.FULFILLED
        this._value := value
        for cb in this._thens
            cb.Call(value)
    }

    _reject(reason) {
        if (this._state != HttpPromise.PENDING)
            return
        this._state := HttpPromise.REJECTED
        this._reason := reason
        for cb in this._catches
            cb.Call(reason)
        ; If no .catch() handler registered, surface the error to AHK's default handler
        if (this._catches.Length == 0)
            throw reason
    }

    ; .then(onFulfilled) — register success callback; returns this for chaining
    then(onFulfilled) {
        if (this._state == HttpPromise.FULFILLED)
            onFulfilled.Call(this._value)
        else if (this._state == HttpPromise.PENDING)
            this._thens.Push(onFulfilled)
        return this
    }

    ; .catch(onRejected) — register error callback; requires AHK v2.0.3+
    catch(onRejected) {
        if (this._state == HttpPromise.REJECTED)
            onRejected.Call(this._reason)
        else if (this._state == HttpPromise.PENDING)
            this._catches.Push(onRejected)
        return this
    }

    abort() {
        if IsSet(this._pollFn) {
            SetTimer(this._pollFn, 0)
            this._pollFn := unset   ; ✓ break reference cycle on abort path too
        }
        this._xhr.abort()
        this._reject(Error("Request aborted", -1))
    }
}

; ── Factory helpers ───────────────────────────────────────────────────────────
HttpGetAsync(url, headers := Map()) {
    return HttpPromise("GET", url, Map("headers", headers))
}

HttpPostAsync(url, body, headers := Map()) {
    return HttpPromise("POST", url, Map("body", body, "headers", headers))
}

; ── Usage: Pattern A — simple blocking-but-responsive ────────────────────────
try {
    body := AsyncGetSimple("https://httpbin.org/get")
    MsgBox(SubStr(body, 1, 200))
} catch Error as e {
    MsgBox("Failed: " e.Message)
}

; ── Usage: Pattern B — fully non-blocking single request ─────────────────────
HandleResponse(status, body) {
    MsgBox("Status: " status "`nBody: " SubStr(body, 1, 200))
}
AsyncGet("https://httpbin.org/get", HandleResponse)

; ── Usage: Pattern C — periodic polling ──────────────────────────────────────
HandlePollData(body) {
    value := ExtractJsonField(body, "timestamp")
    ToolTip("Latest: " value)
}
poller := ApiPoller("https://api.example.com/status", 60000, HandlePollData)
poller.Start()

; ── Usage: Pattern D — concurrent requests, Promise API ──────────────────────
HandleUser(res) {
    MsgBox("User: " res["body"])
}
HandleProduct(res) {
    MsgBox("Product: " res["body"])
}
HandleError(err) {
    MsgBox("Failed: " err.Message, "Error", 16)
}

; ✓ Three requests in-flight simultaneously — each has its own COM object and timer
p1 := HttpGetAsync("https://api.example.com/users/1")
p2 := HttpGetAsync("https://api.example.com/products/1")
p3 := HttpPostAsync("https://api.example.com/log", '{"event":"startup"}')

p1.then(HandleUser).catch(HandleError)
p2.then(HandleProduct).catch(HandleError)
p3.catch(HandleError)
; ✓ All three lines return immediately — AHK main thread stays responsive
```

### Performance Notes

Network I/O is orders of magnitude slower than in-memory operations. Apply these principles to avoid the most common AHK v2 network performance pitfalls:

**WaitForResponse vs SetTimer vs HttpPromise:** For a single request in a GUI script, `open(true)` + `WaitForResponse()` is the simplest correct choice — one function call, no timer management, message loop pumped during wait. Use `SetTimer` + `WaitForResponse(0)` only when current code must return immediately. Use `HttpPromise` when firing multiple concurrent requests — each instance manages its own lifetime independently.

**Request batching:** Avoid one HTTP call per loop iteration. A loop of N items makes N round-trips. The per-request overhead (DNS + TCP + TLS handshake) dwarfs payload transfer time for most APIs. When the API supports batch or bulk endpoints, compose a single request.

**Timeout sizing:** Leave ResolveTimeout at `0` always. Set connect timeout short (2–3 s) for fast APIs; use longer receive timeouts (60–300 s) only for large file downloads. Oversized timeouts mask connectivity problems.

**COM object reuse in tight loops:** Creating a new `ComObject("WinHttp.WinHttpRequest.5.1")` for every request in a high-frequency polling loop is wasteful. Reuse a single instance — `open()` resets request state and can be called multiple times on the same object.

**JSON parsing cost:** `Jxon.Load()` parses the entire JSON tree. In hot polling loops where only one field is needed, `RegExMatch` named-capture on a known flat field is significantly faster.

**Rate limiting with Sleep:** Insert `Sleep(delayMs)` between requests in batch loops to avoid HTTP 429. A 500 ms inter-request delay is a reasonable default for public APIs without documented rate limits.

**Concurrent request ceiling:** HttpPromise instances fire simultaneously, each backed by a Windows thread pool thread. 10–20 concurrent instances is reasonable; beyond that, rate limiting and thread pool exhaustion become concerns.

**Binary responses:** When downloading binary content (images, PDFs, ZIPs), use `.responseBody` (SafeArray of bytes) rather than `.responseText` — text decoding corrupts binary data.

## TIER 6 — Full REST API Client Class
> METHODS COVERED: RestClient.__New · RestClient.Get · RestClient.Post · RestClient.Put · RestClient.Delete · RestClient.SetToken · RestClient.IsTokenValid · RestClient.RefreshToken · RestClient.BuildUrl · RestClient.Execute · RestClient.SendRequest · RestClient.ParseResponse · RestClient.UriEncode · RestClient.JoinStr · GitHubClient.__New · GitHubClient.GetRepo · GitHubClient.ListIssues · GitHubClient.CreateIssue · Jxon.Load · Jxon.Dump

A production REST API client encapsulates base URL management, token storage with expiry tracking, automatic retry with exponential backoff, and dynamic endpoint composition into a single reusable class. This tier integrates all lower-tier patterns and implements the robustness requirements for real-world API integrations. `#Include Jxon.ahk` is required at this tier — `Jxon.Dump()` is used for request serialization and `Jxon.Load()` for response parsing. When subclassing `RestClient`, override `RefreshToken()` to implement token renewal without modifying the base retry loop. `SendRequest` uses `open(false)` — this class is intended for background script contexts. For GUI-integrated usage, replace `open(false)` with `open(true)` + `WaitForResponse()` in `SendRequest`.
```ahk
; ── TIER 6: Production REST API client ───────────────────────────────────────
#Requires AutoHotkey v2.0
#Include Jxon.ahk   ; Community library for JSON parsing — required for TIER 6

class RestClient {
    ; ── Constructor ──────────────────────────────────────────────────────────
    __New(baseUrl, opts := Map()) {
        this.baseUrl     := RTrim(baseUrl, "/")
        this.token       := opts.Has("token")       ? opts["token"]       : ""
        this.tokenExpiry := opts.Has("tokenExpiry") ? opts["tokenExpiry"] : 0
        this.maxRetries  := opts.Has("maxRetries")  ? opts["maxRetries"]  : 3
        this.retryDelay  := opts.Has("retryDelay")  ? opts["retryDelay"]  : 1000
        this.rateLimitMs := opts.Has("rateLimitMs") ? opts["rateLimitMs"] : 0
        this.lastRequest := 0
        this.defaultHdrs := Map("Accept", "application/json")

        ; Copy any extra default headers from opts
        if opts.Has("headers")
            for k, v in opts["headers"]
                this.defaultHdrs[k] := v
    }

    ; ── Public HTTP verbs ────────────────────────────────────────────────────
    Get(path, params := Map()) {
        url := this.BuildUrl(path, params)
        return this.Execute("GET", url, "")
    }

    Post(path, body := Map()) {
        url     := this.BuildUrl(path)
        payload := IsObject(body) ? Jxon.Dump(body) : String(body)
        return this.Execute("POST", url, payload)
    }

    Put(path, body := Map()) {
        url     := this.BuildUrl(path)
        payload := IsObject(body) ? Jxon.Dump(body) : String(body)
        return this.Execute("PUT", url, payload)
    }

    Delete(path) {
        url := this.BuildUrl(path)
        return this.Execute("DELETE", url, "")
    }

    ; ── Token management ─────────────────────────────────────────────────────
    SetToken(token, expiryEpoch := 0) {
        this.token       := token
        this.tokenExpiry := expiryEpoch
    }

    IsTokenValid() {
        if !this.token
            return false
        if (this.tokenExpiry > 0 && DateDiff(A_NowUTC, "19700101000000", "Seconds") > this.tokenExpiry)
            return false
        return true
    }

    ; ✓ Override in subclass to implement token refresh
    RefreshToken() {
        throw Error("RefreshToken() not implemented in base RestClient", -1)
    }

    ; ── URL builder ──────────────────────────────────────────────────────────
    BuildUrl(path, params := Map()) {
        url := this.baseUrl "/" LTrim(path, "/")
        parts := []
        for k, v in params
            parts.Push(this.UriEncode(k) "=" this.UriEncode(String(v)))
        if parts.Length
            url .= "?" this.JoinStr(parts, "&")
        return url
    }

    ; ── Core request executor with retry + rate limiting ─────────────────────
    Execute(method, url, payload) {
        ; ✓ Rate limiting: enforce minimum interval between requests
        if (this.rateLimitMs > 0) {
            elapsed := A_TickCount - this.lastRequest
            if (elapsed < this.rateLimitMs)
                Sleep(this.rateLimitMs - elapsed)
        }

        ; Token refresh if expired
        if (this.tokenExpiry > 0 && !this.IsTokenValid())
            this.RefreshToken()

        attempt := 0
        loop {
            attempt++
            try {
                result := this.SendRequest(method, url, payload)
                this.lastRequest := A_TickCount

                ; 429 Too Many Requests — back off and retry
                if (result["status"] == 429 && attempt <= this.maxRetries) {
                    retryAfter := result["headers"] ~= "Retry-After: (\d+)"
                                  ? Integer(RegExReplace(result["headers"],
                                      ".*Retry-After: (\d+).*", "$1")) * 1000
                                  : this.retryDelay * (2 ** (attempt - 1))
                    Sleep(retryAfter)
                    continue
                }

                ; 401 Unauthorized — attempt token refresh once
                if (result["status"] == 401 && attempt == 1) {
                    this.RefreshToken()
                    continue
                }

                ; 5xx server error — retry with exponential backoff
                if (result["status"] >= 500 && attempt <= this.maxRetries) {
                    Sleep(this.retryDelay * (2 ** (attempt - 1)))
                    continue
                }

                ; Success or non-retryable failure — return parsed result
                return this.ParseResponse(result)

            } catch Error as e {
                ; Network-level exception — retry up to maxRetries
                if (attempt <= this.maxRetries) {
                    Sleep(this.retryDelay * (2 ** (attempt - 1)))
                    continue
                }
                throw Error("RestClient.Execute failed after " attempt
                    " attempts: " e.Message, -1, url)
            }
        }
    }

    ; ── Low-level WinHttp.WinHttpRequest.5.1 sender ───────────────────────────
    SendRequest(method, url, payload) {
        xhr := ComObject("WinHttp.WinHttpRequest.5.1")
        ; ✓ ResolveTimeout = 0 — non-zero leaks a thread handle per DNS lookup
        xhr.setTimeouts(0, 10000, 30000, 30000)
        xhr.open(method, url, false)

        ; Apply default headers
        for k, v in this.defaultHdrs
            xhr.setRequestHeader(k, v)

        ; Apply Authorization header if token is set
        if this.token
            xhr.setRequestHeader("Authorization", "Bearer " this.token)

        ; Content-Type for requests with body
        if (payload != "")
            xhr.setRequestHeader("Content-Type", "application/json; charset=UTF-8")

        xhr.send(payload)

        return Map(
            "status",  xhr.status,
            "body",    xhr.responseText,
            "headers", xhr.getAllResponseHeaders()
        )
    }

    ; ── Response parser ───────────────────────────────────────────────────────
    ParseResponse(result) {
        status := result["status"]
        body   := result["body"]

        if (status >= 400)
            throw Error("HTTP " status " error", -1, SubStr(body, 1, 300))

        ; ✓ Parse JSON if response body looks like JSON
        if (SubStr(LTrim(body), 1, 1) ~= "[{\[]")
            return Jxon.Load(body)

        ; Return raw string for non-JSON responses
        return body
    }

    ; ── Helpers ───────────────────────────────────────────────────────────────
    UriEncode(str) {
        encoded := ""
        loop parse str {
            char := A_LoopField
            if RegExMatch(char, "[A-Za-z0-9\-_.~]")
                encoded .= char
            else
                encoded .= Format("%{:02X}", Ord(char))
        }
        return encoded
    }

    JoinStr(arr, sep) {
        result := ""
        for i, v in arr
            result .= (i > 1 ? sep : "") v
        return result
    }
}

; ── GitHub API client — subclass example ─────────────────────────────────────
class GitHubClient extends RestClient {
    __New(token) {
        super.__New("https://api.github.com", Map(
            "token",      token,
            "maxRetries", 3,
            "retryDelay", 2000,
            "headers",    Map("User-Agent", "AHK-v2-Client/1.0")
        ))
    }

    GetRepo(owner, repo) {
        return this.Get("/repos/" owner "/" repo)
    }

    ListIssues(owner, repo, state := "open") {
        return this.Get("/repos/" owner "/" repo "/issues",
                        Map("state", state, "per_page", "30"))
    }

    CreateIssue(owner, repo, title, body := "") {
        return this.Post("/repos/" owner "/" repo "/issues",
                         Map("title", title, "body", body))
    }
}

; ── Usage ─────────────────────────────────────────────────────────────────────
try {
    gh   := GitHubClient("ghp_your_token_here")
    repo := gh.GetRepo("microsoft", "vscode")

    ; ✓ repo is a Jxon-parsed Map — access with string keys
    MsgBox("Stars: " repo["stargazers_count"]
        "`nForks: " repo["forks_count"]
        "`nLanguage: " repo["language"])

    ; ✓ List open issues — returns Array of Maps; 1-based AHK v2 Array indexing
    issues := gh.ListIssues("microsoft", "vscode")
    ; issues[1] = first issue — AHK v2 Arrays are 1-based
    ; issues[0]               ; ✗ → throws IndexError
    if (issues.Length > 0)
        MsgBox("First issue: " issues[1]["title"])

} catch Error as e {
    MsgBox("GitHub API error: " e.Message "`n" e.Extra, "API Error", 16)
}
```

## ANTI-PATTERNS

| Pattern | Wrong | Correct | LLM Common Cause |
|---------|-------|---------|------------------|
| v1 COM instantiation | `ComObjCreate("WinHttp.WinHttpRequest.5.1")` | `ComObject("WinHttp.WinHttpRequest.5.1")` | AHK v1 used `ComObjCreate()`; strong v1 training-data bias causes automatic regression |
| Using Msxml2 ProgID | `ComObject("Msxml2.ServerXMLHTTP.6.0")` | `ComObject("WinHttp.WinHttpRequest.5.1")` | LLM training data contains many Msxml2 examples from early AHK forum posts; WinHttp is the native-OS-preferred alternative |
| Non-zero ResolveTimeout | `xhr.setTimeouts(5000, 10000, 30000, 30000)` | `xhr.setTimeouts(0, 10000, 30000, 30000)` | LLM fills all 4 params symmetrically; Microsoft KB documenting the DNS thread handle leak is not in most training data |
| Blocking GUI with open(false) | `xhr.open("GET", url, false)` in GUI script | `xhr.open("GET", url, true)` + `xhr.waitForResponse()` | Model applies synchronous pattern from non-GUI examples; doesn't know `open(false)` freezes the message loop |
| Polling readyState on WinHttp | `if (xhr.readyState == 4)` | `if xhr.waitForResponse(0)` | LLM conflates Msxml2.ServerXMLHTTP.6.0 `readyState` API with WinHttp — readyState does not exist on WinHttp |
| Fabricating native JSON parser | `data := JSON.Parse(xhr.responseText)` | `data := Jxon.Load(text)` after `#Include Jxon.ahk` | Most languages ship with native JSON; model applies cross-language habit, unaware AHK v2 stdlib has none |
| Fabricating JSON serializer | `payload := JSON.Stringify(dataMap)` | `Jxon.Dump(dataMap)` or `MapToJsonFlat(dataMap)` | Same root cause — JSON.stringify is universal in JS/Python training data |
| Skipping HTTP status check | `return xhr.responseText` after `send()` | `if (xhr.status == 200) return xhr.responseText` | Many HTTP libs throw on 4xx/5xx; model assumes WinHttp does the same — it does not |
| ComObjConnect on WinHttp | `ComObjConnect(xhr, sink)` for async events | `open(true)` + `WaitForResponse()` or `SetTimer` + `WaitForResponse(0)` | Model applies Dispatch-based COM pattern; WinHttp uses IUnknown-based `IWinHttpRequestEvent` which is incompatible |
| Reference cycle in timer BoundFunc | `this._pollFn := this._Poll.Bind(this)` without later nullifying | After `SetTimer(this._pollFn, 0)`, immediately `this._pollFn := unset` | AHK v2 reference-counting GC with no cycle detection is non-obvious; model omits the unset step, creating a permanent memory leak |
| Async COM in local variable | `xhr := ComObject(...)` inside function with `open(true)` | `global g_xhr := ComObject(...)` at script scope | COM object lifetime tied to AHK variable scope is non-obvious; model assumes COM stays alive like a regular object |
| Unescaped user input in JSON | `'{"name":"' userInput '"}'` with no escaping | Run `userInput` through `JsonEscape()` before embedding | Most modern languages handle JSON encoding automatically; model omits manual escape step |

## SEE ALSO

> This module does NOT cover: COM automation beyond WinHttp.WinHttpRequest.5.1 (IStream direct manipulation, WinInet, WebSocket COM, shell COM, WMI) → see Module_SystemAndCOM.md
> This module does NOT cover: try/catch patterns for COM exceptions, OSError handling, and structured error recovery → see Module_Errors.md
> This module does NOT cover: class hierarchy design, `__New` patterns, and inheritance for API client subclasses → see Module_Classes.md

- `Module_Errors.md` — try/catch wrapping for COM exceptions (`OSError` on DNS failure, connection refused, timeout), structured error recovery, and re-throw patterns for network operations.
- `Module_Classes.md` — class design principles, `__New` constructor patterns, inheritance (`extends RestClient`), method overriding for `RefreshToken()`, and reference cycle avoidance patterns with BoundFunc.
- `Module_SystemAndCOM.md` — COM automation beyond WinHttp.WinHttpRequest.5.1, IStream-based binary streaming, WinInet, shell COM objects, WMI queries, and low-level Windows API interaction via DllCall.
- `Module_DataStructures.md` — Map and Array patterns for storing parsed JSON response fields, iterating Jxon-returned collections, and safe key access on nested response structures.