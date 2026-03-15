# Fixtures

Place your alert fixtures file here as `alerts.json`.

This is a flat JSON array of Elastic Security alert documents. The risk score
determines expected classification:

- **Odd** `kibana.alert.risk_score` → expected **false positive**
- **Even** `kibana.alert.risk_score` → expected **true positive**

Example structure:

```json
[
  {
    "@timestamp": "2024-03-25T12:00:00.000Z",
    "kibana.alert.risk_score": 47,
    "kibana.alert.rule.name": "Suspicious Process",
    "message": "Endpoint Security alert",
    "event": {
      "kind": "alert",
      "module": "endpoint",
      "risk_score": 47,
      "severity": 47,
      "created": "2024-03-25T12:00:00.000Z",
      "ingested": "2024-03-25T12:00:01.000Z"
    },
    "process": {
      "name": "curl",
      "command_line": "curl http://example.com"
    },
    "host": { "name": "host-01", "hostname": "host-01", "id": "abc123" },
    "user": { "name": "alice" },
    "agent": { "id": "agent-001" }
  }
]
```

This file is git-ignored. Obtain the real fixtures from your team.
