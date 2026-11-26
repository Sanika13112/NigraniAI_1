<p align="center">
  <img src="website/assets/nigrani-eye-300.png" alt="NigraniAI" width="180">
</p>

<h1 align="center">NigraniAI</h1>

<p align="center"><b>Real-time DPDP compliance, proven from what your systems actually do.</b></p>

<p align="center">
  <a href="https://nigrani-ai.vercel.app">Website</a> ·
  <a href="dashboard/README.md">Dashboard documentation</a> ·
  <a href="website/README.md">Website documentation</a>
</p>

---

NigraniAI is a compliance platform for India's  **Digital Personal Data Protection Act, 2023** and the
**DPDP Rules, 2025**. Instead of questionnaires, it reads a company's live logs and database
(read-only) and shows, in real time, where the company stands, why, and what to fix: where personal
data lives, whether every use has consent and a purpose, what is due for deletion, whether a breach is
happening, and how long is left on each legal deadline.

This repository holds everything a teammate needs to run, understand and improve it: the
**compliance dashboard** and the **public website**.

> **New to the project?** Read [What NigraniAI does](#1-what-nigraniai-does), then follow
> [Getting started](#5-getting-started) step by step. Once you have the configuration values, the
> dashboard runs on your laptop in about 15 minutes.

---

## Contents

1. [What NigraniAI does](#1-what-nigraniai-does)
2. [How it works](#2-how-it-works)
3. [Technology](#3-technology)
4. [Repository layout](#4-repository-layout)
5. [Getting started](#5-getting-started)
6. [Configuration reference](#6-configuration-reference)
7. [Running tests and checking your work](#7-running-tests-and-checking-your-work)
8. [Working together: forks, branches and pull requests](#8-working-together-forks-branches-and-pull-requests)
9. [Coding conventions](#9-coding-conventions)
10. [Rules we never break](#10-rules-we-never-break)
11. [Deployment](#11-deployment)
12. [Troubleshooting](#12-troubleshooting)
13. [DPDP glossary](#13-dpdp-glossary)
14. [Roadmap](#14-roadmap)
15. [Licence and contact](#15-licence-and-contact)

---

## 1. What NigraniAI does

Each client company gets **its own copy** of the dashboard, connected to its own systems. Today's
reference client is **ShopSmart**, an e-commerce website.

| Section | What it does | DPDP basis |
|---|---|---|
| **Dashboard** | Headline numbers from every section, live log charts, "Needs attention", a status per DPDP area | All |
| **Log Telemetry** | Live structured log feed from every service; personal data in each line shown masked | s.8(5) safeguards |
| **Consent & Purpose** | Checks every processing event for valid consent and a declared purpose; customer timelines; notice versions | s.5, s.6, Rule 3 |
| **Data map** | Which tables hold personal data, which services use it, for which purposes, who receives it; inventory and gaps | s.8(1) accountability |
| **PII detection** | Finds personal data in every column and log line from column names, check-digit validation (Aadhaar, PAN, cards…), links to customers, a local language model and admin feedback | s.8(1), s.8(5) |
| **Retention** | Retention schedule measured on live data: past deadline, due soon, legal holds, erasure checklists, inactive accounts | s.8(7), s.12(3), Rule 8 |
| **Breach Response** | 7 detectors + the anomaly model, signal triage, breach register, **Rule 7 and CERT-In clocks**, notice drafts | s.8(6), Rule 7 |
| **Anomaly detection** | Machine-learning model (Isolation Forest) per service and time slice, with a full explanation for every anomaly | s.8(5) |
| **AI Assistant** | Answers questions from the dashboard's own numbers and drafts fixes | – |
| **Readiness score** | 0–100 DPDP readiness score from eight areas; every finding cites the law and gives a fix | All |
| **Evidence & audit** | Tamper-evident records (hash chain, daily seals) and evidence packs for auditors | s.8(1) accountability |
| **Access (IAM)** | AWS-style permissions for the client's staff: 37 actions, JSON policies, groups, simulator | s.8(5) |
| **Usage, plan & billing** | Usage meters, plan limits, upgrades, GST invoices, Razorpay payments | – |

**The DPDP baseline.** The parts of Consent, Retention, Breach and Anomaly that come from the law are
**locked**: they always run, their legal reference is fixed, and a client can make them stricter but
never weaker (for example, the Board report deadline can be 48 hours but never more than 72). The
server enforces this on every save. Details: [dashboard README §5.11](dashboard/README.md#511-dpdp-baseline-fixed-legal-rules).

The **website** ([nigrani-ai.vercel.app](https://nigrani-ai.vercel.app)) presents the product, explains
the DPDP Act and takes demo bookings.

---

## 2. How it works

```
                         ┌─────────────── Browser ───────────────┐
                         │  dashboard/frontend  (HTML, CSS, JS)  │
                         └──────┬─────────────────────┬──────────┘
            sign-in, log pages  │                     │  every other request (with the user's login)
                                ▼                     ▼
   ┌──── NigraniAI Supabase project ────┐   ┌──── consent_service (Python) ────────────────┐
   │ Auth: the client's staff accounts  │◀──│ checks the login → permissions → usage meters │
   │ schema "guardrail":                │   │ consent · retention · breach detectors        │
   │   activity_logs, database_logs     │   │ anomaly model · PII detection · data map      │
   └──────────────────▲─────────────────┘   │ DPDP baseline · evidence ledger · billing     │
                      │ log lines           │ keeps its own state in consent_service/data/  │
   ┌──────────────────┴─────────────────┐   └───────┬───────────────────────┬───────────────┘
   │ The client's services              │           │ read-only (GET)        │ minimised context
   │ (ShopSmart's apps)                 │           ▼                        ▼
   └────────────────────────────────────┘   The client's database      AI provider (Groq)
                                            customers, orders, …
```

- **The browser** shows everything. It holds only public values (the Supabase URL and *anon* key).
- **`consent_service`** is one Python process. Anything that needs a secret or must be trusted runs
  there: permissions, the bridge to the client's database, detectors, the model, the AI proxy, billing.
- **Two databases, both only read.** Logs come from the NigraniAI Supabase project; customer data comes
  from the client's own database. NigraniAI never writes to either: retention produces worklists and
  notices are drafts.
- **State** (versioned rules, incidents, access policies, usage, invoices, the evidence ledger) is
  kept as JSON files in `dashboard/consent_service/data/`, created on first run and never committed.

Full architecture, every page and every API route: [dashboard/README.md](dashboard/README.md).

---

## 3. Technology

| Part | Technology | Notes |
|---|---|---|
| Dashboard UI | **HTML, CSS, JavaScript** | No framework, no build step, no Node. Each section is one JS module |
| Backend service | **Python 3.10+** | Standard library only: nothing to `pip install` for the main service |
| PII language model (optional) | Python 3.11 venv with Microsoft **Presidio** + **spaCy** | Finds names and places in free text; runs locally |
| Anomaly model | Isolation Forest written in plain Python | No ML libraries; learns per service |
| Logs and sign-in | **Supabase** (PostgreSQL, Auth, Row Level Security) | schema `guardrail` |
| Client data | The client's Supabase / PostgreSQL database | read-only |
| AI assistant | **Groq** (`openai/gpt-oss-120b`) | only aggregated, minimised context is sent |
| Payments | **Razorpay** | Checkout in the browser, verification on the server |
| Website | HTML, CSS, JavaScript + Supabase | hosted on **Vercel** |
| Production hosting | Linux server with nginx and HTTPS | run by the project lead |

---

## 4. Repository layout

```
NigraniAI/
├── README.md                     ← you are here
├── .gitignore                    keeps secrets, live data and caches out of git
│
├── dashboard/                    the compliance system each client gets
│   ├── README.md                 full documentation: every page, API route, setting and file
│   ├── .env.example              every server setting with a placeholder (copy it to .env)
│   ├── frontend/                 the browser app (serve this folder)
│   │   ├── index.html            sign-in, sidebar, every page's container, icons
│   │   ├── style.css             design system, light and dark themes
│   │   ├── config.js             public browser settings (Supabase URL, anon key, service address)
│   │   ├── app.js                session, navigation, permissions, DPDP baseline helper, live latency
│   │   ├── dashboard.js          main dashboard and charts
│   │   ├── telemetry.js          Log Telemetry
│   │   ├── consent.js            Consent & Purpose
│   │   ├── datamap.js            Data map (map, inventory, gaps, detection, declarations)
│   │   ├── retention.js          Retention
│   │   ├── breach.js             Breach Response
│   │   ├── anomaly.js            Anomaly detection
│   │   ├── ai.js                 AI Assistant
│   │   ├── readiness.js          Readiness score
│   │   ├── evidence.js           Evidence & audit
│   │   ├── iam.js                Access management
│   │   ├── usage.js              Usage & plan
│   │   ├── billing.js            Billing and invoices
│   │   ├── account.js            Account menu and page
│   │   └── assets/               logo, eye mark, favicon
│   ├── consent_service/          the Python service
│   │   ├── server.py             the service: API, permissions, detectors, model, AI proxy, billing…
│   │   ├── dpdp_baseline.py      which rules are fixed by law, and their limits
│   │   ├── pii_engine.py         PII detection: validators, checksums, fused verdicts
│   │   ├── pii_ner_worker.py     optional local language model (names, places)
│   │   ├── test_pii_engine.py    unit tests
│   │   ├── provider.json         plan catalogue, prices, seller details, billing rules
│   │   └── default_*.json        starting consent policy, retention schedule, detectors, data map
│   └── docs/                     design notes
│
└── website/                      the public website (static)
    ├── README.md                 how to run and deploy it
    ├── index.html, company.html  home page and company page
    ├── css/style.css, js/        styles, behaviour, Supabase settings for demo bookings
    ├── supabase/schema.sql       the demo_bookings table and its security policy
    ├── assets/                   logos and favicon
    └── vercel.json               static hosting settings and security headers
```

Never in git (see `.gitignore`): `.env` files, `dashboard/consent_service/data/` (live state),
`dashboard/consent_service/.pii-venv/` (the language model) and caches.

---

## 5. Getting started

### 5.1 What you need

| Tool | Version | Check with |
|---|---|---|
| Git | any recent | `git --version` |
| Python | **3.10 or newer** (3.11 for the optional PII language model) | `python3 --version` (Windows: `py --version`) |
| A modern browser | Chrome, Edge, Firefox or Safari | – |
| A GitHub account | – | – |

No Node.js, no package manager and no local database. **macOS, Linux and Windows all work.** The
commands below are for macOS and Linux; Windows (PowerShell) versions follow each block.

> **Windows:** install Python from [python.org](https://www.python.org/downloads/) and tick *Add
> python.exe to PATH*. Use `py` where this guide says `python3`. Before cloning, run
> `git config --global core.autocrlf input` once, so Git doesn't change line endings in your commits.

### 5.2 Fork and clone

1. On GitHub, open this repository and click **Fork** (top right). This makes your own copy.
2. Clone **your fork**:
   ```bash
   git clone https://github.com/<your-username>/NigraniAI.git
   cd NigraniAI
   ```
3. Connect it to the main repository so you can stay up to date:
   ```bash
   git remote add upstream https://github.com/atharva134kadam/NigraniAI.git
   git remote -v        # origin = your fork, upstream = the main repository
   ```

### 5.3 Get access

The dashboard connects to real Supabase projects, so you need values that are **not** in this
repository:

1. **Configuration values.** Ask the project lead for the dashboard `.env` values. They are shared over
   a private channel: never in an issue, a pull request, a group email or a commit.
2. **A sign-in account.** The dashboard's root admin creates your account on the **Access** page and
   attaches a permission policy to it.
3. **The website** needs nothing extra: its public Supabase settings are already in `website/js/config.js`.

### 5.4 Run the dashboard

```bash
cd dashboard
cp .env.example .env          # then open .env and paste in the values you were given
```

Open **two terminals** in the `dashboard` folder:

```bash
# Terminal 1: the Python service → http://localhost:8091
python3 consent_service/server.py
```
```bash
# Terminal 2: the browser app → http://localhost:8090
cd frontend && python3 -m http.server 8090
```

Open **http://localhost:8090** (use `localhost`, not `127.0.0.1`: see
[Troubleshooting](#12-troubleshooting)) and sign in with your account.

The service prints this when it is ready:

```
[consent] listening on http://localhost:8091  (allowed origins: http://localhost:8090)
```

**Windows (PowerShell)**, in two PowerShell windows inside the `dashboard` folder:

```powershell
copy .env.example .env                      # then edit .env in VS Code or Notepad
py consent_service\server.py                 # window 1: the service → http://localhost:8091
cd frontend; py -m http.server 8090          # window 2: the browser app → http://localhost:8090
```

Save `.env` as UTF-8 (the default in VS Code and in current Notepad).

On the first run the service creates `consent_service/data/` with the starting rules from the
`default_*.json` files. That folder is your local copy of the system's state and is ignored by git.

`frontend/config.js` already points at the shared Supabase project and, on `localhost`, at the
service on port 8091, so it normally needs no change.

### 5.5 (Optional) The PII language model

Without it, PII detection still runs its other four layers, and the Detection tab says "Language
model not installed". To add it (about 600 MB):

```bash
cd dashboard/consent_service
python3.11 -m venv .pii-venv
.pii-venv/bin/pip install "presidio-analyzer==2.2.*" "spacy==3.8.*"
.pii-venv/bin/python -m spacy download en_core_web_lg
```

On Windows (PowerShell):

```powershell
cd dashboard\consent_service
py -3.11 -m venv .pii-venv
.pii-venv\Scripts\pip install "presidio-analyzer==2.2.*" "spacy==3.8.*"
.pii-venv\Scripts\python -m spacy download en_core_web_lg
```

Restart the service; it finds `.pii-venv` by itself (`bin/python` or `Scripts\python.exe`). On a machine with little memory, download
`en_core_web_md` instead and set `PII_NER_MODEL=en_core_web_md` in `.env`.

### 5.6 Run the website

```bash
cd website
python3 -m http.server 3011
# open http://localhost:3011
```

The demo booking form writes to the shared Supabase `demo_bookings` table, so submitting it creates a
real booking. Use an obviously fake name when testing, or point it at your own Supabase project: run
`website/supabase/schema.sql` in that project's SQL editor and put its URL and anon key in
`website/js/config.js` (don't commit that change).

### 5.7 Useful service commands

```bash
python3 consent_service/server.py help    # lists every command
python3 consent_service/server.py status  # client, plan, usage this cycle, unpaid invoices
```

Other commands: `plans`, `plan set <plan>`, `invoices`, `bill-now`, `mark-paid <invoice> <ref>`,
`void <invoice>`, `refund <invoice>`. Run them only on your local copy unless the lead asks.

---

## 6. Configuration reference

### `dashboard/.env` (server only, never committed)

Start from [`dashboard/.env.example`](dashboard/.env.example).

| Setting | Required | What it is |
|---|---|---|
| `CLIENT_NAME`, `CLIENT_ID` | yes | The client this copy belongs to; shown across the dashboard |
| `SUPABASE_URL` | yes | NigraniAI Supabase project (sign-in and logs) |
| `SUPABASE_SERVICE_KEY` | yes | Server-only key for managing staff accounts |
| `GUARDRAIL_ANON_KEY` | yes | Public key used to verify sign-ins; logs are read with the user's own login |
| `SUPABASE_DB_SCHEMA` | no | Log schema, default `guardrail` |
| `SHOP_SUPABASE_URL`, `SHOP_SUPABASE_SERVICE_KEY` | yes | The client's database, read-only |
| `CONSENT_PORT` | no | Service port, default `8091` |
| `CONSENT_ALLOWED_ORIGINS` | no | Addresses allowed to call the service, default `http://localhost:8090` |
| `ROOT_EMAIL` | no | The client's root admin account |
| `GROK_API_KEY` | for the AI | AI assistant key (Groq) |
| `GROK_BASE_URL`, `GROK_MODEL`, `GROK_CONTEXT_CHARS` | no | Another AI endpoint, model and context size |
| `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`, `RAZORPAY_WEBHOOK_SECRET` | for payments | Use **test** keys while developing |
| `PII_NER_MODEL` | no | spaCy model for the language model, default `en_core_web_lg` |
| `CENTRAL_URL`, `CENTRAL_ANON_KEY`, `CENTRAL_CLIENT_KEY`, `CENTRAL_REPORT_MINUTES` | no | Optional central reporting; leave empty in development |

The service stops at start-up with `Missing in .env: …` if a required value is absent.

### `dashboard/frontend/config.js` (browser, public values only)

`DASHBOARD_URL`, `DASHBOARD_ANON_KEY` (public by design; Row Level Security protects the data),
`DB_SCHEMA`, and `CONSENT_API` (the service address: `http://localhost:8091` on your laptop, the
page's own address when hosted).

### `dashboard/consent_service/provider.json`

Plan catalogue, prices, GST and seller details, and billing rules. Change it only when pricing
changes; see [dashboard README §7.2](dashboard/README.md#72-plans-providerjson).

### `default_*.json`

The starting consent policy, retention schedule, breach detectors and data-map declarations. They are
also the **DPDP baseline**: the legal floor that saved rules are checked against. Edit them only when
the law or the client's starting setup changes, together with `dpdp_baseline.py`.

---

## 7. Running tests and checking your work

**Unit tests (PII engine):**
```bash
cd dashboard
python3 consent_service/test_pii_engine.py      # expect: Ran 25 tests … OK
```

**Quick syntax checks before committing:**
```bash
python3 -c "import ast; ast.parse(open('consent_service/server.py').read())"
node --check frontend/<file>.js                 # only if you have Node installed
```

**Manual check (always):**
1. Restart the service and reload the page with **Cmd/Ctrl + Shift + R**.
2. Open every page you touched, in light and dark theme, and at phone width (browser DevTools,
   device mode).
3. Keep the browser console open: there should be no red errors.
4. If you changed rules or detectors, try one change that should be allowed and one that weakens a
   legal rule. The second must be refused.

**Your local data:** everything you save goes into `consent_service/data/` on your machine only. To
start fresh, stop the service and delete that folder; it is recreated from the defaults.

---

## 8. Working together: forks, branches and pull requests

### Keep your fork up to date
```bash
git checkout main
git fetch upstream
git merge upstream/main        # or: git rebase upstream/main
git push origin main
```

### Make a change
1. Start from an up-to-date `main` and create a branch named after the work:
   ```bash
   git checkout -b feature/retention-export      # or fix/…, docs/…
   ```
2. Make small, focused commits, with messages that say what the change does:
   ```
   Retention: export the due-soon worklist as CSV
   ```
3. Push the branch to **your fork** and open a **pull request** to `atharva134kadam/NigraniAI` → `main`:
   ```bash
   git push -u origin feature/retention-export
   ```
4. In the pull request, say **what** changed, **why**, and **how you tested it**, and add screenshots
   for anything visible.
5. Wait for a review. Only the project lead merges into `main`.

### Before you open a pull request
- [ ] It runs locally and the pages you touched work (section 7).
- [ ] The unit tests pass.
- [ ] No secrets, `.env` values, personal data, real customer data or `data/` files in the diff
      (check `git diff --staged` before committing).
- [ ] If you changed a `.js` or `.css` file, you increased its `?v=` number in `frontend/index.html`
      (section 9).
- [ ] The documentation is updated if behaviour changed (`dashboard/README.md`).

---

## 9. Coding conventions

**Frontend (`dashboard/frontend/`)**
- Plain JavaScript, no framework and no build. Each section is one module wrapped in a function
  (`const Consent = (() => { … })();`) that exposes `show()` and, for live pages, `tick()`.
- Shared helpers live in `app.js`: `esc()` for every value put into HTML, `can()` for permissions,
  `Baseline` for the DPDP locks.
- **Always escape** text with `esc()` before inserting it into HTML.
- **Cache-busting:** files load as, for example, `app.js?v=64`. When you change a JS or CSS file,
  increase its number in `index.html`, or browsers keep the old file.
- Colours come from CSS variables in `style.css` (`--primary`, `--surface`…), defined for light and
  dark themes. Don't hard-code colours.
- Times are shown in IST (Asia/Kolkata).

**Backend (`dashboard/consent_service/`)**
- Python **standard library only** in `server.py`. If you think you need a package, discuss it first.
- Every route is mapped to a permission action in the route table; a new route without one is denied.
- Saving rules goes through `save_doc()`, which applies the DPDP baseline. Don't bypass it.
- Write state with the existing helpers, so every change is fingerprinted in the evidence ledger.

**Website (`website/`)**: plain HTML, CSS and JavaScript; see [website/README.md](website/README.md).

**Writing**: plain English and short sentences in the UI and docs, Indian English spelling
(organisation, licence), and legal references in the form `DPDP Act 2023 · s.8(7)` or
`DPDP Rules 2025 · Rule 7`.

---

## 10. Rules we never break

1. **No secrets in git.** Service keys, AI keys, Razorpay secrets and `.env` files stay on your machine
   or the server. Only Supabase **anon** keys appear in browser code, by design.
2. **Never write to a client's database.** Every request to it is a `GET`. Retention produces
   worklists; it doesn't delete.
3. **Minimise personal data.** It is masked or aggregated before it reaches the browser or the AI.
   Never paste real customer data into issues, pull requests, chats or screenshots.
4. **The law can't be weakened.** Rules in the DPDP baseline can be made stricter, never weaker. If the
   law changes, update `dpdp_baseline.py` and the defaults together.
5. **No real data in tests.** Use made-up names, emails and numbers.

Found a security problem? Don't open a public issue: message the project lead directly.

---

## 11. Deployment

| Part | Where | How it updates |
|---|---|---|
| Website | Vercel: https://nigrani-ai.vercel.app | Automatically on every push to `main` of the main repository |
| Dashboard | A hosted Linux server (nginx + HTTPS) run by the project lead | The lead deploys reviewed changes from `main` |

Contributors don't deploy: get your change merged into `main` and the lead takes it from there.

---

## 12. Troubleshooting

| Problem | Fix |
|---|---|
| `Missing in .env: SHOP_SUPABASE_URL, …` | Copy `.env.example` to `.env` in `dashboard/` and fill in every required value |
| `python3` is not recognised (Windows) | Use `py` instead, or reinstall Python with *Add python.exe to PATH* ticked |
| `Address already in use` | Something already uses port 8090 or 8091. Stop it, or use another port and update `CONSENT_ALLOWED_ORIGINS` and `config.js` |
| Pages load but every request fails (CORS error in the console) | Open `http://localhost:8090`, not `127.0.0.1`, or add your address to `CONSENT_ALLOWED_ORIGINS` |
| "Sign in to the dashboard first." | Your session expired or your account doesn't exist yet: sign in again, or ask for an account |
| "Access denied: … Ask the root user for access." | Your account lacks a permission. Ask the root admin to attach a policy |
| My change doesn't show | Hard-reload (Cmd/Ctrl + Shift + R) and check that you increased the `?v=` number |
| "Not saved: the DPDP baseline can be made stricter but not weaker." | Working as intended: that change would weaken a legal rule |
| "Language model not installed" | Optional: see [5.5](#55-optional-the-pii-language-model) |
| The hosted dashboard is blocked on a college or office network ("Dynamic DNS" category) | Use mobile data; the hosted address will move to a proper domain |

---

## 13. DPDP glossary

| Term | Meaning |
|---|---|
| **DPDP Act** | Digital Personal Data Protection Act, 2023: India's personal data law |
| **DPDP Rules** | DPDP Rules, 2025: the detailed rules under the Act; full obligations apply from 13 May 2027 |
| **Data Principal** | The person the data is about (a customer) |
| **Data Fiduciary** | The company that decides why and how data is processed (our client) |
| **Data Processor** | A company processing data on the fiduciary's behalf (for example, a payment gateway) |
| **Board** | The Data Protection Board of India, which handles breaches and complaints |
| **Personal data breach** | Any unauthorised processing, disclosure, alteration, loss of access or destruction (s.2(u)) |
| **Rule 7** | Breach intimation: tell affected people and the Board without delay, with a detailed report within 72 hours |
| **CERT-In** | India's cyber-security agency: cyber incidents are reported to it within 6 hours |
| **Rule 8** | Erasure once the purpose ends (with 48 hours' warning) and keeping logs for at least one year |
| **Purpose limitation** | Data may be used only for the purpose the person agreed to (s.5, s.6) |

---

## 14. Roadmap

| Phase | Focus |
|---|---|
| **1. Scale the core** | A dedicated log-ingestion service (collector, queue, batch writer) aiming at 10,000 log lines per second; monitoring, backups, CI |
| **2. Deepen each feature** | Hot/warm/cold log storage, ready-made log connectors, smarter Indian-format PII detection, security hardening (MFA, SSO, penetration test) |
| **3. Enterprise** | Many clients with strict separation, high availability, an on-premise installer, ISO 27001 / SOC 2 |

Pick up work from the issues assigned to you, or ask the project lead what to take next.

---

## 15. Licence and contact

**Licence:** this repository has no open-source licence, so all rights are reserved by the owner. It
is shared with the team for development; don't publish or reuse the code elsewhere without
permission.

**Project lead:** [@atharva134kadam](https://github.com/atharva134kadam) · atharvakadam11056@gmail.com
