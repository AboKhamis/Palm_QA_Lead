# Key Acceptance Criteria

## AC1: API Key Lifecycle Management

### Creation:

✅ User can create a new API key from dashboard UI   
✅ System generates cryptographically secure key (min 32 chars, format: `sk_{env}_{random}`)   
✅ Secret displayed once in plain text after creation (with copy button)   
✅ After dismissal, secret permanently masked (shows last 4 chars `sk_live_****...xyz789`)   
✅ User can assign key to 1+ proxy products during creation (multi-select)   

### Viewing & Labeling:

✅ Keys list shows: name, masked secret, assigned products, environment, status, created date   
✅ User can edit key label/name (changes persist after page refresh)   
✅ User can search by key name or last 4 digits of secret   
✅ Pagination works correctly (default 50 per page, supports up to 100/page)

### Revocation:

✅ User can revoke key via dashboard (confirmation modal: "This action cannot be undone")   
✅ Revoked key returns HTTP 401 Unauthorized for new proxy requests within 60 seconds globally   
✅ In-flight requests at time of revocation complete successfully (graceful, not killed)   
✅ Revoked keys move to "Revoked" tab (not deleted - audit trail preserved)   
✅ Clear error message: `{"error": "API key revoked", "revoked_at": "2025-12-02T14:30:00Z"}`

---

## AC2: Proxy Product Assignment

✅ Keys can be assigned to multiple products: Residential, Premium, Datacenter, Dedicated   
✅ Proxy requests route to correct pool based on key assignment:
- Key assigned to Residential → request uses residential IP
✅ If key not assigned to requested product → return 403 Forbidden: 
"API key not authorized for Residential proxies"   
✅ Product assignment changes take effect within 30 seconds (cache invalidation)   

---

## AC3: Rate Limiting

### Configuration:
✅ User can set custom rate limit per key
✅ Rate limit can be different per environment (e.g., dev: 100/min, prod: 10,000/min)   
### Enforcement:
✅ Rate limit accuracy: actual requests allowed = configured limit ± 2%   
✅ When limit exceeded, return HTTP 429 Too Many Requests with headers:
```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1701529800 (Unix timestamp)
Retry-After: 45 (seconds)
```
✅ Burst protection: 1000 req/min limit does NOT allow 1000 requests in 1 second (distribute across window)   
✅ Rate limit shared across all sources using same key (not per-IP)   

---

## AC4: IP Whitelisting

### Configuration:
✅ Supports CIDR notation (54.123.0.0/16) and wildcards: `54.123.45.*` 
✅ Empty whitelist = allow from any IP (default, clearly documented in UI)   
✅ IP whitelist can be different per environment
### Enforcement:
✅ Request from whitelisted IP → allowed (proceed to rate limit check)
✅ IP validation happens BEFORE rate limit check (fail fast)   
✅ Non-whitelisted IP → **403 Forbidden** with: {"error": "IP 203.0.113.5 not whitelisted"}

---

## AC5: Usage & Activity Logging

✅ Dashboard metrics updated **every 60 seconds**  
✅ Activity log shows **last 1000 requests** or **last 7 days** (whichever fewer)  
✅ Logs include: timestamp, source IP, exit IP/country, status code, response time  
✅ **CSV export** available (max 10,000 rows per export)

---

## AC6: Environment Isolation

### Strict Separation:
✅ Key tagged `dev` **cannot access** `prod` proxy pools  
✅ Environment mismatch → **403 Forbidden**: `dev-key not authorized for production` 
✅ Customer specifies environment via endpoint (dev-proxy.company.com) or header (X-Environment: prod)
### Per-Environment Configuration:
✅ Same key can have **different rate limits** per environment:
- Example: key_abc → 100 req/min (dev), 10,000 req/min (prod)
✅ Same key can have **different IP whitelists** per environment:
- Example: key_abc → allow 192.168.* (dev), allow 54.123.45.67 (prod)
✅ **Configuration inheritance**: Default applies to all environments unless overridden  
✅ **Audit logging**: Environment mismatch attempts logged with IP, timestamp, key name

---

## AC7: High-Scale Performance

✅ **Dashboard loads in <5 seconds** with 100k keys (with pagination)  
✅ **API validation completes in <50ms** at p95 (even with 150k keys in workspace)  
✅ **Search/filter returns results in <2 seconds** on 100k dataset  
✅ **No race conditions**: 1000 concurrent requests → rate limit enforced correctly
