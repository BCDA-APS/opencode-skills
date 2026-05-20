---
name: epics-module
description: Create and configure EPICS IOC applications and support modules -- directory structure, Makefiles, configure files, DBD files, and st.cmd startup scripts
---

# EPICS Module Skill

You are an expert at creating and configuring EPICS IOC applications and support modules. You understand the EPICS build system, directory conventions, and the relationships between Makefiles, DBD files, configure files, and startup scripts.

---

## 1. Module Types

### 1.1 IOC Application

An executable that runs an EPICS IOC (Input/Output Controller). It links against support libraries and loads database files at runtime.

### 1.2 Support Module

A library that provides record types, device support, drivers, or utility functions. It is used by IOC applications but does not run standalone.

### 1.3 Combined Module

Many modules contain both a support library AND an IOC application (for testing or standalone use). The exampleApp template in EPICS base demonstrates this pattern.

---

## 2. Creating Modules with makeBaseApp.pl

**Always use `makeBaseApp.pl` to create new EPICS applications and modules.** Do not manually create the `configure/` directory, boilerplate Makefiles, or top-level Makefile. The tool generates all scaffolding correctly and is located at `$(EPICS_BASE)/bin/$(EPICS_HOST_ARCH)/makeBaseApp.pl`.

### 2.1 Usage Synopsis

```
makeBaseApp.pl -l                          # list available template types
makeBaseApp.pl -t type [options] [app ...] # create application directories
makeBaseApp.pl -i -t type [options] [ioc ...]  # create iocBoot directories
```

### 2.2 Available Template Types

| App Type | Description | Creates |
|----------|-------------|---------|
| `ioc` | Minimal IOC application | `<name>App/src/` with `PROD_IOC`, Main.cpp, DBD; `<name>App/Db/` |
| `support` | Support library module | `<name>App/src/` with `LIBRARY_IOC`, empty DBD; `<name>App/Db/` |
| `caClient` | Channel Access client programs | `<name>App/` (flat, no `src/` subdir) with `PROD_HOST`, example CA sources |
| `example` | Full example IOC with records, device support, SNL | `<name>App/src/`, `<name>App/Db/` with extensive examples |
| `caPerl` | Perl CA client scripts | `<name>App/` with example Perl scripts |

| iocBoot Type | Description |
|--------------|-------------|
| `ioc` | Minimal iocBoot directory with st.cmd |
| `example` | Example iocBoot with detailed st.cmd |

### 2.3 Key Options

| Flag | Description |
|------|-------------|
| `-t type` | Template type (from `-l` list). Defaults to `default` (alias for `ioc`). |
| `-i` | Create iocBoot directories instead of application directories. |
| `-a arch` | Set IOC architecture for `-i` (e.g., `linux-x86_64`, `vxWorks-68040`). Prompted if omitted. |
| `-b base` | Set EPICS_BASE path. Auto-detected from `configure/RELEASE`, environment, or script location. |
| `-p app` | Set the application name for `-i`. Prompted if omitted. |
| `-T top` | Override template top directory (where templates are found). |

### 2.4 Two-Step Pattern: IOC Application

Most IOC applications are created in two steps -- first the application directories, then the iocBoot directories:

```bash
# Step 1: Create application (generates configure/, <name>App/, top-level Makefile)
makeBaseApp.pl -t ioc myApp

# Step 2: Create iocBoot directories (generates iocBoot/iocMyApp/)
makeBaseApp.pl -i -t ioc myApp
```

The first invocation creates:
- `configure/` -- all boilerplate config files with `EPICS_BASE` set
- `myAppApp/` -- application directory with `src/` and `Db/` subdirectories
- `Makefile` -- top-level Makefile

The second invocation (`-i`) creates:
- `iocBoot/Makefile`
- `iocBoot/iocMyApp/` -- with `Makefile`, `st.cmd`

**The app name argument becomes the directory prefix.** `makeBaseApp.pl -t ioc myApp` creates `myAppApp/`. If you pass multiple names, it creates one `*App/` directory per name.

### 2.5 Single-Step Pattern: Support Module

Support modules typically do not need iocBoot:

```bash
makeBaseApp.pl -t support mySupport
```

### 2.6 Single-Step Pattern: Host Client Program

The `caClient` template creates a flat application directory suitable for `PROD_HOST` programs (no `src/` subdirectory, no DBD, no iocBoot):

```bash
makeBaseApp.pl -t caClient myClient
```

This is a good starting point for standalone client programs. After creation, replace or modify the generated source files and Makefile as needed (e.g., to use PVXS instead of CA).

### 2.7 Running in an Existing Directory

`makeBaseApp.pl` is designed to be run from the top of the application tree. If `configure/` already exists, it will not overwrite it (EPICS_BASE is read from the existing `configure/RELEASE`). This allows adding new `*App/` directories to an existing module:

```bash
# Add a second application to an existing module
makeBaseApp.pl -t ioc secondApp
```

### 2.8 Important Notes

- The tool must be run from the intended top-level directory of the application.
- The `-b` flag is only needed if `configure/RELEASE` does not yet exist and `EPICS_BASE` is not in the environment. When creating a brand new application, the tool auto-detects base from its own install location.
- Template files use `_APPNAME_` as a placeholder that gets replaced with the application name.
- The `configure/` directory is shared by all `*App/` directories in the same top. It is only created once.

---

## 3. Directory Structure

### 2.1 IOC Application

```
myTop/
    Makefile                    # Top-level Makefile
    .gitignore
    configure/
        CONFIG
        CONFIG_SITE
        RELEASE
        RULES
        RULES_TOP
        RULES_DIRS
        RULES.ioc
        Makefile
    myApp/
        Makefile                # Subdirectory Makefile (src, Db)
        src/
            Makefile            # PROD_IOC, DBD, SRCS, LIBS
            myAppMain.cpp       # IOC shell main entry point
        Db/
            Makefile            # DB += database files
            *.db                # Database instance files
            *.template          # Database templates
            *.substitutions     # Substitution files
    iocBoot/
        Makefile
        iocMyApp/
            Makefile            # ARCH, TARGETS = envPaths
            st.cmd              # Startup script
```

### 2.2 Support Module

```
mySupport/
    Makefile
    .gitignore
    configure/
        CONFIG
        CONFIG_SITE
        RELEASE
        RULES
        RULES_TOP
        RULES_DIRS
        RULES.ioc
        Makefile
    mySupportApp/
        Makefile
        src/
            Makefile            # LIBRARY_IOC, DBD, SRCS, LIBS
            mySupport.dbd       # DBD fragment
            *.c / *.cpp         # Source files
            *.h                 # Headers
        Db/
            Makefile            # DB += templates
            *.template          # Database templates
```

---

## 4. File Templates

### 4.1 Top-Level Makefile

```makefile
# Makefile at top of application tree
TOP = .
include $(TOP)/configure/CONFIG

# Directories to build, any order
DIRS += configure
DIRS += $(wildcard *Sup)
DIRS += $(wildcard *App)
DIRS += $(wildcard *Top)
DIRS += $(wildcard iocBoot)

# Build order dependencies:
# All dirs except configure depend on configure
$(foreach dir, $(filter-out configure, $(DIRS)), \
    $(eval $(dir)_DEPEND_DIRS += configure))

# Any *App dirs depend on all *Sup dirs
$(foreach dir, $(filter %App, $(DIRS)), \
    $(eval $(dir)_DEPEND_DIRS += $(filter %Sup, $(DIRS))))

# Any *Top dirs depend on all *Sup and *App dirs
$(foreach dir, $(filter %Top, $(DIRS)), \
    $(eval $(dir)_DEPEND_DIRS += $(filter %Sup %App, $(DIRS))))

# iocBoot depends on all *App dirs
iocBoot_DEPEND_DIRS += $(filter %App,$(DIRS))

include $(TOP)/configure/RULES_TOP
```

### 4.2 configure/RELEASE

```makefile
# RELEASE - Location of external support modules
#
# IF YOU CHANGE ANY PATHS in this file or make API changes to
# any modules it refers to, you should do a "make rebuild" in
# this application's top level directory.
#
# This file is parsed by both GNUmake and an EPICS Perl script,
# so it may ONLY contain definitions of paths to other support
# modules, variable definitions used in module paths, and include
# statements that pull in other RELEASE files.
# Build variables that are NOT used in paths should be set in
# the CONFIG_SITE file.

# Variables and paths to dependent modules:
#MODULES = /path/to/modules
#MYMODULE = $(MODULES)/my-module

# If using the sequencer, point SNCSEQ at its top directory:
#SNCSEQ = $(MODULES)/seq-ver

# EPICS_BASE should appear last so earlier modules can override stuff:
EPICS_BASE = /path/to/epics/base

# These lines allow developers to override these RELEASE settings
# without having to modify this file directly.
-include $(TOP)/../RELEASE.local
-include $(TOP)/../RELEASE.$(EPICS_HOST_ARCH).local
-include $(TOP)/configure/RELEASE.local
```

**Rules for RELEASE:**
- ONLY paths and path-related variable definitions. NO build flags.
- `EPICS_BASE` must appear LAST (so earlier modules can override base definitions).
- Module variables must be set BEFORE they are used in other paths.
- Use `-include` for optional local overrides.

### 4.3 configure/CONFIG

```makefile
# CONFIG - Load build configuration data
# Do not make changes to this file!

# Allow user to override where the build rules come from
RULES = $(EPICS_BASE)

# RELEASE files point to other application tops
include $(TOP)/configure/RELEASE
-include $(TOP)/configure/RELEASE.$(EPICS_HOST_ARCH).Common
ifdef T_A
  -include $(TOP)/configure/RELEASE.Common.$(T_A)
  -include $(TOP)/configure/RELEASE.$(EPICS_HOST_ARCH).$(T_A)
endif

# Check EPICS_BASE is set properly
ifneq (file,$(origin EPICS_BASE))
  $(error EPICS_BASE must be set in a configure/RELEASE file)
else
  ifeq ($(wildcard $(EPICS_BASE)/configure/CONFIG_BASE),)
    $(error EPICS_BASE does not point to an EPICS installation)
  endif
endif

CONFIG = $(RULES)/configure
include $(CONFIG)/CONFIG

# Override the Base definition:
INSTALL_LOCATION = $(TOP)

# CONFIG_SITE files contain local build configuration settings
include $(TOP)/configure/CONFIG_SITE
-include $(TOP)/configure/CONFIG_SITE.$(EPICS_HOST_ARCH).Common
ifdef T_A
 -include $(TOP)/configure/CONFIG_SITE.Common.$(T_A)
 -include $(TOP)/configure/CONFIG_SITE.$(EPICS_HOST_ARCH).$(T_A)
endif
```

**Do NOT modify this file.** It is boilerplate.

### 4.4 configure/CONFIG_SITE

```makefile
# CONFIG_SITE
# Make any application-specific changes to the EPICS build
#   configuration variables in this file.

# CHECK_RELEASE controls the consistency checking of the support
#   applications pointed to by the RELEASE* files.
# Normally CHECK_RELEASE should be set to YES.
# Set CHECK_RELEASE to NO to disable checking completely.
# Set CHECK_RELEASE to WARN to perform consistency checking but
#   continue building even if conflicts are found.
CHECK_RELEASE = YES

# Set this when you only want to compile this application
#   for a subset of the cross-compiled target architectures
#   that Base is built for.
#CROSS_COMPILER_TARGET_ARCHS = vxWorks-ppc32

# To install files into a location other than $(TOP) define
#   INSTALL_LOCATION here.
#INSTALL_LOCATION=</absolute/path/to/install/top>

# For application debugging purposes, override the HOST_OPT and/
#   or CROSS_OPT settings from base/configure/CONFIG_SITE
#HOST_OPT = NO
#CROSS_OPT = NO

# These allow developers to override the CONFIG_SITE variable
# settings without having to modify the configure/CONFIG_SITE
# file itself.
-include $(TOP)/../CONFIG_SITE.local
-include $(TOP)/configure/CONFIG_SITE.local
```

### 4.5 configure/RULES, RULES_TOP, RULES_DIRS, RULES.ioc

These are thin wrappers. Do not modify them.

**configure/RULES:**
```makefile
include $(CONFIG)/RULES
# Library should be rebuilt because LIBOBJS may have changed.
$(LIBNAME): ../Makefile
```

**configure/RULES_TOP:**
```makefile
include $(CONFIG)/RULES_TOP
```

**configure/RULES_DIRS:**
```makefile
include $(CONFIG)/RULES_DIRS
```

**configure/RULES.ioc:**
```makefile
include $(CONFIG)/RULES.ioc
```

### 4.6 configure/Makefile

```makefile
TOP=..
include $(TOP)/configure/CONFIG
TARGETS = $(CONFIG_TARGETS)
CONFIGS += $(subst ../,,$(wildcard $(CONFIG_INSTALLS)))
include $(TOP)/configure/RULES
```

### 4.7 .gitignore

```
# Install directories
/bin/
/cfg/
/db/
/dbd/
/html/
/include/
/lib/
/templates/

# Local configuration files
/configure/*.local

# iocBoot generated files
/iocBoot/*ioc*/cdCommands
/iocBoot/*ioc*/dllPath.bat
/iocBoot/*ioc*/envPaths
/iocBoot/*ioc*/relPaths.sh

# iocsh
.iocsh_history

# Build directories
O.*/

# Common files created by other tools
.DS_Store
```

---

## 5. Application Makefiles

### 5.1 App/Makefile (Subdirectory Wrapper)

```makefile
TOP = ..
include $(TOP)/configure/CONFIG

# Directories to be built, in any order.
DIRS += $(wildcard src* *Src*)
DIRS += $(wildcard db* *Db*)

include $(TOP)/configure/RULES_DIRS
```

### 5.2 IOC src/Makefile

```makefile
TOP=../..
include $(TOP)/configure/CONFIG

#=============================
# Build the IOC application

PROD_IOC = myApp

# myApp.dbd will be created and installed
DBD += myApp.dbd

# myApp.dbd will be made up from these files:
myApp_DBD += base.dbd

# Include dbd files from all support applications:
#myApp_DBD += xxxSupport.dbd

# Add all the support libraries needed by this IOC
#myApp_LIBS += xxxSupport

# myApp_registerRecordDeviceDriver.cpp derives from myApp.dbd
myApp_SRCS += myApp_registerRecordDeviceDriver.cpp

# Build the main IOC entry point on workstation OSs.
myApp_SRCS_DEFAULT += myAppMain.cpp
myApp_SRCS_vxWorks += -nil-

# Finally link to the EPICS Base libraries
myApp_LIBS += $(EPICS_BASE_IOC_LIBS)

include $(TOP)/configure/RULES
```

### 5.3 Support Library src/Makefile

```makefile
TOP=../..
include $(TOP)/configure/CONFIG

#==================================================
# build a support library

LIBRARY_IOC += mySupport

# xxxRecord.h will be created from xxxRecord.dbd
#DBDINC += xxxRecord

# install mySupport.dbd into <top>/dbd
DBD += mySupport.dbd

# specify all source files to be compiled and added to the library
mySupport_SRCS += myDriver.c
mySupport_SRCS += myDeviceSupport.c

mySupport_LIBS += $(EPICS_BASE_IOC_LIBS)

include $(TOP)/configure/RULES
```

### 5.4 Combined (Support Library + IOC) src/Makefile

```makefile
TOP=../..
include $(TOP)/configure/CONFIG

# Use typed rset structure (recommended for EPICS 7)
USR_CPPFLAGS += -DUSE_TYPED_RSET -DUSE_TYPED_DSET

#==================================================
# Build a support library
LIBRARY_IOC += mySupport

# Install DBD fragment for downstream consumers
DBD += mySupport.dbd

# Support library sources
mySupport_SRCS += myDriver.c
mySupport_SRCS += myDeviceSupport.c
mySupport_SRCS += myIocshCommands.c

mySupport_LIBS += $(EPICS_BASE_IOC_LIBS)

#==================================================
# Build the IOC application
PROD_IOC = myApp

# Composite DBD assembled from fragments
DBD += myApp.dbd
myApp_DBD += base.dbd
myApp_DBD += mySupport.dbd

# Auto-generated registration code
myApp_SRCS += myApp_registerRecordDeviceDriver.cpp

# Main for workstation OSs only
myApp_SRCS_DEFAULT += myAppMain.cpp
myApp_SRCS_vxWorks += -nil-

# Link the support library
myApp_LIBS += mySupport

# Link QSRV (pvAccess Server) if available
ifdef EPICS_QSRV_MAJOR_VERSION
    myApp_LIBS += qsrv
    myApp_LIBS += $(EPICS_BASE_PVA_CORE_LIBS)
    myApp_DBD += PVAServerRegister.dbd
    myApp_DBD += qsrv.dbd
endif

# EPICS Base libraries last
myApp_LIBS += $(EPICS_BASE_IOC_LIBS)

include $(TOP)/configure/RULES
```

### 5.5 Db/Makefile

```makefile
TOP=../..
include $(TOP)/configure/CONFIG

# Install databases, templates & substitutions
DB += myRecords.db
DB += myTemplate.template
DB += myInstances.substitutions

include $(TOP)/configure/RULES
```

---

## 6. Key Makefile Variables

### 6.1 Products and Libraries

| Variable | Description |
|----------|-------------|
| `PROD_IOC` | IOC executable product name |
| `PROD_HOST` | Host-only executable product name |
| `LIBRARY_IOC` | IOC support library name |
| `LIBRARY_HOST` | Host-only library name |
| `LOADABLE_LIBRARY` | Dynamically loadable library name |
| `TESTPROD` | Test programs (not installed) |

### 6.2 Source Files

| Variable | Description |
|----------|-------------|
| `<name>_SRCS` | Sources for a specific product or library |
| `<name>_SRCS_DEFAULT` | Sources for all targets except those with explicit overrides |
| `<name>_SRCS_<os>` | OS-specific sources (e.g., `_vxWorks`, `_RTEMS`) |
| `-nil-` | Special value meaning "no sources" (use to suppress defaults, e.g., `_SRCS_vxWorks += -nil-`) |
| `SRCS` | Sources for all products and libraries |
| `LIB_SRCS` | Sources for all libraries only |
| `PROD_SRCS` | Sources for all products only |

### 6.3 DBD Files

| Variable | Description |
|----------|-------------|
| `DBD` | DBD files to install into `<top>/dbd/` |
| `<name>_DBD` | DBD fragments to concatenate into `<name>.dbd` |
| `DBDINC` | DBD files that also generate a C header (for record types) |

### 6.4 Database Files

| Variable | Description |
|----------|-------------|
| `DB` | Database files to install into `<top>/db/` (.db, .template, .substitutions) |
| `DB_INSTALLS` | Pre-existing DB files to copy from other locations |

### 6.5 Linking

| Variable | Description |
|----------|-------------|
| `<name>_LIBS` | Libraries to link (in dependency order, most dependent first) |
| `<name>_SYS_LIBS` | System libraries (e.g., pthread, m) |
| `<lib>_DIR` | Search directory for an external library |
| `$(EPICS_BASE_IOC_LIBS)` | All EPICS base IOC libraries -- **always last** |
| `$(EPICS_BASE_HOST_LIBS)` | All EPICS base host libraries |
| `$(EPICS_BASE_PVA_CORE_LIBS)` | PV Access core libraries |

### 6.6 Include Files

| Variable | Description |
|----------|-------------|
| `INC` | Header files to install into `<top>/include/` |
| `INC_<os>` | OS-specific headers |

### 6.7 Build Flags

| Variable | Description |
|----------|-------------|
| `USR_CPPFLAGS` | C preprocessor flags for all compilations |
| `USR_CFLAGS` | C compiler flags |
| `USR_CXXFLAGS` | C++ compiler flags |
| `USR_LDFLAGS` | Linker flags |
| `USR_INCLUDES` | Additional include paths |
| `<file>_CPPFLAGS` | Flags for one source file only |

### 6.8 Version Generation

| Variable | Description |
|----------|-------------|
| `GENVERSION` | Auto-generated version header filename (e.g., `myVersion.h`) |
| `GENVERSIONMACRO` | Macro name defined in the version header |

---

## 7. DBD File Format

DBD (Database Definition) files declare record types, device support, drivers, and IOC shell registrations.

### 7.1 Syntax

```
# Comment
include "otherFile.dbd"                                    # Include another DBD
device(recordType, linkType, dsetName, "Choice String")    # Device support binding
registrar(functionName)                                    # C registrar function
function(functionName)                                     # Sub/aSub callable function
variable(variableName)                                     # IOC shell accessible variable
driver(driverName)                                         # Driver registration
```

### 7.2 DBD for a Support Module

A support module installs a DBD fragment that downstream IOC applications include:

```
# mySupport.dbd
include "xxxRecord.dbd"
device(xxx, CONSTANT, devXxxSoft, "Soft Channel")
device(ai, INST_IO, devMyAi, "My AI Driver")
registrar(myCommandsRegister)
```

### 7.3 DBD for an IOC Application

The IOC application assembles a composite DBD from `base.dbd` and all support module DBDs:

```makefile
# In src/Makefile:
myApp_DBD += base.dbd           # Always first -- standard record types
myApp_DBD += mySupport.dbd      # From support modules
myApp_DBD += otherSupport.dbd
```

The build system concatenates these into `myApp.dbd` and auto-generates `myApp_registerRecordDeviceDriver.cpp` from it.

**CRITICAL: Never hand-edit `_registerRecordDeviceDriver.cpp` -- it is auto-generated from the composite DBD file.**

### 7.4 Link Types for device() Declarations

| Link Type | Description | INP/OUT Format |
|-----------|-------------|----------------|
| `CONSTANT` | Constant value or PV link | `"value"` or `"pvname"` |
| `INST_IO` | Instrument I/O | `"@parameter_string"` |
| `VME_IO` | VME bus | `"#Cn Sn @parm"` |
| `GPIB_IO` | GPIB bus | `"#Ln An @parm"` |
| `CAMAC_IO` | CAMAC bus | `"#Bn Cn Nn An Fn @parm"` |

---

## 8. IOC Main Program

```cpp
/* myAppMain.cpp */
#include <stddef.h>
#include <stdlib.h>
#include <string.h>
#include <stdio.h>

#include "epicsExit.h"
#include "epicsThread.h"
#include "iocsh.h"

int main(int argc, char *argv[])
{
    if (argc >= 2) {
        iocsh(argv[1]);
        epicsThreadSleep(.2);
    }
    iocsh(NULL);
    epicsExit(0);
    return 0;
}
```

This file is only compiled for workstation (host) OSs. For vxWorks, use `_SRCS_vxWorks += -nil-`.

---

## 9. Startup Script (st.cmd)

### 9.1 Standard Host st.cmd

```bash
#!../../bin/linux-x86_64/myApp

< envPaths

cd "${TOP}"

## Register all support components
dbLoadDatabase "dbd/myApp.dbd"
myApp_registerRecordDeviceDriver pdbbase

## Load record instances
dbLoadRecords("db/myRecords.db", "P=TEST:")
dbLoadTemplate("db/myInstances.substitutions")

cd "${TOP}/iocBoot/${IOC}"
iocInit

## Start any sequence programs
#seq sncExample, "user=myuser"
```

### 9.2 st.cmd Ordering Rules

The order of commands in st.cmd is critical:

1. `< envPaths` -- source environment paths (defines TOP, EPICS_BASE, module paths)
2. `cd "${TOP}"` -- all subsequent paths relative to TOP
3. `dbLoadDatabase` -- load the composite DBD file
4. `_registerRecordDeviceDriver pdbbase` -- register all record/device/driver support from the DBD
5. Driver/port configuration commands (e.g., `drvAsynIPPortConfigure`, `drvAsynSerialPortConfigure`)
6. `dbLoadRecords` / `dbLoadTemplate` -- load database instance files
7. `iocInit` -- initialize and start the IOC
8. Post-init commands (e.g., `seq` for State Notation Language programs)

**CRITICAL: `dbLoadDatabase` and `registerRecordDeviceDriver` MUST come before `dbLoadRecords`. `iocInit` MUST come after all `dbLoadRecords` calls.**

### 9.3 iocBoot/iocMyApp/Makefile

```makefile
TOP = ../..
include $(TOP)/configure/CONFIG
ARCH = $(EPICS_HOST_ARCH)
TARGETS = envPaths
include $(TOP)/configure/RULES.ioc
```

For cross-compiled targets (vxWorks), use `ARCH = <target-arch>` and `TARGETS = cdCommands`.

### 9.4 iocBoot/Makefile

```makefile
TOP = ..
include $(TOP)/configure/CONFIG
DIRS += $(wildcard *ioc*)
DIRS += $(wildcard as*)
include $(CONFIG)/RULES_DIRS
```

---

## 10. Adding Dependencies

When adding a support module dependency, three files must be updated consistently:

### Step 1: configure/RELEASE

```makefile
# Add the module path
ASYN = /path/to/asyn
STREAM = /path/to/StreamDevice
# EPICS_BASE must remain last
EPICS_BASE = /path/to/base
```

### Step 2: src/Makefile -- DBD

```makefile
# Add the support module DBD to the composite DBD
myApp_DBD += asyn.dbd
myApp_DBD += stream.dbd
```

### Step 3: src/Makefile -- Libraries

```makefile
# Add support libraries (most dependent first, base last)
myApp_LIBS += stream
myApp_LIBS += asyn
myApp_LIBS += $(EPICS_BASE_IOC_LIBS)
```

**Library ordering rule:** Libraries must be listed in reverse dependency order -- the most dependent library first, the least dependent last. `$(EPICS_BASE_IOC_LIBS)` is always last.

---

## 11. Common Operations

### 11.1 Adding a New Source File

```makefile
# To a specific product or library:
myApp_SRCS += newFile.c

# To all libraries in this Makefile:
LIB_SRCS += newFile.c

# To all products in this Makefile:
PROD_SRCS += newFile.c

# OS-specific source:
myApp_SRCS_linux += linuxSpecific.c
myApp_SRCS_DEFAULT += defaultImpl.c
myApp_SRCS_vxWorks += -nil-
```

### 11.2 Adding a New Database File

In `Db/Makefile`:
```makefile
DB += myNewRecords.db
DB += myTemplate.template
DB += mySubstitutions.substitutions
```

### 11.3 Installing Header Files

```makefile
INC += myPublicApi.h
INC_linux += myLinuxApi.h
```

### 11.4 Adding an iocsh Command

1. Write the C source with the iocsh registration pattern (see epics-device-support skill).
2. Create a `.dbd` file with `registrar(myRegistrar)`.
3. Add both to the Makefile:
```makefile
mySupport_SRCS += myCommands.c
DBD += myCommands.dbd
# or, if part of a support library DBD:
# Add include "myCommands.dbd" to mySupport.dbd
```

### 11.5 Adding a Custom Record Type

1. Write `xxxRecord.dbd` defining the record type.
2. Write `xxxRecord.c` implementing record support.
3. In `src/Makefile`:
```makefile
DBDINC += xxxRecord           # Creates xxxRecord.h from xxxRecord.dbd
mySupport_SRCS += xxxRecord.c
```
4. Add `include "xxxRecord.dbd"` and `device()` declaration to the support DBD file.

### 11.6 Adding QSRV (PV Access Server) Support

```makefile
# In src/Makefile:
ifdef EPICS_QSRV_MAJOR_VERSION
    myApp_LIBS += qsrv
    myApp_LIBS += $(EPICS_BASE_PVA_CORE_LIBS)
    myApp_DBD += PVAServerRegister.dbd
    myApp_DBD += qsrv.dbd
endif
```

---

## 12. Key Rules and Pitfalls

1. **The composite DBD name must match `PROD_IOC` name.** If `PROD_IOC = myApp`, then `DBD += myApp.dbd` and `myApp_DBD += base.dbd ...`.

2. **`_registerRecordDeviceDriver.cpp` is auto-generated** from the composite `.dbd` file. Never hand-edit it. If you add new DBD fragments, the registration code is regenerated automatically.

3. **`_Main.cpp` is only for workstation (host) OSs.** Use `_SRCS_DEFAULT += myAppMain.cpp` and `_SRCS_vxWorks += -nil-`.

4. **Libraries must be listed in reverse dependency order.** Most dependent first, `$(EPICS_BASE_IOC_LIBS)` always last.

5. **`epicsExport.h` must be the LAST EPICS include** in any C/C++ source file that uses `epicsExportAddress()`, `epicsExportRegistrar()`, or `epicsRegisterFunction()`.

6. **`configure/RELEASE` must not contain build variables** -- only paths to module tops. Build settings go in `CONFIG_SITE`.

7. **After changing RELEASE**, always run `make rebuild` (or `make clean uninstall; make`) to ensure consistency.

8. **`CHECK_RELEASE = YES`** in `CONFIG_SITE` enables consistency checking of module paths. Set to `WARN` during development if you need to tolerate mismatches temporarily.

9. **The build system uses `INSTALL_LOCATION = $(TOP)`** by default. All built products are installed into `$(TOP)/bin/`, `$(TOP)/lib/`, `$(TOP)/dbd/`, `$(TOP)/db/`, `$(TOP)/include/`.

10. **For local override files**, use `configure/RELEASE.local` and `configure/CONFIG_SITE.local`. These are in `.gitignore` and should NOT be committed. They allow per-developer path customization.

11. **DBD search path** for `include` statements in DBD files is automatically configured from all modules declared in RELEASE.

12. **When creating a new IOC boot directory**, the Makefile must set `ARCH` and `TARGETS`:
    - Host IOC: `ARCH = $(EPICS_HOST_ARCH)`, `TARGETS = envPaths`
    - vxWorks: `ARCH = <target-arch>`, `TARGETS = cdCommands`
