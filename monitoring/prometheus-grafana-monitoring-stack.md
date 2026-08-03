<!--
Organized from: STEP 12 Full Monitoring Stack Prometheus Grafana TrueNAS Proxmox Homepage(2).txt
Source wording and technical details were preserved as closely as possible.
-->

> [!CAUTION]
> This is a historical implementation record. Verify versions, addresses, paths, and commands against the live environment before applying changes.

# STEP 13 Full Monitoring Stack: Prometheus, Grafana, Proxmox, TrueNAS, and Homepage

Final architecture

```text
Monitoring LXC
├── Prometheus
│   └── :9090
├── Grafana
│   └── :3000
├── Node Exporter
│   └── :9100
├── prometheus-pve-exporter
│   └── :9221
└── graphite_exporter
    ├── receives TrueNAS Graphite metrics on :2003
    └── exposes Prometheus metrics on :9108
```

Use a dedicated Debian 12 LXC called:

Monitoring

Recommended LXC specs:

```text
CPU: 2 cores
RAM: 2 GB minimum, 4 GB preferred
Disk: 20 to 32 GB
Unprivileged: Yes
Nesting: Enable if Grafana has namespace issues
Static IP: recommended
```

---
## PART 1: Create and prepare the Monitoring LXC

Inside the LXC:

```text
apt update && apt upgrade -y
apt install -y curl wget gnupg2 ca-certificates apt-transport-https software-properties-common nano vim tar unzip gpg netcat-openbsd
timedatectl set-timezone America/New_York
```

---
## PART 2: Install Prometheus manually

2.1 Create the Prometheus user correctly

Use a system user, not a normal user. This matters because Debian's prometheus-node-exporter package expects the prometheus account to be a system user.

```text
useradd --system --no-create-home --shell /usr/sbin/nologin prometheus
```

If you already created it wrong and it has UID 1000+, convert it:

```text
OLD_UID=$(id -u prometheus)
OLD_GID=$(id -g prometheus)
```

```text
NEW_UID=$(awk -F: 'BEGIN{for(i=100;i<999;i++) used[i]=0} {used[$3]=1} END{for(i=998;i>=100;i--) if(!used[i]){print i; exit}}' /etc/passwd)
NEW_GID=$(awk -F: 'BEGIN{for(i=100;i<999;i++) used[i]=0} {used[$3]=1} END{for(i=998;i>=100;i--) if(!used[i]){print i; exit}}' /etc/group)
```

```text
groupmod -g "$NEW_GID" prometheus
usermod -u "$NEW_UID" -g prometheus prometheus
```

find / -xdev \
  -path /lost+found -prune -o \
  \( -uid "$OLD_UID" -o -gid "$OLD_GID" \) \
  -exec chown -h prometheus:prometheus {} +

Confirm:

```text
id prometheus
```

You want UID under 1000, like:

uid=995(prometheus) gid=995(prometheus)

2.2 Create Prometheus folders

```text
mkdir -p /etc/prometheus
mkdir -p /var/lib/prometheus
chown prometheus:prometheus /etc/prometheus
chown prometheus:prometheus /var/lib/prometheus
```

2.3 Download Prometheus

```text
cd /tmp
```

PROM_VERSION=$(curl -s https://api.github.com/repos/prometheus/prometheus/releases/latest | grep tag_name | cut -d '"' -f 4 | sed 's/v//')

```text
wget https://github.com/prometheus/prometheus/releases/download/v${PROM_VERSION}/prometheus-${PROM_VERSION}.linux-amd64.tar.gz
```

```text
tar xvf prometheus-${PROM_VERSION}.linux-amd64.tar.gz
cd prometheus-${PROM_VERSION}.linux-amd64
```

Copy binaries:

```text
cp prometheus /usr/local/bin/
cp promtool /usr/local/bin/
```

```text
chown prometheus:prometheus /usr/local/bin/prometheus
chown prometheus:prometheus /usr/local/bin/promtool
```

Important: do not copy consoles or console_libraries. Prometheus 3.x may not include those folders, and they are not needed for this setup.

---
## PART 3: Configure Prometheus

Create config:

```text
nano /etc/prometheus/prometheus.yml
```

Start with this:

global:

```text
  scrape_interval: 15s
  evaluation_interval: 15s
```

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets:
          - "localhost:9090"

Set ownership:

```text
chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

Check config:

promtool check config /etc/prometheus/prometheus.yml

---
## PART 4: Create Prometheus systemd service

```text
nano /etc/systemd/system/prometheus.service
```

Paste:

[Unit]
Description=Prometheus Monitoring System
Wants=network-online.target
After=network-online.target

```text
[Service]
User=prometheus
Group=prometheus
Type=simple
```

ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.listen-address=0.0.0.0:9090 \
  --storage.tsdb.retention.time=30d

```text
Restart=always
```

```text
[Install]
WantedBy=multi-user.target
```

Start it:

```text
systemctl daemon-reload
systemctl enable --now prometheus
systemctl status prometheus --no-pager
```

Test:

```text
curl http://localhost:9090/-/ready
```

You want:

Prometheus Server is Ready.

---
## PART 5: Install Grafana

```text
apt install -y apt-transport-https software-properties-common wget gpg
mkdir -p /etc/apt/keyrings
```

```text
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor > /etc/apt/keyrings/grafana.gpg
```

```text
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" > /etc/apt/sources.list.d/grafana.list
```

```text
apt update
apt install -y grafana
```

The Grafana package normally creates the grafana user automatically.

Check:

```text
id grafana
```

---
## PART 6: Fix Grafana for Proxmox LXC

In this LXC, Grafana may fail with:

status=226/NAMESPACE

Use an LXC-safe custom systemd service.

Stop Grafana:

```text
systemctl stop grafana-server
```

Backup original service:

```text
cp /lib/systemd/system/grafana-server.service /root/grafana-server.service.backup
```

Create replacement service:

```text
nano /etc/systemd/system/grafana-server.service
```

Paste this exact version:

[Unit]
Description=Grafana instance
Documentation=http://docs.grafana.org
Wants=network-online.target
After=network-online.target

```text
[Service]
Type=simple
User=grafana
Group=grafana
WorkingDirectory=/usr/share/grafana
ExecStart=/usr/share/grafana/bin/grafana server --homepath=/usr/share/grafana --config=/etc/grafana/grafana.ini --packaging=deb cfg:default.paths.logs=/var/log/grafana cfg:default.paths.data=/var/lib/grafana cfg:default.paths.plugins=/var/lib/grafana/plugins cfg:default.paths.provisioning=/etc/grafana/provisioning
Restart=on-failure
RestartSec=5
LimitNOFILE=10000
TimeoutStopSec=20
```

```text
[Install]
WantedBy=multi-user.target
```

Fix permissions:

```text
mkdir -p /var/lib/grafana/plugins
mkdir -p /var/log/grafana
```

```text
chown -R grafana:grafana /var/lib/grafana
chown -R grafana:grafana /var/log/grafana
chown -R root:grafana /etc/grafana
chmod -R 750 /etc/grafana
```

Reload and start:

```text
systemctl daemon-reload
systemctl reset-failed grafana-server
systemctl enable --now grafana-server
sleep 5
systemctl status grafana-server --no-pager
```

Check port:

```text
ss -tulpn | grep 3000
```

You want:

*:3000 users:(("grafana",pid=...,fd=...))

Open:

http://MONITORING-LXC-IP:3000

Login:

```text
Username: admin
Password: admin
```

---
## PART 7: Add Prometheus to Grafana

In Grafana:

Connections > Data sources > Add data source > Prometheus

Use:

http://localhost:9090

Click:

Save & test

---
## PART 8: Install Node Exporter on Monitoring LXC

```text
apt install -y prometheus-node-exporter
```

If dpkg breaks with:

adduser: The user `prometheus' already exists, but is not a system user

then your prometheus user was created wrong. Convert it using the commands in Part 2.1, then run:

dpkg --configure -a
apt --fix-broken install -y

Start Node Exporter:

```text
systemctl enable --now prometheus-node-exporter
systemctl status prometheus-node-exporter --no-pager
```

Test:

```text
curl -s http://localhost:9100/metrics | head
```

---
## PART 9: Add Linux VMs to Prometheus

Confirmed working Node Exporter targets:

```text
localhost:9100
192.168.1.245:9100
192.168.1.240:9100
192.168.1.254:9100
192.168.1.239:9100
192.168.1.252:9100
```

Edit Prometheus config:

```text
nano /etc/prometheus/prometheus.yml
```

Use:

global:

```text
  scrape_interval: 15s
  evaluation_interval: 15s
```

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets:
          - "localhost:9090"

  - job_name: "linux-nodes"
    static_configs:
      - targets:
          - "localhost:9100"
        labels:
          instance: "monitoring-lxc"

      - targets:
          - "192.168.1.245:9100"
        labels:
          instance: "vm-245"

      - targets:
          - "192.168.1.240:9100"
        labels:
          instance: "vm-240"

      - targets:
          - "192.168.1.254:9100"
        labels:
          instance: "alma-mgmt"

      - targets:
          - "192.168.1.239:9100"
        labels:
          instance: "vm-239"

      - targets:
          - "192.168.1.252:9100"
        labels:
          instance: "vm-252"

Check and restart:

promtool check config /etc/prometheus/prometheus.yml
systemctl restart prometheus

Check:

http://MONITORING-LXC-IP:9090/targets

---
## PART 10: Install Node Exporter on other VMs

Debian / Ubuntu:

```text
sudo apt update
sudo apt install -y prometheus-node-exporter
sudo systemctl enable --now prometheus-node-exporter
```

AlmaLinux:

```text
sudo dnf install -y epel-release
sudo dnf install -y node_exporter
sudo systemctl enable --now node_exporter
```

openSUSE:

```text
sudo zypper refresh
sudo zypper search node_exporter
sudo zypper install prometheus-node_exporter
```

Then check service name:

```text
systemctl list-unit-files | grep -i exporter
```

---
## PART 11: Proxmox API Exporter

Install requirements:

```text
apt install -y python3 python3-pip python3-venv
```

Create venv:

python3 -m venv /opt/prometheus-pve-exporter

Install exporter:

```text
/opt/prometheus-pve-exporter/bin/pip install prometheus-pve-exporter
```

Create config:

```text
mkdir -p /etc/prometheus-pve-exporter
nano /etc/prometheus-pve-exporter/pve.yml
```

Use:

default:

```text
  user: prometheus@pve
  password: "YOUR_PASSWORD"
  verify_ssl: false
```

Set permissions:

```text
chmod 600 /etc/prometheus-pve-exporter/pve.yml
```

Create service:

```text
nano /etc/systemd/system/prometheus-pve-exporter.service
```

Use:

[Unit]
Description=Prometheus Proxmox VE Exporter
Documentation=https://github.com/prometheus-pve/prometheus-pve-exporter
Wants=network-online.target
After=network-online.target

```text
[Service]
Type=simple
ExecStart=/opt/prometheus-pve-exporter/bin/pve_exporter --config.file /etc/prometheus-pve-exporter/pve.yml
Restart=always
RestartSec=5
```

```text
[Install]
WantedBy=multi-user.target
```

Start:

```text
systemctl daemon-reload
systemctl enable --now prometheus-pve-exporter
systemctl status prometheus-pve-exporter --no-pager
```

Test with your Proxmox IP:

```text
curl -s "http://localhost:9221/pve?target=192.168.1.242" | head
```

Add to Prometheus:

  - job_name: "proxmox"
    metrics_path: /pve
    params:
      target:
        - "192.168.1.242"
    static_configs:
      - targets:
          - "localhost:9221"
        labels:
          instance: "proxmox-api"

Restart:

promtool check config /etc/prometheus/prometheus.yml
systemctl restart prometheus

---
## PART 12: TrueNAS Monitoring with graphite_exporter

TrueNAS should be monitored through:

TrueNAS Graphite exporter settings
        ↓
Monitoring LXC graphite_exporter :2003
        ↓
Prometheus scrapes :9108
        ↓
Grafana dashboard

12.1 Install graphite_exporter

```text
cd /tmp
```

GRAPHITE_EXPORTER_VERSION=$(curl -s https://api.github.com/repos/prometheus/graphite_exporter/releases/latest | grep tag_name | cut -d '"' -f 4 | sed 's/v//')

```text
wget https://github.com/prometheus/graphite_exporter/releases/download/v${GRAPHITE_EXPORTER_VERSION}/graphite_exporter-${GRAPHITE_EXPORTER_VERSION}.linux-amd64.tar.gz
```

```text
tar xvf graphite_exporter-${GRAPHITE_EXPORTER_VERSION}.linux-amd64.tar.gz
```

```text
cp graphite_exporter-${GRAPHITE_EXPORTER_VERSION}.linux-amd64/graphite_exporter /usr/local/bin/
```

Create user:

```text
useradd --system --no-create-home --shell /usr/sbin/nologin graphite_exporter
chown graphite_exporter:graphite_exporter /usr/local/bin/graphite_exporter
```

12.2 Create mapping file

```text
mkdir -p /etc/graphite_exporter
nano /etc/graphite_exporter/mapping.yml
```

Use this better catch-all mapping:

mappings:
  - match: "(.+)"
    match_type: regex
    name: "truenas_graphite_metric"
    labels:
      graphite_path: "$1"

Set ownership:

```text
chown -R graphite_exporter:graphite_exporter /etc/graphite_exporter
```

12.3 Create graphite_exporter service

```text
nano /etc/systemd/system/graphite_exporter.service
```

Use:

[Unit]
Description=Prometheus Graphite Exporter
Wants=network-online.target
After=network-online.target

```text
[Service]
User=graphite_exporter
Group=graphite_exporter
Type=simple
```

ExecStart=/usr/local/bin/graphite_exporter \
  --graphite.listen-address=:2003 \
  --web.listen-address=:9108 \
  --graphite.mapping-config=/etc/graphite_exporter/mapping.yml

```text
Restart=always
RestartSec=5
```

```text
[Install]
WantedBy=multi-user.target
```

Start:

```text
systemctl daemon-reload
systemctl enable --now graphite_exporter
systemctl status graphite_exporter --no-pager
```

Check ports:

```text
ss -tulpn | grep -E '2003|9108'
```

12.4 Configure TrueNAS

In TrueNAS:

System Settings > Reporting > Exporters > Add

Use:

Name: Graphite
Type: GRAPHITE
Enable: checked
Destination IP: Monitoring LXC IP
Destination Port: 2003
Prefix: truenas
Namespace: truenas
Update Every: 10
Buffer On Failures: 10
Send Names Instead Of IDs: checked
Matching Charts: blank

12.5 Test graphite_exporter manually

From Monitoring LXC:

```text
echo "truenas.test.metric 123 $(date +%s)" | nc -u -w1 localhost 2003
curl -s http://localhost:9108/metrics | grep truenas | head
```

You want to see:

truenas_graphite_metric{graphite_path="truenas.test.metric"} 123

12.6 Add TrueNAS to Prometheus

Add this job:

  - job_name: "truenas-graphite"
    static_configs:
      - targets:
          - "localhost:9108"
        labels:
          instance: "truenas"

Restart:

promtool check config /etc/prometheus/prometheus.yml
systemctl restart prometheus

---
## PART 13: Import Grafana dashboards

In Grafana:

Dashboards > New > Import

Linux Node Exporter dashboard:

1860

Proxmox dashboard:

## 10347

TrueNAS dashboards may need custom queries because Graphite metric names depend on what TrueNAS emits. First check:

```text
curl -s http://localhost:9108/metrics | grep truenas | head -n 50
```

Then build panels from:

{job="truenas-graphite"}

---
## PART 14: Add Prometheus and Grafana to Homepage

On your Homepage VM, edit:

```text
nano /path/to/homepage/config/services.yaml
```

Your path might be one of these:

```text
/home/youruser/homepage/config/services.yaml
/opt/homepage/config/services.yaml
/docker/homepage/config/services.yaml
```

Since Homepage is already running, use the same config folder where your current services.yaml is.

Add this under your Monitoring section:

- Monitoring:
    - Prometheus:
        href: http://MONITORING-LXC-IP:9090
        description: Metrics database and scraper
        icon: prometheus.png
        siteMonitor: http://MONITORING-LXC-IP:9090/-/ready
        widget:
          type: prometheusmetric
          url: http://MONITORING-LXC-IP:9090
          refreshInterval: 10000
          metrics:
            - label: Targets Up
              query: count(up == 1)
            - label: Targets Down
              query: count(up == 0)

    - Grafana:

```text
        href: http://MONITORING-LXC-IP:3000
        description: Metrics dashboards
        icon: grafana.png
        siteMonitor: http://MONITORING-LXC-IP:3000
```

Example with Monitoring LXC IP as 192.168.1.250:

- Monitoring:
    - Prometheus:
        href: http://192.168.1.250:9090
        description: Metrics database and scraper
        icon: prometheus.png
        siteMonitor: http://192.168.1.250:9090/-/ready
        widget:
          type: prometheusmetric
          url: http://192.168.1.250:9090
          refreshInterval: 10000
          metrics:
            - label: Targets Up
              query: count(up == 1)
            - label: Targets Down
              query: count(up == 0)

    - Grafana:
        href: http://192.168.1.250:3000
        description: Metrics dashboards
        icon: grafana.png
        siteMonitor: http://192.168.1.250:3000

Restart Homepage.

If Homepage is Docker:

```text
docker restart homepage
```

If it is Docker Compose:

```text
cd /path/to/homepage
docker compose restart
```

If it is systemd:

```text
systemctl restart homepage
```

Then open Homepage and you should see Prometheus and Grafana under Monitoring.

Better Homepage layout option:

Since your Homepage layout already has a Monitoring group, put Prometheus and Grafana inside that group next to Uptime Kuma.

Example:

- Monitoring:
    - Uptime Kuma:
        href: http://UPTIME-KUMA-IP:3001
        description: Service uptime monitoring
        icon: uptime-kuma.png

    - Prometheus:
        href: http://192.168.1.250:9090
        description: Metrics scraper
        icon: prometheus.png
        siteMonitor: http://192.168.1.250:9090/-/ready
        widget:
          type: prometheusmetric
          url: http://192.168.1.250:9090
          refreshInterval: 10000
          metrics:
            - label: Up
              query: count(up == 1)
            - label: Down
              query: count(up == 0)

    - Grafana:
        href: http://192.168.1.250:3000
        description: Metrics dashboards
        icon: grafana.png
        siteMonitor: http://192.168.1.250:3000

---
## FINAL NOTES

Important ports:

Prometheus: 9090
Grafana: 3000
Node Exporter: 9100
Proxmox PVE Exporter: 9221
Graphite receive port for TrueNAS: 2003
Graphite exporter Prometheus scrape port: 9108

Recommended access:

Keep all ports LAN-only.
Do not expose Prometheus or exporters publicly.
Use Grafana as the main visual dashboard.
Use Homepage only as a launcher/status page.
