# lerobot-arm

Leader + follower **SO-100** robot arm pair, driven by Hugging Face
[LeRobot](https://github.com/huggingface/lerobot), with an FSR force sensor
added to the follower gripper for grip-strength measurement.

> **Variant: SO-100, not SO-101.** Every file here targets the SO-100. LeRobot
> robot types are `so100_follower` / `so100_leader`. Upstream marks SO-100 as
> deprecated in favour of the SO-101 — that is a deliberate choice here, taken
> because the SO-100 ships as two clean single-file assemblies. Do not mix in
> SO-101 parts, commands, or docs; the geometry, the leader gearing, and the
> assembly order all differ.

## Layout

```
lerobot-arm/
├── cad/                  STEP assemblies (leader + follower)   -> cad/README.md
├── electrical/           servo driver board reference          -> electrical/README.md
├── simulation/           URDF + meshes for visualisation       -> simulation/README.md
└── software/
    └── required_libraries.md    install, bring-up, training, FSR
```

Start with [`software/required_libraries.md`](software/required_libraries.md).
It is the runnable path from a bare DGX to a teleoperating arm pair.

## Hardware

| Item | Qty | Notes |
|---|:--:|---|
| STS3215 servo, 7.4 V | 12 | 6 per arm; **all the same part** on SO-100 |
| Waveshare Bus Servo Adapter (A) | 2 | one per arm — see `electrical/` |
| 5 V power supply (5.5×2.1 mm) | 2 | one per arm |
| USB-C cable | 2 | one per board |
| Table clamp | 4 | |
| 3D printed frame | 2 sets | PLA, 0.2 mm layer, **13% infill** |
| **USB camera** | **≥1** | **not in the upstream BOM** — see below |
| Interlink FSR 402 + MCU + 10 kΩ | 1 | grip-force sensing |

Upstream BOM totals roughly **$232** for the pair, excluding cameras and the FSR parts.

**The camera gap is real.** Neither upstream BOM lists a camera, but every
LeRobot policy trains on camera frames plus joint states. With zero cameras you
can teleoperate and replay but cannot train a useful policy. Budget for one
overhead and ideally one wrist camera.

## Two SO-100 traps

**Configure motors before assembly.** LeRobot's SO-100 guide is explicit that
the motor connectors are not reachable once the arm is built. Set servo IDs 1–6
with loose motors on the bench or you will disassemble the arm to fix it.

**Remove the gears from all 6 leader motors.** That turns them into low-friction
position encoders, which is what makes the leader back-driveable. A geared
leader is stiff and unpleasant to teleoperate. This step does not exist on the
SO-101, so SO-101 instructions will not mention it.

## Current state

| Subsystem | State |
|---|---|
| CAD | Sourced from upstream, unmodified. No FSR jaw pocket yet. |
| Electrical | Vendor schematic only, for reference. Nothing designed or built. |
| Simulation | URDF + meshes present. Visualisation only — no MJCF for SO-100. |
| Software | Documented end to end. **Never run — no hardware assembled yet.** |
| FSR | Circuit and procedure specified. Not built, not calibrated. |

Nothing in this repository has been executed against a physical robot.

## Conventions

- Units: **millimetres** for geometry, **Newtons** for force, servo counts (0–4095) for position.
- Servo IDs: **1 = shoulder_pan** … **6 = gripper**, base outward.
- Joint names match LeRobot: `shoulder_pan`, `shoulder_lift`, `elbow_flex`,
  `wrist_flex`, `wrist_roll`, `gripper`.
- Arm `id` strings (`--robot.id`, `--teleop.id`) are calibration filenames —
  keep them identical across setup, calibrate, teleoperate, record and rollout.

## Safety

The servos are geared and will strip. `required_libraries.md` caps grip-force
experiments at **30% torque (limit 300)** for that reason. Ramp torque upward
gradually, stop at the first grinding noise, and never leave a torque sweep
running unattended. A stripped STS3215 is $15 plus a teardown.

## Upstream

- Hardware: [TheRobotStudio/SO-ARM100](https://github.com/TheRobotStudio/SO-ARM100) ([SO-100 doc](https://github.com/TheRobotStudio/SO-ARM100/blob/main/SO100.md))
- Software: [huggingface/lerobot](https://github.com/huggingface/lerobot) ([SO-100 guide](https://huggingface.co/docs/lerobot/so100))
