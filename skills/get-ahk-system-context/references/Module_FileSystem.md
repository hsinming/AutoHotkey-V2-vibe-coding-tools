# Module_FileSystem.md
<!-- DOMAIN: File System -->
<!-- SCOPE: Async I/O via IStream COM, file attribute manipulation via FileGetAttrib/FileSetAttrib, and string transformation of file content after reading are not covered — see Module_SystemAndCOM.md and Module_TextProcessing.md. -->
<!-- TRIGGERS: FileOpen, FileRead, FileAppend, FileExist, DirExist, SplitPath, DirCreate, Loop Files, Loop Read, IniRead, IniWrite, "read file", "write file", "parse csv", "check if file exists", "get file extension", "binary file", "create folder", "log data repeatedly" -->
<!-- CONSTRAINTS: Always specify encoding on every FileRead/FileAppend/FileOpen call for text files — omitting it silently corrupts non-ASCII data via system-locale fallback. Never use FileAppend inside a loop — open a FileOpen handle before the loop and close it in both the success and catch paths; AHK v2 __Delete timing is non-deterministic and cannot substitute for explicit Close(). -->
<!-- CROSS-REF: Module_Errors.md, Module_Arrays.md, Module_TextProcessing.md, Module_SystemAndCOM.md -->
<!-- VERSION: AHK v2.0+ -->

## V1 → V2 BREAKING CHANGES

| v1 pattern (LLM commonly writes) | v2 correct form | Consequence |
|----------------------------------|-----------------|-------------|
| `FileRead, var, path` (command syntax, no encoding) | `var := FileRead(path, "UTF-8")` | SyntaxError — v2 has no command mode; LLMs trained on mixed-version AHK omit parentheses and assignment form |
| `FileAppend, text, path` (command form, no encoding) | `FileAppend(text, path, "UTF-8")` | SyntaxError + silent data corruption on any non-ASCII character |
| `FileAppend(text, path)` inside a loop | `fileObj := FileOpen(path, "a", "UTF-8")` before loop; `fileObj.WriteLine(text)` inside loop; `fileObj.Close()` after | Repeated open/write/close per iteration — severe disk thrashing at scale |
| `FileRead(path)` without encoding argument | `FileRead(path, "UTF-8")` | Silent corruption — system locale encoding used; breaks on é ü ñ and any CJK characters |
| `if FileExist(path)` as bare boolean | `if FileExist(path) != ""` | Logic error — FileExist returns an attribute string ("A", "D", "H", etc.); implicit truthy test passes for directories when a file is expected |
| `FileReadLine, var, path, N` (single-line read by index) | `Loop Read, path { if A_Index = N { line := A_LoopReadLine } }` | No direct single-line-by-index API in v2; LLMs trained on v1 hallucinate a command that has no v2 equivalent |
| No `file.Close()` after `FileOpen` | Explicit `.Close()` in a `finally` block, or in both the success path and catch block | File handle leak — v1 auto-closed handles on script exit; v2 `__Delete` timing is non-deterministic and does not guarantee cleanup on exception paths |

## API QUICK-REFERENCE

### Standalone Read/Write Functions
| Method/Property | Signature | Notes |
|-----------------|-----------|-------|
| `FileRead()` | `FileRead(path, encoding?)` | Returns entire file as string; text only — never use for binary files |
| `FileAppend()` | `FileAppend(text, path, encoding?)` | Appends text; creates file if absent; avoid inside loops — use FileOpen instead |
| `FileExist()` | `FileExist(path)` | Returns attribute string ("A", "D", "H", etc.) or `""` — **not a boolean**; test with `!= ""` |
| `DirExist()` | `DirExist(path)` | Returns attribute string or `""` — not a boolean; `""` is falsy, so `!DirExist(p)` is equivalent to `DirExist(p) == ""` |

### File Mutation Functions
| Method/Property | Signature | Notes |
|-----------------|-----------|-------|
| `FileCopy()` | `FileCopy(source, dest, overwrite?)` | overwrite: 0 = no (default), 1 = yes; throws OSError if source absent or dest exists with overwrite=0 |
| `FileDelete()` | `FileDelete(pattern)` | Supports wildcards; throws OSError if file is locked or absent |
| `FileMove()` | `FileMove(source, dest, overwrite?)` | overwrite: 0 = no, 1 = yes; prefer over FileCopy + FileDelete for atomic rename |

### Directory Functions
| Method/Property | Signature | Notes |
|-----------------|-----------|-------|
| `DirCreate()` | `DirCreate(path)` | Creates all missing intermediate directories automatically |
| `DirDelete()` | `DirDelete(path, recurse?)` | recurse: 0 = no (default), 1 = yes — non-empty directories require recurse = 1 |
| `DirCopy()` | `DirCopy(source, dest, overwrite?)` | Copies entire directory tree |
| `DirMove()` | `DirMove(source, dest, flag?)` | Moves or renames a directory; flag: 0 = no overwrite, 1 = overwrite |

### Path Functions
| Method/Property | Signature | Notes |
|-----------------|-----------|-------|
| `SplitPath()` | `SplitPath(path, &name?, &dir?, &ext?, &nameNoExt?, &drive?)` | All output params optional; must use `&` prefix (ByRef); always prefer over manual string parsing |

### FileOpen Handle (File Object)
| Method/Property | Signature | Notes |
|-----------------|-----------|-------|
| `FileOpen()` | `FileOpen(path, mode, encoding?)` | Returns File object; mode: `"r"`=read, `"w"`=write, `"a"`=append, `"rw"`=read+write; omit encoding entirely for binary files |
| `File.Read()` | `File.Read(chars?)` | Reads up to chars characters as string; omit chars to read all remaining |
| `File.ReadLine()` | `File.ReadLine()` | Reads one line, stripping the line ending |
| `File.ReadUInt()` | `File.ReadUInt()` | Reads a 32-bit unsigned integer at the current position |
| `File.Write()` | `File.Write(str)` | Writes string at current position |
| `File.WriteLine()` | `File.WriteLine(str)` | Writes string followed by a newline |
| `File.WriteUInt()` | `File.WriteUInt(n)` | Writes a 32-bit unsigned integer at the current position |
| `File.RawRead()` | `File.RawRead(buffer, bytes?)` | Reads raw bytes into a Buffer object; returns bytes actually read; bytes defaults to Buffer.Size |
| `File.RawWrite()` | `File.RawWrite(data, bytes?)` | Writes raw bytes from a Buffer or string; bytes defaults to Buffer.Size |
| `File.Seek()` | `File.Seek(dist, origin?)` | Moves file pointer; origin: 0 = start (default), 1 = current, 2 = end |
| `File.Close()` | `File.Close()` | Releases file handle — must always be called explicitly; never rely on garbage collection |
| `File.Pos` | `File.Pos` (read/write) | Current byte position; assigning it seeks to that position — equivalent to `.Seek(n, 0)` |
| `File.Length` | `File.Length` (read/write) | File length in bytes; assigning a smaller value truncates the file |
| `File.AtEOF` | `File.AtEOF` (read-only) | Returns non-zero when the file pointer is at end of file |
| `File.Encoding` | `File.Encoding` (read/write) | Encoding used by this file handle; can be reassigned to change the active codec |

### INI Functions
| Method/Property | Signature | Notes |
|-----------------|-----------|-------|
| `IniRead()` | `IniRead(path, section, key, default?)` | Returns value string; returns default when key absent (throws if default omitted) |
| `IniWrite()` | `IniWrite(value, path, section, key)` | Creates file, section, and key if absent |
| `IniDelete()` | `IniDelete(path, section, key?)` | Omit key to delete entire section; section is required and cannot be omitted |

### Loop Constructs and Built-in Loop Variables
| Construct / Variable | Description | Notes |
|----------------------|-------------|-------|
| `Loop Files` | `Loop Files, pattern, mode?` | Mode flags: `"R"` = recurse subdirs, `"D"` = dirs only, `"F"` = files only; flags combinable |
| `A_LoopFilePath` | Full path of the current file in Loop Files | Includes drive + directory + name + extension |
| `A_LoopFileName` | Name and extension of the current file | No directory component |
| `A_LoopFileDir` | Directory of the current file | No trailing backslash |
| `Loop Read` | `Loop Read, filePath` | Iterates all lines of a text file sequentially |
| `A_LoopReadLine` | Current line text in Loop Read | Line ending is stripped |
| `Loop Parse` | `Loop Parse, str, delimiters?, omitChars?` | Use `"CSV"` as delimiter to handle RFC-4180 quoted fields; cannot be applied directly to a file path |
| `A_LoopField` | Current field value in Loop Parse | |

### Buffer Object
| Method/Property | Signature | Notes |
|-----------------|-----------|-------|
| `Buffer()` | `Buffer(byteCount, fillByte?)` | Allocates raw memory for binary I/O; fillByte defaults to 0 |

### String Helper Functions (cross-domain — used in TIER 4 file content processing)
| Function | Signature | Notes |
|----------|-----------|-------|
| `StrSplit()` | `StrSplit(str, delimiters?, omitChars?)` | Splits string into a 1-based Array; see Module_TextProcessing.md for full API |
| `InStr()` | `InStr(haystack, needle, caseSense?, startingPos?, occurrence?)` | Returns 1-based position or 0 if not found; see Module_TextProcessing.md |

### Array Methods (cross-domain — used in file content collection patterns)
| Method | Signature | Notes |
|--------|-----------|-------|
| `Array.Push()` | `arr.Push(val1, val2?, ...)` | Appends elements; no return value — see Module_Arrays.md for full Array API |

## AHK V2 CONSTRAINTS

- Always pass an explicit encoding to `FileRead()`, `FileAppend()`, and `FileOpen()` for all text files — omitting it silently falls back to the system locale, causing non-ASCII data corruption on any non-Western machine.
- ✗ `FileAppend(text, path)` inside a loop — opens, writes, and closes on every iteration, causing repeated disk thrashing.
- ✓ `fileObj := FileOpen(path, "a", "UTF-8")` before the loop; `.WriteLine(text)` inside the loop; `.Close()` in both success and catch paths — one open/close cycle for N writes.
- Always call `file.Close()` explicitly in both the success path and the catch block — AHK v2 `__Delete` is a fallback, not a guarantee; handle leaks are certain on exception paths without explicit close. Prefer `try/finally { file.Close() }` for the most robust cleanup.
- Use `SplitPath()` for all path decomposition — never split by `\`, `StrSplit`, or `RegExMatch`; `SplitPath` correctly handles UNC paths, drive-only paths, and extension-less files.
- ✗ `if FileExist(path)` — truthy for any non-empty string, including `"D"` (directory); silently passes when the path resolves to a directory instead of the expected file.
- ✓ `if FileExist(path) != ""` — explicit string comparison; combine with `InStr(FileExist(path), "D") = 0` when the check must distinguish files from directories.
- Binary files require `FileOpen(path, mode)` without an encoding argument — AHK v2 has no `"b"` flag; binary access is achieved by omitting the encoding parameter. `FileRead()` and any encoding-aware mode apply text decoding that corrupts raw binary data.
- Always anchor paths via `A_WorkingDir` or use absolute paths — relative paths resolve from the current working directory at runtime, which may shift if the script spawns external processes.
- Validate directory existence with `DirExist` before `DirCreate`; validate file existence with `FileExist(path) != ""` before `FileDelete` — these functions throw `OSError` on missing targets outside a `try` block.

Safe-access priority order for file operations:
  1. `FileRead(path, "UTF-8")` — complete text file reads; auto-opens and auto-closes in one call
  2. `FileOpen(path, mode, encoding?)` in `try/finally` — partial reads, writes, binary, or high-frequency loop operations
  3. `FileExist(path) != ""` check before `FileOpen` — when absent-file handling must differ structurally from I/O error handling
  4. `try/catch OSError` on `FileOpen` — when the error message itself carries diagnostic information needed for logging or user feedback

## TIER 1 — Fundamentals: Existence Checks and Simple Read/Write
> METHODS COVERED: FileExist · DirExist · FileRead · FileAppend · DirCreate

`FileRead` and `FileAppend` are single-call operations that auto-open, operate, and auto-close — ideal for low-frequency, complete-file text reads and writes. `FileExist` and `DirExist` return attribute strings, not booleans; always test with `!= ""`. Do not use these convenience functions inside tight loops.
```ahk
targetPath := A_WorkingDir "\config.txt"

; ✓ != "" is the correct FileExist existence test — FileExist returns an attribute string
if FileExist(targetPath) != "" { ; <!-- CONVERTED: bare `if FileExist()` replaced with `!= ""` — FileExist returns an attribute string, not a boolean -->
    ; ✓ Explicit UTF-8 encoding prevents silent corruption on any non-ASCII character
    content := FileRead(targetPath, "UTF-8")
} else {
    ; ✓ Encoding on FileAppend guarantees write fidelity across all system locales
    FileAppend("Default Setting=1`n", targetPath, "UTF-8")
    content := "Default Setting=1`n"
}

; ✓ !DirExist() is falsy only when path is absent — equivalent to DirExist() == ""
; ✓ DirCreate check before create prevents redundant OSError on existing directories
if !DirExist(A_WorkingDir "\Logs") {
    DirCreate(A_WorkingDir "\Logs")
}

; ✗ FileExist as bare boolean — "D" for a directory is truthy; passes silently for dirs
; if FileExist(targetPath) {                      ; → logic error when path is a directory

; ✗ FileRead without encoding — system locale used, corrupts é ü ñ and CJK on non-Western OS
; content := FileRead(targetPath)                 ; → silent data corruption
```

## TIER 2 — Core Mutation: Copy, Move, and Delete
> METHODS COVERED: DirExist · DirCreate · FileCopy · FileExist · FileDelete

File and directory mutations are irreversible and must be wrapped in `try/catch OSError` — file locks, permission errors, and missing paths all throw `OSError`. Always ensure the destination directory exists before `FileCopy`; always pass the overwrite flag explicitly.
```ahk
sourceFile := A_WorkingDir "\data.txt"
destFile   := A_WorkingDir "\backup\data.txt"

; ✓ try/catch OSError wraps all mutations — locks and permission errors throw OSError
try {
    ; ✓ DirExist check before DirCreate — prevents OSError on pre-existing directory
    if !DirExist(A_WorkingDir "\backup") {
        DirCreate(A_WorkingDir "\backup")
    }

    ; ✓ Explicit overwrite flag (1) — default 0 silently does nothing if dest exists
    FileCopy(sourceFile, destFile, 1)

    ; ✓ FileExist != "" before FileDelete — avoids OSError on already-absent source
    if FileExist(sourceFile) != "" { ; <!-- CONVERTED: bare `if FileExist()` replaced with `!= ""` — FileExist returns an attribute string, not a boolean -->
        FileDelete(sourceFile)
    }
} catch OSError as err {
    output := "Mutation failed: " err.Message
}

; ✗ FileCopy without overwrite flag — silently does nothing if destination exists
; FileCopy(sourceFile, destFile)                  ; → silent no-op when dest already present

; ✗ FileDelete without guard outside try/catch — throws OSError if file is absent
; FileDelete(sourceFile)                          ; → OSError if already deleted or locked
```

## TIER 3 — Search and Validation: Path Parsing and Directory Traversal
> METHODS COVERED: SplitPath · Loop Files · A_LoopFilePath · A_LoopFileName · A_LoopFileDir · Array.Push

`SplitPath` decomposes any path into its components via ByRef output parameters — always use it instead of manual string splitting, which breaks on UNC paths and drive-only strings. `Loop Files` iterates matching files; the `"R"` mode flag recurses into all subdirectories.
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

; ✓ Loop Files "R" recurses all subdirectories under the pattern root
Loop Files, A_WorkingDir "\images\*.jpg", "R"
{
    matchedFiles.Push(A_LoopFilePath)
}

; ✗ Manual path parsing via StrSplit — breaks on UNC paths and drive-only strings
; parts := StrSplit(fullPath, "\")               ; → incorrect for \\server\share\ paths

; ✗ RegExMatch for extension extraction — fragile, misses compound extensions like .tar.gz
; RegExMatch(fullPath, "\.(\w+)$", &ext)         ; → use SplitPath &outExt instead
```

## TIER 4 — Transformations: Line Parsing and CSV Processing
> METHODS COVERED: Loop Read · A_LoopReadLine · Loop Parse · A_LoopField · FileRead · StrSplit · InStr · Array.Push

CSV parsing requires a two-loop pattern: `Loop Read` iterates lines from a file, then `Loop Parse` with `"CSV"` splits each line into fields — the CSV delimiter mode handles quoted commas and embedded newlines correctly. For filter operations on in-memory text, use `FileRead` once and split with `StrSplit`.
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

; ✗ Plain "," delimiter for CSV — splits "Smith, John" at the comma inside the quoted field
; Loop Parse, A_LoopReadLine, ","                 ; → incorrect field boundaries for quoted fields
```

## TIER 5 — Advanced Operations: INI Config, FileOpen Lifecycle, and Wrapper Classes
> METHODS COVERED: IniWrite · IniRead · FileOpen · File.WriteLine · File.Close

INI functions provide structured section/key configuration storage without manual parsing. For high-frequency write operations, a single `FileOpen` handle held across the entire loop avoids repeated disk open/close cycles. A wrapper class with `__Delete` provides belt-and-suspenders lifecycle management — `__Delete` is a fallback, not a substitute for an explicit `Close()` call.
```ahk
iniPath := A_WorkingDir "\settings.ini"

; ✓ IniWrite creates the file, section, and key if any are absent — no pre-check needed
IniWrite("Dark", iniPath, "UI", "Theme")
; ✓ IniRead with default arg — returns "Light" if key absent, never throws
themeVal := IniRead(iniPath, "UI", "Theme", "Light")

logPath := A_WorkingDir "\bulk_log.txt"
; ✓ fileObj := "" before try — enables nil-check in catch if FileOpen fails before assignment
fileObj := ""
try {
    ; ✓ FileOpen before loop — one open/close cycle regardless of loop count
    fileObj := FileOpen(logPath, "a", "UTF-8")

    Loop 1000 {
        fileObj.WriteLine("Log entry number: " A_Index)
    }

    fileObj.Close()
} catch OSError as err {
    ; ✓ Close in catch path — handle leak is guaranteed without this on exception paths
    if fileObj
        fileObj.Close()
}

; ✓ Wrapper class with __Delete — fallback cleanup if caller forgets explicit Close()
; ✓ __New wraps FileOpen in try/catch to surface I/O errors at construction time
class SafeFileWriter {
    __New(filePath, mode := "a", encoding := "UTF-8") {
        this.FilePath := filePath
        try {
            this.FileObj := FileOpen(filePath, mode, encoding)
        } catch OSError as err {
            throw Error("Failed to open file: " err.Message)
        }
    }

    WriteLine(text) {
        if (!this.FileObj)
            throw Error("File object is not available.")
        this.FileObj.WriteLine(text)
    }

    Close() {
        if (this.FileObj) {
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

; ✗ FileAppend inside loop — N separate open/write/close disk operations
; Loop 1000 { FileAppend("entry`n", logPath, "UTF-8") } ; → 1000× disk thrash
```

### Performance Notes

File I/O is bottlenecked by disk latency, not CPU. Every `FileAppend()` call inside a loop incurs a full open/write/close cycle. For 500 iterations that is 500 separate disk operations — replace with a single `FileOpen` handle held across the entire loop.
```ahk
; ✗ FileAppend inside loop — 500 separate open/write/close disk operations
Loop 500 {
    FileAppend("Data row " A_Index "`n", A_WorkingDir "\bad_perf.txt")
}

; ✓ FileOpen before loop — one open, 500 writes via buffer, one close
fileObj := FileOpen(A_WorkingDir "\good_perf.txt", "w", "UTF-8")
Loop 500 {
    fileObj.WriteLine("Data row " A_Index)
}
fileObj.Close()

; ✗ FileRead on a massive file — loads entire content into one string in RAM
;    Use Loop Read for line-by-line streaming when file size may exceed available memory
fullText := FileRead(A_WorkingDir "\massive_file.txt", "UTF-8")
```

O(n) vs O(1) tradeoffs: `FileRead` is O(1) disk operations but O(n) memory — acceptable for files well within available RAM. `Loop Read` streams line-by-line with O(1) memory footprint. For binary files, use `Buffer(4096)` chunks with `RawRead`/`RawWrite` to cap memory use regardless of file size. Always prefer built-in `IniRead`/`IniWrite` over hand-rolling INI parsers — the built-in handles encoding, whitespace, and section edge cases automatically.

## TIER 6 — Binary I/O: Raw Streaming and File Pointer Manipulation
> METHODS COVERED: FileOpen · File.WriteUInt · File.ReadUInt · File.Pos · File.AtEOF · File.RawRead · File.RawWrite · Buffer

Binary file I/O requires `FileOpen` without an encoding argument — any encoding applies text decoding that corrupts raw bytes. Use `ReadUInt`/`WriteUInt` for fixed-width integer fields and `RawRead`/`RawWrite` with a `Buffer` for streaming arbitrary binary data. Both file handles must be closed in ALL code paths — the catch block must close any handle that was successfully opened.
```ahk
binPath := A_WorkingDir "\data.bin"
; ✓ Initialize both to "" before try — enables per-handle nil-check in catch
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
    ; ✓ Nil-check each handle — destObj may not be assigned if error occurred before it
    if fileObj
        fileObj.Close()
    if destObj
        destObj.Close()
    return
}

; ✓ Close both handles on success path — catch returns early above, no double-close risk
fileObj.Close()
destObj.Close()

; ✗ FileOpen binary with text encoding — codec corrupts raw byte sequences
; fileObj := FileOpen(binPath, "rw", "UTF-8")    ; → binary data corruption

; ✗ FileRead on a binary file — decodes bytes as text, corrupts bytes outside text range
; content := FileRead(binPath)                    ; → data loss on raw binary content
```

## ANTI-PATTERNS

| Pattern | Wrong | Correct | LLM Common Cause |
|---------|-------|---------|------------------|
| FileAppend inside a loop | `Loop { FileAppend(text, path) }` | `f := FileOpen(path, "a", "UTF-8"); Loop { f.WriteLine(text) }; f.Close()` | AHK v1 had no File object; `FileAppend` was the primary write API and its per-call overhead was invisible at small loop counts in training examples |
| Manual path parsing | `StrSplit(path, "\")[-1]` or `RegExMatch(path, "\\([^\\]+)$")` | `SplitPath(path, &name, &dir, &ext)` | Cross-language habit from Python `os.path.split()` and JS `String.split()`; LLMs miss that `SplitPath` handles UNC paths, drive-only strings, and extension-less files |
| Omitting encoding parameter | `FileRead(path)` / `FileAppend(text, path)` | `FileRead(path, "UTF-8")` / `FileAppend(text, path, "UTF-8")` | AHK v1 encoding was optional and implicitly system-default; most language stdlib encoding arguments are optional; LLMs reproduce the shorter form from mixed-version training data |
| Unclosed FileOpen handle | `f := FileOpen(path, "w"); f.Write(...)` with no `.Close()` | Wrap entire block in `try/finally { f.Close() }` or use a wrapper class with `__Delete` | AHK v1 auto-closed file handles on script exit or variable reassignment; LLMs trained on v1 examples consistently omit explicit close |
| FileExist/DirExist as boolean | `if FileExist(path)` | `if FileExist(path) != ""` | Virtually all language standard libraries return a true boolean from exists checks; AHK v2's attribute-string return value is an unusual API design that LLMs trained on multi-language corpora consistently miss |

## SEE ALSO

> This module does NOT cover: IStream COM-based async I/O and file attribute manipulation (FileGetAttrib/FileSetAttrib) → see Module_SystemAndCOM.md
> This module does NOT cover: string transformation on file content after reading (StrReplace, RegExReplace, Format) → see Module_TextProcessing.md
> This module does NOT cover: try/catch/finally patterns, OSError class structure, and exception propagation for I/O failure recovery → see Module_Errors.md
> This module does NOT cover: storing parsed file lines and CSV fields in typed Arrays or Maps, and filtering/sorting those collections → see Module_Arrays.md

- `Module_Errors.md` — OSError structure, try/catch/finally patterns, and error propagation strategies for FileOpen failure, locked-file recovery, and I/O exception handling.
- `Module_Arrays.md` — storing parsed CSV rows and filtered log lines (from TIER 4 Loop Read patterns) in typed Arrays; sorting, mapping, and filtering file content collections.
- `Module_TextProcessing.md` — StrReplace, RegExReplace, Format, and all string transformation applied to content retrieved via FileRead or Loop Read.
- `Module_SystemAndCOM.md` — IStream COM interface for async streaming, FileGetAttrib/FileSetAttrib for file attribute manipulation, and low-level Win32 file handle operations via DllCall.