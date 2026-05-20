# Matchmaking Engine 3.x — Detailed Technical Review

**Reviewer:** Ryan Willis (with AI assistance)
**Date:** 2026-03-24
**Document Under Review:** `Ditto Matchmaking Engine 3 x 326d2ecb07cc804a90e1da42480d0aad.md`

---

## Section-by-Section Commentary

### Problem Diagnosis (Lines 17-104) — Strong

> **Line 19:** *"A matchmaking system asks: which pair should be brought together now, under uncertainty, with limited feedback, asymmetric preferences, and meaningful downstream consequences?"*

**Comment:** This is the strongest sentence in the document. It correctly frames matchmaking as a **sequential decision problem under uncertainty** rather than a static ranking problem. The word "now" is doing critical work — it introduces temporality into the framing from the very first sentence. This framing justifies the entire multi-head architecture that follows.

> **Line 23:** *"Similarity can be useful for retrieval, but it cannot serve as the full decision engine. It collapses distinct questions into a single score and, in doing so, destroys distinctions the product cannot afford to lose."*

**Comment:** Accurate diagnosis of the current system's core weakness. The existing `getTotalScore()` in `stable-matching.service.ts` does exactly this — collapses 6 dimension scores into one weighted sum. The phrase "destroys distinctions" is precise: once you've computed `weighted_sum = w1*pim + w2*hobby + ... - w3*attractiveness_diff`, you can no longer recover whether a pair is "viable but low-energy" vs "strong but mistimed." However, the document underestimates how much value can be extracted from the existing 7 features **before** building new heads. Directional scoring (A→B vs B→A) using existing PIM scores, plus outcome-weighted feature optimization, could capture 40-60% of the Likelihood head's value with zero ML infrastructure.

> **Lines 76-82:** *"Similarity may help identify plausible candidates as a filtering approach, but quality matchmaking entails at least four distinct questions: Likelihood, Intensity, Chemistry, Readiness"*

**Comment:** The four-question decomposition is conceptually correct but presented as equally important and equally feasible. They are neither. Likelihood is high-value and moderate-feasibility (can train on match acceptance data). Intensity is high-value and moderate-feasibility (can train on conversation depth). Chemistry is uncertain-value and low-feasibility (requires measuring residual uplift, which requires Likelihood + Intensity to be calibrated first). Readiness is moderate-value and low-feasibility (requires temporal state infrastructure that doesn't exist). The document should explicitly tier these by ROI and sequence accordingly.

---

### Five Pillars (Lines 132-147) — Architecturally Sound, Sequencing Unclear

> **Line 134:** *"a shared representation backbone across users, preferences, pair features, and interaction signals"*

**Comment:** The unified backbone is architecturally elegant but represents the highest-risk, highest-investment component. The document describes it as Pillar 1 (suggesting it's foundational), but it's actually the component that needs the most data, the most ML engineering capacity, and the longest development time. At Ditto's current scale (school-by-school matching, ~thousands of active users), a hand-engineered feature pipeline feeding tabular ML models (XGBoost, LightGBM) will outperform a learned backbone that is starved of training data. The backbone becomes valuable at ~50K+ active users with sufficient outcome data to train dense representations. **Recommendation:** Treat the backbone as Phase 9+, not Pillar 1. Build the heads on engineered features first. Migrate to a learned backbone when the data volume justifies it.

> **Line 144:** *"a calibration and modular decision layer that transforms raw head outputs into decision-ready scores, while policy execution occurs through composable agent skills/plugins"*

**Comment:** This is the most immediately actionable pillar. OpenClaw's skills system maps directly to this vision: each policy operator is a SKILL.md file that the agent reads and applies. Plugin hooks (`before_tool_call`) provide hard enforcement. The calibration layer (isotonic regression or Platt scaling) is straightforward to implement once head outputs exist. **This pillar should ship before the heads, not after** — start with calibrated heuristic scores (from Phase 2's optimized features), then upgrade the inputs as ML heads come online.

---

### Unified Representation Space (Lines 353-448) — Over-Engineered for Current Scale

> **Lines 396-398:** *"Research on efficient multi-task architectures demonstrates why. Work on joint intent-and-entity models has shown that fusing cheap sparse lexical features with dense pretrained embeddings inside a single shared encoder can outperform fine-tuning a large language model alone—while training many times faster."*

**Comment:** The research citation is valid (multi-task NLU benefits from hybrid sparse+dense fusion), but the analogy is strained. In NLU, training data is abundant — millions of labeled examples. In Ditto's matchmaking, labeled pair outcomes are sparse (maybe thousands of pairs with Tier 1-2 outcomes over 6 months). The "shared encoder" pattern works when you have enough gradient signal to train a meaningful shared representation. With sparse dating outcomes, the encoder will overfit to Tier 3-4 proxies (match acceptance, chat start) because that's where the data volume is. This is exactly the failure mode the document warns about in Risk R2 ("Sparse Downstream Labels"), but the architecture section doesn't incorporate that warning into the design.

> **Line 406:** *"A monolithic 'embed everything into one vector' approach loses the second and third families. A siloed approach loses the first. The right design is compositional fusion."*

**Comment:** Correct principle, but "compositional fusion" doesn't require a neural backbone. The existing Restate pipeline already does compositional fusion — it computes 6 separate scoring dimensions and aggregates them. The issue isn't the architecture pattern; it's that the aggregation function (weighted sum) is too simplistic. Upgrading to a learned aggregation function (even a simple gradient boosted tree over the 6+1 feature scores) would implement "compositional fusion" without the investment of building embedding towers. **The document conflates "better aggregation" with "neural backbone" when the former is sufficient and the latter is premature.**

> **Lines 410-420:** *"Quality does not arise only from user-level representations. It also arises from pair-level interaction structure: preference satisfaction, trait complementarity, asymmetry, directional effects..."*

**Comment:** Strongly agree. This is one of the most important insights in the document. The existing system computes some pair-level features (PIM forward/reverse, attractiveness difference) but treats them as just more dimensions to weight. They should be first-class cross-features in any ML model. Specifically: `preference_satisfaction(A, B)` (does B match what A wants?), `trait_complementarity(A, B)` (are differences constructive?), and `directional_asymmetry(A, B)` (how different is A→B from B→A?) should be explicit input features, not implicit consequences of embedding distance. This is achievable today with engineered features — no neural backbone required.

---

### Temporal User State Encoder (Lines 452-490) — Correct Concept, Premature Architecture

> **Lines 462-475:** *"The temporal user state should therefore update from: recent likes and dislikes, skipped or ignored profiles, match acceptances, chat starts and chat depth, contact exchanges, dates, post-date feedback, conversational preference edits, recent exposure to repeated archetypes, recency and fatigue effects"*

**Comment:** Comprehensive signal list, but the document doesn't address the cold reality: most of these signals don't exist in the current data infrastructure. The system stores match results and matching status, but doesn't track: skipped profiles (no swipe UI), dwell time, session-level interaction statistics, preference edit timestamps, or archetype exposure history. The temporal encoder can't consume signals that aren't collected. **Recommendation:** Phase 0 instrumentation must be designed with this signal list as the eventual consumer. But build the encoder incrementally — start with signals that already exist (match outcomes, conversation events, status transitions), add new signals as product features enable them.

> **Line 490:** *"the Temporal User State Encoder should apply a recency-weighted attenuation mechanism to user signals. Recent interactions (e.g., within the last 7 days) should carry higher influence, typically accounting for approximately 50–80% of the effective state"*

**Comment:** The 50-80% / 20-50% split is stated as a design parameter but should be a learned parameter. Different users have different preference stability — a user who shifts frequently should have higher recency weight (their history is less relevant), while a user with stable preferences should weight history more. A simple approach: compute preference volatility (variance of feature preferences over time) and use it to modulate the decay rate. Exponential decay with `lambda = base_lambda * (1 + volatility_score)` would achieve this without a neural encoder. The specific numbers (50-80%) should come from outcome data, not design intuition.

---

### Four Decision Heads (Lines 169-319) — Mixed Quality

> **Lines 169-187:** *"The Likelihood Head estimates whether a pair is actually viable. It models directional probabilities: P(A → B), P(B → A). These are combined into a mutual feasibility signal."*

**Comment:** This is the highest-value head and the best-specified. Directional modeling is exactly right — the existing PIM scoring already computes forward/reverse scores, so the concept isn't foreign to the codebase. The implementation path is clear: train a binary classifier on (directed pair features) → (did this user respond positively?), then derive mutual feasibility as `geometric_mean(P(A→B), P(B→A))`. An XGBoost model on the existing 7 feature scores plus pair cross-features would be a strong first implementation. The two-tower neural architecture described later (lines 416-418) is appropriate as a second iteration if the tabular model plateaus.

> **Lines 211-213:** *"The Chemistry Head estimates bounded positive novelty—the potential for a pair to outperform baseline expectations due to complementarity, asymmetry, non-obvious interaction structure, or latent pair effects not fully captured by direct preference satisfaction or surface compatibility."*

**Comment:** This is the weakest head specification. The Chemistry Head is defined as "residual uplift" — performance above what Likelihood + Intensity predict. The problem: measuring residual uplift requires Likelihood + Intensity to be well-calibrated first. If those heads have bias, the "residual" includes model error, not just true chemistry. Furthermore, the training signal is inherently noisy: a pair that outperformed expectations might have done so because of chemistry, or because of luck, or because of timing, or because of external factors. Separating true chemistry from noise in the residual requires much more data than the other heads.

The document's own Risk R6 (line 2695) acknowledges this: *"If the signal is too weak, the training labels too noisy, or the exploration budget too loose, Chemistry injects randomness rather than real uplift."* The mitigation — "require real outcome lift in ablation before scaling" — is correct, but the implication should be stronger: **Chemistry might never graduate from experiment to production.** The document treats it as a core head; it should be treated as a research hypothesis.

> **Lines 241-281:** *"The Readiness Head estimates whether this pair is being surfaced at the right moment for both users... Unlike feasibility, which answers 'can this work?', readiness answers: 'is this the right time for this to work?'"*

**Comment:** The distinction between Readiness and Likelihood is conceptually clear but empirically fragile. Risk R4 (line 2655) correctly identifies the danger: *"Without constraints, it can absorb attractiveness, popularity, segment effects, and baseline desirability — becoming a second Likelihood model."* The proposed mitigation — "constrain Readiness features to temporal and state-derived signals only" — is necessary but may not be sufficient. Even temporal features (recent activity level, response rate) correlate with user popularity, which correlates with Likelihood. Regularizing Readiness against Likelihood outputs during training (as suggested in line 2669) is the right approach but adds training complexity.

**My recommendation:** Don't build Readiness as a separate head initially. Instead, incorporate temporal state features (from Phase 3) as input features to the Likelihood and Intensity heads. This captures most of the timing value without the architectural overhead of a fourth head. Promote Readiness to a separate head only if ablation shows the temporal features are being underutilized by the combined model.

---

### Agent Skills / Plugins (Lines 1121-1200) — Strong, Under-Specified

> **Lines 1131-1133:** *"The models estimate what is true. The plugins decide how product policy should act on those truths."*

**Comment:** Excellent principle. This is the cleanest separation of concerns in the document. It maps directly to OpenClaw's architecture: model predictions are tool outputs; policy decisions are skill logic; enforcement is via plugin hooks. The document should go further: plugins should be the **first** thing built (even before ML heads), operating on calibrated heuristic scores. This validates the plugin framework before the heads exist, and provides immediate product value.

> **Lines 1188-1196:** *"Plugins should not become: a dumping ground for model weaknesses, a hidden ranking system, a substitute for poor representation learning, a workaround for missing pair semantics, unbounded heuristic overrides on top of model outputs"*

**Comment:** Critical guardrail, but organizationally very hard to enforce. In practice, every quality complaint that arrives on a Friday afternoon will be "fixed" by adding a plugin rule. Within 6 months, the plugin layer will have 15+ rules with undocumented interactions. **Concrete enforcement mechanism needed:** maximum 5 active plugins at any time. Every new plugin requires retiring an existing one or demonstrating it cannot be achieved by tuning existing plugin parameters. Quarterly audit: disable each plugin individually and measure impact; remove any with no measurable positive effect.

---

### Learning & Feedback System (Lines 1207-1525) — Theoretically Sound, Practically Distant

> **Lines 1240-1242:** *"Fast Loop (Online / Near Real-Time Updates): Adapt quickly to user state and short-term behavior"*

**Comment:** The two-speed learning architecture is the correct design. But "near real-time" state updates require infrastructure that doesn't exist: event streaming, real-time feature computation, and a state store that can handle concurrent reads and writes per user. At Ditto's current scale, a daily batch computation (Phase 3's approach) captures 90% of the value with 10% of the infrastructure cost. The fast loop should be deferred until the batch approach proves insufficient.

> **Lines 1338-1340:** *"The system only observes outcomes for shown pairs. This introduces selection bias."*

**Comment:** The most important technical paragraph in the Learning section. Selection bias is the fundamental challenge of any closed-loop matching system, and most dating companies never address it. The proposed solution — counterfactual logging + controlled exploration — is correct. But the implementation is harder than described: logging full candidate pools with scores for all candidates (not just winners) requires significant storage overhead (~100x more data per cycle). This must be designed into Phase 0 from day one, not added later.

---

### Agentic RL Training (Lines 2922-3493) — Research Paper, Not Implementation Spec

> **Lines 2926-2927:** *"This is not a future research direction. It is the operational specification for how Ditto's matchmaking models train from real-world outcomes."*

**Comment:** This claim is premature. The section describes PPO with clipped surrogate objectives, Process Reward Models, On-Policy Distillation with hint extraction, progressive reward assignment across 4 tiers, majority-vote PRM judging, and asynchronous training with zero-blocking weight updates. This is a research-grade system that even well-funded ML labs (DeepMind, OpenAI) find challenging to stabilize. The mathematical formulations are rigorous, but they assume:

1. Trained neural heads exist to apply gradients to (they don't yet)
2. PRM judges can reliably assign per-head directional rewards (unvalidated)
3. OPD hint extraction produces actionable token-level advantages from sparse post-date feedback (speculative)
4. The training loop converges stably with delayed, noisy, sparse rewards (the hardest open problem in RL for sparse-reward environments)

**The honest assessment:** This section should be labeled "Research Roadmap — Phase 9+" rather than "operational specification." The immediate learning loop (Phase 7 of our proposed plan) should be supervised retraining on accumulated outcome data with hierarchical signal weighting. RL enters the picture only when supervised learning plateaus and there's enough data volume to stabilize policy gradients.

> **Lines 2955-2957:** *"Binary RL and OPD are not competing methods. They share the same PPO loss structure and differ only in how the advantage At is computed."*

**Comment:** Technically correct — the unified advantage `A_t = w_binary * r_final + w_opd * (teacher - student)` is a clean formulation. But the practical challenge is the `w_opd` term. OPD requires a "teacher" that produces token-level advantages from extracted hints. In matchmaking, a "hint" might be post-date feedback like "we had nothing in common." Converting this natural language feedback into token-level advantage supervision over the model's match decision parameters requires: (1) a PRM judge that reliably extracts directional hints, (2) a hint-enhanced "teacher" model that can be queried for what-if predictions, (3) stable token-level credit assignment from the hint to the decision tokens. Each of these is a non-trivial ML system in its own right.

---

### Milestone Timeline (Lines 2772-2920) — Aggressive

> **Lines 2795-2817:** *"Milestone 2 — Automated Matchmaking... Period: April 2026 (2–5 weeks)... Deploy OpenClaw agent operating system as matchmaking infrastructure"*

**Comment:** 2-5 weeks is feasible for automation if OpenClaw is adopted as-is (not built from scratch). The deliverable — "Agent skills/plugins replace manual expert decisions" — requires: (1) OpenClaw gateway deployed, (2) `@ditto/matchmaking` plugin with custom tools, (3) match reviewer agent with standing orders, (4) at least 3 composable skills, (5) cron scheduling for match cycles, (6) coach notification via channels. With OpenClaw's existing infrastructure, this is achievable in 4-5 weeks. But the 3.x doc treats this as table stakes before the "real" work — it should be treated as the most important milestone because it delivers immediate product value (automated matching) and generates the decision trace data that everything downstream depends on.

> **Lines 2821-2847:** *"Milestone 3 — Intelligent Decision System... Period: April – June 2026 (2–3 months)... Runs in parallel with Milestone 2."*

**Comment:** This milestone proposes shipping, in 2-3 months:
- Shared representation backbone (3a)
- Temporal user state encoder with three-timescale updates and 4 robustness guardrails (3b)
- Likelihood head with directional feasibility (3c)
- Intensity head with state-conditioned ranking (3d)
- Chemistry head with residual uplift modeling (3e)
- Readiness head with timing-dependent conversion (3f)
- Calibration layer across all four heads (3g)
- Gated rerank policy (3h)

**This is 8 major ML components in 8-12 weeks, running in parallel with Milestone 2.** For context: a single well-calibrated binary classifier (the Likelihood head alone) typically requires 2-3 weeks for data preparation, 2-3 weeks for model development, 1-2 weeks for calibration, 1-2 weeks for shadow testing, and 1 week for production deployment — minimum 7-11 weeks for one head. Four heads plus a backbone plus a state encoder plus calibration in the same timeframe is not realistic without a team of 5+ experienced ML engineers working full-time. The team list (line 7) shows ~12 people across multiple roles (engineering, product, design), not all ML.

**Recommendation:** Sequence, don't parallelize. Ship Likelihood first (10 weeks). Then Intensity (8 weeks). Evaluate Chemistry and Readiness only after the first two heads prove their value. Total: ~18-20 weeks for two heads with confidence, not 8-12 weeks for four heads with uncertainty.

---

### Data Infrastructure (Lines 1526-1966) — Correct and Critical

> **Lines 1555-1558:** *"For every exposure or surfaced candidate, the system should be able to reconstruct: what candidate set was retrieved, what representations or features were used, what each head predicted..."*

**Comment:** This is the single most important requirement in the entire document. Decision traceability is the foundation for: debugging model behavior, evaluating policy changes, attributing outcomes to decisions, training future models, and building trust with the product team. The 3.x doc correctly makes this a gate condition for Phase 3. Our proposed plan makes this Phase 0 — the very first thing to build. Every week of matching that runs without decision traces is data that can never be recovered.

> **Lines 1792-1803:** *"A model should never be trained on features that were not actually available at decision time... The infrastructure must prevent hindsight leakage."*

**Comment:** Critical technical principle that most teams violate accidentally. Concretely: if you train a Likelihood model and one of the features is "conversation depth," that feature was only available *after* the match was made, not at decision time. This is temporal leakage and it makes offline evaluation misleadingly optimistic. The fix: every training example must reconstruct the feature vector as it existed at the moment the decision was made, using the time-indexed state snapshots described in Section D (lines 1665-1685). This is hard to implement correctly and easy to get wrong.

---

### Risks (Lines 2595-2770) — Comprehensive, Well-Specified

> **Lines 2655-2670:** *"R4. Readiness Head Absorbs Everything... Without constraints, it can absorb attractiveness, popularity, segment effects, and baseline desirability — becoming a second Likelihood model."*

**Comment:** This is the most technically insightful risk in the document. The mitigation — "Constrain Readiness features to temporal and state-derived signals only; regularize against correlation with Likelihood outputs during training" — is correct. But the document doesn't go far enough: if the diagnostic test (line 2662, "Removing the Readiness Head barely changes system behavior") comes true, the correct response is not "fix the Readiness Head." It's "demote Readiness to input features within the Likelihood head" (as acknowledged in line 2670). This should be the **default** approach, with Readiness as a separate head being the exception that must prove itself.

> **Lines 2695-2712:** *"R6. Chemistry Produces Noise, Not Uplift... if the signal is too weak, the training labels too noisy, or the exploration budget too loose, Chemistry injects randomness rather than real uplift."*

**Comment:** This risk should have higher prominence. The Chemistry Head is the most theoretically appealing component (who doesn't want to predict "spark"?) but also the hardest to validate empirically. The training target — residual uplift over Likelihood + Intensity baseline — is inherently noisy because the "baseline" itself has prediction error. If Likelihood + Intensity have AUC of 0.75 (good but not perfect), the residual contains both true chemistry AND model error. Separating the two requires either: (a) very large data volume to average out model error, or (b) explicit features that capture chemistry but not feasibility/intensity (which the document attempts in Sections A-E, lines 575-609, but these are speculative). **Recommendation:** Budget for Chemistry to fail. Have a plan for what the system looks like without it.

---

### Ancient Systems Section (Lines 643-704) — Remove

> **Line 639:** *"Traditional compatibility frameworks (Vedic, Chinese, Ayurvedic) offer structured priors for Chemistry feature ideation"*

**Comment:** While the caveats are appropriate ("hypotheses to test, not truths to obey"), including this in the architectural specification creates three problems: (1) It consumes disproportionate discussion time relative to its feature engineering value, (2) It creates confirmation bias risk — looking for patterns that validate pre-selected frameworks rather than letting data speak, (3) It's a PR liability ("dating app uses astrology"). The actual feature engineering value — "complementary temperaments," "pacing compatibility" — can be derived from behavioral data without invoking ancient frameworks. **Recommendation:** Remove from the strategy document. If specific researchers want to explore these as feature hypotheses, that's fine as internal research.

---

## Summary of Recommendations

### What to Keep from 3.x

1. **The four-question decomposition** (Likelihood, Intensity, Chemistry, Readiness) — correct conceptual framework
2. **Decision traceability** — make this Phase 0
3. **Agent skills/plugins** — directly implementable via OpenClaw
4. **Hierarchical signal weighting** (Tier 1-4) — critical for avoiding proxy over-optimization
5. **Counterfactual logging** — essential for unbiased learning
6. **Risk analysis** (R1-R9) — comprehensive and well-specified

### What to Change

1. **Sequence, don't parallelize** — M2 + M3 should not run in parallel
2. **Build on existing features before building new backbone** — XGBoost on engineered features before neural backbone
3. **Chemistry and Readiness are conditional, not core** — must prove value before production
4. **RL is Phase 9+, not Phase 4** — supervised retraining first
5. **Remove ancient systems section** — feature engineering from behavioral data
6. **Add concrete ML architecture specifications** — what models, what loss functions, what training data construction
7. **Add explicit data volume thresholds** — minimum pairs needed before each head is viable
8. **Address the global matching problem** — the doc describes per-pair scoring but Ditto uses batch assignment (stable matching). The gated rerank pipeline needs to feed into an assignment algorithm, not just produce a ranked list.
