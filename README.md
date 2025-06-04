# ✅ Complete Deployment Guide: VLESS + WS + TLS + CDN + WARP

---

## 🌐 1. Register Domain (NameSilo)

**Registrar:** [https://www.namesilo.com](https://www.namesilo.com)

### 🪜 Steps:

1. Sign up for an account
2. Search for your desired domain 
3. Add to cart, proceed to payment
4. After registration, go to **"Manage My Domains"**
5. Set nameservers (Cloudflare will provide them in step 2)

---

## ☁️ 2. Configure CDN and DNS (Cloudflare)

**Website:** [https://dash.cloudflare.com](https://dash.cloudflare.com)

### 🪜 Add Site to Cloudflare:

1. Sign in → "Add a Site"
2. Enter your domain name (e.g., `wolverhamptonini.top`)
3. Choose the **Free Plan**

### 🧭 Add DNS Records:

| Type | Name        | Value (Your VPS IP) | Proxy   |
| ---- | ----------- | ------------------- | ------- |
| A    | Domain Name | YOUR.VPS.IP.ADDRESS | Proxied |

⚠️ Ensure the orange cloud icon is ON (CDN active)

### 🔁 Update NameSilo:

Set your domain's nameservers to the two Cloudflare NS values (e.g., `gabe.ns.cloudflare.com`, `candy.ns.cloudflare.com`)

### 🔐 SSL Settings:

* SSL Mode: **Full** or **Full (strict)**
* Enable:

  * Always Use HTTPS
  * Auto Minify
  * Brotli Compression

---

## 🖥️ 3. Purchase VPS (Nacknerd)

**Provider:** [https://nacknerd.com](https://nacknerd.com)

### 🔍 Recommended VPS Specs:

| Parameter | Value                      |
| --------- | -------------------------- |
| Location  | USA - Los Angeles          |
| RAM       | 1 GB+                      |
| Storage   | 10 GB+ SSD                 |
| OS        | Debian 11/12 or Ubuntu 20+ |
| Bandwidth | 500 GB+                    |
| IP Type   | Native IPv4 (no NAT)       |

### 🧳 After Purchase:

* Note your VPS IP, root password
* Connect via SSH:

```bash
ssh root@your-vps-ip
```

---

## 🔒 4. TLS Certificate via acme.sh

### Install acme.sh:

```bash
curl https://get.acme.sh | sh
source ~/.bashrc
```

### Prepare nginx directory:

```bash
mkdir -p /var/www/html
```

### Issue and install certificate:

```bash
~/.acme.sh/acme.sh --issue -d wolverhamptonini.top --webroot /var/www/html

~/.acme.sh/acme.sh --install-cert -d wolverhamptonini.top \
--key-file /etc/ssl/private/xray.key \
--fullchain-file /etc/ssl/certs/xray.crt \
--reloadcmd "systemctl restart xray"
```

---

## 🌐 5. Nginx Reverse Proxy

### Create HTML site:

```bash
echo "<h1>Hello from Xray + CDN</h1>" > /var/www/html/index.html
```

### Nginx Config: `/etc/nginx/conf.d/xray.conf`

```nginx
server {
    listen 443 ssl http2;
    server_name wolverhamptonini.top;

    ssl_certificate /etc/ssl/certs/fullchain.cer;
    ssl_certificate_key /etc/ssl/private/private.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location /xz {
        proxy_redirect off;
        proxy_pass http://127.0.0.1:10000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;

        # Optional: hide real IP
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location / {
        root /var/www/html;
        index index.html;
    }
}
```

```bash
systemctl enable nginx && systemctl restart nginx
```

---

## 📦 6. Install Xray Core

```bash
bash <(curl -Ls https://github.com/XTLS/Xray-install/raw/main/install-release.sh)
```

---

## ⚙️ 7. Configure Xray: `/usr/local/etc/xray/config.json`

```json
{
  "log": { "loglevel": "warning" },
  "dns": {
    "servers": ["https+local://dns.google/dns-query"]
  },
  "inbounds": [
    {
      "port": 10000,
      "listen": "127.0.0.1",
      "protocol": "vless",
      "settings": {
        "clients": [
          { "id": "YOUR_UUID", "flow": "" }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "ws",
        "security": "none",
        "wsSettings": {
          "path": "/xz"
        }
      }
    }
  ],
  "outbounds": [ { "protocol": "freedom" } ]
}
```

🔑 Generate UUID:

```bash
uuidgen
```

Replace `YOUR_UUID` with the generated value.

---

## 🔌 8. Enable systemd

```bash
systemctl daemon-reload
systemctl enable xray
systemctl restart xray
```

---

## 🌍 9. Setup WARP (Cloudflare)

### 🧱 Install WARP script:

```bash
bash <(curl -fsSL https://git.io/warp.sh) install
bash <(curl -fsSL https://git.io/warp.sh) wg4
```

```bash
curl --interface wgcf https://www.google.com
```

If successful, you're using WARP IP.

No SOCKS5 setup needed unless manually required.

---

## 📱 10. Client Configuration (v2rayN, etc.)

### VLESS URI:

```
vless://YOUR_UUID@wolverhamptonini.top:443?encryption=none&security=tls&type=ws&host=domainname&path=%2Fxz#WolverCDN
```

---

## 🔐 Optional: Install Fail2Ban (Prevent SSH brute-force)

```bash
apt install -y fail2ban
systemctl enable fail2ban
systemctl start fail2ban
```

---
