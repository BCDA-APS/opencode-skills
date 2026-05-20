---
name: pvxs-ioc
description: Configure PVXS QSRV2 in EPICS IOCs -- single PV access, Q:group definitions, JSON group files, PVA links, dbLoadGroup, NTTable/NTEnum/NTNDArray groups, access security, and IOC shell commands
---

You are an expert at integrating PVXS (QSRV2) into EPICS IOCs. You know the single PV mapping, group PV JSON syntax, PVA link configuration, info tags, IOC shell commands, and testing patterns. QSRV2 automatically serves all IOC database records via PV Access.

## 1. Setup and Enabling

### DBD and Library Dependencies

In your IOC application `src/Makefile`:

```makefile
myApp_DBD += pvxsIoc.dbd
myApp_LIBS += pvxsIoc
myApp_LIBS += pvxs
```

`pvxsIoc.dbd` registers:
- `pvxsBaseRegistrar` -- main entry point, called by `*_registerRecordDeviceDriver(pdbbase)`
- `link("pva", "lsetPVX")` -- the `pva` JSON link type
- Demo device supports (`devWfPDBQ2Demo`, `devLoPDBQ2UTag`)

### Enabling / Disabling QSRV2

QSRV2 is enabled by default since pvxs 1.3.0. Control via environment variable **before** `*_registerRecordDeviceDriver()`:

```
# In st.cmd
epicsEnvSet("PVXS_QSRV_ENABLE", "YES")   # default
# or
epicsEnvSet("PVXS_QSRV_ENABLE", "NO")    # disable
```

QSRV2 is automatically disabled if QSRV1 is detected (presence of `devWfPDBDemo`).

### Minimal st.cmd

```
dbLoadDatabase("dbd/myApp.dbd")
myApp_registerRecordDeviceDriver(pdbbase)
dbLoadRecords("db/myRecords.db", "P=TST:")
# Optional: dbLoadGroup("db/groups.json", "P=TST:")
iocInit()
```

No additional commands are needed -- QSRV2 starts automatically at `iocInit()`.

## 2. Single PV Access

Every record in the IOC database is automatically available via PVA. `pvxget rec:name` works like `caget rec:name`.

### Structure Served

For numeric records (ai, ao, longin, longout, calc, etc.):

```
epics:nt/NTScalar:1.0
    <type> value
    alarm_t alarm
        int32_t severity
        int32_t status
        string message
    time_t timeStamp
        int64_t secondsPastEpoch
        int32_t nanoseconds
        int32_t userTag
    struct display
        double limitLow
        double limitHigh
        string description
        string units
        int32_t precision
        struct form
            int32_t index
            string[] choices
    struct control
        double limitLow
        double limitHigh
        double minStep
    struct valueAlarm
        ...
```

For mbbi/mbbo records: `epics:nt/NTEnum:1.0` with `value.index` and `value.choices`.

For waveform/aai/aao records: `epics:nt/NTScalarArray:1.0`.

### info Tags for Single PVs

#### `Q:form` -- Display Format Hint

```
record(ai, "$(P)temp") {
    field(EGU, "degC")
    info(Q:form, "Default")
}
```

Values: `Default`, `String`, `Binary`, `Decimal`, `Hex`, `Exponential`, `Engineering`.

`String` on a `CHAR` waveform hints clients to display the array as a string.

#### `Q:time:tag` -- UserTag from Timestamp

```
record(ai, "$(P)val") {
    info(Q:time:tag, "nsec:lsb:8")
}
```

Copies the lowest N bits of `nanoseconds` into the `userTag` field.

### Subscription Event Masks

Subscriptions split into two internal monitors: value/alarm (`DBE_VALUE|DBE_ALARM|DBE_ARCHIVE`) and property (`DBE_PROPERTY`). Property changes (EGU, HOPR, LOPR, etc.) are sent separately.

Client can override via pvRequest: `record[DBE=5]` (integer bitmask: `DBE_VALUE=1`, `DBE_ARCHIVE=2`, `DBE_ALARM=4`).

### Put Processing Options

| pvRequest | Behavior |
|---|---|
| `record[process=true]` | Force processing (like PP) |
| `record[process=false]` | No processing (like NPP) |
| `record[process=passive]` | Process if SCAN=Passive (default) |
| `record[block=true]` | Wait for processing to complete (Base >= 3.16) |

### Long Strings

Fields like `.NAME`, `.INP`, `.OUT`, `.DESC` are automatically served as long strings. Append `$` to `DBF_STRING`/`DBF_*LINK` fields to force string representation: `pvxget "rec:name.INP$"`.

## 3. Group PV Definitions -- info Tag Syntax

Group PVs combine fields from multiple records into a single PVA structure.

### Basic Syntax (in .db file)

```
record(ao, "$(P)X") {
    info(Q:group, {
        "$(P)position":{
            +id:"myapp:position:1.0",
            "x":{
                +type:"scalar",
                +channel:"VAL",
                +trigger:"*"
            }
        }
    })
}

record(ao, "$(P)Y") {
    info(Q:group, {
        "$(P)position":{
            "y":{
                +type:"scalar",
                +channel:"VAL",
                +trigger:"*"
            },
            "":{+type:"meta", +channel:"VAL"}
        }
    })
}
```

Multiple `info(Q:group, ...)` tags across different records are merged by group name.

### All `+` Keys

| Key | Required | Description |
|---|---|---|
| `+id` | No | Structure type ID string (e.g., `"epics:nt/NTTable:1.0"`) |
| `+type` | No | Mapping type (default: `"scalar"`). See section 5. |
| `+channel` | Depends | Record field name (in `info()`) or full PV name (in `.json`). Required for most types. |
| `+trigger` | No | Which group fields to update on change. See section 6. |
| `+putorder` | No | Enables put + controls processing order. See section 7. |
| `+atomic` | No | Lock all records together (default: `true`). See section 8. |
| `+const` | No | Static constant value (used with `+type:"const"`). |

### `+channel` in info Tags

In `info(Q:group, ...)`, `+channel` is a **field name of the enclosing record**, not a full PV name:

```
record(ai, "$(P)temp") {
    info(Q:group, {
        "$(P)grp":{
            "temperature":{+channel:"VAL"}
        }
    })
}
```

Valid short names: `VAL`, `SEVR`, `STAT`, `NAME`, `EGU`, `HOPR`, `LOPR`, `HIHI`, `HIGH`, `LOW`, `LOLO`, `PREC`, `DESC`, etc.

## 4. Group PV Definitions -- JSON File Syntax

Separate JSON files are loaded with `dbLoadGroup()` in `st.cmd`:

```
dbLoadGroup("db/groups.json", "P=TST:")
```

### JSON File Format

```json
{
    "$(P)position": {
        "+id": "myapp:position:1.0",
        "x": {
            "+type": "plain",
            "+channel": "$(P)X.VAL",
            "+trigger": "*"
        },
        "y": {
            "+type": "plain",
            "+channel": "$(P)Y.VAL",
            "+trigger": "*"
        },
        "": {"+type": "meta", "+channel": "$(P)X.SEVR"}
    }
}
```

### Key Differences from info Tag Syntax

| Aspect | `info(Q:group, ...)` | `.json` file |
|---|---|---|
| `+channel` value | Field name only (`"VAL"`) | Full PV name (`"rec:name.VAL"`) |
| `+` key quoting | Unquoted (`+type`) or quoted (`"+type"`) | Must be quoted (`"+type"`) |
| Macro substitution | Via `dbLoadRecords` macros | Via `dbLoadGroup` second argument |
| Scope | Tied to enclosing record | Standalone file |

### dbLoadGroup Management

```
# Load a group file
dbLoadGroup("db/groups.json", "P=TST:")

# Remove a specific file's definitions
dbLoadGroup("-db/groups.json", "P=TST:")

# Remove all previously loaded group definitions
dbLoadGroup("-*")
```

`dbLoadGroup()` must be called **before** `iocInit()`. The JSON is parsed during `initHookAfterInitDatabase`.

### softIocPVX Command Line

```bash
softIocPVX -d my.db -G groups.json
```

The `-G` flag calls `dbLoadGroup()` internally.

## 5. Mapping Types (`+type`)

| Type | `+channel` | Data | Description |
|---|---|---|---|
| `scalar` | Required | Full NTScalar/NTScalarArray sub-structure with alarm, timestamp, display, control, valueAlarm | Default type |
| `plain` | Required | Value field only, no metadata | Lightweight |
| `any` | Required | Variant union containing the value | For heterogeneous types |
| `meta` | Required | Only `alarm` and `timeStamp` fields | Top-level metadata |
| `structure` | No | Only the `+id` label | Structural container |
| `proc` | Required | No data fields; record is processed on PUT | Must set `+putorder` |
| `const` | No | Static value from `+const` | Compile-time constant |

### `const` Examples

```
record(ai, "$(P)dummy") {
    info(Q:group, {
        "$(P)grp": {
            "version":  {+type:"const", +const:3},
            "pi":       {+type:"const", +const:3.14159},
            "label":    {+type:"const", +const:"hello"}
        }
    })
}
```

## 6. Trigger Configuration (`+trigger`)

Controls which group fields are included in a subscription update when the associated record changes.

| Value | Behavior |
|---|---|
| `""` (empty/absent) | Change does NOT trigger a subscription update |
| `"*"` | Updates ALL fields in the group |
| `"field1,field2"` | Updates only the listed fields |

**Default behavior**: If a group has NO `+trigger` mappings at all, every field triggers only itself (split updates). This is usually undesirable.

**Recommended pattern**: Set `+trigger:""` on most fields, `+trigger:"*"` on the last-updated field:

```
record(waveform, "$(P)I") {
    info(Q:group, {
        "$(P)iq":{"I": {+channel:"VAL"}}
    })
}
record(waveform, "$(P)Q") {
    info(Q:group, {
        "$(P)iq":{"Q": {+channel:"VAL", +trigger:"*"}}
    })
}
```

Here, I is silently updated, and Q triggers a combined update containing both I and Q.

## 7. Put and Ordering (`+putorder`)

A field is **not writable** through the group PV unless `+putorder` is set.

```
record(ao, "$(P)A") {
    info(Q:group, {
        "$(P)grp":{"A":{+channel:"VAL", +putorder:0}}
    })
}
record(ao, "$(P)B") {
    info(Q:group, {
        "$(P)grp":{"B":{+channel:"VAL", +putorder:1}}
    })
}
record(calc, "$(P)SUM") {
    field(CALC, "A+B")
    info(Q:group, {
        "$(P)grp":{"SUM":{+channel:"VAL", +putorder:2, +trigger:"*"}}
    })
}
```

- `+putorder` values control processing order (ascending)
- Records with the same `+putorder` are processed in unspecified order
- A record without `+putorder` is read-only through the group
- `+type:"proc"` with `+putorder` creates a "process-only" trigger field

## 8. Atomicity (`+atomic`)

```
info(Q:group, {
    "$(P)grp":{
        +atomic:true,
        ...
    }
})
```

| Value | Behavior |
|---|---|
| `true` (default) | All member records are locked together for get/put/monitor |
| `false` | Records locked individually (faster, but data may be inconsistent) |

Client can override per-request: `record[atomic=true]` or `record[atomic=false]`.

## 9. NTTable Group Example

```
record(aai, "$(P)Labels_") {
    field(FTVL, "STRING")
    field(NELM, "3")
    field(INP, {const:["Name", "X", "Y"]})
    info(Q:group, {
        "$(P)Table":{
            +id:"epics:nt/NTTable:1.0",
            "labels":{+type:"plain", +channel:"VAL"}
        }
    })
}

record(aao, "$(P)Name") {
    field(FTVL, "STRING")
    field(NELM, "10")
    info(Q:group, {
        "$(P)Table":{
            "value.Name":{+type:"plain", +channel:"VAL", +putorder:0}
        }
    })
}

record(aao, "$(P)X") {
    field(FTVL, "DOUBLE")
    field(NELM, "10")
    info(Q:group, {
        "$(P)Table":{
            "value.X":{+type:"plain", +channel:"VAL", +putorder:1}
        }
    })
}

record(aao, "$(P)Y") {
    field(FTVL, "DOUBLE")
    field(NELM, "10")
    info(Q:group, {
        "$(P)Table":{
            "":{+type:"meta", +channel:"VAL"},
            "value.Y":{+type:"plain", +channel:"VAL", +putorder:2, +trigger:"*"}
        }
    })
}
```

Column order in the resulting structure follows `+putorder` values.

## 10. NTEnum Group Example

```
record(longout, "$(P)ENUM:INDEX") {
    field(VAL, "0")
    field(PINI, "YES")
    info(Q:group, {
        "$(P)ENUM":{
            +id:"epics:nt/NTEnum:1.0",
            "value":{+type:"structure", +id:"enum_t"},
            "value.index":{+type:"plain", +channel:"VAL", +putorder:0},
            "":{+type:"meta", +channel:"VAL"}
        }
    })
}

record(aai, "$(P)ENUM:CHOICES") {
    field(FTVL, "STRING")
    field(NELM, "4")
    field(INP, {const:["Off", "On", "Standby", "Error"]})
    field(PINI, "YES")
    info(Q:group, {
        "$(P)ENUM":{
            "value.choices":{+type:"plain", +channel:"VAL"}
        }
    })
}
```

## 11. NTNDArray / Image Group Example

In `.db`:

```
record(waveform, "$(P)ArrayData") {
    field(FTVL, "UCHAR")
    field(NELM, "1000000")
    info(Q:group, {
        "$(P)Image":{
            +id:"epics:nt/NTNDArray:1.0",
            "value":{+type:"any", +channel:"VAL", +putorder:0, +trigger:"*"},
            "":{+type:"meta", +channel:"SEVR"}
        }
    })
}
```

In `.json` (more fields):

```json
{
    "$(P)Image": {
        "+id": "epics:nt/NTNDArray:1.0",
        "value": {"+type": "any", "+channel": "$(P)ArrayData.VAL", "+trigger": "*"},
        "": {"+type": "meta", "+channel": "$(P)ArrayData.SEVR"},
        "dimension[0].size": {"+channel": "$(P)SizeX.VAL", "+type": "plain", "+putorder": 0},
        "dimension[1].size": {"+channel": "$(P)SizeY.VAL", "+type": "plain", "+putorder": 0},
        "attribute[0].name": {"+type": "plain", "+channel": "$(P)ColorMode_.VAL"},
        "attribute[0].value": {"+type": "any", "+channel": "$(P)ColorMode.VAL"}
    }
}
```

### Nested Structure Notation

- `"value.X"` -- creates sub-structure `value` containing field `X`
- `"dimension[0].size"` -- creates struct array `dimension`, element 0 with field `size`
- `"attribute[1].value"` -- struct array element 1, field `value`

## 12. PVA Links

PVA links connect IOC records to PVA channels (local or remote). Requires Base >= 7.0.1, pvxs >= 1.3.0.

### Full Syntax

```
record(ai, "$(P)mirror") {
    field(INP, {pva:{
        pv:"source:pv:name",
        field:"",
        local:false,
        Q:4,
        pipeline:false,
        proc:none,
        sevr:false,
        time:false,
        monorder:0,
        retry:false,
        defer:false,
        atomic:false
    }})
    field(SCAN, "I/O Intr")
}
```

### Short Form

```
field(INP, {pva:"source:pv:name"})
```

### `proc` Values

| Value | Input Link | Output Link |
|---|---|---|
| `none` / `null` | NPP by default | NPP or PP depending on server |
| `true` / `"PP"` | N/A | Process after write |
| `false` / `"NPP"` | N/A | Write without processing |
| `"CP"` | Subscribe; process on every update | N/A |
| `"CPP"` | Like CP if `SCAN=Passive`, else NPP | N/A |

### `sevr` Values

| Value | Behavior |
|---|---|
| `false` / `"NMS"` | Don't propagate alarm severity (default) |
| `"MS"` | Propagate alarm severity |
| `"MSI"` | Propagate only `INVALID` severity |

### Key Options

| Option | Default | Description |
|---|---|---|
| `pv` | (required) | Target PV name |
| `field` | `""` | Sub-field of target (e.g., `"value"`) |
| `local` | `false` | Require target in local IOC database |
| `Q` | `4` | Subscription queue depth (input links) |
| `pipeline` | `false` | Per-subscription flow control (input) |
| `time` | `false` | Copy target timestamp to record TIME field |
| `monorder` | `0` | Relative ordering for CP/CPP processing |
| `retry` | `false` | Retry incomplete PUT on reconnect (output) |
| `defer` | `false` | Cache value without flushing (output); combine multiple fields into one PUT |
| `atomic` | `false` | Lock related records together for CP/CPP (input) |

### Input Link Behavior

Creates a subscription to the target PV. Updates accumulate in a local cache. Link processing reads the most recent value. While disconnected, reads last value with `INVALID` severity.

For `proc:"CP"` or `proc:"CPP"`, the record is processed on each subscription update. Use `field(SCAN, "I/O Intr")` pattern:

```
record(ai, "$(P)mirror") {
    field(INP, {pva:{pv:"$(P)source", proc:"CP", sevr:"MS", time:true}})
    field(SCAN, "I/O Intr")
}
```

### Output Link Behavior

Writes the record's value to the target PV via a network PUT. `defer:true` caches the value without flushing, allowing multiple fields to combine into a single PUT (useful for group PVs).

```
record(ao, "$(P)setter") {
    field(OUT, {pva:{pv:"remote:target", proc:true}})
}
```

### Forward Link Behavior

Sends an empty PUT with `proc:true` (equivalent to writing to `.PROC`):

```
record(fanout, "$(P)trigger") {
    field(LNK1, {pva:{pv:"remote:target", proc:true}})
}
```

## 13. Adding Custom PVs to the IOC Server

Access the IOC's singleton PVXS Server to add custom SharedPVs or Sources.

```cpp
#include <initHooks.h>
#include <epicsExport.h>
#include <pvxs/iochooks.h>
#include <pvxs/server.h>
#include <pvxs/sharedpv.h>
#include <pvxs/nt.h>

using namespace pvxs;

static server::SharedPV myPV;

static void myInitHook(initHookState state)
{
    if(state != initHookAfterIocBuilt)
        return;

    myPV = server::SharedPV::buildMailbox();
    Value initial = nt::NTScalar{TypeCode::Float64}.create();
    initial["value"] = 0.0;
    myPV.open(initial);

    ioc::server().addPV("my:custom:pv", myPV);
}

static void myRegistrar()
{
    initHookRegister(&myInitHook);
}

extern "C" {
    epicsExportRegistrar(myRegistrar);
}
```

In `.dbd`:

```
registrar(myRegistrar)
```

PVs added at `initHookAfterIocBuilt` are available immediately when the server starts. PVs can also be added/removed after `iocInit()`.

## 14. IOC Shell Commands

| Command | Arguments | Description |
|---|---|---|
| `pvxsr` | `level` | Server report. Level 0: config. Level > 0: connected clients. |
| `pvxsl` | `level` | List all PV sources and names |
| `pvxsi` | (none) | Print module versions and target info |
| `pvxgl` | `level, pattern` | Group PV info. Level 0: names. Pattern restricts listing. |
| `dbLoadGroup` | `file, macros` | Load group definitions from JSON file |
| `pvxs_log_config` | `config` | Append logger config (e.g., `"pvxs.*=DEBUG"`) |
| `pvxs_log_reset` | (none) | Reset logging to defaults |
| `pvxrefshow` | (none) | Show internal instance counts |
| `pvxrefsave` | (none) | Save current instance counters |
| `pvxrefdiff` | (none) | Show difference since last `pvxrefsave` |

## 15. Access Security

QSRV2 enforces `.acf` policy loaded by `asSetFilename()`.

- For single PVs: same rules as CA/RSRV
- For group PVs: restrictions apply per member record, not the group itself
- Client hostname is always numeric IP (prevents hostname forgery)
- Set `asCheckClientIP=1` for hostname translation
- `UAG()` supports `role/` prefix for OS group membership: `UAG(admins) { someone, "role/operators" }`

## 16. Testing Patterns

### TestIOC (since pvxs 1.3.0, requires Base >= 3.15)

```cpp
#include <pvxs/iochooks.h>
#include <pvxs/client.h>

MAIN(mytest)
{
    testPlan(0);
    pvxs::testSetup();
    pvxs::logger_config_env();
    {
        pvxs::ioc::TestIOC ioc;

        testdbReadDatabase("mytest.dbd", NULL, NULL);
        mytest_registerRecordDeviceDriver(pdbbase);
        testdbReadDatabase("test.db", NULL, "P=TST:");

        // Optional
        // pvxs::ioc::dbLoadGroup("../groups.json", "P=TST:");

        ioc.init();

        // Create client connected to this IOC
        auto ctxt = pvxs::ioc::server().clientConfig().build();

        // GET
        auto result = ctxt.get("TST:pv").exec()->wait(5.0);
        testOk1(result["value"].as<double>() == 0.0);

        // MONITOR
        // ...

    }  // ioc destroyed here: calls shutdown + cleanup

    epicsExitCallAtExits();
    pvxs::cleanup_for_valgrind();
    return testDone();
}
```

`TestIOC` constructor calls `testdbPrepare()` + `testPrepare()`. Destructor calls `shutdown()` + `testdbCleanup()`.

## 17. Environment Variables Reference

| Variable | Default | Description |
|---|---|---|
| `PVXS_QSRV_ENABLE` | `YES` | Enable/disable QSRV2 (set before `*_registerRecordDeviceDriver`) |
| `EPICS_IOC_IGNORE_SERVERS` | (none) | Include `"qsrv2"` to quietly disable QSRV2 |
| `PVXS_LOG` | (none) | Logger configuration: `"pvxs.*=DEBUG,pvxs.ioc.*=INFO"` |
| `EPICS_PVAS_INTF_ADDR_LIST` | `0.0.0.0` | Server interfaces to bind |
| `EPICS_PVAS_BEACON_ADDR_LIST` | (auto) | Beacon destinations |
| `EPICS_PVAS_AUTO_BEACON_ADDR_LIST` | `YES` | Auto-add broadcast addresses |
| `EPICS_PVAS_SERVER_PORT` | `5075` | TCP port |
| `EPICS_PVAS_BROADCAST_PORT` | `5076` | UDP port |
| `EPICS_PVAS_IGNORE_ADDR_LIST` | (none) | Ignore requests from these addresses |
| `EPICS_PVA_CONN_TMO` | `30` | TCP timeout (multiplied by 4/3) |

## 18. Complete IOC Example

### `configure/RELEASE`

```makefile
PVXS = /path/to/pvxs
EPICS_BASE = /path/to/base
```

### `myApp/src/Makefile`

```makefile
TOP = ../..
include $(TOP)/configure/CONFIG

PROD_IOC = myApp
DBD += myApp.dbd

myApp_DBD += base.dbd
myApp_DBD += pvxsIoc.dbd

myApp_LIBS += pvxsIoc
myApp_LIBS += pvxs
myApp_LIBS += $(EPICS_BASE_IOC_LIBS)

myApp_SRCS += myApp_registerRecordDeviceDriver.cpp
myApp_SRCS += myAppMain.cpp

include $(TOP)/configure/RULES
```

### `myApp/Db/example.db`

```
record(ao, "$(P)setpoint") {
    field(DRVH, "100")
    field(DRVL, "0")
    field(EGU, "mm")
    field(PREC, "3")
    field(PINI, "YES")
    info(Q:form, "Default")
    info(Q:group, {
        "$(P)combined":{
            +id:"myapp:combined:1.0",
            "setpoint":{+type:"scalar", +channel:"VAL", +putorder:0}
        }
    })
}

record(ai, "$(P)readback") {
    field(EGU, "mm")
    field(PREC, "3")
    info(Q:group, {
        "$(P)combined":{
            "readback":{+type:"scalar", +channel:"VAL", +trigger:"*"},
            "":{+type:"meta", +channel:"VAL"}
        }
    })
}
```

### `iocBoot/iocMyApp/st.cmd`

```
#!../../bin/linux-x86_64/myApp

epicsEnvSet("P", "TST:")

dbLoadDatabase("../../dbd/myApp.dbd")
myApp_registerRecordDeviceDriver(pdbbase)

dbLoadRecords("../../db/example.db", "P=$(P)")

iocInit()
```

## 19. Key Rules and Pitfalls

1. **`+channel` differs between `info()` and `.json`**: In `info(Q:group, ...)`, `+channel` is a field name of the enclosing record (e.g., `"VAL"`). In `.json` files, it must be a full PV name (e.g., `"rec:name.VAL"`).

2. **`dbLoadGroup()` before `iocInit()`**: Group JSON files are parsed during `initHookAfterInitDatabase`. Calling `dbLoadGroup()` after `iocInit()` has no effect.

3. **Trigger defaults**: If a group has NO `+trigger` on any field, every field triggers only itself (split updates). Always set at least one `+trigger:"*"` to get combined updates.

4. **`+putorder` required for writes**: Fields without `+putorder` are read-only through the group PV. Set `+putorder` on every field that should be writable.

5. **QSRV1 conflict**: QSRV2 auto-disables if QSRV1 (from pva2pva module) is loaded. Do not include both `qsrv.dbd` and `pvxsIoc.dbd` in the same IOC.

6. **PVA link `proc:"CP"` requires `SCAN`**: For CP/CPP links, set `field(SCAN, "I/O Intr")` on the record. Without this, the record may not process on updates.

7. **Group PV names are global**: Group names must be unique across all `info(Q:group)` tags and JSON files. Duplicate group names from different records are merged. Conflicting definitions produce warnings.

8. **`+type:"meta"` for top-level alarm/timestamp**: Use `"":{+type:"meta", +channel:"VAL"}` (empty field name) to map alarm and timestamp to the top level of the group structure.

9. **`pvxsIoc.dbd` order**: Include `pvxsIoc.dbd` after `base.dbd` in your application DBD list. The registrar must run during `*_registerRecordDeviceDriver()`.

10. **JSON `+` keys must be quoted**: In `.json` files, all special keys must have the `+` prefix quoted: `"+type"`, `"+channel"`, `"+trigger"`, etc. In `info()` tags, the `+` can be unquoted.

11. **PVA link `defer:true` for batch writes**: When writing to a group PV from multiple output records, set `defer:true` on all but the last record to combine them into a single network PUT.

12. **`ioc::server()` timing**: `pvxs::ioc::server()` is available between `initHookAfterIocBuilt` and IOC shutdown. Calling it too early or too late throws `std::logic_error`.
