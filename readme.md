# Enterprise Lab simulation:

**Author:** CBDEVSEC
**Date:** 2025-10-03
**Purpose:** Technical documentation for a small enterprise lab built on VMware Workstation. This repository documents how to build, configure, test, and harden a lab network consisting of:

- pfSense (edge firewall/router)
- Ubuntu Server (GLPI ITSM/inventory + Zabbix monitoring)
- Windows Host (client workstation)

This README is written as a runbook and technical reference intended for GitHub. It includes exact commands, configuration files, rationale for each step, validation tests, and troubleshooting notes.



## Table of Contents
1. Overview & Topology
2. Prerequisites (hardware/software)
3. VMware virtual network setup (VMnet)
4. pfSense — VM creation & full configuration
5. Ubuntu Server — VM creation & services (Apache, MariaDB, GLPI, Zabbix)
6. Windows Host — network & test configuration
7. Firewall rules & security policies (pfSense)
8. Agents (Zabbix & GLPI) installation and registration
9. Tests & validation plan
10. Troubleshooting common issues
11. Backup, snapshots, and recovery
12. Hardening checklist
13. Automation snippets & useful scripts
14. Appendix: config files and snippets



## 1. Overview & Topology

This lab simulates a small enterprise network. It is intentionally compact (to run on machines with limited RAM) but maintains realistic separation of responsibilities.


                      Internet
                         │
                     (VMnet0 / Bridged)
                         │
                      pfSense (WAN)
                     /    |    \
          VMnet2 (Users) | VMnet3 (Servers)
             192.168.1.0/24    192.168.1.0/24
             [Windows Host]    [Ubuntu: GLPI + Zabbix]
```

**IP plan used in this documentation (example):**

- pfSense WAN: DHCP on VMnet0 (bridged) or host NAT IP
- pfSense LAN (Users): `192.168.1.1/24` (gateway for Windows Host)
- Ubuntu server (GLPI/Zabbix): `192.168.1.101/24`
- Windows Host: `192.168.1.100/24`

> Note: In this lab we place all VMs on the same subnet for simplicity (`192.168.1.0/24`). In a more complex build you would separate Users/Servers into different subnets and use VLANs.



## 2. Prerequisites

### Hardware
- Host machine(s) with VMware Workstation (or Player) installed.
- At least 8 GB RAM on host; recommended 16+ GB if you want to run more VMs concurrently.
- CPU with virtualization support (Intel VT-x / AMD‑V).
- SSD for fast VM performance.

### Software & ISOs
- **VMware Workstation Pro/Player** (latest stable)
- **pfSense** ISO: https://www.pfsense.org/download/
- **Ubuntu Server 22.04 LTS** ISO: https://ubuntu.com/download/server
- **Windows 10/11 ISO** (for the Windows host) — use official Microsoft download pages.

### Network knowledge expected
- Familiarity with IPv4 addressing, default gateways, and basic TCP/UDP ports.
- Basic Linux command line and systemd service management.
- Basic pfSense GUI familiarity (Interfaces, Firewall rules, DHCP, NAT).



## 3. VMware virtual network setup (VMnet)

**Goal:** Create virtual networks so VMs can be placed on controlled segments that mimic real networks.

### Create networks
1. Open **VMware Workstation** as Administrator.  
2. Edit → **Virtual Network Editor**.  
3. Ensure **VMnet0** (Bridged) exists. This will be used as WAN (connects pfSense WAN to real LAN/Internet).  
4. Create **VMnet2** (Host-only or Custom) — label `LAN`. This is where Ubuntu and Windows will live.

**Rationale:** Using Host-only for LAN isolates lab traffic from your real network if desired. Bridged VMnet0 allows pfSense WAN to access internet.

**VM network adapter assignment (summary):**
- pfSense: NIC1 -> VMnet0 (WAN/Bridged), NIC2 -> VMnet2 (LAN)
- Ubuntu Server: NIC -> VMnet2
- Windows Host: NIC -> VMnet2

**Tip:** Use `Replicate physical network connection state` for bridged NIC to avoid issues when switching physical connections.



## 4. pfSense — VM creation & full configuration

### 4.1 VM hardware
- vCPU: 2
- RAM: 1.5–2 GB (2 GB recommended)
- Disk: 10–20 GB
- NICs: 2 (VMnet0 = WAN, VMnet2 = LAN)
- Boot from pfSense ISO

### 4.2 Installation steps (console)
1. Boot the VM from pfSense ISO.  
2. Accept default at the installer menus; use `Auto (UFS)` partitioning for simplicity.  
3. When installation completes, reboot and remove ISO.

### 4.3 Initial interface assignment
After first boot pfSense will allocate interfaces (e.g., `em0`, `em1`). Use the console to assign:
- `em0` → WAN (VMnet0)
- `em1` → LAN (VMnet2)

If the console asks to configure VLANs, skip for now (unless you are using VLANs).

### 4.4 Web GUI initial configuration
From a machine on the LAN (Windows Host): point browser to `https://192.168.1.1` (default pfSense LAN IP). Use the web configurator wizard:

- Hostname: `pfsense`
- Domain: `lab.local` (or `corp.lab`)
- DNS Servers: set to public DNS (8.8.8.8) initially, or to the Ubuntu server if it will run DNS later.
- Time Zone: set correctly
- WAN: DHCP (recommended) or set static if you prefer a controlled WAN IP
- LAN IP: `192.168.1.1/24`
- Set a secure admin password

**Why:** The web GUI simplifies configuring DHCP, firewall, NAT, and packages. The wizard sets safe defaults so you can start.

### 4.5 Configure DHCP (LAN)
- Services → DHCP Server → LAN
  - Enable DHCP server on LAN
  - Range: `192.168.1.100` – `192.168.1.200` (reserve static IPs outside this range)
  - DNS server option (if using Ubuntu as DNS set it here)

**Why:** DHCP allows easy IP assignment for Windows host and other client VMs. Static services (DB, web) should be set with static IPs.

### 4.6 NAT & Firewall basics
- NAT: Automatic outbound NAT is fine for lab networks.
- Firewall (LAN rules): By default pfSense allows any outbound from LAN. For better practice, create specific rules:
  1. Allow LAN to DNS (TCP/UDP 53) to your DNS server
  2. Allow LAN to Web servers (TCP 80/443) to `192.168.1.101` (Ubuntu)
  3. Allow LAN to WAN (general web browsing)
  4. Deny other inbound to servers unless explicitly required

**Why:** Explicit rules reduce attack surface and emulate enterprise policies.

### 4.7 Optional packages
- **Suricata** (IDS/IPS) — install and enable for network intrusion detection.
- **OpenVPN** — for secure remote admin access if you need to access the lab from outside your host.

### 4.8 Backup & config export
- Diagnostics → Backup & Restore → Download configuration XML.
- Save this file in your Git repo for reproducibility.



## 5. Ubuntu Server — VM creation & services (Apache, MariaDB, GLPI, Zabbix)

### 5.1 VM hardware
- vCPU: 2
- RAM: 2–4 GB (2 GB minimal)
- Disk: 20–40 GB
- NIC: VMnet2 (LAN)
- Hostname: `mon-glpi` or `ubuntu-mon` (choose consistent naming)

### 5.2 Install Ubuntu Server
- Boot from Ubuntu Server 22.04 LTS ISO and install with OpenSSH selected.
- Set a user account that has sudo privileges.

### 5.3 Network: static IP with netplan
Create or edit `/etc/netplan/01-netcfg.yaml` to set static IP:

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: no
      addresses: [192.168.1.101/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8,1.1.1.1]
```
Apply:
```bash
sudo netplan apply
```

**Why static?** Services (GLPI, Zabbix) require stable addressing. You can also configure DHCP reservation on pfSense instead.

### 5.4 Install LAMP components (Apache + MariaDB + PHP)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y apache2 mariadb-server php php-mysql libapache2-mod-php php-cli \
  php-xml php-gd php-curl php-ldap php-mbstring unzip
sudo systemctl enable --now apache2 mariadb
```

#### Secure MariaDB
Run interactive hardening:
```bash
sudo mysql_secure_installation
```
Follow prompts: set root password, remove anonymous users, disable remote root, remove test DB, reload privilege tables.

**Why:** Default MariaDB is permissive for ease of install; `mysql_secure_installation` is a minimum security step.

### 5.5 Create application databases & users

```sql
# login to MariaDB
sudo mariadb -u root -p

-- GLPI DB
CREATE DATABASE glpi CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'glpiuser'@'localhost' IDENTIFIED BY 'GlpiStrongP@ss!';
GRANT ALL PRIVILEGES ON glpi.* TO 'glpiuser'@'localhost';

-- Zabbix DB
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'ZabbixStrongP@ss!';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';

FLUSH PRIVILEGES;
EXIT;
```

**Why separate DB users?** Principle of least privilege — each app uses credentials limited to its own schema.

### 5.6 Install GLPI (latest stable)

```bash
cd /tmp
# check latest release URL at https://github.com/glpi-project/glpi/releases
wget https://github.com/glpi-project/glpi/releases/download/10.0.16/glpi-10.0.16.tgz
sudo tar -xzf glpi-*.tgz -C /var/www/html/
sudo chown -R www-data:www-data /var/www/html/glpi
sudo chmod -R 755 /var/www/html/glpi
```

Create Apache site configuration `/etc/apache2/sites-available/glpi.conf`:

```apache
<VirtualHost *:80>
  ServerName glpi.lab.local
  DocumentRoot /var/www/html/glpi
  <Directory /var/www/html/glpi>
    Require all granted
    AllowOverride All
  </Directory>
  ErrorLog ${APACHE_LOG_DIR}/glpi-error.log
  CustomLog ${APACHE_LOG_DIR}/glpi-access.log combined
</VirtualHost>
```

Enable site and rewrite:
```bash
sudo a2ensite glpi.conf
sudo a2enmod rewrite
sudo systemctl reload apache2
```

Open the web installer at `http://192.168.1.101/` and follow prompts: supply DB `glpi` and `glpiuser` credentials.

**Why Apache virtual host?** Keeps GLPI files isolated, allows hostnames and later TLS config.

### 5.7 Install Zabbix 7.0 (server + frontend + agent)

Commands (Ubuntu 22.04):

```bash
# add zabbix repo (adjust version if needed)
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_7.0-2+ubuntu22.04_all.deb
sudo dpkg -i zabbix-release_7.0-2+ubuntu22.04_all.deb
sudo apt update
sudo apt install -y zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-agent
```

Import schema and data:

```bash
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql -u zabbix -p zabbix
```

Edit server config `/etc/zabbix/zabbix_server.conf` and set DB credentials:

```
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=ZabbixStrongP@ss!
```

Enable and start services:

```bash
sudo systemctl enable --now zabbix-server zabbix-agent apache2
```

**PHP timezone**: Set `date.timezone = Europe/Algiers` (or appropriate) in `/etc/php/8.1/apache2/php.ini` for the frontend.

**Why Zabbix here?** Zabbix gives centralized monitoring of hosts and services. Hosting it with GLPI is resource-efficient for small labs.



## 6. Windows Host — network & test configuration

The Windows host acts as an admin workstation and client for monitoring and GLPI.

### 6.1 Network
- Configure NIC to use DHCP (pfSense) or set static `192.168.1.100/24` gateway `192.168.1.1`.
- Set DNS to `192.168.1.101` if you run DNS on Ubuntu or pfSense's DNS forwarder otherwise.

### 6.2 Test connectivity
From Windows (PowerShell / CMD):
```powershell
ping 192.168.1.1      # pfSense
ping 192.168.1.101    # Ubuntu GLPI + Zabbix
curl http://192.168.1.101  # should return HTML
```

### 6.3 Browser access
Open `http://192.168.1.101` — the GLPI installer or application should appear. Access Zabbix at `http://192.168.1.101/zabbix`.



## 7. Firewall rules & security policies (pfSense)

Below is a list of recommended rules to implement on pfSense (Interface: LAN):

1. **Allow DHCP (if pfSense provides DHCP)**
   - Protocol: UDP, Source: LAN net, Destination: LAN address, Port: 67/68
2. **Allow DNS (clients -> DNS server)**
   - Protocol: TCP/UDP, Source: LAN net, Destination: 192.168.1.101, Port: 53
3. **Allow HTTP/HTTPS to GLPI/Zabbix**
   - Protocol: TCP, Source: LAN net, Destination: 192.168.1.101, Ports: 80,443
4. **Allow Zabbix agent traffic (Server polls agent)**
   - Protocol: TCP, Source: 192.168.1.101 (Zabbix server), Destination: Windows host, Port: 10050 (agent)
5. **Allow general web egress**
   - Protocol: TCP/UDP, Source: LAN net, Destination: any, Ports: standard web
6. **Deny by default** — put restrictive rules lower. Only allow needed services.

**Why:** Principle of least privilege; reduce lateral movement and unexpected exposure.



## 8. Agents (Zabbix & GLPI) installation and registration

### 8.1 Zabbix Agent (Windows)
1. Download Zabbix Agent MSI matching server version from https://www.zabbix.com/download.
2. Install with parameters:
```powershell
msiexec /i zabbix_agent.msi /qn /norestart ZABBIX_SERVER=192.168.1.101 ZABBIX_SERVER_ACTIVE=192.168.1.101 HOSTNAME=WinClient
```
3. Open Windows firewall for agent port (10050):
```powershell
New-NetFirewallRule -DisplayName "Zabbix Agent" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 10050
```
4. In Zabbix UI: Add host, set agent interface to the Windows host IP, link to appropriate templates.

### 8.2 GLPI Agent (Windows)
1. Download GLPI Agent MSI (https://github.com/glpi-project/glpi-agent/releases).
2. Install and configure server endpoint (GLPI URL): `http://192.168.1.101/glpi/plugins/webservices/` (depends on plugin setup).
3. Run an inventory and confirm the host appears in GLPI.

**Why agents?** Provide real-time metrics (Zabbix) and asset inventory (GLPI). Agents enable deeper, authenticated data collection.



## 9. Tests & validation plan

### Connectivity
- `ping 192.168.1.1` (pfSense)
- `ping 192.168.1.101` (Ubuntu)
- `curl http://192.168.1.101` should return HTML

### Application
- Open `http://192.168.1.101/` — GLPI installer or app
- Open `http://192.168.1.101/zabbix` — Zabbix frontend

### Monitoring
- Add Windows host to Zabbix, verify agent metrics (CPU, memory)
- Trigger a test alert by creating a low threshold (e.g., generate CPU load)

### Security
- Run `nmap -sS -sV 192.168.1.101` from Windows to list open ports (should show 80, 22 (optional), 10050 if agent installed)

### Backup test
- Export GLPI DB and restore to a spare DB to verify backups.



## 10. Troubleshooting common issues

### "Refused to connect" from browser
- Apache not running: `sudo systemctl status apache2` → start if inactive.
- Apache bound to `127.0.0.1`: ensure `/etc/apache2/ports.conf` contains `Listen 80` (not `127.0.0.1:80`).
- Firewall blocking port 80: `sudo iptables -L -n` → allow port 80 or disable rules.

### "Network unreachable" or `ens33: DOWN`
- Interface down: `sudo ip link set ens33 up`
- Netplan misconfigured: ensure `/etc/netplan/*.yaml` references correct interface name (match `ip a`).

### Ping works one-way
- Windows Firewall blocking ICMP inbound: enable **File and Printer Sharing (Echo Request - ICMPv4-In)** rule.
- VMware network mode NAT vs Bridged mismatch: set both host and VM to the same network mode (Bridged recommended).

### Zabbix agent shows "Not supported"
- Ensure agent service is running on host and server has correct IP/hostname mapping in Zabbix.



## 11. Backup, snapshots, and recovery

- **VM snapshots**: before big changes, take snapshots in VMware (pfSense, Ubuntu, Windows). Remember snapshots are not backups — do cold exports if you need permanent backups.
- **Export pfSense configuration**: Diagnostics -> Backup & Restore -> Download config.
- **GLPI & Zabbix DB dumps**:
  ```bash
  mysqldump -u root -p glpi > glpi_$(date +%F).sql
  mysqldump -u root -p zabbix > zabbix_$(date +%F).sql
  ```
- **File backup**: backup `/var/www/html/glpi` and `/etc/zabbix`.

**Recovery test**: Restore DB dumps and web files to a new VM and confirm services start.


## 12. Hardening checklist (minimum for a lab intending to model enterprise best practices)

- Disable remote root login for MariaDB (restrict root to `localhost`).
- Create dedicated DB users with least privilege.
- Secure web UI with TLS (use Let's Encrypt via certbot or pfSense reverse proxy + ACME client).
- Disable unnecessary services (`ftp`, `telnet`, etc.).
- Use strong passwords and rotate credentials.
- Keep system packages updated (`sudo apt update && sudo apt upgrade`).
- Monitor logs (Zabbix + local logs) and configure alerts for failures.
- Regularly export pfSense configuration and DB backups.



## 13. Automation snippets & useful scripts

### Example: create GLPI DB script
```bash
#!/bin/bash
DBPASS='GlpiStrongP@ss!'
mysql -u root -p<<SQL
CREATE DATABASE glpi CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'glpiuser'@'localhost' IDENTIFIED BY '${DBPASS}';
GRANT ALL PRIVILEGES ON glpi.* TO 'glpiuser'@'localhost';
FLUSH PRIVILEGES;
SQL
```

### Example: Netplan template (replace interface name)
```yaml
network:
  version: 2
  ethernets:
    <IFNAME>:
      dhcp4: no
      addresses: [192.168.1.101/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8,1.1.1.1]
```



## 14. Appendix: config files & snippets

### Apache virtual host (GLPI) `/etc/apache2/sites-available/glpi.conf`
```apache
<VirtualHost *:80>
  ServerName glpi.lab.local
  DocumentRoot /var/www/html/glpi
  <Directory /var/www/html/glpi>
    Options Indexes FollowSymLinks
    AllowOverride All
    Require all granted
  </Directory>
  ErrorLog ${APACHE_LOG_DIR}/glpi-error.log
  CustomLog ${APACHE_LOG_DIR}/glpi-access.log combined
</VirtualHost>
```

### MariaDB bind address (disable remote root) `/etc/mysql/mariadb.conf.d/50-server.cnf`
```ini
bind-address = 127.0.0.1
```

If you must allow remote DB connections for testing, set `bind-address = 0.0.0.0` and use firewall rules to restrict access.

### Zabbix server config `/etc/zabbix/zabbix_server.conf`
```
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=ZabbixStrongP@ss!
```

### Netplan sample `/etc/netplan/01-netcfg.yaml`
```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: no
      addresses: [192.168.1.101/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [192.168.1.1,8.8.8.8]
```

