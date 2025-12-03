# Core Test Cases

## UI Tests

✅ Create API key → verify secret shown once with copy button, then permanently masked   
✅ Label/rename key → verify persistence across page refresh and browser sessions   
✅ Revoke key → confirmation modal appears, key moves to Revoked tab, status updated 
✅ Assign key to multiple products → checkboxes update, assignments persist   
✅ Set environment tags → badges display correctly, filtering works   
✅ Configure per-environment rate limits → UI shows overrides clearly ("Default: 1000 | Prod: 50,000")   
✅ Configure per-environment IP whitelists → changes reflected in config panel   
✅ Pagination with 1k+ keys → loads <2s, "Next" button works correctly   
✅ Search by key name → results appear instantly, highlighting matches   
✅ Usage log displays correct metrics (requests, errors, IP, country, timestamp)   
✅ Filter keys by: product, status, environment, date range → combinations work   
✅ Export usage logs as CSV → file downloads with correct data

---

## API Tests

#### Authentication & Authorization:

✅ POST /api/keys → returns 201 + unique key in format `sk_{env}_{random}`  
✅ POST /api/keys with duplicate name → returns 409 Conflict (if name uniqueness enforced) OR allows (if names can duplicate)   
✅ POST /api/keys with invalid characters in name → returns 400 Bad Request   
✅ GET /api/keys → pagination, filtering, sorting work correctly   
✅ GET /api/keys/{id} → returns key details (secret masked)   
✅ PATCH /api/keys/{id} → update label, rate limit, IP whitelist, environment tags   
✅ DELETE /api/keys/{id} → key revoked, returns 204, sets revoked_at timestamp   
✅ Revoked key used in proxy request → returns 401 within 60s globally

#### Proxy Routing:

✅ Key assigned to Residential → proxy request routes to Residential pool (verify exit IP is residential)   
✅ Key assigned to Datacenter → proxy request routes to Datacenter pool (verify exit IP is datacenter)   
✅ Key assigned to BOTH Residential + Datacenter → header `X-Proxy-Type: residential` routes correctly   
✅ Key NOT assigned to Premium → request with `X-Proxy-Type: premium` returns 403

#### Rate Limiting:

✅ 100 req/min limit → send 100 requests → all succeed, 101st returns 429   
✅ 429 response includes correct headers: `X-RateLimit-Remaining: 0`, `Retry-After: <seconds>`  
✅ After rate limit window resets → requests allowed again   
✅ Burst test: send 1000 requests in 1 second with 1000/min limit → verify burst protection (not all allowed instantly)   
✅ Concurrent requests from same key → rate limit shared (not per-source-IP)   
✅ Different keys with different limits → limits enforced independently

#### IP Whitelisting:

✅ Request from whitelisted IP → allowed (200 OK)   
✅ Request from non-whitelisted IP → 403 Forbidden within 100ms   
✅ IPv4 CIDR notation (`54.123.0.0/16`) → IPs in range allowed, outside blocked   
✅ IPv6 address → validated correctly   
✅ Wildcard (`54.123.45.*`) → matches 54.123.45.0 to 54.123.45.255   
✅ Empty whitelist → allows any IP (default behavior)

#### Environment Isolation:

✅ Dev key used on `https://dev-proxy.company.com` → success   
✅ Dev key used on `https://proxy.company.com` (prod) → 403 Forbidden   
✅ Prod key used in dev → 403 Forbidden (if environments strictly isolated)   
✅ Key with both dev+prod tags → can access both environments   
✅ Per-environment rate limit: dev (100/min) vs prod (10k/min) → both enforced correctly   
✅ Per-environment IP whitelist: dev allows `192.168.*`, prod allows `54.123.45.67` → both enforced   
✅ Environment mismatch logged in audit trail with timestamp and source IP

#### Usage Logging:

✅ GET /api/keys/{id}/usage → returns accurate counts (requests, errors, bandwidth)   
✅ Usage stats handle missing data gracefully (no errors if key never used)   
✅ Real-time stats update within 60 seconds of request   
✅ Historical data available for last 30 days minimum  (or what is determined)
✅ Logs show correct exit IP, country, target domain


---

## Data & Integration Tests

✅ Key creation writes to DB with: workspace_id, product_ids[ ], environment[ ], created_at, rate_limit config   
✅ Secret stored as hash (bcrypt/argon2 or any hashing algorithm), never plain text in DB   
✅ Revocation sets revoked_at timestamp, does NOT delete record (audit trail) 
✅ Product assignment stored in junction table (keys_products)
✅ Environment tags stored as array or separate junction table   
✅ Usage log aggregates from proxy request logs (batch job runs every 5 minutes)   
✅ IP whitelist changes propagate to all proxy servers 
✅ Soft delete on revocation (revoked_at NOT NULL) vs hard delete

---

## Performance & Load Tests 🚀 (Critical for Enterprise)

**Context:** With 50k–150k keys per workspace, these tests validate the system won't collapse under real-world enterprise load.
#### **UI Performance (Worst-Case Scale)**

✅ **Dashboard loads in <5s** with 150k keys (using pagination)  
✅ **Search returns results in <2s** across 100k keys  
✅ **Virtual scrolling** implemented (renders only visible rows, not all 150k)  
✅ **No memory leaks**: Sustained browsing session stays <500MB

---
#### **API Performance (p95 Latency)**

✅ **Key validation <50ms** at p95 (even with 150k keys in workspace)  
✅ **POST /api/keys <300ms** (key creation)  
✅ **GET /api/keys <3s** with pagination for 150k keys  
✅ **DELETE /api/keys <200ms** (revocation API call)

---
#### **Critical Path Validation**

✅ **Revocation propagates globally within 60 seconds** (distributed cache invalidation)  
✅ **Rate limit accuracy ±2%** under 10k req/sec across 1000 different keys  
✅ **No race conditions**: 1000 concurrent requests on same key → rate limit enforced correctly

---
#### **Stress & Concurrency**

✅ **Sustained load**: 5000 req/min for 4 hours → no performance degradation  
✅ **Concurrent operations**: 500 reads + 100 writes + 50 deletes simultaneously → no errors  
✅ **Workspace isolation**: 150k-key workspace doesn't slow down 100-key workspace

---
#### **Data Volume Handling**

✅ **Usage log query**: Key with 1M historical requests loads in <2s (paginated)  
✅ **CSV export**: 10k keys exports in <30s without browser freeze

---

