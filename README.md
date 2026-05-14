**[LOGIC]**
`{R0: keyd; R1: yad; R2: libnotify; R3: python; R4: systemd; R5: i3wm}` | `R0..R5 => Integrated Solution`

**[/LOGIC]**

**[BLUEPRINT]**
1. [LOGIC → ANS] 100% — Step-by-step replication guide.
Selected: [LOGIC → ANS]

**[ANS]**

# Keyd State Indicator Setup (Arch/i3)

## 1. Prerequisites
Install core tools:
```bash
sudo pacman -S keyd yad libnotify python
```

## 2. Permissions (No-Sudo Socket Access)
Allow user `dex` to listen to `keyd` without root.
1. **Create Group**: `sudo groupadd keyd && sudo usermod -aG keyd dex`
2. **Set Socket Permissions**: Create `/etc/tmpfiles.d/keyd.conf`:
   ```text
   z /run/keyd.socket 0660 root keyd - -
   ```
3. **Apply**: `sudo systemd-tmpfiles --create /etc/tmpfiles.d/keyd.conf`

## 3. The Indicator Script
Save to `~/scripts/keyd/keyd_indicator.py`:
```python
import subprocess, os

USER = os.getlogin()
UID = str(os.getuid())
ENV = {**os.environ, "DISPLAY": ":0", "DBUS_SESSION_BUS_ADDRESS": f"unix:path=/run/user/{UID}/bus"}

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
    subprocess.run(["pkill", "yad"])
    KeydIndicator().run()
```

## 4. The Reload Script
Save to `~/scripts/keyd/keyd_reload.sh`:
```bash
#!/bin/bash
keyd reload
sudo -u dex DISPLAY=:0 DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1000/bus \
    notify-send -t 400 -h string:bgcolor:#FFFF00 -h string:fgcolor:#000000 \
    "Keyd" "Config Reloaded" &
systemctl --user restart keyd-indicator.service
```
`chmod +x ~/scripts/keyd/keyd_reload.sh`

## 5. Systemd Service (Persistence)
Save to `~/.config/systemd/user/keyd-indicator.service`:
```ini
[Unit]
Description=Keyd Tray Indicator
After=graphical-session.target

[Service]
ExecStart=/usr/bin/python3 /home/dex/scripts/keyd/keyd_indicator.py
Restart=always
Environment=DISPLAY=:0
Environment=DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1000/bus

[Install]
WantedBy=graphical-session.target
```
**Enable**: `systemctl --user enable --now keyd-indicator.service`

## 6. Keyd Configuration
Update `/etc/keyd/default.conf`:
```ini
[main]
# ... existing ...

[extend_caps]
f10 = toggle(programmer_mode)
f12 = command(/home/dex/scripts/keyd/keyd_reload.sh)
```
**Apply**: `sudo keyd reload`

## 7. Logic Summary
*   **Normal**: Red Dot tray. White popup.
*   **Programmer**: Keyboard tray. Green popup.
*   **Emacs**: Independent Pink tray. Pink popup (ON) / Red popup (OFF).
*   **Reload**: Yellow popup. Service restarts (resets state).
*   **Performance**: 0% CPU. Async notifications. Single instance via Systemd.

**[/ANS]**
