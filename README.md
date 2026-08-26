# Veracode CI Templates

Reusable GitHub Actions workflow for Veracode Pipeline Scan with centralised, branch based baseline management.

The default branch of each consuming repository is the security baseline. Every push to it refreshes the stored baselines. Every other branch and pull request is scanned as a delta against those baselines, so a build only fails on findings the change actually introduced.

## The model

| Event | Mode | What happens |
|---|---|---|
| Push to the baseline branch | `baseline` | Repackage, rescan, overwrite the stored baselines for that repo and branch |
| Pull request | `delta` | Scan against the stored baselines and gate on new findings |
| Push to any other branch | `delta` | Same as above |
| `workflow_dispatch` or `schedule` on the baseline branch | `baseline` | On demand or weekly baseline refresh |

The baseline branch is the repository default branch unless the caller sets `baseline_branch`. Baselines are keyed by the **baseline branch**, never by the branch that is running. That is the single most important property here: a feature branch and a pull request both compare against the same stored state.

Storage layout in the baseline repository:

```
baselines/
└── <owner>_<repo>/
    └── <baseline-branch>/
        └── <artifact-file-name>/
            ├── baseline.json                     # full scan results
            └── baseline-mitigated-findings.json  # platform mitigated findings only
```

## Quick start

Copy [`examples/veracode.yml`](examples/veracode.yml) into the consuming repository as `.github/workflows/veracode.yml`, adjust the push branch filter to your default branch, and add the secrets below. The minimum caller is:

```yaml
name: Veracode

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

permissions:
  contents: read

jobs:
  veracode:
    uses: Veracode-CSE-Demos/veracode-ci-templates/.github/workflows/veracode-pipeline.yml@main
    secrets: inherit
```

First run on a new repository: run the workflow manually from the default branch, or merge to it once. Until baselines exist, delta scans report `No baseline` and pass, unless `strict_baselines` is true.

## Inputs

| Input | Default | Purpose |
|---|---|---|
| `mode` | `auto` | `auto`, `baseline` or `delta`. `auto` refreshes on the baseline branch and delta scans everywhere else |
| `baseline_branch` | repo default branch | Branch that owns the baselines |
| `baseline_repo` | `Veracode-CSE-Demos/veracode-ci-templates` | Repository that stores baseline files |
| `baseline_store_branch` | default branch of the store | Branch inside the store where baselines are committed |
| `baseline_type` | `full` | `full`, `mitigated` or `both`. See the note below |
| `templates_repo` | `Veracode-CSE-Demos/veracode-ci-templates` | Repository holding this workflow and its helper scripts |
| `fail_on_severity` | `Very High, High` | Severities that fail a delta scan |
| `fail_on_cwe` | empty | CWEs that fail a delta scan regardless of severity |
| `policy_name` | empty | Veracode policy used to rate findings |
| `strict_baselines` | `false` | Fail when an artifact has no stored baseline |
| `artifacts_glob` | empty | Scan prebuilt artifacts and skip the autopackager entirely |
| `package_source` | `.` | Source path passed to `veracode package` |
| `artifact_extensions` | `jar war ear dll exe nupkg zip tar tgz tar.gz` | Extensions treated as scannable artifacts |
| `scan_timeout` | `60` | Pipeline Scan timeout in minutes |
| `java_version` / `python_version` | `17` / `3.12` | Toolchain versions |
| `runs_on` | `ubuntu-latest` | Runner label |
| `upload_results` | `true` | Attach raw results to the run |
| `upload_sarif` | `false` | Publish delta findings to code scanning. Needs `security-events: write` on the caller job |
| `source_path_prefix` | empty | Prefix added to SARIF paths so they resolve in the repo, for example `src/main/java` |
| `app_name` | empty | Veracode application profile name. Overrides the `VERACODE_APP_NAME` secret |

## Secrets

| Secret | Required | Purpose |
|---|---|---|
| `VERACODE_API_ID` | yes | Veracode API ID |
| `VERACODE_API_KEY` | yes | Veracode API key |
| `CI_PUSH_TOKEN_VCT` | yes | PAT with read access to the baseline repo, and write access for baseline refreshes |
| `VERACODE_APP_NAME` | only for mitigated baselines | Exact Veracode application profile name |

Credentials are written to `~/.veracode/credentials` at the start of the job and are never passed on a command line, so they do not appear in the process table.

Pull requests from forks do not receive secrets. The workflow detects this and fails with an explanation rather than a confusing authentication error. Use branches in the repository itself, and do not switch to `pull_request_target` to work around it.

## Baseline types

The two stored files answer different questions, so pick deliberately.

`baseline.json` is the full result set from the last scan of the baseline branch. Using it as the delta baseline suppresses all pre-existing findings, so the build fails only on newly introduced flaws. This is the normal CI gate and the default.

`baseline-mitigated-findings.json` contains only the findings that have an **approved** mitigation in the Veracode platform, matched back onto the pipeline results by CWE, source file and line. Using it as the delta baseline suppresses nothing except accepted risk, so every unmitigated finding, old or new, fails the build. That is a useful policy style gate but it is not a delta gate. Set `baseline_type: mitigated` only if that is what you want.

`baseline_type: both` runs both comparisons and costs one extra scan per artifact.

## What this replaces

The previous version of this repo had three near identical workflows and a defect that made the whole thing a no-op in practice:

- Baselines were written to `baselines/<repo>/${{ github.ref_name }}` and delta scans read from the same expression. A scan on `feature/x` looked for baselines under `feature/x`, which never existed, so every delta scan reported a missing baseline and passed. Both sides now resolve the same baseline branch.
- On `pull_request`, `github.ref_name` is `<number>/merge`, which pushed baselines into nonsense paths.
- Baselines were only created manually or on a schedule. They now refresh on every push to the baseline branch.
- `vcpipemit.py` was invoked and its output guessed with `ls -t baseline-*.json | head -1`. The script has an `--outputfilename` flag, which is now used, and it runs from scratch space so its `vcpipmit.log` no longer gets committed to the store.
- `baseline.yml`, `delta.yml` and `veracode-delta-mitigated.yml` are gone, replaced by `veracode-pipeline.yml` with a `mode` input.

Other changes worth knowing about:

- Artifacts that disappear from a build no longer leave stale baselines behind. The branch directory is rebuilt on every refresh.
- Baseline pushes resync against the remote and retry, so several repositories can refresh into the shared store at the same time without clobbering each other.
- The baseline repo is checked out sparsely, so a store with hundreds of results stays cheap to clone.
- The job summary reports a severity breakdown per artifact instead of a bare pass or fail count.
- Secrets never reach a command line, and no user controlled value is interpolated into a shell script.
- Pull request scans cancel superseded runs. Baseline refreshes never cancel.

## Repository layout

```
.github/workflows/veracode-pipeline.yml   reusable workflow
.github/workflows/validate.yml            lint and self test for this repo
scripts/pipeline_summary.py               job summary rendering
scripts/results_to_sarif.py               multi artifact SARIF conversion
examples/veracode.yml                     caller template to copy
baselines/                                the baseline store
```

## Troubleshooting

**Every artifact reports `No baseline`.** No baseline refresh has run for that branch yet. Run the workflow from the baseline branch with `mode: baseline`.

**Baselines exist but are still not found.** The path key is derived from the artifact file name, so a change in the packaged artifact name creates a new key. Check the stored path printed in the run plan table at the top of the job summary.

**Mitigated baseline is empty.** The overlay matches platform findings to pipeline findings by CWE, source file and line with a small tolerance. An empty result usually means there is no completed policy scan for that application profile, or the profile name does not match exactly.

**`veracode package` finds nothing.** Autopackaging needs a buildable project. Either fix the packaging, or build in an earlier job and pass `artifacts_glob` instead.

## References

- [Pipeline Scan parameters](https://docs.veracode.com/r/r_pipeline_scan_commands)
- [veracode package](https://docs.veracode.com/r/veracode_package)
- [veracode/veracode-pipeline-mitigation](https://github.com/veracode/veracode-pipeline-mitigation)
