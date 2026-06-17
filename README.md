# 📊 Phase 2 Monitoring Setup: PLG Stack & Prometheus

Dokumentasi ini berisi panduan langkah demi langkah untuk melakukan instalasi dan konfigurasi sistem monitoring terpusat menggunakan **Prometheus, Loki, dan Grafana (PLG Stack)** beserta konfigurasi *alerting* via Telegram.

## 🏗 Arsitektur Sistem

Sistem ini melibatkan dua Virtual Machine (VM) pada infrastruktur AWS EC2:
* **VM1 (Server Aplikasi):** Menjalankan Node Exporter (Metrik CPU/RAM) dan Promtail (Log Forwarder) secara *native* agar ringan dan tidak membebani aplikasi (Aimeos/Nginx).
* **VM2 (Server Monitoring):** Menjalankan Prometheus, Loki, dan Grafana di dalam *container* menggunakan Docker Compose.

---

## 🚀 Tahap 1: Instalasi Agen di VM1 (Server Aplikasi)

### 1. Instalasi Node Exporter
Node Exporter digunakan untuk mengekspor metrik perangkat keras (CPU, RAM, Disk). Node Exporter akan berjalan secara otomatis di latar belakang pada Port `9100`.

```bash
sudo apt update
sudo apt install prometheus-node-exporter unzip -y

```

### 2. Instalasi Promtail

Promtail bertugas membaca log dari Nginx dan aplikasi Laravel, lalu mengirimkannya ke Loki di VM2.

```bash
wget [https://github.com/grafana/loki/releases/download/v2.9.0/promtail-linux-amd64.zip](https://github.com/grafana/loki/releases/download/v2.9.0/promtail-linux-amd64.zip)
unzip promtail-linux-amd64.zip
sudo mv promtail-linux-amd64 /usr/local/bin/promtail
sudo chmod +x /usr/local/bin/promtail

```

### 3. Konfigurasi Promtail

Buat direktori dan file konfigurasi Promtail.

```bash
sudo mkdir -p /etc/promtail
sudo nano /etc/promtail/config.yml

```

Tambahkan konfigurasi berikut. **Penting:** Ganti `<IP_PRIVATE_VM2>` dengan IP Private server monitoring (VM2).

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://<IP_PRIVATE_VM2>:3100/loki/api/v1/push

scrape_configs:
  - job_name: nginx
    static_configs:
    - targets:
        - localhost
      labels:
        job: nginx
        __path__: /var/log/nginx/*log

  - job_name: laravel_aimeos
    static_configs:
    - targets:
        - localhost
      labels:
        job: laravel
        __path__: /home/ubuntu/aimeos/myshop/storage/logs/*.log

 - job_name: syslog
    static_configs:
    - targets:
        - localhost
      labels:
        job: syslog
        __path__: /var/log/syslog

 - job_name: auth.log
    static_configs:
    - targets:
        - localhost
      labels:
        job: auth.log
        __path__: /var/log/auth.log

```

### 4. Setup Promtail sebagai Systemd Service

Agar Promtail dapat berjalan otomatis 24/7 dan *restart* saat *reboot*, buat *service* baru.

```bash
sudo nano /etc/systemd/system/promtail.service

```

Isi dengan konfigurasi berikut:

```ini
[Unit]
Description=Promtail agent
After=network.target

[Service]
User=root
ExecStart=/usr/local/bin/promtail -config.file=/etc/promtail/config.yml
Restart=always

[Install]
WantedBy=multi-user.target

```

Aktifkan dan jalankan *service* Promtail:

```bash
sudo systemctl daemon-reload
sudo systemctl enable promtail
sudo systemctl start promtail

```

---

## 🛠 Tahap 2: Instalasi Inti di VM2 (Server Monitoring)

### 1. Instalasi Docker & Docker Compose

```bash
sudo apt update
sudo apt install docker.io docker-compose-v2 -y
sudo usermod -aG docker $USER

```

> **Catatan:** Setelah menjalankan perintah di atas, keluar dari sesi SSH (`exit`), lalu masuk kembali agar izin grup Docker diterapkan.

### 2. Konfigurasi Prometheus

Prometheus perlu diinstruksikan untuk menarik (*scrape*) data dari VM1.

```bash
mkdir -p ~/monitoring/config
cd ~/monitoring
nano config/prometheus.yml

```

Isi konfigurasi. **Penting:** Ganti `<IP_PRIVATE_VM1>` dengan IP Private server aplikasi.

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node_exporter_vm1'
    static_configs:
      - targets: ['<IP_PRIVATE_VM1>:9100']

```

### 3. Setup Docker Compose Stack

Buat file `docker-compose.yml` di direktori `~/monitoring`.

```bash
nano docker-compose.yml

```

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./config/prometheus.yml:/etc/prometheus/prometheus.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
    ports:
      - "9090:9090"
    restart: unless-stopped

  loki:
    image: grafana/loki:2.9.0
    container_name: loki
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=xxx
      - GF_SECURITY_ADMIN_PASSWORD=xxx
    restart: unless-stopped

```

Jalankan *stack* monitoring:

```bash
docker compose up -d

```

---

## 🔒 Tahap 3: Pengaturan AWS Security Group

Untuk memastikan komunikasi aman dan cepat tanpa biaya *bandwidth* internet (menggunakan IP Private), konfigurasikan AWS Security Group (SG) sebagai berikut:

### 1. Security Group VM1 (Aplikasi)

* **Inbound Rule:** Custom TCP | Port `9100` (Node Exporter)
* **Source:** IP Private VM2 (Contoh: `172.31.xx.xx/32`)
* *Tujuan: Memastikan hanya VM2 yang dapat membaca metrik sistem VM1.*



### 2. Security Group VM2 (Monitoring)

* **Inbound Rule 1:** Custom TCP | Port `3100` (Loki)
* **Source:** IP Private VM1 (Contoh: `172.31.yy.yy/32`)


* **Inbound Rule 2:** Custom TCP | Port `3000` (Grafana Dashboard)
* **Source:** Anywhere-IPv4 (`0.0.0.0/0`)
* *Tujuan: Mengizinkan akses dashboard Grafana melalui browser dari publik.*



---

## 📈 Tahap 4: Integrasi Data ke Grafana (VM2)

1. Akses Grafana melalui browser: `http://<IP_PUBLIC_VM2>:3000`
2. Login menggunakan kredensial yang diset pada Docker Compose:
* Username: `xxx`
* Password: `xxx`



### Menambahkan Data Source

1. Akses menu **Connections** > **Data sources** > **Add data source**.
2. **Prometheus:**
* Pilih **Prometheus**.
* Set URL: `http://prometheus:9090` (Karena berada dalam *network* Docker yang sama).
* Klik **Save & Test**.


3. **Loki:**
* Pilih **Loki**.
* Set URL: `http://loki:3100`.
* Klik **Save & Test**.



### Import Dashboard

1. Akses menu **Dashboards** > **New** > **Import**.
2. Masukkan ID **1860** (Node Exporter Full) lalu klik **Load**.
3. Pilih Data Source **Prometheus** yang baru saja dibuat.
4. Klik **Import**.

---

## 🚨 Tahap 5: Setup Alerting (Telegram)

### 1. Integrasi Bot Telegram

1. Buat bot baru melalui `@BotFather` di Telegram dan simpan **API Token**.
2. Dapatkan **Chat ID** Anda melalui bot `@userinfobot`.
3. Di Grafana, akses menu **Alerting** > **Contact points** > **Add contact point**.
4. Pilih tipe integrasi **Telegram**, lalu masukkan **Token** dan **Chat ID**.

### 2. Konfigurasi Alert Rules

Konfigurasikan Folder dan *Evaluation Group* terlebih dahulu, kemudian buat *rule* berikut:

#### 2.1. Alert CPU

* **Name:** Alert CPU
* **Query (PromQL):**
```promql
avg by (instance, hostname, site, env) (
  100 - (
    rate(node_cpu_seconds_total{mode="idle"}[5m]) * 100
  )
)

```


* **Threshold:**
* Warning: `> 80`
* Pending period: `2m`
* Keep firing for: `2m`


* **Custom Annotation:** `CPU usage on {{ $labels.hostname }} reached {{ printf "%.2f" $values.A.Value }}%`

#### 2.2. Alert Memory

* **Name:** Alert Memory
* **Query (PromQL):**
```promql
avg by (instance, hostname, site, env) (
  100 * (
    1 - (
      node_memory_MemAvailable_bytes
      /
      node_memory_MemTotal_bytes
    )
  )
)

```


* **Threshold:**
* Warning: `> 80`
* Pending period: `3m`
* Keep firing for: `3m`


* **Custom Annotation:** `Memory usage on {{ $labels.hostname }} reached {{ printf "%.2f" $values.A.Value }}%`

#### 2.3. Alert Disk

* **Name:** Alert Disk
* **Query (PromQL):**
```promql
100 * (
  node_filesystem_size_bytes{fstype=~"ext4|xfs", mountpoint!="/boot"}
  - node_filesystem_avail_bytes{fstype=~"ext4|xfs", mountpoint!="/boot"}
)
/
node_filesystem_size_bytes{fstype=~"ext4|xfs", mountpoint!="/boot"}

```


* **Threshold:**
* Warning: `> 80`
* Pending period: `5m`
* Keep firing for: `10m`


* **Custom Annotation:** `Disk usage on {{ $labels.hostname }} reached {{ printf "%.2f" $values.A.Value }}%` *(Catatan: Anotasi disesuaikan untuk Disk).*

---

## 📌 Glosarium Alerting

* **Pending Period (`FOR`):** Waktu minimum suatu kondisi (*threshold*) harus terpenuhi secara terus-menerus sebelum status *alert* berubah dari `pending` menjadi `firing`.
* *Fungsi:* Menyaring lonjakan data sesaat (*spike*) dan menghindari *alert noise*.


* **Keep Firing For (`Grace Period / Hold`):** Waktu di mana sebuah *alert* tetap dianggap berstatus `firing` meskipun kondisi metrik sudah kembali normal.
* *Fungsi:* Mencegah *flapping* (*alert* nyala-mati terlalu cepat) dan memberi waktu untuk memastikan sistem benar-benar telah stabil.
