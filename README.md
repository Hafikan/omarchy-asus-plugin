# G16 Control

An [Omarchy](https://omarchy.org) bar widget for the **ASUS ROG Zephyrus G16 (GA605)**: everything the laptop's hardware can do, in one panel.

> **Part of [omarchy-asus](https://github.com/omarchy-asus/omarchy-asus)**: kernel, NVIDIA and Omarchy fixes for ASUS ROG laptops, as a signed pacman repo. This widget covers the controls; omarchy-asus fixes the rest. That includes the Fn+Super Win-lock keys, the touchpad, sleep with an HDMI monitor, the NVIDIA hibernate hang and USB-C charging profiles. See [what ships](https://github.com/omarchy-asus/omarchy-asus#what-ships) and the [install guide](https://github.com/omarchy-asus/omarchy-asus/blob/main/LAPTOP-SETUP.md).

## What it does

- **Keyboard backlight:** a level for the barrel charger and one for battery / USB-C, applied on every plug and unplug; colour; and the boot/awake/sleep/shutdown lighting.
- **Lid Slash LEDs:** on/off, brightness, pattern (with keyboard navigation: arrows or h/j/k/l, Enter), and when it shows.
- **Internal panel refresh rate** per power source, a gaming rate, or pinned (60/240/Auto). It's set again after a monitor hotplug or `hyprctl reload`.
- **Screen backlight:** optionally dimmed on leaving the barrel charger, and restored on return.
- **CPU power limits per charger** (USB-C, a weak third-party barrel), and panel overdrive per source.
- **Battery charge limit** (60/80/100 %).
- **Fn+Super (Win-lock) mode:** toast or auto-unlock, if the omarchy-asus Win-lock helper is installed.

It tells the 220 W barrel, USB-C chargers and a flapping third-party barrel apart, and gives each its own settings.

## Install

```bash
omarchy-plugin-add https://github.com/omarchy-asus/g16-control --enable
```

Updates: `omarchy-plugin-update omarchy-asus.g16`. The shell reloads the widget by itself.

**Needs:** stock Omarchy (Python and PyGObject are already there) and `asusd` (`asusctl`) for the keyboard, Slash, charge limit and power limits. Without asusd those sections hide. The Fn+Super switch needs `asus-rog-nkey-winlock` and the Win-lock kernel patch, both from [omarchy-asus](https://github.com/omarchy-asus/omarchy-asus).

## Per power source: keyboard, refresh rate, panel overdrive

The widget applies these itself on every plug and unplug (it replaces
`omarchy-power-tune`, which was written for this widget). They live in
`~/.config/omarchy/power-tune.conf`, as plain `KEY=VALUE` lines (never run as a
script):

| key | values | what |
|---|---|---|
| `KBD_AC`, `KBD_BATTERY` | `off` `low` `med` `high` | keyboard backlight; the panel's "On AC" / "On battery" rows write these |
| `HZ_AC`, `HZ_BATTERY`, `HZ_GAMING` | Hz | the internal panel's refresh rate |
| `PANEL_OVERDRIVE_AC`, `PANEL_OVERDRIVE_BATTERY` | `0` `1` | asus-armoury panel overdrive, through asusd |
| `INTERNAL_OUTPUT` | `eDP-1`, or `desc:<EDID description>` | which output is the panel (default: the first `eDP`) |
| `BITDEPTH` | `8` `10` | carried on every modeset (default: the panel's current format) |
| `BRIGHT_BATTERY` | % | on leaving the barrel, dim the screen to this if it's brighter (once, not a cap); back on the barrel the old level returns unless you've gone brighter |
| `BRIGHT_DEVICE` | e.g. `amdgpu_bl1` | the backlight for that (default: the first one that isn't `nvidia_*`), set through logind |

| `PPT_AC`, `PPT_USBC`, `PPT_WEAK_BARREL` | `"SPL SPPT FPPT"` in W, e.g. `"25 30 40"` | CPU power limits on that source (asus-armoury, through asusd), clamped to the firmware's range |
| `BOOST_AC`, `BOOST_USBC`, `BOOST_WEAK_BARREL` | W | NVIDIA Dynamic Boost on that source |

"AC" means the barrel charger. A USB-C charger gets the battery settings:
UPower calls both chargers AC, so the widget tells them apart by asus-armoury's
`charge_mode` (bit 0 barrel, bit 1 USB-C). A barrel that drops out three or
more times in 30 s (a third-party adapter that can't carry the load) is a
**weak barrel**: it keeps the AC settings and gets `PPT_WEAK_BARREL` until it's
unplugged for good. Power-source changes apply once the source has been stable
for 1.5 s, so a flapping adapter doesn't flip the refresh rate back and forth.

**Power limits and the fan curve.** On the GA605KM (BIOS 307) the firmware only
applies PPT limits while the CPU custom fan curve is on; in auto fan mode a
written limit is stored but ignored. So on a source with limits the widget
enables asusd's stored CPU curve for the current profile, then writes the
limits; leaving it turns the curve off and writes the defaults back. The fans
follow that curve while limits apply. The values the G16 used with rog-power:
`PPT_USBC="25 30 40"`, `PPT_WEAK_BARREL="45 55 65"`, `BOOST_WEAK_BARREL=5`
(the OEM barrel left at stock).

A key that isn't set is left alone, so with no file nothing is enforced. The
refresh pin (the panel's 60/240/Auto switch, stored in
`~/.local/state/omarchy/power-tune-hz`) wins over gaming mode
(`power-tune-gaming`), which wins over the power source. For a key binding:

```bash
python3 ~/.config/omarchy/plugins/omarchy-asus.g16/bridge/g16_bridge.py gaming toggle
python3 ~/.config/omarchy/plugins/omarchy-asus.g16/bridge/g16_bridge.py hz 60   # or: hz auto
```

A monitor hotplug or `hyprctl reload` re-applies `monitors.lua`, which puts the
panel back on its native rate; the widget sets the rate again after both.

**Coming from `omarchy-power-tune`:** the settings file and state files are the
same, so nothing needs converting. Stop the old service so two programs don't
set the same things: `systemctl --user disable --now power-tune.service`. Its
overdrive and refresh jobs are covered here; anything else you added to it is
not.

The widget also works without asusd: the keyboard and Slash tabs hide, and the
refresh rate, charge limit and Fn+Super switch stay.

## How it works: one Python bridge, nothing polls

`Service.qml` (manifest kind `service`) holds the widget's state once, however
many bars show it. Everything that touches the machine is one process,
`bridge/g16_bridge.py`. It uses only the Python standard library and
PyGObject's Gio, both on stock Omarchy, and it spawns nothing. It talks to:
- asusd and UPower over D-Bus, and follows their `PropertiesChanged` signals;
- Hyprland over its IPC sockets, including the event socket;
- the keyboard LED's `brightness_hw_changed` (Fn+F2/F3) and the settings
  files, through inotify.

It sends a JSON state line whenever something changes and takes commands on
stdin. Under a shell that doesn't hand out services, each widget starts its own
copy after 10 s.


## Tests

```bash
tests/bridge_test.py      # the bridge against fake asusd, UPower, logind and systemd on private D-Bus buses, and a fake Hyprland
node tests/cursor.test.js # the Slash pattern grid's keyboard cursor
tests/service.test.sh     # Service.qml headless in quickshell (skips without qs)
```

## License

MIT. Part of [omarchy-asus](https://github.com/omarchy-asus/omarchy-asus).
