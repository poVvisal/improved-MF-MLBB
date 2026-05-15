<div align="center">

# 🚀 VLESS WebSocket (No TLS)
### Cloudflare ISP Bypass Guide

[![GitHub last commit](https://img.shields.io/github/last-commit/google/skia)](https://github.com)
[![Maintenance](https://img.shields.io/badge/Maintained%3F-yes-green.svg)](https://github.com)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](http://makeapullrequest.com)

Deploy a **blazing-fast VLESS over WebSocket on Port 80** with Marzban + Cloudflare. Bypass ISP throttling by disguising traffic as CDN requests using the classic **Bug Host** trick.

[Key Features](#-key-features) • [Quick Start](#-quick-start) • [Architecture](#-architecture) • [Setup](#-setup-guide) • [Troubleshooting](#-troubleshooting)

</div>

---

## ✨ Key Features
- 🔒 ISP-proof tunneling that looks like normal CDN traffic
- ⚡ Zero TLS overhead — raw WebSocket on Port 80
- 🌐 Cloudflare shield — ISP only sees whitelisted CF IPs
- 🎮 Gaming ready — great for Mobile Legends, PUBG, more
- 📱 Multi-platform — v2rayNG, V2Box, Streisand
- 🆓 Free-tier friendly — runs on Cloudflare free plan

---

## 🏗️ Architecture

```
┌─────────────┐      ┌──────────────┐      ┌────────────┐      ┌─────────────┐
│  Your App   │─────▶│     ISP      │─────▶│ Cloudflare │─────▶│  Your VPS   │
│  (Client)   │      │ (sees CF IP) │      │    CDN     │      │  (Port 80)  │
└─────────────┘      └──────────────┘      └────────────┘      └─────────────┘
```

**Flow:** Client hits a public Cloudflare IP → ISP sees allowed CDN traffic → Cloudflare forwards plain HTTP WebSocket to your VPS on Port 80.

---

## 🎯 Quick Start

```bash
# Install Marzban
sudo bash -c "$(curl -sL https://github.com/Gozargah/Marzban-scripts/raw/master/marzban.sh)" @ install

# Open dashboard
http://YOUR_VPS_IP:8000/dashboard
```

Then follow the full setup below to harden configs and generate links.

---

## 📋 Prerequisites

| Requirement | Specification | Notes |
|-------------|---------------|-------|
| 🖥️ VPS | Ubuntu 20.04+ | 1 CPU / 1GB RAM is fine |
| 🌐 Domain | Any TLD | Must use Cloudflare DNS |
| ☁️ Cloudflare | Free account | Keep it on the orange cloud |

---

## 🛠️ Setup Guide

### 1) Cloudflare Configuration (critical)
1. Dashboard → **DNS** → add **A record**
   - Name: `vpn` (or any subdomain)
   - IPv4: your VPS IP
   - Proxy status: **Proxied** (orange cloud ON)
2. Dashboard → **SSL/TLS** → **Overview** → set mode to **Flexible** (CF talks HTTP:80 to your VPS). Full/Strict will fail on No-TLS setups.

### 2) VPS Preparation (Ubuntu 20.04+)
1. Base OS prep (as root or sudo):
   ```bash
   sudo apt update && sudo apt upgrade -y
   sudo apt install -y curl ca-certificates gnupg lsof
   ```
2. Install Docker (Marzban uses Docker Compose under the hood):
   ```bash
   curl -fsSL https://get.docker.com | sudo sh
   sudo systemctl enable --now docker
   sudo usermod -aG docker $USER  # re-login to take effect
   ```
3. Open required ports (22 SSH, 80 VLESS, 8000 dashboard) with UFW:
   ```bash
   sudo ufw allow 22/tcp
   sudo ufw allow 80/tcp
   sudo ufw allow 8000/tcp
   sudo ufw enable
   ```
   ```
   sudo chmod -R 777 /home/ubuntu
   sudo chmod 666 /var/run/docker.sock
   sudo chmod 777 /var/run/docker.sock
   ````

   
4. Ensure Port 80 is free:
   ```bash
   sudo lsof -i :80
   ```
   If something is bound (nginx/apache), stop/disable it:
   ```bash
   sudo systemctl stop nginx apache2
   sudo systemctl disable nginx apache2
   ```
5. Install Marzban panel:
   ```bash
   sudo bash -c "$(curl -sL https://github.com/Gozargah/Marzban-scripts/raw/master/marzban.sh)" @ install
   ```
   Create admin creds when prompted, then open `http://YOUR_VPS_IP:8000/dashboard`.
6. (Optional) Create/rotate an admin user via CLI if you skipped the prompt or need a new one:
   ```bash
   sudo docker exec -it marzban bash -lc "marzban cli create-user --username admin --password 'StrongPass123' --expire 0 --data-limit 0"
   ```
   Replace `admin`/`StrongPass123` as desired. `--expire 0` and `--data-limit 0` mean no limits.

### 3) Marzban Core Config (Port 80, clean logging)
Replace Core Settings with:

```json
{
  "log": {
    "loglevel": "warning"
  },
  "routing": {
    "domainStrategy": "IPIfNonMatch",
    "rules": [
      {
        "type": "field",
        "ip": ["geoip:private"],
        "outboundTag": "BLOCK"
      }
    ]
  },
  "inbounds": [
    {
      "tag": "VLESS_WS_80",
      "listen": "0.0.0.0",
      "port": 80,
      "protocol": "vless",
      "settings": {
        "clients": [],
        "decryption": "none",
        "fallbacks": []
      },
      "streamSettings": {
        "network": "ws",
        "security": "none",
        "wsSettings": {
          "path": "/",
          "headers": {}
        }
      }
    }
  ],
  "outbounds": [
    { "protocol": "freedom", "tag": "DIRECT" },
    { "protocol": "blackhole", "tag": "BLOCK" }
  ]
}
```

Save, then Restart Core.

### 4) Create the Inbound (listener)
- Inbounds → **Create Inbound**
- Tag: `MLBB-NoTLS` (any name)
- Protocol: `VLESS`
- Port: `80`
- Network: `ws`
- Path: `/`
- Security: `none`

**Link settings (for instant QR/URIs):**
- Server IP: `104.17.125.32` (any valid Cloudflare IP)
- SNI/Host: `vpn.yourdomain.com` (your proxied subdomain)

### 5) Client Configuration (v2rayNG / V2Box / Streisand)

| Setting | Value | Note |
|---------|-------|------|
| Address | 104.16.125.32 | Use a Cloudflare IP; rotate if slow |
| Port | 80 | Must match inbound |
| Protocol | VLESS | |
| UUID | From Marzban | Copy per-user |
| Network | ws | |
| Path | / | |
| Host | vpn.yourdomain.com | Your CF subdomain |
| SNI | vpn.yourdomain.com | Same as Host |
| TLS | None / off | No TLS overhead |

**Alternative Cloudflare IPs:** 104.16.125.32, 162.159.128.7, 104.21.78.140, 172.64.155.10. Change only the Address, keep Host/SNI.

---

## 🛠️ Troubleshooting
- Connection refused or core fails: Port 80 busy. Run `sudo lsof -i :80` and stop the offender.
- Connected but no internet: Cloudflare SSL mode must be **Flexible**, not Full/Strict.
- 404/403 errors: Path must be `/`; Host/SNI must exactly match your subdomain.
- Slow or high ping: Rotate the Cloudflare IP (Address field) to a different option above.

---

## ⚠️ Disclaimer
This guide is for educational and research purposes only. Use responsibly and comply with local laws and your ISP's terms of service.
