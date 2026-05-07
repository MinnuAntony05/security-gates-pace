### What is SBOM?

**Software Bill of Materials** — borrowed from manufacturing. A car manufacturer lists every bolt, chip, and component that went into building a car. An SBOM does the same for software: it lists every package, library, and OS component inside a container image.

---

### What is Syft?

Syft is a CLI tool by **Anchore** that scans a container image and produces an SBOM. It does this without running the container — it inspects the image layers directly.

```
Docker image
  └── Layer 1 (base Alpine)
  └── Layer 2 (Python install)
  └── Layer 3 (your app)
         ↓
       Syft
         ↓
  sbom.spdx.json (list of all 345 packages)
```

Syft knows how to find packages installed by:
- `apk` (Alpine Linux)
- `apt/dpkg` (Debian/Ubuntu)
- `pip` (Python)
- `npm` (Node.js)
- `gem` (Ruby)
- Go modules, Java JARs, etc.

It reads the package metadata files left behind during installation (e.g., `/usr/lib/python3.12/site-packages/*.dist-info`) and records what was installed.

---

### How SBOM generation works step by step

```
1. CI builds Docker image  (push: false, load: true)
          ↓
2. Syft pulls the image from local Docker daemon
          ↓
3. Syft inspects every filesystem layer
   - Reads /lib/apk/db/installed      → OS packages
   - Reads site-packages/*.dist-info  → Python packages
   - Reads package-lock.json          → Node packages
          ↓
4. For each package found, records:
   - name, version
   - license
   - purl (package URL — universal CVE lookup identifier)
   - supplier
          ↓
5. Writes sbom.spdx.json  (machine format)
       + sbom-report.csv  (spreadsheet)
       + sbom-report.txt  (plain text)
          ↓
6. Uploads as artifact (30 days)
7. Submits to GitHub Dependency Graph
```

---

### Why is SBOM necessary?

**Problem without SBOM:**

A critical CVE drops today — say `CVE-2026-XXXX` affects `aiohttp < 4.0`. You have 20 microservices deployed. Which ones are affected? You have no idea without checking every repo, every Dockerfile, every `requirements.txt`. That takes hours.

**With SBOM:**

```bash
grep "aiohttp" sbom-stage-craft-ai-*.json  → affected: yes, version 3.13.5
grep "aiohttp" sbom-pace-git-lift-*.json   → affected: no
```

Answer in seconds. Across every image ever built.

---

### Three concrete reasons for your customer

**1. Visibility into the base image**
Your developers never explicitly installed `busybox`, `curl`, or `libssl` — they came with the Alpine base image. Without SBOM, those 109 OS packages are invisible. With SBOM, they're tracked and can be scanned for CVEs any time.

**2. Compliance requirements**
- US Executive Order 14028 (2021) mandates SBOM for software sold to the US federal government
- EU Cyber Resilience Act (2024) requires SBOM for CE-marked products
- NIST SSDF, SLSA Level 2+ — both reference SBOM as required evidence

**3. Retrospective vulnerability scanning**
Trivy/Grype scan images at build time. But new CVEs are disclosed every day. The SBOM stored as an artifact lets you scan **past builds** against **today's CVE database** — without rebuilding:

```bash
grype sbom:sbom-stage-craft-ai-25479214998.spdx.json
# → finds CVEs disclosed after that image was built
```

This is the capability you cannot get from Trivy alone at build time.
