# Nextcloud Public Ingress Hardening — Incident & Remediation Runbook

> **🌐 PUBLIC / SANITISED VERSION**
> Hostnames, IP addresses, tunnel identifiers, account names and source addresses replaced with placeholders (`example.com`, RFC 1918 / documentation values). No secrets appear in this document. Substitute your own values throughout.

| | |
|---|---|
| **Maintainer** | Richard Carragher (RC COMMS) |
| **Last updated** | 25 July 2026 |
| **Document version** | 1.0 |
| **Guest** | VM 111 `nextcloud-vm` on node `pve` (Proxmox VE 9) |
| **Trigger** | Uptime monitor flapping — `timeout of 48000ms exceeded` |
| **Outcome** | Origin no longer directly reachable; WAN 80/443 forward removed; ingress via Cloudflare Tunnel |
| **Classification** | Public |

---

## Related documents

- **`nextcloud-vm-PUBLIC.md`** — appliance runbook (revised to v2.0 by this work; §2, §4.1, §4.2, §4.3, §5, §8 all superseded).
- **`proxmox-infrastructure-PUBLIC.md`** — cluster master documentation; node hardware and VM 111 spec corrected by this work.
- **`eurooffice-nextcloud-PUBLIC.md`** — DocumentServer integration; connector URLs confirmed unchanged (§5.6).

## Table of contents

1. [Executive summary](#1-executive-summary)
2. [Incident timeline and symptoms](#2-incident-timeline-and-symptoms)
3. [Root cause analysis](#3-root-cause-analysis)
4. [Remediation — origin hardening](#4-remediation--origin-hardening)
5. [Architecture change — Cloudflare Tunnel migration](#5-architecture-change--cloudflare-tunnel-migration)
6. [Edge constraints and measured limits](#6-edge-constraints-and-measured-limits)
7. [Cloudflare WAF and rate limiting](#7-cloudflare-waf-and-rate-limiting)
8. [Monitoring](#8-monitoring)
9. [Verification matrix](#9-verification-matrix)
10. [Rollback procedures](#10-rollback-procedures)
11. [Known traps and gotchas](#11-known-traps-and-gotchas)
12. [Outstanding items](#12-outstanding-items)
13. [Explain like I'm 5](#13-explain-like-im-5)
14. [References](#14-references)
15. [Appendices](#15-appendices)

---

## 1. Executive summary

An uptime monitor had been flapping the Nextcloud guest for several days with 48-second timeouts; 24-hour availability was 92.95%, 30-day 92.35%. The initial hypothesis was insufficient CPU on the host (an older AMD Opteron-class APU) and a proposed migration of the guest to a newer node.

**That hypothesis was wrong.** The cause was sustained PHP-webshell reconnaissance arriving directly at the origin, at roughly 2.5 requests per second per scanner. Every probe for a non-existent `.php` file forked a `mod_php` interpreter under `mpm_prefork`, exhausting a 16-worker pool and driving CPU to 91% of four vCPUs with `Full` CPU pressure stall at 10–20%. Memory was never a factor (61%, 2.46 of 4 GiB).

The origin was reachable because the consumer router forwarded WAN 80/443 to the guest, and the service hostnames were published in Certificate Transparency logs at issuance. Scanners resolved the names, found the forward, and probed the origin directly. The port-forward contradicted a standing architecture rule of zero inbound router port-forwards.

Sixteen changes were applied across three layers. The guest remains on the same hardware with the same 4 GiB of RAM; it is now unsaturated, and the origin is not directly reachable from the internet. The Nextcloud application log recorded **two** errors on the day of remediation — the lowest daily count in a five-week window, against 430 and 258 on the two worst preceding days.

No evidence of compromise was found: no `.php` files had been written under the webroot or data directory in the preceding month, and `occ integrity:check-core` returned clean.

---

## 2. Incident timeline and symptoms

### 2.1 Observed symptoms

| Source | Symptom |
|---|---|
| Uptime monitor | Repeated `Down`/`Up` cycles, `timeout of 48000ms exceeded`; one `EHOSTUNREACH`; one `status code 500` |
| Browser | "This site can't be reached" during saturation windows |
| Proxmox | Guest CPU 91.22% of 4 vCPUs; CPU Pressure Stall `Full` 10–20%, `Some` peaking 30%+; sustained disk writes 300–500 KB/s (own logging) |
| Guest | `/var/log/apache2/error.log` filled with `[php:error] … not found or unable to stat` |

Guest uptime at capture was consistent with a single restart the previous day; the remaining flapping was saturation, not crashes.

### 2.2 Attack traffic

Two distinct source populations:

| Source | Behaviour |
|---|---|
| Cloud-hosting ranges (a major public cloud provider) | Direct-to-origin against the WAN address; long enumeration runs — one burst covered ~120 distinct paths at ~0.35 s intervals over eight minutes |
| CDN edge addresses | Arrived via a proxied apex DNS record, with `referer: https://example.com/<path>` |
| Miscellaneous | Path traversal probe: `AH10244: invalid URI path (/cgi-bin/../../../../../../../../../../bin/sh)` |

Requested paths were classic webshell names — `shadow-bot.php`, `t00l.php`, `wp-tot.php`, `this_is_a_new_hello_world.php` — plus randomised `*Cdefault.php` strings.

The last `php:error` scan entry and the first `authz_core` denial after remediation were 25 minutes apart, which is how the fix was confirmed.

---

## 3. Root cause analysis

Four conditions had to coincide:

1. **Origin discoverable.** Let's Encrypt issuance publishes hostnames to Certificate Transparency. The service hostnames were grey-clouded (DNS-only) A records pointing at a residential WAN address, so the names resolved straight to the origin.
2. **Origin reachable.** The router forwarded WAN 80/443 to the guest.
3. **Every probe cost an interpreter.** `mpm_prefork` + `mod_php`, with `AllowOverride All` and `Require all granted` on the Nextcloud docroot. A missing `.php` was still handed to PHP, which then failed to `stat` it — roughly 300 ms of process startup versus a few milliseconds for a static deny.
4. **Worker pool small by design.** `MaxRequestWorkers 16` / `ServerLimit 16`, sized at ~70 MB per worker ≈ 1.1 GiB against 4 GiB of RAM. Correct for the memory available, but 16 concurrent PHP forks saturate four vCPUs, and everything behind them queues past the monitor's 48-second timeout.

**Contributing factor.** The unqualified apex A record (proxied) had no matching `ServerName` on any vhost and was absent from `trusted_domains`; requests for it fell through to the default `:443` vhost — which was Nextcloud — and were rejected by PHP's trusted-domain check *after* an interpreter had already been forked.

**Historical corroboration.** The application log shows `MySQL server has gone away`, `SQLSTATE[HY000] [2002] No such file or directory` (database socket absent) and `Redis server … went away` clustering on the same days as error spikes — all three datastores unavailable simultaneously, the signature of resource starvation rather than application faults.

---

## 4. Remediation — origin hardening

### 4.1 PHP front-controller whitelist

Deny at Apache's authorisation layer so a probe never reaches `mod_php`. Shipped as a drop-in conf rather than an edit to the site file, so it is independently reversible and survives future edits.

`/etc/apache2/conf-available/nc-php-whitelist.conf`:

```apache
# Only Nextcloud front controllers may be requested as .php.
# Everything else in the docroot is denied without invoking PHP.
# Reversible: a2disconf nc-php-whitelist && systemctl reload apache2
<Directory /var/www/nextcloud/>
    <FilesMatch "^(?!(index|remote|public|status|cron|v1|v2)\.php$).*\.php$">
        Require all denied
    </FilesMatch>
</Directory>
```

```bash
a2enconf nc-php-whitelist && apache2ctl configtest && systemctl reload apache2
```

`FilesMatch` tests the basename after path resolution, so PATH_INFO routes (`/remote.php/dav/…`, `/index.php/apps/…`) are unaffected. `v1`/`v2` cover `/ocs/v1.php` and `/ocs/v2.php`; `/updater/index.php` is covered by `index`.

**Verification.** A denied request logs `[authz_core:error] AH01630: client denied by server configuration` with **no** accompanying `php:error` — that absence is the proof PHP was never invoked. Status code is a weak signal here: Nextcloud's `.htaccess` sets `ErrorDocument 403 //index.php/error/403`, so its own error controller answers and the client sees **404**, not 403.

**Caveat.** `core/ajax/update.php` is denied by this rule, so the *web* updater breaks. Use the CLI `updater.phar` path, or `a2disconf nc-php-whitelist` for the duration of a major upgrade.

### 4.2 Default (catch-all) vhost

The Nextcloud site file was the first `*:443` vhost in load order and therefore the default for any unmatched `Host` header. A catch-all now takes that role.

`/etc/apache2/sites-available/000-catchall.conf`:

```apache
# Default vhost for unmatched Host headers. Sorts first in sites-enabled,
# so requests not matching a ServerName/ServerAlias land here instead of
# falling through to the Nextcloud docroot.
# Rollback: a2dissite 000-catchall && systemctl reload apache2
<VirtualHost *:80>
    ServerName catchall.invalid
    DocumentRoot /var/www/empty
    <Location />
        Require all denied
    </Location>
</VirtualHost>
<VirtualHost *:443>
    ServerName catchall.invalid
    SSLEngine on
    DocumentRoot /var/www/empty
    <Location />
        Require all denied
    </Location>
</VirtualHost>
```

Only `*:80` and `*:443` are declared, leaving other listeners untouched. The vhost inherits the global certificate from `mods-available/ssl.conf`.

**Prerequisite, applied at the same time.** The `*:80` vhost previously carried no `ServerName` (a `ServerName` at the top of the site file is server-level, not vhost-level) and matched only by being the default. Both vhosts therefore needed explicit names before the catch-all could take precedence, or the HTTPS redirect — and with it HTTP-01 renewal — would have broken:

```apache
<VirtualHost *:80>
    ServerName nextcloud.example.com
    ServerAlias 10.0.0.40
    UseCanonicalName Off
    …
<VirtualHost *:443>
    ServerName nextcloud.example.com
    ServerAlias 10.0.0.40
```

`UseCanonicalName Off` is retained deliberately: it keeps the 301 target derived from the client's `Host` rather than the new `ServerName`. The IP alias preserves the uptime-monitor probe and the DocumentServer's path back to Nextcloud.

Final map (`apache2ctl -S`):

```
*:80    default server catchall.invalid   → 000-catchall.conf:5
        namevhost nextcloud.example.com (alias 10.0.0.40)
*:443   default server catchall.invalid   → 000-catchall.conf:12
        namevhost nextcloud.example.com (alias 10.0.0.40)
        namevhost office.example.com
```

### 4.3 Guest resource bounds

```bash
qm set 111 --cpulimit 3 --cpuunits 100
```

Final state: `cores: 4`, `cpulimit: 3`, `cpuunits: 100`, `memory: 4096`.

- `cores 4` + `cpulimit 3` is preferred over `cores 3`: the guest keeps four-way parallelism for bursty work while never consuming more than three cores' worth of host CPU. Reducing `cores` would cut parallelism without adding protection.
- `cpuunits` was briefly set to 50 as a defensive measure during the incident and **restored to the default 100** afterwards. It is a *relative* scheduling weight; leaving it low would make an interactive service lose CPU to background containers under contention.
- `MaxRequestWorkers 16` was **not** raised. Sixteen concurrent PHP forks already saturate the CPU allowance; more workers would only deepen the queue.
- Memory was **not** increased. Peak observed was 61% (2.46 of 4 GiB) *under active attack*. Revisit only on evidence — sustained low available memory inside the guest, preview generation on large external mounts, growth in user count, or material Redis growth.

### 4.4 Removing a legacy database admin front-end

A web database-administration vhost (Adminer) had been enabled since the appliance was built, on a high non-standard port, reachable from the flat LAN and over the VPN overlay. Its `<Directory>` blocks carried no `Require` (served on the distribution's global grant for `/usr/share`), its hardening flags sat inside dead `<IfModule mod_php5.c>` guards under PHP 8.3, and it presented a mismatched certificate (`AH01909: server certificate does NOT include an ID which matches the server name`).

```bash
a2dissite adminer
sed -i '/^Listen 12322$/d' /etc/apache2/ports.conf   # .bak taken first
apache2ctl configtest
systemctl restart apache2
```

**Removing the `Listen` is mandatory, not tidying.** With the listener present and no vhost claiming it, Apache serves that port from the *main server* configuration — plaintext (the `SSLEngine on` lived inside the disabled vhost) against the compiled-in default DocumentRoot. That is worse than leaving the tool in place.

**`restart`, not `reload`:** Apache does not reliably release or bind listening sockets on a graceful reload. This was the only step in the session that caused a service interruption (1–2 seconds).

### 4.5 OCSP stapling

Let's Encrypt no longer publishes an OCSP URI in its certificates, so `SSLUseStapling On` logged two errors per certificate on every reload (`AH02218`, `AH02604`) attempting a lookup that cannot succeed. Set `SSLUseStapling Off`. `SSLStaplingCache` left in place — inert when stapling is off. Revocation is covered by CRLs and short certificate lifetimes.

Note that `Mutex ssl-stapling` entries still appear in `apache2ctl -S` output; mod_ssl registers those unconditionally and their presence does not mean stapling is active.

---

## 5. Architecture change — Cloudflare Tunnel migration

### 5.1 Design decisions

The previous runbook recorded grey-cloud as a deliberate choice: *"Grey-cloud deliberately (Cloudflare proxy's 100 MB upload cap breaks large syncs)."* A tunnel hostname is a proxied CNAME by definition, so migrating reinstates that cap. Three options were considered:

| | Approach | Removes forward | 100 MB cap | Cost |
|---|---|---|---|---|
| A | Tunnel both hostnames | Yes | Reinstated | Non-chunking WebDAV clients capped |
| B | Tunnel the office host only; Nextcloud via VPN overlay | Yes | Avoided | Every family device depends on the overlay; public share links break |
| **C** | **Tunnel both + local split-DNS internal fast path** | **Yes** | **Sidestepped per-path** | **One extra DNS binding to maintain** |

**Option C adopted**, after empirical validation (§6) showed the cap affects only monolithic `PUT`s. Devices need no client software and no overlay membership; public reachability is unchanged from before the migration.

An earlier plan to use `overwritecondaddr` was dropped: cloudflared preserves the `Host` header and the origin is already HTTPS, so Nextcloud infers its own URL correctly on both paths and no conditional rewrite is needed.

**Rejected: Cloudflare Origin CA certificate.** Normally the clean answer for a tunnel-fronted origin (15-year validity, no ACME). Incompatible with Option C — an Origin CA certificate is trusted only by Cloudflare's edge, so every LAN browser taking the split-DNS fast path would see a warning. Publicly-trusted certificates are therefore required, which makes working DNS-01 renewal load-bearing (§5.6).

**Rejected: Cloudflare Access in front of either hostname.** Access works for a browser-driven secrets manager with a hardware key. Nextcloud's sync and mobile clients cannot complete the Access interstitial. The gate here is Nextcloud's own 2FA plus `bruteforcesettings`, with the WAF in front.

### 5.2 The tunnel

The existing estate pattern is one tunnel per service with the connector co-located with the workload. A third tunnel follows that pattern, with `cloudflared` on the Nextcloud guest itself — loopback means no extra network hop and no new failure domain, since Nextcloud is unavailable anyway if the guest is down.

| Property | Value |
|---|---|
| Tunnel name | `cloud-tunnel` |
| Tunnel ID | `<TUNNEL_ID>` |
| Type | `cloudflared`, dashboard-managed configuration |
| Host | VM 111, the Nextcloud guest |
| Version | 2026.7.3, `linux_amd64`, APT-managed |
| Repo | `deb [signed-by=/usr/share/keyrings/cloudflare-public-v2.gpg] https://pkg.cloudflare.com/cloudflared any main` |
| Unit | `/usr/bin/cloudflared --no-autoupdate tunnel run --token-file /etc/cloudflared/token` |
| Protocol | QUIC |

Routes:

| Public hostname | Service | Origin Server Name |
|---|---|---|
| `nextcloud.example.com` | `https://127.0.0.1` | `nextcloud.example.com` |
| `office.example.com` | `https://127.0.0.1` | `office.example.com` |

**`originServerName` is mandatory.** cloudflared derives SNI from the service URL; `127.0.0.1` is an IP literal, so no SNI is sent and certificate validation against a bare address fails (the certificates carry no IP SAN). Without it the edge returns 502 and the connector logs:

```
tls: failed to verify certificate: x509: certificate is valid for
nextcloud.example.com, not localhost
```

Setting it makes cloudflared present the correct SNI *and* validate against the correct name, keeping TLS to the origin intact rather than resorting to `noTLSVerify`.

`https://127.0.0.1` rather than `https://localhost` is deliberate — this guest has IPv6 enabled with no route, and `localhost` may resolve to `::1`.

**Validation approach.** A throwaway hostname was added first (route + `ServerAlias` + `trusted_domains` entry), exercised, then removed. No production DNS was touched until it passed. A canary for the office host was deliberately *not* used: that vhost sets `ProxyPreserveHost On`, so a test hostname would be passed downstream to the DocumentServer, which generates its own URLs and validates JWT against them — the canary would have failed or passed for reasons not applicable to the real hostname.

### 5.3 DNS zone changes

| Record | Before | After |
|---|---|---|
| `example.com` (apex) | A `<WAN_IP>`, Proxied | **Deleted** — nothing served the apex; it was the referer path for CDN-sourced scanning |
| `www` | A `<WAN_IP>`, Proxied | **Deleted** |
| `nextcloud` | A `<WAN_IP>`, DNS only | **Tunnel CNAME, Proxied** |
| `office` | A `<WAN_IP>`, DNS only | **Tunnel CNAME, Proxied** |
| `pve` | A `<WAN_IP>`, DNS only | **Deleted** — management-plane hostname published in public DNS, contrary to the standing rule that the management plane stays off the internet |
| `<stale-app>` | A `<WAN_IP>`, DNS only | **Deleted** — no such guest and no vhost, so requests were landing on the default `:443` vhost |
| `photos`, `vault` | Existing tunnels | unchanged |

The A records for the two migrating hostnames had to be deleted manually before Cloudflare would create the tunnel CNAMEs.

Cosmetic leftovers from a previous registrar remain and are harmless: in-zone `NS` records for the old provider and an `_autodiscover._tcp` SRV. Delegation was confirmed as pointing at Cloudflare's nameservers.

### 5.4 Split-horizon DNS — internal fast path

Local A record on **both** local DNS resolvers:

```
nextcloud.example.com → 10.0.0.40
```

Benefits: real per-device `REMOTE_ADDR` in the audit log, no 100 MB body cap, no 100-second edge timeout, no hairpin through the WAN, and no CDN bandwidth consumed by in-house traffic.

**The A record alone is not sufficient.** Cloudflare publishes an `HTTPS` (SVCB, type 65) record for proxied hostnames. Pi-hole answers A/AAAA from local records but forwards everything else upstream, so a type-65 query returned the upstream record:

```
1 . alpn="h3,h2" ipv4hint=<edge-v4>,<edge-v4> ech=<base64> ipv6hint=<edge-v6>,…
```

Two independent failures resulted:

1. **ECH.** The `ech=` payload declares a public name of `cloudflare-ech.com`. Chrome attempted Encrypted ClientHello, reached the local Apache (which knows nothing about ECH), and on fallback expected a certificate for that public name. It got the Nextcloud certificate and refused: `ERR_ECH_FALLBACK_CERTIFICATE_INVALID`.
2. **Address hints.** Chrome prefers SVCB `ipv4hint` values over the A record, so even without ECH the browser would have gone back out to the CDN, silently defeating the fast path.

Fix — make dnsmasq authoritative for the name so type-65 is never forwarded. Pi-hole v6 has no `/etc/dnsmasq.d/`; the setting lives in `pihole.toml` and must be written via the CLI because FTL rewrites that file on restart:

```bash
pihole-FTL --config misc.dnsmasq_lines '["local=/nextcloud.example.com/"]'
pihole-FTL --config misc.dnsmasq_lines      # read back before restarting
systemctl restart pihole-FTL
```

Applied to one instance at a time — restarting both together removes LAN name resolution entirely. A split configuration where one resolver leaks the HTTPS record produces intermittent, hard-to-attribute browser failures.

Verification (want the A record, want the HTTPS answer empty):

```bash
dig +short nextcloud.example.com A     @10.0.0.10   # → 10.0.0.40
dig +short nextcloud.example.com HTTPS @10.0.0.10   # → (empty)
```

**Maintenance caveat — this is a hard binding, not a preference.** `local=/…/` means dnsmasq will never forward queries for that name again. If the local A record is removed, or a resolver is rebuilt from fresh configuration, the name becomes **unresolvable on the LAN** rather than falling back to public DNS. The failure presents as a Nextcloud outage, not a DNS fault.

The office hostname is deliberately **not** overridden — it has no upload path and no cap concern, so it routes via the tunnel and there is one fewer binding to maintain. Chrome's own "Use secure DNS" (DoH) setting must be off, or it bypasses the local resolver entirely and the fast path never applies.

### 5.5 Reverse-proxy trust

Before this change every tunnel request appeared to Nextcloud as originating from `127.0.0.1` — confirmed by a `suspicious_login` email naming that address.

```bash
occ config:system:set trusted_proxies 0 --value=127.0.0.1
occ config:system:set forwarded_for_headers 0 --value=HTTP_CF_CONNECTING_IP
```

Severity of the pre-fix state, in order: `bruteforcesettings` throttles per source IP, so with all public traffic collapsed to one address a single attacker's failed logins would have throttled **every** user account simultaneously — a self-inflicted denial of service; `suspicious_login` would flag every login forever; and audit logging lost client attribution entirely.

`CF-Connecting-IP` rather than `X-Forwarded-For` deliberately: Cloudflare overwrites that header at the edge unconditionally, so a client cannot forge it, whereas `X-Forwarded-For` is append-only and spoofable. Trusting `127.0.0.1` as a proxy means anything able to reach loopback on this guest could assert a client IP — acceptable, as only Apache and cloudflared are on the box, and it is the same trust boundary the tunnel already implies.

Requests arriving via the LAN address or the split-DNS path do not come from a trusted proxy, so `REMOTE_ADDR` is used unchanged.

Final overwrite state — intentionally minimal:

| Key | Value |
|---|---|
| `overwrite.cli.url` | `https://nextcloud.example.com` |
| `overwritehost` | *not set* |
| `overwriteprotocol` | *not set* |
| `overwritecondaddr` | *not set* |
| `trusted_proxies` | `0 => 127.0.0.1` |
| `forwarded_for_headers` | `0 => HTTP_CF_CONNECTING_IP` |
| `trusted_domains` | `localhost`, `10.0.0.40`, `nextcloud.example.com` |

A stale workstation IP (`10.0.0.150`, added during an earlier build session for access by IP) was identified and removed, closing a long-standing backlog item.

### 5.6 Certificate migration to DNS-01

Once the WAN forward is removed, port 80 no longer reaches the origin and an HTTP-01 challenge cannot complete. Both hostnames now use certbot with the `dns-cloudflare` authenticator — the mechanism the office host already used.

```bash
certbot certonly --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/cloudflare.ini \
  -d nextcloud.example.com
```

Global certificate directives repointed in `/etc/apache2/mods-available/ssl.conf` — **global**, because the catch-all vhost inherits them too, and because the previous appliance file was a *combined* cert+key PEM (`grep -c 'PRIVATE KEY' … ` → 2), so a `SSLCertificateKeyFile` directive had to be **added**, not merely changed:

```apache
SSLCertificateFile /etc/letsencrypt/live/nextcloud.example.com/fullchain.pem
SSLCertificateKeyFile /etc/letsencrypt/live/nextcloud.example.com/privkey.pem
```

Renewal configuration corrections applied to both `/etc/letsencrypt/renewal/*.conf`:

```ini
dns_cloudflare_propagation_seconds = 60
renew_hook = systemctl reload apache2
```

Three findings behind those lines:

- **10 seconds of propagation is not enough.** The first `certbot renew --dry-run` failed with `DNS problem: NXDOMAIN looking up TXT for _acme-challenge.office.example.com`. Cloudflare's API acknowledges the record before it is live across their authoritative edge. 60 seconds succeeded. An initial CNAME-occlusion hypothesis (RFC 1034 §3.6.2) was **disproven** — it would have failed the other hostname too.
- **`renew --dry-run` does not persist command-line flags.** Only a full `certonly` writes them into the renewal configuration. A dry run passing with the flag on the command line proves nothing about the unattended `certbot.timer` run.
- **A bare `certonly` creates no `renew_hook`.** Without it, renewal writes new files and Apache keeps serving the old certificate from memory until something reloads it.

The appliance's own ACME cron job was retired by moving it aside. No dpkg package owned it, so no upgrade will restore it. **That wrapper hardcoded `--force`** — once inside its 30-day window it would have forced reissue daily and burned Let's Encrypt's five-duplicates-per-week limit while failing.

A related pre-existing trap was fixed and is now moot but recorded: the base `dehydrated` config pointed `DOMAINS_TXT` at a file that does not exist; the appliance's wrapper passes its own config explicitly. Automatic renewal therefore worked, but *manual* invocation — exactly what the troubleshooting table instructs on failure — would have failed on an empty domain list.

**Connector URLs confirmed unchanged.** An earlier proposal to re-point the DocumentServer's `StorageUrl` at the bare LAN IP was **withdrawn as incorrect**: the DocumentServer container carries a hosts pin mapping the public Nextcloud name to the LAN address, so it already resolves locally — traffic stays on the LAN *and* TLS name validation succeeds against a certificate with no IP SAN. Re-pointing to the bare IP would have broken validation. Post-migration the only path crossing the CDN is the browser's.

### 5.7 WAN port-forward removal

The router's forward of WAN 80/443 to the guest was removed after the tunnel was validated. Confirmed refused in 4 ms and validated externally from a mobile data connection. The estate now has zero inbound router port-forwards.

Order matters: certificates were moved to DNS-01 **before** the forward was dropped, while HTTP-01 still had a working fallback.

---

## 6. Edge constraints and measured limits

Empirically established rather than assumed:

| Constraint | Measurement | Consequence |
|---|---|---|
| Request body limit (Free plan) | 1.3 GiB chunked upload via web UI **succeeded**; single 150 MiB `PUT` to `/remote.php/dav/files/…` returned **413** (rejected early — 1.1 KB received against 2.8 MB sent) | Chunked clients unaffected; monolithic `PUT` capped at 100 MB |
| Origin timeout | 100 s at the edge, versus `ProxyTimeout 300` in the office vhost | DocumentServer operations over 100 s will now 524 |
| Non-HTML content | Cloudflare self-serve terms restrict serving large volumes | Accepted; precedent already set by the photo service |

**Client classification:**

- **Unaffected (chunked):** Nextcloud desktop, iOS, Android, web UI — all use `/remote.php/dav/uploads/…` with a default 10 MB `max_chunk_size`. `rclone` also supports Nextcloud chunking.
- **Capped at 100 MB (monolithic):** davfs2, Windows Explorer "Map network drive", macOS Finder Connect to Server, most generic third-party DAV apps. Note Windows' built-in WebDAV client has its own 50 MB download limit (`FileSizeLimitInBytes`) and is poor at scale regardless.
- **Note-sync clients (e.g. Joplin):** unaffected in practice — notes and attachments are individually small.

Under Option C, LAN and overlay clients bypass the CDN entirely, so even the monolithic case is uncapped on the internal path.

---

## 7. Cloudflare WAF and rate limiting

Plan is **Free**, which gates what is available: the basic Cloudflare Free Managed Ruleset is auto-deployed and not customisable; the OWASP Core Ruleset and full managed rules are paid-tier; five custom rules; one rate-limiting rule; and **no `Log` action**, so rules cannot be dry-run and must be deployed live one at a time with immediate testing.

### 7.1 Pre-flight — settings that break sync clients

Both disabled for the zone:

- **Bot Fight Mode** — challenges non-browser user agents, which is exactly what the Nextcloud desktop/mobile clients, WebDAV, and DocumentServer callbacks are.
- **Browser Integrity Check** — same failure class for DAV.

**These are accepted exceptions.** Cloudflare's Account Security Insights will continue to recommend enabling Bot Fight Mode. Do not "fix" it on a zone serving sync clients.

### 7.2 Custom rules (2 of 5 used)

Both scoped by hostname so other proxied hostnames in the zone are unaffected. `ends_with`/`starts_with` rather than regex, which is Business-tier and above.

**Rule 1 — front-controller whitelist at the edge** (order: First, action: Block) — mirrors the Apache rule so probes never consume origin bandwidth or a worker:

```
http.host eq "nextcloud.example.com"
and ends_with(http.request.uri.path, ".php")
and not http.request.uri.path in {"/index.php" "/remote.php" "/public.php" "/status.php" "/cron.php" "/ocs/v1.php" "/ocs/v2.php"}
```

**Rule 2 — common scanner prefixes** (order: after Rule 1, action: Block):

```
http.host eq "nextcloud.example.com"
and (starts_with(http.request.uri.path, "/wp-")
  or starts_with(http.request.uri.path, "/cgi-bin")
  or starts_with(http.request.uri.path, "/.env")
  or starts_with(http.request.uri.path, "/.git"))
```

PATH_INFO routes do not end in `.php`, so Rule 1 leaves `/remote.php/dav/…` and `/index.php/apps/…` alone.

### 7.3 Rate limiting (1 of 1 used)

Characteristic: IP; 5 requests per 10 seconds; action Block for 10 seconds (the only granularity the Free plan offers; throttling with Block is Enterprise-only):

```
http.host eq "nextcloud.example.com"
and http.request.method eq "POST"
and http.request.uri.path in {"/login" "/index.php/login"}
```

**Scoping to `POST /login` is the safety property.** Desktop and mobile clients authenticate via Login Flow v2 (`/login/flow`) and thereafter use app passwords over DAV, so none of them touch this rule. Rate-limiting `/remote.php/dav/` would throttle every sync client.

**Set expectations honestly:** 5 per 10 s permits roughly 30 attempts per minute sustained. This is a thin outer speed bump. The real brute-force defence is Nextcloud's `bruteforcesettings`, which only became effective once `trusted_proxies` was configured (§5.5).

### 7.4 Coverage limitation

LAN clients using the split-DNS fast path **bypass the WAF entirely**. Correct for a trusted network, but it means WAF rules are not a control on internal traffic.

---

## 8. Monitoring

### 8.1 Cloudflare tunnel health notifications — configured

Account → Notifications: **Tunnel Health Alert**, all tunnels, on becoming healthy/degraded/down, by email. Configured for every tunnel and delivery-tested. Genuinely external to the estate, so it survives a total site failure. A `Passive Origin Monitoring` notification is also enabled.

### 8.2 Uptime monitoring — known gap

A monitor was added against the public hostname intending to exercise the tunnel. **It does not.** Both the internal and the "public" monitor report the *same* certificate expiry — that of the origin's certbot certificate, not the CDN's edge certificate. The monitoring host resolves the hostname through the local DNS resolver, which is now *authoritative* for it and returns the LAN address. Response times were consistent with both being LAN.

The certificate-expiry field is a reliable discriminator for which endpoint a monitor is actually reaching — worth remembering as a diagnostic.

**Planned fix.** Point the monitoring host at public resolvers instead of the local ones. Existing monitors target IP addresses, so nothing breaks, and it is consistent with the principle that already placed monitoring outside the cluster's failure domain — monitoring should not depend on the DNS it is watching.

Lighter alternative: set the monitor URL to a CDN edge IP with a `{"Host": "nextcloud.example.com"}` header and tick *Ignore TLS/SSL error* (Cloudflare routes on the Host header). Caveats: hardcoded anycast address, and certificate monitoring is lost on that monitor.

### 8.3 Bare-IP probe fragility — do not break this

A probe against `https://10.0.0.40/status.php` works because a bare-IP connection sends **no SNI**: Apache handshakes using the default vhost (the catch-all) and then matches `Host: 10.0.0.40` to the Nextcloud vhost. Both inherit the *same* global certificate, so TLS parameters agree and the request is served.

If anyone later gives the Nextcloud vhost per-vhost certificate directives, that agreement breaks and the bare-IP path returns **421 Misdirected Request** — the monitor goes permanently red for a reason that looks nothing like a certificate problem. This is exactly why the office vhost, which does carry its own certificate directives, returns 421 to a no-SNI request. Test it with `curl --resolve office.example.com:443:127.0.0.1 …` instead.

---

## 9. Verification matrix

| Test | Command / method | Result |
|---|---|---|
| Whitelist denies without PHP | `error.log` after probe | `AH01630 client denied`, no `php:error` |
| Front controller intact | `curl -H 'Host: nextcloud…' https://127.0.0.1/status.php` | `200` |
| Apex dead-ends | `curl -H 'Host: example.com' https://127.0.0.1/` | `403` (was `400` from PHP) |
| HTTP redirect intact | `curl -H 'Host: nextcloud…' http://127.0.0.1/` | `301 → https://nextcloud.example.com/` |
| Legacy admin port closed | `ss -tlnp \| grep 12322` | empty |
| Vhost map | `apache2ctl -S` | catch-all default on :80/:443 |
| Served certificate | `openssl s_client -servername nextcloud… \| openssl x509 -enddate` | new certificate |
| Renewal, persisted settings | `certbot renew --dry-run` | both certificates succeed |
| Split DNS | `dig A` / `dig HTTPS` @ both resolvers | LAN address / empty |
| WAN closed | `curl -m 10 https://<WAN_IP>/` | refused in 4 ms |
| Chunked upload | 1.3 GiB zip via web UI | success |
| Monolithic PUT boundary | 150 MiB `curl -T` to DAV | `413` |
| Office suite | document open | working |
| External paths | mobile cellular, mobile Wi-Fi, overlay off, LAN browser | all working |
| Tunnel alerts | Cloudflare "Test" | delivered |
| Backup | `vzdump 111 --mode snapshot` | taken |

Application-log outcome: **2 errors on the day of remediation**, against 430 and 258 on the two worst preceding days.

---

## 10. Rollback procedures

| Change | Rollback |
|---|---|
| PHP whitelist | `a2disconf nc-php-whitelist && systemctl reload apache2` |
| Catch-all vhost | `a2dissite 000-catchall && systemctl reload apache2` |
| WAF / rate-limit rules | Toggle off in dashboard |
| Split DNS | `pihole-FTL --config misc.dnsmasq_lines '[]'` + remove local A record, restart FTL, on each instance |
| `cpulimit` / `cpuunits` | `qm set 111 --delete cpulimit --delete cpuunits` |
| Legacy admin vhost | `a2ensite adminer`, restore `Listen` line from `ports.conf` backup, `systemctl restart apache2` |
| Stapling | Restore `ssl.conf` backup |
| Certificate | Restore `ssl.conf` backup pointing at the appliance PEM |
| Tunnel | `systemctl disable --now cloudflared`; re-add A records, DNS-only; **re-instate the router WAN forward** |
| Vhost names | Restore `nextcloud.conf` backups |

`/etc` on the guest is an **etckeeper git repository**, which provides a further rollback path (`git -C /etc log`, `git -C /etc diff`). Note that this also means `/etc/letsencrypt/archive/*/privkey*.pem` is committed to git history permanently — local and root-only, but it belongs in the security posture record.

Full-guest rollback: the post-change vzdump.

---

## 11. Known traps and gotchas

1. **`ServerName` silently overwrites.** It is a single-value directive. A second `ServerName` in the same vhost replaces the first — and `apache2ctl configtest` returns `Syntax OK`. This took the production hostname offline mid-session: the vhost no longer matched it, so requests fell through to the catch-all and returned 403. **Additional names go on `ServerAlias`, which is multi-value.**
2. **SNI selects the certificate; `Host` selects the vhost.** If they resolve to vhosts with *different* TLS configuration, Apache returns **421 Misdirected Request**. A bare-IP connection sends no SNI, so it handshakes on the default vhost's certificate.
3. **`originServerName` is required when the tunnel service is an IP literal.** Otherwise: 502 and `x509: certificate is valid for …, not localhost`.
4. **Cloudflare flattens proxied CNAMEs**, so `dig CNAME` returns an empty answer even when the record exists. Query `A` and look for edge addresses instead.
5. **Proxied hostnames publish an `HTTPS`/SVCB record**, carrying both an ECH config and `ipv4hint` values. Either one defeats a local A-record override. `local=/name/` in dnsmasq is the fix; the ECH toggle is not exposed on Free plans.
6. **`local=/name/` is a hard binding.** dnsmasq will never forward that name again — lose the local record and the name becomes unresolvable on the LAN rather than falling back to public DNS.
7. **`certbot renew --dry-run` does not persist command-line flags.** A passing dry run with a flag proves nothing about the unattended timer run.
8. **Cloudflare DNS-01 needs ~60 s propagation, not the 10 s default.** The API acknowledges before the record is live edge-wide.
9. **The appliance ACME cron wrapper hardcodes `--force`.** A persistent challenge failure inside the renewal window burns the CA's duplicate-certificate limit daily rather than backing off.
10. **The base `dehydrated` config is not the file the appliance wrapper uses.** Automatic renewal worked; manual invocation — the documented troubleshooting step — would not have.
11. **`trusted_proxies` unset behind a loopback tunnel collapses every client to `127.0.0.1`**, inverting `bruteforcesettings` into a self-DoS across all accounts.
12. **A `Listen` directive without a matching vhost serves the main server config** — plaintext, default DocumentRoot. Removing a vhost without removing its `Listen` is worse than leaving it enabled.
13. **`Listen` changes need `systemctl restart`, not `reload`.**
14. **Nextcloud's `ErrorDocument` routes through `index.php`**, so a request denied by the `.php` whitelist returns **404**, not 403. Judge the whitelist by the absence of `php:error` in the log, not by status code.
15. **Bot Fight Mode and Browser Integrity Check break Nextcloud (and Immich) sync clients.** The CDN's own security insights will keep recommending them.
16. **Free-plan WAF has no `Log` action**, so custom rules cannot be dry-run. Deploy one at a time with immediate client testing.
17. **A monitor's certificate-expiry field reveals which endpoint it actually reaches.** Two monitors reporting the same expiry are hitting the same TLS terminator regardless of the URL configured.
18. **`chmod -x` is not a durable disable for package-shipped cron jobs.** Check ownership with `dpkg -S`; use `dpkg-divert --rename --add` for owned files, `mv` for unowned ones.
19. **Certificate Transparency is an origin-discovery channel.** Issuing a certificate for a hostname publishes it. Any name with a DNS-only record pointing at a residential WAN address will be scanned within days.
20. **Unquoted `<placeholder>` in a shell command is read as input redirection** — `bash: user: No such file or directory`.

---

## 12. Outstanding items

| Priority | Item |
|---|---|
| **High** | **MFA on the CDN/DNS provider account.** That account holds DNS for every zone, the tunnel credentials carrying **all** inbound traffic, the WAF rules, and static-site deployment. Compromise bypasses every control in this document at once. Register a hardware key. |
| High | A web-based system administration panel still listening on all interfaces on a high port — larger surface than the database front-end just removed. Decide: bind to loopback, firewall, or remove. |
| High | Point the monitoring host at public resolvers, closing the gap in §8.2. |
| Medium | Nextcloud **34.0.2** available; running 34.0.0.12. Use the CLI `updater.phar` path — the web updater is blocked by the `.php` whitelist (§4.1). |
| Medium | Enforce 2FA. Providers installed, enforcement off. Deferred until users are settled on the new services; enforce for administrators first. |
| Medium | SMB external storage errors in the log (`Storage with mount id … is not available`, `Storage unauthorized`). Check `occ files_external:list` then `occ files_external:verify <id>`. |
| Medium | Redis stability — fatal `Redis server … went away` entries. It carries `memcache.locking`; its death causes file-lock failures. |
| Medium | Minimum TLS 1.0 → 1.2; enable Always Use HTTPS; confirm SSL/TLS mode is **Full (strict)**. |
| Medium | DMARC record — MX and SPF present, no `_dmarc` TXT. Start with `v=DMARC1; p=none; rua=…`. |
| Medium | Credential rotation pass, now also covering the tunnel token in shell history, the token file on disk, and the etckeeper git history containing `privkey.pem`. |
| Low | Delete the orphaned appliance PEM once past its expiry — it still contains a private key. |
| Low | `occ preview:repair` — preview cache damaged by earlier disk-full events. |
| Low | AppAPI deploy daemon warning — External Apps not used. **Accepted exception.** |
| Low | Server identifier warning — cosmetic on a single-server deployment. |
| Low | Zone tidy-up: remove stale in-zone NS records and the `_autodiscover._tcp` SRV. |
| Low | `security.txt`, Turnstile, AI crawler controls — low-severity insights; optional. |

---

## 13. Explain like I'm 5

The family cloud lives in a small computer in the house. Until today the front door of the house had a letterbox opening straight onto the street, and the street directory listed the address. Strangers walked past all day pushing thousands of notes through it asking "is there a spare key under this mat?" Every single note had to be picked up, read and thrown away by hand — and there were only sixteen pairs of hands. Nothing was ever stolen, but the hands were too busy to answer the family when they knocked.

Three things changed. The hands now recognise nonsense notes and bin them without reading (that is nearly free). The letterbox onto the street was bricked up entirely. And all real visitors now arrive through a courier company that already knows the family, checks the parcels, and knows the way in — while anyone standing in the house can still just walk down the hall as before, which is quicker.

---

## 14. References

- Cloudflare Tunnel — <https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/>
- Cloudflare Tunnel origin configuration (`originServerName`, `httpHostHeader`) — <https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/configure-tunnels/>
- Cloudflare WAF rate-limiting rules and parameters — <https://developers.cloudflare.com/waf/rate-limiting-rules/>
- Cloudflare WAF custom rules — <https://developers.cloudflare.com/waf/custom-rules/>
- Cloudflare rate-limiting best practices (login-endpoint protection) — <https://developers.cloudflare.com/waf/rate-limiting-rules/best-practices/>
- Nextcloud reverse-proxy configuration — <https://docs.nextcloud.com/server/latest/admin_manual/configuration_server/reverse_proxy_configuration.html>
- Nextcloud 34.0.2 changelog — <https://nextcloud.com/changelog/#34-0-2>
- Apache `mod_authz_core` / `FilesMatch` — <https://httpd.apache.org/docs/2.4/mod/mod_authz_core.html>
- Apache name-based virtual host matching (SNI, 421) — <https://httpd.apache.org/docs/2.4/vhosts/name-based.html>
- certbot `dns-cloudflare` plugin — <https://certbot-dns-cloudflare.readthedocs.io/>
- dnsmasq `local=` / Pi-hole v6 configuration — <https://docs.pi-hole.net/>
- RFC 9460 (SVCB/HTTPS resource records) — <https://www.rfc-editor.org/rfc/rfc9460.html>
- Proxmox VE Administration Guide, `qm` CPU limits — <https://pve.proxmox.com/pve-docs/chapter-qm.html>

---

## 15. Appendices

### Appendix A — files created, modified and backed up

| Path | Action |
|---|---|
| `/etc/apache2/conf-available/nc-php-whitelist.conf` | created |
| `/etc/apache2/sites-available/000-catchall.conf` | created, enabled |
| `/var/www/empty` | created |
| `/etc/apache2/sites-available/nextcloud.conf` | `ServerName` + `ServerAlias` added (backups taken) |
| `/etc/apache2/sites-available/adminer.conf` | disabled |
| `/etc/apache2/ports.conf` | legacy `Listen` removed (backup taken) |
| `/etc/apache2/mods-available/ssl.conf` | stapling off; certificate paths repointed (backups taken) |
| `/etc/dehydrated/config` | `DOMAINS_TXT` corrected (backup taken) |
| `/etc/cron.daily/confconsole-dehydrated` | moved aside |
| `/etc/letsencrypt/renewal/*.conf` | propagation seconds + renew hook added (backups taken) |
| `/etc/letsencrypt/live/nextcloud.example.com/` | new certbot certificate |
| `/etc/cloudflared/token`, systemd unit | created by `cloudflared service install` |
| APT sources | Cloudflare repo added |
| Pi-hole `pihole.toml` on both resolvers | `misc.dnsmasq_lines` + local A record |

Disk state after work: root filesystem 75 G, 15 G used (21%); Nextcloud data 9.6 G; Apache logs 161 M. 819 MB reclaimed by `apt autoremove` (two orphaned kernels).

### Appendix B — command reference

```bash
# Whitelist / vhosts
a2enconf nc-php-whitelist ; a2disconf nc-php-whitelist
a2ensite 000-catchall     ; a2dissite 000-catchall
apache2ctl configtest ; apache2ctl -S
systemctl reload apache2          # config changes
systemctl restart apache2         # Listen changes only

# Served certificate
echo | openssl s_client -connect 127.0.0.1:443 \
  -servername nextcloud.example.com 2>/dev/null \
  | openssl x509 -noout -subject -issuer -enddate
certbot renew --dry-run
certbot certificates

# Vhost with its own cert (avoids 421)
curl -ksS -o /dev/null -w '%{http_code}\n' \
  --resolve office.example.com:443:127.0.0.1 \
  https://office.example.com/healthcheck

# Tunnel
systemctl status cloudflared ; journalctl -u cloudflared -f
pgrep -a cloudflared              # confirm exactly one connector

# Nextcloud proxy state
occ config:system:get trusted_domains
occ config:system:get trusted_proxies
occ config:system:get forwarded_for_headers

# Split DNS (per resolver)
pihole-FTL --config misc.dnsmasq_lines
dig +short nextcloud.example.com A     @10.0.0.10
dig +short nextcloud.example.com HTTPS @10.0.0.10

# Confirm no PHP fork on a denied probe
grep 'AH01630' /var/log/apache2/error.log | tail -3
grep 'not found or unable to stat' /var/log/apache2/error.log | tail -1
```

### Appendix C — change log

| Date | Version | Change |
|---|---|---|
| 25 Jul 2026 | 1.0 | Initial runbook. Incident diagnosis (PHP-webshell reconnaissance against a directly reachable origin, not a CPU shortfall); origin hardening (PHP front-controller whitelist, catch-all vhost, `cpulimit`, legacy DB front-end removal, stapling off); Cloudflare Tunnel migration for both public hostnames; DNS zone cleanup; split-DNS internal fast path with `local=` binding; `trusted_proxies`/`forwarded_for_headers`; both certificates migrated to certbot DNS-01 with 60 s propagation and reload hooks, appliance ACME retired; WAN 80/443 forward removed; WAF custom rules and rate limiting; tunnel health notifications. |

### Appendix D — glossary (additions to the appliance runbook)

| Term | Meaning |
|---|---|
| **Cloudflare Tunnel / cloudflared** | Outbound-only connector from the origin to Cloudflare's edge; removes the need for inbound port-forwards |
| **`originServerName`** | Per-route setting telling cloudflared which name to use for SNI and certificate validation against the origin |
| **SVCB / HTTPS record** | DNS record type 65 (RFC 9460) carrying connection hints — ALPN, address hints and ECH configuration |
| **ECH** | Encrypted ClientHello; encrypts the SNI. Requires client and terminating server to agree, so it breaks split-horizon DNS to a non-ECH origin |
| **421 Misdirected Request** | Apache's refusal to serve a request whose `Host` maps to a vhost with different TLS configuration than the one that completed the handshake |
| **Chunked upload** | Nextcloud's `/remote.php/dav/uploads/` mechanism splitting large files into ~10 MB requests, keeping each under proxy body limits |
| **`local=` (dnsmasq)** | Declares a name locally authoritative — answered from local configuration, never forwarded upstream |
| **CPU pressure stall (PSI)** | Kernel metric; `Full` means every runnable task was waiting on CPU |
| **Certificate Transparency** | Public append-only log of issued certificates; publishes hostnames at issuance |

---

*End of document (v1.0, 25 July 2026). Sanitised counterpart of `nextcloud-tunnel-hardening-INTERNAL.md`.*
