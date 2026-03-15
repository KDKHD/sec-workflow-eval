# sec-workflow-eval

End-to-end evaluation harness for Kibana Security workflow alert classification.
Indexes alert fixtures, triggers detection rules with a workflow action, polls
until the workflow classifies every alert, then reports TP/FP accuracy with a
confusion matrix.

## Prerequisites

- **bash** (4.x+)
- **curl**
- **python3** (3.8+, stdlib only — no pip install needed)
- A running **Kibana + Elasticsearch** instance (local dev or remote)

## Quick start

```bash
git clone <repo-url> && cd sec-workflow-eval

# 1. Add your fixtures (not included in the repo — get them from your team)
cp /path/to/your/alerts.json fixtures/alerts.json

# 2. Run the full eval (auto-detects local Kibana at localhost:5601)
./sec-workflow-eval
```

That's it. The scripts auto-detect Kibana at `localhost:5601` with the default
dev credentials (`elastic:changeme`). To point at a different instance:

```bash
# via env vars
export KIBANA_URL=https://my-kibana:5601
export KIBANA_AUTH=elastic:secret
./sec-workflow-eval
```

## What it does

1. **`sec-workflow-alerts`** — Deletes existing alerts, ensures the Insights
   detection rule exists, attaches a workflow action to all rules, indexes
   fixture data into `insights-alerts-*`, and triggers the rule.

2. **`sec-workflow-eval`** — Runs `sec-workflow-alerts`, then polls
   Elasticsearch until every alert has been classified by the workflow. Reports
   per-alert results, a confusion matrix, and overall accuracy.

### Accuracy contract

The fixture's `kibana.alert.risk_score` determines the expected classification:

| Risk score | Expected label   |
|------------|------------------|
| Even       | True Positive    |
| Odd        | False Positive   |

## Usage

### sec-workflow-eval

```
Usage:
  sec-workflow-eval [OPTIONS] [-- sec-workflow-alerts OPTIONS...]

Options:
  --timeout SECS     Polling timeout in seconds (default: 600)
  --poll SECS        Polling interval in seconds (default: 10)
  --skip-run         Skip running sec-workflow-alerts; evaluate current state
  --kibana-url URL   Override Kibana URL (or set KIBANA_URL env var)
  --auth USER:PASS   Override auth (or set KIBANA_AUTH env var)
  -h, --help         Show this help
```

### sec-workflow-alerts

```
Usage:
  sec-workflow-alerts [OPTIONS]

Options:
  -n, --events NUM        Number of source events (default: 20)
  -h, --hosts NUM         Number of hosts (default: 2)
  -u, --users NUM         Number of users (default: 2)
  --episodes IDS          Comma-separated episode IDs (default: noise1)
  --start-date DATE       Date math start (default: 1d)
  --end-date DATE         Date math end (default: now)
  --workflow-id ID        Workflow ID for rule action
  --lookback EXPR         Rule lookback expression (default: now-2d)
  --wait SECS             Wait for rule execution (default: 20)
  --fixtures FILE         Path to alerts JSON file (default: fixtures/alerts.json)
  --help                  Show this help
```

### Examples

```bash
# Run with more alerts
./sec-workflow-eval -- -n 50 --hosts 4

# Evaluate the current state without re-indexing
./sec-workflow-eval --skip-run

# Run just the alert indexing step
./sec-workflow-alerts -n 30 --fixtures /custom/path/alerts.json

# Shorter timeout, faster polling
./sec-workflow-eval --timeout 120 --poll 5
```

## Fixtures

The `fixtures/` directory is git-ignored. Place your `alerts.json` there.
See [`fixtures/README.md`](fixtures/README.md) for the expected format.
