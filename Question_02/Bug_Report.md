

## Detailed Bug Ticket (Bug #4 - P0)

### Title

**[P0] Revoked API keys still authenticate successfully when used from APAC regions**

### Severity

**P0 - Critical (Security)**

### Customer Impact

**Who's affected:** Enterprise customers with revoked keys, especially those using proxies for compliance-sensitive scraping (finance, healthcare)

**Impact:** Revoked keys continue to work for 5-15 minutes in APAC regions (Singapore, Tokyo, Sydney), allowing unauthorized proxy usage after revocation

**Business risk:**

- Security breach (compromised keys not immediately disabled)
- Compliance violation (customers rely on instant revocation for SOC2/GDPR)
- Unexpected billing (revoked keys generate usage charges)
- Trust erosion ("Your revoke button doesn't work")

### Environment

**Affected:** Production (confirmed), Staging (reproduced)   
**Regions:** APAC proxy POPs (Singapore, Tokyo, Sydney)   
**Not affected:** US/EU regions (revocation works instantly)   
**Browser/Client:** Any (API-level issue, not frontend)

### Steps to Reproduce

**1. Create API key in dashboard:**

```
- Navigate to "API Keys" page
- Click "Create new key"
- Assign to "Residential" product
- Copy secret key: sk_live_abc123...
```

**2. Use key from APAC region:**

```bash
# From Singapore IP (203.0.113.45)
curl -x http://proxy.company.com:8080 \
     -H "Proxy-Authorization: Bearer sk_live_abc123..." \
     https://httpbin.org/ip

# Response: {"origin": "residential_ip_address"}
# ✅ Works as expected
```

**3. Revoke key in dashboard:**

```
- Click "Revoke" button next to key
- Confirm revocation modal
- Key status changes to "Revoked" (gray badge)
```

**4. Immediately retry request from same APAC IP:**

```bash
curl -x http://proxy.company.com:8080 \
     -H "Proxy-Authorization: Bearer sk_live_abc123..." \
     https://httpbin.org/ip

# Expected: 401 Unauthorized
# Actual: {"origin": "residential_ip_address"} ✅ Still works!
```

**5. Wait 10-15 minutes and retry:**

```bash
# Now returns: 401 Unauthorized {"error": "API key revoked"}
# ✅ Eventually works, but delayed
```

**6. Compare with US/EU region:**

```bash
# Same test from US IP immediately after revocation
# Returns: 401 Unauthorized (instant)
# ✅ Works correctly in US/EU
```

### Expected vs Actual Behavior

**Expected:**

- Revoked API key returns `401 Unauthorized` immediately in all regions
- Error response: `{"error": "API key has been revoked", "code": "key_revoked"}`
- Applies globally within 1-2 seconds (cache invalidation propagates)

**Actual:**

- Revoked key continues to authenticate successfully in APAC regions for 5-15 minutes
- Returns successful proxy response (200 OK with residential IP)
- Eventually returns 401 after cache TTL expires

### Logs / Evidence

**Dashboard logs (revocation event):**

```json
{
  "event": "api_key.revoked",
  "key_id": "key_abc123",
  "workspace_id": "ws_enterprise_001",
  "timestamp": "2024-03-15T10:23:45Z",
  "user": "customer@example.com"
}
```

**Proxy API logs (APAC - Singapore POP):**

```json
{
  "timestamp": "2024-03-15T10:23:50Z",  // 5 seconds after revocation
  "key_id": "key_abc123",
  "status": "authenticated",  // ❌ Should be "revoked"
  "region": "ap-southeast-1",
  "cache_hit": true,  // 🔍 Cache returned stale data
  "cache_ttl_remaining": 840  // 14 minutes remaining
}
```

**Redis cache inspection (Singapore):**

```bash
redis-cli -h singapore-redis.internal
GET api_key:key_abc123:status
"active"  # ❌ Stale! Should be "revoked"
TTL api_key:key_abc123:status
782  # 13 minutes remaining
```

**Sample API call:**

```bash
curl -v -x http://proxy.company.com:8080 \
     -H "Proxy-Authorization: Bearer sk_live_abc123..." \
     https://httpbin.org/ip
```

**Response headers:**

```
HTTP/1.1 200 OK
X-Proxy-Region: ap-southeast-1
X-Cache-Status: HIT  // 🔍 Served from cache
```

**Screenshot:**

- Dashboard shows key status: "Revoked" ✅
- Proxy logs show successful auth ❌
- Timestamp mismatch proves lag



