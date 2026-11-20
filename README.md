# Postman API Onboarding — Financial Services Platform

## Overview

This repository demonstrates automated API onboarding into Postman Enterprise for a financial services platform. Three services across payments, lending, and insurance are onboarded using pre-built GitHub Actions, turning OpenAPI specs into discoverable, testable, governed APIs in Postman — automatically.

**Services onboarded:**
- Payment Processing API — Refund Service (Lambda/API Gateway, OAuth2/JWT)
- Lending Platform API — Loan Origination Service (ECS/ALB, mTLS)
- Insurance Platform API — Claims Processing Service (mixed Lambda/ECS, GitLab CI, API key)

---

## How It Works

Each service has a dedicated GitHub Actions workflow under `.github/workflows/`. When triggered, the workflow calls `postman-cs/postman-api-onboarding-action@v0` which runs two phases internally:

**Phase 1 — Bootstrap** (`postman-bootstrap-action`):
- Creates a Postman workspace for the service
- Uploads the OpenAPI spec to Spec Hub
- Generates 3 collections from the spec: Baseline, Smoke, Contract
- Configures environments with base URLs per tier (prod, staging, dev, etc.)

**Phase 2 — Repo Sync** (`postman-repo-sync-action`):
- Exports generated collections back to this repo under `postman/collections/`
- Exports environments to `postman/environments/`
- Sets up a monitor scheduled to run the smoke collection every weekday at 9am
- Links the workspace to this repository

The action reads the OpenAPI spec at build time from the raw GitHub URL, extracts server URLs and auth schemes, and uses that information to build all Postman assets automatically — no manual collection authoring required.

---

## Repo Structure

```
.github/workflows/
  onboard-payment-refund.yml       # Workflow for payment refund service
  onboard-loan-origination.yml     # Workflow for loan origination service
  onboard-claims-processing.yml    # Workflow for claims processing service

spec/
  payment-refund-api-openapi.yaml
  loan-origination-api-openapi.yaml
  claims-processing-api-openapi.yaml

postman/
  collections/
    [Baseline] payment-refund-service/
    [Smoke] payment-refund-service/
    [Contract] payment-refund-service/
    [Baseline] loan-origination-service/
    [Smoke] loan-origination-service/
    [Contract] loan-origination-service/
    [Baseline] claims-processing-service/
    [Smoke] claims-processing-service/
    [Contract] claims-processing-service/
  environments/
    prod.postman_environment.json
    staging.postman_environment.json
    dev.postman_environment.json
    uat.postman_environment.json
    qa.postman_environment.json
  globals/
    workspace.globals.yaml
```

---

## Running the Workflows

### Prerequisites

| Requirement | Notes |
|---|---|
| Postman Enterprise trial | Free plan does not support Spec Hub or collection generation |
| `POSTMAN_API_KEY` | Generate from Postman account settings → API Keys |
| `POSTMAN_ACCESS_TOKEN` | Run `postman login` then `cat ~/.postman/postmanrc \| jq -r '.login._profiles[].accessToken'` |

Add both as repository secrets under **Settings → Secrets and variables → Actions**.

### How to Run

1. Go to the **Actions** tab in this repo
2. Select the workflow for the service you want to onboard
3. Click **Run workflow**
4. Run one service at a time — do not trigger multiple workflows simultaneously

**Important:** Run workflows sequentially, not in parallel. Each workflow commits generated assets back to `main`. Running them in parallel causes git push conflicts due to simultaneous commits to the same branch.

### Auto-trigger

Each workflow also triggers automatically when its spec file changes on `main`:

```
spec/payment-refund-api-openapi.yaml   → triggers onboard-payment-refund.yml
spec/loan-origination-api-openapi.yaml → triggers onboard-loan-origination.yml
spec/claims-processing-api-openapi.yaml → triggers onboard-claims-processing.yml
```

This means the Postman catalog stays current whenever a developer updates a spec — no manual re-run needed.

---

## What's Universal vs What Changes Per Service

### Universal — same across all services

- Workflow structure (checkout → clear state → onboard action)
- Action inputs: `generate-ci-workflow`, `monitor-cron`, `breaking-change-mode`
- Output structure: 3 collections (Baseline, Smoke, Contract), environments, globals
- Repo sync: collections committed back to `postman/` folder
- Secrets: `POSTMAN_API_KEY` and `POSTMAN_ACCESS_TOKEN`

### Changes per service

| Input | Payment Refund | Loan Origination | Claims Processing |
|---|---|---|---|
| `project-name` | payment-refund-service | loan-origination-service | claims-processing-service |
| `spec-url` | payment-refund spec URL | loan-origination spec URL | claims-processing spec URL |
| `domain` | payments | lending | insurance |
| `environments-json` | prod, uat, qa, dev | prod, staging, dev | prod, staging, dev |
| `env-runtime-urls-json` | payments URLs | lending URLs | — (uses spec servers) |
| `ssl-client-cert/key` | — | Required (mTLS) | — |
| `breaking-change-mode` | pr-native | pr-native | off (GitLab CI) |
| `onboarding-scope` | full | full | spec-only |

---

## What the Generated Collections Establish Automatically

The action reads the OpenAPI spec and generates tests automatically. Here is what it can and cannot infer.

### What specs infer automatically

- **All endpoints** — every path and method in the spec becomes a request in the Baseline collection
- **Request shapes** — parameters, headers, request body schemas pre-populated from spec definitions
- **Response validation** — Contract collection validates response status codes and schema shapes against the spec
- **Environment URLs** — extracted directly from the `servers` block in the spec
- **Auth scheme** — OAuth2, JWT, mTLS, API key schemes detected from `securitySchemes`
- **Health check** — `/health` endpoint detected and added to the Smoke collection
- **Smoke scope** — key happy-path flows identified and isolated into the Smoke collection

### What still requires human input

- **Real auth tokens** — the spec defines the auth scheme but not the actual credentials. Environment variables like `{{accessToken}}` and `{{apiKey}}` need to be populated with real values by the platform team
- **Test data** — the spec cannot infer valid `transactionId`, `applicationId`, or `claimId` values. These need to be seeded or generated per environment
- **Business logic assertions** — the Contract collection validates schema shapes but not business rules (e.g. "a refund cannot exceed the original transaction amount")
- **Chained flow tests** — multi-step scenarios (create claim → adjudicate → settle) require pre-request scripts that the action cannot generate from a spec alone
- **mTLS certificates** — the Loan Origination service requires client certificates. The spec declares mTLS as the auth scheme but the actual cert/key must be provided as secrets

This distinction matters operationally: the generated collections provide immediate value as a baseline, but they are not a substitute for service-specific test coverage. The platform team should treat them as a starting point, not a complete test suite.

---

## Claims Processing — Spec Quality Issues

The claims processing spec has structural issues that blocked full collection generation:

- Schema definitions use inline types instead of `$ref` references to reusable components
- Several operations are missing 5xx error responses

As a result, `onboarding-scope: spec-only` is set for claims — the spec is uploaded to Spec Hub but collections are not generated until the spec quality issues are resolved.

**Breaking change detection** is also disabled for claims (`breaking-change-mode: off`) because it uses GitLab CI, not GitHub Actions. The `pr-native` mode requires a GitHub PR context which does not apply to this service.

**What the platform team needs to fix before full onboarding:**
1. Refactor inline schemas to use `$ref` component references
2. Add 5xx responses to all operations
3. After spec fixes, change `onboarding-scope` back to `full` and re-run

---

## What the Platform Team Needs to Configure

These are things that cannot be done without access to the customer's internal infrastructure:

| Item | Why needed | Who provides it |
|---|---|---|
| Real environment base URLs | Current URLs are from spec placeholders. Monitors and CLI tests need real endpoints to run | Platform/ops team per environment |
| Auth credentials per environment | `accessToken`, `apiKey`, `x-api-key` variables need real values in each Postman environment | Service team per service |
| mTLS certificates for loan origination | `LOAN_SSL_CLIENT_CERT` and `LOAN_SSL_CLIENT_KEY` secrets needed for authenticated test runs | Security/platform team |
| GitLab CI integration for claims processing | The Postman CLI can be installed in GitLab CI pipelines using the same collection and environment exported here. The GitLab `.gitlab-ci.yml` needs a `postman collection run` step added | Platform team with GitLab access |
| Repo permissions for each service | Each service team needs to add `POSTMAN_API_KEY` and `POSTMAN_ACCESS_TOKEN` to their own repo secrets if this pattern is replicated per-repo | Platform team |

---

## Known Limitations and Trade-offs

**Single repo, multiple services:** This repo hosts all three services for demonstration purposes. In production, each service team would maintain their own repo with their own workflow and isolated Postman state. The action stores state in `.postman/resources.yaml` — with multiple services sharing one file, workflows must run sequentially and each workflow clears the shared state before running to avoid cross-service workspace contamination.

**CLI status shows Failed:** The smoke and contract collections run against the URLs defined in the spec (`api.payments.example.com`, etc.). These are placeholder URLs — the services are not deployed. CLI status will show green once real environment URLs are provided.

**Breaking change detection requires a GitHub PR:** `breaking-change-mode: pr-native` only fires on PR events. Running workflows manually (`workflow_dispatch`) does not trigger breaking change comparison. For automated breaking change detection, a spec change should be made via a PR, not a direct push to main.

---

## AI Usage

This implementation was built with Claude Code assistance. Claude was used for:
- Writing and iterating on the GitHub Actions workflow YAML
- Reading and interpreting upstream action documentation
- Drafting this README

All workflow logic, debugging decisions, and architectural choices were validated and directed by the engineer. The final workflows reflect deliberate decisions made during the build, including the trade-offs documented above.
