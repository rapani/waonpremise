# WhatsApp Gateway On-Premise  — RLRADIUS

WhatsApp Gateway ON-PREMISE / Serever Mandiri untuk aplikasi billing [RLRADIUS](https://rlradius.id).  
Berjalan di VPS Ubuntu dengan **Node.js**, **PM2**, **Apache2**, dan **Let's Encrypt (SSL/WSS)**.

## Fitur

- Kirim pesan WhatsApp via HTTP API (`POST /send-message`)
- Pairing & cek status via WebSocket (QR code)
- Integrasi dengan panel RLRADIUS (vendor **ON-PREMISE**)

## Arsitektur

```
Browser / RLRADIUS (HTTPS)
    │
    ├─► https://wa.domainanda.com/             ──► Apache :443 ──► Node.js :8000 (HTTP API)
    │
    └─► wss://wa.domainanda.com/ws             ──► Apache :443 ──► Node.js :8001 (WebSocket)
```

Port `8000` dan `8001` hanya diakses **localhost** (internal VPS). Yang terbuka ke publik cukup port **443**.

## Prasyarat

| Item | Keterangan |
|------|------------|
| VPS | Ubuntu 20.04 / 22.04 / 24.04 LTS |
| RAM | Minimal 1 GB |
| Domain | Subdomain (contoh: `wa.domainanda.com`) → A record ke IP VPS |
| Panel RLRADIUS | Akses HTTPS (wajib untuk WebSocket `wss://`) |

> **Penting:** Let's Encrypt membutuhkan **domain**. Sertifikat tidak bisa dipasang hanya dengan IP address.

---

## Langkah 1 — Login VPS

```bash
sudo su
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl git build-essential python3 ca-certificates
```

---

## Langkah 2 — Install Node.js 20 LTS

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
node -v
npm -v
```

---

## Langkah 3 — Clone repo dari GitHub

```bash
sudo mkdir -p /opt
cd /opt
sudo git clone https://github.com/rapani/waonpremise.git
cd /opt/waonpremise
mkdir -p creds
chmod 777 creds
chmod 777 /opt/waonpremise
chmod 644 app.js package.json
```

---

## Langkah 4 — Install dependency

```bash
cd /opt/waonpremise
npm install
```

---

## Langkah 5 — Jalankan permanen dengan PM2

```bash
sudo npm install -g pm2
cd /opt/waonpremise
pm2 start app.js --name wa
pm2 save
pm2 startup
```

Perintah berguna:

```bash
pm2 status
pm2 logs wa
pm2 restart wa
```

Gateway listen di localhost:

| Service | Port internal |
|---------|---------------|
| HTTP API | `8000` |
| WebSocket | `8001` |

---

## Langkah 6 — Setup DNS (Subdomain)

Di panel domain / web hosting Anda:

1. Buat **subdomain**, contoh: `wa.domainanda.com`
2. Buat record **A** → arahkan ke **IP VPS**
3. Tunggu propagasi DNS (30 menit – beberapa jam)

Ganti semua `wa.domainanda.com` di panduan ini sesuai subdomain Anda.

Cek DNS:

```bash
ping wa.domainanda.com
```

---

## Langkah 7 — Install Apache & Let's Encrypt

### 7.1 Install Apache2

```bash
sudo apt install -y apache2
sudo systemctl enable apache2
sudo systemctl start apache2
sudo systemctl status apache2
```

### 7.2 Aktifkan modul yang dibutuhkan

```bash
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod proxy_wstunnel
sudo a2enmod ssl
sudo a2enmod headers
sudo a2enmod rewrite
sudo systemctl restart apache2
```

### 7.3 VirtualHost port 80 (sebelum SSL)

```bash
sudo nano /etc/apache2/sites-available/wa-gateway.conf
```

Isi file dipakai sementara untuk verifikasi Let's Encrypt **hanya port 80**:

```apache
<VirtualHost *:80>
    ServerName wa.domainanda.com

    DocumentRoot /var/www/html

    <Directory /var/www/html>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

Aktifkan site:

```bash
sudo a2ensite wa-gateway.conf
sudo a2dissite 000-default.conf
sudo apache2ctl configtest
sudo systemctl reload apache2
```

### 7.4 Install Certbot (Let's Encrypt)

```bash
sudo apt install -y certbot python3-certbot-apache
sudo certbot --apache -d wa.domainanda.com
```

Certbot otomatis membuat file SSL:

```
/etc/apache2/sites-available/wa-gateway-le-ssl.conf
/etc/apache2/sites-enabled/wa-gateway-le-ssl.conf   (symlink)
```

Sertifikat tersimpan di:

```
/etc/letsencrypt/live/wa.domainanda.com/fullchain.pem
/etc/letsencrypt/live/wa.domainanda.com/privkey.pem
```

Cek auto-renew:

```bash
sudo certbot renew --dry-run
```

### 7.5 Konfigurasi Apache final — pakai `wa-gateway.conf` saja

Setelah Certbot selesai, **nonaktifkan dan hapus** file SSL bawaan Certbot (`wa-gateway-le-ssl.conf`). File itu berisi `DocumentRoot /var/www/html`.

```bash
sudo a2dissite wa-gateway-le-ssl.conf
sudo rm -f /etc/apache2/sites-enabled/wa-gateway-le-ssl.conf
```

Edit **`wa-gateway.conf`** — satu file untuk port 80 dan 443:

```bash
sudo nano /etc/apache2/sites-available/wa-gateway.conf
```

Ganti isi file dengan:

```apache
<VirtualHost *:80>
    ServerName wa.domainanda.com
    RewriteEngine On
    RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [R=301,L]
</VirtualHost>

<IfModule mod_ssl.c>
<VirtualHost *:443>
    ServerName wa.domainanda.com
    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/wa.domainanda.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/wa.domainanda.com/privkey.pem
    Include /etc/letsencrypt/options-ssl-apache.conf
    ProxyPreserveHost On
    ProxyRequests Off
    # WebSocket — HARUS di atas ProxyPass /
    ProxyPass        /ws ws://127.0.0.1:8001/
    ProxyPassReverse /ws ws://127.0.0.1:8001/
    # HTTP API
    ProxyPass        / http://127.0.0.1:8000/
    ProxyPassReverse / http://127.0.0.1:8000/
    ErrorLog  ${APACHE_LOG_DIR}/wa-ssl-error.log
    CustomLog ${APACHE_LOG_DIR}/wa-ssl-access.log combined
</VirtualHost>
</IfModule>
```

Pastikan hanya **`wa-gateway.conf`** yang aktif:

```bash
sudo a2ensite wa-gateway.conf
sudo apache2ctl -S
```

Harusnya hanya **satu** vhost `:443` untuk domain Anda (dari `wa-gateway.conf`).

Reload Apache:

```bash
sudo apache2ctl configtest
sudo systemctl reload apache2
```

### 7.6 Firewall (opsional, disarankan)

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

Port `8000` dan `8001` **tidak perlu** dibuka ke publik — cukup localhost + Apache proxy.

---

## Langkah 8 — Konfigurasi di Panel RLRADIUS

Buka **WhatsApp → Setting → Vendor ON-PREMISE**, isi:

| Field | Nilai contoh |
|-------|----------------|
| No. WhatsApp | `628123456789` |
| Domain yang mengarah ke IP VPS | `https://wa.domainanda.com` |
| PORT HTTPS | `443` |

> URL **wajib** diawali `https://`, bukan `http://`.  
---



## Langkah 9 — Perpanjang SSL (manual)

Auto-renew sudah aktif via Certbot. Jika perlu manual:

```bash
sudo certbot renew
sudo rm -f /etc/apache2/sites-enabled/wa-gateway-le-ssl.conf
sudo systemctl reload apache2
```

## Langkah 10 — Update gateway ke versi terbaru

```bash
cd /opt/waonpremise
pm2 stop wa
git pull origin main
npm install
pm2 restart wa
```

---

## Troubleshooting

### Network ERROR! di panel RLRADIUS
Pesan **"Network ERROR!"** biasanya muncul karena:
- Propagasi DNS belum selesai — domain belum dikenali jaringan Anda (tunggu beberapa jam)
- Port **443** diblokir di VPS, firewall cloud, atau router
- Node.js/PM2 tidak jalan (`pm2 status`)

### WebSocket / API gagal — cek log

```bash
sudo tail -f /var/log/apache2/wa-ssl-error.log
pm2 logs wa
```

### Certbot gagal verifikasi

- Pastikan DNS subdomain sudah mengarah ke IP VPS
- Port **80** terbuka dan Apache jalan
- Cek: `curl -I http://wa.domainanda.com`


## Lisensi

MIT — © [Rapani](https://github.com/rapani)

## Dukungan

Issue & bug report: [GitHub Issues](https://github.com/rapani/waonpremise/issues)
