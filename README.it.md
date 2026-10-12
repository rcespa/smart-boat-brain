# smart-boat-brain

English | [Italiano](README.it.md)

Un sistema di monitoraggio della barca costruito con hardware che
probabilmente hai già in un cassetto: uno smartphone Android dismesso e un
microcontrollore, invece di un computer marino dedicato.

La mia barca a vela è un **Comet 1050**, costruito da **Comar SpA** nel
**1985** (10,3 m f.t.). La sua elettronica è minimale e quasi interamente
basata su **NMEA 0183**, e quasi tutti i progetti open source di
monitoraggio nautico danno per scontato un Raspberry Pi. Questo progetto
riutilizza quello che avevo: un **Xiaomi Mi 9T** (6 GB di RAM, batteria
integrata, GPS, sensori, fotocamere, modem cellulare opzionale) come
"cervello" di bordo, più una scheda **ESP32** come gateway dei sensori.

> **Nota:** le scelte hardware qui sotto (Mi 9T, Sailor Hat with ESP32,
> sensore vento Calypso Ultrasonic, sensori di allagamento ThirdReality)
> sono *il caso d'uso specifico di questo progetto* — ciò che avevo o mi
> serviva. L'architettura è generica: qualunque telefono Android con
> Termux può fare da server, e qualunque scheda basata su ESP32 può fare
> da gateway sensori. Lo SH-ESP32 è semplicemente un ESP32 in versione
> marina (ingresso 12-24V, I/O protetti, connettore antenna esterna); una
> comune devkit ESP32 con un convertitore DC-DC funziona esattamente
> allo stesso modo. Allo stesso modo il sensore vento è quello che hai:
> il Calypso Ultrasonic è semplicemente quello che possiedo io, che ha il
> vantaggio di un protocollo BLE documentato con driver open source già
> esistenti.

## L'elettronica esistente della barca

| Dispositivo | Interfaccia | Note |
|---|---|---|
| VHF con DSC | NMEA 0183 | posizione / chiamate DSC sul bus |
| Trasponditore AIS (attivo) | NMEA 0183 | traffico navale |
| Multiplexer **NMEA4WIFI** ([vela-navega.com](https://www.vela-navega.com/index.php/nmea4wifi)) | NMEA 0183 ⇄ WiFi (TCP/UDP) | basato su ESP32: 4 ingressi 0183 optoisolati (+SeaTalk1), 1 uscita; è così che il bus 0183 legacy arriva in rete |
| Eco | NASA Marine | NMEA 0183 |
| Batteria servizi | **LiFePO4** 330 Ah | il pacco energetico da monitorare |
| BMS | **Bluetooth** | BMS del pacco LiFePO4 |
| Regolatore solare | **SmartSolar MPPT 100/30** (Victron Energy) | **Bluetooth** (protocollo Victron Smart) |
| Caricabatterie DC-DC | **Orion XS 12/12-50** (Victron Energy) | **Bluetooth** (protocollo Victron Smart) |

Tutto ciò che parla **Bluetooth** (Victron SmartSolar, Victron Orion XS,
il BMS LiFePO4) verrà portato dentro Signal K per costruire dashboard
dedicate sull'energia: produzione solare, carica, consumi, stato batterie.

## L'idea

Un telefono Android con [Termux](https://termux.com/) ospita tutta la
parte software:

- server **[Signal K](https://signalk.org/)** — il bus dati aperto per le barche
- **[zigbee2mqtt](https://www.zigbee2mqtt.io/)** + **Mosquitto** — integrazione
  sensori ZigBee (via coordinatore *esterno* in rete LAN, non su USB)
- **[AvNav](https://www.wellenvogel.net/software/avnav/)** (app Android
  nativa) — plotter e display strumenti, riceve le notifiche Signal K come allarmi

Il mondo NMEA 0183 (VHF, AIS, eco) entra in Signal K attraverso il
multiplexer **NMEA4WIFI** già presente: trasmette già il bus su TCP/UDP, e
Signal K ha un input nativo NMEA 0183 su TCP/UDP — per quella parte non
serve nuovo hardware.

Il telefono offre cose che un Raspberry Pi non ha:

- **UPS integrato** — la sua batteria tiene in vita il sistema anche quando
  scolleghi la batteria servizi della barca
- **GPS + bussola + fotocamere integrati** — posizione, watch dell'ancora,
  controllo visivo della barca da remoto
- **SIM dati opzionale** — monitoraggio fuori dalla rete del porto
- **uno schermo touch** che possiedi già

Il **gateway ESP32** fa ciò in cui Android è scarso: leggere il sensore
vento via BLE — nel mio caso un **Calypso Ultrasonic** — e pubblicare i
dati su Signal K (con SensESP), in modo affidabile e indipendente dal
telefono.

## Cosa viene monitorato

| Sottosistema | Sensore | Percorso |
|---|---|---|
| Vento (velocità, angolo, rotta, rollio/beccuccio) | Calypso Ultrasonic (BLE, in testa d'albero) | gateway ESP32 → Signal K TCP |
| Profondità | eco NASA Marine | NMEA 0183 → NMEA4WIFI → Signal K |
| AIS / DSC / VHF | strumenti esistenti | NMEA 0183 → NMEA4WIFI → Signal K |
| Energia: solare, carica, stato batterie | SmartSolar MPPT 100/30, Orion XS 12/12-50, BMS LiFePO4 | Bluetooth → Signal K (dashboard dedicate) |
| Frigoriferi x3 + temperatura locale motore | sensori di temperatura ZigBee | coordinatore → zigbee2mqtt → MQTT → Signal K |
| Sentine / locali stagni x4 | sensori di allagamento ZigBee ThirdReality (con buzzer locale da 120 dB) | stesso percorso |
| Posizione / watch àncora | GPS del telefono | Signal K |
| Controllo visivo | fotocamera del telefono | app remota |

Gli allarmi partono da Signal K verso le notifiche di AvNav e i messaggi
push (Telegram): un'allagamento, un frigo troppo caldo o una batteria
piatta mi raggiungono anche da riva.

## Stato

Lavoro in corso. Verificato fin qui:

- Signal K server 2.33 gira su Termux (Node.js 24 LTS); endpoint
  REST/WebSocket/TCP confermati
- `npm install -g signalk-server` funziona su Termux; i moduli nativi
  saltati sono irrilevanti con coordinatore TCP
- Mosquitto 2.1.2 e zigbee2mqtt 2.14.2 installati e verificati sullo
  stesso telefono (z2m parte e parla TCP con un coordinatore — l'
  EHOSTUNREACH verso un IP segnaposto è atteso)
- autostart allo boot (Termux:Boot + wake-lock) scriptato; l'autostart MIUI
  si è rivelato inaffidabile — documentato il fallback a un tocco con
  Termux:Widget

Prossimi passi: coordinatore ZigBee (hardware), firmware ESP32 (SensESP +
driver NimBLE per Calypso), input NMEA4WIFI → Signal K, ponte Bluetooth
per i dispositivi Victron e il BMS, plugin di mappatura MQTT→Signal K,
regole di allarme.

## Perché

L'elettronica marina è cara e chiusa. I progetti DIY sono ottimi ma danno
per scontato un Raspberry Pi, che richiede case separato, regolazione di
potenza, UPS, modulo GPS... Uno smartphone moderno è a tutti gli effetti
un single-board computer migliore, per questo lavoro — servono solo lo
stack software giusto e un microcontrollore come sidekick. Questo repo
documenta tutto il lavoro, trappole comprese (MIUI non è amichevole per i
server: mettetevi l'animo in pace contro gli ottimizzatori di batteria).

## Documentazione

- [SETUP.it.md](SETUP.it.md) — guida di installazione passo-passo per
  esseri umani ([English](SETUP.md))
- [AGENT.md](AGENT.md) — istruzioni per un agente AI che esegue il setup:
  cosa automatizzare, cosa delegare all'umano, trappole note
  (solo in inglese: la maggior parte dei modelli risparmia token in inglese)

## Licenza

Da definire (probabilmente MIT per il software, CC-BY-SA per le note
hardware).
