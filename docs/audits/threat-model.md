# Threat Model — aws-sam-typescript-bootstrap @ fb17c74

**Date**: 2026-05-16
**Source-of-truth DFD**: [`../architecture/dfd.md`](../architecture/dfd.md)
**Workload class**: PHI (operator-declared — HIPAA + GDPR Article 9)
**Verdict**: **FAIL** (2 critical, 5 high, 5 medium; score 0/100)

> The STRIDE walk below iterates each trust-boundary crossing recorded in the DFD's § "Trust boundaries" table. The DFD already flagged three boundary violations (unauthenticated PHI endpoint, PHI in plaintext logs, broken winston file transport); they reappear here mapped to their STRIDE category, severity, and concrete mitigation.

## Attack surface

| Counted from the DFD | Value |
|---|---|
| Entry points | 1 (`GET /patients/{patientId}/appointments` — unauthenticated) |
| Processes | 2 (API Gateway, Lambda) |
| Data stores | 1 in-process mock + 1 broken local-file path + CloudWatch Logs (managed) |
| External integrations | 3 AWS managed services (CloudWatch Logs, X-Ray, Application Insights) |
| Boundary crossings | 7 (3 already flagged as violations in the DFD) |

## Threats by STRIDE category

| # | Category | Threat | Severity | Entry point / crossing | Mitigation |
|---|---|---|---|---|---|
| T1 | Spoofing + Info Disclosure | Anonymous GET on PHI endpoint — any caller can fetch any patient's appointments by guessing UUIDs (IDOR / BOLA) | CRITICAL | `client → API Gateway` | Add API Gateway authorizer: Cognito user pool, Lambda authorizer with JWT verification, OR IAM-signed requests. Bind `Events.GetAppointments` with `Auth: { Authorizer: ... }` in `template.yaml`. Enforce `patientId == caller_subject_or_authorized_for(patient)` in handler before calling `appointmentService` |
| T2 | Information Disclosure | PHI + PII written in plaintext to CloudWatch Logs — `logger.info('Received event', { event })` dumps full APIGatewayProxyEvent (headers, sourceIp, requestContext) and `logger.info('Appointments retrieved', { patientId, count })` writes the patientId | CRITICAL | `Lambda → CloudWatch Logs` | (1) Remove the `event` payload from the receive log: log only request id + method + path. (2) Hash/tokenise `patientId` in logs (`sha256(patientId+pepper)`). (3) Enable CloudWatch log-group KMS encryption + a 30-day retention policy (current default is "Never Expire"). (4) Add a CloudWatch Logs subscription filter scrubbing leftover PHI patterns. (5) Sign a BAA with AWS for HIPAA workloads |
| T3 | Information Disclosure + Spoofing | CORS `Access-Control-Allow-Origin: *` (and Headers/Methods `*`) on a PHI endpoint — once auth is added, malicious origins can still read responses if cookie/header credentials leak | HIGH | `client → API Gateway` | Replace `*` with the explicit allowed origin list. If only first-party callers use this API, drop the CORS headers entirely (CORS only matters to browsers; server-to-server callers ignore them). Lives at `src/command/lambda/patient-appointments.ts:7-11` |
| T4 | Denial of Service | No throttling — API Gateway default account-level limits only; no per-method or per-caller throttle. PHI endpoint + no rate limit = trivial scraping or DoS | HIGH | `client → API Gateway` | Add `Auth.UsagePlan` (api keys + plan) OR `MethodSettings` with throttling in `template.yaml`. Suggested: 10 RPS per caller, 100 burst. Pair with CloudWatch alarm on `4XXError` and `Throttle` metrics |
| T5 | Information Disclosure + Tampering | Lambda runtime is Node.js 18 (EOL'd 2025-04, in AWS Lambda deprecation). Unpatched runtime = known-CVE exposure. For a PHI workload this is a compliance issue, not just hygiene | HIGH | `Lambda runtime` | Bump `Runtime: nodejs18.x` → `nodejs20.x` (or 22) in `template.yaml`. Bump `@types/node` to match. Re-run `sam build` to confirm `Target: es2020` esbuild config still works |
| T6 | Spoofing + Elevation of Privilege | CI uses static long-lived AWS access keys (`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` in `cd.yml`). If leaked (PR artifact log, fork build, repo compromise), attacker gets deploy rights to the AWS account. Note: `.github/aws-oidc-config.yaml` exists, indicating intent to migrate, but the workflow never reads it | HIGH | `GitHub → AWS` | Migrate to OIDC: use `aws-actions/configure-aws-credentials@v4` with `role-to-assume: <IAM-role-ARN>` and remove the two static secrets. Pre-existing `.github/aws-oidc-config.yaml` is the starting point. Scope the IAM role narrowly: only `cloudformation:*`, `lambda:*`, `iam:PassRole` on the function role, `s3:*` on the deploy bucket |
| T7 | Information Disclosure | `Tracing: Active` in `template.yaml` Globals — X-Ray captures invocation metadata. If handler code later adds X-Ray subsegment annotations (`AWSXRay.captureFunc(...)`), patientId can land in trace spans visible to anyone with X-Ray read access | MEDIUM | `Lambda → X-Ray` | Define a project rule: never pass patientId (or any PHI field) to X-Ray annotations. Use opaque correlation IDs for trace context. Document in the project README. Today's code doesn't do this — preventive, not remediation |
| T8 | Information Disclosure | 500 response body returns `err.message` — leaks internal error details to anonymous callers (`src/command/lambda/patient-appointments.ts:42-44`). For a PHI workload, even framing details (DB driver names, env paths) help attackers | MEDIUM | `Lambda → client (error path)` | Return `{ "message": "Internal error", "requestId": "<lambda-request-id>" }`. Log the full `err` server-side with the request id so support can correlate without exposing it to callers |
| T9 | Repudiation | Winston `File` transports target `error.log` + `combined.log` in the Lambda working directory — Lambda's root filesystem is read-only outside `/tmp`. Writes fail silently or error. If these were intended as a redundant audit trail, the audit trail doesn't exist | MEDIUM | `Lambda → local fs` (broken flow) | Remove the `transports.File(...)` entries from `src/utils/logger.ts`. CloudWatch Logs via the console transport is the authoritative audit log; the file transports add nothing in Lambda and lull operators into thinking there's a redundant trail |
| T10 | Denial of Service + Elevation of Privilege | (a) Lambda has no reserved concurrency — a runaway caller can amplify cost / starve other functions. (b) Execution role is unbounded in `template.yaml` (only `CAPABILITY_IAM` declared; no explicit `Policies` block) — defaults to the SAM-implicit role with broader-than-needed permissions | MEDIUM | `Lambda execution context` | (a) Set `ReservedConcurrentExecutions: 50` (or appropriate) on the function. (b) Add an explicit `Policies:` block restricting to only the AWS APIs the function actually calls (CloudWatch Logs `PutLogEvents`, X-Ray `PutTraceSegments`). Drop any default permissions |
| T11 | Information Disclosure | CloudWatch Application Insights resource (`ApplicationInsightsMonitoring` in `template.yaml`) consumes CloudWatch logs and surfaces in dashboards/alarms. Inherits the T2 problem: as long as PHI lands in logs, it lands in Insights dashboards (and any alarm body / SNS topic Insights writes to) | MEDIUM | `CloudWatch → Application Insights` | Resolves once T2 is fixed (redact at the source). Until then: scope IAM read access to the Application Insights dashboard and any SNS topic it publishes to |
| T12 | Tampering + Info Disclosure | Dependencies are 1-3 majors behind (TS 4.8, ESLint 8.8, prettier 2.x, @typescript-eslint 5.x) with no `npm audit` in CI. For a PHI workload, an unpatched transitive CVE is a compliance breach surface | HIGH | Supply chain | Run `/audit-deps aws-sam-typescript-bootstrap` to enumerate concrete CVEs. Add `npm audit --audit-level=high` to a CI job; add Dependabot (or Renovate) for automated PRs. Re-baseline once green |

**Summary**: 12 threats — **2 critical, 5 high, 5 medium, 0 low**. Score: 0/100. Verdict: **FAIL**.

## Recommended priority

Fix in this order — earlier items unblock or amplify later items:

1. **T1 — Add authentication to the endpoint** (CRITICAL). Without this, every other mitigation is treating symptoms of "PHI is publicly readable".
2. **T2 — Remove PHI from logs** (CRITICAL). Independent of T1; even with auth the log breach is its own compliance violation.
3. **T3 — Lock down CORS** (HIGH). Cheap; do alongside T1.
4. **T6 — Migrate CI to OIDC** (HIGH). Independent change with material upside; the config file is already in the repo.
5. **T5 — Bump Lambda runtime to Node 20/22** (HIGH). Coordinated with `@types/node` bump.
6. **T4 — Add API Gateway throttling** (HIGH). Once auth lands, throttle per-caller; before then, throttle per source IP.
7. **T12 — Run `/audit-deps` and add CI audit gate** (HIGH).
8. **T9 — Remove broken file transports** (MEDIUM, trivial). One-line fix.
9. **T8 — Sanitise error responses** (MEDIUM). Three-line fix.
10. **T10 — Reserved concurrency + scoped IAM** (MEDIUM).
11. **T7 — Document the X-Ray-no-PHI rule** (MEDIUM, preventive).
12. **T11 — Resolves transitively with T2** (no separate work).

## OWASP cross-check

| OWASP item | Finding | Status |
|---|---|---|
| A01 — Broken Access Control | T1 — no auth on PHI endpoint → IDOR | **OPEN (critical)** |
| A02 — Cryptographic Failures | T2 — PHI in plaintext logs; CloudWatch log group not KMS-encrypted | **OPEN (critical)** |
| A03 — Injection | No DB queries, no shell-out, no eval. `patientId` from URL is used as filter key against in-memory mock only | N/A |
| A04 — Insecure Design | Architecture allows anonymous access to PHI; missing auth was a design omission, not an implementation bug | OPEN (subsumed under T1) |
| A05 — Security Misconfiguration | T3 (CORS `*`), T6 (static CI keys), T10 (unbounded IAM) | **OPEN (high)** |
| A06 — Vulnerable Components | T5 (Node 18 EOL), T12 (deps 1-3 majors behind, no audit gate) | **OPEN (high)** |
| A07 — Auth Failures | T1 again — no auth, no rate limit on a sensitive endpoint | OPEN (subsumed under T1 + T4) |
| A08 — Software/Data Integrity Failures | T6 — static CI keys allow code-supply-chain compromise via deploy | OPEN (subsumed under T6) |
| A09 — Logging Failures | T2 (PHI in logs), T9 (broken file transports → silent audit gap) | **OPEN (medium-high)** |
| A10 — SSRF | No outbound HTTP from handler. Not applicable | N/A |

Net: **OWASP Top 10 — 6 of 10 categories with open findings**, 2 critical.

## Trend

`< 2 prior runs — trend section silent on first run.` See `/threat-model --help` for re-run cadence.
