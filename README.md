# AIRS Executive Report — Standalone HTML

Generates **one self-contained HTML file** summarizing the three Prisma AIRS pillars into an executive-level view, for a selectable time window. No Streamlit, no server, nothing to keep running — open the file in any browser, email it, or produce it headless in CI.

| Pillar | Source | Mode |
|---|---|---|
| **API-Intercept** (AI Runtime Security) | SCM Log Viewer CSV export | Offline (CSV) |
| **Model Scanning** | `model-security-client` SDK | Live API |
| **Red-Teaming** | AI Red-Teaming REST API | Live API |

Each pillar **degrades gracefully** to an "unavailable" note if its inputs are missing, so the report always renders. Cross-platform — runs on Windows and macOS.

> Built by the TME team for Prisma AIRS.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## Get the code

```bash
git clone https://github.com/ansh13/airs-html-report.git
cd airs-html-report
```

> This is a **public** tool, but it expects a service-account credential at runtime. **Never commit your real `credentials.json` or any CSV export** — both are git-ignored by default (see [Security notes](#security-notes)).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## Prerequisites

- **API-Intercept pillar:** any Python 3 — it uses only the standard library plus your CSV.
- **Model Scanning + Red-Teaming pillars:** a Prisma SASE **service account** with access to the AIRS APIs, network access, and — for Model Scanning — the private SDK, which requires **Python 3.11–3.13** (the SDK does **not** support 3.14).

### Pick the right Python (3.11–3.13)

If your default `python3` is 3.14, the Model Scanning SDK won't install. Use an explicit 3.12/3.13 interpreter, ideally inside a virtual environment:

**macOS / Linux**
```bash
python3.12 -m venv .venv
source .venv/bin/activate
```

**Windows (PowerShell)**
```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
```

> There are no mandatory public dependencies. The `model-security-client` SDK is **not on public PyPI** — do not `pip install model-security-client` from PyPI (that name is a broken stub). The script **auto-installs the real SDK from the private Palo Alto index on first run**, using your `credentials.json`. If that install can't succeed (most often because you're on Python 3.14), the Model Scanning section shows a notice and the other two pillars keep working.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## Setup

### 1. Credentials

Copy the example file and fill in your real service-account values:

```bash
cp example-credentials.json credentials.json
```

```json
{
  "client_id": "svc@123456789.iam.panserviceaccount.com",
  "client_secret": "YOUR_SERVICE_ACCOUNT_SECRET",
  "tsg_id": "123456789"
}
```

- `tsg_id` is **optional** — if omitted, it is derived from the numeric part of `client_id`.
- The same single token is used for all SASE API calls.
- `credentials.json` is **git-ignored** and will never be committed. Only `example-credentials.json` is tracked.

### 2. (Optional) API-Intercept CSV

To populate the API-Intercept pillar, export the log from SCM:

> **SCM → Log Viewer → log type _Prisma AIRS AI Runtime Security API_ → Export (CSV)**

Then either drop the file next to the script (its name must start with `ai_security_airs_api_intercept`), or pass its path explicitly (see below). If no CSV is found, the other two pillars still work — this pillar just shows a notice.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## Run

```bash
# newest CSV in the folder, last 7 days, writes ./airs_report.html
python3 html_report.py

# explicit CSV + 7-day window
python3 html_report.py "ai_security_airs_api_intercept_2026-11-16.csv" --days 7

# group API-Intercept by a different column and write somewhere specific
python3 html_report.py export.csv --group-by "AI Application Name" --out /tmp/report.html
```

Open the generated `airs_report.html` in any browser.

### Command-line options

| Flag | Purpose | Default |
|---|---|---|
| `csv_path` (positional) | API-Intercept CSV export to read | newest `ai_security_airs_api_intercept*.csv` in the script/run folder |
| `--days N` | Time window for **all three** pillars | `7` |
| `--group-by COL` | Column to group API-Intercept by | `API Key Name` |
| `--creds PATH` | Path to `credentials.json` | script folder, then cwd |
| `--out PATH` | Output HTML file path | `airs_report.html` next to the script |

### How the `--days` window applies

`--days` filters **all three pillars consistently**:

- **Model Scanning** and **Red-Teaming** query their APIs with the time range directly.
- **API-Intercept** filters CSV rows by the **`Time Generated`** column, keeping only rows within the window. The source line in the report shows how many rows were excluded, e.g. *"last 3 days (1,446,776 of 1,500,000 rows outside window excluded)"*.
- If an **older CSV export has no `Time Generated` column**, the API-Intercept pillar falls back to the full export (no date filter) rather than failing — in that case, export a CSV already scoped to your window.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## Where the script looks for files

The script searches for `credentials.json` and the API-Intercept CSV in this order:

1. The script's own folder (where `html_report.py` lives)
2. The directory you launched the command from

The `API_INTERCEPT_CSV` environment variable overrides the CSV search entirely.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## What you'll see

- **API-Intercept** — total requests, benign vs. violations (with request/response split), blocked vs. alert, and per-group reason-of-violation breakdown.
- **Model Scanning** — total scans, allowed / allowed-with-alerts / blocked, unique models scanned, and top violation reasons.
- **Red-Teaming** — targets configured, scans run, targets scanned, and a table of scans in the window. Shows a clear notice if the service account isn't entitled for Red-Teaming (HTTP 401).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## Security notes

- **Never commit `credentials.json`** or any real CSV export — both are git-ignored. The CSV can contain scanned content and PII.
- The **generated `airs_report.html` is git-ignored** too — it can contain tenant data (API key names, violations, model names).
- Treat the service-account secret like a password. Rotate it if it is ever exposed.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| Model Scanning shows "unavailable" | On Python 3.14 (SDK needs 3.11–3.13), or no network / SDK install failed. |
| API-Intercept shows "No CSV found" | No export present — pass a path or set `API_INTERCEPT_CSV`. |
| API-Intercept not filtered by `--days` | Your CSV export lacks a `Time Generated` column — export a window-scoped CSV. |
| Red-Teaming shows HTTP 401 | Service account not entitled for the AI Red-Teaming API. |
| "No credentials.json found" | File missing from the script/run folder — copy `example-credentials.json`. |
