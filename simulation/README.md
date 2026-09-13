# simulation/

Kinematic model of the **SO-100** arm, for visualisation and offline work.

```
simulation/
├── so100.urdf        7 links, 6 revolute joints
├── so100.rrd         pre-recorded rerun session (1.4 MB)
└── assets/           13 STL meshes, referenced by relative path
```

Copied from upstream `Simulation/SO100/`. Mesh paths are relative
(`assets/Base.stl`), so the folder is self-contained — **keep `so100.urdf` and
`assets/` together** or every mesh reference breaks.

## SO-100 has no MuJoCo model

Upstream ships MJCF **only for the SO-101**. For the SO-100 there is a URDF and
nothing else. That means:

| Want | SO-100 | SO-101 |
|---|---|---|
| Visualise kinematics | ✅ URDF | ✅ URDF |
| MuJoCo physics / contact sim | ❌ none | ✅ `so101_new_calib.xml`, `scene.xml` |
| Calibration variants | ❌ single model | ✅ old + new calibration |

If you need physics — contact forces, grasp simulation, RL — you either convert
this URDF to MJCF yourself, or use the SO-101 model and accept that the geometry
does not match your hardware. **Converting is the honest option**; borrowing the
SO-101 model silently introduces link-length errors.

This limitation is the main practical cost of choosing SO-100 over SO-101.

## Joints

Names match LeRobot's convention exactly, base outward, matching servo IDs 1–6:

| Joint | Type | Lower | Upper | Effort | Servo ID |
|---|---|---:|---:|---:|:--:|
| `shoulder_pan` | revolute | −2.0 | 2.0 | 35 | 1 |
| `shoulder_lift` | revolute | 0.0 | 3.5 | 35 | 2 |
| `elbow_flex` | revolute | −3.14159 | 0.0 | 35 | 3 |
| `wrist_flex` | revolute | −2.5 | 1.2 | 35 | 4 |
| `wrist_roll` | revolute | −3.14159 | 3.14159 | 35 | 5 |
| `gripper` | revolute | −0.2 | 2.0 | 35 | 6 |

Limits are radians. The `effort=35` figure is uniform across all joints and is a
placeholder, not a measured torque — do not treat it as a real actuator limit.

## The gripper trap

**The URDF models the gripper as a revolute joint in radians. LeRobot exposes it
as a linear value from 0 (closed) to 100 (open).** These are different
parameterisations of the same axis, and the mapping is *not* encoded anywhere in
this model.

So a gripper command that works against LeRobot will be wrong against the URDF,
and vice versa. Anything crossing that boundary — replaying a recorded dataset
in sim, comparing simulated to real joint states, driving the model from policy
output — has to convert explicitly. Upstream flags this and has not fixed it.

The five arm joints do not have this problem; radians are radians.

## Viewing it

```bash
pip install rerun-sdk        # arrives with lerobot[viz] / [core_scripts]
rerun so100.urdf
```

The URDF loader plugin is required for rerun to render a URDF:
<https://github.com/rerun-io/rerun-loader-python-example-urdf>

`so100.rrd` is a pre-recorded session — open it to see the model without
installing the plugin or having hardware:

```bash
rerun so100.rrd
```

On a headless DGX, run the viewer locally and point it at the file; forwarding a
rerun session over SSH is more trouble than copying a 1.4 MB `.rrd`.

## How the model was made

Upstream generated these with
[onshape-to-robot](https://github.com/Rhoban/onshape-to-robot) from the Onshape
CAD, then hand-edited:

- `package://` mesh URIs rewritten to relative paths, so no ROS package is needed
- **base collision meshes removed** — they behaved badly during collision
  checking and motion planning

That second point matters if you plan anything contact-based: the base has
visual geometry but nothing to collide against. Add collision geometry yourself
before trusting a planner near the base.

Note that the live Onshape document IDs exposed in `.part` metadata files exist
only for the **SO-101** upstream. There is no equivalent for SO-100, so this
URDF cannot be regenerated from CAD without recreating the Onshape source.

## Adding the FSR

Not done. If you want grip force in simulation, the shape of the work is:

1. Add a `<link>` for the sensor pad on the moving jaw, positioned to match the
   physical FSR — which also means the CAD pocket must exist first
   (see [`../cad/README.md`](../cad/README.md)).
2. For contact forces you need MuJoCo, which means converting the URDF to MJCF
   as described above.
3. Match the observation key used on the real robot so a policy sees the same
   input in both places.

Step 2 is the real cost. A URDF alone cannot report contact force.

## Scope

Visualisation and kinematics only. No physics, no contact, no controller, and no
dynamic parameters worth trusting — masses and inertias came from CAD density
estimates, not measurement. Nothing here has been validated against a physical
arm.
