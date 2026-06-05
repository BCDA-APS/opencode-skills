---
name: aeroscript-motion
description: Write motion programs in AeroScript -- enable/home/disable, linear/rapid/arc moves, PVT/PT motion, coordinated and asynchronous moves, velocity blending, camming/gearing, transformations, and lookahead behavior
---

# AeroScript Motion Skill

You are an expert at writing AeroScript motion programs for Aerotech Automation1 controllers. You understand the full motion API including coordinated (synchronous) and independent (asynchronous) moves, velocity profiling, PVT motion, camming, gearing, transformations, and lookahead behavior.

---

## 1. Axis Initialization

```aeroscript
Enable($axis as axis)                  // Enable amplifier (apply torque)
Enable($axes[] as axis)                // Enable multiple axes
Home($axis as axis)                    // Synchronous home (task waits)
Home($axes[] as axis)                  // Home multiple axes
HomeAsync($axis as axis)               // Asynchronous home (task continues)
Disable($axis as axis)                 // Disable amplifier (remove torque)
Abort($axis as axis)                   // Emergency stop (ramps to zero)
```

```aeroscript
program
    var $axes[] as axis = [X, Y, Z]
    Enable($axes)
    Home($axes)
    // ... motion ...
    Disable($axes)
end
```

---

## 2. Independent (Asynchronous) Motion

Independent moves execute immediately and the program continues without waiting. Each axis moves at its own speed, independently of other axes.

### 2.1 Point-to-Point Moves

```aeroscript
// Absolute move -- target is an absolute position
MoveAbsolute($axis as axis, $position as real, $speed as real)
MoveAbsolute($axes[] as axis, $positions[] as real, $speeds[] as real)

// Incremental move -- target is relative to current position
MoveIncremental($axis as axis, $distance as real, $speed as real)
MoveIncremental($axes[] as axis, $distances[] as real, $speeds[] as real)
```

These moves are **asynchronous** -- the program continues immediately. Use `WaitForMotionDone()` or `WaitForInPosition()` to synchronize.

```aeroscript
MoveAbsolute([X, Y, Z], [10.0, 20.0, 5.0], [50.0, 50.0, 25.0])
WaitForInPosition([X, Y, Z])
```

### 2.2 Freerun (Constant Velocity)

```aeroscript
MoveFreerun($axis as axis, $velocity as real)        // Start freerun (sign = direction)
MoveFreerun($axes[] as axis, $velocities[] as real)
MoveFreerunStop($axis as axis)                        // Decelerate to zero
MoveFreerunStop($axes[] as axis)
```

```aeroscript
MoveFreerun(X, 100.0)     // Run forward at 100 units/sec
Dwell(2.0)                // Run for 2 seconds
MoveFreerun(X, -50.0)     // Reverse at 50 units/sec
Dwell(1.0)
MoveFreerunStop(X)        // Decelerate to stop
WaitForMotionDone(X)
```

---

## 3. Coordinated (Synchronous) Motion

Coordinated moves synchronize multiple axes so they start and stop together, following a defined path in vector space. The **coordinated speed** is the vector speed along the path, not the individual axis speed.

### 3.1 Linear Moves

```aeroscript
MoveLinear($axis as axis, $distance as real)
MoveLinear($axes[] as axis, $distances[] as real)
MoveLinear($axis as axis, $distance as real, $coordinatedSpeed as real)
MoveLinear($axes[] as axis, $distances[] as real, $coordinatedSpeed as real)
```

G-code equivalent: `G1`. Target mode (absolute/incremental) applies.

```aeroscript
SetupTaskTargetMode(TargetMode.Absolute)
SetupCoordinatedSpeed(100.0)       // Vector speed 100 units/sec
MoveLinear([X, Y], [10.0, 20.0])   // Move to (10, 20)
MoveLinear([X, Y], [30.0, 10.0])   // Move to (30, 10)
```

### 3.2 Rapid Moves

```aeroscript
MoveRapid($axis as axis, $distance as real)
MoveRapid($axes[] as axis, $distances[] as real)
MoveRapid($axis as axis, $distance as real, $speed as real)
MoveRapid($axes[] as axis, $distances[] as real, $speeds[] as real)
```

G-code equivalent: `G0`. Uses per-axis speeds (not coordinated vector speed). Program waits until all axes reach endpoints.

### 3.3 Circular Arc Moves

```aeroscript
// Clockwise arc (center point offsets, always incremental from start)
MoveCw($axes[2] as axis, $endpoints[2] as real, $center[2] as real)
// Clockwise arc (radius)
MoveCw($axes[2] as axis, $endpoints[2] as real, $radius as real)
// Counter-clockwise variants
MoveCcw($axes[2] as axis, $endpoints[2] as real, $center[2] as real)
MoveCcw($axes[2] as axis, $endpoints[2] as real, $radius as real)
```

G-code equivalents: `G2` (CW), `G3` (CCW). Endpoints obey the task target mode. Center offsets are always incremental from the start position. For a full circle, set endpoints equal to the start position.

```aeroscript
// Half circle with center point offset
MoveCw([X, Y], [10.0, 0.0], [5.0, 0.0])

// Full circle with radius
MoveCw([X, Y], [0.0, 0.0], 5.0)    // endpoints = start = full circle

// Counter-clockwise arc
MoveCcw([X, Y], [-10.0, 0.0], [-5.0, 0.0])
```

---

## 4. Dwell and Wait

```aeroscript
Dwell($seconds as real)              // Pause motion AND program execution
DelayMotion($seconds as real)        // Pause motion only (program continues)

WaitForMotionDone($axis as axis)     // Wait until velocity command = 0
WaitForMotionDone($axes[] as axis)
WaitForInPosition($axis as axis)     // Wait until velocity = 0 AND within InPositionDistance
WaitForInPosition($axes[] as axis)
```

`Dwell` causes deceleration to zero velocity in a velocity-blended sequence. `WaitForMotionDone` and `WaitForInPosition` only wait; they do not cause deceleration.

---

## 5. Motion Setup

### 5.1 Target Mode

```aeroscript
SetupTaskTargetMode(TargetMode.Absolute)      // G90
SetupTaskTargetMode(TargetMode.Incremental)    // G91
```

Absolute mode: positions relative to home. Incremental mode: positions relative to current. Does NOT apply to `MoveAbsolute()`, `MoveIncremental()`, or `MoveFreerun()` (these have explicit modes in their names).

### 5.2 Wait Mode

```aeroscript
SetupTaskWaitMode(WaitMode.InPosition)    // G361 -- wait for settle
SetupTaskWaitMode(WaitMode.MotionDone)    // G360 -- wait for velocity = 0
SetupTaskWaitMode(WaitMode.Auto)          // G359 -- minimize dead time between moves
```

### 5.3 Speed Configuration

```aeroscript
// Coordinated (vector) speed for MoveLinear, MoveCw, MoveCcw
SetupCoordinatedSpeed($speed as real)           // G-code: F command

// Dependent axis speed
SetupDependentCoordinatedSpeed($speed as real)  // G-code: E command

// Per-axis speed for MoveRapid only
SetupAxisSpeed($axis as axis, $speed as real)
```

### 5.4 Acceleration/Deceleration Ramping

```aeroscript
// Ramp type
SetupCoordinatedRampType(RampType.Linear)         // Constant acceleration
SetupCoordinatedRampType(RampType.Sine)            // Sinusoidal (low jerk)
SetupCoordinatedRampType(RampType.SCurve, 50.0)    // S-curve, 50% smoothing

// Separate accel/decel ramp types
SetupCoordinatedRampType(RampType.SCurve, 80.0, RampType.Linear, 0)

// Ramp value
SetupCoordinatedRampValue(RampMode.Rate, 1000.0)   // 1000 units/sec^2
SetupCoordinatedRampValue(RampMode.Time, 0.5)       // 0.5 sec ramp duration

// Per-axis ramping
SetupAxisRampType($axis, RampType.Linear)
SetupAxisRampValue($axis, RampMode.Rate, 2000.0)
```

**RampType enum:** `Linear` (0), `SCurve` (1), `Sine` (2).

**RampMode enum:** `Rate` (0) = units/sec^2 (constant acceleration), `Time` (1) = fixed duration (acceleration varies with speed). Rate-based ramping is required for velocity blending.

### 5.5 Distance and Time Units

```aeroscript
SetupTaskDistanceUnits(DistanceUnits.Primary)      // G70 (CountsPerUnit)
SetupTaskDistanceUnits(DistanceUnits.Secondary)     // G71 (CountsPerUnit / SecondaryUnitsScaleFactor)

SetupTaskTimeUnits(TimeUnits.Seconds)               // G75
SetupTaskTimeUnits(TimeUnits.Minutes)                // G76
```

---

## 6. PVT and PT Motion

Position-Velocity-Time motion specifies exact positions and velocities at each waypoint with cubic polynomial interpolation between them.

```aeroscript
MovePvt($axes[] as axis, $positions[] as real, $velocities[] as real, $time as real)
```

Target mode (absolute/incremental) applies to positions. Velocities are always in absolute units/sec.

```aeroscript
#define PI 3.14159265358979

program
    Enable(X)
    Home(X)
    SetupTaskTargetMode(TargetMode.Absolute)

    // Generate sinusoidal oscillation via PVT segments
    var $numSegments as integer = 100
    var $frequency as real = 10.0       // Hz
    var $amplitude as real = 5.0        // mm
    var $dt as real = 1.0 / ($frequency * $numSegments)

    CriticalSectionStart()
    for $i = 0 to $numSegments - 1
        var $t as real = $i * $dt
        var $pos as real = $amplitude * Sin(360.0 * $frequency * $t)
        var $vel as real = $amplitude * 2.0 * PI * $frequency * Cos(360.0 * $frequency * $t)
        MovePvt([X], [$pos], [$vel], $dt)
    end
    // Final segment to zero
    MovePvt([X], [0.0], [0.0], $dt)
    CriticalSectionEnd()

    WaitForInPosition(X)
    Disable(X)
end
```

---

## 7. Velocity Blending

Blends multiple coordinated moves (MoveLinear, MoveCw, MoveCcw) into a continuous path without decelerating to zero between moves.

```aeroscript
VelocityBlendingOn()                  // G108 -- enable
VelocityBlendingOff()                 // G109 -- disable
```

Activates lookahead. The controller looks ahead at upcoming moves to maintain the coordinated speed. Rate-based ramping (`RampMode.Rate`) is required.

```aeroscript
SetupCoordinatedRampType(RampType.SCurve, 50.0)
SetupCoordinatedRampValue(RampMode.Rate, 2000.0)
SetupCoordinatedSpeed(100.0)

VelocityBlendingOn()
MoveLinear([X, Y], [10, 0])
MoveLinear([X, Y], [10, 10])    // No deceleration between moves
MoveLinear([X, Y], [0, 10])
MoveLinear([X, Y], [0, 0])
VelocityBlendingOff()
```

**Causes of deceleration to zero velocity during blending:**
- MaxLookaheadMoves exceeded
- Change in dominant/dependent axes
- Change in ramp settings during blending
- Non-tangent acceleration, circular acceleration, or duration limits exceeded
- Breakpoints or step mode

---

## 8. Corner Rounding

Adds smooth arcs between consecutive non-tangent linear moves. Works with velocity blending for faster corner traversal.

```aeroscript
CornerRoundingSetAxes([X, Y])               // Up to 2 axes
CornerRoundingSetTolerance(0.01)            // Max deviation from path (mm)
CornerRoundingOn()
// ... linear moves ...
CornerRoundingOff()
```

The arc radius is calculated automatically from the tolerance and the angle between consecutive moves: `R = tolerance / (1 - cos(angle/2))`.

---

## 9. Camming and Gearing

### 9.1 Camming (Table-Based Synchronization)

```aeroscript
// Load cam table from arrays
CammingLoadTableFromArray($tableNum as integer, $leaderValues[] as real,
    $followerValues[] as real, $numValues as integer,
    $unitsMode as CammingUnits, $interpolationMode as CammingInterpolation,
    $wrapMode as CammingWrapping, $tableOffset as real)

// Activate camming
CammingOn($followerAxis as axis, $leaderAxis as axis, $tableNum as integer,
    $source as CammingSource, $output as CammingOutput)

// Deactivate
CammingOff($followerAxis as axis)
CammingFreeTable($tableNum as integer)
```

**CammingSource:** `PositionFeedback` (0), `PositionCommand` (1), `AuxiliaryFeedback` (2), `SyncPortA` (3), `SyncPortB` (4).

**CammingOutput:** `RelativePosition` (1), `AbsolutePosition` (2), `Velocity` (3).

**CammingInterpolation:** `Linear` (0), `Cubic` (1).

### 9.2 Gearing (Fixed-Ratio Synchronization)

```aeroscript
GearingSetLeaderAxis($followerAxis, $leaderAxis, GearingSource.PositionCommand)
GearingSetRatio($followerAxis, 2.0)      // 2:1 ratio (follower:leader)
GearingOn($followerAxis, GearingFilter.None)

// ... leader moves, follower follows at 2x ...

GearingOff($followerAxis)
```

---

## 10. Transformations

Matrix transformations apply rotation, mirroring, and translation to axis coordinates in real-time.

```aeroscript
// Create transformation matrices (returns handle)
var $mirror as handle = MatrixCreateMirror()
var $rotate as handle = MatrixCreateRotateK(45.0)    // 45-degree rotation about Z
var $translate as handle = MatrixCreateTranslate(10.0, 20.0)

// Configure transformation (index 0-31, up to 512 matrices per index)
TransformationConfigure(0, [$rotate, $mirror], [X, Y], [X, Y])

// Enable/disable
TransformationEnable(0)
// ... motion commands are now in transformed coordinates ...
TransformationDisable(0)

// Cleanup
MatrixDelete($rotate)
MatrixDelete($mirror)
MatrixDelete($translate)
```

Rotation matrices: `MatrixCreateRotateI` (about first axis), `MatrixCreateRotateJ` (about second), `MatrixCreateRotateK` (about third). The angle argument can be a `real` constant or an `axis` for dynamic rotation driven by an axis position.

---

## 11. Motion Restriction

Prevents motion on specified axes from all sources (all tasks, Studio, APIs).

```aeroscript
MotionRestrictionOn([X, Y])   // Block all motion/enable/disable/home/abort
MotionRestrictionOn([X], [MotionRestrictionType.Motion])  // Block motion only

// Allow section (temporarily lift restriction)
MotionRestrictionAllowSectionStart()
MoveLinear(X, 10.0, 5.0)     // This move is allowed
MotionRestrictionAllowSectionEnd()

MotionRestrictionOff([X, Y])
```

**WARNING:** Not for safety-critical use. Use hardware ESTOP or STO inputs instead.

---

## 12. Lookahead Behavior

When velocity blending, corner rounding, or cutter compensation is active, the controller uses lookahead to plan future moves. Functions are classified by their effect on lookahead:

### Functions that PASS THROUGH Lookahead (no deceleration)

- `MoveLinear`, `MoveCw`, `MoveCcw`, `MoveRapid`, `MovePvt`
- `SetupCoordinatedSpeed`, `SetupCoordinatedRampType`, `SetupCoordinatedRampValue`
- `SetupTaskTargetMode`, `SetupTaskWaitMode`

### Functions that BLOCK Lookahead (cause deceleration to zero)

- `Enable`, `Disable`, `Home`, `Abort`
- `MoveAbsolute`, `MoveIncremental`, `MoveFreerun`, `MoveFreerunStop`
- `WaitForMotionDone`, `WaitForInPosition`, `Dwell`
- All I/O functions, parameter changes, status queries
- All PSO functions
- `CammingOn/Off`, `GearingOn/Off`
- `VelocityBlendingOff`, `CornerRoundingOff`
- All file, socket, and callback functions

**Key implication:** Do not insert I/O reads, status queries, or print statements between velocity-blended moves -- they will cause the path to decelerate to zero.

---

## 13. Complete Example: Coordinated Motion with Blending

```aeroscript
program
    var $axes[] as axis = [X, Y]

    Enable($axes)
    Home($axes)

    // Configure ramping
    SetupCoordinatedRampType(RampType.SCurve, 50.0)
    SetupCoordinatedRampValue(RampMode.Rate, 2000.0)
    SetupCoordinatedSpeed(100.0)
    SetupTaskTargetMode(TargetMode.Absolute)

    // Square path with velocity blending and corner rounding
    VelocityBlendingOn()
    CornerRoundingSetAxes($axes)
    CornerRoundingSetTolerance(0.05)
    CornerRoundingOn()

    repeat 5
        MoveLinear($axes, [50, 0])
        MoveLinear($axes, [50, 50])
        MoveLinear($axes, [0, 50])
        MoveLinear($axes, [0, 0])
    end

    CornerRoundingOff()
    VelocityBlendingOff()

    WaitForInPosition($axes)
    Disable($axes)
end
```

---

## 14. Key Rules and Pitfalls

1. **Coordinated speed is vector speed**, not per-axis speed. A `MoveLinear([X,Y], [30,40], 50)` moves at 50 units/sec along the diagonal, not 50 on each axis.

2. **`MoveAbsolute`/`MoveIncremental` are asynchronous** -- the program continues immediately. Use `WaitForMotionDone()` or `WaitForInPosition()` to synchronize.

3. **`MoveLinear`/`MoveCw`/`MoveCcw` are synchronous** by default -- the task waits for completion. The wait mode (`SetupTaskWaitMode`) controls what "completion" means.

4. **Arc center offsets are always incremental** from the start position, even in absolute target mode. Only the endpoints obey the target mode.

5. **Velocity blending requires rate-based ramping** (`RampMode.Rate`). Time-based ramping causes an error.

6. **Do not insert blocking calls between blended moves.** Any I/O, status query, parameter change, or print statement between velocity-blended moves will cause deceleration to zero.

7. **`Dwell` causes deceleration** in a blended sequence. Use it intentionally as a "stop point." Use `DelayMotion` if you want a pause without affecting the motion profile.

8. **Camming/gearing feedback sources have ~10ms latency** except `PositionCommand` (no latency). Use `PositionCommand` for tight synchronization.

9. **PVT segments should be enclosed in `CriticalSectionStart()`/`CriticalSectionEnd()`** to prevent the task scheduler from inserting gaps between segments.

10. **`Abort` stops motion immediately** and halts program execution. It does not return to the calling code. Use `MoveFreerunStop` + `WaitForMotionDone` for a controlled stop that allows the program to continue.
