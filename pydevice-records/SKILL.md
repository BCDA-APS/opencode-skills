---
name: pydevice-records
description: Write EPICS database records and Python modules for PyDevice -- DTYP pydev link syntax, field macros, all 13 supported record types, waveform and array handling, pydev.iointr with I/O Intr scanning, and the pycalc record
---

# PyDevice Records Skill

You are an expert at writing EPICS database records that execute Python through PyDevice, and at structuring the Python modules they call. Your task covers link syntax, field macros, type conversion, interrupt scanning, and the `pycalc` record.

For building PyDevice, linking it into an IOC, `st.cmd` setup, the worker thread pool, and `TPRO` debugging, use the **`pydevice-ioc`** skill instead.

**Reference version:** This skill documents PyDevice at commit `7fa3fea` (`R1.3.3-2-g7fa3fea`), two commits past the `R1.3.3` tag. Source: <https://github.com/klemenv/PyDevice>.

---

## 1. Hello World

```
record(longout, "PyDev:Test:HelloWorld") {
    field(DTYP, "pydev")
    field(OUT,  "@print('Hello world!')")
}
```

Every time this record processes, `Hello world!` appears on the IOC console. Three required pieces: `DTYP` is `pydev`, the code lives in `INP` or `OUT`, and it is prefixed with `@`.

---

## 2. Link Syntax

Links are `INST_IO`. The `@` is consumed by the link parser; everything after it is Python source.

```
field(INP, "@<single line of Python>")
field(OUT, "@<single line of Python>")
```

### 2.1 Expressions and statements both work

`PyWrapper::compile()` tries expression mode first, then falls back to statement mode:

```cpp
PyObject* bytecode = Py_CompileString(code.c_str(), "", Py_eval_input);
if (bytecode == NULL) {
    // Ignore error, try with Py_file_input which works for 'import xxx' etc.
    PyErr_Clear();
    bytecode = Py_CompileString((code+"\n").c_str(), "", Py_file_input);
    ...
}
```

So all of these are valid:

```
field(OUT, "@device1.connect(timeout=2.0)")        # call
field(INP, "@device1.sent")                        # attribute read
field(INP, "@VAL**2")                              # expression
field(OUT, "@b=VAL")                               # assignment statement
field(OUT, "@import mymodule")                     # import statement
```

Only an expression produces a value. Statements return `None`, which matters for input records (§3).

### 2.2 Single line only

The code is one string compiled in one shot. There is no multi-line form, no semicolon-free statement sequencing beyond what Python allows on one line, and no indentation-based blocks.

For anything non-trivial, put the logic in a Python module and call it:

```
field(OUT, "@device1.run_calibration_sequence(VAL)")
```

Semicolons do work for simple sequences: `@import time; t0 = time.time()`.

### 2.3 Newlines inside string literals

An escaped newline in a Python string literal is fine:

```
record(bo, "PyDev:NewLine") {
  field(DTYP, "pydev")
  field(OUT,  "@print('Hello\nworld!')")
}
```

---

## 3. Supported Record Types

Thirteen record types, all with `DTYP "pydev"` and `INST_IO`.

From `src/pydev.dbd`:

```
device(ao,INST_IO,devPyDevAo,"pydev")
device(longout,INST_IO,devPyDevLongout,"pydev")
device(bo,INST_IO,devPyDevBo,"pydev")
device(mbbo,INST_IO,devPyDevMbbo,"pydev")
device(stringout,INST_IO,devPyDevStringout,"pydev")

device(ai,INST_IO,devPyDevAi,"pydev")
device(longin,INST_IO,devPyDevLongin,"pydev")
device(bi,INST_IO,devPyDevBi,"pydev")
device(mbbi,INST_IO,devPyDevMbbi,"pydev")
device(stringin,INST_IO,devPyDevStringin,"pydev")
device(waveform,INST_IO,devPyDevWaveform,"pydev")

registrar(pydevRegister)
```

Plus `src/pydev315.dbd`, built only on base 3.15+:

```
device(lsi,INST_IO,devPyDevLsi,"pydev")
device(lso,INST_IO,devPyDevLso,"pydev")
```

### 3.1 Return value requirements and conversion

| Record | Link | Return required | Assigned to | Conversion |
|---|---|---|---|---|
| `ai` | `INP` | **yes** | `VAL` | `get_double()`, then `ASLO`/`AOFF`/`SMOO` applied |
| `longin` | `INP` | **yes** | `VAL` | `get_long()` |
| `bi` | `INP` | **yes** | `RVAL` | `get_bool()` |
| `mbbi` | `INP` | **yes** | `RVAL` | `get_long()` |
| `stringin` | `INP` | **yes** | `VAL` | `get_string()`, truncated to 39 chars |
| `lsi` | `INP` | **yes** | `VAL` | `get_string()`, truncated to `SIZV-1`, sets `LEN` |
| `waveform` | `INP` | **yes** | `BPTR` | array per `FTVL`, sets `NORD` |
| `ao` | `OUT` | no | `VAL` | `get_double()` |
| `longout` | `OUT` | no | `VAL` | `get_long()` |
| `bo` | `OUT` | no | `RVAL` | `get_bool()` |
| `mbbo` | `OUT` | no | `VAL` | `get_long()` |
| `stringout` | `OUT` | no | `VAL` | `get_string()` |
| `lso` | `OUT` | no | `VAL` | `get_string()`, truncated to `SIZV-1` |

Output records wrap the assignment in a try/catch that swallows conversion failures:

```cpp
auto ret = PyWrapper::eval(ctx->bytecode, args, (rec->tpro == 1));
try {
    rec->val = ret.get_long();
} catch (std::exception& e) {
    // pass
}
rec->udf = 0;
```

This is what allows `@print(...)` on a `longout` -- `print()` returns `None`, the conversion throws, and the record still completes normally.

> **Version note.** The "output records do not need a return value" behavior comes from commit `7fa3fea`, which is *after* the `R1.3.3` release tag. On released `R1.3.3` and earlier, output records **did** require a convertible return value, so `@print(...)` on a `longout` left the record in `INVALID`/`CALC`. If you target a tagged release, give output records something convertible or expect the alarm.

Input records have no such guard -- a failed conversion propagates and alarms the record.

### 3.2 Initialization return codes

* `ao`, `bo`, `mbbo` -- `init_record` returns `2`, meaning "do not set `VAL` from `RVAL`"
* `ai` -- `process` returns `2`, meaning "conversion already done" (PyDevice applies `ASLO`/`AOFF` itself)
* `longout`, `stringout`, `lso` -- return `0`

### 3.3 Choosing a record type

| Need | Use |
|---|---|
| Fire an action, ignore the result | `longout` or `bo` with `@obj.method()` |
| Read a number | `longin` (integer) or `ai` (float) |
| Read a string up to 39 chars | `stringin` |
| Read a longer string | `lsi` with `SIZV` (base 3.15+) |
| Read or write an array | `waveform` |
| Enumerated state | `mbbi`/`mbbo` |
| Pass more than one parameter | `pycalc` (§11) |

---

## 4. Field Macros

Uppercase field names in the link are replaced with the live field value, bound as a native Python object. Think of them as built-in variables available to your expression.

### 4.1 Complete tables

Extracted from the device support sources, so these include macros the README omits.

| Record | Macros |
|---|---|
| `longin`, `longout` | `VAL` `NAME` `EGU` `HOPR` `LOPR` `HIGH` `HIHI` `LOW` `LOLO` `TPRO` |
| `ai`, `ao` | `VAL` `RVAL` `ORAW` `NAME` `EGU` `HOPR` `LOPR` `PREC` `TPRO` |
| `bi`, `bo` | `VAL` `RVAL` `NAME` `ZNAM`* `ONAM`* `TPRO` |
| `stringin`, `stringout` | `VAL` `NAME` `TPRO` |
| `lsi`, `lso` | `VAL` `NAME` `SIZV` `LEN` `TPRO` |
| `waveform` | `VAL` `TPRO` |
| `mbbi`, `mbbo` | `VAL` `RVAL` `NAME` `TPRO` plus all 16 value fields `ZRVL` `ONVL` `TWVL` `THVL` `FRVL` `FVVL` `SXVL` `SVVL` `EIVL` `NIVL` `TEVL` `ELVL` `TVVL` `TTVL` `FTVL` `FFVL` and all 16 string fields `ZRST` `ONST` `TWST` `THST` `FRST` `FVST` `SXST` `SVST` `EIST` `NIST` `TEST` `ELST` `TVST` `TTST` `FTST` `FFST` |
| `pycalc` | `NAME` `TPRO` plus `A` through `J` |

Two additions relative to the upstream README:

* **`TPRO` is supported by every record type** but is undocumented. Writing `TPRO` in an expression injects the trace-processing flag as an integer.
* **`bi`/`bo` also accept `RVAL`**, which the README's field table omits.

\* `ZNAM` and `ONAM` are **broken** -- see §12.1.

### 4.2 Substitution rules

A token is recognized as a macro only when it is all uppercase, at most 4 characters, and bounded by non-alphanumeric characters:

```cpp
const long MAX_FIELD_LEN = 4;
...
if (isupper(text.at(i))) {
    if (!isalpha(prevChar) || !token.empty()) {
        token += text.at(i);
    }
} else if (islower(text.at(i))) {
    token.clear();
}
```

Verified by the unit tests in `src/unittest/test_util.cpp`:

```cpp
testOk1(Util::getMacros("NAMEA").size() == 0);          // 5 chars, too long
testOk1(Util::getMacros("aVAL").size() == 0);           // preceded by lowercase
testOk1(Util::getMacros("VALa").size() == 0);           // followed by lowercase
testOk1(Util::getMacros("function(NAME,A)").size() == 2);
testOk1(Util::replaceMacro("ThisIsVALtest", "VAL", "Hello") == "ThisIsVALtest");
testOk1(Util::replaceMacro("ThisIs%VAL%test", "VAL", "Hello") == "ThisIsHellotest");
```

### 4.3 The %FIELD% form

When a field name is embedded in a larger token, wrap it in `%`:

```
record(stringin, "PyDev:EscapeField") {
  field(DTYP, "pydev")
  # Use % sign when field to be replaced is part of the string
  field(INP,  "@ThisIsRecord%VAL%")
}
```

`%VAL%` is also useful when the bare form would be ambiguous:

```
field(INP, "@[v.upper() for v in %VAL%]")
```

### 4.4 Values are native Python objects

Macros are not textual substitution. The field value is converted to a real Python object and bound to a `pydev`-prefixed global:

```cpp
for (auto& macro: Util::getMacros(code)) {
    if      (macro == "VAL")  { args["pydevVAL"]  = Variant(rec->val);  code = Util::replaceMacro(code, "VAL",  "pydevVAL");  }
    else if (macro == "NAME") { args["pydevNAME"] = Variant(rec->name); code = Util::replaceMacro(code, "NAME", "pydevNAME"); }
    ...
}
```

So an `ao` record's `VAL` arrives as a Python `float`, a `longin`'s as an `int`, and `NAME` as a `str`. They work in f-strings and arithmetic directly:

```
record(mbbo, "PyDev:Enum:PrintVal") {
  field(DTYP, "pydev")
  # VAL and RVAL fields become Python variables, and as such can be used in f-string
  field(OUT,  "@print(f'val={VAL} rval={RVAL}')")
  field(ZRVL, "0")
  field(ONVL, "10")
  field(ZRST, "zero")
  field(ONST, "ten")
}
```

### 4.5 Pre-1.3.0 compatibility

Before PyDevice 1.3.0, fields were substituted as **string representations**. This lost float precision and double-converted types. Code written for the old behavior breaks:

```
# Before 1.3.0 -- VAL was spliced in as text, so quotes were needed
field(OUT,  "@print('VAL')")

# 1.3.0 and later -- VAL is a Python object
field(OUT,  "@print(VAL)")
```

On 1.3.0+, `'VAL'` is just a literal string containing the letters V, A, L.

---

## 5. Strings

### 5.1 stringin / stringout -- 40 bytes

```cpp
strncpy(rec->val, val.c_str(), sizeof(rec->val)-1);
```

`VAL` is `char[40]`, so content is silently truncated at 39 characters plus NUL. No warning is issued.

### 5.2 lsi / lso -- long strings

Available on base 3.15+. Honors `SIZV` and sets `LEN`:

```cpp
strncpy(rec->val, val.c_str(), rec->sizv - 1);
```

```
record(lsi, "PyDev:LongString") {
  field(DTYP, "pydev")
  field(INP,  "@device1.get_banner()")
  field(SIZV, "256")
}
```

### 5.3 ASCII only

Strings coming back from Python go through `PyUnicode_AsASCIIString`. Non-ASCII content fails conversion and alarms the record `INVALID`/`CALC`. Encode or sanitize in Python if the device can emit non-ASCII:

```
field(INP, "@device1.read().encode('ascii', 'replace').decode()")
```

---

## 6. Waveforms and Arrays

### 6.1 Reading an array from Python

The returned Python list is converted according to `FTVL`:

```cpp
if (rec->ftvl == menuFtypeFLOAT || rec->ftvl == menuFtypeDOUBLE) {
    std::vector<double> arr = val.get_double_array();
    toRecArrayVal(rec, arr);
} else if (rec->ftvl == menuFtypeSTRING) {
    std::vector<std::string> arr = val.get_string_array();
    toRecArrayVal(rec, arr);
} else {
    std::vector<long long int> arr = val.get_long_array();
    toRecArrayVal(rec, arr);
}
```

Supported `FTVL`: `CHAR` `UCHAR` `SHORT` `USHORT` `LONG` `ULONG` `FLOAT` `DOUBLE` `STRING`. Anything else throws `ConvertError`.

`NORD` is set to `min(len(list), NELM)`; longer lists are truncated silently.

```
record(waveform, "PyDev:Array:Pow2") {
  field(DTYP, "pydev")
  field(INP,  "@list(map(lambda x: x**2, VAL))")
  field(NELM, "5")
  field(FTVL, "LONG")
}
record(waveform, "PyDev:Array:Stat") {
  field(DTYP, "pydev")
  field(INP,  "@[min(VAL),max(VAL),sum(VAL)/len(VAL)]")
  field(NELM, "5")
  field(FTVL, "DOUBLE")
}
```

### 6.2 Passing the array into Python

The `VAL` macro on a waveform becomes a Python **list**:

| `FTVL` | Python list element type | Elements read |
|---|---|---|
| `CHAR` `UCHAR` `SHORT` `USHORT` `LONG` `ULONG` | `int` | `NELM` |
| `FLOAT` `DOUBLE` | `float` | `NELM` |
| `STRING` | `str` | `NORD` |

Note the asymmetry: numeric types read the **entire buffer** (`NELM`), not just the valid portion (`NORD`). A `NELM 1000` waveform holding 3 valid points hands Python a 1000-element list with 997 stale or zero entries. Use `NORD` explicitly if that matters, or size `NELM` tightly.

### 6.3 Strings to CHAR waveforms

A Python `str` returned to a `CHAR`/`LONG` waveform expands to its character codes, because `Variant::get_long_array()` has a scalar-string case:

```cpp
} else if (type == Type::STRING) {
    std::vector<long long int> out;
    for (auto& c: s) {
        out.push_back(c);
    }
    return out;
}
```

This is the idiom for publishing arbitrary-length text:

```
record(waveform, "PyDev:Array:FromString") {
  field(DTYP, "pydev")
  field(INP,  "@str(license)")
  field(NELM, "100")
  field(FTVL, "CHAR")
}
```

Read it back with `caget -S`.

### 6.4 STRING waveforms

Each element is capped at `MAX_STRING_SIZE - 1` = 39 characters. Truncation is reported **only** when `TPRO` is set:

```cpp
if (rec->tpro) {
    if (sval.size() > MAX_STRING_SIZE - 1) {
        // Indicate on console which element will be truncated where
        printf("%s%s' too long\n%s^\n", ...);
    }
}
```

```
record(waveform, "PyDev:Array:Strings") {
  field(DTYP, "pydev")
  field(INP,  "@[b'q3m2d1r', b'd05l2d1r', b'ABCDEFGHIJKLMNOPQRSTUVWXYZ.01234567890']")
  field(NELM, "20")
  field(FTVL, "STRING")
  field(PINI, "YES")
  field(TPRO, 1)
}
```

### 6.5 Heterogeneous lists are silently filtered

The list converter latches onto the type of the first element and **skips** every element of a different type:

```cpp
Variant::Type t = Variant::Type::NONE;
for (Py_ssize_t i = 0; i < PyList_Size(in); i++) {
    PyObject* el = PyList_GetItem(in, i);
    if (PyLong_Check(el) && (t == Variant::Type::NONE || t == Variant::Type::VECTOR_LONG)) {
        ...
        t = Variant::Type::VECTOR_LONG;
    }
    ...
}
```

`[1, 2.5, 3]` yields a 2-element LONG array `[1, 3]` -- no error, no warning, wrong `NORD`. Always return a homogeneous list:

```
field(INP, "@[float(x) for x in device1.readings()]")
```

---

## 7. Record Processing Model

Every `pydev` record is asynchronous. Processing queues the Python onto a worker pool, sets `PACT=1`, and returns; the record processes a second time when the worker completes.

Practical consequences for database design:

* `caput` returns immediately; `caput -c` waits for the Python
* `FLNK` fires only after the Python has completed
* A slow Python call does not block the scan thread
* Records sharing Python state can still interleave -- the record-level ordering guarantee is only what `FLNK` chains give you

Details of the thread pool, `PYDEV_NUM_THREADS`, and GIL behavior are in the `pydevice-ioc` skill.

---

## 8. Error Handling

Any failure gives the same result:

```cpp
recGblSetSevr(rec, epicsAlarmCalc, epicsSevInvalid);
```

* `SEVR` = `INVALID`, `STAT` = `CALC`
* Nothing is printed unless `TPRO` is 1

Triggers: syntax error, any Python exception, or a return value that cannot be converted to the record type.

Because exceptions surface as record alarms, you can let Python raise and read the result from EPICS:

```
record(longout, "Device1:Connect")
{
    field(DTYP, "pydev")
    field(OUT,  "@device1.connect(timeout=2.0)")
    field(PINI, "YES")
}
```

If `connect()` raises, the record goes `INVALID`/`CALC`.

Always set `TPRO 1` while developing a new record.

---

## 9. pydev.iointr() and I/O Intr Scanning

### 9.1 The function

PyDevice injects a built-in module `pydev` with exactly one function:

```python
pydev.iointr(param, value=None)
```

It is installed into `builtins`, so **no import is needed** in user code:

```cpp
exec("import builtins", true);
exec("builtins.pydev=pydev", true);
```

Two modes:

| Call | Role | Returns |
|---|---|---|
| `pydev.iointr('name', value)` | **Push.** Caches the value and immediately triggers all records registered on `'name'`. | `True` |
| `pydev.iointr('name')` | **Pull.** Returns the cached value. | the value, or `None` |

The push is what records process; the pull is what the record's link evaluates to.

If no record registered that name, a push silently does nothing and still returns `True`.

### 9.2 Database side

```
record(longin, "Device1:SentCb") {
  field(DTYP, "pydev")
  field(INP,  "@pydev.iointr('bytes_sent')")
  field(SCAN, "I/O Intr")
}
```

### 9.3 Python side

```python
def send(self, msg):
    sent = self.socket.send(msg)
    self.sent += sent
    # Send an I/O Intr to any interested record
    pydev.iointr('bytes_sent', self.sent)
```

The name is an arbitrary string. It is not tied to a record name or a Python variable; the database and the Python code simply have to agree.

### 9.4 There is no automatic tracking

PyDevice does not watch Python variables. A record with `SCAN "I/O Intr"` will **never process** unless Python explicitly calls the two-argument form. This is the most common source of "my record is stuck in UDF".

### 9.5 Link syntax is matched literally -- not parsed

This is a hard constraint, not a style preference:

```cpp
// This could be better checked with regex
if (addr.find("pydev.iointr('") == 0 && addr.substr(addr.size()-2) == "')") {
    std::string param = addr.substr(14, addr.size()-16);
```

Requirements:

* Must start at **position 0** of the link (immediately after `@`)
* Must use **single quotes**
* Must end with exactly `')`
* No surrounding whitespace or expression

| Link | Registers? |
|---|---|
| `@pydev.iointr('temp')` | yes |
| `@pydev.iointr("temp")` | **no** -- double quotes |
| `@ pydev.iointr('temp')` | **no** -- leading space |
| `@float(pydev.iointr('temp'))` | **no** -- not at position 0 |
| `@pydev.iointr('temp') * 2` | **no** -- trailing text |

A non-matching link still *evaluates* correctly when the record happens to process -- it just never gets an `IOSCANPVT`, so `SCAN "I/O Intr"` silently does nothing. Do any post-processing in Python instead:

```
field(INP, "@pydev.iointr('temp_scaled')")
```

### 9.6 One callback per name, globally

```cpp
void PyWrapper::registerIoIntr(const std::string& name, const Callback& cb)
{
    params[name].first = cb;
    params[name].second = nullptr;
}
```

Assignment, not accumulation. Meanwhile each of the 13 device support files keeps its own `static std::map<std::string, IOSCANPVT> ioScanPvts`.

Result:

* **Multiple records of the same type** sharing a name are fine -- they share one `IOSCANPVT`, and all scan.
* **Records of different types** sharing a name are **not** fine -- each type registers its own callback for that name, and the last one initialized overwrites the others. Only that type's records scan.

```
# BROKEN -- longin and stringin both claim 'status'
record(longin,   "Dev:Code")  { field(INP, "@pydev.iointr('status')") field(SCAN, "I/O Intr") }
record(stringin, "Dev:Msg")   { field(INP, "@pydev.iointr('status')") field(SCAN, "I/O Intr") }

# CORRECT -- distinct names
record(longin,   "Dev:Code")  { field(INP, "@pydev.iointr('status_code')") field(SCAN, "I/O Intr") }
record(stringin, "Dev:Msg")   { field(INP, "@pydev.iointr('status_msg')")  field(SCAN, "I/O Intr") }
```

Python then pushes both:

```python
pydev.iointr('status_code', int(code))
pydev.iointr('status_msg', msg)
```

### 9.7 I/O Intr cannot be selected at runtime

`SCAN "I/O Intr"` is only honored if set when the database is loaded. Registration happens in `init_record`. Writing `I/O Intr` to `.SCAN` via `caput` later will not wire up the interrupt.

### 9.8 Push happens in the Python thread's context

On base 3.16+ the scan callback is synchronous:

```cpp
#  if EPICS_VERSION_INT < VERSION_INT(3,16,0,0)
    scanIoRequest(scan);
#  else
    scanIoImmediate(scan, priorityHigh);
    scanIoImmediate(scan, priorityMedium);
    scanIoImmediate(scan, priorityLow);
#  endif
```

`scanIoImmediate` queues the records inline, so `pydev.iointr('x', v)` blocks the calling Python code at all three priorities before returning. Avoid calling it in a tight loop from a thread holding the GIL.

---

## 10. Writing Python Modules for PyDevice

### 10.1 The dual-mode pattern

Define a stub `pydev` class under `__main__` so the module runs standalone outside the IOC:

```python
import socket

class HttpClient(object):
    def __init__(self, host, port=80):
        self._socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self._addr = (host, port)

    def get(self, url="/"):
        req = "GET %s HTTP/1.1\r\n\r\n" % url
        self._socket.connect(self._addr)
        self._socket.sendall(req)
        rsp = self._socket.recv(1024)
        self._socket.close()
        try:
            proto, code, msg = rsp.split("\r\n")[0].split(" ")
            pydev.iointr('proto', proto)
            pydev.iointr('code',  int(code))
            pydev.iointr('msg',   msg)
        except:
            pass
        return rsp


if __name__ == "__main__":
    class pydev:
        @staticmethod
        def iointr(name, value):
            print("pydev.iointr('%s', '%s')" % (name, value))
    google = HttpClient("www.google.com")
    rsp = google.get()
```

The README's variant uses a no-op body; printing is more useful during development.

### 10.2 Device class shape

A clean PyDevice module exposes a class with methods that map one-to-one onto records:

```python
import socket

class MyDevice(object):

  def __init__(self, ip, port=4011):
    self.ip = ip
    self.port = port
    self.socket = None
    self.sent = 0

  def connect(self, timeout=1.0):
    """ Connects to device, throws exception if can't connect """
    if self.socket:
      self.socket.close()
    self.socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    self.socket.settimeout(timeout)
    self.socket.connect((self.ip, self.port))

  def disconnect(self):
    if self.socket:
      self.socket.close()
      self.socket = None

  def send(self, msg):
    if not self.socket:
      raise RuntimeError("not connected")
    sent = self.socket.send(msg)
    if sent == 0:
      self.disconnect()
      raise RuntimeError("socket connection broken")
    self.sent += sent
    pydev.iointr('bytes_sent', self.sent)
```

Design rules:

| Rule | Reason |
|---|---|
| Raise exceptions on error | They become `INVALID`/`CALC` automatically |
| Return convertible types | `int`, `float`, `str`, `bool`, or a homogeneous list |
| Keep state in the object | Avoids global name collisions |
| Push with `iointr` | Only way to drive `SCAN "I/O Intr"` |
| Thread safety | Workers call methods concurrently; guard shared mutable state |
| Never call `sys.exit()` | It would terminate the IOC |

### 10.3 Thread safety

With the default three workers, two records can call into the same object concurrently. The GIL protects interpreter internals but not your invariants -- a non-atomic read-modify-write on `self.sent`, or a socket shared between a read and a write record, will corrupt. Use `threading.Lock` where it matters:

```python
import threading

class MyDevice(object):
    def __init__(self):
        self._lock = threading.Lock()

    def transact(self, cmd):
        with self._lock:
            self.socket.sendall(cmd)
            return self.socket.recv(1024)
```

---

## 11. The pycalc Record

Base records pass at most one parameter. `pycalc` passes up to ten.

It is a **record type**, not device support, so it needs no `DTYP` -- but it does need `myioc_DBD += pycalcRecord.dbd`.

### 11.1 Minimal example

```
record(pycalc, "PyCalcTest:MathExpr") {
    field(INPA, "17")
    field(INPB, "3")
    field(CALC, "A*B")
}
```

Result is `51.0`, not `51`: constants take their type from `FTA`/`FTB`, which default to `DOUBLE`.

### 11.2 Fields

| Field | Type | Purpose |
|---|---|---|
| `CALC` | `STRING(255)` | Python code. Evaluated every process; changeable at runtime. |
| `INPA`..`INPJ` | `INLINK` | DB links, CA links, or constants |
| `A`..`J` | per `FTx` | Fetched values, exposed to Python as `A`..`J` |
| `FTA`..`FTJ` | menu | Type of each input. Default `DOUBLE`. |
| `MEA`..`MEJ` | `ULONG` | Max elements. `1` = scalar (default); `>1` = array. |
| `NEA`..`NEJ` | `ULONG` | Actual element count at runtime |
| `VAL` | per `FTVL` | Result |
| `FTVL` | menu | `LONG`, `DOUBLE`, or `STRING`. Default `DOUBLE`. |
| `MEVL` | `ULONG` | Max elements in `VAL`. Default `1`. |
| `NEVL` | `ULONG` | Actual element count in `VAL` |
| `OUT` | `OUTLINK` | Optional; `VAL` is forwarded here |

Input types for `FTx`: `CHAR` `UCHAR` `SHORT` `USHOR` `LONG` `ULONG` `INT64` `UINT64` `FLOAT` `DOUBLE` `STRING`. (`INT64`/`UINT64` require `HAVE_EPICS_INT64`.)

Output types for `FTVL` are limited to `LONG`, `DOUBLE`, `STRING`.

### 11.3 Processing

Standard `SCAN`/`PINI` fields apply. An `INPx` link with `CP` or `CPP` makes the record process when that link's monitor posts.

On process the record fetches all `INPx` values, substitutes `A`..`J` (plus `NAME` and `TPRO`) as `pydev`-prefixed globals, runs `CALC`, converts the result to `FTVL` via `dbFastPutConvertRoutine`, posts monitors, and writes `OUT`.

Like device support records, `pycalc` executes on the worker pool with `PACT`.

### 11.4 Arrays

`MEx > 1` makes an input an array; it arrives in Python as a list. `MEVL > 1` makes the output an array; `CALC` must then return a list. Lists longer than `MEVL` are truncated to the first `MEVL` elements, and `NEVL` records the actual count.

### 11.5 Worked examples

**Adaptive types** -- `FTx` forces conversion regardless of the link's native type:

```
record(pycalc, "PyCalcTest:AdaptiveTypes") {
    field(INPA, "PyCalcTest:Input1 CP")
    field(INPB, "PyCalcTest:Input2 CP")
    field(FTA,  "LONG")
    field(FTB,  "DOUBLE")
    field(CALC, "pow(max([A, B]), 2)")
    field(FTVL, "LONG")
}
```

**Alarm on failure:**

```
record(pycalc, "PyCalcTest:InvalidAlarm") {
    field(CALC, "unknown_function()")
}
```

Goes to `CALC`/`INVALID`. Set `TPRO 1` for the traceback.

**Joining two arrays of different types:**

```
record(waveform, "PyCalcTest:InputNumbers") {
    field(FTVL, "DOUBLE")
    field(NELM, "3")
    # waveform initialization only works with EPICS 7.0.2+
    field(INP,  [1.7, 2.3, 14.91])
}
record(waveform, "PyCalcTest:InputStrings") {
    field(FTVL, "STRING")
    field(NELM, "3")
    field(INP,  [one, two, three])
}
record(pycalc, "PyCalcTest:MergeArrays") {
    field(INPA, "PyCalcTest:InputNumbers")
    field(INPB, "PyCalcTest:InputStrings")
    field(FTA,  "DOUBLE")
    field(FTB,  "STRING")
    field(MEA,  "10")
    field(MEB,  "10")
    field(CALC, "list(map(lambda tuple: tuple[1]+':'+str(tuple[0]), zip(A,B)))")
    field(FTVL, "STRING")
    field(MEVL, "10")
    field(PINI, "1")
}
```

```
$ caget -S PyCalcTest:MergeArrays
PyCalcTest:MergeArrays 3 one:1.7 two:2.3 three:14.91
```

Waveform initialization via `field(INP, [...])` needs EPICS 7.0.2+. Otherwise push values first:

```
caput -a PyCalcTest:InputNumbers 3 1.7 2.3 14.91
caput -a PyCalcTest:InputStrings 3 one two three
caput PyCalcTest:MergeArrays.PROC 1
```

### 11.6 A..J are reserved

Because `pycalc` scans for single-letter macros `A` through `J`, those names cannot be used as ordinary Python identifiers in `CALC`. `NAME` and `TPRO` are likewise reserved.

---

## 12. Gotchas

Behaviors not covered by the upstream documentation, verified against commit `7fa3fea` (`R1.3.3-2-g7fa3fea`). Line references are to that commit -- re-check them against newer releases.

### 12.1 ZNAM and ONAM raise NameError on bi/bo

The argument is stored under the wrong key while the code is rewritten to a different name:

```cpp
// src/pydev_bi.cpp:92-93  and  src/pydev_bo.cpp:92-93
else if (macro == "ZNAM") { args["pydevEGU"]  = Variant(rec->znam); code = Util::replaceMacro(code, "ZNAM", "pydevZNAM");  }
else if (macro == "ONAM") { args["pydevHOPR"] = Variant(rec->onam); code = Util::replaceMacro(code, "ONAM", "pydevONAM"); }
```

The code references `pydevZNAM`, but the value was bound to `pydevEGU`, so evaluation raises `NameError` and the record alarms `INVALID`/`CALC`.

These are the only four such mismatches in the codebase; every other record's macros bind correctly.

**Workaround** -- select the string in Python using `VAL`:

```
# BROKEN
field(OUT, "@print(ZNAM)")

# WORKS
field(OUT, "@print('Off' if VAL == 0 else 'On')")
```

### 12.2 iointr link parsing is a literal string match

`src/pydev_longin.cpp:56-57` and the equivalent in all 13 files. Single quotes, position 0, exact `')` terminator. See §9.5.

### 12.3 One iointr callback per name globally

`src/pywrapper.cpp:204` -- `params[name].first = cb` overwrites. See §9.6.

### 12.4 Heterogeneous lists silently filtered

`src/pywrapper.cpp:277-302` -- the converter latches on the first element's type and drops the rest. See §6.5.

### 12.5 Short uppercase identifiers collide with field names

`src/util.cpp:13-42` -- any all-uppercase token of 4 characters or fewer is a macro candidate. A Python constant, class, or enum member named `TCP`, `MAX`, or `LOW` inside a link will be substituted if it matches a supported field for that record type.

```
# 'LOW' is a longin field macro -- substituted, not your constant
field(INP, "@device1.check(LOW)")
```

Use longer names, or lowercase, for anything that is not meant to be a field.

### 12.6 Single shared global namespace

`src/pywrapper.cpp` -- all records share `globDict`, and arguments are injected there rather than into locals (needed so lambdas and comprehensions resolve). State set by one record is visible to all others. Avoid bare globals in links; keep state inside objects. The names `pydevVAL`, `pydevNAME`, `pydevRVAL`, `pydevA`..`pydevJ`, etc. are reserved.

### 12.7 Numeric waveforms read NELM, not NORD

`src/pydev_waveform.cpp:181-198` -- numeric `FTVL` uses `vl.assign(a, a+rec->nelm)` while `STRING` uses `rec->nord`. Oversized numeric waveforms pass stale trailing elements into Python. See §6.2.

### 12.8 Waveform support sits in the write dset slot

`src/pydev_waveform.cpp` ends with `DEVSUPFUN write{(DEVSUPFUN)processRecord};` even though the waveform is read through `INP`. Harmless -- the waveform record calls that slot -- but surprising when reading the source.

### 12.9 Strings must be ASCII

`PyUnicode_AsASCIIString` throughout. Non-ASCII returns fail conversion. See §5.3.

### 12.10 Output records on tagged releases need a return value

Commit `7fa3fea` is not in `R1.3.3`. On the tagged release, `@print(...)` on an output record alarms. See §3.1.

---

## 13. Quick Reference

```
# Fire-and-forget action
record(longout, "Dev:Connect") {
  field(DTYP, "pydev")
  field(OUT,  "@dev.connect(timeout=2.0)")
  field(PINI, "YES")
}

# Read a scalar
record(ai, "Dev:Temp") {
  field(DTYP, "pydev")
  field(INP,  "@dev.temperature()")
  field(SCAN, "1 second")
  field(PREC, "2")
  field(EGU,  "C")
}

# Write a scalar
record(ao, "Dev:Setpoint") {
  field(DTYP, "pydev")
  field(OUT,  "@dev.set_setpoint(VAL)")
  field(EGU,  "C")
}

# Push-driven read
record(longin, "Dev:Count") {
  field(DTYP, "pydev")
  field(INP,  "@pydev.iointr('count')")
  field(SCAN, "I/O Intr")
}

# Array read
record(waveform, "Dev:Spectrum") {
  field(DTYP, "pydev")
  field(INP,  "@dev.spectrum()")
  field(NELM, "1024")
  field(FTVL, "DOUBLE")
  field(SCAN, "1 second")
}

# Long string via CHAR waveform
record(waveform, "Dev:Banner") {
  field(DTYP, "pydev")
  field(INP,  "@dev.banner()")
  field(NELM, "256")
  field(FTVL, "CHAR")
}

# Multi-parameter call
record(pycalc, "Dev:Interp") {
  field(INPA, "Dev:Raw CP")
  field(INPB, "Dev:Gain")
  field(INPC, "Dev:Offset")
  field(CALC, "dev.calibrate(A, B, C)")
  field(FTVL, "DOUBLE")
}
```

Development checklist for a new record:

```
[ ] DTYP "pydev"  (not needed for pycalc)
[ ] Link starts with @
[ ] Code is a single line
[ ] Input records return a convertible value
[ ] Field macros are uppercase, <= 4 chars, word-bounded; %FIELD% if embedded
[ ] No ZNAM/ONAM on bi/bo
[ ] Lists are homogeneous
[ ] waveform NELM sized tightly; FTVL matches the Python element type
[ ] iointr links use single quotes and nothing else in the link
[ ] iointr names unique across record types
[ ] SCAN "I/O Intr" set in the db file, not at runtime
[ ] TPRO 1 while developing
```
