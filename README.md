# 🚀 VLESS-WebSocket-NoTLS (Cloudflare CDN Bypass)

![GitHub last commit](https://img.shields.io/github/last-commit/google/skia)
![Uptime Robot status](https://img.shields.io/uptimerobot/status/m778918-2888a)

A comprehensive guide to deploying a **VLESS over WebSocket (Port 80)** VPN tunnel using **Marzban** and **Cloudflare**. This setup is designed to bypass ISP throttling by disguising traffic as legitimate CDN requests (the "Bug Host" method).

---

## 📋 Prerequisites

Before you begin, ensure you have:
* **A VPS (Virtual Private Server):** Ubuntu 20.04+ (Recommended: 1 CPU / 1GB+ RAM).
* **A Domain Name:** Managed via Cloudflare.
* **Cloudflare Account:** Free tier is sufficient.

---

## 🛠️ Architecture

**The "Sandwich" Model:**
`[Client App]` ➔ `[ISP (sees Cloudflare IP)]` ➔ `[Cloudflare CDN]` ➔ `[Your VPS (Port 80)]`

* **Client Side:** Connects to a public Cloudflare IP (e.g., `104.17.x.x`) but requests your specific domain in the Host Header.
* **Cloudflare Side:** Receives the HTTPS request, handles the encryption, and forwards plain HTTP traffic to your VPS.
* **Server Side:** Listens on Port 80 (No TLS) for the WebSocket connection.

---

## ⚙️ Step 1: Cloudflare Configuration

1.  **DNS Records:**
    * Go to **DNS** > **Records**.
    * Add an **A Record**:
        * **Name:** `vpn` (or your preferred subdomain).
        * **IPv4:** Your VPS IP Address.
        * **Proxy Status:** ✅ **Proxied** (Orange Cloud).

2.  **SSL/TLS Settings (CRITICAL):**
    * Go to **SSL/TLS** > **Overview**.
    * Set encryption mode to **Flexible**.
    * *Why?* The connection between Cloudflare and your VPS must be unencrypted (Port 80), while the client connection remains secure.

---

## 🖥️ Step 2: VPS Preparation

SSH into your server and prepare the environment.

### 1. Clear Port 80
Ensure no web servers (Nginx/Apache) are occupying Port 80.
```bash
sudo lsof -i :80
# If a process is found (e.g., nginx), stop it:
sudo systemctl stop nginx
sudo systemctl disable nginx