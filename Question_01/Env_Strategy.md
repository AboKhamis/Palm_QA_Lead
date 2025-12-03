
# Environment Strategy: Dev/Stage/Prod

---

## Test Data Strategy

|Environment|API Keys|Proxy Pools|Usage Logs|Data Management|
|---|---|---|---|---|
|**Dev**|Auto-generated via factory (10–100 keys)|Mock pools (no real proxies)|Minimal (last 7 days)|Ephemeral, reset nightly|
|**Staging**|Mix of static + dynamic (1k–10k keys)|Real proxy pools (limited quota)|Realistic volume (30 days)|Persistent, cleaned weekly|
|**Prod**|Read-only tests only (5-10 dedicated keys)|Real pools (dedicated test quota)|Full production data|No writes, use isolated test workspace|

---

## Key Principles

### Dev

**Focus:** Unit + integration, fast feedback, no external dependencies

- Fast test execution (<5 min)
- Mocked external services (proxies, third-party APIs)
- Local database (PostgreSQL/SQLite)
- Rate limiting disabled (easier debugging)
- Full CRUD operations (create/delete keys freely)

**Why:** Enable developers to iterate quickly without waiting for staging or consuming real resources.

### Staging

**Focus:** Full E2E, realistic scale, catches integration bugs

- Real proxy connections (limited quota: 10k requests/day)
- Realistic data volume (10k keys, 500k logs)
- Rate limiting enabled (matches prod)
- Mix of static baseline data + dynamic test data
- Tests run on main/develop branches (not every PR)

**Why:** High-confidence validation before production deployment. Catches bugs that mocks miss.

### Prod

**Focus:** Smoke tests only (health checks, key validation), no destructive actions

- Read-only operations (no create/delete)
- Pre-created test keys (manually provisioned)
- Isolated test workspace (separate from customers)
- Dedicated proxy quota (1k requests/day)
- Runs post-deployment only

**Why:** Validate deployment succeeded without risking production data or customer experience.

---

## Config & Secrets Management

### Dev Configuration

**.env.dev** (committed to repo, no secrets)

```bash
API_URL=http://localhost:5000
PROXY_URL=http://mock-proxy:8080
DB_HOST=localhost
DB_PASSWORD=dev_password  # Safe to commit (local only)
RATE_LIMIT_ENABLED=false  # Easier debugging
REAL_PROXY_ENABLED=false  # Use mocks
```

### Staging Configuration

**.env.staging** (stored in GitLab Secrets, NOT committed)

```bash
API_URL=https://staging-api.company.com
PROXY_URL=https://staging-proxy.company.com
DB_HOST=staging-db.internal
DB_PASSWORD=${STAGING_DB_PASSWORD}  # GitLab Secret
RATE_LIMIT_ENABLED=true
API_ADMIN_KEY=${STAGING_API_ADMIN_KEY}  # GitLab Secret
```

### Production Configuration

**.env.prod** (stored in 1Password + GitLab Secrets, NOT committed)

```bash
API_URL=https://api.company.com
PROXY_URL=https://proxy.company.com
DB_HOST=prod-db-replica.internal  # Read-only replica
DB_PASSWORD=${PROD_DB_READONLY_PASSWORD}  # 1Password vault
RATE_LIMIT_ENABLED=true
API_ADMIN_KEY=${PROD_API_READONLY_KEY}  # Read-only scope
TEST_WORKSPACE_ID=workspace_smoke_tests  # Isolated
```

### Secrets Handling

**Never commit secrets to repo:**

- Use `.env.example` templates (committed, no real values)
- Add `.env.*` to `.gitignore`
- Pre-commit hooks to detect secrets

**Storage:**

- **Dev:** Local files (no real secrets)
- **Staging:** GitLab CI/CD Variables (masked, protected)
- **Prod:** 1Password vault (primary) + GitLab Secrets (synced)

**Rotation:**

- **Staging secrets:** Monthly (automated via Terraform)
- **Prod secrets:** Monthly (manual, requires approval)
- **Test API keys:** Quarterly

**Audit:**

- Log all prod secret access (who, when, what)
- Alert if secret accessed outside CI/CD pipeline

---

## Environment-Specific Risks

### Dev

#### ⚠️ Mock proxy pools don't catch real routing bugs

**Impact:** Bugs slip to staging (delays release)

**Mitigation:**

- Run nightly "staging-like" suite in dev with real proxy dependencies
- Weekly manual testing against staging

#### ⚠️ Rate limiter disabled → might miss 429 edge cases

**Impact:** Production customers hit unexpected rate limits

**Mitigation:**

- Dedicated rate limit test suite with `RATE_LIMIT_ENABLED=true`
- Run rate limit tests on every PR (even in dev)

### Staging

#### ⚠️ Shared environment → other teams' tests exhaust proxy quota

**Impact:** Test failures due to "Quota exceeded" (false negatives)

**Mitigation:**

- Monitor quota usage (alert at 80%)
- Dedicated staging cluster for load tests (separate quota)
- Coordinate heavy test runs with other teams

#### ⚠️ High-scale tests (100k keys) impact performance for everyone

**Impact:** Staging slowdown affects other teams

**Mitigation:**

- Run high-scale tests only on weekends (nightly Saturday)
- Use dedicated cluster for performance tests
- Cleanup immediately after test (don't leave 100k keys)

### Prod

#### ⚠️ Test workspace might interfere with real customer traffic (shared rate limits)

**Impact:** Customer requests get 429 errors due to test traffic

**Mitigation:**

- Isolated test workspace with separate rate limit pool (critical)
- Conservative smoke tests (100 requests/day max)
- Monitor customer complaints for false positives

#### ⚠️ Revoked key tests could trigger fraud alerts

**Impact:** Security team investigates false alarms (alert fatigue)

**Mitigation:**

- Whitelist test API keys in fraud detection system
- Whitelist test IPs (known CI/CD runner IPs)
- Document in fraud runbook: "Ignore `workspace_smoke_tests`"

#### ⚠️ Secrets leakage if test results logged to unsecured dashboards

**Impact:** Security incident, compliance violation

**Mitigation:**

- Sanitize all logs before storing:
- Mask API keys: `sk_live_REDACTED`
- Mask IPs: `203.0.*.*`
- Use pytest plugin with auto-redact formatter
- Separate Grafana dashboard for prod (restricted access, MFA)

---

## Summary

### Decision Matrix:

|Decision|Dev|Staging|Prod|
|---|---|---|---|
|Use real proxies?|No|Yes (limited)|Yes (dedicated)|
|Create/delete keys?|Yes|Yes|No|
|Database writes?|Yes|Yes|No (read-only)|
|Data scale?|Small (10-100)|Realistic (1k-10k)|Minimal (5-10)|
|When to run?|Every commit|Main/develop|Post-deployment|
|Primary goal?|Speed|Realism|Safety|

### Why this strategy:

- **Dev** optimized for **speed** (mocks, small scale)
- **Staging** optimized for **realism** (real proxies, realistic scale)
- **Prod** optimized for **safety** (read-only, isolated, no writes)

---
