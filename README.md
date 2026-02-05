# Centralized Log Monitoring with Grafana, Loki, Promtail, Prometheus, and Alerts with Docker Containerization

Di sebuah IT infarstruktur yang modern, sebagai seorang DevOps Engineer meng handle banyak server, yang digunakan untuk tempat aplikasi dan service berjalan. Setiap server membuat logs (seperti system errors, application events, access records) dan (CPU, Memory, Disk Space, etc). manage resource secara individu itu akan sulit karena:
- Butuh check logs dari masing masing server jika terjadi masalah
- Logs mungkin bisa hilang atau terhapus
- Sulit mencari masalah jika lebih dari 1 server

Centralized logging mengatasi masalah tersebut, seperti menyimapan server logs dan metrics dalam 1 tempat. dimana kita bisa analyze, visualize, and alert ke masing masing server menggunakan 1 aplikasi opensource.

Berikut masing-masing role dari tools yg digunakan :
1. Grafana - sebuah alat dashboard visual, untuk monitoring traffic, metrics, dan logs. membuatnya menjadi sebuah graphs, chart, tables yang mudah dilihat.
2. Loki - Sistem agregasi log yang mirip dengan Elasticsearch tetapi lebih ringan. Sistem ini menyimpan log dari beberapa server dan terintegrasi dengan sempurna dengan Grafana.
3. Promtail - Sebuah agen yang diinstal pada server yang mengumpulkan log (dari berkas seperti /var/log/messages) dan mengirimkannya ke Loki.
4. Prometheus -  Sistem pengumpulan metrik dan pemberitahuan. Sistem ini mengumpulkan metrik CPU, memori, penggunaan disk, dan aplikasi dari server.
5. Node Exporter - Ini adalah agen yang berjalan di setiap server untuk mengumpulkan metrik tingkat sistem untuk Prometheus.
6. Alertmanager - Mengirimkan pemberitahuan (email, Slack, dll.) berdasarkan metrik atau pola log yang didefinisikan di Prometheus.

Dengan menggunakan Docker, kita dapat mengimplementasikan semua alat ini dengan cepat tanpa perlu khawatir tentang ketergantungan atau perbedaan sistem operasi. Docker juga memudahkan untuk menskalakan pemantauan ke beberapa mesin virtual (VM), seperti server Infra, dengan menjalankan Promtail dan Node Exporter di setiap server.

<center><b>Architecture Overview :</b></center>

![architecture](./images/architecture.png)

Environment Details:
- Monitoring Host: <b>idm8-monitoring - 10.0.0.223 </b>  

dan 2 server target untuk di monitoring:
- <b>10.0.1.189 idm9-target1</b>
- <b>10.0.4.14  idm10-target2</b>

<b>Operating System: `Debian Linux ` </b>

Prerequisites:

1. 1 Linux host untuk monitoring 2 server lainnya
2. Docker and Docker Compose
3. Internet Connectivity
4. Port yg perlu terbuka :
- 3000 (Grafana | Web Dashboard) 
- 3100 (Loki | Log Aggregation)
- 9090 (Prometheus | Metrics Collection)
- 9093 (Alert Manager | Alerts and Notification)
- 9100 (Node Exporter | System Metrics )

## Implementation

1. Steps Install Docker and Docker Compose
```bash
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```
- install steps 2
```bash
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
- install steps 3
```bash
sudo systemctl status docker
sudo systemctl start docker
sudo docker run hello-world
```
- remove container hello-world
```bash
sudo docker rm <container-id>
```

2. Create Project Directory and file with below command
```bash
sudo mkdir -p /root/monitoring_os/loki/chunks /root/monitoring_os/rules && sudo touch /root/monitoring_os/alertmanager.yml /root/monitoring_os/docker-compose.yml /root/monitoring_os/loki-config.yml /root/monitoring_os/prometheus.yml /root/monitoring_os/promtail-config.yml /root/monitoring_os/rules/alerts.yml
```

### File dan Konfigurasi untuk monitoring stack
Di direktori /root/monitoring_os, kita mengelola berkas konfigurasi dan folder berikut:
- docker-compose.yml – Menentukan semua layanan pemantauan: Grafana, Prometheus, Loki, Alertmanager, dan Node Exporter
- alertmanager.yml – Konfigurasi untuk pengalihan peringatan dan penerima
- prometheus.yml – Pengaturan global Prometheus, konfigurasi pengambilan data, dan aturan peringatan
- promtail-config.yml – Konfigurasi Promtail untuk pengiriman log dari /var/log ke Loki
- loki-config.yml – Konfigurasi Loki untuk penyimpanan log, pengindeksan, dan pengaturan server
- rules/alerts.yml – Aturan peringatan Prometheus untuk penggunaan CPU, memori, dan disk
- loki/chunks/ – Direktori untuk Loki menyimpan potongan log
- loki/rules/ – Direktori untuk aturan Loki (jika ada)

3. Prometheus Setup
file `prometheus.yml`
```bash
global:
  scrape_interval: 15s

rule_files:
  - "rules/alerts.yml"

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']

  - job_name: node_exporter
    static_configs:
      - targets:
          - 10.0.1.189      #idm9-target1
          - 10.0.4.14      # idm10-target2
          - 10.0.0.223      #idm8-monitoring
```

4. Alert Rules (rules/alerts.yml)
file `/root/monitoring_os/rules/alerts.yml`
```bash
groups:
  - name: system_alerts
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High CPU Usage on {{ $labels.instance }}"
          description: "CPU usage > 80% for more than 2 minutes."

      - alert: HighMemoryUsage
        expr: ((node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes) * 100 > 80
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High Memory Usage on {{ $labels.instance }}"
          description: "Memory usage > 80% for more than 2 minutes."

      - alert: HighDiskUsage
        expr: ((node_filesystem_size_bytes{mountpoint="/"} - node_filesystem_avail_bytes{mountpoint="/"}) / node_filesystem_size_bytes{mountpoint="/"}) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High Disk Usage on {{ $labels.instance }}"
          description: "Disk usage > 85% on root filesystem."
```

5. Loki Setup & Configuration 
file `loki-config.yml`
```bash
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h
```

6. Promtail Configuration
  - File konfigurasi promtail ini butuh di implementasikan ke vm target atau Nodes. Copy it (file `promtail-config.yml`)
```bash
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://10.0.0.223:3100/loki/api/v1/push  # Central Loki server

scrape_configs:
  - job_name: system-logs
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          host: idm8-monitoring  # Change per server: idm8, idm9, idm10
          __path__: /var/log/*.log
```

7. Prometheus Configuration file `prometheus.yml`
```bash
global:
  scrape_interval: 15s

rule_files:
  - "rules/alerts.yml"

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: node_exporter
    static_configs:
      - targets:
          - 10.0.1.189 #idm9-target1
          - 10.0.4.14 # idm10-target2
          - 10.0.0.223 #idm8-monitoring
```

8. Alertmanager Configuration file alertmanager.yml
```bash
global:
  resolve_timeout: 5m

route:
  receiver: 'default-receiver'

receivers:
  - name: 'default-receiver'
```

9. Docker Compose file
```bash
version: "3.8"

services:
  node-exporter:
    image: prom/node-exporter
    container_name: node-exporter
    ports:
      - "9100:9100"
    restart: unless-stopped

  prometheus:
    image: prom/prometheus
    container_name: prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./rules:/etc/prometheus/rules
    ports:
      - "9090:9090"
    depends_on:
      - node-exporter
    restart: unless-stopped

  loki:
    image: grafana/loki:2.9.0
    container_name: loki
    volumes:
      - ./loki-config.yml:/etc/loki/config.yml
      - ./loki/chunks:/loki/chunks
      - ./loki/rules:/loki/rules
    ports:
      - "3100:3100"
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager
    container_name: alertmanager
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
    ports:
      - "9093:9093"
    restart: unless-stopped

  grafana:
    image: grafana/grafana
    container_name: grafana
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
      - loki
    restart: unless-stopped
```

10. jalankan command berikut untuk menggunakan stack docker
```bash
docker compose up -d 
```

11. Check container status up or down
```bash
docker ps -a
```

12. Install Node Exporter and Promtail di `kedua vm nodes`
```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz
tar xvfz node_exporter-1.10.2.linux-amd64.tar.gz
sudo mv node_exporter-1.10.2.linux-amd64/node_exporter /usr/local/bin/
```
steps 2
```bash
sudo tee /etc/systemd/system/node_exporter.service <<EOF
[Unit]
Description=Prometheus Node Exporter
After=network.target

[Service]
User=root
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=default.target
EOF
```
steps 3
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
curl http://localhost:9100/metrics
```
steps 4 - download Promtail
```bash
wget https://github.com/grafana/loki/releases/download/v2.9.0/promtail-linux-amd64.zip
sudo apt install unzip -y
unzip promtail-linux-amd64.zip
sudo mv promtail-linux-amd64 /usr/local/bin/promtail
sudo chmod +x /usr/local/bin/promtail
```
steps 5 - promtail config (copy isi file `promtail-config-node.yml` and paste di promtail-config.yml milik nodes jangan ubah nama file. di repo hanya contoh )
```bash
sudo mkdir -p /root/monitoring_os/
sudo vi /root/monitoring_os/promtail-config.yml
```
steps 6 - promtail systemd service 
```bash
sudo tee /etc/systemd/system/promtail.service <<EOF
[Unit]
Description=Promtail service
After=network.target

[Service]
ExecStart=/usr/local/bin/promtail -config.file /root/monitoring_os/promtail-config.yml
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```
steps 7
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now promtail
sudo journalctl -u promtail -f
```

13. Access URLs
   - Kita bisa akses monitoring GUI Grafana di host idm8-monitoring, menggunakan ip public karena saya menggunakan EC2 Instance AWS
   - Grafana (Dashboard & Visualization ): http://3.91.67.81:3000
![grafana-ss](./images/image1.png)

14. Prometheus (Metrics Collection & Target): http://3.91.67.81:9090
![prometheus-ss](./images/image2.png)

15. Loki (Log Aggregation): 
```bash
curl -s "http://3.91.67.81:3100/loki/api/v1/labels"
```
![loki-ss](/images/image3.png)

16. Alert Manager (Alerts & Notifications): http://3.91.67.81:9093
![alert-ss](./images/image4.png)

17. Node Exporter (System Metrics in Plain text)
![nodeexporter-ss](./images/image5.png)

### Sekarang kita sudah berhasil mengakses semua container dan dapat melihat data serta logs nya.

18. Grafana Setups
    - Sekarang akses grafana dengan credentials default username `admin` dan password `admin`

19. Add Prometheus Data Source: Navigate Connections -> Data Sources -> Add new data source -> prometheus url : http://10.0.0.223:9090 -> Click save & test
![ss-gf-datasource](./images/image6.png)
![ss-gf-test](./images/image7.png)

20. Add Loki Data Source: Navigate Connection -> Data Sources -> Add new data source -> Loki URL : http://10.0.0.223:3100 Click Save & Test
![ss-gf-loki](./images/image8.png)

21. Kita bisa check semua target hosts
![ss-check](./images/image9.png)

22. Configuring the dashboard 
    - CPU Usage Panel, Click menu Dashboard -> Create new Dashboard -> Add Visualization -> Klik prometheus 
    ![ss-createdashboard](./images/image10.png)
    - Query (Prometheus): 
    ```bash
    100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
    ```
    - legend: {{nodename}}
    - Unit: Percent (%)
    - Click: Apply
    ![ss-created-panel](./images/image11.png)
    - Save Dashboard -> Beri title -> Save
    - Click Dasboatd menu -> Click save dashboard untuk save panel di dashboard -> klik save
    - Click Title Dashboard
    ![ss-panel-ds](./images/image12.png)

23. Untuk membuat panel lengkap, bisa pakai import file `.json` yg udh saya upload dengan name file `template-dashboard.json`, 
- Click Dashboard menu -> Click import .
![import-dashboard](./images/image13.png) 
- Paste code json di kolom `Import via dashboard JSON model` -> Klik Load -> optional ganti title -> klik import
![import-json-dashboard](./images/image14.png)
- Seperti ini tampilannya, nanti bisa edit panelnya untuk melihat query nya dan option lainnya
![full-dashboard](./images/image15.png)

24. Konfigurasi Alert Based, untuk memberikan notif alert ketika sistem seperti CPU yang traffic nya naik, dan menyebabkan server bisa down.
- Click + New Alert rule -> isi name `Test memory 5%` -> Define query pilih prometheus lalu masukan query berikut -> Click Run Queries
  ```bash
  100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))
  ```
- Click + New Folder -> isi name `alert` -> Click New evaluation group -> isi `Every` lalu intervalnya 1m -> Click create 
- Pending period 1m -> keep firing for none 
- Configure notifications -> contact point `grafana-default-email` -> Click Save
![ss-detail1](./images/image16.png)
![ss-detail2](./images/image17.png)


## DOKUMENTASI PROJECT
![doc-1](./images/Screenshot_1.png)
![doc-2](./images/Screenshot_2.png)
![doc-3](./images/Screenshot_3.png)
![doc-4](./images/Screenshot_4.png)
![doc-5](./images/Screenshot_5.png)
![doc-6](./images/Screenshot_6.png)