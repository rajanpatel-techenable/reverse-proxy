# Reverse Proxy Guide (nginx-reverseproxy, 192.168.0.105)

How the reverse proxy in front of MCX, Compass and Inventory works, what each
config file does, and how to operate it.

Last updated: 2026-11-03. Config backup: [../configs/](../configs/)

---

## 1. Overview

All traffic from the internet reaches the internal services through two hops:

1. **FortiGate** (public IP `103.174.244.121`) port-forwards to this server.
2. **nginx-reverseproxy** on `192.168.0.105` decides which backend gets it.

```
                           Internet clients
                                  │
                     103.174.244.121 (FortiGate)
   TCP 443, 4434, 7755 │                    │ UDP public 50000-50500 → 61000-61500
                      ▼                    ▼
      ┌──────────────────────── 192.168.0.105 ────────────────────────┐
      │  nginx-reverseproxy                                           │
      │                                                               │
      │  stream :443 ── reads TLS SNI ─┬─ compass / inventory ─► :10443 (HTTPS, decrypts)
      │   (no decryption)              │                          ├─ compass.samiksha.ai   ─► 127.0.0.1:3000
      │                                │                          ├─ inventory.samiksha.ai ─► 192.168.0.13:80
      │                                │                          └─ anything else ─► stock nginx page
      │                                └─ no SNI / bare IP ──► :10444 ─► 192.168.8.10:443 (MCX, TLS untouched)
      │                                                               │
      │  stream tcp :4434, :7755 ────────────────────────────► 192.168.8.10 same port (MCX)
      │  stream udp :61000-61500 ────────────────────────────► 192.168.8.10:50000-50500 (MCX media)
      │                                                               │
      │  http :9112 ─► 127.0.0.1:3000 (Compass)      plain HTTP, LAN/Tailscale (not forwarded by FortiGate)
      │  http :9113 ─► 192.168.0.13:80 (Inventory)                    │
      └───────────────────────────────────────────────────────────────┘

   TCP 22 (SFTP): FortiGate DNATs straight to 192.168.8.10:22, bypassing this host.
```

### Backends

| Service | Backend | Reached through |
|---|---|---|
| MCX server (signalling, TLS) | `192.168.8.10:443` | `https://103.174.244.121` (bare IP, no hostname) |
| MCX server (audio/video media) | `192.168.8.10:50000-50500/udp` | `103.174.244.121:50000-50500/udp` (proxy-side 61000-61500) |
| MCX server (extra TCP services) | `192.168.8.10:4434`, `192.168.8.10:7755` | `103.174.244.121:4434`, `:7755` |
| MCX server (SFTP) | `192.168.8.10:22` | `103.174.244.121:22` (FortiGate direct, not via proxy) |
| Compass | `127.0.0.1:3000` (the separate `compass-nginx` instance) | `https://compass.samiksha.ai`, or `:9112` internally |
| Inventory | `192.168.0.13:80` | `https://inventory.samiksha.ai`, or `:9113` internally |

### Other nginx instances on this host (not part of this proxy)

| Service | Config | Role |
|---|---|---|
| `compass-nginx` | `/opt/compass/system/nginx/nginx.conf` | Compass frontend + API on `:3000` (a backend of this proxy) |
| `trinetra-nginx` | `/opt/trinetra/local-node/system/nginx/nginx.conf` | Trinetra; also serves port 80 and the Let's Encrypt webroot `/var/www/letsencrypt` |

All three run the same `/usr/sbin/nginx` binary with different `-c` config
files. Use the matching service name, and always pass
`-c /etc/nginx-reverseproxy/nginx.conf` when you run nginx commands for this
proxy. Without `-c`, nginx loads `/etc/nginx/nginx.conf`, which this proxy
doesn't use.

---

## 2. Traffic flows

### 2.1 HTTPS on port 443: one port shared by MCX and the websites

Port 443 is shared by MCX (which handles its own TLS) and the websites (whose
certificate is on this proxy). To tell them apart without decrypting, the
`stream` block reads only the **SNI**: the hostname the client sends in
plain text at the start of the TLS handshake.

| Client connects to | SNI sent | Routed to |
|---|---|---|
| `https://compass.samiksha.ai` | `compass.samiksha.ai` | `web` → `127.0.0.1:10443` |
| `https://inventory.samiksha.ai` | `inventory.samiksha.ai` | `web` → `127.0.0.1:10443` |
| `https://103.174.244.121` (MCX clients) | none, or an IP | `mcx` → `127.0.0.1:10444` → `192.168.8.10:443` |
| any other hostname | that name | `web` → stock nginx page (catch-all) |

**Website path:**
1. The stream server passes the TLS connection unchanged to `127.0.0.1:10443`,
   adding a **PROXY protocol** header that carries the real client IP.
2. The `http` server on `:10443` reads that header, decrypts TLS using the
   Let's Encrypt certificate, and picks the site by `server_name`.
3. It forwards plain HTTP to the backend with `X-Real-IP`, `X-Forwarded-For`
   and `X-Forwarded-Proto: https`. WebSocket upgrades are passed through.

**MCX path:**
1. The stream server sends the connection to `127.0.0.1:10444`, also with a
   PROXY protocol header.
2. The MCX server doesn't understand PROXY protocol, so the `:10444` server
   accepts the header, removes it, and forwards the raw TLS to `192.168.8.10:443`.
3. TLS is end-to-end between the client and MCX; the proxy never decrypts it.
   MCX sees every connection as coming from `192.168.0.105`.

### 2.2 MCX audio/video media: UDP 50000-50500 (proxy-side 61000-61500)

```
client ─► 103.174.244.121:50xyz ─► FortiGate VIP ─► 192.168.0.105:61xyz ─► nginx ─► 192.168.8.10:50xyz
          (50000..50500)              (port +11000)    (61000..61500)        (port −11000)
```

- Clients and MCX both use 50000–50500. Only the hop through this host uses
  61000–61500. The FortiGate adds 11000 to the port, and nginx subtracts it
  again: `61xyz` → `50xyz`, so `61000` → `50000` and `61500` → `50500`.
- nginx listens on all 501 ports with one `listen` line. A `map` on
  `$server_port` (the port the packet arrived on) works out the MCX port.
- Each client address becomes a UDP "session". nginx sends to MCX from its own
  random source port, and replies from MCX come back on that session and are
  returned to the client.
- `reuseport` makes the kernel deliver every packet of one client flow to the
  same nginx worker. Without it, one call's packets could be split across
  workers, and MCX would see the stream jump between source ports.
- A session closes after 60 seconds without traffic (`proxy_timeout 60s`).
- **Why the proxy doesn't listen on 50000–50500 itself:** Linux picks random
  local ports from 32768–60999 (`net.ipv4.ip_local_port_range`), so other
  programs get ports in 50000–50500 at random. AnyDesk also binds UDP 50001
  on this host, and nginx would fail to start if it couldn't bind that port.
  61000–61500 is above the random range, so no other program on the host
  gets those ports by chance.

**What the MCX server must be configured for:**
- RTP/RTCP media port range: 50000–50500.
- The media address it gives clients (in SDP): the FortiGate public IP
  `103.174.244.121`, not `192.168.8.10`.
- Symmetric RTP / latching: reply to the address packets actually arrive
  from. All media reaches MCX from `192.168.0.105`.

### 2.3 MCX extra TCP ports 4434 / 7755, and SFTP on 22

```
client ─► 103.174.244.121:P ─► FortiGate VIP ─► 192.168.0.105:P ─► nginx ─► 192.168.8.10:P   (P = 4434, 7755)
client ─► 103.174.244.121:22 ─► FortiGate VIP ─────────────────────────────► 192.168.8.10:22   (SFTP, no proxy)
```

- 4434 and 7755 are plain TCP passthrough. The proxy doesn't decrypt or
  inspect anything and adds no PROXY protocol header, so MCX sees connections
  coming from `192.168.0.105`.
- SFTP doesn't go through this host. This host's own sshd already listens on
  `0.0.0.0:22`, so nginx can't bind port 22. The FortiGate is already the
  route between 192.168.0.0/24 and 192.168.8.10, so it forwards 22 straight
  to MCX, and MCX sees real client IPs for SFTP.

### 2.4 Plain HTTP on 9112 / 9113

Older plain-HTTP entry points, kept for internal use. They listen on all
interfaces and ufw allows them from anywhere, so they're reachable from the
LAN and over Tailscale. They aren't exposed to the internet only because
the FortiGate doesn't forward 9112/9113. Traffic on them is unencrypted.

| Port | Backend |
|---|---|
| 9112 | Compass `127.0.0.1:3000` |
| 9113 | Inventory `192.168.0.13:80` |

---

## 3. Config files, one by one

All files are in `/etc/nginx-reverseproxy/`. A copy of each is in
`../configs/etc/nginx-reverseproxy/`.

```
/etc/nginx-reverseproxy/
├── nginx.conf                 main file: loads stream module, http{} and stream{} blocks
├── conf.d/                    included inside http{}   (HTTP / decrypted HTTPS)
│   ├── 00-default.conf        catch-all HTTPS site for unknown hostnames
│   ├── compass.conf           Compass on :9112 and compass.samiksha.ai
│   └── inventory.conf         Inventory on :9113 and inventory.samiksha.ai
└── stream.d/                  included inside stream{} (raw TCP/UDP, no decryption)
    ├── tls-443.conf           443 SNI router + MCX TLS passthrough
    ├── mcx-media-udp.conf     MCX UDP media 61000-61500 -> MCX 50000-50500
    └── mcx-tcp.conf           MCX TCP 4434 and 7755 passthrough
```

Only files ending in `.conf` are loaded. Files named `*.bak` or
`*.removed-*` are old versions kept for reference and are ignored by nginx.

### 3.1 `nginx.conf`

```nginx
load_module /usr/lib/nginx/modules/ngx_stream_module.so;
```
Loads the `stream` module (raw TCP/UDP proxying). On Ubuntu it ships as a
dynamic module, and this instance doesn't read `/etc/nginx/modules-enabled/`,
so it must be loaded here.

```nginx
worker_processes auto;            # one worker per CPU core (32)
pid /run/nginx-reverseproxy.pid;  # must match PIDFile= in the systemd unit
error_log /var/log/nginx-reverseproxy/error.log;
events { worker_connections 1024; }
```

```nginx
http {
    ...
    client_body_temp_path /var/lib/nginx-reverseproxy/body;
    proxy_temp_path       /var/lib/nginx-reverseproxy/proxy;
```
Own temp directories, so this instance doesn't share `/var/lib/nginx` with
the other nginx instances.

```nginx
    set_real_ip_from 127.0.0.1;
    real_ip_header proxy_protocol;
```
HTTPS requests reach `http{}` from the 443 stream router on `127.0.0.1`.
These two lines make nginx use the client IP from the PROXY protocol header
as `$remote_addr`, so logs and `X-Real-IP` show the real visitor instead of
`127.0.0.1`.

```nginx
    include /etc/nginx-reverseproxy/conf.d/*.conf;
}
stream {
    include /etc/nginx-reverseproxy/stream.d/*.conf;
}
```

### 3.2 `stream.d/tls-443.conf`: the 443 router

```nginx
map $ssl_preread_server_name $tls_backend {
    compass.samiksha.ai      web;
    inventory.samiksha.ai    web;
    ""                       mcx;     # no SNI (client connected by IP)
    ~^[0-9.]+$               mcx;     # SNI is an IPv4 address
    default                  web;     # unknown hostnames -> catch-all page
}
upstream web { server 127.0.0.1:10443; }
upstream mcx { server 127.0.0.1:10444; }
```
`$ssl_preread_server_name` is the SNI that `ssl_preread on` reads. The map
turns it into an upstream name.

```nginx
server {
    listen 192.168.0.105:443;
    ssl_preread on;          # peek at the TLS ClientHello, don't decrypt
    proxy_pass $tls_backend;
    proxy_protocol on;       # prepend real client IP for the next hop
    access_log /var/log/nginx-reverseproxy/stream.log tls;
}
```
It listens only on the LAN address, which is where the FortiGate forwards 443.
The `tls` log format records SNI, chosen backend, bytes and duration for
every connection.

```nginx
server {
    listen 127.0.0.1:10444 proxy_protocol;   # accept and strip PROXY header
    proxy_pass 192.168.8.10:443;             # MCX gets plain TLS
    access_log off;
}
```

### 3.3 `stream.d/mcx-media-udp.conf`: MCX media

```nginx
map $server_port $mcx_media_port {
    "~^61(?<mcx_port_suffix>[0-9][0-9][0-9])$"  50$mcx_port_suffix;
}

server {
    listen 192.168.0.105:61000-61500 udp reuseport;
    proxy_pass 192.168.8.10:$mcx_media_port;
    proxy_timeout 60s;
    access_log off;
}
```
| Directive | Meaning |
|---|---|
| `map $server_port ...` | keeps the last three digits of the arrival port and puts `50` in front: `61xyz` → `50xyz` (a fixed −11000 shift) |
| `"~^61(...)$"` | regex key. The quotes are required because `{`/`}` and some characters would otherwise break the nginx parser |
| `listen ...:61000-61500 udp` | one listener per port in the range (nginx ≥ 1.15.10) |
| `reuseport` | each worker gets its own socket per port; the kernel keeps a client flow on one worker |
| `proxy_timeout 60s` | close idle UDP sessions after 60 s |
| `access_log off` | media is high-volume, so logging every session would flood the disk |

501 ports × 32 workers ≈ 16,000 sockets, which is why the systemd unit raises
the open-file limit (see 3.8).

### 3.4 `stream.d/mcx-tcp.conf`: MCX TCP 4434 / 7755

```nginx
server { listen 192.168.0.105:4434; proxy_pass 192.168.8.10:4434; }
server { listen 192.168.0.105:7755; proxy_pass 192.168.8.10:7755; }
```
Plain TCP passthrough on the same port. It listens only on the LAN address,
where the FortiGate forwards. To add another MCX TCP port, copy one block,
add a ufw rule, and add a FortiGate VIP.

### 3.5 `conf.d/00-default.conf`: catch-all

```nginx
server {
    listen 127.0.0.1:10443 ssl proxy_protocol default_server;
    server_name _;
    ssl_certificate     /etc/letsencrypt/live/compass.samiksha.ai/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/compass.samiksha.ai/privkey.pem;
    root /usr/share/nginx/html;
    location / { try_files $uri /index.html; }
}
```
Any HTTPS request whose hostname matches no site gets the stock nginx page.
Without this, nginx would use the first site it loaded (Compass) as the
default and expose it to every hostname.

### 3.6 `conf.d/compass.conf`

Two server blocks:

| Block | Listens | Purpose |
|---|---|---|
| plain HTTP | `:9112` (all interfaces) | internal access to Compass |
| HTTPS | `127.0.0.1:10443 ssl proxy_protocol`, `server_name compass.samiksha.ai` | public site behind the 443 router |

Both `proxy_pass http://127.0.0.1:3000` and set `Host`, `X-Real-IP`,
`X-Forwarded-For`, `X-Forwarded-Proto`. The HTTPS block also uses
`proxy_http_version 1.1` with `Upgrade`/`Connection` headers so WebSockets
work.

### 3.7 `conf.d/inventory.conf`

Same structure as Compass: `:9113` plain HTTP and
`inventory.samiksha.ai` on `:10443`, both forwarding to
`http://192.168.0.13:80`.

### 3.8 systemd unit: `/etc/systemd/system/nginx-reverseproxy.service`

```ini
ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx-reverseproxy/nginx.conf   # refuse to start on bad config
ExecStart=/usr/sbin/nginx -c /etc/nginx-reverseproxy/nginx.conf
ExecReload=/usr/sbin/nginx -s reload -c /etc/nginx-reverseproxy/nginx.conf
PIDFile=/run/nginx-reverseproxy.pid
Restart=on-failure
```

Override `/etc/systemd/system/nginx-reverseproxy.service.d/limits.conf`:
```ini
[Service]
LimitNOFILE=65536
```
The default soft limit of 1024 open files is far below the ~16,000 sockets
the UDP media listener needs. nginx would fail to start without this.

### 3.9 TLS certificate

- One Let's Encrypt ECDSA certificate covers `compass.samiksha.ai` and
  `inventory.samiksha.ai`: `/etc/letsencrypt/live/compass.samiksha.ai/`.
- Renewal: certbot `webroot` method, using `/var/www/letsencrypt`. The HTTP
  challenge on port 80 is answered by **trinetra-nginx**, not by this proxy.
- Current expiry: 2027-01-07. Certbot renews 30 days before expiry.
- The MCX server uses its own certificate (its TLS passes through untouched).

### 3.10 Firewall (ufw) rules that matter for the proxy

| Rule | Purpose |
|---|---|
| `443/tcp ALLOW Anywhere` | HTTPS / MCX signalling |
| `192.168.0.105 61000:61500/udp on enp5s0 ALLOW` | MCX media |
| `192.168.0.105 4434,7755/tcp on enp5s0 ALLOW` | MCX extra TCP services |
| `9112/tcp`, `9113/tcp ALLOW Anywhere` | plain-HTTP Compass / Inventory (LAN; not forwarded by FortiGate) |
| `Anywhere on tailscale0 ALLOW` | everything is reachable over Tailscale |

### 3.11 FortiGate (configured on the firewall, not on this host)

| VIP | External | Mapped to |
|---|---|---|
| HTTPS | `103.174.244.121:443/tcp` | `192.168.0.105:443` |
| MCX media | `103.174.244.121:50000-50500/udp` | `192.168.0.105:61000-61500` (port +11000) |
| MCX TCP 4434 | `103.174.244.121:4434/tcp` | `192.168.0.105:4434` |
| MCX TCP 7755 | `103.174.244.121:7755/tcp` | `192.168.0.105:7755` |
| MCX SFTP | `103.174.244.121:22/tcp` | `192.168.8.10:22` (direct to MCX, not via proxy) |

Each VIP has a matching WAN → LAN policy (WAN → MCX zone for SFTP). The
proxy's default gateway is `192.168.0.1`, so replies return through the
FortiGate.

Facts about this FortiGate (`Tidal_Wave_Old_Office_FW`):
- The WAN link is PPPoE interface `ppp2`, a member of SD-WAN
  (`virtual-wan-link`). Policies use `virtual-wan-link` → `internal1`.
- Media policy: ID 22 `mcx_udp`, destination VIP `MCX-media-udp`, service
  `ALL`, NAT off (MCX and the proxy see the real client IP). It used to use a
  custom service `mcx_udp` limited to UDP 61000-61500. After the public range
  moved to 50000-50500 that service stopped matching, and every packet was
  `Denied by forward policy check (policy 0)`.
- The TCP 4434/7755 policy has NAT on, so those connections reach the proxy
  from `192.168.0.1`. It still works, but turn NAT off to see real client IPs.
- Tests from inside the office to the public IP (hairpin) arrive on
  `internal1` and need a separate `internal1 → internal1` policy (NAT on).
  The media VIP isn't in one, so UDP tests from the LAN are denied. Always
  test from outside (for example a phone on mobile data).

Debug a VIP on the FortiGate:
```
diagnose debug reset
diagnose debug flow filter clear
diagnose debug flow filter proto 17          # 6 for TCP
diagnose debug flow filter dport 50000
diagnose debug flow show function-name enable
diagnose debug flow trace start 20
diagnose debug enable
# send traffic, read the output, then:
diagnose debug disable
```

---

## 4. Operating the proxy

### Check status

```bash
systemctl status nginx-reverseproxy
sudo ss -tlnp | grep -E '192.168.0.105:(443|4434|7755)|:1044[34]|:911[23]'
sudo ss -ulnp | grep nginx | grep -c '192.168.0.105:61'    # expect 16032 (501 ports x 32 workers)
```

### Change config safely

```bash
sudo nano /etc/nginx-reverseproxy/...                       # edit
sudo nginx -t -c /etc/nginx-reverseproxy/nginx.conf         # always test first
sudo systemctl reload nginx-reverseproxy                    # applies without dropping connections
cd ~/reverse-proxy/configs && ./backup.sh                   # snapshot the change
```
Use `reload`, not `restart`, so existing 443 connections aren't dropped. A
restart is only needed after changing the systemd unit or its limits.

### Logs

| File | Contents |
|---|---|
| `/var/log/nginx-reverseproxy/stream.log` | every 443 connection: client IP, SNI, backend, bytes, duration |
| `/var/log/nginx-reverseproxy/access.log` | HTTP requests to Compass / Inventory / catch-all |
| `/var/log/nginx-reverseproxy/error.log` | errors for all of the above, including UDP media upstream errors |

```bash
sudo tail -f /var/log/nginx-reverseproxy/stream.log                  # who is hitting 443 and where it goes
sudo grep '192.168.8.10:50' /var/log/nginx-reverseproxy/error.log    # media problems
```

### Test the UDP media path

```bash
# On the proxy: watch media traffic on both sides of the proxy
sudo tcpdump -ni enp5s0 'udp portrange 61000-61500 or (host 192.168.8.10 and udp portrange 50000-50500)'

# From a machine outside the network:
echo test | nc -u 103.174.244.121 50000
```
Expected in the capture: `<client IP> > 192.168.0.105.61000`, then
`192.168.0.105.<random> > 192.168.8.10.50000`.
- `ICMP ... udp port 50000 unreachable` from 192.168.8.10 means the path is
  fine but MCX has no listener on that port right now. Many media servers
  open ports only during an active call.
- Test the proxy alone (skips the FortiGate) from the proxy host:
  `echo test | nc -u -w1 192.168.0.105 61123` should show
  `> 192.168.8.10.50123`.

### Test the MCX TCP ports

```bash
nc -vz 103.174.244.121 4434      # from outside: full path
nc -vz 192.168.0.105 7755        # from the LAN: proxy only
```
If the connection opens and then closes at once, look for
`connect() failed (111: Connection refused) ... upstream: "192.168.8.10:4434"`
in `error.log`. That means the proxy works but MCX isn't listening on that
port.
- Tests sent to the public IP from inside the LAN show the FortiGate
  (`192.168.0.1`) as the source. That's normal for traffic sent out to the
  public IP and back in from the inside, and doesn't test the WAN policy.

### Add a new HTTPS site

1. Point the DNS name at `103.174.244.121`.
2. Add the hostname to the certificate (certbot, webroot `/var/www/letsencrypt`).
3. Create `conf.d/<site>.conf` with a server block on
   `listen 127.0.0.1:10443 ssl proxy_protocol;` and the new `server_name`
   (copy `inventory.conf`).
4. Add `<hostname> web;` to the map in `stream.d/tls-443.conf`. Strictly
   optional, because `default` already sends to `web`, but it documents
   intent.
5. Test, reload, back up.

---

## 5. Backups

The `../configs/` folder is a git repository holding a snapshot of every file
above plus runtime state (`nginx -T` output, bound ports, firewall status,
certificate expiry). Run `./backup.sh` after any change; it commits only when
something differs. Restore steps are in `../configs/README.md`.

TLS private keys are intentionally not in the backup.

---

## 6. Known gaps / recommendations

1. **No reload after certificate renewal.** Certbot renews the certificate
   files, but nothing reloads this nginx, so it keeps serving the old
   certificate from memory until the next reload. Add a deploy hook before the
   next renewal (around 2026-12-08):
   ```bash
   printf '#!/bin/sh\nsystemctl reload nginx-reverseproxy\n' | sudo tee /etc/letsencrypt/renewal-hooks/deploy/reload-nginx-reverseproxy.sh
   sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx-reverseproxy.sh
   ```
2. **No log rotation.** `/var/log/nginx-reverseproxy/` isn't in
   `/etc/logrotate.d/`; `access.log` is already ~37 MB and growing. Add a
   logrotate entry whose `postrotate` runs
   `kill -USR1 $(cat /run/nginx-reverseproxy.pid)`.
3. **MCX sees the proxy IP, not the client IP**, for 443, 4434, 7755 and media (not SFTP),
   because MCX doesn't accept PROXY protocol. If MCX needs real client IPs, it
   would have to support PROXY protocol, or the proxy hop would have to be
   replaced by FortiGate DNAT directly to 192.168.8.10.

---

## 7. Change history

| Date | Change |
|---|---|
| 2026-10-09 | Earlier `nginx.conf` (http only, no stream router) saved as `nginx.conf.bak-20261009-144641` |
| 2026-10-30 | 443 SNI router added (`tls-443.conf`), PROXY protocol + real IP, catch-all site; Trinetra `:9111` site removed (`trinetra.conf.removed-20261030-154739`) |
| 2026-11-03 | MCX UDP media 61000–61500 added (`mcx-media-udp.conf`), systemd `LimitNOFILE=65536`, ufw rule for 61000–61500/udp |
| 2026-11-03 | MCX port list from the MCX team: media now goes to MCX on 50000–50500 (public 50000–50500 → proxy 61000–61500 → MCX 50000–50500 via `map`; old file at `mcx-media-udp.conf.bak-20261103-105446`). Added `mcx-tcp.conf` for TCP 4434/7755 and a ufw rule for them. SFTP 22 goes FortiGate → MCX directly |
