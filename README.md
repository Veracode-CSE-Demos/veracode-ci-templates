# Veracode Pipeline Scan workflows with Baseline Management

This repository provides reusable **GitHub Actions workflows** for integrating Veracode Pipeline Scans with automated baseline tracking and adding mitigation context. It is designed to be used across multiple repositories without duplicating CI logic, using a centralized baseline repository (e.g., `your-org/veracode-ci-baselines`) as a storage location for scan baselines and Pipeline scan templates.

By default, all consuming repositories push and pull baseline files to this shared repository, organized by GitHub repository name and branch.

---

## Scan Modes Supported

### Fail Only on New Findings

When using `results.json` with the `--baseline_file` flag, the scan compares current results to a previous version and only fails the pipeline if new vulnerabilities are introduced. This helps enforce security gating while ignoring existing technical debt.

### Apply Veracode Mitigation Context

When supplying a generated `baseline.json` (produced via Veracode’s mitigation API), the scan gains context about flaws that have been triaged, mitigated, or accepted in the Veracode application profile.

---

## Repository Structure

Each consuming project includes a minimal `.github/workflows/` setup:

```yaml
**name: Veracode Scans

on:
  push:
  pull_request:
  workflow_dispatch:
  schedule:
    - cron: '0 2 * * 1'  # Every Monday at 2 AM UTC

jobs:
  baseline:
    # Only run baseline on manual trigger or scheduled runs
    if: github.event_name == 'workflow_dispatch' || github.event_name == 'schedule'
    uses: your-org/veracode-ci-templates/.github/workflows/baseline.yml@main
    secrets: inherit

  delta:
    # Run delta scan on any repo change: push, pull_request, etc.
    if: github.event_name == 'push' || github.event_name == 'pull_request'
    uses: your-org/veracode-ci-templates/.github/workflows/delta.yml@main
    secrets: inherit**

```

---

## How It Works

### veracode-baseline.yml

* Compiles the project and runs a full Veracode pipeline scan
* Generates `results.json` and `baseline.json` (if applicable)
* Commits both files to the shared baseline repo, under a path based on GitHub repo and branch

### veracode-delta.yml

* Downloads the previously committed `baseline.json` from the shared repo
* Performs a delta scan, comparing the new scan results against the stored baseline
* Fails the build only if new, unmitigated findings are introduced

---

## File Storage Layout

All baseline and results files are committed to the shared repo (e.g., `your-org/veracode-ci-baselines`) using this structure:

```
baselines/
└── <org_repo>/
    └── <branch>/
        ├── baseline.json     # Mitigation-aware baseline
        └── results.json      # Latest pipeline scan result
```

### Example:

```
baselines/
└── my-org_project-api/
    └── main/
        ├── baseline.json
        └── results.json
```

This allows multiple branches across multiple repos to track isolated baselines safely.

---

## Required GitHub Secrets

To use these templates, the following GitHub repository secrets must be configured:

| Variable Name              | Required | Description                                                            |
| -------------------------- | -------- | ---------------------------------------------------------------------- |
| `VERACODE_API_ID`          | Yes      | Veracode API ID tied to a scanning account                             |
| `VERACODE_API_KEY`         | Yes      | Veracode API Key paired with the above                                 |
| `VERACODE_APP_NAME`        | Yes      | Application name as defined in your Veracode platform                  |
| `CI_PUSH_TOKEN_VCT`        | Yes      | GitHub personal access token with **repo** scope (to push to baseline) |
| `VERACODE_BASELINE_REPO`   | No       | Defaults to `your-org/veracode-ci-baselines`                           |
| `VERACODE_BASELINE_BRANCH` | No       | Defaults to `main`                                                     |

---

### Creating the CI Push Token

1. Go to your GitHub account → **Settings → Developer Settings → Personal Access Tokens**
2. Click **"Generate new token (classic)"** or fine-grained PAT
3. Name: `CI_PUSH_TOKEN_VCT`
4. Scope: `repo` (just this is sufficient)
5. Save the token and add it to your secret\`s under the name `CI_PUSH_TOKEN_VCT`

Ensure the PAT belongs to a user or bot account that has push access to the shared baseline repo.

---

## Trigger Behavior

| Job Type      | Trigger Type       | Behavior                                                          |
| ------------- | ------------------ | ----------------------------------------------------------------- |
| Baseline Scan | Manual / Scheduled | Creates `results.json` and `baseline.json`, pushes to shared repo |
| Delta Scan    | Push / PR          | Fetches baseline.json and runs comparison scan for new findings   |

Baseline commits use `[skip ci]` to prevent triggering new pipelines when committing baseline files.

---

## References

* [Veracode Pipeline Scan Documentation](https://docs.veracode.com/r/r_pipeline_scan_commands)
* [Veracode Mitigation Script](https://github.com/tjarrettveracode/veracode-pipeline-mitigation)
