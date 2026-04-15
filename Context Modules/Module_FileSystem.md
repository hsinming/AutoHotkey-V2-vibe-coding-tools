# Module_FileSystem.md
<!-- DOMAIN: File System -->
<!-- SCOPE: Async I/O via IStream COM, file attribute manipulation (FileGetAttrib/FileSetAttrib/FileSetTime/FileGetTime), shortcut management (FileCreateShortcut/FileGetShortcut), and string transformation of file content after reading are not covered — see Module_SystemAndCOM.md and Module_TextProcessing.md. -->
<!-- TRIGGERS: FileOpen, FileRead, FileAppend, FileExist, DirExist, SplitPath, DirCreate, DirDelete, FileCopy, FileMove, FileDelete, FileGetSize, Loop Files, Loop Read, IniRead, IniWrite, FileRecycle, "read file", "write file", "parse csv", "check if file exists", "get file extension", "binary file", "create folder", "log data repeatedly", "stream", "text file", "directory traversal", "ini config" -->
<!-- CONSTRAINTS: Always specify encoding on every FileRead/FileAppend/FileOpen call for text files — omitting it silently corrupts non-ASCII data via system-locale fallback. FileOpen() throws OSError on failure in AHK v2 — never check `if !f`; wrap in try/catch OSError. Never use FileAppend inside a loop — open a FileOpen handle before the loop and close it in finally; AHK v2 __Delete timing is non-deterministic and cannot substitute for explicit Close(). Binary files use FileOpen without an encoding argument and RawRead/RawWrite; FileRead() with no RAW option corrupts binary data. -->
<!-- CROSS-REF: Module_Errors.md, Module_Arrays.md, Module_TextProcessing.md, Module_SystemAndCOM.md -->
<!-- VERSION: AHK v2.0+ -->

## V1 → V2 BREAKING CHANGES

| v1 pattern (LLM commonly writes) | v2 correct form | Consequence |
|----------------------------------|-----------------|-------------|
| `FileRead, var, path` (command syntax, no encoding) | `var := FileRead(path, "UTF-8")` | SyntaxError — v2 has no command mode; LLMs trained on mixed-version AHK omit parentheses and assignment form |
| `FileAppend, text, path` (command form, no encoding) | `FileAppend(text, path, "UTF-8")` | SyntaxError + silent data corruption on any non-ASCII character |
| `if !f` after `FileOpen()` to catch failure | `try { f := FileOpen(...) } catch OSError` | Logic error — FileOpen() throws OSError on failure in v2; returning 0 was v1 behavior; `if !f` guard never fires but misleads readers about the error contract |
| `FileAppend(text, path)` inside a loop | `f := FileOpen(path, "a", "UTF-8")` before loop; `f.WriteLine(text)` inside loop; `f.Close()` in finally | Repeated open/write/close per iteration — severe disk thrashing at scale |
| `FileRead(path)` without encoding argument | `FileRead(path, "UTF-8")` | Silent corruption — system locale encoding used; breaks on é ü ñ and CJK characters |
| `if FileExist(path)` as bare boolean | `if FileExist(path) != ""` | Logic error — FileExist returns an attribute string ("A", "D", "H", etc.); bare truthy test passes for directories when a file is expected |
| `FileReadLine, var, path, N` (single-line read by index) | `Loop Read, path { if A_Index = N { line := A_LoopReadLine } }` | No direct single-line-by-index API in v2; LLMs trained on v1 hallucinate a command with no v2 equivalent |
| No `file.Close()` after `FileOpen` | Explicit `.Close()` in a `finally` block | File handle leak — v1 auto-closed handles on script exit; v2 `__Delete` timing is non-deterministic and does not guarantee cleanup on exception paths |
| `FileRead(path)` on a binary file | `FileRead(path, "RAW")` → returns Buffer object; or `FileOpen(path, "r")` + `.RawRead(buf)` | FileRead without RAW decodes bytes as text — corrupts binary data irrecoverably |

## API QUICK-REFERENCE

### Standalone Read/Write Functions

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `FileRead()` | `FileRead(path, options?)` | String (or Buffer if RAW) | `OSError` | options: encoding name, `"RAW"`, `"m1024"` (size cap), `` "`n" `` (EOL translate); text only unless RAW specified |
| `FileAppend()` | `FileAppend(text, path, options?)` | — | `OSError` | Creates file if absent; avoid inside loops — use FileOpen instead; accepts Buffer for binary append |
| `FileExist()` | `FileExist(path)` | Attribute string or `""` | — | Returns `""` when absent — **not a boolean**; test with `!= ""`; returns `"D"` for directories |
| `DirExist()` | `DirExist(path)` | Attribute string or `""` | — | Returns `""` when absent; `!DirExist(p)` is equivalent to `DirExist(p) == ""`; never throws |
| `FileGetSize()` | `FileGetSize(path?, units?)` | Integer | `OSError` | units: `"B"` (default), `"K"`, `"M"`; omit path inside Loop Files to use current file |

### File Mutation Functions

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `FileCopy()` | `FileCopy(source, dest, overwrite?)` | — | `OSError` | overwrite: 0 = no (default), 1 = yes; throws if source absent (no wildcard) or dest exists with overwrite=0 |
| `FileDelete()` | `FileDelete(pattern)` | — | `OSError` | Supports wildcards; throws if file is locked or absent |
| `FileMove()` | `FileMove(source, dest, overwrite?)` | — | `OSError` | overwrite: 0 = no, 1 = yes; prefer over FileCopy + FileDelete for atomic rename |
| `FileRecycle()` | `FileRecycle(pattern)` | — | `OSError` | Sends file to Recycle Bin; falls back to permanent delete if recycle not possible |

### Directory Functions

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `DirCreate()` | `DirCreate(path)` | — | `OSError` | Creates all missing intermediate directories automatically |
| `DirDelete()` | `DirDelete(path, recurse?)` | — | `OSError` | recurse: 0 = no (default), 1 = yes — non-empty directories require recurse = 1 |
| `DirCopy()` | `DirCopy(source, dest, overwrite?)` | — | `OSError` | Copies entire directory tree |
| `DirMove()` | `DirMove(source, dest, flag?)` | — | `OSError` | Moves or renames a directory; flag: 0 = no overwrite, 1 = overwrite |

### Path Functions

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `SplitPath()` | `SplitPath(path, &name?, &dir?, &ext?, &nameNoExt?, &drive?)` | — | — | All output params optional ByRef; always prefer over manual string parsing; handles UNC, drive-only, extension-less paths |

### FileOpen Handle (File Object)

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `FileOpen()` | `FileOpen(path, flags, encoding?)` | File object | `OSError` | Throws on failure in v2 (not returns 0); flags: `"r"` read, `"w"` write, `"a"` append, `"rw"` read+write; omit encoding for binary |
| `File.Read()` | `File.Read(chars?)` | String | — | Reads up to chars chars; omit to read all remaining content |
| `File.ReadLine()` | `File.ReadLine()` | String | — | Reads one line; strips the line ending; supports `\`r`, `\`n`, `\`r\`n` |
| `File.ReadUInt()` | `File.ReadUInt()` | Integer | — | Reads a 32-bit unsigned integer at the current position |
| `File.Write()` | `File.Write(str)` | Integer (bytes written) | — | Writes string at current position |
| `File.WriteLine()` | `File.WriteLine(str)` | Integer (bytes written) | — | Writes string followed by a newline |
| `File.WriteUInt()` | `File.WriteUInt(n)` | Integer (bytes written) | — | Writes a 32-bit unsigned integer at the current position |
| `File.RawRead()` | `File.RawRead(buffer, bytes?)` | Integer (bytes read) | — | Reads raw bytes into a Buffer object; bytes defaults to Buffer.Size |
| `File.RawWrite()` | `File.RawWrite(data, bytes?)` | Integer (bytes written) | — | Writes raw bytes from Buffer or string; bytes defaults to Buffer.Size |
| `File.Seek()` | `File.Seek(dist, origin?)` | 1 on success | — | origin: 0 = start, 1 = current, 2 = end; defaults to 0 when Distance ≥ 0, 2 when Distance < 0 |
| `File.Close()` | `File.Close()` | — | — | Must always be called explicitly — never rely on garbage collection |
| `File.Pos` | `File.Pos` (read/write) | Integer | — | Current byte position; assigning seeks to that position — equivalent to `.Seek(n, 0)` |
| `File.Length` | `File.Length` (read/write) | Integer | — | File length in bytes; assigning a smaller value truncates the file |
| `File.AtEOF` | `File.AtEOF` (read-only) | Non-zero at EOF | — | Returns non-zero when the file pointer is at end of file |
| `File.Encoding` | `File.Encoding` (read/write) | String | — | Encoding used by this handle; reassign to change the active codec mid-session |
| `File.Handle` | `File.Handle` (read-only) | Integer (Win32 handle) | — | Flushes write buffer and resets read buffer before returning; use with DllCall |

### INI Functions

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `IniRead()` | `IniRead(path, section, key, default?)` | String | `OSError` (if default omitted) | Returns default when key absent; omit key to retrieve entire section; omit both section and key to list section names |
| `IniWrite()` | `IniWrite(value, path, section, key)` | — | `OSError` | Creates file, section, and key if absent |
| `IniDelete()` | `IniDelete(path, section, key?)` | — | `OSError` | Omit key to delete entire section |

### Loop Constructs and Built-in Loop Variables

| Construct / Variable | Description | Notes |
|----------------------|-------------|-------|
| `Loop Files` | `Loop Files, pattern, mode?` | Mode flags: `"R"` = recurse subdirs, `"D"` = dirs only, `"F"` = files only; flags combinable |
| `A_LoopFilePath` | Full path of current file in Loop Files | Includes drive + directory + name + extension |
| `A_LoopFileName` | Name and extension of current file | No directory component |
| `A_LoopFileDir` | Directory of current file | No trailing backslash |
| `A_LoopFileSize` | Size in bytes of current file | Integer; usable for folder-size aggregation |
| `Loop Read` | `Loop Read, filePath, outputFile?` | Iterates all lines; optional outputFile routes FileAppend calls inside loop to that path |
| `A_LoopReadLine` | Current line text in Loop Read | Line ending stripped |
| `Loop Parse` | `Loop Parse, str, delimiters?, omitChars?` | Use `"CSV"` as delimiter for RFC-4180 quoted fields; cannot be applied directly to a file path |
| `A_LoopField` | Current field value in Loop Parse | |

### Buffer Object

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `Buffer()` | `Buffer(byteCount, fillByte?)` | Buffer object | — | Allocates raw memory for binary I/O; fillByte defaults to 0 |

### String/Array Helpers (cross-domain — used in file content processing tiers)

| Function | Signature | Returns | Throws | Notes |
|----------|-----------|---------|--------|-------|
| `StrSplit()` | `StrSplit(str, delimiters?, omitChars?)` | Array (1-based) | — | See Module_TextProcessing.md for full API |
| `InStr()` | `InStr(haystack, needle, caseSense?, startingPos?, occurrence?)` | Integer (1-based pos) or 0 | — | See Module_TextProcessing.md |
| `Array.Push()` | `arr.Push(val1, val2?, ...)` | — | — | Appends elements; see Module_Arrays.md for full Array API |

## AHK V2 CONSTRAINTS

- Always pass an explicit encoding to `FileRead()`, `FileAppend()`, and `FileOpen()` for all text files — omitting it silently falls back to the system locale (CP0), causing non-ASCII data corruption on any non-Western machine.
- **`FileOpen()` throws `OSError` on failure in AHK v2** — wrap in `try/catch OSError`, never check `if !f` or `if f = 0` as a post-call guard.
  - ✗ `f := FileOpen(path, "r", "UTF-8")` followed by `if !f` guard — guard never fires; the `if !f` pattern masks the real v1→v2 behavioral change
  - ✓ `try { f := FileOpen(path, "r", "UTF-8") } catch OSError as e { throw Error("open failed: " e.Message) }` — correct v2 error contract
- Use `FileAppend(text, path)` only for single write operations — inside loops, open a `FileOpen` handle before the loop and close it in `finally`. `FileAppend` incurs a full open/write/close disk cycle on each call.
  - ✗ `Loop 1000 { FileAppend("entry\`n", logPath, "UTF-8") }` — 1000 separate disk open/close operations
  - ✓ `f := FileOpen(logPath, "a", "UTF-8")` before loop; `f.WriteLine(text)` inside loop; `f.Close()` in `finally`
- Always call `file.Close()` in a `finally` block — AHK v2 `__Delete` is a fallback, not a guarantee; handle leaks are certain on exception paths without an explicit close.
- Use `SplitPath()` for all path decomposition — never split by `\`, `StrSplit`, or `RegExMatch`; `SplitPath` correctly handles UNC paths, drive-only paths, and extension-less files.
- ✗ `if FileExist(path)` — truthy for any non-empty attribute string, including `"D"` (directory); passes silently when path resolves to a directory.
- ✓ `if FileExist(path) != ""` — explicit string comparison; combine with `InStr(FileExist(path), "D") = 0` when the check must distinguish files from directories.
- Binary files require `FileOpen(path, mode)` without an encoding argument — AHK v2 has no `"b"` flag; binary access is achieved by omitting the encoding parameter. `FileRead()` without the `"RAW"` option applies text decoding that corrupts raw binary data irrecoverably. Use `FileRead(path, "RAW")` to get a Buffer object, or `FileOpen` + `RawRead/RawWrite`.
- Always anchor paths via `A_WorkingDir` or use absolute paths — relative paths resolve from the current working directory at runtime, which may shift if the script spawns external processes.
- Call `FileGetSize()` before `FileRead()` on potentially large files — `FileRead()` loads the entire file into a single string in RAM; loading a multi-gigabyte file without a size check will exhaust available memory.

Safe-access priority order for file operations:
  1. `FileRead(path, "UTF-8")` — complete text file reads; auto-opens and auto-closes in one call
  2. `FileOpen(path, mode, encoding?)` in `try/finally` — partial reads, writes, binary, or high-frequency loop operations
  3. `FileExist(path) != ""` check before `FileOpen` — when absent-file handling must differ structurally from I/O error handling
  4. `try/catch OSError` around `FileOpen` or other mutation calls — when the error message carries diagnostic information needed for logging or user feedback

Unset variable handling: when using multiple `FileOpen` calls in sequence, initialize all handle variables to `""` before `try` — this allows `if handle` checks in the `catch` block to distinguish which handles were successfully opened when the error occurred mid-sequence.

Resource lifecycle: every `FileOpen()` call must have a corresponding `f.Close()` in a `finally` block because `__Delete` timing is not guaranteed in AHK v2.

## AGENT QA CHECKLIST

- [ ] Did I wrap every `FileOpen()` call in `try/catch OSError` instead of checking `if !f` after the call?
- [ ] Did I specify an explicit encoding parameter on every `FileRead()`, `FileAppend()`, and text-mode `FileOpen()` call?
- [ ] Did I call `file.Close()` in a `finally` block for every `FileOpen()` call, not relying on `__Delete`?
- [ ] Am I using `FileOpen()` without an encoding argument (plus `RawRead`/`RawWrite`) for binary files, rather than `FileRead()` without the `"RAW"` option?

## RUNTIME ERROR MAPPING

| Exception Class | Trigger Condition | Detection Code | Fix |
|----------------|-------------------|----------------|-----|
| `OSError` | `FileOpen()`, `FileRead()`, or `FileAppend()` on a locked, missing, or permission-denied file | `e.Message` contains the OS error description; `e.Extra` contains the Win32 error code | Wrap in `try/catch OSError`; pre-check `FileExist(path) != ""` for missing-file case; implement retry with `Sleep(200)` for transient lock |
| `OSError` | `FileDelete()`, `FileCopy()`, or `FileMove()` when source is absent (no wildcard) or destination exists with overwrite=0 | `e.Message` contains path description | Guard with `FileExist(path) != ""` before delete; pass explicit overwrite=1 to copy/move; check dest directory exists before copy |
| `OSError` | `IniRead()` when file or section/key is absent and no default argument is supplied | `e.Message` mentions the path | Always supply a `default` parameter to `IniRead()`; the default causes a safe return instead of throw |

## TIER 1 — Fundamentals: Existence Checks and Simple Read/Write
> METHODS COVERED: FileExist · DirExist · FileRead · FileAppend · DirCreate

`FileRead` and `FileAppend` are single-call operations that auto-open, operate, and auto-close — ideal for low-frequency, complete-file text reads and single writes. `FileExist` and `DirExist` return attribute strings, not booleans; always test with `!= ""`. Do not use `FileAppend` inside tight loops; use a `FileOpen` handle instead.

```ahk
targetPath := A_WorkingDir "\config.txt"

; ✓ != "" is the correct FileExist existence test — FileExist returns an attribute string, not a boolean
if FileExist(targetPath) != "" {
    ; ✓ Explicit UTF-8 encoding prevents silent corruption on any non-ASCII character
    content := FileRead(targetPath, "UTF-8")
} else {
    ; ✓ Encoding on FileAppend guarantees write fidelity across all system locales
    FileAppend("Default Setting=1`n", targetPath, "UTF-8")
    content := "Default Setting=1`n"
}

; ✓ !DirExist() is falsy only when path is absent — equivalent to DirExist() == ""
; ✓ Checking before DirCreate prevents a redundant OSError on an already-existing directory
if !DirExist(A_WorkingDir "\Logs") {
    DirCreate(A_WorkingDir "\Logs")
}

; ✗ FileExist as bare boolean — "D" for a directory is truthy; passes silently for dirs
; if FileExist(targetPath) {                      ; → logic error when path is a directory

; ✗ FileRead without encoding — system locale used, corrupts é ü ñ and CJK on non-Western OS
; content := FileRead(targetPath)                 ; → silent data corruption
```

## TIER 2 — Core Mutation: Copy, Move, and Delete
> METHODS COVERED: DirExist · DirCreate · FileCopy · FileExist · FileDelete · FileMove · FileRecycle

File and directory mutations are irreversible and must be wrapped in `try/catch OSError` — file locks, permission errors, and missing paths all throw `OSError`. Always ensure the destination directory exists before `FileCopy`; always pass the overwrite flag explicitly. Prefer `FileMove` over `FileCopy` + `FileDelete` for atomic rename within the same volume.

```ahk
sourceFile := A_WorkingDir "\data.txt"
destFile   := A_WorkingDir "\backup\data.txt"

; ✓ try/catch OSError wraps all mutations — locks and permission errors throw OSError
try {
    ; ✓ DirExist check before DirCreate — prevents OSError on pre-existing directory
    if !DirExist(A_WorkingDir "\backup") {
        DirCreate(A_WorkingDir "\backup")
    }

    ; ✓ Explicit overwrite flag (1) — default 0 does nothing if dest exists
    FileCopy(sourceFile, destFile, 1)

    ; ✓ FileExist != "" before FileDelete — avoids OSError on already-absent source
    if FileExist(sourceFile) != "" {
        FileDelete(sourceFile)
    }
} catch OSError as err {
    output := "Mutation failed: " err.Message
}

; ✓ FileRecycle sends to Recycle Bin rather than permanently deleting
; FileRecycle(sourceFile)

; ✗ FileCopy without overwrite flag — silently does nothing if destination exists
; FileCopy(sourceFile, destFile)                  ; → silent no-op when dest already present

; ✗ FileDelete without guard outside try/catch — throws OSError if file is absent
; FileDelete(sourceFile)                          ; → OSError if already deleted or locked
```

## TIER 3 — Search and Validation: Path Parsing and Directory Traversal
> METHODS COVERED: SplitPath · Loop Files · A_LoopFilePath · A_LoopFileName · A_LoopFileDir · A_LoopFileSize · Array.Push

`SplitPath` decomposes any path into its components via ByRef output parameters — always use it instead of manual string splitting, which breaks on UNC paths and drive-only strings. `Loop Files` iterates matching files and directories; the `"R"` mode flag recurses into all subdirectories. `A_LoopFileSize` provides per-file byte size without a separate `FileGetSize()` call inside the loop.

```ahk
fullPath     := A_WorkingDir "\images\photo.jpg"
outFileName  := ""
outDir       := ""
outExt       := ""
outNameNoExt := ""
outDrive     := ""

; ✓ SplitPath handles UNC paths, drive-only strings, and extension-less files correctly
; ✓ All output vars use & prefix — AHK v2 requires & for ByRef output parameters
SplitPath(fullPath, &outFileName, &outDir, &outExt, &outNameNoExt, &outDrive)

matchedFiles := []
totalSize    := 0

; ✓ Loop Files "R" recurses all subdirectories under the pattern root
; ✓ "F" restricts to files only — excludes directory entries from A_LoopFilePath
Loop Files, A_WorkingDir "\images\*.jpg", "RF"
{
    matchedFiles.Push(A_LoopFilePath)
    ; ✓ A_LoopFileSize is available without a separate FileGetSize() call inside the loop
    totalSize += A_LoopFileSize
}

; ✗ Manual path parsing via StrSplit — breaks on UNC paths and drive-only strings
; parts := StrSplit(fullPath, "\")               ; → incorrect for \\server\share\ paths

; ✗ RegExMatch for extension extraction — fragile, misses compound extensions like .tar.gz
; RegExMatch(fullPath, "\.(\w+)$", &ext)         ; → use SplitPath &outExt instead

; ✗ Loop Files without mode flag — includes both files and directories in results
; Loop Files, A_WorkingDir "\images\*.jpg"       ; → directories matching *.jpg also included
```

## TIER 4 — Transformations: Line Parsing and CSV Processing
> METHODS COVERED: Loop Read · A_LoopReadLine · Loop Parse · A_LoopField · FileRead · FileGetSize · StrSplit · InStr · Array.Push

CSV parsing requires a two-loop pattern: `Loop Read` iterates lines from a file, then `Loop Parse` with `"CSV"` splits each line into fields — the CSV delimiter mode handles quoted commas and embedded newlines correctly. For filter operations on in-memory text, call `FileGetSize()` first to confirm the file fits in RAM, then use `FileRead` once and split with `StrSplit`.

```ahk
csvPath    := A_WorkingDir "\data.csv"
parsedData := []

; ✓ Two-loop CSV pattern: Loop Read iterates lines; Loop Parse "CSV" splits fields
; ✓ "CSV" delimiter handles RFC-4180 quoted commas — a plain "," delimiter does not
Loop Read, csvPath
{
    rowArr := []
    Loop Parse, A_LoopReadLine, "CSV"
    {
        rowArr.Push(A_LoopField)
    }
    parsedData.Push(rowArr)
}

; ✓ FileGetSize() guards against loading multi-GB files that would exhaust RAM
fileSizeMB := FileGetSize(A_WorkingDir "\log.txt", "M")
if fileSizeMB > 100
    throw Error("Log file too large for in-memory filter: " fileSizeMB " MB")

; ✓ FileRead once then StrSplit — one open/close cycle for the full in-memory filter
textContent   := FileRead(A_WorkingDir "\log.txt", "UTF-8")
; ✓ Both "`n" and "`r" in omitChars strip Windows CRLF and Unix LF endings uniformly
lines         := StrSplit(textContent, "`n", "`r")
filteredLines := []

for index, line in lines {
    if InStr(line, "ERROR") {
        filteredLines.Push(line)
    }
}

; ✗ Loop Parse applied directly to a file path — iterates characters of the path string
; Loop Parse, csvPath, "CSV"                      ; → parses the path string, not the file

; ✗ Plain "," delimiter for CSV — splits "Smith, John" at the comma inside a quoted field
; Loop Parse, A_LoopReadLine, ","                 ; → incorrect field boundaries for quoted fields
```

## TIER 5 — Advanced Operations: INI Config, FileOpen Lifecycle, and Wrapper Classes
> METHODS COVERED: IniWrite · IniRead · IniDelete · FileOpen · File.WriteLine · File.Close

INI functions provide structured section/key configuration storage without manual parsing. For high-frequency write operations, a single `FileOpen` handle held across the loop avoids repeated disk open/close cycles. A wrapper class with `__Delete` provides belt-and-suspenders lifecycle management — `__Delete` is a fallback, not a substitute for an explicit `Close()` in `finally`.

```ahk
iniPath := A_WorkingDir "\settings.ini"

; ✓ IniWrite creates the file, section, and key if any are absent — no pre-check needed
IniWrite("Dark", iniPath, "UI", "Theme")
; ✓ IniRead with default arg — returns "Light" if key absent, never throws
themeVal := IniRead(iniPath, "UI", "Theme", "Light")

; ✓ IniDelete with key omitted removes the entire section
IniDelete(iniPath, "TempSection")

logPath := A_WorkingDir "\bulk_log.txt"
; ✓ fileObj := "" before try — allows `if fileObj` nil-check in catch to guard Close()
;   when a later operation (not FileOpen itself) raises OSError after the handle is open
fileObj := ""
try {
    ; ✓ FileOpen before loop — one open/close cycle regardless of loop count
    ; ✓ try/catch OSError is the correct v2 pattern — FileOpen throws, does not return 0
    fileObj := FileOpen(logPath, "a", "UTF-8")

    Loop 1000 {
        fileObj.WriteLine("Log entry number: " A_Index)
    }
} catch OSError as err {
    ; ✓ nil-check in catch — fileObj is "" if FileOpen threw; set if WriteLine threw
    if fileObj
        fileObj.Close()
    throw
} finally {
    ; ✓ finally guarantees Close() on both success and exception paths
    if fileObj
        fileObj.Close()
}

; ✓ Wrapper class with __Delete — fallback cleanup if caller forgets explicit Close()
; ✓ __New wraps FileOpen in try/catch to surface I/O errors at construction time
class SafeFileWriter {
    __New(filePath, mode := "a", encoding := "UTF-8") {
        this.FilePath := filePath
        this.FileObj  := ""
        try {
            ; ✓ FileOpen throws OSError on failure — caught here and re-thrown with context
            this.FileObj := FileOpen(filePath, mode, encoding)
        } catch OSError as err {
            throw Error("SafeFileWriter: failed to open '" filePath "' — " err.Message)
        }
    }

    WriteLine(text) {
        if !this.FileObj
            throw Error("SafeFileWriter: file handle is not open")
        this.FileObj.WriteLine(text)
    }

    Close() {
        if this.FileObj {
            this.FileObj.Close()
            this.FileObj := ""
        }
    }

    ; ✓ __Delete as safety net — fires when object goes out of scope or is overwritten
    ; ✗ Do not rely solely on __Delete — AHK v2 GC timing is non-deterministic
    __Delete() {
        this.Close()
    }
}

; ✗ IniRead without default arg — throws OSError when key is absent
; val := IniRead(iniPath, "UI", "MissingKey")    ; → OSError if key not found

; ✗ FileAppend inside loop — N separate open/write/close disk operations
; Loop 1000 { FileAppend("entry`n", logPath, "UTF-8") } ; → 1000× disk thrash
```

## TIER 6 — Binary I/O: Raw Streaming and File Pointer Manipulation
> METHODS COVERED: FileOpen · File.WriteUInt · File.ReadUInt · File.Pos · File.AtEOF · File.RawRead · File.RawWrite · File.Length · File.Handle · Buffer

Binary file I/O requires `FileOpen` without an encoding argument — any encoding applies text decoding that corrupts raw bytes. Use `ReadUInt`/`WriteUInt` for fixed-width integer fields and `RawRead`/`RawWrite` with a `Buffer` for streaming arbitrary binary data. When two file handles are used, initialize both to `""` before `try` so the `catch` block can distinguish which handles are safe to close.

```ahk
binPath := A_WorkingDir "\data.bin"
; ✓ Initialize both to "" before try — enables per-handle nil-check in catch
;   FileOpen throws OSError on failure; "" init distinguishes "never opened" from "open"
fileObj := ""
destObj := ""

try {
    ; ✓ No encoding argument — encoding would apply text decoding and corrupt raw bytes
    fileObj := FileOpen(binPath, "rw")

    numVal := 0x1A2B3C4D
    ; ✓ WriteUInt writes exactly 4 bytes at the current file position
    fileObj.WriteUInt(numVal)

    ; ✓ .Pos := 0 seeks to file start — equivalent to .Seek(0, 0)
    fileObj.Pos := 0

    readVal := fileObj.ReadUInt()

    ; ✓ No encoding on destination binary file either
    destObj := FileOpen(A_WorkingDir "\copy.bin", "w")
    ; ✓ Buffer(4096) caps memory use to 4096 bytes regardless of source file size
    buffer  := Buffer(4096)

    fileObj.Pos := 0
    ; ✓ AtEOF loop streams in 4096-byte chunks — O(1) memory for arbitrary file size
    while !fileObj.AtEOF {
        bytesRead := fileObj.RawRead(buffer)
        ; ✓ Pass bytesRead explicitly — last chunk may be smaller than Buffer.Size
        destObj.RawWrite(buffer, bytesRead)
    }
} catch OSError as err {
    ; ✓ Per-handle nil-check — destObj may not be assigned if fileObj opened but destObj threw
    if destObj
        destObj.Close()
    if fileObj
        fileObj.Close()
    throw
} finally {
    if destObj
        destObj.Close()
    if fileObj
        fileObj.Close()
}

; ✗ FileOpen binary with text encoding — codec corrupts raw byte sequences
; fileObj := FileOpen(binPath, "rw", "UTF-8")    ; → binary data corruption

; ✗ FileRead on a binary file without RAW option — decodes bytes as text, corrupts raw data
; content := FileRead(binPath)                    ; → data loss on raw binary content

; ✓ Alternative: FileRead with RAW option returns a Buffer object instead of string
; buf := FileRead(binPath, "RAW")                 ; → buf is a Buffer object, no decoding
```

### Performance Notes

File I/O is bottlenecked by disk latency, not CPU. Every `FileAppend()` call inside a loop incurs a full open/write/close cycle. For 500 iterations that is 500 separate disk operations — replace with a single `FileOpen` handle held across the entire loop.

`FileRead()` is O(1) disk operations but O(n) memory — acceptable for files well within available RAM. Always call `FileGetSize()` before `FileRead()` on user-supplied paths to guard against memory exhaustion. `Loop Read` streams line-by-line with O(1) memory footprint and is the correct choice when file size is unknown or potentially large.

For binary files, use `Buffer(4096)` chunks with `RawRead`/`RawWrite` to cap memory use at a fixed ceiling regardless of file size. Avoid growing a string incrementally with concatenation for binary data — each concatenation allocates a new string.

Always prefer built-in `IniRead`/`IniWrite` over hand-rolling INI parsers — the built-in handles encoding, whitespace, and section edge cases automatically.

`FileMove` is more efficient than `FileCopy` + `FileDelete` for rename operations within the same volume — it maps to a Win32 atomic rename that does not copy bytes.

## DROP-IN RECIPES

```ahk
; SafeReadText — robust text file reader with size guard, existence check, and typed error
; ✓ Checks file existence and size before reading — prevents silent failure and OOM crash
SafeReadText(path, encoding := "UTF-8", maxMB := 50) {
    if !(path is String) || path = ""
        throw TypeError("SafeReadText: path must be a non-empty string", -1)
    if FileExist(path) = ""
        throw Error("SafeReadText: file not found — " path, -1)
    sizeMB := FileGetSize(path, "M")
    if sizeMB > maxMB
        throw Error("SafeReadText: file exceeds " maxMB " MB limit (" sizeMB " MB) — " path, -1)
    try
        return FileRead(path, encoding)
    catch OSError as e
        throw Error("SafeReadText: read failed on '" path "' — " e.Message, -1)
}
; Call site: content := SafeReadText("config.txt")
; Call site: content := SafeReadText("large_file.log", "UTF-8", 200)

; SafeAppendLog — bulk-append multiple lines using a single FileOpen handle with retry
; ✓ One open/close cycle for N lines — avoids FileAppend loop disk thrashing
; ✓ Validates that lines is an Array before iterating — never silently writes nothing
SafeAppendLog(path, lines, encoding := "UTF-8") {
    if !(path is String) || path = ""
        throw TypeError("SafeAppendLog: path must be a non-empty string", -1)
    if !(lines is Array)
        throw TypeError("SafeAppendLog: lines must be an Array", -1)
    f := ""
    loop 2 {
        try {
            f := FileOpen(path, "a", encoding)
            for line in lines
                f.WriteLine(line)
            return
        } catch OSError as e {
            if f {
                f.Close()
                f := ""
            }
            if A_Index < 2
                Sleep(200)
            else
                throw Error("SafeAppendLog: could not write to '" path "' after retry — " e.Message, -1)
        } finally {
            if f {
                f.Close()
                f := ""
            }
        }
    }
}
; Call site: SafeAppendLog("events.log", [FormatTime(, "yyyy-MM-dd HH:mm:ss") " startup", "init complete"])
```

## ANTI-PATTERNS

| Pattern | Wrong | Correct | LLM Common Cause |
|---------|-------|---------|------------------|
| FileAppend inside a loop | `Loop { FileAppend(text, path) }` | `f := FileOpen(path, "a", "UTF-8"); try { Loop { f.WriteLine(text) } } finally { f.Close() }` | AHK v1 had no File object; `FileAppend` was the primary write API and its per-call overhead was invisible at small loop counts in training examples |
| `if !f` guard after FileOpen | `f := FileOpen(path, "r"); if !f { ... }` | `try { f := FileOpen(path, "r") } catch OSError as e { ... }` | AHK v1 returned 0 on FileOpen failure; LLMs trained on v1 and mixed-version examples reproduce the `if !f` pattern that no longer triggers in v2 |
| Manual path parsing | `StrSplit(path, "\")[-1]` or `RegExMatch(path, "\\([^\\]+)$")` | `SplitPath(path, &name, &dir, &ext)` | Cross-language habit from Python `os.path.split()` and JS `String.split()`; LLMs miss that `SplitPath` handles UNC paths, drive-only strings, and extension-less files |
| Omitting encoding parameter | `FileRead(path)` / `FileAppend(text, path)` | `FileRead(path, "UTF-8")` / `FileAppend(text, path, "UTF-8")` | AHK v1 encoding was optional and implicitly system-default; most language stdlib encoding arguments are optional; LLMs reproduce the shorter form from mixed-version training data |
| FileExist/DirExist as boolean | `if FileExist(path)` | `if FileExist(path) != ""` | Virtually all language standard libraries return a true boolean from exists checks; AHK v2's attribute-string return value is an unusual API design that LLMs trained on multi-language corpora consistently miss |
| FileRead on binary without RAW | `content := FileRead("img.png")` | `buf := FileRead("img.png", "RAW")` → Buffer object; or `FileOpen("img.png", "r")` + `.RawRead(buf)` | AHK v1 had no Buffer type and no clean binary/text API distinction; LLMs default to the simple `FileRead` form |

## SEE ALSO

> This module does NOT cover: IStream COM-based async I/O and file attribute manipulation (FileGetAttrib/FileSetAttrib/FileGetTime/FileSetTime) → see Module_SystemAndCOM.md
> This module does NOT cover: string transformation on file content after reading (StrReplace, RegExReplace, Format) → see Module_TextProcessing.md
> This module does NOT cover: try/catch/finally patterns, OSError class structure, and exception propagation for I/O failure recovery → see Module_Errors.md
> This module does NOT cover: storing parsed file lines and CSV fields in typed Arrays or Maps, and filtering/sorting those collections → see Module_Arrays.md

- `Module_Errors.md` — OSError structure, try/catch/finally patterns, and error propagation strategies for FileOpen failure, locked-file recovery, and I/O exception handling.
- `Module_Arrays.md` — storing parsed CSV rows and filtered log lines (from TIER 4 Loop Read patterns) in typed Arrays; sorting, mapping, and filtering file content collections.
- `Module_TextProcessing.md` — StrReplace, RegExReplace, Format, and all string transformation applied to content retrieved via FileRead or Loop Read.
- `Module_SystemAndCOM.md` — IStream COM interface for async streaming, FileGetAttrib/FileSetAttrib for file attribute manipulation, and low-level Win32 file handle operations via DllCall.