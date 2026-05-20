---
name: pvxs-server
description: Write PVXS PV Access server programs in C++ -- Server, SharedPV, StaticSource, Source interface, ChannelControl, ConnectOp/ExecOp/MonitorControlOp callbacks, and Value construction
---

You are an expert at writing EPICS PV Access server programs using the PVXS library. You know the high-level SharedPV API, the low-level Source interface, callback signatures, threading guarantees, and Value construction patterns.

## 1. Headers and Linking

```cpp
#include <pvxs/server.h>      // Server, Config
#include <pvxs/sharedpv.h>    // SharedPV, StaticSource
#include <pvxs/source.h>      // Source, ChannelControl, ConnectOp, ExecOp, MonitorSetupOp, MonitorControlOp
#include <pvxs/srvcommon.h>   // ClientCredentials, OpBase, ExecOp base
#include <pvxs/data.h>        // Value, TypeDef, TypeCode, Member
#include <pvxs/sharedArray.h> // shared_array<T>
#include <pvxs/nt.h>          // NTScalar, NTEnum, NTTable, NTNDArray, NTURI
#include <pvxs/log.h>         // DEFINE_LOGGER, log_*_printf, logger_config_env
#include <pvxs/util.h>        // SigInt, MPMCFIFO
```

**Makefile** (EPICS build system):

```makefile
PROD_LIBS += pvxs
PROD_LIBS += Com
```

Namespaces: `pvxs`, `pvxs::server`, `pvxs::nt`, `pvxs::members`.

## 2. Value and TypeDef -- The Data Container

`pvxs::Value` replaces the entire pvDataCPP `PVField` hierarchy. A `Value` references a node in a type/data tree.

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
        String("message"),
    }),
    Float64A("waveform"),
});
Value val = def.create();

// From Normative Type
Value val = nt::NTScalar{TypeCode::Float64}.create();
```

### Member Helpers

Scalar: `Bool`, `Int8`..`Int64`, `UInt8`..`UInt64`, `Float32`, `Float64`, `String`, `Any`
Array: `BoolA`, `Int8A`..`Int64A`, `UInt8A`..`UInt64A`, `Float32A`, `Float64A`, `StringA`, `AnyA`
Compound: `Struct`, `Union`, `StructA`, `UnionA` (each with optional type ID and children)

### Field Access

```cpp
Value v = top["value"];                 // child field
Value sev = top["alarm.severity"];      // dotted path
Value elem = top["dimension[0].size"]; // array element
Value sel = top["->booleanValue"];     // union selector
```

### Reading and Writing

```cpp
// Read
double d = top["value"].as<double>();          // throws NoField/NoConvert
top["value"].as<double>([](double d) { ... }); // lambda form

// Write (also marks the field as changed)
top["value"] = 42.0;
top["value"].from(42.0);
top.update("value", 42.0).update("alarm.severity", 0);

// Arrays
shared_array<double> arr({1.0, 2.0, 3.0});
top["value"] = arr.freeze();  // must freeze mutable -> const

auto data = top["value"].as<shared_array<const double>>();
```

### Cloning

```cpp
Value copy = val.clone();         // deep copy with values
Value empty = val.cloneEmpty();   // same type, default values, nothing marked
```

### Marking (Change Tracking)

```cpp
val["value"].mark();                    // mark as changed
val["value"].isMarked();                // check
val["value"].unmark();                  // clear
// from() and operator= automatically mark
```

### shared_array

```cpp
shared_array<double> arr({1.0, 2.0, 3.0});
shared_array<const double> frozen = arr.freeze();  // mutable -> const; clears arr
shared_array<double> thawed = frozen.thaw();        // const -> mutable; copies if shared
```

### Normative Types

```cpp
// NTScalar (or NTScalarArray with array TypeCode)
Value val = nt::NTScalar{TypeCode::Float64}.create();
Value val = nt::NTScalar{TypeCode::Float64, true, true, true}.create();
//                       type,              display, control, valueAlarm

// NTEnum
Value val = nt::NTEnum{}.create();

// NTTable
nt::NTTable tbl;
tbl.add_column(TypeCode::StringA, "name", "Name");
tbl.add_column(TypeCode::Float64A, "pos", "Position");
Value val = tbl.create();

// NTNDArray
Value val = nt::NTNDArray{}.create();
```

## 3. Server and Config

`pvxs::server::Server` is the PVA server instance.

```cpp
using namespace pvxs::server;

// From environment (most common)
Server serv = Server::fromEnv();

// From explicit Config
Config conf;
conf.interfaces = {"0.0.0.0"};
conf.tcp_port = 5075;
conf.udp_port = 5076;
conf.beaconDestinations = {"192.168.1.255"};
conf.auto_beacon = true;
Server serv(conf);

// Isolated (loopback, random ports -- for tests)
auto conf = Config::isolated();
Server serv(conf);
```

### Lifecycle

```cpp
serv.start();       // begin serving (non-blocking)
serv.stop();        // stop serving

serv.run();         // start() + block until interrupt()/SIGINT/SIGTERM
serv.interrupt();   // request run() to return
```

### Server Environment Variables

| Variable | Fallback | Default | Description |
|---|---|---|---|
| `EPICS_PVAS_INTF_ADDR_LIST` | -- | `0.0.0.0` | Interfaces to bind |
| `EPICS_PVAS_BEACON_ADDR_LIST` | `EPICS_PVA_ADDR_LIST` | (auto) | Beacon destinations |
| `EPICS_PVAS_AUTO_BEACON_ADDR_LIST` | `EPICS_PVA_AUTO_ADDR_LIST` | `YES` | Auto-add broadcast addrs |
| `EPICS_PVAS_SERVER_PORT` | `EPICS_PVA_SERVER_PORT` | `5075` | TCP port |
| `EPICS_PVAS_BROADCAST_PORT` | `EPICS_PVA_BROADCAST_PORT` | `5076` | UDP port |
| `EPICS_PVAS_IGNORE_ADDR_LIST` | -- | (none) | Ignore requests from these addrs |
| `EPICS_PVA_CONN_TMO` | -- | `30` | TCP timeout (multiplied by 4/3) |

## 4. SharedPV -- High-Level Server API

`SharedPV` is a single data value accessible by multiple clients. Use this for predetermined PV names with simple get/put/monitor semantics.

### Factory Methods

```cpp
// Mailbox: clients can put; default handler calls post() with client value
SharedPV pv = SharedPV::buildMailbox();

// Read-only: put requests are rejected
SharedPV pv = SharedPV::buildReadonly();
```

### Lifecycle

```cpp
// 1. Create initial value and open
Value initial = nt::NTScalar{TypeCode::Float64}.create();
initial["value"] = 0.0;
pv.open(initial);    // sets type + initial value; must not be already open

// 2. Update value (notifies all subscribers)
auto update = initial.cloneEmpty();
update["value"] = 42.0;
pv.post(update);     // only marked fields are sent to subscribers

// 3. Query current value
Value current = pv.fetch();

// 4. Close (force-disconnects clients, discards data)
pv.close();

// 5. Reopen (potentially with different type)
pv.open(newInitial);
```

### Adding to Server

```cpp
// Via built-in StaticSource (simplest)
serv.addPV("my:pv:name", pv);
serv.removePV("my:pv:name");

// Via explicit StaticSource
StaticSource src = StaticSource::build();
src.add("my:pv:one", pv1);
src.add("my:pv:two", pv2);
serv.addSource("mySource", src.source(), 0);  // priority 0
```

## 5. SharedPV Callbacks

### onPut -- Custom Put Handler

```cpp
pv.onPut([](SharedPV& pv,
             std::unique_ptr<server::ExecOp>&& op,
             Value&& top)
{
    // top contains client-provided values (marked fields only)
    double val = top["value"].as<double>();

    // Validate / clamp
    if(val < 0.0) top["value"] = 0.0;
    if(val > 100.0) top["value"] = 100.0;

    // Add timestamp if client didn't provide one
    if(!top["timeStamp"].isMarked(true, true)) {
        epicsTimeStamp now;
        if(!epicsTimeGetCurrent(&now)) {
            top["timeStamp.secondsPastEpoch"] = now.secPastEpoch + POSIX_TIME_AT_EPICS_EPOCH;
            top["timeStamp.nanoseconds"] = now.nsec;
        }
    }

    pv.post(top);    // update cache + notify subscribers
    op->reply();     // tell client PUT succeeded
});
```

### onRPC -- RPC Handler

```cpp
pv.onRPC([](SharedPV& pv,
             std::unique_ptr<server::ExecOp>&& op,
             Value&& arg)
{
    auto lhs = arg["query.lhs"].as<double>();
    auto rhs = arg["query.rhs"].as<double>();

    auto reply = nt::NTScalar{TypeCode::Float64}.create();
    reply["value"] = lhs + rhs;
    op->reply(reply);
    // Or on error: op->error("something went wrong");
});
```

### onFirstConnect / onLastDisconnect

```cpp
pv.onFirstConnect([](SharedPV& pv) {
    // First client connected. Could open() here for lazy init.
});

pv.onLastDisconnect([](SharedPV& pv) {
    // Last client disconnected.
});
```

### Callback Signature Reference

| Method | Signature |
|---|---|
| `onPut` | `void(SharedPV&, std::unique_ptr<ExecOp>&&, Value&&)` |
| `onRPC` | `void(SharedPV&, std::unique_ptr<ExecOp>&&, Value&&)` |
| `onFirstConnect` | `void(SharedPV&)` |
| `onLastDisconnect` | `void(SharedPV&)` |

## 6. StaticSource

Maps PV names to SharedPV instances. A SharedPV can appear under multiple names.

```cpp
StaticSource src = StaticSource::build();

SharedPV pv1 = SharedPV::buildMailbox();
SharedPV pv2 = SharedPV::buildReadonly();
// ... open pv1, pv2 ...

src.add("pv:one", pv1);
src.add("pv:two", pv2);
src.add("pv:alias", pv1);  // same PV, different name

serv.addSource("myPVs", src.source());

// Later
src.remove("pv:alias");
src.close();  // close() all PVs
```

## 7. Periodic Updates (Ticker Pattern)

```cpp
SharedPV pv = SharedPV::buildReadonly();
Value initial = nt::NTScalar{TypeCode::UInt32}.create();
initial["value"] = 0u;
pv.open(initial);

Server serv = Server::fromEnv();
serv.addPV("my:counter", pv);
serv.start();  // non-blocking

epicsEvent done;
SigInt handle([&done]() { done.signal(); });

uint32_t count = 0;
while(!done.wait(1.0)) {
    auto update = initial.cloneEmpty();
    update["value"] = count++;
    epicsTimeStamp now;
    if(!epicsTimeGetCurrent(&now)) {
        update["timeStamp.secondsPastEpoch"] = now.secPastEpoch + POSIX_TIME_AT_EPICS_EPOCH;
        update["timeStamp.nanoseconds"] = now.nsec;
    }
    pv.post(update);
}

serv.stop();
```

## 8. Source Interface -- Low-Level Server API

Subclass `server::Source` for dynamic PV names (gateways, proxies, computed PVs).

```cpp
struct MySource : public server::Source {
    void onSearch(Search& op) override
    {
        for(auto& name : op) {
            if(shouldClaim(name.name()))
                name.claim();
        }
    }

    void onCreate(std::unique_ptr<server::ChannelControl>&& chan) override
    {
        const std::string& pvname = chan->name();

        chan->onOp([pvname](std::unique_ptr<server::ConnectOp>&& op) {
            // GET or PUT setup
            Value prototype = buildType(pvname);
            op->connect(prototype);

            op->onGet([pvname](std::unique_ptr<server::ExecOp>&& op) {
                Value val = fetchData(pvname);
                op->reply(val);
            });

            op->onPut([pvname](std::unique_ptr<server::ExecOp>&& op, Value&& val) {
                storeData(pvname, std::move(val));
                op->reply();
            });
        });

        chan->onRPC([pvname](std::unique_ptr<server::ExecOp>&& op, Value&& arg) {
            auto reply = processRPC(pvname, std::move(arg));
            op->reply(reply);
        });

        chan->onSubscribe([pvname](std::unique_ptr<server::MonitorSetupOp>&& op) {
            Value prototype = buildType(pvname);
            auto ctrl = op->connect(prototype);

            ctrl->onStart([ctrl_raw = ctrl.get()](bool start) {
                if(start) {
                    // client resumed; post current value
                }
            });

            // Store ctrl somewhere to call ctrl->post(val) later
        });

        chan->onClose([pvname](const std::string& msg) {
            // cleanup
        });
    }

    List onList() override
    {
        auto names = std::make_shared<std::set<std::string>>();
        names->insert("dynamic:pv:1");
        names->insert("dynamic:pv:2");
        return List(names, true);  // dynamic=true means list may change
    }
};

// Register
auto src = std::make_shared<MySource>();
serv.addSource("myDynamic", src, 1);  // priority 1
```

## 9. ConnectOp and ExecOp

### ConnectOp -- Operation Setup

Called when a client initiates a GET or PUT channel operation.

```cpp
chan->onOp([](std::unique_ptr<server::ConnectOp>&& op) {
    // Build the prototype Value (defines the type for this operation)
    Value proto = nt::NTScalar{TypeCode::Float64}.create();
    op->connect(proto);  // send type to client; throws if pvRequest selects no fields

    // Or reject
    // op->error("not available");

    op->onGet([](std::unique_ptr<server::ExecOp>&& op) {
        // Client requests data
        Value val = ...;
        op->reply(val);      // send data
        // op->error("msg"); // or reject
    });

    op->onPut([](std::unique_ptr<server::ExecOp>&& op, Value&& val) {
        // Client sends data
        // val contains marked fields from client
        op->reply();          // acknowledge (no data for PUT)
        // op->error("msg");  // or reject
    });

    op->onClose([](const std::string& msg) {
        // client closed the operation
    });
});
```

### ExecOp Methods

| Method | Purpose |
|---|---|
| `reply()` | Complete without data (PUT acknowledgment) |
| `reply(const Value&)` | Complete with data (GET, RPC) |
| `error(const std::string&)` | Report error to client |
| `onCancel(fn)` | Callback if client cancels before reply |
| `name()` | Channel name |
| `credentials()` | Client authentication info |
| `pvRequest()` | The pvRequest blob |

## 10. MonitorSetupOp and MonitorControlOp

### Setting Up a Monitor

```cpp
chan->onSubscribe([](std::unique_ptr<server::MonitorSetupOp>&& op) {
    Value proto = nt::NTScalar{TypeCode::Float64}.create();
    std::unique_ptr<server::MonitorControlOp> ctrl = op->connect(proto);

    ctrl->onStart([](bool start) {
        // start=true: client resumed; start=false: client paused
    });

    // Store ctrl to post updates later
    // ctrl->post(update);
});
```

### Posting Updates

| Method | Behavior |
|---|---|
| `post(val)` | Enqueue or squash into last element if full |
| `tryPost(val)` | Enqueue only if space; no-op if full |
| `forcePost(val)` | Always enqueue (may overfill) |
| `finish()` | Signal end of subscription (not an error) |

```cpp
auto update = proto.cloneEmpty();
update["value"] = 42.0;
ctrl->post(update);  // only marked fields are transferred
```

### Flow Control

```cpp
ctrl->setWatermarks(2, 6);  // low=2, high=6 (of negotiated queue size)

ctrl->onHighMark([]() {
    // queue filled above high watermark; consider throttling
});
```

## 11. ClientCredentials

Available on all server-side operation objects.

```cpp
chan->onOp([](std::unique_ptr<server::ConnectOp>&& op) {
    auto& cred = *op->credentials();
    std::cout << "Peer: " << cred.peer << "\n";
    std::cout << "Method: " << cred.method << "\n";   // "ca" or "anonymous"
    std::cout << "Account: " << cred.account << "\n";

    auto roles = cred.roles();  // OS groups of the remote user (local lookup)
    if(roles.count("operators")) {
        // authorized
    }
});
```

## 12. Threading Model

- Server callbacks are invoked from internal worker threads.
- **Guarantee**: callbacks for a given PV/channel are never invoked concurrently.
- This extends to all callbacks stored through a `ChannelControl` and its related `*Op` objects.
- It is safe to call `op->reply()` from within the callback or defer it to another thread (by capturing the `unique_ptr<ExecOp>` into a `shared_ptr` or moving it).
- `SharedPV::post()` is thread-safe and can be called from any thread.

## 13. Complete Mailbox Example

```cpp
#include <iostream>
#include <pvxs/server.h>
#include <pvxs/sharedpv.h>
#include <pvxs/nt.h>
#include <pvxs/log.h>
#include <pvxs/util.h>

using namespace pvxs;

int main(int argc, char* argv[])
{
    logger_config_env();

    if(argc < 2) {
        std::cerr << "Usage: " << argv[0] << " <pvname> [pvname ...]\n";
        return 1;
    }

    server::Server serv = server::Server::fromEnv();

    std::vector<server::SharedPV> pvs;
    for(int i = 1; i < argc; i++) {
        auto pv = server::SharedPV::buildMailbox();

        pv.onPut([](server::SharedPV& pv,
                     std::unique_ptr<server::ExecOp>&& op,
                     Value&& top)
        {
            double val = top["value"].as<double>();
            if(val < -100.0) top["value"] = -100.0;
            if(val > 100.0) top["value"] = 100.0;
            pv.post(top);
            op->reply();
        });

        Value initial = nt::NTScalar{TypeCode::Float64}.create();
        initial["value"] = 0.0;
        pv.open(initial);

        serv.addPV(argv[i], pv);
        pvs.push_back(std::move(pv));
    }

    std::cout << "Serving:\n" << serv;
    serv.run();  // blocks until SIGINT

    return 0;
}
```

## 14. Complete Low-Level Source Example

```cpp
#include <iostream>
#include <map>
#include <mutex>
#include <pvxs/server.h>
#include <pvxs/source.h>
#include <pvxs/nt.h>
#include <pvxs/log.h>
#include <pvxs/util.h>

using namespace pvxs;

struct DynamicSource : public server::Source {
    const std::string prefix;
    mutable std::mutex lock;
    std::map<std::string, Value> data;

    DynamicSource(const std::string& prefix) : prefix(prefix) {}

    void onSearch(Search& op) override
    {
        for(auto& name : op) {
            if(std::string(name.name()).substr(0, prefix.size()) == prefix)
                name.claim();
        }
    }

    void onCreate(std::unique_ptr<server::ChannelControl>&& chan) override
    {
        auto self = std::dynamic_pointer_cast<DynamicSource>(
            chan->name().empty() ? nullptr : shared_from_this());
        // Note: Source must be stored as shared_ptr for shared_from_this()

        const std::string pvname = chan->name();

        chan->onOp([this, pvname](std::unique_ptr<server::ConnectOp>&& op) {
            Value proto = nt::NTScalar{TypeCode::Float64}.create();
            op->connect(proto);

            op->onGet([this, pvname](std::unique_ptr<server::ExecOp>&& op) {
                std::lock_guard<std::mutex> G(lock);
                auto it = data.find(pvname);
                if(it != data.end()) {
                    op->reply(it->second);
                } else {
                    Value val = nt::NTScalar{TypeCode::Float64}.create();
                    val["value"] = 0.0;
                    data[pvname] = val;
                    op->reply(val);
                }
            });

            op->onPut([this, pvname](std::unique_ptr<server::ExecOp>&& op, Value&& val) {
                std::lock_guard<std::mutex> G(lock);
                data[pvname].assign(val);
                op->reply();
            });
        });
    }

    List onList() override
    {
        return List({}, true);  // dynamic list
    }
};

int main()
{
    logger_config_env();

    auto src = std::make_shared<DynamicSource>("dyn:");
    auto serv = server::Server::fromEnv();
    serv.addSource("dynamic", src);

    std::cout << serv;
    serv.run();
    return 0;
}
```

## 15. Key Rules and Pitfalls

1. **Call `open()` before `post()`**: `SharedPV::post()` has no effect if the PV is not open. Always call `open(initial)` first.

2. **`cloneEmpty()` for updates**: When posting updates, use `initial.cloneEmpty()` to create an update Value of the same type. Only set and mark the fields that changed. `post()` transmits only marked fields to subscribers.

3. **Do not modify a Value after `post()`**: The Value passed to `post()` may be read concurrently by the server internals. Clone before modifying again.

4. **`ExecOp::reply()` is required**: Every `ExecOp` must eventually get exactly one `reply()` or `error()` call. Failing to reply leaves the client hanging. If the ExecOp is destroyed without reply, the client receives a disconnect.

5. **`SharedPV::buildMailbox()` auto-posts**: The default mailbox onPut handler calls `post(top)` + `op->reply()`. If you set a custom `onPut`, you must call both yourself.

6. **Callback threading guarantee**: Callbacks for a given channel are serialized (never concurrent). But callbacks for different channels may run concurrently. Protect any shared state with a mutex.

7. **`connect()` before `onGet()`/`onPut()`**: In the low-level API, you must call `ConnectOp::connect(prototype)` before the `onGet`/`onPut` callbacks will be invoked. The prototype defines the type seen by the client.

8. **Source must be kept alive**: The `shared_ptr<Source>` passed to `addSource()` is held by the Server. Removing the source or stopping the server releases it. If your Source subclass uses `shared_from_this()`, it must inherit from `std::enable_shared_from_this`.

9. **`MonitorControlOp::post()` uses marks**: Like `SharedPV::post()`, only marked fields in the Value are transmitted. Use `cloneEmpty()` and set only changed fields.

10. **Server `run()` blocks on SIGINT**: `Server::run()` calls `start()` then blocks until `interrupt()` is called or a signal is received. Use `start()`/`stop()` for non-blocking operation (e.g., when posting periodic updates from main thread).

11. **`Config::isolated()` for tests**: Use `Config::isolated()` to create a loopback-only server on random ports. Pair with `serv.clientConfig().build()` to create a client that connects to the test server.

12. **Logging**: Call `pvxs::logger_config_env()` early in `main()`. Set `PVXS_LOG="pvxs.server.*=DEBUG"` for server-side debug output.
