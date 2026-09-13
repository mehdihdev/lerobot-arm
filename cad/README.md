# cad/

STEP assemblies for the **SO-100** leader and follower arms.

```
cad/
├── Leader.step      6.0 MB   leader arm, complete
└── Follower.step    6.7 MB   follower arm, complete
```

## Provenance

Both files are **byte-identical copies** of upstream assemblies, renamed:

| File here | Upstream path |
|---|---|
| `Leader.step` | `STEP/SO100/Leader_Specific/SO_5DOF_ARM100_Assembly.step` |
| `Follower.step` | `STEP/SO100/Follower_Specific/SO_5DOF_ARM100_Assembly.step` |

Source: [TheRobotStudio/SO-ARM100](https://github.com/TheRobotStudio/SO-ARM100).
Upstream ships them under the same filename in different folders, hence the rename.

They were chosen over the SO-101 because the SO-101 publishes only **one**
assembly file — the follower — with no leader equivalent. The SO-100 ships both
as self-contained single files, which is the whole reason this project targets it.

## What each assembly contains

The two arms share the base, shoulder, upper arm, forearm and wrist; they differ
only past the wrist.

| | Leader | Follower |
|---|---|---|
| Shared chain | `Base_08q`, `Rotation_Pitch_08i`, `Wrist_Roll_Pitch_08i`, `SO_5DOF_ARM100_08k` | same |
| Distal | `Wrist_Roll_05m_Leader_02ba`, `Trigger_04e` | `Wrist_Roll_08c`, `Moving_Jaw_08d` |

Grep any assembly to confirm what you have:

```bash
grep -oE "PRODUCT\('[^']*'" Follower.step | sort -u | head -30
```

## Working with these files

**Units are millimetres.** STEP carries units internally; every CAD package will
read them correctly. Do not rescale on import.

**No feature history.** STEP is a boundary representation — parts arrive as
static solids with no sketches, no parameters, no feature tree. Fine for
mounting, fixtures, clearance checks and measurement. Re-modelling a part means
rebuilding its features from scratch.

**Importing to Onshape.** Upload the file; it becomes an Assembly tab plus Part
Studios. There are **no Onshape document IDs available for the SO-100** — the
`.part` metadata files that expose live Onshape links exist only for the SO-101
in upstream's `Simulation/SO101/assets/`. If parametric editing matters more
than having a pre-split leader/follower pair, that is the one real argument for
switching variants.

## Printing

The STEP files are for reference and modification, **not for printing**.
Upstream ships plate-ready STL and 3MF, pre-oriented to minimise supports:

```
STL/SO100/Follower/Print_Follower_SO_ARM100_08k_Ender.STL
STL/SO100/Leader/Print_Leader_SO_ARM100_08k_Ender.STL
STL/SO100/{Follower,Leader}/Print_*_Bambu_P1P.3mf
STL/SO100/Individual/{Follower,Leader}/        one file per part
```

Per-printer variants exist for Ender, Prusa, UP and Bambu A1 Mini (the A1 Mini
files are split into `part1` / `part2` for bed size).

Settings from the upstream SO-100 doc: **PLA**, 0.4 mm nozzle at 0.2 mm layer
height (or 0.6 mm at 0.4 mm), **13% infill**, supports everywhere but ignoring
slopes steeper than 45°, and **no supports in horizontal-axis screw holes**.

## Planned modification: FSR pocket

The grip-force work needs a rigid puck seated over the FSR on the jaw face, with
the load path passing through the sensor rather than around it. That means
editing the **follower's** moving jaw (`Moving_Jaw_08d`).

Constraints for whoever does it:

- Keep the jaw **rigid**. FSRs read substrate bending as force, so any
  compliance under load corrupts the measurement.
- Pocket depth must seat the 0.2 mm sensor plus the puck without changing the
  effective jaw face position, or grip geometry shifts.
- Puck diameter slightly **smaller** than the FSR's 12.7 mm sensing area, so
  load concentrates uniformly instead of spilling past the edge.
- Route the sensor tail so closing the jaw does not pinch it.

Since the STEP has no feature tree, this is a direct-edit job: cut the pocket as
new geometry on the imported solid.

## Untouched

Neither file has been modified. If you change one, note it here with the date
and what changed — there is no other record, and the originals are recoverable
from upstream.
