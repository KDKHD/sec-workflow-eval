# sec-workflow-eval

End-to-end evaluation harness for Kibana Security workflow alert classification.

This repo contains two scripts:

- **`sec-workflow-alerts`** — Sets up and indexes alert data into Kibana/ES.
- **`sec-workflow-eval`** — Runs `sec-workflow-alerts`, then polls until the
  workflow classifies every alert and reports TP/FP accuracy.

You can run them independently or together.

## Prerequisites

- **bash** (4.x+)
- **curl**
- **python3** (3.8+, stdlib only — no pip install needed)
- A running **Kibana + Elasticsearch** instance (local dev or cloud)

## Quick start

```bash
git clone <repo-url> && cd sec-workflow-eval

# Add your fixtures (not included in the repo — get them from your team)
cp /path/to/your/alerts.json fixtures/alerts.json

# Run the full eval (auto-detects local Kibana at localhost:5601)
./sec-workflow-eval
```

---

## Connecting to Kibana & Elasticsearch

Both scripts share three environment variables for connection:

| Variable      | Purpose                        | Default (local dev)              |
|---------------|--------------------------------|----------------------------------|
| `KIBANA_URL`  | Kibana base URL                | Auto-detected at `localhost:5601`|
| `KIBANA_AUTH` | Credentials as `user:password` | `elastic:changeme`               |
| `ES_URL`      | Elasticsearch base URL         | Derived from `KIBANA_URL` (port 5601 → 9200) |

### Local dev

No configuration needed — Kibana is auto-detected at `localhost:5601` or
`localhost:5601` (HTTPS) with `elastic:changeme`.

### Remote / Elastic Cloud

On cloud deployments, Kibana and Elasticsearch have **different hostnames**, so
you must set `ES_URL` explicitly (the automatic port-swap doesn't apply):

```bash
export KIBANA_URL=https://my-deployment.kb.us-west2.gcp.elastic-cloud.com
export ES_URL=https://my-deployment.es.us-west2.gcp.elastic-cloud.com
export KIBANA_AUTH=elastic:my-cloud-password
```

Then run either script as normal.

---

## `sec-workflow-alerts`

Standalone script that prepares alert data. No Kibana repo required.

**What it does:**

1. Deletes existing security alerts
2. Ensures the Insights detection rule exists
3. Attaches a workflow action to all rules
4. Indexes fixture data into `insights-alerts-*` (50% noise / 50% attack)
5. Updates the rule lookback window and triggers it

### Flags

```
sec-workflow-alerts [OPTIONS]

  -n, --events NUM        Number of source events (default: 20)
  --hosts NUM             Number of hosts (default: 2)
  -u, --users NUM         Number of users (default: 2)
  --episodes IDS          Comma-separated episode IDs (default: noise1)
  --start-date DATE       Date math start (default: 1d)
  --end-date DATE         Date math end (default: now)
  --workflow-id ID        Workflow ID for rule action
  --lookback EXPR         Rule lookback expression (default: now-2d)
  --wait SECS             Wait for rule execution (default: 20)
  --fixtures FILE         Path to alerts JSON file (default: fixtures/alerts.json)
  --es-url URL            Override Elasticsearch URL (or set ES_URL env var)
  --help                  Show this help
```

### Examples

```bash
# Basic run against local Kibana
./sec-workflow-alerts

# Index 50 alerts across 4 hosts
./sec-workflow-alerts -n 50 --hosts 4

# Use a custom fixtures file
./sec-workflow-alerts --fixtures /path/to/my-alerts.json

# Point at a cloud instance
export KIBANA_URL=https://my-deployment.kb.us-west2.gcp.elastic-cloud.com
export ES_URL=https://my-deployment.es.us-west2.gcp.elastic-cloud.com
export KIBANA_AUTH=elastic:my-cloud-password
./sec-workflow-alerts -n 30
```

---

## `sec-workflow-eval`

Runs the full evaluation: calls `sec-workflow-alerts` internally, then polls
Elasticsearch until every alert is classified by the workflow. Reports per-alert
results, a confusion matrix, and overall accuracy.

### Flags

```
sec-workflow-eval [OPTIONS] [-- sec-workflow-alerts OPTIONS...]

  --timeout SECS     Polling timeout in seconds (default: 600)
  --poll SECS        Polling interval in seconds (default: 10)
  --skip-run         Skip sec-workflow-alerts; evaluate existing alerts
  --kibana-url URL   Override Kibana URL (or set KIBANA_URL env var)
  --es-url URL       Override Elasticsearch URL (or set ES_URL env var)
  --auth USER:PASS   Override auth (or set KIBANA_AUTH env var)
  -h, --help         Show this help
```

### Examples

```bash
# Basic run — indexes alerts, waits for classification, reports accuracy
./sec-workflow-eval

# Evaluate existing alerts without re-indexing
./sec-workflow-eval --skip-run

# Shorter timeout and faster polling
./sec-workflow-eval --timeout 120 --poll 5

# Cloud instance via flags
./sec-workflow-eval \
  --kibana-url https://my-deployment.kb.us-west2.gcp.elastic-cloud.com \
  --es-url https://my-deployment.es.us-west2.gcp.elastic-cloud.com \
  --auth elastic:my-cloud-password
```

### Passing flags through to `sec-workflow-alerts`

Since `sec-workflow-eval` calls `sec-workflow-alerts` internally, you can
forward flags to it using `--` as a separator. Everything after `--` is passed
directly to `sec-workflow-alerts`:

```bash
# Forward -n and --hosts to sec-workflow-alerts
./sec-workflow-eval -- -n 50 --hosts 4

# Forward a custom fixtures path
./sec-workflow-eval -- --fixtures /path/to/my-alerts.json

# Combine eval flags with alert flags
./sec-workflow-eval --timeout 300 --poll 5 -- -n 100 --hosts 8 --users 4
```

---

## Accuracy contract

The fixture's `kibana.alert.risk_score` determines the expected classification:

| Risk score | Expected label |
|------------|----------------|
| Even       | True Positive  |
| Odd        | False Positive |

## Fixtures

The `fixtures/` directory is git-ignored. Place your `alerts.json` there.
See [`fixtures/README.md`](fixtures/README.md) for the expected format.
