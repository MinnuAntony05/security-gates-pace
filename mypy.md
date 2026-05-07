## How the mypy Gate Works — PACE CI

### What it does
Runs `mypy` static type analysis on Python source code. Every error mypy finds is classified into a **PACE-defined risk tier** and reported as an inline annotation in GitHub. A configurable gate threshold decides whether the build fails.

### Risk Tier Classification

These tiers are **PACE-defined** based on estimated correctness impact. They are not official mypy severity scores.

| Tier | Icon | mypy codes | What it means |
|---|---|---|---|
| CRITICAL | 🔴 | `unused-coroutine`, `name-defined` | Likely runtime-impacting — name doesn't exist, coroutine never awaited |
| HIGH | 🟠 | `override`, `return-value`, `call-arg`, `call-overload`, `attr-defined`, `no-redef` | Associated with correctness/runtime risks — wrong return type, bad attribute access, wrong arguments |
| MEDIUM | 🟡 | `assignment`, `arg-type`, `index`, `operator`, `union-attr`, `misc`, `valid-type`, `type-arg` | Probable type bugs requiring dev review. Unknown codes also land here. |
| LOW | 🔵 | `var-annotated`, `annotation-unchecked` | Missing annotations — no runtime impact, tech debt only |
| INFO | ⚪ | `import-untyped` | Missing third-party stubs — fix with `pip install types-*` |

### Gate Behaviour

Default: gate triggers on **CRITICAL + HIGH**. MEDIUM/LOW/INFO are always reported but never fail the build.

```yaml
fail_on_risk_tier: "high"   # default — change to medium/low/info to tighten
```

Strict mode — fail on any error regardless of tier:
```yaml
strict_mode: "true"
```

### Custom Overrides

Teams can reclassify individual codes without touching the shared library:
```yaml
custom_risk_map: |
  attr-defined=critical
  assignment=low
```

### Backward Compatibility

`fail_on_severity` (old input name) still works as a deprecated alias for `fail_on_risk_tier`. Both inputs are accepted — `fail_on_risk_tier` takes precedence.

### Execution Flow

```
Install mypy
   ↓
Install project deps (poetry.lock or requirements.txt)
   ↓
Detect source dir + config (mypy.ini / pyproject.toml / PACE defaults)
   ↓
Run mypy → mypy-raw.txt
   ↓
Classify each error → risk tier → GitHub annotation
   ↓
Build Step Summary table
   ↓
Evaluate gate threshold → pass / fail
   ↓
Upload mypy-raw.txt as artifact (30-day retention)
```

### Where outputs go

- **Inline PR annotations** — red/yellow/blue underlines on the exact line in the diff
- **GitHub Step Summary** — table with tier counts and per-error breakdown (capped at 50 rows per tier, rest in artifact)
- **Build artifact** — `mypy-raw.txt` full raw output, retained for 30 days

--- 
