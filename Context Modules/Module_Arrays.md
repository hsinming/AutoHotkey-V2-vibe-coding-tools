# Module_Arrays.md
<!-- DOMAIN: Arrays -->
<!-- SCOPE: Key-value storage with Map(), deep-clone of circular-reference graphs, GUI control binding, and file-path batch operations are not covered — see Module_Objects.md, Module_GUI.md, and Module_FileSystem.md respectively. -->
<!-- TRIGGERS: Array(), Push(), Pop(), InsertAt(), RemoveAt(), Clone(), Get(), Has(), Delete(), "store multiple values", "iterate collection", "list", "vector", "remove duplicates", "deep copy", "transform elements", ArrayMap, QuickSort, Unique, IndexOf, Filter, Reduce -->
<!-- CONSTRAINTS: Arrays are 1-based — arr[0] always throws IndexError; arr[-1] is the last element (reverse indexing). Array objects have NO built-in .Sort() method; use a custom QuickSort() with a comparator callback. Array(N) does NOT pre-allocate N slots — it creates [N] (one element whose value is N); use arr.Capacity := N instead. Fat-arrow functions with block bodies `(x) => { return x }` are invalid in AHK v2.0; use named nested functions for multi-statement callbacks. Never name a user function "Map" — it shadows the built-in Map class globally. -->
<!-- CROSS-REF: Module_Objects.md, Module_Errors.md, Module_GUI.md, Module_Functions.md -->
<!-- VERSION: AHK v2.0+ -->

## V1 → V2 BREAKING CHANGES

| v1 pattern (LLM commonly writes) | v2 correct form | Consequence |
|----------------------------------|-----------------|-------------|
| `array[0]` for first element | `array[1]` | Throws `IndexError` — 0 is not a valid Array index in v2 |
| `array.Sort()` called directly | Custom `QuickSort(array, comparator?)` | `TypeError` — Array objects have no built-in `.Sort()` method in v2 |
| `Array(10)` to pre-allocate 10 slots | `arr := [], arr.Capacity := 10` | Creates `[10]` — a one-element array containing the value `10`, not 10 empty slots |
| `new Array()` constructor syntax | `Array()` or `[]` literal | `NameError` — `new` is treated as an unresolved variable reference at runtime in AHK v2 |
| `(x) => { return x * 2 }` fat-arrow block body | Named nested function inside the caller | `SyntaxError` in v2.0 — block bodies on fat-arrow functions are a v2.1 alpha feature only |
| `for i, v in array` assuming `i` starts at 0 | `i` starts at `1` — first iteration yields `i = 1` | Off-by-one logic errors throughout loop body |
| User-defined function named `Map` | Rename to `ArrayMap` | Shadows the built-in `Map` class — `Map()` calls in scope will fail with `TypeError` |

## API QUICK-REFERENCE

### Array (built-in)

| Method / Property | Signature | Returns | Throws | Notes |
|-------------------|-----------|---------|--------|-------|
| `[]` literal | `[val1, val2, ...]` | Array | — | Preferred creation syntax for known values |
| `Array()` | `Array(val*)` | Array | — | `Array()` with no args creates empty array; `Array(N)` creates `[N]`, not N slots |
| `.Push()` | `.Push(val*)` | — | — | Appends one or more values; no return value |
| `.Pop()` | `.Pop()` | Any (removed value) | Error (if empty) | Equivalent to `.RemoveAt(-1)` |
| `.InsertAt()` | `.InsertAt(index, val*)` | — | ValueError (index out of bounds) | Negative index counts from end; index 0 is same as Length+1 |
| `.RemoveAt()` | `.RemoveAt(index, Length?)` | Any (removed value, if Length omitted) | ValueError (range out of bounds) | Negative index counts from end; elements shift left |
| `.Get()` | `.Get(index, default?)` | Any | IndexError (out of range); UnsetItemError (no value & no default & no `.Default`) | Safe access; omitting default is equivalent to direct bracket access |
| `.Has()` | `.Has(index)` | Integer (non-zero if valid and set) | — | Returns 0 for valid-range but `.Delete()`d slots |
| `.Delete()` | `.Delete(index)` | Any (removed value, blank if none) | ValueError (index out of range) | Does NOT shift elements or change `.Length` |
| `.Clone()` | `.Clone()` | Array (shallow copy) | — | Nested objects are shared references, not duplicated |
| `.Length` | `.Length` | Integer | — | Readable and writable; setting lower truncates, setting higher adds unset slots |
| `.Capacity` | `.Capacity` | Integer | — | Pre-allocate backing storage without adding elements; avoids repeated realloc on `.Push()` |
| `.Default` | `.Default := value` | Any | — | When set, returned instead of throwing UnsetItemError for unset slots; does not suppress IndexError |
| `.__Item` | `arr[index]` / `arr.__Item[index]` | Any | IndexError (out of bounds) | Invoked implicitly by bracket notation; override to customise indexing behaviour |
| `__Enum` | `for index, value in array` | Enumerator | — | 2-var form yields 1-based index and value; `for value in array` yields value only; unset slots still enumerated (value becomes uninitialized) |

### Global Functions Used with Arrays

| Function | Signature | Returns | Throws | Notes |
|----------|-----------|---------|--------|-------|
| `Type()` | `Type(obj)` | String | — | Returns `"Array"` for Array objects — canonical type guard |
| `IsObject()` | `IsObject(val)` | Integer (0/1) | — | Returns `1` for any object including Array, Map, user classes |
| `IsSet()` | `IsSet(var)` | Integer (0/1) | — | Tests whether a variable or optional parameter was assigned a value |
| `Mod()` | `Mod(dividend, divisor)` | Integer/Float | ZeroDivisionError | Integer modulus — used in filter predicates and chunking |

### Module Utility Functions (defined in this module)

| Function | Signature | Returns | Throws | Notes |
|----------|-----------|---------|--------|-------|
| `IsValidIndex()` | `IsValidIndex(array, index)` | Integer (0/1) | — | Returns `1` if `index` is in the 1-based valid range |
| `GetRange()` | `GetRange(array, start, end)` | Array | — | Returns a new array slice from `start` to `end` (inclusive, 1-based) |
| `SafeInsert()` | `SafeInsert(array, index, value)` | Array | — | Clamps `index` to valid range then calls `.InsertAt()` |
| `RemoveValue()` | `RemoveValue(array, value)` | Array | — | Removes all occurrences of `value` in-place; returns the mutated array |
| `IndexOf()` | `IndexOf(array, value, fromIndex?)` | Integer | — | Returns 1-based index or `0` if not found |
| `LastIndexOf()` | `LastIndexOf(array, value)` | Integer | — | Scans right-to-left; returns 1-based index or `0` |
| `Contains()` | `Contains(array, value)` | Integer (0/1) | — | Boolean wrapper around `IndexOf()` |
| `FindIndex()` | `FindIndex(array, callback)` | Integer | — | Returns first index where `callback(value, index, array)` is truthy, else `0` |
| `ArrayMap()` | `ArrayMap(array, callback)` | Array | — | Returns new array of `callback(value, index, array)` results — immutable style |
| `Filter()` | `Filter(array, callback)` | Array | — | Returns new array of elements where `callback(value, index, array)` is truthy |
| `Reduce()` | `Reduce(array, callback, initialValue?)` | Any | Error (empty array with no initialValue) | Folds array into single value |
| `DeepClone()` | `DeepClone(obj)` | Array/Map/Object | — | Recursively copies Array, Map, and Object graphs — does not handle circular references |
| `QuickSort()` | `QuickSort(array, callback?, left?, right?)` | Array | — | In-place sort; `callback(a, b)` returns negative/zero/positive |
| `SortBy()` | `SortBy(array, fields*)` | Array | — | Multi-field sort on arrays of Map objects |
| `Unique()` | `Unique(array)` | Array | — | Returns new array with duplicates removed, preserving first-seen order |
| `UniqueBy()` | `UniqueBy(array, keyFunc)` | Array | — | Deduplication using a custom key extractor callback |
| `FastContains()` | `FastContains(array, value)` | Integer (0/1) | — | Builds a Map lookup for O(1) membership test — use when querying the same array repeatedly |
| `ModifyInPlace()` | `ModifyInPlace(array, modifier)` | Array | — | Applies `modifier(value)` to every element without allocating a new array |
| `Difference()` | `Difference(array1, array2)` | Array | — | Elements in `array1` not present in `array2` |
| `Intersection()` | `Intersection(array1, array2)` | Array | — | Elements present in both arrays; each pair matched once |
| `Union()` | `Union(arrays*)` | Array | — | Merged unique elements from all input arrays |
| `Without()` | `Without(array, excludeValues*)` | Array | — | Array minus the explicitly listed values |

## AHK V2 CONSTRAINTS

- **1-based indexing is mandatory** — `array[1]` is the first element; `array[0]` is always wrong and always throws `IndexError` — every loop counter, range boundary, and index calculation must start from 1. Negative indices are valid and count from the end (`array[-1]` is the last element).
- **No built-in `.Sort()`** — calling `array.Sort()` throws `TypeError`; use the module's `QuickSort(array, comparator?)` with an optional `(a, b) => a - b` numeric comparator or `(a, b) => b - a` for reverse.
- **`Array(N)` ≠ pre-allocation** — `Array(10)` creates a one-element array `[10]`; pre-allocate with `arr := [], arr.Capacity := 10` to reserve backing storage without inserting elements.
- **No fat-arrow block bodies in v2.0** — `(x) => { return x * 2 }` is a v2.1 alpha construct; any multi-statement callback must be a named nested function (closure) defined inside the calling function.
- **Never name a user function `Map`** — this shadows the built-in `Map` class globally within scope, breaking all `Map()` constructor calls with `TypeError`.
- **`.Clone()` is shallow** — nested arrays or Map objects share the same reference; mutating a nested element in the clone also mutates the original — use `DeepClone()` for full independence.
- **Do not add or remove elements while iterating with `for`** — calling `.RemoveAt()` or `.Push()` inside `for index, value in array` corrupts the enumeration; use a `while`-loop with manual index management or iterate over `.Clone()`. (Modifying values at existing indices via `array[index] := x` is safe.)

Safe-access priority order for Array elements:
  1. `.Get(index, default)` — optional slot, one-line resolution, never throws `UnsetItemError`; `IndexError` still applies for out-of-range indices
  2. `.Has(index)` — when the present/absent branch logic differs meaningfully
  3. `arr.Length` guard via `IsValidIndex()` — when a bounds check before direct access is clearest
  4. `try/catch` — only when the exception itself carries diagnostic information

Pair every prohibition with its consequence and positive alternative:

- ✗ `val := array[idx]` — `UnsetItemError` if the slot was `.Delete()`d or never set
- ✓ `val := array.Get(idx, "fallback")` — safe for unset slots, never throws `UnsetItemError`, one line

- ✗ `arr.Sort()` — `TypeError`: no such method on Array
- ✓ `QuickSort(arr)` or `QuickSort(arr, (a,b) => a - b)` for numeric order

- ✗ `arr := Array(10)` — creates `[10]`, not 10 empty slots
- ✓ `arr := [], arr.Capacity := 10` — reserves 10 slots, `.Length` stays 0

- ✗ `for i, v in arr { arr.RemoveAt(i) }` — corrupts enumeration, may skip elements
- ✓ Use `while`-loop with manual index management (see `RemoveValue()` in TIER 2)

Unset variable handling: always call `.Has(index)` or `.Get(index, default)` before accessing a slot that may have been `.Delete()`d — deleted slots are within the valid index range but have no value, so bracket access throws `UnsetItemError`.

## AGENT QA CHECKLIST

- [ ] Did I use 1-based indexing for all array access and loop counters (no `arr[0]` anywhere)?
- [ ] Did I use `.Get(idx, default)` or guard with `.Has(idx)` before accessing any slot that may be unset or deleted?
- [ ] Did I avoid fat-arrow block bodies — are all multi-statement callbacks written as named nested functions?
- [ ] Did I avoid naming any helper function `Map`, `Filter`, or any other built-in class name that could be shadowed?

## RUNTIME ERROR MAPPING

| Exception Class | Trigger Condition | Detection Code | Fix |
|----------------|-------------------|----------------|-----|
| `IndexError` | Accessing `arr[0]`, `arr[arr.Length + 1]`, or any index whose absolute value exceeds `.Length` | `e.Message` contains "Index" | Switch to 1-based index; use `IsValidIndex(arr, idx)` guard before access |
| `UnsetItemError` | Accessing `arr[idx]` after `.Delete(idx)` was called, or when an unset sparse slot is read directly | `e.Message` contains "no value" | Replace with `arr.Get(idx, default)` or guard with `arr.Has(idx)` |
| `Error` | Calling `.Pop()` on an empty array (`.Length = 0`) | `e.Message` contains "empty" | Guard with `if arr.Length > 0` before `.Pop()`; or check the length and handle the empty case explicitly |

## TIER 1 — Fundamentals: Creation, Access, and Type Verification
> METHODS COVERED: `[]` · `Array()` · `.Length` · `.Capacity` · `.Get()` · `.Has()` · `.Default` · `Type()` · `IsValidIndex()` · `GetRange()`

Arrays are 1-based dynamic collections that hold variant-typed values. Use `[]` literals for known values and `Array()` for programmatic construction. Pre-allocate backing storage with `.Capacity` when the final size is known; this avoids repeated heap reallocation during sequential `.Push()` calls. Always verify type with `Type(obj) = "Array"` — never rely on duck typing or `IsObject()` alone when the code path must reject Maps and plain Objects. The `.Default` property can be set to suppress `UnsetItemError` for unset slots while still throwing `IndexError` for truly out-of-range access.

```ahk
; ✓ [] literal is the canonical v2 creation syntax for known values
numbers := [1, 2, 3, 4, 5]
strings := ["alpha", "beta", "gamma"]
mixed   := ["text", 42, true, [1, 2]]    ; mixed types are supported
empty   := []
dynamic := Array()

; ✓ Capacity pre-allocates backing storage without inserting elements
preAllocated := []
preAllocated.Capacity := 10              ; .Length stays 0 — no elements added

; ✗ Array(N) does NOT pre-allocate N slots — it inserts N as a value
; dynamic := Array(10)                  ; → creates [10], .Length = 1

; ✓ Type() is the authoritative guard — distinguishes Array from Map and Object
isArray  := Type(numbers) = "Array"
length   := numbers.Length
hasItems := numbers.Length > 0

; ✓ 1-based indexing — [1] is always the first element
first  := numbers[1]
last   := numbers[numbers.Length]
middle := numbers[numbers.Length // 2]

; ✓ Negative index counts from end — [-1] is the last element
lastViaReverse := numbers[-1]
secondToLast   := numbers[-2]

; ✓ .Get() provides safe access for unset in-range slots — never throws UnsetItemError; IndexError still applies for out-of-range indices
sparseTest := [1, , 3]                  ; index 2 is unset
safe := sparseTest.Get(2, "default")    ; returns "default" — no UnsetItemError

; ✗ Direct access on an unset in-range slot throws UnsetItemError — use .Get() or .Has() instead
; val := sparseTest[2]                  ; → UnsetItemError

; ✓ .Default suppresses UnsetItemError for unset slots — IndexError still fires for out-of-range
sparse := ["a", , "c"]                  ; index 2 is unset
sparse.Default := "N/A"
MsgBox sparse[2]                        ; "N/A" — no exception
; MsgBox sparse[99]                     ; → IndexError — Default does not suppress out-of-range

IsValidIndex(array, index) {
    return index >= 1 && index <= array.Length
}

GetRange(array, start, end) {
    result := []
    loop end - start + 1 {
        if IsValidIndex(array, start + A_Index - 1)
            result.Push(array[start + A_Index - 1])
    }
    return result
}
```

## TIER 2 — Mutation: Add, Remove, and Clear
> METHODS COVERED: `.Push()` · `.Pop()` · `.InsertAt()` · `.RemoveAt()` · `.Delete()` · `.Length` (setter) · `SafeInsert()` · `RemoveValue()`

All built-in mutation methods operate in-place and shift elements automatically. `.Push()` and `.Pop()` are O(1) amortised at the tail. `.InsertAt()` and `.RemoveAt()` are O(n) because they shift subsequent elements. Setting `.Length := 0` clears in-place (preferred over reassignment when other references to the same array must also see it emptied). `.Delete()` marks a slot as unset without shifting — use it only when sparse arrays are intentional.

```ahk
; ✓ Push appends one or multiple values in a single call
array.Push(value)
array.Push(val1, val2, val3)

; ✓ InsertAt uses 1-based position; negative index counts from end; 0 appends
array.InsertAt(1, "first")          ; prepend
array.InsertAt(3, "middle")         ; insert before index 3
array.InsertAt(-1, "beforeLast")    ; insert before last element
array.InsertAt(0, "append")         ; 0 is same as Length+1 — appends

; ✓ Pop removes and returns the last element — use for stack patterns
if array.Length > 0
    lastItem := array.Pop()

; ✗ Pop on empty array throws Error — always guard with .Length check
; lastItem := emptyArray.Pop()      ; → Error: "Array is empty"

; ✓ RemoveAt with count removes a range in one call
array.RemoveAt(1)                   ; remove first element
array.RemoveAt(3, 2)                ; remove elements at index 3 and 4
array.RemoveAt(-1)                  ; remove last element

; ✓ Delete marks slot as unset without shifting — for intentional sparse arrays
array.Delete(2)                     ; index 2 is now unset; .Length unchanged
; array[2]                          ; → UnsetItemError — use .Has(2) or .Get(2, default) first

; ✓ Length := 0 clears in-place — other references to the same array also see empty
array.Length := 0

; ✓ Reassignment creates a new array object — old reference is abandoned
array := []

SafeInsert(array, index, value) {
    if index < 1
        index := 1
    else if index > array.Length + 1
        index := array.Length + 1
    array.InsertAt(index, value)
    return array
}

RemoveValue(array, value) {
    ; ✓ while-loop with manual index handles removal during iteration correctly
    index := 1
    while index <= array.Length {
        if array[index] = value
            array.RemoveAt(index)   ; index stays the same after removal — next element shifts down
        else
            index++
    }
    return array
}

; ✗ Removing elements inside a for-in loop corrupts enumeration
; for index, value in array {
;     if value = target
;         array.RemoveAt(index)     ; → skips elements and may throw IndexError
; }
```

## TIER 3 — Search, Predicates, and Type Guards
> METHODS COVERED: `IndexOf()` · `LastIndexOf()` · `Contains()` · `FindIndex()` · `.Has()` · `IsSet()` · `Type()`

AHK v2 Array objects have no built-in search method. These module functions implement the standard search contract: return a 1-based index on success, `0` on failure — never `-1`, which is the v1/JavaScript convention. `FindIndex()` accepts a predicate callback, enabling arbitrary search criteria without writing a custom loop at the call site.

```ahk
; ✓ IndexOf returns 1-based index or 0 — never -1 (not the AHK v2 convention)
IndexOf(array, value, fromIndex := 1) {
    loop array.Length - fromIndex + 1 {
        currentIndex := fromIndex + A_Index - 1
        if array[currentIndex] = value
            return currentIndex
    }
    return 0
}

LastIndexOf(array, value) {
    loop array.Length {
        currentIndex := array.Length - A_Index + 1
        if array[currentIndex] = value
            return currentIndex
    }
    return 0
}

; ✓ Contains is a boolean wrapper — use when only presence matters, not position
Contains(array, value) {
    return IndexOf(array, value) > 0
}

; ✓ FindIndex takes a predicate callback — no need to write a custom loop per criterion
FindIndex(array, callback) {
    for index, value in array {
        if callback(value, index, array)
            return index
    }
    return 0
}

; ✓ Single-line fat-arrow callbacks are valid in v2.0
predicate        := (val, idx, arr) => val > 10
firstLargeIndex  := FindIndex(numbers, predicate)

; ✗ Fat-arrow block bodies are invalid in AHK v2.0 — syntax error at runtime
; predicate := (val, idx, arr) => {    ; → SyntaxError in v2.0
;     return val > 10
; }
```

## TIER 4 — Transformations: Clone, DeepClone, Map, Filter, Reduce
> METHODS COVERED: `.Clone()` · `DeepClone()` · `ArrayMap()` · `Filter()` · `Reduce()` · `IsObject()` · `IsSet()` · `Type()`

`.Clone()` produces a shallow copy — sufficient when nested objects are read-only. `DeepClone()` recursively copies Array, Map, and plain Object graphs; it does not handle circular references. `ArrayMap()`, `Filter()`, and `Reduce()` follow an immutable style, each returning a new array rather than mutating the input. Name the map function `ArrayMap` — never `Map` — to avoid shadowing the built-in `Map` class.

```ahk
; ✓ .Clone() is the built-in shallow copy — prefer it over manual loop copies
shallowCopy := array.Clone()

; ✓ DeepClone recurses through Array, Map, and Object — not circular-reference safe
DeepClone(obj) {
    if !IsObject(obj)
        return obj

    switch Type(obj) {
        case "Array":
            result := []
            for value in obj
                result.Push(DeepClone(value))
            return result
        case "Map":
            result := Map()
            for key, value in obj
                result[key] := DeepClone(value)
            return result
        default:
            result := {}
            for key, value in obj.OwnProps()
                result.%key% := DeepClone(value)
            return result
    }
}

; ✓ ArrayMap returns a new array — the original is never mutated
ArrayMap(array, callback) {
    result := []
    for index, value in array
        result.Push(callback(value, index, array))
    return result
}

; ✓ Filter returns elements for which the predicate returns truthy
Filter(array, callback) {
    result := []
    for index, value in array {
        if callback(value, index, array)
            result.Push(value)
    }
    return result
}

; ✓ Reduce with IsSet(initialValue) handles both seeded and unseeded forms
Reduce(array, callback, initialValue := unset) {
    startIndex := 1
    if !IsSet(initialValue) {
        if array.Length = 0
            throw Error("Reduce of empty array with no initial value")
        accumulator := array[1]
        startIndex  := 2
    } else {
        accumulator := initialValue
    }

    loop array.Length - startIndex + 1 {
        index       := startIndex + A_Index - 1
        accumulator := callback(accumulator, array[index], index, array)
    }
    return accumulator
}

numbers := [1, 2, 3, 4, 5]
doubled := ArrayMap(numbers, (x) => x * 2)
evens   := Filter(numbers, (x) => Mod(x, 2) = 0)
sum     := Reduce(numbers, (acc, x) => acc + x)

; ✗ Never name a function "Map" — it shadows the built-in Map class
; Map(array, fn) { ... }               ; → subsequent Map() constructor calls throw TypeError
```

## TIER 5 — Sorting, Deduplication, and Performance Patterns
> METHODS COVERED: `QuickSort()` · `SortBy()` · `Unique()` · `UniqueBy()` · `Map.Has()` · `Map.Delete()` · `FastContains()` · `ModifyInPlace()`

Array objects have no built-in sort. `QuickSort()` sorts in-place using the Lomuto partition scheme; the optional comparator `(a, b)` must return negative for a-before-b, zero for equal, positive for b-before-a. `Unique()` and `UniqueBy()` use a Map as a seen-set, giving O(n) deduplication rather than O(n²) nested loops.

```ahk
QuickSort(array, callback := "", left := 1, right := unset) {
    if !IsSet(right)
        right := array.Length
    if left >= right
        return array
    pivot := array[right]
    i     := left - 1
    loop right - left {
        j   := left + A_Index - 1
        cmp := callback ? callback(array[j], pivot) : (array[j] <= pivot ? -1 : 1)
        if cmp <= 0 {
            i++
            temp     := array[i]
            array[i] := array[j]
            array[j] := temp
        }
    }
    i++
    temp         := array[i]
    array[i]     := array[right]
    array[right] := temp
    QuickSort(array, callback, left, i - 1)
    QuickSort(array, callback, i + 1, right)
    return array
}

numbers := [3, 1, 4, 1, 5]
QuickSort(numbers)                              ; string comparison (default)
QuickSort(numbers, (a, b) => a - b)             ; ascending numeric
QuickSort(numbers, (a, b) => b - a)             ; descending numeric
QuickSort(numbers, (a, b) => (b - a) != 0 ? b - a : 0)

students := [
    Map("name", "Alice",   "grade", 85),
    Map("name", "Bob",     "grade", 92),
    Map("name", "Charlie", "grade", 78)
]
QuickSort(students, (a, b) => a["grade"] - b["grade"])

; ✓ SortBy uses a named nested closure — valid multi-statement callback in v2.0
SortBy(array, fields*) {
    Comparator(a, b) {
        for field in fields {
            aVal := a[field]
            bVal := b[field]
            if aVal != bVal
                return aVal < bVal ? -1 : 1
        }
        return 0
    }
    return QuickSort(array, Comparator)
}

SortBy(students, "grade", "name")

; ✓ Unique uses Map as O(n) seen-set — never use nested loops for deduplication
Unique(array) {
    result := []
    seen   := Map()
    for item in array {
        if !seen.Has(item) {
            seen[item] := true
            result.Push(item)
        }
    }
    return result
}

UniqueBy(array, keyFunc) {
    result := []
    seen   := Map()
    for item in array {
        key := keyFunc(item)
        if !seen.Has(key) {
            seen[key] := true
            result.Push(item)
        }
    }
    return result
}

people := [
    Map("id", 1, "name", "Alice"),
    Map("id", 2, "name", "Bob"),
    Map("id", 1, "name", "Alice")
]
uniquePeople := UniqueBy(people, (p) => p["id"])

; ✓ FastContains: build lookup Map once for repeated membership tests in same dataset
FastContains(array, value) {
    lookupMap := Map()
    for item in array
        lookupMap[item] := true
    return lookupMap.Has(value)
}

; ✓ ModifyInPlace avoids allocating a new array when mutation is intentional
ModifyInPlace(array, modifier) {
    for index, value in array
        array[index] := modifier(value)
    return array
}
```

## TIER 6 — Set Operations: Difference, Intersection, Union, Without
> METHODS COVERED: `Difference()` · `Intersection()` · `Union()` · `Without()` · `Map.Has()` · `Map.Delete()`

Set operations are implemented with Map-backed seen-sets rather than nested loops, keeping time complexity O(n + m). `Intersection()` calls `.Delete()` on the seen-set after each match to prevent the same element from being counted twice when duplicates appear in `array2`. `Union()` accepts a variadic argument list so any number of arrays can be merged in one call.

```ahk
; ✓ Difference: elements in array1 that do not appear in array2 — O(n + m)
Difference(array1, array2) {
    result := []
    set2   := Map()
    for item in array2
        set2[item] := true

    for item in array1 {
        if !set2.Has(item)
            result.Push(item)
    }
    return result
}

; ✓ Intersection: .Delete() after match prevents double-counting array2 duplicates
Intersection(array1, array2) {
    result := []
    set2   := Map()
    for item in array2
        set2[item] := true

    for item in array1 {
        if set2.Has(item) {
            result.Push(item)
            set2.Delete(item)       ; consume the match — each pair matched once
        }
    }
    return result
}

; ✓ Union accepts variadic arrays — all inputs merged into one deduplicated result
Union(arrays*) {
    result := []
    seen   := Map()
    for array in arrays {
        for item in array {
            if !seen.Has(item) {
                seen[item] := true
                result.Push(item)
            }
        }
    }
    return result
}

; ✓ Without excludes a variadic list of values in a single pass
Without(array, excludeValues*) {
    excludeSet := Map()
    for value in excludeValues
        excludeSet[value] := true

    result := []
    for item in array {
        if !excludeSet.Has(item)
            result.Push(item)
    }
    return result
}

arr1     := [1, 2, 3, 4]
arr2     := [3, 4, 5, 6]
diff     := Difference(arr1, arr2)       ; [1, 2]
inter    := Intersection(arr1, arr2)     ; [3, 4]
union    := Union(arr1, arr2)            ; [1, 2, 3, 4, 5, 6]
filtered := Without(arr1, 2, 4)         ; [1, 3]

; ✗ Nested-loop membership test is O(n × m) — use Map seen-set for large arrays
; Difference_Slow(a1, a2) {
;     result := []
;     for item in a1 {
;         found := false
;         for x in a2 {
;             if x = item {
;                 found := true
;                 break
;             }
;         }
;         if !found
;             result.Push(item)   ; → O(n × m) — degrades severely with large inputs
;     }
;     return result
; }
```

### Performance Notes

**O(1) vs O(n) access:** `.Push()` and `.Pop()` at the tail are O(1) amortised. `.InsertAt(1, …)` and `.RemoveAt(1)` at the head are O(n) — avoid them in tight loops on large arrays. Membership testing with a bare loop (`Contains()`) is O(n) per call; if the same array is queried repeatedly, build a `Map` lookup once (`FastContains()`) for O(1) per subsequent test.

**In-place vs copy:** `ArrayMap()`, `Filter()`, and `Reduce()` always allocate a new array — appropriate for pipelines where the original must be preserved. When the original is no longer needed, `ModifyInPlace()` eliminates the allocation. For sorting, `QuickSort()` always sorts in-place; call `array.Clone()` first if the original order must be preserved.

**Pre-allocation:** Set `arr.Capacity := expectedSize` before a loop that calls `.Push()` repeatedly. This avoids the exponential reallocation series that occurs when `.Capacity` is allowed to grow automatically. The `.Length` remains 0 until elements are actually pushed.

**DeepClone cost:** `DeepClone()` is O(n) in total graph nodes and allocates one new container per node. Avoid calling it in hot loops or on large nested structures — share read-only sub-arrays as references using `.Clone()` when the nested data will not be mutated.

**Map-backed deduplication:** `Unique()` and set operations (`Difference`, `Intersection`, `Union`) all build a Map as their seen-set, giving O(n + m) total complexity rather than O(n × m) for naive nested-loop implementations. This is the preferred pattern for any uniqueness or membership operation on arrays larger than a handful of elements.

## DROP-IN RECIPES

```ahk
; SafePop — guarded pop that returns a default instead of throwing on empty array
; ✓ Handles the empty-array case in one call; no caller-side length check required
SafePop(array, default := unset) {
    if !(array is Array)
        throw TypeError("SafePop: expected Array, got " Type(array), -1)
    if array.Length > 0
        return array.Pop()
    if IsSet(default)
        return default
    throw Error("SafePop: array is empty and no default was provided", -1)
}
; Call site: last := SafePop(queue, "")

; ChunkArray — splits an array into sub-arrays of at most `size` elements
; ✓ Returns an array of arrays; last chunk may be smaller than `size`
ChunkArray(array, size) {
    if !(array is Array)
        throw TypeError("ChunkArray: expected Array, got " Type(array), -1)
    if !(size is Integer) || size < 1
        throw ValueError("ChunkArray: size must be a positive integer", -1)
    chunks := []
    chunk  := []
    chunks.Capacity := Ceil(array.Length / size)
    for index, value in array {
        chunk.Push(value)
        if Mod(index, size) = 0 {
            chunks.Push(chunk)
            chunk := []
        }
    }
    if chunk.Length > 0
        chunks.Push(chunk)          ; push the final partial chunk
    return chunks
}
; Call site: pages := ChunkArray(allRows, 50)

; Flatten — recursively flattens nested arrays up to `depth` levels
; ✓ depth := 1 flattens one level; depth := -1 flattens all levels
Flatten(array, depth := 1) {
    if !(array is Array)
        throw TypeError("Flatten: expected Array, got " Type(array), -1)
    result := []
    FlattenInto(array, result, depth)
    return result

    FlattenInto(src, dest, remaining) {
        for item in src {
            if (item is Array) && remaining != 0
                FlattenInto(item, dest, remaining - 1)
            else
                dest.Push(item)
        }
    }
}
; Call site: flat := Flatten([[1, 2], [3, [4, 5]]], 1)  ; → [1, 2, 3, [4, 5]]
```

## ANTI-PATTERNS

| Pattern | Wrong | Correct | LLM Common Cause |
|---------|-------|---------|------------------|
| Zero-based indexing | `array[0]` | `array[1]` / `array[-1]` for last | Dominant habit from JavaScript, Python, C training data |
| Built-in sort assumed | `arr.Sort()` | `QuickSort(arr, (a,b) => a - b)` | Missing v2 API knowledge — LLM infers `.Sort()` by analogy with String |
| Array(N) pre-allocation | `Array(10)` | `arr := [], arr.Capacity := 10` | Missing v2 constructor semantics — `Array(N)` parallels `Array(val)` not `new Array(n)` |
| Fat-arrow block body | `(x) => { return x * 2 }` | Named nested function inside caller | AHK v1 / other-language habit; block bodies are v2.1 alpha only |
| Shadowing Map class | `Map(arr, fn) { ... }` | `ArrayMap(arr, fn) { ... }` | JavaScript `Array.prototype.map` naming convention transferred to AHK |
| Mutating during for-in | `for i, v in arr { arr.RemoveAt(i) }` | `while`-loop with manual index | Cross-language habit; Python/JS raise RuntimeError — AHK silently corrupts |
| Nested-loop membership | `for x in a1 { for y in a2 { if x=y ... } }` | `Map`-backed seen-set (`Unique`, `Difference`) | O(n²) pattern learned from algorithm examples without performance annotations |

## SEE ALSO

> This module does NOT cover: key-value storage, object property bags, and Map() API — see Module_Objects.md
> This module does NOT cover: try/catch patterns for out-of-bounds access and type errors — see Module_Errors.md
> This module does NOT cover: GUI control binding (ListView, ComboBox population from arrays) — see Module_GUI.md
> This module does NOT cover: file-path batch operations and directory listing into arrays — see Module_FileSystem.md

- `Module_Objects.md` — Map() as an O(1) key-value store; plain Object property bags; when to choose Map over Array.
- `Module_Errors.md` — try/catch patterns for `IndexError`, `UnsetItemError`, and `TypeError` thrown by Array methods.
- `Module_Functions.md` — closures, variadic functions, and `.Bind()` patterns used when passing callbacks to ArrayMap/Filter/Reduce.
- `Module_GUI.md` — populating ListViews, ComboBoxes, and DropDownLists from Array data sources.
