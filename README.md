# VulnScan Pro

A lightweight **web application vulnerability scanner** built with Python and Flask. Point it at a URL and it runs a battery of reconnaissance and active security checks, assigns a **security score (0–100) and letter grade (A–F)**, maps every finding to the **OWASP Top 10 / CWE / CVSS**, and generates a downloadable **PDF report** — all through a simple web interface with live progress.

> **Educational project.** VulnScan Pro was built as a project-based-learning exercise to demonstrate how vulnerability scanners work end-to-end. It uses heuristic/signature-based detection and is **not** a replacement for mature tools like OWASP ZAP, Burp Suite, or Nessus.

---

## ⚖️ Legal & ethical use

**Only scan systems you own or have explicit written permission to test.** Unauthorized scanning of systems you do not control is illegal in most jurisdictions. This project ships with its own deliberately-vulnerable test application (see [Testing](#-testing-with-the-bundled-vulnerable-app)) so you can try everything safely on `localhost`.

---

## ✨ Features

- **13 scan modules** covering recon and active testing
- **Live progress** streamed to the browser via Server-Sent Events (SSE)
- **Security score & letter grade** for an at-a-glance verdict
- **OWASP Top 10 + CWE + CVSS** mapping for every finding
- **PDF report** generation
- **Optional notifications** via Slack webhook or email
- A bundled **deliberately-vulnerable web app** for safe testing

### Scan modules

| Module | What it checks | Technique |
|---|---|---|
| Port Scan | Open ports & risky services | Threaded TCP connect scan |
| Security Headers | CSP, HSTS, X-Frame-Options, cookie flags… | HTTP response inspection |
| SSL/TLS | Cert validity, expiry, self-signed, weak ciphers, TLS version | TLS handshake inspection |
| DNS Recon | Records, SPF/DMARC, zone transfer | DNS queries |
| Crawler | Forms, scripts, emails, comments, sensitive paths | BFS same-domain crawl |
| XSS | Reflected cross-site scripting | Payload injection + unescaped-reflection check |
| SQL Injection | Error-based, time-based blind, boolean-based blind | Payload injection + response analysis |
| Tech Fingerprint | Server / CMS / JS libraries + known CVEs | Header + HTML signatures |
| Favicon Analyzer | Default-software fingerprinting | MurmurHash (Shodan-style) |
| WAF Detection | Web application firewall presence | Signature detection |
| Open Redirect | Unvalidated redirects | Redirect-parameter testing |
| CSRF Detection | State-changing forms missing anti-CSRF tokens | Form token analysis |
| JavaScript Analysis | Hard-coded secrets / API keys in JS | Regex secret scanning |

---

## 🏗️ Architecture

```
Browser (HTML/CSS/JS)
      │  POST /api/scan            ┌───────────────────────────┐
      ├──────────────────────────▶│  Flask API  (backend/app.py)│
      │  GET  /api/scan/<id>/stream│                            │
      │◀────── SSE progress ───────│  spawns a background thread │
      │  GET  /api/scan/<id>/results        │                   │
      │                            └─────────┼───────────────────┘
      │                                      ▼
      │                         ┌────────────────────────────┐
      │                         │  Scan orchestration          │
      │                         │  WAF → recon → crawler →      │
      │                         │  XSS/SQLi/JS → redirect/CSRF  │
      │                         │  → score → OWASP map → PDF    │
      │                         └──────────────┬───────────────┘
      │                                        ▼
      │                         SQLite (ScanTask, ScanEvent)  +  reports/<id>.pdf
```

**How a scan flows:**

1. `POST /api/scan` validates the URL, creates a `ScanTask` row (SQLite), and launches `bg_run_scan()` on a **background thread**, returning `202 Accepted` with a `scan_id` immediately so the UI never blocks.
2. The browser opens an **SSE stream** (`/api/scan/<id>/stream`) that polls the `ScanEvent` table and pushes live progress messages.
3. The orchestrator runs modules in dependency order — the **crawler runs before** XSS/SQLi/JS analysis because those consume the forms and scripts it discovers.
4. All findings are normalized into a common shape, **scored** (start at 100, deduct per severity), and **enriched** with OWASP/CWE/CVSS data from a knowledge-base lookup.
5. A **PDF report** is written to `reports/<scan_id>.pdf`, results are persisted, and optional Slack/email notifications fire.

### Tech stack

- **Backend:** Python 3.10+, Flask, Flask-SQLAlchemy (SQLite)
- **Scanning:** requests, BeautifulSoup, dnspython, Python `ssl`/`socket`, mmh3
- **Reporting:** ReportLab (PDF)
- **Frontend:** vanilla HTML / CSS / JavaScript (Server-Sent Events)

---

## 🚀 Getting started

**Prerequisites:** Python 3.10+ and pip.

```bash
# 1. Clone
git clone https://github.com/akankshasingar/web-vuln-scanner.git
cd web-vuln-scanner

# 2. Create and activate a virtual environment
python -m venv venv
# Windows (PowerShell):
.\venv\Scripts\Activate.ps1
# macOS / Linux:
source venv/bin/activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Run the app
cd backend
python app.py
```

Then open **http://127.0.0.1:5000** in your browser.

> **Tip:** the port scanner may need to run as Administrator/root for some socket operations. See [STARTUP.md](STARTUP.md) for troubleshooting.

---

## 🧪 Testing with the bundled vulnerable app

The repo includes a deliberately-insecure Flask app under [`vulnerable-webapp/`](vulnerable-webapp/) so you can run full scans safely on your own machine.

```bash
# In a second terminal, with the venv active:
cd vulnerable-webapp
python app.py            # serves HTTP on :5001 and HTTPS on :5002
```

Then scan `http://127.0.0.1:5001` from the VulnScan Pro UI. It contains intentional SQL injection, reflected/stored XSS, a token-less CSRF form, an open redirect, an unrestricted file upload, and stripped security headers — so you can see every module light up and produce a Grade **F** report.

> If `vulnapp.db` is ever missing or corrupted, rebuild it with `python db.py` inside `vulnerable-webapp/`.

---

## 📡 API reference

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/scan` | Start a scan. Body: `{ "url": "...", "modules": [...], "webhook_url": "...", "email": "..." }`. Returns `scan_id`. |
| `POST` | `/api/estimate` | Estimate scan duration for selected modules. |
| `GET` | `/api/scan/<id>/stream` | Live progress stream (Server-Sent Events). |
| `GET` | `/api/scan/<id>/results` | Full scan results as JSON. |
| `GET` | `/api/report/<id>` | Download the generated PDF report. |

Module names accepted in `modules`: `Port Scan`, `Security Headers`, `SSL/TLS`, `DNS Recon`, `Crawler`, `XSS`, `SQL Injection`, `Tech Fingerprinting`, `WAF Detection`, `Open Redirect`, `CSRF Detection`, `JavaScript Analysis`.

---

## 📊 Scoring

Each scan starts at **100 points** and deducts by severity:

| Severity | Deduction |
|---|---|
| Critical | 25 |
| High | 15 |
| Medium | 8 |
| Low | 3 |
| Info | 1 |

The final score (clamped 0–100) maps to a grade: **A** ≥ 90, **B** ≥ 80, **C** ≥ 70, **D** ≥ 60, **F** below. A detected WAF grants a small positive adjustment. This 0–100 score is a friendly summary metric; per-finding risk is expressed with standard **CVSS** scores in the OWASP mapping.

---

## ⚠️ Known limitations & future work

Being honest about scope is part of good engineering:

- **Heuristic detection** (keyword matching, a small hard-coded CVE dictionary) → possible false positives/negatives.
- **Background threads + SQLite** don't scale and don't survive a restart. A production design would use a task queue (e.g. Celery + Redis) and a server-grade database.
- **No JavaScript rendering** — the crawler uses `requests`, so single-page apps aren't fully crawled. (Playwright is a listed dependency reserved for this future work.)
- **Open-redirect coverage** currently tests the start URL and discovered forms, not every link parameter.
- Flask runs in debug mode for development — this should be disabled before any non-local deployment.

---

## 📁 Project structure

```
backend/
  app.py                 # Flask API, orchestration, SSE, DB models
  scanner/               # the 13 scan modules + score engine + OWASP mapper
  knowledge_base/        # OWASP/CWE/CVSS lookup table
  report/pdf_generator.py
  tests/                 # backtest harness + test target
  notifier.py            # Slack / email notifications
frontend/                # HTML/CSS/JS UI
vulnerable-webapp/       # deliberately-vulnerable test target
docs/                    # project report, presentation, guides
```

---

## 📄 License

Released under the [MIT License](LICENSE).
