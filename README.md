# Mini SOC — Security Monitoring Dashboard

A lightweight, entry-level **Security Operations Center (SOC)** simulator built in Python. It collects local operating system logs across Linux and Windows platforms, detects three classes of suspicious activity (brute-force logins, port scans, and blacklisted processes), and displays real-time security events on a live, dark-mode Flask + Bootstrap 5 dashboard — all backed by a thread-safe SQLite database.

This project is intentionally lightweight and zero-dependency beyond standard Python libraries. There is no Elasticsearch, OpenSearch, Logstash, Kibana, Sigma, Docker, or external SIEM required. It demonstrates the _core concepts_ behind cross-platform log collection, normalization, detection engineering, and SOC dashboard visualization in a clean, production-ready codebase.

---

## Features

| #   | Feature                           | Description                                                                                                                                                                                                                                              |
| --- | --------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | **Cross-Platform Log Collection** | Collects Linux `auth.log`, `syslog`, systemd `journalctl`, and Windows Event Logs (Security & System channels: Event IDs 4624, 4625, 4740). Automatically falls back to `sample_logs/` in demo mode. Normalizes all events into a canonical JSON schema. |
| 2   | **Failed Login Detection**        | Flags brute-force attempts: N failed logins from the same IP/user within a rolling time window. Deduplicates alerts to avoid spam.                                                                                                                       |
| 3   | **Port Scan Detection**           | Detects remote hosts connecting to N distinct local ports within a rolling window via `psutil`. Records scan details to dedicated database tables.                                                                                                       |
| 4   | **Suspicious Process Detection**  | Scans running processes against a configurable blacklist (`mimikatz`, `nc.exe`, `ncat`, `netcat`, `powershell.exe`, `cmd.exe`, `psexec`, etc.) using word-boundary regex matching to eliminate false positives.                                          |
| 5   | **Real-Time SOC Dashboard**       | Dark-themed Flask UI with live stat cards, severity badges, and auto-refresh (default: 5 seconds) via REST API endpoints.                                                                                                                                |
| 6   | **SQLite Storage & Schema**       | Thread-safe SQLite database with WAL mode and dedicated tables: `logs`, `alerts`, `processes`, `failed_logins`, `port_scans`.                                                                                                                            |
| 7   | **YAML & File Configuration**     | Fully configurable via `config.yaml` (with PyYAML or built-in fallback parser), `config.py`, and environment variable overrides.                                                                                                                         |
| 8   | **Structured Logging**            | Centralized application logging to `logs/application.log` with `RotatingFileHandler` support.                                                                                                                                                            |

---

## Project Architecture

```
mini_soc/
├── app.py                     # Main entry point (Flask app factory + daemon background services)
├── config.yaml                # Central YAML configuration file
├── config.py                  # Static configuration container (YAML loader + fallback parser)
├── requirements.txt
├── README.md
│
├── collector/                 # Log collection layer
│   ├── base_collector.py      # Abstract base class + file-tailing helper
│   ├── linux_collector.py     # Parses auth.log / syslog / systemd journalctl with fallback
│   ├── windows_collector.py   # Parses Windows Event Logs (Security 4624/4625/4740 & System)
│   └── log_collector.py       # Multi-platform collector factory & service loop
│
├── detectors/                 # Detection engine
│   ├── failed_login_detector.py # Brute-force detection logic
│   ├── port_scan_detector.py   # Distinct port scan detection logic
│   ├── process_detector.py     # Blacklisted process detector with regex word matching
│   └── detection_service.py   # Detection engine coordinator
│
├── dashboard/                 # Flask dashboard
│   ├── routes.py               # Blueprint: server-rendered UI + JSON API endpoints
│   ├── templates/
│   │   ├── base.html           # Dark theme layout with Inter font and live indicator
│   │   └── dashboard.html      # Responsive dashboard with stat cards & tables
│   └── static/
│       ├── css/style.css       # Premium cybersecurity dark theme CSS
│       └── js/dashboard.js     # Client-side auto-refresh AJAX engine
│
├── database/                  # Data access layer
│   ├── models.py               # SQL DDL schemas for logs, alerts, processes, failed_logins, port_scans
│   └── db_manager.py           # Thread-safe SQLite manager with WAL mode
│
├── utils/                     # Shared utilities
│   ├── logger.py               # Centralized application logging to logs/application.log
│   └── json_normalizer.py      # Canonical log event builder and timestamp normalizer
│
├── scripts/
│   └── generate_demo_data.py  # Demo script: writes synthetic log lines + simulates a port scan
│
├── sample_logs/               # Fallback log files for demo & non-root environments
├── logs/                      # Application log files (application.log)
├── data/                      # SQLite database file (mini_soc.db)
├── docs/                      # Complete documentation set
└── tests/                     # Comprehensive pytest test suite
```

---

## Quick Start

### 1. Installation

```bash
# Navigate to the project directory
cd mini_soc

# Create and activate a virtual environment
python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

# Install required dependencies
pip install -r requirements.txt
```

### 2. Run Mini SOC

```bash
# Recommended for a demo / no real system logs available:
MINI_SOC_DEMO=1 python app.py

# Or, to try reading real system logs first (falls back to sample_logs/
# automatically if they're missing or unreadable):
python app.py
```

Open your browser to **http://127.0.0.1:5000**.

- **Linux**: Automatically checks `/var/log/auth.log`, `/var/log/secure`, `journalctl`, or falls back seamlessly to `sample_logs/` (forced by `MINI_SOC_DEMO=1`).
- **Windows**: Automatically reads Windows Security Event Logs (Event IDs 4624, 4625, 4740) and System logs via `pywin32`.

### 3. Generate Demo Security Events

In a second terminal (with the virtual environment activated), run:

```bash
python scripts/generate_demo_data.py
```

This writes synthetic login attempts (normal + a brute-force burst) to `sample_logs/auth.log` and `sample_logs/syslog`, then opens 25 loopback TCP connections and holds them open for ~20 seconds to reliably trigger port-scan detection. It does **not** write to the database directly — every event flows through the real collector/detector pipeline, exactly like a real attack would. Within one collector cycle (~10s) and one detection cycle (~15s), the dashboard auto-updates to show new logs plus **failed_login** and **port_scan** alerts.

---

## Configuration (`config.yaml`)

You can customize thresholds, polling intervals, and blacklists in `config.yaml`:

```yaml
collector:
  polling_interval_seconds: 10
  batch_size: 500

detection:
  failed_login_threshold: 5
  failed_login_window_seconds: 120
  port_scan_port_threshold: 20
  port_scan_window_seconds: 60
  process_scan_interval_seconds: 15

blacklist:
  - mimikatz
  - nc.exe
  - ncat
  - netcat
  - hydra
  - powershell.exe
  - cmd.exe
  - psexec

demo_mode: false

dashboard:
  host: "0.0.0.0"
  port: 5000
  refresh_interval_seconds: 5
```

Environment variables can also override defaults:

- `MINI_SOC_AUTH_LOG`
- `MINI_SOC_SYSLOG`
- `MINI_SOC_HOST`
- `MINI_SOC_PORT`
- `MINI_SOC_DEBUG`
- `MINI_SOC_DEMO`

---

## Testing

### Automated tests

```bash
pytest tests/ -v
```

All 20 unit tests cover the database manager, log normalizer, Linux log collector (including timezone-correct timestamp parsing), and all three detection engines — without requiring root permissions or touching your real system logs/processes/network.

### Manually exercising each detector

With `MINI_SOC_DEMO=1 python app.py` running in one terminal:

| Detector           | Command (second terminal)                                                  | Expect                                                        |
| ------------------ | -------------------------------------------------------------------------- | ------------------------------------------------------------- |
| Failed login       | `python scripts/generate_demo_data.py`                                     | `failed_login` alert within ~25s (8 attempts vs. threshold 5) |
| Port scan          | Included automatically in the script above (opens 25 loopback connections) | `port_scan` alert within ~25s (25 ports vs. threshold 20)     |
| Suspicious process | `ncat -l 4444`                                                             | Appears in **Suspicious Processes** within ~15s               |

> `nmap -p 1-100 localhost` is a common way to _think about_ triggering a port scan, but in practice a fast scan against closed ports often finishes and closes its connections before the next detection poll — `PortScanDetector` only sees a periodic snapshot of `psutil.net_connections()`, not live packets. `generate_demo_data.py` works around this by holding connections open long enough to guarantee detection; see `docs/INTERVIEW_QA.md` Q45 for details.

---

## Documentation

Comprehensive documentation is available in the `docs/` directory:

- **`docs/SETUP_GUIDE.md`** — Detailed installation and system setup instructions.
- **`docs/ARCHITECTURE.md`** — System architecture, database model, and data flow.
- **`docs/MODULE_DOCUMENTATION.md`** — Detailed module-by-module documentation.
- **`docs/DEMO_SCRIPT.md`** — Step-by-step walkthrough script for portfolio demonstrations.
- **`docs/INTERVIEW_QA.md`** — Cybersecurity & technical interview preparation questions.
- **`docs/RESUME_BULLETS.md`** — Recommended resume bullet points highlighting key accomplishments.

---

## License

Provided as an open security portfolio and educational project. Feel free to adapt for learning or portfolio demonstrations.
