---
name: aeroscript-runtime
description: AeroScript runtime, I/O, communications, and program lifecycle -- task management, program load/run/control, immediate commands, I/O, TCP sockets, callbacks, parameters, status, faults, data collection, and MachineApps integration
---

# AeroScript Runtime Skill

You are an expert at AeroScript runtime features for Aerotech Automation1 controllers: task and program management, I/O, TCP sockets, callbacks, parameters, status queries, fault handling, data collection, and MachineApps application integration.

---

## 1. Host-Application Interaction Overview

Some AeroScript functions are designed to interact with host applications (Automation1 Studio, MachineApps, or custom .NET/C/Python apps connected via the MDK). These functions execute as **non-real-time callbacks** -- the AeroScript program pauses, the callback is handled by the communication service, and then the program resumes.

### Functions Designed for Host Interaction

| Category | Functions | Requires Host |
|----------|-----------|---------------|
| **Custom Callbacks** | `Callback()` | Yes -- .NET/C/Python handler must be registered |
| **Message Dialogs** | `AppMessageDisplay()`, `AppMessageBox()`, `AppMessageInputBox()`, `AppMessageMenu()` | Yes -- Studio or MachineApps |
| **File Dialogs** | `AppMessageFileOpen()`, `AppMessageFileSave()` | Yes -- Studio or MachineApps |
| **Indicators** | `AppIndicatorOn()`, `AppIndicatorOff()` | Yes -- MachineApps |
| **Buttons** | `AppButtonSetState()` | Yes -- MachineApps |
| **Data Collection Snapshot** | `AppDataCollectionSnapshot()`, `AppDataCollectionStop()` | Yes -- Studio |
| **Program Load/Run** | `ProgramLoad()`, `ProgramRun()` | Partial -- can also be called from AeroScript |

### Functions That Are Always Non-Real-Time (No Host Required)

File I/O, socket, and DateTime functions execute as callbacks but do not require a connected host application -- they are handled by the controller's communication service internally.

### Checking for Host Registration

Always check before calling host-dependent functions:

```aeroscript
if AppMessageDisplayIsRegistered()
    AppMessageDisplay("Status: running")
end

if CallbackIsRegistered(42)
    Callback(42, [], [], [], ref $iout, ref $rout, ref $sout)
end
```

---

## 2. Task and Program Management

### 2.1 Task Model

Each Automation1 controller has 1-31 tasks (standard: 4, plus license). Each task:
- Runs one program at a time
- Has its own state: `Idle`, `ProgramReady`, `ProgramRunning`, `ProgramPaused`, `ProgramComplete`, `Error`
- Has task-local variables (`$rtask[]`, `$itask[]`, `$stask[]`)
- Shares controller globals (`$rglobal[]`, `$iglobal[]`, `$sglobal[]`) with all other tasks

### 2.2 Program Lifecycle

```aeroscript
// Load a compiled program onto a task (does not start it)
ProgramLoad($taskIndex as integer, $controllerFileName as string)

// Start a loaded program (task must be in ProgramReady state)
ProgramStart($taskIndex as integer)

// Load and start in one call
ProgramRun($taskIndex as integer, $controllerFileName as string)

// Pause a running program (decelerates axes to stop)
ProgramPause($taskIndex as integer)
ProgramPause()                          // Pause current task

// Exit the current program (sets state to ProgramComplete)
ProgramExit()

// Stop and unload a program on another task
ProgramStop($taskIndex as integer)

// Reset a program (rewinds to beginning, state -> ProgramReady)
ProgramReset()
ProgramReset($taskIndex as integer)

// Reset and restart in one call
ProgramRestart()

// Remove a compiled program from the controller
ProgramRemove($controllerFileName as string)
```

**Note:** `ProgramLoad` and `ProgramRun` are non-real-time callbacks. Max 100 programs/libraries loaded simultaneously.

### 2.3 Program Debugging from AeroScript

```aeroscript
ProgramStepInto($taskIndex as integer)   // Step into next line/function
ProgramStepOver($taskIndex as integer)   // Step over (don't enter functions)
ProgramStepOut($taskIndex as integer)    // Step out of current function
```

### 2.4 Get Current Task Index

```aeroscript
var $myTask as integer = TaskGetIndex()
```

### 2.5 Multi-Task Example

```aeroscript
// Main program loads a helper on another task
program
    ProgramRun(2, "helperProgram.a1exe")
    // ... do work on this task ...
    ProgramStop(2)
end
```

---

## 3. Critical Sections and Task Scheduling

```aeroscript
CriticalSectionStart()                     // No yielding until end
CriticalSectionStart($timeoutUs as integer) // With timeout in microseconds
CriticalSectionEnd()
CriticalSectionEndAll()                     // End all nested sections
```

Inside a critical section, the task executes continuously without yielding to the scheduler. This is essential for:
- PVT motion sequences (prevent gaps between segments)
- Tight timing loops
- Atomic multi-register writes

Critical sections are nestable. Non-real-time functions (file I/O, sockets, callbacks) will still pause execution.

---

## 4. Feedhold and Motion Override

### 4.1 Feedhold

```aeroscript
TaskFeedholdOn($taskIndex as integer)    // Decelerate to stop along path
TaskFeedholdOff($taskIndex as integer)   // Resume motion
```

Feedhold decelerates all axes to a stop along the programmed path. Does NOT affect PVT/PT, Home, or follower axes.

### 4.2 Motion/Spindle Override

```aeroscript
TaskMfo($taskIndex as integer, $percentage as real)   // Motion feedrate override (0-100+)
TaskMso($taskIndex as integer, $percentage as real)   // Spindle speed override
```

### 4.3 Interrupt Motion

```aeroscript
TaskInterruptMotionOn($taskIndex as integer)
// ... execute immediate commands (e.g., jog to a different position) ...
TaskInterruptMotionOff($taskIndex, ReturnType.Interrupt)   // Return to interrupted position
TaskInterruptMotionOff($taskIndex, ReturnType.Offset)      // Continue from new position
```

### 4.4 Task Control Restriction

```aeroscript
TaskControlRestrict(TaskGetIndex())    // Prevent external control of this task
TaskControlAllow(TaskGetIndex())       // Re-enable external control
```

Prevents Studio, other AeroScript tasks, or APIs from pausing, stopping, feedholding, MFO/MSO, or debugging on the restricted task. Useful for safety-critical or uninterruptible operations.

---

## 5. Analog and Digital I/O

### 5.1 Analog I/O

```aeroscript
var $voltage as real = AnalogInputGet($axis, $inputNum as integer)
AnalogOutputSet($axis, $outputNum as integer, $value as real)
var $currentOutput as real = AnalogOutputGet($axis, $outputNum as integer)
```

I/O executes in real-time (no callback pause).

### 5.2 Digital I/O

```aeroscript
var $state as integer = DigitalInputGet($axis, $inputNum as integer)
DigitalOutputSet($axis, $outputNum as integer, $value as integer)
var $currentState as integer = DigitalOutputGet($axis, $outputNum as integer)
```

### 5.3 Virtual I/O (Inter-Task Communication)

Virtual I/O is not connected to hardware. Used for communication between tasks or with host applications.

```aeroscript
// Binary (0/1)
VirtualBinaryInputSet($index as integer, $value as integer)
var $val as integer = VirtualBinaryInputGet($index as integer)
VirtualBinaryOutputSet($index as integer, $value as integer)
var $val as integer = VirtualBinaryOutputGet($index as integer)

// Register (real values)
VirtualRegisterInputSet($index as integer, $value as real)
var $val as real = VirtualRegisterInputGet($index as integer)
VirtualRegisterOutputSet($index as integer, $value as real)
var $val as real = VirtualRegisterOutputGet($index as integer)
```

### 5.4 Advanced Analog Output Modes

```aeroscript
// Array mode: output values from drive array, triggered by PSO/time
AnalogOutputConfigureArrayMode($axis, $outputNum, AnalogOutputUpdateEvent.PsoEvent,
    $driveArrayStartAddress, $numberOfPoints, $divisor, $enableRepeat)

// Axis tracking mode: output proportional to axis position/velocity/current
AnalogOutputConfigureAxisTrackingMode($axis, $outputNum,
    AnalogOutputAxisTrackingItem.VelocityFeedback,
    $scaleFactor, $offset, $minVoltage, $maxVoltage)

// Return to default (manual) mode
AnalogOutputConfigureDefaultMode($axis, $outputNum)
```

---

## 6. TCP Sockets

### 6.1 TCP Client

```aeroscript
var $socket as handle = SocketTcpClientCreate("192.168.1.100", 8080, 10000)
// timeout in milliseconds

// Or connect by hostname
var $socket as handle = SocketTcpClientCreateFromHost("myserver",
    HostAddressType.IPv4, 8080, 10000)

// Check connection
if SocketTcpClientIsConnected($socket)
    SocketWriteString($socket, "HELLO\n")
    var $response as string = SocketReadString($socket, 256)
end

SocketClose($socket)
```

### 6.2 TCP Server

```aeroscript
var $server as handle = SocketTcpServerCreate(9001)

// Wait for client
while !SocketTcpServerIsClientPending($server)
    Dwell(0.01)
end

var $client as handle = SocketTcpServerAccept($server)
SocketSetDataReadTimeout($client, 1000)

// Read/write
var $data as string = SocketReadString($client, 256)
SocketWriteString($client, "ACK\n")

SocketClose($client)
SocketClose($server)
```

### 6.3 Binary Read/Write

```aeroscript
SocketWriteInt32($socket, 42)
SocketWriteFloat64($socket, 3.14)
var $n as integer = SocketReadInt32($socket)
var $v as real = SocketReadFloat64($socket)

// Array variants
var $data[100] as real
SocketReadFloat64Array($socket, ref $data, 100)
SocketWriteFloat64Array($socket, $data, 100)
```

Available types: `Float32`, `Float64`, `Int8`, `Int16`, `Int32`, `Int64`, `UInt8`, `UInt16`, `UInt32`.

### 6.4 Socket Configuration

```aeroscript
SocketSetDataReadTimeout($socket, $timeoutMs as integer)
SocketSetDataWriteTimeout($socket, $timeoutMs as integer)
SocketSetDataByteOrder($socket, SocketByteOrder.BigEndian)
var $avail as integer = SocketGetReadBytesAvailable($socket)
```

**Note:** All socket functions are non-real-time callbacks.

---

## 7. Custom Callbacks (Host-Application)

```aeroscript
var $iInputs[] as integer = [1, 2, 3]
var $rInputs[] as real = [3.14]
var $sInputs[] as string = ["hello"]
var $iOutputs[10] as integer
var $rOutputs[10] as real
var $sOutputs[10] as string

// Check if a handler is registered
if CallbackIsRegistered(42)
    // Call callback ID 42 -- pauses program until host handler returns
    Callback(42, $iInputs, $rInputs, $sInputs,
             ref $iOutputs, ref $rOutputs, ref $sOutputs)
end
```

- Callback IDs: 0-65535 (user-defined)
- Array lengths: max 100 each
- The host application (.NET/C/Python) must register a handler for the callback ID
- If no handler is registered, a `CallbackNotRegistered` error occurs
- The AeroScript program pauses until the host handler completes and returns output data

---

## 8. Parameters

### 8.1 Read Active Parameters

```aeroscript
var $val as real = ParameterGetAxisValue($axis, AxisParameter.CountsPerUnit)
var $str as string = ParameterGetAxisStringValue($axis, AxisParameter.AxisName)
var $tval as real = ParameterGetTaskValue($taskIndex, TaskParameter.MaxLookaheadMoves)
var $sval as real = ParameterGetSystemValue(SystemParameter.ServoUpdateRate)
```

### 8.2 Set Active Parameters (Temporary -- Lost on Controller Reset)

```aeroscript
ParameterSetAxisValue($axis, AxisParameter.DefaultAxisSpeed, 100.0)
ParameterSetTaskValue(TaskGetIndex(), TaskParameter.MaxLookaheadMoves, 1000)
ParameterSetSystemValue(SystemParameter.DigitalOutputDebounceTime, 5)
```

### 8.3 Read Configured Parameters (Persistent, from MCD File)

```aeroscript
var $configured as real = ConfiguredParameterGetAxisValue($axis, AxisParameter.CountsPerUnit)
```

Configured values are read-only from AeroScript. They represent what is saved in the controller configuration file.

---

## 9. Controller Status

### 9.1 Axis Status

```aeroscript
// Read a status item
var $pos as real = StatusGetAxisItem(X, AxisStatusItem.PositionCommand)
var $vel as real = StatusGetAxisItem(X, AxisStatusItem.VelocityFeedback)
var $err as real = StatusGetAxisItem(X, AxisStatusItem.PositionError)

// Check bitmask status
var $isEnabled as integer = StatusGetAxisItem(X, AxisStatusItem.DriveStatus, DriveStatus.Enabled)
var $isHomed as integer = StatusGetAxisItem(X, AxisStatusItem.AxisStatus, AxisStatus.Homed)
var $isDone as integer = StatusGetAxisItem(X, AxisStatusItem.AxisStatus, AxisStatus.MotionDone)
```

Common `AxisStatusItem` values: `PositionCommand`, `PositionFeedback`, `VelocityCommand`, `VelocityFeedback`, `CurrentCommand`, `CurrentFeedback`, `PositionError`, `AxisStatus`, `DriveStatus`, `AxisFault`, `ProgramPosition`, `AnalogInput0`-`AnalogInput3`, `DigitalInput0`-`DigitalInput3`.

### 9.2 Task Status

```aeroscript
var $state as real = StatusGetTaskItem($taskIndex, TaskStatusItem.TaskState)
var $lineNum as real = StatusGetTaskItem($taskIndex, TaskStatusItem.ExecutionLineNumber)
```

### 9.3 System Status

```aeroscript
var $uptime as real = StatusGetSystemItem(SystemStatusItem.Timer)
```

### 9.4 Fast Status (High-Rate Sampling)

```aeroscript
var $samples[100] as real
StatusGetAxisItemFast(X, AxisStatusItem.PositionCommandRaw, 0, 10, ref $samples)
// $sampleRate: 1, 10, 20, 50, or 100 (kHz)
```

---

## 10. Fault and Error Handling

### 10.1 Error Handler Registration

```aeroscript
onerror(MyErrorHandler())

function MyErrorHandler()
    var $msg as string = TaskGetErrorMessage(TaskGetIndex())
    MessageLogWrite("Error: " + $msg, MessageLogSeverity.Error)
    AcknowledgeAll()
    Dwell(0.1)
    ProgramRestart()
end
```

### 10.2 Fault Functions

```aeroscript
AcknowledgeAll()                           // Clear ALL axis faults and task errors
FaultAcknowledge($axis as axis)            // Clear faults on one axis
FaultAcknowledge($axes[] as axis)          // Clear faults on multiple axes
FaultThrow($axis, AxisFault.UserFault)     // Manually trigger a fault

TaskSetError($taskIndex, "Custom error")   // Set custom task error
TaskSetWarning($taskIndex, "Custom warn")  // Set custom warning
TaskClearError($taskIndex)                 // Clear task error
TaskClearWarning($taskIndex)               // Clear warning
```

### 10.3 AxisFault Enum (Bitmask)

Key fault values: `PositionErrorFault`, `OverCurrentFault`, `CwEndOfTravelLimitFault`, `CcwEndOfTravelLimitFault`, `CwSoftwareLimitFault`, `CcwSoftwareLimitFault`, `AmplifierFault`, `FeedbackFault`, `CommunicationLostFault`, `EmergencyStopFault`, `InternalFault`, `MotorTemperatureFault`, `AmplifierTemperatureFault`, `ExternalFault`, `UserFault`.

### 10.4 Robust Error-Recovery Pattern

```aeroscript
program
    onerror(OnError())
    TaskControlRestrict(TaskGetIndex())
    ParameterSetTaskValue(TaskGetIndex(), TaskParameter.TaskTerminationAxes, 0x0)

    Enable(X)
    Home(X)
    while true
        MoveLinear(X, 10.0, 5.0)
        WaitForInPosition(X)
        MoveLinear(X, 0.0, 5.0)
        WaitForInPosition(X)
    end
end

function OnError()
    AcknowledgeAll()
    TaskClearError(TaskGetIndex())
    Dwell(0.1)
    ProgramRestart()
end
```

---

## 11. Data Collection

Real-time deterministic data capture at up to 200 kHz.

```aeroscript
DataCollectionReset()

// Add signals to collect
DataCollectionAddSystemSignal(SystemDataSignal.DataCollectionSampleTime)
DataCollectionAddAxisSignal(X, AxisDataSignal.PositionCommand)
DataCollectionAddAxisSignal(X, AxisDataSignal.PositionFeedback)
DataCollectionAddAxisSignal(X, AxisDataSignal.CurrentFeedback)
DataCollectionAddAxisSignal(X, AxisDataSignal.AxisStatus)
DataCollectionAddTaskSignal(1, TaskDataSignal.TaskState)

// Start collection: filename, max samples, sample time in ms
DataCollectionStart("testData.dat", 5000, 1.0)

// ... do motion ...

DataCollectionStop()
```

**Sample rates:** 1.0 ms (1 kHz), 0.2 ms (5 kHz), 0.1 ms (10 kHz), 0.05 ms (20 kHz), 0.01 ms (100 kHz, PC only).

**Signal limits per axis:** 40 at 1 kHz, 8 at 5 kHz, 4 at 10 kHz, 2 at 20 kHz.

Output file is `.dat` format (ASCII, positions in counts, time in seconds), viewable in Automation1 Data Visualizer.

### Studio-Triggered Data Collection

```aeroscript
AppDataCollectionSnapshot()    // Tell Studio to capture data (non-real-time callback)
// ... do motion ...
AppDataCollectionStop()        // Tell Studio to stop capture
```

---

## 12. MachineApps Application Functions

These functions interact with MachineApps (Aerotech's operator interface builder). They require a connected MachineApps application with registered handlers.

### 12.1 Message Display

```aeroscript
AppMessageDisplay("Cycle complete")
AppMessageDisplay("Warning: temperature high", MessageSeverity.Warning)
AppMessageDisplayDismiss()
AppMessageDisplayDismiss(MessageSeverity.Warning)
```

### 12.2 Message Box (Custom Buttons)

```aeroscript
var $buttons[] as string = ["Continue", "Retry", "Abort"]
var $result as integer = AppMessageBox("Confirmation", "Proceed with operation?",
    $buttons, MessageSeverity.Information, true)
// $result: 0 = Continue, 1 = Retry, 2 = Abort, -1 = cancelled
```

### 12.3 Input Box

```aeroscript
var $userInput as string = AppMessageInputBox("Enter Value",
    "Enter the part number:", "12345", MessageSeverity.Information, true)
```

### 12.4 Menu Selection

```aeroscript
var $options[] as string = ["Option A", "Option B", "Option C"]
var $selected as integer = AppMessageMenu("Select Mode",
    "Choose operating mode:", $options, 0, MessageSeverity.Information, true)
```

### 12.5 File Dialogs

```aeroscript
var $filePath as string = AppMessageFileOpen("Select File", "C:\\Data",
    "", "Dat files|*.dat|All files|*.*", true)
```

### 12.6 Indicators

```aeroscript
AppIndicatorOn(0, "Running", IndicatorAnimationType.Flash,
    "#00FF00", "#FFFFFF", "#000000", "#FFFFFF", 500)
AppIndicatorOff(0)
```

### 12.7 Buttons

```aeroscript
AppButtonSetState(0, 1)           // By ID
AppButtonSetState("StartBtn", "Active")   // By name
```

### 12.8 MessageSeverity Enum

`Information` (1), `Warning` (2), `Error` (3).

---

## 13. Message Log

```aeroscript
MessageLogWrite("System initialized", MessageLogSeverity.Information)
MessageLogWrite("Retrying connection", MessageLogSeverity.Warning)
MessageLogWrite("Fatal error occurred", MessageLogSeverity.Error)
MessageLogWrite("Debug: $pos = " + RealToString($pos), MessageLogSeverity.Debug)
MessageLogClear()
```

`MessageLogSeverity`: `Debug` (0), `Information` (1), `Warning` (2), `Error` (3).

---

## 14. Drive Functions

### 14.1 Brake Control

```aeroscript
DriveBrakeOff($axis)    // Disengage brake (allow motion)
DriveBrakeOn($axis)     // Engage brake (prevent motion)
```

### 14.2 Drive Array (Low-Level Memory)

```aeroscript
var $data[100] as real
DriveArrayWrite($axis, $data, $startAddress, $numElements, DriveArrayType.PsoDistanceEventDistances)
DriveArrayRead($axis, $startAddress, ref $data, $numElements, DriveArrayType.DataCapturePositions)
```

### 14.3 Encoder Output

```aeroscript
DriveEncoderOutputConfigureInput($axis, EncoderOutputChannel.SyncPortA,
    EncoderInputChannel.PrimaryEncoder)
DriveEncoderOutputOn($axis, EncoderOutputChannel.SyncPortA)
DriveEncoderOutputOff($axis, EncoderOutputChannel.SyncPortA)
```

---

## 15. Key Rules and Pitfalls

1. **Check `*IsRegistered()` before calling host-dependent functions.** If no host application has registered for `AppMessageBox`, `AppIndicatorOn`, `Callback`, etc., the call will raise an error.

2. **File, socket, DateTime, and callback functions are non-real-time.** They pause the AeroScript program for an indeterminate time. Do not use them in tight real-time control loops or between velocity-blended moves.

3. **`ParameterSetAxisValue` changes are temporary.** They are lost on controller reset. Use Automation1 Studio to make persistent configuration changes.

4. **`ProgramRun` = `ProgramLoad` + `ProgramStart`.** Use `ProgramLoad` separately when you need to prepare a program without starting it (e.g., for coordinated multi-task startup).

5. **`onerror()` replaces any previous handler.** Only one error handler per task. The handler function should call `AcknowledgeAll()` or `FaultAcknowledge()` to clear faults before restarting.

6. **`TaskControlRestrict` prevents ALL external control** of the task, including from Studio. Use carefully -- if the program hangs, you cannot stop it from the IDE without restarting the controller.

7. **Critical sections prevent task scheduling.** Long critical sections starve other tasks. Keep them as short as possible. Non-real-time functions (file I/O, sockets) still cause pauses inside critical sections.

8. **Sockets are automatically closed** when the task stops or an error occurs. Always handle reconnection logic if using sockets in a long-running program.

9. **Data collection at high rates limits the number of signals per axis.** At 20 kHz, only 2 signals per axis are allowed. Plan your signal list carefully.

10. **Virtual I/O is the recommended way to communicate between tasks.** It is real-time, globally accessible, and does not require sockets or files. Use `VirtualRegisterInputSet/Get` for real values and `VirtualBinaryInputSet/Get` for flags.
