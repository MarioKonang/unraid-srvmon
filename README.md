# srvmon – Server Monitor für Unraid

srvmon überwacht den Zustand eines Unraid-Servers und meldet relevante Ereignisse über Unraids eigenes Benachrichtigungssystem, auf Wunsch per E-Mail. Zusätzlich bietet es eine Live-Statusseite in der WebGUI.

Version: 2026.10.18i
Zielplattform: Unraid 6.12 und neuer (ausgelegt für 7.2.x, siehe [Ungeprüfte Annahmen](#ungeprüfte-annahmen))

---

## Inhalt

1. [Funktionen](#funktionen)
2. [Benachrichtigungslogik](#benachrichtigungslogik)
3. [Statusseite](#statusseite)
4. [Installation](#installation)
5. [Konfiguration](#konfiguration)
6. [Kommandozeile](#kommandozeile)
7. [Dateien und Pfade](#dateien-und-pfade)
8. [Datenquellen](#datenquellen)
9. [Sicherheit](#sicherheit)
10. [Bekannte Grenzen](#bekannte-grenzen)
11. [Ungeprüfte Annahmen](#ungeprüfte-annahmen)
12. [Fehlersuche](#fehlersuche)
13. [Änderungen](#änderungen)

---

## Funktionen

Jede Prüfung lässt sich in den Einstellungen einzeln abschalten. Die Level entsprechen den Unraid-Stufen `normal` (Notice), `warning` und `alert`.

### Disks

| Prüfung | Bedingung | Level |
|---|---|---|
| Disk-Status | Status ungleich `DISK_OK` | alert |
| E/A-Fehler | Unraids Fehlerzähler `numErrors` > 0 | alert |
| Temperatur HDD/SSD | ab `TEMP_HDD_WARN` / ab `TEMP_HDD_CRIT` | warning / alert |
| Temperatur NVMe | ab `TEMP_NVME_WARN` / ab `TEMP_NVME_CRIT` | warning / alert |

Leere Slots werden ignoriert: Status `DISK_NP` oder Slots ohne Gerät und ohne ID. Unraid führt einen unbelegten Parity2-Slot zum Beispiel mit `DISK_NP_DSBL`. Eine zugewiesene, aber fehlende Disk behält ihre ID in `disks.ini` und wird weiterhin gemeldet.

Der Flash-Stick wird bei Temperatur und SMART übersprungen. Disks im Spindown melden keine Temperatur und werden in diesem Lauf nicht bewertet.

### SMART

SMART wird mit `smartctl -n standby` gelesen. **Schlafende Disks werden dadurch nicht geweckt.** Ist eine Disk im Standby, bleibt der zuletzt bekannte Zustand bestehen.

| Prüfung | Bedingung | Level |
|---|---|---|
| Gesamtstatus | SMART-Status `FAILED` | alert |
| Reallocated_Sector_Ct (5) | Wert gestiegen | alert |
| Reported_Uncorrect (187) | Wert gestiegen | alert |
| Current_Pending_Sector (197) | Wert gestiegen | alert |
| Offline_Uncorrectable (198) | Wert gestiegen | alert |
| UDMA_CRC_Error_Count (199) | Wert gestiegen (meist Kabel oder Backplane) | warning |
| NVMe Media Errors | Wert gestiegen | alert |
| NVMe Critical Warning | ungleich 0 | alert |
| NVMe Verschleiß | `percentage_used` ≥ 90 % | warning |

Gemeldet wird der **Anstieg** eines Zählers gegenüber dem gespeicherten Stand. Beim ersten Lesen gibt es keinen gespeicherten Stand; ein Wert größer 0 wird dann einmalig gemeldet.

### Speicherplatz

Für jeden Pfad in `SPACE_PATHS` wird die Belegung geprüft: ab `SPACE_WARN` warning, ab `SPACE_CRIT` alert. Nicht gemountete Pfade, zum Beispiel bei gestopptem Array, werden übersprungen.

Standardmäßig enthalten sind:

- `/mnt/user`: Array und Pools gesamt
- `/mnt/cache`: Cache-Pool
- `/var/log`: tmpfs, ein volles Log ist ein häufiges Unraid-Problem
- `/var/lib/docker`: Docker-Image beziehungsweise -Verzeichnis

### Docker

| Prüfung | Bedingung | Level |
|---|---|---|
| Healthcheck | Container meldet `unhealthy` (gilt für alle Container) | warning |
| Überwachte Container | Container aus `DOCKER_WATCH` läuft nicht | alert |
| Container fehlt | Name aus `DOCKER_WATCH` existiert nicht | warning |
| Modus `*` | Ein Container wechselt von „läuft“ zu „läuft nicht“ (einmaliges Ereignis) | warning |

### VMs

| Prüfung | Bedingung | Level |
|---|---|---|
| Absturz | VM-Zustand `crashed` (gilt für alle VMs) | alert |
| Überwachte VMs | VM aus `VM_WATCH` läuft nicht | alert |
| Modus `*` | Eine VM wechselt von „läuft“ zu „läuft nicht“ (einmaliges Ereignis) | warning |

### System

| Prüfung | Bedingung | Level |
|---|---|---|
| Neustart | Neue `boot_id` seit dem letzten Lauf (einmaliges Ereignis) | warning |
| CPU-Last | 5-Minuten-Load, bezogen auf die Anzahl logischer CPUs, ab `CPU_LOAD_WARN` % | warning |
| RAM | Belegung (Basis `MemAvailable`) ab `RAM_WARN` % | warning |

### USV (apcupsd)

| Prüfung | Bedingung | Level |
|---|---|---|
| Batteriebetrieb | Status enthält `ONBATT` | alert |
| Verbindung verloren | Status enthält `COMMLOST` | warning |
| Batterie tauschen | Status enthält `REPLACEBATT` | warning |

Ist apcupsd nicht aktiv, entfällt die Prüfung.

### Docker-Updates

| Prüfung | Bedingung | Level |
|---|---|---|
| Update verfügbar | Unraids Update-Prüfung meldet für ein Image ein neueres Remote-Image (einmalig je neuem Update, mit Containernamen) | normal |
| Container aktualisiert | Ein bestehender Container läuft mit einer neuen Image-ID bei gleicher Image-Referenz (einmaliges Ereignis, mit Version alt und neu, sofern vorhanden) | normal |
| Image gewechselt | Ein bestehender Container nutzt eine andere Image-Referenz, zum Beispiel einen anderen Tag (einmaliges Ereignis) | normal |

**Verfügbare Updates:** srvmon fragt selbst keine Registry ab, sondern liest das Ergebnis von Unraids eigener Update-Prüfung. Wie aktuell diese Information ist, hängt deshalb von deren Zeitplan ab. Den Zeitplan für die Docker-Update-Prüfung legst du unter *Settings > Notifications* fest.

**Durchgeführte Updates:** srvmon merkt sich pro Container die Image-ID. Unraid erstellt einen Container bei einem Update mit dem neuen Image neu; dadurch ändert sich die Image-ID. Das erkennt srvmon unabhängig davon, wie das Update ausgelöst wurde, also auch bei automatischen Updates durch andere Plugins. Die Version stammt aus den Image-Labels, in dieser Reihenfolge: `org.opencontainers.image.version`, `version`, `build_version`. Fehlen die Labels, enthält die Meldung keine Version.

Nicht als Update gemeldet werden:

- ein Neuerstellen mit unverändertem Image, zum Beispiel nach dem Ändern der Template-Einstellungen,
- umbenannte oder neu angelegte Container (kein Vorher-Stand vorhanden),
- Updates, die stattfinden, während srvmon nicht installiert ist.

**Wichtig:** Alle Update-Meldungen haben das Level `normal`. Damit sie per Mail kommen, muss unter *Settings > Notifications* der E-Mail-Agent für **Notices** aktiv sein.

Updates für Unraid OS und Plugins prüft srvmon nicht. Dafür bietet Unraid unter *Settings > Notifications* eigene Optionen.

### TLS-Zertifikate (Nginx Proxy Manager)

srvmon prüft Zertifikate auf zwei Wegen. Beide sind unabhängig voneinander nutzbar.

**Zertifikatsdateien:** In jedem Verzeichnis aus `CERT_DIRS` sucht srvmon nach `<verzeichnis>/*/fullchain.pem`, also der Struktur von Let's Encrypt unter `letsencrypt/live`. Bei Nginx Proxy Manager heißen die Unterverzeichnisse `npm-1`, `npm-2` und so weiter. In Meldungen erscheint statt dieser Kennung der Domainname aus dem Zertifikat.

**Ausgelieferte Zertifikate:** Für jeden Eintrag aus `CERT_HOSTS` baut srvmon eine TLS-Verbindung mit SNI auf und wertet das Zertifikat aus, das nginx tatsächlich ausliefert.

| Prüfung | Bedingung | Level |
|---|---|---|
| Ablauf (Datei und Live) | noch höchstens `CERT_WARN_DAYS` Tage gültig | warning |
| Ablauf (Datei und Live) | noch höchstens `CERT_CRIT_DAYS` Tage gültig oder abgelaufen | alert |
| Vertrauen (Live) | Kette oder Hostname ungültig, wenn `CERT_VERIFY` aktiv | warning |
| Veraltet ausgeliefert (Live) | Im Dateisystem liegt für den Host ein Zertifikat, das mehr als einen Tag länger gültig ist als das ausgelieferte | warning |
| Nicht erreichbar (Live) | Keine TLS-Verbindung möglich | warning |
| Verzeichnis fehlt | Pfad aus `CERT_DIRS` existiert nicht | warning |
| Datei nicht lesbar | `fullchain.pem` lässt sich nicht auswerten | warning |

**Warum beide Wege:** Let's Encrypt erneuert Zertifikate normalerweise 30 Tage vor Ablauf. Mit der Standardschwelle von 20 Tagen bedeutet eine Warnung daher, dass die Erneuerung seit mindestens 10 Tagen fehlschlägt. Der Vergleich zwischen Datei und Live-Zertifikat erkennt zusätzlich den Fall, dass die Erneuerung zwar geklappt hat, nginx das neue Zertifikat aber nicht geladen hat.

**Prüfrhythmus:** Zertifikate prüft srvmon höchstens einmal pro Stunde, unabhängig von `INTERVAL`. Mit `--force` lässt sich die Prüfung sofort auslösen. Bestehende Probleme bleiben zwischen den Prüfungen erhalten, inklusive Erinnerungen.

**Ersetzte Zertifikate:** Legt Nginx Proxy Manager ein Zertifikat neu an, entsteht ein neues Verzeichnis `npm-<id>`; das alte kann liegen bleiben. srvmon ignoriert deshalb ein Zertifikat, wenn ein anderes alle seine Domains abdeckt und später abläuft. Auf der Statusseite erscheint es grau mit dem Hinweis „ersetzt durch npm-…“. Zertifikate ohne Nachfolger, etwa für nicht mehr genutzte Domains, werden weiterhin gemeldet; diese in NPM löschen oder in `CERT_IGNORE` eintragen.

**Hosts angeben:** mehrere Einträge kommagetrennt, ohne Port gilt 443.

| Eintrag | Bedeutung |
|---|---|
| `cloud.example.com` | Verbindung zu cloud.example.com auf Port 443 |
| `cloud.example.com:8443` | wie oben, Port 8443 |
| `cloud.example.com@192.168.1.10` | Verbindung zu 192.168.1.10:443 mit SNI cloud.example.com |
| `cloud.example.com@192.168.1.10:18443` | Verbindung zu 192.168.1.10:18443 mit SNI cloud.example.com |

Die Form mit `@adresse` ist sinnvoll, wenn der Server seine öffentliche Adresse nicht selbst erreicht, etwa weil der Router kein NAT-Loopback (Hairpin-NAT) unterstützt. Die Adresse ist dann typischerweise die lokale IP des Servers mit dem Port, auf dem Nginx Proxy Manager HTTPS annimmt.

**Pfad ermitteln:** Der Standardpfad `/mnt/user/appdata/NginxProxyManager/letsencrypt/live` passt nicht zu jedem NPM-Template. Den tatsächlichen Pfad zeigt das Volume-Mapping des Containers im Docker-Tab: Gesucht ist der Host-Pfad, der auf `/etc/letsencrypt` im Container zeigt; dort das Unterverzeichnis `live` angeben. Prüfen lässt sich der Pfad so:

```bash
ls /mnt/user/appdata/NginxProxyManager/letsencrypt/live/
```

Eigene, in NPM hochgeladene Zertifikate (Custom Certificates) liegen in einem anderen Verzeichnis. Dieses Verzeichnis kannst du zusätzlich in `CERT_DIRS` eintragen, sofern es dieselbe Struktur `<verzeichnis>/*/fullchain.pem` hat.

---

## Benachrichtigungslogik

Es gibt zwei Arten von Meldungen.

**Zustände** sind anhaltende Probleme, zum Beispiel eine hohe Temperatur oder ein voller Speicher. Sie werden gemeldet:

- beim ersten Auftreten,
- wenn das Level steigt (warning → alert),
- als Erinnerung „Weiterhin: …“ nach `REMIND_HOURS` Stunden, solange das Problem besteht (0 = keine Erinnerung),
- als „Behoben: …“ (Level `normal`), sobald der Zustand wieder in Ordnung ist, wenn `NOTIFY_RESOLVED` aktiv ist.

**Ereignisse** sind einmalige Vorkommnisse, zum Beispiel ein Neustart, ein gestiegener SMART-Zähler oder ein neues Update. Sie werden genau einmal gemeldet.

**Weitere Regeln:**

- Kann eine Prüfung keine Daten liefern, etwa weil der Docker-Dienst aus ist, bleiben bestehende Zustände unverändert. Es gibt dann weder eine „Behoben“-Meldung noch eine Erinnerung.
- Pro Lauf werden höchstens 10 Meldungen verschickt. Sind es mehr, folgt eine Sammelmeldung „Weitere Meldungen unterdrückt“.
- Jede Meldung wird zusätzlich mit dem Tag `srvmon` ins Syslog geschrieben.
- Ob eine Meldung als E-Mail verschickt wird, entscheidet Unraid anhand der Agenten-Einstellungen pro Level (siehe [Installation](#installation)).

Der Betreff hat die Form `srvmon: <Kurzbeschreibung>`, der Absender heißt in Unraid „Server Monitor“.

---

## Weitere Benachrichtigungswege

Zusätzlich zu Unraids Benachrichtigungen kann srvmon Meldungen per Webhook weitergeben und sich bei einer externen Ausfallüberwachung melden. Beides ist optional und standardmäßig aus.

### Webhook (zum Beispiel Home Assistant und Alexa)

Ist `WEBHOOK_URL` gesetzt, schickt srvmon jede Meldung ab dem Level `WEBHOOK_LEVEL` zusätzlich per HTTP-POST als JSON:

```json
{
  "level": "warning",
  "subject": "Zertifikat cloud.example.com läuft in 9 Tagen ab",
  "description": "Zertifikat für cloud.example.com (npm-2, ...) gültig bis ...",
  "speech": "Warnung vom Server Tower: Zertifikat cloud.example.com läuft in 9 Tagen ab",
  "host": "Tower",
  "time": 1792330214
}
```

Das Feld `speech` ist für Sprachausgaben gedacht: kurz, mit Level und Servername, ohne die lange Beschreibung.

Regeln:

- Pro Lauf gehen höchstens 3 Webhook-Aufrufe raus. Gibt es mehr Meldungen, folgt ein Sammelaufruf „… weitere Meldungen“. So spricht Alexa bei einer Störung mit vielen Folgemeldungen nicht minutenlang.
- Ist das Ziel nicht erreichbar, wird das im Syslog vermerkt (`grep srvmon /var/log/syslog`). Die Unraid-Benachrichtigung ist davon nicht betroffen.
- Die Testmeldung (`check.php --test`) geht immer auch an den Webhook, unabhängig vom eingestellten Level.

#### Home Assistant einrichten

1. **Alexa-Anbindung:** Home Assistant braucht eine Integration, die Alexa-Ansagen auslösen kann. Verbreitet ist die inoffizielle Integration „Alexa Media Player“ (über HACS). Sie meldet sich mit deinem Amazon-Konto an und kann durch Änderungen bei Amazon jederzeit ausfallen.
2. **Webhook-ID erzeugen.** Die ID wirkt wie ein Passwort: Wer sie kennt, kann im lokalen Netz Ansagen auslösen. Erzeugen im Unraid-Terminal:
   ```bash
   echo "srvmon-$(openssl rand -hex 16)"
   ```
3. **Automation anlegen** (YAML-Modus), ID und Gerätenamen anpassen:
   ```yaml
   alias: Unraid srvmon an Alexa
   mode: queued
   triggers:
     - trigger: webhook
       webhook_id: srvmon-HIER_DIE_ERZEUGTE_ID
       allowed_methods: [POST]
       local_only: true
   conditions:
     # Alerts immer, Warnungen nur tagsüber
     - condition: template
       value_template: >
         {{ trigger.json.level == 'alert' or (7 <= now().hour < 22) }}
   actions:
     - action: notify.alexa_media_echo_wohnzimmer
       data:
         message: "{{ trigger.json.speech }}"
         data:
           type: announce
   ```
   Den genauen Namen der Aktion für dein Echo-Gerät findest du in Home Assistant unter *Entwicklerwerkzeuge > Aktionen* (Suche nach `alexa_media`).

   **Mehrere Geräte:** Statt der gerätespezifischen Aktion die allgemeine Aktion `notify.alexa_media` mit einer Liste von Zielen verwenden. Die Entitätsnamen der Echo-Geräte (`media_player.…`) stehen unter *Einstellungen > Geräte & Dienste > Alexa Media Player*.

   ```yaml
   actions:
     - action: notify.alexa_media
       data:
         message: "{{ trigger.json.speech }}"
         target:
           - media_player.echo_wohnzimmer
           - media_player.echo_buero
         data:
           type: announce
   ```

   **Alle Geräte:** Die Ziele per Template aus allen Mediaplayern der Integration ermitteln. Je nach Alexa-Konfiguration sind darin auch Lautsprechergruppen enthalten, was zu doppelten Ansagen führen kann; dann besser die Liste von Hand pflegen.

   ```yaml
         target: "{{ integration_entities('alexa_media') | select('match', 'media_player\\.') | list }}"
   ```

   **Je nach Level unterschiedlich**, zum Beispiel Alerts überall, Warnungen nur im Büro:

   ```yaml
   actions:
     - action: notify.alexa_media
       data:
         message: "{{ trigger.json.speech }}"
         target: >
           {% if trigger.json.level == 'alert' %}
             {{ integration_entities('alexa_media') | select('match', 'media_player\\.') | list }}
           {% else %}
             {{ ['media_player.echo_buero'] }}
           {% endif %}
         data:
           type: announce
   ``` Die Syntax `triggers`/`trigger:` und `actions`/`action:` gilt für aktuelle Home-Assistant-Versionen; ältere Versionen erwarten `trigger:`/`platform:` und `action:`/`service:`.
4. **In srvmon eintragen:** `WEBHOOK_URL` = `http://<IP von Home Assistant>:8123/api/webhook/<ID>`, zum Beispiel `http://192.168.1.20:8123/api/webhook/srvmon-...`. Läuft Home Assistant im Netzwerk `br0` mit eigener IP, diese IP verwenden.
5. **Testen:** `php /usr/local/emhttp/plugins/srvmon/scripts/check.php --test`

**Grenze:** Läuft Home Assistant als Container auf demselben Unraid-Server, fällt dieser Weg zusammen mit Docker oder dem Server aus. Dafür ist der Heartbeat gedacht. Den Home-Assistant-Container selbst kannst du über `DOCKER_WATCH` überwachen; diese Meldung kommt dann per Mail oder Push, nicht über Alexa.

### Heartbeat (Ausfall des gesamten Servers)

Eine Überwachung auf dem Server kann dessen Totalausfall nicht selbst melden. Deshalb ruft srvmon bei jedem Prüflauf die `HEARTBEAT_URL` auf. Ein externer Dienst schlägt Alarm, wenn diese Aufrufe ausbleiben.

Geeignet sind zum Beispiel:

- **healthchecks.io:** Check anlegen, als Periode das srvmon-Intervall (Standard 5 Minuten) und eine Karenzzeit von etwa 10 Minuten wählen, die Ping-URL als `HEARTBEAT_URL` eintragen. Benachrichtigen kann der Dienst per Mail, Telegram und anderen Wegen.
- **Uptime Kuma auf einem anderen Gerät:** Monitor vom Typ „Push“ anlegen und dessen URL eintragen.

Wichtig ist, dass der externe Dienst nicht auf dem überwachten Server läuft und nicht über Home Assistant auf diesem Server benachrichtigt.

---

## Statusseite

Die Seite findest du unter *Settings > User Utilities > Server Monitor Status*. Sie zeigt:

- eine Zusammenfassung (Anzahl kritischer Meldungen und Warnungen oder „Alles in Ordnung“) mit dem Zeitpunkt der Live-Daten und der letzten Prüfung,
- die aktiven Probleme mit dem Zeitpunkt ihres Auftretens,
- System: CPU-Last, RAM und Uptime,
- USV: Status, Ladung, Restlaufzeit und Last,
- Speicherplatz als Balken pro Pfad,
- Disks als Tabelle mit Status, Temperatur, Fehlern und SMART-Werten,
- Docker und VMs mit ihrem Zustand, überwachte Einträge sind markiert; bei Containern zusätzlich ein Hinweis „Update“, die Image-Version und der Zeitpunkt des letzten erkannten Updates,
- verfügbare Docker-Updates,
- TLS-Zertifikate mit Quelle (Datei oder Live), Ablaufdatum, Resttagen, Aussteller und Hinweisen. Diese Daten stammen aus der letzten stündlichen Prüfung, der Zeitpunkt steht unter der Tabelle.

Die Seite lädt ihre Daten alle 30 Sekunden neu, solange der Browser-Tab sichtbar ist. Die Farben richten sich nach den konfigurierten Schwellwerten.

SMART-Werte werden für die Anzeige nicht live gelesen. Die Seite zeigt den Stand der letzten Prüfung. Abgeschaltete Prüfungen erscheinen auf der Seite nicht.

Auf der Einstellungsseite *Server Monitor* gibt es zusätzlich eine kurze Liste der aktiven Probleme.

---

## Installation

### Voraussetzungen

- Unraid 6.12 oder neuer
- Eingerichteter Mailversand in Unraid, falls Benachrichtigungen per E-Mail kommen sollen

### 1. E-Mail-Versand in Unraid einrichten

srvmon speichert keine Zugangsdaten. Der Versand läuft vollständig über Unraid.

1. Unter *Settings > Notifications* die SMTP-Einstellungen eintragen: Server, Port, Absender und Empfänger.
2. Unter *Settings > Notifications* beim E-Mail-Agenten die gewünschten Level aktivieren:
   - **Warnings** und **Alerts**: empfohlen
   - **Notices**: nötig, wenn Docker-Update-Meldungen (verfügbar und durchgeführt), „Behoben“-Meldungen und Testmeldungen per Mail kommen sollen
3. Unter *Settings > Notifications* den Zeitplan für die Docker-Update-Prüfung von Unraid aktivieren. srvmon nutzt deren Ergebnis.
4. Mit der Testfunktion von Unraid prüfen, ob der Versand grundsätzlich funktioniert.

### 2. Plugin installieren

**Empfohlen: über eine URL (mit Update-Funktion)**

1. `srvmon.plg` in ein Git-Repository legen, zum Beispiel auf GitHub.
2. In der `.plg` die Entity `pluginURL` auf die Raw-URL dieser Datei setzen:
   ```xml
   <!ENTITY pluginURL "https://raw.githubusercontent.com/<user>/<repo>/main/srvmon.plg">
   ```
3. In Unraid unter *Plugins > Install Plugin* dieselbe URL eintragen und *Install* klicken.

Danach erkennt Unraid neue Versionen über *Plugins > Check for updates*.

**Alternative: lokale Datei (ohne Update-Funktion)**

1. `srvmon.plg` auf den Flash kopieren, zum Beispiel nach `/boot/srvmon.plg`.
2. Im Terminal ausführen:
   ```bash
   plugin install /boot/srvmon.plg
   ```

### 3. Einrichtung prüfen

```bash
# Testbenachrichtigung senden
php /usr/local/emhttp/plugins/srvmon/scripts/check.php --test

# Probelauf: zeigt, was gemeldet würde, ohne etwas zu senden
php /usr/local/emhttp/plugins/srvmon/scripts/check.php --force --dry-run
```

Anschließend die Einstellungen unter *Settings > User Utilities > Server Monitor* anpassen.

### Aktualisieren

- **Bei Installation über URL:** In der `.plg` die Entity `version` erhöhen, die Datei ins Repository hochladen und in Unraid unter *Plugins* aktualisieren. Ohne höhere Versionsnummer erkennt Unraid kein Update.
- **Bei lokaler Installation:** Das alte Plugin entfernen und die neue Datei installieren:
  ```bash
  plugin remove srvmon.plg
  plugin install /boot/srvmon.plg
  ```

Konfiguration und Zustand bleiben in beiden Fällen erhalten.

### Deinstallieren

Über *Plugins* oder im Terminal:

```bash
plugin remove srvmon.plg
```

Dabei werden der Cron-Eintrag, die Programmdateien und die temporären Dateien entfernt. Konfiguration und Zustand in `/boot/config/plugins/srvmon/` bleiben erhalten. Für eine vollständige Entfernung musst du dieses Verzeichnis anschließend manuell löschen.

---

## Konfiguration

Die Einstellungen liegen unter *Settings > User Utilities > Server Monitor*. Die Seite ist in die Abschnitte Allgemein, Disks und SMART, Speicherplatz, Docker, VMs, System und USV sowie TLS-Zertifikate gegliedert. Sie werden in `/boot/config/plugins/srvmon/srvmon.cfg` gespeichert. Nicht gesetzte Werte übernimmt srvmon aus `default.cfg`.

Änderungen in der Datei werden beim nächsten Lauf wirksam, ein Neustart ist nicht nötig.

### Allgemein

| Schlüssel | Standard | Beschreibung |
|---|---|---|
| `INTERVAL` | `5` | Prüfintervall in Minuten (1 bis 1440) |
| `REMIND_HOURS` | `24` | Erinnerung bei anhaltenden Problemen nach x Stunden, `0` = aus |
| `NOTIFY_RESOLVED` | `yes` | Meldung, sobald ein Problem behoben ist |

### Prüfungen ein- und ausschalten

| Schlüssel | Standard | Beschreibung |
|---|---|---|
| `CHECK_DISKS` | `yes` | Disk-Status, E/A-Fehler, Temperatur |
| `CHECK_SMART` | `yes` | SMART-Auswertung (nur zusammen mit `CHECK_DISKS`) |
| `CHECK_SPACE` | `yes` | Speicherplatz |
| `CHECK_DOCKER` | `yes` | Docker-Container |
| `CHECK_VMS` | `yes` | VMs |
| `CHECK_SYSTEM` | `yes` | CPU, RAM, Neustart |
| `CHECK_UPS` | `yes` | USV über apcupsd |
| `CHECK_UPDATES` | `yes` | Verfügbare Docker-Updates melden |
| `CHECK_DOCKER_UPDATED` | `yes` | Durchgeführte Container-Updates melden (nur zusammen mit `CHECK_DOCKER`) |

### TLS-Zertifikate

| Schlüssel | Standard | Beschreibung |
|---|---|---|
| `CHECK_CERTS` | `yes` | Zertifikatsprüfung ein/aus |
| `CERT_DIRS` | `/mnt/user/appdata/NginxProxyManager/letsencrypt/live` | Verzeichnisse mit `*/fullchain.pem`, kommagetrennt; leer = Dateiprüfung aus |
| `CERT_HOSTS` | leer | Live zu prüfende Hosts, Format `host`, `host:port`, `host@adresse` oder `host@adresse:port`; leer = Live-Prüfung aus |
| `CERT_WARN_DAYS` | `20` | Warnung, wenn höchstens so viele Tage Restlaufzeit |
| `CERT_CRIT_DAYS` | `7` | Alarm, wenn höchstens so viele Tage Restlaufzeit |
| `CERT_VERIFY` | `yes` | Vertrauenskette und Hostname bei der Live-Prüfung prüfen |
| `CERT_IGNORE` | leer | Domains, deren Zertifikatsdateien ignoriert werden, kommagetrennt |

### Schwellwerte

| Schlüssel | Standard | Beschreibung |
|---|---|---|
| `TEMP_HDD_WARN` | `45` | °C, Warnung für HDD/SATA-SSD |
| `TEMP_HDD_CRIT` | `50` | °C, Alarm für HDD/SATA-SSD |
| `TEMP_NVME_WARN` | `60` | °C, Warnung für NVMe |
| `TEMP_NVME_CRIT` | `70` | °C, Alarm für NVMe |
| `SPACE_WARN` | `85` | % Belegung, Warnung |
| `SPACE_CRIT` | `95` | % Belegung, Alarm |
| `CPU_LOAD_WARN` | `90` | % des 5-Minuten-Loads, bezogen auf die Anzahl logischer CPUs |
| `RAM_WARN` | `90` | % RAM-Belegung |

### Weitere Benachrichtigungswege

| Schlüssel | Standard | Beschreibung |
|---|---|---|
| `WEBHOOK_URL` | leer | Ziel für Webhook-Aufrufe (HTTP-POST mit JSON); leer = aus |
| `WEBHOOK_LEVEL` | `warning` | Mindest-Level für Webhooks: `normal` (alles), `warning` (Warnings und Alerts) oder `alert` |
| `HEARTBEAT_URL` | leer | Wird bei jedem Prüflauf per GET aufgerufen; leer = aus |

### Pfade und Watchlists

| Schlüssel | Standard | Beschreibung |
|---|---|---|
| `SPACE_PATHS` | `/mnt/user,/mnt/cache,/var/log,/var/lib/docker` | Zu prüfende Mountpoints, kommagetrennt |
| `DOCKER_WATCH` | `*` | Überwachte Container (siehe unten) |
| `VM_WATCH` | leer | Überwachte VMs (siehe unten) |

### Watchlists im Detail

`DOCKER_WATCH` und `VM_WATCH` kennen drei Modi:

| Wert | Verhalten |
|---|---|
| leer | Keine gezielte Überwachung. Es gelten nur die allgemeinen Prüfungen: `unhealthy` bei Containern, `crashed` bei VMs. |
| `name1,name2` | Solange ein genannter Eintrag nicht läuft, besteht ein Alarm, inklusive Erinnerungen. |
| `*` | Jeder Wechsel von „läuft“ zu „läuft nicht“ wird einmalig gemeldet, auch bei manuellem Stopp. |

Hinweise:

- Namen müssen exakt stimmen, Groß- und Kleinschreibung zählt. Die korrekten Namen zeigen diese Befehle:
  ```bash
  docker ps -a --format '{{.Names}}'
  virsh list --all --name
  ```
- `*` nicht mit Namen mischen. Ist `*` enthalten, werden die Namen ignoriert.
- Namen mit Komma sind nicht möglich, weil das Komma das Trennzeichen ist.
- Bei Docker meldet srvmon nicht existierende Namen. Bei VMs gibt es diese Prüfung nicht.

### Beispiel `srvmon.cfg`

```ini
INTERVAL="5"
SPACE_PATHS="/mnt/user,/mnt/cache,/var/log,/var/lib/docker"
DOCKER_WATCH="nextcloud,mariadb,NginxProxyManager"
VM_WATCH="HomeAssistant"
RAM_WARN="95"
REMIND_HOURS="12"
CERT_HOSTS="cloud.example.com@192.168.1.10,app.example.com@192.168.1.10"
```

---

## Kommandozeile

```bash
php /usr/local/emhttp/plugins/srvmon/scripts/check.php [Optionen]
```

| Option | Wirkung |
|---|---|
| ohne | Normaler Lauf, wird aber übersprungen, wenn das Intervall noch nicht abgelaufen ist |
| `--force` | Intervall ignorieren und sofort prüfen |
| `--dry-run` | Meldungen nur auf der Konsole ausgeben; nichts senden, keinen Zustand speichern |
| `--test` | Testbenachrichtigung senden |

Der Cron-Job ruft das Skript jede Minute auf, das Skript selbst prüft das Intervall. Ein Lock verhindert parallele Läufe.

---

## Dateien und Pfade

### Programmdateien (RAM, werden beim Booten aus der `.plg` neu erzeugt)

| Pfad | Inhalt |
|---|---|
| `/usr/local/emhttp/plugins/srvmon/README.md` | Kurzbeschreibung für die Pluginliste |
| `/usr/local/emhttp/plugins/srvmon/default.cfg` | Standardwerte |
| `/usr/local/emhttp/plugins/srvmon/include/lib.php` | Gemeinsame Funktionen und Datensammler |
| `/usr/local/emhttp/plugins/srvmon/include/data.php` | JSON-Endpunkt für die Statusseite |
| `/usr/local/emhttp/plugins/srvmon/scripts/check.php` | Prüfskript |
| `/usr/local/emhttp/plugins/srvmon/srvmon.page` | Einstellungsseite |
| `/usr/local/emhttp/plugins/srvmon/SrvmonStatus.page` | Statusseite |

### Persistente Dateien (Flash)

| Pfad | Inhalt |
|---|---|
| `/boot/config/plugins/srvmon.plg` | Plugin-Definition, wird von Unraid beim Booten installiert |
| `/boot/config/plugins/srvmon/srvmon.cfg` | Deine Einstellungen |
| `/boot/config/plugins/srvmon/srvmon.cron` | Cron-Eintrag |
| `/boot/config/plugins/srvmon/state.json` | Aktive Probleme, SMART-Stand, letzte Zustände von Containern und VMs, Image-ID und Version pro Container, bekannte Updates |

`state.json` wird nur geschrieben, wenn sich der Inhalt tatsächlich ändert. Das schont den Flash-Stick.

### Temporäre Dateien (RAM)

| Pfad | Inhalt |
|---|---|
| `/tmp/srvmon/lastrun` | Zeitpunkt des letzten Laufs |
| `/tmp/srvmon/lock` | Sperre gegen parallele Läufe |
| `/tmp/srvmon/status.json` | Status des letzten Laufs für die WebGUI |
| `/tmp/srvmon/lastcert` | Zeitpunkt der letzten Zertifikatsprüfung |
| `/tmp/srvmon/certs.json` | Ergebnis der letzten Zertifikatsprüfung für die Statusseite |

---

## Datenquellen

| Bereich | Quelle |
|---|---|
| Disks | `/var/local/emhttp/disks.ini` |
| SMART | `smartctl -n standby -j -H -A /dev/<gerät>` |
| Speicherplatz | `mountpoint -q` sowie die PHP-Funktionen `disk_total_space` und `disk_free_space` |
| Docker | `docker ps -a`, `docker inspect` (Image-ID, Image-Referenz, Labels) |
| VMs | `virsh list --all --name`, `virsh domstate` |
| CPU, RAM, Uptime | `/proc/loadavg`, `/proc/meminfo`, `/proc/uptime`, `nproc` |
| Neustart | `/proc/sys/kernel/random/boot_id` |
| USV | `apcaccess status` |
| Zertifikatsdateien | `openssl x509` auf `<CERT_DIRS>/*/fullchain.pem` |
| Ausgelieferte Zertifikate | `openssl s_client` mit SNI und `-verify_hostname` |
| Docker-Updates | `/var/lib/docker/unraid-update-status.json` |
| Versand | `/usr/local/emhttp/webGui/scripts/notify` |

Alle externen Aufrufe laufen mit `timeout`, damit ein hängender Dienst den Lauf nicht blockiert.

---

## Sicherheit

- **Keine Zugangsdaten im Plugin.** Der Mailversand nutzt die SMTP-Konfiguration von Unraid.
- **Schreibzugriff nur bei Bedarf.** Das Plugin schreibt außer in die eigenen Pfade nichts auf den Server; die externen Befehle werden nur lesend aufgerufen.
- **Statusseite hinter dem Login.** Der JSON-Endpunkt liegt im Plugin-Verzeichnis der WebGUI und ist damit durch den WebGUI-Login geschützt. Disk-IDs, die meist Seriennummern enthalten, gibt er nicht an den Browser aus.
- **Root-Rechte.** Plugins laufen mit Root-Rechten. Installiere nur aus Quellen, die du kontrollierst. Wer Schreibzugriff auf das Repository hinter `pluginURL` hat, kann über ein Update beliebigen Code auf dem Server ausführen.
- **Webhook-ID und Heartbeat-URL sind Geheimnisse.** Sie stehen in `srvmon.cfg` auf dem Flash, nicht in der `.plg`. Die Konfigurationsdatei deshalb nicht veröffentlichen. Bei Home Assistant `local_only: true` beibehalten, damit der Webhook nur aus dem lokalen Netz auslösbar ist.
- **Daten an Dritte.** Über Webhooks und Heartbeat verlassen Daten den Server: beim Webhook die Meldungstexte, beim Heartbeat nur der Aufruf selbst. Bei externen Diensten wie healthchecks.io bleibt es beim Heartbeat ohne Inhalt.
- **Keine Tokens in der URL.** Bei privaten Repositories gehört kein Zugriffstoken in `pluginURL`, denn es stünde im Klartext auf dem Flash und im Log.

---

## Bekannte Grenzen

- **`DOCKER_WATCH="*"`** meldet auch manuelle Stopps und Stopps durch Backup-Plugins. Für weniger Meldungen nur konkrete Namen eintragen.
- **ZFS:** Der ARC zählt als belegter RAM. Auf ZFS-Systemen `RAM_WARN` höher setzen.
- **SMART hinter manchen Controllern:** Disks, die bei `smartctl` einen `-d`-Typ benötigen, liefern keine SMART-Daten. Unraids eigene SMART-Einstellung pro Disk wird nicht ausgewertet.
- **Fehlende Disks:** Dass eine zugewiesene, aber fehlende Disk ihre ID in `disks.ini` behält, ist eine Annahme und wurde nicht auf einem echten System geprüft.
- **Disk-Status:** Alle Werte außer `DISK_OK` lösen bei belegten Slots einen Alarm aus, also auch Übergangszustände wie beim Hinzufügen einer neuen Disk.
- **`/mnt/user`:** Die Belegung ist ein Gesamtwert über Array und Pools. Für einzelne Disks die Pfade `/mnt/diskN` in `SPACE_PATHS` ergänzen.
- **CPU-Temperatur** wird nicht überwacht, weil die Sensorkonfiguration je nach Hardware stark variiert.
- **Unraid OS- und Plugin-Updates** prüft srvmon nicht; dafür gibt es die Optionen in Unraid.
- **Update-Hinweis auf der Statusseite:** Nach einem Update kann der Hinweis „Update“ stehen bleiben, bis Unraids eigene Update-Prüfung das nächste Mal läuft.
- **Zuordnung Image zu Container:** Image-Referenzen ohne Tag werden als `:latest` behandelt. Bei abweichenden Schreibweisen derselben Registry, etwa mit und ohne `docker.io/`, kann die Zuordnung fehlschlagen; die Meldung nennt dann nur das Image.
- **Verwaiste Zertifikate:** srvmon liest die NPM-Datenbank nicht und weiß daher nicht, welche Zertifikate NPM tatsächlich nutzt. Ersetzte Zertifikate werden erkannt (siehe oben); Zertifikate nicht mehr genutzter Domains ohne Nachfolger werden weiterhin gemeldet. Solche Domains in `CERT_IGNORE` eintragen oder die Zertifikate in NPM löschen.
- **Vertrauensprüfung:** Sie nutzt die CA-Zertifikate des Unraid-Systems. Fehlen diese, meldet die Prüfung auch gültige Let's-Encrypt-Zertifikate als nicht vertrauenswürdig. In dem Fall `CERT_VERIFY="no"` setzen.
- **Array-Disks:** Liegt appdata auf dem Array statt auf einem Pool, kann die stündliche Dateiprüfung eine schlafende Disk wecken.
- **Erkennungsverzögerung:** Die Auflösung entspricht dem Prüfintervall. Ein Problem, das zwischen zwei Läufen auftritt und wieder verschwindet, wird nicht erkannt.

---

## Ungeprüfte Annahmen

srvmon wurde mit simulierten Unraid-Dateien und Tools getestet, nicht auf einem echten Unraid 7.2.3. Diese Punkte solltest du auf deinem System prüfen:

| Annahme | Prüfung |
|---|---|
| `notify`, `update_cron` und PHP-CLI sind an den erwarteten Pfaden vorhanden | `ls -l /usr/local/emhttp/webGui/scripts/notify /usr/local/sbin/update_cron /usr/bin/php` |
| Schlüssel und Statuswerte in `disks.ini` | `grep -E '^\[\|^(status\|temp\|numErrors\|type\|device)=' /var/local/emhttp/disks.ini` |
| Format der Docker-Update-Datei (Felder `local` und `remote`) | `head -c 500 /var/lib/docker/unraid-update-status.json` |
| Pfad der NPM-Zertifikate | `ls <CERT_DIRS>` muss Unterverzeichnisse wie `npm-1` mit `fullchain.pem` zeigen |
| `openssl` mit `-ext` und `-verify_hostname` (OpenSSL 1.1.1 oder neuer) | `openssl version` |
| CA-Zertifikate für die Vertrauensprüfung vorhanden | `echo \| openssl s_client -connect letsencrypt.org:443 -servername letsencrypt.org 2>/dev/null \| grep "Verify return code"` muss `0 (ok)` zeigen |
| Docker wandelt `\t` im `--format`-Template in Tabulatoren um | `docker inspect --format '{{.Name}}\t{{.Image}}' $(docker ps -aq)` muss tabulatorgetrennte Zeilen liefern |
| Speichern der Einstellungen über `/update.php` mit CSRF-Token | Wert in der GUI ändern, dann `cat /boot/config/plugins/srvmon/srvmon.cfg` |
| Plugin-Attribute `icon="heartbeat"` und `launch="Settings/srvmon"` werden in der Pluginliste ausgewertet | Pluginliste ansehen: Icon und anklickbarer Name |
| Header `Markdown="false"` in der Statusseite wird unterstützt | Statusseite öffnen und auf Darstellungsfehler prüfen |
| Darstellung im responsiven Layout von 7.2.x | Beide Seiten auf Desktop und Smartphone öffnen |

Die Ausgabe von `disks.ini` enthält keine Seriennummern, wenn du den Befehl wie angegeben verwendest. Prüfe trotzdem jede Ausgabe, bevor du sie weitergibst.

---

## Fehlersuche

| Problem | Vorgehen |
|---|---|
| Keine Mails | `check.php --test` ausführen. Erscheint die Meldung in der Unraid-Glocke, aber nicht als Mail, sind SMTP oder der E-Mail-Agent unter *Settings > Notifications* das Problem. |
| Keine Alexa-Ansage | `check.php --test` ausführen; in Home Assistant unter *Einstellungen > Automationen* die Ablaufverfolgung (Trace) der Automation prüfen; im Unraid-Syslog nach „Webhook fehlgeschlagen“ suchen |
| Meldungen nachvollziehen | `grep srvmon /var/log/syslog` |
| Was würde gemeldet? | `check.php --force --dry-run` |
| Läuft der Cron-Job? | `date -d @$(cat /tmp/srvmon/lastrun)` muss einen Zeitpunkt innerhalb des Intervalls zeigen, `grep -r srvmon /etc/cron.d/` den Eintrag mit `check.php` |
| Cron-Eintrag fehlt | `/usr/local/sbin/update_cron` ausführen und erneut prüfen. Ab Version 2026.10.18e liest das Installationsskript den Eintrag 15 Sekunden nach der Installation automatisch nochmals ein. |
| Einstellungen werden nicht gespeichert | `cat /boot/config/plugins/srvmon/srvmon.cfg` prüfen, Browser-Konsole auf Fehler prüfen |
| Statusseite zeigt „Daten konnten nicht geladen werden“ | `php /usr/local/emhttp/plugins/srvmon/include/data.php` im Terminal ausführen und die Ausgabe auf PHP-Fehler prüfen |
| Zertifikatsprüfung sofort ausführen | `check.php --force --dry-run` (umgeht auch die stündliche Sperre) |
| Alte Meldungen hängen fest | Zustand zurücksetzen: `rm /boot/config/plugins/srvmon/state.json`. Danach werden bestehende Probleme erneut gemeldet und SMART-Zähler > 0 einmalig wiederholt. |

---

## Änderungen

### 2026.10.18i
- Webhook-Versand, zum Beispiel an Home Assistant für Alexa-Ansagen; Mindest-Level einstellbar, höchstens 3 Aufrufe pro Lauf plus Sammelmeldung
- Heartbeat-URL für eine externe Ausfallüberwachung (healthchecks.io, Uptime Kuma)

### 2026.10.18h
- Einstellungsseite in Abschnitte mit Trennlinien gegliedert, Felder thematisch sortiert

### 2026.10.18g
- Zertifikatsdateien, die durch ein neueres Zertifikat für dieselben Domains ersetzt wurden, werden ignoriert
- `CERT_HOSTS` akzeptiert den Port auch hinter der Adresse (`host@adresse:port`)
- Hinweis auf die Schreibweise mit LAN-Adresse, wenn die Live-Prüfung keine Verbindung bekommt

### 2026.10.18f
- Leere Disk-Slots ohne Gerät und ID werden ignoriert. Behebt den Fehlalarm bei einem leeren Parity2-Slot mit `DISK_NP_DSBL`.

### 2026.10.18e
- Cron-Eintrag wird nach Abschluss der Installation erneut eingelesen. Ursache des Fehlers: `update_cron` berücksichtigt nur bereits registrierte Plugins, srvmon ist während des eigenen Installationsskripts aber noch nicht registriert.
- Icon, Beschreibung und Link zur Einstellungsseite in der Pluginliste

### 2026.10.18d
- TLS-Zertifikate: Ablauf der Zertifikatsdateien (Nginx Proxy Manager) und der live ausgelieferten Zertifikate
- Erkennung, wenn nginx nach einer Erneuerung noch das alte Zertifikat ausliefert
- Prüfung von Vertrauenskette und Hostname, Statusseite mit Zertifikatsübersicht

### 2026.10.18c
- Meldung, wenn ein Container mit einem neuen Image gestartet wurde, inklusive Version aus den Image-Labels
- Meldung zu verfügbaren Updates nennt die betroffenen Container
- Statusseite: Update-Hinweis, Version und Zeitpunkt des letzten Updates pro Container
- Neue Option `CHECK_DOCKER_UPDATED`

### 2026.10.18b
- Neue Seite „Server Monitor Status“ mit Live-Ansicht, Aktualisierung alle 30 s
- Datensammlung in `include/lib.php` ausgelagert, JSON-Endpunkt `include/data.php`

### 2026.10.18a
- CSRF-Token im Einstellungsformular explizit gesetzt

### 2026.10.18
- Erste Version: Disks, SMART, Speicher, Docker, VMs, System, USV, Docker-Updates
