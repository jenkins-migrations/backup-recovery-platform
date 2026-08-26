# Jenkins to GitHub Actions Migration Report

## Summary

Migrated the repository's Jenkins pipeline definitions to GitHub Actions workflows and archived the original Jenkins configuration files.

## Source Pipelines

| Original Jenkins file | Pipeline type | New workflow |
| --- | --- | --- |
| `Jenkinsfile` | Scripted Jenkins pipeline with parallel Linux and Windows node builds | `.github/workflows/multi-platform-build.yml` |
| `msbuilddotnet/Jenkinsfile` | Scripted Jenkins pipeline for a Windows/MSBuild .NET build | `.github/workflows/msbuild-dotnet.yml` |

## Workflow Mapping

### Multi-Platform Build

- Jenkins node labels `precise`, `trusty`, and `windows` were converted to a GitHub Actions platform matrix.
- The historical Ubuntu labels are approximated with supported GitHub-hosted runners: `precise` runs on `ubuntu-22.04`, `trusty` runs on `ubuntu-24.04`, and `windows` runs on `windows-latest`. The original Ubuntu 12.04/14.04 environments are no longer available as GitHub-hosted runners, so the migration uses two maintained Ubuntu LTS images to preserve separate Linux matrix coverage.
- Jenkins `checkout scm` maps to `actions/checkout`.
- Jenkins `sh 'make'` and `sh 'make test'` map to Bash steps on Linux runners.
- Jenkins `bat 'build.bat'` and `bat 'test.bat'` map to PowerShell steps on Windows runners.
- Jenkins `archiveArtifacts` maps to `actions/upload-artifact` for `build/**` when artifacts exist.

### MSBuild .NET

- Jenkins `checkout scm` maps to `actions/checkout`.
- Jenkins `nuget restore SolutionName.sln` maps to a NuGet restore step after `NuGet/setup-nuget`.
- Jenkins MSBuild tool usage maps to `microsoft/setup-msbuild` and an `msbuild` command using `github.run_number` in place of `env.BUILD_NUMBER`.
- Jenkins VSTest usage maps to `vstest.console.exe` and TRX artifact upload when results exist.
- Jenkins artifact and HTML report publishing map to `actions/upload-artifact` when files exist; the full release artifact is uploaded on non-master branches, while master publishes the same build output as the deployment package to avoid duplicate artifacts.
- Jenkins `branch 'master'` deploy condition maps to a master-only deployment package artifact because GitHub-hosted runner filesystems are ephemeral and no external deployment target was defined in Jenkins.

## Archived Files

The original Jenkins files were moved to `.github/ci-archive/`:

- `.github/ci-archive/Jenkinsfile`
- `.github/ci-archive/msbuilddotnet/Jenkinsfile`

## Actions and Security

Marketplace actions are pinned to commit SHAs:

- `actions/checkout@11d5960a326750d5838078e36cf38b85af677262` (v4)
- `actions/upload-artifact@ea165f8d65b6e75b540449e92b4886f43607fa02` (v4)
- `NuGet/setup-nuget@d105a947828025cd7a980103c35ba2bfae586d0f` (v2)
- `microsoft/setup-msbuild@6fb02220983dee41ce7ae257b6f4d8f9bf5ed4ce` (v2)

## Secrets and Variables

No Jenkins credentials, `withCredentials` bindings, or secret variables were present in the source Jenkins pipelines. No GitHub Actions secrets are required for the migrated workflows.

## Validation

- Workflow syntax validation: `actionlint` completed successfully.
- Source Jenkins configurations archived: complete.
- Shared library expansion: not applicable; no shared library calls were present.

## Notes

This repository currently contains sample pipeline definitions but no Makefile, Windows batch files, solution file, test assembly, or build artifacts. The generated workflows preserve the Jenkins stage structure while skipping missing sample build inputs so that the migration workflow itself can run in this repository without failing before application build files are added.
