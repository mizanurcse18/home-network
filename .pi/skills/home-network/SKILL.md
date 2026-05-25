---
name: home-network
description: Manages a private LAN between two Windows laptops for hosting apps, Remote Desktop, file sharing, internet-accessible websites, Docker .NET 8 microservices, and network troubleshooting. Covers router/direct connection, IP config (static/DHCP reservation), firewall rules, RDP setup, password recovery, port forwarding, reverse proxy, Cloudflare Tunnel, ngrok, Docker Compose with PostgreSQL/MySQL, WSL, and keeps a dated session history of all changes.
---

# Home Network Setup

Manages a private LAN between two Windows laptops for application hosting, Remote Desktop access, file sharing, and more.

> ⏱ **Session history is tracked below.** Always read the [Session History](#-session-history) section first to understand what has already been configured before making changes.

## Overview

This skill helps you:
1. Connect two Windows laptops on the same private network (router or direct cable)
2. Configure static or DHCP-assigned IP addresses (DHCP reservation or manual)
3. Open Windows Firewall ports for hosted applications
4. **Enable and access Remote Desktop (RDP)** between laptops
5. **Reset or change Windows passwords** via command line
6. **Host a website accessible from the internet** (port forwarding, reverse proxy, Cloudflare Tunnel, ngrok)
7. **Run .NET 8 microservices in Docker on Laptop B with database on Laptop A** (PostgreSQL/MySQL)
8. **Prepare for Linux production deployment**
9. Set up WSL with port forwarding
10. Share files via SMB
11. **Track all actions with date/time stamps** for future reference

## Prerequisites

- Two Windows laptops (10 or 11)
- For Option 1: A Wi-Fi router with available ports/SSID
- For Option 2: An Ethernet cable
- Administrator access on both laptops (for firewall and system changes)

## Usage

### 1. Identify the network configuration

On each laptop, run in Command Prompt:

```cmd
ipconfig
```

Look for the active adapter's **IPv4 Address** and **Default Gateway**.

> Both laptops must be on the **same subnet** (e.g. `192.168.1.x/24` or `192.168.68.x/24`).

### 2. Verify connectivity

From Laptop A, ping Laptop B:

```cmd
ping <LAPTOP_B_IP>
```

From Laptop B, ping Laptop A:

```cmd
ping <LAPTOP_A_IP>
```

Expected output:
```
Reply from 192.168.x.x: bytes=32 time<1ms TTL=128
```

### 3. Host a test application (Python)

On the host laptop:

```cmd
python -m http.server 8000
```

On the client laptop, open a browser to:
```
http://<HOST_IP>:8000
```

### 4. Open Windows Firewall (if needed)

If the browser cannot reach the server, allow the port on the **host laptop**:

#### Via Command Prompt (Admin):

```cmd
netsh advfirewall firewall add rule name="Home Network App" dir=in action=allow protocol=TCP localport=8000
```

#### Via GUI:
1. Press **Win + R**, type `wf.msc`, press Enter
2. **Inbound Rules** → **New Rule…**
3. Port → TCP → Specific local ports: `8000` → Allow → Name: "Home Network App"

> **Important:** For any application, bind to `0.0.0.0` (all interfaces), not `127.0.0.1` (localhost only).

### 5. Fix Dynamic IP (Make IP Static)

If a laptop's IP changes after reboot, pick one method:

#### Option A: DHCP Reservation on Router (✅ Recommended)
1. Open browser → `http://<GATEWAY_IP>` (e.g. `192.168.68.1`)
2. Log into router admin
3. Find **DHCP Reservation** or **Address Reservation**
4. Look up the laptop's MAC address (`ipconfig /all` → Physical Address)
5. Assign a fixed IP and save

#### Option B: Static IP on Windows (Command Line)
Run as Administrator on the target laptop:

```cmd
:: Set static IP (replace Wi-Fi with your adapter name if needed)
netsh interface ip set address name="Wi-Fi" static 192.168.68.107 255.255.255.0 192.168.68.1

:: Set DNS
netsh interface ip set dns name="Wi-Fi" static 8.8.8.8
```

#### Option C: Static IP via GUI
**Control Panel** → **Network and Sharing Center** → **Change adapter settings** → Right-click adapter → Properties → IPv4 → Properties → "Use the following IP address"

### 6. Enable Remote Desktop (RDP)

#### Step 1: Enable RDP on the target laptop
Run as Administrator:

```cmd
reg add "HKLM\System\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f
```

#### Step 2: Allow RDP through firewall
```cmd
netsh advfirewall firewall add rule name="Remote Desktop" dir=in action=allow protocol=TCP localport=3389
```

#### Step 3: (Optional) Allow user via GUI
1. **Win + R** → `sysdm.cpl` → Enter
2. **Remote** tab → Check **"Allow remote connections to this computer"**
3. Uncheck "Allow connections only from computers with Remote Desktop..." (for same-network access)

#### Step 4: Connect from the other laptop
1. Press **Win + R** → `mstsc` → Enter
2. Computer: `<TARGET_IP>` (e.g. `192.168.68.107`)
3. Username: `<LAPTOP_NAME>\<USERNAME>` (e.g. `200413-mizanur\mizanur.rahman`)
4. Enter the target laptop's login password

#### Step 5: Verify RDP port is open
From the client laptop:

```cmd
Test-NetConnection <TARGET_IP> -Port 3389
```

### 7. Reset / Change Windows Password via Command Line

If you're logged in (via PIN or fingerprint) but forgot the password:

#### Method A: Command Prompt (Admin)
```cmd
net user <USERNAME> *
```
Type the new password twice (characters won't show). Ensure it meets complexity:
- At least 8 characters
- Upper + lower case letters
- At least 1 number
- At least 1 special character (e.g. `!@#$`)

#### Method B: Windows Settings
**Settings** → **Accounts** → **Sign-in options** → **Password** → **Change** (verify with PIN/fingerprint)

> **Note:** If using a Microsoft account, the password policy is enforced by Microsoft's servers, not the local machine.

### 8. WSL Integration (Optional)

If using WSL for application hosting:

```powershell
# Find WSL IP (from within WSL)
ip addr show eth0 | grep inet

# Forward port from Windows to WSL (Admin PowerShell)
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=3000 connectaddress=172.x.x.x connectport=3000

# Verify
netsh interface portproxy show all
```

Alternatively, enable mirrored networking in `%USERPROFILE%\.wslconfig`:
```ini
[wsl2]
networkingMode=mirrored
dhcp=true
```

### 10. Host a Website Accessible from the Internet

You have several options to make a site hosted on a laptop accessible from outside your home network.

---

#### Option A: Port Forwarding on Router (Most Common)

Forward a public port from your router directly to the target laptop.

```
Internet ──► Router ──► Laptop (192.168.68.x:PORT)
```

**Steps:**
1. Log into your router at `http://<GATEWAY_IP>` (e.g. `192.168.68.1`)
2. Find **Port Forwarding** or **Virtual Server**
3. Add a rule:

   | Setting | Value |
   |---------|-------|
   | External Port | e.g. `8080` |
   | Internal IP | Target laptop's IP |
   | Internal Port | Your app's port |
   | Protocol | TCP |

4. On the target laptop, bind app to `0.0.0.0:<PORT>` and add firewall rule:
   ```cmd
   netsh advfirewall firewall add rule name="Web App <PORT>" dir=in action=allow protocol=TCP localport=<PORT>
   ```

5. Find your public IP:
   ```cmd
   curl ifconfig.me
   ```
   Access: `http://<PUBLIC_IP>:<PORT>`

> ⚠️ Most home ISPs change your public IP. Use **Dynamic DNS** (DDNS) like `noip.com` for a fixed hostname.

---

#### Option B: Reverse Proxy via Another Laptop

If one laptop already has internet access, use it as a reverse proxy.

```
Internet ──► Laptop A (port 8080) ──► Laptop B (192.168.68.107:8080)
```

**Using Node.js on the proxy laptop:**

Create `proxy.js`:
```javascript
const http = require('http');
const httpProxy = require('http-proxy');
const proxy = httpProxy.createProxyServer({});
const TARGET = 'http://192.168.68.107:8080';
http.createServer((req, res) => {
  proxy.web(req, res, { target: TARGET });
}).listen(8080, '0.0.0.0', () => {
  console.log('Proxy running on port 8080 -> Laptop B');
});
```

```cmd
npm install http-proxy
node proxy.js
```

Add firewall rule on the proxy laptop:
```cmd
netsh advfirewall firewall add rule name="Web Proxy 8080" dir=in action=allow protocol=TCP localport=8080
```

---

#### Option C: Cloudflare Tunnel (✅ Best — No public IP needed)

Free, secure, no port forwarding. Works with CGNAT and dynamic IPs.

**On the target laptop:**
1. Download `cloudflared` from [Cloudflare Tunnel docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
2. Authenticate and create a tunnel:
   ```cmd
   cloudflared tunnel login
   cloudflared tunnel create my-site
   cloudflared tunnel route dns my-site mysite.example.com
   cloudflared tunnel run my-site
   ```
Your site is live at `https://mysite.example.com`.

---

#### Option D: ngrok (Easiest for Testing)

Single command on the target laptop:
```cmd
ngrok http 8080
```
Returns a public URL like `https://abc123.ngrok.io`.

---

#### Quick Comparison

| Option | Public IP Needed | Setup Time | Security | Best For |
|--------|-----------------|------------|----------|---------|
| Port Forwarding | Yes (or DDNS) | Medium | Moderate | Permanent sites |
| Reverse Proxy | Depends on proxy host | Medium | Good | Leveraging existing setup |
| Cloudflare Tunnel | No | Medium | Excellent | Production, custom domain |
| ngrok | No | 1 min | Good | Quick testing |

---

### 11. File Sharing via SMB

1. Right-click a folder → **Properties** → **Sharing** → **Share**
2. Add **Everyone** with Read/Write permissions
3. On the other laptop, open **File Explorer** and type:
   ```
   \\<HOST_IP>\SharedFolder
   ```

---

### 12. Docker .NET 8 Microservices (Laptop B) + Database (Laptop A)

Run .NET 8 microservices in Docker on Laptop B, with PostgreSQL/MySQL database on Laptop A. Designed to mirror a Linux production environment.

#### Architecture

```
Laptop B (Docker)                    Laptop A (Database)
─────────────────                    ────────────────────
security-api:5001 ──── LAN ────────► PostgreSQL:5432
scm-api:5002       ──── (Wi-Fi) ───► or MySQL:3306
file-service:5003  ────            
```

#### Quick Start

**On Laptop A (database):**
1. Install PostgreSQL or MySQL
2. Configure for remote access (`listen_addresses = '*'`)
3. Add firewall rule:
   ```cmd
   netsh advfirewall firewall add rule name="PostgreSQL" dir=in action=allow protocol=TCP localport=5432
   ```
4. Create databases and user

**On Laptop B (Docker):**
1. Install Docker Desktop (WSL 2 backend)
2. Create `docker-compose.yml` with your services
3. Add firewall rules:
   ```cmd
   netsh advfirewall firewall add rule name="Security API" dir=in action=allow protocol=TCP localport=5001
   netsh advfirewall firewall add rule name="SCM API" dir=in action=allow protocol=TCP localport=5002
   netsh advfirewall firewall add rule name="File Service" dir=in action=allow protocol=TCP localport=5003
   ```
4. Run:
   ```cmd
   docker compose up -d --build
   ```

#### Sample docker-compose.yml

```yaml
version: '3.8'
services:
  security-api:
    build: ./security-api
    container_name: security-api
    ports:
      - "5001:8080"
    environment:
      - ConnectionStrings__DefaultConnection=Host=192.168.68.113;Port=5432;Database=security_db;Username=appuser;Password=YourPassword123!
    restart: unless-stopped
```

#### Dockerfile (for each .NET 8 service)

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 8080

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["src/Service.Api.csproj", "src/"]
RUN dotnet restore "src/Service.Api.csproj"
COPY . .
RUN dotnet build "src/Service.Api.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "src/Service.Api.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENV ASPNETCORE_URLS=http://+:8080
ENTRYPOINT ["dotnet", "Service.Api.dll"]
```

#### Production Migration

When moving to Linux production server:
1. Docker Compose stays the same (Linux containers)
2. Update connection strings to production DB
3. Use `.env` files for environment-specific config
4. Add reverse proxy (nginx) in front of services
5. Use Docker registry to push/pull images

> See full guide: `docker-deployment/ARCHITECTURE.md` in this project

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Ping fails | Check firewalls on both laptops, verify same subnet |
| App not accessible | Bind to `0.0.0.0`, not `127.0.0.1`; add firewall rule |
| RDP won't connect | Check RDP is enabled (`fDenyTSConnections = 0`), port 3389 open in firewall, user added to "Remote Desktop Users" group |
| RDP "credentials did not work" | Verify username format: `LAPTOP_NAME\USERNAME` or just `USERNAME`. Check password with `net user` command |
| RDP "user not allowed" | Run on target: `net localgroup "Remote Desktop Users" /add <USERNAME>` |
| Password rejected by policy | Use 8+ chars with upper, lower, number, and special character |
| WSL app not reachable | Use `netsh interface portproxy` to forward the port |
| IP changed after reboot | Set up DHCP reservation on router or static IP on Windows |

Quick diagnostics:

```cmd
ipconfig                         # Your own IP
ping <OTHER_IP>                  # Test connectivity
Test-NetConnection <IP> -Port 8000   # Test port reachability
Test-NetConnection <IP> -Port 3389   # Test RDP port
netsh advfirewall firewall show rule name=all   # List firewall rules
```

---

## ⏱ Session History

> **Agent instructions:** Before making any changes, always read this section to understand what's already configured. After making changes, append a new entry with the current date/time and description of what was done.

### Entry Format

Each session entry follows this format:

```
### YYYY-MM-DD HH:MM — Brief title

**Action:** What was done
**Laptop(s):** Which laptop(s) were involved
**Configuration:** Key IPs, ports, settings applied
**Result:** ✅ Success / ❌ Failure / ⚠️ Partial
**Notes:** Any observations or next steps
```

---

### 2026-05-25 ~14:00 — Initial setup: Both laptops connected via Wi-Fi router

**Action:** Connected both laptops to the same home Wi-Fi network, discovered IPs, verified connectivity, hosted a test HTTP server.
**Laptop(s):** Both
**Configuration:**
- Router/Gateway: `192.168.68.1`
- Subnet: `255.255.255.0` (/24)
- Laptop A: `192.168.68.113` (hostname: `LAPTOP-5N7...`)
- Laptop B: `192.168.68.107`
- Test server: Node.js HTTP on port 8000, bound to `0.0.0.0`
- Firewall rule added: `Home Network Test 8000` (TCP 8000, inbound)
**Result:** ✅ Success — Laptop B reached `http://192.168.68.113:8000`
**Notes:** Both laptops on Wi-Fi, no static IPs set yet. If IPs change after reboot, set up DHCP reservation on the router.

---

### 2026-05-25 ~15:00 — Enabled Remote Desktop on Laptop B + fixed dynamic IP concern

**Action:** Enabled RDP on Laptop B, opened port 3389 in firewall, verified connectivity from Laptop A. User reset password via `net user` command after forgetting it. Documented methods to fix dynamic IP (DHCP reservation / static IP).
**Laptop(s):** Both
**Configuration:**
- Laptop B RDP: Enabled (`fDenyTSConnections = 0`)
- Firewall rule added: `Remote Desktop` (TCP 3389, inbound)
- RDP port verified: `TcpTestSucceeded : True` from Laptop A
- Laptop B username: `200413-mizanur\mizanur.rahman`
- Password reset via: `net user mizanur.rahman *`
- Laptop A already has RDP enabled for outside/internet access (pre-existing)
**Result:** ✅ Success — RDP connection from Laptop A to Laptop B works with correct credentials
**Notes:** Laptop B still has dynamic IP. Recommend setting up DHCP reservation on router at `192.168.68.1` for permanent fix. Write down the new password!

---

### 2026-05-25 ~16:00 — Documented options for hosting a site accessible from the internet

**Action:** Added comprehensive documentation for making a website on Laptop B reachable from the internet. Covered port forwarding, reverse proxy via Laptop A, Cloudflare Tunnel, and ngrok.
**Laptop(s):** Both
**Configuration:**
- All four methods documented with step-by-step instructions
- Port forwarding: Router `192.168.68.1` → Laptop B `192.168.68.107`
- Reverse proxy: Laptop A (`192.168.68.113`) acts as proxy to Laptop B
- Cloudflare Tunnel: No public IP required
- ngrok: Quick testing URL
**Result:** ✅ Documentation added to both README.md and SKILL.md
**Notes:** Laptop A already has internet-facing access which can be leveraged for reverse proxy. Cloudflare Tunnel is recommended for production use.

---

### 2026-05-25 ~17:00 — Architected Docker .NET 8 microservices deployment

**Action:** Designed and documented full architecture for running .NET 8 microservices (security, SCM, file service) in Docker on Laptop B with PostgreSQL/MySQL database on Laptop A. Created `docker-deployment/ARCHITECTURE.md` with step-by-step setup, Dockerfiles, docker-compose.yml, production migration guide, and troubleshooting.
**Laptop(s):** Both
**Configuration:**
- Laptop B: Docker Desktop (WSL 2) running 3 .NET 8 services on ports 5001, 5002, 5003
- Laptop A: PostgreSQL (5432) and/or MySQL (3306) with remote access enabled
- Network: Services connect to database via LAN at `192.168.68.113`
- Docker Compose with health checks, volumes, and restart policies
**Result:** ✅ Documentation complete — ARCHITECTURE.md, SKILL.md, README.md updated
**Notes:** This design mirrors a Linux production environment. Transition to production involves updating connection strings, adding nginx reverse proxy, and using Docker registry.

---

*Add new sessions below this line using the entry format above.*
