# Guida di installazione (per esseri umani)

[English](SETUP.md) | Italiano

Istruzioni passo-passo per riprodurre questo server di bordo su un
telefono Android. Scritte per un essere umano col telefono in mano: alcuni
passi richiedono il touch screen e non sono eseguibili da remoto.

## Cosa serve

- Un telefono Android qualunque (il nostro: Xiaomi Mi 9T, Android 11, 6 GB).
  MIUI aggiunge attrito — vedi Fase 0; su altre interfacce è più semplice.
- Un computer sulla stessa rete LAN (per inviare file e lavorare via SSH).
- Un cavo USB.

## Fase 0 — Preparazione del telefono

1. Reset di fabbrica (consigliato per un telefono dedicato a server di bordo).
2. Completa la configurazione iniziale **restando offline** (modalità aereo
   ON): così si saltano le schermate di account forzate. Se la procedura
   crasha, fai un wipe data dalla recovery e riprova offline.
3. Installa [F-Droid](https://f-droid.org) dal browser, poi da F-Droid
   installa **Termux**, **Termux:API**, **Termux:Boot** e
   **Termux:Widget**. Tutte dalla *stessa fonte*: mescolare Play Store e
   F-Droid rompe le app aggiuntive di Termux (firme diverse).
   - Su MIUI disattiva prima la **Modalità pura** nell'app Sicurezza,
     altrimenti le installazioni laterali vengono bloccate.
4. Batteria/avvio automatico (MIUI): per Termux, Termux:API, Termux:Boot e
   Termux:Widget imposta **Avvio automatico = ON** e **Risparmio batteria =
   Nessuna restrizione** — in *entrambi* i posti (info app E il gestore di
   avvio automatico dell'app Sicurezza). Disattiva l'"Espansione memoria"
   (RAM virtuale) nelle opzioni sviluppatore.

## Fase 1 — Base Termux

Apri Termux ed esegui:

```bash
pkg update && pkg upgrade
pkg install openssh nodejs-lts mosquitto termux-api curl
termux-setup-storage   # compare un popup di permesso: tocca CONSENTI (Termux deve essere in primo piano SUL TELEFONO)
```

## Fase 2 — Accesso SSH (fortemente consigliato)

```bash
passwd     # imposta una password semplice (solo LAN, basta)
sshd       # avvia il server SSH sulla porta 8022
whoami     # il tuo utente Termux, es. u0_a226
ifconfig wlan0   # l'IP del telefono
```

Dal computer: `ssh -p 8022 <utente>@<ip-telefono>`.
Più tardi, metti la tua chiave pubblica SSH in `~/.ssh/authorized_keys`
(`chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys`) per lavorare senza
password. Non digitare mai chiavi lunghe con la tastiera del telefono:
copia/incolla dal computer.

## Fase 3 — Lo stack software

### Server Signal K

```bash
npm install -g signalk-server
mkdir -p ~/.signalk
nano ~/.signalk/settings.json
```

`~/.signalk/settings.json` minimale:

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

Avvialo e verifica:

```bash
nohup signalk-server > ~/signalk.log 2>&1 &
```

Apri `http://<ip-telefono>:3000` nel browser del telefono e **crea
l'account admin** (chiesto solo alla prima visita). Endpoint: REST
`:3000/signalk/v1/api`, WebSocket `:3000/signalk/v1/stream`, delta TCP
`:8375` (è la porta a cui si collegano i gateway ESP32).

### Broker MQTT

```bash
mosquitto -d -p 1883
mosquitto_pub -p 1883 -t test -m hi
mosquitto_sub -p 1883 -t test -C 1 -W 5    # deve stampare: hi
```

### zigbee2mqtt (richiede un coordinatore in LAN — su Termux l'USB NON funziona)

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
  port: tcp://<ip-coordinatore>:6638   # es. SMLIGHT SLZB-06/07
  adapter: zstack
frontend:
  port: 8080
advanced:
  log_level: info
```

```bash
ZIGBEE2MQTT_DATA=~/z2m nohup zigbee2mqtt > ~/z2m.log 2>&1 &
```

Finché non c'è un coordinatore reale logga `EHOSTUNREACH` — è atteso e
dimostra che l'installazione funziona.

## Fase 4 — Tenere in vita il tutto

Crea `~/.termux/boot/01-start.sh` (Termux:Boot esegue gli eseguibili in
`~/.termux/boot/` — NON `~/boot/`):

```bash
#!/data/data/com.termux/files/usr/bin/bash
termux-wake-lock
pgrep -x sshd >/dev/null || sshd
pgrep -x mosquitto >/dev/null || mosquitto -d -p 1883
pgrep -f "signalk-serv[e]r" >/dev/null || nohup signalk-server > "$HOME/signalk.log" 2>&1 &
```

```bash
chmod +x ~/.termux/boot/01-start.sh
```

**Verità scomoda:** i nostri primi due "fallimenti" dell'avvio automatico
erano in realtà questo percorso sbagliato (`~/boot/` invece di
`~/.termux/boot/`). Con il percorso corretto, se MIUI bloccasse ancora
l'avvio, il piano B che funziona sempre: installa **Termux:Widget**, metti
uno script eseguibile in `~/bin` (es. `~/bin/AVVIA-SERVIZI` che richiama
lo script di boot) e aggiungi il widget Termux:Widget alla schermata home.
Un tocco e parte tutto.

Tieni il telefono alimentato con limite di carica intorno all'80% (salute
della batteria).

## Fase 5 — Verifica dopo ogni riavvio

Dal computer:

```bash
ssh -p 8022 <utente>@<ip-telefono> 'pgrep -f "signalk-serv[e]r" | wc -l; pgrep -x sshd | wc -l'
curl -s http://<ip-telefono>:3000/signalk/ | head -c 200
```

## Trappole note (imparate a nostre spese)

- Termux **non ha `/tmp`** — scrivi in `$HOME`.
- `pkill -f <nome>` via SSH ammazza la sessione SSH stessa (il pattern
  matcha la shell wrapper). Usa il pattern con parentesi (`signalk-serv[e]r`)
  e separa kill e avvio in comandi distinti.
- `termux-setup-storage` (e ogni popup di permesso Android) funziona solo
  con Termux in primo piano **sul telefono**.
- Signal K legge esattamente `~/.signalk/settings.json` — non altri nomi.
- npm avvisa di script di installazione saltati (moduli serial/CAN nativi):
  innocuo, su Android non useremo mai l'USB.
