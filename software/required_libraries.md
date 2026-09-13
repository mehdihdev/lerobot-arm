# SO-100 Setup on DGX: LeRobot Stack + FSR Grip-Force Logging

Everything needed to (a) run a leader + follower SO-100 pair under LeRobot on an
NVIDIA DGX, and (b) read calibrated grip force off an FSR taped to the gripper jaw.

Target platform: **DGX OS / Ubuntu, NVIDIA GPU**. Commands verified against the
LeRobot docs on 2026-09-13.

---

# Part 0 — Read This Before You Start

## 0.1 The DGX is probably not next to the robot

LeRobot's control loop talks to the arms over **USB serial**. There is no network
transport. Whatever machine runs `lerobot-teleoperate` and `lerobot-record` must
be **physically cabled** to both arms — and, for the FSR, to a third USB device.

That gives you two workable topologies:

| Topology | How it works | When to use |
|---|---|---|
| **DGX does everything** | Arms + MCU plugged directly into the DGX. Requires physical access to its USB ports. | DGX Spark / Station on a desk |
| **Split** (common) | Record on a small machine beside the robot (laptop, NUC, Pi); rsync/push the dataset; **train on the DGX**; deploy back. | Rack-mounted A100/H100/B200 |

If your DGX lives in a rack, plan on the split topology. Parts 1 and 3 apply to
whichever machine is cabled to the hardware; Part 4 (training) is the DGX.
Nothing below assumes they're the same box.

## 0.2 Identify your platform

Two things change the install. Get them now:

```bash
uname -m                                                   # x86_64 or aarch64
nvidia-smi --query-gpu=name,driver_version --format=csv    # GPU + driver
```

| `uname -m` | Typical DGX | Consequence |
|---|---|---|
| `x86_64` | A100, H100, H200, B200, older Station | TorchCodec available → **install ffmpeg** |
| `aarch64` | **DGX Spark (GB10)**, Station GB300 | TorchCodec **not** available on Linux ARM → LeRobot falls back to `pyav`, **skip ffmpeg entirely** |

Note the driver version — §1.4 picks a CUDA wheel from it.

## 0.3 You may not have sudo

On a shared DGX you might not. The steps needing root are: `apt` build
dependencies (§1.6), the `dialout` group (§2.1), and udev rules (§2.2). If you
lack sudo, ask your admin for those three once — everything else installs into
your own environment.

---

# Part 1 — LeRobot

## 1.1 Prerequisites

| Requirement | Version | Notes |
|---|---|---|
| Python | **>= 3.12** | DGX OS ships older — install your own |
| PyTorch | **>= 2.10** | pulled in automatically |
| ffmpeg | 7.1.1 or 8.x | **x86_64 only** — skip on aarch64 |
| NVIDIA driver | see §1.4 | determines the CUDA wheel |

## 1.2 Environment

Pick one. `uv` is fastest and needs no root; conda is what the docs assume.

**uv:**
```bash
curl -LsSf https://astral.sh/uv/install.sh | sh      # if not present
uv python install 3.12
uv venv --python 3.12
source .venv/bin/activate
```
Replace every `pip install` below with `uv pip install`.

**conda (miniforge):**
```bash
wget "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-$(uname)-$(uname -m).sh"
bash Miniforge3-$(uname)-$(uname -m).sh
conda create -y -n lerobot python=3.12
conda activate lerobot          # re-run in every new shell
```

## 1.3 ffmpeg — x86_64 only

```bash
# Check first:
if [ "$(uname -m)" = "aarch64" ]; then echo "aarch64 -> SKIP ffmpeg, pyav fallback"; fi
```

**x86_64 with conda:**
```bash
conda install ffmpeg -c conda-forge
# if libsvtav1 is missing (check: ffmpeg -encoders) or torchcodec complains:
conda install ffmpeg=7.1.1 -c conda-forge
```

**x86_64 with uv/venv** — system ffmpeg works because LeRobot requires PyTorch ≥ 2.10:
```bash
sudo apt install ffmpeg
```

**aarch64 (DGX Spark / GB300):** install nothing. TorchCodec has no Linux-ARM
build; LeRobot detects this and uses `pyav`. Installing ffmpeg here is harmless
but pointless.

## 1.4 CUDA wheel — match your driver

Read the driver from §0.2, then:

| Driver version | GPU generation | Command |
|---|---|---|
| **≥ 580.65** | any, incl. Blackwell | PyPI default (cu130) — just install, no extra step |
| **570.86 – 580.64** | Ampere / Ada / Hopper / Blackwell | pin cu128, below |
| < 570.86 | older | update the driver, or pin cu126 |

Pin an older CUDA before installing LeRobot:
```bash
pip install --index-url https://download.pytorch.org/whl/cu128 torch torchvision
```

With `uv` installing from PyPI, one flag does it:
```bash
uv pip install --torch-backend cu128 lerobot
```
Accepted values: `auto`, `cpu`, `cu126`, `cu128`, `cu129`, `cu130`, plus `rocm*` / `xpu`.

> Blackwell (B200, GB10, GB300) wants cu130 on driver ≥ 580. `--torch-backend auto`
> is a reasonable default if you'd rather not reason about it.

## 1.5 Install LeRobot

**From PyPI** — fine if you're not modifying LeRobot:
```bash
pip install 'lerobot[core_scripts,feetech,training]'
```

**From source** — needed for a custom robot class (§3.8) or to contribute:
```bash
git clone https://github.com/huggingface/lerobot.git
cd lerobot
pip install -e ".[core_scripts,feetech,training]"
```

### What each extra pulls in

| Extra | Adds | Why |
|---|---|---|
| `feetech` | Feetech STS3215 SDK | **Mandatory.** Without it the SO-100 will not talk. |
| `core_scripts` | `dataset` + `hardware` + `viz` | `lerobot-record`, `-replay`, `-calibrate` |
| `training` | `dataset` + `accelerate`, `wandb` | **The reason you have a DGX.** |
| `dataset` | `datasets`, `av`, `torchcodec`, `jsonlines` | dataset create/load |
| `hardware` | `pynput`, **`pyserial`**, `deepdiff` | real-robot I/O — `pyserial` is what Part 3 uses for the FSR |
| `viz` | `rerun-sdk` | live camera + joint visualisation |

Policy extras, added when you pick an architecture:
```bash
pip install 'lerobot[act]'          # ACT — start here
pip install 'lerobot[smolvla]'      # SmolVLA
pip install 'lerobot[pi]'           # Pi0 / Pi0.5 / Pi0-FAST
pip install 'lerobot[diffusion]'    # Diffusion policy
```

> `lerobot[hardware]` already provides `pyserial`, so Part 3 needs **no extra
> install** beyond `numpy` / `matplotlib` for plots.

## 1.6 Build dependencies, if the install fails

```bash
sudo apt-get install cmake build-essential python3-dev pkg-config \
  libavformat-dev libavcodec-dev libavdevice-dev libavutil-dev \
  libswscale-dev libswresample-dev libavfilter-dev
```

## 1.7 Verify

```bash
python -c "import lerobot, torch; print(lerobot.__version__, torch.__version__)"
python -c "import torch; print('cuda:', torch.cuda.is_available(), torch.cuda.device_count(), 'gpus')"
lerobot-find-port --help
```

`torch.cuda.is_available()` must print `True` before you bother training.

## 1.8 Running inside a container (NGC / Docker)

DGX workflows are often containerised. USB devices are **not** visible by default:

```bash
docker run --gpus all -it \
  --device=/dev/ttyACM0 --device=/dev/ttyACM1 --device=/dev/ttyACM2 \
  -v /dev/bus/usb:/dev/bus/usb \
  -v $HOME/.cache/huggingface:/root/.cache/huggingface \
  nvcr.io/nvidia/pytorch:25.01-py3
```

Pass one `--device` per arm plus one for the FSR MCU. Mounting the HF cache
keeps datasets outside the container. If devices are hot-plugged after start,
the container won't see them — use `--privileged -v /dev:/dev` or restart it.

---

# Part 2 — Serial Ports (do this once, save yourself pain)

You will have **three** `/dev/ttyACM*` devices: follower, leader, FSR MCU. Linux
enumerates them in whatever order they appear, so `ttyACM0` is **not stable
across reboots or replugs**. Fix that now.

## 2.1 Permissions — the persistent way

Skip `sudo chmod 666 /dev/ttyACM0`; it resets on every replug. Instead:

```bash
sudo usermod -aG dialout $USER
# log out and back in (or: newgrp dialout), then confirm:
groups | grep dialout
```

## 2.2 Stable device names via udev

Inspect what identifies each adapter:

```bash
udevadm info -a -n /dev/ttyACM0 | grep -E 'ATTRS\{(serial|idVendor|idProduct)\}|KERNELS' | head
```

**If the adapters report unique serials** (preferred), match on serial:

```bash
sudo tee /etc/udev/rules.d/99-so100.rules >/dev/null <<'EOF'
SUBSYSTEM=="tty", ATTRS{idVendor}=="1a86", ATTRS{serial}=="SERIAL_A", SYMLINK+="so100_follower"
SUBSYSTEM=="tty", ATTRS{idVendor}=="1a86", ATTRS{serial}=="SERIAL_B", SYMLINK+="so100_leader"
SUBSYSTEM=="tty", ATTRS{idVendor}=="2886", SYMLINK+="fsr_mcu"
EOF
sudo udevadm control --reload-rules && sudo udevadm trigger
```

**If they don't** — some CH34x parts ship without unique serials — match on the
physical USB port instead, and always plug each arm into the same socket:

```bash
SUBSYSTEM=="tty", KERNELS=="1-3.1:1.0", SYMLINK+="so100_follower"
SUBSYSTEM=="tty", KERNELS=="1-3.2:1.0", SYMLINK+="so100_leader"
```

Then use the stable names everywhere:
```bash
FOLLOWER_PORT=/dev/so100_follower
LEADER_PORT=/dev/so100_leader
FSR_PORT=/dev/fsr_mcu
```

## 2.3 Identify the arms the first time

```bash
lerobot-find-port
```
Run once per arm, unplugging the one being identified when prompted.

> **Waveshare board:** both jumpers must be on the **`B` channel (USB)** or the
> board will not enumerate at all.

---

# Part 3 — Two-Arm Bring-Up

The `id` you assign becomes the calibration filename. **Keep it identical across
setup, calibrate, teleoperate, record, and rollout.**

## 3.1 Set motor IDs — once per motor, BEFORE assembly

> **SO-100 only, and it matters.** LeRobot's docs are explicit: *"Unlike the
> SO-101, the motor connectors are not easily accessible once the arm is
> assembled, so the configuration step must be done beforehand."* If you
> assemble first you will be taking the arm apart again. Do §3.1 with loose
> motors on the bench.

Writes to motor EEPROM. Connect **one motor at a time**, not daisy-chained.

```bash
lerobot-setup-motors --robot.type=so100_follower --robot.port=$FOLLOWER_PORT
lerobot-setup-motors --teleop.type=so100_leader  --teleop.port=$LEADER_PORT
```

The script walks backwards from `gripper` (id 6) to `shoulder_pan` (id 1).

**All 12 servos are the same part.** Unlike the SO-101, the SO-100 uses one
STS3215 variant throughout — there is no per-joint gearing to keep straight.

**The leader's 6 motors need their gears removed.** That converts them into
position encoders with almost no friction, which is what makes the leader
back-driveable by hand. See the gear-removal video in the
[SO-100 assembly guide](https://huggingface.co/docs/lerobot/so100). Do this
before assembling the leader; a geared leader is stiff and unpleasant to
teleoperate.

## 3.2 Calibrate both arms

```bash
lerobot-calibrate --robot.type=so100_follower --robot.port=$FOLLOWER_PORT --robot.id=my_follower
lerobot-calibrate --teleop.type=so100_leader  --teleop.port=$LEADER_PORT  --teleop.id=my_leader
```

Move to mid-range on all joints, press Enter, then sweep each joint end to end.

> `Software/WEBUI_CALIBRATION.md` in this repo describes a 3-point WebUI
> alternative, but it calls `pi_servo_studio.py`, **which does not exist in this
> repository**. Use `lerobot-calibrate`.

## 3.3 Headless and SSH — what works and what doesn't

DGX sessions are usually SSH. Per the LeRobot docs:

| Feature | Over SSH / headless? |
|---|---|
| `lerobot-record` control keys (`→`/`n`, `←`/`r`, `ESC`/`q`) | ✅ works — needs an interactive TTY, no `$DISPLAY` |
| Keyboard **teleoperation** (driving with arrow keys) | ❌ needs X11 / a desktop session |
| Leader-arm teleoperation | ✅ unaffected — it's serial, not keyboard |
| `--display_data=true` (rerun) | ⚠️ needs a rerun viewer; forward it or drop the flag |

Run recording from an interactive terminal (`ssh -t`), keep it focused, and use
`n` / `r` / `q` rather than arrow keys if anything misbehaves. Leader-arm
teleop — which is what you have — is unaffected by all of this.

## 3.4 Teleoperate

```bash
lerobot-teleoperate \
    --robot.type=so100_follower --robot.port=$FOLLOWER_PORT --robot.id=my_follower \
    --teleop.type=so100_leader  --teleop.port=$LEADER_PORT  --teleop.id=my_leader
```

With cameras + live view (needs a display or forwarded rerun viewer):
```bash
lerobot-teleoperate \
    --robot.type=so100_follower --robot.port=$FOLLOWER_PORT --robot.id=my_follower \
    --robot.cameras="{ front: {type: opencv, index_or_path: 0, width: 640, height: 480, fps: 30}}" \
    --teleop.type=so100_leader  --teleop.port=$LEADER_PORT  --teleop.id=my_leader \
    --display_data=true
```

> **Cameras are not in this repo's BOM.** Neither parts list in `README.md`
> includes one, but ACT and every other policy here train on camera frames plus
> joint states. Budget for at least one USB camera — realistically two (overhead
> + wrist). `Optional/` has the mounts; you supply the camera.

## 3.5 Record a dataset

```bash
hf auth login --token ${HUGGINGFACE_TOKEN} --add-to-git-credential
HF_USER=$(NO_COLOR=1 hf auth whoami | awk -F': *' 'NR==1 {print $2}')

lerobot-record \
    --robot.type=so100_follower --robot.port=$FOLLOWER_PORT --robot.id=my_follower \
    --robot.cameras="{ front: {type: opencv, index_or_path: 0, width: 1920, height: 1080, fps: 30}}" \
    --teleop.type=so100_leader  --teleop.port=$LEADER_PORT  --teleop.id=my_leader \
    --dataset.repo_id=${HF_USER}/record-test \
    --dataset.num_episodes=5 \
    --dataset.single_task="Grab the black cube"
```

Keys: `→`/`n` next · `←`/`r` re-record · `ESC`/`q` stop.
Local cache: `~/.cache/huggingface/lerobot/{repo-id}`.
Offline: `--dataset.push_to_hub=False`. Resume: `--resume=true` with
`--dataset.root=<local_path>`, and set `num_episodes` to the **additional** count.

Aim for ≥50 episodes, ~10 per object location, cameras fixed.

## 3.6 Split topology — moving data to the DGX

If you recorded elsewhere:
```bash
# via the Hub (simplest)
hf upload ${HF_USER}/record-test ~/.cache/huggingface/lerobot/record-test --repo-type dataset
# then on the DGX, lerobot-train pulls it by repo_id automatically

# or direct
rsync -avP ~/.cache/huggingface/lerobot/record-test/ dgx:~/.cache/huggingface/lerobot/record-test/
```

---

# Part 4 — Training on the DGX

This is what the machine is for. Locally, not on HF Jobs.

```bash
lerobot-train \
  --dataset.repo_id=${HF_USER}/record-test \
  --policy.type=act \
  --output_dir=outputs/train/act_so100 \
  --job_name=act_so100 \
  --policy.device=cuda \
  --wandb.enable=true \
  --policy.repo_id=${HF_USER}/my_policy
```

Resume from a checkpoint:
```bash
lerobot-train --config_path=outputs/train/act_so100/checkpoints/last/pretrained_model/train_config.json --resume=true
```

Weights & Biases (in the `training` extra):
```bash
wandb login
```

**Pinning a GPU** on a multi-GPU DGX so you don't collide with other users:
```bash
CUDA_VISIBLE_DEVICES=0 lerobot-train ...
nvidia-smi                    # check what's free first
```

`accelerate` ships with the `training` extra if you scale to multi-GPU.

Deploy the trained policy back to the robot-connected machine:
```bash
lerobot-rollout \
  --strategy.type=base \
  --policy.path=${HF_USER}/my_policy \
  --robot.type=so100_follower --robot.port=$FOLLOWER_PORT --robot.id=my_follower \
  --task="Grab the black cube" --duration=60
```

`--strategy.type` options: `base` (no recording), `sentry` (continuous + upload),
`highlight` (ring buffer), `dagger` (human-in-loop), `episodic`.

---

# Part 5 — FSR Grip-Force Logging

## 5.1 Bill of materials

| Item | Qty | ~Cost | Notes |
|---|:--:|---|---|
| Interlink **FSR 402** (12.7 mm round) | 1 | $8 | 0.2 mm thick, tapes to the jaw pad |
| Microcontroller with ADC | 1 | $5 | Seeed XIAO, Pi Pico, any Arduino |
| Resistor, **10 kΩ** 1% | 1 | — | divider leg; see 5.3 |
| Rigid disc ("puck"), Ø ~10 mm | 1 | — | **not optional** — see 5.2 |
| Known masses (100 g–2 kg) | set | — | calibration; kitchen weights are fine |
| USB cable for the MCU | 1 | — | third USB device |

The Waveshare servo board **cannot** read this — it's a CH343 USB-UART plus the
half-duplex servo transceiver, with no spare ADC or GPIO. The MCU is mandatory.

## 5.2 Two things that will ruin your data

**Use a rigid puck.** An FSR integrates force over its active area; an uneven jaw
pad pressing on it gives noise, not force. Tape a hard disc *slightly smaller*
than the sensing area on top so load lands uniformly. This is the single biggest
determinant of whether the readings mean anything.

**Mount on a rigid jaw.** FSRs respond to substrate bending as if it were force,
so anything that flexes under load reads its own deformation. The stock SO-100
jaw is rigid PLA, which is what you want. (The upstream compliant TPU gripper is
an SO-101 part and has no SO-100 variant — if you ever port it over, do not put
an FSR on it.)

Also: exercise the sensor 10–20 times before trusting it, and let each reading
settle ~2 s — FSRs creep under constant load.

## 5.3 Wiring

```
  3V3 ──────[ FSR ]──────┬────── ADC pin
                         │
                      [ 10k ]
                         │
  GND ───────────────────┘
```

More force → lower FSR resistance → **higher** ADC reading.

Pick `R_FIXED` to centre the divider over your force range: sensitivity peaks
where `R_FIXED ≈ R_FSR`. 10 kΩ suits roughly 1–10 N. Saturating near full scale?
Drop to 3.3 kΩ. Readings hugging zero? Go to 47 kΩ.

## 5.4 Toolchain — flashing the MCU

```bash
curl -fsSL https://raw.githubusercontent.com/arduino/arduino-cli/master/install.sh | sh
export PATH="$PWD/bin:$PATH"        # or move arduino-cli onto your PATH
arduino-cli config init
arduino-cli core update-index
```

Install the core for your board, then compile and upload:

```bash
# Seeed XIAO SAMD21
arduino-cli core install Seeeduino:samd
arduino-cli compile --fqbn Seeeduino:samd:seeed_XIAO_m0 fsr_stream
arduino-cli upload  --fqbn Seeeduino:samd:seeed_XIAO_m0 -p $FSR_PORT fsr_stream

# Raspberry Pi Pico
arduino-cli core install rp2040:rp2040
arduino-cli compile --fqbn rp2040:rp2040:rpipico fsr_stream
arduino-cli upload  --fqbn rp2040:rp2040:rpipico -p $FSR_PORT fsr_stream
```

The sketch must live in a directory matching its filename
(`fsr_stream/fsr_stream.ino`). List connected boards with `arduino-cli board list`.

## 5.5 Firmware

```cpp
// fsr_stream.ino — CSV force stream: t_ms,raw,volts,r_ohm,newtons
const int   FSR_PIN   = A0;
const float VCC       = 3.3f;      // 5.0 on a classic Uno
const float R_FIXED   = 10000.0f;
const int   ADC_BITS  = 12;        // Uno = 10, SAMD21/RP2040 = 12
const int   N_AVG     = 16;
const unsigned long PERIOD_MS = 20;   // 50 Hz

// Power-law fit from 5.7 — REPLACE after calibrating. F = CAL_A * G^CAL_B
float CAL_A = 1.0f, CAL_B = 1.0f;

float adcMax;
unsigned long tNext = 0;

void setup() {
  Serial.begin(115200);
  while (!Serial) {}
  #if defined(ARDUINO_ARCH_SAMD) || defined(ARDUINO_ARCH_RP2040)
    analogReadResolution(ADC_BITS);
  #endif
  adcMax = (float)((1 << ADC_BITS) - 1);
  Serial.println("t_ms,raw,volts,r_ohm,newtons");
}

void loop() {
  if (millis() < tNext) return;
  tNext = millis() + PERIOD_MS;

  long acc = 0;
  for (int i = 0; i < N_AVG; i++) acc += analogRead(FSR_PIN);
  float raw   = (float)acc / N_AVG;
  float volts = raw * VCC / adcMax;

  float r_ohm, newtons;
  if (volts < 0.01f) {                 // open circuit = no contact
    r_ohm = -1.0f; newtons = 0.0f;
  } else {
    r_ohm = R_FIXED * (VCC - volts) / volts;
    float g = 1.0f / r_ohm;            // conductance
    newtons = CAL_A * powf(g, CAL_B);
  }

  Serial.print(millis());   Serial.print(',');
  Serial.print(raw, 1);     Serial.print(',');
  Serial.print(volts, 4);   Serial.print(',');
  Serial.print(r_ohm, 1);   Serial.print(',');
  Serial.println(newtons, 3);
}
```

## 5.6 Host logger

Needs only `pyserial`, already present via `lerobot[hardware]`.

```bash
pip install pyserial numpy matplotlib
```

```python
#!/usr/bin/env python3
"""fsr_log.py — log the FSR stream to CSV and report peak force.

    python fsr_log.py --port /dev/fsr_mcu --out grip_test.csv
"""
import argparse, csv, time
import serial

p = argparse.ArgumentParser()
p.add_argument("--port", required=True, help="MCU port (NOT an arm port)")
p.add_argument("--baud", type=int, default=115200)
p.add_argument("--out", default="fsr_log.csv")
p.add_argument("--duration", type=float, default=0.0, help="0 = until Ctrl-C")
a = p.parse_args()

ser = serial.Serial(a.port, a.baud, timeout=1.0)
time.sleep(2.0)          # board resets on open
ser.reset_input_buffer()

peak, n, t0 = 0.0, 0, time.time()
with open(a.out, "w", newline="") as fh:
    w = csv.writer(fh)
    w.writerow(["t_ms", "raw", "volts", "r_ohm", "newtons"])
    print(f"logging -> {a.out}   (Ctrl-C to stop)")
    try:
        while True:
            if a.duration and time.time() - t0 > a.duration:
                break
            line = ser.readline().decode("utf-8", "replace").strip()
            if not line or line.startswith("t_ms"):
                continue
            parts = line.split(",")
            if len(parts) != 5:
                continue
            try:
                row = [float(x) for x in parts]
            except ValueError:
                continue
            w.writerow(row)
            n += 1
            if row[4] > peak:
                peak = row[4]
            if n % 25 == 0:
                print(f"\r  now {row[4]:7.2f} N   peak {peak:7.2f} N   "
                      f"n={n}", end="", flush=True)
    except KeyboardInterrupt:
        pass

ser.close()
print(f"\n\npeak force: {peak:.2f} N   ({peak/9.81*1000:.0f} g equivalent)")
print(f"{n} samples -> {a.out}")
```

Peak force is the number you want for grip strength.

## 5.7 Calibration — raw counts to Newtons

FSR conductance is roughly a power law in force, so fit in log-log space.

**Procedure.** Lay the sensor flat, puck up. Stack a known mass on the puck, wait
~2 s for creep to settle, note the steady `r_ohm`. Repeat across your range
(100 g, 200 g, 500 g, 1 kg, 2 kg). Force in Newtons is `mass_kg * 9.81`.

```python
#!/usr/bin/env python3
"""fsr_calibrate.py — fit F = A * G^B from (grams, r_ohm) pairs."""
import numpy as np

# EDIT: your measurements — (grams, r_ohm at rest under that mass)
DATA = [
    (100,  47000.0),
    (200,  24000.0),
    (500,   9500.0),
    (1000,  4600.0),
    (2000,  2100.0),
]

g = np.array([d[0] for d in DATA], dtype=float)
r = np.array([d[1] for d in DATA], dtype=float)
F = g / 1000.0 * 9.81          # N
G = 1.0 / r                    # conductance

B, logA = np.polyfit(np.log(G), np.log(F), 1)
A = np.exp(logA)

pred = A * G**B
err = 100 * np.abs(pred - F) / F
print(f"CAL_A = {A:.6g}")
print(f"CAL_B = {B:.6g}")
print(f"\nfit error: mean {err.mean():.1f}%  max {err.max():.1f}%")
for gi, fi, pi, ei in zip(g, F, pred, err):
    print(f"  {gi:6.0f} g   actual {fi:6.2f} N   fit {pi:6.2f} N   ({ei:4.1f}%)")
```

Paste the printed `CAL_A` / `CAL_B` into the sketch and reflash (§5.4).

Expect **±15–25%** absolute even after this — the FSR's honest limit. Fine for
"setting 40 squeezes twice as hard as setting 20"; not a calibrated load cell.
If you need real Newtons, swap in a compression load cell + HX711; everything
else in this section still applies.

## 5.8 Getting force into a LeRobot dataset

LeRobot has no FSR support. To record force alongside joint states you subclass
`SO100Follower`, add a key to `observation_features`, and merge the serial read
into `get_observation()`. Requires the **source install** from §1.5.

Not needed for bench characterisation (§5.9).

## 5.9 The measurement worth doing

Since this is characterisation, do it **once**:

1. Tape the FSR + puck to the rigid jaw.
2. Sweep the gripper's torque-limit register across its range.
3. At each setting, close on the sensor and record peak force.
4. Fit torque-limit → force, and servo present-current → force.

You then read grip force from **servo current alone**, forever, with nothing
attached to the robot — and the FSR comes off.

> **Sweep upward gradually.** `Software/WEBUI_CALIBRATION.md` caps torque at 30%
> precisely because STS3215 gears strip. Stop at the first grinding noise.

---

## Quick reference

```bash
source .venv/bin/activate               # or: conda activate lerobot

FOLLOWER_PORT=/dev/so100_follower
LEADER_PORT=/dev/so100_leader
FSR_PORT=/dev/fsr_mcu

lerobot-find-port
lerobot-setup-motors --robot.type=so100_follower --robot.port=$FOLLOWER_PORT
lerobot-calibrate    --robot.type=so100_follower --robot.port=$FOLLOWER_PORT --robot.id=my_follower
lerobot-teleoperate  --robot.type=so100_follower --robot.port=$FOLLOWER_PORT --robot.id=my_follower \
                     --teleop.type=so100_leader  --teleop.port=$LEADER_PORT  --teleop.id=my_leader
lerobot-train        --dataset.repo_id=${HF_USER}/record-test --policy.type=act \
                     --output_dir=outputs/train/act_so100 --job_name=act_so100 --policy.device=cuda

python fsr_log.py --port $FSR_PORT --out grip_test.csv
```

## Preflight checklist

- [ ] `uname -m` known; ffmpeg installed (x86_64) or skipped (aarch64)
- [ ] `nvidia-smi` driver noted; CUDA wheel matched (§1.4)
- [ ] `torch.cuda.is_available()` is `True`
- [ ] User in `dialout`; udev symlinks resolve
- [ ] Both arms calibrated under stable `--robot.id` / `--teleop.id`
- [ ] **At least one camera** sourced — not in this repo's BOM
- [ ] `arduino-cli` installed and the FSR sketch flashed

## References

- [LeRobot installation](https://huggingface.co/docs/lerobot/installation)
- [SO-100 setup](https://huggingface.co/docs/lerobot/so100)
- [Imitation learning on real robots](https://huggingface.co/docs/lerobot/il_robots)
- [arduino-cli](https://arduino.github.io/arduino-cli/)
- [Interlink FSR 402](https://www.interlinkelectronics.com/fsr-402)
