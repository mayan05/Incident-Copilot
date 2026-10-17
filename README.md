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
| `status_page.py` | Local web UI for Cloud Shell Web Preview. Reads an incident's saved checkpoint state and renders it. |
| `requirements.txt` | Pinned packages (`langgraph`, `langgraph-checkpoint-sqlite`, `langchain`, `langchain-google-genai`). |
| `data/alerts.json` | Two alerts to triage: `INC-2041` (checkout-api 5xx errors) and `INC-2042` (search-api latency). |
| `data/logs.jsonl` | Application logs from the services involved, one JSON object per line. |
| `data/deploys.json` | Recent deploys, with the service, owning team, time, and change. |
| `data/metrics.json` | Metric time series per service. |
| `data/services.json` | Service catalog: owning team, on-call handle, and dependencies. |

The data is made up and small on purpose — agents read it through tools, so runs are cheap and the answers can be checked.

## Set up

Runs anywhere Python does — no Google Cloud project, billing, or `gcloud` CLI required.

1. Get a free Gemini API key from [aistudio.google.com/apikey](https://aistudio.google.com/apikey).
2. Clone and install:

   ```bash
   git clone https://github.com/mayan05/incident-copilot.git
   cd incident-copilot
   python3 -m venv .venv
   source .venv/bin/activate   # Windows: .venv\Scripts\activate
   python3 -m pip install -r requirements.txt
   ```

3. Set your API key as an environment variable (do this once per shell, or add it to your shell profile / a `.env` file — never commit it):

   ```bash
   export GOOGLE_API_KEY="your-key-here"          # macOS/Linux
   setx GOOGLE_API_KEY "your-key-here"             # Windows (restart terminal after)
   ```

If a model call returns a permission or quota error, double-check the key is set in the terminal you're running from (`echo $GOOGLE_API_KEY`).

## Running the workflow

### Start a new incident

```bash
python3 app.py start --thread inc-2041 --alert INC-2041
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

```bash
python3 app.py resume --thread inc-2041 --team checkout-team --decision approve
python3 app.py resume --thread inc-2041 --team payments-team --decision reject
```

Each call resolves only the interrupts owned by that team. Once every gated action has a decision, the run continues on to `publish_runbook` and writes `runbooks/inc-2041.md`.

### Retry a failed run

If a branch raises (e.g. a flaky tool call), resume without `--decision` to retry from the last checkpoint:

```bash
python3 app.py resume --thread inc-2041
```

### Try the other fixture

```bash
python3 app.py start --thread inc-2042 --alert INC-2042
```

## Testing individual pieces

### Run one specialist in isolation

Useful for checking a specialist's tool and prompt without going through the full graph:

```bash
python3 specialists.py logs INC-2041
python3 specialists.py deploys INC-2041
python3 specialists.py metrics INC-2042
```

Each prints the specialist's structured finding (summary, evidence, suspected cause, severity) as JSON.

### Simulate a specialist failure

`metric_window` can be forced to raise, to exercise the failure/resume path:

```bash
TRIAGE_FAIL=metrics python3 app.py start --thread fail-test --alert INC-2041
```

Use `display_failure`'s output (saved findings, nodes waiting to re-run) to confirm which branches completed before the crash, then resume normally:

```bash
python3 app.py resume --thread fail-test
```

### View the status page

```bash
python3 status_page.py
```

Opens a server on port 8080 — open `http://localhost:8080` in your browser. The page lists every saved thread in the left nav and auto-refreshes every 5 seconds, so you can start/resume a run in one terminal and watch its state change in the browser.

### Full test suite (solution branch)

The `solution` branch has offline tests that don't require live Gemini calls:

```bash
git checkout solution
python3 -m pip install pytest
python3 -m pytest
```

Use it to check your own implementation against the finished build if you get stuck.

## Cost

Each run makes a few Gemini 3.7 Flash calls. The Gemini API free tier covers this comfortably for testing.

## Branches

- `main` — the starting point for the project build.
- `solution` — the finished build, with offline tests (`python3 -m pytest`).
