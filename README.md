# owrt-cloudflared

Repository di pacchetti OpenWrt per [`cloudflared`](https://github.com/cloudflare/cloudflared), il client di Cloudflare Tunnel.

Il progetto compila automaticamente le release stabili di `cloudflared` e le pubblica come pacchetti `.apk` per OpenWrt 25.12 tramite un repository APK firmato su GitHub Pages.

> **Compatibilità:** OpenWrt 25.12.x  
> **Target:** `rockchip/armv8`  
> **Architettura pacchetti:** `aarch64_generic`

## Perché esiste

Il pacchetto `cloudflared` disponibile nei repository OpenWrt può non essere aggiornato all’ultima release upstream. Questa repository fornisce build più recenti senza richiedere la sostituzione manuale di binari sul router.

Il flusso è:

```text
Nuova release cloudflared
        ↓
GitHub Actions
        ↓
Compilazione con OpenWrt SDK
        ↓
Creazione pacchetto .apk
        ↓
Firma dell’indice packages.adb
        ↓
Pubblicazione su GitHub Pages
        ↓
Aggiornamento manuale dal router
```

Il repository può rilevare e pubblicare nuove release automaticamente, ma **il router non si aggiorna da solo**: l’aggiornamento va avviato manualmente, perché comporta il riavvio del servizio `cloudflared`.

## Repository pubblico

Dopo una build riuscita, l’indice del repository è disponibile qui:

```text
https://alanb2.github.io/owrt-cloudflared/releases/25.12.6/aarch64_generic/packages.adb
```

La chiave pubblica del repository è:

```text
https://alanb2.github.io/owrt-cloudflared/owrt-cloudflared.pem
```

## Installazione su OpenWrt

### 1. Installa la chiave pubblica

Sul router:

```sh
mkdir -p /etc/apk/keys /etc/apk/repositories.d

curl -fL \
  https://alanb2.github.io/owrt-cloudflared/owrt-cloudflared.pem \
  -o /etc/apk/keys/owrt-cloudflared.pem
```

### 2. Aggiungi il repository

```sh
printf '%s\n' \
  "https://alanb2.github.io/owrt-cloudflared/releases/25.12.6/aarch64_generic/packages.adb" \
  > /etc/apk/repositories.d/owrt-cloudflared.list
```

### 3. Aggiorna l’indice

```sh
apk update
```

Se la configurazione è corretta, `apk update` non deve restituire errori di firma.

### 4. Installa o aggiorna cloudflared

Per installarlo:

```sh
apk add cloudflared
```

Per aggiornarlo:

```sh
apk upgrade cloudflared
```

Poi verifica la versione:

```sh
cloudflared --version
```

## Configurazione del tunnel

Il pacchetto utilizza i file di configurazione standard di OpenWrt per `cloudflared`. Durante un aggiornamento, eventuali file di configurazione già presenti non vengono sovrascritti: le nuove versioni fornite dal pacchetto vengono salvate con estensione `.apk-new`.

Puoi confrontarle con:

```sh
diff -u /etc/cloudflared/config.yml /etc/cloudflared/config.yml.apk-new
diff -u /etc/config/cloudflared /etc/config/cloudflared.apk-new
```

Non sostituire automaticamente le configurazioni esistenti con i file `.apk-new`.

## Gestione del servizio

```sh
service cloudflared status
service cloudflared start
service cloudflared restart
service cloudflared stop
```

Per consultare i log:

```sh
logread -e cloudflared | tail -n 100
```

Dopo un aggiornamento, riavvia il servizio:

```sh
service cloudflared restart
```

## Come funziona la build

Il workflow GitHub Actions:

1. Controlla l’ultima release stabile di `cloudflared`.
2. Confronta la versione con l’ultima GitHub Release già pubblicata.
3. Se è cambiata, scarica l’SDK OpenWrt 25.12.6 per `rockchip/armv8`.
4. Prepara i feed OpenWrt.
5. Importa e aggiorna l’infrastruttura Go alla versione richiesta.
6. Aggiorna la ricetta `cloudflared` alla nuova versione.
7. Compila il pacchetto per `aarch64_generic`.
8. Crea e firma l’indice `packages.adb`.
9. Pubblica il pacchetto su GitHub Pages e crea una GitHub Release.

## Versione Go

`cloudflared` richiede una versione di Go recente. Se una futura release upstream richiederà una versione Go successiva a quella configurata nel workflow, la build fallirà con un messaggio esplicito e dovrà essere aggiornata la relativa ricetta.

## Firma del repository

Il repository usa una coppia di chiavi EC P-256:

| File | Ruolo |
|---|---|
| Chiave privata | Conservata esclusivamente nei GitHub Secrets |
| Chiave pubblica | Pubblicata nella repository e su GitHub Pages |

La chiave privata firma l’indice `packages.adb`; la chiave pubblica, installata sul router in `/etc/apk/keys/`, consente ad `apk` di verificarne l’autenticità.

**Non committare mai la chiave privata.**

## Struttura della repository

```text
.
├── .github/
│   └── workflows/
│       └── build.yml
├── packages/
│   └── cloudflared/
│       └── Makefile
├── owrt-cloudflared.pem
└── README.md
```

## Note di sicurezza

- Non inserire mai la chiave privata nella repository.
- Verifica sempre la versione con `cloudflared --version` dopo un aggiornamento.
- Non usare `--allow-untrusted` per l’installazione quotidiana dal repository firmato.
- Se il repository non viene aggiornato per lungo tempo, verifica che le esecuzioni pianificate di GitHub Actions siano ancora abilitate.

## Licenza

La ricetta di pacchetto è derivata dal feed OpenWrt `packages` e mantiene la licenza indicata nel file originale. `cloudflared` è distribuito secondo la licenza del progetto upstream Cloudflare.