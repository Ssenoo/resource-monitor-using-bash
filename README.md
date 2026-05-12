# 🖥️ Bash System Monitor

A full-stack system monitoring dashboard that uses **Bash scripts** to collect real-time hardware metrics, a **Node.js/Express** backend to serve and parse the data, and a **React** frontend to visualize everything in live charts and historical reports.

---

## ✨ Features

- **Real-Time Monitoring** — Live CPU, GPU, Memory, Disk, and Network stats streamed via WebSocket
- **Historical Reports** — Browse timestamped report folders and replay collected metrics as interactive charts
- **SMART Disk Health** — Reads and displays drive health status from `smartmontools`
- **System Uptime & Load** — Tracks uptime duration, logged-in users, and 1/5/15-minute load averages
- **Network Traffic** — Per-interface incoming/outgoing bytes with rate calculation (Bps)

---

## 🏗️ Architecture

```
┌──────────────────────────────────────────────────────────┐
│                        Frontend (React + Vite)           │
│  ┌─────────────┐   ┌──────────────┐                      │
│  │  Monitor    │   │   Reports    │                      │
│  │  (WebSocket)│   │  (REST API)  │                      │
│  └──────┬──────┘   └──────┬───────┘                      │
└─────────┼────────────────┼──────────────────────────────┘
          │ ws://           │ http://
┌─────────┼────────────────┼──────────────────────────────┐
│         │   Backend (Express + ws)                       │
│  ┌──────▼──────┐   ┌─────▼────────┐                      │
│  │  monitor.js │   │  reports.js  │                      │
│  │  (WS → bash)│   │  (REST API)  │                      │
│  └──────┬──────┘   └─────┬────────┘                      │
└─────────┼────────────────┼──────────────────────────────┘
          │                │
  ┌───────▼───────┐  ┌─────▼──────────────┐
  │   test.sh     │  │  system_reports/   │
  │ (Bash script) │  │  (log files)       │
  └───────────────┘  └────────────────────┘
```

### Data Flow

1. The **Bash script** (`test.sh`) collects system metrics (CPU, GPU, memory, disk, network, SMART) and writes them to timestamped `.log` files under `backend/system_reports/<timestamp>/`.
2. The **WebSocket server** (`monitor.js`) spawns the Bash script and streams its `stdout` output directly to connected browser clients.
3. The **REST API** (`/reports`) serves parsed log data from the `system_reports` directory to the Reports page.
4. The **Frontend** renders live data on the Monitor page and historical data with recharts on the Reports page.

---

## 🛠️ Tech Stack

| Layer    | Technology                                  |
|----------|---------------------------------------------|
| Shell    | Bash, `bc`, `lm-sensors`, `smartmontools`, `intel-gpu-tools`, `iproute2` |
| Backend  | Node.js, Express 5, `ws` (WebSocket), ES Modules |
| Frontend | React 19, Vite, React Router, Recharts, Axios |
| DevOps   | Docker (Ubuntu 22.04), Node.js 22           |

---

## 🚀 Getting Started

### Option 1 — Docker (Recommended)

> Requires [Docker](https://docs.docker.com/get-docker/) to be installed.

```bash
# Build the image
docker build -t bash-system-monitor .

# Run the container (expose both ports)
docker run -p 8000:8000 -p 5173:5173 bash-system-monitor
```

Then open your browser:
- **Frontend:** http://localhost:5173
- **Backend API:** http://localhost:8000

---

### Option 2 — Manual Setup (Linux / WSL)

> The Bash monitoring script requires a Linux environment. On Windows, use [WSL](https://learn.microsoft.com/en-us/windows/wsl/).

#### Prerequisites

```bash
sudo apt-get install -y bc lm-sensors smartmontools intel-gpu-tools iproute2
```

#### 1. Backend

```bash
cd backend
npm install
npm run main        # production
# or
npm run dev         # development (hot reload with nodemon)
```

The backend starts on **http://localhost:8000**.

#### 2. Frontend

```bash
cd frontend
npm install
npm run dev
```

The frontend starts on **http://localhost:5173**.

---

## 📁 Project Structure

```
bash_task_manager-main/
├── Dockerfile
├── backend/
│   ├── package.json
│   └── src/
│       ├── index.js              # Express + WebSocket server entry point
│       ├── controllers/
│       │   ├── foldersLogs.js    # Parses and returns log data for a report folder
│       │   └── reportfolders.js  # Lists available report folders
│       ├── middleware/
│       │   ├── logger.js         # Request logger
│       │   ├── error.js          # Global error handler
│       │   └── notfound.js       # 404 handler
│       ├── routes/
│       │   └── reports.js        # /reports route definitions
│       ├── utils/
│       │   └── parseLogs.js      # Log file parsers (CPU, GPU, Memory, Disk, Network, SMART, Uptime)
│       ├── ws/
│       │   └── monitor.js        # WebSocket handler (START/STOP monitoring)
│       └── test.sh               # Bash script: collects all system metrics
└── frontend/
    ├── index.html
    ├── vite.config.js
    ├── package.json
    └── src/
        ├── App.jsx
        ├── main.jsx
        ├── index.css
        ├── components/
        │   └── navbar/           # Navigation bar component
        ├── pages/
        │   ├── monitor/
        │   │   └── Monitor.jsx   # Live monitoring page (WebSocket consumer)
        │   └── reports/
        │       └── Reports.jsx   # Historical reports page (REST consumer)
        └── charts/               # Reusable chart components (Recharts wrappers)
```

---

## 🌐 API Reference

### REST API — Base URL: `http://localhost:8000`

| Method | Endpoint                       | Description                              |
|--------|--------------------------------|------------------------------------------|
| `GET`  | `/reports`                     | List all available report folder names   |
| `GET`  | `/reports/:folderName`         | Get all parsed log data for a folder     |

**Folder name format:** `YYYY-MM-DD-HHh-MMmin-SSsec`

**Example response for `/reports/:folderName`:**
```json
{
  "folder": "2025-05-10-14h-30min-00sec",
  "cpu":     [{ "time": "...", "usage": 42.5, "temperature": 65 }],
  "gpu":     [{ "time": "...", "usage": 18.0, "temperature": 58 }],
  "memory":  { "ram": [...], "virtual": [...] },
  "disk":    [{ "timestamp": "...", "disks": [...] }],
  "network": [{ "time": "...", "interface": "eth0", "incomingBps": 1024, "outgoingBps": 512 }],
  "smart":   { "status": "PASSED" },
  "uptimeData": [{ "time": "...", "uptime": "2 days, 4:30", "users": 1, "loadOne": 0.5, "loadFive": 0.6, "loadFifteen": 0.7 }]
}
```

### WebSocket — `ws://localhost:8000`

Send plain text messages to control the monitoring process:

| Message | Description                          |
|---------|--------------------------------------|
| `START` | Spawns `test.sh` and streams output  |
| `STOP`  | Sends `SIGTERM` to the Bash process  |

---

## 📊 Monitored Metrics

| Metric       | Tool Used           | Data Points                                      |
|--------------|---------------------|--------------------------------------------------|
| CPU          | `/proc/stat`, `sensors` | Usage %, Temperature (°C)                   |
| GPU (Intel)  | `intel_gpu_top`     | Usage %, Temperature (°C)                        |
| Memory       | `free`              | RAM & Virtual: Usage %, Used GB, Total GB        |
| Disk         | `lsblk`, `df`       | Disk name, size, partitions, mount, used space   |
| Network      | `/proc/net/dev`     | Per-interface total bytes + real-time Bps rate   |
| SMART        | `smartctl`          | Drive health self-assessment result              |
| Uptime/Load  | `uptime`            | Duration, user count, 1/5/15-min load averages   |

---

## 📄 License

ISC
