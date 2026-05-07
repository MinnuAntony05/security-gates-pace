## How the K8s Policy Validation Gate Works

### The Big Picture

```
CI pipeline checks out the edgeops-modules Helm chart repo
            ↓
helm template renders it to a plain YAML manifest
  (using your service's values-<name>.yaml)
            ↓
Conftest runs every Rego rule against that manifest
            ↓
Violations → pipeline fails / Warnings → reported only
            ↓
Artifacts: rendered-manifest.yaml + conftest-report.json
```

The key insight: **no Kubernetes cluster is involved**. This is pure static analysis — it simulates what Kubernetes would deploy and checks it for security issues before a single image is pushed.

---

### Tools Used

| Tool | What it is | Role here |
|---|---|---|
| **Helm** | Kubernetes package manager | Renders the chart template + values into real YAML |
| **Conftest** | Policy testing CLI by OPA | Runs the Rego policies against the rendered YAML |
| **OPA/Rego** | Open Policy Agent language | The policy rules themselves |
| **yq** | YAML processor | Parsing utility |

---

### What is Customized (PACE-specific)

1. **The Helm chart is edgeops-modules** — the central PACE chart that all services deploy through. It's checked out privately using `K8S_POLICY_VALIDATION_PAT`.

2. **Values file is per-service** — `values-stage-craft-ai.yaml`, `values-pace-git-lift.yaml`, etc. The rendered manifest reflects *your exact deployment config*, not generic defaults.

3. **Policy file is PACE-authored** (k8s_security.rego) — written specifically for PACE's Kubernetes setup. It handles all workload types (Deployment, StatefulSet, DaemonSet, Job, CronJob, Pod) and uses a normalization layer to extract pod specs regardless of resource kind.

4. **`fail_on_warn: false`** — warnings are reported but don't break the build. Currently `continue-on-error: true` also on the step itself.

---

### What is Standardized (Industry baseline)

The policy is written against three established standards:
- **NIST SP 800-190** — Application Container Security Guide
- **Kubernetes Pod Security Standards (PSS)** — Restricted profile
- **CIS Kubernetes Benchmark v1.8**

---

### All Checks — Complete List

**DENY rules (D1–D15) — block the pipeline:**

| Code | Check | What it catches |
|---|---|---|
| D1 | `runAsNonRoot: true` | Container running as root UID |
| D2 | `allowPrivilegeEscalation: false` | Container can escalate to root via setuid |
| D3 | `readOnlyRootFilesystem: true` | Container can write to root filesystem |
| D4 | No `privileged: true` | Container has full host access |
| D5 | No `:latest` tag, must have tag | Mutable/untracked image references |
| D6 | `resources` block must exist | No CPU/memory constraints at all |
| D7 | `resources.requests.cpu` required | No CPU reservation |
| D8 | `resources.requests.memory` required | No memory reservation |
| D9 | `resources.limits.cpu` required | CPU can starve other pods |
| D10 | `resources.limits.memory` required | Memory leak can kill the node |
| D11 | No dangerous capabilities added | `SYS_ADMIN`, `NET_ADMIN`, `SYS_PTRACE`, etc. |
| D12 | `capabilities.drop: [ALL]` required | Keeps all Linux capabilities by default |
| D13 | `hostNetwork: false` | Container sees host network interfaces |
| D14 | `hostPID: false` | Container sees all host processes |
| D15 | `hostIPC: false` | Container can read shared host memory |

**WARN rules (W1–W7) — reported, configurable to block:**

| Code | Check |
|---|---|
| W1 | Container has no `securityContext` at all |
| W2 | Workload deployed to `default` namespace |
| W3 | Missing `livenessProbe` |
| W4 | Missing `readinessProbe` |
| W5 | Missing pod-level `securityContext` |
| W6 | `fsGroup: 0` (root group) |
| W7 | `runAsUser: 0` (root UID explicitly set) |

---

### Output and Reliability

**What's produced:**

| File | Contents |
|---|---|
| `rendered-manifest.yaml` | The actual YAML that would be applied to Kubernetes — full transparency |
| `conftest-report.json` | Machine-readable results — violation messages, rule codes, pass/fail per resource |
| GitHub Step Summary | Human-readable table of violations count |

**Reliability:**

- ✅ **High reliability for what it checks** — the rules are deterministic. If D1 fires, `runAsNonRoot` is genuinely missing.
- ⚠️ **Limited to what Helm renders** — if a value is templated at runtime (e.g., injected by a Kubernetes Admission Controller like Istio), it won't appear in the static render.
- ⚠️ **Does not check runtime behavior** — a container that drops all capabilities and runs as non-root is still compliant even if the app inside escalates privileges via a vulnerability.
- ⚠️ **`continue-on-error: true` currently** — violations are found and reported but the build doesn't actually fail yet. This needs to be flipped before it becomes a hard gate.
