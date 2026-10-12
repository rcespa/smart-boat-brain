# AGENT.md — istruzioni per agenti AI

[English](AGENT.md) | Italiano

Sei un agente AI che aiuta a installare lo stack smart-boat-brain (Termux +
Signal K + Mosquitto + zigbee2mqtt su un telefono Android). Questo file
dice cosa puoi fare in autonomia, cosa devi delegare all'umano e le
trappole che a noi sono costate ore. La guida per l'umano è `SETUP.it.md`.

## Modello operativo

- Lavori da una workstation via **SSH verso il Termux del telefono**
  (`ssh -p 8022 <utente>@<ip-telefono>`). Tutto ciò che è nelle Fasi 1, 3,
  4 è scriptabile — fallo tu, e verifica ogni passo con output reale.
- Tutto ciò che mostra **un'interfaccia Android** (popup di permessi,
  installazione app, creazione dell'account admin Signal K, impostazioni
  MIUI, procedura di configurazione iniziale) va delegato: dai all'umano
  UNA azione breve alla volta, con i nomi esatti delle schermate, poi
  attendi e verifica il risultato dal tuo lato.
- Non far mai digitare all'umano stringhe lunghe (chiavi SSH, URL) sulla
  tastiera del telefono. Pusha tu i file (scp/curl dalla workstation),
  dai one-liner copiaincollabili.

## Regole dure (ognuna ci è costata un fallimento)

1. **Su Termux non c'è `/tmp`.** Usa `$HOME` per i file temporanei.
2. **`pkill -f NOME` via SSH ammazza la tua sessione SSH** (la cmdline del
   wrapper ssh contiene NOME). Usa pattern con parentesi
   (`signalk-serv[e]r`) e separa kill e riavvio in chiamate SSH distinte.
3. **`termux-setup-storage` e ogni popup di permesso richiedono Termux in
   primo piano sul telefono.** Eseguito via SSH non fa nulla, in silenzio.
   Chiedi all'umano di lanciarlo nel Termux sul dispositivo e toccare
   CONSENTI.
4. **Il file di config di Signal K è esattamente
   `~/.signalk/settings.json`.** Altri nomi vengono ignorati ("Settings
   file does not exist"). Se un server Node ignora la tua config, grep nel
   suo `dist/` la stringa di log di avvio per trovare il nome vero.
5. **I warning di npm sugli install-scripts sono attesi**
   (`@serialport/bindings-cpp`, canboatjs): i moduli USB/CAN nativi non si
   compilano su Android e non ci servono — il coordinatore ZigBee è sempre
   TCP. Non provare a "sistemarli".
6. **MIUI ammazza background/autostart in modo aggressivo.** Termux:Boot ha
   fallito l'avvio automatico due volte anche con tutti i permessi
   documentati. Offri sempre il piano B: Termux:Widget + script in `~/bin/`
   (avviatore idempotente). Non spendere più di un un paio di tentativi
   sull'autostart MIUI.
7. **Verifica, non assumere.** Dopo ogni installazione: esegui il binario,
   chiama l'endpoint (`curl http://<ip>:3000/signalk/`), fai un pub/sub
   MQTT di prova. Un `npm install` riuscito non è un servizio funzionante.
8. zigbee2mqtt senza coordinatore logga `EHOSTUNREACH <ip>:6638` ed esce —
   è il segnale di SUCCESSO per la parte software. Non inseguirlo.

## Checklist di delega (umano, in ordine)

| # | Chiedi all'umano di... | Tu verifichi con... |
|---|---|---|
| 1 | Reset di fabbrica + wizard offline (modalità aereo) | — |
| 2 | MIUI: Modalità pura OFF; avvio automatico ON + nessuna restrizione batteria per Termux/API/Boot/Widget, in ENTRAMBI i posti (info app e app Sicurezza) | test di riavvio alla fine |
| 3 | Installare F-Droid, poi Termux, Termux:API, Termux:Boot (+Widget) da F-Droid | `termux-info` mostra `APK_RELEASE=F_DROID` |
| 4 | Eseguire `termux-setup-storage` nel Termux SUL TELEFONO, toccare CONSENTI | `ls ~/storage/shared` funziona |
| 5 | Eseguire `passwd` + `sshd` nel Termux e darti utente/IP | `ssh -p 8022 ...` si connette |
| 6 | Creare l'admin Signal K su `http://<ip>:3000` (prima visita) | loginStatus / API rispondono |
| 7 | (opzionale) Installare l'APK Termux:Widget che hai pushato in Download/, aggiungere il widget | l'umano tocca AVVIA-SERVIZI, tu vedi i processi |

## Checklist autonoma (tua)

```bash
pkg update && pkg upgrade
pkg install -y openssh nodejs-lts mosquitto termux-api curl
npm install -g signalk-server          # ~1 min, warning normali
# scrivere ~/.signalk/settings.json (vedi SETUP.it.md)
nohup signalk-server > ~/signalk.log 2>&1 &   # poi curl sugli endpoint
mosquitto -d -p 1883 && round-trip pub/sub
npm install -g zigbee2mqtt             # config in ~/z2m, ZIGBEE2MQTT_DATA=~/z2m
# scrivere ~/boot/01-start.sh (wake-lock, sshd, avvio con guardia) + ~/bin/AVVIA-SERVIZI
```

## Etichetta di sessione

- L'IP del telefono è DHCP: ricontrollalo (`ip neigh`/ARP) prima di darlo
  per buono.
- Tieni un file di note/skill di progetto con: utente+IP SSH, cosa è
  verificato, cosa è in sospeso. Le sessioni future non devono riscoprire
  le trappole qui sopra.
- Quando pushi binari/APK: scarica sulla workstation, checksum, scp in
  `~/storage/shared/Download/`, verifica il checksum sul telefono, all'umano
  comunica solo il nome del file.
