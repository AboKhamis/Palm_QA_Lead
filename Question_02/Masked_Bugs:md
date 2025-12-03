

## Cascading Bug Analysis (Bug #4)

### Primary Bug: Revoked key still works in APAC regions

### How It Masks/Causes Hidden Issues

#### Hidden Issue 1: Billing Discrepancies

**Connection:**

- Revoked key continues to generate proxy traffic for 5-15 minutes
- Usage meters count this traffic
- Customer billed for "unauthorized" usage

**Symptom:**

- Customer revokes key at 10:00 AM, sees usage spike until 10:15 AM
- Support ticket: "I revoked this key, why am I still being charged?"
- Billing team disputes charges (manual refunds)

**Why it's hidden:**

- Small time window (15 min) - easy to miss in monthly bills
- Attributed to "normal usage lag" (not investigated)
- Billing system doesn't correlate revocation timestamp with usage

**Impact:**

- Revenue leakage (refunds)
- Customer trust erosion ("They're overcharging me")
- Support overhead (manual billing adjustments)

**Detection:**

- Cross-reference revocation logs with usage logs
- Alert if usage recorded >5 min after revocation
- Billing audit report: "Orphaned usage (revoked keys)"

---

#### Hidden Issue 2: Activity Log Corruption (Explains Bug #2)

**Connection:**

- **Bug #2:** "Activity log stays empty after revoke + retry"
- **Root cause:** Same cache issue as Bug #4
- Activity log service queries cache for key status before logging
- Cache says "revoked" → log service skips writing entry (assumes invalid request)
- But proxy API cache (different region) says "active" → request succeeds
- **Result:** Successful request with no log entry

**Symptom:**

- Customer revokes key, immediately tests it
- Request succeeds (Bug #4) but no log entry (Bug #2)
- Customer: "Your logs are broken, my request worked but doesn't show up"

**Why it's hidden:**

- Looks like two separate bugs (dashboard UI vs. proxy API)
- Actually same root cause (cache inconsistency across services)
- Investigating Bug #2 in isolation wouldn't find proxy cache issue

**Impact:**

- Compliance failure (incomplete audit logs)
- Customer can't verify if revoked key is actually blocked
- Forensics impossible ("Was my key used after revocation?")

**Detection:**

- Compare proxy access logs with activity log DB
- Alert if proxy logs show request but activity log empty
- Reconciliation job: Identify missing log entries

---

#### Hidden Issue 3: Fraud Detection False Negatives

**Connection:**

- Fraud detection system monitors for "revoked key usage" (indicator of compromise)
- System queries cache for key status
- Cache says "active" (stale) → fraud system ignores activity
- Actual attacker using stolen revoked key goes undetected

**Symptom:**

- Customer revokes key at 10:00 AM (compromised)
- Attacker uses key from APAC region 10:00-10:15 AM
- Fraud system doesn't flag (cache says "active")
- Customer sees suspicious usage in logs (hours later)

**Why it's hidden:**

- Fraud alerts only trigger for "revoked key usage"
- If cache says active, no alert generated
- By the time cache refreshes, attacker moved on
- Appears as "normal usage" (not fraud)

**Impact:**

- Security breach (stolen keys go undetected)
- Customer liability (attacker racks up charges)
- Reputational damage ("Your fraud detection missed this")

**Detection:**

- Cross-check revocation timestamp with access logs
- Alert if usage spike immediately after revocation
- Fraud detection should query DB, not cache (for revoked status)

---

#### Hidden Issue 4: Cache Stampede on Revocation

**Connection:**

- When cache finally expires (15 min TTL), all APAC servers fetch from DB simultaneously
- If customer has 10k keys, and many expire at once → DB overload
- Database slow queries → dashboard timeout (Bug #1 worsens)

**Symptom:**

- Dashboard loading becomes slower after mass revocations
- DB metrics spike (connection pool exhausted)
- Cascading failures (Redis down → everyone hits DB)

**Why it's hidden:**

- Appears as separate performance issue
- Not obviously connected to cache TTL
- Happens during "peak usage" (assumed normal load)

**Impact:**

- Dashboard instability (Bug #1 severity increases)
- Database outages (affecting all customers)
- Requires expensive DB scaling (not fixing root cause)

**Detection:**

- Monitor DB query patterns (spike every 15 min?)
- Correlate with cache expiry events
- Alert if DB connections spike after TTL window

---

### Summary: Why Bug #4 is Especially Dangerous

**Single root cause (cache inconsistency) manifests as:**

1. Security issue (revoked keys work)
2. Billing issue (unexpected charges)
3. Logging issue (missing entries)
4. Fraud detection failure (attackers undetected)
5. Performance issue (cache stampede)

**QA Lesson:**

- Distributed systems bugs rarely exist in isolation
- Cache inconsistency is a "multiplier bug" (affects many systems)
- Fix priority should consider cascading impact, not just primary symptom
- Post-fix: Audit all systems relying on key status cache

**Recommended Actions:**

1. Fix Bug #4 (cache invalidation)
2. Audit billing logs (refund orphaned usage)
3. Verify Bug #2 resolves (activity logs)
4. Review fraud detection logic (query DB, not cache)
5. Add cache monitoring (detect regional inconsistency)
6. Implement graceful degradation (cache miss → query DB)

---

## Summary

### Bug Triage Priorities:

- **P0:** Security-critical (revoked keys)
- **P1:** Enterprise blockers (dashboard loading)
- **P2:** Quality/velocity (flaky tests, logs)
- **P3:** Cosmetic (typos)

### Workflow Principles:

- Support → QA triage → Dev → QA validation → Release
- Keep stakeholders informed (Slack, Jira, email)
- Hotfix P0, schedule P1-P3

### AI Assistance Risks:

- Overly optimistic assumptions (timeouts, single-region)
- Missing edge cases (distributed systems)
- **Prevent with:** code review, architecture docs, regional testing

### Cascading Bugs:

- Single cache issue → 5 different symptoms
- Always investigate: "What else relies on this component?"
