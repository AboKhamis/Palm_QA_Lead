
# Test Plan: API Keys & Proxy Pools Management

**Feature:** Team API key management with proxy pool routing, rate limiting, and environment isolation   
**Enterprise Scale:** 50k–150k keys per workspace   
**Focus Areas:** Proxies, Rate Limits, Abuse Scenarios

---

## 📋 Key Acceptance Criteria

### AC1: API Key Lifecycle Management

- **Creation:** Secure key generation (`sk_{env}_{random}`), secret shown once, multi-product assignment
- **Viewing & Labeling:** Masked display, search/filter, pagination
- **Revocation:** Confirmation modal, 401 within 60s globally, audit trail preserved (not deleted)

### AC2: Proxy Product Assignment

- Multi-product support (Residential, Premium, Datacenter, Dedicated)
- Routing based on assignment, 403 if unauthorized
- Changes propagate within 30s

### AC3: Rate Limiting

- **Configuration:** Custom limits per key, per-environment overrides
- **Enforcement:** ±2% accuracy, sliding window, burst protection, 429 with headers, shared across IPs

### AC4: IP Whitelisting

- **Configuration:** CIDR + wildcard support, empty = allow all, per-environment
- **Enforcement:** Validation before rate limit, 403 within 100ms for non-whitelisted

### AC5: Usage & Activity Logging

- Real-time metrics (60s refresh), last 1000 requests or 7 days
- CSV export (max 10k rows), detailed logs (timestamp, IPs, status, response time)

### AC6: Environment Isolation

- **Strict Separation:** Dev cannot access prod, 403 on mismatch
- **Per-Environment Config:** Different rate limits & IP whitelists per env, config inheritance
- **Audit Logging:** Environment mismatch attempts logged

### AC7: High-Scale Performance

- Dashboard <5s with 100k keys, API validation <50ms p95
- Search/filter <2s, no race conditions under load
- Supports 150k keys per workspace

📚 **See Appendix A for detailed acceptance criteria**

---

## 🧪 Core Test Coverage

| Layer                       | Tests | Key Areas                                                                                       |
| --------------------------- | ----- | ----------------------------------------------------------------------------------------------- |
| UI Tests                    | 12    | Key creation, revocation, multi-product assignment, per-env config, pagination, search, export  |
| API - Auth & Lifecycle      | 8     | CRUD operations, key format validation, revocation propagation                                  |
| API - Proxy Routing         | 4     | Product-based routing, exit IP validation, unauthorized access                                  |
| API - Rate Limiting         | 6     | Limit enforcement, 429 responses, window resets, burst protection, concurrency                  |
| API - IP Whitelisting       | 6     | CIDR/wildcard validation, IPv4/IPv6, default behavior, performance                              |
| API - Environment Isolation | 7     | Cross-env access control, per-env configs, audit logging                                        |
| API - Usage Logging         | 5     | Metrics accuracy, real-time updates, historical data, export                                    |
| Data & Integration          | 8     | Secret hashing, soft delete, product assignment storage, cache propagation                      |
| Performance Tests           | 16    | UI responsiveness (150k keys), API latency, revocation propagation, stress testing, concurrency |

**Total:** 68 functional + 16 performance = 80+ structured test cases

📚 **See Appendix B for detailed test cases**

---

## 🔍 Exploratory Testing (Proxies, Rate Limits, Abuse)

| Category                     | Scenarios | Key Risks Explored                                                                               |
| ---------------------------- | --------- | ------------------------------------------------------------------------------------------------ |
| 🌐 Proxies                   | 8         | Pool failover, routing errors, geo-restrictions, session persistence, exit IP blacklisting       |
| ⏱️ Rate Limits               | 7         | Distributed counting, burst attacks, clock skew, cache failures, boundary conditions             |
| 🚨 Abuse Scenarios           | 8         | Key sharing, credential stuffing, DDoS, rate limit gaming, replay attacks, revocation under load |
| 🚧 Environment Isolation     | 2         | Cross-env breaches, header spoofing, config inheritance bugs, data leakage                       |
| 🔒 Security & Data Integrity | 7         | Secret exposure, SQL injection, transaction rollback, cache invalidation bugs                    |
| 📊 High-Scale Behavior       | 4         | Memory leaks at 150k keys, workspace limits, CSV export timeouts, rate limiter stress            |

**Total:** 36 exploratory scenarios covering edge cases, security vulnerabilities, and distributed system challenges

📚 **See Appendix C for detailed exploratory scenarios**

---

## 📌 Appendices Reference

- **Appendix A:** Detailed Acceptance Criteria (Full specifications for AC1-AC7)
- **Appendix B:** Core Test Cases (Complete test scenarios: UI, API, Data, Performance)
- **Appendix C:** Exploratory Testing (All 35 scenarios with detailed descriptions)

---
