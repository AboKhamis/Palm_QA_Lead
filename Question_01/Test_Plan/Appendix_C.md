# Exploratory Testing Angles (Proxies / Rate Limits / Abuse)

## **🌐 Proxies**

🔍 **Proxy pool failover:** Residential pool exhausted → error doesn't expose internal infrastructure (IPs, server names)
🔍 **Wrong product routing:** Key assigned to Residential, requests Datacenter → 403 with clear error (not generic 500)
🔍 **Concurrent proxy usage:** Same key from 50 different source IPs simultaneously → rate limit still shared correctly
🔍 **Geo-routing restrictions:** Key restricted to US proxies → non-US request blocked with clear error
🔍 **Session persistence:** Same session ID across requests → verify returns same exit IP for duration
🔍 **Pool switching speed:** Change key assignment Residential → Datacenter → new requests route correctly within 30s
🔍 **Proxy authentication failure:** Key sent in wrong format (header vs auth string) → clear error message
🔍 **Exit IP blacklisted:** Target site blocks proxy IP → how is error communicated to customer?


## **⏱️ Rate Limits** 

🔍 **Distributed rate limiting:** Request hits Server A (900/1000 used) → immediate request to Server B → Server B enforces remaining 100 limit (shared state)
🔍 **Burst traffic:** 10,000 requests in 10 seconds → rate limiter doesn't crash, 429 responses consistent
🔍 **Rate limit bypass attempts:** Client manipulates X-RateLimit-Remaining header → server ignores (server-side enforcement only)
🔍 **Concurrent requests:** 1000 simultaneous requests on same key → rate limit enforced correctly, no race conditions
🔍 **Rate limit reset boundary:** Exactly at midnight UTC → counter resets correctly, no off-by-one errors
🔍 **Multiple rate limits:** Key has per-minute AND per-day limits → both enforced independently
🔍 **Clock skew impact:** Servers with 5-second time difference → rate limit calculations still accurate

## **🚨 Abuse Scenarios**

🔍 **Key sharing detection:** Same key used from 100 different IPs in 1 minute → anomaly flagged in logs/alerts?
🔍 **Rapid revoke/recreate:** Create key → revoke → create → revoke (50 cycles) → system remains stable
🔍 **Rate limit gaming:** Customer creates 1000 keys, 1 request each → bypassing per-key limits (workspace-level limit needed?)
🔍 **Credential stuffing:** 10,000 failed auth attempts with similar key patterns → rate limited at authentication layer?
🔍 **DDoS (Distributed Denial of Service) with valid key:** 100k req/sec using valid key → system doesn't fall over, rate limit enforced
🔍 **Revoke during active attack:** Key being abused → revoke → propagates globally within 60s (under load)
🔍 **IP spoofing attempts:** Requests claim different source IPs via headers → real IP validated server-side
🔍 **Replay attacks:** Same request signature sent 1000 times → idempotency enforced (or detected as suspicious)?

## **🚧 Environment Isolation Edge Cases**

🔍 **Key tagged dev+staging**: can it access prod if someone misconfigures routing?   
🔍 **Environment config inheritance bug**: delete prod override → does it really revert to default or break?   

## **🔒 Security & Data Integrity**

🔍 **Secret exposure:** Error messages/logs never show plain-text key (always masked)
🔍 **SQL injection:** Key name = '; DROP TABLE api_keys; -- → input sanitized
🔍 **Transaction rollback:** Database connection lost mid-key-creation → no partial data committed
🔍 **Environment spoofing:** Dev key sends X-Environment: prod header → server rejects
🔍 **Cross-environment data leakage:** Dev workspace logs don't show prod request data
🔍 **Usage log batch job fails:** missing data in dashboard? Retry logic?
🔍 **Cache invalidation bug**: revoke key → cache still shows as active for 10 minutes  

## **📊 High-Scale Behavior** (Enterprise Critical)

🔍 **Dashboard memory leak:** 150k keys in workspace → browser doesn't crash, stays <500MB memory
🔍 **Workspace key limit:** Create key when workspace already has 150k → is limit enforced or unlimited?
🔍 **CSV export at scale:** Export 150k keys → doesn't timeout (streams response or paginates export)
🔍 **Rate limiter stress test:** 10k keys all hitting rate limit simultaneously → rate limiter service doesn't crash
