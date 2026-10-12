# AGENT.md — instructions for AI agents


You are an AI agent helping set up the smart-boat-brain stack (Termux +
Signal K + Mosquitto + zigbee2mqtt on an Android phone). This file tells
you what you can do autonomously, what you must delegate to the human, and
the pitfalls that cost us hours. The human-facing guide is `SETUP.md`.

## Operating model

- You work from a workstation over **SSH to the phone's Termux**
  (`ssh -p 8022 <user>@<phone-ip>`). Everything in Phases 1, 3, 4 is
  scriptable — do it yourself, verify each step with real output.
- Anything that shows an **Android UI** (permission dialogs, app installs,
  the Signal K admin account creation, MIUI settings, the setup wizard)
  must be delegated: give the human ONE short action at a time, with exact
  screen names, then wait and verify the result from your side.
- Never make the human type long strings (SSH keys, URLs) on the phone
  keyboard. Push files yourself (scp/curl from the workstation), give
  copy-pasteable one-liners.

## Hard rules (each one cost us a failure)

1. **No `/tmp` on Termux.** Use `$HOME` for temp files.
2. **`pkill -f NAME` over SSH kills your own session** (the ssh wrapper
   cmdline contains NAME). Use bracket patterns (`signalk-serv[e]r`) and
   split kill and (re)start into separate SSH calls.
3. **`termux-setup-storage` and any permission dialog require Termux in
   the foreground on the phone.** Running it over SSH silently does nothing.
   Ask the human to run it in Termux on the device and tap ALLOW.
4. **Signal K config file is exactly `~/.signalk/settings.json`.** Other
   names are ignored ("Settings file does not exist"). If a Node server
   ignores your config, grep its `dist/` for the startup log string to
   find the real filename.
5. **npm install-scripts warnings are expected** (`@serialport/bindings-cpp`,
   canboatjs): native USB/CAN modules can't build on Android and we don't
   need them — the ZigBee coordinator is always TCP. Do not try to "fix"
   them.
6. **MIUI kills background/autostart aggressively.** Termux:Boot reads
   scripts from `~/.termux/boot/` (NOT `~/boot/`) — a wrong path looks
   exactly like an MIUI block. Verify the path before blaming MIUI. If it
   still fails with the right path, offer the fallback: Termux:Widget +
   `~/bin/` script (idempotent starter). Do not spend more than a couple
   of attempts on MIUI autostart.
7. **Verify, don't assume.** After each install: run the binary, hit the
   endpoint (`curl http://<ip>:3000/signalk/`), pub/sub a test MQTT
   message. A successful `npm install` is not a working service.
8. zigbee2mqtt without a coordinator logs `EHOSTUNREACH <ip>:6638` and
   exits — that is the SUCCESS signal for the software part. Don't chase it.

## Delegation checklist (human, in order)

| # | Ask the human to... | You verify by... |
|---|---|---|
| 1 | Factory reset + offline setup wizard (airplane mode) | — |
| 2 | MIUI: Pure Mode OFF; autostart ON + no battery restrictions for Termux/API/Boot/Widget, in BOTH app-info and Security app | reboot test at the end |
| 3 | Install F-Droid, then Termux, Termux:API, Termux:Boot (+Widget) from F-Droid | `termux-info` shows `APK_RELEASE=F_DROID` |
| 4 | Run `termux-setup-storage` in Termux ON THE PHONE, tap ALLOW | `ls ~/storage/shared` works |
| 5 | Run `passwd` + `sshd` in Termux, tell you user/IP | `ssh -p 8022 ...` connects |
| 6 | Create Signal K admin at `http://<ip>:3000` (first visit) | loginStatus / API responds |
| 7 | (optional) Install Termux:Widget APK you pushed to Download/, add widget | human taps AVVIA-SERVIZI, you see processes |

## Your autonomous checklist

```bash
pkg update && pkg upgrade
pkg install -y openssh nodejs-lts mosquitto termux-api curl
npm install -g signalk-server          # ~1 min, warnings OK
# write ~/.signalk/settings.json (see SETUP.md)
nohup signalk-server > ~/signalk.log 2>&1 &   # then curl the endpoints
mosquitto -d -p 1883 && mosquitto_pub/sub round-trip
npm install -g zigbee2mqtt             # config in ~/z2m, ZIGBEE2MQTT_DATA=~/z2m
# write ~/.termux/boot/01-start.sh (wake-lock, guarded sshd/mosquitto/signalk) + ~/bin/AVVIA-SERVIZI
```

## Session etiquette

- The phone IP is DHCP: re-check (`ip neigh`/ARP) before assuming it.
- Keep a project skill/notes file with: SSH user+IP, what's verified,
  what's pending. Future sessions must not rediscover the pitfalls above.
- When pushing binaries/APKs: download on the workstation, checksum, scp to
  `~/storage/shared/Download/`, verify checksum on the phone, tell the
  human the filename only.
