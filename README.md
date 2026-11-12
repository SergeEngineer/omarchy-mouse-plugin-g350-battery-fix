# Omarchy Mouse Plugin (`optimistprime.mouse`)

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Platform: Linux](https://img.shields.io/badge/Platform-Linux%20%2F%20Hyprland-purple.svg)](#)
[![Shell: Omarchy Quickshell](https://img.shields.io/badge/Shell-Omarchy%20Quickshell-orange.svg)](#)

A native, interactive **Omarchy shell** bar widget and control engine for Logitech gaming mice (Logitech G Pro Wireless, G Pro X Superlight, G502, G305, G703, G903) and other `libratbag`-compatible devices.

![Omarchy Mouse Widget Preview](preview.png)

---

## Table of contents

- [Features](#-features)
- [Prerequisites](#-prerequisites)
- [Installation](#-installation)
- [Uninstallation](#-uninstallation)
- [Troubleshooting](#-troubleshooting)
- [Command line interface](#-command-line-interface-omarchy-mouse-control)
- [Architecture & design notes](#-architecture--design-notes)
- [License](#-license)

---

## Features

* **Real-time battery telemetry**
  * Live battery percentage and charging state.
  * Reads from the kernel (`hidpp_battery`), then UPower, then the Solaar CLI. The Solaar fallback is what makes mice like the **G305** work, because they report only a coarse level ("Normal") to the kernel instead of a percentage.
  * Proportional battery meter bar.
  * Right-click the bar icon to toggle the inline battery percentage on your status bar.
* **Hardware DPI sensitivity**
  * Continuous hardware slider (`200` to `6400` DPI in 50 DPI steps).
  * Instant jump presets: **400**, **800**, **1000**, **1200**, **1600**, and **3200** DPI.
  * Committed directly to the mouse's onboard memory profile.
* **Polling rate switching**
  * Switch the hardware report rate on the fly: **125 Hz**, **250 Hz**, **500 Hz**, or **1000 Hz**.
* **Hyprland pointer speed synchronization**
  * Live slider for desktop pointer speed (`-1.00` to `+1.00`).
  * Applied via `hyprctl` and persisted to `~/.config/hypr/input.lua`.
* **RGB lighting suite & Omarchy theme synchronization**
  * Master on/off toggle.
  * Color mode with custom hex input, clickable Omarchy theme color chips, and a breathe effect with adjustable pulse speed.
  * Selecting a theme chip binds the mouse to that color slot. When you change Omarchy themes, the mouse color updates automatically through Omarchy's `theme-set.d` hook.
  * Rainbow 360° mode with cycle speed tuning (`1s` to `10s`).
  * Brightness slider (0% to 100%).
* **Zero-daemon hardware architecture**
  * All lighting modes run inside the mouse's own MCU.
  * No background polling daemons or continuous RF writes, which protects wireless battery life and avoids RF transceiver sleep timeouts.

---

## Prerequisites

* **`libratbag` / `ratbagd`** (required): provides DPI, polling rate, and lighting control.
* **`python`** (required): runs the control script.
* **Hyprland** (`hyprctl`): needed for the pointer-speed slider.
* **`solaar`** (optional, recommended for Logitech wireless mice): needed for a real battery percentage on mice like the G305, where the kernel only exposes a coarse level. Only the Solaar CLI is used, so the GUI does not need to be running.
* **`upower`** (optional): secondary battery source, normally already installed.

On Arch Linux / Omarchy:

```bash
sudo pacman -S libratbag python solaar
sudo systemctl enable --now ratbagd.service
```

Verify that `ratbagd` is running and can see your mouse before installing the plugin:

```bash
systemctl is-active ratbagd     # should print "active"
ratbagctl list                  # should list your mouse
```

---

## Installation

### Option 1: Install via Omarchy (recommended)

Install directly with Omarchy's plugin manager:

```bash
omarchy plugin add https://github.com/MagosOptimistPrime/omarchy-mouse-plugin.git --enable
omarchy restart shell
```

If you are using a fork (for example, one that contains the G305 battery fallback), replace the URL with your fork's:

```bash
omarchy plugin add https://github.com/<your-username>/omarchy-mouse-plugin.git --enable
```

### Option 2: Local installation

Clone the repository and run the installer script:

```bash
git clone https://github.com/MagosOptimistPrime/omarchy-mouse-plugin.git
cd omarchy-mouse-plugin
chmod +x install.sh
./install.sh
```

The installer:

1. Checks for `ratbagctl`, `python3`, and `hyprctl`.
2. Copies `omarchy-mouse-control` to `~/.local/bin/omarchy-mouse-control`.
3. Installs the plugin files (`manifest.json`, `Panel.qml`, `omarchy-mouse-control`) to `~/.config/omarchy/plugins/optimistprime.mouse/`.
4. Installs the theme-sync hook at `~/.config/omarchy/hooks/theme-set.d/mouse-theme-sync`.
5. Validates the plugin and rescans the shell plugins.

Then enable the widget on the **right** side of the bar and reload the shell:

```bash
omarchy plugin enable optimistprime.mouse right
omarchy restart shell
```

Alternatively, add it by hand to `~/.config/omarchy/shell.json`, under `bar.layout.right`. Keep your existing entries and just add the new one:

```json
{
  "bar": {
    "layout": {
      "right": [
        "optimistprime.mouse",
        "omarchy.bluetooth",
        "omarchy.network",
        "omarchy.audio"
      ]
    }
  }
}
```

### Verify the installation

```bash
omarchy plugin list                       # optimistprime.mouse should be listed and enabled
omarchy-mouse-control state               # prints JSON with name, DPI, battery, ...
```

---

## Uninstallation

### 1. Remove the plugin from Omarchy

```bash
omarchy plugin list
omarchy plugin remove optimistprime.mouse
```

### 2. Remove the widget from your bar layout

If you added the widget by hand, delete `"optimistprime.mouse",` from `bar.layout.right` in `~/.config/omarchy/shell.json`, then reload the shell:

```bash
omarchy restart shell
```

### 3. Remove the files installed by `install.sh` (Option 2 only)

If you installed with `./install.sh`, these files remain after removing the plugin. Delete them to finish a clean uninstall:

```bash
rm -f  ~/.local/bin/omarchy-mouse-control
rm -rf ~/.config/omarchy/plugins/optimistprime.mouse
rm -f  ~/.config/omarchy/hooks/theme-set.d/mouse-theme-sync
```

### 4. Optional: remove saved settings and cache

These hold your lighting settings and the cached battery reading. Keep them if you plan to reinstall:

```bash
rm -f ~/.config/omarchy/mouse-lighting.json
rm -f ~/.cache/omarchy-mouse-battery.json
```

Your DPI, polling rate, and lighting changes are stored on the mouse itself (onboard profile 0), so removing the plugin does not reset them. Use Piper or `ratbagctl` if you want to change them back.

---

## Troubleshooting

### The DPI buttons do nothing, or the widget shows "no mouse"

The plugin talks to the mouse through `ratbagd`. Check it first:

```bash
sudo systemctl status ratbagd
```

If it is inactive (dead) or disabled:

```bash
sudo systemctl enable --now ratbagd
```

If it is running but the plugin still cannot control the mouse, **restart `ratbagd`** and check that it sees your device:

```bash
sudo systemctl restart ratbagd
systemctl is-active ratbagd            # should print "active"
ratbagctl list                         # should list your mouse
```

Then read the current DPI straight from the device (replace `<id>` with the name shown by `ratbagctl list`, for example `sinuous-abyss`):

```bash
ratbagctl <id> profile 0 dpi get       # should print something like "800dpi"
```

If that works, reload the shell:

```bash
omarchy restart shell
```

If `ratbagd` fails to start or restarts repeatedly, read its log:

```bash
journalctl -u ratbagd -b --no-pager | tail -n 50
```

Also check that:

* **The mouse is awake.** Wireless mice such as the G305 sleep when idle. Move the mouse or click, then try again.
* **The mouse is supported.** `ratbagctl list` must show it. If it does not, the device is not in `libratbag`'s database (check Piper's supported devices list). Replugging the USB receiver and restarting `ratbagd` sometimes helps.
* **Only one tool is configuring the mouse.** Close Piper while using the plugin, since two programs writing settings at once can conflict.

### Test the backend script directly

This separates problems in the script from problems in the widget:

```bash
omarchy-mouse-control state            # full device state as JSON
omarchy-mouse-control set-dpi 1600     # should print "ok"
ratbagctl <id> profile 0 dpi get       # should now show 1600dpi
```

If `omarchy-mouse-control` is "command not found", make sure `~/.local/bin` is on your `PATH`.

### Battery is missing or shows -1

Check what each battery source reports:

```bash
ls /sys/class/power_supply/
grep . /sys/class/power_supply/hidpp_battery*/{capacity,capacity_level,status} 2>&1
upower -e
solaar show
```

* If `solaar show` prints a `Battery: NN%` line for your mouse but the widget does not, you are running a version of the script without the Solaar fallback, or `solaar` is not installed. Install it with `sudo pacman -S solaar`, then reinstall the plugin.
* If `capacity` is missing and only `capacity_level` (for example `Normal`) exists, the kernel is exposing a coarse level only. This is normal for the G305, and the Solaar fallback exists for exactly this case.
* If the mouse is asleep or out of range, the widget keeps the last known value (cached for up to a minute) until it wakes.
* Wireless Logitech mice report battery in coarse steps. The percentage can jump, for example from 50% to 30%, instead of ticking down by 1%.

### The widget does not appear in the bar

```bash
omarchy plugin list                                 # is optimistprime.mouse enabled?
omarchy plugin validate ~/.config/omarchy/plugins/optimistprime.mouse
omarchy plugin enable optimistprime.mouse right
omarchy restart shell
qs log -p "$OMARCHY_PATH/shell" --tail 100          # look for QML errors
```

### RGB lighting resets or does not apply

Lighting writes are pinned to onboard profile 0. If your mouse shows the wrong profile or lighting snaps back to factory red, run:

```bash
ratbagctl <id> profile active set 0
```

then apply the color again from the widget.

### Quick reset checklist

When in doubt, run these in order:

```bash
sudo systemctl restart ratbagd
ratbagctl list
omarchy-mouse-control state
omarchy restart shell
```

---

## Command line interface (`omarchy-mouse-control`)

The backend script also provides a standalone CLI for scripting, hotkeys, or terminal inspection:

```bash
# Query complete device state as JSON
omarchy-mouse-control state

# Set DPI
omarchy-mouse-control set-dpi 1200

# Set polling report rate (Hz)
omarchy-mouse-control set-rate 1000

# Set Hyprland pointer sensitivity
omarchy-mouse-control set-sensitivity 0.35

# Lighting controls
omarchy-mouse-control set-lighting-toggle 1
omarchy-mouse-control set-lighting-mode color             # color | rainbow | off
omarchy-mouse-control set-lighting-color "#eb8b54" orange # hex [slot_name]
omarchy-mouse-control set-lighting-breathe 1              # 1 = breathe, 0 = solid
omarchy-mouse-control set-lighting-duration 8             # 1 to 10 seconds (or ms)
omarchy-mouse-control set-lighting-brightness 200         # 0 to 255
omarchy-mouse-control sync-theme                          # Synchronize active theme slot
```

---

## Architecture & design notes

* **Self-contained execution:** `Panel.qml` resolves the location of `omarchy-mouse-control` in its own directory, so the plugin works as soon as it is installed without requiring root.
* **Profile 0 pinning:** modern Logitech mice have multiple onboard memory slots, some of which may be disabled. All writes are pinned to `profile 0`, which prevents lighting from resetting to the factory red.
* **Sleep immunity:** wireless mice enter a low-power RF sleep state after idling for about 1 to 2 minutes. Using one-shot hardware commands and Omarchy `theme-set.d` hooks, instead of continuous write loops, lets the mouse sleep naturally and keeps the 2.4 GHz Lightspeed link stable.
* **Battery source chain:** exact percentages win over approximate levels. The script tries kernel sysfs, then UPower, then the Solaar CLI, and uses the first exact percentage it finds. If no source has an exact value, it falls back to an approximate level ("Normal" is about 60%). Solaar results are cached for 60 seconds because `solaar show` is slow.

---

## License

Distributed under the **MIT License**. See [`LICENSE`](LICENSE) for details.

Developed by **Jason Stewart (Optimist Prime)**.
