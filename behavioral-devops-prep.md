# Behavioral DevOps Interview Prep — Senior Engineer (11 yrs)

Use **STAR** (Situation, Task, Action, Result) and always end with **what you learned or improved** to show seniority. Quantify impact (downtime, cost, MTTR) when you can.

---

## Story bank (fill once — use this to pick the right example under pressure)

| # | Question theme        | My go-to example (one line) |
|---|------------------------|-----------------------------|
| 1 | Production outage      | ____________________________ |
| 2 | Deployment failure     | ____________________________ |
| 3 | Rollback               | ____________________________ |
| 4 | Terraform issue        | ____________________________ |
| 5 | Cost reduction         | ____________________________ |
| 6 | Monitoring alert       | ____________________________ |
| 7 | Long CI/CD debug       | ____________________________ |
| 8 | K8s scaling            | ____________________________ |
| 9 | Mistake + learning     | ____________________________ |
|10 | First improvement      | ____________________________ |

---

## 1. What was the last production outage you handled? What was the root cause?

**Senior differentiator:** You led triage, communicated with stakeholders, and drove the postmortem and preventive changes — not just “I helped fix it.”

**What they want:** Incident ownership, calm under pressure, clear root-cause thinking, and follow-up (postmortem, prevention).

**Answer framework:**
- **Situation:** One sentence — what broke, who noticed, severity (e.g. “API down for 15 min, 50% error rate”).
- **Task:** Your role (e.g. on-call, lead responder).
- **Action:** How you triaged (logs, metrics, recent deploys), how you mitigated (rollback, scale, feature flag, cache clear), and how you found root cause.
- **Result:** Time to detect / mitigate / resolve (MTTD, MTTR), and **what you changed** (alerting, runbooks, tests, config change, postmortem, blameless culture).

**Your example (fill in):**
- Outage: _______________________________________
- Root cause: _______________________________________
- What you did to fix: _______________________________________
- What you improved after: _______________________________________

---

## 2. What deployment failure did you debug recently?

**Senior differentiator:** You systematically compared working vs failing (env, config, version) and added a guard so it doesn’t happen again.

**What they want:** Debugging method, use of pipeline logs and env-specific checks, not just “we restarted.”

**Answer framework:**
- **What failed:** Stage (build / test / deploy), environment (e.g. prod only).
- **How you debugged:** Pipeline logs, agent logs, diff between working vs failing env (config, secrets, permissions, versions), network/firewall, dependency or artifact issue.
- **Fix:** Concrete change (e.g. env var, RBAC, pipeline step, retry/timeout).
- **Prevention:** Validation step, pre-deploy check, or documentation.

**Your example (fill in):**
- Failure: _______________________________________
- Debug steps: _______________________________________
- Fix: _______________________________________
- Prevention: _______________________________________

---

## 3. Have you ever rolled back a release? Why?

**Senior differentiator:** You made the call (or recommended it), executed it safely, and improved the process (runbook, canary, feature flags).

**What they want:** You’ve done rollbacks, you know when it’s the right call, and you do it in a controlled way.

**Answer framework:**
- **Why:** Bug in new release, bad config, failed migration, performance regression — one clear reason.
- **How:** Automated rollback (pipeline “rollback” job), redeploy previous artifact, traffic shift (blue/green, canary), or DB migration rollback.
- **Result:** Service restored in X minutes; no data loss (or how you handled it).
- **Lesson:** Feature flags, canary, better tests, or rollback runbook.

**Your example (fill in):**
- Reason for rollback: _______________________________________
- How you rolled back: _______________________________________
- What you improved: _______________________________________

---

## 4. What was the last Terraform issue you fixed?

**Senior differentiator:** You fixed state/provider/lifecycle issues yourself and improved how the team runs Terraform (backend, locking, CI).

**What they want:** Real IaC experience — state, provider, or resource lifecycle issues, not just “I wrote Terraform.”

**Answer framework:**
- **Issue:** State drift, state lock, provider version mismatch, resource already exists, destroy order, `cycle` in graph, or policy/validation failure.
- **How you fixed it:** `terraform state` commands, `-target`, state mv/rm, backend/config change, or code refactor.
- **Prevention:** State in remote backend with locking, CI validation (plan in PR), consistent provider versions, modules.

**Your example (fill in):**
- Issue: _______________________________________
- Fix: _______________________________________
- Prevention: _______________________________________

---

## 5. How did you reduce infrastructure cost in your project?

**Senior differentiator:** You owned the initiative, took concrete actions, and can cite a number (%, $) and ongoing controls (tagging, alerts, reviews).

**What they want:** Ownership of cost, concrete actions, and measurable outcome.

**Answer framework:**
- **What you did:** Right-sizing (CPU/memory), reserved/spot instances, scaling policies (scale-to-zero, schedule), removing unused resources, cleaning storage/lifecycle, optimizing queries or caching to reduce DB/compute.
- **Result:** “X% reduction in spend” or “$Y per month saved” or “Z% better utilization.”
- **Ongoing:** Tagging, dashboards, budgets, alerts, FinOps reviews.

**Your example (fill in):**
- Actions: _______________________________________
- Result (%, $, or utilization): _______________________________________
- How you keep it under control: _______________________________________

---

## 6. What monitoring alert did you personally configure?

**Senior differentiator:** You designed and implemented it (threshold, runbook, avoiding alert fatigue), not just “we use the default alerts.”

**What they want:** You don’t only use existing alerts; you design and implement them.

**Answer framework:**
- **What you alerted on:** Latency, error rate, saturation (CPU/memory/disk), specific business or dependency failure, cost anomaly.
- **Where:** Azure Monitor, Prometheus/Grafana, Datadog, PagerDuty, etc.
- **Design:** Threshold, window, severity, runbook link, avoidance of alert fatigue (aggregation, correlation).
- **Outcome:** Earlier detection, fewer false positives, or clearer on-call response.

**Your example (fill in):**
- Alert: _______________________________________
- Tool: _______________________________________
- Why this threshold/design: _______________________________________

---

## 7. Which CI/CD failure took you the longest to debug? Why?

**Senior differentiator:** You explain why it was hard (flakiness, env-specific, tooling) and what you changed so the team doesn’t hit it again.

**What they want:** Persistence, methodical debugging, and what made it hard (flakiness, env-specific, tooling).

**Answer framework:**
- **What failed:** Intermittent vs consistent, which stage.
- **Why it was long:** Flaky tests, race conditions, “works on my machine,” poor logging, multiple systems involved, or rare edge case.
- **How you solved it:** Isolating variables, extra logging, replay, narrowing to one agent/env/version, or fixing root cause in code/config.
- **Lesson:** Better observability, isolation, retry policy, or test stability improvements.

**Your example (fill in):**
- Failure: _______________________________________
- Why it took long: _______________________________________
- How you solved it: _______________________________________
- What you improved: _______________________________________

---

## 8. What scaling issue did you face in Kubernetes?

**Senior differentiator:** You tuned real levers (HPA, requests/limits, node sizing) and can discuss the trade-off (cost vs performance, stability).

**What they want:** Hands-on K8s — HPA, resource limits, bottlenecks, or platform limits.

**Answer framework:**
- **Issue:** Pods OOMKilled, node pressure, slow scale-up, throttling, single-point bottlenecks (no replicas), or cluster/quota limits.
- **What you did:** Tuned requests/limits, HPA metrics (CPU/custom), PDB, node pool sizing, or app-level scaling/caching.
- **Result:** Stable under load, predictable scaling, or cost vs performance trade-off.
- **Lesson:** Right-sizing, testing under load, or observability.

**Your example (fill in):**
- Issue: _______________________________________
- Actions: _______________________________________
- Result: _______________________________________

---

## 9. Tell me one mistake you made in production and what you learned.

**Senior differentiator:** A real mistake, clear impact, and concrete process/tech changes you introduced — no humble-brag.

**What they want:** Honesty, ownership, and concrete learning — not a humble-brag.

**Answer framework:**
- **Mistake:** One clear example (wrong config, skipped step, assumed state, manual change without backup).
- **Impact:** Brief — what broke or risked.
- **What you did:** How you fixed it and communicated.
- **What you learned:** Process (checklist, approval, automation), technical (safety checks, rollback), or culture (blameless, postmortem).

**Your example (fill in):**
- Mistake: _______________________________________
- Impact: _______________________________________
- What you learned: _______________________________________

---

## 10. If I join your project today, what would you improve first in the infrastructure?

**Senior differentiator:** One specific, high-impact improvement with a clear “first 30–60 days” plan — shows you think like an owner and can onboard someone.

**What they want:** You see gaps and prioritize; you think like an owner and can onboard someone with a clear first win.

**Answer framework:**
- Pick **one** high-impact, feasible improvement (e.g. observability, cost visibility, security baseline, runbooks, DR test, or tech debt in IaC).
- **Why first:** Risk reduction, reliability, or developer experience; tie to business impact if you can.
- **What you’d do:** First 30–60 days — audit, small pilot, document, or automate one flow.
- Keep it specific and honest (e.g. “our runbooks are outdated” or “we don’t run chaos drills”).

**Your example (fill in):**
- First improvement: _______________________________________
- Why: _______________________________________
- First steps: _______________________________________

---

## Quick checklist before the interview

- [ ] Have 1–2 concrete examples per question (with numbers if possible).
- [ ] Rehearse 2–3 min per answer; avoid rambling.
- [ ] For outages/rollbacks/mistakes: always end with “what we improved.”
- [ ] Sound like an owner: “I drove…”, “I implemented…”, “I reduced…” without dismissing team.
- [ ] If you don’t have a perfect example, say what you’d do and relate to a similar situation.

Good luck — you’ve got the experience; this structure will help you deliver it clearly.
