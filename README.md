# Incident Copilot

A resumable, multi-agent incident triage system built with LangGraph and Gemini.

Given an alert, the system acts as an incident response team:

1. **Router (Incident Commander)** — classifies severity and decides which specialists to dispatch.
2. **Specialists** — logs, deploys, and metrics investigators run in parallel, each using one tool to gather evidence.
3. **Analyst** — merges every specialist's findings into a single diagnosis with proposed remediation actions.
4. **Policy check** — decides which actions are risky enough (high severity, or language like "restart" / "delete" / "production") to require human sign-off.
5. **Approval gates** — the workflow pauses (`interrupt()`) once per gated action, routed to the team that owns the affected service, and resumes from a durable checkpoint once a decision is recorded.
6. **Runbook** — approved actions and the full audit trail are published as a Markdown runbook.

A read-only status page shows the live state of any incident thread in the browser.

## What's in the repo

| Path | What it is |
|---|---|
| `app.py` | The LangGraph workflow: router, parallel specialists, analyst, policy check, per-team approval gates, runbook publishing. `start` and `resume` CLI commands. |
| `specialists.py` | The Gemini model connection, fixture loader, and the three specialist agents (logs, deploys, metrics), each with one tool. |
| `status_page.py` | Local web UI showing live incident state. `read_incident()` reads a thread's saved checkpoint via `get_state()`, so viewing it never advances or resumes the graph. |
| `requirements.txt` | Pinned packages (`langgraph`, `langgraph-checkpoint-sqlite`, `langchain`, `langchain-google-genai`). |
| `data/alerts.json` | Two alerts to triage: `INC-2041` (checkout-api 5xx errors) and `INC-2042` (search-api latency). |
| `data/logs.jsonl` | Application logs from the services involved, one JSON object per line. |
| `data/deploys.json` | Recent deploys, with the service, owning team, time, and change. |
| `data/metrics.json` | Metric time series per service. |
| `data/services.json` | Service catalog: owning team, on-call handle, and dependencies. |

The data is made up and small on purpose — agents read it through tools, so runs are cheap and the answers can be checked.

## Set up (Windows)

You need a Google Cloud project with billing enabled, and the [Google Cloud CLI](https://cloud.google.com/sdk/docs/install) installed.

```powershell
gcloud init
gcloud auth application-default login
gcloud config set project YOUR_PROJECT_ID
gcloud services enable aiplatform.googleapis.com

git clone https://github.com/mayan05/incident-copilot.git
cd incident-copilot
python -m venv .venv
.venv\Scripts\activate
python -m pip install -r requirements.txt
```

Set `GOOGLE_CLOUD_PROJECT` so the app can find your project:

```powershell
setx GOOGLE_CLOUD_PROJECT "your-project-id"
```

(`setx` only takes effect in new terminals — restart your terminal after running it, or use `$env:GOOGLE_CLOUD_PROJECT="your-project-id"` to set it for the current session only.)

If the first model call returns `403 PERMISSION_DENIED`, the API was enabled only moments ago — wait a few minutes and try again.

## Running the workflow

### Start a new incident

```powershell
python app.py start --thread inc-2041 --alert INC-2041
```

This loads the `INC-2041` fixture, runs the router and specialists, produces a diagnosis, and runs the policy check. If any action needs approval, the run pauses:

```
WORKFLOW PAUSED: 2 team approvals pending
- checkout-team (checkout-api): roll back config deploy dep-9183
- payments-team (payments-api): restart payment worker pool
```

If nothing needs approval, it publishes straight to a runbook under `runbooks/`.

### Approve or reject a pending action

Resume the same thread, naming the team and decision:

```powershell
python app.py resume --thread inc-2041 --team checkout-team --decision approve
python app.py resume --thread inc-2041 --team payments-team --decision reject
```

Each call resolves only the interrupts owned by that team. Once every gated action has a decision, the run continues on to `publish_runbook` and writes `runbooks/inc-2041.md`.

### Retry a failed run

If a branch raises (e.g. a flaky tool call), resume without `--decision` to retry from the last checkpoint:

```powershell
python app.py resume --thread inc-2041
```

### Try the other fixture

```powershell
python app.py start --thread inc-2042 --alert INC-2042
```

## Testing individual pieces

### Run one specialist in isolation

Useful for checking a specialist's tool and prompt without going through the full graph:

```powershell
python specialists.py logs INC-2041
python specialists.py deploys INC-2041
python specialists.py metrics INC-2042
```

Each prints the specialist's structured finding (summary, evidence, suspected cause, severity) as JSON.

### Simulate a specialist failure

`metric_window` can be forced to raise, to exercise the failure/resume path:

```powershell
$env:TRIAGE_FAIL="metrics"
python app.py start --thread fail-test --alert INC-2041
```

Use `display_failure`'s output (saved findings, nodes waiting to re-run) to confirm which branches completed before the crash, then resume normally:

```powershell
Remove-Item Env:\TRIAGE_FAIL
python app.py resume --thread fail-test
```

### View the status page

```powershell
python status_page.py
```

Opens a server on port 8080 — open `http://localhost:8080` in your browser. The page lists every saved thread in the left nav and auto-refreshes every 5 seconds, so you can start/resume a run in one terminal and watch its state change in the browser.

### Full test suite (solution branch)

The `solution` branch has offline tests that don't require live Gemini calls:

```powershell
git checkout solution
python -m pip install pytest
python -m pytest
```

Use it to check your own implementation against the finished build if you get stuck.

## Cost

Each run makes a few Gemini 3.7 Flash calls, which costs a few cents at most.

## Branches

- `main` — the finished build: router, parallel specialists, policy-gated approvals, runbook publishing, and a live status page.
- `solution` — the reference build, with offline tests (`python -m pytest`).
