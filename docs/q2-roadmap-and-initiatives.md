# Q2 Roadmap and Initiatives

<aside>
📋

April 1 through June 30. **Deepen Quality → Scale → Summer Launch.**

</aside>

## Q1 Recap

### Metrics

| **Metric** | **Start of Q1 (Jan 5)** | **Mid-Q1 (Feb 17)** | **Change** | **End of Q1 (Mar 28)** | **Change** | **Q1 Goal** |
| --- | --- | --- | --- | --- | --- | --- |
| Successful dates | 698 | 871 | ⬆️ 25% | 1,203 | ⬆️ 38% | 2,000 ❌ |
| Matches | 3,507 | 4,996 | ⬆️ 42% | 7,738 | ⬆️ 55% |  |
| Matches in a single Wednesday | ~150 | ~450 | ⬆️ 200% | 634 | ⬆️ 40% | 2,500 ❌ |
| Registered Users (Phone Verified) | 8,497 | 70,127 | ⬆️ 700% | 118,656 | ⬆️ 70% |  |
| User messages | 25,577 | 158,870 | ⬆️ 521% | 771,481 | ⬆️ 386% |  |
| iMessage Lines | 10 | 62 | ⬆️ 500% | 117 | ⬆️ 89% |  |

### Initiative Results

| **Initiative** | **Status** | **Notes** |
| --- | --- | --- |
| Infrastructure Hardening | ❌ Missed | Did not conduct load/chaos testing. iMessage line flagging emerged as a larger, more urgent problem than raw infrastructure capacity. |
| Chatbot Reliability | ❌ Missed | iMessage service uptime was high, but lines were constantly getting flagged. Root cause identified: insufficient user engagement on iMessage lines triggers carrier flags. This insight shapes a new Q2 initiative. |
| Shadow Profile / Feedback Loop | ❌ Scrapped | Scrapped the approach after 6 weeks of work. Replacing with a simpler **User Memory** system that extracts useful information from chat for matchmaking. UFL (chat segmentation) also dropped. |
| AI Matchmaker v3 | ❌ Missed | The version initiated by the former Chief Architect was not actionable. Focusing on improving v2 while brainstorming the next leap. |
| Yik Yak Match Execution | ✅ Success | Execution went well. Still need to: write scripts for additional conversions and pull data on conversion numbers. |
| Follow-up & Engagement | ❌ Not started | Never began. Moving to Q2, combined with the new iMessage engagement initiative. |
| Workflow Automations | 🟡 Partial | Mostly done. Poster generation still incomplete. |
| Data & LLM Analytics | ✅ Good progress | Implemented an LLM analytics tool in Claude that has been very helpful across the team. More progress needed in this direction. |

### Key Takeaways from Q1

- **We missed the two hard numeric goals** — successful dates (1,203 vs 2,000) and matches per Wednesday (634 vs 2,500) — primarily due to not shipping matchmaker v3 and limited matchmaking bandwidth.
- **Too much time spent firefighting.** We need to run everything more systematically so the team can focus on building rather than reacting.
- **iMessage line flagging is a solvable problem.** The root cause is user engagement patterns, not infrastructure. This is now a first-class initiative.
- **Simpler approaches win.** The Shadow Profile / UFL approach was over-engineered. User Memory is the right abstraction.
- **The Yik Yak partnership proved our model works** at scale with external partners. Conversion and retention are the next levers.

---

## Important Dates

| **Date** | **Event** | **Notes** |
| --- | --- | --- |
| Apr 1 | Q2 begins |  |
| Apr 14 | Chatbot refactoring complete | Context improvement refactor must land within two weeks |
| Mid-May | Summer soft launch | Nationwide matching live for testing. Small-scale marketing begins. Targets semester-system schools ending mid-May. |
| Mid-June | Summer full launch | ⭐ Full nationwide launch with marketing at scale. All systems 100% ready. Targets all college students |
| Jun 30 | Q2 closes | Quarter wrap-up, Q3 planning begins |

---

## Summer Campaign

- City wide matchmaking for college interns & students from Mid-May to Mid-Aug
- Start with high density regions like SF, LA, NYC, Chicago, and Boston
- Will run in person events throughout the summer during these 2 months
- It’s a replicate version of the last summer’s initiative but 5x bigger

---

## Top Priorities

1. **Summer launch campaign.** This will be a combined effort of the engineering, product, and marketing teams. We will define the product experience, ensuring it runs smoothly and delivers the experience to as many people as possible.
2. **Deepen match quality and user understanding.** User Memory, chatbot context improvements, and matchmaker v2 enhancements must come together so matches feel meaningfully better by summer.
3. **Iterate 3x faster.** We are still too slow on everything, especially on the full-stack side implementing new features and fixing bugs.
4. **Solve iMessage line health.** If lines keep getting flagged, nothing else works. Driving user engagement on iMessage is the fix.

---

## Key Initiatives

### Initiative 1: Summer Launch

**Objective:** Remove school-based matchmaking boundaries and launch Ditto nationwide with a summer dating campaign.

**Owner:** Product Team
Other Executors:  Marketing + Engineering Teams

**What's happening:**

- Remove school-based matching constraints so users can be matched across schools
- Summer-themed marketing campaigns centered on getting a date over the summer
- **Soft launch (mid-May):** Product ready for limited testing, small-scale marketing moves
- **Full launch (mid-June):** Product 100% ready, full marketing push across all channels
- Coordination across product (UX for experience), engineering (matching algorithm changes, infrastructure), and marketing (campaigns, content)

<aside>
✅

**Success looks like:**

- Nationwide matching is live and stable
- Soft launch runs without critical issues by mid-May
- Full launch executes at scale by mid-June
- Summer campaign drives measurable user acquisition and engagement
</aside>

---

### Initiative 2: User Memory System

**Objective:** Build a unified user memory layer that extracts useful information from conversations and other signals into a representation usable by the matchmaker and chatbot.

**Owner:** @Charlie Zhuang  
Other Executors: @Nelson Cheuk-Yam Siu @Srikanth Banda @Rajat Jaiswal 

**What's happening:**

- We had a lot of different terms (Shadow Profile, UFL, and Dynamic User Profile), we are scratching all of them and making them one thing: User Memory.
- Extracts key preferences, behaviors, and signals from chat history and user actions
- Produces a structured memory that the matchmaker can use for better matching and the chatbot can use for personalized conversations
- This is the canonical user understanding layer for all downstream AI systems

Important Milestones:

- Team wide demo: Apr 3rd
- In production by: Apr 10th
- Adopted by matchmaker by: Apr 24th

<aside>
✅

**Success looks like:**

- User memory is populated for all users and updates as new signals come in
- Helps the matchmaker in making better matches
- Chatbot references memory for context-aware conversations
- Single source of truth for who a user is, replacing fragmented signal
</aside>

---

### Initiative 3: Chatbot Improvements - Context, Capability, Evals, and Speed

**Objective:** Improve the conversational agent in all

**Owner:** @Muskan Shaikh 
Other executors: @Ryan Willis @Srikanth Banda @Yuki Han 

**What's happening:**

- Refactor the chatbot's context system so it has the right information at the right time, producing better and more relevant conversations. Replacing the current multi-agent system with a skills-based single agent. This gives the agent the right knowledge at the right time through on-demand retrieval
- Refactoring must be complete within two weeks (by mid-April)
- Playbook feature: a self-learning knowledge base that grows from resolved escalations. When the agent can't answer a question, it escalates to team. The team reviews and approves, and the next user with the same question gets an instant response. Escalation rate decreases automatically over time.
- Improving how the chatbot retrieves and uses context during conversations
- Moving from noisy raw chat history to structured, relevant context
- Be able to take in new events and add new tools fast. Adding a new event goes from changes across 5-6 files to adding a single knowledge file
- Evals: leveraging the new observability stack to build conversation quality scoring. Exploring building our own eval suite that covers basic user scenarios across all statuses (Waiting, Matched, NeedMoreInfo, etc, will expand on this later.

<aside>
✅

**Success looks like:**

- Chatbot refactoring done by mid-April
- Measurable improvement in efficiency in adding new context
- Measurable improvement in conversation quality and relevance
- Reduced token consumption from more efficient context retrieval
- Escalation rate: establish baseline, then track reduction after playbook launch
- Full conversation tracing across multiple observability platforms
</aside>

**Asks (from product team):**

- Finalize event/pool strategy that should be followed. (Example: how to handle users after an event? will they be switched to Wednesday matching or will be asked to opt in for Wednesday matching)
- Give us an overview on the new updates to the product. When we add cross school matching

---

### Initiative 4: Matchmaker v2 Improvements

**Objective:** Improve the current matchmaker's quality using real data and learnings from Q1, while planning the next major leap.

**Co-Owners:** @Nelson Cheuk-Yam Siu  @Sylvia Liu @Nathan Standiford 
Other Executers: @Bill Peng 

**What's happening:**

- Dealbreaker Layer
- Make the current 6 feature scores comparable
    - creating a trust score
- feature improvements/iterations
    - modifying AttractivenessScoring to identify and analyze a user in a group photo
    - other feedback from @Sylvia Liu
- Iterating on v2 with real match outcome data from Q1
- Adapting the algorithm for nationwide (cross-school) matching
- Brainstorming the architecture for the next major version
- Eval-driven approach: measuring match quality improvements with data, not intuition
- **Evals:** Build an offline evaluation framework for matchmaking — regression suites, outcome tracking, and human-calibrated quality assessments. No changes ship without before/after measurement.

<aside>
✅

**Success looks like:**

- Measurable improvement in match acceptance rate over Q1 baseline
- Measurable increase in number of matches by expanding the matching bandwidth
- Algorithm handles cross-school matching effectively for summer launch
- Clear technical plan for the next matchmaking leap
- Evaluation framework established for ongoing quality measurement
</aside>

---

### Initiative 5: iMessage Engagement & Intelligent Follow-up

**Objective:** Increase user engagement on iMessage to prevent line flagging, and build automated follow-up flows to recover drop-offs.

**Owner:** @Yuki Han 
Other Executors: @Srikanth Banda @Ryan Willis 

**What's happening:**

- **Engagement side:** Design and ship product/UX changes that make iMessage conversations more engaging and natural, reducing the pattern of one-sided messaging that triggers carrier flags
- **Follow-up side:** Build automated recovery flows for stalled users (onboarding drop-offs, inactive user re-engagement)
- Smart Timing & Message Pacing: Improve iMessage line health by optimizing send behavior:
adaptive follow-up timing based on user activity, avoid bulk outbound messaging patterns
smart throttling during active nights, dynamic pacing based on response behavior
- iMessage health: As part of this initiative, we also need to ensure the iMessage lines are healthy by implementing customized algorithms and working directly with iMessage providers.

<aside>
✅

**Success looks like:**

- Significant reduction in iMessage line flagging rate
- ≥30% recovery rate on stalled users
- ≥60% feedback completion rate after dates
- Measurable reduction in manual follow-ups by ops team
</aside>

---

### Initiative 6: Engineering Velocity & Operational Efficiency

**Objective:** Drastically reduce the time it takes for features and bug fixes to reach production. Build the processes and standards that let us iterate 3x faster.

**Owners:** @Eric Liu and DevOps Team (@Nathan Standiford @Kelly Guo @Bill Peng and new member(s)) 

**What's happening:**

- 1-2 more hires on the full-stack side
- Define and enforce clear SLAs for feature delivery and bug fixes (e.g., critical bugs in production within 24 hours, standard features within 1 sprint)
- Identify and remove bottlenecks in the current development-to-production pipeline
- Establish guidelines for code review turnaround, deployment frequency, and testing requirements
- Reduce context-switching and firefighting through better monitoring, alerting, and runbooks
- Improve onboarding documentation so new team members can contribute faster
- @Eric Liu and @Allen Wang will also work on a Ditto guideline

<aside>
✅

**Success looks like:**

- Measurable reduction in average time from PR to production
- Critical bugs are fixed and deployed within 24 hours
- Standard features ship within one sprint cycle
- Less than 20% of engineering time spent on unplanned firefighting
- Clear, documented development guidelines that the whole team follows
</aside>

---

### Initiative 7: Metrics Definition & Analytics

**Objective:** Define Ditto's North Star metric and all core business metrics. Build the analytics layer so any metric, graph, or dashboard can be generated within hours.

**Co-Owners:** Product Team + @Praveen Kuruvangi Parameshwara 

**What's happening:**

- **Define the North Star metric** — the single metric that best captures the value Ditto delivers to users (candidates: successful dates per week, weekly active matched users, etc.). Align the entire team around it.
- **Define all core business metrics** — acquisition, activation, engagement, retention, matchmaking quality, conversion, and operational health. Each metric must have a clear definition, data source, and owner.
- **Build rapid analytics capability** — expand the LLM analytics tool from Q1 so that any team member can generate the graph, dashboard, or data export they need within hours, not days
- **Data consistency** — ensure metrics are computed from clean, consistent data sources. Resolve discrepancies between systems. Currently there are two many discrepancies between different sources.

<aside>
✅

**Success looks like:**

- North Star metric is defined, tracked, and visible to the entire team
- All core business metrics are documented with clear definitions and dashboards
- Any data request can be fulfilled within hours (not days)
- Team makes decisions based on metrics, not gut feeling
</aside>

| **Step Name** | **Priority** |
| --- | --- |
| Data Cleanup | Critical |
| Address Data Mismatch | Critical |
| Product Team Dashboard Discussion | High |
| MongoDB → Snowflake Sync Setup | Critical |
| Build Aggregate Models | Critical |
| Understand DB Load & Plan Right Databases | High |
| Build Refresh Pipeline (Frequency & Cost) | High |
| Create New Charts & Dashboards | High |
| Julius AI + Claude Analytics Tool | High |
| Monitor - Maintain & Enhance | High |

---

### Initiative 8: Product Experiments & New Experiences

**Objective:** Rapidly test new product concepts that expand how users experience Ditto. Run at least 3 experiments this quarter.

**Owner:** Product Team

**What's happening:**

- **Double Dates** — Experiment with matching two pairs for a group date experience. Design, prototype, and test with a subset of users.
- **Experiment 2** — *[To be defined]*
- **Experiment 3** — *[To be defined]*
- Each experiment follows a lightweight cycle: hypothesis → prototype → small-scale test → measure → decide (keep, iterate, or kill)
- Experiments should be scoped small enough to test within 2–4 weeks each

<aside>
✅

**Success looks like:**

- At least 3 product experiments launched and measured by end of Q2
- Double Dates tested with real users and clear signal on whether to invest further
- At least 1 experiment graduates to a full feature
- A repeatable lightweight experiment process the team can reuse
</aside>

---

### Initiative 9: Internal Agent & Knowledge Base

**Objective:** Build an internal AI agent ("Company Jarvis") that knows everything about Ditto's product and engineering, has its own knowledge base and memory, and can act on most operational tasks.

**Owner:** @Eric Liu 

**Reference:** ‣

**What's happening:**

- Deploy OpenClaw as Ditto's company-wide AI agent, accessible via Slack and API
- Three specialized agents: **code** (codebase Q&A, code review, PR creation), **ops** (DB queries, monitoring, test environments), **general** (knowledge base, writing, search)
- Role-based access control: admin → engineer → member, with credential isolation at the database level
- Knowledge base powered by real-time access to Notion, Linear, GitHub repos, and agent-generated knowledge — no vector DB, no stale sync pipelines
- **Rollout:** @Eric Liu  builds and validates solo first → selected members → full team
- Initial deployment on a dedicated GCP VM with systemd, health monitoring via Better Stack, and a kill switch for safety

<aside>
✅

**Success looks like:**

- Agent is live and used daily by Eric within the first two weeks of Q2
- Agent can answer codebase questions, query production data, and surface knowledge from Notion/Linear
- Engineering team onboarded by mid-Q2
- Full team has access to the general agent by end of Q2
- Measurable reduction in time spent answering repetitive questions and looking up information
</aside>

---

### Initiative 10: Influencer Campaign Establishment

Objective: we want to build a scalable and robust influencer marketing system that is automatic, AI-native, and also targeting our ICP correctly. 

Owner: @Kong Jane 

**What's happening:**

- AI native automatic outreaching system, influencer managed system, and data tracking system
- Recap and debrief on the past performance of the influencer campaign and improve the CPM and conversion overall.
- Start planning influencer campaign for the summer CY matchmaking.

---

### Initiative 11: Canada Colleges Expansion

Objective: Bring Ditto to most of the major Canada schools asap

Owner: @Elsa Cai 

**What's happening:** 

- Campus leads: establish campus lead and campus team at each of the schools.
- User Growth: we are aiming for 500 to 800 completely registered users before the summer at each of these schools.

## Updated Success Metrics & KPIs

| **Metric** | **Descriptions** | Category | **Start of Q1** | **Current** | **Q2 Goal** |
| --- | --- | --- | --- | --- | --- |
| **Core** |  |  |  |  |  |
| Percentage of users getting a **date** each week out of all active agents | Measures our ability to provide value. | Effectiveness |  |  |  |
| Average number of matches user get before they churn |  | Retention / Effectiveness |  | 2.8 |  |
| Dates per week | Magnitude of the value we are delivering. This is *amount of matches per week* X *match-to-date rate*.  | Influence X Effectiveness | 15 | 89 | 500 |
| Feedback completion rate | Fuels the learning loop | Engagement |  |  | ≥60% |
| Onboarding Conversion |  | Conversion |  | 25% | 30%-35%? |
| Chat Engagement |  | Engagement |  |  |  |
| Other |  |  |  |  |  |
| iMessage lines flagging rate | Lines going down = product goes down | Stability | 12.7% | 0% | <1% |
|  | Evaluating cross-school matching vs same-school matching |  |  |  |  |
| Urgent Issues Handling Timeframe |  | Team Speed |  |  | 6 hours |

---

## Risks

<aside>
🔴

**Firefighting over building** — In Q1, too much engineering time went to reacting to production issues instead of shipping new features. If this pattern continues, we will miss the summer launch. Mitigation: Initiative 6 (Engineering Velocity) directly targets this. Clear SLAs, better monitoring, runbooks, and onboarding docs to prevent fires instead of fighting them.

</aside>

<aside>
🔴

**iMessage line flagging** — Still the highest-risk dependency. If engagement improvements don't reduce flagging, the entire product is at risk. Mitigation: this is now a dedicated initiative, not a side project.

</aside>

<aside>
🔴

**Summer launch readiness** — Nationwide matching is a fundamental product change. If it's not stable by mid-May, the soft launch slips and the full June launch is at risk. Mitigation: hard deadlines, phased rollout, chatbot refactor lands by mid-April.

</aside>

<aside>
🟡

**Matchmaking quality at nationwide scale** — Cross-school matching changes the dynamics significantly. Match quality could regress before it improves. Mitigation: eval-driven approach, measure before and after, rollback plan.

</aside>

---

## How We Work

*Will be expanded into a Ditto guideline*

- Bi-weekly sprints
- Weekly all-hands & progress tracking
- Weekly engineering stand-ups
- Evidence > speed > authority when we disagree
- If we can't measure it, we don't understand it well enough to ship it
- **New for Q2:** Run systematically, not reactively. Build processes that prevent fires instead of fighting them.