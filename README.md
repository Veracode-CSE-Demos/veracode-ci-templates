
# Veracode Pipeline Scan Templates with Baseline Files Management

This repository provides reusable **GitHub Actions workflows** for integrating Veracode Pipeline Scans with **automated multi-artifact baseline tracking** and **mitigation context** support.

It’s designed for organizations that maintain multiple repositories but want a **centralized baseline store**, typically in a repo such as
`your-org/veracode-ci-templates`.

All consuming repositories pull and push their baselines there, organized by **GitHub repo name**, **branch**, and **artifact name**.

---

## Scan Modes Supported

### 1. **Full Baseline Scans**

When using the **baseline.json** file (generated directly from `results.json`), the delta scan compares the current results to previous full scan results and only fails if **new findings** are introduced.

This mode ignores existing vulnerabilities (technical debt) but still shows unmitigated ones.

---

### 2. **Mitigation-Aware Scans**

When using **baseline-mitigated-findings.json** (generated via the [Veracode Mitigation Script](https://github.com/tjarrettveracode/veracode-pipeline-mitigation)),
the delta scan ignores **already mitigated or accepted** findings within the Veracode platform, reporting only *new, unmitigated* issues.

---

## Repository Structure (for consuming projects)

Each repository only needs minimal workflow definitions:

```yaml
name: Veracode Scans

on:
  push:
  pull_request:
  workflow_dispatch:
  schedule:
    - cron: '0 2 * * 1'  # Every Monday at 2 AM UTC

jobs:
  baseline:
    if: github.event_name == 'workflow_dispatch' || github.event_name == 'schedule'
    uses: your-org/veracode-ci-templates/.github/workflows/veracode-baseline.yml@main
    secrets: inherit

  delta:
    if: github.event_name == 'push' || github.event_name == 'pull_request'
    uses: your-org/veracode-ci-templates/.github/workflows/veracode-delta.yml@main
    secrets: inherit

  delta-mitigated:
    if: github.event_name == 'push' || github.event_name == 'pull_request'
    uses: your-org/veracode-ci-templates/.github/workflows/veracode-delta-mitigated.yml@main
    secrets: inherit
```

---

## How It Works

### **veracode-baseline.yml**

* Uses the Veracode CLI `autopackage` command to collect all build artifacts (JARs, DLLs, etc.) into `verascan/`.
* Runs a **Pipeline Scan** on each artifact:

  * Saves the scan results as `baseline.json` (full findings)
  * Runs the mitigation overlay to produce `baseline-mitigated-findings.json` (mitigation-aware)
* Commits both files per artifact to the shared baseline repo.

**Output folder structure in the baseline repo:**

```
baselines/
└── my-org_project-api/
    └── main/
        ├── app.jar/
        │   ├── baseline.json
        │   └── baseline-mitigated-findings.json
        ├── helper.dll/
        │   ├── baseline.json
        │   └── baseline-mitigated-findings.json
```

---

### **veracode-delta.yml**

* Repackages the code (using the same autopackager).
* For each artifact, downloads its corresponding **baseline.json** from the baseline repo.
* Runs a **delta comparison** scan using:

  ```bash
  --baseline_file baseline.json
  ```
* Fails only if **new findings** appear compared to the stored baseline.

---

### **veracode-delta-mitigated.yml**

* Works the same as `veracode-delta.yml`, but instead reads:

  ```
  baseline-mitigated-findings.json
  ```
* Ignores any findings previously mitigated or accepted in Veracode.
* Only fails the pipeline for new, unmitigated findings.

---

## File Storage Layout

All baselines are pushed to the shared repository under this structure:

```
baselines/
└── <repo>_<org>/
    └── <branch>/
        └── <artifact>/
            ├── baseline.json
            └── baseline-mitigated-findings.json
```

### Example:

```
baselines/
└── my-org_webapp-api/
    └── main/
        ├── api.jar/
        │   ├── baseline.json
        │   └── baseline-mitigated-findings.json
        ├── ui.war/
        │   ├── baseline.json
        │   └── baseline-mitigated-findings.json
```

This structure supports multiple repositories, branches, and artifacts simultaneously, keeping baselines isolated per build target.

---

## Required GitHub Secrets

| Variable Name              | Required | Description                                                        |
| -------------------------- | -------- | ------------------------------------------------------------------ |
| `VERACODE_API_ID`          | ✅        | Veracode API ID                                                    |
| `VERACODE_API_KEY`         | ✅        | Veracode API Key                                                   |
| `VERACODE_APP_NAME`        | ✅        | Application name in Veracode (On the repos to be scanned)          |
| `CI_PUSH_TOKEN_VCT`        | ✅        | GitHub PAT with repo write access                                  |
| `VERACODE_BASELINE_REPO`   | ✅        | Shared baseline repository (e.g. `your-org/veracode-ci-baselines`) |
| `VERACODE_BASELINE_BRANCH` | ❌        | Optional; defaults to `main`                                       |

---

### Creating the CI Push Token

1. Go to **Settings → Developer Settings → Personal Access Tokens**
2. Click **Generate new token**
3. Choose **repo** scope (read/write)
4. Add it to your repository secrets as `CI_PUSH_TOKEN_VCT`

---

## Trigger Behavior

| Job                               | Trigger            | Behavior                                                                                                  |
| --------------------------------- | ------------------ | --------------------------------------------------------------------------------------------------------- |
| **Baseline Scan**                 | Manual / Scheduled | Packages all artifacts and commits both `baseline.json` + `baseline-mitigated-findings.json` per artifact |
| **Delta Scan**                    | Push / PR          | Runs delta against each artifact’s `baseline.json`                                                        |
| **Delta Scan (Mitigation-Aware)** | Push / PR          | Runs delta against each artifact’s `baseline-mitigated-findings.json`                                     |

Each baseline update uses `[skip ci]` to avoid infinite workflow loops.

---

## References

* [Veracode Pipeline Scan Docs](https://docs.veracode.com/r/r_pipeline_scan_commands)
* [Veracode Mitigation Helper Script](https://github.com/tjarrettveracode/veracode-pipeline-mitigation)
* [Veracode CLI Documentation](https://docs.veracode.com/r/CLI_Reference)

---
