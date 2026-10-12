# smart-boat-brain

[Italiano](README.it.md) | English

A boat monitoring system built from hardware you probably already own: a
retired Android phone and a microcontroller, instead of a dedicated marine
computer.

My sailboat is from 1985. Her electronics are minimal, and almost every
open-source boat-monitoring project assumes a Raspberry Pi. This project
reuses what I had lying around: a **Xiaomi Mi 9T** (6 GB RAM, built-in
battery, GPS, sensors, cameras, optional cellular modem) as the boat's
server brain, plus an **ESP32** board as the sensor gateway.

> **Note:** the hardware choices below (Mi 9T, Sailor Hat with ESP32,
> Calypso Ultrasonic wind sensor, ThirdReality leak sensors) are *this
> project's specific use case* — what I happened to own or need. The
> architecture is generic: any Android phone with Termux can be the
> server, and any ESP32-based board can be the sensor gateway. The
> SH-ESP32 is just a convenient marine-grade ESP32 (12-24V power input,
> protected I/O, external antenna connector); a plain ESP32 devkit with
> a DC-DC converter works exactly the same. Likewise the wind sensor is
> whatever you have: the Calypso Ultrasonic is simply the one I own, and
> it happens to have a documented BLE protocol with existing open-source
> drivers.

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

The **ESP32 gateway** handles what Android is bad at: reading the wind
sensor over BLE — in my case a **Calypso Ultrasonic** wireless anemometer
— and publishing the data to Signal K (via SensESP), reliably and
independently of the phone.

## What is monitored

| Subsystem | Sensor | Path |
|---|---|---|
| Wind (speed, angle, heading, roll/pitch) | Calypso Ultrasonic (BLE, masthead) | ESP32 gateway → Signal K TCP |
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
- Mosquitto 2.1.2 and zigbee2mqtt 2.14.2 installed and verified on the
  same phone (z2m starts and speaks TCP to a coordinator — the
  EHOSTUNREACH against a placeholder IP is expected)
- Boot autostart (Termux:Boot + wake-lock) scripted; MIUI autostart proved
  unreliable — documented Termux:Widget one-tap fallback

Next: ZigBee coordinator (hardware), ESP32 firmware (SensESP + NimBLE
Calypso driver), MQTT→Signal K mapping plugin, alarm rules.

## Why

Marine electronics is expensive and closed. DIY marine projects are great
but assume a Raspberry Pi, which needs a separate case, power regulation,
UPS, GPS module... A modern phone is a strictly better single-board
computer for this job — it just needs the right software stack and a
microcontroller sidekick. This repo documents the whole build, pitfalls
included (MIUI is not server-friendly: expect to fight battery
optimizers).

## Documentation

- [SETUP.md](SETUP.md) — step-by-step installation guide for humans
  ([Italiano](SETUP.it.md))
- [AGENT.md](AGENT.md) — instructions for an AI agent doing the setup:
  what to automate, what to delegate to the human, known pitfalls
  ([Italiano](AGENT.it.md))

## License

To be defined (likely MIT for software, CC-BY-SA for hardware notes).
