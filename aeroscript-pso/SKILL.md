---
name: aeroscript-pso
description: Write Position-Synchronized Output (PSO) programs in AeroScript for laser firing, marking, and encoder-triggered events -- distance/event/output/window/waveform/transformation modes, fixed/array/continuous outputs, and part-speed PSO
---

# AeroScript PSO Skill

You are an expert at writing Position-Synchronized Output (PSO) programs in AeroScript for Aerotech Automation1 controllers. PSO is a hardware-based subsystem on the drive that generates precisely-timed digital outputs synchronized to axis position. It is used for laser firing, marking, scanning, encoder-triggered data capture, and other position-synchronized operations.

---

## 1. PSO Architecture

The PSO subsystem consists of modular, composable hardware blocks:

```
Encoder Feedback --> [Transformation] --> [Distance] --> [Event] --> [Waveform] --> [Output] --> Physical Pin
                                     \-> [Window] --------/--mask--/--mask-----/--source--/
                                          [Bit] ---------/--mask--/----------/--source--/
                                          [Laser] -------/
```

| Module | Purpose |
|--------|---------|
| **Distance** | Generates events at specified travel intervals (1D, 2D, or 3D) |
| **Window** | Tracks absolute position ranges; output active when inside window |
| **Event** | Combines events from Distance/Window/Laser/manual; applies masks |
| **Waveform** | Generates pulse trains, PWM, or toggle on each event |
| **Output** | Routes the selected signal (Waveform, Window, Bitmap) to physical pin |
| **Transformation** | Math operations (sum, difference, average) on two feedback inputs |
| **Bit/Bitmap** | Sequence of binary values consumed per event (for masking or output) |

---

## 2. Reset

Always reset PSO before configuring:

```aeroscript
PsoReset($axis as axis)
```

Restores all PSO settings to defaults. Must be called before any new PSO configuration.

---

## 3. Distance Module

### 3.1 Configure Tracking Inputs

```aeroscript
PsoDistanceConfigureInputs($axis as axis, $inputs[] as PsoDistanceInput)
```

The number of inputs determines the mode:
- 1 input = 1D (events in either direction along one axis)
- 2 inputs = 2D (events based on vector distance in 2 axes)
- 3 inputs = 3D (events based on vector distance in 3 axes)

```aeroscript
// 1D: track X primary feedback
PsoDistanceConfigureInputs(X, [PsoDistanceInput.XC4PrimaryFeedback])

// 2D: track X and Y
PsoDistanceConfigureInputs(X, [PsoDistanceInput.XC4PrimaryFeedback,
                                PsoDistanceInput.XC4SyncPortA])
```

Input sources are drive-specific. Common values:
- `XC4PrimaryFeedback`, `XC4AuxiliaryFeedback`, `XC4SyncPortA`, `XC4SyncPortB`
- `XC4ePrimaryFeedback`, `XC4eAuxiliaryFeedback`, `XC4eSyncPortA`, `XC4eSyncPortB`
- `GL4PrimaryFeedbackAxis1Encoder0`, `GL4PrimaryFeedbackAxis2Encoder0`
- `GL4DrivePulseStreamAxis1`, `GL4DrivePulseStreamAxis2` (for part-speed PSO)

### 3.2 Fixed Distance Events

```aeroscript
PsoDistanceConfigureFixedDistance($axis as axis, $distance as integer)
```

The distance is in **emulated quadrature counts** (not user units). Convert with:

```aeroscript
var $distCounts as integer = Round(UnitsToCounts($axis, $userDistance) /
    ParameterGetAxisValue($axis, AxisParameter.PrimaryEmulatedQuadratureDivider))
PsoDistanceConfigureFixedDistance($axis, $distCounts)
```

### 3.3 Array Distance Events (Variable Spacing)

```aeroscript
PsoDistanceConfigureArrayDistances($axis as axis, $driveArrayStartAddress as integer,
    $numberOfDistances as integer, $enableRepeat as integer)
```

Load distances into the drive array first:

```aeroscript
var $distances[3] as real
$distances[0] = UnitsToCounts(X, 5.0) / ParameterGetAxisValue(X, AxisParameter.PrimaryEmulatedQuadratureDivider)
$distances[1] = UnitsToCounts(X, 10.0) / ParameterGetAxisValue(X, AxisParameter.PrimaryEmulatedQuadratureDivider)
$distances[2] = UnitsToCounts(X, 15.0) / ParameterGetAxisValue(X, AxisParameter.PrimaryEmulatedQuadratureDivider)
DriveArrayWrite(X, $distances, 0, 3, DriveArrayType.PsoDistanceEventDistances)
PsoDistanceConfigureArrayDistances(X, 0, 3, false)
```

### 3.4 Distance Counter and Event Control

```aeroscript
PsoDistanceCounterOn($axis)     // Start tracking position
PsoDistanceCounterOff($axis)    // Freeze counters (hold values)
PsoDistanceEventsOn($axis)      // Allow events to reach Event module
PsoDistanceEventsOff($axis)     // Block events
```

### 3.5 Distance Scaling

```aeroscript
PsoDistanceConfigureScaling($axis as axis, $scaleFactors[] as integer)
```

Apply integer divider per tracking input (useful when feedback resolutions differ between axes).

### 3.6 Event Direction

```aeroscript
PsoDistanceConfigureAllowedEventDirection($axis, PsoDistanceAllowedEventDirection.Both)
PsoDistanceConfigureAllowedEventDirection($axis, PsoDistanceAllowedEventDirection.Positive)
PsoDistanceConfigureAllowedEventDirection($axis, PsoDistanceAllowedEventDirection.Negative)
```

### 3.7 Counter Reset Conditions

```aeroscript
PsoDistanceConfigureCounterReset($axis, PsoDistanceCounterResetMask.ResetOutsideWindow)
```

Mask values (OR together): `ResetUntilMarker` (1), `ResetOnMarker` (2), `ResetOutsideWindow` (4), `ResetWhenLaserOff` (8), `ResetWhenOutputOff` (16).

---

## 4. Window Module

Windows define absolute position ranges. The output is active when the position counter is within the range.

### 4.1 Configure Window Input

```aeroscript
PsoWindowConfigureInput($axis as axis, $windowNumber as integer,
    $input as PsoWindowInput, $reverseDirection as integer)
```

Two windows are available (0 and 1). Combined output requires BOTH windows active.

### 4.2 Fixed Range

```aeroscript
PsoWindowConfigureFixedRange($axis, $windowNumber,
    $lowerBound as integer, $upperBound as integer)
```

Bounds are in emulated quadrature counts:

```aeroscript
var $lower as integer = Round(UnitsToCounts(X, 5.0) /
    ParameterGetAxisValue(X, AxisParameter.PrimaryEmulatedQuadratureDivider))
var $upper as integer = Round(UnitsToCounts(X, 10.0) /
    ParameterGetAxisValue(X, AxisParameter.PrimaryEmulatedQuadratureDivider))
PsoWindowConfigureFixedRange(X, 0, $lower, $upper)
```

### 4.3 Array Ranges (Variable Windows)

```aeroscript
PsoWindowConfigureArrayRanges($axis, $windowNumber,
    $driveArrayStartAddress, $numberOfRanges, $enableRepeat)
```

Write pairs of (lower, upper) bounds with `DriveArrayType.PsoWindowRanges`. The next range loads automatically when the counter exits the current range.

### 4.4 Window Output

```aeroscript
PsoWindowOutputOn($axis, $windowNumber)
PsoWindowOutputOff($axis, $windowNumber)
```

### 4.5 Window Events

```aeroscript
PsoWindowConfigureEvents($axis, PsoWindowEventMode.Enter)    // Event on entering window
PsoWindowConfigureEvents($axis, PsoWindowEventMode.Exit)     // Event on exiting
PsoWindowConfigureEvents($axis, PsoWindowEventMode.Both)     // Both
PsoWindowConfigureEvents($axis, PsoWindowEventMode.None)     // No events
```

---

## 5. Event Module

Receives events from Distance, Window, Laser, and manual sources. Applies masking.

### 5.1 Event Mask

```aeroscript
PsoEventConfigureMask($axis, $mask as integer)
```

Mask values (OR together):
- `PsoEventMask.WindowMask` (1) -- require window output active
- `PsoEventMask.WindowMaskInvert` (2) -- require window output inactive
- `PsoEventMask.LaserMask` (4) -- require laser active
- `PsoEventMask.BitMask` (8) -- require bitmap bit active

```aeroscript
// Only fire when inside window AND laser is active
PsoEventConfigureMask(X, PsoEventMask.WindowMask | PsoEventMask.LaserMask)
```

### 5.2 Manual Events

```aeroscript
PsoEventGenerateSingle($axis)     // Fire one event immediately
PsoEventContinuousOn($axis)       // Start continuous events (waveform fires ASAP)
PsoEventContinuousOff($axis)      // Stop continuous events
```

---

## 6. Waveform Module

Generates output waveforms in response to events.

### 6.1 Mode Selection

```aeroscript
PsoWaveformConfigureMode($axis, PsoWaveformMode.Pulse)    // Configurable pulse train
PsoWaveformConfigureMode($axis, PsoWaveformMode.Pwm)      // PWM (duty cycle per event)
PsoWaveformConfigureMode($axis, PsoWaveformMode.Toggle)    // Toggle on each event
```

### 6.2 Enable/Disable

```aeroscript
PsoWaveformOn($axis)      // Events trigger waveform output
PsoWaveformOff($axis)     // Events ignored by waveform
```

### 6.3 Pulse Mode (Most Common)

Configure three parameters, then apply:

```aeroscript
PsoWaveformConfigurePulseFixedTotalTime($axis, $totalTime as real)    // Period in microseconds
PsoWaveformConfigurePulseFixedOnTime($axis, $onTime as real)          // Pulse width in microseconds
PsoWaveformConfigurePulseFixedCount($axis, $count as integer)         // Pulses per event
PsoWaveformApplyPulseConfiguration($axis)                              // MUST call after configuring
```

Time range: 0.0 to 42,949,672.95 us (resolution 0.01 us). Count range: 1 to 4,294,967,295.

```aeroscript
// 3 pulses per event, 50% duty cycle, 20ms period
PsoWaveformConfigurePulseFixedTotalTime(X, 20000)    // 20ms total
PsoWaveformConfigurePulseFixedOnTime(X, 10000)       // 10ms on
PsoWaveformConfigurePulseFixedCount(X, 3)            // 3 pulses
PsoWaveformApplyPulseConfiguration(X)
```

### 6.4 Array-Based Pulse Parameters

For variable pulse timing per event:

```aeroscript
PsoWaveformConfigurePulseArrayTotalTimes($axis, $driveArrayAddr, $numPoints, $enableRepeat)
PsoWaveformConfigurePulseArrayOnTimes($axis, $driveArrayAddr, $numPoints, $enableRepeat)
PsoWaveformConfigurePulseArrayCounts($axis, $driveArrayAddr, $numPoints, $enableRepeat)
```

Write data with `DriveArrayType.PsoPulseTimes` or `DriveArrayType.PsoPulseCounts`.

### 6.5 Pulse Event Queue

```aeroscript
PsoWaveformConfigurePulseEventQueue($axis, $maxQueuedEvents as integer)
```

Valid range: 0-15. Events arriving during an active pulse are queued and replayed immediately after.

### 6.6 Pulse Delay

```aeroscript
PsoWaveformConfigureDelay($axis, $delayMicroseconds as real)
```

Delay between event and waveform start.

### 6.7 PWM Mode

Fixed period, variable duty cycle from drive array:

```aeroscript
PsoWaveformConfigurePwmTotalTime($axis, $totalTime as real)
PsoWaveformConfigurePwmArrayOnTimes($axis, $driveArrayAddr, $numPoints, $enableRepeat)
PsoWaveformApplyPwmConfiguration($axis)
```

### 6.8 Waveform Scaling (Speed-Dependent)

Scale pulse timing based on axis velocity or analog input:

```aeroscript
PsoWaveformScalingConfigure($axis, PsoWaveformScalingMode.ScaleTotalTimeAndOnTime,
    PsoWaveformScalingInput.DrivePulseStreamVelocity,
    [$minSpeed, $maxSpeed], [$minScale, $maxScale])
PsoWaveformScalingOn($axis)
```

---

## 7. Output Module

### 7.1 Output Source

```aeroscript
PsoOutputConfigureSource($axis, PsoOutputSource.Waveform)         // Waveform module drives output
PsoOutputConfigureSource($axis, PsoOutputSource.WindowOutput)     // Window state drives output
PsoOutputConfigureSource($axis, PsoOutputSource.WindowOutputInvert)
PsoOutputConfigureSource($axis, PsoOutputSource.Bitmap)           // Bitmap active bit
```

### 7.2 Output Pin Selection

```aeroscript
PsoOutputConfigureOutput($axis, PsoOutputPin.XC4DedicatedOutput)
```

Pin values are drive-specific (XC4, XC4e, GL4, XL4s, XR3, XC6e variants).

### 7.3 Manual Override

```aeroscript
PsoOutputOn($axis)     // Force output active (override signal)
PsoOutputOff($axis)    // Force output inactive (override signal)
```

After an override, call `PsoOutputConfigureSource()` to resume signal-driven output.

---

## 8. Transformation Module

Performs math on two feedback inputs before they reach Distance or Window modules.

```aeroscript
PsoTransformationConfigure($axis, $channel as integer,
    $inputA as PsoTransformationInput, $inputB as PsoTransformationInput,
    $function as PsoTransformationFunction)
PsoTransformationOn($axis, $channel)
PsoTransformationOff($axis, $channel)
```

Functions: `None` (pass-through), `Average` ((A+B)/2), `Sum` (A+B), `Difference` (A-B).

Up to 4 transformation channels. Both inputs must have the same resolution.

---

## 9. Bitmap Module

Defines a sequence of binary values consumed per event:

```aeroscript
PsoBitmapConfigureArray($axis, $driveArrayStartAddress, $numberOfPoints, $enableRepeat)
```

Each 32-bit word provides 32 bits, consumed MSB-first. Write with `DriveArrayType.PsoBitmapBits`. Used as an event mask (`PsoEventMask.BitMask`) or as a direct output source (`PsoOutputSource.Bitmap`).

---

## 10. Part-Speed PSO

For axes without linear encoder feedback (e.g., rotary stages with nonlinear mapping), part-speed PSO provides a virtual encoder via pulse streaming.

```aeroscript
// Configure pulse stream: output on X, driven by X and Y position
DrivePulseStreamConfigure(X, [X, Y], [1.0, 1.0])
DrivePulseStreamOn(X)

// Then configure PSO to use the pulse stream as input
PsoDistanceConfigureInputs(X, [PsoDistanceInput.GL4DrivePulseStreamAxis1])
```

Max output rate: 95 MHz. MaxSpeed = 95,000,000 / CountsPerUnit.

---

## 11. Unit Conversion

PSO distances and window bounds are in **emulated quadrature counts**, not user units. Always convert:

```aeroscript
function PsoUserToCount($axis as axis, $userUnits as real) as integer
    return Round(UnitsToCounts($axis, $userUnits) /
        ParameterGetAxisValue($axis, AxisParameter.PrimaryEmulatedQuadratureDivider))
end
```

---

## 12. Complete Examples

### 12.1 Fixed-Distance Pulse Output

```aeroscript
program
    var $axis as axis = X
    SetupTaskTargetMode(TargetMode.Incremental)
    Enable($axis)
    Home($axis)

    PsoReset($axis)
    PsoDistanceConfigureInputs($axis, [PsoDistanceInput.XC4PrimaryFeedback])
    PsoDistanceConfigureFixedDistance($axis,
        Round(UnitsToCounts($axis, 10) /
        ParameterGetAxisValue($axis, AxisParameter.PrimaryEmulatedQuadratureDivider)))
    PsoDistanceCounterOn($axis)
    PsoDistanceEventsOn($axis)

    PsoWaveformConfigureMode($axis, PsoWaveformMode.Pulse)
    PsoWaveformConfigurePulseFixedTotalTime($axis, 20000)
    PsoWaveformConfigurePulseFixedOnTime($axis, 10000)
    PsoWaveformConfigurePulseFixedCount($axis, 3)
    PsoWaveformApplyPulseConfiguration($axis)
    PsoWaveformOn($axis)

    PsoOutputConfigureSource($axis, PsoOutputSource.Waveform)

    MoveLinear($axis, 35)
    Disable($axis)
end
```

### 12.2 Window Output (Position-Gated)

```aeroscript
program
    var $axis as axis = X
    SetupTaskTargetMode(TargetMode.Absolute)
    Enable($axis)
    Home($axis)

    PsoReset($axis)
    PsoWindowConfigureInput($axis, 0, PsoWindowInput.XC4PrimaryFeedback, false)
    PsoWindowConfigureFixedRange($axis, 0,
        Round(UnitsToCounts($axis, 5) /
            ParameterGetAxisValue($axis, AxisParameter.PrimaryEmulatedQuadratureDivider)),
        Round(UnitsToCounts($axis, 10) /
            ParameterGetAxisValue($axis, AxisParameter.PrimaryEmulatedQuadratureDivider)))
    PsoWindowOutputOn($axis, 0)
    PsoOutputConfigureSource($axis, PsoOutputSource.WindowOutput)

    MoveLinear($axis, 20)     // Output ON between 5-10 user units
    Disable($axis)
end
```

---

## 13. Key Rules and Pitfalls

1. **Always call `PsoReset()` before configuring.** Residual state from a previous configuration can cause unexpected behavior.

2. **Always call `PsoWaveformApplyPulseConfiguration()` after setting pulse parameters.** The parameters are staged in software and only transferred to hardware on `Apply`.

3. **Distances and window bounds are in emulated quadrature counts**, not user units. Always convert using `UnitsToCounts()` divided by `PrimaryEmulatedQuadratureDivider`.

4. **`PsoDistanceCounterOn()` and `PsoDistanceEventsOn()` are separate.** Counters can track position without generating events (useful for priming). Events can be enabled/disabled independently of counter tracking.

5. **PSO functions block lookahead.** Do not configure PSO between velocity-blended moves. Configure before starting the blended sequence.

6. **PSO stops when motion aborts** (unless in `DataCollectionSyncPulse` mode). This is a safety feature.

7. **The bitmap advances only on unmasked events.** If an event is blocked by a window mask or other condition, the bitmap does NOT advance to the next bit.

8. **`PsoOutputOn()`/`PsoOutputOff()` are overrides.** After using them, you must call `PsoOutputConfigureSource()` to return to signal-driven output.

9. **2D/3D PSO uses vector distance** (Euclidean sum-of-squares), not per-axis distance. Events fire when the total travel in the vector space reaches the configured distance.

10. **Latency is 40-50 ns** for distance events (1D: 40ns, 2D/3D: 50ns). For applications requiring sub-microsecond synchronization, this is the hardware limit.
