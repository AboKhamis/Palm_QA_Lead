
# Automation Priority
## Context

**Team:** 2 QA Engineers (1 Senior + 1 Junior)   
**Timeline:** 1 day (8 hours each = 16 total hours)   
**Scope:** API Keys & Proxy Pools Management (enterprise scale: 50k–150k keys)   
**Constraints:** Limited time, must focus on highest-impact tests that prevent critical regressions

---

## Automation Strategy

### Time Allocation (16 Total Hours)

|Activity|Time|Owner|Notes|
|---|---|---|---|
|Test framework setup|2h|Senior|API client, auth helpers, test data factories|
|Priority 1-2 automation|6h|Senior|Revocation + Rate Limiting (most complex)|
|Priority 3-4 automation|5h|Junior|Environment Isolation + IP Whitelisting (guided by Senior)|
|Priority 5 automation|2h|Junior|Proxy Routing (straightforward)|
|Code review + fixes|1h|Both|Pair review, fix issues|

**Total: 16 hours**

---

## Top 5 Test Cases for Automation (Priority Order)

### Priority 1: API Key Revocation Propagation 🔴 CRITICAL

**Test Case:**

```
GIVEN an active API key being used for proxy requests
WHEN the key is revoked via DELETE /api/keys/{id}
THEN within 60 seconds globally:
  - New proxy requests return 401 Unauthorized
  - Error message: {"error": "API key revoked", "revoked_at": "<timestamp>"}
  - In-flight requests complete successfully (graceful handling)
  - Key status in DB: revoked_at timestamp set
  - Key moves to "Revoked" tab in UI
```

**Why Automate First:**

- **Critical path:** Revocation is a security feature (compromised keys must be blocked immediately)
- **High regression risk:** Distributed cache invalidation across proxy servers (complex)
- **Scale impact:** With 150k keys, cache invalidation bugs could affect many customers
- **Frequent use:** Security incidents require immediate revocation
- **Complexity:** Tests distributed system behavior (cache propagation, global state)

**Automation Approach:**

- API test: Revoke key → poll proxy endpoint every 5s for 60s → verify 401
- Verify in-flight request completes (start long request, revoke mid-flight, verify success)
- Check DB: revoked_at timestamp set correctly

**Estimated Time:** 3 hours (Senior QA)

---

### Priority 2: Rate Limit Enforcement 🔴 CRITICAL

**Test Case:**

```
GIVEN a key with rate limit of 100 requests/minute
WHEN 101 requests are sent within 60 seconds
THEN:
  - First 100 requests return 200 OK
  - 101st request returns 429 Too Many Requests
  - Response headers include:
    * X-RateLimit-Limit: 100
    * X-RateLimit-Remaining: 0
    * X-RateLimit-Reset: <unix_timestamp>
    * Retry-After: <seconds>
  - Rate limit accuracy: ±2% (98-102 requests allowed)
  - Burst protection: 100 req/min ≠ 100 req in 1 second
  - Limit shared across all source IPs using same key
```

**Why Automate Second:**

- **Billing impact:** Incorrect rate limiting = revenue loss or customer overcharges
- **High frequency:** Every API request checks rate limit (regression would affect all customers)
- **Distributed system complexity:** Rate limit state shared across servers (Redis/cache)
- **Scale emphasis:** With 150k keys and 10k req/sec, accuracy is critical
- **Regression risk:** Clock skew, cache failures, race conditions can break enforcement

**Automation Approach:**

- API test: Send exactly 100 requests → verify all succeed
- Send 101st → verify 429 with correct headers
- Burst test: Send 100 requests in 1 second → verify some are rejected (sliding window)
- Concurrent test: 10 threads, same key → verify shared limit (not per-thread)

**Estimated Time:** 3 hours (Senior QA)

---

### Priority 3: Environment Isolation 🟠 HIGH

**Test Case:**

```
GIVEN a key tagged with environment "dev"
WHEN the key is used to access production proxy endpoint
THEN:
  - Request returns 403 Forbidden
  - Error message: {"error": "dev key not authorized for production"}
  - Attempt logged in audit trail (timestamp, IP, key name)

AND per-environment config enforcement:
GIVEN key_abc with:
  - Dev: rate limit 100/min, IP whitelist 192.168.*
  - Prod: rate limit 10,000/min, IP whitelist 54.123.45.67
WHEN requests are made from different environments
THEN each environment enforces its own config independently
```

**Why Automate Third:**

- **Security critical:** Dev key accessing prod = major security breach (data leakage, unauthorized access)
- **New emphasis in test plan:** Added 4 edge cases for environment isolation (config inheritance, cross-env breaches)
- **Complex caching/routing:** Environment routing involves cache, headers, endpoint logic
- **High regression risk:** Config inheritance bugs could break production overrides
- **Can be automated at API level:** Clear pass/fail (403 vs 200)

**Automation Approach:**

- API test: Create dev key → call prod endpoint → verify 403
- Test per-env rate limits: dev key hits 100/min limit, prod key hits 10k/min limit
- Test per-env IP whitelists: dev allows 192.168.*, prod allows specific IP
- Test config inheritance: Delete prod override → verify reverts to default (not null)

**Estimated Time:** 3 hours (Junior QA with Senior guidance)

---

### Priority 4: IP Whitelist Validation 🟠 HIGH

**Test Case:**

```
GIVEN a key with IP whitelist: ["54.123.45.67", "54.123.0.0/16"]
WHEN requests are made from different IPs:
  - 54.123.45.67 → 200 OK (exact match)
  - 54.123.100.50 → 200 OK (CIDR range match)
  - 203.0.113.5 → 403 Forbidden within 100ms
  - Error: {"error": "IP 203.0.113.5 not whitelisted"}

AND validation happens BEFORE rate limit check (fail fast)
AND empty whitelist = allow all IPs (default behavior)
AND wildcard support: 54.123.45.* matches 54.123.45.0-255
```

**Why Automate Fourth:**

- **Security critical:** Prevents unauthorized access from non-whitelisted IPs
- **High frequency:** Every request validates IP (regression affects all customers)
- **Performance requirement:** Must fail within 100ms (fail-fast before rate limit)
- **Complexity:** CIDR notation, wildcards, IPv4/IPv6 validation
- **Regression risk:** IP parsing bugs, cache issues, default behavior changes

**Automation Approach:**

- API test: Whitelist specific IP → request from that IP → 200 OK
- Request from non-whitelisted IP → 403 within 100ms
- Test CIDR notation: 54.123.0.0/16 allows 54.123.x.x
- Test wildcard: 54.123.45.* allows 54.123.45.0-255
- Test empty whitelist → allows any IP
- Verify IP validation happens before rate limit (non-whitelisted IP doesn't consume rate limit)

**Estimated Time:** 2 hours (Junior QA)

---

### Priority 5: Proxy Product Routing 🟡 MEDIUM

**Test Case:**

```
GIVEN a key assigned to "Residential" proxy product
WHEN a proxy request is made
THEN:
  - Request routes to Residential proxy pool
  - Exit IP is from residential range (verify via IP lookup)
  - Response includes X-Proxy-Type: residential header

GIVEN a key NOT assigned to "Premium" product
WHEN a request is made with X-Proxy-Type: premium header
THEN:
  - Request returns 403 Forbidden
  - Error: {"error": "API key not authorized for Premium proxies"}

GIVEN a key assigned to BOTH Residential + Datacenter
WHEN request includes X-Proxy-Type: datacenter header
THEN request routes to Datacenter pool (not Residential)
```

**Why Automate Fifth:**

- **Core product functionality:** Incorrect routing = customer gets wrong proxy type (billing/quality issue)
- **Customer-facing:** Routing errors directly impact customer experience
- **Moderate complexity:** Product assignment lookup, pool routing logic
- **Regression risk:** Product assignment changes, cache invalidation
- **Straightforward to automate:** Clear pass/fail (exit IP type verification)

**Automation Approach:**

- API test: Create key assigned to Residential → make request → verify exit IP is residential (use IP geolocation API)
- Create key NOT assigned to Premium → request Premium → verify 403
- Create key with multiple products → use header to specify product → verify correct routing
- Test product assignment change propagates within 30s (cache invalidation)

**Estimated Time:** 2 hours (Junior QA)

---

## Why These 5 (Not Others)?

### ✅ Included:

Revocation, Rate Limits, Environment Isolation, IP Whitelisting, Proxy Routing

**Reasoning:** Critical path, high regression risk, security-critical, high frequency, can be automated at API level

### ❌ Excluded (for now):

**High-Scale Performance Tests (150k keys, dashboard load times):**

- **Why not:** Performance/load tests require different tooling (JMeter, k6, Locust)
- **Why not day 1:** Infrastructure setup takes longer, run less frequently (nightly/weekly)
- **When:** Automate after core functional tests, run in CI/CD pipeline separately

**UI Tests (dashboard, CSV export, search/filter):**

- **Why not:** UI tests are slower, more brittle, require Selenium/Playwright setup
- **Why not day 1:** API tests give better ROI (faster, more stable, easier to maintain)
- **When:** Automate critical UI flows after API coverage is solid

**Exploratory Scenarios (abuse, edge cases, chaos):**

- **Why not:** Exploratory tests are manual by nature (unpredictable, creative)
- **When:** Some can be automated later (e.g., burst traffic, concurrent requests) but not day 1 priority

---

## Success Criteria (End of Day 1)

### Deliverables:

✅ 5 automated test suites (one per priority) running in CI/CD   
✅ Test framework with reusable helpers (API client, auth, test data)   
✅ Documentation: README with setup instructions, how to run tests   
✅ Code review completed: Senior reviews Junior's code, fixes merged

### Coverage:

✅ Critical path covered: Revocation, rate limits, environment isolation   
✅ Security covered: IP whitelisting, environment isolation   
✅ Core functionality covered: Proxy routing

### Quality Metrics:

✅ All 5 test suites pass on first CI/CD run   
✅ No flaky tests: Tests are deterministic, repeatable   
✅ Fast execution: All 5 suites run in <5 minutes total

---

## Risk Mitigation

### Risk 1: Junior QA struggles with complex tests

- **Mitigation:** Senior pairs with Junior on Priority 3 (Environment Isolation) for first hour
- **Mitigation:** Junior starts with Priority 5 (Proxy Routing - straightforward) to build confidence

### Risk 2: API endpoints not ready/unstable

- **Mitigation:** Use mocks/stubs for unavailable endpoints
- **Mitigation:** Focus on contract testing (expected request/response format)

### Risk 3: Time overruns

- **Mitigation:** If behind schedule, drop Priority 5 (Proxy Routing) - less critical than others
- **Mitigation:** Timebox each priority (hard stop at allocated hours)

### Risk 4: Environment/infrastructure issues

- **Mitigation:** Use Docker containers for consistent test environment
- **Mitigation:** Senior sets up framework first (2h) so Junior can start coding immediately

---

## Tools & Framework

**Language:** Python
**Framework:** pytest (Python)
**API Client:** requests (Python)
**Assertions:** pytest assertions
**CI/CD:** GitLab CI   
**Reporting:** Allure or pytest-html

---
