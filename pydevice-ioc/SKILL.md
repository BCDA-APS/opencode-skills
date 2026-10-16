---
name: pydevice-ioc
description: Build and configure EPICS IOCs with PyDevice -- python-config linking, RELEASE and CONFIG.PyDevice integration, PYTHONPATH and virtual environments, the pydev iocsh command, worker threads and GIL behavior, and debugging with TPRO
---

# PyDevice IOC Skill

You are an expert at building and configuring EPICS IOCs that use PyDevice. PyDevice is device support that lets EPICS records execute Python code. Your task covers everything outside the database file: compiling the module, linking it into an IOC, setting up the Python search path, understanding the concurrency model, and debugging failures.

For database record syntax, field macros, `pydev.iointr()`, waveforms, and the `pycalc` record, use the **`pydevice-records`** skill instead.

**Reference version:** This skill documents PyDevice at commit `7fa3fea` (`R1.3.3-2-g7fa3fea`), two commits past the `R1.3.3` tag. Source: <https://github.com/klemenv/PyDevice>. License: GPL v3.

---

## 1. What PyDevice Is

PyDevice embeds a CPython interpreter in the IOC process and exposes it through three things:

| Surface | Count | Detail |
|---|---|---|
| Device support type | 1 | `DTYP "pydev"`, `INST_IO` links, 13 record types |
| Custom record type | 1 | `pycalc` -- passes up to 10 parameters to Python |
| iocsh command | 1 | `pydev("<single line of Python>")` |
| Python API | 1 function | `pydev.iointr(param, value=None)` |

The design intent is that a Python module is written and tested standalone, then wired to EPICS PVs without modification. There is no code generation and no stub layer -- records call Python source text held in their `INP`/`OUT` fields.

Key consequence: **there is a single global Python interpreter and a single global namespace shared by every record in the IOC.**

---

## 2. Dependencies

* EPICS 3.14+ or 7.0+
* Python 2.x or 3.5+ -- **headers and libraries**, not just the runtime

```bash
# RHEL / CentOS / Rocky
yum install python3-devel

# Debian / Ubuntu
apt install python3-dev     # for Python 3
apt install python-dev      # for Python 2
```

The build invokes `python-config`, which ships in the `-devel`/`-dev` package. If `python-config` is missing, the build fails with an empty `CONFIG.PyDevice` rather than a clear error.

---

## 3. Building PyDevice from Source

### 3.1 Point at EPICS base

Edit `configure/RELEASE.local` (preferred) rather than `configure/RELEASE`:

```make
EPICS_BASE = /path/to/epics/base
```

`configure/RELEASE` already ends with `-include $(TOP)/configure/RELEASE.local`, so the local file wins.

Note `configure/CONFIG` overrides the base default:

```make
# Override the Base definition:
INSTALL_LOCATION = $(TOP)
```

So headers, libraries, and DBD files install in-tree under `$(TOP)/lib`, `$(TOP)/dbd`, etc. An IOC referencing `PYDEVICE` picks them up from the source tree directly.

### 3.2 Select the right python-config

`configure/CONFIG` hard-codes the default:

```make
# As soon as python3 is the default configuration
# PYTHON_CONFIG=python3-config
PYTHON_CONFIG=python-config
```

On most modern systems there is no bare `python-config`. Override it in `configure/CONFIG_SITE.local`:

```make
PYTHON_CONFIG = python3-config
```

A specific interpreter version or a venv-local tool works too:

```make
PYTHON_CONFIG = python3.11-config
PYTHON_CONFIG = /opt/python/3.11/bin/python3-config
```

### 3.3 The generated CONFIG.PyDevice

`configure/Makefile` generates `configure/CONFIG.PyDevice` at build time. It is gitignored and must never be edited by hand:

```make
TARGETS = $(CONFIG_TARGETS) CONFIG.PyDevice

# Starting with Python 3.8, python-config needs --embed flag
ifeq (,$(findstring embed,$(shell $(PYTHON_CONFIG) --help)))
	PYDEV_SYS_PROD_LIBS = $(patsubst -l%,%,$(filter -l%,$(shell $(PYTHON_CONFIG) --ldflags)))
else
	PYDEV_SYS_PROD_LIBS = $(patsubst -l%,%,$(filter -l%,$(shell $(PYTHON_CONFIG) --ldflags --embed)))
endif
PYDEV_USR_CXXFLAGS = $(filter -I%,$(shell $(PYTHON_CONFIG) --cflags))

CONFIG.PyDevice:
	@echo "SYS_PROD_LIBS += $(PYDEV_SYS_PROD_LIBS)" > $(TOP)/configure/$@
	@echo "USR_CXXFLAGS += $(PYDEV_USR_CXXFLAGS)" >> $(TOP)/configure/$@
```

The result is a two-line file:

```make
SYS_PROD_LIBS += python3.11
USR_CXXFLAGS += -I/usr/include/python3.11
```

The `--embed` detection matters: from Python 3.8 onward, plain `python3-config --ldflags` omits `-lpython3.x` because the expected consumer is an extension module, not an embedding application. PyDevice embeds, so it needs `--embed`. The `findstring embed` test on `--help` output handles both eras automatically.

**If linking fails with undefined `Py_Initialize`:** inspect the generated file. An empty `SYS_PROD_LIBS` means the `--embed` probe failed, usually because `PYTHON_CONFIG` points at something that is not a real `python-config`.

### 3.4 Build

```bash
cd /path/to/PyDevice
make
```

Products:

* `lib/$(EPICS_HOST_ARCH)/libpydev.so` -- the device support library
* `dbd/pydev.dbd`, `dbd/pydev315.dbd`, `dbd/pycalcRecord.dbd`
* `bin/$(EPICS_HOST_ARCH)/pydevioc` -- a demo IOC
* `include/pycalcRecord.h` -- generated from `DBDINC`

Run the demo:

```bash
cd iocBoot/iocpydev
./st.cmd
```

After this, database files and Python modules can be edited without rebuilding C++.

### 3.5 Build flags and conditionals

`src/Makefile` requires two flags:

```make
USR_CXXFLAGS += -std=c++11 -O2 -DUSE_TYPED_RSET
```

* `-std=c++11` -- the code uses move semantics (`ByteCode`), lambdas, `std::atomic`, and in-class initializers
* `-DUSE_TYPED_RSET` -- required for the `pycalc` record's typed `rset` structure

Version conditionals:

| Condition | Effect |
|---|---|
| `ifdef BASE_3_15` | Adds `pydev315.dbd` plus `pydev_lsi.cpp` and `pydev_lso.cpp`. On base 3.14, `lsi`/`lso` are unavailable. |
| `EPICS_VERSION_INT < VERSION_INT(3,16,0,0)` | I/O Intr uses `scanIoRequest()`. On 3.16+, uses `scanIoImmediate()` at all three priorities. See §6.3. |
| `HAVE_EPICS_INT64` | Enables `INT64`/`UINT64` input types in the `pycalc` record. |
| `PY_MAJOR_VERSION < 3` | Selects `PyString_*` vs `PyUnicode_*` and the `__builtin__` vs `builtins` module name. |

Two cosmetic defects in `src/Makefile` worth knowing so you do not copy them:

```make
CXXFLAGS += -g -ggdb -O0          # line 9: contradicts the -O2 above
pydevice_LIBS += $(EPICS_BASE_IOC_LIBS)   # line 43: typo, library is 'pydev', so this is a no-op
```

### 3.6 Run the unit tests

```bash
make runtests
```

Two host test programs:

| Test | Covers |
|---|---|
| `testutil` | 53 assertions on `Util::replaceMacro`, `getMacros`, `escape`, `join` -- the macro substitution rules |
| `testpywrapper` | 48 assertions on `PyWrapper::exec()` return-value type coercion |

`testpywrapper` links `pywrapper.cpp` and `variant.cpp` directly and calls `PyWrapper::init()`, so it embeds Python without any EPICS record machinery. It is the fastest way to check that the Python linkage is correct.

---

## 4. Adding PyDevice to an Existing IOC

Four steps.

### Step 1 -- Declare the module

In the IOC's `configure/RELEASE` (or `RELEASE.local`):

```make
PYDEVICE = /path/to/PyDevice
```

`CHECK_RELEASE = YES` is set in PyDevice's own `CONFIG_SITE`, so inconsistent module paths across the dependency tree will hard-fail the build rather than warn.

### Step 2 -- Include the generated config

Near the **top** of the IOC's `xxxApp/src/Makefile`, right after `CONFIG`:

```make
TOP=../..
include $(TOP)/configure/CONFIG
include $(PYDEVICE)/configure/CONFIG.PyDevice
```

Placement matters: this supplies `USR_CXXFLAGS` (Python headers) and `SYS_PROD_LIBS` (the Python library), which must be set before the rules are read.

### Step 3 -- Add the DBD and library

```make
myioc_DBD  += pydev.dbd
myioc_LIBS += pydev
```

> **Correction to upstream documentation.** `README.md` line 258 says `<yourioc>_LIB += pydev`. The correct EPICS variable is `_LIBS`. The working example in `testApp/src/Makefile` uses `pydevioc_LIBS += pydev`. Using `_LIB` silently does nothing and the link fails with undefined `devPyDev*` symbols.

### Step 4 (optional) -- Add the pycalc record

Only if the IOC uses `pycalc`:

```make
myioc_DBD += pycalcRecord.dbd
```

### Complete reference Makefile

Adapted from `testApp/src/Makefile`:

```make
TOP=../..

include $(TOP)/configure/CONFIG
include $(PYDEVICE)/configure/CONFIG.PyDevice

PROD_IOC = myioc
DBD += myioc.dbd

myioc_DBD += base.dbd
myioc_DBD += pydev.dbd
myioc_DBD += pycalcRecord.dbd

myioc_LIBS += pydev

myioc_SRCS += myioc_registerRecordDeviceDriver.cpp
myioc_SRCS_DEFAULT += main.cpp
myioc_SRCS_vxWorks += -nil-

myioc_LIBS += $(EPICS_BASE_IOC_LIBS)

include $(TOP)/configure/RULES
```

Keep `myioc_LIBS += $(EPICS_BASE_IOC_LIBS)` last -- base libraries must come after support libraries on the link line.

---

## 5. IOC Startup (st.cmd)

### 5.1 Complete working example

From `iocBoot/iocpydev/st.cmd`:

```shell
#!../../bin/linux-x86_64/pydevioc

< envPaths

# PYTHONPATH points to folders where Python modules are.
epicsEnvSet("PYTHONPATH","$(TOP)/python")

cd ${TOP}

## Register all support components
dbLoadDatabase "${TOP}/dbd/pydevioc.dbd"
pydevioc_registerRecordDeviceDriver pdbbase

## Load record instances
dbLoadRecords("${TOP}/db/pydevtest.db")
dbLoadRecords("${TOP}/db/pycalcrectest.db")

cd ${TOP}/iocBoot/${IOC}

pydev("import time")
pydev("import pydevtest")
pydev("google = pydevtest.HttpClient('www.google.com', 80)")

iocInit
```

### 5.2 Ordering rules

| Must happen | Before | Why |
|---|---|---|
| `epicsEnvSet("PYTHONPATH", ...)` | first `pydev()` call | the interpreter resolves `sys.path` from the environment at import time |
| `registerRecordDeviceDriver` | first `pydev()` call | the `pydevRegister` registrar is what initializes the interpreter and registers the `pydev` iocsh command |
| `dbLoadRecords` | `iocInit` | standard; also required because I/O Intr registration happens in `init_record` |
| `pydev()` object setup | `iocInit` | records with `PINI YES` process during `iocInit` and will reference names that must already exist |

Putting `pydev("device = MyDevice(...)")` after `iocInit` is a common mistake -- any `PINI YES` record referencing `device` raises `NameError` and goes to `INVALID`/`CALC`.

### 5.3 PYTHONPATH

Set it before launching the IOC, or in the shell. Chaining across modules works because `epicsEnvSet` expands the previous value:

```shell
epicsEnvSet("PYTHONPATH","$(MODULE)/python")
epicsEnvSet("PYTHONPATH","$(PYTHONPATH):$(TOP)/python")
```

Python then searches `$(MODULE)/python/` and `$(TOP)/python/`.

### 5.4 Virtual environments

Activate before launching the IOC. No special configuration is needed:

```bash
source /home/controls/common/python_env/main/env_scan/bin/activate
./st.cmd
```

Packages installed in the venv become importable from records.

**Caveat:** the embedded interpreter is the one linked at build time via `PYTHON_CONFIG`. The venv only supplies `sys.path` entries. A venv built on a *different minor version* than the linked `libpython` will produce import failures for compiled extension modules (`.so` files with ABI tags). Match the venv's Python minor version to the `PYTHON_CONFIG` used at build time.

---

## 6. Runtime Architecture

### 6.1 The pydev iocsh command

```cpp
epicsShareFunc int pydev(const char *line)
{
    try {
        PyWrapper::exec(line, true);
    } catch (...) {
        // pass
    }
    return 0;
}

static const iocshArg pydevArg0 = { "pycode", iocshArgString };
static const iocshArg *const pydevArgs[] = { &pydevArg0 };
static const iocshFuncDef pydevDef = { "pydev", 1, pydevArgs };
```

Properties:

* Exactly one string argument -- a **single line** of Python
* `debug` is hard-coded `true`, so tracebacks always print to the console (unlike records, which need `TPRO`)
* Exceptions are swallowed and the return is always `0`, so a failing `pydev()` call **will not abort st.cmd**
* Executes in the global namespace shared with all records

Useful interactively after `iocInit` for inspecting state:

```
epics> pydev("print(device1.sent)")
epics> pydev("import sys; print(sys.path)")
```

### 6.2 Initialization and shutdown

`src/epicsdevice.cpp` registers everything once:

```cpp
static void pydevRegister(void)
{
    static bool firstTime = true;
    if (firstTime) {
        firstTime = false;
        auto numThreads = Util::getEnvConfig("PYDEV_NUM_THREADS", 3);
        if (numThreads < 1)
            numThreads = 3;

        PyWrapper::init();
        AsyncExec::init(numThreads);
        iocshRegister(&pydevDef, pydevCall);
        epicsAtExit(pydevUnregister, 0);
    }
}
epicsExportRegistrar(pydevRegister);
```

Shutdown order is deliberate:

```cpp
static void pydevUnregister(void*)
{
    AsyncExec::shutdown();   // stop and join all workers FIRST
    PyWrapper::shutdown();   // then Py_Finalize()
}
```

Workers are joined before `Py_Finalize()` so no thread is inside the interpreter when it is torn down.

### 6.3 Concurrency model

Every `pydev` record processes **asynchronously**. The pattern is identical in all 13 device support files:

```cpp
static long processRecord(longoutRecord* rec)
{
    auto ctx = reinterpret_cast<PyDevContext*>(rec->dpvt);
    if (ctx == nullptr) {
        // Keep PACT=1 to prevent further processing
        rec->pact = 1;
        recGblSetSevr(rec, epicsAlarmUDF, epicsSevInvalid);
        return -1;
    }

    if (rec->pact == 1) {        // second pass, after the worker finished
        rec->pact = 0;
        return ctx->processCbStatus;
    }
    rec->pact = 1;               // first pass: queue the work and return

    auto scheduled = AsyncExec::schedule([rec]() {
        processRecordCb(rec);
    });
    return (scheduled ? 0 : -1);
}
```

The worker finishes by requesting re-processing at the record's own priority:

```cpp
callbackRequestProcessCallback(&ctx->callback, rec->prio, rec);
```

Sequence:

1. Record processes. Python source is captured, macros substituted, work pushed onto a FIFO queue. `PACT=1`. The scan thread returns immediately.
2. A worker thread dequeues and executes the Python.
3. The worker stores the result in the record and calls `callbackRequestProcessCallback`.
4. The record processes a second time on the EPICS callback thread, clears `PACT`, returns the saved status, and posts monitors and the forward link.

Implications:

* `caput` without callback returns immediately
* `caput -c` (put-callback) correctly waits for the Python to finish
* `FLNK` fires only after the Python completes
* A slow Python call does **not** block the scan thread

### 6.4 Worker thread pool

`src/asyncexec.cpp` implements a plain FIFO pool:

```cpp
void run() override
{
    while (running) {
        AsyncExec::Callback cb;
        if (g_tasks.dequeue(1.0, cb)) {
            cb();
        }
    }
}
```

| Property | Value |
|---|---|
| Thread names | `PyDeviceExec_0`, `PyDeviceExec_1`, ... (visible in `epicsThreadShowAll`) |
| Stack size | `epicsThreadStackMedium` |
| Queue | `std::list` guarded by `epicsMutex`, signalled by `epicsEvent` |
| Dequeue timeout | 1.0 s, so shutdown responds within one second |
| Default count | 3 |

Control the count with an environment variable, read via `Util::getEnvConfig("PYDEV_NUM_THREADS", 3)`:

```bash
export PYDEV_NUM_THREADS=8
./st.cmd
```

Values below 1 are coerced back to 3.

**Saturation behavior.** The queue is unbounded; it never rejects work. If all workers are busy, requests simply accumulate and latency grows. `AsyncExec::schedule()` returns `false` only when the pool is empty (initialization failed) or the callback is null -- in which case `processRecord` returns `-1` and the record alarms.

Size the pool to the number of concurrently *blocking* Python operations. The demo database exercises this with two records sleeping 1 s and 2 s:

```
record(longout, "PyDev:Threads:Exec1") {
  field(DTYP, "pydev")
  field(OUT,  "@time.sleep(1.0)")
  field(FLNK, "PyDev:Threads:Print1")
}
```

### 6.5 GIL handling

A RAII guard wraps every interpreter entry point (`compile`, `eval`, `destroy`):

```cpp
struct PyGIL
{
    PyGILState_STATE state;
    PyGIL()  { state = PyGILState_Ensure(); }
    ~PyGIL() { PyGILState_Release(state); }
};
```

Initialization releases the GIL and saves the main thread state so workers can acquire it:

```cpp
mainThread = PyEval_SaveThread();
```

Shutdown restores it before finalizing:

```cpp
void PyWrapper::shutdown()
{
    PyEval_RestoreThread(mainThread);
    mainThread = nullptr;
    Py_DecRef(globDict);
    Py_DecRef(locDict);
    Py_Finalize();
}
```

**What parallelism you actually get.** The GIL serializes bytecode execution, so N workers do not give N times the throughput for pure-Python computation. Real concurrency comes from operations that release the GIL:

* Socket and file I/O
* `time.sleep()`
* NumPy, SciPy, and most compiled extension modules during heavy computation
* `subprocess` waits

A record running a tight pure-Python loop will hold the GIL and stall every other `pydev` record regardless of `PYDEV_NUM_THREADS`.

### 6.6 Global namespace

All records share one namespace. Arguments are injected into the **globals** dict, not locals:

```cpp
PyDict_SetItemString(globDict, keyval.first.c_str(), item);
...
PyObject *r = PyEval_EvalCode(code, globDict, locDict);
```

Globals are used deliberately -- lambdas and comprehensions do not capture the local dict, so `@list(map(lambda x: x**2, VAL))` would fail otherwise.

Two consequences:

**Intentional:** state persists and is shared between records.

```
record(longout, "PyDev:Long:SetB") {
  field(DTYP, "pydev")
  field(OUT,  "@b=VAL")
  field(FLNK, "PyDev:Long:ReadB")
}
record(longin, "PyDev:Long:ReadB") {
  field(DTYP, "pydev")
  field(INP,  "@b")
}
```

**Hazardous:** two unrelated records using the same variable name silently clobber each other. There is no per-record scope. Also note that every field macro lands in globals as `pydevVAL`, `pydevNAME`, `pydevA`, and so on -- avoid those names in your own modules.

Mitigate by keeping record links thin and holding state inside module-level objects:

```
field(OUT, "@device1.set_setpoint(VAL)")      # good -- state lives in device1
field(OUT, "@setpoint = VAL")                 # risky -- bare global
```

### 6.7 Bytecode caching

Each record caches its compiled bytecode and recompiles only when the macro-substituted source changes:

```cpp
if (ctx->code != code) {
    PyWrapper::destroy(std::move(ctx->bytecode));
    ctx->bytecode = PyWrapper::compile(code, (rec->tpro == 1));
    ctx->code = code;
}
auto ret = PyWrapper::eval(ctx->bytecode, args, (rec->tpro == 1));
```

Because macro values are passed as *arguments* rather than spliced into the text, the substituted string is normally stable and compilation happens once. The exception is `@VAL` on a `stringout`, where the record value *is* the code -- that recompiles on every change, which is the intended behavior for an "exec arbitrary Python" PV.

---

## 7. Debugging

### 7.1 Error reporting model

Any failure -- syntax error, exception, or unconvertible return value -- produces the same record state:

```cpp
} catch (std::exception& e) {
    if (rec->tpro == 1) {
        printf("[%s] %s\n", rec->name, e.what());
    }
    recGblSetSevr(rec, epicsAlarmCalc, epicsSevInvalid);
    ctx->processCbStatus = -1;
}
```

* `SEVR` = `INVALID`
* `STAT` = `CALC`
* Console output **only if `TPRO` is 1**

This is the single most important debugging fact: by default a broken record alarms silently. There is no traceback until you enable `TPRO`.

### 7.2 Enable tracing

```bash
caput MyRecord.TPRO 1
```

Or in the database during development:

```
record(waveform, "PyDev:Array:Strings") {
  field(DTYP, "pydev")
  field(INP,  "@[b'one', b'two']")
  field(NELM, "20")
  field(FTVL, "STRING")
  field(TPRO, 1)
}
```

With `TPRO=1` you get:

* The C++ exception text, e.g. `Failed to compile '...', syntax error`
* A full Python traceback via `PyErr_Print()`
* Record-specific warnings such as waveform string truncation

### 7.3 Interrogate state from iocsh

The `pydev` command always prints errors, so it is a better probe than a record:

```
epics> pydev("print(device1)")
epics> pydev("import traceback; traceback.print_stack()")
epics> pydev("print([n for n in dir() if not n.startswith('_')])")
```

### 7.4 Diagnostic checklist

| Symptom | Likely cause |
|---|---|
| Link error, undefined `Py_Initialize` | `CONFIG.PyDevice` has empty `SYS_PROD_LIBS`; wrong `PYTHON_CONFIG`, or missing `--embed` support |
| Link error, undefined `devPyDev*` | Makefile used `_LIB` instead of `_LIBS` |
| `Record "x" has unknown DTYP "pydev"` | `pydev.dbd` not in `myioc_DBD` |
| `Record type "pycalc" not found` | `pycalcRecord.dbd` not in `myioc_DBD` |
| `ModuleNotFoundError` | `PYTHONPATH` set after the first `pydev()` call, or wrong path |
| `NameError` at `iocInit` | `pydev()` object construction placed after `iocInit` |
| Record `INVALID`/`CALC`, no console output | Normal -- set `TPRO 1` to see why |
| I/O Intr record never processes | Link syntax; see `pydevice-records` §9 |
| All records stall together | A record is holding the GIL in pure-Python compute |
| Latency grows under load | Worker pool too small; raise `PYDEV_NUM_THREADS` |
| Import works standalone but not in IOC | venv Python minor version differs from the linked `libpython` |

### 7.5 Thread inspection

```
epics> epicsThreadShowAll
```

Look for `PyDeviceExec_0` through `PyDeviceExec_N-1`. If they are absent, `AsyncExec::init()` failed and every record will alarm. If they are all perpetually busy, the pool is undersized or something is blocking without releasing the GIL.

---

## 8. Deployment Checklist

```
[ ] Python dev package installed (python3-devel / python3-dev)
[ ] PYTHON_CONFIG set in configure/CONFIG_SITE.local
[ ] EPICS_BASE set in configure/RELEASE.local
[ ] PyDevice builds; configure/CONFIG.PyDevice has non-empty SYS_PROD_LIBS
[ ] make runtests passes
[ ] PYDEVICE defined in the IOC's configure/RELEASE
[ ] include $(PYDEVICE)/configure/CONFIG.PyDevice near the top of the app Makefile
[ ] myioc_DBD  += pydev.dbd
[ ] myioc_LIBS += pydev          (LIBS, not LIB)
[ ] myioc_DBD  += pycalcRecord.dbd   (only if using pycalc)
[ ] PYTHONPATH set in st.cmd before any pydev() call
[ ] pydev() imports and object construction placed before iocInit
[ ] PYDEV_NUM_THREADS sized for concurrent blocking operations
[ ] Python modules import cleanly standalone (python3 -c "import mymodule")
[ ] venv Python minor version matches the linked libpython
```
