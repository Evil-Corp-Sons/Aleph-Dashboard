# Running Aleph Dashboard V2 Locally

A complete step-by-step guide for setting up and running the Aleph Dashboard from a fresh clone.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Clone the Repository](#2-clone-the-repository)
3. [Get Your API Keys](#3-get-your-api-keys)
4. [Configure Environment Variables](#4-configure-environment-variables)
5. [Create a Python Virtual Environment](#5-create-a-python-virtual-environment)
6. [Install Dependencies](#6-install-dependencies)
7. [Start the Server](#7-start-the-server)
8. [Open the Dashboard](#8-open-the-dashboard)
9. [Populate Data Faster (Optional)](#9-populate-data-faster-optional)
10. [Verify Everything Is Working](#10-verify-everything-is-working)
11. [Configuration Reference](#11-configuration-reference)
12. [Troubleshooting](#12-troubleshooting)

---

## 1. Prerequisites

Make sure the following are installed on your machine before you start:

| Requirement | Minimum Version | How to Check |
|-------------|----------------|--------------|
| **Python** | 3.9 or higher | `python --version` |
| **pip** | (comes with Python) | `pip --version` |
| **Git** | Any recent version | `git --version` |

### Installing Python

- **Windows:** Download the installer from [python.org/downloads](https://www.python.org/downloads/). During installation, **check the box that says "Add Python to PATH"** — this is critical.
- **macOS:** Use Homebrew: `brew install python`
- **Linux (Debian/Ubuntu):** `sudo apt update && sudo apt install python3 python3-pip python3-venv`

To confirm Python is installed and on your PATH, open a terminal and run:

```bash
python --version
```

You should see something like `Python 3.11.5` (any version 3.9+ is fine).

> **Windows note:** If `python` doesn't work, try `python3` or `py`. Use whichever command responds with a 3.9+ version throughout this guide.

---

## 2. Clone the Repository

Open a terminal (Command Prompt, PowerShell, Terminal.app, or your Linux terminal) and run:

```bash
git clone https://github.com/your-username/aleph-dashboard-v2.git
cd aleph-dashboard-v2
```

> Replace `your-username/aleph-dashboard-v2` with the actual GitHub repository URL.

After cloning, your directory should contain files like `CLAUDE.md`, `requirements.txt`, `server/`, `dashboard/`, etc. You can verify with:

```bash
ls
```

---

## 3. Get Your API Keys

The dashboard requires two external API keys to function. Both have free tiers.

### 3a. OpenAI API Key (Required)

This powers the AI extraction and synthesis (turning raw news into strategic intelligence).

1. Go to [platform.openai.com](https://platform.openai.com/) and create an account (or sign in).
2. Navigate to **API Keys**: [platform.openai.com/api-keys](https://platform.openai.com/api-keys)
3. Click **"Create new secret key"**.
4. Copy the key immediately — you won't be able to see it again.
5. **Billing:** OpenAI requires a payment method. The dashboard uses GPT-4o by default. Typical usage costs a few dollars per month depending on how frequently the pipelines run.

### 3b. Brave Search API Key (Required)

This powers the news discovery pipeline (finding competitor news articles).

1. Go to [brave.com/search/api](https://brave.com/search/api/) and create an account.
2. Choose the **Free plan** (2,000 queries/month — more than enough for default settings).
3. After signing up, go to your dashboard and find your API key.
4. Copy the key.

### 3c. Admin API Key (Optional)

If you want to protect the `POST /api/run` endpoint (manual pipeline trigger) from external access, you can generate a random token:

```bash
python -c "import secrets; print(secrets.token_urlsafe(32))"
```

This is optional — the endpoint already allows requests from the dashboard itself without a key.

---

## 4. Configure Environment Variables

The application reads configuration from a `.env` file in the project root.

1. **Copy the example file:**

   ```bash
   # macOS / Linux
   cp .env.example .env

   # Windows (Command Prompt)
   copy .env.example .env

   # Windows (PowerShell)
   Copy-Item .env.example .env
   ```

2. **Open `.env` in any text editor** (VS Code, Notepad, vim, nano, etc.) and fill in your API keys:

   ```ini
   # --- REQUIRED — paste your keys here ---
   BRAVE_SEARCH_API_KEY=your_brave_api_key_here
   OPENAI_API_KEY=your_openai_api_key_here
   ```

3. **Save the file.**

That's the only required configuration. Everything else has sensible defaults. See [Configuration Reference](#11-configuration-reference) below for optional settings you can tweak.

> **Important:** Never commit your `.env` file to Git. It is already listed in `.gitignore`.

---

## 5. Create a Python Virtual Environment

A virtual environment keeps this project's dependencies isolated from your system Python. This step is strongly recommended.

### Windows

```bash
python -m venv venv
venv\Scripts\activate
```

After activation, your terminal prompt will show `(venv)` at the beginning.

### macOS / Linux

```bash
python3 -m venv venv
source venv/bin/activate
```

After activation, your terminal prompt will show `(venv)` at the beginning.

> **To deactivate later:** Just type `deactivate` in the terminal.
>
> **Every time you open a new terminal** to work on this project, you need to activate the virtual environment again using the same activation command above.

---

## 6. Install Dependencies

With your virtual environment activated (you should see `(venv)` in your prompt), run:

```bash
pip install -r requirements.txt
```

This installs all required Python packages:

| Package | Purpose |
|---------|---------|
| `flask` | Web server and API framework |
| `flask-cors` | Cross-origin request handling |
| `requests` | HTTP client for fetching status pages and APIs |
| `feedparser` | RSS/Atom feed parsing |
| `openai` | OpenAI GPT-4o API client |
| `apscheduler` | Background task scheduling |
| `python-dotenv` | Loads `.env` file into environment |
| `tenacity` | Retry logic for flaky API calls |
| `pytest` | Test runner (development only) |

The install typically takes under a minute. You should see `Successfully installed ...` at the end.

---

## 7. Start the Server

Run the following command from the **project root directory** (the folder containing `CLAUDE.md`):

```bash
python -m server.api_server
```

> **Critical:** Always use `python -m server.api_server` (module syntax). Do **not** run `python server/api_server.py` directly — it will fail with import errors because the module paths won't resolve correctly.

### What happens when the server starts

You'll see log output in your terminal as the server boots up:

1. **Database initialization** — Creates the SQLite database at `data/aleph_v2.db` if it doesn't exist. If it already exists, this step is a safe no-op.
2. **Competitor seeding** — Populates the database with the 17 tracked cloud providers (AWS, Azure, GCP, CoreWeave, etc.).
3. **Background schedulers start** — Three recurring pipelines begin running automatically:

   | Pipeline | Runs Every | What It Does |
   |----------|-----------|--------------|
   | **Status** | 12 minutes | Polls provider status pages for outages and incidents |
   | **Discovery** | 45 minutes | Fetches RSS feeds + runs Brave Search queries, then uses GPT-4o to extract events |
   | **Synthesis** | 60 minutes | Uses GPT-4o to generate macro trend analysis and strategic signals from collected events |

4. **Web server starts** — You'll see a line like:
   ```
   Aleph Dashboard V2 starting on http://0.0.0.0:8080
   ```

The server is now running. **Keep this terminal open** — closing it stops the server.

---

## 8. Open the Dashboard

Open your web browser and go to:

### [http://localhost:8080](http://localhost:8080)

You should see the Aleph Dashboard V2.2 interface — a dark-themed (Palantir Gotham-style) intelligence dashboard.

### First launch: the dashboard will be mostly empty

This is normal. The background pipelines need time to:
- Fetch status pages (~1-2 minutes for the first status poll)
- Discover news via RSS and Brave Search (~2-5 minutes)
- Run GPT-4o extraction and synthesis (~3-5 minutes after discovery)

After about 5-10 minutes, refresh the page and you should start seeing data populate across all sections.

If you don't want to wait, see the next section.

---

## 9. Populate Data Faster (Optional)

Instead of waiting for the background schedulers, you can trigger the full pipeline manually.

### Option A: Use the manual trigger script

Open a **second terminal window**, navigate to the project directory, activate your virtual environment, and run:

```bash
# Activate venv first (see Step 5)
python tools/run_pipeline.py
```

This runs the complete pipeline (status + discovery + extraction + synthesis) synchronously. It will print a JSON summary when finished. This usually takes 2-5 minutes depending on API response times.

### Option B: Use the API endpoint

With the server running, you can trigger the pipeline via HTTP:

```bash
curl -X POST http://localhost:8080/api/run
```

Or in PowerShell:

```powershell
Invoke-RestMethod -Method POST -Uri http://localhost:8080/api/run
```

You should get back `{"status": "started"}`. The pipeline runs in the background — check the server terminal for progress logs.

### After running the pipeline

Refresh the dashboard at [http://localhost:8080](http://localhost:8080). You should now see:

- **Server Outages Rail** — Live status of all 17 providers
- **Headlines Carousel** — Top critical/high severity events
- **Tactical Feed** — Recent competitive intelligence events
- **Macro Trend** — AI-generated trend analysis
- **Momentum Synthesis** — Thematic intelligence cards
- **Strategic Signals** — Actionable strategic recommendations
- **Watchlist** — Providers to watch

---

## 10. Verify Everything Is Working

You can check the health endpoint to confirm all systems are operational:

```bash
curl http://localhost:8080/api/v2/health
```

Or open [http://localhost:8080/api/v2/health](http://localhost:8080/api/v2/health) in your browser.

This returns a JSON object showing:
- Pipeline last-run timestamps
- Brave Search budget usage
- Whether synthesis data is fresh or stale

### Quick API smoke test

Try these URLs in your browser to verify data is flowing:

| URL | What You Should See |
|-----|---------------------|
| `http://localhost:8080/api/v2/stats` | Event counts and severity breakdown |
| `http://localhost:8080/api/v2/csp-status` | Status of all 17 providers |
| `http://localhost:8080/api/v2/headlines` | Top headlines (may be empty until first pipeline run) |
| `http://localhost:8080/api/v2/events` | List of extracted intelligence events |
| `http://localhost:8080/api/v2/synthesis/trend` | AI-generated macro trend analysis |
| `http://localhost:8080/api/v2/synthesis/signals` | Strategic signal cards |
| `http://localhost:8080/api/v2/momentum` | Thematic momentum cards + watchlist |

---

## 11. Configuration Reference

All settings are configured via environment variables in your `.env` file. Only the two API keys are required — everything else has working defaults.

### Required

| Variable | Description |
|----------|-------------|
| `BRAVE_SEARCH_API_KEY` | Your Brave Search API key |
| `OPENAI_API_KEY` | Your OpenAI API key |

### Optional — Server

| Variable | Default | Description |
|----------|---------|-------------|
| `SERVER_PORT` | `8080` | Port the web server listens on |
| `SERVER_HOST` | `0.0.0.0` | Host binding (`0.0.0.0` = all interfaces, `127.0.0.1` = localhost only) |
| `ADMIN_API_KEY` | _(empty)_ | Bearer token required for `POST /api/run` from external clients |

### Optional — AI Models

| Variable | Default | Description |
|----------|---------|-------------|
| `OPENAI_MODEL` | `gpt-4o` | Model used for event extraction |
| `OPENAI_SYNTHESIS_MODEL` | `gpt-4o` | Model used for trend/signal synthesis |

### Optional — Pipeline Intervals

| Variable | Default | Description |
|----------|---------|-------------|
| `STATUS_POLL_SECONDS` | `720` (12 min) | How often to poll provider status pages |
| `DISCOVERY_INTERVAL_MINUTES` | `45` | How often to run RSS + Brave discovery |
| `SYNTHESIS_INTERVAL_MINUTES` | `60` | How often to regenerate AI synthesis |

### Optional — Brave Budget

| Variable | Default | Description |
|----------|---------|-------------|
| `BRAVE_MAX_MONTHLY_CALLS` | `2000` | Monthly Brave API call limit (matches free tier) |
| `BRAVE_DEFAULT_COUNT` | `5` | Results per Brave query |
| `BRAVE_QUERY_LIMIT_PER_RUN` | `10` | Max Brave queries per discovery run |

### Optional — Advanced

| Variable | Default | Description |
|----------|---------|-------------|
| `DB_PATH` | `data/aleph_v2.db` | Path to SQLite database |
| `MIN_SEVERITY` | `Medium` | Minimum severity for event extraction |
| `DOCUMENT_BATCH_SIZE` | `40` | Documents processed per extraction batch |
| `TREND_MAX_AGE_MINUTES` | `180` | Max age before trend data is considered stale |
| `SIGNALS_MAX_AGE_MINUTES` | `180` | Max age before signal data is considered stale |

---

## 12. Troubleshooting

### "No module named 'server'" error

You ran `python server/api_server.py` instead of `python -m server.api_server`. Always use the `-m` module syntax.

Also make sure you're running the command from the **project root directory** (the folder that contains `server/`, `dashboard/`, and `CLAUDE.md`).

### Dashboard loads but shows no data

- **First launch?** The pipelines need a few minutes to collect and process data. Wait 5-10 minutes and refresh, or run the pipeline manually (see [Step 9](#9-populate-data-faster-optional)).
- **Check the terminal** where the server is running for error messages. Common issues:
  - `OPENAI_API_KEY` is missing or invalid
  - `BRAVE_SEARCH_API_KEY` is missing or invalid
  - OpenAI billing is not set up (free trial expired)

### API returns stale data after editing Python code

Flask doesn't auto-reload cached bytecode in production mode. Delete the `__pycache__` directories and restart:

```bash
# macOS / Linux
find . -type d -name __pycache__ -exec rm -rf {} +

# Windows (PowerShell)
Get-ChildItem -Recurse -Directory -Filter __pycache__ | Remove-Item -Recurse -Force

# Windows (Command Prompt)
for /d /r . %d in (__pycache__) do @if exist "%d" rd /s /q "%d"
```

Then restart the server with `python -m server.api_server`.

### "Address already in use" error

Another process is using port 8080. Either:
- Stop the other process, or
- Change the port in your `.env` file: `SERVER_PORT=9090` and access the dashboard at `http://localhost:9090`

### Brave Search quota exhausted

The free Brave plan allows 2,000 queries/month. If you hit the limit:
- The discovery pipeline will skip Brave queries (RSS feeds still work)
- Check usage at [api.search.brave.com](https://api.search.brave.com/)
- The quota resets on the 1st of each month

### OpenAI API errors (rate limits / billing)

- Ensure your OpenAI account has billing set up at [platform.openai.com/account/billing](https://platform.openai.com/account/billing)
- Check your usage limits at [platform.openai.com/account/limits](https://platform.openai.com/account/limits)
- The pipeline uses retry logic with exponential backoff, so transient errors usually resolve themselves

### Virtual environment not activating

- **Windows:** Make sure you use `venv\Scripts\activate` (backslashes), not `venv/Scripts/activate`
- **PowerShell:** You may need to run `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser` first to allow script execution
- **macOS/Linux:** Make sure you use `source venv/bin/activate`

### Python version too old

If you see syntax errors or missing features, verify your Python version:
```bash
python --version
```
You need Python 3.9 or higher. On some systems, you may need to use `python3` instead of `python`.

---

## Stopping the Server

Press `Ctrl+C` in the terminal where the server is running. This gracefully shuts down the web server and background schedulers.

---

## Quick Start Summary

```bash
git clone https://github.com/your-username/aleph-dashboard-v2.git
cd aleph-dashboard-v2
cp .env.example .env              # then edit .env with your API keys
python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate
pip install -r requirements.txt
python -m server.api_server       # open http://localhost:8080
```
