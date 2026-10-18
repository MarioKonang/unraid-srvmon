# srvmon – Server Monitor for Unraid

srvmon monitors the health of an Unraid server and reports relevant events through Unraid's own notification system (email, push services) and optionally via webhook, for example to Home Assistant for Alexa announcements. It also provides a live status page in the WebGUI.

Version: 2026.10.18o
Repository: https://github.com/MarioKonang/unraid-srvmon
Target platform: Unraid 6.12 and newer (used on 7.2.4, see [Verification status](#verification-status))

---

## Contents

1. [Features](#features)
2. [Notification logic](#notification-logic)
3. [Additional notification channels](#additional-notification-channels)
4. [Status page](#status-page)
5. [Installation](#installation)
6. [Configuration](#configuration)
7. [Command line](#command-line)
8. [Files and paths](#files-and-paths)
9. [Data sources](#data-sources)
10. [Security](#security)
11. [Known limitations](#known-limitations)
12. [Verification status](#verification-status)
13. [Troubleshooting](#troubleshooting)
14. [Changelog](#changelog)

---

## Features

Every check can be switched off individually in the settings. Levels correspond to Unraid's levels `normal` (notice), `warning` and `alert`.

### Disks

| Check | Condition | Level |
|---|---|---|
| Disk status | status other than `DISK_OK` | alert |
| I/O errors | Unraid's error counter `numErrors` > 0 | alert |
| Temperature HDD/SATA SSD | from `TEMP_HDD_WARN` / from `TEMP_HDD_CRIT` | warning / alert |
| Temperature NVMe | from `TEMP_NVME_WARN` / from `TEMP_NVME_CRIT` | warning / alert |

Empty slots are ignored: status `DISK_NP`, or slots without device and without ID. Unraid reports an unassigned parity2 slot as `DISK_NP_DSBL`, for example. An assigned but missing disk keeps its ID in `disks.ini` and is still reported.

The flash drive is skipped for temperature and SMART. Spun-down disks report no temperature and are not evaluated in that run.

### SMART

SMART is read with `smartctl -n standby`. **Sleeping disks are not woken up.** If a disk is in standby, its last known state is kept.

| Check | Condition | Level |
|---|---|---|
| Overall status | SMART status `FAILED` | alert |
| Reallocated_Sector_Ct (5) | value increased | alert |
| Reported_Uncorrect (187) | value increased | alert |
| Current_Pending_Sector (197) | value increased | alert |
| Offline_Uncorrectable (198) | value increased | alert |
| UDMA_CRC_Error_Count (199) | value increased (usually cable or backplane) | warning |
| NVMe media errors | value increased | alert |
| NVMe critical warning | not 0 | alert |
| NVMe wear | `percentage_used` ≥ 90 % | warning |

srvmon reports an **increase** of a counter compared to the stored value. On the first read there is no stored value; a value greater than 0 is then reported once.

### Storage space

For every path in `SPACE_PATHS` the usage is checked: warning from `SPACE_WARN`, alert from `SPACE_CRIT`. Paths that are not mounted (e.g. array stopped) are skipped.

Included by default:

- `/mnt/user`: array and pools combined
- `/mnt/cache`: cache pool
- `/var/log`: tmpfs; a full log is a common Unraid problem
- `/var/lib/docker`: Docker image or directory

### Docker

| Check | Condition | Level |
|---|---|---|
| Health check | container reports `unhealthy` (applies to all containers) | warning |
| Watched containers | container from `DOCKER_WATCH` is not running | alert |
| Container missing | name from `DOCKER_WATCH` does not exist | warning |
| Mode `*` | a container changes from running to not running (one-off event) | warning |

### VMs

| Check | Condition | Level |
|---|---|---|
| Crash | VM state `crashed` (applies to all VMs) | alert |
| Watched VMs | VM from `VM_WATCH` is not running | alert |
| Mode `*` | a VM changes from running to not running (one-off event) | warning |

### System

| Check | Condition | Level |
|---|---|---|
| Reboot | new `boot_id` since the last run (one-off event) | warning |
| CPU load | 5-minute load relative to the number of logical CPUs, from `CPU_LOAD_WARN` % | warning |
| RAM | usage (based on `MemAvailable`) from `RAM_WARN` % | warning |

### UPS (apcupsd)

| Check | Condition | Level |
|---|---|---|
| On battery | status contains `ONBATT` | alert |
| Connection lost | status contains `COMMLOST` | warning |
| Replace battery | status contains `REPLACEBATT` | warning |

If apcupsd is not running, the check is skipped.

### Docker updates

| Check | Condition | Level |
|---|---|---|
| Update available | Unraid's update check reports a newer remote image (once per new update, with container names) | normal |
| Container updated | an existing container runs with a new image ID and the same image reference (one-off event, with old and new version if available) | normal |
| Image changed | an existing container uses a different image reference, e.g. another tag (one-off event) | normal |

**Available updates:** srvmon does not query registries itself but reads the result of Unraid's own update check. How current this information is depends on its schedule, which you set under *Settings > Notifications*.

**Performed updates:** srvmon remembers the image ID of every container. When Unraid updates a container, it recreates it with the new image, so the image ID changes. srvmon detects this regardless of how the update was triggered, including automatic updates by other plugins. The version is taken from the image labels in this order: `org.opencontainers.image.version`, `version`, `build_version`. Without these labels the notification contains no version.

Not reported as an update:

- recreating a container with an unchanged image, e.g. after editing the template,
- renamed or newly created containers (no previous state),
- updates that happen while srvmon is not installed.

**Important:** all update notifications have the level `normal`. To receive them by email, the email agent for **Notices** must be enabled under *Settings > Notifications*.

### Unraid OS and plugin updates

| Check | Condition | Level |
|---|---|---|
| Unraid OS update | Unraid's OS update check reports a newer version that is not ignored (once per new version) | normal |
| Plugin update | Unraid's plugin check downloaded a newer `.plg` for an installed plugin (once per new version, all plugins of a run in one notification) | normal |
| Plugin updated | the installed version of a plugin changed since the last run (one-off event, old and new version, all plugins of a run in one notification) | normal |

**Source:** as with Docker, srvmon does not contact the release server or plugin repositories itself. It reads the results of Unraid's own checks:

- **Unraid OS:** `/tmp/unraidcheck/result.json`, written by Unraid's OS update check. The installed version comes from `/etc/unraid-version`. Versions ignored in the *Update OS* dialog (`/tmp/unraidcheck/ignored.json`) are not reported. If the update period of the license has ended (`isEligible` is `false`), the update is still reported, with a note that the license must be extended first and the end date of the update period.
- **Plugins:** Unraid's plugin check downloads the current `.plg` of every installed plugin to `/tmp/plugins/`. srvmon compares its version with the installed plugin in `/var/log/plugins/`, using the same rules as Unraid: string comparison of the version, and the `min`/`max` OS version of the new plugin must fit the installed Unraid version. srvmon itself is included.

Both results live in RAM and are empty after a reboot until Unraid's next check. In that case srvmon reports nothing and keeps its last state, so known updates are not reported again.

**Performed plugin updates:** srvmon remembers the version of every installed plugin (`/var/log/plugins/*.plg`, without the Unraid OS itself). When the version changes, it reports the update with old and new version. This works regardless of how the update was installed: via the *Plugins* tab, the command line, or an update applied during boot. Newly installed plugins are not reported, as there is no previous version. A plugin that is missing in a run, e.g. because it is still being installed during boot, loses its stored version; this avoids false reports, but an update in exactly that moment is not reported. srvmon reports its own updates as well, starting with the update after 2026.10.18n.

**Schedule:** how current the information is depends on Unraid's checks. Enable them under *Settings > Notifications* (*Unraid OS update check* and *Plugins update check*). Opening *Tools > Update OS* or clicking *Check for updates* in the *Plugins* tab also refreshes the results.

**Duplicate notifications:** Unraid's scheduled checks send their own notification as well, so an update usually appears twice. What srvmon adds is the webhook (e.g. Alexa) and the overview on the status page. If the duplicates bother you, set `CHECK_OS_UPDATES` or `CHECK_PLUGIN_UPDATES` to `no`; the corresponding part of the status page card then disappears as well. Unraid has no notification for performed plugin updates, so `CHECK_PLUGIN_UPDATED` causes no duplicates.

### TLS certificates (Nginx Proxy Manager)

srvmon checks certificates in two independent ways.

**Certificate files:** In every directory listed in `CERT_DIRS`, srvmon looks for `<directory>/*/fullchain.pem`, i.e. the Let's Encrypt `live` layout. Nginx Proxy Manager names the subdirectories `npm-1`, `npm-2` and so on. Notifications show the domain name from the certificate instead.

**Served certificates:** For every entry in `CERT_HOSTS`, srvmon opens a TLS connection with SNI and evaluates the certificate that is actually served.

| Check | Condition | Level |
|---|---|---|
| Expiry (file and live) | valid for at most `CERT_WARN_DAYS` days | warning |
| Expiry (file and live) | valid for at most `CERT_CRIT_DAYS` days, or expired | alert |
| Trust (live) | invalid chain or hostname, if `CERT_VERIFY` is enabled | warning |
| Outdated certificate served (live) | a certificate file for the host is valid more than one day longer than the served certificate | warning |
| Unreachable (live) | no TLS connection possible | warning |
| Directory missing | path from `CERT_DIRS` does not exist | warning |
| File unreadable | `fullchain.pem` cannot be parsed | warning |

**Why both:** Let's Encrypt normally renews certificates 30 days before expiry. With the default threshold of 20 days, a warning therefore means that renewal has been failing for at least 10 days. Comparing file and served certificate also catches the case where renewal worked but nginx has not loaded the new certificate.

**Superseded certificates:** When Nginx Proxy Manager recreates a certificate, it creates a new `npm-<id>` directory and the old one may remain. srvmon therefore ignores a certificate if another certificate covers all of its domains and expires later. On the status page it is shown greyed out with the note "superseded by npm-…". Certificates without a successor, e.g. for domains no longer in use, are still reported; delete them in NPM or add them to `CERT_IGNORE`.

**Check interval:** certificates are checked at most once per hour, independent of `INTERVAL`. `--force` triggers the check immediately. Existing problems are kept between checks, including reminders.

**Host notation:** several entries separated by commas; without a port, 443 is used.

| Entry | Meaning |
|---|---|
| `cloud.example.com` | connect to cloud.example.com on port 443 |
| `cloud.example.com:8443` | as above, port 8443 |
| `cloud.example.com@192.168.1.10` | connect to 192.168.1.10:443 with SNI cloud.example.com |
| `cloud.example.com@192.168.1.10:8443` | connect to 192.168.1.10:8443 with SNI cloud.example.com |

The `@address` form is useful if the server cannot reach its own public address, e.g. because the router does not support NAT loopback (hairpin NAT). The address is then the LAN IP of the server and the host port on which Nginx Proxy Manager accepts HTTPS. `docker port <NPM container>` shows the mapping; look for the container port 443 (image `jc21/nginx-proxy-manager`) or 4443 (image `jlesage/nginx-proxy-manager`).

**Finding the path:** the default path `/mnt/user/appdata/NginxProxyManager/letsencrypt/live` does not match every NPM template. The volume mapping of the container in the Docker tab shows the actual path: look for the host path mapped to `/etc/letsencrypt` (or `/config/letsencrypt`) inside the container and append `live`. To check:

```bash
ls /mnt/user/appdata/NginxProxyManager/letsencrypt/live/
```

Custom certificates uploaded in NPM live in a different directory. You can add it to `CERT_DIRS` as long as it uses the same `<directory>/*/fullchain.pem` layout.

### Reachability from the internet (UptimeRobot)

The checks above run on the server itself. They cannot see whether a service is reachable **from the internet**: a broken DynDNS entry, router port forwarding, IPv6 release or an application that answers with an error page (e.g. `502 Bad Gateway` from Nginx Proxy Manager) go unnoticed. srvmon therefore uses the free service [UptimeRobot](https://uptimerobot.com), which checks the URLs from outside every 5 minutes. srvmon creates the monitors, reads their results and reports outages through its own notification chain (Unraid, webhook, Alexa, reminders, "Resolved").

| Check | Condition | Level |
|---|---|---|
| Not reachable from the internet | UptimeRobot reports a monitor as `DOWN` (any monitor of the account, not only those created by srvmon) | alert |
| API key rejected | UptimeRobot answers with HTTP 401/403 | warning |
| UptimeRobot not reachable | the API could not be reached in 3 runs in a row (e.g. the internet connection is down) | warning |
| Invalid entry | an entry of `EXT_MONITORS` cannot be read | warning |
| Monitor could not be created | UptimeRobot refused to create a monitor | warning |
| Monitor created / removed | srvmon created or deleted a monitor (one-off event) | normal |

`LOOKS_DOWN` (UptimeRobot is still confirming an outage) and paused monitors are not reported.

**Entries in `EXT_MONITORS`** (comma-separated):

| Entry | UptimeRobot checks |
|---|---|
| `nextcloud:https://cloud.example.com` | `https://cloud.example.com/status.php` must contain `"maintenance":false` (also catches maintenance mode and error pages) |
| `immich:https://photos.example.com` | `https://photos.example.com/api/server/ping` must contain `pong` |
| `https://app.example.com/health` | HTTP status: 2xx/3xx is up (UptimeRobot's default) |

An explicit path after the host replaces the default path of a preset.

**Local diagnosis:** append `@<LAN-IP>:<NPM-HTTPS-port>` to an entry, e.g. `nextcloud:https://cloud.example.com@192.168.1.10:4443` (the same notation as for `CERT_HOSTS`). When UptimeRobot reports an outage, srvmon then calls the same URL locally through Nginx Proxy Manager and adds the result to the notification:

- *Local check OK*: Nginx Proxy Manager and the application work, so the problem is on the way from the internet (DynDNS, router port forwarding, IPv6).
- *Local check failed* (e.g. `HTTP 502`, no connection, expected content missing): the problem is in Nginx Proxy Manager or the application.

The local call only happens during an outage. It ignores certificate errors; certificates are checked separately.

**Managing the monitors:** srvmon creates a monitor for every entry (interval 5 minutes, tag `srvmon`). If you remove an entry, srvmon deletes its monitor. Monitors without the tag `srvmon` are never changed or deleted, but they are evaluated as well. If an entry is invalid, srvmon deletes nothing in that run, so a typo cannot remove a monitor. If you change an entry, srvmon deletes the old monitor and creates a new one; the history of the old one is lost.

**Setup:**

1. Create a free account at [uptimerobot.com](https://uptimerobot.com).
2. Create an API key under *Integrations & API > API*. srvmon needs the **main API key**, because it creates and deletes monitors.
3. Enter the key and the services in the srvmon settings (section *External checks*) and enable the check.
4. Check the result: `php /usr/local/emhttp/plugins/srvmon/scripts/check.php --force --dry-run` shows what srvmon would create without changing anything. The next regular run creates the monitors and reports this with "External check created".

**UptimeRobot's own alerts:** srvmon does not assign alert contacts to the monitors it creates. If the whole server or the internet connection is down, srvmon cannot report anything; for that case use the [heartbeat](#heartbeat-outage-of-the-whole-server), or add your email address as alert contact to the monitors in UptimeRobot (then you receive outages twice while the server is running).

**Limits of the free plan:** 50 monitors, check every 5 minutes, 10 API requests per minute. srvmon reads all monitors with one request per run (one more per 200 monitors).

---

## Notification logic

There are two kinds of notifications.

**Conditions** are ongoing problems, e.g. a high temperature or a full disk. They are reported:

- when they first occur,
- when the level rises (warning → alert),
- as a reminder "Still active: …" after `REMIND_HOURS` hours while the problem persists (0 = no reminder),
- as "Resolved: …" (level `normal`) once the condition is back to normal, if `NOTIFY_RESOLVED` is enabled.

**Events** are one-off occurrences, e.g. a reboot, an increased SMART counter or a new update. They are reported exactly once.

**Further rules:**

- If a check cannot deliver data, e.g. because the Docker service is off, existing conditions remain unchanged. There is no "Resolved" notification and no reminder in that case.
- At most 10 notifications are sent per run. If there are more, a summary "Further notifications suppressed" follows.
- Every notification is also written to the syslog with the tag `srvmon`.
- Whether a notification is delivered by email is decided by Unraid based on the agent settings per level (see [Installation](#installation)).

The subject has the form `srvmon: <summary>`; the sender is shown in Unraid as "Server Monitor".

### Language

Notifications can be sent in English, German, French or Spanish. The language is set separately for Unraid's notifications (`NOTIFY_LANG`) and for the webhook (`WEBHOOK_LANG`), e.g. English emails and German Alexa announcements.

- Translated: subject, description and the `speech` field of the webhook, including "Still active" reminders and "Resolved" notifications.
- Not translated: values that come from other programs, e.g. the Docker status ("Exited (1) 5 minutes ago"), apcupsd values ("80.0 Percent"), OpenSSL error messages and disk states such as `DISK_DSBL`. Numbers keep the decimal point.
- Always English: the syslog, the settings page and the status page.
- If a text has no translation, it is sent in English. Translations are stored in `include/lang.php`.

---

## Additional notification channels

In addition to Unraid's notifications, srvmon can forward notifications via webhook and report to an external outage monitor. Both are optional and off by default.

### Webhook (e.g. Home Assistant and Alexa)

If `WEBHOOK_URL` is set, srvmon additionally sends every notification from level `WEBHOOK_LEVEL` upwards as an HTTP POST with JSON:

```json
{
  "level": "warning",
  "subject": "Certificate cloud.example.com expires in 9 days",
  "description": "Certificate for cloud.example.com (npm-2, ...) valid until ...",
  "speech": "Warning from server unraid: Certificate cloud.example.com expires in 9 days",
  "lang": "en",
  "host": "unraid",
  "time": 1792330214
}
```

The `speech` field is meant for voice output: short, with level and server name, without the long description. `subject`, `description` and `speech` are in the language set in `WEBHOOK_LANG` (see [Language](#language)); `lang` contains its code (`en`, `de`, `fr` or `es`).

Rules:

- At most 3 webhook calls per run. If there are more notifications, a summary call "… more notifications" follows. This keeps a voice assistant from talking for minutes during an incident with many follow-up notifications.
- If the target is unreachable, this is logged to the syslog (`grep srvmon /var/log/syslog`). The Unraid notification is not affected.
- The test notification (`check.php --test`) is always sent to the webhook as well, regardless of the configured level.

#### Setting up Home Assistant

1. **Alexa integration:** Home Assistant needs an integration that can trigger Alexa announcements. A common choice is the unofficial "Alexa Media Player" integration, installed via HACS. It signs in with your Amazon account and may break at any time due to changes on Amazon's side.
2. **Create a webhook ID.** The ID acts like a password: anyone who knows it can trigger announcements from the local network. Create one in the Unraid terminal:
   ```bash
   echo "srvmon-$(openssl rand -hex 16)"
   ```
3. **Create an automation** (YAML mode) and adjust the ID and entity IDs. It sends every notification as a push message to the Alexa app, so you can read it later if you missed the announcement, and announces it on the Echo device (alerts always, warnings only during the day):
   ```yaml
   alias: Unraid srvmon to Alexa
   mode: queued
   triggers:
     - trigger: webhook
       webhook_id: srvmon-YOUR_GENERATED_ID
       allowed_methods: [POST]
       local_only: true
   actions:
     # push message to the Alexa app: always, with the full text
     - action: notify.alexa_media
       continue_on_error: true
       data:
         message: "{{ trigger.json.subject }}: {{ trigger.json.description }}"
         target:
           - media_player.echo_living_room
         data:
           type: push
     # voice announcement: alerts always, warnings only during the day
     - if:
         - condition: template
           value_template: "{{ trigger.json.level == 'alert' or (7 <= now().hour < 22) }}"
       then:
         - action: notify.alexa_media
           data:
             message: "{{ trigger.json.speech }}"
             target:
               - media_player.echo_living_room
             data:
               type: announce
   ```
   For `type: push`, one Echo device as `target` is enough; the message goes to the Alexa app of the Amazon account. `continue_on_error` makes sure the announcement still happens if the push fails. Without the push, remove the first action and keep only the content of `then:`.
   You find the entity IDs of your Echo devices (`media_player.…`) under *Settings > Devices & services > Alexa Media Player > Entities*. Under `target:` only `media_player.…` entities belong, never `notify.…`. Check under *Developer tools > Actions* that the action `notify.alexa_media` exists. The syntax `triggers`/`trigger:` and `actions`/`action:` applies to current Home Assistant versions; older versions expect `trigger:`/`platform:` and `action:`/`service:`.

   **All devices:** replace the `target` list of the announcement with a template:
   ```yaml
             target: "{{ integration_entities('alexa_media') | select('match', 'media_player\\.') | list }}"
   ```
   Depending on your Alexa setup, speaker groups may be included, which can lead to duplicate announcements; in that case maintain the list by hand.

   **Push via the Home Assistant app instead:** Alexa Media Player depends on an unofficial Amazon sign-in. The Home Assistant companion app is more reliable for push messages and can deliver alerts as critical notifications. Replace the first action with:
   ```yaml
     - action: notify.mobile_app_my_phone
       data:
         title: "{{ trigger.json.subject }}"
         message: "{{ trigger.json.description }}"
   ```

   **Different devices per level** for the announcement, e.g. alerts everywhere, warnings only in the office:
   ```yaml
             target: >
               {% if trigger.json.level == 'alert' %}
                 {{ integration_entities('alexa_media') | select('match', 'media_player\\.') | list }}
               {% else %}
                 {{ ['media_player.echo_office'] }}
               {% endif %}
   ```
4. **Configure srvmon:** `WEBHOOK_URL` = `http://<Home Assistant IP>:8123/api/webhook/<ID>`, e.g. `http://192.168.1.20:8123/api/webhook/srvmon-...`. If Home Assistant runs on the `br0` network with its own IP, use that IP.
5. **Test Home Assistant directly** from the Unraid terminal:
   ```bash
   curl -X POST -H 'Content-Type: application/json' \
     -d '{"level":"warning","speech":"Test from the server"}' \
     http://<Home Assistant IP>:8123/api/webhook/<ID>
   ```
6. **Test end to end:** `php /usr/local/emhttp/plugins/srvmon/scripts/check.php --test`

**Home Assistant on `br0` with its own IP:** by default, Unraid blocks connections from the host to containers with their own IP on custom networks. srvmon runs on the host and then cannot reach Home Assistant (`curl: (7) Failed to connect`). Fix: *Settings > Docker*, set *Enable Docker* to *No*, enable *Advanced View*, set *Host access to custom networks* to *Enabled*, then set *Enable Docker* back to *Yes*. All containers stop briefly during this.

**Limitation:** if Home Assistant runs as a container on the same Unraid server, this channel fails together with Docker or the server. That is what the heartbeat is for. You can watch the Home Assistant container itself via `DOCKER_WATCH`; that notification then arrives by email or push, not via Alexa.

### Heartbeat (outage of the whole server)

Monitoring that runs on the server cannot report the server's own total failure. Therefore srvmon calls the `HEARTBEAT_URL` on every check run. An external service raises an alarm when these calls stop.

Suitable services include:

- **healthchecks.io:** create a check, set the period to the srvmon interval (default 5 minutes) and a grace time of about 10 minutes, and enter the ping URL as `HEARTBEAT_URL`. The service can notify you by email, Telegram and other channels.
- **Uptime Kuma on another device:** create a monitor of type "Push" and enter its URL.

Make sure the external service does not run on the monitored server and does not notify via Home Assistant on that server.

---

## Status page

The page is located under *Settings > User Utilities > Server Monitor Status*. It shows:

- a summary (number of critical problems and warnings, or "All good") with the time of the live data and of the last check,
- active problems with the time they occurred,
- system: CPU load, RAM and uptime,
- UPS: status, charge, time left and load,
- storage space as a bar per path,
- disks as a table with status, temperature, errors and SMART values,
- Docker and VMs with their state, watched entries are marked; for containers additionally an "Update" hint, the image version and the time of the last detected update,
- available Docker updates,
- Unraid OS and plugin updates: installed OS version, available OS version, plugins with installed and available version, time of Unraid's last check, plugins updated in the last 7 days,
- TLS certificates with source (file or live), expiry date, days left, issuer and notes. These come from the last hourly check; the time is shown below the table.

The page reloads its data every 30 seconds while the browser tab is visible. Colours follow the configured thresholds.

SMART values are not read live for the page; it shows the state of the last check. Disabled checks do not appear on the page.

The settings page *Server Monitor* additionally shows a short list of active problems.

---

## Installation

### Requirements

- Unraid 6.12 or newer
- For email: working mail delivery in Unraid (see below)

### 1. Set up delivery in Unraid

srvmon stores no credentials. Delivery is handled entirely by Unraid.

1. Under *Settings > Notifications* (SMTP settings) enter the SMTP server, port, sender and recipient.
2. Under *Settings > Notification Settings*, enable the email agent (or another agent such as Telegram, Pushover or ntfy) for the desired levels:
   - **Warnings** and **Alerts**: recommended
   - **Notices**: required for update notifications (Docker, Unraid OS, plugins), "Resolved" notifications and the srvmon test notification
3. Enable the schedules for Unraid's Docker, Unraid OS and plugin update checks under *Settings > Notifications*. srvmon uses their results.
4. Use Unraid's test function to check that delivery works in general.

**Microsoft accounts:** Microsoft has disabled or announced the end of password-based SMTP sign-in (basic authentication) and only allows OAuth2, which Unraid's SMTP settings do not support as far as known. For private Outlook.com accounts, password-based SMTP is very likely no longer available. Use a sending account with a provider that still allows SMTP with a password or app password; the recipient can still be your Microsoft address.

### 2. Install the plugin

**Recommended: via URL (with update function)**

In Unraid, under *Plugins > Install Plugin*, enter this URL and click *Install*:

```
https://raw.githubusercontent.com/MarioKonang/unraid-srvmon/main/srvmon.plg
```

Unraid then detects new versions via *Plugins > Check for updates*.

**Alternative: local file (without update function)**

1. Copy `srvmon.plg` to the flash drive, e.g. to `/boot/srvmon.plg`.
2. Run in the terminal:
   ```bash
   plugin install /boot/srvmon.plg
   ```

### 3. Verify the setup

```bash
# send a test notification
php /usr/local/emhttp/plugins/srvmon/scripts/check.php --test

# dry run: shows what would be reported now, without sending anything
php /usr/local/emhttp/plugins/srvmon/scripts/check.php --force --dry-run

# is the cron job active?
grep -r srvmon /etc/cron.d/
```

Then adjust the settings under *Settings > User Utilities > Server Monitor*.

### Updating

- **Installed via URL:** click *Check for updates* under *Plugins* and install the update. For developers: a new version must have a higher `version` entity in the `.plg`, otherwise Unraid does not detect an update. Unraid does not overwrite existing files during an update; the first step of the `.plg` therefore deletes `/usr/local/emhttp/plugins/srvmon` before the files are written. Updating from 2026.10.18l or older to 2026.10.18m or newer works normally, because the cleanup step of the new version runs.
- **Installed from a local file:** remove the old plugin and install the new file:
  ```bash
  plugin remove srvmon.plg
  plugin install /boot/srvmon.plg
  ```

Configuration and state are kept in both cases.

### Uninstalling

Via *Plugins* or in the terminal:

```bash
plugin remove srvmon.plg
```

This removes the cron entry, the program files and the temporary files. Configuration and state in `/boot/config/plugins/srvmon/` are kept. Delete this directory manually for a complete removal.

---

## Configuration

The settings are located under *Settings > User Utilities > Server Monitor*. The page is divided into the sections General, Disks and SMART, Storage space, Docker, Unraid OS and plugins, VMs, System and UPS, TLS certificates, External checks and Additional notification channels. Settings are stored in `/boot/config/plugins/srvmon/srvmon.cfg`. Values not set there are taken from `default.cfg`.

Changes to the file take effect on the next run; no restart is needed.

### General

| Key | Default | Description |
|---|---|---|
| `INTERVAL` | `5` | check interval in minutes (1 to 1440) |
| `REMIND_HOURS` | `24` | reminder for ongoing problems after x hours, `0` = off |
| `NOTIFY_RESOLVED` | `yes` | notify when a problem is resolved |
| `NOTIFY_LANG` | `en` | language of Unraid's notifications: `en`, `de`, `fr` or `es` (see [Language](#language)) |

### Enabling and disabling checks

| Key | Default | Description |
|---|---|---|
| `CHECK_DISKS` | `yes` | disk status, I/O errors, temperature |
| `CHECK_SMART` | `yes` | SMART evaluation (only together with `CHECK_DISKS`) |
| `CHECK_SPACE` | `yes` | storage space |
| `CHECK_DOCKER` | `yes` | Docker containers |
| `CHECK_VMS` | `yes` | VMs |
| `CHECK_SYSTEM` | `yes` | CPU, RAM, reboot |
| `CHECK_UPS` | `yes` | UPS via apcupsd |
| `CHECK_UPDATES` | `yes` | notify about available Docker updates |
| `CHECK_DOCKER_UPDATED` | `yes` | notify about performed container updates (only together with `CHECK_DOCKER`) |
| `CHECK_OS_UPDATES` | `yes` | notify about available Unraid OS updates |
| `CHECK_PLUGIN_UPDATES` | `yes` | notify about available plugin updates |
| `CHECK_PLUGIN_UPDATED` | `yes` | notify about performed plugin updates |

### TLS certificates

| Key | Default | Description |
|---|---|---|
| `CHECK_CERTS` | `yes` | certificate check on/off |
| `CERT_DIRS` | `/mnt/user/appdata/NginxProxyManager/letsencrypt/live` | directories containing `*/fullchain.pem`, comma-separated; empty = file check off |
| `CERT_HOSTS` | empty | hosts to check live: `host`, `host:port`, `host@address` or `host@address:port`; empty = live check off |
| `CERT_WARN_DAYS` | `20` | warning at this many days left or fewer |
| `CERT_CRIT_DAYS` | `7` | alert at this many days left or fewer |
| `CERT_VERIFY` | `yes` | verify trust chain and hostname in the live check |
| `CERT_IGNORE` | empty | domains whose certificate files are ignored, comma-separated |

### External checks

| Key | Default | Description |
|---|---|---|
| `CHECK_EXTERNAL` | `no` | reachability from the internet via UptimeRobot on/off |
| `UPTIMEROBOT_KEY` | empty | main API key of the UptimeRobot account (secret, see [Security](#security)) |
| `EXT_MONITORS` | empty | services to check, comma-separated: `[nextcloud: or immich:]https://host[/path][@LAN-IP:port]`, see [Reachability from the internet](#reachability-from-the-internet-uptimerobot) |

### Additional notification channels

| Key | Default | Description |
|---|---|---|
| `WEBHOOK_URL` | empty | target for webhook calls (HTTP POST with JSON); empty = off |
| `WEBHOOK_LEVEL` | `warning` | minimum level for webhooks: `normal` (everything), `warning` (warnings and alerts) or `alert` |
| `WEBHOOK_LANG` | `en` | language of the webhook texts: `en`, `de`, `fr` or `es` |
| `HEARTBEAT_URL` | empty | called via GET on every check run; empty = off |

### Thresholds

| Key | Default | Description |
|---|---|---|
| `TEMP_HDD_WARN` | `45` | °C, warning for HDD/SATA SSD |
| `TEMP_HDD_CRIT` | `50` | °C, alert for HDD/SATA SSD |
| `TEMP_NVME_WARN` | `60` | °C, warning for NVMe |
| `TEMP_NVME_CRIT` | `70` | °C, alert for NVMe |
| `SPACE_WARN` | `85` | % usage, warning |
| `SPACE_CRIT` | `95` | % usage, alert |
| `CPU_LOAD_WARN` | `90` | % of the 5-minute load relative to the number of logical CPUs |
| `RAM_WARN` | `90` | % RAM usage |

### Paths and watch lists

| Key | Default | Description |
|---|---|---|
| `SPACE_PATHS` | `/mnt/user,/mnt/cache,/var/log,/var/lib/docker` | mount points to check, comma-separated |
| `DOCKER_WATCH` | `*` | watched containers (see below) |
| `VM_WATCH` | empty | watched VMs (see below) |

### Watch lists in detail

`DOCKER_WATCH` and `VM_WATCH` support three modes:

| Value | Behaviour |
|---|---|
| empty | no targeted monitoring; only the general checks apply: `unhealthy` for containers, `crashed` for VMs |
| `name1,name2` | as long as a listed entry is not running, an alert is active, including reminders |
| `*` | every change from running to not running is reported once, including manual stops |

Notes:

- Names must match exactly; case matters. These commands show the correct names:
  ```bash
  docker ps -a --format '{{.Names}}'
  virsh list --all --name
  ```
- Do not mix `*` with names. If `*` is present, names are ignored.
- Names containing a comma are not possible, as the comma is the separator.
- For Docker, srvmon reports names that do not exist. VMs have no such check.

### Example `srvmon.cfg`

```ini
INTERVAL="5"
SPACE_PATHS="/mnt/user,/mnt/cache,/var/log,/var/lib/docker"
DOCKER_WATCH="nextcloud,mariadb,NginxProxyManager"
VM_WATCH="HomeAssistant"
RAM_WARN="95"
REMIND_HOURS="12"
CERT_HOSTS="cloud.example.com@192.168.1.10,app.example.com@192.168.1.10"
CERT_IGNORE="old.example.com"
WEBHOOK_LANG="de"
```

---

## Command line

```bash
php /usr/local/emhttp/plugins/srvmon/scripts/check.php [options]
```

| Option | Effect |
|---|---|
| none | normal run; skipped if the interval has not elapsed yet |
| `--force` | ignore the interval (and the hourly certificate limit) and check immediately |
| `--dry-run` | print notifications to the console only; send nothing, save no state. If a language other than English is set, the translated Unraid notification and webhook announcement are printed below each line |
| `--test` | send a test notification (also to the webhook) |

Note: `--dry-run` only shows what would be sent **in this run**. Problems already stored in the state are not repeated. The status page shows the complete current state.

The cron job calls the script every minute; the script checks the interval itself. A lock prevents parallel runs.

---

## Files and paths

### Program files (RAM, recreated from the `.plg` on boot)

| Path | Content |
|---|---|
| `/usr/local/emhttp/plugins/srvmon/README.md` | short description for the plugin list |
| `/usr/local/emhttp/plugins/srvmon/default.cfg` | default values |
| `/usr/local/emhttp/plugins/srvmon/include/lib.php` | shared functions and data collectors |
| `/usr/local/emhttp/plugins/srvmon/include/lang.php` | translations of the notification texts |
| `/usr/local/emhttp/plugins/srvmon/include/data.php` | JSON endpoint for the status page |
| `/usr/local/emhttp/plugins/srvmon/scripts/check.php` | check script |
| `/usr/local/emhttp/plugins/srvmon/srvmon.page` | settings page |
| `/usr/local/emhttp/plugins/srvmon/SrvmonStatus.page` | status page |

### Persistent files (flash)

| Path | Content |
|---|---|
| `/boot/config/plugins/srvmon.plg` | plugin definition, installed by Unraid on boot |
| `/boot/config/plugins/srvmon/srvmon.cfg` | your settings |
| `/boot/config/plugins/srvmon/srvmon.cron` | cron entry |
| `/boot/config/plugins/srvmon/state.json` | active problems, SMART baseline, last known container and VM states, image ID and version per container, known Docker, OS and plugin updates, installed plugin versions with time of the last update |

`state.json` is only written when its content actually changes, to spare the flash drive.

### Temporary files (RAM)

| Path | Content |
|---|---|
| `/tmp/srvmon/lastrun` | time of the last run |
| `/tmp/srvmon/lock` | lock against parallel runs |
| `/tmp/srvmon/status.json` | status of the last run for the WebGUI |
| `/tmp/srvmon/lastcert` | time of the last certificate check |
| `/tmp/srvmon/certs.json` | result of the last certificate check for the status page |
| `/tmp/srvmon/ext.json` | result of the last external check for the status page |
| `/tmp/srvmon/ext_fail` | number of failed UptimeRobot requests in a row |

---

## Data sources

| Area | Source |
|---|---|
| Disks | `/var/local/emhttp/disks.ini` |
| SMART | `smartctl -n standby -j -H -A /dev/<device>` |
| Storage space | `mountpoint -q` and the PHP functions `disk_total_space` and `disk_free_space` |
| Docker | `docker ps -a`, `docker inspect` (image ID, image reference, labels) |
| VMs | `virsh list --all --name`, `virsh domstate` |
| CPU, RAM, uptime | `/proc/loadavg`, `/proc/meminfo`, `/proc/uptime`, `nproc` |
| Reboot | `/proc/sys/kernel/random/boot_id` |
| UPS | `apcaccess status` |
| Docker updates | `/var/lib/docker/unraid-update-status.json` |
| Unraid OS updates | `/tmp/unraidcheck/result.json`, `/tmp/unraidcheck/ignored.json`, `/etc/unraid-version` |
| Plugin updates | `/tmp/plugins/*.plg` compared with `/var/log/plugins/*.plg` |
| Performed plugin updates | `version` attribute of `/var/log/plugins/*.plg` |
| Certificate files | `openssl x509` on `<CERT_DIRS>/*/fullchain.pem` |
| Served certificates | `openssl s_client` with SNI and `-verify_hostname` |
| Reachability from the internet | UptimeRobot API v3 (`https://api.uptimerobot.com/v3/monitors`) via `curl`; local diagnosis with `curl --resolve` |
| Delivery | `/usr/local/emhttp/webGui/scripts/notify`, `curl` for webhook and heartbeat |

All external calls run with `timeout`, so a hanging service does not block the run.

---

## Security

- **No credentials in the plugin.** Email delivery uses Unraid's SMTP configuration.
- **Read-only checks.** Apart from its own paths, srvmon writes nothing to the server; external commands are only called for reading.
- **Status page behind the login.** The JSON endpoint lives in the plugin directory of the WebGUI and is therefore protected by the WebGUI login. It does not send disk IDs, which usually contain serial numbers, to the browser.
- **Webhook ID and heartbeat URL are secrets.** They are stored in `srvmon.cfg` on the flash drive, not in the `.plg`. Do not publish the configuration file. In Home Assistant keep `local_only: true` so the webhook can only be triggered from the local network.
- **UptimeRobot API key.** The main key gives full access to the UptimeRobot account (monitors, alert contacts). It is stored in `srvmon.cfg` on the flash drive like the webhook ID; the settings page shows it as a password field and the status page never receives it. srvmon passes it to `curl` via a temporary file with mode 0600, not on the command line, so it does not appear in the process list. If the key is leaked, delete it in UptimeRobot and create a new one.
- **Data leaving the server.** Webhook calls contain the notification texts; the heartbeat only consists of the call itself. With external checks enabled, srvmon sends the configured URLs to UptimeRobot and reads the status of all monitors of the account.
- **Root privileges.** Plugins run as root. Only install from sources you control. Anyone with write access to the repository behind `pluginURL` can run arbitrary code on the server through an update. Protect the GitHub account with two-factor authentication.
- **No tokens in the URL.** For private repositories, do not put an access token into `pluginURL`; it would be stored in plain text on the flash drive and in the log.
- **WebGUI over HTTP.** If the Unraid WebGUI is only reachable via HTTP, the root password travels unencrypted when logging in. Unraid offers HTTPS under *Settings > Management Access*.

---

## Known limitations

- **`DOCKER_WATCH="*"`** also reports manual stops and stops by backup plugins. For fewer notifications, list specific names instead.
- **ZFS:** the ARC counts as used RAM. On ZFS systems, set `RAM_WARN` higher.
- **SMART behind some controllers:** disks that require a `-d` type for `smartctl` provide no SMART data. Unraid's own per-disk SMART setting is not evaluated.
- **Disk status:** for assigned slots, every value other than `DISK_OK` raises an alert, including transitional states such as adding a new disk.
- **Missing disks:** that an assigned but missing disk keeps its ID in `disks.ini` is an assumption and has not been tested on a real system.
- **`/mnt/user`:** usage is a combined value across array and pools. For individual disks, add the paths `/mnt/diskN` to `SPACE_PATHS`.
- **CPU temperature** is not monitored, because the sensor configuration varies a lot between hardware.
- **Unraid OS and plugin updates** are only as current as Unraid's own checks; srvmon does not trigger them. After a reboot there is no information until the next check.
- **Update hint on the status page:** after an update, the "Update" hint may remain until Unraid's own update check runs again.
- **Image to container mapping:** image references without a tag are treated as `:latest`. With different spellings of the same registry (e.g. with and without `docker.io/`), the mapping may fail; the notification then names only the image.
- **Orphaned certificates:** srvmon does not read the NPM database and therefore does not know which certificates NPM actually uses. Superseded certificates are detected (see above); certificates of unused domains without a successor are still reported.
- **Trust verification:** it uses the CA certificates of the Unraid system. If they are missing, valid Let's Encrypt certificates are also reported as untrusted. In that case set `CERT_VERIFY="no"`.
- **Array disks:** if appdata is on the array instead of a pool, the hourly file check may wake a sleeping disk.
- **Detection delay:** resolution equals the check interval. A problem that appears and disappears between two runs is not detected.

---

## Verification status

srvmon was developed and tested with simulated Unraid files and tools and has since been used on Unraid 7.2.3 and 7.2.4.

**Confirmed on Unraid 7.2.3 and 7.2.4:**

- installation via `plugin install`, plugin list entry
- cron entry via `update_cron` (fix for the missing entry after installation is included since 2026.10.18e)
- reading `disks.ini`, including the empty parity2 slot with `DISK_NP_DSBL`
- saving settings via `/update.php`
- status page rendering
- certificate files of Nginx Proxy Manager (image `jlesage`), detection of superseded certificates
- live check with `openssl s_client`, including trust verification with the system's CA certificates
- webhook to Home Assistant and Alexa announcement
- format of the OS update check result `/tmp/unraidcheck/result.json` (fields `isNewer`, `version`, `isEligible`, `params.update_exp`) and the downloaded `.plg` files in `/tmp/plugins/` (checked on 7.2.4)
- update via the *Plugins* tab replaces the program files (since 2026.10.18m)

**Not yet confirmed:**

| Assumption | How to check |
|---|---|
| UptimeRobot API v3 as described in its OpenAPI specification (tested against a replica, not against the real service): monitor list, creating keyword monitors with `tagNames`, deleting | enable the external check, run `check.php --force --dry-run`, then wait for "External check created" |
| Immich answers `/api/server/ping` with `pong` (path of current versions) | open `https://<immich host>/api/server/ping` in the browser |
| cron entry is present after a reboot without manual action | after a reboot: `date -d @$(cat /tmp/srvmon/lastrun)` |
| format of the Docker update file (fields `local` and `remote`) | `head -c 500 /var/lib/docker/unraid-update-status.json` |
| Docker converts `\t` in `--format` templates to tabs | `docker inspect --format '{{.Name}}\t{{.Image}}' $(docker ps -aq)` must print tab-separated lines |
| plugin attributes `icon`, `launch` and `support` are evaluated in the plugin list | check the plugin list: icon, clickable name, support link |
| header `Markdown="false"` on the status page | open the status page and check for rendering errors |

---

## Troubleshooting

| Problem | What to do |
|---|---|
| No email | run `check.php --test`. If the notification appears in Unraid's bell but not as an email, SMTP or the email agent under *Settings > Notifications* is the problem. |
| No Alexa announcement | run `check.php --test`; in Home Assistant check the automation's trace (*Settings > Automations*, three-dot menu, *Traces*); search the Unraid syslog for "Webhook failed" |
| `curl: (7) Failed to connect` to Home Assistant | Home Assistant runs with its own IP on `br0`; enable *Host access to custom networks* (see [Webhook](#webhook-eg-home-assistant-and-alexa)) |
| Trace notifications | `grep srvmon /var/log/syslog` |
| What would be reported? | `check.php --force --dry-run` (shows only new notifications, see [Command line](#command-line)) |
| Is the cron job running? | `date -d @$(cat /tmp/srvmon/lastrun)` must show a time within the interval, `grep -r srvmon /etc/cron.d/` the entry with `check.php` |
| Cron entry missing | run `/usr/local/sbin/update_cron` and check again |
| Settings are not saved | check `cat /boot/config/plugins/srvmon/srvmon.cfg` and the browser console |
| Status page shows "Could not load data" | run `php /usr/local/emhttp/plugins/srvmon/include/data.php` in the terminal and check the output for PHP errors |
| Run the certificate check now | `check.php --force --dry-run` (also bypasses the hourly limit) |
| TLS check fails with "No TLS connection" | use the notation `host@<LAN-IP>:<NPM HTTPS port>`, see [TLS certificates](#tls-certificates-nginx-proxy-manager) |
| Stale notifications stick | reset the state: `rm /boot/config/plugins/srvmon/state.json`. Existing problems are then reported again and SMART counters > 0 are repeated once. |

---

## Changelog

### 2026.10.18o
- Reachability from the internet via UptimeRobot: srvmon creates and deletes the monitors (presets for Nextcloud and Immich), reads their status and reports outages through its notification chain
- Local diagnosis during an outage via Nginx Proxy Manager: says whether the problem is on the way from the internet or in the application
- Status page: new card "Reachability from the internet"
- New options `CHECK_EXTERNAL`, `UPTIMEROBOT_KEY` and `EXT_MONITORS`

### 2026.10.18n
- Notification when a plugin was updated (installed version changed), with old and new version; all plugins of a run in one notification
- Status page: plugins updated in the last 7 days
- New option `CHECK_PLUGIN_UPDATED`
- Language of the notifications selectable: English, German, French or Spanish, separately for Unraid's notifications (`NOTIFY_LANG`) and the webhook (`WEBHOOK_LANG`); the webhook payload contains the new field `lang`

### 2026.10.18m
- Fix: updates via the Plugins tab did not replace the program files, because Unraid does not overwrite existing files. The previous version kept running although the new version was shown. The installation now removes the old program files first; configuration and state are kept.
- Unraid OS update: if the update period of the license has ended (`isEligible` in the check result), the notification and the status page say so

### 2026.10.18l
- Notification about available Unraid OS and plugin updates, based on Unraid's own update checks, once per new version
- Status page: new card "Unraid OS and plugins"
- New options `CHECK_OS_UPDATES` and `CHECK_PLUGIN_UPDATES`
- Documentation: Home Assistant automation additionally sends a push message to the Alexa app (or the Home Assistant app), so missed announcements can be read later

### 2026.10.18k
- All texts (user interface, notifications, documentation) translated to English
- Dates shown in ISO format (YYYY-MM-DD)

### 2026.10.18j
- Published on GitHub, updates via the Plugins tab

### 2026.10.18i
- Webhook delivery, e.g. to Home Assistant for Alexa announcements; minimum level configurable, at most 3 calls per run plus summary
- Heartbeat URL for external outage monitoring (healthchecks.io, Uptime Kuma)

### 2026.10.18h
- Settings page grouped into sections with separators, fields sorted by topic

### 2026.10.18g
- Certificate files superseded by a newer certificate for the same domains are ignored
- `CERT_HOSTS` also accepts the port after the address (`host@address:port`)
- Hint about the LAN address notation when the live check cannot connect

### 2026.10.18f
- Empty disk slots without device and ID are ignored; fixes the false alarm for an empty parity2 slot with `DISK_NP_DSBL`

### 2026.10.18e
- Cron entry is reloaded after the installation has finished. Cause of the bug: `update_cron` only includes plugins that are already registered, but srvmon is not registered yet while its own install script runs.
- Icon, description and link to the settings page in the plugin list

### 2026.10.18d
- TLS certificates: expiry of certificate files (Nginx Proxy Manager) and of live served certificates
- Detection of nginx still serving the old certificate after a renewal
- Trust chain and hostname verification, certificate overview on the status page

### 2026.10.18c
- Notification when a container was started with a new image, including the version from the image labels
- Notification about available updates names the affected containers
- Status page: update hint, version and time of the last update per container
- New option `CHECK_DOCKER_UPDATED`

### 2026.10.18b
- New page "Server Monitor Status" with live view, refreshed every 30 s
- Data collection moved to `include/lib.php`, JSON endpoint `include/data.php`

### 2026.10.18a
- CSRF token set explicitly in the settings form

### 2026.10.18
- First version: disks, SMART, storage, Docker, VMs, system, UPS, Docker updates
