# Behavioral & Leadership — Interview Scenarios

> 80 scenarios spanning conflict, failure, leadership, influence, ambiguity, and company-specific LP mapping.
> Format: Question → What's tested → How to frame your answer (not a script — framework + triggers)

---

## Amazon Leadership Principles Reference

Before each story, identify the primary LP:
1. Customer Obsession | 2. Ownership | 3. Invent and Simplify | 4. Are Right, A Lot
5. Learn and Be Curious | 6. Hire and Develop the Best | 7. Insist on the Highest Standards
8. Think Big | 9. Bias for Action | 10. Frugality | 11. Earn Trust | 12. Dive Deep
13. Have Backbone; Disagree and Commit | 14. Deliver Results | 15. Strive to Be Earth's Best Employer
16. Success and Scale Bring Broad Responsibility

---

## Part 1: Technical Leadership & Architecture Decisions

---

### B1. Walk me through the most complex infrastructure you've designed.
**Tests:** Depth of technical thinking, trade-off awareness, scale reasoning.
**Frame:** Start with the problem scale (users, QPS, data volume), drive through every architectural decision as an explicit choice with alternatives rejected, end with what you'd do differently.
**Must include:** Why you chose each component over alternatives. Cost or operational trade-off.

---

### B2. Tell me about a time you had to make a significant architectural change to an existing system.
**Tests:** Ownership, change management, risk mitigation, delivery.
**Frame:** Describe why the existing design was a bottleneck. Explain your migration strategy (strangler fig? big bang? parallel run?). Show how you managed risk and rollback.
**Must include:** How you got buy-in. What broke. How you recovered.

---

### B3. Describe a time you introduced a new technology or tool that had significant impact.
**Tests:** Invent and Simplify, Learn and Be Curious, Earn Trust.
**Frame:** Why did you choose it over alternatives? How did you de-risk the adoption? How did you get the team on board?
**Must include:** A specific measurable improvement (MTTR, deploy frequency, cost).

---

### B4. Tell me about a time you had to sunset or migrate away from a technology.
**Tests:** Long-term thinking, stakeholder management, execution discipline.
**Frame:** What drove the change (cost, security, scale limits)? How did you manage the transition with zero/minimal downtime? What was the business impact?

---

### B5. Tell me about a time you solved a very difficult production problem that others couldn't solve.
**Tests:** Dive Deep, problem-solving under pressure, technical credibility.
**Frame:** Don't make others look bad. Focus on the complexity of the problem. Show your systematic debugging approach (what you eliminated, what tools you used, what the "a-ha" moment was).

---

### B6. Describe a time you drove a major reliability improvement.
**Tests:** Insist on the Highest Standards, Ownership, Deliver Results.
**Frame:** What was the baseline? (E.g., "We had 3–4 P1 incidents per month, MTTR was 4 hours.") What did you change? (Observability, runbooks, on-call process, architecture?). What was the result?
**Must include:** Before/after metrics. Who needed convincing.

---

### B7. Tell me about a time you made a wrong technical decision. What happened and what did you learn?
**Tests:** Intellectual honesty, growth mindset, judgment recovery.
**Frame:** Own it fully. Don't deflect. Explain what information you had at the time, what you'd weight differently now, and what process change you made after.
**Must include:** A specific lesson that changed your future decision-making.

---

### B8. How do you evaluate whether to build vs buy for a new capability?
**Tests:** Frugality, Think Big, Are Right A Lot.
**Frame:** Walk through your framework: core vs commodity? differentiation? TCO? team expertise? vendor lock-in risk? Give a real example where you made this call both ways.

---

### B9. Tell me about a time you disagreed with the technical direction chosen by your team or manager.
**Tests:** Have Backbone; Disagree and Commit.
**Frame:** Explain your position clearly. Show you made your case with data. Describe how you committed once the decision was made. Critical: don't show lingering resentment.
**Must include:** What you did to make the decision succeed after committing.

---

### B10. Describe a time you had to balance technical debt against feature delivery.
**Tests:** Are Right A Lot, long-term thinking, stakeholder negotiation.
**Frame:** Be specific about the tech debt (what was fragile, what was it costing in engineer time). Explain how you quantified the cost in business terms. How did you negotiate the prioritization?

---

### B11. Tell me about a time you reduced costs significantly.
**Tests:** Frugality, Ownership, Deliver Results.
**Frame:** Start with the cost baseline. Walk through what you identified (idle resources? over-provisioned VMs? inefficient architecture?). Show the analysis, the change, and the savings amount.
**Must include:** Dollar figure or percentage reduction, timeline, how you confirmed savings.

---

### B12. Describe a time you improved developer productivity significantly.
**Tests:** Invent and Simplify, Think Big, Impact at scale.
**Frame:** What was the pain point engineers were experiencing? (Slow CI, flaky tests, bad local dev, unclear runbooks?). What did you build or change? What was the before/after?

---

## Part 2: Conflict, Influence, and Teamwork

---

### B13. Tell me about a time you had a conflict with a peer engineer.
**Tests:** Earn Trust, communication, maturity.
**Frame:** Stay neutral in tone. Focus on the disagreement idea, not the person. Show you sought to understand their view. Describe how you resolved it constructively.
**Avoid:** Making the other person the villain. Showing you "won."

---

### B14. Describe a time you had to influence without authority.
**Tests:** Earn Trust, Have Backbone, Think Big.
**Frame:** Classic platform/DevOps scenario — you need teams to adopt a standard you don't own. Show how you built credibility (demonstrated value first, reduced friction, removed blockers), not how you mandated compliance.

---

### B15. Tell me about a time you had to push back on a stakeholder's request.
**Tests:** Have Backbone, Earn Trust, customer obsession.
**Frame:** Show respect for the stakeholder's intent. Explain your concern clearly and with data. Describe the alternative you proposed. Be clear about outcome.

---

### B16. Describe a time you had to work with a team that had very different working styles.
**Tests:** Earn Trust, collaboration, adaptability.
**Frame:** Don't complain about the other team. Focus on what you adjusted in your own approach. Show learning from the collaboration.

---

### B17. Tell me about a time you had to get alignment across multiple teams on a controversial decision.
**Tests:** Earn Trust, Think Big, Bias for Action.
**Frame:** Who were the stakeholders? What were the conflicting interests? How did you facilitate consensus? What compromise (if any) did you make?

---

### B18. Describe a time you had to say no to a feature or request that engineering was capable of doing.
**Tests:** Insist on the Highest Standards, Frugality, Ownership.
**Frame:** Why was it the right call? How did you communicate the no? What alternative did you offer?

---

### B19. Tell me about a time you had to overcome significant resistance to a change you were driving.
**Tests:** Have Backbone, Earn Trust, Deliver Results.
**Frame:** What was the resistance? (Fear of change? Different priorities? Past failures?) How did you address the root concern vs the stated objection?

---

### B20. Describe a time you had to deliver bad news to a stakeholder or manager.
**Tests:** Earn Trust, communication, ownership.
**Frame:** Show you delivered early, not late. Show you came with context and a plan, not just the problem. Describe how the conversation went and what happened next.

---

### B21. Tell me about a time you worked across time zones or remote teams effectively.
**Tests:** Collaboration, process thinking, communication practices.
**Frame:** Be concrete about the process changes you made (async documentation, overlap hours, decision logs). Show the outcome wasn't worse than co-located work.

---

### B22. Describe how you handle a team member who is consistently underperforming.
**Tests:** Hire and Develop the Best, Earn Trust, management judgment.
**Frame:** Show a clear progression: notice → discuss → set expectations → support → escalate if needed. Don't jump to "I let them go." Show you tried to help first.

---

## Part 3: Failure, Incidents, and Learning

---

### B23. Tell me about a major production incident you caused or contributed to.
**Tests:** Ownership, Learn and Be Curious, Earn Trust.
**Frame:** Own it directly. No hedging, no blame-sharing. Describe the impact clearly. Walk through the timeline. Explain the root cause and the systemic fix.
**Must include:** The post-mortem changes that prevented recurrence. What you changed about your own practice.

---

### B24. Describe a project that failed. What happened?
**Tests:** Ownership, growth mindset, judgment.
**Frame:** Be specific. Under-delivery, wrong problem solved, technical failure — pick a real one. Explain the decisions that led to failure. Be clear about your role. Show what changed after.

---

### B25. Tell me about a time you missed a deadline.
**Tests:** Ownership, Deliver Results, communication.
**Frame:** Key is: did you see it coming? Did you raise it early? Show you communicated proactively, not after the fact. What was the cause, and what did you change?

---

### B26. Tell me about a time you made a decision that looked right but turned out to be wrong.
**Tests:** Are Right A Lot, intellectual honesty.
**Frame:** The best answer shows good decision-making process with an outcome that still didn't match expectation. Focus on: "Given the information I had, this was reasonable, but here's what I learned to look for next time."

---

### B27. Describe a time you had to roll back a deployment or revert a change.
**Tests:** Bias for Action, Ownership, technical judgment.
**Frame:** What made you decide to roll back vs fix forward? Show your decision framework. What was the user impact? What did the post-mortem say?

---

### B28. Tell me about the most stressful period in your career. How did you handle it?
**Tests:** Resilience, self-awareness, leadership under pressure.
**Frame:** Be honest about what was hard. Show you maintained quality of decision-making. Show you protected your team from chaos.

---

### B29. What's the biggest mistake you've made in your career?
**Tests:** Intellectual honesty, growth.
**Frame:** Choose something real, not trivially small. The lesson should be substantive. The arc: mistake → recognize → learn → change.

---

### B30. Tell me about a time when a project you were leading went off track.
**Tests:** Ownership, recovery, leadership.
**Frame:** At what point did you realize it was off track? What corrective action did you take? How did you communicate to stakeholders?

---

## Part 4: Ambiguity, Prioritization, and Judgment

---

### B31. Tell me about a time you had to make an important decision with very little data.
**Tests:** Bias for Action, Are Right A Lot, judgment.
**Frame:** What decision was it? What information did you have vs wish you had? How did you structure your reasoning under uncertainty? What was the outcome?

---

### B32. How do you prioritize when everything is urgent?
**Tests:** Deliver Results, Bias for Action, stakeholder management.
**Frame:** Give a framework: impact vs effort, business criticality, risk of delay. Then give a specific example where you applied this.

---

### B33. Describe a time you had to change your mind significantly about something.
**Tests:** Are Right A Lot (and humility), Learn and Be Curious.
**Frame:** What caused the change? New data? A compelling argument? Experience? Show you can update beliefs with evidence.

---

### B34. Tell me about a time you had too many things on your plate. What did you do?
**Tests:** Deliver Results, communication, prioritization.
**Frame:** Show you triaged proactively, delegated where appropriate, and negotiated scope — not that you just worked harder.

---

### B35. Describe a time when you had to navigate significant organizational change.
**Tests:** Resilience, adaptability, leadership.
**Frame:** Reorg, acquisition, layoff, leadership change — whatever your real experience was. Show how you helped your team adapt, how you maintained productivity.

---

### B36. Tell me about a time you had incomplete requirements and still had to deliver.
**Tests:** Bias for Action, customer obsession, judgment.
**Frame:** Show you didn't wait for perfect requirements. You made reasonable assumptions, validated with the customer/stakeholder, and iterated.

---

### B37. How do you decide when something is "good enough" vs when to keep improving it?
**Tests:** Insist on the Highest Standards vs Bias for Action tension.
**Frame:** Give your framework. Then a real example with stakes. The business context should drive the answer, not perfectionism.

---

### B38. Tell me about a time you took a calculated risk that paid off.
**Tests:** Bias for Action, Think Big, Ownership.
**Frame:** What was the risk? How did you assess it? What mitigation did you put in place? What was the payoff?

---

## Part 5: Mentorship, Culture, and Team Building

---

### B39. Tell me about someone you've mentored and the impact you had.
**Tests:** Hire and Develop the Best, Earn Trust.
**Frame:** Be specific about the person's starting point, what you worked on together, and where they are now. Show you were generous with your time and knowledge.

---

### B40. How do you bring a new engineer up to speed quickly?
**Tests:** Hire and Develop the Best, process thinking.
**Frame:** Give your actual onboarding approach: pairing, documentation, starter tasks, feedback loops. Show you've thought about this systematically.

---

### B41. Describe a time you helped create or improve engineering culture.
**Tests:** Think Big, Earn Trust, Strive to Be Earth's Best Employer.
**Frame:** Guild, blameless post-mortems, on-call rotation, internal tech talks — whatever you drove. Show the before/after and how it landed with the team.

---

### B42. Tell me about a time you gave difficult feedback to someone.
**Tests:** Earn Trust, Hire and Develop the Best.
**Frame:** Show you were specific, timely, and focused on the behavior not the person. Describe how they received it and what changed.

---

### B43. Describe a time you received critical feedback and how you responded.
**Tests:** Learn and Be Curious, Earn Trust.
**Frame:** Be honest. Show you listened, reflected, and made a change. Don't be defensive even in retrospect.

---

### B44. How do you create psychological safety in your team?
**Tests:** Earn Trust, Strive to Be Earth's Best Employer, leadership philosophy.
**Frame:** Give examples of specific behaviors: modeling vulnerability, celebrating incident learning, not rewarding hero culture. Then a case where it mattered.

---

### B45. Tell me about a time you promoted someone or advocated for someone's career growth.
**Tests:** Hire and Develop the Best.
**Frame:** What did you see in them? What did you do to support their growth? What was the outcome of your advocacy?

---

## Part 6: Delivering at Scale and Under Pressure

---

### B46. Tell me about a time you delivered a project under extreme time pressure.
**Tests:** Deliver Results, Bias for Action, Ownership.
**Frame:** What was the hard deadline and why? What trade-offs did you make (scope, quality, risk)? Were those trade-offs the right call? What was the outcome?

---

### B47. Describe a project where you had to coordinate many teams to deliver.
**Tests:** Think Big, Earn Trust, Deliver Results.
**Frame:** Large cross-team infrastructure migrations are classic here. Show your coordination mechanism: shared roadmap, clear owners, escalation path.

---

### B48. Tell me about a time you improved the on-call experience for your team.
**Tests:** Insist on the Highest Standards, Earn Trust, Strive to Be Earth's Best Employer.
**Frame:** What was the baseline? (E.g., "Engineers were getting paged 10+ times/week, most were false positives.") What did you change? Alert tuning? Runbooks? Rotation? What improved?

---

### B49. Describe a time you had to scale a system that was never designed to scale.
**Tests:** Invent and Simplify, Think Big, technical depth.
**Frame:** This is a technical + leadership story. Show the architectural analysis, the migration path, and how you kept the existing system running while scaling it.

---

### B50. Tell me about a time you drove standardization across multiple teams.
**Tests:** Insist on the Highest Standards, Earn Trust, Think Big.
**Frame:** CI/CD standards, IaC module library, observability standards — pick a real case. Show how you got adoption without mandating top-down.

---

## Part 7: SRE & Reliability Deep Dives

---

### B51. Walk me through how you handle an ongoing production incident.
**Tests:** Ownership, communication, systematic thinking.
**Frame:** Use your incident command framework. Roles (IC, comms, SMEs). Timeline. Stakeholder communication cadence. Decision checkpoints (rollback vs fix-forward). Post-mortem trigger.

---

### B52. How do you define SLOs for a service you own?
**Tests:** Dive Deep, customer obsession, standards.
**Frame:** Walk through the SLI selection process (what does the customer actually care about?). Then SLO target setting (based on historical data + business requirement). Error budget policy.

---

### B53. Tell me about the most impactful post-mortem you led.
**Tests:** Ownership, Insist on Highest Standards, Learn and Be Curious.
**Frame:** What happened? How did you run the post-mortem (blameless, timeline-focused, action-item-driven)? What systemic changes came out of it? Did they stick?

---

### B54. How do you prevent alert fatigue on your team?
**Tests:** Insist on Highest Standards, Frugality (of eng time), empathy.
**Frame:** Specifics: alert audit process, signal vs noise framework (alert only on SLO burn rate), routing to non-pager channels for low-severity. Show before/after.

---

### B55. Describe how you manage on-call rotation and runbooks.
**Tests:** Strive to Be Earth's Best Employer, Insist on Highest Standards.
**Frame:** Schedule fairness. Runbook quality (every alert has a runbook). Post-shift review. Automation of known responses. Escalation policy.

---

## Part 8: Company-Specific Framing

---

### B56–B60: Amazon LP-specific questions

**B56.** "Tell me about a time you went above and beyond for a customer (internal or external)." → LP: Customer Obsession

**B57.** "Tell me about a time you did something without being asked that made a significant difference." → LP: Ownership

**B58.** "Tell me about a time you simplified a complex process." → LP: Invent and Simplify

**B59.** "Tell me about a time you took a shortcut that created problems later." → LP: Insist on the Highest Standards (negative)

**B60.** "Tell me about the last time you dove deep into data to solve a problem." → LP: Dive Deep

---

### B61–B65: Google-style (systems thinking + cross-team)

**B61.** "Tell me about a time you identified a systemic problem and fixed it at the root."

**B62.** "How do you ensure your infrastructure changes don't negatively impact other teams?"

**B63.** "Tell me about a time you made a trade-off decision affecting multiple teams. How did you handle it?"

**B64.** "Describe a time when you balanced short-term delivery against long-term system health."

**B65.** "Tell me about a time you had to debug a distributed system issue with multiple potential causes."

---

### B66–B70: Startup/Product Company (builder mindset, scrappiness)

**B66.** "Tell me about a time you built something from scratch with minimal resources."

**B67.** "How do you decide what to automate vs what to do manually in an early-stage platform?"

**B68.** "Tell me about a time you wore multiple hats and kept everything from falling apart."

**B69.** "Describe a time you had to make a platform decision that would affect the company for years."

**B70.** "How do you maintain reliability when you're also moving fast to ship product?"

---

## Part 9: Questions You Should Ask the Interviewer

---

### B71. For Engineering Culture
"What does a blameless post-mortem look like here in practice — not policy, but what actually happens when something goes wrong?"

### B72. For On-Call & Reliability
"How many alerts does an average on-call engineer receive per week? What's the ratio of actionable to noise?"

### B73. For Technical Direction
"What's the biggest infrastructure or platform challenge the team is working on in the next 6 months?"

### B74. For Team Health
"How do senior engineers grow here? What's the path from Senior to Staff?"

### B75. For Platform Maturity
"How self-service is the developer platform today? What do teams still have to come to platform/infra for?"

### B76. For Decision-Making Culture
"Give me an example of a recent architectural decision — how was it made and by whom?"

### B77. For Incidents
"When was your last major outage? How long did it take to resolve, and what changed as a result?"

### B78. For The Role
"What would make this hire a clear success in the first 90 days?"

---

## Part 10: Story Sizing and Calibration

---

### B79. Junior vs Senior STAR Story Calibration

| Dimension | Junior Story | Senior Story |
|---|---|---|
| Scope | One team or feature | Multi-team, system-wide, or business-critical |
| Decision-making | Followed guidance | Made the call, drove consensus |
| Trade-offs | Not articulated | Explicitly named and reasoned |
| Result | Task completed | Measurable business/ops impact |
| Complexity | Technical only | Technical + organizational + stakeholder |
| Ownership | Contributed | Accountable end-to-end |

---

### B80. Pre-Interview Story Checklist

For each story you prepare, verify:
- [ ] I say "I" not "we" as the primary actor
- [ ] The scope is senior-level (multi-team or business-critical)
- [ ] I name at least one explicit trade-off
- [ ] There is a quantified result (even if approximate: "reduced costs by ~30%", "cut MTTR from 4h to 45min")
- [ ] I can deliver it in under 3 minutes
- [ ] I know which 2–3 LP/principles this story maps to
- [ ] I've practiced it out loud at least twice
- [ ] I have a "what I'd do differently" ready for the follow-up
