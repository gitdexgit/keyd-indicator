**[LOGIC]**
`{R0: keyd; R1: yad; R2: libnotify; R3: python; R4: systemd; R5: i3wm}` | `R0..R5 => Generic Template`
**[/LOGIC]**

**[BLUEPRINT]**
1. [LOGIC → ANS] 100% — Generalized README with placeholders.
Selected: [LOGIC → ANS]

**[ANS]**

# keyd-indicator

Real-time tray icons and colored notifications for `keyd` layers. Optimized for Arch Linux + i3wm.

## 1. Prerequisites
Install dependencies:
```bash
# Arch
sudo pacman -S keyd yad libnotify python
```

## 2. Permissions (Socket Access)
Allow your user to listen to `keyd` events without `sudo`.
1. **Create Group**: `sudo groupadd keyd && sudo usermod -aG <YOUR_USERNAME> keyd`
2. **Set Socket Permissions**: Create `/etc/tmpfiles.d/keyd.conf`:
   ```text
   # Type Path               Mode UID  GID  Age Argument
   z     /run/keyd.socket    0660 root keyd -   -
   ```
3. **Apply**: `sudo systemd-tmpfiles --create /etc/tmpfiles.d/keyd.conf`

## 3. The Indicator Script
Save to `~/scripts/keyd/keyd_indicator.py`:
```python
import subprocess, os

# --- USER CONFIG ---
USER = os.getlogin()
UID = str(os.getuid()) # Usually 1000
# Ensure DISPLAY and DBUS match your session
ENV = {**os.environ, "DISPLAY": ":0", "DBUS_SESSION_BUS_ADDRESS": f"unix:path=/run/user/{UID}/bus"}

# Icon names from your system theme or absolute paths to .png
ICONS = {"normal": "media-record", "prog": "input-keyboard", "emacs": "emacs"}
COLORS = {"normal": "#FFFFFF", "prog": "#00FF00", "emacs": "#FF69B4", "red": "#FF0000"}

class KeydIndicator:
    def __init__(self):
        self.emacs_proc = None
        self.main_yad = subprocess.Popen(
            ["yad", "--notification", "--listen", "--image", ICONS["normal"], "--text", "Normal"],
            stdin=subprocess.PIPE, text=True, env=ENV
        )

    def notify(self, title, msg, color, icon):
        # Async notification (fire and forget)
        subprocess.Popen([
            "notify-send", "-t", "400",
            "-h", f"string:bgcolor:{color}",
            "-h", "string:frcolor:#000000",
            "-h", "string:fgcolor:#000000",
            "-i", icon, title, msg
        ], env=ENV)

    def update_main(self, icon, tooltip):
        if self.main_yad.stdin:
            self.main_yad.stdin.write(f"icon:{icon}\n")
            self.main_yad.stdin.write(f"tooltip:{tooltip}\n")
            self.main_yad.stdin.flush()

    def run(self):
        # stdbuf -oL forces line-buffering for zero latency
        proc = subprocess.Popen(["stdbuf", "-oL", "keyd", "listen"], stdout=subprocess.PIPE, text=True, bufsize=1)
        if proc.stdout:
            for line in proc.stdout:
                l = line.strip()
                if l == "+programmer_mode":
                    self.update_main(ICONS["prog"], "Programmer Mode")
                    self.notify("Keyd", "Programmer ON", COLORS["prog"], ICONS["prog"])
                elif l == "-programmer_mode":
                    self.update_main(ICONS["normal"], "Normal Mode")
                    self.notify("Keyd", "Normal", COLORS["normal"], ICONS["normal"])
                elif l == "+keyd_emacs":
                    if not self.emacs_proc:
                        self.emacs_proc = subprocess.Popen(["yad", "--notification", "--image", ICONS["emacs"], "--text", "Emacs"], env=ENV)
                    self.notify("Keyd", "Emacs ON", COLORS["emacs"], ICONS["emacs"])
                elif l == "-keyd_emacs":
                    if self.emacs_proc:
                        self.emacs_proc.terminate()
                        self.emacs_proc = None
                    self.notify("Keyd", "Emacs OFF", COLORS["red"], ICONS["normal"])

if __name__ == "__main__":
    subprocess.run(["pkill", "yad"]) # Kill orphaned trays
    KeydIndicator().run()
```

## 4. The Reload Script
Save to `~/scripts/keyd/keyd_reload.sh`:
```bash
#!/bin/bash
# Replace <YOUR_USERNAME> and <YOUR_UID>
USER_NAME="<YOUR_USERNAME>"
USER_ID="<YOUR_UID>"

keyd reload

# Notify user of reload
sudo -u $USER_NAME DISPLAY=:0 DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$USER_ID/bus \
    notify-send -t 400 -h string:bgcolor:#FFFF00 -h string:fgcolor:#000000 \
    "Keyd" "Config Reloaded" &

# Restart the tray service to reset state
systemctl --user restart keyd-indicator.service
```
`chmod +x ~/scripts/keyd/keyd_reload.sh`

## 5. Systemd Service
Save to `~/.config/systemd/user/keyd-indicator.service`:
```ini
[Unit]
Description=Keyd Tray Indicator
After=graphical-session.target

[Service]
# Update path to your script
ExecStart=/usr/bin/python3 /home/<YOUR_USERNAME>/scripts/keyd/keyd_indicator.py
Restart=always
Environment=DISPLAY=:0
Environment=DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/<YOUR_UID>/bus

[Install]
WantedBy=graphical-session.target
```
**Enable**: `systemctl --user enable --now keyd-indicator.service`

## 6. Keyd Configuration
Update `/etc/keyd/default.conf`:
```ini
[extend_caps]
# Toggle programmer mode
f10 = toggle(programmer_mode)
# Reload everything
f12 = command(/home/<YOUR_USERNAME>/scripts/keyd/keyd_reload.sh)
```

## 7. Features
*   **Main Tray**: Toggles between Red Dot (Normal) and Keyboard (Programmer).
*   **Emacs Tray**: Independent Pink icon appears only when Emacs layer is active.
*   **Popups**: High-contrast, color-coded notifications (400ms duration).
*   **Performance**: Event-driven. 0% CPU usage.

**[/ANS]**
