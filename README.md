# Monitor KVM Switch

Automatically switches your monitors' physical inputs (DDC/CI) to follow
Synergy as you move your mouse between computers — so when you switch which
PC has keyboard/mouse focus, the monitors switch to show that PC too,
without touching a physical KVM button.

It watches Synergy's log file for screen-switch events and drives each
monitor's `Input Select` (VCP 0x60) over DDC/CI accordingly.


- **`monitor_kvm_switch.exe`** — the background watcher. Runs from the tray,
  tails the Synergy log, and does the actual switching.
- **`gui.exe`** — the settings editor. Lets you map hosts to monitor inputs,
  test values live, identify which physical monitor is which, and edit
  everything `monitor_kvm_switch.exe` reads.

Each can launch the other: the GUI has a **Launch watcher** button, and the
tray has an **Open Settings** item.

## Requirements

- Windows, Rust with the MSVC toolchain (`rustup default stable-msvc`)
- Monitors that support DDC/CI (enable it in the monitor's OSD if it's off)
- Synergy (or compatible fork) already set up and logging to
  `C:\Program Files\Synergy\synergy-daemon.log` (configurable)


## First-time setup

1. Run `monitor_kvm_switch.exe` once with no `config.toml` present — it
   drops into an interactive console wizard: it lists every Synergy screen
   name it's ever seen in the log, asks which monitors (if any) each host
   needs to switch, and tests each candidate DDC/CI value by switching to
   it for 10 seconds so you can visually confirm before it's recorded.
2. Once `config.toml` exists, running the exe again starts the real
   watcher instead.
3. For anything beyond the initial setup, use `gui.exe` — it's the fuller
   editor (see below) and is what you'll come back to most.

Config lives at `%ProgramData%\MonitorKvmSwitch\config.toml`, shared by
both binaries.

## Using the GUI

- **Hosts sidebar** — every host you've configured, with a status dot:
  green (fully decided for every monitor), yellow (partial), gray (empty).
  Hosts seen in the Synergy log but not yet configured show separately so
  you can add them with one click.
- **Settings card** — settle delay (ms a screen must stay active before
  switching; `0` = instant) and the Synergy log path.
- **Monitors card** — one box per detected monitor: name (editable,
  auto-filled from EDID where the monitor provides it), physical position
  (left/right/top/bottom, from Windows), full serial number, manufacturer
  code, current input, and an **Identify** button.
- **Host detail** — for the selected host, each monitor is one of three
  states:
  - *not set* — undecided; **Use current** records whatever the monitor is
    already showing (for the host you're currently on), or **Test N**
    switches to a candidate value for 10 seconds so you can confirm it
    before committing
  - *no switch needed* — a deliberate decision that this host doesn't need
    this monitor to change (e.g. it has its own dedicated screen)
  - *input N* — an active mapping, with a checkbox to enable/disable it
    without losing the value
- **Identify** — pops up a real window on the actual physical monitor for
  5 seconds, showing its index, name, current input + connection type
  (e.g. "Input 15 • DisplayPort 1"), and full serial number. This is the
  reliable way to tell two identical monitors apart — position/EDID
  detection are both best-effort guesses, Identify is ground truth.

## config.toml

```toml
log_path = 'C:\Program Files\Synergy\synergy-daemon.log'
settle_ms = 0

[monitor_names]
0 = "ASUS VP28U"
1 = "ASUS VP28U"

[inputs.xxvii-253e9ec9]
0 = 15
1 = 15

[inputs.neon-88954047]
1 = 18

[disabled]
# host = ["monitor index", ...]  -- has a value but temporarily off

[no_switch]
surface3-65ce4094 = ["0", "1"]
```

Keys under `[inputs.*]`, `[disabled]`, and `[no_switch]` are **monitor
indices**, not names — this matters because two identical monitors will
often get the same auto-detected name, and names were the original (buggy)
join key in an earlier version of this schema. `monitor_names` is purely a
display label and can safely be duplicated.

Host names must be the literal Synergy client ID as it appears in the log
(e.g. `xxvii-253e9ec9`), not a nickname.

## Tray menu

Grouped by host, since the underlying tray library (`tray-item`) has no
submenu support:

```
Watching for switches...
---
xxvii-253e9ec9                    (grayed header, not clickable)
  [🟢] Monitor 0: ASUS VP28U
  [🟢] Monitor 1: ASUS VP28U
---
surface3-65ce4094
  [🟢] Monitor 0: ASUS VP28U
  [⚪] Monitor 1: ASUS VP28U
---
Reload config
Open Settings
Open config folder
---
Quit
```

Clicking a monitor toggles switching for that host+monitor combination for
the current session only (not saved) — for a permanent change, use the
GUI's checkbox and save.

## Known limitations

- **Tray submenus/icons**: `tray-item` supports neither. The host grouping
  is a flat list with grayed headers and indentation, not real nested
  menus; the 🟢/⚪ circles are the closest thing to per-item icons this
  crate allows.
- **VCP code 19+ isn't labeled**: codes 1–18 are standardized by MCCS and
  shown with real names (DisplayPort, HDMI, etc); anything above that
  varies by manufacturer, so it's deliberately left unlabeled rather than
  guessed.
- **EDID name/serial are optional fields**: not every monitor sets them.
  When absent, the name falls back to `monitor_N` (editable) and the
  serial/manufacturer fields just don't show.
- **Position and monitor-index correlation are both ordinal guesses**:
  `ddc-hi`'s enumeration order, `EnumDisplayMonitors`' order, and the
  registry's EDID enumeration order are three independent mechanisms that
  usually — but aren't guaranteed to — agree on which index is which
  physical monitor. Identify is the way to verify this for certain.

## Troubleshooting

- **Nothing switches**: check the host name in `config.toml` matches the
  Synergy log's client ID exactly (case-sensitive).
- **Wrong monitor switches, or one switches when it shouldn't**: use
  Identify on both monitors and re-check which index is actually which
  physical screen before trusting position labels.
- **Tray icon shows the generic Windows icon instead of the custom one**:
  harmless fallback — check the console/log output for why
  `LoadImageW` failed (usually a missing/corrupted `tray_icon.ico` next to
  `config.toml`, which gets rewritten fresh on every launch anyway).
