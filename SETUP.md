# Setup guide (for humans)

English | [Italiano](SETUP.it.md)

Step-by-step instructions to reproduce this boat server on an Android phone.
Written for a human holding the phone: several steps require the touchscreen
and cannot be done remotely.

## What you need

- Any Android phone (ours: Xiaomi Mi 9T, Android 11, 6 GB). MIUI adds
  friction — see Phase 0; on other skins the steps are simpler.
- A computer on the same LAN (to push files and work over SSH).
- A USB cable.

## Phase 0 — Phone preparation

1. Factory reset (recommended for a dedicated boat server).
2. Go through the setup wizard **staying offline** (airplane mode ON): this
   skips the forced account screens. If the wizard crashes, wipe data from
   recovery and retry offline.
3. Install [F-Droid](https://f-droid.org) from the browser, then from
   F-Droid install **Termux**, **Termux:API**, **Termux:Boot** and
   **Termux:Widget**. All from the *same source*: mixing Play Store and
   F-Droid versions breaks the Termux add-on apps (different signatures).
   - On MIUI, disable **Pure Mode** in the Security app first, otherwise
     side-loaded installs are blocked.
4. Battery/autostart (MIUI): for Termux, Termux:API, Termux:Boot and
   Termux:Widget set **Autostart = ON** and **Battery saver = No
   restrictions** — in *both* places (app info AND the Security app's own
   autostart manager). Disable "Memory extension" (virtual RAM) in
   developer options.

## Phase 1 — Termux base

Open Termux and run:

```bash
pkg update && pkg upgrade
pkg install openssh nodejs-lts mosquitto termux-api curl
termux-setup-storage   # a permission dialog appears: tap ALLOW (Termux must be in the foreground on the phone)
```

## Phase 2 — SSH access (strongly recommended)

```bash
passwd     # set a simple password (LAN-only, fine)
sshd       # starts the SSH server on port 8022
whoami     # your Termux user, e.g. u0_a226
ifconfig wlan0   # the phone's IP
```

From the computer: `ssh -p 8022 <user>@<phone-ip>`.
Later, put your SSH public key into `~/.ssh/authorized_keys`
(`chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys`) to work without
passwords. Never type long keys on the phone keyboard — copy/paste from the
computer.

## Phase 3 — The software stack

### Signal K server

```bash
npm install -g signalk-server
mkdir -p ~/.signalk
nano ~/.signalk/settings.json
```

Minimal `~/.signalk/settings.json`:

```json
{
  "configVersion": 1,
  "appPath": "",
  "settingsPath": "~/.signalk",
  "interfaces": { "nmea0183": true, "signalk-node-server": true, "canbus": false, "anchormanager": true },
  "wsCompression": false,
  "ssl": false,
  "port": 3000
}
```

Start it and check:

```bash
nohup signalk-server > ~/signalk.log 2>&1 &
```

Open `http://<phone-ip>:3000` in the phone browser and **create the admin
account** (asked only on first visit). Endpoints: REST
`:3000/signalk/v1/api`, WebSocket `:3000/signalk/v1/stream`, TCP delta
`:8375` (this is what ESP32 gateways connect to).

### MQTT broker

```bash
mosquitto -d -p 1883
mosquitto_pub -p 1883 -t test -m hi
mosquitto_sub -p 1883 -t test -C 1 -W 5    # should print: hi
```

### zigbee2mqtt (requires a LAN coordinator — USB does NOT work on Termux)

```bash
npm install -g zigbee2mqtt
mkdir -p ~/z2m
nano ~/z2m/configuration.yaml
```

```yaml
homeassistant: false
permit_join: false
mqtt:
  base_topic: zigbee2mqtt
  server: mqtt://localhost:1883
serial:
  port: tcp://<coordinator-ip>:6638   # e.g. SMLIGHT SLZB-06/07
  adapter: zstack
frontend:
  port: 8080
advanced:
  log_level: info
```

```bash
ZIGBEE2MQTT_DATA=~/z2m nohup zigbee2mqtt > ~/z2m.log 2>&1 &
```

Until a real coordinator exists it logs `EHOSTUNREACH` — that is expected
and proves the install works.

## Phase 4 — Keep it running

Create `~/.termux/boot/01-start.sh` (Termux:Boot executes executables in
`~/.termux/boot/` — NOT `~/boot/`):

```bash
#!/data/data/com.termux/files/usr/bin/bash
# Boot autostart: wake lock + sshd + mosquitto + Signal K (guarded, absolute paths)
export PATH=/data/data/com.termux/files/usr/bin:$PATH
cd "$HOME"
/data/data/com.termux/files/usr/bin/termux-wake-lock
pgrep -x sshd >/dev/null || /data/data/com.termux/files/usr/bin/sshd
pgrep -x mosquitto >/dev/null || /data/data/com.termux/files/usr/bin/mosquitto -d -p 1883
pgrep -f "node.*signalk-server" >/dev/null || nohup /data/data/com.termux/files/usr/bin/signalk-server > "$HOME/signalk.log" 2>&1 &
```

The boot context has a minimal environment: use **absolute paths** (or
export PATH) — bare `signalk-server` fails with "No such file or
directory" at boot even though it works in an interactive Termux shell.
The `node.*signalk-server` pgrep pattern also avoids matching the script's
own wrapper process.

```bash
chmod +x ~/.termux/boot/01-start.sh
```

**Watchdog (the real fix for MIUI):** Termux:Boot only runs after the
screen is unlocked at least once — useless for a silent reboot. Register
an Android JobScheduler job instead, which survives reboots and needs no
user interaction:

```bash
termux-job-scheduler -s ~/.termux/boot/01-start.sh --job-id 1 --period-ms 900000 --persisted true
termux-job-scheduler -p    # verify the job is pending
```

Every 15 minutes (Android's minimum) the idempotent script re-checks all
services. Combined with the Termux:Boot script (fast start when the
screen is unlocked) and the Termux:Widget icon (manual start), the system
self-heals after any reboot, silent or not.

Note: `pgrep -x sshd` / `pgrep -x mosquitto` are unreliable on Android
(processes show with full paths); use `pgrep -f "/usr/bin/ss[h]d"` style
patterns (bracket avoids matching the caller itself), and a pidfile for
nohup'd Node servers.

**Reality check:** our first two autostart "failures" were actually this
wrong path (`~/boot/` instead of `~/.termux/boot/`). With the correct
path, if MIUI still blocks autostart, the fallback that always works:
install **Termux:Widget**, put an executable script in `~/bin` (e.g.
`~/bin/AVVIA-SERVIZI` that calls the boot script), then add the
Termux:Widget widget to the home screen. One tap starts everything.

Keep the phone powered with a charge limit around 80% (battery health).

## Phase 5 — Verify after every reboot

From the computer:

```bash
ssh -p 8022 <user>@<phone-ip> 'pgrep -f "signalk-serv[e]r" | wc -l; pgrep -x sshd | wc -l'
curl -s http://<phone-ip>:3000/signalk/ | head -c 200
```

## Known pitfalls (learned the hard way)

- Termux has **no `/tmp`** — write to `$HOME`.
- `pkill -f <name>` over SSH kills your own SSH session (the pattern
  matches the wrapper shell). Use bracket patterns (`signalk-serv[e]r`) and
  split kill/start into separate commands.
- `termux-setup-storage` (and every Android permission dialog) only works
  with Termux in the foreground **on the phone**.
- Signal K reads exactly `~/.signalk/settings.json` — not any other name.
- npm warns about skipped install scripts (native serial/CAN modules):
  harmless, we never use USB on Android.
