## Ubuntu home-lab server MAANO 

Engineer: Abdirahman Dahir

Hardware: Dell OptiPlex 5060

Date: July 2026

## Architecture Decision -- Ubuntu Server over Proxmox VE

I evaluated two platforms for this home lab: Proxmox VE and bare Ubuntu
Server 24.04 LTS

## Proxmox advantages I acknowledged

-   Full VM isolation: each service runs in its own virtual machine

-   Native snapshots: roll back any VM in seconds

-   LXC containers: lightweight alternative to full VMs

-   Better long-term scalability

### Why I chose Ubuntu Server:

Proxmox requires a Linux bridge (vmbr0) to give VMs network access. WiFi
network cards cannot be bridged due to the 802.11 protocol, it only
allows one MAC address per association. This makes Proxmox
non-functional on a WiFi-only machine. Rather than build a broken
foundation, I made a deliberate architectural decision to use bare
Ubuntu Server and document the trade-offs.

# Phase 1 -- System Health Verification

Verifying the systems health before starting using it as a home-lab.

**Findings**

  --------------------------------------------------------------------------------------------
  **Check**      **Command**                                      **Result**      **Status**
  -------------- ------------------------------------------------ --------------- ------------
  OS version     (lsb_release -a)                                 Ubuntu 24.04.4  Pass
                                                                  LTS             

  Kernel         uname -r                                         6.8.0-124       Pass
                                                                  generic         

  Root disk      df-h                                             87 GB free of   Pass
                                                                  98GB            

  RAM            free -h                                          7.6 GB total,   Pass
                                                                  7.0 GB          
                                                                  available       

  Wifi           ip a                                             <SERVER-LAN-IP> on    Pass
                                                                  wlol, state UP  

  Firewall       ufw status                                       Active          Pass

  Updating       -upgradable 2\>/dev/null \| wc -l                0 -- meaning no Pass
  status                                                          packages are    
                                                                  outstanding to  
                                                                  be upgraded     

  Network        ip route show                                    Default via     Pass
  connectivity                                                    <ROUTER-IP> points 
                                                                  to my wifi      

  Connectivity   ping <ROUTER-IP> and the cloudfare with no DNS      0% packeyt loss Pass
  test                                                                            

  Hostname I     Hostnamect1                                      Verifies        Pass
                                                                  everything      
                                                                  about the       
                                                                  server          

  SSH            systemcl status ssh                              Active now, I   Pass
                                                                  used systemctl  
                                                                  start ssh &     
                                                                  enable ssh-     
                                                                  This allows me  
                                                                  to access the   
                                                                  server remotely 

  Firewall       Sudo ufw allow ssh \| ufw enable                 Its active deny Pass
  enabling                                                        incming blocked 
                                                                  pot 22 allowed  
                                                                  where I'm       
                                                                  accessing from  
                                                                  my the server   
                                                                  through my pc   
                                                                  powershell      

  System error   Sudo journalctl -p err -b --no-pager \| head -40 SGX disabled by Pass
  logs                                                            BIOS, PAM       
                                                                  missing --      
                                                                  minor problem,  
                                                                  No disk memory  
                                                                  or service      
                                                                  failures found  

  Device health  sudo smartctl -a /dev/sda \| grep -E \'SMART     SMART overall:  Pass
  (sda)          overall\|Reallocated\|Pending\|Uncorrectable\'   passed,         
                                                                  Relocated       
                                                                  sectors 0,      
                                                                  pending sectors 
                                                                  0,              
                                                                  uncorrectable   
                                                                  errors 0        
  --------------------------------------------------------------------------------------------

# Phase 2: GitHub Repository 

**Objectivity**

Create a professional version-controlled repository that documents every
configuration, decision and lesson learned throughout this journey.

**What I've learned deeply**

**Git vs GitHub vs SSH**

-   Git is the tool installed on the server that tracks and moves files

-   GitHub is the online storage where copies are kept and employers can
    see your work

-   SSH key is the permission slip that proves to GitHub that your
    server is trusted

-   Without the SSH key GitHub blocks your server from uploading
    anything

-   Created the SSH using ed25519 algorithm

-   id_ed25519 --- private key --- never shared, never leaves the server

-   id_ed25519.pub --- public key --- given to GitHub, like handing over
    an open padlock

**Why Identity Configuration Matters**

-   Every commit is signed with your name and email

-   Employers and recruiters see your name on every change you made

-   Without it commits show as unknown --- unprofessional on a portfolio

-   init.defaultBranch main prevents branch naming conflicts between Git
    and GitHub

**Cloud Equivalent**

GitHub is equivalent to AWS CodeCommit or AZURE repos. Git push is
equivalent to a CI/CD pipeline triggering from a code change. Version
controlling infrastructure configuration is standard practice in every
professional DevOps environment.

**Commit Message Convention Learned**

-   chore: --- maintenance and setup

-   feat: --- new service or capability

-   fix: --- correcting something broken

-   docs: --- documentation updates

-   sec: --- security changes

-   

Repository created, SSH authentication working, first commit pushed
successfully. Server and GitHub are permanently connected. Ready for
Phase 3.

# Phase 3: Security Hardening the Ubuntu Server

**What I did**

  ----------------------------------------------------------------------------------
  **Action**              **Command**                        **Result**
  ----------------------- ---------------------------------- -----------------------
  Applied all updates     (apt update && apt upgrade)        Kernel updated 124134

  Auto Security Updates   (unattended-upgrades)              Runs nightly
                                                             automatically

  Installed Fail2Ban      (apt install fail2ban)             Bans Ips after 3 failed
                                                             attempts to login

  Haedened SSH config     (nano /etc/ssh/ssh_config)         Root blocked, max login
                                                             3 tries, 30s grace for
                                                             unknown IP

  Activated firewall      (ufw enable)                       Deny all incoming
                                                             except port 22

  Set Static IP           (nano                              Server permanently at
                          /etc/netplan/50-cloud-init-yaml)   the IP I set in netplan
  ----------------------------------------------------------------------------------

**What I Learned**

-   **Ports are doors --** UFW is the security guard that controls which
    doors are open

-   **Fail2ban --** bans attackers after 3 failed attempts (brute force
    becomes impossible)

-   **Static IP --** It is a mandatory for a server, DHCP changes and
    assigns new Ips that breaks every service and config

-   **SSH hardening --** blocking root and limiting attempts closes the
    most common attack vectors

-   **Cloudfare Tunnel --** Means attackers never see my IP they hit the
    cloudfare wall first

-   **Security is layered --** Cloudfare UFW Fail2ban SSH Config. Four
    walls.

**Internet attacker Cloudflare -** hides your IP, filters malicious
traffic**. UFW -** blocks everything except port 22 **Fail2Ban -** bans
after 3 failed attempts **SSH config -** root blocked, 3 try limit

**Cloud Equivalent**

UFW = AWS Security Groups, Fail2Ban = AWS WAF rate limiting, Static IP =
AWS Elastic IP, Security Auto Updates = AWS System Manager Patch Manager

# Phase 4: Docker, Portainer, Nginx, Prometheus, Grafna

**What I built**

  -------------------------------------------------------------------------
  **Sevice**              **URL**                   **Purpose**
  ----------------------- ------------------------- -----------------------
  Nginx                   <http://<SERVER-LAN-IP>>        Reverse proxy

  Portainer               <http://<SERVER-LAN-IP>:9443>   Visual Docker
                                                    management dashboard

  Prometheus              <http://<SERVER-LAN-IP>:9090>   Collects server metrics
                                                    every 15 seconds

  Grafana                 <http://<SERVER-LAN-IP>:3000>   Visualises metrics as
                                                    dashboards

  Node-exporter           Port9100                  Expose host
                                                    CPU/RAM/disk metric
  -------------------------------------------------------------------------

**What I learned**

**Docker:**

-   Containers are isolated lunchboxes, each service runs independently.
    It bundles applications code, libraries, and dependencies together.

-   (docker run) flags control networking, storage and restart behaviour

-   (\--restart=always) means services survive server reboots
    automatically

-   GPG key verifies Docker packages when downloading that they are
    genuine before installing

-   No output after a command = success in linux

**Networking:**

-   (-p 9090:9090) maps a port on the server to a port inside a
    container

-   Containers on different networks cannot reach each other by default

-   UFW blocks container-to-host communication, had to open port 9100
    explicitly

-   (host.docker.internal) is how containers reference the host machine

**Mentoring Stack:**

-   Prometheus scrapes metrics every 15 seconds from node-exporter

-   Grafana reads from Prometheus and displays visual dashboards

-   Dashboard ID 1860 is the standard Node Exporter Full community
    dashboard

-   Current server stats: CPU 0.5%, RAM 13.4%, Disk 14.8%, plenty of
    headroom

**Nginx:**

-   Reverse Porxy means one public door routes to many internal services

-   (**sudo nginx-t)** must always be run before reloading it catches
    syntax errors

-   Symlink in sites-enabled activates a config from sites-available

**Mistakes and Lessons**

-   Misspelled monitoring as monitering --- fixed with mv command

-   Prometheus could not reach node-exporter because UFW was blocking
    port 9100 --- solved by opening the port explicitly

-   Portainer timed out before admin account was created --- restarted
    container and created account immediately

**Cloud Equivalent**

  -----------------------------------------------------------------------
  **Home-Lab**                        **AWS Equivalent**
  ----------------------------------- -----------------------------------
  Docker                              EC2

  Portainer                           EC2 Console

  Prometheus                          CloudWatch metrics

  Grafana                             CloudWatch Dashboards

  Node-exporter                       CloudWatch Agent

  Nginx reverse proxy                 Application Load Balancer
  -----------------------------------------------------------------------

# Phase 5: Security Monitoring and Backups

  -----------------------------------------------------------------------
  **Components**          **Status**              **Details**
  ----------------------- ----------------------- -----------------------
  Wazuh SIEM              Complete                Running in Docker
                                                  single-node

  Wazuh Agent             Deferred                Down after patch cycle
                                                  (see final phase)

  Grafana Discord Alerts  Complete                CPU/RAM/disk alerts to
                                                  Discord

  Restic Backups          Pending                 B2 API auth issues ---
                                                  planned
  -----------------------------------------------------------------------

**Wazuh SIEM**

**What it does**

Wazuh monitors the server 24/7 for security events --- failed logins,
file integrity changes, suspicious processes, and compliance violations.

**How it is deployed**

**Docker single-node deployment**

-   Wazuh.manager -- process security events

-   Wazuh.indexer -- stores events (Opensearch)

-   wazuh.dashboard -- web UI at :433 port

**What is actively monitored**

-   SSH login attempts and failures

-   File integrity changes

-   System log analysis

-   User privilege escalation

-   Process execution monitoring

**How to check the alerts manually**

docker exec single-node-wazuh.manager-1 tail -f
/var/ossec/logs/alerts/alerts.log

or via dashboard :433 security Events

**Grafana Discord Alerts**

**What it does**

Sends real-time performance alerts to Discord when thresholds are
exceeded.

  -----------------------------------------------------------------------
  **Alert**               **Threshold**           **Channel**
  ----------------------- ----------------------- -----------------------
  High CPU                Above 80% for 5 min     Discord

  High RAM                Above 85%               Discord

  Disk Full               Above 90%               Discord
  -----------------------------------------------------------------------

**How it works step by step:**

Prometheus collects metric every 15s Grafana Evaluates alert rule every
1 minute Threshold exceeded for 5 minutes Discord webhook fires
notification.

**Restic Backups --- Deferred**

**Plan**

-   Tool: Restic (encrypted, deduplicated backup)

-   Destination: Backblaze B2 cloud storage

-   Schedule: Daily at 3 AM via cron

-   Retention: 7 daily, 4 weekly

**Why deferred**

Backblaze B2 API returned 401 Unauthorized despite correct credentials.
Issue not resolved within time budget. Deferred to post-launch phase.

**What will be backed up when implemented**

bash

restic backup \~/homelab /etc/nginx /etc/netplan

**Portfolio note**

\"Planned encrypted offsite backup strategy using restic to Backblaze B2
following the 3-2-1 backup rule: live data on server, configs in GitHub,
encrypted backups offsite.\"

**Security stack**

Internet attacker

\|

Cloudflare --- filters malicious traffic (Phase 6)

\|

UFW --- blocks all ports except allowed

\|

Fail2Ban --- bans after 3 failed attempts

\|

SSH config --- root blocked, 3 try limit

\|

Wazuh SIEM --- monitors everything internally

\|

Grafana --- alerts you on Discord if something is wrong

**What I Learned**

-   **SIEM** stands for Security Information and Event Management ---
    centralises all security events in one place

-   Wazuh has thousands of built-in detection rules --- no manual rule
    creation needed for common threats

-   Running security tools in Docker has limitations --- some processes
    need direct OS access

-   Version matching between agent and manager is mandatory in Wazuh

-   Grafana alerting is a practical alternative to SIEM-based
    notifications for performance monitoring

-   Security is layered --- no single tool protects everything

# Phase 6: Domain, Cloudflare Tunnel and Public Access

Only after the server was hardened (Phase 3), running core services
(Phase 4), and monitored (Phase 5) did I connect it to the internet.
This ordering is deliberate: nothing goes public until it is safe to
expose. The site is served from my own server at home --- the internet
never reaches that server directly.

+----------------------+-----------------------+-----------------------+
| Component            | Status                | Details               |
+======================+=======================+=======================+
| Domain (Cloudflare   | Complete              | dahirabdirahman.com   |
| Registrar)           |                       | --- Cloudflare        |
|                      |                       | nameservers, WHOIS    |
|                      |                       | privacy               |
+----------------------+-----------------------+-----------------------+
| Portfolio Site       | Complete              | Static HTML/CSS/JS    |
| (Nginx)              |                       | served on             |
|                      |                       | localhost:80          |
+----------------------+-----------------------+-----------------------+
| Git Deploy Workflow  | Complete              | Private repo +        |
|                      |                       | read-only deploy key, |
|                      |                       | updates via git pull  |
+----------------------+-----------------------+-----------------------+
| Cloudflare Tunnel    | Complete              | Outbound-only,        |
|                      |                       | systemd service,      |
|                      |                       | auto-starts on reboot |
+----------------------+-----------------------+-----------------------+
| DNS Routing          | Complete              | CNAME → tunnel; home  |
|                      |                       | IP never exposed      |
+----------------------+-----------------------+-----------------------+
| Cloudflare Edge      | Complete              | Always Use HTTPS, SSL |
| Security             |                       | Full, WAF Managed     |
|                      |                       | Rules, Bot Fight Mode |
+----------------------+-----------------------+-----------------------+
| Reboot Recovery Test | Complete              | cloudflared + nginx   |
|                      |                       | confirmed to          |
|                      |                       | auto-start unattended |
+----------------------+-----------------------+-----------------------+
| Real-IP Restoration  | Pending               | Nginx logs show       |
|                      |                       | 127.0.0.1 (tunnel     |
|                      |                       | loopback) --- planned |
+----------------------+-----------------------+-----------------------+
| UFW Cleanup          | Pending               | Remove public 80/443, |
|                      |                       | scope 9100 to LAN --- |
|                      |                       | planned               |
+----------------------+-----------------------+-----------------------+
| Nginx Security       | Pending               | server_tokens off +   |
| Headers              |                       | security headers ---  |
|                      |                       | planned               |
+----------------------+-----------------------+-----------------------+
| Wazuh Restoration    | Pending               |                       |
|                      |                       |  -------------------- |
|                      |                       |      wazuh-db daemon  |
|                      |                       |      down after patch |
|                      |                       |                       |
|                      |                       |     cycle --- planned |
|                      |                       |                       |
|                      |                       |  -- ----------------- |
|                      |                       |                       |
|                      |                       |                       |
|                      |                       |  -------------------- |
+----------------------+-----------------------+-----------------------+

## Domain --- Cloudflare Registrar

**What it does**

Registers dahirabdirahman.com and provides authoritative DNS. My name is
the domain, so one registration acts as an umbrella --- the portfolio
lives on the root, and future sites can live on subdomains (e.g.
lab.dahirabdirahman.com) at no extra cost.

**How it is deployed**

-   Registered directly through Cloudflare Registrar (at-cost pricing,
    free WHOIS privacy)

-   Domain automatically uses Cloudflare nameservers --- no nameserver
    change, no propagation wait

-   Appears immediately as an active zone, so Tunnel DNS routing works
    instantly

**Why Cloudflare Registrar over another registrar**

The whole stack lives in Cloudflare (Tunnel, WAF, DNS). Registering here
means the domain is on Cloudflare nameservers from the first minute,
eliminating the manual nameserver-repointing step a third-party
registrar would require.

## Portfolio Site --- Nginx (system service)

**What it does**

Serves the static portfolio (HTML/CSS/JS + assets) to visitors. Nginx
also acts as a reverse proxy / front desk --- it receives all traffic on
port 80 and will route future sites to the correct location by hostname.

**How it is deployed**

-   Nginx runs as a **host system service** (apt-installed), not in
    Docker

-   Document root: /home/\<user\>/portfolio

-   Site config: /etc/nginx/sites-available/portfolio (symlinked into
    sites-enabled)

-   The default site was removed so the portfolio config owns port 80

server {

listen 80 default_server;

listen \[::\]:80 default_server;

root /home/\<user\>/portfolio;

index index.html;

server_name dahirabdirahman.com www.dahirabdirahman.com;

location / {

try_files \$uri \$uri/ =404;

}

Nginx runs as user www-data, which cannot traverse a locked-down home
directory by default. Fixed with least-privilege traversal, not blanket
read:

chmod o+x /home/\<user\> \# allow traverse into the path

chmod -R o+rX /home/\<user\>/portfolio \# read files, traverse dirs only

**How to verify it is serving**

curl -s http://localhost:80 \| head -n 5 \# should show real HTML

curl -s -o /dev/null -w \"%{http_code}\\n\" http://localhost/style.css
\# should return 200

Setting the document root once means CSS, JS, and everything in assets/
are served automatically by relative path --- no per-file configuration.

## Cloudflare Tunnel --- the public access mechanism

**What it does**

Creates an outbound-only encrypted connection from the server to
Cloudflare\'s edge. No inbound router ports are opened, and the home IP
is never exposed. Visitors reach Cloudflare; Cloudflare passes the
request back down the tunnel to Nginx.

**How it is deployed**

-   cloudflared installed and run as a **systemd service** (auto-starts
    on boot)

-   Tunnel name: homelab (UUID \<TUNNEL-UUID\>)

-   Config and credentials moved to the system path /etc/cloudflared/ so
    the root-run service can find them

-   Four redundant QUIC connections to Cloudflare edges (Montreal /
    Toronto)

tunnel: \<TUNNEL-UUID\>

credentials-file: /etc/cloudflared/\<TUNNEL-UUID\>.json

ingress:

\- hostname: dahirabdirahman.com

service: http://localhost:80

\- hostname: www.dahirabdirahman.com

service: http://localhost:80

\- service: http_status:404

**The security rule of the ingress list**

Only hostnames listed in ingress are reachable from the internet.
Grafana, Portainer, Wazuh, and SSH are **not** listed, so they are
invisible externally. This is by design, not an oversight. Any internal
tool exposed in future goes behind Cloudflare Access (authentication)
--- never raw in the ingress list.

**DNS routing**

cloudflared tunnel route dns homelab dahirabdirahman.com

cloudflared tunnel route dns homelab
[www.dahirabdirahman.com](http://www.dahirabdirahman.com)

The domain resolves to a Cloudflare IP (e.g. 172.64.80.1). The home IP
appears nowhere in public DNS.

**How to verify the tunnel**

sudo systemctl is-active cloudflared \# active

cloudflared tunnel info homelab \# shows registered connections

## Cloudflare Edge Security

**What it does**

All visitor traffic hits Cloudflare first, so protection is applied at
the edge before a request reaches the server.

  -----------------------------------------------------------------------
  Setting                 Status                  Purpose
  ----------------------- ----------------------- -----------------------
  Always Use HTTPS        ON                      Redirects all http →
                                                  https

  SSL/TLS mode            FULL                    Encrypted
                                                  edge-to-origin (no
                                                  insecure leg)

  WAF Managed Rules       ON                      OWASP-style protection
                                                  (injection, XSS,
                                                  exploits)

  Bot Fight Mode          ON                      Challenges automated
                                                  bot traffic at the edge
  -----------------------------------------------------------------------

## Security stack

Internet visitor / attacker

\|

Cloudflare edge --- WAF, Bot Fight Mode, HTTPS, DDoS absorption (Phase
7)

\|

Cloudflare Tunnel --- outbound-only, encrypted; NO open inbound ports;
home IP hidden

\|

Nginx --- serves the static portfolio on localhost:80

\|

UFW --- deny-by-default firewall

\|

Fail2Ban --- bans after 3 failed SSH attempts

\|

SSH config --- root blocked, 3-try limit, hardening in drop-in file

\|

Wazuh SIEM --- internal monitoring (currently deferred --- see status
table)

\|

Grafana --- Discord performance alerts

## What I learned

-   **Cloudflare Tunnel is outbound-only** --- the server dials out, so
    no port forwarding and no exposed home IP. This is what let me
    expose the site without weakening the network.

-   **A document root is set once** --- point Nginx at the folder and
    relative paths serve CSS, JS, and assets automatically; you don\'t
    configure each file.

-   **System services vs. Docker** --- Nginx runs as a host service, not
    a container. A root-run service (sudo cloudflared service install)
    cannot read a user\'s home directory, so its config must live in
    /etc.

-   **Config drift from copy-paste is real** --- pasting from formatted
    text mangled hostnames (Markdown link syntax leaked into YAML).
    Hand-typing config files avoids it.

-   **Pre-flight checks catch silent regressions** --- the SSH hardening
    reverting on apt upgrade would have gone unnoticed without a health
    check before going public.

-   **Preventive vs. detective controls** --- preventive controls
    (firewall, Fail2Ban, SSH, tunnel) block attacks; detective controls
    (Wazuh) only observe. The site is safe to run with detection
    temporarily down because the preventive layer is intact --- a
    documented, deliberate trade-off.

-   **A static site is a tiny attack surface** --- the flood of
    /wp-admin bot probes all return 404, because there is no server-side
    code, database, or WordPress to exploit.
