# Native Gamepad/Controller Navigation for Pixie SDDM

A guide to adding full gamepad navigation (D-pad, buttons) to the [Pixie SDDM](https://github.com/xCaptaiN09/pixie-sddm) login screen — useful for TV/HTPC setups (e.g. alongside KDE Plasma Bigscreen) where there's no keyboard or mouse attached.

> **Note:** As of writing, Pixie SDDM does not natively support gamepad input, and its user/session dropdown can only be opened by mouse click. This guide patches around both limitations. If native support ever lands upstream, most of this becomes unnecessary — check the [pixie-sddm repo](https://github.com/xCaptaiN09/pixie-sddm) first before following this guide.

## What this achieves

- D-pad → Arrow keys (navigate the login screen and any open dropdown)
- Cross (✕) → Enter (confirm / log in)
- Circle (○) → Escape (back / cancel)
- Triangle (△) → Opens the user profile dropdown
- Fully automatic: starts before login, stops itself once you're actually logged in, so your controller behaves normally in games afterward

Tested on CachyOS (Arch-based) with Plasma 6.7, SDDM 0.21 (Qt6), Pixie SDDM v3.0, and a PS4 DualShock controller over Bluetooth. Should generalize to any Arch-based distro and most Xbox/PlayStation-style controllers with minor button-code adjustments.

---

## Why this is needed

SDDM's Qt-based greeter only listens for keyboard, mouse, and touch input — there's no built-in gamepad/joystick support at the display-manager level. Tools like AntiMicroX can't help here either, since they map gamepad input into an *active desktop session*, and there is no session running yet at the login screen.

The fix is to map gamepad input directly into a **virtual keyboard device** at the kernel level using `evdev` + `uinput`. This makes the controller's button presses appear as real keyboard key presses system-wide — including to SDDM's greeter, which has no idea the input didn't come from a physical keyboard.

Separately, Pixie SDDM's profile-picker dropdown is only wired to a `MouseArea` click in its QML source — there's no keyboard shortcut for it out of the box. We patch in a modifier-key shortcut for that specifically, since a single letter key gets swallowed by the password text field.

---

## Prerequisites

- Arch-based distro with SDDM and Pixie SDDM already installed and working
- `python-evdev` and Python 3
- A gamepad the kernel already detects (check with `cat /proc/bus/input/devices`)
- Root/sudo access

```bash
sudo pacman -S python python-pip
python3 -m pip install evdev --break-system-packages
```

Make sure the `uinput` kernel module is loaded and loads automatically on boot:

```bash
sudo modprobe uinput
echo "uinput" | sudo tee /etc/modules-load.d/uinput.conf
```

---

## Step 1 — Identify your gamepad's input device and button codes

Connect your controller (pair over Bluetooth if needed via `bluetoothctl`), then find it:

```bash
cat /proc/bus/input/devices | grep -A5 -i "wireless\|controller"
```

Note the `Handlers=` line — you want the one with an `eventN` entry that is the main button/axis device (not the motion sensors or touchpad sub-devices some controllers expose separately).

Confirm the exact button/axis codes it sends:

```bash
python3 -c "
import evdev
dev = evdev.InputDevice('/dev/input/eventN')  # replace N
print(dev)
print(dev.capabilities(verbose=True))
"
```

For a standard PS4 controller, the D-pad shows up as `ABS_HAT0X` / `ABS_HAT0Y` (values -1/0/1), and face buttons as `BTN_SOUTH` (✕), `BTN_EAST` (○), `BTN_NORTH` (△), `BTN_WEST` (▢). Xbox controllers report the same standard `BTN_SOUTH`/`BTN_EAST`/etc. codes, so this mapping should work unmodified — just double check with the command above.

---

## Step 2 — Patch Pixie SDDM's QML to add a dropdown shortcut

Pixie SDDM's user-switcher popup only opens via mouse click:

```qml
MouseArea {
    id: userClickArea
    anchors.fill: parent
    onClicked: userPopup.open()
}
```

We add a keyboard shortcut for it. A plain letter key (like `P`) **will not work** — the password field has focus by default and swallows plain character keys as text input before any global `Shortcut` fires. Use a modifier combo instead:

```bash
sudo python3 << 'PYEOF'
path = "/usr/share/sddm/themes/pixie/Main.qml"
with open(path) as f:
    content = f.read()

old = '''    Shortcut {
        sequences: ["Return", "Enter"]
        enabled: loginState.visible
        onActivated: container.doLogin()
    }'''

new = old + '''

    Shortcut {
        sequence: "Ctrl+Alt+P"
        enabled: loginState.visible
        onActivated: userPopup.open()
    }'''

if old not in content:
    print("ERROR: anchor text not found — theme file may have changed, patch manually")
else:
    content = content.replace(old, new, 1)
    with open(path, "w") as f:
        f.write(content)
    print("SUCCESS: Ctrl+Alt+P shortcut inserted")
PYEOF
```

Verify it landed as its own clean block (not nested inside another `Shortcut`):

```bash
grep -B7 -A2 "Ctrl+Alt+P" /usr/share/sddm/themes/pixie/Main.qml
```

You should see the Return/Enter block's closing `}`, a blank line, then the new `Shortcut { sequence: "Ctrl+Alt+P" ... }` block as a sibling.

> **Tip:** Always work from a clean backup of `Main.qml` (copy it right after running Pixie's `install.sh`) before patching, so you can restore instantly if something goes wrong. Manual editing with `sed`/`nano` is error-prone here — mismatched braces will crash the whole theme (black screen or QML errors) rather than fail gracefully.

---

## Step 3 — Write the gamepad-to-keyboard mapper

Save this as `/usr/local/bin/gamepad-to-keyboard.py`:

```python
#!/usr/bin/env python3
import time
import subprocess
import evdev
from evdev import UInput, ecodes as e

GAMEPAD_NAME = "Wireless Controller"  # match your controller's reported name

KEY_MAP = {
    e.ABS_HAT0X: {-1: e.KEY_LEFT, 1: e.KEY_RIGHT},
    e.ABS_HAT0Y: {-1: e.KEY_UP, 1: e.KEY_DOWN},
}
BTN_MAP = {
    e.BTN_SOUTH: e.KEY_ENTER,   # Cross / A
    e.BTN_EAST: e.KEY_ESC,      # Circle / B
}


def find_gamepad():
    for path in evdev.list_devices():
        dev = evdev.InputDevice(path)
        if dev.name == GAMEPAD_NAME:
            return dev
    return None


def has_active_session():
    """Returns True once a real user session (not the greeter) is active,
    so the mapper can stop and stay out of the way of normal gamepad use."""
    try:
        out = subprocess.check_output(
            ["loginctl", "list-sessions", "--no-legend"], text=True
        )
        for line in out.strip().splitlines():
            parts = line.split()
            if len(parts) >= 3:
                user = parts[2]
                if user not in ("sddm", "gdm", "lightdm"):
                    session_id = parts[0]
                    state = subprocess.check_output(
                        ["loginctl", "show-session", session_id, "-p", "State"],
                        text=True,
                    ).strip()
                    if "State=active" in state:
                        return True
    except Exception:
        pass
    return False


def send_combo(ui, keys):
    for k in keys:
        ui.write(e.EV_KEY, k, 1)
    ui.syn()
    for k in reversed(keys):
        ui.write(e.EV_KEY, k, 0)
    ui.syn()


def main():
    print("Waiting for gamepad...")
    dev = None
    while dev is None:
        dev = find_gamepad()
        if dev is None:
            time.sleep(2)

    print(f"Found gamepad: {dev.path}")
    dev.grab()  # exclusive control pre-login only

    ui = UInput(name="gamepad-virtual-keyboard")

    try:
        for evdev_event in dev.read_loop():
            if has_active_session():
                print("User session active, stopping mapper.")
                break

            if evdev_event.type == e.EV_ABS and evdev_event.code in KEY_MAP:
                mapping = KEY_MAP[evdev_event.code]
                if evdev_event.value in mapping:
                    key = mapping[evdev_event.value]
                    ui.write(e.EV_KEY, key, 1)
                    ui.syn()
                    ui.write(e.EV_KEY, key, 0)
                    ui.syn()

            elif evdev_event.type == e.EV_KEY and evdev_event.code == e.BTN_NORTH:
                if evdev_event.value == 1:  # Triangle pressed
                    print("Triangle pressed -> Ctrl+Alt+P")
                    send_combo(ui, [e.KEY_LEFTCTRL, e.KEY_LEFTALT, e.KEY_P])

            elif evdev_event.type == e.EV_KEY and evdev_event.code in BTN_MAP:
                key = BTN_MAP[evdev_event.code]
                ui.write(e.EV_KEY, key, evdev_event.value)
                ui.syn()

    finally:
        try:
            dev.ungrab()
        except Exception:
            pass
        ui.close()


if __name__ == "__main__":
    main()
```

Set `GAMEPAD_NAME` to match what you found in Step 1, then make it executable:

```bash
sudo chmod +x /usr/local/bin/gamepad-to-keyboard.py
```

---

## Step 4 — Run it as a systemd service

Create `/etc/systemd/system/gamepad-login-nav.service`:

```ini
[Unit]
Description=Gamepad to keyboard mapper for SDDM login screen
After=multi-user.target
Before=sddm.service

[Service]
Type=simple
ExecStart=/usr/bin/python3 /usr/local/bin/gamepad-to-keyboard.py
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

Enable and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now gamepad-login-nav.service
```

Check it's running and picked up the controller:

```bash
sudo journalctl -u gamepad-login-nav.service -f
```

You should see `Waiting for gamepad...` then `Found gamepad: /dev/input/eventN`. Since a real session is likely active while you test this over SSH, it'll immediately print `User session active, stopping mapper.` and exit — that's correct behavior, not a bug. It stays running for real once you're sitting at the actual login screen with no session active.

---

## Step 5 — Test at the actual login screen

Reboot to land on the greeter with no active session:

```bash
sudo reboot
```

At the login screen:
- **D-pad** should move focus / navigate
- **Cross** should confirm / submit
- **Circle** should back out
- **Triangle** should pop open the user profile dropdown — once it's open, D-pad browses the list and Cross confirms the selection

---

## Troubleshooting

**Testing with `sddm-greeter --test-mode` fails with "could not connect to display"**
This happens whenever you're already inside an active graphical session (e.g. testing over SSH while logged into a desktop) — the test-mode greeter has nothing to attach to. It's not a sign anything is broken; do a full reboot to test against the real greeter instead.

**A single letter shortcut (like plain `P`) doesn't do anything, or types into the password box**
The password `TextField` has focus by default and consumes plain character keys as text before any global `Shortcut` sees them. Use a modifier combo (`Ctrl+Alt+P` or similar) instead — modifier combos aren't swallowed as text input.

**QML edits crash the theme (black screen, error text about unavailable types)**
Almost always mismatched braces from a manual edit — a `Shortcut {}` block accidentally nested inside another one. Restore from a clean backup of `Main.qml` and re-patch using a scripted text replacement (as in Step 2) rather than hand-editing with `nano`, which is much easier to get wrong on a long single-file QML theme.

**The mapper works, but pressing gamepad buttons in a game afterward double-fires input**
Confirm the `has_active_session()` check is actually working and the service is stopping once you log in (check `sudo journalctl -u gamepad-login-nav.service`). If it's not detecting your session correctly, check what `loginctl list-sessions` reports for your setup — some display setups report session state differently and the check may need adjusting.

**Missing QML modules (`org.kde.plasma.private.*` errors) when running Bigscreen alongside this**
Unrelated to gamepad navigation specifically — this happens when the Bigscreen shell is missing supporting Plasma packages (`powerdevil`, `plasma-nm`, `plasma-pa`, etc.). Install the full `plasma` group rather than just `plasma-bigscreen` to avoid this.

---

## Adapting this for other controllers or SDDM themes

- **Different controller:** Xbox controllers use the same standard `BTN_SOUTH`/`BTN_EAST`/`BTN_NORTH`/`BTN_WEST` codes as PlayStation controllers under Linux's evdev subsystem, so the button mapping should work unchanged — just update `GAMEPAD_NAME` to match what `evtest`/`evdev` reports for your device.
- **Different SDDM theme:** The systemd service and Python mapper are theme-agnostic — they just send keyboard events. Only Step 2 (the QML patch) is Pixie-specific; for another theme, look for how its user/session switcher is triggered and add an equivalent keyboard shortcut if one doesn't already exist.

---

## Credits

Built as a workaround while [Pixie SDDM](https://github.com/xCaptaiN09/pixie-sddm) by [xCaptaiN09](https://github.com/xCaptaiN09) doesn't yet have native gamepad support. If that changes, this whole guide becomes unnecessary — check the upstream repo first.
