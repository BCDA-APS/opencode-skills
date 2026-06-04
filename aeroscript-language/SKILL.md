---
name: aeroscript-language
description: Write AeroScript programs for Aerotech Automation1 controllers -- syntax, data types, variables and scoping, control flow, functions, libraries, preprocessor, G-code embedding, and built-in math/string/datetime/file functions
---

# AeroScript Language Skill

You are an expert at writing AeroScript programs for Aerotech Automation1 motion controllers. AeroScript is a high-level, real-time scripting language that runs on the controller's deterministic execution engine. It combines C-like control flow with embedded G-code support and a rich standard library for motion, I/O, file access, and communication.

AeroScript is NOT an EPICS technology. It is Aerotech's proprietary controller programming language. Programs are written as `.ascript` files and run on controller tasks.

---

## 1. Program Structure

```aeroscript
program
    // Top-level declarations (program-global scope)
    var $myVar as real = 0.0

    // Code executed when the program runs
    Enable(X)
    Home(X)
    MoveLinear(X, 10.0, 5.0)
    Disable(X)
end
```

Every AeroScript program must be wrapped in a `program ... end` block. Functions are defined outside the `program` block (before or after).

```aeroscript
function MyHelper($axis as axis, $distance as real)
    MoveLinear($axis, $distance, 10.0)
    WaitForInPosition($axis)
end

program
    Enable(X)
    Home(X)
    MyHelper(X, 25.0)
    Disable(X)
end
```

---

## 2. Comments

```aeroscript
// Single-line comment (C++ style)
;  Single-line comment (G-code style)
'  Single-line comment (BASIC style)

/* Multi-line
   comment */
```

Metadata comments for version tracking:
```aeroscript
//! @unique-id "my-program-v1"
//! @version 2
```

---

## 3. Data Types

### 3.1 Fundamental Types

| Type | Description | Size |
|------|-------------|------|
| `real` | 64-bit IEEE 754 floating-point | 8 bytes |
| `integer` | 64-bit signed integer | 8 bytes |
| `string` | UTF-8 string (default capacity 256 chars, max 2500) | Variable |
| `axis` | Represents a physical or virtual axis | -- |
| `handle` | Opaque reference (file handles, sockets, matrices) | -- |

### 3.2 Variable Declaration

All variables must be prefixed with `$`:

```aeroscript
var $count as integer = 0
var $voltage as real = 3.14
var $name as string = "Hello"
var $myAxis as axis = X
var $file as handle

// Multiple declarations
var $x as real, $y as real, $z as real

// String with explicit capacity
var $longString as string(1000) = ""

// Constants
var $PI as real = 3.14159265358979
```

### 3.3 Arrays

```aeroscript
var $data[100] as real                    // 1D array of 100 reals
var $matrix[4][4] as real                 // 2D array
var $cube[3][3][3] as integer             // 3D array
var $axes[] as axis = [X, Y, Z]           // Initialized from literal

// Array initialization
var $values[] as real = [1.0, 2.0, 3.0]
var $zeros[10] as integer = [0]           // First element 0, rest default
```

Array sizes must be integer literals (not variables or expressions). Use `length($array)` to get the element count.

### 3.4 Structs

```aeroscript
struct Point
    var $x as real
    var $y as real
    var $z as real
end

var $origin as Point
$origin.x = 0.0
$origin.y = 0.0
$origin.z = 0.0
```

### 3.5 Enums

```aeroscript
enum Color
    Red = 0
    Green = 1
    Blue = 2
end

var $c as Color = Color.Red
```

### 3.6 The axis Type and @ Operator

```aeroscript
var $myAxis as axis = X              // Direct assignment
var $axisNum as integer = 0
var $dynAxis as axis = @$axisNum     // Integer -> axis via @
var $axisName as string = "X"
var $strAxis as axis = @$axisName    // String -> axis via @
```

---

## 4. Variable Scoping

### 4.1 Controller Global Variables

Persist across programs and tasks. Shared by all programs on all tasks:

```aeroscript
$rglobal[0] = 3.14       // Real globals (index 0-511)
$iglobal[0] = 42          // Integer globals (index 0-511)
$sglobal[0] = "hello"     // String globals (index 0-63)
```

### 4.2 Task Variables

Shared by all programs on the same task:

```aeroscript
$rtask[0] = 1.0           // Real task variables (index 0-63)
$itask[0] = 100            // Integer task variables (index 0-63)
$stask[0] = "task data"    // String task variables (index 0-15)
```

### 4.3 Program Global Variables

Declared at the top level of the `program` block. Persist for the lifetime of the program. Initialized once.

### 4.4 Function Local Variables

Declared inside a function body. Created on function entry, destroyed on exit.

### 4.5 Block Local Variables

Declared inside `if`, `while`, `for`, etc. Created and destroyed each time the block executes.

---

## 5. Expressions and Operators

### 5.1 Literals

```aeroscript
42                  // Integer decimal
0xFF                // Integer hexadecimal
3.14                // Real
1.5e-3              // Real (scientific notation)
"Hello\nWorld"      // String with escape
true                // Boolean (integer 1)
false               // Boolean (integer 0)
```

### 5.2 String Escape Sequences

| Escape | Character |
|--------|-----------|
| `\\` | Backslash |
| `\"` | Double quote |
| `\n` | Newline |
| `\r` | Carriage return |
| `\t` | Tab |
| `\0` | Null |
| `\xHH` | Hex byte |
| `\uHHHH` | Unicode (16-bit) |
| `\UHHHHHHHH` | Unicode (32-bit) |

### 5.3 Operators (by precedence, highest first)

| Operator | Description |
|----------|-------------|
| `()` | Grouping / function call |
| `[]` | Array index |
| `.` | Struct member access |
| `@` | Integer/string to axis conversion |
| `++`, `--` | Increment, decrement (postfix) |
| `!`, `~`, `-` (unary) | Logical NOT, bitwise NOT, negation |
| `**` | Exponentiation |
| `*`, `/`, `%` | Multiply, divide, modulo |
| `+`, `-` | Add, subtract |
| `+` (string) | String concatenation |
| `<<`, `>>` | Bitwise shift |
| `<`, `<=`, `>`, `>=` | Comparison |
| `==`, `!=` | Equality |
| `&` | Bitwise AND |
| `^` | Bitwise XOR |
| `\|` | Bitwise OR |
| `&&` | Logical AND (short-circuit) |
| `\|\|` | Logical OR (short-circuit) |
| `=`, `+=`, `-=`, `*=`, `/=`, `%=` | Assignment |

### 5.4 Intrinsic Functions

```aeroscript
length($array)           // Number of elements in an array
argument()               // Number of arguments in current function call (for overloads)
defined(MACRO_NAME)      // True if preprocessor macro is defined
```

---

## 6. Statements

### 6.1 Conditional

```aeroscript
if $x > 10
    // ...
elseif $x > 5
    // ...
else
    // ...
end

switch $mode
    case 0
        // ...
    case 1
        // ...
    default
        // ...
end
```

### 6.2 Loops

```aeroscript
while $running
    // ...
end

for $i = 0 to 9
    // 0, 1, 2, ..., 9 (inclusive)
end

for $i = 0 to 100 step 10
    // 0, 10, 20, ..., 100
end

foreach var $ax in $axes
    Enable($ax)
end

repeat 5
    // Execute body 5 times
end
```

### 6.3 Flow Control

```aeroscript
break              // Exit innermost loop
continue           // Skip to next iteration
return             // Return from function (no value)
return $value      // Return value from function
goto MyLabel       // Jump to label
MyLabel:           // Label definition
```

### 6.4 Wait

```aeroscript
wait($condition)                    // Block until condition is true
wait($condition, $timeoutMs)        // Block with timeout in milliseconds
```

Returns `true` if condition was met, `false` if timeout expired.

### 6.5 Error Handling

```aeroscript
onerror(MyErrorHandler())

function MyErrorHandler()
    AcknowledgeAll()
    Dwell(0.1)
    ProgramRestart()
end
```

`onerror()` registers a function to call when a fault or error occurs on the current task. Only one handler active at a time. Call `onerror()` with no arguments to remove the handler.

### 6.6 Critical Sections

```aeroscript
CriticalSectionStart()
// Code here executes without yielding to the task scheduler
// (except for multi-ms functions like I/O, file, sockets)
CriticalSectionEnd()

CriticalSectionStart(5000)  // With timeout in microseconds
```

---

## 7. Functions

```aeroscript
function Add($a as real, $b as real) as real
    return $a + $b
end

function Swap(ref $a as real, ref $b as real)
    var $temp as real = $a
    $a = $b
    $b = $temp
end
```

- Functions must be defined outside the `program` block.
- Arguments are pass-by-value by default. Use `ref` for pass-by-reference (output parameters).
- Return types can only be scalar (`real`, `integer`, `string`, `axis`, `handle`). Cannot return arrays or structs.
- Overloading by argument count and type is supported.

### 7.1 Synchronized Function Calls

```aeroscript
// Execute function synchronized with motion (in the servo loop context)
sync(MyCallback(), blocking)
sync(MyCallback(), nonblocking)
sync(MyCallback(), blocking, 2)    // Execute on task 2
```

---

## 8. Libraries

### 8.1 Creating a Library

```aeroscript
// mylib.ascriptlib
library

function LibraryFunc($x as real) as real
    return $x * 2.0
end

struct LibraryStruct
    var $value as real
end

enum LibraryEnum
    OptionA = 0
    OptionB = 1
end
```

Only items inside a `library` block are exported. Compile to `.a1lib` for distribution.

### 8.2 Importing a Library

```aeroscript
import "mylib.a1lib" as static    // Linked at compile time
import "mylib.a1lib" as dynamic   // Loaded at runtime
```

Static: library code is embedded in the compiled program.
Dynamic: library is loaded from the controller file system at runtime (allows independent updates).

---

## 9. Preprocessor

```aeroscript
#define MAX_AXES 8
#define PI 3.14159265358979

// Parameterized macro
#define SQUARE($x) (($x) * ($x))

// Multi-line macro
#define SETUP_AXIS($ax) \
    Enable($ax) \
    Home($ax)

#include "myHelpers.ascript"

#ifdef DEBUG
    AppMessageDisplay("Debug mode")
#endif

#if defined(USE_FEATURE_A) && !defined(LEGACY)
    // ...
#elif defined(USE_FEATURE_B)
    // ...
#else
    // ...
#endif

#undef MAX_AXES
```

**Note:** `.st` files are preprocessed by the C preprocessor. `.stt` files skip preprocessing.

---

## 10. G-Code Embedding

AeroScript supports RS-274 G-code commands directly:

```aeroscript
program
    Enable([X, Y])
    Home([X, Y])

    G90                         // Absolute positioning
    G1 X10 Y20 F100             // Linear move at F=100
    G2 X20 Y10 R5               // Clockwise arc, radius 5
    G3 X10 Y20 I-5 J0           // CCW arc, center offset
    G4 P0.5                     // Dwell 0.5 seconds
    G108                        // Velocity blending on
    G1 X30 Y30 F200
    G1 X40 Y10
    G109                        // Velocity blending off
    G0 X0 Y0                   // Rapid home

    Disable([X, Y])
end
```

### Key G-Codes

| Code | Function |
|------|----------|
| `G0` | Rapid point-to-point move |
| `G1` | Coordinated linear move |
| `G2` | Clockwise circular arc |
| `G3` | Counter-clockwise circular arc |
| `G4 Pn` | Dwell for n seconds |
| `G70`/`G71` | Primary/secondary distance units |
| `G75`/`G76` | Time in seconds/minutes |
| `G90`/`G91` | Absolute/incremental positioning |
| `G92` | Position offset |
| `G93` | Inverse time feedrate mode |
| `G94` | Units/time feedrate mode (default) |
| `G108`/`G109` | Velocity blending on/off |
| `G359`/`G360`/`G361` | Wait mode: auto/motion-done/in-position |
| `F` | Set coordinated feedrate |
| `E` | Set dependent axis feedrate |
| `M0` / `M2` | Program stop / program end |
| `N` | Line label (for goto) |

### Custom G-Codes

```aeroscript
gcode G300($param1 as real, $param2 as real)
    MoveLinear([X, Y], [$param1, $param2], 10.0)
    WaitForInPosition([X, Y])
end
```

---

## 11. Built-in Math Functions

| Function | Description |
|----------|-------------|
| `Abs($n as real) as real` | Absolute value |
| `Ceil($n as real) as real` | Ceiling |
| `Floor($n as real) as real` | Floor |
| `Round($n as real) as real` | Round to nearest integer |
| `Trunc($n as real) as real` | Truncate toward zero |
| `Frac($n as real) as real` | Fractional part |
| `Sqrt($n as real) as real` | Square root |
| `Exp($n as real) as real` | e^n |
| `Log($n as real) as real` | Natural log (ln) |
| `Log10($n as real) as real` | Log base 10 |
| `Log2($n as real) as real` | Log base 2 |
| `Sin($n as real) as real` | Sine (degrees) |
| `Cos($n as real) as real` | Cosine (degrees) |
| `Tan($n as real) as real` | Tangent (degrees) |
| `Asin($n as real) as real` | Arcsine (returns degrees) |
| `Acos($n as real) as real` | Arccosine (returns degrees) |
| `Atan($n as real) as real` | Arctangent (returns degrees) |
| `Atan2($y, $x as real) as real` | Two-argument arctangent (degrees) |

**Note:** Trigonometric functions use **degrees**, not radians.

---

## 12. Built-in String Functions

| Function | Description |
|----------|-------------|
| `StringLength($s) as integer` | Number of characters |
| `StringCapacity($s) as integer` | Maximum capacity |
| `StringInsert($s, $index, $toInsert) as string` | Insert at position |
| `StringReplace($s, $old, $new) as string` | Replace all occurrences |
| `StringSubstring($s, $start, $length) as string` | Extract substring |
| `StringSplit($s, $delimiter, ref $parts[]) as integer` | Split into array, returns count |
| `StringFindSubstringIndex($s, $sub) as integer` | Find first occurrence (-1 if not found) |
| `StringCharacterAt($s, $index) as string` | Character at position |
| `StringEquals($a, $b) as integer` | Case-sensitive comparison |
| `StringToLowerCase($s) as string` | Convert to lowercase |
| `StringToUpperCase($s) as string` | Convert to uppercase |
| `StringTrim($s) as string` | Remove leading/trailing whitespace |
| `IntegerToString($n) as string` | Integer to string |
| `IntegerToString($n, $base as NumberBase) as string` | With base (Decimal, Hexadecimal) |
| `RealToString($n) as string` | Real to string |
| `RealToString($n, $precision) as string` | With precision |
| `StringToInteger($s) as integer` | Parse integer |
| `StringToReal($s) as real` | Parse real |
| `AxisToString($ax) as string` | Axis name |

---

## 13. Built-in DateTime Functions

| Function | Description |
|----------|-------------|
| `DateTimeGet() as string` | Current date/time as string (default format) |
| `DateTimeGet($format as DateTimeStringFormat) as string` | With format (Iso8601Local, Iso8601Utc, etc.) |
| `DateTimeExtractYear() as integer` | Current year |
| `DateTimeExtractMonth() as integer` | Current month (1-12) |
| `DateTimeExtractDay() as integer` | Current day (1-31) |
| `DateTimeExtractHour() as integer` | Current hour (0-23) |
| `DateTimeExtractMinute() as integer` | Current minute (0-59) |
| `DateTimeExtractSecond() as integer` | Current second (0-59) |

**Note:** DateTime functions are non-real-time callbacks. They pause the AeroScript program briefly.

---

## 14. Built-in File and Directory Functions

### File Operations

```aeroscript
var $fh as handle = FileOpenText("data.txt", FileMode.Overwrite)
FileTextWriteString($fh, "line 1\n")
FileClose($fh)

$fh = FileOpenText("data.txt", FileMode.Read)
var $line as string = FileTextReadLine($fh)
FileClose($fh)
```

| Function | Description |
|----------|-------------|
| `FileOpenText($path, $mode as FileMode) as handle` | Open text file |
| `FileOpenBinary($path, $mode as FileMode) as handle` | Open binary file |
| `FileClose($fh)` | Close file |
| `FileTextReadLine($fh) as string` | Read one line |
| `FileTextReadString($fh, $numChars) as string` | Read N characters |
| `FileTextWriteString($fh, $text)` | Write string |
| `FileSize($fh) as integer` | File size in bytes |
| `FileIsEndOfFile($fh) as integer` | EOF check |
| `FileGetByteOffset($fh) as integer` | Current position |
| `FileSetByteOffset($fh, $offset)` | Seek to position |

Binary read/write: `FileBinaryReadFloat64`, `FileBinaryWriteFloat64`, `FileBinaryReadInt32`, etc. (all scalar and array variants).

### INI File Operations

```aeroscript
FileIniWriteValue("config.ini", "Section", "Key", "Value")
var $val as string = FileIniReadValue("config.ini", "Section", "Key")
```

### Directory Operations

| Function | Description |
|----------|-------------|
| `DirectoryFileExists($path) as integer` | Check if file exists |
| `DirectoryFileDelete($path)` | Delete file |
| `DirectoryFileCopy($src, $dst)` | Copy file |
| `DirectoryFileMove($src, $dst)` | Move/rename file |
| `DirectoryFileCount($dir) as integer` | Count files in directory |
| `DirectoryFileGetFileName($dir, $index) as string` | Get filename by index |

`FileMode` enum: `Overwrite` (0), `Read` (1), `Append` (2).

**Note:** File and directory functions are non-real-time callbacks. They pause the AeroScript program briefly.

---

## 15. Error Handling

```aeroscript
onerror(HandleError())

function HandleError()
    var $msg as string = TaskGetErrorMessage(TaskGetIndex())
    AppMessageDisplay("Error: " + $msg)
    AcknowledgeAll()
    Dwell(0.1)
    ProgramRestart()    // Or ProgramExit() to stop
end
```

| Function | Description |
|----------|-------------|
| `AcknowledgeAll()` | Clear all axis faults and task errors |
| `FaultAcknowledge($axis)` | Clear faults on one axis |
| `FaultThrow($axis, $fault as AxisFault)` | Manually trigger a fault |
| `TaskGetErrorMessage($taskIndex) as string` | Get error message text |
| `TaskGetWarningMessage($taskIndex) as string` | Get warning text |
| `TaskSetError($taskIndex, $msg)` | Set custom error |
| `TaskSetWarning($taskIndex, $msg)` | Set custom warning |
| `TaskClearError($taskIndex)` | Clear task error |

---

## 16. Key Rules and Pitfalls

1. **All variables must start with `$`**. `myVar` is invalid; use `$myVar`.

2. **Array sizes must be integer literals**, not variables or expressions. `var $arr[10] as real` is valid; `var $arr[$n] as real` is not.

3. **Trigonometric functions use degrees**, not radians. `Sin(90)` returns 1.0.

4. **`string` has a fixed capacity** (default 256 characters). If you need longer strings, declare with explicit capacity: `var $s as string(2500)`.

5. **Functions cannot return arrays or structs.** Use `ref` parameters for output data.

6. **`real` is 64-bit float, `integer` is 64-bit signed.** There is no 32-bit type in AeroScript (though binary file I/O supports 32-bit reads/writes).

7. **File, socket, and DateTime functions are non-real-time.** They pause the AeroScript program and execute as callbacks in the communication service. Do not use them in tight real-time loops.

8. **`onerror()` replaces any previous handler.** Only one error handler is active per task. Call `onerror()` with no arguments to remove it.

9. **`for` loops include the upper bound.** `for $i = 0 to 9` iterates 10 times (0 through 9 inclusive), unlike C's `for (i=0; i<10; i++)`.

10. **There is no `switch` fall-through.** Each `case` implicitly breaks. No `break` statement is needed (or available) in `switch`.
