# STEP BY STEP SETUP GRAFANA DASHBOARD

> This tutorial for create system monitoring using grafana and prometheus node exporter

## B1. Build grafana

### Install grafana

```zsh
sudo apt-get install -y apt-transport-https
sudo apt-get install -y software-properties-common wget
sudo wget -q -O /usr/share/keyrings/grafana.key https://apt.grafana.com/gpg.key
echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get install grafana
sudo apt-get install grafana-enterprise
```


### Config grafana at  <b>/etc/grafana/grafana.ini</b>

```zsh
sudo vi /etc/grafana/grafana.ini
```

config port tại dòng <b>http_port = 8012</b>

### Mở firewall bằng lệnh

```zsh
sudo ufw allow <port>
```


### Start grafana

```zsh
sudo systemctl daemon-reload
sudo systemctl start grafana
```


## B2. Build prometheus

### Install prometheus

```zsh
 wget https://github.com/prometheus/prometheus/releases/download/v2.42.0/prometheus-2.42.0.linux-amd64.tar.gz
 tar -xvf prometheus-2.42.0.linux-amd64.tar.gz
 sudo mv prometheus-2.42.0.linux-amd64/prometheus /usr/local/bin/
 sudo mv prometheus-2.42.0.linux-amd64/promtool /usr/local/bin/
 sudo useradd --no-create-home --shell /bin/false prometheus

```

### Setup user

```zsh
 sudo groupadd --system prometheus
 sudo usermod -aG prometheus prometheus
 sudo mkdir /var/lib/prometheus
 sudo chown prometheus:prometheus /var/lib/prometheus
```


### Config prometheus
> tạo thư mục config prometheus và file prometheus.yml

```zsh
sudo mkdir /etc/prometheus
sudo vi /etc/prometheus/prometheus.yml
```

> Nội dung file config

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node_exporter_222'
    static_configs:
      - targets: ['10.1.10.222:8014']
    metrics_path: '/metrics'
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        replacement: node_exporter
      - source_labels: [__address__]
        target_label: __address__
        replacement: 10.1.10.222:8014
```

### Start prometheus bằng systemctl

> tạo file prometheus.service

```zsh
sudo vi /etc/systemd/system/prometheus.service
```

> cần run tại port 8016 và config file ở trên

```yaml
[Unit]
Description=Prometheus Monitoring

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file=/etc/prometheus/prometheus.yml \
    --web.listen-address=:8016 \
    --storage.tsdb.path /var/lib/prometheus

[Install]
WantedBy=multi-user.target
```

> Start prometheus

```zsh
sudo systemctl daemon-reload
sudo systemctl start prometheus.service
sudo systemctl status prometheus.service
```

## B3. Setup Node exporter trên từng máy cân Monitoring

### Install Node Exporter

```zsh
wget https://github.com/prometheus/node_exporter/releases/download/v1.5.0/node_exporter-1.5.0.linux-amd64.tar.gz
tar xvzf node_exporter-1.5.0.linux-amd64.tar.gz
sudo cp node_exporter-1.5.0.linux-amd64/node_exporter  /usr/local/bin
```

### Config Node exporter

```zsh
sudo vi /etc/systemd/system/node_exporter.service
```

> file config

```yaml
[Unit]
Description=Node Exporter Full
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter \
  --collector.cpu \
  --collector.diskstats \
  --collector.netdev \
  --collector.systemd \
  --collector.processes \
  --web.listen-address=:8014

[Install]
WantedBy=multi-user.target
```

> --colector : lấy dữ liệu cho trước vào metrics, web.listen: quy định port của node-exporter

> config user, firewall

```zsh
sudo useradd -rs /bin/false node_exporter
sudo ufw allow 8014
```

### Start node_exporter

```zsh
 sudo systemctl daemon-reload
 sudo systemctl start node_exporter.service  && sudo systemctl status node_exporter
```

### ADD job vào Prometheus

> quay lại máy chứa prometheus, thêm job vào trong file prometheus.yml

```yaml
- job_name: 'node_exporter_222'
    static_configs:
      - targets: ['10.1.10.222:8014']
    metrics_path: '/metrics'
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        replacement: node_exporter
      - source_labels: [__address__]
        target_label: __address__
        replacement: 10.1.10.222:8014
```

## B4. SETUP DASHBOARD

 Vào browser máy client, truy cập ip, port đã config ở grafana, ví dụ 10.1.10.222:8012 và Đăng nhập vào grafana
> NOTE: Mặc định là admin, admin

 Setting -> Add data source -> Prometheus > Nhập ip và port của prometheus, ví dụ 10.1.10.222:8016
 Tạo dashboard, sử dụng câu query theo node_exporter

### Examles:

Query CPU: 
```python 
100 - (avg by (instance) (irate(node_cpu_seconds_total{mode="idle",job="node_exporter_218"}[5m])) * 100)
```

Query RAM: 
```py
100 - ((node_memory_MemFree_bytes{job="node_exporter_218"} + node_memory_Buffers_bytes{job="node_exporter_218"} + node_memory_Cached_bytes{job="node_exporter_218"}) / node_memory_MemTotal_bytes{job="node_exporter_218"}) * 100
```

Query Disk usage: 
```py
100 - (node_filesystem_avail_bytes{mountpoint="/",job="node_exporter_218" } / node_filesystem_size_bytes{mountpoint="/",job="node_exporter_218"} ) * 100
```


