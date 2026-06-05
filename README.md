# OpenCode Skills for EPICS

A collection of [OpenCode](https://opencode.ai) skills for EPICS (Experimental Physics and Industrial Control System) development. These skills provide the AI assistant with detailed, accurate reference material for writing EPICS code.

## Available Skills

| Skill | Description |
|-------|-------------|
| `streamdevice` | Write StreamDevice protocol files (`.proto`) and database records for byte-stream device communication (serial, TCP/IP, GPIB) |
| `epics-database` | Write EPICS database files (`.db`), templates (`.template`), and substitution files (`.substitutions`) for all 34 standard record types |
| `epics-module` | Create and configure EPICS IOC applications and support modules -- directory structure, Makefiles, configure files, DBD files, and st.cmd startup scripts |
| `epics-device-support` | Write custom EPICS device support in C/C++ -- dset structures, init/read/write routines, async processing, I/O Intr scanning, iocsh commands, and sub/aSub functions |
| `epics-ca-client` | Write Channel Access client programs in C -- context management, get/put/monitor, DBR types, callbacks, and error handling |
| `epics-pva-client` | Write PV Access client and server programs in C++ -- pvac API, pvData structures, normative types, SharedPV servers, and QSRV group configuration |
| `epics-libcom` | Use libCom OS-independent APIs -- threading, mutexes, events, message queues, ring buffers, timers, time stamps, error logging, linked lists, and iocsh registration |
| `asyn-port-driver` | Write asynPortDriver subclasses in C++ -- parameter library, virtual read/write methods, interface/interrupt masks, background threads, and callbacks |
| `asyn-database` | Write database records for asyn drivers -- DTYP choices, INP/OUT link format, setpoint/readback patterns, I/O Intr scanning, and array records |
| `asyn-port-config` | Configure asyn ports in st.cmd -- IP, serial, and server ports, serial options, EOS settings, trace control, and diagnostic commands |
| `asyn-gpib` | Write GPIB/SCPI device support using the devGpib framework -- gpibCmd command tables, DSET macros, EFAST tables, and custom conversions |
| `motor-driver` | Write model-3 asyn motor drivers -- module creation with createMotorDriverModule, dual-build mode, asynMotorController and asynMotorAxis subclasses, and iocsh snippets |
| `motor-ioc` | Configure and deploy motor IOCs -- database templates, substitution files, motor record fields, st.cmd startup, motorUtil, and driver submodule integration |
| `areadetector-driver` | Write areaDetector camera/detector drivers -- ADDriver subclasses, NDArray allocation and lifecycle, acquisition thread patterns, image modes, and shutter control |
| `areadetector-ioc` | Configure and deploy areaDetector IOCs -- plugin chain configuration, database templates, commonPlugins.cmd, file writing patterns, and build configuration |
| `areadetector-plugin` | Write custom areaDetector plugins -- NDPluginDriver processing plugins and NDPluginFile file writer plugins with processCallbacks and NDArray handling |
| `snl` | Write State Notation Language (SNL) programs (.st/.stt) -- state machines, PV interaction, event flags, built-in functions, safe mode, embedded C code, and build integration |
| `modbus` | Configure Modbus IOCs for PLC and device communication -- register map translation, drvModbusAsynConfigure, 37 data types, function codes, 30 database templates, and TCP/serial setup |
| `synapps-deploy` | Deploy and configure EPICS synApps -- assemble_synApps script usage, module customization, building, creating IOCs from the xxx template, and managing module versions |
| `synapps-ioc` | Create and configure synApps IOCs using mkioc and the xxx template -- mkioc options, post-creation workflow, xxx template structure, hardware configuration examples, and module customization |
| `aeroscript-language` | Write AeroScript programs for Aerotech Automation1 -- syntax, data types, scoping, control flow, functions, libraries, preprocessor, G-code, and built-in functions |
| `aeroscript-motion` | AeroScript motion programming -- enable/home, linear/rapid/arc, PVT, async/sync moves, velocity blending, camming/gearing, transformations, lookahead |
| `aeroscript-pso` | AeroScript Position-Synchronized Output -- distance/event/output/window/waveform modules, fixed/array/continuous outputs, part-speed PSO |
| `aeroscript-runtime` | AeroScript runtime and I/O -- tasks, program lifecycle, I/O, TCP sockets, callbacks, parameters, status, faults, data collection, MachineApps |

Note: The AeroScript skills cover Aerotech controller programming, which is separate from the EPICS ecosystem. They are useful for writing reproducer programs for Aerotech support.

## Installation

Clone this repository into your OpenCode skills directory:

```bash
git clone git@git.aps.anl.gov:kpetersn/opencode-skills.git ~/.config/opencode/skills
```

If the `~/.config/opencode/skills` directory already exists, clone to a temporary location and copy:

```bash
git clone git@git.aps.anl.gov:kpetersn/opencode-skills.git /tmp/opencode-skills
cp -r /tmp/opencode-skills/*/ ~/.config/opencode/skills/
rm -rf /tmp/opencode-skills
```

Or, if you want to manage the skills directory as the git repo itself:

```bash
rm -rf ~/.config/opencode/skills   # Remove existing (back up first if needed)
git clone git@git.aps.anl.gov:kpetersn/opencode-skills.git ~/.config/opencode/skills
```

## Verification

After installation, restart OpenCode. The skills should appear in the available skills list. You can verify by checking for the skill directories:

```bash
ls ~/.config/opencode/skills/*/SKILL.md
```

Each skill is activated automatically when OpenCode detects a task matching the skill's description.

## Project-Local Installation

Skills can also be installed into a specific project's `.opencode/skills/` directory to make them available only within that project:

```bash
cd /path/to/my/project
mkdir -p .opencode/skills
git clone git@git.aps.anl.gov:kpetersn/opencode-skills.git /tmp/opencode-skills
cp -r /tmp/opencode-skills/*/ .opencode/skills/
rm -rf /tmp/opencode-skills
```

## Sources

These skills were derived from analysis of the [EPICS base 7.0](https://github.com/epics-base/epics-base), [asyn 4.45](https://github.com/epics-modules/asyn), [motor R7](https://github.com/epics-modules/motor), [areaDetector R3](https://github.com/areaDetector/areaDetector), [sequencer 2.2](https://github.com/epics-modules/sequencer), [modbus R3-4](https://github.com/epics-modules/modbus), [assemble_synApps](https://github.com/EPICS-synApps/assemble_synApps) and [synApps support](https://github.com/EPICS-synApps/support), [mkioc](https://github.com/BCDA-APS/mkioc) and [xxx](https://github.com/epics-modules/xxx), [StreamDevice 2.8.25+](https://github.com/paulscherrerinstitute/StreamDevice) source code and [documentation](https://paulscherrerinstitute.github.io/StreamDevice/), and the [Aerotech Automation1 help documentation](https://help.aerotech.com/automation1/), including record type definitions, build system templates, device support headers, CA/PVA client APIs, libCom headers, asynPortDriver class, asyn device support, devGpib framework, asynMotorController/asynMotorAxis base classes, motor record fields, ADDriver/NDPluginDriver/NDPluginFile base classes, NDArray lifecycle, plugin architecture, SNL language reference and built-in functions, Modbus data types and database templates, synApps assembly and build infrastructure, mkioc IOC creation workflow, xxx template IOC structure, StreamDevice protocol syntax and format converters, AeroScript language reference and motion/PSO/runtime APIs, and example programs.
