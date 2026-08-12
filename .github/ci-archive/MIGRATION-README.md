# Jenkins → GitHub Actions Migration Report

**Repository:** `jenkins-migrations/backup-recovery-platform`
**Date:** 2026-08-12
**Status:** Workflows created locally, **NOT committed/pushed** per explicit user instruction.

> ⚠️ The user explicitly requested that no commit, push, or PR be created for this
> migration. All changes described below exist only in the local working tree of
> this session and have **not** been persisted to the remote repository or opened
> as a pull request, overriding the agent's normal "always open a PR" requirement.

---

## 1. Source Files Migrated

| Original Jenkins file | Pipeline type | Archived to |
|---|---|---|
| `Jenkinsfile` (repo root) | Scripted pipeline (`node` + `parallel`) | `.github/ci-archive/Jenkinsfile` |
| `msbuilddotnet/Jenkinsfile` | Scripted pipeline (`node`, single agent) | `.github/ci-archive/msbuilddotnet/Jenkinsfile` |

Both files are sample pipelines sourced from `jenkinsci/pipeline-examples` (MIT
License), as noted in their file headers. No Jenkins shared library (`vars/`)
calls were present in either file, so no shared-library expansion was required.

## 2. New GitHub Actions Workflows

| New workflow | Replaces | Trigger |
|---|---|---|
| `.github/workflows/multi-platform-build.yml` | root `Jenkinsfile` | `push`, `pull_request`, `workflow_dispatch` |
| `.github/workflows/msbuild-dotnet.yml` | `msbuilddotnet/Jenkinsfile` | `push`, `pull_request`, `workflow_dispatch` |

Supporting file: `.github/actionlint.yaml` — declares the custom self-hosted
runner label (`deploy`) used by the deploy job so actionlint doesn't flag it as
unknown.

### 2.1 `multi-platform-build.yml`

Converts the root Jenkinsfile's `parallel` map over three node labels
(`precise`, `trusty`, `windows`) into a matrix `build` job:

- `precise` and `trusty` are legacy Ubuntu 12.04/14.04 Jenkins agent labels
  that ran **identical** Unix build steps. Both are preserved as separate
  matrix entries (mapped to `ubuntu-latest`) purely to keep the original
  3-way parallel fan-out structure intact; functionally they are duplicate
  builds and could be collapsed to a single Ubuntu entry if that redundancy
  is not desired going forward.
- `windows` → `windows-latest`.
- `isUnix()`-gated `sh`/`bat` steps → `if: runner.os != 'Windows'` /
  `if: runner.os == 'Windows'` steps with `shell: bash` / `shell: cmd`.
- `archiveArtifacts artifacts: 'build/**/*', fingerprint: true` →
  `actions/upload-artifact` per matrix leg (fingerprinting is inherent to
  artifact storage/hashing in Actions, no separate flag needed).
- `checkout scm` → `actions/checkout`.

### 2.2 `msbuild-dotnet.yml`

Converts the single-node `.NET` MSBuild pipeline:

- `checkout scm` → `actions/checkout`.
- `tool 'MSBuild'` → `microsoft/setup-msbuild` (adds `msbuild` to `PATH`).
- `nuget restore` (implicit CLI) → `NuGet/setup-nuget` + `nuget restore`.
- `tool 'VSTest'` + manual `vstest.console.exe` invocation →
  `microsoft/vstest-action` (runs tests, uploads `.trx` result logs as an
  artifact).
- `publishTestResults testResultsPattern: '**/*.trx'` → `dorny/test-reporter`
  (`reporter: dotnet-trx`), publishing a GitHub Checks summary of test results.
- `archiveArtifacts artifacts: 'ProjectName/bin/Release/**/*', fingerprint: true`
  → `actions/upload-artifact` (`build-output`).
- `publishHTML(...)` (Jenkins HTML Publisher plugin) → **no direct GitHub
  Actions equivalent**. The `index.html` report is preserved by uploading it
  as a downloadable artifact (`build-report`) instead of an inline hosted
  page. If a browsable report is required, consider publishing to GitHub
  Pages or an external static host as a follow-up enhancement.
- `Deploy` stage (`when { branch 'master' }` → `xcopy ... C:\Deploy\...`) →
  separate `deploy` job gated on `if: github.ref == 'refs/heads/master' &&
  github.event_name == 'push'`, preserving the Jenkins branch condition.
  - **Important limitation:** the original step copies files to a local
    filesystem path (`C:\Deploy\ProjectName\`) *on the Jenkins build agent
    itself*. GitHub-hosted runners are ephemeral VMs with no persistent
    `C:\Deploy` target, so this cannot run on `windows-latest`. The `deploy`
    job is configured with `runs-on: [self-hosted, windows, deploy]` and
    requires:
    1. A self-hosted Windows runner registered with the `deploy` label.
    2. A repository/environment **variable** `DEPLOY_TARGET_PATH` set to the
       real deployment directory (kept as a variable, not hardcoded, so the
       destination isn't baked into the workflow file).
  - The deploy job downloads the `build-output` artifact produced by the
    isolated build job before performing `xcopy`; GitHub Actions jobs do not
    share a workspace.
  - If no self-hosted runner is available, this job will simply stay queued/fail
    to pick up a runner — it does not silently do nothing, so this should be
    addressed (provision the runner, or replace this stage with a real
    deployment action such as `azure/webapps-deploy`, an artifact-based
    release step, etc.) before relying on this job.

## 3. Actions Used (Marketplace, SHA-Pinned)

All actions are pinned to a full commit SHA (not a mutable tag) per security
guardrails, with the human-readable version kept as a trailing comment.

| Action | Version | Commit SHA | Publisher |
|---|---|---|---|
| `actions/checkout` | v4.2.2 | `11bd71901bbe5b1630ceea73d27597364c9af683` | GitHub (verified) |
| `actions/download-artifact` | v4.1.8 | `fa0a91b85d4f404e444e00e005971372dc801d16` | GitHub (verified) |
| `actions/upload-artifact` | v4.4.3 | `b4b15b8c7c6ac21ea08fcf65892d2ee8f75cf882` | GitHub (verified) |
| `microsoft/setup-msbuild` | v2 | `6fb02220983dee41ce7ae257b6f4d8f9bf5ed4ce` | Microsoft (verified) |
| `NuGet/setup-nuget` | v2.0.1 | `323ab0502cd38fdc493335025a96c8fdb0edc71f` | NuGet/Microsoft (verified) |
| `microsoft/vstest-action` | v1.0.0 | `6ef4755ea07a3144e3da36f10d76187086eee5e6` | Microsoft (verified) |
| `dorny/test-reporter` | v1.9.1 | `31a54ee7ebcacc03a09ea97a7e5465a47b84aea5` | dorny (widely-used, actively maintained community action; no first-party Microsoft/GitHub action publishes `.trx` results as GitHub Checks) |

## 4. Secrets & Variables Mapping

Neither Jenkinsfile contained a `withCredentials` / credentials binding block,
so **no Jenkins credentials required migration to GitHub Secrets**.

| Item | Type | Used by | Notes |
|---|---|---|---|
| `DEPLOY_TARGET_PATH` | Repository/Environment **variable** (`vars.*`, not a secret) | `msbuild-dotnet.yml` → `deploy` job | Replaces the hardcoded `C:\Deploy\ProjectName\` path from the Jenkins `Deploy` stage. Not sensitive, so modeled as an Actions **variable** rather than a secret. Must be set before the `deploy` job can run correctly. |
| *(none)* | Secret | — | No API keys, passwords, tokens, SSH keys, or certificate credentials were present in the source pipelines. |

The `.NET` workflow grants `checks: write` in addition to `contents: read`,
which is required by `dorny/test-reporter` to publish its test-result check.

**Required self-hosted runner:** a Windows runner labeled `deploy` must be
registered in the repository/organization for the `deploy` job in
`msbuild-dotnet.yml` to execute (see §2.2).

## 5. Validation

- **YAML syntax:** both workflows parse cleanly with `yaml.safe_load` (Python).
- **actionlint:** installed via `go install github.com/rhysd/actionlint/cmd/actionlint@v1.7.12`
  since no pre-built binary/network mirror was reachable in this
  environment. Ran:
  ```
  actionlint .github/workflows/*.yml
  ```
  Initial run flagged the custom `deploy` self-hosted runner label as unknown
  (`runner-label` rule). Added `.github/actionlint.yaml` declaring
  `self-hosted-runner.labels: [deploy]` to resolve it.
  Final run: **0 errors, 0 warnings** (exit code 0).
- **Secret scan:** `runtime-tools-secret_scanning` run against both workflow
  files and `.github/actionlint.yaml` — no secrets detected.
- **CodeQL:** the Actions workflow analysis found **0 alerts**.

## 6. Knowledge Base Access Note

This migration could not fetch `jenkins-migrations/.github-private` (the
internal-visibility knowledge base repo referenced in the agent instructions):
`GET /repos/jenkins-migrations/.github-private` returned `404 Not Found` for
the credentials available in this session (repo confirmed to exist via org
repository search, but not readable — likely restricted to org members/an
internal GitHub App installation this session's token isn't part of). The
migration was therefore completed using the standardized process, mapping
conventions, and guardrails (SHA-pinning, verified publishers, credential
handling, archive/report structure) embedded directly in this agent's
operating instructions, which mirror the knowledge-base document set. No
document-specific guidance beyond that was available or applied.

## 7. Outstanding Follow-Ups for Repository Owners

1. Register a self-hosted Windows runner labeled `deploy` (or replace the
   `deploy` job with a proper deployment action) before relying on
   `msbuild-dotnet.yml`'s deploy stage.
2. Set the `DEPLOY_TARGET_PATH` repository/environment variable.
3. Decide whether the duplicate `precise`/`trusty` Ubuntu matrix legs in
   `multi-platform-build.yml` should be collapsed to a single Ubuntu entry.
4. Consider publishing the `.NET` HTML build report to GitHub Pages if an
   inline browsable report (rather than a downloadable artifact) is required.
5. This report and the new workflow/archive files currently exist **only in
   the local working tree** — per user instruction, nothing was committed,
   pushed, or opened as a PR. A maintainer will need to review and commit
   these changes manually if the migration is approved.
