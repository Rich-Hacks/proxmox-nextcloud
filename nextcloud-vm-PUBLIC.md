# Nextcloud VM (nextcloud-vm) — Appliance Runbook

> **🌐 PUBLIC / SANITISED VERSION**
> Hostnames, IPs, MACs and identifiers replaced with placeholders (`example.com`, RFC 1918 / documentation values). No secrets appear in this document. Substitute your own values throughout.

| | |
|---|---|
| **Maintainer** | Richard Carragher (RC COMMS) |
| **Last updated** | 10 June 2026 |
| **Document version** | 1.0 |
| **Guest** | VM 111 `nextcloud-vm` on node `pve` (Proxmox VE) |
| **Appliance** | TurnKey Nextcloud 18.1 (Debian 12 bookworm), provisioned via community-scripts ProxmoxVE helper |
| **Nextcloud** | Hub 26 Spring (34.0.0) |
| **Stack** | Apache 2.4 · PHP 8.3.31 · MariaDB 10.11.14 · Redis (unix socket) |
| **Classification** | Public |

---

## Related documents

- **`proxmox-infrastructure-PUBLIC.md`** — cluster master documentation — VM placement, storage, networking context.
- **`eurooffice-nextcloud-PUBLIC.md`** — Euro-Office DocumentServer (CT 141) integration: reverse proxy, JWT, connector. This VM hosts the Apache layer that fronts both services.

## Table of contents

1. [Overview](#1-overview)
2. [VM specification](#2-vm-specification)
3. [Guest stack](#3-guest-stack)
4. [Web layer: Apache, TLS, DNS](#4-web-layer-apache-tls-dns)
5. [Nextcloud configuration](#5-nextcloud-configuration)
6. [Apps inventory](#6-apps-inventory)
7. [Operational runbook](#7-operational-runbook)
8. [Security posture](#8-security-posture)
9. [Troubleshooting](#9-troubleshooting)
10. [Known quirks](#10-known-quirks)
11. [Explain like I'm 5](#11-explain-like-im-5)
12. [References](#12-references)
13. [Appendices](#13-appendices)

---

## 1. Overview

VM 111 (`nextcloud-vm`) is the family cloud: file sync/share, photos, calendars/contacts (DAV), and — since June 2026 — collaborative office editing via the Euro-Office connector to CT 141. It is the **only** guest exposed to the internet: the consumer router forwards WAN 80/443 to 10.0.0.40, where Apache terminates TLS for two public hostnames and reverse-proxies one of them to the document server LXC.

### Why a VM rather than an LXC?

The Nextcloud deployment is a self-contained appliance image (TurnKey) bundling its own OS, webserver, database and configuration console — the classic case for KVM over LXC on this cluster (see cluster doc §1). The community-scripts ProxmoxVE "Nextcloud VM" helper was used for provisioning; under the hood it deploys the TurnKey Nextcloud appliance, which is why the guest identifies as `turnkey-nextcloud-18.1-bookworm-amd64` while the VM carries the `community-script` tag.

---

## 2. VM specification

From `qm config 111` (10 June 2026):

| Property | Value |
|---|---|
| VMID / name | 111 / `nextcloud-vm` |
| Node | pve |
| Machine / BIOS | q35 / SeaBIOS (+ 4 MB `efidisk0` present but SeaBIOS boots `scsi0` directly) |
| vCPU / RAM | 2 cores / 4096 MB |
| `scsi0` | `local-lvm:vm-111-disk-1`, 40 GB, discard=on, ssd=1 |
| `scsi1` | `local-lvm:vm-111-disk-2`, 828 MB, discard=on, ssd=1 |
| SCSI HW | virtio-scsi-pci |
| NIC | virtio, MAC `<MAC>`, bridge `vmbr0` |
| IP | 10.0.0.40/24 (DHCP — **reservation required**, see cluster doc backlog) |
| QEMU guest agent | enabled (gives clean shutdowns and fsfreeze-consistent snapshot backups) |
| Onboot | yes — `startup order=4, up=60` |
| Latest snapshot | `Update_20260610_160603` (BassT23 updater pre-update) |
| UUID / VMGENID | `<UUID>` / `<VMGENID>` |
| Tags | `community-script` |

### Guest filesystem

| | |
|---|---|
| Root | `/dev/mapper/turnkey-root` (LVM), 36 GB — 13 GB used (38%) |
| Nextcloud data | `/var/www/nextcloud-data` — **on the root filesystem** |
| Nextcloud code | `/var/www/nextcloud` |

> **Capacity note:** user data shares the 36 GB root volume. At 38% with light family use this is fine, but photo libraries grow fast. The escape paths, in order of preference: grow `scsi0` + LVM + filesystem online; or add a dedicated virtual disk for `/var/www/nextcloud-data`; or external storage (`files_external` is already enabled) against the NAS (CT 101). Decide before, not after, it fills.

---

## 3. Guest stack

| Layer | Detail |
|---|---|
| OS | Debian 12 (bookworm), TurnKey 18.1 build |
| Web | Apache 2.4 (`mpm_prefork` per TurnKey default), TLS on :443, redirect vhost on :80 |
| PHP | 8.3.31 NTS (CLI + Apache module) |
| DB | MariaDB 10.11.14, db `nextcloud`, user `nextcloud`, prefix `oc_`, utf8mb4 |
| Cache/locking | Redis via unix socket `/var/run/redis/redis.sock` — both `memcache.local` and `memcache.locking`; `filelocking.enabled = true` |
| Background jobs | **cron mode** — `www-data` crontab runs `php -f /var/www/nextcloud/cron.php` every 5 minutes |
| Mail | SMTP via <SMTP_HOST>:465 (SSL, auth `<SMTP_USER>`), from `<MAIL_FROM>`, TLS peer verification on |
| Management consoles | Webmin `https://10.0.0.40:12321` **enabled (LAN-only — never forward)**; shellinabox :12320 **disabled** |
| No `sudo` | TurnKey ships without it — see Quirk 1 / the `occ` wrapper (Appendix B) |

---

## 4. Web layer: Apache, TLS, DNS

### 4.1 DNS (Cloudflare zone `example.com`)

| Record | Type | Value | Proxy |
|---|---|---|---|
| `nextcloud` | A | <WAN_IP> | DNS only |
| `office` | A | <WAN_IP> | DNS only |

Grey-cloud deliberately (Cloudflare proxy's 100 MB upload cap breaks large syncs). Residential WAN IP — DDNS updater on the cluster doc backlog.

### 4.2 Virtual hosts

| File | Hostname | Role |
|---|---|---|
| `/etc/apache2/sites-available/nextcloud.conf` | `cloud.example.com` (+ default for bare-IP requests) | Serves Nextcloud from `/var/www/nextcloud`; port-80 vhost 301-redirects to HTTPS **and must remain — dehydrated's HTTP-01 renewals depend on it**; HSTS `max-age=15552000; includeSubDomains` |
| `/etc/apache2/sites-available/office.conf` | `office.example.com` | TLS-terminating reverse proxy (HTTP + WebSocket) → CT 141 nginx :80. Full file in the eurooffice runbook, Appendix A |

Enabled modules beyond TurnKey defaults: `proxy proxy_http proxy_wstunnel rewrite headers`.

### 4.3 Certificates — two mechanisms, both on this VM

| Hostname | Challenge / client | Cert location | Renewal |
|---|---|---|---|
| `nextcloud.…` | HTTP-01 / TurnKey confconsole (dehydrated) | `/etc/ssl/private/cert.pem` — referenced **globally** in `/etc/apache2/mods-available/ssl.conf`; the nextcloud vhost inherits it (no per-vhost cert directives) | `/etc/cron.daily/confconsole-dehydrated`; Let's Encrypt (issuer E8 at time of writing), ~90-day cycle |
| `office.…` | DNS-01 / certbot + `python3-certbot-dns-cloudflare` | `/etc/letsencrypt/live/office.example.com/` — per-vhost directives in `office.conf` override the global default | `certbot.timer`; deploy hook reloads Apache; Cloudflare token (zone-scoped) in `/root/.secrets/cloudflare.ini` (0600) |

Health checks: `openssl x509 -in /etc/ssl/private/cert.pem -noout -dates -issuer` and `certbot renew --dry-run` — both **on this VM** (running the dry-run on CT 141 reports "no renewals", as nothing lives there).

---

## 5. Nextcloud configuration

Non-secret `config.php` state as documented 10 June 2026 (secrets — `secret`, `passwordsalt`, DB and SMTP passwords, `updater.secret` — live only in `config.php` and the password manager):

| Key | Value | Note |
|---|---|---|
| `version` | 34.0.0.12 | Hub 26 Spring |
| `trusted_domains` | `localhost`, `10.0.0.40`, `10.0.0.150`, `cloud.example.com` | `.150` entry — identify or remove (backlog) |
| `datadirectory` | `/var/www/nextcloud-data` | on root FS — see §2 capacity note |
| `overwrite.cli.url` | `https://cloud.example.com` | fixed 10 Jun 2026 (was `http://localhost`) |
| `default_phone_region` | `GB` | set 10 Jun 2026 |
| `log_rotate_size` | 104857600 (100 MB) | set 10 Jun 2026 |
| `memcache.local` / `memcache.locking` | Redis (unix socket) | |
| `loglevel` / `logfile` | 3 / `/var/www/nextcloud-data/nextcloud.log` | |
| `maintenance_window_start` | 1 (01:00 UTC) | heavy background jobs window |
| `mysql.utf8mb4` | true | |
| Mail | SMTP SSL :465, the mail provider, verified peer | |
| `config_preset` | 2 | TurnKey preset |

Connector app config (`eurooffice`): `DocumentServerUrl = https://office.example.com/`, `DocumentServerInternalUrl = http://10.0.0.41/`, `StorageUrl = https://cloud.example.com/`, `jwt_secret = <in password manager>`.

---

## 6. Apps inventory

Enabled third-party / notable apps as of 10 June 2026 (full `occ app:list` output is the authoritative record — re-run after changes):

| App | Version | Purpose |
|---|---|---|
| `eurooffice` | 11.0.0 | Euro-Office DocumentServer connector ("Nextcloud Office" in the store) |
| `office` | 1.0.0 | Office-suite selector (distinct from the connector — see eurooffice runbook Quirk 5) |
| `twofactor_totp` / `twofactor_webauthn` / `twofactor_backupcodes` / `twofactor_nextcloud_notification` | 16.0.0 / 2.7.0 / 1.23.0 / 8.0.0 | 2FA methods |
| `suspicious_login` | 12.0.0-dev.0 | ML-based login anomaly detection |
| `bruteforcesettings` | 7.0.0 | Brute-force protection tuning |
| `admin_audit` | 1.24.0 | Audit logging |
| `files_external` | 1.26.0 | External storage (currently unused — candidate for NAS-backed data growth) |
| `integration_onedrive` | 3.5.1 | OneDrive import |
| `guests` | 4.7.5 | Restricted guest accounts |
| `circles` / `files_lock` / `files_reminders` / `files_downloadlimit` | 34.0.0 / 34.0.0 / 1.7.0 / 5.2.0 | Collaboration extras |
| `photos` | 7.0.0 | Photo management |
| `text` | 8.0.0 | Markdown/text editor (non-Office formats) |
| Core: `activity`, `dav`, `files_*`, `theming`, `notifications`, `serverinfo`, `updatenotification`, etc. | — | Stock Hub 26 set |

Family-sharing plan (from the eurooffice runbook §10): `family` group + **Group folders** app (not yet installed — backlog) for a shared "Family Documents" folder.

---

## 7. Operational runbook

### 7.1 occ usage

Always via the wrapper (`/usr/local/bin/occ`, Appendix B) or `su -s /bin/sh www-data -c "php /var/www/nextcloud/occ …"`. **`sudo` does not exist on this box.**

### 7.2 Updating Nextcloud (minor/point releases)

```bash
qm snapshot 111 pre-nc-update-$(date +%Y%m%d)        # on pve
occ maintenance:mode --on
# Web updater (Settings → Administration → Overview) or:
su -s /bin/sh www-data -c "php /var/www/nextcloud/updater/updater.phar"
occ upgrade
occ maintenance:mode --off
occ db:add-missing-indices && occ db:add-missing-columns
# verify: occ status ; Settings → Overview warnings ; open a document (exercises eurooffice)
```

Major version jumps: read the NC release notes first; the `eurooffice` connector supports NC 33–35 — check connector compatibility **before** any jump beyond 35.

### 7.3 OS patching

Covered by the monthly **BassT23 Proxmox Updater** run (cluster doc §7), which snapshots first (hence `Update_*` snapshot names). Manual: `apt update && apt upgrade` — TurnKey tracks Debian bookworm security.

### 7.4 Backups

- **Current:** manual vzdump to `node1_backup` (directory storage on `tank`, pve).
- **Target state (set up June 2026):** scheduled job, weekly, **snapshot mode** (guest agent gives fsfreeze consistency — MariaDB/Redis state is crash-consistent and Nextcloud tolerates this well), retention instead of keep-all:

```
# Datacenter → Backup → Add  (or /etc/pve/jobs.cfg)
vzdump: backup-weekly-cloud
        schedule sun 02:30
        storage node1_backup
        vmid 111,141
        mode snapshot
        compress zstd
        prune-backups keep-last=4,keep-weekly=4,keep-monthly=3
        notes-template {{guestname}} weekly
        enabled 1
```

- 02:30 sits after Nextcloud's 01:00 maintenance window. For belt-and-braces application-level consistency, an optional pre/post hook can toggle `occ maintenance:mode` — usually unnecessary with the agent.
- **Restore drill (quarterly, per cluster backup runbook):** restore to a spare VMID (`qmrestore … --unique`), boot isolated, verify login + file access, destroy.
- Off-site: this VM is the highest-value guest — it should be first in line when the cluster off-site gap (backup runbook §gap analysis) is closed.

### 7.5 Scheduled checks

| Cadence | Check |
|---|---|
| Weekly | Settings → Administration → Overview — zero warnings |
| Weekly | `df -h /` — root FS trend (§2 capacity note) |
| Monthly | `certbot renew --dry-run` + dehydrated cert expiry (§4.3) |
| Monthly | `occ app:update --all` (review, then apply) |
| Monthly | Verify cron heartbeat: Settings → Basic settings → background jobs "Last job execution" < 10 min |
| Quarterly | scan.nextcloud.com; external port scan (only 80/443 on <WAN_IP>); restore drill |

### 7.6 Logs

| What | Where |
|---|---|
| Nextcloud app log | `/var/www/nextcloud-data/nextcloud.log` (loglevel 3, rotates at 100 MB) — or `occ log:tail` |
| Apache | `/var/log/apache2/{access,error}.log` |
| Audit | `admin_audit` app → log |
| ACME | `/var/log/letsencrypt/`; dehydrated via daily cron output |

---

## 8. Security posture

1. **Exposure:** only 80/443 forwarded at the router, to this VM alone; 80 is redirect-only. Webmin :12321 LAN-only (verify it is excluded from any forward); shellinabox :12320 disabled.
2. **TLS:** HSTS with includeSubDomains on both vhosts; Let's Encrypt on both names; no TLS 1.0/1.1 (TurnKey default ssl.conf).
3. **Auth:** 2FA apps installed (TOTP, WebAuthn, backup codes) — **enforce for admin at minimum** (Settings → Security); `suspicious_login` and brute-force protection active; `password_policy` enabled.
4. **App-layer:** `admin_audit` logging; server-side encryption **not** enabled (deliberate — key management risk outweighs benefit for a LAN-hosted box with disk-level trust).
5. **Secrets hygiene:** config.php is 0640 www-data; SMTP credentials rotated 10 Jun 2026 after exposure during the build session; JWT + secure_link + Cloudflare token in password manager only.
6. **External validation:** scan.nextcloud.com run 10 Jun 2026 (record rating in Appendix D on receipt); SSL Labs A-grade target.
7. **Open question (backlog):** identify `10.0.0.150` in trusted_domains — remove if stale.

---

## 9. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| "Document security token…" / editor errors | CT 141 side — JWT (`default.json` quirk) | eurooffice runbook §9 + `jwttest.py` |
| 502 opening documents | docservice booting/down on CT 141 | `pct enter 141` → healthcheck, logs |
| App store install fails (cURL timeout to GitHub) | IPv6 with no route (Quirk 2) | `wget -4` manual install; gai.conf already set |
| Background jobs stale | www-data crontab missing / PHP error | `crontab -u www-data -l`; run cron.php manually and read output |
| Web UI slow / locks | Redis socket perms or service down | `systemctl status redis`; `redis-cli -s /var/run/redis/redis.sock ping` |
| "Trusted domain" error on access | accessing via an IP/hostname not in trusted_domains | `occ config:system:get trusted_domains`; add if legitimate |
| Cert expired on nextcloud vhost | dehydrated cron failed silently | run `/usr/lib/confconsole/plugins.d/Lets_Encrypt/dehydrated-wrapper` manually; check port-80 vhost intact |
| occ: "sudo: not found" | TurnKey has no sudo | wrapper / `su -s /bin/sh www-data -c` |

---

## 10. Known quirks

1. **No `sudo`** — TurnKey convention; all occ docs/snippets online assume `sudo -u www-data`. Use the wrapper (Appendix B).
2. **IPv6 enabled, no route** — broke ACME and app-store fetches. Fixed system-wide with `precedence ::ffff:0:0/96 100` in `/etc/gai.conf`; PHP paths can still misbehave, so manual app installs use `wget -4`. Full IPv6 disable via sysctl remains an option.
3. **Global certificate wiring** — the nextcloud vhost has no cert directives; it inherits `/etc/ssl/private/cert.pem` from `mods-available/ssl.conf`. Any new vhost needing a *different* cert must carry per-vhost directives (as `office.conf` does). Easy to miss when adding sites.
4. **Appliance lineage** — `community-script` tag on the VM but TurnKey inside; documentation for *both* applies (TurnKey for confconsole/dehydrated/Webmin, community-scripts only for provisioning).
5. **`efidisk0` present but BIOS is SeaBIOS** — harmless leftover from the provisioning script; the 4 MB EFI disk is unused. Don't "fix" it; changing BIOS type on an installed guest breaks boot.
6. **`scsi1` 828 MB disk** — provisioning artefact (TurnKey install medium); currently attached and in the boot order after scsi0. Candidate for detachment after verifying nothing mounts it (`lsblk` in guest) — backlog.

---

## 11. Explain like I'm 5

This VM is the family's toy box *and* the front door of the house. Every visitor from the internet knocks on one door (port 443); the doorman (Apache) checks the name on the envelope and either opens the toy box (Nextcloud) or walks them down the hall to the craft table room (the Euro-Office LXC). The doorman renews his two door-badges (certificates) by himself — one by hanging a note on the door (HTTP-01), one by whispering to the phone book company (DNS-01). A little robot (cron) tidies the toy box every five minutes, and every night at half two on Sundays a photographer (vzdump) takes a picture of the whole room so we can rebuild it exactly if anything breaks.

---

## 12. References

1. TurnKey Nextcloud appliance: https://www.turnkeylinux.org/nextcloud
2. community-scripts ProxmoxVE (provisioning helper): https://community-scripts.github.io/ProxmoxVE/
3. Nextcloud 34 admin manual: https://docs.nextcloud.com/server/34/admin_manual/
4. Nextcloud occ reference: https://docs.nextcloud.com/server/34/admin_manual/occ_command.html
5. Nextcloud server tuning (Redis, cron, opcache): https://docs.nextcloud.com/server/34/admin_manual/installation/server_tuning.html
6. certbot dns-cloudflare: https://certbot-dns-cloudflare.readthedocs.io/
7. Proxmox vzdump / backup jobs: https://pve.proxmox.com/wiki/Backup_and_Restore
8. Companion documents: `proxmox-infrastructure-PUBLIC.md`, `eurooffice-nextcloud-PUBLIC.md`

---

## 13. Appendices

### Appendix A — `/etc/apache2/sites-available/nextcloud.conf` (as deployed, 10 June 2026)

```apache
ServerName localhost

<VirtualHost *:80>
    UseCanonicalName Off
    ServerAdmin webmaster@localhost
    RewriteEngine On
    RewriteRule ^(.*)$ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]
</VirtualHost>

<VirtualHost *:443>
    ServerName cloud.example.com
    SSLEngine on
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/nextcloud/
    <IfModule mod_headers.c>
        Header always set Strict-Transport-Security "max-age=15552000; includeSubDomains"
    </IfModule>
</VirtualHost>

<Directory /var/www/nextcloud/>
    Options +FollowSymLinks
    AllowOverride All
    Require all granted
</Directory>
```

(The port-80 vhost is load-bearing for dehydrated HTTP-01 renewals. `office.conf` is documented in the eurooffice runbook.)

### Appendix B — `/usr/local/bin/occ` wrapper

```bash
#!/bin/bash
cd /var/www/nextcloud
exec su -s /bin/sh www-data -c "/usr/bin/php occ $(printf '%q ' "$@")"
```

### Appendix C — Quick reference

```bash
occ status                                   # version / maintenance state
occ maintenance:mode --on|--off
occ config:system:get trusted_domains
occ app:list --enabled
occ log:tail
occ user:list ; occ group:list
occ files:scan --all                         # after any out-of-band file changes
occ db:add-missing-indices
qm snapshot 111 <name>      # on pve, before any risky change
qm agent 111 ping           # guest agent health
```

### Appendix D — Change log

| Date | Version | Change |
|---|---|---|
| 10 Jun 2026 | 1.0 | Initial runbook. Captured post-EuroOffice-integration state: NC 34.0.0 on TurnKey 18.1; office.conf reverse-proxy vhost + DNS-01 cert added; eurooffice 11.0.0 connector configured; config.php hygiene (cli.url, phone region, log rotation); SMTP credentials rotated; gai.conf IPv4 preference applied. scan.nextcloud.com run — rating to be recorded. |

### Appendix E — Outstanding items / improvement backlog

| Priority | Item | Notes |
|---|---|---|
| High | **DHCP reservation for .40** | Load-bearing IP (public vhosts, CT 141 hosts pin) — pin in the router |
| High | **Activate the scheduled vzdump job** | §7.4 — replaces manual backups; includes CT 141 |
| High | **Enforce 2FA for admin** | Apps installed; enforcement not yet confirmed |
| Medium | **Identify/remove `10.0.0.150`** from trusted_domains | Unknown entry |
| Medium | **Install Group folders + create `family` group** | Family sharing plan (eurooffice runbook §10) |
| Medium | **Data-growth plan** | Root FS at 38%; pick: grow scsi0 / dedicated data disk / files_external → NAS |
| Low | **Detach `scsi1` (828 MB)** | After `lsblk` confirms unused |
| Low | **Record scan.nextcloud.com rating** | Appendix D |

### Appendix F — Glossary

| Term | Meaning |
|---|---|
| **TurnKey appliance** | Pre-built Debian-based VM image with the application stack and a config console (confconsole) baked in |
| **confconsole / dehydrated** | TurnKey's configuration console and its ACME (Let's Encrypt) client |
| **occ** | Nextcloud's command-line administration tool (ownCloud Console, historically) |
| **cron mode** | Background-job strategy where system cron invokes `cron.php` — preferred over AJAX/webcron |
| **trusted_domains** | Hostnames/IPs Nextcloud will accept requests for — anti-host-header-injection |
| **HTTP-01 / DNS-01** | ACME challenge types: file on port 80 vs DNS TXT record |
| **fsfreeze** | Guest-agent filesystem quiesce during snapshot backups — gives crash-consistent images |
| **vzdump snapshot mode** | Backup of a running guest from a point-in-time snapshot — no downtime |

---

*End of document (v1.0, 10 June 2026). Store this version in your password manager / encrypted vault. Update after every change to the VM and keep aligned with the sanitised public counterpart.*
