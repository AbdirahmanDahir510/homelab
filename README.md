# Ubuntu Home-Lab Server

**Engineer:** Abdirahman Dahir
**Hardware:** Dell OptiPlex 5060
**Live site:** [dahirabdirahman.com](https://dahirabdirahman.com)
**Started:** July 2026

A self-hosted infrastructure project: a hardened, monitored Ubuntu server running a containerized service stack, exposed to the internet through a Cloudflare Tunnel — built in deliberate phases, with nothing going public until it was safe to expose.

---

## Architecture Decision — Ubuntu Server over Proxmox VE

I evaluated two platforms for this home lab: Proxmox VE and bare Ubuntu Server 24.04 LTS.

**Proxmox advantages I acknowledged:**

- Full VM isolation — each service runs in its own virtual machine
- Native snapshots — roll back any VM in seconds
- LXC containers — lightweight alternative to full VMs
- Better long-term scalability

**Why I chose Ubuntu Server:**

Proxmox requires a Linux bridge (`vmbr0`) to give VMs network access. WiFi network cards cannot be bridged, because the 802.11 protocol allows only one MAC address per association. This makes Proxmox non-functional on a WiFi-only machine. Rather than build on a broken foundation, I made a deliberate architectural decision to use bare Ubuntu Server and document the trade-off.

---

## Phase 1 — System Health Verification

Verifying the system's health before using it as a home lab.

| Check | Command | Result | Status |
|---|---|---|---|
| OS version | `lsb_release -a` | Ubuntu 24.04.4 LTS | Pass |
| Kernel | `uname -r` | 6.8.0-124-generic | Pass |
| Root disk | `df -h` | 87 GB free of 98 GB | Pass |
| RAM | `free -h` | 7.6 GB total, 7.0 GB available | Pass |
| WiFi | `ip a` | `<server-ip>` on `wlo1`, state UP | Pass |
| Firewall | `ufw status` | Active | Pass |
| Update status | `apt list --upgradable 2>/dev/null \| wc -l` | 0 — no packages outstanding | Pass |
| Network connectivity | `ip route show` | Default via `<router-ip>` (WiFi) | Pass |
| Connectivity test | `ping <router-ip>` | 0% packet loss | Pass |
| Hostname | `hostnamectl` | Verified | Pass |
| SSH | `systemctl status ssh` | Active (enabled for remote access) | Pass |
| Firewall enabling | `ufw allow ssh && ufw enable` | Active, deny incoming, port 22 allowed | Pass |
| System error logs | `journalctl -p err -b --no-pager \| head -40` | Only minor notices (SGX disabled by BIOS); no disk or service failures | Pass |
| Device health (sda) | `smartctl -a /dev/sda` | SMART passed; 0 reallocated / pending / uncorrectable | Pass |

---

## Phase 2 — GitHub Repository

**Objective:** create a professional, version-controlled repository that documents every configuration, decision, and lesson learned throughout this project.

**Git vs GitHub vs SSH:**

- **Git** is the tool on the server that tracks and moves files.
- **GitHub** is the online storage where copies are kept and employers can see the work.
- The **SSH key** is the permission slip that proves to GitHub the server is trusted — without it, GitHub blocks the server from uploading anything.
- Created the key using the `ed25519` algorithm.
- `id_ed25519` — private key — never shared, never leaves the server.
- `id_ed25519.pub` — public key — given to GitHub, like handing over an open padlock.

**Why identity configuration matters:**

- Every commit is signed with your name and email.
- Recruiters see your name on every change you made.
- Without it, commits show as `unknown` — unprofessional on a portfolio.
- `init.defaultBranch main` prevents branch-naming conflicts between Git and GitHub.

**Commit message convention:**

| Prefix | Meaning |
|---|---|
| `chore:` | maintenance and setup |
| `feat:` | new service or capability |
| `fix:` | correcting something broken |
| `docs:` | documentation updates |
| `sec:` | security changes |

**Cloud equivalent:** GitHub ≈ AWS CodeCommit / Azure Repos. `git push` ≈ a CI/CD pipeline triggering from a code change. Version-controlling infrastructure configuration is standard practice in professional DevOps environments.

---

## Phase 3 — Security Hardening

| Action | Command | Result |
|---|---|---|
| Applied all updates | `apt update && apt upgrade` | Kernel updated |
| Auto security updates | `unattended-upgrades` | Runs nightly automatically |
| Installed Fail2Ban | `apt install fail2ban` | Bans IPs after 3 failed login attempts |
| Hardened SSH config | drop-in in `/etc/ssh/sshd_config.d/` | Root blocked, max 3 tries, grace window for unknown IPs |
| Activated firewall | `ufw enable` | Deny all incoming except port 22 |
| Set static IP | `/etc/netplan/*.yaml` | Server permanently at the IP set in Netplan |

**What I learned:**

- **Ports are doors** — UFW is the security guard controlling which doors are open.
- **Fail2Ban** — bans attackers after 3 failed attempts, making brute force impractical.
- **Static IP** — mandatory for a server; DHCP reassigns IPs and breaks every service and config.
- **SSH hardening** — blocking root and limiting attempts closes the most common attack vectors.
- **Security is layered** — Cloudflare, UFW, Fail2Ban, SSH config: multiple independent walls.

**Cloud equivalent:** UFW ≈ AWS Security Groups · Fail2Ban ≈ AWS WAF rate limiting · Static IP ≈ AWS Elastic IP · Auto updates ≈ AWS Systems Manager Patch Manager.

---

## Phase 4 — Docker, Portainer, Nginx, Prometheus, Grafana

**What I built:**

| Service | URL | Purpose |
|---|---|---|
| Nginx | `http://<server-ip>` | Reverse proxy / web server |
| Portainer | `http://<server-ip>:9443` | Visual Docker management dashboard |
| Prometheus | `http://<server-ip>:9090` | Collects server metrics every 15 seconds |
| Grafana | `http://<server-ip>:3000` | Visualizes metrics as dashboards |
| Node-exporter | port `9100` | Exposes host CPU/RAM/disk metrics |

**Docker:**

- Containers are isolated bundles — each service runs independently with its own code, libraries, and dependencies.
- `docker run` flags control networking, storage, and restart behavior.
- `--restart=always` means services survive server reboots automatically.
- GPG key verification confirms Docker packages are genuine before installing.

**Networking:**

- `-p 9090:9090` maps a port on the server to a port inside a container.
- Containers on different networks cannot reach each other by default.
- UFW blocks container-to-host communication — had to open port `9100` explicitly.
- `host.docker.internal` is how a container references the host machine.

**Monitoring stack:**

- Prometheus scrapes metrics every 15 seconds from node-exporter.
- Grafana reads from Prometheus and displays visual dashboards.
- Dashboard ID `1860` is the standard *Node Exporter Full* community dashboard.

**Nginx:**

- A reverse proxy means one public door routes to many internal services.
- `sudo nginx -t` must always be run before reloading — it catches syntax errors.
- A symlink in `sites-enabled` activates a config from `sites-available`.

**Mistakes and lessons:**

- Misspelled a directory (`monitering`) — fixed with `mv`.
- Prometheus couldn't reach node-exporter because UFW was blocking port `9100` — solved by opening the port explicitly.
- Portainer timed out before the admin account was created — restarted the container and created the account immediately.

**Cloud equivalent:**

| Home-Lab | AWS Equivalent |
|---|---|
| Docker | EC2 |
| Portainer | EC2 Console |
| Prometheus | CloudWatch Metrics |
| Grafana | CloudWatch Dashboards |
| Node-exporter | CloudWatch Agent |
| Nginx reverse proxy | Application Load Balancer |

---

## Phase 5 — Security Monitoring and Backups

| Component | Status | Details |
|---|---|---|
| Wazuh SIEM | Complete | Running in Docker (single-node) |
| Wazuh Agent | Deferred | Down after patch cycle (see Phase 6) |
| Grafana Discord Alerts | Complete | CPU/RAM/disk alerts to Discord |
| Restic Backups | Pending | Backblaze B2 API auth issue — planned |

### Wazuh SIEM

**What it does:** monitors the server 24/7 for security events — failed logins, file-integrity changes, suspicious processes, and compliance violations.

**How it is deployed** (Docker single-node):

- `wazuh.manager` — processes security events
- `wazuh.indexer` — stores events (OpenSearch)
- `wazuh.dashboard` — web UI

**What is actively monitored:**

- SSH login attempts and failures
- File-integrity changes
- System log analysis
- User privilege escalation
- Process-execution monitoring

**How to check alerts manually:**

```bash
docker exec single-node-wazuh.manager-1 tail -f /var/ossec/logs/alerts/alerts.log
```

### Grafana Discord Alerts

Sends real-time performance alerts to Discord when thresholds are exceeded.

| Alert | Threshold | Channel |
|---|---|---|
| High CPU | above 80% for 5 min | Discord |
| High RAM | above 85% | Discord |
| Disk full | above 90% | Discord |

**How it works:** Prometheus collects metrics every 15s → Grafana evaluates alert rules every 1 min → threshold exceeded for 5 min → Discord webhook fires the notification.

### Restic Backups — Deferred

**Plan:**

- **Tool:** Restic (encrypted, deduplicated backup)
- **Destination:** Backblaze B2 cloud storage
- **Schedule:** daily at 3 AM via cron
- **Retention:** 7 daily, 4 weekly

**Why deferred:** Backblaze B2 API returned `401 Unauthorized` despite correct credentials; not resolved within the time budget. Deferred to a post-launch phase.

**What will be backed up:**

```bash
restic backup ~/homelab /etc/nginx /etc/netplan
```

Planned as a 3-2-1 backup strategy: live data on the server, configs in GitHub, encrypted backups offsite.

**What I learned:**

- **SIEM** = Security Information and Event Management — centralizes all security events in one place.
- Wazuh ships thousands of built-in detection rules — no manual rule creation needed for common threats.
- Running security tools in Docker has limitations — some processes need direct OS access.
- Version matching between agent and manager is mandatory in Wazuh.
- Grafana alerting is a practical alternative to SIEM-based notifications for performance monitoring.
- Security is layered — no single tool protects everything.

---

## Phase 6 — Domain, Cloudflare Tunnel, and Public Access

Only after the server was hardened (Phase 3), running core services (Phase 4), and monitored (Phase 5) did I connect it to the internet. This ordering is deliberate: nothing goes public until it is safe to expose. The site is served from my own server at home — the internet never reaches that server directly.

| Component | Status | Details |
|---|---|---|
| Domain (Cloudflare Registrar) | Complete | `dahirabdirahman.com` — Cloudflare nameservers, WHOIS privacy |
| Portfolio Site (Nginx) | Complete | Static HTML/CSS/JS served on `localhost:80` |
| Git Deploy Workflow | Complete | Private repo + read-only deploy key, updates via `git pull` |
| Cloudflare Tunnel | Complete | Outbound-only, systemd service, auto-starts on reboot |
| DNS Routing | Complete | CNAME → tunnel; home IP never exposed |
| Cloudflare Edge Security | Complete | Always Use HTTPS, SSL Full, WAF Managed Rules, Bot Fight Mode |
| Reboot Recovery Test | Complete | cloudflared + nginx confirmed to auto-start unattended |
| Real-IP Restoration | Pending | Nginx logs show `127.0.0.1` (tunnel loopback) — planned |
| UFW Cleanup | Pending | Remove public 80/443, scope 9100 to LAN — planned |
| Nginx Security Headers | Pending | `server_tokens off` + security headers — planned |
| Wazuh Restoration | Pending | `wazuh-db` daemon down after patch cycle — planned |

### Domain — Cloudflare Registrar

**What it does:** registers `dahirabdirahman.com` and provides authoritative DNS. My name is the domain, so one registration acts as an umbrella — the portfolio lives on the root, and future sites can live on subdomains (e.g. `lab.dahirabdirahman.com`) at no extra cost.

**How it is deployed:**

- Registered directly through Cloudflare Registrar (at-cost pricing, free WHOIS privacy).
- Domain automatically uses Cloudflare nameservers — no nameserver change, no propagation wait.
- Appears immediately as an active zone, so Tunnel DNS routing works instantly.

**Why Cloudflare Registrar:** the whole stack lives in Cloudflare (Tunnel, WAF, DNS). Registering here puts the domain on Cloudflare nameservers from the first minute, eliminating the manual nameserver-repointing step a third-party registrar would require.

### Portfolio Site — Nginx (system service)

**What it does:** serves the static portfolio (HTML/CSS/JS + assets). Nginx also acts as a reverse proxy — it receives all traffic on port 80 and routes each hostname to the correct site.

**How it is deployed:**

- Nginx runs as a **host system service** (apt-installed), not in Docker.
- Document root: `/home/<user>/portfolio`
- Site config: `/etc/nginx/sites-available/portfolio` (symlinked into `sites-enabled`).
- The default site was removed so the portfolio config owns port 80.

```nginx
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    root /home/<user>/portfolio;
    index index.html;
    server_name dahirabdirahman.com www.dahirabdirahman.com;
    location / {
        try_files $uri $uri/ =404;
    }
}
```

**The home-directory permissions gotcha:** Nginx runs as user `www-data`, which can't traverse a locked-down home directory by default. Fixed with least-privilege traversal, not blanket read:

```bash
chmod o+x /home/<user>                 # allow traverse into the path
chmod -R o+rX /home/<user>/portfolio   # read files, traverse dirs only
```

**How to verify it is serving:**

```bash
curl -s http://localhost:80 | head -n 5                              # should show real HTML
curl -s -o /dev/null -w "%{http_code}\n" http://localhost/style.css  # should return 200
```

Setting the document root once means CSS, JS, and everything in `assets/` are served automatically by relative path — no per-file configuration.

### Git Deploy Workflow

- Portfolio kept in its own **private** repo (the contact form uses a client-side Web3Forms key — public by design, but a private repo is correct default hygiene).
- The server authenticates with a dedicated **read-only deploy key**, added to the repo with write access disabled — least privilege.
- Update workflow: `edit on laptop → git push → (on server) git pull → live`.

### Cloudflare Tunnel — the public access mechanism

**What it does:** creates an outbound-only encrypted connection from the server to Cloudflare's edge. No inbound router ports are opened, and the home IP is never exposed. Visitors reach Cloudflare; Cloudflare passes the request back down the tunnel to Nginx.

**How it is deployed:**

- `cloudflared` installed and run as a **systemd service** (auto-starts on boot).
- Tunnel name: `homelab` (UUID stored locally, not published).
- Config and credentials live in `/etc/cloudflared/` so the root-run service can find them.
- Four redundant QUIC connections to Cloudflare edges (Montreal / Toronto).

```yaml
tunnel: <TUNNEL-UUID>
credentials-file: /etc/cloudflared/<TUNNEL-UUID>.json

ingress:
  - hostname: dahirabdirahman.com
    service: http://localhost:80
  - hostname: www.dahirabdirahman.com
    service: http://localhost:80
  - service: http_status:404
```

**The security rule of the ingress list:** only hostnames listed in `ingress` are reachable from the internet. Grafana, Portainer, Wazuh, and SSH are **not** listed, so they are invisible externally — by design. Any internal tool exposed in future goes behind Cloudflare Access (authentication), never raw in the ingress list.

**DNS routing:**

```bash
cloudflared tunnel route dns homelab dahirabdirahman.com
cloudflared tunnel route dns homelab www.dahirabdirahman.com
```

The domain resolves to a Cloudflare IP; the home IP appears nowhere in public DNS.

**How to verify the tunnel:**

```bash
sudo systemctl is-active cloudflared   # active
cloudflared tunnel info homelab        # shows registered connections
```

### Cloudflare Edge Security

All visitor traffic hits Cloudflare first, so protection is applied at the edge before a request reaches the server.

| Setting | Status | Purpose |
|---|---|---|
| Always Use HTTPS | On | Redirects all `http` → `https` |
| SSL/TLS mode | Full | Encrypted edge-to-origin (no insecure leg) |
| WAF Managed Rules | On | OWASP-style protection (injection, XSS, exploits) |
| Bot Fight Mode | On | Challenges automated bot traffic at the edge |

---

## Security Stack (full architecture)

```
Internet visitor / attacker
        │
Cloudflare edge — WAF, Bot Fight Mode, HTTPS, DDoS absorption
        │
Cloudflare Tunnel — outbound-only, encrypted; NO open inbound ports; home IP hidden
        │
Nginx — serves the static portfolio on localhost:80
        │
UFW — deny-by-default firewall
        │
Fail2Ban — bans after 3 failed SSH attempts
        │
SSH config — root blocked, 3-try limit, hardening in drop-in file
        │
Wazuh SIEM — internal monitoring (currently deferred — see status table)
        │
Grafana — Discord performance alerts
```

---

## What I Learned (Phase 6)

- **Cloudflare Tunnel is outbound-only** — the server dials out, so no port forwarding and no exposed home IP. This is what let me expose the site without weakening the network.
- **A document root is set once** — point Nginx at the folder and relative paths serve CSS, JS, and assets automatically.
- **System services vs. Docker** — Nginx runs as a host service, not a container. A root-run service (`sudo cloudflared service install`) cannot read a user's home directory, so its config must live in `/etc`.
- **Config drift from copy-paste is real** — pasting from formatted text mangled hostnames (Markdown link syntax leaked into YAML). Hand-typing config files avoids it.
- **Pre-flight checks catch silent regressions** — an `apt upgrade` reverted the SSH hardening; a health check before going public caught it.
- **Preventive vs. detective controls** — preventive controls (firewall, Fail2Ban, SSH, tunnel) block attacks; detective controls (Wazuh) only observe. The site is safe to run with detection temporarily down because the preventive layer is intact — a documented, deliberate trade-off.
- **A static site is a tiny attack surface** — the flood of `/wp-admin` bot probes all return 404, because there is no server-side code, database, or WordPress to exploit.
