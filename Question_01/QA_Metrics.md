

# QA Metrics & Dashboards: API Keys & Proxy Pools

---

## Overview

This document defines 3 core QA metrics that surface in CI pipelines and dashboards to drive data-driven decisions about quality, stability, and release readiness.

---

## Metric 1: Critical Flow Pass Rate

### Definition

**Formula:**

$$\text{Critical Flow Pass Rate} = \frac{\text{Passed Critical Flows}}{\text{Total Critical Flow Runs}} \times 100\%$$

### What qualifies as a "Critical Flow"?

End-to-end scenarios that validate core business functionality:

1. **API Key Lifecycle:** Create Key → Use in Proxy Request → Revoke → Verify 401
2. **Rate Limiting:** Create Key (1000/min limit) → Send 1000 requests → Verify all succeed → Send 1001st → Verify 429
3. **Environment Isolation:** Create Dev Key → Use on Prod Endpoint → Verify 403
4. **IP Whitelisting:** Create Key (whitelist: 203.0.113.5) → Request from whitelisted IP → Success → Request from other IP → Verify 403
5. **Proxy Routing:** Create Key (assigned: Residential) → Make request → Verify exit IP is residential → Request Premium → Verify 403

### Why This Metric?

**Revenue Protection:**

- If key creation fails → Customers can't onboard → Lost sales
- If revocation doesn't work → Security breach → Reputational damage
- If rate limiting breaks → Service abuse → Infrastructure costs spike

**Customer Trust:**

- IP whitelisting failure → Enterprise security requirements not met → Churn
- Proxy routing errors → Wrong product delivered → Contract violations

**Business Continuity:**

- These flows generate revenue (key creation = new customer)
- These flows prevent losses (rate limiting, revocation)
- Failure directly impacts bottom line

### Targets

|Environment|Target Pass Rate|Action Threshold|
|---|---|---|
|Main branch|≥99%|<99% → Block deployment|
|Feature branches|≥95%|<95% → Block merge|
|Production smoke tests|100%|<100% → Immediate rollback|

### How It Influences Decisions

#### Scenario 1: Feature Branch at 92% Pass Rate (Example)

**Situation:**

```
MR fails 2/25 critical flows:
  - test_key_revocation_propagation (Priority 1)
  - test_rate_limit_enforcement (Priority 2)
```

**Decision:**

- ❌ Block merge to develop branch
- 🚨 Assign to MR author with test failure details
- 📋 Require fix before re-review (no exceptions)

**Rationale:** These tests protect revenue and security - cannot compromise.

---

## Metric 2: Test Flakiness Rate

### Definition

**Formula:**

$$\text{Flakiness Rate} = \frac{\text{Failed runs that passed on retry}}{\text{Total test runs}} \times 100\%$$

### What is a "flaky" test?

A test that exhibits non-deterministic behavior - passes/fails without code changes.

**Common causes:**

- Race conditions (element not loaded yet in Selenium)
- Timing issues (hardcoded `sleep()` instead of explicit waits)
- External dependencies (staging DB latency, network flakiness)
- State pollution (test B fails if test A didn't clean up)

### Why This Metric?

**Developer Productivity:**

- Flaky tests waste time ("Is this failure real or just flaky?")
- Engineers lose trust in test suite → Ignore failures → Real bugs slip through
- CI/CD pipeline slowdown (re-running flaky tests)

**Decision Quality:**

- Can't make confident merge/deploy decisions if tests are unreliable
- False positives → Block good code
- False negatives → Ship broken code

**Technical Debt:**

- High flakiness indicates deeper issues (poor test design, unstable infrastructure)
- Accumulates over time if not addressed

### Targets

|Metric|Target|Action Threshold|
|---|---|---|
|Per-test flakiness|<3%|>5% → Quarantine test|
|Critical flow flakiness|<1%|>1% → Immediate investigation|
|Overall suite flakiness|<2%|>3% → Sprint planning item|

### How It Influences Decisions

#### Scenario 1: Individual Test at 8% Flakiness

**Situation:**

```
test_dashboard_load_100k_keys has 8% flakiness (failed 8/100 runs, passed on retry)
Pattern: 6/8 failures = Selenium TimeoutException
```

**Decision:**

- 🚧 Quarantine test immediately (mark as `@pytest.mark.skip` with reason)
- 📋 Assign to owner (test author or QA lead)
- ⏰ Deadline: Fix within 2 sprints or delete test

**Rationale:** 8% flakiness = test provides negative value (wastes time, erodes trust).

---

## Metric 3: Dashboard Regression Rate

### Definition

**Formula:**

$$\text{Dashboard Regression Rate} = \frac{\text{Dashboard tests that passed previously but now fail}}{\text{Total dashboard tests}} \times 100\%$$

### What qualifies as a "Dashboard Regression"?

Any previously passing UI/dashboard test that fails after a code change:

- API schema changes broke the UI (form expects `rate_limit` but API returns `rateLimit`)
- Frontend component bugs (button click doesn't trigger action)
- Selenium element selectors broken (UI refactor changed CSS classes/IDs)
- Dashboard performance degradation (page load timeout with 150k keys)

**Dashboard Test Categories:**

- **Key Management UI:** Create/revoke/edit API keys via dashboard
- **Usage Dashboard:** View logs, filter by date, export CSV
- **Configuration Pages:** IP whitelist, product assignment, rate limits
- **Performance/Scale:** Load dashboard with 100k keys, pagination, search

### Why This Metric?

**User Experience:**

- Dashboard is primary customer interface
- If users can't create keys via UI → Frustration → Support tickets → Churn
- If CSV export breaks → Enterprise customers blocked from reporting

**Revenue Impact:**

- Can't create new API keys → Onboarding blocked → Lost sales
- Can't configure rate limits → Wrong pricing tier → Revenue leakage

**Cross-Team Coordination:**

- Backend changes often break frontend unknowingly
- This metric forces coordination ("Your API change broke the dashboard")
- Catches integration issues early (before prod)

**Product-Specific Relevance:**

- This feature has heavy dashboard component (not just API)
- Many users prefer UI over API for key management
- Dashboard failures = highly visible to customers

### Targets

|Environment|Target Regression Rate|Action Threshold|
|---|---|---|
|Main branch|0%|>0% → Immediate investigation|
|Feature branches|≤2%|>2% → Block merge|
|Release candidate|0%|>0% → Block release|

**Why strict 0% on main/release?**  
Dashboard regressions directly impact customers (highly visible, breaks user workflows).

### How It Influences Decisions

#### Scenario 1: API Change Breaks Dashboard (15% Regression)

**Situation:**

```
MR: "Refactor API: rate_limit → rateLimit"

API tests: ✅ 100% pass
Dashboard tests: ❌ 15% regression (3/20 failed)
  - test_create_key_via_dashboard (field not found)
  - test_edit_key_rate_limit (API returns 400)
  - test_usage_dashboard_filters (schema mismatch)
```

**Decision:**

- ❌ Block merge immediately (>2% threshold)
- 🤝 Coordinate with frontend team:
- Option 1: Update frontend to use `rateLimit` (breaking change)
- Option 2: Add API backward compatibility (accept both formats)
- Option 3: Simultaneous deployment (backend + frontend together)
- 📋 Require fix before merge (no exceptions)

**Rationale:** Main branch regressions = production risk. Zero tolerance.

---

## Summary: How These Metrics Drive Decisions

### Critical Flow Pass Rate → Revenue & Security Protection

**When it drops:**

- <95% on feature branch → Block merge
- <99% on main → Block deployment
- Sudden drop → Freeze releases, investigate

**Why it matters:**  
Core business functionality = revenue generation and security compliance.

### Test Flakiness Rate → Developer Productivity & Trust

**When it rises:**

- >5% per test → Quarantine test
- >3% overall → Sprint planning item (address technical debt)
- Pattern detected → Investigate infrastructure

**Why it matters:**  
Flaky tests erode confidence and waste time. Unreliable tests = no tests.

### Dashboard Regression Rate → User Experience & Coordination

**When it rises:**

- >0% on main → Freeze deployments, investigate
- >2% on feature branch → Block merge
- API schema changes → Coordinate with frontend team

**Why it matters:**  
Dashboard is customer-facing. Regressions are highly visible and impact revenue.

---

## Why These 3 Metrics?

### Coverage:

- Backend functionality (Critical Flow Pass Rate)
- Frontend functionality (Dashboard Regression Rate)
- Test suite health (Flakiness Rate)

### Actionable:

- Clear thresholds (when to block merge/deployment)
- Clear ownership (who fixes what)
- Clear impact (revenue, UX, productivity)

### Product-Aligned:

- This feature has API + Dashboard components
- Critical flows protect revenue and security
- Dashboard regressions impact customer experience

### Balanced:

- Not too few (would miss important signals)
- Not too many (would create alert fatigue)
- Each metric serves distinct purpose

---
