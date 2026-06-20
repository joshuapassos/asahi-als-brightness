# asahi-als-brightness

Automatic display brightness based on the ambient light sensor (ALS) for Apple Silicon laptops running [Asahi Linux](https://asahilinux.org/).

Works with the **VD6286** ALS sensor found across Apple Silicon laptops (M1 Pro/Max MacBook Pro, M2 MacBook Air, and others), driven by the `aop_als` kernel module from the [Asahi fairydust](https://github.com/AsahiLinux/linux/tree/fairydust) kernel.

> ### Credits / based on
> This project is a **desktop-agnostic fork of [juicecultus/asahi-auto-brightness](https://github.com/juicecultus/asahi-auto-brightness)** (original work © juicecultus — see [`LICENSE`](LICENSE)).
>
> What changed versus the original:
> - **No KDE/PowerDevil dependency** — drives the kernel backlight directly via systemd-logind D-Bus (or a direct sysfs write), so it runs under any Wayland compositor (niri, sway, Hyprland, …), KDE, GNOME, or a bare TTY.
> - **Hysteresis (deadband)** so sensor jitter no longer causes perpetual ramping.
> - **Manual-override detection at any time** (including mid-ramp), by reading the backlight node back.

## How It Works

```
VD6286 sensor → aop_als kernel module → IIO sysfs
    → auto-brightness daemon (reads lux from IIO sysfs)
    → Linux backlight subsystem (sysfs write OR systemd-logind D-Bus)
```

The daemon reads lux directly from the IIO sensor sysfs node and sets brightness
through the kernel backlight device. It is **desktop-agnostic** — no KDE/PowerDevil
dependency — so it works under any Wayland compositor (niri, sway, Hyprland, …),
KDE, GNOME, or a bare TTY. A write backend is auto-selected at startup:

1. **Direct sysfs write** — if `/sys/class/backlight/<dev>/brightness` is writable
   (e.g. via the bundled `99-backlight.rules` udev rule).
2. **systemd-logind D-Bus** (`org.freedesktop.login1` `SetBrightness`) — needs no
   root and no udev rule; logind performs the privileged write for your active
   session. This is the default when the sysfs node is not writable.

Key features (inspired by [macbook-ambient-sensor](https://github.com/juicecultus/macbook-ambient-sensor)):
- **No desktop dependency** — works on niri/sway/Hyprland/KDE/GNOME/TTY
- **Imperceptible transitions** — 0.5%-of-range brightness steps every 0.25s
- **Manual override** — respects brightness changes from compositor keys / `brightnessctl` until ambient light shifts by ≥75% (or ≥5 lux absolute in low light)
- **Minimum brightness floor** — prevents black screen
- **Auto-detects sensor and backlight** — IIO device number varies across reboots; both found automatically

## Prerequisites

- **Kernel**: Asahi fairydust branch with `CONFIG_IIO_AOP_SENSOR_ALS=m`
- **Firmware**: `apple/aop-als-cal.bin` in `/lib/firmware/` (see [Extracting Calibration](#extracting-calibration-data))
- **Packages**: `python3-dbus` (only needed for the logind backend)
- **Desktop**: any — runs under any Wayland compositor (niri, sway, Hyprland), KDE, GNOME, or a bare TTY
- **Brightness write access**: an active systemd-logind session (the normal case under any compositor), *or* a writable backlight node via the bundled udev rule

## Installation

### 1. Extract ALS calibration from macOS

Boot into macOS and capture the ioreg dump:

```bash
ioreg -l -a > ioreg-full.xml
```

Copy `ioreg-full.xml` to your Linux system, then extract the calibration:

```bash
python3 extract-als-cal.py ioreg-full.xml
sudo cp aop-als-cal.bin /lib/firmware/apple/aop-als-cal.bin
sudo dracut --force --kver $(uname -r)
```

### 2. Install the auto-brightness daemon

```bash
# Install the script
mkdir -p ~/.local/bin
cp auto-brightness ~/.local/bin/auto-brightness
chmod +x ~/.local/bin/auto-brightness

# Install the systemd user service
mkdir -p ~/.config/systemd/user
cp auto-brightness.service ~/.config/systemd/user/auto-brightness.service
systemctl --user daemon-reload
systemctl --user enable --now auto-brightness.service
```

### 3. Reboot

After reboot, verify:

```bash
# ALS sensor is active (device number may vary)
cat /sys/bus/iio/devices/iio:device*/in_illuminance_input

# Daemon is running
systemctl --user status auto-brightness
```

## Extracting Calibration Data

The ALS sensor requires factory calibration data from macOS. The `extract-als-cal.py` script parses a macOS `ioreg -l -a` XML dump and extracts the `CalibrationData` blob from the `AppleSPUVD6286` node.

**Without this calibration file, the ALS sensor will report 0 lux.**

The calibration data is device-specific (per-unit factory calibration), so you must extract it from your own machine's macOS installation.

## Brightness Backends

The daemon picks a write backend automatically at startup (shown in its log line):

- **logind D-Bus** (default): uses `org.freedesktop.login1` `SetBrightness`. No root,
  no udev rule. Requires an active logind session on the seat — which every Wayland
  compositor and display manager provides.
- **sysfs direct write**: used when `/sys/class/backlight/<dev>/brightness` is writable.
  To enable it, install the bundled udev rule (optional — only if you prefer to avoid
  D-Bus entirely):

  ```bash
  sudo cp 99-backlight.rules /etc/udev/rules.d/99-backlight.rules
  sudo udevadm control --reload && sudo udevadm trigger -s backlight
  ```

Manual brightness changes (compositor brightness keys, `brightnessctl`, etc.) are
detected by reading the backlight node and respected: auto-brightness pauses until
ambient light changes by ≥75%, so it won't immediately override your preference.

## Customizing the Brightness Curve

Edit the `LUX_CURVE` table in `auto-brightness`. Values are percentages of the panel's
brightness range (the daemon reads `max_brightness` and scales automatically):

```python
LUX_CURVE = [
    (0,     2),    # 0 lux    →  2% brightness
    (1,     3),    # 1 lux    →  3% brightness
    (5,     6),    # 5 lux    →  6% brightness
    (20,   12),    # 20 lux   → 12% brightness
    (50,   24),    # 50 lux   → 24% brightness
    (100,  40),    # 100 lux  → 40% brightness
    (200,  60),    # 200 lux  → 60% brightness
    (400,  80),    # 400 lux  → 80% brightness
    (700,  92),    # 700 lux  → 92% brightness
    (1000, 100),   # 1000+ lux → 100% brightness
]
```

Values are linearly interpolated between points.

## Tuning Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `POLL_INTERVAL` | 0.25s | How often to read the sensor |
| `SMOOTH_STEP_PCT` | 0.5% | Brightness change per cycle, as % of `max_brightness` |
| `LUX_CHANGE_PCT` | 75% | Lux change to resume after manual override |
| `LUX_CHANGE_MIN` | 5 | Minimum absolute lux change (low-light) |
| `MIN_BRIGHTNESS_PCT` | 2% | Floor brightness, as % of `max_brightness` |

## Tested Hardware

- MacBook Pro 16" M1 Max (`apple,j316c` / `t6001`) — VD6286 ALS, Asahi Fedora, niri, backlight scale 0–500
- MacBook Air 15" M2 (J415) — original upstream testing ([juicecultus/asahi-auto-brightness](https://github.com/juicecultus/asahi-auto-brightness))

## Troubleshooting

**ALS reads 0 lux**: Missing or wrong calibration file. Re-extract from macOS.

**Daemon crash-loops after reboot**: The IIO device number may change. The daemon auto-detects it, but if the `aop_als` module loads late, the daemon will retry every 5 seconds (`Restart=always`).

**Brightness doesn't change**: Check `journalctl --user -u auto-brightness` for the backend line. With the logind backend you need an active session — confirm `loginctl session-status` shows `State: active`. If neither backend is available, the daemon exits with `no brightness backend available`; install the udev rule (see [Brightness Backends](#brightness-backends)).

**Manual change ignored**: The daemon detects external brightness changes (compositor keys, `brightnessctl`) and enters manual override mode. It resumes auto-brightness when ambient light changes by ≥75%.

## License

MIT
