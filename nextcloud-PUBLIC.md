# Nextcloud on Proxmox LXC — HA Build & Operations Runbook

> **🌐 PUBLIC / SANITISED VERSION**
> Hostnames, IPs, MACs and identifiers replaced with placeholders (`example.com`, RFC 1918 / documentation values). No secrets appear in this document. Substitute your own values throughout.

| | |
|---|---|
| **Maintainer** | Richard Carragher (RC COMMS) |
| **Last updated** | 9 October 2026 |
| **Document version** | 2.0 (supersedes the v1.0 TurnKey VM runbook) |
| **Platform** | Proxmox VE 9.2 three-node cluster, ZFS on every node |
| **Guests** | 3 × unprivileged Debian 13 (trixie) LXC + 1 stateless tunnel connector |
| **Nextcloud** | Hub 26 Spring (35.0.1) |
| **Stack** | nginx · PHP-FPM 8.4 · MariaDB 11.8 · Redis (unix socket) · APCu |
| **Exposure** | Cloudflare Tunnel only — no inbound ports |
| **Classification** | Public |

---

## Related documents

- **`proxmox-infrastructure-PUBLIC.md`** — cluster master documentation: nodes, storage, networking.
- **`eurooffice-nextcloud-PUBLIC.md`** — Euro-Office DocumentServer integration (JWT, connector). *Note: its reverse-proxy section describes the retired VM; the document server is now published directly through the tunnel (§3.6 below).*
- **`nextcloud-tunnel-hardening-PUBLIC.md`** — Cloudflare Tunnel and edge hardening background.

## Table of contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Build reference](#3-build-reference)
4. [Migrating from the TurnKey VM](#4-migrating-from-the-turnkey-vm)
5. [Data protection](#5-data-protection)
6. [Routine operations](#6-routine-operations)
7. [Failover and failback](#7-failover-and-failback)
8. [Verification checklist](#8-verification-checklist)
9. [Known issues and gotchas](#9-known-issues-and-gotchas)
10. [Explain like I'm 5](#10-explain-like-im-5)
11. [References](#11-references)
12. [Change log](#12-change-log)

---

## 1. Overview

The family cloud (files, calendars/contacts, photos, collaborative office editing) was rebuilt in October 2026 from a single TurnKey appliance VM into **one-service-per-container LXCs** on a three-node Proxmox cluster:

- **App** and **database** containers, replicated every five minutes to a second node and managed by Proxmox HA.
- **Two Cloudflare Tunnel connectors** on different nodes, so the public endpoint survives the loss of a node.
- **No inbound ports** on the router. LAN clients use the same Cloudflare path as everyone else (no split DNS).
- Existing desktop, mobile and CalDAV/CardDAV clients carried on **without reconfiguration** — same hostname, same instance ID and secrets, database migrated intact.

### Why move off the appliance VM?

| Problem with the VM | Fix in the LXC design |
|---|---|
| Data on the OS root disk, which kept filling | Dedicated ZFS data volume, grows online |
| Appliance OS (TurnKey) with no supported in-place Debian upgrade | Stock Debian 13: PHP 8.4 and MariaDB 11.8 native, no third-party repos |
| VM disk on non-replicable local LVM | ZFS volumes, replicated with `pvesr` |
| Single node, single point of failure | HA restart on a second node + a second tunnel connector |
| Router port-forward 80/443 to the VM | Cloudflare Tunnel only |

---

## 2. Architecture

| Component | Guest | Placeholder IP | Node |
|---|---|---|---|
| App — nginx, PHP-FPM, Redis, APCu, cron | CT `nc-app` | 10.0.0.48 | node B (HA) |
| Database — MariaDB | CT `nc-db` | 10.0.0.49 | node B (HA) |
| Tunnel connector #1 | CT `cf-connector-1` | 10.0.0.12 | node B (HA) |
| Tunnel connector #2 | CT `cf-connector-2` | 10.0.0.18 | node C (pinned, stateless) |
| Document server (Euro-Office) | CT `docserver` | 10.0.0.41 | node B |
| File backend (Samba NAS) | CT `nas` | 10.0.0.32 | node A |

```
Client ─HTTPS─► Cloudflare edge (TLS, WAF) ─tunnel─► cf-connector-1 / cf-connector-2
       ─HTTP:80─► nc-app nginx ─► php-fpm ─► nc-db :3306
                                     ├─► Redis (unix socket)
                                     ├─► SMB shares on nas (External Storage)
                                     └─► docserver (internal URL)
docserver ─HTTP:80─► nc-app (StorageUrl: file fetch / save callbacks)
```

Public hostnames: `cloud.example.com` (Nextcloud) and `office.example.com` (document server).

---

## 3. Build reference

Common to every guest:
- Template: `debian-13-standard`.
- Settings: `unprivileged 1`, `nesting=1`, storage on a ZFS pool **with the same storage ID on every node** (needed for replication), static IP, and Pi-hole DNS.
- Management: SSH disabled (`systemctl disable --now ssh.socket ssh.service`); manage with `pct enter` / `pct exec`.

### 3.1 Containers

| CT | Cores / RAM / swap | rootfs | Extra volume |
|---|---|---|---|
| `nc-app` | 4 / 4 GiB / 1 GiB | 16 GB | `mp0` 150 GB → `/var/www/nextcloud-data`, `backup=0`, ZFS `recordsize=1M` |
| `nc-db` | 2 / 2 GiB / 512 MiB | 12 GB | `mp0` 20 GB → `/var/lib/mysql`, ZFS `recordsize=16K` |
| connectors | 1 / 256 MiB / 256 MiB | 4 GB | — |

```bash
pct create <ID> local:vztmpl/debian-13-standard_13.6-1_amd64.tar.zst \
  --hostname nc-db --unprivileged 1 --ostype debian \
  --cores 2 --memory 2048 --swap 512 \
  --rootfs <zfs-storage>:12 \
  --mp0 <zfs-storage>:20,mp=/var/lib/mysql \
  --net0 name=eth0,bridge=vmbr0,ip=10.0.0.49/24,gw=10.0.0.1 \
  --features nesting=1 --onboot 1
zfs set recordsize=16K <pool>/subvol-<ID>-disk-1     # before any data is written
```

Use **Proxmox-managed mount points**, not host bind mounts. Managed volumes replicate, migrate and snapshot with the container, and unprivileged UID mapping is handled for you.

### 3.2 Database container

`/etc/mysql/mariadb.conf.d/60-nextcloud.cnf`:

```ini
[mysqld]
bind_address            = 10.0.0.49
skip_name_resolve       = ON
transaction_isolation   = READ-COMMITTED
character_set_server    = utf8mb4
collation_server        = utf8mb4_general_ci
innodb_buffer_pool_size = 768M
max_connections         = 100
tmp_table_size          = 64M
max_heap_table_size     = 64M
innodb_log_file_size    = 256M
innodb_doublewrite      = 0      # only safe because the data volume uses recordsize=16K on ZFS
innodb_flush_neighbors  = 0
slow_query_log          = ON
slow_query_log_file     = /var/log/mysql/mariadb-slow.log
long_query_time         = 2
```

- **Database and user:** database `nextcloud` (utf8mb4). User `nextcloud@10.0.0.48` only, with privileges on `nextcloud.*`.
- **Password handling:** generate the password inside the container straight into a root-only `[client]` file. That way it never appears on screen or in shell history.
- **Proxmox firewall** (`/etc/pve/firewall/<ID>.fw`): allow 3306/tcp from the app container only, plus LAN ICMP. Set `policy_in: DROP` and `firewall=1` on the NIC.

### 3.3 App container

Packages, all from the trixie repos:

```bash
apt install -y nginx redis-server php-fpm php-cli php-mysql php-gd php-curl php-mbstring \
  php-intl php-xml php-zip php-bcmath php-gmp php-bz2 php-apcu php-redis php-imagick \
  libmagickcore-7.q16-10-extra php-smbclient ffmpeg mariadb-client unzip
```

> If any External Storage mounts are SMB, **`php-smbclient` is mandatory**. Appliances often bundle it silently. Without it, every SMB mount shows 0 KB.

**Redis on a socket only.** Set `port 0` in `/etc/redis/redis.conf` and append:
```
unixsocket /run/redis/redis-server.sock
unixsocketperm 770
```
Then run `usermod -aG redis www-data`.

**PHP**, in `/etc/php/8.4/fpm/conf.d/90-nextcloud.ini` (copy the same file into `cli/conf.d/`):

```ini
memory_limit = 1G
upload_max_filesize = 16G
post_max_size = 16G
max_execution_time = 3600
max_input_time = 3600
output_buffering = 0
date.timezone = Europe/London
opcache.enable = 1
opcache.memory_consumption = 256
opcache.interned_strings_buffer = 32
opcache.max_accelerated_files = 20000
opcache.revalidate_freq = 60
opcache.save_comments = 1
apc.enable_cli = 1
```

**Dedicated FPM pool**, in `/etc/php/8.4/fpm/pool.d/nextcloud.conf`. Rename the stock `www.conf` so it no longer loads.

```ini
[nextcloud]
user = www-data
group = www-data
listen = /run/php/php8.4-fpm-nextcloud.sock
listen.owner = www-data
listen.group = www-data
listen.mode = 0660
pm = dynamic
pm.max_children = 24
pm.start_servers = 6
pm.min_spare_servers = 6
pm.max_spare_servers = 12
pm.max_requests = 500
request_terminate_timeout = 300
env[HOSTNAME] = $HOSTNAME
env[PATH] = /usr/local/bin:/usr/bin:/bin
env[TMP] = /tmp
env[TMPDIR] = /tmp
env[TEMP] = /tmp
```

**nginx.** Use the [Nextcloud Admin Manual reference vhost](https://docs.nextcloud.com/server/latest/admin_manual/installation/nginx.html) with these site-specific changes:

```nginx
upstream php-handler { server unix:/run/php/php8.4-fpm-nextcloud.sock; }
server {
    listen 10.0.0.48:80;                 # specific address, never 0.0.0.0
    server_name cloud.example.com;
    root /var/www/nextcloud;
    set_real_ip_from 10.0.0.12;          # every tunnel connector
    set_real_ip_from 10.0.0.18;
    real_ip_header   CF-Connecting-IP;
    client_max_body_size 16G;
    # … reference config …
    # inside the PHP location:  fastcgi_param HTTPS on;
}
```

Because nginx binds to a specific IP, make it wait for the network. Otherwise a reboot can leave it failed with a 502. Create `/etc/systemd/system/nginx.service.d/10-network-online.conf`:

```ini
[Unit]
Wants=network-online.target
After=network-online.target
```

**Background jobs.** Use a systemd timer, as the Admin Manual's *Background jobs* page describes:
- `nextcloudcron.service`: `User=www-data`, `ExecCondition=php -f /var/www/nextcloud/occ status -e`, `ExecStart=/usr/bin/php -f /var/www/nextcloud/cron.php`.
- `nextcloudcron.timer`: `OnUnitActiveSec=5min`.

Then run `occ background:cron`.

**`occ` wrapper.** Create `/usr/local/bin/occ` containing:
```sh
#!/bin/sh
exec runuser -u www-data -- php /var/www/nextcloud/occ "$@"
```
`pct enter` gives a reduced `PATH`, so add `/usr/local/bin` to root's `.bashrc`.

**Proxmox firewall.** Allow 80/tcp from the tunnel connectors and the document server only, plus LAN ICMP, with `policy_in: DROP`.

### 3.4 `config.php` essentials

| Key | Value |
|---|---|
| `datadirectory` | Keep the **same path** as the old server (§4) |
| `instanceid`, `passwordsalt`, `secret` | Carried over — **never regenerate**. They encrypt stored external-storage credentials, sessions and app passwords |
| `dbhost` | `10.0.0.49` |
| `memcache.local` | `\OC\Memcache\APCu` |
| `memcache.distributed`, `memcache.locking` | `\OC\Memcache\Redis` |
| `redis` | `host` = socket path, `port` 0 |
| `trusted_domains` | `localhost`, app IP, `cloud.example.com` |
| `trusted_proxies` | every connector IP |
| `forwarded_for_headers` | `HTTP_CF_CONNECTING_IP` |
| `overwriteprotocol` | `https` (the origin is plain HTTP behind the tunnel) |
| `allow_local_remote_servers` | `true` if the document server is on the LAN |

Document-server connector settings:
- the **public** document-server URL
- an **internal** URL (`http://10.0.0.41/`)
- a **StorageUrl** of `http://10.0.0.48/`, so the document server reaches Nextcloud directly on the LAN
- **thumbnails off** (§9)

### 3.5 Tunnel connectors

1. Install `cloudflared` from Cloudflare's apt repository.
2. Create the service user **before** the token directory:

```bash
useradd --system --no-create-home --shell /usr/sbin/nologin cloudflared
install -d -m 750 -o root -g cloudflared /etc/cloudflared
# token file: /etc/cloudflared/<tunnel>.token, mode 640 root:cloudflared — piped in, never echoed
```

3. Write the systemd unit, `cloudflared-<tunnel>.service`:

```ini
[Service]
Type=notify
User=cloudflared
Group=cloudflared
ExecStart=/usr/bin/cloudflared --no-autoupdate tunnel run --token-file /etc/cloudflared/<tunnel>.token
Restart=on-failure
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
```

To verify, run `journalctl -u cloudflared-<tunnel> | grep "Registered tunnel connection"`; you should see 4 lines. Recent versions print a long pre-check banner, so grep rather than `tail`.

Run **two connectors with the same token on different nodes**. Cloudflare balances between them and keeps serving through a node failure.

**Every connector IP must be trusted in three places before the connector starts:**
- the app container's firewall
- nginx `set_real_ip_from`
- Nextcloud `trusted_proxies`

### 3.6 Cloudflare

- **Routes:** add published-application routes pointing at **LAN IPs**: `cloud.example.com` → `http://10.0.0.48:80` and `office.example.com` → `http://10.0.0.41:80`. Never use `localhost`: each connector resolves it to its own container.
- **WAF custom rules (free plan):**
  - Block `*.php` paths except `/index.php`, `/remote.php`, `/public.php`, `/status.php`, `/ocs/v1.php` and `/ocs/v2.php`.
  - Block common scanner paths (`/wp-`, `/.env`, `/.git`, `/cgi-bin`, `/phpmyadmin`).
  - Managed challenge for browser logins (`/login`, `/index.php/login`, `/login/v2/flow*`) from outside your home countries.
- **Rate limit:** login form POSTs, 5 per 10 seconds per IP.
- **Never challenge client API paths** (`/remote.php`, `/ocs`, `/status.php`, `/.well-known`, `/login/v2/poll`). Sync clients cannot solve challenges.
- **Keep Bot Fight Mode off.** On the free plan it is zone-wide and breaks Nextcloud, DAV, password-manager and photo-app clients.
- The web updater is blocked by the `.php` rule. That is deliberate: upgrade with `occ` only.

---

## 4. Migrating from the TurnKey VM

Rough timings: about 20 minutes of downtime for about 42 GB of data, plus a 400k-row file cache.

1. **Build** the new containers. Install the **exact same Nextcloud version** as the old server, after checking the SHA-256 and GPG signature. Do not run the installer.
2. **Copy third-party apps** byte-for-byte from the old `apps/` directory, using a `tar` pipe.
3. **Mount the data volume at the old data path.** Nextcloud records that path in the database (`oc_storages`, `local::/path/`). Changing it orphans file IDs, shares and app data.
4. **Pre-seed the data** while the old server is live: `rsync -aH --numeric-ids --chown=100033:100033 old:/data/ /<pool>/subvol-<ID>-disk-1/`.
5. **Copy `config.php`** and change only the database, cache, proxy and trusted-domain keys (§3.4).
6. **Cut over:**
   1. Turn on maintenance mode on the old server.
   2. Pipe `mariadb-dump --single-transaction --routines --triggers` into the new database, and compare table and `oc_filecache` row counts.
   3. Run a final `rsync --delete`.
   4. Run `occ upgrade` (it should report nothing to do) and `occ maintenance:data-fingerprint`.
7. **Repoint** the document server's StorageUrl, and remove any hosts-file pins.
8. **Switch** the tunnel routes to the new LAN IPs.
9. **Check local DNS** for overrides of the public hostname: Pi-hole Local DNS records **and** `misc.dnsmasq_lines` `local=/host/` entries.
10. **Retire** the old VM: stop its connector, set `onboot 0`, shut it down, and keep it with a snapshot for two weeks.

---

## 5. Data protection

| Layer | What | Schedule |
|---|---|---|
| Storage replication (`pvesr`) | app + DB containers to a second node over the storage network | every 5 min (connector every 15) |
| Proxmox HA | app, DB, connector #1; node-affinity across two nodes + positive resource-affinity keeping app and DB together | continuous |
| vzdump | container root filesystems and the DB volume (data volume excluded) | weekly |
| Logical DB dump | `mariadb-dump --single-transaction` → zstd, 7 days local | nightly, before restic |
| restic → off-site storage box | data volume (previews excluded) + dump directory, own snapshot group/tag | nightly |

**Restore the database only:**
```bash
occ maintenance:mode --on
zstd -dc nextcloud-<TS>.sql.zst | pct exec <db-ID> -- mariadb nextcloud
occ maintenance:mode --off && occ maintenance:data-fingerprint
```

**Restore files:**
```bash
restic restore <snapshot> --target /restore --include <data-path>/<user>/files/<path>
# copy back as 100033:100033, then:
occ files:scan --path="<user>/files/<path>"
```

---

## 6. Routine operations

| Task | How |
|---|---|
| Upgrade Nextcloud | Snapshot app + DB → maintenance on → CLI updater (`updater.phar` as www-data) → `occ upgrade` → `occ db:add-missing-indices` → maintenance off |
| Upgrade apps | `occ app:update --all` |
| PHP updates | `systemctl reload php8.4-fpm` afterwards |
| Pool health | `tail /var/log/php8.4-fpm.log`. Any `max_children` warning means saturation (§9) |
| Grow storage | `pct resize <ID> mp0 +50G` (online) |
| Replication | `pvesr status` |

---

## 7. Failover and failback

- **Node failure:** HA fences the failed node and starts the app, DB and connector #1 on the second node from the last replica. You lose at most 5 minutes of changes, and there are about 2–3 minutes of 502s. Connector #2 keeps the tunnel up throughout.
- **After a failover:**
  1. Run `occ status` and `occ files:scan --all` (the DB and files can be up to one replication interval apart), then `occ maintenance:repair`.
  2. Expect node-scoped backup jobs to skip the moved guests. Run a manual vzdump if the outage is long.
- **Failback** (in a planned window): `ha-manager crm-command relocate` the DB, then the app (the together rule keeps them paired), then the connector.

---

## 8. Verification checklist

- [ ] `https://cloud.example.com/status.php` returns `"maintenance":false`
- [ ] Connectors and the document server get `200` from the app container. Anything else is blocked by the firewall.
- [ ] Web login works; images show previews; SMB folders show real sizes
- [ ] Desktop client syncs and uploads; the mobile app refreshes; CalDAV/CardDAV refresh
- [ ] An office document opens, edits and saves, and a new version appears
- [ ] A public share link opens in a private window
- [ ] *Administration → Overview* shows no warnings
- [ ] The tunnel shows 2 healthy replicas

---

## 9. Known issues and gotchas

| Symptom | Cause | Fix |
|---|---|---|
| 504 Gateway Timeout; FPM log shows `reached pm.max_children` | Document-server **thumbnails**: each preview holds a PHP worker while the document server downloads the file via StorageUrl, which needs *another* worker. A folder full of office files exhausts the pool. | Turn off the connector's preview option; size the pool at 24 with `request_terminate_timeout`; in an emergency, restart php-fpm |
| SMB folders show 0 KB | `php-smbclient` missing | Install it and restart php-fpm |
| "Resharing is not allowed" | `enable_sharing` disabled on that External Storage mount | Expected by policy, or enable it per mount |
| 502 after the app container reboots | nginx started before its IP existed | `network-online` override |
| Intermittent 502s with two connectors | Route points at `localhost` | Use LAN IPs |
| LAN gets NXDOMAIN while external works | Leftover Pi-hole `local=/host/` dnsmasq line | Remove it from `misc.dnsmasq_lines` on every Pi-hole |
| `cloudflared` "permission denied" on the token | Directory created under `umask 077` | `750 root:cloudflared` on the directory |
| Positive resource-affinity rule rejected (PVE 9) | Not allowed together with a node-affinity rule that uses different priorities | Give the nodes equal weight |
| `preview:cleanup` removed the whole preview folder | Behaviour in Nextcloud 35 | Harmless. Previews regenerate; expect a CPU burst |

---

## 10. Explain like I'm 5

The old setup was one big toy box that did everything and sat by the front door. The new setup is three small boxes:
- one that shows you your toys (the app)
- one that remembers where every toy is (the database)
- one that answers the door (the tunnel)

Every five minutes, a helper copies the first two boxes into a second room. If the first room's lights go out, the boxes are carried next door and opened again. There are two doormen in two different rooms, so somebody always answers. Nobody can walk straight in from the street: every visitor first checks in at the Cloudflare reception desk.

---

## 11. References

1. Nextcloud Admin Manual — [NGINX configuration](https://docs.nextcloud.com/server/latest/admin_manual/installation/nginx.html), [Server tuning](https://docs.nextcloud.com/server/latest/admin_manual/installation/server_tuning.html), [Background jobs](https://docs.nextcloud.com/server/latest/admin_manual/configuration_server/background_jobs_configuration.html), [Reverse proxy](https://docs.nextcloud.com/server/latest/admin_manual/configuration_server/reverse_proxy_configuration.html), [Migrating to a different server](https://docs.nextcloud.com/server/latest/admin_manual/maintenance/migrating.html)
2. Proxmox VE Admin Guide — [Container storage](https://pve.proxmox.com/wiki/Linux_Container), [Storage Replication](https://pve.proxmox.com/wiki/Storage_Replication), [High Availability](https://pve.proxmox.com/wiki/High_Availability), [Firewall](https://pve.proxmox.com/wiki/Firewall)
3. OpenZFS — [Workload tuning: MySQL/InnoDB](https://openzfs.github.io/openzfs-docs/Performance%20and%20Tuning/Workload%20Tuning.html)
4. Cloudflare — [Tunnel replicas](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/deploy-tunnels/tunnel-availability/), [WAF custom rules](https://developers.cloudflare.com/waf/custom-rules/), [Rate limiting rules](https://developers.cloudflare.com/waf/rate-limiting-rules/)

---

## 12. Change log

| Date | Version | Change |
|---|---|---|
| 10 Jun 2026 | 1.0 | TurnKey Nextcloud VM runbook (Apache, router port-forward, Let's Encrypt on the VM) |
| 9 Oct 2026 | 2.0 | Rebuilt as HA LXCs on Debian 13 / PHP 8.4 / MariaDB 11.8; Cloudflare Tunnel with two connectors; per-guest firewalls; replication + HA; DB dump + restic off-site; WAF rules; PHP-FPM saturation fix; VM retired |

*End of document (v2.0, 9 October 2026). Keep aligned with the INTERNAL counterpart.*
