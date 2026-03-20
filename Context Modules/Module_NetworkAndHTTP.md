# Module_NetworkAndHTTP.md
<!-- DOMAIN: Network Requests and HTTP — Msxml2.ServerXMLHTTP.6.0 COM, file download, JSON handling, REST API clients -->
<!-- SCOPE: COM automation beyond Msxml2.ServerXMLHTTP.6.0 (IStream, WebSocket COM, WinInet), OAuth multi-step flows, TLS certificate management, and raw socket I/O are not covered — see Module_SystemAndCOM.md -->
<!-- TRIGGERS: Msxml2.ServerXMLHTTP, ServerXMLHTTP, ComObject HTTP, Download(), HttpGet, HttpPost, REST API, JSON parse, JSON stringify, "HTTP request", "fetch URL", "web request", "API call", "async HTTP", "Bearer token", "API client", responseText, setRequestHeader, Jxon, "network request", "rate limit", "retry request", readyState -->
<!-- CONSTRAINTS: Use ComObject("Msxml2.ServerXMLHTTP.6.0") — ComObjCreate() does not exist in AHK v2 and WinHttp.WinHttpRequest.5.1 is the older less-mature alternative. AHK v2 has NO native JSON parser — never generate JSON.Parse() or JSON.Stringify(); use Jxon library or RegExMatch. Always call setTimeouts() before send() and check xhr.status before reading xhr.responseText. Msxml2.ServerXMLHTTP.6.0 does NOT support ComObjConnect event sinks — use SetTimer readyState polling for async patterns. Download() throws OSError on failure (network-level error or disk write error); HTTP 4xx/5xx responses are NOT exceptions — the response body is silently written to the destination file. -->
<!-- CROSS-REF: Module_Errors.md, Module_Classes.md, Module_SystemAndCOM.md -->
<!-- VERSION: AHK v2.0+ -->

## V1 → V2 BREAKING CHANGES

| v1 pattern (LLM commonly writes) | v2 correct form | Consequence |
|----------------------------------|-----------------|-------------|
| `ComObjCreate("Msxml2.ServerXMLHTTP.6.0")` | `ComObject("Msxml2.ServerXMLHTTP.6.0")` | MethodError at runtime — `ComObjCreate()` was removed in AHK v2; no HTTP session is created |
| `ComObject("WinHttp.WinHttpRequest.5.1")` | `ComObject("Msxml2.ServerXMLHTTP.6.0")` | Works but uses the older, less-mature COM server — Msxml2.ServerXMLHTTP.6.0 has better proxy support, stricter RFC compliance, and is the recommended replacement |
| `JSON.Parse(xhr.responseText)` | `Jxon.Load(xhr.responseText)` after `#Include Jxon.ahk` | MethodError — no native JSON parser exists in AHK v2 standard library |
| `JSON.Stringify(obj)` or `JSON.Dump(obj)` | `Jxon.Dump(obj)` or custom `MapToJsonFlat()` | MethodError — same root cause; agent must use community library or manual serializer |
| `data["items"][0]` on Jxon-returned Array | `data["items"][1]` | Silent wrong data — AHK v2 Arrays are 1-based; index 0 returns empty |
| `xhr.open(...)` then `xhr.send()` with no `xhr.setTimeouts()` | `xhr.setTimeouts(5000, 10000, 30000, 30000)` called before `open()` | Indefinite hang on any DNS failure, slow server, or network drop — Msxml2 default timeout is infinite |
| Reading `xhr.responseText` immediately after `send()` without status check | `if (xhr.status == 200) return xhr.responseText` | Silent wrong data or empty string — Msxml2.ServerXMLHTTP.6.0 does not throw on HTTP 4xx/5xx; caller must inspect `.status` |
| `ComObjConnect(xhr, sink)` for async events | `SetTimer(pollFn, 50)` + poll `xhr.readyState == 4` | ComObjConnect event sinks are WinHttp-specific — they do not fire on Msxml2.ServerXMLHTTP.6.0; async requires readyState polling |
| Multi-statement arrow function as async callback: `(s, b) => { MsgBox(s) ... }` | Define a named function: `HandleResponse(s, b) { MsgBox(s) }` | SyntaxError — AHK v2 arrow functions are single-expression only; callbacks with multiple statements require named functions |

## API QUICK-REFERENCE

### Download (built-in)
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `Download()` | `Download(url, destPath)` | Fully synchronous — blocks AHK thread for entire duration; throws `OSError` on failure (network-level error or disk write error); HTTP 4xx/5xx responses are NOT exceptions — the response body is silently written to the destination file |

### Msxml2.ServerXMLHTTP.6.0 COM Object
| Method/Property | Signature | Notes |
|----------------|-----------|-------|
| `ComObject()` | `ComObject("Msxml2.ServerXMLHTTP.6.0")` | Creates HTTP session — preferred over `WinHttp.WinHttpRequest.5.1`; better proxy support and RFC compliance |
| `.setTimeouts()` | `.setTimeouts(resolve, connect, send, receive)` | All values in milliseconds; must be called before `open()`; omitting leaves infinite wait |
| `.open()` | `.open(method, url, async)` | `async=false` = blocking; `async=true` = non-blocking, poll `readyState` for completion |
| `.setRequestHeader()` | `.setRequestHeader(name, value)` | Must be called after `open()` and before `send()` |
| `.send()` | `.send(body?)` | Accepts string or empty for GET/HEAD; returns immediately in async mode |
| `.abort()` | `.abort()` | Cancels an in-flight async request and resets `readyState` to 0 |
| `.status` | `.status` (read-only) | HTTP status integer (200, 404, etc.); never auto-throws on 4xx/5xx |
| `.responseText` | `.responseText` (read-only) | Response body as string; check `.status` first — returns empty string on HTTP error |
| `.statusText` | `.statusText` (read-only) | HTTP reason phrase (e.g., `"Not Found"`) |
| `.readyState` | `.readyState` (read-only) | 0=Uninitialised, 1=Open, 2=Sent, 3=Receiving, 4=Done — poll this for async completion |
| `.getAllResponseHeaders()` | `.getAllResponseHeaders()` | Returns all response headers as a single CRLF-delimited string |
| `.getResponseHeader()` | `.getResponseHeader(name)` | Returns one named response header value |

### Msxml2.ServerXMLHTTP.6.0 — Why Over WinHttp.WinHttpRequest.5.1
| Aspect | WinHttp.WinHttpRequest.5.1 | Msxml2.ServerXMLHTTP.6.0 |
|--------|---------------------------|---------------------------|
| Maturity | Older, less maintained | Actively maintained, part of MSXML6 |
| Proxy support | Basic | Full WPAD, WinHTTP proxy config |
| RFC compliance | Partial | Stricter, more consistent |
| Async mechanism | `ComObjConnect()` event sink | `readyState` polling via `SetTimer` |
| `readyState` property | Not available | 0–4 lifecycle states |
| `abort()` | Not available | Cancels in-flight async request |

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

- Use `ComObject("Msxml2.ServerXMLHTTP.6.0")` for every HTTP session — `ComObjCreate()` was removed in AHK v2 and throws `MethodError` immediately, producing no HTTP session and no diagnostic message beyond the exception. `WinHttp.WinHttpRequest.5.1` also works but is the older less-maintained alternative.
- AHK v2 ships with no JSON parser — never generate `JSON.Parse()`, `JSON.Stringify()`, `JSON.Load()`, or any similar built-in form; use the `Jxon` or `cjson` community library (`#Include Jxon.ahk`), or use `RegExMatch` named-capture patterns for known single-field extraction from flat JSON only.
- Always call `xhr.setTimeouts(resolve, connect, send, receive)` before `xhr.open()` on every Msxml2.ServerXMLHTTP.6.0 instance — the default timeout is infinite, so any DNS failure, connection refusal, or slow server causes the AHK script to hang permanently with no error.
- Always check `xhr.status` after synchronous `send()` before reading `xhr.responseText` — Msxml2.ServerXMLHTTP.6.0 does not throw on HTTP 4xx/5xx responses; accessing `responseText` on a failed request silently returns an empty string or error body.
- `Msxml2.ServerXMLHTTP.6.0` does NOT support `ComObjConnect()` event sinks — this pattern is WinHttp-specific. For async requests, use `xhr.open(method, url, true)` + `xhr.send()`, then poll `xhr.readyState == 4` with `SetTimer`.
- Store async `Msxml2.ServerXMLHTTP.6.0` COM objects in script-scope (global or static) variables — a local-variable COM object goes out of scope when the enclosing function returns, triggering garbage collection and silently cancelling the in-flight request before `readyState` reaches 4.
- Escape all user-supplied strings with `JsonEscape()` before embedding in JSON payloads — unescaped `"` and `\` produce malformed JSON silently; the COM object sends the malformed body without error.
- ✗ `xhr := ComObjCreate("Msxml2.ServerXMLHTTP.6.0")` — MethodError, no session
- ✓ `xhr := ComObject("Msxml2.ServerXMLHTTP.6.0")` — correct v2 COM instantiation
- ✗ `data := JSON.Parse(xhr.responseText)` — MethodError, no native parser
- ✓ `data := Jxon.Load(xhr.responseText)` — community library after `#Include Jxon.ahk`
- ✗ `xhr.send()` without prior `xhr.setTimeouts(...)` — indefinite hang on network failure
- ✓ `xhr.setTimeouts(5000, 10000, 30000, 30000)` before every `open()`/`send()`
- ✗ `return xhr.responseText` immediately after `send()` — silent empty string on 4xx/5xx
- ✓ `if (xhr.status == 200) return xhr.responseText` — always validate status first
- ✗ `ComObjConnect(xhr, sink)` — silently does nothing on Msxml2.ServerXMLHTTP.6.0
- ✓ `SetTimer(pollFn, 50)` + poll `xhr.readyState == 4` — correct async pattern

Safe-access priority order for network responses:
  1. `Download(url, path)` in `try/catch` — complete file fetches, single-call, auto-close
  2. `ComObject("Msxml2.ServerXMLHTTP.6.0")` + synchronous `open(false)` — partial reads, POST, custom headers, status inspection
  3. `open(true)` async + `SetTimer` readyState polling — only when non-blocking behavior is explicitly required
  4. `SetTimer` + synchronous GET loop (`ApiPoller`) — periodic polling; simpler than async, no COM lifetime risk

## TIER 1 — Basic File Download
> METHODS COVERED: Download · FileExist · DirExist · DirCreate · SplitPath

`Download()` is AHK v2's built-in single-function file fetcher. It accepts a URL and a local destination path, executes synchronously on the calling thread for its entire duration, and throws an `OSError` on network-level failure (DNS, timeout, refused connection) or disk write failure. HTTP 4xx/5xx responses are not exceptions — the response body is silently saved to the destination path. All TIER 1 usage must wrap `Download()` in `try/catch` and verify the file exists after the call. Never use `Download()` for large files in GUI contexts — the AHK thread blocks completely until the transfer finishes.
```ahk
; ── TIER 1: Download() with error handling ────────────────────────────────────
; <!-- CONVERTED: changed fence language identifier from cpp to ahk -->
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
        MsgBox("Network error: " e.Message "`nCode: " e.Extra, "Download Failed", 16)
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
; Download("https://example.com/data.csv", "data.csv")   ; → uncaught Error on 404/timeout

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

A synchronous GET request uses `ComObject("Msxml2.ServerXMLHTTP.6.0")` with `open()` in blocking mode (`async=false`), then reads `xhr.responseText` after `send()` completes. This tier establishes the canonical baseline for all higher tiers: `ComObject()` instantiation, timeout configuration before `open()`, HTTP status validation before `responseText`, and COM exception handling. Every Msxml2.ServerXMLHTTP.6.0 session in AHK v2 must follow this initialization sequence — `ComObject()` → `setTimeouts()` → `open()` → `setRequestHeader()` calls → `send()` → check `.status` → read `.responseText`.
```ahk
; ── TIER 2: Synchronous GET request ──────────────────────────────────────────
#Requires AutoHotkey v2.0

HttpGet(url) {
    ; ✓ Correct AHK v2 COM instantiation — preferred over WinHttp.WinHttpRequest.5.1
    xhr := ComObject("Msxml2.ServerXMLHTTP.6.0")

    ; ✗ Wrong — ComObjCreate() was removed; produces MethodError immediately
    ; xhr := ComObjCreate("Msxml2.ServerXMLHTTP.6.0")   ; → MethodError

    ; ✓ Always configure timeouts before open/send — default is infinite wait
    ; Params: resolve, connect, send, receive (milliseconds)
    xhr.setTimeouts(5000, 10000, 30000, 30000)

    try {
        ; false = synchronous (blocking) — script waits here until response arrives
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
> METHODS COVERED: .setRequestHeader · .getAllResponseHeaders · BasicAuthHeader · HttpSend · MapToJsonFlat · JsonEscape · PostJsonWithBearer · StrJoin

POST requests extend the TIER 2 baseline by adding `setRequestHeader()` calls for `Content-Type`, `Authorization`, and any custom headers, then passing a string payload to `send()`. This tier also covers Bearer token injection, Basic Auth construction via `CryptBinaryToStringW`, and the `PUT`/`PATCH`/`DELETE` method variants that follow identical patterns. The `HttpSend()` function in this tier is the general-purpose multi-method sender that all higher tiers build on.
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
    xhr := ComObject("Msxml2.ServerXMLHTTP.6.0")
    xhr.setTimeouts(5000, 10000, 30000, 30000)

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
;     ; first := data["items"][0]       ; ✗ returns empty — AHK Arrays are 1-based
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

## TIER 5 — Asynchronous HTTP Requests
> METHODS COVERED: .open (async=true) · .send · .readyState · .abort · SetTimer · AsyncGet · PollReady · ApiPoller · .Start · .Stop · .Poll

`Msxml2.ServerXMLHTTP.6.0` supports async mode by passing `true` as the third argument to `open()`. Unlike `WinHttp.WinHttpRequest.5.1`, it does **not** support `ComObjConnect()` event sinks — assigning a `ComObjConnect` sink to an Msxml2.ServerXMLHTTP.6.0 object silently does nothing. The correct async pattern is: call `open(method, url, true)` + `send()`, then poll `xhr.readyState == 4` (Done) with `SetTimer`. The COM object must be stored at script scope — a local-variable COM object is garbage-collected when the function returns, cancelling the in-flight request silently.

There are two practical async patterns. **Pattern A (single async request):** store the COM object globally, send with `open(true)`, poll `readyState` with a 50 ms `SetTimer`. **Pattern B (periodic polling):** use `SetTimer` + synchronous `open(false)` in a class — simpler, no COM lifetime management, and the preferred choice for periodic API checks.
```ahk
; ── TIER 5A: Async open + readyState polling ─────────────────────────────────
#Requires AutoHotkey v2.0
#SingleInstance Force

; ✓ Script-scope variables keep the COM object alive for the async duration
; ✗ Wrong: local variable inside AsyncGet() — GC'd when function returns, readyState
;   never reaches 4 and the callback never fires
global g_xhr    := unset
global g_pollFn := unset

AsyncGet(url, onComplete) {
    global g_xhr, g_pollFn

    ; ✓ Correct: Msxml2.ServerXMLHTTP.6.0 — async-capable and more mature than WinHttp
    g_xhr := ComObject("Msxml2.ServerXMLHTTP.6.0")
    g_xhr.setTimeouts(5000, 10000, 30000, 30000)

    ; true = async — send() returns immediately; poll readyState for completion
    g_xhr.open("GET", url, true)
    g_xhr.send()

    ; ✗ Wrong: ComObjConnect(g_xhr, sink) — silently does nothing on Msxml2.ServerXMLHTTP.6.0
    ; ✓ Correct: SetTimer polling on readyState
    g_pollFn := PollReady.Bind(onComplete)
    SetTimer(g_pollFn, 50)   ; check every 50 ms
}

; Polls until readyState == 4 (Done), then calls the callback and stops the timer
PollReady(callback) {
    global g_xhr, g_pollFn

    if (g_xhr.readyState != 4)
        return   ; still in progress — timer fires again in 50 ms

    ; ✓ Stop polling before invoking callback
    SetTimer(g_pollFn, 0)

    status := g_xhr.status
    body   := (status == 200) ? g_xhr.responseText : ""

    ; Callback must be a Func or BoundFunc — named function, not multi-statement arrow
    callback.Call(status, body)
}

; ── Abort an in-flight async request ─────────────────────────────────────────
AbortAsync() {
    global g_xhr, g_pollFn
    if IsSet(g_pollFn)
        SetTimer(g_pollFn, 0)   ; stop polling first
    if IsSet(g_xhr) && (g_xhr.readyState != 0)
        g_xhr.abort()           ; abort() cancels the request and resets readyState to 0
}

; ── TIER 5B: SetTimer polling pattern (RECOMMENDED for periodic checks) ───────
; ✓ For periodic API checks, prefer SetTimer + synchronous open(false):
;   simpler than async open, no COM lifetime risk, easier to debug

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
            xhr := ComObject("Msxml2.ServerXMLHTTP.6.0")
            xhr.setTimeouts(3000, 5000, 10000, 10000)
            xhr.open("GET", this.url, false)
            xhr.send()
            if (xhr.status == 200)
                this.onData.Call(xhr.responseText)
        } catch Error as e {
            ; Silent fail on polling interval — log if needed
        }
    }
}

; ── Usage ─────────────────────────────────────────────────────────────────────
; Pattern A: single async GET — callback is a named function (cannot be arrow function)
HandleResponse(status, body) {
    MsgBox("Status: " status "`nBody: " SubStr(body, 1, 200))
}

AsyncGet("https://httpbin.org/get", HandleResponse)

; Pattern B: polling — check every 60 seconds
HandlePollData(body) {
    value := ExtractJsonField(body, "timestamp")
    ToolTip("Latest: " value)
}

poller := ApiPoller("https://api.example.com/status", 60000, HandlePollData)
poller.Start()
```

### Performance Notes

Network I/O is orders of magnitude slower than in-memory operations. Apply these principles to avoid the most common AHK v2 network performance pitfalls:

**Request batching:** Avoid one HTTP call per loop iteration. A loop of N items makes N round-trips. When the API supports batch or bulk endpoints, compose a single request with comma-joined IDs or an array body. The per-request overhead (DNS + TCP + TLS handshake) dwarfs the payload transfer time for most APIs.

**Timeout sizing:** Match timeout values to the expected response characteristics. Use short timeouts (2–3 s resolve/connect) for LAN or fast APIs to fail fast; use longer receive timeouts (60–300 s) only for file downloads or endpoints known to return large payloads. Oversized timeouts mask connectivity problems.

**COM object reuse in tight loops:** Creating a new `ComObject("Msxml2.ServerXMLHTTP.6.0")` for every request in a high-frequency polling loop is wasteful. Reuse a single instance — `open()` resets the request state and can be called multiple times on the same COM object without re-instantiation.

**JSON parsing cost:** `Jxon.Load()` parses the entire JSON tree. In hot polling loops where only one field is needed, `RegExMatch` named-capture on a known flat field is significantly faster. Pay the `Jxon` cost only when you need multi-field or nested access.

**Rate limiting with Sleep:** Insert `Sleep(delayMs)` between requests in batch loops to avoid triggering HTTP 429 responses. A 500 ms inter-request delay is a reasonable default for public APIs without documented rate limits.

**SetTimer vs async open for polling:** For simple periodic API checks, `SetTimer` + synchronous `open(false)` (`ApiPoller`) is faster to implement, easier to debug, and eliminates async COM lifetime management. Reserve async `open(true)` + `readyState` polling for cases where the script must remain responsive during a long single request.

## TIER 6 — Full REST API Client Class
> METHODS COVERED: RestClient.__New · RestClient.Get · RestClient.Post · RestClient.Put · RestClient.Delete · RestClient.SetToken · RestClient.IsTokenValid · RestClient.RefreshToken · RestClient.BuildUrl · RestClient.Execute · RestClient.SendRequest · RestClient.ParseResponse · RestClient.UriEncode · RestClient.JoinStr · GitHubClient.__New · GitHubClient.GetRepo · GitHubClient.ListIssues · GitHubClient.CreateIssue · Jxon.Load · Jxon.Dump

A production REST API client encapsulates base URL management, token storage with expiry tracking, automatic retry with exponential backoff, and dynamic endpoint composition into a single reusable class. This tier integrates all lower-tier patterns and implements the robustness requirements for real-world API integrations. `#Include Jxon.ahk` is required at this tier — `Jxon.Dump()` is used for request serialization and `Jxon.Load()` for response parsing. When subclassing `RestClient`, override `RefreshToken()` to implement token renewal without modifying the base retry loop.
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

    ; ── Low-level Msxml2.ServerXMLHTTP.6.0 sender ────────────────────────────
    SendRequest(method, url, payload) {
        xhr := ComObject("Msxml2.ServerXMLHTTP.6.0")
        xhr.setTimeouts(5000, 10000, 30000, 30000)
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
    ; issues[0]               ; ✗ → empty, not the first element
    if (issues.Length > 0)
        MsgBox("First issue: " issues[1]["title"])

} catch Error as e {
    MsgBox("GitHub API error: " e.Message "`n" e.Extra, "API Error", 16)
}
```

## ANTI-PATTERNS

| Pattern | Wrong | Correct | LLM Common Cause |
|---------|-------|---------|------------------|
| v1 COM instantiation | `ComObjCreate("Msxml2.ServerXMLHTTP.6.0")` | `ComObject("Msxml2.ServerXMLHTTP.6.0")` | AHK v1 used `ComObjCreate()`; strong v1 training-data bias causes automatic regression |
| Using older WinHttp ProgID | `ComObject("WinHttp.WinHttpRequest.5.1")` | `ComObject("Msxml2.ServerXMLHTTP.6.0")` | Model defaults to the first ProgID it encountered in training data; Msxml2 is the mature replacement |
| Fabricating native JSON parser | `data := JSON.Parse(xhr.responseText)` | `data := Jxon.Load(text)` after `#Include Jxon.ahk` | Most languages ship with native JSON; model applies cross-language habit, unaware AHK v2 stdlib has none |
| Fabricating JSON serializer | `payload := JSON.Stringify(dataMap)` | `Jxon.Dump(dataMap)` or `MapToJsonFlat(dataMap)` | Same root cause — JSON.stringify is universal in JS/Python training data |
| Skipping HTTP status check | `return xhr.responseText` immediately after `send()` | `if (xhr.status == 200) return xhr.responseText` | Many HTTP libs throw on 4xx/5xx; model assumes Msxml2 does the same — it does not |
| Omitting setTimeouts | `xhr.open(...)` then `xhr.send()` with no timeout config | `xhr.setTimeouts(5000, 10000, 30000, 30000)` before `open()` | Most HTTP clients have finite default timeouts; Msxml2 default is infinite — model doesn't know this |
| ComObjConnect on Msxml2 | `ComObjConnect(xhr, sink)` for async events | `SetTimer(pollFn, 50)` + poll `xhr.readyState == 4` | Model learns ComObjConnect from WinHttp examples; incorrectly applies it to Msxml2.ServerXMLHTTP.6.0 where it silently does nothing |
| Async COM in local variable | `xhr := ComObject(...)` inside function with async open | `global g_xhr := ComObject(...)` at script scope | COM object lifetime tied to AHK variable scope is non-obvious; model assumes COM stays alive like a regular object |
| Unescaped user input in JSON | `'{"name":"' userInput '"}'` with no escaping | Run `userInput` through `JsonEscape()` before embedding | Most modern languages handle JSON encoding automatically; model omits manual escape step |

## SEE ALSO

> This module does NOT cover: COM automation beyond Msxml2.ServerXMLHTTP.6.0 (IStream, WinInet, WebSocket COM, shell COM, WMI) → see Module_SystemAndCOM.md
> This module does NOT cover: try/catch patterns for COM exceptions, OSError handling, and structured error recovery → see Module_Errors.md
> This module does NOT cover: class hierarchy design, `__New` patterns, and inheritance for API client subclasses → see Module_Classes.md

- `Module_Errors.md` — try/catch wrapping for COM exceptions (`OSError` on DNS failure, connection refused, timeout), structured error recovery, and re-throw patterns for network operations.
- `Module_Classes.md` — class design principles, `__New` constructor patterns, inheritance (`extends RestClient`), and method overriding for `RefreshToken()` implementations.
- `Module_SystemAndCOM.md` — COM automation beyond Msxml2.ServerXMLHTTP.6.0, IStream-based binary streaming, WinInet, shell COM objects, WMI queries, and low-level Windows API interaction via DllCall.
- `Module_DataStructures.md` — Map and Array patterns for storing parsed JSON response fields, iterating Jxon-returned collections, and safe key access on nested response structures.