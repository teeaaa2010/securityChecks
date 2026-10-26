# Security Pipeline: GitHub Actions Example

A sample GitHub Actions workflow that integrates multiple security checks into a CI/CD pipeline. Independent scanners run in parallel, findings are published to the GitHub Security tab, and a single gate job decides whether a deploy may proceed.

## What's included

| Stage | Job | Tool | Purpose |
|---|---|---|---|
| Secrets | `secret-scan` | Gitleaks | Detect committed credentials across full git history |
| SAST | `codeql` | CodeQL | Deep semantic code analysis (`security-extended` queries) |
| SAST | `semgrep` | Semgrep | Fast rule-based analysis (CI and OWASP Top 10 rulesets) |
| SCA | `dependency-review` | Dependency Review | Block risky dependency and license changes in PRs |
| SCA | `dependency-scan` | Trivy (fs) | Scan lockfiles for known CVEs |
| IaC | `iac-scan` | Trivy (config) | Find misconfigurations in Terraform, Kubernetes, Dockerfiles, and workflows |
| Container | `container-scan` | Trivy (image) + Anchore | Scan the built image and generate an SPDX SBOM |
| DAST | `dast` | OWASP ZAP | Baseline scan of the running application |
| Gate | `security-gate` | jq | Aggregate results into one pass/fail status |
| Release | `deploy` | n/a (placeholder) | Deploy only after the gate passes |

## Pipeline flow

```
secret-scan ──► container-scan ──┐
codeql ──────────────────────────┤
semgrep ─────────────────────────┤
dependency-review (PRs only) ────┼──► security-gate ──► deploy (main only)
dependency-scan ─────────────────┤
iac-scan ────────────────────────┤
dast (non-PR; needs container-scan, codeql, semgrep) ─┘
```

## Triggers

- **Push to `main`**: full pipeline, including DAST and deploy
- **Pull requests to `main`**: all checks except DAST and deploy (keeps feedback fast); adds Dependency Review
- **Weekly schedule** (Mondays 03:00 UTC): catches newly published CVEs in code that hasn't changed
- **Manual** (`workflow_dispatch`)

## Setup

1. Copy `security-pipeline.yml` to `.github/workflows/` in your repository.
2. Adjust to your project:
   - **CodeQL languages**: edit the `matrix.language` list.
   - **Dockerfile and build context**: the `container-scan` and `dast` jobs assume a `Dockerfile` at the repo root.
   - **App port and health path**: the DAST job expects the app on `localhost:8080` and responding to `GET /`.
   - **Image name**: change `IMAGE_NAME` in the top-level `env` block.
3. Add secrets if needed:
   - `GITLEAKS_LICENSE` is required for Gitleaks on organization-owned repos (uncomment the line in the workflow).
4. Replace the `deploy` job's placeholder step with your real deployment, using OIDC (`id-token: write`) rather than long-lived cloud credentials.

## Repository settings to configure

- **Branch protection on `main`**: mark **Security gate** as the only required status check. Jobs that are legitimately skipped (for example, DAST on PRs) won't block merges because the gate treats `skipped` as acceptable.
- **Environments**: create a `production` environment with required reviewers so deploys need manual approval.
- **Code scanning**: SARIF uploads require GitHub Advanced Security on private repositories. Public repositories get it free.
- **Dependabot**: enable version updates for `github-actions` so pinned action versions stay current.

## How failures are handled

- Scanners that support it run with a non-zero exit code on HIGH/CRITICAL findings (`exit-code: "1"`, or `--error` for Semgrep).
- SARIF upload steps use `if: always()` so findings appear in the Security tab even when the scan fails the job.
- `security-gate` runs with `if: always()` and fails if any upstream job `failure`d or was `cancelled`.

## Hardening notes

- **Pin actions to commit SHAs.** Tags in this example (`@v4`, `@0.28.0`, and so on) are for readability. For production, use `uses: owner/repo@<40-char-sha> # vX.Y.Z`.
- **Least privilege.** The workflow defaults to `contents: read`; each job adds only the permissions it needs (`security-events: write` for SARIF, `pull-requests: write` for PR comments, and so on).
- **Concurrency control** cancels superseded runs on the same ref to save runner minutes.
- **Tune severity thresholds.** `HIGH,CRITICAL` with `ignore-unfixed: true` is a pragmatic starting point; tighten or loosen as your backlog allows.
- **Triage noise deliberately.** Use `.trivyignore`, Semgrep `nosemgrep` comments, or CodeQL dismissals with written justifications rather than disabling checks.

## Outputs

- **Security tab** (Code scanning alerts): CodeQL, Semgrep, and the three Trivy scans, each under its own category
- **PR comments**: Dependency Review summary on failure
- **Artifacts**: `sbom` (SPDX JSON, retained 30 days)
- **Issues**: ZAP baseline opens a summary issue (requires `issues: write`)

## Extending the pipeline

Common additions:

- Sign images with Cosign and attach build provenance attestations (SLSA)
- Add language-specific linters or audit tools (`npm audit`, `pip-audit`, `govulncheck`)
- Run Checkov or tfsec for deeper Terraform analysis
- Upload SBOMs to a dependency-tracking platform
- Notify a chat channel when the weekly scheduled scan fails

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| SARIF upload fails with "Advanced Security must be enabled" | Private repo without GitHub Advanced Security |
| Gitleaks fails with a license error | Organization repo missing `GITLEAKS_LICENSE` |
| CodeQL autobuild fails | Compiled language needs explicit build steps in place of `autobuild` |
| DAST job times out waiting for the app | Wrong port or health path, or the app needs env vars or a database |
| Gate fails but all jobs look green | A job was `cancelled` (often by the concurrency group); re-run the workflow |

## License

Provided as a sample; adapt freely to your own projects.
