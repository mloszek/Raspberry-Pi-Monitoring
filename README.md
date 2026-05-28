# Monitoring Your Raspberry Pi with Prometheus and Grafana

This guide provides a step-by-step process to set up **Prometheus**, **Node Exporter**, and **Grafana** on a Raspberry Pi for monitoring system metrics like CPU, memory, and temperature on your local network. Ideal for DevOps beginners or Raspberry Pi enthusiasts exploring observability. 

Tested on a **Raspberry Pi** running **Raspbian GNU/Linux 12 (bookworm)**.

## Why Monitor Your Raspberry Pi?

- **Prometheus**: A time-series database for collecting and storing metrics.
- **Node Exporter**: Gathers system metrics (CPU, memory, disk, etc.) for Prometheus.
- **Grafana**: Visualizes metrics through customizable dashboards.

Together, these tools enable real-time performance tracking with insightful visualizations.

### Prerequisites

- Raspberry Pi with Raspbian installed.
- SSH enabled (`sudo raspi-config` > Interface Options > SSH > Enable).
- Basic terminal familiarity.
- Pi’s local IP address (find it with `hostname -I`).

## Step 1: Update Your Raspberry Pi

Ensure your system is up-to-date to prevent compatibility issues:

```bash
sudo apt update && sudo apt upgrade -y
```

## Step 2: Install Prometheus

Prometheus collects and stores metrics. We'll install it for the Pi’s ARMv7 architecture.

1. Create a Prometheus user for security:

   ```bash
   sudo useradd --no-create-home --shell /bin/false prometheus
   ```

2. Download Prometheus (replace `v2.53.0` with the latest `linux-armv7` version from [Prometheus downloads](https://prometheus.io/download/)):

   ```bash
   cd /tmp
   wget https://github.com/prometheus/prometheus/releases/download/v2.53.0/prometheus-2.53.0.linux-armv7.tar.gz
   tar xvfz prometheus-2.53.0.linux-armv7.tar.gz
   cd prometheus-2.53.0.linux-armv7
   ```

3. Install binaries:

   ```bash
   sudo cp prometheus promtool /usr/local/bin/
   sudo chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
   ```

4. Set up directories and configuration:

   ```bash
   sudo mkdir /etc/prometheus /var/lib/prometheus
   sudo chown prometheus:prometheus /etc/prometheus /var/lib/prometheus
   sudo nano /etc/prometheus/prometheus.yml
   ```

5. Paste the following configuration to scrape Prometheus and Node Exporter:

   ```yaml
   global:
     scrape_interval: 15s

   scrape_configs:
     - job_name: 'prometheus'
       static_configs:
         - targets: ['localhost:9090']
     - job_name: 'node'
       static_configs:
         - targets: ['localhost:9100']
   ```

   Save with `Ctrl+O`, `Enter`, then `Ctrl+X`.

6. Create a systemd service:

   ```bash
   sudo nano /etc/systemd/system/prometheus.service
   ```

   Paste:

   ```ini
   [Unit]
   Description=Prometheus
   Wants=network-online.target
   After=network-online.target

   [Service]
   User=prometheus
   Group=prometheus
   Type=simple
   ExecStart=/usr/local/bin/prometheus \
     --config.file /etc/prometheus/prometheus.yml \
     --storage.tsdb.path /var/lib/prometheus/ \
     --web.console.templates=/etc/prometheus/consoles \
     --web.console.libraries=/etc/prometheus/console_libraries

   [Install]
   WantedBy=multi-user.target
   ```

   Save, then enable and start the service:

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl start prometheus
   sudo systemctl enable prometheus
   ```

7. Verify Prometheus:
   - On a device on the same network, visit `http://<pi-local-ip>:9090` (replace `<pi-local-ip>` with your Pi’s IP).
   - Query `up` in the Prometheus UI to confirm it’s running.

## Step 3: Install Node Exporter

Node Exporter collects system metrics for Prometheus.

1. Download Node Exporter (replace `v1.8.2` with the latest `linux-armv7` version from [Prometheus downloads](https://prometheus.io/download/)):

   ```bash
   cd /tmp
   wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-armv7.tar.gz
   tar xvfz node_exporter-1.8.2.linux-armv7.tar.gz
   cd node_exporter-1.8.2.linux-armv7
   ```

2. Install and configure:

   ```bash
   sudo cp node_exporter /usr/local/bin/
   sudo useradd --no-create-home --shell /bin/false node_exporter
   sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
   ```

3. Create a systemd service:

   ```bash
   sudo nano /etc/systemd/system/node_exporter.service
   ```

   Paste:

   ```ini
   [Unit]
   Description=Prometheus Node Exporter
   Wants=network-online.target
   After=network-online.target

   [Service]
   User=node_exporter
   Group=node_exporter
   Type=simple
   ExecStart=/usr/local/bin/node_exporter

   [Install]
   WantedBy=multi-user.target
   ```

   Save, then enable and start the service:

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl start node_exporter
   sudo systemctl enable node_exporter
   ```

4. Verify Node Exporter:
   - Visit `http://<pi-local-ip>:9100/metrics` to see raw metrics (CPU, memory, etc.).
   - In Prometheus (`http://<pi-local-ip>:9090`), query `up` to confirm the `node` job is `1`.

## Step 4: Install Grafana

Grafana visualizes Prometheus metrics in dashboards.

1. Add the Grafana repository:

   ```bash
   sudo mkdir -p /etc/apt/keyrings/
   wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
   echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
   sudo apt-get update
   ```

2. Install and start Grafana:

   ```bash
   sudo apt-get install -y grafana
   sudo systemctl start grafana-server
   sudo systemctl enable grafana-server
   ```

3. Access Grafana:
   - Visit `http://<pi-local-ip>:3000` and log in with `admin`/`admin` (change the password when prompted).

4. Connect Grafana to Prometheus:
   - Go to **Configuration** (gear icon) > **Data Sources** > **Add data source** > **Prometheus**.
   - Set the URL to `http://localhost:9090`.
   - Click **Save & Test** (should confirm “Data source is working”).

5. Create a dashboard:
   - Go to **Dashboards** > Select **New** > **New Dashboard**.
   - Click big **+** icon in **Panel** section > Select **Configure visualisation**.
   - Select the Prometheus data source.
   - Try queries like:
     - `node_memory_MemFree_bytes` (Memory free bytes)
     - `node_hwmon_temp_celsius` (Pi temperature)
   - Save as “Pi Monitoring”.

## Step 5: Explore Your Metrics

- Open Grafana (`http://<pi-local-ip>:3000`) and view your “Pi Monitoring” dashboard.
- Example metrics to visualize:
  - **CPU**: `rate(node_cpu_seconds_total{mode="user"}[5m])`
  - **Memory**: `node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100`
  - **Temperature**: `node_hwmon_temp_celsius`

Add more panels to customize your dashboard and monitor your Pi’s performance.

## Troubleshooting

- **Prometheus not running?** Check logs: `sudo journalctl -u prometheus`
- **Node Exporter down?** Verify status: `sudo systemctl status node_exporter`
- **Grafana not loading?** Check status: `sudo systemctl status grafana-server`
- **Firewall issues?** Allow ports: `sudo ufw allow 9090`, `sudo ufw allow 9100`, `sudo ufw allow 3000`
