# Free-Tier OSS Security Stack

> **Goal**: Implement GitHub Actions hardening against [CICD-SEC-04 (Poisoned Pipeline Execution)](https://owasp.org/www-project-top-10-ci-cd-security-risks/CICD-SEC-04-Poisoned-Pipeline-Execution) using **only free tools** — no paid GitHub features, no custom runners, no Kubernetes.

## What this covers

| Threat | Control | Free? |
|---|---|---|
| Workflow script injection | [Zizmor](https://github.com/woodruffw/zizmor) (SAST) | ✅ |
| `pull_request_target` + fork checkout | Zizmor | ✅ |
| Unpinned action references | Zizmor + Dependabot | ✅ |
| Secrets in source code | [Gitleaks](https://github.com/gitleaks/gitleaks) + GitHub secret scanning | ✅ (public repo) |
| Push protection | GitHub built-in (public repos) | ✅ |
| Code vulnerabilities | [CodeQL](https://codeql.github.com) (public repos) | ✅ |
| Supply chain posture | [OSSF Scorecard](https://github.com/ossf/scorecard) | ✅ |
| Artifact tampering | [SLSA Level 3 provenance](https://github.com/slsa-framework/slsa-github-generator) | ✅ |
| Org-level policy drift | GitHub branch protection rulesets | ✅ |
| Dependency version pinning + cooldown | [Renovate](https://github.com/renovatebot/renovate) | ✅ |

## Architecture

```
GitHub-hosted runner (ubuntu-latest, ephemeral)
          │
          ├── on: pull_request
          │     ├── zizmor         → Code scanning alerts (SARIF)
          │     ├── gitleaks       → Secret scan (blocks merge if found)
          │     └── codeql         → SAST results (SARIF)
          │
          ├── on: push (main)
          │     ├── scorecard      → Supply chain score (securityscorecards.dev)
          │     └── gitleaks       → Full history secret scan
          │
          └── on: push (tag v*)
                └── slsa-provenance → SLSA L3 .intoto.jsonl attached to release
```

## Workflows

| Workflow | Trigger | Purpose |
|---|---|---|
| [zizmor.yml](.github/workflows/zizmor.yml) | push/PR to `*.yml` | Workflow SAST — injection, PPE, unpinned actions |
| [secret-scan.yml](.github/workflows/secret-scan.yml) | push/PR | Secret detection (160+ patterns) |
| [codeql.yml](.github/workflows/codeql.yml) | push/PR/weekly | Application code SAST |
| [scorecard.yml](.github/workflows/scorecard.yml) | push main / weekly | OSSF supply chain score |
| [slsa-provenance.yml](.github/workflows/slsa-provenance.yml) | push semver tag | SLSA L3 signed provenance |
| [ci.yml](.github/workflows/ci.yml) | push/PR | Hardened CI template (reference) |

## Org-level security settings (applied via `gh` CLI)

- **Default workflow token**: read-only (`contents: read`)
- **Actions allowlist**: GitHub-owned + verified marketplace creators + pinned OSS tools
- **Fork PR approval**: required for all outside collaborators

## Secure workflow patterns (copy-paste safe)

### ✅ Minimal permissions
```yaml
permissions: {}          # deny all at workflow level

jobs:
  build:
    permissions:
      contents: read     # grant only what the job needs
```

### ✅ Safe data passing (no script injection)
```yaml
# BAD — injects untrusted data directly into shell
- run: echo "${{ github.event.pull_request.title }}"

# GOOD — pass via env:, shell treats it as data not code
- run: echo "$PR_TITLE"
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
```

### ✅ SHA-pinned actions
```yaml
# BAD — tag can be moved to a malicious commit
- uses: actions/checkout@v4

# GOOD — immutable reference
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
```

### ✅ Safe trigger (not pull_request_target)
```yaml
# BAD — pull_request_target runs in base repo context with full secrets for fork PRs
on:
  pull_request_target:

# GOOD — pull_request runs in fork context with no secrets
on:
  pull_request:
```

### ✅ Persist-credentials: false
```yaml
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
  with:
    persist-credentials: false   # removes git credentials from runner after checkout
```

## What's NOT covered (paid features)

| Feature | Requires | OSS Alternative |
|---|---|---|
| Secret scanning on **private** repos | GitHub Secret Protection | Gitleaks (this repo) |
| CodeQL on **private** repos | GitHub Code Security | Semgrep OSS |
| Dependency review (blocking) on **private** repos | GitHub Code Security | Manual review |
| Artifact attestations on **private** repos | GitHub Enterprise Cloud | SLSA generator (this repo) |
| Org-wide security dashboard | Code Security / Secret Protection license | [Legitify](https://github.com/Legit-Labs/legitify) |

## Cost

**$0/month** — GitHub Free plan, GitHub-hosted runners, all OSS tools.
Free-tier OSS security stack for GitHub Actions hardening (CICD-SEC-04 PoC)
