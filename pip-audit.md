## How the pip-audit Gate Works — PACE CI

### What it does
Scans all Python dependencies for known CVEs using `pip-audit` (PyPA). Every vulnerability is looked up against the full OSV advisory record, enriched with a CVSS severity score, and reported in a structured YAML report + GitHub Step Summary. A configurable threshold decides whether the build fails.

### Execution Flow

```
Set up Python runtime
      ↓
Prepare dependency list
  poetry.lock → poetry export → pip-audit-input.txt
  requirements.txt → copied directly
  pyproject.toml → parsed via tomllib → pip-audit-input.txt
      ↓
Run pip-audit → pip-audit-results.json
  (OSV or PyPI database, no install needed — name+version only)
      ↓
Severity enrichment per vuln ID (the key extra step)
      ↓
Categorise + gate decision
      ↓
Write pip-audit-report.yaml + GitHub Step Summary
      ↓
Upload artifacts (YAML report + raw JSON + input list)
```

### The Severity Enrichment Step

This is the important extra step. Raw pip-audit output **does not include severity** — it only returns the vulnerability ID and description. Without enrichment, every finding is `UNKNOWN` and the gate can't make meaningful decisions.

For each unique vuln ID, the action calls the OSV API directly:
```
GET https://api.osv.dev/v1/vulns/{id}
```

This fetches the **full advisory record**, which contains richer data than what pip-audit returns from its batch lookup. Severity is extracted in this lookup order:

| Priority | Source | Field | Notes |
|---|---|---|---|
| 1 | GitHub Advisory | `database_specific.severity` | String (`HIGH`, `MODERATE`, `LOW`) — available immediately on advisory publish |
| 2 | GitHub Advisory | `database_specific.cvss.score` | Numeric CVSS if GitHub computed one |
| 3 | NVD | `severity[].score` (CVSS vector) | Parsed with the `cvss` library to get numeric base score |

If none of the above are present → severity stays `UNKNOWN`. `UNKNOWN` never fails the gate regardless of threshold.

**Why not use pip-audit's built-in querybatch?** pip-audit's batch endpoint returns stub records that omit `database_specific` and `severity[]` CVSS fields. This means recent GHSA advisories (not yet processed by NVD) always come back as `UNKNOWN`. The direct `GET /v1/vulns/{id}` endpoint returns the full record and solves this.

### Severity Classification (NVD CVSS v3)

| Severity | CVSS Score Range | Gate behaviour (default) |
|---|---|---|
| 🔴 CRITICAL | ≥ 9.0 | ✅ Triggers gate |
| 🟠 HIGH | 7.0 – 8.9 | ✅ Triggers gate |
| 🟡 MEDIUM | 4.0 – 6.9 | — Reported only |
| 🟢 LOW | 0.1 – 3.9 | — Reported only |
| ⚪ UNKNOWN | N/A | — Never triggers gate |

Default threshold is `high` — only CRITICAL + HIGH fail the build.

### Gate Configuration

```yaml
min_severity: "high"      # critical | high | medium | low
fail_on_vuln: "true"      # set false for report-only mode
vulnerability_service: "osv"   # osv (recommended) | pypi
ignore_vulns: "GHSA-xxxx PYSEC-yyyy"   # space-separated IDs to suppress
```

### Outputs

| Output | Description |
|---|---|
| `vuln_count` | Total across all severities |
| `critical_count` / `high_count` / ... | Per-severity counts |
| `result` | `pass` or `fail` |

### Artifacts uploaded (30-day retention)

| File | Contents |
|---|---|
| `pip-audit-report.yaml` | Structured YAML — scan metadata, summary, full per-vuln detail with CVSS scores and fix versions |
| `pip-audit-results.json` | Raw pip-audit JSON output |
| `pip-audit-input.txt` | Pinned dependency list that was scanned |

---
