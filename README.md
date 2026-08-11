# 📊 docker-grafana-prometheus-monitoring

A containerized monitoring stack on Windows — **Grafana** + **Prometheus** running via Docker, connected over a custom network, with a live PromQL-powered dashboard.

---

## 📖 Overview

This project sets up a full monitoring stack using Docker on Windows — no native installs, no scattered config files. Grafana and Prometheus each run in their own container, joined to a shared Docker network so they can talk to each other by container name. The result is a live Grafana dashboard, built from scratch, showing the real-time status of a Prometheus target.

---

## 🧱 Architecture

```
Windows
   │
   ▼
Docker Desktop
   │
   └── monitoring network
          │
          ├── Grafana     :3000
          │
          └── Prometheus  :9090
```

---

## 🛠️ Tech Stack

| Tool | Purpose |
|------|---------|
| **Docker Desktop** | Container runtime (Windows) |
| **Grafana** | Visualization & dashboards |
| **Prometheus** | Metrics collection & storage |
| **PromQL** | Query language for metrics |
| **Docker Volumes** | Persistent storage for Grafana |
| **Docker Networks** | Container-to-container communication |

---

## 🚀 Setup Walkthrough

### 1. Install & Verify Docker Desktop

Make sure Docker Desktop is installed and running, then verify from PowerShell:

```powershell
docker --version
docker info
```

If both commands return output without errors, you're good to continue.

---

### 2. Download the Grafana Docker Image

```powershell
docker pull grafana/grafana
```

Verify it downloaded:

```powershell
docker images
```

You should see `grafana/grafana` in the list.

> **What happened?** Docker pulled the official Grafana image from Docker Hub and stored it locally:
> `Docker Hub → grafana/grafana image → stored locally in Docker`

---

### 3. Create a Grafana Docker Volume

```powershell
docker volume create grafana-storage
```

Verify:

```powershell
docker volume ls
```

You should see `grafana-storage` in the list.

> **Why do we need a volume?** Without persistent storage, important Grafana data — dashboards, data-source configuration, settings, and other application data — could be lost when the container is removed.
>
> Conceptually: `Grafana Container → /var/lib/grafana → Docker Volume (grafana-storage)`

---

### 4. Run Grafana

```powershell
docker run -d --name grafana -p 3000:3000 -v grafana-storage:/var/lib/grafana grafana/grafana
```

| Flag | What it does |
|------|---------------|
| `-d` | Runs Grafana in the background |
| `--name grafana` | Names the container `grafana` |
| `-p 3000:3000` | Maps Windows port 3000 to Grafana's container port 3000 |
| `-v grafana-storage:/var/lib/grafana` | Mounts the Docker volume into Grafana's data directory |
| `grafana/grafana` | The image to use |

---

### 5. Check Grafana

```powershell
docker ps
```

You should see `grafana` listed.

Open your browser: **http://localhost:3000** — Grafana is now running.

---

### 6. Download the Prometheus Docker Image

```powershell
docker pull prom/prometheus
```

Verify:

```powershell
docker images
```

You should see `prom/prometheus` in the list. Prometheus, like Grafana, runs through Docker rather than being installed directly on Windows.

---

### 7. Create a Prometheus Project Folder

```powershell
mkdir prometheus-monitoring
cd prometheus-monitoring
```

Check your current location:

```powershell
pwd
```

---

### 8. Create `prometheus.yml`

Create the Prometheus configuration file directly from PowerShell:

```powershell
@"
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
"@ | Set-Content prometheus.yml
```

Check the file:

```powershell
Get-Content prometheus.yml
```

You should see:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
```

---

### 9. Understand `prometheus.yml`

This configuration tells Prometheus what to monitor.

| Setting | Meaning |
|---------|---------|
| `scrape_interval: 15s` | Prometheus collects metrics every 15 seconds |
| `job_name: "prometheus"` | Creates a monitoring job called `prometheus` |
| `targets: ["localhost:9090"]` | Tells Prometheus to monitor the Prometheus service itself |

For this beginner project, self-monitoring is intentional — it gives us a target to query without needing extra services.

---

### 10. Create a Docker Network

```powershell
docker network create monitoring
```

Verify:

```powershell
docker network ls
```

You should see `monitoring` in the list.

---

### 11. Connect Grafana to the Network

```powershell
docker network connect monitoring grafana
```

Now both containers can communicate:

```
        monitoring network
               │
       ┌───────┴───────┐
       │               │
       ▼               ▼
    Grafana        Prometheus
     :3000            :9090
```

---

### 12. Start Prometheus

From inside your `prometheus-monitoring` directory:

```powershell
docker run -d --name prometheus --network monitoring -p 9090:9090 -v ${PWD}/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
```

| Flag | What it does |
|------|---------------|
| `--name prometheus` | Names the container |
| `--network monitoring` | Connects it to the same Docker network as Grafana |
| `-p 9090:9090` | Makes Prometheus available at `localhost:9090` |
| `-v ${PWD}/prometheus.yml:/etc/prometheus/prometheus.yml` | Mounts your configuration file into the Prometheus container |

---

### 13. Check Both Containers

```powershell
docker ps
```

You should now have both `grafana` and `prometheus` running.

Your setup at this point:

```
Windows
   │
   ▼
Docker Desktop
   │
   └── monitoring network
          │
          ├── Grafana :3000
          │
          └── Prometheus :9090
```

---

### 14. Test Prometheus

Open **http://localhost:9090**, then go to:

```
Status → Targets
```

You should see your Prometheus target listed as **UP**. This confirms Prometheus is working.

---

### 15. Connect Prometheus to Grafana

Open **http://localhost:3000**. In Grafana:

```
Connections → Data sources → Add data source → Prometheus
```

For the URL, enter:

```
http://prometheus:9090
```

Click **Save & test** — you should get a successful connection.

> **Why `prometheus:9090` and not `localhost:9090`?**
> Grafana and Prometheus are both Docker containers on the same network, so Docker resolves the container name `prometheus` directly:
> `Grafana → prometheus:9090 → Prometheus container`

---

### 16. Create the Grafana Dashboard

```
Dashboards → Create dashboard → Add visualization
```

You'll enter the panel editor — select the **Prometheus** data source (you already confirmed this connection works in step 15).

---

### 17. Use Your First PromQL Query

Under the query editor, select **Code**, then enter:

```promql
up
```

Click **Run queries**. You should get:

```
1
```

| Value | Meaning |
|-------|---------|
| `1` | Target is UP |
| `0` | Target is DOWN |

So `up = 1` means your monitored Prometheus target is currently available.

---

### 18. Change the Visualization

On the right side of the panel editor, under **All visualizations**, select:

```
Stat
```

This gives a simple status panel showing the Prometheus status as a single number.

---

### 19. Name the Panel

Under panel options, set the title to:

```
Prometheus Status
```

---

### 20. Save the Dashboard

Click **Save**, and name the dashboard:

```
Docker Monitoring
```

---

## ✅ Final Result

A live Grafana dashboard showing real-time status of the Prometheus target — fully containerized, network-isolated, and backed by persistent storage.

```
CONTAINER    IMAGE              STATUS
grafana      grafana/grafana    Up
prometheus   prom/prometheus    Up
```

**Dashboard:** `Docker Monitoring` → **Panel:** `Prometheus Status` → **Query:** `up` → **Visualization:** `Stat` → **Result:** `1` (target UP)

---

## 📌 Key Concepts Learned

- Running observability tools as **Docker containers** instead of native installs
- Using **Docker volumes** for persistent application data
- Using **Docker networks** for container-to-container name resolution
- Writing a basic **Prometheus scrape config**
- Querying metrics with **PromQL**
- Building a simple **Grafana Stat panel**

---

## 🔭 Possible Next Steps

- [ ] Add more scrape targets (e.g., Node Exporter for system metrics)
- [ ] Build out a multi-panel dashboard (CPU, memory, uptime)
- [ ] Add alerting rules in Prometheus
- [ ] Persist Prometheus data with its own dedicated volume
- [ ] Automate the whole stack with `docker-compose`

---

## 🙋 About

Part of an ongoing series of hands-on cloud & DevOps learning projects, documented step-by-step as part of a portfolio-building journey.
