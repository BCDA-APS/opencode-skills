---
name: epics-pva-client
description: Write EPICS PV Access (PVA) client and server programs in C++ -- pvac client API, pvData structures, normative types, SharedPV servers, and QSRV group configuration
---

# EPICS PV Access Client/Server Skill

You are an expert at writing EPICS PV Access (PVA) client and server code in C++. PV Access is the modern EPICS network protocol supporting structured data, atomic multi-field updates, and RPC services. You understand the pvac client API, pvData type system, normative types, server-side SharedPV patterns, and QSRV group configuration.

---

## 1. Module Overview

PV Access in EPICS 7 consists of these modules:

| Module | Namespace | Purpose |
|--------|-----------|---------|
| pvData | `epics::pvData` | Type system: `PVStructure`, `PVField`, `FieldBuilder` |
| pvAccess | `epics::pvAccess`, `pvac::`, `pvas::` | Network protocol, client API, server API |
| normativeTypes | `epics::nt` | Standard data types: `NTScalar`, `NTTable`, etc. |
| pvaClient | `epics::pvaClient` | High-level client wrapper |
| pvDatabase | `epics::pvDatabase` | Server-side PV records |
| pva2pva (QSRV) | -- | IOC records exposed via PVA |

### 1.1 Headers and Linking

```cpp
// pvac client API (recommended for most client code)
#include <pva/client.h>

// pvData (type system)
#include <pv/pvData.h>
#include <pv/pvIntrospect.h>

// Normative types
#include <pv/nt.h>         // master include for all NT types
#include <pv/ntscalar.h>   // individual NT type

// Server API
#include <pva/server.h>
#include <pva/sharedstate.h>
```

```makefile
# Client program
PROD_HOST += myPvaClient
myPvaClient_SRCS += myPvaClient.cpp
myPvaClient_LIBS += pvAccess pvData Com
# or use:
myPvaClient_LIBS += $(EPICS_BASE_PVA_CORE_LIBS) $(EPICS_BASE_HOST_LIBS)
```

### 1.2 Common Namespace Aliases

```cpp
namespace pvd = epics::pvData;
namespace pva = epics::pvAccess;
namespace nt  = epics::nt;
```

---

## 2. pvData Type System

### 2.1 Scalar Types

```cpp
enum ScalarType {
    pvBoolean, pvByte, pvShort, pvInt, pvLong,
    pvUByte, pvUShort, pvUInt, pvULong,
    pvFloat, pvDouble, pvString
};
```

### 2.2 Building Structure Types with FieldBuilder

```cpp
using namespace epics::pvData;

// Simple structure
StructureConstPtr type = getFieldCreate()->createFieldBuilder()
    ->add("value", pvDouble)
    ->add("alarm", getStandardField()->alarm())
    ->add("timeStamp", getStandardField()->timeStamp())
    ->createStructure();

// Nested structure
StructureConstPtr complexType = getFieldCreate()->createFieldBuilder()
    ->add("value", pvDouble)
    ->addNestedStructure("position")
        ->add("x", pvDouble)
        ->add("y", pvDouble)
        ->add("z", pvDouble)
    ->endNested()
    ->addArray("waveform", pvDouble)
    ->createStructure();
```

### 2.3 Creating and Accessing PVStructure

```cpp
PVStructurePtr pvs = getPVDataCreate()->createPVStructure(type);

// Set values
pvs->getSubFieldT<PVDouble>("value")->put(3.14);
pvs->getSubFieldT<PVString>("alarm.message")->put("High");
pvs->getSubFieldT<PVInt>("alarm.severity")->put(1);

// Get values (throwing version -- throws if field not found)
double val = pvs->getSubFieldT<PVDouble>("value")->get();

// Get values (non-throwing version -- returns nullptr if not found)
PVDoublePtr pv = pvs->getSubField<PVDouble>("value");
if (pv) {
    double val = pv->get();
}
```

### 2.4 Array Handling with shared_vector

```cpp
// Create and fill an array
shared_vector<double> data(1024);
for (size_t i = 0; i < data.size(); i++)
    data[i] = sin(i * 0.01);

// Zero-copy assignment to PVScalarArray
PVDoubleArrayPtr arr = pvs->getSubFieldT<PVDoubleArray>("waveform");
arr->replace(freeze(data));  // freeze converts mutable -> const (zero-copy)

// Zero-copy read from PVScalarArray
shared_vector<const double> view;
arr->getAs(view);  // zero-copy read access
for (size_t i = 0; i < view.size(); i++)
    printf("%f ", view[i]);
```

### 2.5 JSON Serialization

```cpp
#include <pv/json.h>

// PVStructure to JSON string
std::ostringstream oss;
printJSON(oss, *pvs);
std::string json = oss.str();

// JSON string to PVStructure
PVStructurePtr parsed = parseJSON(jsonString);
```

---

## 3. pvac Client API

The `pvac::` namespace provides the recommended client API (simpler than the low-level `pvAccess` interfaces).

### 3.1 Client Provider and Channel

```cpp
#include <pva/client.h>

// Create a client provider ("pva" for PV Access, "ca" for Channel Access)
pvac::ClientProvider provider("pva");

// Connect to a PV
pvac::ClientChannel channel(provider.connect("MY:PV:NAME"));
```

### 3.2 Simple Get

```cpp
// Blocking get with timeout
pvd::PVStructure::const_shared_pointer result = channel.get(5.0);
double value = result->getSubFieldT<pvd::PVDouble>("value")->get();

// Get with request (select specific fields)
pvd::PVStructure::const_shared_pointer result =
    channel.get(5.0, pvd::createRequest("field(value,alarm,timeStamp)"));
```

### 3.3 Simple Put

```cpp
// Fluent put API
channel.put()
    .set("value", 42.0)
    .exec(5.0);  // 5 second timeout

// Put with multiple fields
channel.put()
    .set("value", 3.14)
    .set("value.index", 0)
    .exec();
```

### 3.4 Synchronous Monitor

```cpp
pvac::MonitorSync mon(channel.monitor());

while (true) {
    if (!mon.wait(5.0)) {
        std::cerr << "Timeout waiting for update\n";
        continue;
    }

    switch (mon.event.event) {
    case pvac::MonitorEvent::Data:
        while (mon.poll()) {
            // mon.root is the current PVStructure
            // mon.changed is a BitSet of changed fields
            double val = mon.root->getSubFieldT<pvd::PVDouble>("value")->get();
            std::cout << "Value = " << val << "\n";
        }
        break;
    case pvac::MonitorEvent::Disconnect:
        std::cout << "Disconnected\n";
        break;
    case pvac::MonitorEvent::Fail:
        std::cerr << "Error: " << mon.event.message << "\n";
        break;
    case pvac::MonitorEvent::Cancel:
        return;
    }
}
```

### 3.5 Callback-Based Monitor

```cpp
struct MyMonitorCB : public pvac::ClientChannel::MonitorCallback {
    void monitorEvent(const pvac::MonitorEvent& evt) override {
        switch (evt.event) {
        case pvac::MonitorEvent::Data:
            // Poll all available updates
            while (auto update = evt.monitor->poll()) {
                std::cout << update->pvStructurePtr << "\n";
            }
            break;
        case pvac::MonitorEvent::Disconnect:
            std::cout << "Disconnected\n";
            break;
        case pvac::MonitorEvent::Fail:
            std::cerr << "Monitor error: " << evt.message << "\n";
            break;
        default:
            break;
        }
    }
};

MyMonitorCB cb;
pvac::Monitor mon = channel.monitor(&cb);
// mon stays active until destroyed
```

### 3.6 RPC Call

```cpp
// Build RPC arguments
pvd::PVStructurePtr args = pvd::getPVDataCreate()->createPVStructure(
    pvd::getFieldCreate()->createFieldBuilder()
        ->add("query", pvd::pvString)
        ->createStructure()
);
args->getSubFieldT<pvd::PVString>("query")->put("SELECT * FROM data");

// Execute RPC with 10 second timeout
pvd::PVStructure::const_shared_pointer result = channel.rpc(10.0, args);
```

### 3.7 Channel Info

```cpp
// Get channel type information
pvac::detail::SharedPut::Info info = channel.info();
// info.structure contains the server's type definition
```

---

## 4. Normative Types

Normative Types (NTs) are standardized PVStructure layouts for common data patterns.

### 4.1 NTScalar

```cpp
#include <pv/ntscalar.h>

// Build an NTScalar with alarm and timestamp
nt::NTScalarPtr ntScalar = nt::NTScalar::createBuilder()
    ->value(pvd::pvDouble)
    ->addAlarm()
    ->addTimeStamp()
    ->addDisplay()
    ->addControl()
    ->create();

// Access the wrapped PVStructure
pvd::PVStructurePtr pvs = ntScalar->getPVStructure();
ntScalar->getValue<pvd::PVDouble>()->put(42.0);
```

### 4.2 NTScalarArray

```cpp
#include <pv/ntscalarArray.h>

nt::NTScalarArrayPtr ntArr = nt::NTScalarArray::createBuilder()
    ->value(pvd::pvDouble)
    ->addAlarm()
    ->addTimeStamp()
    ->create();

// Set array data
pvd::shared_vector<double> data(100);
for (int i = 0; i < 100; i++) data[i] = i * 0.1;
ntArr->getValue()->replace(pvd::freeze(data));
```

### 4.3 NTTable

```cpp
#include <pv/nttable.h>

nt::NTTablePtr table = nt::NTTable::createBuilder()
    ->addColumn("name", pvd::pvString)
    ->addColumn("value", pvd::pvDouble)
    ->addColumn("severity", pvd::pvInt)
    ->create();

pvd::PVStructurePtr pvs = table->getPVStructure();
// Set column data via the value field's sub-arrays
```

### 4.4 NTEnum

```cpp
#include <pv/ntenum.h>

nt::NTEnumPtr ntEnum = nt::NTEnum::createBuilder()
    ->addAlarm()
    ->addTimeStamp()
    ->create();

// Set choices and index
pvd::shared_vector<std::string> choices(3);
choices[0] = "Off"; choices[1] = "On"; choices[2] = "Error";
ntEnum->getChoices()->replace(pvd::freeze(choices));
ntEnum->getIndex()->put(1);  // "On"
```

### 4.5 NTURI (for RPC Arguments)

```cpp
#include <pv/nturi.h>

nt::NTURIPtr uri = nt::NTURI::createBuilder()
    ->addQueryString("pv")
    ->addQueryDouble("timeout")
    ->create();

uri->getPath()->put("myService");
uri->getQueryT<pvd::PVString>("pv")->put("MY:PV");
uri->getQueryT<pvd::PVDouble>("timeout")->put(5.0);
```

### 4.6 NTNDArray (Area Detector Images)

```cpp
#include <pv/ntndarray.h>

nt::NTNDArrayPtr ndarray = nt::NTNDArray::createBuilder()
    ->addAlarm()
    ->addTimeStamp()
    ->create();
```

---

## 5. PVA Server with SharedPV

### 5.1 Mailbox Pattern (Clients Can Put)

```cpp
#include <pva/server.h>
#include <pva/sharedstate.h>

namespace pvd = epics::pvData;
namespace pva = epics::pvAccess;

// Define the data type
pvd::StructureConstPtr type = pvd::getFieldCreate()->createFieldBuilder()
    ->add("value", pvd::pvDouble)
    ->add("timeStamp", pvd::getStandardField()->timeStamp())
    ->createStructure();

// Create a mailbox PV (clients can put values)
pvas::SharedPV::shared_pointer pv(pvas::SharedPV::buildMailbox());
pv->open(type);  // Initialize with empty structure of given type

// Add to a provider
pvas::StaticProvider provider("myProvider");
provider.add("MY:PV:NAME", pv);

// Create and run the server
pva::ServerContext::shared_pointer server(
    pva::ServerContext::create(
        pva::ServerContext::Config()
            .provider(provider.provider())
    )
);

server->printInfo();
// Server runs until destroyed
```

### 5.2 Read-Only PV (Server Updates Only)

```cpp
// Create a read-only PV
pvas::SharedPV::shared_pointer pv(pvas::SharedPV::buildReadonly());
pv->open(type);

// Server-side update
pvd::BitSet changed;
{
    pvas::SharedPV::Guard G(*pv);
    pvd::PVStructurePtr current = pv->build();
    current->getSubFieldT<pvd::PVDouble>("value")->put(3.14);
    changed.set(current->getSubFieldT<pvd::PVDouble>("value")->getFieldOffset());
    pv->post(*current, changed);
}
```

### 5.3 Handler Pattern (Custom Put Processing)

```cpp
struct MyHandler : public pvas::SharedPV::Handler {
    void onPut(const pvas::SharedPV::shared_pointer& pv,
               std::unique_ptr<pvas::ExecOp>&& op,
               pvd::PVStructure& value,
               const pvd::BitSet& changed) override
    {
        // Validate or transform the put value
        double val = value.getSubFieldT<pvd::PVDouble>("value")->get();
        if (val < 0) {
            op->error("Value must be non-negative");
            return;
        }
        // Accept the put
        pv->post(value, changed);
        op->reply();
    }
};

auto handler = std::make_shared<MyHandler>();
pvas::SharedPV::shared_pointer pv(pvas::SharedPV::buildMailbox());
pv->setHandler(handler);
pv->open(type);
```

---

## 6. QSRV -- IOC Records via PVA

QSRV (part of pva2pva) automatically exposes IOC database records via PV Access. Each record is accessible by its record name as an NTScalar, NTEnum, or NTScalarArray.

### 6.1 Enabling QSRV in an IOC

In `src/Makefile`:
```makefile
ifdef EPICS_QSRV_MAJOR_VERSION
    myApp_LIBS += qsrv
    myApp_LIBS += $(EPICS_BASE_PVA_CORE_LIBS)
    myApp_DBD += PVAServerRegister.dbd
    myApp_DBD += qsrv.dbd
endif
```

QSRV starts automatically with `iocInit` when linked.

### 6.2 Group PVs with info(Q:group)

Group PVs combine fields from multiple records into a single PVA structure with atomic updates.

```
record(calc, "$(user):circle:angle") {
    field(CALC, "A+B")
    field(SCAN, "1 second")
    info(Q:group, {
        "$(user):circle":{
            "angle": {+channel:"VAL"}
        }
    })
}

record(calc, "$(user):circle:x") {
    field(CALC, "COS(D2R*A)")
    field(INPA, "$(user):circle:angle NPP NMS")
    info(Q:group, {
        "$(user):circle":{
            "x": {+channel:"VAL"}
        }
    })
}

record(calc, "$(user):circle:y") {
    field(CALC, "SIN(D2R*A)")
    field(INPA, "$(user):circle:angle NPP NMS")
    info(Q:group, {
        "$(user):circle":{
            "y": {+channel:"VAL", +trigger:"*"}
        }
    })
}
```

This creates a PVA PV named `$(user):circle` with fields `angle`, `x`, and `y`, updated atomically when `y` changes (due to `+trigger:"*"`).

### 6.3 Q:group JSON Syntax

```json
{
    "groupPVName": {
        "fieldName": {
            "+channel": "RECORD_FIELD",
            "+type": "plain",
            "+trigger": "*",
            "+putorder": 0
        }
    }
}
```

| Key | Description |
|-----|-------------|
| `+channel` | Record field to map (e.g., `"VAL"`, `"SEVR"`, `"TIME"`) |
| `+type` | `"plain"` (default), `"any"`, `"proc"`, `"structure"` |
| `+trigger` | Which group fields to update: `""` (none), `"*"` (all), `"fieldName"` (specific) |
| `+putorder` | Processing order for group puts (lower = first) |

### 6.4 Loading Group Definitions from JSON Files

```
# In st.cmd:
dbLoadGroup("db/myGroups.json")
```

### 6.5 PVA Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `EPICS_PVA_ADDR_LIST` | (empty) | PVA search addresses |
| `EPICS_PVA_AUTO_ADDR_LIST` | `YES` | Auto-detect broadcast |
| `EPICS_PVA_SERVER_PORT` | `5075` | PVA server TCP port |
| `EPICS_PVA_BROADCAST_PORT` | `5076` | PVA UDP search port |
| `EPICS_PVAS_SERVER_PORT` | (= SERVER_PORT) | Server-side port override |
| `EPICS_PVAS_INTF_ADDR_LIST` | `0.0.0.0` | Server bind interface |

---

## 7. Command-Line Tools

PVA ships with command-line tools analogous to CA's `caget`/`caput`/`camonitor`:

| Tool | Description |
|------|-------------|
| `pvget` | Get PV value(s): `pvget MY:PV:NAME` |
| `pvput` | Put PV value: `pvput MY:PV:NAME 42.0` |
| `pvmonitor` | Monitor PV updates: `pvmonitor MY:PV:NAME` |
| `pvinfo` | Show PV type info: `pvinfo MY:PV:NAME` |
| `pvlist` | List available PVs on a server: `pvlist` |
| `pvcall` | RPC call: `pvcall MY:SERVICE query="test"` |

These tools support both PVA (`-p pva`) and CA (`-p ca`) providers.

---

## 8. Complete Client Example

```cpp
#include <iostream>
#include <pva/client.h>
#include <pv/pvData.h>

namespace pvd = epics::pvData;

int main(int argc, char *argv[])
{
    if (argc < 2) {
        std::cerr << "Usage: " << argv[0] << " pvname [pvname ...]\n";
        return 1;
    }

    try {
        // Create client provider
        pvac::ClientProvider provider("pva");

        for (int i = 1; i < argc; i++) {
            // Connect and get
            pvac::ClientChannel channel(provider.connect(argv[i]));
            pvd::PVStructure::const_shared_pointer result = channel.get(5.0);

            // Print the structure
            std::cout << argv[i] << "\n" << *result << "\n\n";
        }
    } catch (std::exception& e) {
        std::cerr << "Error: " << e.what() << "\n";
        return 1;
    }

    return 0;
}
```

---

## 9. Key Rules and Pitfalls

1. **Use `pvac::` API for client code**, not the low-level `pvAccess` `ChannelProvider`/`Channel` interfaces. The `pvac::` API is simpler and handles lifecycle correctly.

2. **Use `getSubFieldT<T>()` (throwing) vs `getSubField<T>()` (non-throwing).** The throwing version is cleaner for required fields. The non-throwing version returns nullptr and is better for optional fields.

3. **`shared_vector` uses copy-on-write semantics.** Use `freeze()` to convert mutable to const (for passing to PVA), and `thaw()` to convert const back to mutable. Never modify a frozen vector.

4. **Monitor event loop must call `poll()` until it returns false.** Multiple updates may be queued. Failing to drain the queue causes data loss.

5. **PVA structures are immutable after creation.** You cannot add/remove fields from an existing `PVStructure`. Create a new type with `FieldBuilder` if you need a different structure.

6. **QSRV `+trigger:"*"` should be on exactly one field** in a group to avoid redundant updates. The triggered field's record processing causes all group fields to be collected and posted atomically.

7. **PVA supports 64-bit integers** (`pvLong`, `pvULong`), unlike Channel Access which is limited to 32-bit. Use PVA for data that exceeds 32-bit range.

8. **The "ca" provider** (`pvac::ClientProvider("ca")`) allows PVA client code to access CA servers. This is useful for gradual migration from CA to PVA.

9. **PVA array transfers have no size limit** (unlike CA's `EPICS_CA_MAX_ARRAY_BYTES`). Large arrays transfer efficiently via PVA.

10. **Server-side `pvas::SharedPV::post()` publishes updates** to all connected monitors. The `BitSet` argument indicates which fields changed, enabling efficient partial updates.
