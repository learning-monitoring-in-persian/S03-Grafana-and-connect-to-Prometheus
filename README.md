[English](README.md) | [فارسی](README-persian.md)

# Set up Grafana

> [!NOTE]
> If you plan to install **Grafana** alongside **Prometheus** or other tools, I highly recommend using the **Docker** version. It makes configuration (like provisioning dashboards/datasources) and maintenance much easier in the future.
>
> However, if the machine doesn't have Docker, you can install the Grafana binary directly.

## Install Grafana on Linux (Debian/Ubuntu)

Grafana provides two main editions: **Enterprise** (which includes paid features but is free to use for basic things) and **OSS** (Open Source Software, completely free). We will install the **OSS** edition.

The recommended way to install Grafana on Debian/Ubuntu is using the official APT repository so you can easily update it later.

```bash
# 1. Install prerequisites
sudo apt-get install -y apt-transport-https software-properties-common wget gnupg

# 2. Add the Grafana GPG key
sudo mkdir -p /etc/apt/keyrings
sudo wget -O /etc/apt/keyrings/grafana.asc https://apt.grafana.com/gpg-full.key
sudo chmod 644 /etc/apt/keyrings/grafana.asc

# 3. Add the Grafana repository (Stable release)
echo "deb [signed-by=/etc/apt/keyrings/grafana.asc] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

# 4. Update APT and install Grafana OSS
sudo apt-get update
sudo apt-get install -y grafana
```


When you install Grafana via APT, it automatically creates the necessary users, folders, and the `systemd` service for you. 

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

> [!NOTE]
> On the first startup, Grafana might take a minute or two to initialize its database and download default plugins. During this time, the web interface won't be available. You can run `ss -ntlup | grep 3000` to check when port **3000** actually starts listening.

After that, Grafana will be available on port **3000**. You can access the UI at:

- `http://{IP_ADDRESS}:3000`

> [!IMPORTANT]
> Ensure port `3000/tcp` is open in your firewall (`ufw`, `iptables`, etc.) so you can access the web interface.

> [!NOTE]
> Default login credentials are:
> - **Username**: `admin`
> - **Password**: `admin`
>
> You will be prompted to change the password upon first login.

## Set up Grafana with Docker Compose

Example `docker-compose.yml`:

```yaml
services:
  grafana:
    image: grafana/grafana-oss:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=secure_password
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - ./grafana/dashboards:/var/lib/grafana/dashboards:ro

volumes:
  grafana_data:
```

> [!NOTE]
> We use environment variables (`GF_SECURITY_ADMIN_USER` and `GF_SECURITY_ADMIN_PASSWORD`) to set the admin credentials. If you don't set these, Grafana uses `admin/admin` by default.

### Suggested folder layout

```text
.
├─ docker-compose.yml
└─ grafana/
   ├─ provisioning/
   │  ├─ datasources/
   │  │  └─ prometheus.yml
   │  └─ dashboards/
   │     └─ dashboards.yml
   └─ dashboards/
      └─ node_exporter.json     #1860_rev45.json
```

Then start:

```bash
docker compose up -d
```

## Connect Grafana to Prometheus

You need to tell Grafana where Prometheus is located (so it can fetch metrics). There are two ways to do this:

### Method 1: Via the Grafana UI (Manual)
1. Go to **Connections -> Data sources** (or **Add new connection**).
2. Click **Add data source** and select **Prometheus**.
3. Set the **Prometheus server URL** (e.g., `http://{PROMETHEUS_IP}:9090`).
4. Click **Save & test**.

### Method 2: Via Provisioning File (Recommended for Automation)
You can configure Grafana to automatically connect to Prometheus using a YAML file.

Create `grafana/provisioning/datasources/prometheus.yml` (For Docker: bind mounted to `/etc/grafana/provisioning/datasources/`):

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://{PROMETHEUS_IP}:9090 # Replace with your Prometheus IP/Hostname
```

> [!NOTE]
> Read more about [Data source provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/#data-sources) in the official docs.

## Add a Dashboard (e.g., Node Exporter)

Grafana allows you to build custom dashboards or import pre-made ones from the community. For Node Exporter, some popular dashboard IDs are:
- [Node Exporter Full (1860)](https://grafana.com/grafana/dashboards/1860-node-exporter-full/)
- [Node Exporter Dashboard (24784)](https://grafana.com/grafana/dashboards/24784-node-exporter-dashboard-20240520/)

You can add dashboards in two ways:

### Method 1: Via the Grafana UI (Manual)
1. Go to **Dashboards -> New -> Import**.
2. **By ID**: Enter the dashboard ID (e.g., `1860`) and click **Load**.
3. **By JSON**: Or, paste the raw JSON code of the dashboard into the text box.
4. Select your Prometheus data source and click **Import**.

### Method 2: Via Provisioning File (Recommended)
You can place the JSON file of the dashboard on your server and tell Grafana to load it automatically.

1. **Download the JSON**: Go to the dashboard link (e.g. ID `1860`), download its JSON file, and save it as `grafana/dashboards/node_exporter.json` (mapped to `/var/lib/grafana/dashboards` in Docker, or `/var/lib/grafana/dashboards` in a Binary setup).
2. **Create the provisioning config**: Create `grafana/provisioning/dashboards/dashboards.yml` (mapped to `/etc/grafana/provisioning/dashboards/` in Docker, or `/etc/grafana/conf/provisioning/dashboards/` in Binary) with the following content:

```yaml
apiVersion: 1

providers:
  - name: 'Default'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    options:
      path: /var/lib/grafana/dashboards # Path inside the Grafana container/server where JSON files live
```

> [!IMPORTANT]
> When provisioning dashboards via JSON files, the data source UID inside the JSON might not match your Prometheus data source. If your panels show "Datasource not found" or "No data", you may need to open the JSON file and replace the data source references (e.g., `${DS_PROMETHEUS}`) with your actual data source name or UID. Alternatively, you can set a specific `uid` in your `prometheus.yml` data source config to match what the JSON expects.

> [!NOTE]
> Read more about [Dashboard provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/#dashboards).

## Explore Feature

Grafana has an **Explore** tab in the sidebar. 
This feature allows you to run ad-hoc queries (like [PromQL](https://prometheus.io/docs/prometheus/latest/querying/basics/)) and inspect metrics, logs, or traces **without creating a dashboard**. It is heavily used for troubleshooting and exploring what metrics are available in your Prometheus server.

## Grafana Alerts

Grafana has its own built-in **Alerting** engine (similar to Prometheus Alertmanager, but with a visual interface). 

The flow works like this:
1. **Alert Rule**: You write a query (e.g., "Is CPU > 80%?").
2. **Contact Point**: Where the alert should be sent (e.g., Slack, Email, Telegram, Discord, Webhook).
3. **Notification Policy**: Routes specific alerts to specific Contact Points (e.g., "Send Critical alerts to Slack, Warnings to Email").

You can configure all of this visually from the **Alerting** menu in Grafana.

> [!TIP]
> **Alerting as Code (Provisioning):** You can also manage your alerts via YAML files instead of the UI! By placing your YAML config files in the `grafana/provisioning/alerting/` folder (mapped to `/etc/grafana/provisioning/alerting/`), Grafana will automatically load your Alert Rules, Contact Points, and Notification Policies. Read more about [Alerting Provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/#alerting).

> [!NOTE]
> For more details on supported endpoints and configurations, check out the [Grafana Alerting documentation](https://grafana.com/docs/grafana/latest/alerting/).

## Users, Roles, and Access Control

If you are new to Grafana, understanding access control is simple. It works like this:

- **Users**: These are the accounts people use to log in (e.g., John, Mary).
- **Roles**: This defines what a user can *do*. The basic roles are:
  - **Admin**: Can do everything (add data sources, manage users, edit anything).
  - **Editor**: Can create and modify dashboards, but cannot change system settings.
  - **Viewer**: Can only view dashboards. They cannot edit or break anything.
- **Teams (Groups)**: Instead of giving permissions to users one by one, you can group them into a "Team" (e.g., "DevOps Team", "Developer Team") and give that team access to specific dashboard folders.

You can manage all of this from the **Administration -> Users and access** menu. 

> [!NOTE]
> For more advanced setups, Grafana offers Role-Based Access Control (RBAC). Read more in the [Grafana RBAC docs](https://grafana.com/docs/grafana/latest/administration/roles-and-permissions/).

---

> [!TIP]
> **Explore and Learn:** Grafana is a massive and powerful tool. To learn it deeply, the best approach is to navigate through the UI, explore the different menus, and experiment with building panels and dashboards yourself. Have fun exploring! :)
