<div align="center">

# 🚀 VLESS & Trojan WebSocket Tunnel
### Cloudflare ISP Bypass Guide

[![GitHub last commit](https://img.shields.io/github/last-commit/google/skia)](https://github.com)
[![Maintenance](https://img.shields.io/badge/Maintained%3F-yes-green.svg)](https://github.com)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](http://makeapullrequest.com)

Deploy **blazing-fast VLESS or Trojan over WebSocket** with Marzban + Cloudflare. Bypass ISP throttling by disguising traffic as CDN requests using the classic **Bug Host** trick. Two protocols, one guide — pick your balance of speed vs security.

[Key Features](#-key-features) • [Quick Start](#-quick-start) • [Architecture](#-architecture) • [VLESS Setup](#-vless-no-tls-setup) • [Trojan Setup](#-trojan-tls-setup) • [Troubleshooting](#-troubleshooting)

</div>

---

## ✨ Key Features
- 🔒 ISP-proof tunneling that looks like normal CDN traffic
- ⚡ **VLESS**: Zero TLS overhead — raw WebSocket on Port 80
- 🔐 **Trojan**: TLS-encrypted WebSocket on Port 443
- 🌐 Cloudflare shield — ISP only sees whitelisted CF IPs
- 🎮 Gaming ready — great for Mobile Legends, PUBG, more
- 📱 Multi-platform — v2rayNG, V2Box, Streisand, Nekobox
- 🆓 Free-tier friendly — runs on Cloudflare free plan
- 🎯 Dual protocol support — run both simultaneously

---

## 🏗️ Architecture

### VLESS No-TLS (Port 80)
```
┌─────────────┐      ┌──────────────┐      ┌────────────┐      ┌─────────────┐
│  Your App   │─────▶│     ISP      │─────▶│ Cloudflare │─────▶│  Your VPS   │
│  (Client)   │      │ (sees CF IP) │      │    CDN     │      │  (Port 80)  │
└─────────────┘      └──────────────┘      └────────────┘      └─────────────┘
                                            Flexible SSL
```

### Trojan TLS (Port 443)
```
┌─────────────┐      ┌──────────────┐      ┌────────────┐      ┌─────────────┐
│  Your App   │─────▶│     ISP      │─────▶│ Cloudflare │─────▶│  Your VPS   │
│  (Client)   │      │ (sees CF IP) │      │    CDN     │      │ (Port 443)  │
└─────────────┘      └──────────────┘      └────────────┘      └─────────────┘
                                            Full SSL        TLS Termination
                                                            on YOUR VPS
```

**Flow:** Client hits a public Cloudflare IP → ISP sees allowed CDN traffic → Cloudflare forwards to your VPS on Port 80 (VLESS) or Port 443 (Trojan).

---

## 📊 Protocol Comparison

| Feature | VLESS No-TLS | Trojan TLS |
|---------|-------------|------------|
| Port | 80 | 443 |
| Encryption | None (CF handles) | End-to-end TLS |
| Cloudflare SSL | Flexible | Full |
| Overhead | Minimal | ~5-10ms extra |
| Certificate | Not required | Required on VPS |
| ISP disguise | HTTP traffic | HTTPS traffic |
| Gaming latency | Best | Slightly higher |
| Security level | Basic | High |

---

## 🎯 Quick Start

```bash
# Install Marzban
sudo bash -c "$(curl -sL https://github.com/Gozargah/Marzban-scripts/raw/master/marzban.sh)" @ install

# Open dashboard
http://YOUR_VPS_IP:8000/dashboard
```

Then follow the protocol-specific setup below.

---

## 📋 Prerequisites

| Requirement | Specification | Notes |
|-------------|---------------|-------|
| 🖥️ VPS | Ubuntu 20.04+ | 1 CPU / 1GB RAM is fine |
| 🌐 Domain | Any TLD | Must use Cloudflare DNS |
| ☁️ Cloudflare | Free account | Keep it on the orange cloud |
| 🔐 Certificate | Only for Trojan | Cloudflare Origin CA or Let's Encrypt |

---

## 🛠️ Common VPS Preparation

### 1) Cloudflare Configuration
1. Dashboard → **DNS** → add **A records**
   - Name: `vpn` → your VPS IP → **Proxied** (orange cloud)
   - Name: `trojan` → your VPS IP → **Proxied** (orange cloud)
2. Dashboard → **SSL/TLS** → **Overview**
   - For VLESS: **Flexible**
   - For Trojan: **Full** (not Strict)

### 2) VPS Base Setup (Ubuntu 20.04+)
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl ca-certificates gnupg lsof
```

### 3) Install Docker
```bash
curl -fsSL https://get.docker.com | sudo sh
sudo systemctl enable --now docker
sudo usermod -aG docker $USER  # re-login to take effect
```

### 4) Open Firewall Ports
```bash
# Common ports
sudo ufw allow 22/tcp
sudo ufw allow 8000/tcp

# VLESS port
sudo ufw allow 80/tcp

# Trojan port
sudo ufw allow 443/tcp

sudo ufw enable
```

### 5) Docker Socket Permissions
```bash
sudo chmod 666 /var/run/docker.sock
sudo chmod 777 /var/run/docker.sock
```

### 6) Install Marzban
```bash
sudo bash -c "$(curl -sL https://github.com/Gozargah/Marzban-scripts/raw/master/marzban.sh)" @ install
```
Create admin creds when prompted, then open `http://YOUR_VPS_IP:8000/dashboard`.

### 7) Admin User (if needed)
```bash
marzban cli admin create --sudo
```

---

## ⚡ VLESS No-TLS Setup (Port 80)

### Cloudflare SSL: **Flexible**

### Core Configuration
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

Save, then **Restart Core**.

### Create VLESS Inbound
- **Tag:** `VLESS-NoTLS`
- **Protocol:** `VLESS`
- **Port:** `80`
- **Network:** `ws`
- **Path:** `/`
- **Security:** `none`

### VLESS Client Config

| Setting | Value | Note |
|---------|-------|------|
| Address | `104.16.125.32` | Cloudflare IP; rotate if slow |
| Port | `80` | |
| Protocol | `VLESS` | |
| UUID | From Marzban | Copy per-user |
| Network | `ws` | |
| Path | `/` | |
| Host | `vpn.yourdomain.com` | Your CF subdomain |
| SNI | `vpn.yourdomain.com` | Same as Host |
| TLS | `None / off` | No TLS overhead |

**Alternative Cloudflare IPs:** `104.16.125.32`, `162.159.128.7`, `104.21.78.140`, `172.64.155.10`

---

## 🔐 Trojan TLS Setup (Port 443)

### Cloudflare SSL: **Full**

### Step 1: Generate Cloudflare Origin Certificate
1. Cloudflare Dashboard → **SSL/TLS** → **Origin Server**
2. Click **Create Certificate**
3. Use these settings:
   - Private key type: `RSA (2048)`
   - Hostnames: `*.yourdomain.com, yourdomain.com`
4. Copy both **Origin Certificate** and **Private Key**

### Step 2: Install Certificate on VPS
```bash
# Create cert directory
sudo mkdir -p /var/lib/marzban/certs

# Create certificate file
sudo nano /var/lib/marzban/certs/cert.pem
# Paste Origin Certificate (including BEGIN/END lines)

# Create private key file
sudo nano /var/lib/marzban/certs/key.pem
# Paste Private Key (including BEGIN/END lines)

# Set permissions
sudo chmod 644 /var/lib/marzban/certs/cert.pem
sudo chmod 600 /var/lib/marzban/certs/key.pem
sudo chown -R root:root /var/lib/marzban/certs
```

### Step 3: Verify Certificate
```bash
openssl x509 -in /var/lib/marzban/certs/cert.pem -text -noout | grep -E "Subject:|DNS:|Not After"
```

Expected output: `DNS:*.yourdomain.com, DNS:yourdomain.com`

### Step 4: Mount Certs in Docker
Edit `docker-compose.yml`:
```bash
cd /opt/marzban
sudo nano docker-compose.yml
```

Add under `volumes:`:
```yaml
services:
  marzban:
    volumes:
      - /var/lib/marzban:/var/lib/marzban
      - /var/lib/marzban/certs:/var/lib/marzban/certs:ro
```

### Step 5: Core Configuration
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
      "tag": "TROJAN_WS_TLS",
      "listen": "0.0.0.0",
      "port": 443,
      "protocol": "trojan",
      "settings": {
        "clients": [],
        "fallbacks": []
      },
      "streamSettings": {
        "network": "ws",
        "security": "tls",
        "tlsSettings": {
          "serverName": "trojan.yourdomain.com",
          "certificates": [
            {
              "certificateFile": "/var/lib/marzban/certs/cert.pem",
              "keyFile": "/var/lib/marzban/certs/key.pem"
            }
          ]
        },
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

Save, then **Restart Core**.

### Step 6: Restart Marzban
```bash
marzban restart
```

### Step 7: Verify TLS
```bash
# Check listening
sudo lsof -i :443 -n -P

# Test TLS locally
curl -v -k https://127.0.0.1:443/ 2>&1 | grep -E "Connected|TLS|Server hello"
```

### Create Trojan Inbound
- **Tag:** `Trojan-WS-TLS`
- **Protocol:** `Trojan`
- **Port:** `443`
- **Network:** `ws`
- **Path:** `/`
- **Security:** `tls`
- **SNI:** `trojan.yourdomain.com`

### Trojan Client Config

| Setting | Value | Note |
|---------|-------|------|
| Address | `172.64.151.167` | Cloudflare IP; rotate if slow |
| Port | `443` | |
| Protocol | `Trojan` | |
| Password | UUID from Marzban | Copy per-user |
| Network | `ws` | |
| Path | `/` | |
| Host | `trojan.yourdomain.com` | Your CF subdomain |
| SNI | `trojan.yourdomain.com` | Same as Host |
| TLS | `On / tls` | |
| ALPN | `h2,http/1.1` | |

**Alternative Cloudflare IPs:** `172.64.151.167`, `104.21.78.140`, `172.64.155.10`, `104.16.125.32`

---

## 🔄 Running Both Protocols Simultaneously

Combine both inbounds in a single Core Config:

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
    },
    {
      "tag": "TROJAN_WS_TLS",
      "listen": "0.0.0.0",
      "port": 443,
      "protocol": "trojan",
      "settings": {
        "clients": [],
        "fallbacks": []
      },
      "streamSettings": {
        "network": "ws",
        "security": "tls",
        "tlsSettings": {
          "serverName": "trojan.yourdomain.com",
          "certificates": [
            {
              "certificateFile": "/var/lib/marzban/certs/cert.pem",
              "keyFile": "/var/lib/marzban/certs/key.pem"
            }
          ]
        },
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

Then create **two separate inbounds** in Marzban dashboard — one for each protocol.

---

## 🛠️ Troubleshooting

| Issue | Protocol | Fix |
|-------|----------|-----|
| Port 80/443 busy | Both | `sudo lsof -i :80` or `:443`, stop the offender |
| Connected but no internet | VLESS | Cloudflare SSL must be **Flexible**, not Full |
| Connected but no internet | Trojan | Cloudflare SSL must be **Full**, not Flexible |
| 404/403 errors | Both | Path must be `/`; Host/SNI must match subdomain exactly |
| SSL certificate error | Trojan | Verify cert has `*.yourdomain.com` in SAN field |
| Certificate permissions | Trojan | `cert.pem`: 644, `key.pem`: 600 |
| Xray listening IPv6 only | Both | Set `"listen": "0.0.0.0"` in core config |
| Slow or high ping | Both | Rotate Cloudflare IP in Address field |
| Certs not found in container | Trojan | Verify `docker-compose.yml` volume mount, rebuild container |
| Cloudflare 502 | Both | Check firewall, verify xray is running: `sudo docker ps` |

### Quick Diagnostic Commands
```bash
# Check all listening ports
sudo lsof -i :80 -n -P
sudo lsof -i :443 -n -P

# Check Marzban status
marzban status

# Check Xray logs
sudo docker exec marzban-marzban-1 cat /var/log/xray/error.log 2>/dev/null | tail -20

# Test VLESS locally
curl -v -H "Host: vpn.yourdomain.com" http://127.0.0.1:80/

# Test Trojan locally
curl -v -k https://127.0.0.1:443/
```

---

## ⚠️ Disclaimer
This guide is for educational and research purposes only. Use responsibly and comply with local laws and your ISP's terms of service.
