---
name: home-network
description: Manages a private LAN between two Windows laptops for hosting and accessing applications. Covers router/direct connection, IP config, firewall rules, WSL, and keeps a dated session history of all changes.
---

# Home Network Setup

Manages a private LAN between two Windows laptops for application hosting and file sharing.

> ⏱ **Session history is tracked below.** Always read the [Session History](#-session-history) section first to understand what has already been configured before making changes.

## Overview

This skill helps you:
1. Connect two Windows laptops on the same private network
2. Configure static or DHCP-assigned IP addresses
3. Open Windows Firewall ports for hosted applications
4. Test connectivity between laptops
5. Optionally set up WSL with port forwarding
6. **Track all actions with date/time stamps** for future reference

## Prerequisites

- Two Windows laptops (10 or 11)
- For Option 1: A Wi-Fi router with available ports/SSID
- For Option 2: An Ethernet cable
- Administrator access on both laptops (for firewall changes)

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

### 5. Static IP (Optional — for permanent setup)

#### Router-side (Recommended):
Log into the router admin panel (`http://<GATEWAY_IP>`) → **DHCP Reservation** → Map each laptop's MAC address to a fixed IP.

#### Windows-side:
**Control Panel** → **Network and Sharing Center** → **Change adapter settings** → Right-click adapter → Properties → IPv4 → Properties → "Use the following IP address"

### 6. WSL Integration (Optional)

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

### 7. File Sharing via SMB

1. Right-click a folder → **Properties** → **Sharing** → **Share**
2. Add "Everyone" with Read/Write permissions
3. On the other laptop, open File Explorer and type:
   ```
   \\<HOST_IP>\SharedFolder
   ```

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Ping fails | Check firewalls on both laptops, verify same subnet |
| App not accessible | Bind to `0.0.0.0`, not `127.0.0.1`; add firewall rule |
| WSL app not reachable | Use `netsh interface portproxy` to forward the port |
| IP changed after reboot | Set up DHCP reservation on router or static IP on Windows |

Quick diagnostics:

```cmd
ipconfig                         # Your own IP
ping <OTHER_IP>                  # Test connectivity
Test-NetConnection <IP> -Port 8000   # Test port reachability
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

*Add new sessions below this line using the entry format above.*
