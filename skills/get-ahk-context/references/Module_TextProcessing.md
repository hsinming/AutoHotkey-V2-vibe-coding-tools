# Module_TextProcessing.md
<!-- DOMAIN: Text Processing -->
<!-- SCOPE: File I/O text encoding, GUI control text content, hotstring trigger definitions, and clipboard operations are not covered here — see Module_FileSystem.md, Module_GUI.md, and Module_InputAndHotkeys.md -->
<!-- TRIGGERS: StrReplace(), RegExMatch(), RegExReplace(), InStr(), SubStr(), StrSplit(), StrCompare(), StrTitle(), Format(), Trim(), "string", "text", "escape", "backtick", "regex", "pattern", "join array", "replace text", "validate input", "split string", "parse text", "special character", "concatenate", "substring" -->
<!-- CONSTRAINTS: AHK v2 uses backtick (`) as the sole escape character — never backslash; `"\n"` is literally backslash-n, not a newline. Regex mode flags use `option)` prefix syntax (`i)pattern`), never suffix or inline flag syntax. RegExMatch() output var requires `&` ByRef prefix — omitting it silently prevents capture group population. Array objects have no `.Join()` method — use a ternary-separator loop or RTrim() pattern. StrReplace() returns a new string and never modifies in place — always capture the return value. -->
<!-- CROSS-REF: Module_FileSystem.md, Module_GUI.md, Module_InputAndHotkeys.md, Module_Errors.md, Module_Arrays.md -->
<!-- VERSION: AHK v2.0+ -->

## V1 → V2 BREAKING CHANGES

| v1 pattern (LLM commonly writes) | v2 correct form | Consequence |
|----------------------------------|-----------------|-------------|
| `"\n"` or `"\t"` as newline/tab | `` "`n" `` or `` "`t" `` | Backslash has no special meaning in AHK strings — `"\n"` is the 2-char string `\n`, not a newline |
| `StringReplace, out, str, old, new` | `StrReplace(str, old, new)` | v1 command syntax throws a parse error in v2 |
| `RegExMatch(str, pat, match)` (bare output var) | `RegExMatch(str, pat, &match)` | v1 used bare variable name; v2 requires `&` ByRef prefix — match object is never populated without it, `match[1]` throws UnsetError |
| `StringLower, out, in` / `StringUpper, out, in` | `StrLower(str)` / `StrUpper(str)` | v1 command syntax — throws parse error in v2 |
| `IfInString, var, needle` | `if InStr(var, needle)` | v1 control-flow command — not valid AHK v2 syntax |
| `StringTrimLeft, out, in, n` | `LTrim(str)` or `SubStr(str, n+1)` | v1 command removed in v2; `LTrim()` strips all leading whitespace, not a fixed count |
| `"hello/i"` or `"(?i)hello"` for case-insensitive regex | `"i)hello"` | AHK v2 regex options must prefix the pattern as `option)` — any other placement is silently ignored |
| `StringLen, out, str` | `StrLen(str)` or `str.Length` | v1 command syntax — throws parse error in v2 |

## API QUICK-REFERENCE

### String Functions

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `StrLen()` | `StrLen(str)` | Integer (char count) | — | Equivalent to `str.Length` property on a string value |
| `StrUpper()` | `StrUpper(str)` | String | — | Returns uppercase copy — does not modify the original variable |
| `StrLower()` | `StrLower(str)` | String | — | Returns lowercase copy — does not modify the original variable |
| `StrTitle()` | `StrTitle(str)` | String | — | Returns title-case copy — capitalises first letter of each word |
| `StrCompare()` | `StrCompare(str1, str2, caseSense?)` | Integer (neg / 0 / pos) | — | caseSense: false=case-insensitive (default), true=case-sensitive, `"Locale"`=locale-aware; negative means str1 < str2 |
| `StrReplace()` | `StrReplace(haystack, needle, replacement?, caseSense?, &count?, limit?)` | String | — | Returns modified string; optional `&count` receives the number of replacements made |
| `StrSplit()` | `StrSplit(str, delimiters?, padding?, maxParts?)` | Array | — | Returns an Array of substrings; delimiters may be a string or an Array of strings |
| `InStr()` | `InStr(haystack, needle, caseSense?, startPos?, occurrence?)` | Integer (1-based pos) | — | Returns 0 if not found; negative `startPos` searches from end; `occurrence` selects nth match |
| `SubStr()` | `SubStr(str, startPos, length?)` | String | — | 1-based; negative `startPos` counts from string end; omit `length` for remainder of string |
| `Format()` | `Format(formatStr, values*)` | String | — | `{1}` `{2}` positional placeholders; `{1:.2f}` numeric; `{1:05d}` zero-padded; `{1:U}` uppercase |
| `Trim()` | `Trim(str, chars?)` | String | — | Strips both leading and trailing chars (default: whitespace and null) |
| `LTrim()` | `LTrim(str, chars?)` | String | — | Left-trim only |
| `RTrim()` | `RTrim(str, chars?)` | String | — | Right-trim only; used in array-join pattern to remove trailing separator |
| `String()` | `String(value)` | String | — | Converts Integer or Float to its string representation; calls `ToString()` on objects — throws `MethodError` if no `ToString()` method exists |
| `VarSetStrCapacity()` | `VarSetStrCapacity(&var, charCount?)` | Integer (char capacity) | — | Pre-allocates buffer for DllCall/repeated concatenation optimisation only; call with -1 to resync length after DllCall mutation |

### Regex Functions

| Method/Property | Signature | Returns | Throws | Notes |
|----------------|-----------|---------|--------|-------|
| `RegExMatch()` | `RegExMatch(haystack, needleRegex, &matchObj?, startPos?)` | Integer (1-based pos or 0) | `Error` on invalid pattern | `&matchObj` populated ByRef — `&` prefix is mandatory |
| `RegExReplace()` | `RegExReplace(haystack, needleRegex, replacement?, &count?, limit?, startPos?)` | String | `Error` on invalid pattern | `$1`/`$2` or `${name}` for group backreferences in replacement |

### RegExMatch Object (matchObj)

| Property | Returns | Notes |
|----------|---------|-------|
| `matchObj[0]` | String | The entire matched string |
| `matchObj[n]` | String | nth numbered capture group value (1-based) |
| `matchObj["name"]` | String | Named capture group value; group defined as `(?P<n>...)` in pattern |
| `matchObj.Pos[n]` | Integer | 1-based position of nth group within the haystack |
| `matchObj.Len[n]` | Integer | Character length of nth group |
| `matchObj.Count` | Integer | Total number of capture groups in the pattern |

## AHK V2 CONSTRAINTS

- **Backtick (`` ` ``) is AHK v2's only escape character** — never use `\n`, `\t`, or any backslash escape inside AHK string literals; `"hello\n"` is the 7-character literal `hello\n`, not a newline — use `` "`n" `` instead — applies to all string literals including regex patterns assigned as variables.
- **Regex mode flags use `option)` prefix syntax only** — `"i)pattern"` for case-insensitive; `"im)pattern"` for combined options; placing flags anywhere other than the leading `option)` prefix is silently ignored, producing incorrect match behaviour.
- **RegExMatch output var requires `&` ByRef prefix** — `RegExMatch(str, pat, &match)` not `RegExMatch(str, pat, match)`; without `&`, AHK v2 passes the variable by value, the match object is never populated, and any `match[1]` access throws UnsetError.
- **Array objects have no `.Join()` method** — join arrays with a `for` loop using a ternary separator, or append a separator in the loop then `RTrim()` the trailing delimiter; calling `.Join()` throws a MethodError.
- **Single and double quoted strings are identical in behaviour** — both interpret backtick escape sequences in the same way; choose the quote type that minimises escaping: use single quotes when the string contains double quotes, use double quotes when the string contains apostrophes.
- **`StrReplace()` returns the modified string; it does not modify in place** — always capture the return value: `str := StrReplace(str, old, new)`; discarding the return value silently produces no change.
  - ✗ `StrReplace(str, "old", "new")` — result discarded, original unchanged
  - ✓ `str := StrReplace(str, "old", "new")` — captures and reassigns the result
- **`InStr()` returns a 1-based position, not a boolean** — return value of `0` means not found; using the result as a boolean works only because `0` is falsy, but an explicit `!= 0` comparison is clearer and avoids confusion between "found at position 1" and "not found".
  - ✗ `if InStr(text, needle)` — ambiguous when needle is at position 0 (impossible) but unclear intent
  - ✓ `if InStr(text, needle) != 0` — explicit not-found test
- **`StrCompare()` returns a signed integer, not a boolean** — negative means first string precedes second; 0 means equal; positive means first string follows second — do not compare the result against `true`/`false`.
  - ✗ `if StrCompare(a, b) = true` — never matches; result is never exactly true
  - ✓ `if StrCompare(a, b) = 0` — correct equality test

Safe-access priority order for text processing operations:
  1. Built-in functions (`StrReplace`, `InStr`, `SubStr`, `Trim`) — single-call, no loop, no error path
  2. Ternary separator pattern — array join without a `.Join()` method call, exact separator control
  3. `RTrim()` trailing-separator pattern — simpler than ternary when element order matters less than clarity
  4. `try/catch` on `RegExMatch`/`RegExReplace` — only when the pattern itself is user-supplied and malformed input must be caught

Unset variable handling: always declare the match variable before calling `RegExMatch()` if the pattern might not match — accessing `match[1]` when `RegExMatch()` returned 0 means `match` was never assigned, causing UnsetError.

## AGENT QA CHECKLIST

- [ ] Did I pass `&match` (with the `&` prefix) as the third argument to every `RegExMatch()` call?
- [ ] Did I capture the return value of every `StrReplace()` call — `str := StrReplace(str, ...)`?
- [ ] Did I use `` `n `` / `` `t `` (backtick escapes) and NOT `\n` / `\t` in all string literals?
- [ ] Did I use `option)pattern` prefix syntax (e.g., `"i)pattern"`) for every regex with mode flags?
- [ ] Did I avoid calling `.Join()` on an Array and instead use a loop or `RTrim()` pattern?

## RUNTIME ERROR MAPPING

| Exception Class | Trigger Condition | Detection Code | Fix |
|----------------|-------------------|----------------|-----|
| `UnsetError` | Accessing `match[1]` after `RegExMatch()` with no `&` prefix on the output var, or when the pattern did not match | `e.Message` contains "unset" | Add `&` prefix: `RegExMatch(str, pat, &match)`; guard with `if RegExMatch(...) != 0` before accessing capture groups |
| `MethodError` | Calling `.Join()` on an Array returned by `StrSplit()` or any Array object | `e.Message` contains "Join" | Replace with ternary separator `for` loop or `RTrim()` pattern — Array has no `.Join()` method |
| `Error` | Passing a malformed pattern string to `RegExMatch()` or `RegExReplace()` — unclosed group, invalid option prefix, or unbalanced bracket | `e.Message` contains pattern text | Validate user-supplied patterns with `try/catch`; log `e.Message` to surface the PCRE syntax error |

## TIER 1 — String Building, Concatenation, and Quote Strategy
> METHODS COVERED: RTrim · .= operator · A_Index

String concatenation in AHK v2 uses the `.=` compound operator to incrementally build a string from parts. Array joining has no built-in `.Join()` method — use either a ternary separator inside the loop (exact output) or an append-then-RTrim pattern (simpler loop body). Choose single vs double quotes to minimise escape characters rather than using `\` escapes, which have no effect in AHK string literals.

```ahk
; ✓ .= is the correct v2 string-append operator — returns and reassigns in one step
result := ""
result .= "First part"
result .= " and second part"

; ✓ Array assigned to a named variable first — .Length is a property, not a method
myArr := ["apple", "banana", "orange"]
arrayJoining := ""
for item in myArr
    arrayJoining .= item (A_Index < myArr.Length ? ", " : "")

; ✓ RTrim pattern — simpler loop body, trailing separator stripped after the loop
alternativeJoining := ""
for item in myArr
    alternativeJoining .= item ", "
result := RTrim(alternativeJoining, ", ")

; ✓ Single quotes avoid escaping the double quotes inside the string
singleQuotes := 'He said "Hello"'

; ✓ Double quotes avoid escaping the apostrophe inside the string
doubleQuotes := "It's working"

; ✗ Array object has no .Join() method — throws MethodError
; result := myArr.Join(", ")   ; → MethodError: no method named Join
```

## TIER 2 — Escape Sequences: The Backtick System
> METHODS COVERED: (no function calls — escape sequence reference tier)

AHK v2 uses backtick (`` ` ``) as its sole escape character inside string literals. Backslash has no special meaning and produces itself literally. This tier covers the complete escape sequence inventory and context-specific escaping rules for hotstrings, GUI controls, command-line arguments, and INI files.

```ahk
; ✓ Double-backtick produces a single literal backtick character
literalBacktick := "This is a backtick: ``"

; ✓ Backtick-quote escapes a quote character inside same-type quoted string
quotesInString := "He said `"Hello`" to me"

; ✓ `n is the correct AHK v2 newline — not \n
newlineText := "First line`nSecond line"

; ✓ `t is the correct AHK v2 tab character — not \t
tabSeparated := "Name`tAge`tCity"

; ✓ Backtick-semicolon escapes a semicolon that would otherwise begin a comment
literalSemicolon := "Command `; parameter"

; ✓ `a produces the bell/alert character (ASCII 7)
alertSound := "Warning`a"

; ✓ `r is carriage return (ASCII 13) — used with `n for Windows CRLF line endings
windowsNewline := "Line1`r`nLine2"

; ✓ Single-quote string wraps paths containing double quotes — no escaping needed
pathWithSpaces := '"C:\Program Files\My App\program.exe"'

; ✗ Backslash has no special meaning in AHK strings — "\n" is literally backslash-n
; badNewline := "First line\nSecond line"   ; → literal 13-char string, not two lines
```

Essential Escape Sequences:
- ` `` ` → `` ` `` (literal backtick)
- `` `" `` → `"` (literal double quote inside double-quoted string)
- `` `' `` → `'` (literal single quote inside single-quoted string)
- `` `n `` → newline (LF, ASCII 10)
- `` `r `` → carriage return (CR, ASCII 13)
- `` `t `` → tab (ASCII 9)
- `` `s `` → space (ASCII 32)
- `` `b `` → backspace (ASCII 8)
- `` `: `` → `:` (literal colon — required in hotstring definitions)
- `` `; `` → `;` (literal semicolon — prevents comment parse)
- `` `a `` → alert/bell (ASCII 7)
- `` `v `` → vertical tab (ASCII 11)
- `` `f `` → form feed (ASCII 12)

Context-specific rules:
- **Hotstrings**: escape colons with `` `: ``
- **GUI controls**: use `` `n `` for line breaks in multi-line edit controls
- **Command-line arguments**: quote entire arguments containing spaces
- **INI files**: be careful with `=` and `[` in value strings

## TIER 3 — Built-in String Methods
> METHODS COVERED: Trim · LTrim · RTrim · StrUpper · StrLower · StrTitle · StrReplace · StrSplit · InStr · SubStr · StrLen · StrCompare

AHK v2 provides a complete set of built-in string functions that cover nearly all common manipulation tasks. Prefer these over manual character loops — they are implemented in native code, handle Unicode correctly, and express intent more clearly. None of these functions modify the input string in place; all return a new string.

```ahk
; ✓ Trim() removes leading and trailing whitespace in one call
text := "  Hello World  "
cleaned := Trim(text)              ; "Hello World"

; ✓ StrUpper/StrLower/StrTitle return copies — assign back to mutate the variable
upper := StrUpper(text)            ; "  HELLO WORLD  "
lower := StrLower(text)            ; "  hello world  "
titled := StrTitle("hello world")  ; "Hello World"

; ✓ StrCompare returns signed integer — 0 means equal, negative means str1 < str2
if StrCompare("apple", "banana") < 0
    MsgBox("apple sorts before banana")

; ✓ StrCompare with true for case-sensitive comparison
if StrCompare("Hello", "hello", true) != 0
    MsgBox("Case-sensitive: strings differ")

; ✓ StrReplace returns the modified string — result must be captured
replaced := StrReplace(text, "World", "Universe")  ; "  Hello Universe  "

; ✓ StrSplit returns an Array — access elements by 1-based index
csv := "apple,banana,orange"
parts := StrSplit(csv, ",")
; parts[1] = "apple", parts[2] = "banana", parts[3] = "orange"

; ✓ StrSplit with padding strips surrounding whitespace from each part
spacedCsv := "apple , banana , orange"
trimmedParts := StrSplit(spacedCsv, ",", " ")
; trimmedParts[1] = "apple", trimmedParts[2] = "banana"

; ✓ InStr returns 1-based position or 0 — explicit != 0 is clearer than bare truthiness
found := InStr(text, "World")
if found != 0
    MsgBox("Found at position " found)

; ✓ SubStr with 1-based index — negative startPos counts from string end
extracted := SubStr(text, 3, 5)   ; "Hello" (start at char 3, take 5 chars)
last4 := SubStr(text, -4)         ; last 4 characters of the string

; ✗ StrReplace result discarded — original variable unchanged
; StrReplace(text, "Hello", "Hi")  ; → silent no-op, text still "  Hello World  "

; ✗ Zero-based index on Array returned by StrSplit — AHK v2 arrays are 1-based
; first := parts[0]                ; → UnsetItemError
```

## TIER 4 — Regex Fundamentals: RegExMatch and RegExReplace
> METHODS COVERED: RegExMatch · RegExReplace

AHK v2 uses PCRE regex with a unique option syntax: mode flags prefix the pattern as `option)`, e.g., `"i)pattern"`. `RegExMatch()` returns the 1-based match position (0 = no match) and populates a match object via `&matchObj`. `RegExReplace()` returns the modified string and supports `$1`/`$2` or `${name}` backreferences in the replacement.

```ahk
; ✓ i) prefix enables case-insensitive matching — option before the closing paren
email := "USER@DOMAIN.COM"
if RegExMatch(email, "i)^[^@]+@[^@]+\.[^@]+$")
    MsgBox("Valid email (case-insensitive)")

; ✓ m) prefix makes ^ and $ match line boundaries, not just string start/end
multilineText := "Line 1`nLine 2`nLine 3"
if RegExMatch(multilineText, "m)^Line 2$")
    MsgBox("Found Line 2 at start of line")

; ✓ &match ByRef prefix required — without & the match object is never populated
phonePattern := "(\d{3})-(\d{3})-(\d{4})"
if RegExMatch("123-456-7890", phonePattern, &match) {
    area     := match[1]   ; "123"
    exchange := match[2]   ; "456"
    number   := match[3]   ; "7890"
}

; ✓ matchObj.Count reveals the number of capture groups in the pattern
MsgBox("Groups captured: " match.Count)  ; 3

; ✓ RegExReplace returns modified string — capture the return value
text := "Hello 123 World 456"
cleaned := RegExReplace(text, "\D+", " ")              ; " 123  456"
swapped := RegExReplace(text, "(\w+) (\d+)", "$2 $1")  ; "123 Hello 456 World"

; ✓ Named capture groups via (?P<n>...) — access by string key
RegExMatch("Price: $25.99", "\$(?P<dollars>\d+)\.?(?P<cents>\d*)", &priceMatch)
dollars := priceMatch["dollars"]  ; "25"
cents   := priceMatch["cents"]    ; "99"

; ✓ x) option allows comments and insignificant whitespace in pattern
annotatedPat := "x)
    (\d{3})   # area code
    -
    (\d{3})   # exchange
    -
    (\d{4})   # subscriber"
RegExMatch("123-456-7890", annotatedPat, &m)

; ✗ Bare output var without & — match object never populated
; RegExMatch("123-456-7890", phonePattern, match)  ; → match[1] throws UnsetError

; ✗ Suffix flag syntax from PCRE/Perl/Python — silently ignored in AHK
; if RegExMatch(email, "^[^@]+@[^@]+\.[^@]+$/i")   ; → flags after / have no effect
```

Regex Options (prefix syntax only):
- `i)` → case-insensitive matching
- `m)` → multiline mode (`^` and `$` match line boundaries)
- `s)` → single-line mode (`.` matches newlines)
- `x)` → extended mode (ignore unescaped whitespace, allow `#` comments in pattern)
- Combine: `"im)pattern"` for both multiline and case-insensitive

Common Character Classes:
- `\d` → digit `[0-9]` · `\D` → non-digit
- `\w` → word character `[a-zA-Z0-9_]` · `\W` → non-word
- `\s` → whitespace `[ \t\n\r\f]` · `\S` → non-whitespace

Capture Groups:
- `()` → numbered capturing group
- `(?:)` → non-capturing group
- `(?P<n>)` → named capture group
- `$1`, `$2`, `${name}` → backreferences in replacement string

## TIER 5 — String Validation Classes
> METHODS COVERED: RegExMatch · StringValidator.IsEmail · StringValidator.IsPhone · StringValidator.HasOnlyAlphanumeric · StringValidator.ValidateInput

Organise related validation logic into static-method classes that return consistent results. Regex-based validators compose well with variadic rule functions, enabling flexible multi-rule validation without conditional chains. This pattern keeps validation rules co-located and reusable across scripts. Note: `StringValidator` methods listed above are user-defined class methods demonstrated in this tier — they are not AHK v2 built-ins.

```ahk
; ✓ Static methods return RegExMatch result directly — 0 or match position, truthy/falsy
class StringValidator {
    static IsEmail(str) {
        return RegExMatch(str, "^[^@\s]+@[^@\s]+\.[^@\s]+$")
    }

    static IsPhone(str) {
        return RegExMatch(str, "^\d{3}-\d{3}-\d{4}$")
    }

    static HasOnlyAlphanumeric(str) {
        return RegExMatch(str, "^[a-zA-Z0-9]+$")
    }

    ; ✓ Variadic rules* parameter accepts any number of callable validators
    static ValidateInput(str, rules*) {
        errors := []
        for rule in rules {
            if !rule(str)
                errors.Push("Validation failed")
        }
        return errors.Length = 0
    }
}

; ✓ Pass validator methods as first-class function references — no Func() wrapper needed in v2
isValid := StringValidator.ValidateInput(
    "test@example.com",
    StringValidator.IsEmail,
    (s) => StrLen(s) <= 100
)

; ✗ Func() wrapper syntax from v1 — not required in v2 for named functions
; isValid := StringValidator.ValidateInput(str, Func("StringValidator.IsEmail"))  ; → unneeded
```

## TIER 6 — StringBuilder and EscapeValidator: Advanced String Construction
> METHODS COVERED: StringBuilder.Add · StringBuilder.AddLine · StringBuilder.AddFormattedLine · StringBuilder.Join · StringBuilder.ToString · EscapeValidator.ValidateString · EscapeValidator.SanitizeForDisplay · EscapeValidator.EscapeForCommand · StrReplace · InStr · RegExMatch · Format · RTrim

`StringBuilder` amortises repeated string allocation by collecting parts in an Array and joining once, which outperforms incremental `.=` concatenation for large fragment counts. `EscapeValidator` handles the three most common safe-text scenarios: detecting malformed escape sequences in user input, sanitising strings for display, and quoting command-line arguments containing spaces or tabs. Note: `StringBuilder` and `EscapeValidator` methods are user-defined class methods demonstrated in this tier — they are not AHK v2 built-ins.

```ahk
; ✓ Fluent interface — each method returns this for chaining
class StringBuilder {
    __New() {
        this.parts := []
    }

    Add(text) {
        this.parts.Push(text)
        return this
    }

    AddLine(text) {
        this.parts.Push(text . "`n")
        return this
    }

    ; ✓ Format() handles positional placeholders — cleaner than manual concatenation
    AddFormattedLine(fmt, values*) {
        line := Format(fmt, values*)
        this.parts.Push(line . "`n")
        return this
    }

    ; ✓ Ternary separator in loop — no .Join() method exists on Array
    Join(separator := "") {
        result := ""
        for i, part in this.parts
            result .= part (i < this.parts.Length ? separator : "")
        return result
    }

    ; ✓ RTrim removes trailing newline from the last AddLine call
    ToString() {
        return RTrim(this.Join(), "`n")
    }
}

class EscapeValidator {
    static ValidateString(str) {
        errors := []

        if InStr(str, "``") && !InStr(str, "````")
            errors.Push("Unescaped backtick detected")

        ; ✓ Double-backtick in pattern literal = one backtick; character class excludes valid sequences
        if RegExMatch(str, "``[^``;:nrtbsvaf`"'``]")
            errors.Push("Invalid escape sequence found")

        return errors
    }

    static SanitizeForDisplay(str) {
        ; ✓ StrReplace("``", "````") replaces one backtick with two — escape for display
        str := StrReplace(str, "``", "````")
        str := StrReplace(str, "`n", " ")
        str := StrReplace(str, "`t", " ")
        return str
    }

    static EscapeForCommand(str) {
        if InStr(str, " ") || InStr(str, "`t")
            ; ✓ Single-quoted outer string; replacement '``"' produces the two-char sequence `"
            return '"' . StrReplace(str, '"', '``"') . '"'
        return str
    }
}

; ✓ Fluent chaining — each AddLine returns the StringBuilder instance
systemReport := StringBuilder()
systemReport.AddLine("System Information")
            .AddLine("==================")
            .AddLine("OS: " . A_OSVersion)
            .AddLine("User: " . A_UserName)
            .AddLine("Computer: " . A_ComputerName)

; ✓ Continuation expression — parentheses allow multi-line expression, variables evaluated
winWidth  := 1280
winHeight := 720
configFile := (
    "[Settings]`n"
    "AutoStart=true`n"
    "Theme=dark`n"
    "WindowSize=" winWidth "x" winHeight "`n"
    "LastUpdate=" A_Now
)

; ✓ Validation errors collected in Array — joined with loop because Array has no .Join()
try {
    userInput := InputBox("Enter text:", "Input").Value
    errors := EscapeValidator.ValidateString(userInput)
    if errors.Length > 0 {
        errMsg := ""
        for e in errors
            errMsg .= e . ", "
        throw Error("Invalid input: " . RTrim(errMsg, ", "))
    }
    processedText := EscapeValidator.SanitizeForDisplay(userInput)
} catch Error as e {
    MsgBox("Error: " . e.Message)
}
```

### Performance Notes

**Prefer built-in string functions over manual loops** — `StrReplace()`, `InStr()`, `SubStr()`, and `Trim()` are implemented in native code; a manual character loop for the same task runs 10–100× slower for typical string lengths.

**Regex compilation cost** — AHK v2 caches the 100 most recently compiled patterns in memory, so recompilation only occurs on a cache miss. For patterns used in tight loops, assigning the pattern to a variable and reusing the variable across calls is clearer and avoids accidental cache eviction from cache-size overflow.

**`StrReplace()` vs regex for literal substitution** — `StrReplace()` is faster than `RegExReplace()` for fixed-string replacement because no pattern compilation occurs; use `StrReplace()` whenever the needle contains no metacharacters.

**StringBuilder pattern vs inline `.=` concatenation** — for building strings from fewer than ~20 parts, inline `.=` concatenation is fast enough; the `StringBuilder` class (TIER 6) amortises allocation cost by collecting parts in an Array and joining once, which matters when building strings from hundreds of fragments in a loop.

**`StrSplit()` result size** — splitting a large text by a single character produces an Array with one element per separator occurrence; for very large inputs, process line-by-line with `FileOpen` + `.ReadLine()` (see Module_FileSystem.md) rather than splitting an entire file content string into memory at once.

**`VarSetStrCapacity()` for hot loops** — pre-allocating with `VarSetStrCapacity(&buf, expectedLen)` before a tight `.=` loop avoids repeated internal buffer reallocation; this is a micro-optimisation relevant only when concatenating thousands of fragments in performance-critical code.

## DROP-IN RECIPES

```ahk
; SafeRegExExtract — wraps RegExMatch with error handling for user-supplied patterns
; Returns the specified capture group string, or defaultVal if no match or invalid pattern
; ✓ try/catch catches Error from invalid pattern — never throws to caller on bad input
SafeRegExExtract(haystack, pattern, groupIndex := 0, defaultVal := "") {
    if !(haystack is String)
        throw TypeError("SafeRegExExtract: haystack must be a string", -1)
    if !(pattern is String) || pattern = ""
        throw TypeError("SafeRegExExtract: pattern must be a non-empty string", -1)
    try {
        if !RegExMatch(haystack, pattern, &m)
            return defaultVal
        return m[groupIndex]
    } catch Error as e {
        throw ValueError("SafeRegExExtract: invalid regex pattern — " e.Message, -1)
    }
}
; Call site: email := SafeRegExExtract(rawLine, "[\w.]+@[\w.]+", 0, "")

; JoinArray — canonical array-join without .Join() (which does not exist on Array)
; Returns elements joined by sep; handles empty arrays and non-string elements via String()
; ✓ Validates input type — throws TypeError rather than silently producing wrong output
JoinArray(arr, sep := ", ") {
    if !(arr is Array)
        throw TypeError("JoinArray: first argument must be an Array", -1)
    if arr.Length = 0
        return ""
    result := ""
    for i, item in arr
        result .= String(item) (i < arr.Length ? sep : "")
    return result
}
; Call site: MsgBox(JoinArray(["alpha", "beta", "gamma"], " | "))

; NormalizeWhitespace — trim and collapse internal runs of whitespace to single spaces
; Useful for cleaning user input, CSV fields, or text extracted from GUI controls
; ✓ Uses RegExReplace for internal whitespace — faster than a manual character loop
NormalizeWhitespace(str) {
    if !(str is String)
        throw TypeError("NormalizeWhitespace: argument must be a string", -1)
    str := Trim(str)
    str := RegExReplace(str, "\s+", " ")
    return str
}
; Call site: clean := NormalizeWhitespace("  Hello   World  ")  ; → "Hello World"
```

## ANTI-PATTERNS

| Pattern | Wrong | Correct | LLM Common Cause |
|---------|-------|---------|------------------|
| Backslash escape sequences | `"\n"` `"\t"` in AHK string | `` "`n" `` `` "`t" `` | All major language training data uses `\n`; AHK v2 uses backtick — strongest source of v1/cross-language regression |
| v1 command-style string ops | `StringReplace, out, str, old, new` | `str := StrReplace(str, old, new)` | AHK v1 training data; command syntax was dominant and widely documented |
| Bare output var in RegExMatch | `RegExMatch(str, pat, match)` | `RegExMatch(str, pat, &match)` | AHK v1 used bare variable names for output; v2 `&` ByRef prefix is easy to overlook |
| Calling Array .Join() | `arr.Join(", ")` | Loop with ternary separator or `RTrim()` pattern | JavaScript/Python training data; both languages have a native `.join()` method |
| Wrong regex flag placement | `"pattern/i"` or `"(?i)pattern"` | `"i)pattern"` | Perl/PCRE/JS regex flag syntax is pervasive in training data; AHK option prefix syntax is unique |
| StrReplace result discarded | `StrReplace(str, old, new)` without capture | `str := StrReplace(str, old, new)` | Python `str.replace()` in typical usage context trains habit of ignoring return value |
| Treating StrCompare as boolean | `if StrCompare(a, b)` to test equality | `if StrCompare(a, b) = 0` | C `strcmp()` / Python `==` habits; `StrCompare` returns a signed integer not a boolean |

## SEE ALSO

> This module does NOT cover: reading and writing text files with encoding flags → see Module_FileSystem.md
> This module does NOT cover: GUI Edit/Text control content manipulation and multi-line display → see Module_GUI.md
> This module does NOT cover: hotstring trigger definitions, colon-colon syntax, and hotstring options → see Module_InputAndHotkeys.md
> This module does NOT cover: try/catch patterns for regex errors, Error subclasses, and structured error propagation → see Module_Errors.md

- `Module_FileSystem.md` — FileOpen/FileRead/FileAppend with explicit encoding flags; line-by-line text iteration that avoids loading entire file content into a string.
- `Module_Arrays.md` — typed Array construction, Push/Pop/InsertAt, and iteration patterns used to store StrSplit results and StringValidator error collections.
- `Module_Errors.md` — try/catch wrapping for user-supplied regex patterns, Error subclass selection, and structured error propagation from validation classes.
- `Module_GUI.md` — GUI Edit control content, multi-line text display, and how `` `n `` interacts with Edit control line endings.
- `Module_InputAndHotkeys.md` — hotstring definitions, colon escaping in `::trigger::replacement` syntax, and hotstring option flags.