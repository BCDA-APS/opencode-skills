---
name: pvxs-client
description: Write PVXS PV Access client programs in C++ -- Context, Get/Put/Monitor/RPC operations, Value data container, TypeDef, shared_array, Normative Types, and pvRequest composition
---

You are an expert at writing EPICS PV Access client programs using the PVXS library (the modern replacement for pvAccessCPP + pvDataCPP). You know the pvxs::client API, the pvxs::Value data container, builder patterns, callback signatures, and threading rules.

## 1. Headers and Linking

```cpp
#include <pvxs/client.h>      // Context, Get/Put/Monitor/RPC builders, Operation, Subscription
#include <pvxs/data.h>        // Value, TypeDef, TypeCode, Member
#include <pvxs/sharedArray.h> // shared_array<T>
#include <pvxs/nt.h>          // NTScalar, NTEnum, NTTable, NTNDArray, NTURI
#include <pvxs/log.h>         // DEFINE_LOGGER, log_*_printf, logger_config_env
#include <pvxs/util.h>        // SigInt, MPMCFIFO, escape
```

**Makefile** (EPICS build system):

```makefile
PROD_LIBS += pvxs
PROD_LIBS += Com
# If using EPICS Base >= 7:
PROD_SYS_LIBS += event_core event_pthreads
# Or if pvxs bundles libevent:
# (handled automatically by pvxs configure rules)
```

Namespace: `pvxs`, `pvxs::client`, `pvxs::nt`, `pvxs::members`.

## 2. Value -- The Data Container

`pvxs::Value` replaces the entire pvDataCPP `PVField` hierarchy with a single class. A `Value` references a node in a type/data tree.

### Creating Values

```cpp
using namespace pvxs;
using namespace pvxs::members;

// From TypeDef
TypeDef def(TypeCode::Struct, "my_struct_t", {
    Int32("count"),
    Float64("position"),
    Struct("alarm", "alarm_t", {
        Int32("severity"),
        Int32("status"),
        String("message"),
    }),
    Float64A("waveform"),
});
Value val = def.create();

// From Normative Type
Value val = nt::NTScalar{TypeCode::Float64}.create();
```

### Field Access

```cpp
// operator[] -- returns invalid Value if field not found
Value v = top["value"];
Value sev = top["alarm.severity"];      // dotted path
Value elem = top["dimension[0].size"];  // array element access
Value sel = top["->booleanValue"];      // union selector

// .lookup() -- throws LookupError if not found
Value v = top.lookup("value");

// Check validity before use
if(auto v = top["optional_field"])
    doSomething(v);
```

### Reading Values

```cpp
// .as<T>() -- extract with type conversion; throws NoField/NoConvert
int32_t ival = top["value"].as<int32_t>();
double dval = top["value"].as<double>();
std::string sval = top["value"].as<std::string>();

// .as<T>(ref) -- returns false instead of throwing
double d;
if(top["value"].as(d))
    std::cout << d;

// .as<T>(lambda) -- calls lambda only if convertible
top["value"].as<double>([](double d) {
    std::cout << d;
});

// Arrays
auto arr = top["value"].as<shared_array<const double>>();
auto raw = top["value"].as<shared_array<const void>>();
```

### Writing Values

```cpp
// .from<T>() -- assign with type conversion; throws NoField/NoConvert
top["value"].from(42.0);
top["alarm.severity"].from(1);

// operator= shorthand (not for T=Value)
top["value"] = 42.0;
top["alarm.message"] = "HIHI";

// .update(key, val) -- shorthand for (*this)[key].from(val); returns *this
top.update("value", 42.0).update("alarm.severity", 1);

// .tryFrom() -- returns false instead of throwing
top["value"].tryFrom(42.0);

// Arrays
shared_array<double> arr({1.0, 2.0, 3.0});
top["value"] = arr.freeze();  // must freeze mutable -> const
```

### Type Introspection

```cpp
TypeCode tc = val.type();          // e.g. TypeCode::Float64
bool isStruct = tc == TypeCode::Struct;
bool isArr = tc.isarray();
const std::string& id = val.id(); // struct type ID, e.g. "epics:nt/NTScalar:1.0"
val.idStartsWith("epics:nt/NTScalar:");
```

### Marking (Change Tracking)

`Value` fields track which fields have been modified. Network operations use marks to send only changed fields.

```cpp
val["value"].mark();                      // mark this field as changed
val["value"].unmark();                    // clear mark
bool changed = val["value"].isMarked();   // check mark
Value v = val["value"].ifMarked();        // returns valid Value only if marked

// from() and operator= automatically mark the field
```

### Cloning

```cpp
Value copy = val.clone();         // deep copy with values
Value empty = val.cloneEmpty();   // same type, default values, nothing marked
```

### Iteration

```cpp
// All descendants (depth-first)
for(auto fld : val.iall())
    std::cout << val.nameOf(fld) << " : " << fld.type() << "\n";

// Immediate children only
for(auto child : val.ichildren())
    std::cout << val.nameOf(child) << "\n";

// Only marked descendants
for(auto fld : val.imarked())
    std::cout << val.nameOf(fld) << " = " << fld << "\n";
```

### Formatting / Printing

```cpp
std::cout << val;                                 // default tree format
std::cout << val.format().delta();                // delta format (marked fields only)
std::cout << val.format().arrayLimit(10);         // truncate arrays
std::cout << val.format().showValue(false);       // type info only
```

## 3. TypeDef and Member

`TypeDef` builds type definitions. `Member` defines individual fields.

```cpp
using namespace pvxs::members;

// Scalar helpers: Bool, Int8..Int64, UInt8..UInt64, Float32, Float64, String, Any
// Array helpers:  BoolA, Int8A..Int64A, UInt8A..UInt64A, Float32A, Float64A, StringA, AnyA
// Compound helpers: Struct, Union, StructA, UnionA

TypeDef def(TypeCode::Struct, "my:type:1.0", {
    Float64("value"),
    Struct("alarm", "alarm_t", {
        Int32("severity"),
        String("message"),
    }),
    StringA("labels"),
    Float64A("data"),
    Union("variant", {
        Float64("d"),
        String("s"),
    }),
});

Value val = def.create();

// Append members to existing TypeDef
def += {
    Int32("extra_field"),
};

// Use a TypeDef as a sub-member of another
TypeDef inner(TypeCode::Struct, "inner_t", { Int32("x") });
TypeDef outer(TypeCode::Struct, "outer_t", {
    inner.as("nested"),           // embed as field named "nested"
});
```

## 4. shared_array

Reference-counted contiguous array. Follows copy-on-write semantics via `freeze()`/`thaw()`.

```cpp
using namespace pvxs;

// Construction
shared_array<double> arr({1.0, 2.0, 3.0});          // from initializer_list
shared_array<double> arr(100);                        // 100 default-constructed elements
shared_array<double> arr(100, 0.0);                   // 100 elements initialized to 0.0

// Element access
arr[0] = 42.0;
double v = arr.at(0);  // bounds-checked

// Size
size_t n = arr.size();
bool empty = arr.empty();

// STL iteration
for(auto& x : arr) x *= 2.0;
std::sort(arr.begin(), arr.end());

// freeze: mutable -> const (transfers ownership, clears source)
shared_array<const double> frozen = arr.freeze();
// arr is now empty; frozen holds the data

// thaw: const -> mutable (copies if not unique, clears source)
shared_array<double> thawed = frozen.thaw();

// Assigning to a Value field
shared_array<double> data({1.0, 2.0, 3.0});
val["value"] = data.freeze();

// Reading from a Value field
auto data = val["value"].as<shared_array<const double>>();

// Type-erased (void) arrays
shared_array<const void> raw = val["value"].as<shared_array<const void>>();
ArrayType atype = raw.original_type();  // e.g. ArrayType::Float64
auto typed = raw.castTo<const double>(); // throws if type mismatch
```

## 5. Normative Types

Pre-built type definitions following the EPICS Normative Types specification.

### NTScalar / NTScalarArray

```cpp
using namespace pvxs::nt;

// Minimal
Value val = NTScalar{TypeCode::Float64}.create();

// With metadata sub-structures
Value val = NTScalar{TypeCode::Float64, true, true, true}.create();
//                   value type,        display, control, valueAlarm

// NTScalarArray: pass an array TypeCode
Value val = NTScalar{TypeCode::Float64A}.create();

// Populate
val["value"] = 42.0;
val["alarm.severity"] = 2;
val["alarm.message"] = "HIHI";
val["timeStamp.secondsPastEpoch"] = ts.secPastEpoch + POSIX_TIME_AT_EPICS_EPOCH;
val["timeStamp.nanoseconds"] = ts.nsec;
val["display.units"] = "mm";
```

### NTEnum

```cpp
Value val = NTEnum{}.create();
val["value.index"] = 1;
// choices must be set as a string array
shared_array<const std::string> choices({"Off", "On", "Error"});
val["value.choices"] = choices;
```

### NTTable

```cpp
NTTable tbl;
tbl.add_column(TypeCode::StringA, "name", "Name");
tbl.add_column(TypeCode::Float64A, "position", "Position");
Value val = tbl.create();  // also populates labels

// Fill columns
val["value.name"] = shared_array<const std::string>({"m1", "m2"});
val["value.position"] = shared_array<const double>({1.5, 2.7}).freeze();
```

### NTNDArray

```cpp
Value val = NTNDArray{}.create();
```

### NTURI (for RPC arguments)

```cpp
NTURI uri({
    Member(TypeCode::Float64, "lhs"),
    Member(TypeCode::Float64, "rhs"),
});
Value arg = uri.call(1.0, 2.0);  // creates Value with query.lhs=1.0, query.rhs=2.0
```

## 6. Client Context and Config

`pvxs::client::Context` is the main client entry point. It manages connections, caching, and search.

```cpp
using namespace pvxs::client;

// From environment variables (most common)
auto ctxt = Context::fromEnv();

// From explicit Config
Config conf;
conf.addressList = {"192.168.1.255"};
conf.nameServers = {"10.0.0.1:5075"};
conf.udp_port = 5076;
conf.tcp_port = 5075;
conf.autoAddrList = false;
auto ctxt = conf.build();

// From environment then override
auto conf = Config::fromEnv();
conf.addressList.push_back("10.0.0.5");
auto ctxt = conf.build();
```

### Client Environment Variables

| Variable | Default | Description |
|---|---|---|
| `EPICS_PVA_ADDR_LIST` | (none) | Space-separated search destinations |
| `EPICS_PVA_AUTO_ADDR_LIST` | `YES` | Append local broadcast addresses |
| `EPICS_PVA_NAME_SERVERS` | (none) | TCP name server addresses |
| `EPICS_PVA_BROADCAST_PORT` | `5076` | Default UDP search port |
| `EPICS_PVA_CONN_TMO` | `30` | TCP inactivity timeout (seconds, internally multiplied by 4/3) |

## 7. Get Operation

```cpp
// Synchronous (simplest)
Value result = ctxt.get("my:pv").exec()->wait(5.0);
std::cout << result;
double val = result["value"].as<double>();

// With field selection (pvRequest)
Value result = ctxt.get("my:pv")
    .field("value")
    .exec()->wait(5.0);

// Asynchronous with callback
auto op = ctxt.get("my:pv")
    .result([](Result&& r) {
        try {
            Value val = r();  // throws RemoteError, Disconnect
            std::cout << val << "\n";
        } catch(std::exception& e) {
            std::cerr << "Error: " << e.what() << "\n";
        }
    })
    .exec();
// op must be kept alive; dropping it cancels the operation
```

### Info (type introspection without data)

```cpp
Value typeInfo = ctxt.info("my:pv").exec()->wait(5.0);
std::cout << typeInfo.format().showValue(false);
```

## 8. Put Operation

### Simple Put with `.set()`

```cpp
ctxt.put("my:pv")
    .set("value", 42.0)
    .exec()->wait(5.0);

// Multiple fields
ctxt.put("my:pv")
    .set("value", 42.0)
    .set("alarm.severity", 0)
    .exec()->wait(5.0);
```

### General Put with `.build()` callback

The build callback receives the current server value (or an empty prototype) and must return the Value to send.

```cpp
ctxt.put("my:pv")
    .build([](Value&& current) -> Value {
        auto val = current.cloneEmpty();
        val["value"] = current["value"].as<double>() + 1.0;
        return val;
    })
    .exec()->wait(5.0);

// Skip fetching current value (blind put)
ctxt.put("my:pv")
    .fetchPresent(false)
    .build([](Value&& proto) -> Value {
        proto["value"] = 99.0;
        return proto;
    })
    .exec()->wait(5.0);
```

### Put with completion callback

```cpp
auto op = ctxt.put("my:pv")
    .set("value", 42.0)
    .result([](Result&& r) {
        try {
            r();  // Value is always empty on success; throws on error
        } catch(std::exception& e) {
            std::cerr << "Put failed: " << e.what() << "\n";
        }
    })
    .exec();
```

### pvRequest options

```cpp
ctxt.put("my:pv")
    .record("process", true)    // request server to process after put
    .record("block", true)      // wait for processing to complete
    .set("value", 42.0)
    .exec()->wait(5.0);
```

## 9. Monitor (Subscribe)

### Basic pattern with MPMCFIFO

```cpp
MPMCFIFO<std::shared_ptr<Subscription>> workqueue(42u);

auto sub = ctxt.monitor("my:pv")
    .event([&workqueue](Subscription& sub) {
        // Called when queue transitions empty -> non-empty.
        // Must not block. Just enqueue work.
        workqueue.push(sub.shared_from_this());
    })
    .exec();

// Worker loop (typically in main thread or dedicated worker)
while(auto work = workqueue.pop()) {
    try {
        while(auto update = work->pop()) {
            std::cout << update << "\n";
        }
    } catch(client::Connected& e) {
        std::cout << "Connected to " << e.peerName << "\n";
    } catch(client::Disconnect& e) {
        std::cout << "Disconnected\n";
    } catch(client::Finished&) {
        std::cout << "Subscription complete\n";
        break;
    } catch(client::RemoteError& e) {
        std::cerr << "Server error: " << e.what() << "\n";
    }
}
```

### Stopping the monitor

```cpp
// Cancel from any thread
sub->cancel();

// Or simply drop the shared_ptr
sub.reset();
```

### Monitor options

```cpp
auto sub = ctxt.monitor("my:pv")
    .record("queueSize", 16)        // server-side queue depth
    .record("pipeline", true)       // per-subscription flow control
    .maskConnected(false)           // include Connected exceptions in pop() (default: masked)
    .maskDisconnected(false)        // include Disconnect exceptions in pop() (default: not masked)
    .event(...)
    .exec();
```

### Pause/Resume

```cpp
sub->pause();     // ask server to stop sending
sub->resume();    // ask server to resume
```

## 10. RPC

```cpp
// With NTURI-style arguments
auto result = ctxt.rpc("my:rpc:pv")
    .arg("lhs", 3.0)
    .arg("rhs", 4.0)
    .exec()->wait(5.0);
std::cout << result;

// With explicit Value argument
Value arg = nt::NTScalar{TypeCode::String}.create();
arg["value"] = "hello";
auto result = ctxt.rpc("my:rpc:pv", arg).exec()->wait(5.0);

// Asynchronous
auto op = ctxt.rpc("my:rpc:pv")
    .arg("cmd", "status")
    .result([](Result&& r) {
        try {
            std::cout << r() << "\n";
        } catch(std::exception& e) {
            std::cerr << e.what() << "\n";
        }
    })
    .exec();
```

## 11. Connect (Channel Status)

Track connection state without performing an operation.

```cpp
auto conn = ctxt.connect("my:pv")
    .onConnect([]() {
        std::cout << "Connected\n";
    })
    .onDisconnect([]() {
        std::cout << "Disconnected\n";
    })
    .exec();

// Poll connection status
bool isUp = conn->connected();
```

## 12. Logging

```cpp
#include <pvxs/log.h>

// Define a logger (file scope)
DEFINE_LOGGER(mylog, "myapp.component");

// Use it
log_info_printf(mylog, "Starting operation on %s\n", pvname.c_str());
log_err_printf(mylog, "Failed with code %d\n", code);
log_debug_printf(mylog, "Detail: val=%f\n", val);

// Configure from environment at startup
pvxs::logger_config_env();  // reads $PVXS_LOG

// Programmatic configuration
pvxs::logger_level_set("myapp.*", pvxs::Level::Debug);
```

| Level | Macro | Value |
|---|---|---|
| `Crit` | `log_crit_printf` | 10 |
| `Err` | `log_err_printf` | 20 |
| `Warn` | `log_warn_printf` | 30 |
| `Info` | `log_info_printf` | 40 |
| `Debug` | `log_debug_printf` | 50 |

Environment variable `PVXS_LOG`: `"key=VAL,..."`. Keys may use `*` wildcard. Values: `CRIT`, `ERR`, `WARN`, `INFO`, `DEBUG`. Example: `PVXS_LOG="pvxs.*=DEBUG,myapp.*=INFO"`.

## 13. Utilities

### SigInt

Portable SIGINT/SIGTERM handler. Handler runs in a thread context (safe for locks).

```cpp
#include <pvxs/util.h>

epicsEvent done;
SigInt handle([&done]() { done.signal(); });
// ... do work ...
done.wait();  // blocks until SIGINT or done.signal()
```

### MPMCFIFO

Thread-safe bounded multi-producer, multi-consumer FIFO queue.

```cpp
MPMCFIFO<int> queue(100);  // capacity 100; 0 = unbounded
queue.push(42);
int val = queue.pop();     // blocks if empty
```

### escape

```cpp
std::cout << pvxs::escape(someString);  // print with non-printable chars escaped
```

## 14. CLI Tools Reference

| Tool | Purpose | Example |
|---|---|---|
| `pvxget` | PVA Get | `pvxget my:pv` |
| `pvxput` | PVA Put | `pvxput my:pv 42` |
| `pvxmonitor` | PVA Monitor | `pvxmonitor my:pv` |
| `pvxinfo` | Type introspection | `pvxinfo my:pv` |
| `pvxlist` | List PVs from servers | `pvxlist` |
| `pvxcall` | PVA RPC call | `pvxcall my:rpc lhs=1 rhs=2` |

## 15. Complete Client Example

```cpp
#include <iostream>
#include <pvxs/client.h>
#include <pvxs/log.h>
#include <pvxs/util.h>

using namespace pvxs;

DEFINE_LOGGER(log, "myapp");

int main(int argc, char* argv[])
{
    logger_config_env();

    if(argc < 2) {
        std::cerr << "Usage: " << argv[0] << " <pvname>\n";
        return 1;
    }
    const std::string pvname(argv[1]);

    auto ctxt = client::Context::fromEnv();

    // GET
    try {
        Value result = ctxt.get(pvname).exec()->wait(5.0);
        std::cout << "GET result:\n" << result << "\n";
    } catch(std::exception& e) {
        std::cerr << "GET error: " << e.what() << "\n";
    }

    // PUT
    try {
        ctxt.put(pvname)
            .set("value", 0.0)
            .exec()->wait(5.0);
        std::cout << "PUT complete\n";
    } catch(std::exception& e) {
        std::cerr << "PUT error: " << e.what() << "\n";
    }

    // MONITOR
    epicsEvent done;
    SigInt handle([&done]() { done.signal(); });

    MPMCFIFO<std::shared_ptr<client::Subscription>> workqueue(42u);

    auto sub = ctxt.monitor(pvname)
        .event([&workqueue](client::Subscription& sub) {
            workqueue.push(sub.shared_from_this());
        })
        .exec();

    bool running = true;
    std::thread worker([&]() {
        while(running) {
            auto s = workqueue.pop();
            if(!s) break;
            try {
                while(auto update = s->pop()) {
                    std::cout << update << "\n";
                }
            } catch(client::Disconnect&) {
                std::cout << "Disconnected\n";
            } catch(client::Finished&) {
                done.signal();
                return;
            } catch(std::exception& e) {
                std::cerr << e.what() << "\n";
            }
        }
    });

    done.wait();
    running = false;
    sub->cancel();
    workqueue.push(nullptr);
    worker.join();

    return 0;
}
```

## 16. Key Rules and Pitfalls

1. **Operation lifetime**: `exec()` returns a `shared_ptr<Operation>` or `shared_ptr<Subscription>`. Dropping the pointer implicitly cancels the operation. Store it in a variable that outlives the operation.

2. **Value validity**: Always check `Value::valid()` or `operator bool()` before accessing fields. `operator[]` returns an invalid Value (not an exception) when a field is not found. Use `.lookup()` if you want an exception.

3. **`shared_array` freeze before assign**: You must call `.freeze()` to convert `shared_array<T>` to `shared_array<const T>` before assigning to a Value field. Assigning a mutable array will not compile.

4. **Monitor event callback must not block**: The `event()` callback is invoked from an internal worker thread. It must return quickly. Use `MPMCFIFO` or similar to hand off work.

5. **Monitor pop() loop**: After the event callback fires, call `pop()` in a loop until it returns an invalid Value (empty queue). The event callback fires again only when the queue transitions from empty to non-empty.

6. **Monitor exception types**: `pop()` may throw `Connected`, `Disconnect`, `Finished`, or `RemoteError` instead of returning a Value. Always wrap `pop()` in try/catch. `Finished` means the subscription ended normally. `Disconnect` means connection lost.

7. **Put `.build()` vs `.set()`**: Use `.set()` for simple field assignments. Use `.build()` when you need to read the current value, do computation, or conditionally set fields. The build callback receives the current value (or empty prototype if `fetchPresent(false)`).

8. **Put result Value is empty**: A successful Put returns an empty Value from `Result::operator()()`. Only Get and RPC return data.

9. **Call `logger_config_env()` early**: Call `pvxs::logger_config_env()` near the start of `main()` before creating any Context. This reads `$PVXS_LOG` for debug logging configuration.

10. **`wait()` throws on timeout**: `Operation::wait(double)` throws `client::Timeout` if the operation does not complete within the specified duration. `Operation::wait()` (no argument) waits ~indefinitely.

11. **Context is thread-safe**: A single `Context` may be shared across threads. All builder methods and `exec()` are thread-safe. Create one Context and reuse it.

12. **pvRequest string syntax**: `.pvRequest("field(value,alarm)")` selects which fields to transfer. `.field("value")` is a shorthand for adding a single field. `.record("process", true)` sets record-level options. These are composable on any builder.
