# Adaptation Analysis — Payment Refund vs Claims Processing

## Purpose

This document analyzes what changes when applying the same onboarding pattern to a second service with meaningfully different characteristics. The Payment Refund service was the primary build. Claims Processing is the adaptation target.

The goal is to show that the onboarding pattern is a repeatable system — not a series of custom projects — and to identify where platform-team coordination is required versus where the automation handles it automatically.

---

## Service Comparison

| Dimension | Payment Refund (Primary) | Claims Processing (Adaptation) |
|---|---|---|
| Infrastructure | AWS Lambda + API Gateway | Mixed Lambda + ECS |
| CI/CD | GitHub Actions | GitLab CI |
| Auth (external) | OAuth 2.0 (authorization code flow) | OAuth 2.0 (client credentials flow) |
| Auth (internal) | JWT Bearer token | API key (X-API-Key header) |
| Environments | prod, uat, qa, dev (4 tiers) | prod, staging, dev (3 tiers) |
| Spec quality | Well-structured, full $ref usage | Inline schemas, missing 5xx responses |
| API version | v2 | v3 |
| Domain | Payments | Insurance |
| Teams | GitHub-native | GitLab CI — no GitHub Actions in delivery pipeline |

---

## What Stays the Same

Everything structural about the onboarding workflow is identical:

- The GitHub Actions workflow file structure (trigger, permissions, checkout, action call)
- The upstream action `postman-cs/postman-api-onboarding-action@v0`
- The two-phase execution model (bootstrap → repo sync)
- Secrets required: `POSTMAN_API_KEY` and `POSTMAN_ACCESS_TOKEN`
- The output: workspace created, spec in Spec Hub, collections committed to repo, environments exported
- The auto-trigger pattern: spec file change on `main` fires the workflow
- Monitor setup: smoke collection scheduled to run on a cron

The onboarding workflow for claims is exactly 3 inputs different from payment refund. That is the repeatable system in practice.

---

## What Changes

### 1. CI/CD System — GitLab CI, Not GitHub Actions

This is the most significant difference. Claims Processing deploys via GitLab CI. The GitHub Actions workflow in this repo can still onboard the spec and generate Postman assets — that part works fine. But wiring those generated collections into the claims team's actual CI/CD pipeline requires accessing their GitLab instance, which is outside the scope of this engagement.

**What the GitHub Actions workflow does:**
- Uploads spec to Spec Hub ✓
- Generates collections and exports them to this repo ✓
- Creates environments ✓
- Sets up Postman monitor ✓

**What requires GitLab coordination:**
- Adding a `postman collection run` step to the claims team's `.gitlab-ci.yml`
- Providing the Postman CLI and collection IDs to the GitLab runner
- Storing `POSTMAN_API_KEY` as a GitLab CI variable (not a GitHub secret)
- Configuring the GitLab runner to have network access to the claims API endpoints

**What you would need from the platform team:**
- GitLab project access or a GitLab CI template they can apply themselves
- Confirmation that the GitLab runner has outbound network access to the claims API
- The GitLab CI variable store for adding Postman credentials

### 2. Auth Scheme — API Key Instead of JWT

Payment Refund uses JWT Bearer tokens for internal service calls. Claims Processing uses an `X-API-Key` header for internal service-to-service communication.

The action detects both schemes from the spec's `securitySchemes` block and generates the correct auth setup in collections. No workflow change needed.

What changes operationally: the platform team needs to populate `{{X-API-Key}}` in each Postman environment instead of `{{accessToken}}`. The variable name is different; the concept is the same.

For the external OAuth2 flow, Claims Processing uses **client credentials** flow instead of **authorization code** flow. This is a machine-to-machine pattern — no browser redirect, just a direct token exchange. The collection pre-request scripts need to be updated to use `grant_type=client_credentials` instead of an authorization code exchange. This is a one-time manual step per environment.

### 3. Spec Quality — Blocked Contract Collection Generation

The claims spec has two structural issues that blocked contract collection generation:

**Issue 1 — Inline schemas instead of `$ref` references:**
```yaml
# Claims spec (problematic)
requestBody:
  content:
    application/json:
      schema:
        type: object
        properties:
          policyId:
            type: string
```
versus the payment refund spec which uses:
```yaml
# Payment refund spec (correct)
requestBody:
  content:
    application/json:
      schema:
        $ref: '#/components/schemas/CreateRefundRequest'
```

The contract generator requires reusable component references to generate meaningful contract assertions. Without them it cannot determine what schema to validate responses against.

**Issue 2 — Missing 5xx responses:**
Every operation in the claims spec is missing 5xx error response definitions. The contract collection generator expects at minimum a 500 response definition per operation to generate negative-path contract tests.

**Impact:** `onboarding-scope: spec-only` was used for claims. The spec is in Spec Hub but the Baseline, Smoke, and Contract collections are not generated until the spec is fixed.

**Workflow change required:** One input change after spec is fixed:
```yaml
# Change from:
onboarding-scope: spec-only

# To:
onboarding-scope: full
```

### 4. Environments — 3 Tiers Instead of 4

Payment Refund has four environment tiers: prod, uat, qa, dev.
Claims Processing has three: prod, staging, dev. No UAT or QA tiers.

```yaml
# Payment Refund
environments-json: '["prod","uat","qa","dev"]'

# Claims Processing
environments-json: '["prod","staging","dev"]'
```

The action creates the correct number of environments automatically. This is a one-line input change.

### 5. Breaking Change Detection — Off for Claims

Payment Refund uses `breaking-change-mode: pr-native` — the action compares the spec on a PR branch against the baseline and fails the PR if breaking changes are detected.

This does not apply to Claims Processing because:
- The service uses GitLab CI for deployments
- Spec changes go through GitLab merge requests, not GitHub PRs
- `pr-native` mode requires a GitHub PR event context

For breaking change detection to work for claims, the platform team would need to either:
- Mirror spec changes through a GitHub PR process (additional coordination overhead), or
- Use `breaking-change-mode: baseline-only` to compare against the last known good spec regardless of trigger type

For now it is set to `off`.

---

## Side-by-Side Workflow Diff

| Input | Payment Refund | Claims Processing | Change required |
|---|---|---|---|
| `project-name` | payment-refund-service | claims-processing-service | Yes — different name |
| `spec-url` | payment-refund raw URL | claims-processing raw URL | Yes — different file |
| `domain` | payments | insurance | Yes — different domain |
| `environments-json` | prod, uat, qa, dev | prod, staging, dev | Yes — fewer tiers |
| `env-runtime-urls-json` | 4 URLs | not set (uses spec) | Yes — removed |
| `breaking-change-mode` | pr-native | off | Yes — GitLab CI |
| `onboarding-scope` | full | spec-only | Yes — spec quality |
| `ssl-client-cert/key` | not set | not set | No change |
| `monitor-cron` | 0 9 * * 1-5 | 0 9 * * 1-5 | No change |
| `generate-ci-workflow` | false | false | No change |
| Secrets | POSTMAN_API_KEY, POSTMAN_ACCESS_TOKEN | POSTMAN_API_KEY, POSTMAN_ACCESS_TOKEN | No change |

**6 inputs change. 5 stay the same.** The workflow structure, action version, secrets, monitor schedule, and CI workflow setting are identical. This is the repeatable system.

---

## What the Platform Team Needs to Provide for Claims

| Item | Why | Who |
|---|---|---|
| GitLab CI access or template handoff | To wire Postman CLI into the claims deployment pipeline | Platform team with GitLab access |
| GitLab CI variable store access | To add `POSTMAN_API_KEY` as a GitLab CI variable | GitLab admin |
| Real `X-API-Key` values per environment | To populate environment variables so collections can run | Claims service team |
| OAuth2 client credentials (client_id, client_secret) | For machine-to-machine token exchange in collections | Claims service team |
| Spec fix: `$ref` schemas + 5xx responses | Required before full collection generation can run | Claims engineering team |
| GitLab runner network access confirmation | Postman CLI on the runner must reach claims API endpoints | Infrastructure/platform team |

---

## Operational Implications

**Short term (before spec is fixed):** The claims spec is in Spec Hub and is discoverable. Engineers can find the API, see its contract, and understand its endpoints. But there are no generated collections to run against it and no automated tests in the CI/CD pipeline.

**After spec is fixed:** Re-run the claims workflow with `onboarding-scope: full`. Full collection generation runs, collections are committed to the repo, and the monitor activates.

**GitLab CI integration:** This is a separate working session with the claims platform team. The deliverable is a `.gitlab-ci.yml` snippet that runs `postman collection run` with the smoke and contract collections on every deploy. The collection IDs are already in `postman/collections/`. The platform team adds the Postman CLI step to their existing pipeline — it does not require rebuilding their CI/CD.

---

## Key Takeaway

The onboarding pattern transfers directly. The GitHub Actions workflow for claims is structurally identical to payment refund — the only differences are spec path, domain label, environment tiers, and two flag changes driven by the spec quality and CI/CD system differences. 

The harder work is not the Postman automation — it is the platform-team coordination required to wire the GitLab CI pipeline and fix the spec. Those are dependencies this engagement surfaces and documents; they are not blockers to getting the spec into the catalog and discoverable today.
