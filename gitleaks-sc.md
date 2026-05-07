### How Gitleaks Works

**Step 1 — Scan mode**
On `workflow_dispatch` (which this run was), gitleaks runs in `detect` mode scanning the **full git history**:
```
git log -p -U0 --full-history --all
```
This means it reads every diff of every commit ever pushed — not just the current files. That's why you see findings from commits dating back to 2025.

**Step 2 — Pattern matching**
For each line in the git diff output, it runs every rule defined in the config against the line. Each rule is a regex. If the regex matches → candidate finding.

**Step 3 — Entropy filter**
After a regex match, gitleaks optionally computes the Shannon entropy of the matched value. High entropy (random-looking strings like `AKIA...`, private keys) = more likely a real secret. Low entropy = probably a placeholder or username.

The entropy threshold varies per rule. This is why `curl-auth-user` still fires even at entropy 2.55 — that rule has a very low (or no) entropy threshold.

---

### Why `curl -u` Gets Flagged

The `curl-auth-user` rule matches the pattern of `-u user:password` in curl commands. The rule regex looks roughly like:
```
curl.*-u\s+\S+:\S+
```

Gitleaks doesn't understand context — it just sees `-u something:something` and treats `something:something` as a credential. It doesn't know:
- whether the target is `localhost` (dev-only)
- whether `admin:admin` is a real password or a default
- whether the file is a test script vs production config

**All 15+ `curl-auth-user` hits** come from `scripts/test-pipeline-ingestion.sh` and `data/example_*/vars/*.groovy` — all hitting `localhost:8080` (Jenkins) and `localhost:15672` (RabbitMQ). These are dev environment defaults.

---

### How a Secret Is "Classified"

```
Regex match
    ↓
Extract matched value
    ↓
Calculate Shannon entropy of the value
    ↓
Compare entropy against rule threshold
    ↓
If above threshold → Finding (with RuleID from the matched rule)
    ↓
No severity tiers — gitleaks has no CRITICAL/HIGH/LOW
Every finding is equally a "leak"
```

**Key point**: gitleaks has **no severity classification**. Every finding is treated identically — it either leaks or doesn't. The "severity" concept you see in pip-audit/mypy doesn't exist here. This is why a `curl -u admin:admin localhost:8080` gets the same treatment as a GCP private key.

---

### Summary of What Actually Happened

| Rule | How it triggers | Your case |
|---|---|---|
| `generic-api-key` | Variable name contains `key/token/secret` + regex on value | `jwt_token=`, `LANGFUSE_SECRET_KEY=`, JDBC password |
| `curl-auth-user` | `-u <value>` in a curl command | All the `localhost` scripts |
| `private-key` | PEM header pattern + high entropy | The `.credentials/` JSON file |
| `aws-access-token` | Variable named `secret_token` matched AWS-like pattern | Test file in `tests/db/` |

The `private-key` finding (entropy 6.02) is the only one where both pattern AND entropy strongly indicate a real secret. Everything else fired on pattern alone with low entropy, which is the classic false positive signature.
