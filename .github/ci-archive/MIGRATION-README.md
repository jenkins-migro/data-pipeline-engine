# Jenkins → GitHub Actions Migration Report

## Summary

| Source Jenkinsfile | Pipeline Type | New GitHub Actions Workflow |
| --- | --- | --- |
| `Jenkinsfile` (repo root) | Declarative | `.github/workflows/process.yml` |
| `zomatoanypoint/Jenkinsfile` | Declarative | `.github/workflows/zomato-anypoint-deploy.yml` |

Both original Jenkinsfiles have been archived (unmodified) in this directory for reference and are no longer used to drive CI/CD.

---

## 1. `Jenkinsfile` → `.github/workflows/process.yml`

### Analysis

- **Agent:** `agent { label 'master' }` → `runs-on: ubuntu-latest`
- **Trigger:** No explicit trigger was defined in the Jenkinsfile, so the workflow uses `workflow_dispatch` (manual trigger). Add additional `on:` triggers (e.g. `push`, `schedule`) if the pipeline should run automatically.
- **Stage `Process`:**
  - Sets `INPUT_FILE` environment variable to `${WORKSPACE}/input.csv` → mapped to `${{ github.workspace }}/input.csv`.
  - `sh` step decodes a Base64 string and writes it to the input file, then verifies the file exists → converted 1:1 to a `run:` step.
  - `archiveArtifacts artifacts: 'input.csv', allowEmptyArchive: true` → `actions/upload-artifact@v4.6.2` with `if-no-files-found: ignore`.

### Conversion Notes

- No shared libraries were referenced in this Jenkinsfile.
- No credentials were referenced in this Jenkinsfile.
- Actions pinned to commit SHA per security guardrails:
  - `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683` (`v4.2.2`)
  - `actions/upload-artifact@ea165f8d65b6e75b540449e92b4886f43607fa02` (`v4.6.2`)

### Required Secrets / Variables

None.

---

## 2. `zomatoanypoint/Jenkinsfile` → `.github/workflows/zomato-anypoint-deploy.yml`

### Analysis

- **Agent:** `agent any` → `runs-on: ubuntu-latest`
- **Trigger:** No explicit trigger was defined, so the workflow uses `workflow_dispatch` (manual trigger).
- **Environment / Credentials:** `ANYPOINT_CREDENTIALS = credentials('anypoint.credentials')` is a Jenkins username/password credential binding, exposing `ANYPOINT_CREDENTIALS_USR` and `ANYPOINT_CREDENTIALS_PSW`. These are mapped to two GitHub Actions repository secrets: `ANYPOINT_USERNAME` and `ANYPOINT_PASSWORD`.
- **Stages → Jobs** (converted to sequential jobs using `needs:` to preserve original stage order, since GitHub Actions has no direct "stage" concept in a single job that mirrors Jenkins' sequential-by-default stage execution shown here):
  - `Unit Test` → job `unit-test` (`mvn clean test`)
  - `Deploy Standalone` → job `deploy-standalone` (`mvn deploy -P standalone`)
  - `Deploy to AnyPoint` → job `deploy-anypoint` (`mvn deploy -P arm ...`)
  - `Deploy to CloudHub` → job `deploy-cloudhub` (`mvn deploy -P cloudhub ...`)
  - Each job adds `actions/checkout` and `actions/setup-java` (Temurin 17, Maven cache) since Maven/JDK are required for the `mvn` commands, matching the implicit build tooling expected by the Jenkins agent.
- **Post actions:**
  - `post { success { ... } failure { ... } }` → converted to a final `notify` job that always runs (`if: always()`) and checks the result of the last deployment job (`needs.deploy-cloudhub.result`).

### Conversion Notes

- No shared libraries were referenced in this Jenkinsfile.
- Credential binding `anypoint.credentials` (username/password) mapped to GitHub Actions secrets `ANYPOINT_USERNAME` / `ANYPOINT_PASSWORD`, exposed as job-level environment variables and referenced via Maven `-D` properties.
- Actions pinned to commit SHA per security guardrails:
  - `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683` (`v4.2.2`)
  - `actions/setup-java@c5195efecf7bdfc987ee8bae7a71cb8b11521c00` (`v4.7.1`)

### Required Secrets / Variables

| Name | Type | Description |
| --- | --- | --- |
| `ANYPOINT_USERNAME` | Secret | Anypoint Platform username (previously `ANYPOINT_CREDENTIALS_USR`) |
| `ANYPOINT_PASSWORD` | Secret | Anypoint Platform password (previously `ANYPOINT_CREDENTIALS_PSW`) |

Configure these under **Settings → Secrets and variables → Actions → New repository secret**.

---

## Validation

- Both workflows were validated with [`actionlint`](https://github.com/rhysd/actionlint) with no errors reported.
- Both workflows declare an explicit `permissions: contents: read` block (least-privilege `GITHUB_TOKEN`), verified with CodeQL's Actions analysis (0 alerts).

## Follow-up

- Confirm the intended trigger(s) for each workflow (currently `workflow_dispatch` only) and add `push`/`pull_request`/`schedule` triggers as appropriate.
- Add the `ANYPOINT_USERNAME` and `ANYPOINT_PASSWORD` repository secrets before running the `zomato-anypoint-deploy` workflow.
