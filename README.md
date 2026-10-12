# smart-boat-brain

[Italiano](README.it.md) | English

A boat monitoring system built from hardware you probably already own: a
retired Android phone and a microcontroller, instead of a dedicated marine
computer.

My sailboat is from 1985. Her electronics are minimal, and almost every
open-source boat-monitoring project assumes a Raspberry Pi. This project
reuses what I had lying around: a **Xiaomi Mi 9T** (6 GB RAM, built-in
battery, GPS, sensors, cameras, optional cellular modem) as the boat's
server brain, plus a **Sailor Hat with ESP32** (SH-ESP32) as the sensor
gateway.

## The idea

One Android phone running [Termux](https://termux.com/) hosts the whole
software stack:

- **[Signal K](https://signalk.org/)** server — the open data bus for boats
- **[zigbee2mqtt](https://www.zigbee2mqtt.io/)** + **Mosquitto** — ZigBee
  sensor integration (via an *external* coordinator on the LAN, not USB)
- **[AvNav](https://www.wellenvogel.net/software/avnav/)** (native Android
  app) — chartplotter and instrument display, receiving Signal K alarms

The phone gives things a Raspberry Pi cannot:

- **built-in UPS** — its own battery keeps the system alive when the boat's
  service battery is disconnected
- **built-in GPS + compass + cameras** — position, anchor watch, visual
  inspection of the boat from anywhere
- **optional data SIM** — remote monitoring without marina WiFi
- **a screen and a touch interface** you already have

The **SH-ESP32** handles what Android is bad at: reading the
**Calypso Ultrasonic** wireless wind sensor over BLE and publishing the
data to Signal K (via SensESP), reliably and independently of the phone.

## What is monitored

| Subsystem | Sensor | Path |
|---|---|---|
| Wind (speed, angle, heading, roll/pitch) | Calypso Ultrasonic (BLE, masthead) | SH-ESP32 → Signal K TCP |
| Fridges x3 + engine room temperature | ZigBee temperature sensors | coordinator → zigbee2mqtt → MQTT → Signal K |
| Bilges / watertight compartments x4 | ThirdReality ZigBee water leak sensors (with local 120 dB buzzer) | same path |
| Position / anchor watch | phone GPS | Signal K |
| Visual check | phone camera | remote app |

Alarms flow from Signal K to AvNav notifications and to push messages
(Telegram) so a flooded bilge or a warm fridge reaches me ashore.

## Status

Work in progress. Verified so far:

- Signal K server 2.33 runs on Termux (Node.js 24 LTS), REST/WebSocket/TCP
  endpoints confirmed
- `npm install -g signalk-server` works on Termux; skipped native modules
  are irrelevant for a TCP-connected coordinator
- Boot autostart (Termux:Boot + wake-lock) scripted and tested

Next: MQTT broker + zigbee2mqtt feasibility, ZigBee coordinator, SH-ESP32
firmware (SensESP + NimBLE Calypso driver), alarm rules.

## Why

Marine electronics is expensive and closed. DIY marine projects are great
but assume a Raspberry Pi, which needs a separate case, power regulation,
UPS, GPS module... A modern phone is a strictly better single-board
computer for this job — it just needs the right software stack and a
microcontroller sidekick. This repo documents the whole build, pitfalls
included (MIUI is not server-friendly: expect to fight battery
optimizers).

## License

To be defined (likely MIT for software, CC-BY-SA for hardware notes).
