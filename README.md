# 🏠 Home Network Setup Guide

## Two Windows Laptops – Private Network & Hosting Applications

**Created:** 2026-05-25

---

## Table of Contents

1. [Overview](#overview)
2. [Option 1: Both Laptops on the Same Router (Recommended)](#option-1-both-laptops-on-the-same-router-recommended)
3. [Option 2: Direct Ethernet Connection (No Router)](#option-2-direct-ethernet-connection-no-router)
4. [Verifying Connectivity](#verifying-connectivity)
5. [Hosting & Accessing Applications](#hosting--accessing-applications)
6. [Windows Firewall Configuration](#windows-firewall-configuration)
7. [Static IP Assignment (Optional)](#static-ip-assignment-optional)
8. [Using WSL](#using-wsl)
9. [Troubleshooting](#troubleshooting)
10. [Session Log](#session-log)
11. [Pi Skill](#pi-skill)

---

## Overview

This guide explains how to set up a private local area network (LAN) between **two Windows laptops**, allowing you to:

- Share files between laptops
- Host web applications, APIs, databases, or game servers on one laptop
- Access those applications from the other laptop via private IP addresses
- Work entirely offline on your internal network

### What you'll need

| Requirement | Option 1 (Router) | Option 2 (Direct) |
|-------------|------------------|-------------------|
| Wi-Fi Router | ✅ Yes | ❌ No |
| Ethernet Cable | ❌ No | ✅ Yes |
| Both laptops on same subnet | Automatic via DHCP | Must assign manually |

---
## Option 1: Both Laptops on the Same Router (Recommended)

This is the simplest setup. Both laptops connect to the same home router (Wi-Fi or Ethernet).

### Step 1: Connect both laptops to the same router

- **Wi-Fi:** Connect to your home Wi-Fi network on both laptops.
- **Ethernet:** Plug each laptop into a LAN port on the router.

### Step 2: Find each laptop's private IP

On each Windows laptop, open **Command Prompt** (cmd) and type:

```
ipconfig
```

Look for the active network adapter (Wi-Fi or Ethernet) and find the **IPv4 Address**.

**Example output:**

```
Wireless LAN adapter Wi-Fi:
   IPv4 Address. . . . . . . . . . . : 192.168.1.10
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.1.1
```

| Laptop | Typical IP Address |
|--------|-------------------|
| **Laptop A** | 192.168.1.10 |
| **Laptop B** | 192.168.1.11 |

### Step 3: Note down the IPs

Write down the IP addresses of both laptops — you'll use them to connect.

---
## Option 2: Direct Ethernet Connection (No Router)

If you don't have a router, connect the two laptops directly with an Ethernet cable.

### Step 1: Connect the Ethernet cable

Plug one end into Laptop A and the other into Laptop B.

> Most modern laptops support **Auto MDI-X**, so a regular (straight-through) cable works fine.

### Step 2: Assign static IP addresses

Since there's no router to assign IPs automatically, configure them manually.

#### On Laptop A:

1. Open **Control Panel** → **Network and Sharing Center** → **Change adapter settings**
2. Right-click **Ethernet** adapter → **Properties**
3. Select **Internet Protocol Version 4 (TCP/IPv4)** → **Properties**
4. Choose **"Use the following IP address"** and enter:

   | Setting | Value |
   |---------|-------|
   | IP address | 192.168.1.1 |
   | Subnet mask | 255.255.255.0 |
   | Default gateway | (leave blank) |
5. Click **OK** → **Close**

#### On Laptop B:

Use the same steps, but with:

   | Setting | Value |
   |---------|-------|
   | IP address | 192.168.1.2 |
   | Subnet mask | 255.255.255.0 |
   | Default gateway | (leave blank) |

---
## Verifying Connectivity

### Ping test from Laptop A:
```
ping 192.168.1.2
```

### Ping test from Laptop B:
```
ping 192.168.1.1
```

**Expected result:**
```
Reply from 192.168.1.2: bytes=32 time<1ms TTL=128
```

---
## Hosting & Accessing Applications

Once the network is working, host apps on one laptop and access from the other.

### Quick test with Python HTTP server

#### On the host laptop (e.g. Laptop A — 192.168.1.1):
```
python -m http.server 8000
```

#### On the client laptop (Laptop B):
Open a browser and go to:
```
http://192.168.1.1:8000
```

### Host a custom application

For any application (web app, API, database, game server):

1. **Start your application** listening on **0.0.0.0** (all interfaces) — not 127.0.0.1 (localhost only).
2. **Allow the port** through Windows Firewall (see next section).
3. **Access it** from the other laptop via http://192.168.1.1:PORT

#### Example: Node.js Express app
```javascript
const express = require('express');
const app = express();
app.get('/', (req, res) => res.send('Hello from Laptop A!'));
app.listen(3000, '0.0.0.0', () => console.log('Server running on port 3000'));
```

From Laptop B: http://192.168.1.1:3000

### File sharing via SMB (Windows)

1. On the host laptop, right-click a folder → **Properties** → **Sharing** tab → **Share**.
2. Add **Everyone** with desired permissions.
3. On the client laptop, open **File Explorer** and type:
```
\\192.168.1.1\SharedFolder
```

---
## Windows Firewall Configuration

Windows Firewall blocks incoming connections by default. You must open ports.

### Quick test (disable firewall temporarily)
> ⚠️ Only for testing — re-enable after.
```
netsh advfirewall set allprofiles state off
```
To re-enable:
```
netsh advfirewall set allprofiles state on
```

### Add a firewall rule for a specific port (Recommended)

1. Press **Win + R**, type **wf.msc**, press Enter
2. Click **Inbound Rules** → **New Rule…**
3. Select **Port** → **Next**
4. Choose **TCP** → **Specific local ports**: 8000,3000 (your app ports)
5. **Allow the connection** → **Next**
6. Check all profiles → **Next**
7. Give it a name (e.g. "Home Network Apps") → **Finish**

### Via Command Prompt
```
netsh advfirewall firewall add rule name="Home App 8000" dir=in action=allow protocol=TCP localport=8000
```

---
## Static IP Assignment (Optional)

By default, your router assigns IPs via DHCP, which can change. For a fixed IP:

### Router-side (Recommended)

Log into your router admin panel (typically http://192.168.1.1) and find **DHCP Reservation** or **Static DHCP**. Map each laptop's MAC address to a fixed IP.

### Windows-side
1. **Control Panel** → **Network and Sharing Center** → **Change adapter settings**
2. Right-click adapter → **Properties** → **IPv4** → **Properties**
3. Choose **"Use the following IP address"**

   | Setting | Value |
   |---------|-------|
   | IP address | e.g. 192.168.1.100 |
   | Subnet mask | 255.255.255.0 |
   | Default gateway | Router IP (e.g. 192.168.1.1) |
   | Preferred DNS | 8.8.8.8 or router IP |

---
## Using Windows Subsystem for Linux (WSL)

If you prefer a Linux environment for hosting:

### Install WSL
```
wsl --install
```

### Find WSL's IP
From within WSL:
```
ip addr show eth0 | grep inet
```

### Forward ports from Windows to WSL
```powershell
# Run PowerShell as Administrator
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=3000 connectaddress=172.x.x.x connectport=3000
# Verify
netsh interface portproxy show all
```

### Alternative: mirrored networking mode
Create %USERPROFILE%\.wslconfig:
```ini
[wsl2]
networkingMode=mirrored
dhcp=true
```
Then restart WSL:
```
wsl --shutdown
wsl
```
This makes WSL apps reachable directly without port forwarding.

---
## Troubleshooting

| Symptom | Possible Cause | Solution |
|---------|---------------|----------|
| ❌ Ping fails | Firewall blocking ICMP | Test with Test-NetConnection or temporarily disable firewall |
| ❌ Ping fails | Wrong subnet | Both must be on same subnet (e.g. 192.168.1.x/24) |
| ❌ Can't access app | App bound to 127.0.0.1 | Bind to 0.0.0.0 instead |
| ❌ Can't access app | Firewall blocking port | Add inbound rule for the port |
| ❌ No IP (direct cable) | Static IP not set | Assign static IPs manually |
| ❌ WSL app inaccessible | Port not forwarded | Use netsh interface portproxy |
| ❌ Cannot ping direct Ethernet | Network profile is "Public" | Change to "Private" in Windows Settings |
| ❌ IP changed after reboot | DHCP assigned new IP | Set static IP or use router reservation |

### Quick diagnostic commands
```
:: Check your own IP
ipconfig

:: Test connectivity
ping 192.168.1.2

:: Check if a port is reachable
Test-NetConnection 192.168.1.2 -Port 8000

:: See active firewall rules
netsh advfirewall firewall show rule name=all
```

---
## Quick Reference
```
+----------+      Wi-Fi / Ethernet      +----------+
| Laptop A |<-------------------------->| Laptop B |
| Host app |                             | Consume  |
+----------+                             +----------+

Host app:  bind to 0.0.0.0:PORT
Client:    http://192.168.1.10:PORT
Firewall:  allow PORT via wf.msc or netsh
```

---

## Session Log — Actual Walkthrough (2026-05-25)

This section documents the real steps taken during the initial setup session on **2026-05-25**.

### Laptops

| Laptop | Hostname | IP Address | Connection |
|--------|----------|-----------|------------|
| **Laptop A** | `LAPTOP-5N7...` | `192.168.68.113` | Wi-Fi |
| **Laptop B** | *(user's other laptop)* | `192.168.68.107` | Wi-Fi |

### Network Info

- **Router/Gateway:** `192.168.68.1`
- **Subnet:** `255.255.255.0` (`/24`)
- **Method:** Option 1 — Both laptops connected to the same Wi-Fi router

### Steps Taken

1. **Connected both laptops** to the same home Wi-Fi network.
2. **Discovered IPs** via `ipconfig` on both laptops:
   - Laptop A: `192.168.68.113`
   - Laptop B: `192.168.68.107`
3. **Verified connectivity** using `Test-Connection` (PowerShell) from Laptop A to Laptop B — successful with ~4–28ms latency.
4. **Hosted a test HTTP server** on Laptop A using Node.js (port 8000, bound to `0.0.0.0`):
   ```javascript
   const http = require('http');
   const server = http.createServer((req, res) => {
     res.end('<h1>Home Network Works!</h1>');
   });
   server.listen(8000, '0.0.0.0');
   ```
5. **Created a firewall rule** to allow port 8000 inbound on Laptop A (netsh).
6. **Accessed the server** from Laptop B via `http://192.168.68.113:8000` — ✅ **successful!**

### Commands Used

```cmd
:: On Laptop A — find IP
ipconfig

:: On Laptop B — find IP
ipconfig

:: On Laptop A — test connectivity to Laptop B (PowerShell)
Test-Connection -ComputerName 192.168.68.107 -Count 3

:: On Laptop A — start test server (Node.js)
node -e "const http=require('http');http.createServer((q,r)=>{r.end('<h1>OK</h1>')}).listen(8000,'0.0.0.0')"

:: On Laptop A — allow port through firewall (Admin cmd)
netsh advfirewall firewall add rule name="Home Network Test 8000" dir=in action=allow protocol=TCP localport=8000
```

### Result

Both laptops now communicate on the private `192.168.68.x` network. Applications hosted on Laptop A (bound to `0.0.0.0`) are accessible from Laptop B via `http://192.168.68.113:PORT`.

---

---
## Hosting a Website Accessible from the Internet

Since both laptops are on your private network (`192.168.68.x`), you have several options to make a site hosted on **Laptop B** available from the outside world.

---

### Option 1: Port Forwarding on Router (Most Common)

Forward a public port from your router directly to Laptop B.

```
Internet ──► Router (192.168.68.1) ──► Laptop B (192.168.68.107:8080)
```

**Steps:**

1. Log into your router at `http://192.168.68.1`
2. Find **Port Forwarding** or **Virtual Server**
3. Add a rule:

   | Setting | Value |
   |---------|-------|
   | External Port | `8080` (or any available port) |
   | Internal IP | `192.168.68.107` |
   | Internal Port | `8080` (your app's port) |
   | Protocol | TCP |

4. On **Laptop B**, bind your app to `0.0.0.0:8080` and allow the port through firewall:
   ```cmd
   netsh advfirewall firewall add rule name="Web App 8080" dir=in action=allow protocol=TCP localport=8080
   ```

5. Find your public IP:
   ```cmd
   curl ifconfig.me
   ```
   Then access: `http://<YOUR_PUBLIC_IP>:8080`

> ⚠️ Most home ISPs change your public IP periodically. Use **Dynamic DNS** (DDNS) like `noip.com` or your router's built-in DDNS for a fixed hostname.

---

### Option 2: Reverse Proxy via Laptop A (Leverage existing setup)

Since **Laptop A already has internet-facing access**, use it as a reverse proxy to forward traffic to Laptop B.

```
Internet ──► Laptop A (port 8080) ──► Laptop B (192.168.68.107:8080)
```

#### Using Node.js (install Node.js first)

On **Laptop A**, create `proxy.js`:

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

Install & run:
```cmd
npm install http-proxy
node proxy.js
```

Add firewall rule on Laptop A:
```cmd
netsh advfirewall firewall add rule name="Web Proxy 8080" dir=in action=allow protocol=TCP localport=8080
```

Now access from internet: `http://<LAPTOP_A_PUBLIC_IP>:8080`

---

### Option 3: Cloudflare Tunnel (✅ Best — No public IP needed)

Free, secure, no port forwarding required. Works even with CGNAT or dynamic IPs.

**Steps on Laptop B:**

1. Download `cloudflared` from [Cloudflare Tunnel docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
2. Authenticate and create a tunnel:
   ```cmd
   cloudflared tunnel login
   cloudflared tunnel create my-site
   cloudflared tunnel route dns my-site mysite.example.com
   cloudflared tunnel run my-site
   ```

Your site is live at `https://mysite.example.com` — no router config needed!

---

### Option 4: ngrok (Easiest for Testing)

No setup, single command on **Laptop B**:

```cmd
ngrok http 8080
```

It returns a public URL like `https://abc123.ngrok.io` that tunnels to your local port.

---

### Quick Comparison

| Option | Public IP Needed | Setup Time | Security | Best For |
|--------|-----------------|------------|----------|---------|
| Port Forwarding | Yes (or DDNS) | Medium | Moderate | Permanent sites |
| Reverse Proxy (Laptop A) | Depends on Laptop A | Medium | Good | Leveraging existing setup |
| Cloudflare Tunnel | No | Medium | Excellent | Production, custom domain |
| ngrok | No | 1 min | Good | Quick testing |

---

## Pi Skill

A reusable pi skill with **date/time session tracking** is included in this project:

**Location:** `D:\Doucuments\home-network\.pi\skills\home-network\SKILL.md`

### How it works

The skill has a **`⏱ Session History`** section that records every action with:

| Field | Example |
|-------|---------|
| **Timestamp** | `2026-05-25 ~14:00` |
| **Action** | Initial setup: both laptops connected via Wi-Fi |
| **Laptop(s)** | Both |
| **Configuration** | IPs, ports, firewall rules |
| **Result** | ✅ Success |
| **Notes** | Observations and next steps |

### What the agent will do next time

1. Read the skill → check the session history
2. Know what's already configured (IPs, ports, firewall rules)
3. Only apply changes that haven't been done yet
4. **Append a new timestamped entry** for any new changes

### How to trigger the skill

- The agent **auto-discovers** skills from `.pi/skills/` — no action needed
- Or manually: `/skill:home-network`

### Current session history (in the skill)

The skill already records the initial 2026-05-25 session. Any future work will be appended with new timestamps.

---

## Next Steps / Customization

- **Need a specific application hosted?** Let me know what you're running (web app, database, game server, etc.) and I'll add specific instructions.
- **Want remote access from outside your home?** That requires port forwarding on your router and caution with security — ask me about setting up a VPN instead.
- **Need to share files regularly?** Set up a permanent SMB share or install a NAS-like tool (e.g., Resilio Sync, Syncthing).
- **Want static IPs?** Log into your router at `http://192.168.68.1` and set up DHCP reservation for both laptops.

---

*End of document.*
