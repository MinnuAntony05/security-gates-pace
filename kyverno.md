### Kyverno — Design, Customization, and Architecture

---

### What is Kyverno?

Kyverno is a **Kubernetes-native policy engine**. It sits as a webhook inside the cluster — every pod creation/update request passes through it before Kubernetes admits it. Unlike the OPA/Rego CI gate (which is static analysis), Kyverno is **live runtime enforcement** on the actual cluster.

```
Developer / Jenkins deploys a pod
          ↓
API Server receives the request
          ↓
Kyverno admission webhook intercepts it
          ↓
  Checks: is this image cosign-signed?
          ↓
  Signed   → admitted, pass in PolicyReport
  Unsigned → Audit: admitted anyway, FAIL in PolicyReport
           → Enforce (future): BLOCKED, deployment fails
```

---

### How it's designed in PACE

**Two files define the entire setup:**

| File | Purpose |
|---|---|
| kyverno-install-values.yaml | How Kyverno itself is installed (Helm values) |
| cluster-policy-cosign-dev.yaml | What policy Kyverno enforces |

---

### What is Customized

#### 1. Installation (kyverno-install-values.yaml)

| Setting | Default | PACE value | Why |
|---|---|---|---|
| `reportsController.memory limit` | 128Mi | **1Gi** | OOMKilled at 128Mi and 512Mi — cluster has 2158 pods across 194 namespaces |
| `webhookTimeout` | 10s | **15s** | Cosign verification hits public Sigstore/Rekor — needs network headroom |
| `webhookConfiguration.failurePolicy` | Fail | **Ignore** | Safe for initial rollout — if Kyverno crashes, pods still deploy |
| `cleanupJobs image` | `bitnami/kubectl:1.28.5` | **`registry.k8s.io/kubectl:v1.33.8`** | Bitnami dropped versioned tags entirely from Docker Hub — was causing ImagePullBackOff |
| `cleanupJobs securityContext.runAsUser` | (none) | **65532** | Chart enforces `runAsNonRoot: true` but the kubectl image runs as root (uid 0) — this was causing `CreateContainerConfigError` on all 7 cleanup jobs |
| `policyExceptions.enabled` | false | **true, namespace: `"*"`** | Lets teams create namespace-scoped opt-outs for vendor images that can't be signed |
| `replicas` | varies | **1** | Single-node consideration — scale to 3 for HA production |

#### 2. Policy (cluster-policy-cosign-dev.yaml)

| Setting | What it does | Why customized this way |
|---|---|---|
| `validationFailureAction: Audit` | Violations logged, nothing blocked | Safe rollout — cosign signing not yet on main branch |
| `background: false` | Only checks newly admitted pods | Avoids 2158 noisy violations from existing pods that predate signing |
| `namespaces: [dev]` | Only `dev` namespace affected | 193 other namespaces untouched — scoped rollout |
| `imageReferences: ["registry.ustpace.com/docker/*"]` | Only PACE-built images checked | Everything else (ECR, Docker Hub, Istio sidecars, nginx) silently passes |
| `imageRegistryCredentials.secrets: [artifactory]` | Kyverno authenticates to private registry | Without this → `UNAUTHORIZED` error when Kyverno tries to fetch the image manifest to verify the signature. The `artifactory` secret was copied from `dev` namespace to `kyverno` namespace |
| `mutateDigest: false` | Don't rewrite image tags to digests | Not allowed in Audit mode — would require Enforce |
| Single rule only (no Rule 2) | One rule covers containers + initContainers | Kyverno v1.12+ handles both automatically. A separate initContainers rule caused autogen JMESPath errors on Deployment resources |

---

### What is Standardized (Kyverno defaults kept as-is)

- **4 core controllers** — admissionsController, backgroundController, cleanupController, reportsController (standard Kyverno 3.2.x architecture)
- **PolicyReport format** — standard `wgpolicyk8s.io/v1alpha2` CRD written by Kyverno
- **Sigstore/Fulcio/Rekor** — public keyless signing infrastructure, no key management needed
- **Helm chart** — official `kyverno/kyverno` chart version `3.2.6`

---

### How Cosign Verification Works at Runtime

```
Pod admission request arrives
          ↓
Kyverno sees image: registry.ustpace.com/docker/stage-craft-ai:25479214998
          ↓
Authenticates to registry.ustpace.com using `artifactory` secret
          ↓
Fetches the image manifest (to get digest)
          ↓
Looks in the registry for a cosign signature OCI artifact
  (stored alongside the image, tag: sha256-<digest>.sig)
          ↓
Contacts Rekor (public transparency log) to verify the signature
          ↓
Checks the Fulcio certificate embedded in the signature:
  - Issuer = https://token.actions.githubusercontent.com  ✓
  - Subject = .../reusable_ci_template_python_3_12.yaml@*  ✓
          ↓
Match → PASS (recorded in PolicyReport)
No signature found → FAIL (recorded in PolicyReport, pod still admitted in Audit)
```

The identity check (issuer + subject) means **only images signed by the PACE shared CI workflow** pass. An image signed by a different workflow, a different org, or manually signed locally will fail — even if the signature itself is valid.

---

### Current State vs Target State

| | Now | Target (after cosign on main) |
|---|---|---|
| Mode | Audit | Enforce |
| failurePolicy | Ignore | Fail |
| Replicas | 1 | 3 (HA) |
| Result | Violations logged, pods admitted | Unsigned images blocked at admission |
