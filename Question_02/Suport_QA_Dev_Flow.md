

## Support → QA → Dev → Release Workflow

### Workflow Table

| Stage                    | Owner        | Actions                                                                                                                                                                                                    | Exit Criteria                                                                                              | Notifications                                                                                           |
| ------------------------ | ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| **1. Support Intake**    | Support      | • Customer reports issue via ticket  <br>• Initial triage: Severity (Critical/High/Medium/Low)  <br>• Gather: Steps to reproduce, account ID, environment  <br>• Tag: "needs-qa-triage"                    | • Issue logged in Jira  <br>• Basic info collected (what/when/who)                                         | • Slack: #support-escalations (P0/P1 only)  <br>• Email: QA Lead (P0 only)                              |
| **2. QA Triage**         | QA Lead      | • Validate reproducibility (staging/prod)  <br>• Assign priority (P0-P3)  <br>• Classify: Bug/Feature/Config issue  <br>• Create detailed bug ticket (if confirmed)  <br>• Attempt workaround for customer | • Priority assigned  <br>• Reproduction steps documented  <br>• Logs/screenshots attached                  | • Slack: #qa-team (all bugs)  <br>• Slack: #dev-team (P0/P1)  <br>• Jira: Mention Support rep           |
| **3. Dev Investigation** | Dev Team     | • Root cause analysis  <br>• Estimate fix effort (hours/days)  <br>• Propose solution + rollback plan  <br>• Implement fix + unit tests  <br>• Self-test in dev environment                                | • Fix implemented  <br>• Unit tests pass  <br>• Code reviewed (peer review)  <br>• Ready for QA validation | • Slack: Mention QA Lead when ready  <br>• Jira: Status → "Ready for QA"                                |
| **4. QA Validation**     | QA Team      | • Test fix in staging (original repro steps)  <br>• Regression testing (related flows)  <br>• Edge case validation  <br>• Performance check (if applicable)  <br>• Approve or reject fix                   | • Original bug fixed ✅  <br>• No new regressions ✅  <br>• Automated test added ✅                           | • Slack: #qa-team (if rejected)  <br>• Jira: Status → "Approved" or "Failed QA"                         |
| **5. Release Decision**  | QA Lead + PM | • P0: Hotfix (immediate deploy)  <br>• P1: Next release (within 48h)  <br>• P2/P3: Scheduled release  <br>• Create release notes  <br>• Notify Support with workarounds                                    | • Release approved  <br>• Rollback plan documented  <br>• Stakeholders notified                            | • Slack: #releases (all)  <br>• Email: Support team (workarounds)  <br>• Slack: Founder/Product (P0/P1) |
| **6. Post-Release**      | QA Lead      | • Monitor production (30 min - 24h)  <br>• Smoke tests on prod  <br>• Check Support tickets (issue resolved?)  <br>• Update bug ticket: "Deployed"                                                         | • No regressions detected  <br>• Customer confirms fix  <br>• Metrics stable                               | • Slack: #releases ("Fix deployed")  <br>• Jira: Close ticket  <br>• Email: Customer (via Support)      |

### Workflow Diagram

```mermaid
graph TD
    A[Customer reports issue] --> B[Support: Initial triage]
    B --> C{Severity?}
    C -->|P0/P1| D[Immediate Slack alert to QA Lead]
    C -->|P2/P3| E[Jira ticket created]
    D --> F[QA Lead: Reproduce + Prioritize]
    E --> F
    F --> G{Confirmed bug?}
    G -->|Yes| H[Create detailed bug ticket]
    G -->|No| I[Close as: Config/User error/Not reproducible]
    I --> J[Notify Support with resolution]
    H --> K[Assign to Dev team]
    K --> L[Dev: Fix + Unit tests]
    L --> M[QA: Validate in staging]
    M --> N{Fix approved?}
    N -->|No| L
    N -->|Yes| O{Priority?}
    O -->|P0| P[Hotfix deploy immediately]
    O -->|P1| Q[Deploy within 48h]
    O -->|P2/P3| R[Schedule for next release]
    P --> S[Monitor prod + Smoke tests]
    Q --> S
    R --> S
    S --> T[Notify Support + Customer]
    T --> U[Close ticket]
```

### Key Principles

**When Support hands to QA:**

- **P0/P1:** Immediately via Slack (within 15 min)
- **P2/P3:** Daily digest (end of day)
- Always include: Account ID, environment, reproduction steps

**When QA pulls in Dev:**

- After confirming reproducibility (don't waste dev time on false alarms)
- With detailed bug ticket (logs, steps, expected vs actual)
- **P0:** Immediate (Slack mention)
- **P1:** Within 2 hours
- **P2/P3:** Next standup

**Before fix goes live:**

- QA approval required (test in staging)
- Automated test added (prevent regression)
- Rollback plan documented (especially P0 hotfixes)
- Smoke tests ready (run post-deploy)

**Keeping stakeholders in loop:**

**Support:** Jira updates (auto-notify), workarounds shared immediately

**Founder/Product:**

- **P0:** Immediate Slack + status updates every 2 hours until resolved
- **P1:** Daily summary ("Bug X in progress, ETA 48h")
- **P2/P3:** Weekly bug summary

**Customers (via Support):**

- Workaround provided within 4 hours (if available)
- Fix ETA shared ("We expect this fixed by [date]")
- Follow-up when deployed ("This issue is now resolved")
