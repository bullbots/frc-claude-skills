---
name: frc-review
description: Reviews First Robotics Competition (FRC) robot codebases for safety issues, bugs, and competition-readiness problems. Use this skill whenever someone asks to review, audit, check, or analyze FRC robot code — even if they just say "look at my robot code", "find bugs in my code", or "is this ready for competition". Also trigger for any mention of WPILib, command-based programming, subsystems, motor controllers, autonomous routines, RobotPy, PathPlanner, AdvantageKit, or CTRE/REV hardware. If the user is on an FRC team and asking for a code review of any kind, use this skill.
---

# FRC Codebase Review

You are performing a focused, practical code review of a First Robotics Competition robot codebase. Your goal is to surface real issues that affect safety, competition reliability, and code correctness — not to rewrite the code or give generic advice.

## Step 1: Orient yourself

Identify what you're working with before diving in:

- **Language/framework**: Java command-based (most common), C++ WPILib, or Python RobotPy?
- **Vendor libraries in use**: CTRE Phoenix 5 or 6? REV SparkMAX/SparkFlex? navX? Limelight?
- **Drive type**: swerve (YAGSL, SwerveLib, custom) or differential (tank/arcade)? Check imports and subsystem class names.
- **Relevant entry points**: `Robot.java` / `RobotContainer.java` for command-based; `robot.py` for RobotPy
- **Build file**: `build.gradle` or `build.gradle.kts` — check WPILib and vendor dep versions
- **Deploy directory**: `src/main/deploy/` — look for `swerve/` JSON configs (YAGSL), PathPlanner paths, Choreo trajectories

Use Glob to find all source files (`**/*.java`, `**/*.cpp`, `**/*.h`, `**/*.py`). Read the most important ones (RobotContainer, subsystems, commands, Robot.java) carefully. Skim supporting files quickly.

## Step 2: Check for issues by category

Work through each category. For every issue you find, record:
- File path and line number
- What the problem is
- Why it matters in an FRC context
- A concrete fix

Only flag things you actually found. Do not speculate about code you haven't read.

---

### SWERVE-SPECIFIC checks (skip if differential drive)

If the codebase uses swerve (YAGSL, SwerveLib, or custom), run these in addition to the general checks below.

**YAGSL configuration**
- `src/main/deploy/swerve/` directory missing or empty — YAGSL reads JSON configs at runtime; if they're absent the robot won't initialize
- Module JSON files (`frontleft.json`, `frontright.json`, etc.) have mismatched CAN IDs or wrong motor/encoder types — this is the #1 source of a swerve module spinning wrong or not at all
- Absolute encoder offsets not set (all zeros) — wheels won't home to straight, robot will crab immediately on enable
- Drive/steer motor inversion wrong for module position — check each module's physical wiring against its JSON config

**Swerve drive code**
- `driveFieldOriented()` vs `drive()` confusion — field-relative requires the gyro heading; robot-relative does not. Using the wrong one causes the robot to drive relative to its own facing instead of the field
- Gyro not zeroed at the start of autonomous when field-relative driving is used — robot will drive in the wrong direction every auto run
- Deadband missing on all three swerve axes independently: translation X, translation Y, and rotation — even a small resting joystick value causes continuous wheel steering adjustments and motor heat
- `SwerveDriveKinematics` module positions don't match the physical robot layout — causes odometry drift and incorrect autonomous paths
- No X-lock (wheels pointed inward) when stopped defensively — robot gets pushed easily

**PathPlanner + swerve**
- `AutoBuilder.configureHolonomic()` not called, or called with wrong parameters — swerve autos require holonomic path follower, not the differential `configureRamsete()`
- Translation and rotation PID constants in `AutoBuilder.configureHolonomic()` are all zero — robot will not follow paths accurately
- Robot width/length used in `SwerveDriveKinematics` is wrong (measured to wheel center, not bumper edge) — path following will overshoot

---

### CRITICAL — safety or match reliability at stake

**Motor runaway / no safety limits**
- Motors driving an arm, elevator, or shooter with no soft limits and no limit switch handling
- `setSafetyEnabled(false)` without a comment explaining why it's intentional
- Current limits absent on NEOs (`setSmartCurrentLimit`), Falcons/Krakens (`CurrentLimitsConfigs`), or VictorSPX/TalonSRX (`configContinuousCurrentLimit`)
- Missing stall detection on motors that could burn out holding a position

**Commands that break the scheduler**
- `isFinished()` hardcoded to `return false` on a command that is used in a sequential group — this blocks the whole autonomous routine
- Commands in `SequentialCommandGroup` / `andThen()` chains that never terminate
- Missing `addRequirements()`: two commands can fight over the same subsystem silently
- `CommandScheduler.getInstance().run()` called multiple times in `robotPeriodic()`, or missing entirely in a `TimedRobot`

**Autonomous**
- `Thread.sleep()` or `wait()` called inside any `periodic()`, `execute()`, or command method — this freezes the robot loop
- Blocking while-loops inside commands instead of state machines
- Drive not explicitly stopped at the end of auto (robot coasts into field elements)
- PathPlanner/Choreo path files referenced in code but not present in `src/main/deploy/`
- Gyro not reset at auto start when field-relative driving is used

**Sensor configuration**
- Encoder with no conversion factor set — `getPosition()` returns raw ticks/rotations when the code expects meters or degrees
- Encoder direction not set, causing PID to drive to negative infinity
- navX not initialized before use; no check for calibration complete

---

### WARNINGS — likely to cause problems at competition

**Competition-day performance**
- `System.out.println()` / `printf` / Python `print()` inside any `periodic()` or `execute()` method — runs every 20ms, floods logs, can cause loop overruns
- More than ~8–10 `SmartDashboard.putX()` / `Shuffleboard.getTab()...withWidget()` calls per periodic cycle — networktables writes have overhead
- `DataLogManager` or `SignalLogger` not initialized — no logs to review after a match-ending fault
- `Robot.java` or `RobotContainer` constructor doing slow initialization (camera streams, long sleeps, firmware version checks) that could cause the DS to report "Robot code not started"

**Motor controller configuration**
- No brake/coast mode explicitly set — default varies by controller and season; assume it needs to be explicit
- No ramp rate / open-loop ramp on drivetrain motors — aggressive throttle causes voltage sag and brownouts
- PID gains of exactly `0.0` for kP on a position/velocity controller — likely a placeholder that was never tuned
- Drive motor inversion inconsistent: one side inverted at the controller, the other in software (causes double-inversion bug)

**Joystick / driver controls**
- No deadband on any joystick axis used for drive — even a resting stick sends ~0.02–0.05, causing constant creep and motor heat
- `getX()` / `getY()` axis directions not verified against WPILib coordinate convention (Y is inverted on most controllers)
- **Duplicate button bindings**: the same button or trigger bound to two different commands/command groups in RobotContainer. WPILib does not warn you — the second `.onTrue()` / `.whileTrue()` call silently overwrites the first, so one command will never trigger. Scan every `JoystickButton`, `Trigger`, and controller shorthand (`.a()`, `.b()`, etc.) and verify each port+button combination appears only once. This is a competition killer because it only shows up when the operator presses the button and nothing happens.

**Subsystem design**
- Heavy computation (path generation, vision processing, inverse kinematics) inside `periodic()` rather than offloaded to a command or background thread
- Subsystem state tracked as raw `boolean`/`double` fields instead of an enum — makes it easy to have undefined states
- No null check on optional hardware (limit switch, camera, secondary sensor) that might not be physically present

---

### SUGGESTIONS — good practice, worth noting

- Vendor library or WPILib version appears outdated for the current season (check build.gradle)
- Mixing unit systems (rotations, radians, degrees) without explicit conversion — comment or use `Units.degreesToRadians()` helpers
- No structured logging (AdvantageKit, Monologue, DataLog) — post-match debugging is much harder without it
- `getSelectedSensorPosition()` used with a Phoenix 6 device — that's the Phoenix 5 API and will silently return 0
- Commands written as inline lambdas rather than named classes — fine for simple cases, but complex logic is hard to debug
- `withTimeout()` missing on commands that poll a sensor and could hang if the sensor fails
- `CommandScheduler.getInstance().onCommandInterrupt()` not set — missed opportunity to log unexpected command cancellations

---

## Step 3: Write the report

Use this exact structure. Omit any section that has zero findings — don't write "None found."

```
# FRC Code Review: [Project Name or directory]

**Framework:** [e.g., Command-Based Java with WPILib 2025.x]
**Vendor libs detected:** [e.g., CTRE Phoenix 6, REV SparkMAX, navX]
**Files reviewed:** [count]

---

## Critical Issues

> These must be fixed. Robot safety or match reliability is directly at risk.

### [Issue title]
**File:** `SubsystemName.java:42`
**Problem:** [One sentence on what's wrong]
**Impact:** [What breaks or how the robot behaves badly]
**Fix:** [Specific code change, e.g., "Add `motor.setSmartCurrentLimit(40)` after instantiation"]

---

## Warnings

> Likely to cause problems at competition. Fix before your first event.

### [Issue title]
**File:** `...`
...

---

## Suggestions

> Code quality and long-term improvements.

- **[File:line]** [One-line description of the suggestion]

---

## Summary

| Category   | Count |
|------------|-------|
| Critical   | N     |
| Warnings   | N     |
| Suggestions| N     |

**Overall assessment:** [2–3 sentences. What is the team's biggest risk? What are they doing well? What should they tackle first?]
```

Keep the report honest and specific. If a team has zero critical issues, say so clearly — that's good news. Prioritize issues by how likely they are to cause a match loss or a safety incident, not by how interesting they are to discuss.
