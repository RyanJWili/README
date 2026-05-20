# Ditto Matchmaking Engine 3.x

![image.png](Ditto%20Matchmaking%20Engine%203%20x/image.png)

**Status:** Internal Strategy Draft

**Lead members:** @Gavin Wang  @Sylvia Liu @Charlie Zhuang    @Praveen Kuruvangi Parameshwara @Nathan Standiford  @Nelson Cheuk-Yam Siu  @Eric Liu  @Rajat Jaiswal  @Yuki Han @Muskan Shaikh @Ryan Willis @Zax Shen @Leon Wang 

---

![image.png](Ditto%20Matchmaking%20Engine%203%20x/image%201.png)

Over the past three months, Ditto’s matchmaking progress has been driven substantially by feature engineering, culminating in the current [seven-feature](https://www.notion.so/Matchmaker-3-0-1-7-Features-311d2ecb07cc80dc83c5c8306de2bee6?pvs=21) foundation led by @Sylvia Liu @Nelson Cheuk-Yam Siu @Nathan Standiford @Eric Liu  for pair evaluation. That work remains valuable. Matchmaking 3.x combines a shared representation backbone, temporal user-state modeling, four explicit decision heads, calibrated policy control, and a downstream learning flywheel to improve real relationship outcomes over time.

![image.png](Ditto%20Matchmaking%20Engine%203%20x/image%202.png)

## What Is  Matchmaking 3.x?

A matchmaking system asks: *which pair should be brought together now, under uncertainty, with limited feedback, asymmetric preferences, and meaningful downstream consequences?*

It is inherently **two-sided**. It is not enough for one person to like another; the system must reason about mutuality. It is inherently **temporal**. A pair that is right in one moment may be wrong in another because receptivity, intent, energy, and recent experiences change. It is also **sparse**, **delayed**, **noisy**, and **partially observed**. Most candidate pairs are never shown. Most shown pairs never produce a deep signal. And the most important outcomes—real chemistry, date quality, continuation—often emerge later and only partially on-platform.

Similarity can be useful for retrieval, but it cannot serve as the full decision engine. It collapses distinct questions into a single score and, in doing so, destroys distinctions the product cannot afford to lose. A strong matchmaking system must separately reason about whether a pair is viable, how strong the interaction is likely to be, whether there is upside beyond the obvious, and whether it is the right moment to surface that pair.

That is why Ditto’s matchmaking 3.x is not “better ranking.” It is a transition to a **stateful, multi-head matchmaking decision system**.

1. **hard constraints** to define the valid solution space
2. a **shared representation and retrieval layer** to make search tractable at scale
3. a **temporal user-state layer** to model evolving taste, receptivity, and recent outcomes
4. **four explicit matchmaking heads** to preserve distinct decision semantics:
    - **Likelihood** — can this pair realistically work?
    - **Intensity** — how strong will it be if it works?
    - **Chemistry** — is there additional upside beyond the obvious?
    - **Readiness** — is this the right moment for this pair?
5. a **calibrated decision layer**, with **agent skills/plugins** that transform head outputs into match decisions
6. a **learning layer** that improves over time from downstream outcomes by learning from experience and simulation

![image.png](Ditto%20Matchmaking%20Engine%203%20x/image%203.png)

## **The Real Product Reality of Matchmaking**

To design the right engine, we need to begin from the actual product reality rather than from recommender-system defaults.

Matchmaking is a sparse-feedback environment. Meaningful outcomes are rare relative to the number of candidate pairs that could exist. Most possible pairs are never shown. Of those shown, many are ignored. Of those that match, only some progress into conversation. Of those conversations, only some lead to contact exchange, dates, or continued interaction.

Matchmaking is also a delayed-feedback environment. The most important signals often arrive much later than the match itself. A user might match today, chat tomorrow, exchange contact details several days later, go on a date the following week, and only then reveal whether the interaction was actually meaningful. This delay makes naive online learning brittle.

It is also a noisy environment. Not every rejection means low compatibility. Not every match means genuine interest. Not every conversation indicates long-term potential. Timing, mood, availability, readiness, competing options, and external life context all affect outcomes.

And critically, matchmaking is a partially observed environment. A great deal of the most important signal lives outside the immediate product surface. The product may see likes, matches, messages, and feedback forms, but not the full quality of an in-person interaction, the emotional texture of a date, or the continuation dynamics that unfold off platform.

This means the system cannot be trained as though early interaction signals are equivalent to ground truth. They are not. They are proxies of varying quality.

At a minimum, we should distinguish the following levels of outcome:

- **Upstream proxies:** impressions, likes, skips, profile saves
- **Mid-funnel outcomes:** matches, chat starts, message depth, contact exchange
- **Downstream outcomes:** dates, post-date sentiment, second-date likelihood, continuation quality

These signals matter differently. They should not be blended naively.

The product reality is also inherently **two-sided**. Every match is a pair decision, not a one-way recommendation. The system must reason about directional likelihoods, mutual compatibility, and pair-level interaction effects.

On top of that, preferences are not static. A user’s taste state changes with time, experience, recent interactions, disappointment, excitement, preference edits, and the types of profiles they have recently seen. A candidate who is correct for a user in one week may be wrong the next week, not because the candidate changed, but because the user's state changed.

Finally, chemistry is only partially observable and cannot be reduced to simple profile overlap. Some of it is latent. Some of it emerges from complementarity. Some of it only becomes visible once feasibility is already satisfied.

This is why Ditto’s matchmaking engine should not be framed as a generic recommender. It is closer to a stateful, two-sided decision system operating under delayed and uncertain feedback, where the goal is to improve real human outcomes rather than optimize a shallow click surface

## **Why Similarity-based Ranking Falls Short**

Most matching systems begin from a simple assumption: if two users are sufficiently similar across enough dimensions, they should rank highly for one another.

That assumption is intuitive, but it is incomplete.

Similarity may help identify plausible candidates as a filtering approach, but quality matchmaking entails at least four distinct questions:

- **Likelihood:** can this pair realistically convert into a mutual match or viable interaction?
- **Intensity:** among viable pairs, which ones are likely to generate stronger pull, deeper engagement, and more meaningful follow-through?
- **Chemistry:** within the set of viable and potentially strong matches, where is there positive novelty, complementarity, or asymmetry that creates spark rather than bland compatibility?
- **Readiness:** even if a pair is good in principle, is this the right time to surface it for both people?

A similarity-based system cannot answer all four well. Once these questions are collapsed into one scalar, the engine loses decision semantics. It can no longer distinguish:

- a pair that is mutually viable but low-energy
- a pair that is strong but mistimed
- a pair that is safe but flat
- a pair that is feasible, timely, and unexpectedly high-upside

As a result, the system becomes conservative in the wrong way. It repeatedly serves what is easiest to justify on paper rather than what is most likely to create real-world outcomes.

This leads to predictable structural failures.

First, static ranking **overweights resemblance**. Users who look similar across profile attributes are repeatedly surfaced even when those matches lead to weak downstream outcomes.

Second, it **under-models mutuality**. Knowing that A may like B is not enough. Matchmaking requires reasoning about whether B is likely to like A, whether both are receptive, and whether the pair is viable now rather than only in the abstract.

Third, it **misses bounded novelty**. Some of the best matches are not those with maximum overlap, but those with the right kind of complementarity or non-obvious fit. Similarity alone has no principled way to represent that.

Fourth, it **learns the wrong lessons from downstream data**. If the ranking formulation is wrong, more data does not solve the core problem. It simply makes the system more confident inside a compressed and mis-specified objective.

Finally, it **converges toward local optima**. The engine keeps serving predictable archetypes, repeated patterns, and low-variance outcomes because those are easiest to defend under a narrow notion of relevance.

For Ditto, this is not enough. The goal is not to maximize shallow engagement on a weak proxy surface. Rather, it is to improve real matchmaking outcomes over time. That requires an engine that preserves the distinctions static ranking destroys.

## **What the Matchmaking Must Be Able to Decide**

A serious matchmaking engine must be able to answer at least four different decision questions for every candidate pair:

### **1. Can this pair realistically work?**

This is the feasibility question. It requires reasoning about directionality, mutuality, and basic pair viability rather than surface resemblance alone.

### **2. How strong is this likely to be if it works?**

This is the interaction-strength question. Among viable pairs, some will produce weak, low-energy interactions while others will create momentum, engagement depth, and follow-through.

### **3. Is there upside beyond the obvious?**

This is the bounded-novelty question. Some pairs may outperform baseline expectations because of complementarity, asymmetry, or latent interaction effects that are not reducible to profile overlap.

### **4. Is this the right moment for this pair?**

This is the timing question. Even a strong pair can fail if surfaced when one or both users are not receptive, fatigued, distracted, or misaligned in short-term intent.

These are not cosmetic distinctions. **They are the core *semantics* of matchmaking judgment**.

Any system that cannot represent them separately will eventually blur them together, make brittle tradeoffs, and become difficult to govern. That is why Ditto’s next architecture must be built to preserve them explicitly.

---

## **Five Pillars of Ditto Matchmaking Intelligence 3.X**

1. a **shared representation backbone** across users, preferences, pair features, and interaction signals
2. a **temporal user-state model** that tracks evolving taste, readiness, and recent outcomes
3. four explicit, gated matchmaking heads:
    - **Likelihood** for feasibility and mutuality
    - **Intensity** for the strength of pull
    - **Chemistry** for bounded novelty, complementarity, etc
    - **Readiness** → whether this is the right moment for the pair
    
    ***New heads can be added as the system matures.** A new head is warranted when outcome data reveals a distinct decision dimension that existing heads cannot separate—for example, how likely the match will be accepted by both and long-term relationship potential—and when that dimension can be independently labeled, trained, and evaluated without collapsing into an existing head.*
    
4. **a calibration and modular decision layer** that transforms raw head outputs into decision-ready scores, while policy execution occurs through composable **agent skills/plugins**—modular, standalone units of matchmaking logic that can be added, tested, tuned, or retired without changing the core engine. These agent skills/plugins act as modular policy operators on top of calibrated model outputs, allowing the decision layer to **evolve rapidly, safely, and at scale without requiring changes to the underlying representation or prediction engin**e.
5. a **matchmaking learning layer** that converts real-world outcomes into persistent system learning (e.g., case-based signals), enabling policy refinement, better diagnostics, and continuous improvement over time

Over time, this flywheel allows Ditto to evolve from a matching product into a **self-improving decision intelligence system for human relationships**—and ultimately, a category-defining agentic social network.

![image.png](Ditto%20Matchmaking%20Engine%203%20x/image%204.png)

---

## **Architecture Overview**

This architecture below is the minimum coherent response to the actual structure of the problem.

![image.png](Ditto%20Matchmaking%20Engine%203%20x/image%205.png)

At its core, this transition reframes matchmaking from a ranking problem into **experience-driven, multi-head control system under uncertainty**.

At the top sits the **hard constraints layer**. This is a deterministic filter, not a learned one. It enforces eligibility and trust boundaries such as orientation, age range, distance limits, relationship intent, safety rules, and blocks. This layer is never violated. It exists to protect product integrity and ensure that learning only happens inside the valid solution space.

Next is the **unified representation and fast retrieval layer**. This layer encodes users, preferences, content, pairwise signals, and relevant interaction context into a shared space that supports scalable candidate generation. The point here is not to make the final decision; it is to create a semantically meaningful, high-recall candidate pool under the hard constraints.

After retrieval comes the **temporal user state encoder**. This layer updates the active representation of a user using recent behavior and outcomes, including likes, dislikes, matches, messaging patterns, dates, post-date feedback, and preference changes gathered through conversational AI or other product surfaces. The goal is to model who the user appears to want **now**. In practice, this is a critical step that turns a static matching system into a stateful decision system.

From this state-aware candidate pool, the engine computes four explicit matchmaking heads. Each head corresponds to a distinct decision dimension and is **learned through neural models over shared representations and pairwise features**. Together, they restore structured decision semantics that would otherwise collapse into a single opaque score, enabling the system to reason separately about **viability**, **strength**, **novelty, and readiness**.

### **Head 1: Likelihood (Feasibility & Mutuality)**

The **Likelihood Head** estimates whether a pair is *actually viable*.

It models directional probabilities such as:

- **P(A → B)**: probability that A will positively respond to B
- **P(B → A)**: probability that B will positively respond to A

These are combined into a **mutual feasibility signal**, which represents the likelihood that the pair can realistically convert into a match or meaningful interaction.

This head functions as the **primary gate** of the system. Its role is to:

- eliminate low-probability or one-sided pairs
- reduce wasted exposure
- ensure that downstream ranking operates only on candidates that are plausibly viable *now*, not just in theory

Without this head, the system confuses surface similarity with real-world compatibility.

---

### **Head 2: Intensity (Strength of Interaction)**

The **Intensity Head** estimates how *strong* a match is likely to be, *conditional on feasibility*. Intensity tells the system which pairs will **pull better**, whereas Likelihood tells the system which pairs can work.

It predicts expected depth and quality of interaction, including signals such as:

- likelihood of conversation initiation
- expected message depth and engagement
- probability of progression (e.g., contact exchange, date)

Its role is to distinguish between:

- **“possible” matches** (technically viable but low energy)
- **“compelling” matches** (high likelihood of meaningful engagement)

This head drives **primary ranking among feasible candidates**, prioritizing matches that are more likely to create momentum and real outcomes.

Without intensity modeling, the system overproduces safe but low-impact matches.

---

### **Head 3: Chemistry (Bounded Novelty & Complementarity)**

The **Chemistry Head** estimates **bounded positive novelty**—the potential for a pair to outperform baseline expectations due to complementarity, asymmetry, non-obvious interaction structure, or latent pair effects not fully captured by direct preference satisfaction or surface compatibility. In particular, it is intended to detect cases where a pair may seem only weakly compatible under standard matching signals, yet still contains meaningful upside because of deeper relational fit, hidden resonance, or emergent interaction dynamics.

It captures match potential that is not fully explained by:

- direct preference satisfaction
- profile similarity
- baseline feasibility

Instead, it focuses on **interaction effects**, such as:

- complementary traits or lifestyles
- asymmetric but compatible preferences
- non-obvious pair dynamics that may create “spark”

Critically, this head is **constrained by feasibility**:

- it does **not** override strong negative signals
- it operates only within **feasibility-safe regions**
- it introduces **controlled exploration**, not unbounded creativity

Its role is to:

- surface high-upside, non-obvious matches
- prevent the system from collapsing into repetitive, low-variance patterns
- enable discovery without degrading overall quality

In system terms, chemistry represents the **residual match potential** beyond what is explained by feasibility and strength.

### **Head 4: Readiness (Temporal Alignment & Receptivity)**

The **Readiness Head** estimates whether this pair is being surfaced at the **right moment** for both users.

It models dynamic, time-sensitive factors such as:

- current responsiveness and activity level
- recent interaction outcomes (positive or negative)
- conversational bandwidth and intent
- short-term preference shifts or fatigue
- alignment of both users’ current engagement state

Unlike feasibility, which answers *“can this work?”*, readiness answers:

> *“is this the right time for this to work?”*
> 

This distinction is critical.

A pair may be:

- highly feasible
- potentially strong
- even high in latent chemistry

—but still fail if surfaced at the wrong moment.

The role of the Readiness Head is to:

- reduce mistimed exposure
- improve conversion from match → conversation → outcome
- avoid wasting high-quality candidates when users are not receptive
- synchronize pair timing rather than treating users independently

Without readiness modeling, the system suffers from **temporal misalignment**:

- good matches shown too early or too late
- decreased response rates despite high compatibility
- misleading negative signals that corrupt learning

In system terms, readiness acts as a **temporal gate and multiplier** on otherwise strong candidates.

---

#### **Why This Decomposition Matters**

These four heads correspond to four distinct questions:

1. **Likelihood:** *Can this pair realistically work?*
2. **Intensity:** *How strong will it be if it works?*
3. **Chemistry:** *Is there additional upside beyond the obvious?*
4. **Readiness:** *Is this the right moment for this pair?*

Separating these neural-network-based signals allows the system to:

- make more interpretable decisions
- apply controlled policy (e.g., gating, reranking, exploration)
- learn from different outcome signals at different stages of the funnel
- avoid collapsing all matchmaking quality into a single opaque score

This structure is what transforms matchmaking from a ranking problem into a **multi-dimensional decision system**.

However, these four heads are not intended to be exhaustive or fixed.

They represent the **current core abstraction of matchmaking intelligence**, based on what we understand today. As the system scales—through increased user volume, richer interaction data, newly observed behavioral patterns, and insights from domain experts—the architecture is designed to **accommodate additional heads**.

Examples of future extensions may include

- **Communication Compatibility** (will interaction styles align?)
- **Long-Term Potential** (does this pair sustain beyond initial engagement?)
- **Outcome-Specific Heads** → e.g., date likelihood vs relationship continuation

These new heads can be introduced **without restructuring the core system**, as they plug into the same representation, state, calibration, and policy layers.

This extensibility is critical.

It means the system is not bounded by a fixed definition of matchmaking quality. Instead, it can **continuously expand its decision surface** as we discover new patterns, collect better data, and refine our understanding of human relationships.

In that sense, the multi-head architecture is not just a modeling choice—it is the foundation for **open-ended, continuously growing matchmaking intelligence**.

These raw predictions are then transformed through a **Calibration Layer with Agent Skills / Plugins**.

This layer serves two tightly coupled roles:

First, it **stabilizes raw model outputs into decision-ready scores**. Calibration ensures that quantities like feasibility, strength, and chemistry become reliable, interpretable, and consistent across segments and time. This enables stable thresholds, predictable behavior, and meaningful product control.

Second, it provides a **modular decision surface via composable agent skills and plugins**. These are independently deployable units of matchmaking logic that operate on top of calibrated signals. Examples include:

- thresholding and gating policies (e.g., minimum mutual feasibility)
- diversity and freshness injectors
- exploration budget controllers
- cohort-specific adjustments
- safety or trust-aware modifiers
- product-specific heuristics and experiments

These skills do not retrain or alter the core models. Instead, they **compose around calibrated outputs**, allowing the decision policy to evolve rapidly without requiring changes to the underlying representation or prediction layers.

Because these plugins are modular and decoupled:

- they can be added, tested, tuned, or removed independently
- experimentation becomes faster and safer
- policy iteration does not require full model retraining cycles
- system behavior remains observable and controllable

On top of this calibrated and modular layer sits the **Gated Rerank Policy**, which executes the final decision logic.

Candidates below feasibility thresholds are removed. The remaining pool is reranked primarily by expected strength, with chemistry applied as a bounded modifier within feasibility-safe bands. Agent skills/plugins can further shape this stage by injecting diversity, enforcing exploration limits, or applying product-specific policies in a controlled and auditable way.

Finally, once matches are delivered and real user behavior unfolds, a **Post-Match and Post-Date Feedback Loop** updates both the user-state layer and the model training pipeline. Fast feedback updates refine user state in near real time, while slower aggregation of downstream outcomes improves model parameters and calibration over longer learning cycles.

---

## **Zoom into Unified Representation Space**

The representation layer is one of the most important long-term investments in the system.

Many matching systems evolve by accumulating disconnected feature pipelines: structured profile attributes in one place, text embeddings in another, preference filters elsewhere, image signals in another silo, and pairwise heuristics layered on top. This fragmentation limits generalization, increases duplicated logic, and makes improvement unnecessarily slow. Worse, it creates **hidden divergence**: the features used for retrieval silently drift from the features used for scoring, so the candidate pool and the ranker are no longer aligned—a root cause of hard-to-diagnose quality problems.

The better approach is to build a **shared, modular representation backbone** that encodes heterogeneous inputs within a common framework while preserving their different roles.

### **Key Design Principle: Pair-Level Intelligence is First-Class**

Matchmaking quality does not emerge solely from user-level representations.

It fundamentally depends on **interactions between representations**.

The system must explicitly support:

- **Preference satisfaction** (A wants B, B wants A)
- **Trait complementarity** (differences that create synergy)
- **Symmetry / asymmetry patterns** (imbalanced but viable dynamics)
- **Non-linear interactions** (features whose value depends on context)

These should not be left for downstream models to rediscover implicitly.

Instead, the representation layer should **surface them as structured, learnable signals**.

### **What the backbone must encode**

At minimum, the backbone should ingest and align these signal families:

| Signal family | Examples | Role |
| --- | --- | --- |
| **Structured profile attributes** | Age, location, education, occupation, relationship intent, lifestyle tags | Hard constraints, coarse retrieval filters, explicit matching dimensions |
| **Free-text self-description** | Bio, prompts, open-ended answers | Soft personality, values, communication style, humor |
| **Explicit preferences** | Declared dealbreakers, age/distance ranges, trait priorities | Directional filtering, preference-satisfaction features |
| **Implicit preferences** | click patterns, dwell time, skip velocity, archetype drift | Revealed taste, latent interest beyond declared preferences |
| **Image and photo-style signals** | Photo embeddings, aesthetic style, scene context, photo ordering | Visual attraction, lifestyle cues, presentation personality |
| **Historical interaction summaries** | Chat depth, response latency, contact exchange, dates, post-date feedback | Outcome-weighted behavioral profile, engagement quality |
| **Pair cross-features** | Preference satisfaction, trait complementarity, symmetry, directional asymmetry, interaction-style fit | Pair-level compatibility structure that does not live in either user vector alone |

Each family carries different **statistical properties** (dense vs sparse, continuous vs categorical, user-level vs pair-level, stable vs fast-changing). The backbone must respect those differences rather than flatten everything into one undifferentiated embedding.

### **Why hybrid fusion, not a single embedding model**

A crucial principle is that the representation layer should support **compositional feature integration**—not just one dense encoder.

Research on efficient multi-task architectures demonstrates why. Work on joint intent-and-entity models has shown that **fusing** cheap sparse lexical features (discrete tokens, character n-grams) with dense pretrained embeddings inside a **single shared encoder** can outperform fine-tuning a large language model alone—while training many times faster. The key insight: **no single representation family dominates**. Sparse features preserve exact structure that dense models compress away; dense features capture latent semantics that sparse features cannot generalize; and a **shared encoder** forces both families into a common space so every downstream head benefits from both simultaneously.

The same principle transfers directly to matchmaking:

- **Dense semantic representations** (text embeddings, image embeddings, conversational encoders) capture latent meaning, soft similarity, and generalization across users. They are essential for open-ended signals like bios, chat style, and visual presentation.
- **Structured and sparse signals** (categorical attributes, bucketed behavioral counts, hashed interaction patterns, explicit preference flags) preserve **interpretable constraints**, exact values, and relational detail that purely neural compression often washes out. They are essential for governed decision-making: hard filters, auditability, calibration diagnostics, and cohort analysis all depend on features that remain individually readable.
- **Pair-level cross-features** (preference satisfaction scores, trait complementarity tensors, asymmetry indicators, directional interaction effects) capture compatibility structure that **does not live in either user vector alone**. Matchmaking is fundamentally a relation, not a sum of two profiles.

A monolithic "embed everything into one vector" approach loses the second and third families. A siloed approach loses the first. The right design is **compositional fusion**: each signal family has its own **featurization pathway** (a modality tower or encoder), a **learned projection** aligns dimensions into a common space, and a **shared encoder** (transformer or equivalent) combines them so all downstream consumers—retrieval and every scoring head—operate on the **same representational geometry**.

**Why this matters specifically for dating surfaces.** Short bios, taps, skips, and preference clicks behave more like **task-oriented sparse feedback** than long-form prose. A generic document-level language model is the wrong single hammer. Empirical results from multi-task NLU research show that even a purely supervised sparse-feature setup can outperform frozen large-model embeddings, and that **domain-matched** dense encoders (trained on conversational rather than generic web data) consistently beat generic ones by a wide margin. The same logic holds here: the backbone should deliberately combine explicit tabular attributes, high-recall sparse behavioral features, and dense vectors from encoders **chosen or fine-tuned for social, conversational, and visual domains**—not only generic web-text models. Per-modality towers can evolve independently (swap text encoder, upgrade image model, add a new conversational summarizer) while the **fusion contract** to the rest of the engine stays stable.

### **Pair-level structure: fuse early, not only in late heuristics**

Quality does not arise only from user-level representations. It also arises from **pair-level interaction structure**: preference satisfaction, trait complementarity, asymmetry, directional effects, and compatibility patterns that live in cross-features.

The representation layer should **materialize these interactions in the shared space** rather than leaving them to be inferred implicitly by downstream heads from isolated user embeddings. Concretely, this means:

- **Explicit pair tensors**: compute cross-features (e.g. trait-A × trait-B interactions, preference-satisfaction vectors, asymmetry scores) and feed them into the shared encoder alongside user-level features.
- **Interaction layers**: learned bilinear or attention-based layers that model how one user’s attributes relate to the other’s, producing pair-level representations that are richer than concatenation or element-wise products.
- **Directional encoding**: model A→B and B→A as distinct representations, since matchmaking is inherently asymmetric (A liking B does not imply B liking A).

Late pairwise heuristics can still exist as lightweight post-processing, but they should complement—not replace—early, learnable pair structure. If the backbone gets pair modeling right, every downstream head (likelihood, intensity, chemistry, readiness) starts from a richer foundation.

### **Single backbone for retrieval and multi-head scoring**

The unified representation layer should serve both **retrieval** and **downstream multi-head scoring**. Candidate generation and the feasibility / intensity / chemistry / readiness models should consume a **common foundation**, analogous to how a single shared transformer in multi-task NLU feeds both a sequence tagger and a sentence classifier simultaneously. That alignment:

- improves **coherence** across stages: no train–serve skew between “retrieval embeddings” and “ranker features,” so what retrieval surfaces is exactly what scoring can evaluate well;
- enables **transfer** of learned structure: improvements to the backbone’s understanding of pair dynamics flow into every head and into retrieval quality simultaneously;
- reduces **parallel, diverging feature stacks** that silently decouple over time and create maintenance debt;
- supports **joint training** where retrieval and scoring losses can optionally back-propagate into the same backbone, creating a tighter learning loop (ablation studies in multi-task NLU have shown that joint training can improve task performance by several absolute F1 points over single-task setups, precisely because correlated objectives reinforce shared representations).

The goal is not a single undifferentiated blob vector. The goal is a **shared but structured representation space**: expressive enough for neural learning, **modular** enough that modalities and pair blocks can be added or upgraded, and **explicit** enough to preserve the signals required for **governed** matchmaking decisions—safety constraints, auditability, cohort analysis, calibration, and controlled exploration.

### **Masking and self-supervised regularization**

Multi-task NLU architectures have shown the value of an auxiliary **masked-input prediction** objective alongside primary classification and tagging losses. The purpose is regularization: by forcing the shared encoder to reconstruct randomly masked inputs, the model learns **more general features** rather than overfitting to narrow classification boundaries.

The same idea applies to the matchmaking backbone. Auxiliary self-supervised objectives—predicting masked profile attributes, reconstructing withheld behavioral signals, or predicting held-out interaction outcomes—can **regularize** the shared space and improve generalization, especially during early training when labeled matchmaking outcomes are sparse. This is not a required component at launch, but it is a **natural extension** once the backbone exists: the architecture should be designed so that auxiliary objectives can be toggled on or off without restructuring the core encoder.

### **Extensibility and the self-improvement loop**

Over time, the unified backbone becomes a compounding systems advantage:

- **New modalities** (voice notes, video intros, richer conversational signals) attach to the **same fusion contract** without redesigning the engine.
- **New pair features** (discovered through outcome analysis, chemistry research, or ancient-wisdom-inspired hypotheses) plug into the cross-feature layer and immediately benefit all downstream heads.
- **New prediction heads** (future extensions like communication compatibility or long-term potential) consume the same shared space, inheriting the backbone’s existing learned structure rather than starting from scratch.
- **Better encoders** (a stronger text model, a fine-tuned dating-domain image tower) can be swapped into their modality slot; the rest of the pipeline does not change.

This is the foundation of a self-improving matchmaking intelligence system. Every improvement to data, features, or encoders compounds into **one** learning surface. Without a unified backbone, each improvement requires re-integration across disconnected stacks. With it, the system gets **structurally** better with every iteration.

---

## **Zoom into Temporal User State and the Four Explicit Heads**

The real intelligence of the system begins once static representations are turned into live decision context.

### **1. Gated Temporal User State Encoder**

The Temporal User State Encoder is the mechanism that prevents Ditto from becoming a static, profile-matching system.

Users do not have one timeless preference vector. Their actual receptivity changes. Their standards shift. Their curiosity changes. Their tolerance for repetition changes. Their willingness to explore changes after good or bad experiences. Their explicit preferences may lag behind what their behavior reveals. Their recent interactions shape what the right next match should be.

The temporal user state should therefore update from:

- recent likes and dislikes
- skipped or ignored profiles
- match acceptances
- chat starts and chat depth
- contact exchanges
- dates
- post-date feedback
- conversational preference edits
- recent exposure to repeated archetypes
- recency and fatigue effects

This state should not simply memorize actions. It should summarize evolving taste and current context. A user who has recently seen too many of one archetype may benefit from higher diversity inside the feasible set. A user who just had a strong outcome may show different receptivity than a user who has experienced several disappointing conversations in a row.

A stateful engine can reason about the same candidate differently depending on when that candidate is surfaced. That is a core advantage.

A highly responsive state encoder introduces sensitivity to noise, outliers, and adversarial behavior. Without explicit safeguards, even a single anomalous session can induce disproportionate state shifts and degrade downstream decision quality. The following control mechanisms should be incorporated:

---

| **Area** | **Goal** | **What to Detect / Classify** | **System Action** | **Why It Matters** |
| --- | --- | --- | --- | --- |
| **Anomaly Detection & Gating** | Prevent abnormal sessions from corrupting state | - Extreme swipe velocity- All-left / all-right streaks- Abnormal acceptance/rejection rates- Irregular session duration | Down-weight or quarantine anomalous signals before state update | Ensures robustness against noise, erratic behavior, or non-representative sessions |
| **Intent Classification** | Distinguish true preference shifts from noise or manipulation | Classify sessions as:- Normal- Exploration- Noise- Manipulation | Apply different update weights or gating based on intent class | Prevents misinterpreting exploration or noise as stable preference changes |
| **Reversion to Prior** | Recover from temporary behavioral anomalies | Detect return-to-baseline behavior after abnormal sessions | Accelerate decay of anomalous influence relative to normal updates | Avoids long-term distortion from short-lived abnormal behavior |
| **Feedback Loop Dampening** | Prevent self-reinforcing negative cycles | Patterns like:- Fatigue ↑ → safer matches → boredom → more fatigue | Introduce momentum dampening in state updates and decision policies | Breaks local minima loops and preserves long-term user experience quality |

Also, the Temporal User State Encoder should apply a recency-weighted attenuation mechanism to user signals. Recent interactions (e.g., within the last 7 days) should carry higher influence, typically accounting for approximately 50–80% of the effective state, as they better reflect the user’s current intent, readiness, and context. Historical interactions should be progressively down-weighted (e.g., 20–50%) to preserve long-term preference signals without allowing outdated behavior to dominate real-time decisions. This attenuation should be continuous rather than discrete, using smooth decay functions to avoid abrupt shifts in state. In addition, the attenuation can be made adaptive—reducing the impact of anomalous or low-confidence sessions more aggressively—so that the encoder remains both responsive and robust to noise or manipulation.

### **2. Likelihood Head**

The Likelihood Head is the feasibility gate.

Its role is to estimate whether the pair is realistically viable. This should include directional likelihoods rather than only a fused score. At a minimum, the model should estimate the probability that A would respond positively to B, the probability that B would respond positively to A, and a derived mutual feasibility quantity.

This head replaces naive similarity as the system’s primary gating signal.

Its purpose is not to rank the entire set perfectly. Its purpose is to eliminate dead ends, reduce wasted exposure, and align the engine with actual pair viability.

Without this head, the system will continue to confuse surface resemblance with actual match feasibility.

### **3. Intensity Head**

Among feasible pairs, not all matches are equally meaningful.

The Intensity Head is responsible for estimating strength of pull: expected engagement depth, response energy, and the likelihood that a match becomes more than a technically valid interaction.

This head is what lifts the system out of low-conviction matchmaking. It helps prioritize matches that feel stronger, more promising, and more likely to create real momentum.

Without a distinct intensity layer, the system tends to overproduce “fine” matches that do not convert into memorable outcomes.

### **4. Chemistry Head**

The Chemistry Head handles bounded emergence.

Its role is to capture the positive novelty that can arise from complementarity, asymmetry, or non-obvious pair structure. It is not there to produce chaos. It is there to identify when a pair may have spark that is not reducible to direct similarity-like algorithms.

This is important because some of the best matches are not the ones with maximum profile overlap. **They are the ones where the interaction structure creates energy**.

But this head must remain controlled. Chemistry should operate only inside feasible ranges. It should never override trust, eligibility, or strong feasibility signals. The right frame is not “creative override.” It is “bounded exploration inside safety.”

Taken together, these four heads reintroduce decision semantics into the engine. They allow the product to ask, separately, whether a match is viable, how strong it is, and whether it has meaningful spark.

The Chemistry Head should focus on **pair-level interaction patterns** that may increase match quality in ways not already explained by feasibility or expected baseline strength.

### **1. Complementarity**

The pair is not simply similar, but their differences are constructive rather than conflicting.

Examples:

- similar values, but different strengths
- one person initiates, the other stabilizes
- one is expansive, the other grounding
- one brings novelty, the other brings emotional steadiness
- one is socially expressive, the other highly attentive

This is not “opposites attract” in the naïve sense. It is closer to **functional complementarity on top of shared foundations**.

### **2. Non-obvious mutual resonance**

The pair may not look maximally compatible on paper, yet their interaction style, tone, pacing, or emotional shape suggests unusual fit.

Examples:

- strong conversational rhythm
- aligned humor despite different backgrounds
- high comfort despite modest attribute overlap
- unusual alignment in vulnerability or curiosity
- one person’s self-presentation specifically resonating with the other’s inferred preferences

### **3. Memorable distinctiveness**

Some pairs contain enough novelty to stand out without becoming destabilizing.

This matters because overly safe systems often produce repetitive, low-variance outcomes. A chemistry-aware system can surface pairs that feel **alive rather than generic**, while still protecting overall quality.

### **4. Emergent pair effects**

There are interaction effects that do not live at the individual level, but at the pair level.

Examples:

- profile A is generally moderate, profile B is generally moderate, but together their trait interaction is highly promising
- candidate B is not universally desirable, but is unusually right for user A
- each person fills a gap in the other’s relational pattern
- the combination produces expected uplift that would not be obvious from either profile alone

This is one reason pair modeling matters so much in matchmaking.

The Chemistry Head should draw from features that are especially relevant to pair-level uplift and emergent interaction quality.

### **A. Complementarity Features**

- similar core values, but distinct functional strengths
- compatible asymmetries in energy, style, or pace
- emotional or behavioral balance rather than exact sameness
- overlap in life direction with difference in execution style

### **B. Interaction-Style Features**

- expected conversational fit
- humor alignment
- candor / vulnerability compatibility
- communication rhythm compatibility
- curiosity symmetry or complementarity
- comfort-level compatibility

### **C. Pair Cross-Features**

- pairwise trait interactions
- inferred role fit
- asymmetry patterns that historically correlate with strong outcomes
- relational structure beyond marginal user descriptors

### **D. Novelty-within-Safety Features**

- how different the pair is from the user’s recent exposure history
- whether the novelty is constructive rather than random
- whether the candidate expands the search space without breaking the viability floor

### **E. History-Aware Features**

- whether the user has been stuck in repetitive match archetypes
- whether similar “safe” matches underperformed recently
- whether the system should selectively broaden the user’s candidate pattern

The key principle is that chemistry should be informed by **interaction potential**, not just profile content.

---

## **How Chemistry Relates to Research**

What people experience as “spark” is usually not reducible to one dimension such as physical attractiveness or declared preference satisfaction. It often includes:

- positive interaction
- mutuality
- comfort
- compatibility
- uniqueness
- conversational ease
- felt momentum

That does **not** mean the system can perfectly predict chemistry.

It means the system should treat chemistry as **multi-dimensional and partially observable**, rather than pretending it is either fully predictable or irrelevant.

The correct product stance is:

> **Chemistry is real, valuable, partly learnable, and partly uncertain.**
> 

That is exactly why it belongs in the architecture as a bounded head rather than as a deterministic rule.

---

*Traditional compatibility frameworks (Vedic, Chinese, Ayurvedic) offer **structured priors for Chemistry feature ideation** — multi-dimensional compatibility, complementarity patterns, energy-type dynamics, and balance/amplification principles. See Appendix A for details.*

---

## **What Ditto Can Learn from Ancient Systems**

If these systems are used carefully, they can contribute in four bounded ways.

### **1. Hypothesis generation**

They suggest candidate dimensions that modern systems may want to test, such as:

- emotional resonance
- constitutional balance
- intellectual friendship
- power-dynamics fit
- pacing compatibility
- temperament complementarity

### **2. Feature engineering**

They can inspire latent pair features or persona embeddings that are then validated against real outcomes.

### **3. Interpretability**

They can provide user-friendly language for why a match may feel resonant, balanced, grounding, energizing, or unusually aligned.

### **4. Cold-start priors**

They may offer weak initial structure before richer behavioral and downstream outcome data accumulate.

This is especially useful for chemistry because chemistry is one of the hardest signals to observe directly early on.

---

## **How Ancient Systems Should Be Used — and How They Should Not**

### **They may be used as**

- structured prior libraries
- exploratory feature families
- persona-clustering hypotheses
- interpretable compatibility vocabularies
- weak cold-start priors

### **They should not be used as**

- hard constraints
- deterministic compatibility rules
- unvalidated ranking overrides
- substitutes for user behavior and real outcomes
- unquestioned truth claims

The balanced principle is:

> **Ancient systems may provide useful priors about compatibility, complementarity, rhythm, and relational structure. But the modern matchmaking engine should treat them as hypotheses to test, not truths to obey.**
> 

In practice, this means the Chemistry Head may draw inspiration from these frameworks when forming pair-level features or priors, but final decisions should remain governed by:

- learned outcomes
- calibration
- feasibility constraints
- timing logic
- policy control

## **5. Readiness Head**

The **Readiness Head** estimates whether this pair is being surfaced at the **right moment** for both users.

Its role is not to decide whether the pair is intrinsically compatible. That work is already handled elsewhere:

- **Likelihood** estimates whether the pair is realistically viable
- **Intensity** estimates how strong the interaction is likely to be
- **Chemistry** estimates whether there is bounded upside beyond the obvious baseline

The Readiness Head asks a different question:

> **Even if this pair is viable, potentially strong, and even high in chemistry, is this the right time to spend exposure on it?**
> 

This distinction matters.

A good pair can still fail if surfaced at the wrong moment:

- one or both users may be fatigued
- one may be disengaged, distracted, or emotionally unavailable
- one may be recovering from recent poor outcomes
- one may be active but not actually receptive
- the pair may be right in principle but mistimed in practice

A system without readiness modeling risks wasting high-quality candidates, producing misleading negative signals, and confusing temporal misalignment with true incompatibility.

That is why readiness deserves its own head.

---

## **Why Readiness Must Be Modeled Separately**

Without a distinct readiness signal, the engine tends to make one of two mistakes.

### **1. It treats all low-response outcomes as compatibility failures**

A user may reject or ignore a strong candidate not because the match is poor, but because the timing is wrong. If the system does not model readiness, it will incorrectly push that evidence into feasibility or preference learning.

This pollutes the learning loop.

### **2. It spends high-quality candidates when the user is not in a state to convert**

Some candidates are too valuable to waste during periods of low receptivity. A strong engine should not only pick good matches. It should pick them **when they are most likely to succeed**.

This is especially important in sparse-feedback environments where every exposure carries opportunity cost.

---

## **What Readiness Is — and What It Is Not**

To keep the architecture clean, readiness must also be defined carefully.

### **Readiness is**

- temporal receptivity
- pair-level timing alignment
- exposure timing quality
- current willingness and capacity to engage
- the probability that now is a good moment to surface this pair

### **Readiness is not**

- baseline compatibility
- long-term relationship potential
- pure activity level
- raw user availability alone
- a rebranding of feasibility
- a generic confidence score

This distinction is important.

A user can be:

- highly active but not emotionally receptive
- open to dating in general but tired of repetitive match archetypes
- a good fit for a candidate overall, but poorly timed this week
- likely to respond positively in principle, but unlikely to convert right now

The Readiness Head exists to capture that gap between **can work** and **right now may work**.

---

## **What the Readiness Head Is Trying to Detect**

The Readiness Head should focus on **time-sensitive, state-sensitive, and pair-sensitive factors** that affect whether surfacing this pair now is a good decision.

### **1. Current receptivity**

Is the user presently in a state where they are likely to respond well to new introductions?

Signals may include:

- recent responsiveness
- app engagement quality, not just quantity
- conversation follow-through
- recent acceptance or rejection patterns
- evidence of fatigue, burnout, or overload

### **2. Temporal alignment**

Even if both users are individually promising, are they aligned **at the same moment**?

Examples:

- both users are currently active and responsive
- both appear open to connection rather than browsing passively
- both are in periods of stable rather than chaotic engagement
- neither is in a temporary low-receptivity trough

### **3. Recovery and opportunity cost**

Has one of the users recently had a disappointing run of outcomes, repeated exposure, or low-quality interactions?

If so, the system may want to avoid spending a high-upside candidate immediately and instead wait for a better decision window.

### **4. Short-term preference drift**

A user’s recent behavior may reveal temporary shifts in what they are receptive to:

- more comfort-seeking than novelty-seeking
- more emotionally safe matches than intense ones
- more conversation-friendly matches than ambitious stretch matches

This is not a permanent preference update. It is a **state-dependent receptivity shift**.

### **5. Timing-sensitive pair fit**

Some pairs are especially sensitive to when they are introduced.

For example:

- a high-chemistry, high-novelty pair may need stronger readiness than a stable, high-likelihood pair
- a user showing fatigue may do better with warmth and ease than with highly demanding novelty
- a user who has recently regained momentum may now be ready for stronger candidates that would have underperformed earlier

This is why readiness is not just a user feature. It is partly a **pair-exposure timing feature**.

---

## **What Inputs Should Inform Readiness**

The Readiness Head should draw from features that help estimate whether now is a good time to surface this pair.

### **A. Engagement-State Features**

- recency of activity
- message reply latency
- app session quality
- match follow-through rate
- recent conversation abandonment
- engagement consistency rather than raw activity spikes

### **B. Outcome-Trajectory Features**

- recent rejections or ignores
- recent strong conversations
- recent dates or post-date outcomes
- disappointment streaks
- signs of renewed openness after a good interaction

### **C. Fatigue and Saturation Features**

- recent exposure to repetitive archetypes
- declining response rates
- skip velocity
- evidence of decision fatigue
- reduced curiosity or shrinking engagement breadth

### **D. Temporal Context Features**

- day-of-week effects
- time-of-day effects
- user-specific responsiveness rhythms
- recent cadence of introductions
- short-term life context inferred from conversational signals

### **E. Pair-Specific Timing Features**

- whether the candidate is better suited for a high-readiness versus low-readiness moment
- whether the pair is high-upside but timing-sensitive
- whether the pair should be delayed until receptivity improves
- whether both users appear aligned in short-term intent and bandwidth

The key principle is that readiness should be informed by **state and timing**, not confused with static compatibility.

---

## **How Readiness Relates to the Temporal User State**

The Temporal User State Encoder and the Readiness Head are closely related, but they are not the same thing.

The **Temporal User State Encoder** summarizes evolving user context:

- taste drift
- fatigue
- recent outcomes
- current behavioral patterns
- openness to exploration

The **Readiness Head** uses that evolving context to answer a more specific decision question:

> **Should this pair be surfaced now?**
> 

So user state is the substrate; readiness is the timing judgment built on top of it.

This separation is useful architecturally:

- the state encoder represents changing context
- the readiness head converts that context into a decision signal

That keeps the system more interpretable and prevents temporal effects from being smeared across the other heads.

---

## **How Readiness Should Affect Decision-Making**

Readiness should not replace feasibility, intensity, or chemistry. It should act as a **temporal gate or multiplier** on otherwise promising candidates.

A clean decision logic is:

1. **Likelihood** determines whether the pair is plausibly viable
2. **Intensity** estimates expected strength among viable pairs
3. **Chemistry** estimates bounded pair-level upside beyond baseline fit
4. **Readiness** determines whether now is a good moment to surface the pair

In practice, this means readiness can:

- filter out mistimed exposures
- downweight strong-but-poorly-timed pairs
- prioritize candidates that are both high-quality and timely
- preserve some high-upside candidates for later rather than spending them immediately

This makes the engine not only smarter about **who**, but also smarter about **when**.

---

## **Illustrative Examples**

### **Example 1: Strong pair, bad timing**

A pair is:

- high in likelihood
- high in intensity
- moderate in chemistry

But one user has:

- low recent responsiveness
- several recent disappointments
- signs of fatigue and reduced follow-through

The system should avoid interpreting a likely rejection as evidence against compatibility and may delay exposure until receptivity improves.

### **Example 2: Moderate pair, excellent timing**

A pair is:

- feasible
- moderately strong
- not especially novel

But both users are:

- responsive
- recently engaged
- showing high follow-through
- aligned in near-term intent

The system may reasonably surface this pair now because timing improves expected conversion.

### **Example 3: High-chemistry pair, timing-sensitive**

A pair has:

- acceptable feasibility
- strong chemistry
- good predicted upside

But chemistry-driven pairs may be more fragile when users are fatigued or unreceptive.

The system may wait until both users are in a better state before surfacing the pair, preserving potential rather than wasting it.

---

## **Training and Labeling Considerations**

Readiness should not be supervised by raw acceptance alone.

A user may accept while being poorly timed, or reject while the match itself is still strong. Readiness labels should therefore be inferred from time-sensitive outcome patterns such as:

- conversion conditional on recent state
- response quality conditional on timing
- conversation start and follow-through conditional on recent receptivity
- timing-dependent lift relative to baseline pair quality
- user-specific responsiveness rhythms

A useful framing is:

> **Train readiness on temporal conversion efficiency, not just one-shot response labels.**
> 

This helps keep readiness from collapsing into either general desirability or activity prediction.

---

## **Evaluation of the Readiness Head**

The Readiness Head should be evaluated by asking whether it improves **timing quality** without suppressing too many good candidates.

Key questions include:

- does readiness improve match-to-conversation conversion?
- does it reduce wasted exposure on low-receptivity moments?
- does it improve downstream follow-through for similar-quality candidates?
- does it reduce false learning from mistimed rejections?
- does it improve candidate allocation efficiency across weeks or sessions?

Important evaluation views include:

### **A. Timing lift**

Do candidates surfaced in high-readiness moments outperform similar candidates surfaced in low-readiness moments?

### **B. Exposure efficiency**

Does readiness help preserve high-upside candidates rather than wasting them during poor timing windows?

### **C. Learning cleanliness**

Does readiness reduce the number of rejections that are incorrectly treated as evidence of incompatibility?

### **D. Segment stability**

Does readiness behave sensibly across user cohorts, or does it overfit to raw activity patterns in misleading ways?

---

## **Why Readiness Is Strategically Important**

Without readiness, the system may still find good pairs, but it will often spend them inefficiently.

That is a major weakness in a sparse, delayed-feedback environment.

A timing-aware engine has three advantages:

- it wastes fewer high-quality opportunities
- it learns more cleanly from outcomes
- it aligns match quality with user receptivity rather than treating exposure as timing-neutral

In that sense, readiness is not a cosmetic addition.

It is one of the mechanisms that turns matchmaking from a static scoring problem into a true **sequential decision system**.

---

## **Calibration, Gated Rerank, and Decision Policy**

Raw model scores are not product decisions.

Even if the four-head structure is correct, we still need a mechanism that turns predictions into stable match outputs. That is the role of calibration and decision policy.

### **1. Why Calibration Matters**

Modern predictive models are often miscalibrated. A model may assign a score of 0.82 without that number corresponding to a stable real-world likelihood. If product thresholds are attached directly to such scores, system behavior becomes brittle and hard to control.

Calibration solves this by making score meaning more stable. It helps transform raw feasibility outputs into quantities that support reliable thresholding, consistent ranking behavior, and predictable product knobs.

For Ditto, calibration is especially important because:

- feedback is delayed
- labels are noisy
- pair outcomes are sparse
- product trust depends on stable quality

A calibrated engine is easier to govern and debug. It also makes it easier to reason about segment-level performance, risk tradeoffs, and exploration budgets.

### **2. Gated Rerank Logic**

Once the system has calibrated outputs, it can apply a disciplined decision policy.

A plausible decision sequence is:

1. enforce hard constraints
2. retrieve a broad, semantically relevant candidate set
3. compute temporal-state-aware pair scores
4. remove candidates below mutual feasibility threshold
5. rerank remaining candidates using intensity
6. apply chemistry as a bounded modifier within feasible bands
7. respect any exploration, fairness, diversity, or freshness rules
8. commit final match outputs

This makes the engine understandable. It also gives us clear levers.

If the product needs more safety, we adjust feasibility thresholds.

If we need stronger quality among already feasible pairs, we focus on intensity.

If we need more spark without destabilizing quality, we adjust chemistry budgets or bounded novelty logic.

If calibration drifts, we can observe that directly rather than blaming the whole engine.

### **3. Exploration Without Recklessness**

Any high-performing matchmaking system eventually faces the exploration problem. If the engine only exploits what it already knows, it may become narrow and repetitive. If it explores too aggressively, it can degrade experience and trust.

The right answer is not unrestricted novelty. The right answer is governed exploration.

That means chemistry should be allowed to promote candidates only within feasibility-safe bands, with explicit budgets and measurable outcomes. Exploration should be framed as a controlled resource, not as an unbounded model impulse.

This is one of the places where Ditto can become substantially better than simpler systems. We can be more adaptive without becoming erratic.

**Calibration + Agent Skills / Plugins** → *turns raw head outputs into stable, decision-ready signals and applies modular policy logic such as thresholding, diversity, freshness, bounded exploration, and trust-aware controls before final match commitment.*

### **1. Agent Skills / Plugins**

Even calibrated model outputs are still not enough to define final product behavior.

The system also needs a modular layer that can apply governed policy on top of calibrated signals. This is the role of **agent skills / plugins**.

Agent skills / plugins are **modular policy operators** that act on calibrated model outputs without changing the underlying representation, temporal state, or prediction heads. They allow Ditto to evolve decision logic quickly, safely, and observably without retraining core models every time policy changes.

In other words:

> **The models estimate what is true. The plugins decide how product policy should act on those truths.**
> 

This is strategically important because not every product rule belongs inside the models themselves.

Some logic is better handled as modular policy:

- easier to experiment with
- easier to audit
- faster to change
- safer to remove
- more transparent to product and engineering teams

---

### **2. What Agent Skills / Plugins Should Do**

Agent skills / plugins should apply decision logic that sits **around** the calibrated outputs rather than **inside** the core predictive models.

Examples include:

- minimum mutual-feasibility thresholds
- readiness-aware delay or holdout policies
- diversity injectors
- freshness controllers
- exploration budget controllers
- anti-repetition rules
- trust and safety modifiers
- cohort-specific adjustments
- product-specific experiments
- fallback logic for sparse-data or cold-start situations

These plugins should be:

- independently deployable
- composable
- measurable
- easy to add, tune, or retire
- constrained by auditability and policy control

This modularity allows the decision surface to evolve faster than the core models.

For example:

- a new diversity policy can be tested without retraining the Likelihood Head
- a new anti-fatigue rule can be introduced without changing the state encoder
- a new exploration budget can be tuned without rewriting chemistry modeling
- a trust-aware modifier can be applied without touching the representation backbone

That is exactly the kind of flexibility a live matchmaking system needs.

---

### **3. What Agent Skills / Plugins Should Not Do**

The boundary here is extremely important.

Plugins should **not** become:

- a dumping ground for model weaknesses
- a hidden ranking system
- a substitute for poor representation learning
- a workaround for missing pair semantics
- unbounded heuristic overrides on top of model outputs

If the system starts using plugins to compensate for broken feasibility modeling, weak temporal state, or poor chemistry learning, then the architecture becomes fragile and hard to reason about.

So the governing principle should be:

> **Agent skills / plugins should shape policy around calibrated outputs; they should not replace the underlying representation, temporal state, or prediction layers.**
> 

That keeps the system disciplined.

---

# **Learning & Feedback System**

*Most matching systems ship a model and hope. This one learns.*

*The system closes the loop between **decision → outcome → learning → improved decision**, learning continuously from real-world outcomes under delayed, noisy, and partially observed feedback.*

![image.png](Ditto%20Matchmaking%20Engine%203%20x/image%206.png)

---

## **1. Core Principle**

> **Not all signals are equal, and not all feedback arrives on time.**
> 

Matchmaking operates under:

- **delayed rewards** (dates happen days later)
- **noisy signals** (likes ≠ real interest)
- **partial observability** (off-platform outcomes)
- **selection bias** (we only observe what we choose to show)

Therefore, the learning system must:

- distinguish **signal quality**
- attribute outcomes across time
- avoid contaminating models with **mistimed or biased feedback**
- support both **fast adaptation** and **stable long-term learning**

---

## **2. Two-Speed Learning Architecture**

The system should operate on two distinct but connected learning loops:

### **A. Fast Loop (Online / Near Real-Time Updates)**

**Purpose:**

Adapt quickly to user state and short-term behavior

**Updates:**

- temporal user state encoder
- readiness estimation
- short-term preference drift

**Signals used:**

- likes / skips
- match acceptance
- message response
- recent engagement patterns

**Characteristics:**

- high frequency
- lower signal quality
- strong recency weighting
- robust to noise via gating (anomaly detection, intent classification)

---

### **B. Slow Loop (Offline / Batch Learning)**

**Purpose:**

Improve model parameters and calibration using higher-quality outcomes

**Updates:**

- Likelihood / Intensity / Chemistry / Readiness models
- calibration curves
- policy evaluation metrics

**Signals used (hierarchical):**

| Tier | Signal Type | Examples |
| --- | --- | --- |
| **Tier 1 (highest quality)** | Downstream outcomes | Dates, post-date sentiment, continuation |
| **Tier 2** | Mid-funnel engagement | Contact exchange, deep conversation |
| **Tier 3** | Early interaction | Match, chat start |
| **Tier 4 (lowest quality)** | Proxy behavior | Likes, skips, dwell |

**Characteristics:**

- lower frequency
- higher signal quality
- aggregated and de-noised
- used for stable model updates

---

## **3. Delayed Reward Attribution**

A key challenge is linking **early exposure decisions** to **later outcomes**.

### **Problem**

A match shown at time *t* may produce:

- conversation at *t+1 day*
- contact exchange at *t+3 days*
- date at *t+7 days*

Naively training on early signals leads to **misaligned learning**.

---

### **Approach**

The system should:

- maintain **event timelines per pair**
- attribute downstream outcomes back to:
    - exposure decision
    - user state at time of exposure
    - head predictions at that moment
- use **time-windowed aggregation**, e.g.:
    - 7-day outcome windows
    - conditional conversion rates

---

### **Principle**

> **Train on outcome trajectories, not isolated events.**
> 

---

## **4. Counterfactual Logging & Bias Control**

The system only observes outcomes for **shown pairs**.

This introduces **selection bias**:

- we do not know what would have happened for unshown candidates

---

### **Solution: Counterfactual Logging**

For each decision:

- log:
    - candidate pool
    - head scores (all candidates, not just selected ones)
    - final decision
    - policy modifiers (plugins, thresholds)

This enables:

- **off-policy evaluation**
- comparison of:
    - selected vs non-selected candidates
- estimation of:
    - missed opportunities
    - over/under-exploration

---

### **Exploration Support**

Introduce controlled exploration:

- small probability of:
    - non-top-ranked but feasible candidates
- governed by:
    - feasibility constraints
    - bounded chemistry
    - uncertainty signals

This ensures:

- broader coverage of state space
- less biased learning
- discovery of new patterns

---

## **5. Learning Targets by Head (High-Level)**

Each head should learn from **different slices of the funnel**:

| Head | Primary Learning Signal | Supporting Signals |
| --- | --- | --- |
| **Likelihood** | Mutual match / viable interaction | chat start, rejection patterns |
| **Intensity** | Engagement depth, contact exchange, dates | message depth, reply latency |
| **Chemistry** | Positive deviation vs baseline expectation | uplift over predicted intensity |
| **Readiness** | Timing-dependent conversion efficiency | response conditional on state |

---

### **Key Principle**

> **Do not collapse all outcomes into a single label. Each head learns from its own signal domain.**
> 

---

## **6. Calibration & Feedback Stability**

Raw model outputs must be continuously aligned with reality.

### **Calibration Loop**

- compare predicted vs actual outcomes
- adjust:
    - likelihood probabilities
    - intensity expectations
    - readiness scaling

---

### **Why This Matters**

- stabilizes thresholds
- prevents drift from delayed feedback
- enables consistent product behavior

---

## **7. Simulation & Offline Evaluation Layer**

Because real-world feedback is:

- slow
- expensive
- noisy

the system should support **simulation environments**.

---

### **Simulation Uses**

- policy testing (before production)
- exploration strategy evaluation
- counterfactual scenario analysis
- stress-testing edge cases

---

### **Types of Simulation**

- **Replay-based simulation**
    - use historical logs
- **Synthetic user models**
    - approximate behavioral responses
- **counterfactual evaluation**
    - estimate “what if” outcomes

---

### **Principle**

> **Never rely only on online A/B testing for system evolution.**
> 

---

## **8. Feedback Quality Control**

Not all signals should enter learning equally.

### **Required Controls**

- anomaly filtering (extreme sessions)
- intent classification (exploration vs noise)
- trust-weighting of signals
- decay of low-confidence feedback

---

### **Example**

- one abnormal interaction session
    
    → should not reshape user preference
    
    → nor pollute model training
    

---

## **9. Learning Flywheel**

Putting it together:

```
1. Candidate selection (policy + heads)
2. User interaction unfolds
3. Multi-stage signals collected
4. Signals filtered & weighted
5. Fast loop updates user state
6. Slow loop updates models
7. Calibration aligns predictions
8. Policy improves decision quality
→ Repeat
```

---

## **10. Strategic Outcome**

With this system in place, Ditto evolves into:

> **a continuously learning matchmaking intelligence system**
> 

That:

- improves from real-world outcomes
- adapts to temporal user state
- avoids learning from noise and mistiming
- balances exploration and stability
- compounds system intelligence over time

## **Data Infrastructure Requirements**

A matchmaking engine of this kind cannot run on loosely connected product logs or ad hoc analytics tables.

It requires a **serious, unified data backbone** that treats matchmaking as a **traceable decision system**, not just a surface-level recommendation feature.

At minimum, Ditto’s data infrastructure must support reliable linkage across:

- **user-level entities**
- **pair-level entities**
- **candidate retrieval events**
- **head-level model outputs**
- **calibration outputs**
- **policy / plugin decisions**
- **final exposure decisions**
- **interaction funnel events**
- **feedback events**
- **downstream outcomes**
- **temporal user-state snapshots**
- **training datasets and label views**
- **experiment / policy versions**
- **simulation and replay logs**

This linkage is not a convenience. It is a foundational requirement.

Without it, the system cannot be **debugged, evaluated, governed, or improved in a disciplined way**.

---

### **1. Core Principle: Every Match Decision Must Be Reconstructible**

For every exposure or surfaced candidate, the system should be able to reconstruct:

- what candidate set was retrieved
- what representations or features were used
- what each head predicted
- how outputs were calibrated
- what thresholds, gates, or plugins were applied
- which candidates were filtered out
- which candidate was ultimately surfaced
- what the user did afterward
- what downstream outcome eventually followed

In other words:

> **Every match decision should leave a full decision trace.**
> 

This is essential for:

- debugging model behavior
- understanding why good candidates were missed
- diagnosing bad exposures
- evaluating policy changes
- attributing outcomes back to earlier decisions
- training future versions of the system

If the system cannot answer *why a match happened*, it cannot become a trustworthy decision engine.

---

### **2. Required Data Layers**

The infrastructure should support several tightly connected data layers.

### **A. Entity Layer**

Stable, versioned representations of the core objects in the system:

- **users**
- **profiles**
- **pair entities**
- **conversations**
- **matches**
- **dates / post-date feedback records**
- **safety / trust records**
- **preference states**
- **cohort / marketplace segments**

This layer should ensure that all downstream events and model decisions can be tied back to durable entity identifiers.

---

### **B. Event Layer**

An append-only event stream for all major product and model actions, including:

- profile view
- like / skip / save
- retrieval event
- candidate ranking event
- match exposure
- mutual match
- message start
- message depth milestones
- contact exchange
- date scheduling / confirmation
- post-date feedback
- preference edit
- trust / safety intervention
- session-level interaction statistics

This layer should preserve:

- **event time**
- **actor(s)**
- **pair context**
- **session context**
- **model / policy version**
- **experiment assignment**

The event layer is the raw behavioral substrate of the learning system.

---

### **C. Decision Trace Layer**

This is one of the most important layers.

For each matchmaking decision, the system should store:

- retrieval candidate pool
- retrieval scores / relevance features
- state snapshot at decision time
- Likelihood / Intensity / Chemistry / Readiness outputs
- calibrated versions of those outputs
- plugin or policy modifiers
- thresholding decisions
- rerank ordering
- final selected candidates
- filtered candidates and reasons for filtering
- uncertainty estimates if available

This layer makes the engine interpretable and debuggable.

Without it, model outputs become disconnected from actual product decisions.

---

### **D. State Snapshot Layer**

Because the system is temporal, it must store **time-indexed snapshots** of important dynamic state, such as:

- user preference state
- recent outcome summaries
- fatigue / saturation indicators
- exploration state
- readiness-related features
- recent archetype exposure history
- pair-level temporal context where relevant

These snapshots should be reconstructible **as of decision time**, not only in their latest form.

That is critical because model training and evaluation depend on knowing:

> **what the system believed at the moment the decision was made**
> 

—not what later data revisions imply after the fact.

---

### **E. Outcome & Label Layer**

The system needs derived, reliable views for:

- early funnel outcomes
- mid-funnel outcomes
- downstream outcomes
- delayed outcomes
- pair outcome trajectories
- user-level trajectory summaries
- head-specific training labels
- calibration targets
- policy evaluation labels

This layer should encode **signal quality hierarchy**, distinguishing:

- weak proxies
- medium-quality engagement signals
- high-quality downstream outcomes

Not all feedback should be treated equally.

---

### **F. Experimentation & Policy Layer**

Every decision should be attributable to the exact system configuration that produced it, including:

- model version
- feature version
- calibration version
- plugin set
- threshold policy
- exploration budget
- experiment arm
- simulation or replay condition if applicable

Without this, it becomes impossible to answer whether a performance change came from:

- a better model
- a policy adjustment
- a threshold change
- a segment shift
- or data drift

---

### **3. Key Questions the Infrastructure Must Be Able to Answer**

A serious data backbone should allow Ditto to answer questions like:

### **Decision Reconstruction**

- Which candidates were retrieved for a user at a given time?
- Which features and state snapshot were used?
- What did each head predict?
- What was the calibrated score?
- Which thresholds or plugins were applied?
- Why was a candidate surfaced, down-ranked, delayed, or filtered?

### **Outcome Attribution**

- What happened after a candidate was surfaced?
- Did the pair match?
- Did conversation start?
- Was there meaningful engagement?
- Did the pair exchange contact info?
- Did a date occur?
- What was the eventual downstream quality?

### **Learning Diagnostics**

- Which predictions were systematically overconfident?
- Where is the Likelihood Head wrong?
- Which users are being overexposed to repetitive archetypes?
- Is Readiness improving timing efficiency?
- Is Chemistry finding real uplift or just noise?
- Are mistimed rejections contaminating feasibility learning?

### **Marketplace / Policy Diagnostics**

- Are some cohorts underexposed?
- Are diversity plugins helping or hurting?
- Is exploration budget being spent effectively?
- Are certain policy operators suppressing high-upside candidates?
- Is the system collapsing toward local optima?

These are not “nice to have” analytics. They are core requirements for operating the engine responsibly.

---

### **4. Traceability, Versioning, and Time Correctness**

Because the system is sequential and constantly evolving, the data infrastructure must support:

- **time-correct joins**
- **feature versioning**
- **model versioning**
- **state versioning**
- **policy versioning**
- **label versioning**

This is critical.

A model should never be trained on features that were not actually available at decision time.

A policy should never be evaluated without knowing which thresholds and plugins were active.

A state snapshot should never silently reflect information that arrived after exposure.

In other words:

> **The infrastructure must prevent hindsight leakage.**
> 

If time correctness is weak, offline evaluation becomes misleading and model improvement becomes fragile.

---

### **5. Online + Offline Infrastructure Must Be Aligned**

The system will need both:

### **Online infrastructure**

for:

- serving retrieval
- scoring candidates
- reading current state
- applying policy
- logging decisions in real time

### **Offline infrastructure**

for:

- training datasets
- replay analysis
- calibration
- outcome attribution
- simulation
- experimentation analysis
- long-term diagnostics

These two worlds must remain aligned.

A common failure mode in intelligent systems is that:

- online serving uses one representation of the world
- offline analysis uses another
- model training uses a third

That creates silent divergence and weakens the entire engine.

The goal should be:

> **One coherent data contract across serving, learning, analytics, and policy evaluation.**
> 

---

### **6. Requirements for Training and Replay**

To support model training and self-improvement, the infrastructure should make it easy to build:

- user-level training examples
- pair-level training examples
- head-specific label sets
- temporal sequence datasets
- exposure-to-outcome trajectories
- hard-negative and counterfactual candidate sets
- replay logs for offline policy evaluation
- calibration datasets by cohort and time window

This is especially important because each head depends on different supervision.

For example:

- **Likelihood** needs viable-pair and mutuality signals
- **Intensity** needs engagement-depth and progression signals
- **Chemistry** may depend on residual uplift relative to baseline expectation
- **Readiness** needs timing-conditioned conversion views

The data infrastructure must make these training views straightforward to construct and refresh.

Otherwise the architecture will exist on paper but remain painful to improve in practice.

---

### **7. Support for Counterfactual Evaluation and Controlled Exploration**

Because the system only observes outcomes for surfaced candidates, the infrastructure should support:

- logging of full candidate pools, not only winners
- exposure probabilities where applicable
- ranking positions
- reasons for exclusion
- plugin interventions
- uncertainty signals
- exploration flags

This is essential for:

- counterfactual analysis
- off-policy evaluation
- exploration diagnostics
- missed-opportunity analysis
- bias reduction in learning

A matchmaking engine that cannot compare **selected vs non-selected feasible candidates** will struggle to learn efficiently.

---

### **8. Data Quality, Reliability, and Governance**

A system this central to product quality also requires strong governance around data itself.

This includes:

- schema discipline
- event consistency checks
- identity resolution across entities
- deduplication
- late-event handling
- missingness monitoring
- anomaly detection in logging
- label-quality auditing
- privacy and access controls
- trust / safety data handling boundaries

The system is only as trustworthy as the data it learns from.

If logs are incomplete, entities drift, outcomes are inconsistently linked, or timestamps are unreliable, then even a strong modeling architecture will degrade.

---

### **9. Suggested Minimum Infrastructure Capabilities**

At minimum, Ditto should invest in infrastructure that supports:

- **entity graph / relational linkage** across users, pairs, exposures, and outcomes
- **append-only event logging**
- **decision trace storage**
- **time-versioned feature and state snapshots**
- **model + policy version tracking**
- **derived label tables by funnel stage**
- **replay / backtesting datasets**
- **counterfactual logging support**
- **calibration and diagnostics dashboards**
- **cohort and marketplace health monitoring**

This should not be treated as analytics afterthought. It is part of the engine itself.

---

### **10. Strategic Conclusion**

This is why the matchmaking engine and the unified data infrastructure must be developed together.

The intelligence layer is only as strong as the infrastructure that connects:

> **decision → exposure → interaction → outcome → learning**
> 

If that chain is weak, the system may still produce scores, but it will not become a true decision engine.

If that chain is strong, Ditto gains something much more valuable:

- interpretable model behavior
- reliable policy control
- faster experimentation
- cleaner learning loops
- stronger debugging ability
- long-term compounding intelligence

In that sense, the data infrastructure is not a support function.

It is part of the core architecture of Matchmaking 3.x itself.

# **Evaluation System**

Because matchmaking operates in a **sparse, delayed, noisy, and partially observed environment**, evaluation cannot rely on a single metric or a single stage.

Ditto requires a **layered evaluation framework** that connects:

> **model quality → decision quality → user experience → real-world outcomes**
> 

The goal is not to optimize proxies in isolation, but to ensure the system is **improving real matchmaking outcomes over time**.

---

## **1. Core Principles**

### **Multi-Level Evaluation**

Evaluation must operate across four layers:

- **Head-level** → model correctness
- **Decision-level** → ranking and policy behavior
- **Product-level** → user funnel outcomes
- **System-level** → marketplace health

---

### **Outcome Hierarchy Awareness**

Signals must be treated according to quality:

- upstream proxies (likes, skips)
- mid-funnel engagement (matches, chats)
- downstream outcomes (dates, continuation)

> Improvement in weak proxies is insufficient if downstream outcomes do not improve.
> 

---

### **Timing-Aware Evaluation**

Evaluation must incorporate **when** decisions are made, not just what is selected:

- readiness-conditioned performance
- timing-sensitive conversion
- reduction of mistimed exposure

---

### **Avoid Proxy Over-Optimization**

The system must guard against improving:

- CTR
- match rate
- message count

while degrading:

- real connection quality
- user satisfaction
- long-term retention

---

## **2. Head-Level Evaluation (Model Quality)**

Each head should be evaluated independently, aligned with its role.

---

### **Likelihood (Feasibility & Mutuality)**

- mutual match prediction accuracy
- precision / recall within feasible region
- directional accuracy (A→B, B→A)
- calibration error
- false positive rate (wasted exposure)
- false negative rate (missed viable pairs)

---

### **Intensity (Strength of Interaction)**

- ranking quality among feasible pairs
- correlation with engagement depth
- prediction of:
    - conversation start
    - message depth
    - contact exchange
- lift over baseline ranking
- calibration of engagement likelihood

---

### **Chemistry (Bounded Uplift)**

- incremental lift vs baseline (Likelihood + Intensity)
- outcome improvement for chemistry-promoted pairs
- degradation risk (noise introduction)
- stability across cohorts

> Key test: **Does chemistry produce real outcome lift, not just novelty?**
> 

---

### **Readiness (Timing Quality)**

- conversion lift conditional on timing
- performance difference between high vs low readiness
- reduction in mistimed rejections
- improvement in response and follow-through

---

### **Cross-Head Diagnostics**

- interaction effects between heads
- over-reliance on a single head
- conflicting signals (e.g. high intensity, low feasibility)
- calibration consistency

---

## **3. Decision-Level Evaluation (Policy Behavior)**

This evaluates how model outputs translate into actual decisions.

---

### **Core Metrics**

- top-K selection quality
- win-rate vs baseline policy
- exposure efficiency (outcomes per exposure)
- wasted exposure rate
- missed opportunity rate

---

### **Policy Diagnostics**

- effectiveness of feasibility thresholds
- readiness gating impact
- chemistry contribution within safe bounds
- plugin effects (diversity, freshness, safety)
- exploration vs exploitation balance

---

### **Counterfactual Diagnostics**

Using logged candidate pools:

- compare selected vs non-selected feasible candidates
- estimate opportunity loss (regret)
- evaluate ranking errors

---

## **4. Product-Level Evaluation (User Outcomes)**

Measures real user-facing impact across the funnel.

---

### **Core Funnel Metrics**

- match acceptance quality
- conversation start rate
- conversation depth
- response latency
- contact exchange rate
- first-date conversion
- post-date satisfaction
- second-date continuation (or equivalent)

---

### **Trajectory Metrics**

- progression across funnel stages
- drop-off points
- time-to-conversion
- session-to-session consistency

---

### **User Experience Signals**

- perceived match quality
- novelty vs repetition
- fatigue indicators
- re-engagement patterns

---

### **Retention Metrics**

- short-term retention
- medium-term engagement stability
- outcome-driven retention

---

## **5. System-Level Evaluation (Marketplace Health)**

Ensures the system remains healthy at scale.

---

### **Marketplace Metrics**

- diversity without quality loss
- repetition rate of archetypes
- exposure distribution across users
- long-tail visibility
- fairness across cohorts

---

### **Segment Performance**

- new vs experienced users
- high vs low activity users
- cold-start users
- cohort-level outcome differences

---

### **System Stability**

- week-over-week lift
- metric volatility
- robustness to distribution shifts

---

### **Failure Mode Monitoring**

- local optima (repetitive matches)
- exposure concentration
- degradation in long-tail outcomes
- exploration collapse or over-exploration

---

## **6. Offline + Online Evaluation Stack**

### **Offline Evaluation**

Supports:

- model iteration
- calibration
- ranking diagnostics
- cohort analysis

But cannot fully capture:

- delayed outcomes
- policy interaction effects
- user adaptation

---

### **Online Evaluation**

Required for validation:

- A/B testing
- cohort-based experiments
- guardrail metrics (quality, fairness, safety)
- long-term holdouts

---

## **7. Evaluation Flywheel**

```
1. Model produces predictions
2. Policy converts predictions into decisions
3. Users interact with surfaced candidates
4. Outcomes are collected across stages
5. Outcomes are attributed back to decisions
6. Metrics evaluate performance across layers
7. Insights drive model and policy updates
→ Repeat
```

---

## **Strategic Outcome**

A strong evaluation framework ensures:

- alignment with real outcomes
- controlled iteration
- system transparency
- long-term product quality

# **Lightweight Simulation & Policy Testing**

Evaluation tells us what happened.

Simulation allows us to safely explore what **could happen**.

Because real-world feedback is **slow, sparse, and expensive**, Ditto requires a **simulation and policy testing layer** to accelerate iteration and reduce risk.

---

## **1. Core Role of Simulation**

Simulation acts as a **decision sandbox** that enables:

- policy experimentation before deployment
- counterfactual reasoning
- exploration strategy testing
- stress-testing of edge cases

---

## **2. Replay Framework (Foundation Layer)**

Replay is the most immediate and practical simulation tool.

---

### **What Replay Does**

- re-run decision logic on historical logs
- simulate alternative ranking or policy decisions
- compare:
    - actual decisions vs counterfactual decisions

---

### **Requirements**

- full candidate pools logged
- head outputs recorded
- policy decisions traceable
- outcome trajectories available

---

### **Key Use Cases**

- evaluating new ranking strategies
- testing threshold changes
- estimating missed opportunities
- validating exploration policies

---

## **3. Shadow Mode (Pre-Deployment Validation)**

New models or policies should first run in **shadow mode**.

---

### **How It Works**

- generate decisions without affecting users
- compare shadow outputs to production outputs
- evaluate expected impact using historical data

---

### **Benefits**

- reduces deployment risk
- identifies regressions early
- validates calibration and ranking logic

---

## **4. Counterfactual Evaluation**

Because only shown candidates produce outcomes, simulation must estimate:

> **What would have happened if a different decision was made?**
> 

---

### **Capabilities Needed**

- candidate pool logging
- ranking positions
- exposure decisions
- outcome comparison

---

### **Outputs**

- regret estimation
- opportunity loss
- policy comparison metrics

---

## **5. Exploration Strategy Testing**

Simulation enables safe testing of exploration policies.

---

### **Examples**

- chemistry-based exploration
- uncertainty-driven exploration
- diversity injection
- readiness-aware delays

---

### **Evaluation Goals**

- improve discovery without degrading quality
- balance exploration vs stability
- avoid local optima

---

## **6. Synthetic / Behavioral Simulation (Longer-Term)**

Over time, Ditto should develop approximate **user behavior models**.

---

### **Use Cases**

- simulate user responses
- test policy changes at scale
- accelerate iteration cycles
- evaluate rare scenarios

---

### **Types**

- replay-based simulation (data-grounded)
- behavioral models (approximate user reactions)
- hybrid simulation (real + synthetic signals)

---

### **Principle**

> Simulation complements real-world evaluation — it does not replace it.
> 

---

## **7. Stress Testing & Edge Case Analysis**

Simulation should be used to test:

- cold-start scenarios
- low-signal users
- extreme preference profiles
- adversarial or noisy behavior
- marketplace imbalance scenarios

---

## **8. Integration with Decision System**

Simulation should plug directly into:

- model iteration
- policy design
- experimentation pipeline

---

## **Simulation Loop**

```
1. Load historical logs
2. Apply alternative model / policy
3. Generate counterfactual decisions
4. Estimate outcomes (observed + inferred)
5. Compare against baseline
6. Select promising strategies
→ Move to shadow mode or A/B test
```

---

## **Strategic Outcome**

A mature simulation system enables Ditto to:

- iterate faster without waiting for slow outcomes
- reduce risk of regressions
- discover better policies
- improve exploration efficiency
- build toward more advanced decision optimization

---

## **Rollout Plan**

Seven phases. Each phase ships a working capability. Each gate is a pass/fail checkpoint — the next phase does not start until the gate clears. Data infrastructure runs in parallel from day one.

---

### **Phase 1 — Build the Data Backbone**

*Runs in parallel with all subsequent phases.*

| # | Action | Done when |
| --- | --- | --- |
| 1.1 | Define and deploy **entity schema**: users, pairs, matches, conversations, dates, feedback records, preference states, safety/trust records. Assign stable IDs. | All core entities queryable; pair-level joins work end-to-end |
| 1.2 | Stand up **append-only event stream**: swipes, likes, skips, matches, messages, contact exchange, dates, post-date feedback, preference edits, session stats. | Event completeness > 99%; ingestion latency < 5 min |
| 1.3 | Build **decision trace logger**: for every match decision, store candidate pool, head scores (all candidates, not just winners), calibration outputs, plugin modifiers, final selection, filtering reasons. | Pick any historical decision → full trace reconstructible within 24h of launch |
| 1.4 | Implement **time-indexed state snapshots**: capture user state vector at the moment each decision is made. | State-at-decision-time recoverable; verify zero hindsight leakage in offline joins |
| 1.5 | Create **outcome & label tables**: derived views per funnel stage (Tier 1 downstream → Tier 4 proxy). Tag signal quality tier on every label row. | Per-head training views constructible; outcome attribution pipeline producing labels |
| 1.6 | Log **experiment metadata** per decision: model version, feature version, calibration version, plugin set, experiment arm. | Any metric movement attributable to a specific system configuration |

**Gate:** Infrastructure operational. Any match decision is reconstructible. Per-head label views exist. No phase beyond Phase 2 launches without this.

---

### **Phase 2 — Ship Retrieval**

| # | Action | Done when |
| --- | --- | --- |
| 2.1 | Finalize **profile + interaction export schema**. Map all seven signal families (structured attributes, free-text, explicit prefs, implicit prefs, images, interaction history, pair cross-features) to the feature pipeline. | All signal families flowing into featurization; schema documented |
| 2.2 | Build **hybrid featurization pipeline**: structured/sparse tower + dense tower (domain-matched text encoder, image encoder). Learned projection to align dimensions. | Feature pipeline producing aligned vectors for all active users |
| 2.3 | Fine-tune **shared embedding backbone** on matchmaking-relevant contrastive + preference-satisfaction objectives. | Retrieval recall@100 exceeds target on held-out viable pairs |
| 2.4 | Deploy **ANN index** (HNSW or equivalent) over embedded user space. Enforce hard constraints at retrieval time. | Retrieval latency ≤ budget; zero constraint violations; candidate pools logged per 1.3 |
| 2.5 | Build **positive / hard-negative construction pipeline** for contrastive training. | Pipeline producing balanced batches; hard negatives verified as non-trivial |
| 2.6 | Compute initial **pair cross-features** (preference satisfaction, trait complementarity, directional asymmetry) and make available at scoring time. | Pair tensors computable for any candidate pair within latency budget |
| 2.7 | Run **shadow test**: new retrieval vs current candidate generation. Measure recall, latency, constraint compliance, semantic coverage. | Shadow test report shows recall improvement with no quality regression |

**Gate:** Retrieval live in shadow mode. Recall improved. No constraint violations. Decision traces logging full candidate pools.

---

### **Phase 3 — Make the System Stateful + Deploy Likelihood Head**

| # | Action | Done when |
| --- | --- | --- |
| 3.1 | Define **state schema**: taste profile, receptivity level, exploration appetite, outcome trajectory, archetype saturation. | Schema reviewed and approved; dimensions documented |
| 3.2 | Implement **three-timescale state update**: real-time (within-session swipes/skips), session-boundary (match outcomes, chat depth, preference edits), slow drift (dates, post-date feedback, week-over-week shifts). | State vector updating at all three timescales; verified with test users |
| 3.3 | Deploy **temporal attenuation**: 7-day recency window carries 50–80% weight; per-signal-type decay (swipes fast, conversations moderate, dates slow, explicit edits persist). Smooth decay, not step function. | Attenuation verified; old signals demonstrably fading; explicit edits persisting |
| 3.4 | Deploy **state robustness guardrails**: anomaly detection & gating (extreme velocity, all-left/right streaks), intent classification (normal/exploration/noise/manipulation), reversion-to-prior, feedback loop dampening. | Inject synthetic anomalous sessions → verify they are down-weighted |
| 3.5 | Implement **cold-start bootstrap**: initial state from profile attributes + declared preferences + cohort priors. Mark as low-confidence. Wire to wider exploration budgets downstream. | New users get reasonable initial state; exploration budgets verified wider for cold-start |
| 3.6 | Train **Likelihood Head**: directional feasibility model predicting P(A→B), P(B→A), and mutual feasibility. | Feasibility precision/recall > similarity baseline on held-out pairs |
| 3.7 | Construct **Likelihood labels**: mutual match = positive; viable interaction (chat start within 48h) = soft positive; unmatched exposure = negative. 7-day attribution window. | Labels constructible from outcome layer; label quality audited |
| 3.8 | Deploy **calibration layer** for Likelihood outputs. Log thresholds. Monitor calibration error per segment. | Calibration error < target; thresholds stable across user segments |
| 3.9 | Run **A/B test**: stateful Likelihood-gated system vs similarity baseline. Primary metric: match-to-conversation conversion. Guardrail: no downstream quality regression. | Statistically significant lift on primary metric; guardrails hold |

**Gate:** Likelihood system outperforms similarity baseline in A/B test. State encoder updating correctly. Robustness guardrails verified. Calibration stable.

---

### **Phase 4 — Add Intensity Reranking**

| # | Action | Done when |
| --- | --- | --- |
| 4.1 | Construct **Intensity labels**: conversation depth ≥ threshold = strong; contact exchange = stronger; date occurred = strongest. Tier 1–Tier 2 signals. 7-day attribution window. | Labels constructible; quality audited |
| 4.2 | Train **Intensity Head** on engagement-depth and progression signals, conditioned on feasibility. | Intensity ranking quality among feasible pairs > Likelihood-only ordering on held-out data |
| 4.3 | Add **user-state-conditioned intensity**: recent outcome trajectory, archetype saturation, and exploration appetite inform strength prediction. | Personalized intensity outperforms non-personalized on held-out data |
| 4.4 | Deploy **two-head reranking**: Likelihood gates → Intensity ranks. Calibrate Intensity outputs. Log rerank ordering. | Reranked order producing stronger matches in shadow comparison |
| 4.5 | Run **A/B test**: two-head system vs Likelihood-only. Primary metrics: conversation depth, contact exchange rate. Secondary: downstream conversion. | Statistically significant lift on ≥ 2 mid-funnel metrics |

**Gate:** Intensity reranking shows measurable lift in A/B test. Two heads operational and calibrated.

---

### **Phase 5 — Add Chemistry + Readiness + Deploy Plugin Framework**

| # | Action | Done when |
| --- | --- | --- |
| 5.1 | Build **Chemistry features**: complementarity tensors, pair cross-features beyond baseline, novelty-within-safety indicators, history-aware broadening signals. | Feature pipeline producing Chemistry-specific pair features |
| 5.2 | Construct **Chemistry labels**: positive deviation vs Likelihood + Intensity baseline expectation. Pair outcomes that exceeded predicted intensity = positive uplift. | Labels constructible; residual uplift measurable |
| 5.3 | Train **Chemistry Head** as residual uplift model. Constrain within feasibility-safe region. Set bounded exploration budget. | Chemistry-promoted pairs show positive outcome lift on held-out data without degrading overall quality |
| 5.4 | Construct **Readiness labels**: timing-dependent conversion efficiency. Same-pair outcomes conditioned on user state at exposure time. Temporal lift metric. | Labels constructible from state snapshots + outcome layer |
| 5.5 | Train **Readiness Head** using temporal state + pair-specific timing features. | Readiness improves conversion for high-readiness exposures vs low-readiness on held-out data |
| 5.6 | Build **plugin runtime**: define plugin API contract, versioning scheme, per-plugin KPI dashboard, rollback mechanism, kill-switch. | Runtime deployed; plugins independently deployable and rollbackable |
| 5.7 | Deploy **3 initial plugins**: (a) feasibility threshold gate, (b) diversity injector, (c) exploration budget controller. | All three plugins active; per-plugin KPIs tracked; product team can adjust parameters without eng deployment |
| 5.8 | Deploy **trust-aware plugin**: apply safety modifiers; define Chemistry exploration-harm guardrails (e.g. Chemistry cannot promote pairs flagged by trust system). | No increase in safety incidents; guardrails verified |
| 5.9 | Establish **fairness monitoring**: track exposure distribution, outcome quality, and conversion rates across cohorts (gender, age, activity level, new vs experienced). | Fairness dashboard live; no cohort showing systematic degradation |
| 5.10 | Run **A/B test**: four-head + plugins vs two-head system. Primary: downstream conversion. Guardrails: safety, fairness, quality floor. | Four-head system outperforms two-head. Chemistry shows real outcome lift. Readiness reduces mistimed rejections. Plugin framework operational. |

**Gate:** All four heads live and contributing. Chemistry producing real uplift, not just novelty. Readiness reducing mistimed exposure. Plugin framework operational with rollback. Fairness metrics stable.

---

### **Phase 6 — Close the Learning Loop**

| # | Action | Done when |
| --- | --- | --- |
| 6.1 | Verify **fast loop** in production: state updates reflecting within-session behavior at all three timescales. Anomaly gating operational at scale. | State updates verified end-to-end; no anomalous session corruption observed over 2-week window |
| 6.2 | Build **batch retraining pipeline**: per-head label refresh, hierarchical signal weighting (Tier 1–Tier 4), time-windowed outcome attribution (7-day windows). | Pipeline produces fresh training sets; retrained models evaluatable on held-out data |
| 6.3 | Deploy **counterfactual logging**: log full candidate pools with scores for ALL candidates (not just selected), policy modifier flags, exposure probabilities, ranking positions, filtering reasons. | Off-policy evaluation runnable; missed-opportunity analysis producing results |
| 6.4 | Implement **delayed reward attribution**: build event timelines per pair; link exposure decisions to 7-day outcome windows; attribute downstream outcomes back to head predictions at decision time. | Training labels incorporate Tier 1–Tier 2 downstream outcomes, not just Tier 3–Tier 4 proxies |
| 6.5 | Deploy **continuous calibration monitoring**: automated drift detection, per-segment calibration refresh, alerting when calibration error exceeds threshold. | Calibration drift caught within 48h; auto-refresh operational |
| 6.6 | Execute **first full learning cycle**: exposure → outcome → label → retrain → evaluate → deploy improved model. | Retrained model outperforms previous version on held-out eval; improvement deployed to production |
| 6.7 | Run **counterfactual analysis**: compare selected vs non-selected feasible candidates. Estimate regret. Identify systematic missed opportunities. | Analysis produces ≥ 3 actionable insights (e.g. "Chemistry head under-exploring in segment X") |

**Gate:** At least one full learning cycle completed end-to-end. Retrained model shows improvement. Counterfactual logging operational. Calibration stable. Evidence that the flywheel is turning.

---

### **Phase 7 — Simulation, Continuous Improvement, Scale**

| # | Action | Done when |
| --- | --- | --- |
| 7.1 | Build **replay framework**: re-run decision logic on historical logs; compare actual vs counterfactual decisions; estimate regret and opportunity loss. | Replay produces actionable insights; ≥ 1 policy improvement validated through replay before production |
| 7.2 | Deploy **shadow mode infrastructure**: new models/policies generate decisions without affecting users; automated shadow-vs-production comparison. | Shadow mode operational; ≥ 1 regression caught in shadow before reaching production |
| 7.3 | Build **exploration diagnostics**: measure exploration budget efficiency, over/under-exploration rates, missed-opportunity rate per cohort. | Exploration budget demonstrably producing discovery without quality degradation |
| 7.4 | Run **stress tests**: cold-start scenarios, low-signal users, extreme preference profiles, adversarial behavior injection, marketplace imbalance simulation. | System degrades gracefully under all stress scenarios; no catastrophic failures |
| 7.5 | Build **synthetic simulation** (longer-term): approximate user behavior models for policy testing at scale; hybrid simulation (real + synthetic signals). | Simulation predictions directionally consistent with real A/B outcomes |
| 7.6 | Establish **continuous improvement cadence**: weekly retrain cycle, calibration refresh, plugin performance review, monthly policy audit, quarterly marketplace health assessment. | Cadence running; week-over-week lift measurable; system intelligence compounding |
| 7.7 | Publish **internal system scorecard**: per-head quality, decision-level efficiency, product-level funnel, marketplace health, fairness, exploration efficiency — updated weekly. | Scorecard live and reviewed weekly by AI team + product leadership |

**Gate (ongoing):** The system is demonstrably improving over time. Each retrain cycle produces measurable lift. Simulation insights translate into production gains. The flywheel is turning.

---

### **Risks**

---

### **R1. Data Infrastructure Delays**

**What breaks:** Every phase after Phase 1 depends on traceable decisions, reconstructible state, and per-head label views. If the data backbone ships late or incomplete, models cannot be debugged, evaluation is guesswork, and the learning loop never closes. The entire flywheel stalls.

**How it manifests:**

- Models ship but no one can explain why a match was surfaced or filtered
- Offline evaluation produces misleading results because state snapshots have hindsight leakage
- Retraining pipeline cannot construct labels because outcome attribution is missing

**Mitigation:**

- Phase 1 runs in parallel from day one — not after modeling starts
- Define minimum viable traceability (decision trace + state snapshot + outcome linkage) as a hard prerequisite for the Phase 3 gate
- Assign dedicated infrastructure ownership separate from modeling

---

### **R2. Sparse Downstream Labels**

**What breaks:** The heads that matter most (Intensity, Chemistry, Readiness) depend on Tier 1–Tier 2 signals: dates, post-date feedback, contact exchange, conversation depth. These are rare. If training defaults to Tier 3–Tier 4 proxies (likes, skips), the heads learn shallow correlations and the system optimizes for engagement rather than real outcomes. The flywheel compounds the wrong thing.

**How it manifests:**

- Intensity Head learns to predict likes, not real interaction strength
- Chemistry Head finds no residual uplift because the baseline is proxy-fitted
- Post-date feedback is too sparse to train on directly

**Mitigation:**

- Enforce hierarchical signal weighting: Tier 1 outcomes carry disproportionate training weight even when rare
- Deploy delayed reward attribution (7-day outcome windows) so downstream signals reach training labels
- Build synthetic label pipeline for cold-start dimensions (e.g. conversational AI collecting early taste indicators)
- Track label quality metrics: what fraction of training signal comes from Tier 1–Tier 2 vs Tier 3–Tier 4

---

### **R3. Selection Bias in Learning**

**What breaks:** The system only observes outcomes for pairs it chose to show. If it never shows a candidate, it never learns whether that candidate would have worked. Over time, models reinforce the current policy rather than discovering better matches. The system becomes confidently narrow.

**How it manifests:**

- High-upside candidates that the current model under-scores are never surfaced, so no evidence accumulates for them
- Retrained models look "better" on logged data but are just more confident inside the same biased distribution
- Counterfactual regret grows invisibly

**Mitigation:**

- Deploy counterfactual logging in Phase 6: log scores for ALL candidates in the pool, not just selected ones
- Maintain controlled exploration budgets: a small fraction of exposures go to non-top-ranked but feasible candidates, governed by feasibility constraints and bounded chemistry
- Use off-policy evaluation (inverse propensity weighting, doubly-robust estimation) to estimate performance of alternative policies from logged data
- Track exploration efficiency as a system-level metric: is the exploration budget discovering new patterns or wasting exposures?

---

### **R4. Readiness Head Absorbs Everything**

**What breaks:** The Readiness Head is designed to model temporal receptivity. But without constraints, it can absorb attractiveness, popularity, segment effects, and baseline desirability — becoming a second Likelihood model that is opaque and uninterpretable. The architectural separation between state, readiness, and feasibility collapses.

**How it manifests:**

- Readiness scores correlate strongly with user popularity rather than temporal receptivity
- Removing the Readiness Head barely changes system behavior (because Likelihood already captures the same signal)
- Timing-specific evaluation metrics (e.g. high-readiness vs low-readiness conversion lift) show no meaningful difference

**Mitigation:**

- Constrain Readiness features to **temporal and state-derived signals only** — no static profile features, no popularity proxies
- Regularize against correlation with Likelihood outputs during training
- Evaluate Readiness with timing-specific diagnostics: does it improve conversion **conditional on** pair quality being held constant?
- If Readiness collapses into Likelihood on evaluation, demote it to a feature within the state encoder rather than a separate head

---

### **R5. Plugin Proliferation**

**What breaks:** The plugin layer is designed for modular policy iteration. But if plugins accumulate without discipline, the layer becomes a tangle of interacting heuristics that compensate for model weaknesses rather than governing policy. System behavior becomes impossible to attribute: is a regression caused by the model, a plugin, or an interaction between plugins?

**How it manifests:**

- More than 10 active plugins with unclear interaction effects
- Removing a single plugin causes unexpected regressions in unrelated metrics
- Product team cannot predict the effect of changing a plugin parameter
- A/B tests show inconsistent results because plugin interactions create confounds

**Mitigation:**

- Enforce the boundary: **plugins shape policy around calibrated outputs; they do not replace representation, state, or prediction layers**
- Track per-plugin KPIs: every plugin must have a measurable objective and a kill-switch
- Limit active plugin count; require justification for each addition
- Run quarterly plugin audits: disable plugins one at a time and measure impact; remove any plugin that has no measurable positive effect
- Log plugin decisions in the decision trace layer so every intervention is attributable

---

### **R6. Chemistry Produces Noise, Not Uplift**

**What breaks:** The Chemistry Head is meant to surface bounded positive novelty. But if the signal is too weak, the training labels too noisy, or the exploration budget too loose, Chemistry injects randomness rather than real uplift. Users receive surprising but low-quality matches. Trust degrades. The product feels unpredictable.

**How it manifests:**

- Chemistry-promoted pairs underperform or perform no better than non-promoted pairs on downstream outcomes
- User feedback on Chemistry-surfaced matches trends negative
- Ablation test: removing Chemistry does not hurt (or improves) overall metrics

**Mitigation:**

- Chemistry operates only within feasibility-safe regions — it cannot override Likelihood
- Chemistry has an explicit exploration budget, not an unbounded score modifier
- Require real outcome lift (Tier 1–Tier 2 signals) in ablation before scaling Chemistry’s influence
- Run ongoing ablation: periodically measure system with vs without Chemistry; if lift is not sustained, reduce budget
- Track Chemistry-specific evaluation: incremental lift vs baseline, degradation risk, stability across cohorts

---

### **R7. Calibration Drift**

**What breaks:** Calibrated scores are the foundation for thresholds, plugin behavior, and product-level controls. If calibration drifts — because of data distribution shifts, model updates, or changing user behavior — a threshold that once meant "80% mutual feasibility" no longer means that. Product behavior becomes unpredictable. Plugin logic that depends on score semantics breaks silently.

**How it manifests:**

- Match quality changes without any model or policy update (threshold semantics shifted)
- Per-segment behavior diverges (calibration holds for one cohort but drifts for another)
- Product team adjusts thresholds to compensate, creating a cascade of manual fixes

**Mitigation:**

- Deploy continuous calibration monitoring with automated alerting when error exceeds threshold
- Per-segment calibration: monitor and refresh separately for key cohorts (new users, high-activity, low-activity, gender, geography)
- Require calibration stability as a gate condition in every phase
- Log calibration outputs in the decision trace so drift is diagnosable after the fact

---

### **R8. Cold-Start Quality Gap**

**What breaks:** New users have no interaction history. The state encoder has no behavioral signal. Every head operates on low-confidence inputs. If the system treats cold-start users the same as established users, match quality is poor and early churn spikes. First impressions are disproportionately important in dating — a bad first session may mean the user never returns.

**How it manifests:**

- New user match-to-conversation conversion significantly below established user baseline
- New users churn within first week at higher rates than comparable products
- State encoder for new users is dominated by noise from the first few sessions

**Mitigation:**

- Bootstrap initial state from profile attributes + declared preferences + cohort-level priors; explicitly mark as low-confidence
- Wire low-confidence state to wider exploration budgets and more conservative feasibility gating (avoid wasting exposures on high-risk matches early)
- Accelerate state maturation through conversational onboarding (collect taste indicators through dialogue before the first match)
- Track cold-start-specific metrics: time-to-first-meaningful-match, first-week retention, state confidence trajectory
- Define a "state maturity threshold" below which the system applies a cold-start policy regime

---

### **R9. Feedback Loop Amplification**

**What breaks:** The closed loop itself can amplify problems. A bad week shifts the user’s state toward fatigue; the system responds by serving safer, more conservative matches; the user gets bored; the state reinforces fatigue further. The system locks the user into a local minimum they never chose. The same dynamic can happen at the marketplace level: if the system converges on a narrow set of "safe" archetypes, diversity collapses and the product feels stale for everyone.

**How it manifests:**

- Declining engagement for users who entered a temporary low period, even after their real-world state recovers
- System-wide archetype concentration: top-10% of profiles receive disproportionate exposure; long-tail profiles are systematically under-surfaced
- Week-over-week diversity metrics decline despite no model or policy change

**Mitigation:**

- Deploy feedback loop dampening in the state encoder: momentum dampening that prevents self-reinforcing spirals
- Reversion-to-prior after anomalous periods: if subsequent behavior returns to baseline, accelerate decay of the negative period’s influence
- System-level diversity monitoring: track archetype concentration, exposure distribution, and long-tail visibility as ongoing metrics
- Diversity plugins with explicit budgets that counteract concentration tendencies
- If user-level engagement enters a sustained decline loop, trigger a "reset exploration" intervention that temporarily widens the candidate pattern

## **Milestone Timeline**

The evolution of Ditto’s matchmaking intelligence follows five milestones, mapped to a concrete 2026 calendar. Milestones 2 and 3 run in parallel. Data infrastructure runs continuously from day one.

---

![image.png](Ditto%20Matchmaking%20Engine%203%20x/image%207.png)

### **Milestone 1 — Seven-Feature Foundation ✔ COMPLETED**

**Period:** December 2025 – February 2026 (3 months)

**What was achieved:**

- Established the seven-feature pair evaluation model as the foundation for all matchmaking decisions
- Built similarity-based candidate retrieval
- Validated core signal families through manual feature engineering and expert-driven curation
- All matchmaking decisions required expert comparison and manual intervention

**Significance:** This milestone created the signal foundation that every subsequent milestone builds on. The seven features are not replaced — they become the core of an automated, scalable, machine-driven system.

---

### **Milestone 2 — Automated Matchmaking Process**

**Period:** April 2026 (2–5 weeks)

**Runs in parallel with Milestone 3.**

**What ships:**

| # | Deliverable | Result |
| --- | --- | --- |
| 2a | Deploy **OpenClaw agent operating system** as matchmaking infrastructure | Agent skills/plugins replace manual expert decisions |
| 2b | Build first set of **composable automation skills**: feasibility gating, diversity injection, exploration budget control | Matchmaking decisions execute automatically, not through human comparison |
| 2c | Wire the seven-feature foundation into **automated pipeline** | Same core signals, machine-driven execution |
| 2d | Ship automated matchmaking to **production** | Users receive matches generated by the agent OS, not by manual curation |

**What changes for the product:**

- Matchmaking transitions from expert-driven to **machine-driven**
- Decisions become faster, consistent, and scalable
- Agent skills can be added, tested, tuned, and retired without engineering redeployment
- **Immediate, measurable product improvement from day one**

**Gate:** Automated matches live in production. Match quality ≥ manual baseline. Agent skills operational and composable.

---

### **Milestone 3 — Intelligent Decision System**

**Period:** April – June 2026 (2–3 months)

**Runs in parallel with Milestone 2.**

**What ships:**

| # | Deliverable | Result |
| --- | --- | --- |
| 3a | Build **shared representation backbone**: hybrid fusion of all 7 signal families (structured + sparse + dense + pair cross-features) | One unified representational geometry for retrieval and scoring |
| 3b | Deploy **temporal user state encoder**: three-timescale updates, temporal attenuation (7-day recency weighting), robustness guardrails (anomaly detection, intent classification, reversion-to-prior, feedback loop dampening) | System becomes stateful — reasons about who the user wants *now*, not who they wanted last month |
| 3c | Train and deploy **Likelihood Head**: directional feasibility P(A→B), P(B→A), mutual feasibility | Eliminates non-viable pairs; reduces wasted exposure |
| 3d | Train and deploy **Intensity Head**: engagement depth, progression prediction, user-state-conditioned ranking | Ranks feasible candidates by expected interaction strength |
| 3e | Train and deploy **Chemistry Head**: residual uplift over baseline, bounded novelty, complementarity | Surfaces high-upside, non-obvious matches within safety bounds |
| 3f | Train and deploy **Readiness Head**: timing-dependent conversion efficiency, pair-exposure timing | Matches surfaced when users are most receptive; reduces mistimed exposure |
| 3g | Deploy **calibration layer** across all four heads | Stable, interpretable thresholds; governed decision behavior |
| 3h | Deploy **gated rerank policy**: Likelihood gates → Intensity ranks → Chemistry modifies → Readiness times | Full four-head decision pipeline operational |

**What changes for the product:**

- Matchmaking evolves from automated execution to **multi-dimensional machine intelligence**
- The system separately reasons about viability, strength, bounded novelty, and timing
- Decisions are not just fast — they are **structurally smarter**
- New heads can be added without restructuring the engine

**Gate:** All four heads live. Full system outperforms automated baseline on downstream conversion. Chemistry shows real outcome lift. Readiness reduces mistimed rejections. Calibration stable.

---

### **Milestone 4 — Simulation & Learning**

**Period:** July – October 2026 (2–4 months)

**What ships:**

| # | Deliverable | Result |
| --- | --- | --- |
| 4a | Deploy **two-speed learning**: fast loop (near-real-time state updates) + slow loop (batch retraining with Tier 1–Tier 4 hierarchical signal weighting) | System learns from its own decisions at two timescales |
| 4b | Deploy **counterfactual logging**: full candidate pool scores, policy modifiers, exposure probabilities for ALL candidates (not just selected) | Off-policy evaluation and missed-opportunity analysis operational |
| 4c | Implement **delayed reward attribution**: event timelines per pair, 7-day outcome windows, downstream outcomes attributed back to head predictions at decision time | Training incorporates real outcomes (dates, post-date quality), not just proxies |
| 4d | Build **replay framework**: re-run decisions on historical logs; compare actual vs counterfactual; estimate regret | Policy improvements validated offline before production deployment |
| 4e | Deploy **shadow mode**: new models/policies generate decisions without affecting users; automated comparison | Regressions caught before reaching production |
| 4f | Deploy **continuous calibration monitoring**: automated drift detection, per-segment refresh, alerting | Calibration drift caught within 48h; thresholds remain meaningful |
| 4g | Execute **first full learning cycle**: exposure → outcome → label → retrain → evaluate → deploy improved model | The flywheel completes its first full turn |

**What changes for the product:**

- The system **improves from the consequences of its own decisions**
- Better decisions create better outcomes, which create stronger learning data, which improve future decisions
- The compounding flywheel begins — this is the transition from a matching product to a **self-improving decision engine**

**Gate:** First full learning cycle completed. Retrained model outperforms previous version. Counterfactual analysis producing actionable insights. Flywheel evidence visible. Calibration stable.

---

### **Milestone 5 — Scalable Agentic Social Intelligence**

**Period:** October 2026 → Ongoing (continuous)

**What ships:**

| # | Deliverable | Result |
| --- | --- | --- |
| 5a | Establish **continuous improvement cadence**: weekly retrain, calibration refresh, plugin review, monthly policy audit, quarterly marketplace health assessment | System intelligence compounds week over week |
| 5b | Deploy **exploration diagnostics**: budget efficiency, over/under-exploration rates, missed-opportunity rate per cohort | Exploration is governed, measured, and improving |
| 5c | Build **synthetic simulation**: approximate user behavior models for policy testing at scale | Iteration speed no longer bottlenecked by slow real-world feedback |
| 5d | Add **new decision heads** as understanding grows (e.g. Communication Compatibility, Long-Term Potential) | Decision surface continuously expands without restructuring the core |
| 5e | Extend intelligence **beyond dating**: apply the decision engine framework to broader social contexts | Category-defining agentic social network |
| 5f | Publish **internal system scorecard**: per-head quality, decision efficiency, product funnel, marketplace health, fairness — updated weekly | Full transparency and accountability across the system |

**What changes for the product:**

- Ditto operates as a **scalable, self-improving agentic social intelligence system**
- New matchmaking behaviors ship as skills and heads, not as core code changes
- The system gets structurally better with every iteration
- Intelligence extends beyond dating into a **category-defining agentic social network**

**Gate (ongoing):** Week-over-week lift measurable. Each retrain cycle produces improvement. Simulation insights translate into production gains. The flywheel is turning. The system is compounding.

---

### **Calendar Summary**

| Milestone | What | When | Duration |
| --- | --- | --- | --- |
| **M1** | Seven-Feature Foundation | Dec ’25 – Feb ’26 | 3 months ✔ Done |
| **M2** | Automated Matchmaking (OpenClaw Agent OS) | **April 2026** | 2–5 weeks |
| **M3** | Intelligent Decision System (4 heads + backbone) | **April – June 2026** | 2–3 months |
| **M4** | Simulation & Learning (closed-loop flywheel) | **July – October 2026** | 2–4 months |
| **M5** | Scalable Agentic Social Intelligence | **October 2026 →** | Ongoing |

**M2 + M3 run in parallel.** Data infrastructure runs continuously from day one.

**Key dates:**

- **April 2026** — automated matches live in production
- **June 2026** — full four-head intelligent decision system live
- **October 2026** — flywheel turning; system improving from its own decisions
- **Onward** — scalable agentic social intelligence compounding

## **Training Matchmaking Models from Live Interactions

Agentic Reinforcement Learning**

*The matchmaking engine described in this document makes decisions. This section specifies how it **learns from the outcomes of those decisions** — continuously, from live interactions, using agentic reinforcement learning. It covers the mathematical formulation of the training objective, the reward design for each of the four heads, the asynchronous training pipeline, the per-plugin and per-state-encoder learning mechanisms, and the safety constraints that keep the system governed while it improves. This is not a future research direction. It is the operational specification for how Ditto’s matchmaking models train from real-world outcomes.*

---

## **1. The Core Idea: Next-State Signals Are Training Data**

The main document's Learning & Feedback System opens with: *"Most matching systems ship a model and hope. This one learns."*

The mechanism for that learning is next-state signal recovery. Every matchmaking decision produces a next-state signal — the user's response, the interaction that follows, and eventually the downstream outcome. Research on agent-based reinforcement learning has shown that these signals encode two recoverable forms of information:

**Evaluative signals** answer: *Was this decision good or bad?* A user who skips a candidate, a conversation that dies after three messages, a date that never happens — these are implicit evaluations of the match decision. They can be converted into scalar process rewards via a PRM (Process Reward Model) judge.

**Directive signals** answer: *How should this decision have been different?* A user who edits preferences afterward ("I want someone more adventurous"), a post-date comment ("we had nothing in common"), or a pattern of re-examining previously filtered candidates — these carry directional information about what the system should change. They can be recovered through Hindsight-Guided On-Policy Distillation (OPD), which extracts the directive content and converts it into token-level advantage supervision.

The main document's signal tier hierarchy (Learning Targets by Head) already defines four tiers of outcome quality. This maps directly onto Ditto’s signal tier hierarchy:

| Main document tier | Signal type | RL method | Why this method |
| --- | --- | --- | --- |
| **Tier 4** — Proxy behavior (likes, skips, dwell) | Evaluative only | **Binary RL** — scalar reward | Abundant, low-quality; provides broad gradient coverage across all decisions |
| **Tier 3** — Early interaction (match, chat start) | Evaluative only | **Binary RL** — scalar reward | Moderate quality; still mostly implicit |
| **Tier 2** — Mid-funnel (contact exchange, deep conversation) | Evaluative (strong) | **Binary RL** — strong scalar reward | Clear positive signal; user crossed a commitment threshold |
| **Tier 1** — Downstream outcomes (dates, post-date sentiment, continuation) | **Evaluative + Directive** | **Binary RL + OPD** — scalar reward AND token-level advantage | Highest quality; often carries explicit or inferable directive content |

**Why the combination matters:** Binary RL alone reduces all information to +1/-1/0 per decision. This works for abundant Tier 3-4 signals but wastes the rich content in Tier 1-2 signals. OPD alone requires extractable directive content, which is sparse. Combining them ensures the system learns from the **full spectrum** — coarse-but-broad from Binary RL, sparse-but-precise from OPD.

---

### **The Unified Advantage: You Don’t Have to Choose**

Binary RL and OPD are not competing methods. They share the **same PPO loss structure** and differ only in how the advantage At*At* is computed. This makes them **natively complementary** — they combine into a single training objective with zero architectural overhead.

---

**The PPO Clipped Surrogate Objective:**

![image.png](Ditto%20Matchmaking%20Engine%203%20x/image%208.png)

---

**The Unified Advantage At:**

The key innovation is that the advantage At*At* combines both signal types in one expression:

![image.png](Ditto%20Matchmaking%20Engine%203%20x/image%209.png)

---

**How the two terms interact across signal tiers:**

![image.png](Ditto%20Matchmaking%20Engine%203%20x/image%2010.png)

The system **always** has the Binary RL term (broad coverage). It **additionally** has the OPD term when directive content is present (sparse but rich). No signal is wasted.

---

**Why the OPD term is fundamentally richer than scalar reward:**

Binary RL assigns the same advantage to every token in the response: if the match decision was good (+1), every aspect of the decision is equally reinforced. But not every aspect was equally good.

![image.png](Ditto%20Matchmaking%20Engine%203%20x/image%2011.png)

Within a single match decision, some aspects may be reinforced while others are suppressed. This is training signal that no scalar reward can provide.

---

**The PRM Judge and Hint Extraction:**

![image.png](Ditto%20Matchmaking%20Engine%203%20x/image%2012.png)

**Quality filtering for OPD:**

- Only hints from positive-scored votes are used
- Hints must be >10 characters and actionable
- Among valid hints, the longest (most informative) is selected
- If no valid hint exists, the sample falls back to Binary RL only

This strict filtering ensures OPD trains only on **high-confidence directional signals**. Binary RL handles everything else.

---

### **The Engine Is Already Running**

The most important property of this integration is that **it requires no separate data collection phase**.

![image.png](Ditto%20Matchmaking%20Engine%203%20x/image%2013.png)

The four async components ensure **zero blocking**:

| Component | Runs | Does not wait for |
| --- | --- | --- |
| **Policy Serving** | Serves the next match decision | Training to finish |
| **Environment** (user interaction) | Produces next-state signals | Serving or judging |
| **PRM Judge** | Evaluates previous decisions | Serving or training |
| **Training Engine** | Updates policy weights | Serving, environment, or judging |

The model improves **while serving**, and serves **while improving**. Graceful weight updates swap parameters between serving requests with zero downtime.

---

**What each interaction teaches, mapped to the architecture:**

| User action | Next-state signal | Which component learns | What it learns |
| --- | --- | --- | --- |
| Swipe right / left | Like / skip | **Likelihood Head** | Was the feasibility prediction correct for this direction? |
| Both swipe right | Mutual match | **Likelihood Head** | Mutual feasibility confirmed; reinforce this pair pattern |
| Deep conversation | Message depth, reply quality | **Intensity Head** | Predicted interaction strength vs actual — calibrate |
| Contact exchanged | Commitment threshold crossed | **Intensity Head** | Strong Tier 2 signal; upgrade reward for this decision |
| Date occurs | Real-world outcome | **All heads** | Strongest evaluative signal; upgrade to Tier 1 reward |
| Post-date: "amazing connection" | Positive feedback + directive | **Chemistry Head** (OPD) | What kind of complementarity worked? Upweight this pattern |
| Post-date: "no chemistry" | Negative feedback + directive | **Chemistry Head** (OPD) | Where did the Chemistry Head over-predict? Correct |
| Rejection despite high scores | Mistimed exposure | **Readiness Head** | User was not receptive; do not attribute rejection to pair quality |
| Preference edit | Explicit directive | **State Encoder** (OPD) | How should the user’s state representation change? |
| Anomalous session (binge swipe) | Flagged by intent classifier | **Robustness guardrails** | Down-weight or quarantine; do not train on this session |
| Plugin-injected diverse match | Exploration outcome | **Plugin policy** | Did exploration produce discovery or noise? Adjust budget |

**The engine is already running. Every interaction is already producing training data. The RL training loop ensures the system is learning from every revolution of the flywheel.**

---

## **2. How Each Head Receives Training Signal**

The main document specifies per-head learning targets. Each head receives a concrete reward and distillation mechanism.

### **2.1 Likelihood Head — Feasibility & Mutuality**

**Main document specification:** Primary learning signal = mutual match / viable interaction. Supporting signals = chat start, rejection patterns.

**Reward design:**

The Likelihood Head models **directional feasibility**. For a pair (A, B), the head predicts:

![image.png](Ditto%20Matchmaking%20Engine%203%20x/image%2014.png)

The PRM judge assigns **directional rewards** — a skip from User B does not mean the head was wrong about A’s interest:

![image.png](Ditto%20Matchmaking%20Engine%203%20x/image%2015.png)

**Why directional rewards matter for Likelihood:** The main document specifies that Likelihood models P(A→B) and P(B→A) separately. The PRM judge should assign **directional** rewards: a skip from User B does not mean the Likelihood Head was wrong about A's interest — it means the B→A direction was wrong. This requires the judge to have access to the pair-level decision trace from the Data Infrastructure's Decision Trace Layer.

**OPD opportunity for Likelihood:** Rare but high-value. When a user explicitly says "I would never be interested in someone like that" (a strong directive signal about feasibility boundaries), the OPD pipeline extracts: *"this user has a hard boundary on [attribute]. The Likelihood Head should have gated this pair."* This teaches the head not just that it was wrong, but **why**.

### **2.2 Intensity Head — Strength of Interaction**

**Main document specification:** Primary learning signal = engagement depth, contact exchange, dates. Supporting signals = message depth, reply latency.

**Reward design:**

The main document's Architecture Overview defines a four-tier strength hierarchy for Intensity:

| Strength tier | Signal | RL reward | Attribution window |
| --- | --- | --- | --- |
| **Tier 1 (strongest)** | Date occurrence, post-date continuation | +1 (Tier 1 weight: 4x) | 14-day window |
| **Tier 2** | Contact exchange | +1 (Tier 2 weight: 3x) | 7-day window |
| **Tier 3** | Conversation depth threshold, sustained exchange | +1 (Tier 3 weight: 2x) | 3-day window |
| **Tier 4 (weakest)** | Conversation start, first message sent | +1 (Tier 4 weight: 1x) | 24h window |

**Hierarchical reward accumulation:** As stronger signals arrive over time, the reward for the original match decision is **upgraded** using a **tier-weighted progressive scheme**:

![image.png](Ditto%20Matchmaking%20Engine%203%20x/image%2016.png)

**OPD for Intensity:** When post-date feedback says "the conversation was great but there was no real chemistry in person," the OPD pipeline extracts: *"this pair had high conversational Intensity but the head over-predicted physical/in-person pull. Adjust: Intensity should weight in-person compatibility signals, not just text-based engagement."*

### **2.3 Chemistry Head — Bounded Novelty & Complementarity**

**Main document specification:** Primary learning signal = positive deviation vs baseline expectation. Supporting signals = uplift over predicted intensity.

**Reward design:**

Chemistry is the hardest head to provide reward for, because it is defined as **residual uplift** — performance above what Likelihood + Intensity already predict. The PRM judge for Chemistry must:

1. Retrieve the Likelihood and Intensity predictions for this pair (from the Decision Trace Layer)
2. Compute the **expected baseline outcome** from those two predictions
3. Compare the **actual outcome** against that baseline
4. Award a Chemistry reward only if the actual outcome **exceeded** the baseline

The Chemistry reward is defined as **residual uplift** over the Likelihood + Intensity baseline:

![image.png](Ditto%20Matchmaking%20Engine%203%20x/image%2017.png)

**OPD for Chemistry:** When a user provides feedback on a Chemistry-promoted match — "I never would have picked them myself, but we had an amazing connection" — the OPD pipeline extracts: *"this pair worked because of [specific complementarity]. Chemistry should upweight this pattern."* Conversely, "they seemed interesting on paper but we had nothing to talk about" extracts: *"the Chemistry Head over-valued profile-level novelty without pair-level conversational fit."*

**Integration with the main document's Chemistry features (A-E):** Each OPD hint is tagged with the feature family it relates to:

- **Complementarity Features** — did functional complementarity produce real uplift?
- **Interaction-Style Features** — did conversational fit predictions hold?
- **Pair Cross-Features** — did pair-level trait interactions behave as expected?
- **Novelty-within-Safety Features** — did bounded novelty produce discovery or noise?
- **History-Aware Features** — did broadening the user's archetype pattern help?

### **2.4 Readiness Head — Temporal Alignment & Receptivity**

**Main document specification:** Primary learning signal = timing-dependent conversion efficiency. Supporting signals = response conditional on state.

**Reward design:**

Readiness rewards require a **counterfactual comparison**: did this pair perform differently because of *when* it was surfaced, holding pair quality constant?

The Readiness reward is defined as the **timing-conditional lift**:

![image.png](Ditto%20Matchmaking%20Engine%203%20x/image%2018.png)

| Scenario | PRM evaluation | Readiness reward |
| --- | --- | --- |
| High-quality pair + high readiness = strong outcome | Readiness was correct to surface now | +1 |
| High-quality pair + low readiness = weak outcome | Readiness should have delayed; timing polluted the outcome | -1 (flag: do not attribute negative outcome to Likelihood/Intensity/Chemistry) |
| Moderate pair + high readiness = strong outcome | Readiness correctly identified a conversion-friendly window | +1 |
| Any pair surfaced during detected anomalous session | Readiness should have gated entirely | -2 (strong negative; connects to robustness guardrails) |

**Critical integration with the Temporal State Encoder:** The main document's robustness table (anomaly detection, intent classification, reversion-to-prior, feedback loop dampening) defines four guardrail mechanisms. The Readiness Head's RL training must be **constrained by these guardrails**: if a session is classified as anomalous by the intent classifier, the Readiness Head should not train on that session's outcomes.

**OPD for Readiness:** When a user rejects a strong candidate and later says "I wasn't in the mood for that kind of match today," the OPD pipeline extracts: *"User was in a low-receptivity state despite being active. Readiness Head should have detected this and delayed exposure."*

---

## **3. The PRM Judge: Detailed Design for Matchmaking**

### **3.1 Judge Architecture**

The PRM judge converts raw matchmaking outcomes into training-usable rewards. It is a model-based evaluator that reasons about the quality of a match decision given the outcome and the decision context.

**Input to the judge (from the Data Infrastructure):**

From the Decision Trace Layer:

- User state vector at decision time (taste profile, receptivity, exploration appetite, outcome trajectory, archetype saturation)
- Candidate pair representation (pair cross-features, directional encoding A to B and B to A)
- All four head predictions: Likelihood=L, Intensity=I, Chemistry=C, Readiness=R
- Calibrated scores for each head
- Plugin modifiers applied (which plugins intervened, what parameters)
- Final decision: which candidate was surfaced, ranking position

From the Event Layer:

- The outcome event chain: like/skip, match, conversation, contact, date, feedback
- Timestamps for each event
- User behavior after the decision (preference edits, session changes)

**Output from the judge:**

For **Binary RL**: a reward r in {+1, -1, 0} per decision, per head, via majority vote across m=3 independent queries.

For **OPD**: if the outcome contains directive content, a textual hint in [HINT_START]...[HINT_END] format, filtered for quality (more than 10 characters, actionable, concise).

### **3.2 Per-Head Judge Prompts**

**Likelihood Judge Prompt:**

"You are evaluating whether a matchmaking decision correctly assessed pair feasibility. You have the decision context including user states, pair representation, and the Likelihood prediction P(A to B), P(B to A), and mutual feasibility score. You also have the outcome. Evaluate: Was the Likelihood Head's feasibility assessment correct? Score +1 if feasibility was confirmed, -1 if it was wrong, 0 if unclear. If directive information is present (user stated WHY they skipped), extract hint."

**Intensity Judge Prompt:**

"You are evaluating whether a matchmaking decision correctly predicted interaction strength. You have the Intensity prediction and the actual outcome tier (Tier 1: date, Tier 2: contact exchange, Tier 3: deep conversation, Tier 4: conversation start). Evaluate: Did the Intensity Head correctly predict how strong this interaction would be? Score accordingly. If post-interaction feedback reveals WHY the conversation was weak or strong, extract hint."

**Chemistry Judge Prompt:**

"You are evaluating whether Chemistry added real value to this match decision. You have the Likelihood and Intensity scores (baseline prediction), the Chemistry score (residual uplift prediction), and whether this was a Chemistry-promoted match. Evaluate: Did the actual outcome significantly exceed the baseline? If yes, Chemistry found genuine uplift (+1). If outcome matched baseline, Chemistry added nothing (0). If outcome was below baseline, Chemistry promoted noise (-1). If user feedback reveals WHY the match surprised, extract hint referencing which Chemistry feature family was relevant."

**Readiness Judge Prompt:**

"You are evaluating whether a match was surfaced at the right time. You have the user state at decision time, the Readiness prediction, the pair quality scores (L, I, C), and the expected outcome at that quality level. Evaluate: Was the timing right? If outcome matched or exceeded expected for this pair quality, timing was right (+1). If outcome was significantly below expected despite good pair quality, it was mistimed (-1). If the user was in an anomalous session, it should not have been surfaced (-2). If evidence suggests WHY timing was wrong, extract hint."

### **3.3 Majority Vote and Quality Filtering**

Each judge query runs m=3 times independently with temperature=0.6. The final reward is the majority vote. For OPD hints:

- Only hints from positive-scored votes are considered
- Hints must be more than 10 characters and actionable
- Among valid hints, the longest (most informative) is selected
- If no valid hint exists, the sample is used for Binary RL only (OPD is dropped for this sample)

---

## **4. The Asynchronous Training Pipeline**

### **4.1 Four Decoupled Components**

The main document's Closed-Loop Learning section specifies a two-speed learning architecture (fast loop + slow loop). The agent-RL integration implements this through four async components:

**Component 1: Policy Serving**

- Serves live match decisions to users
- Logs decision traces (candidate pool, head scores, calibrated outputs, plugin modifiers, final selection) to the Decision Trace Layer
- **Never interrupted** by training — graceful weight updates swap model parameters between serving requests

**Component 2: Environment (User Interaction)**

- The "environment" is the user's interaction with the surfaced match
- Generates next-state signals as events unfold: like/skip, match, conversation, contact, date, feedback
- Each event is appended to the Event Layer with timestamps
- **Session-aware classification**: main-line turns (trainable match decisions) vs side turns (profile browsing, settings changes — logged but not trained on)

**Component 3: PRM Judge**

- Runs asynchronously — does not block serving or environment
- Triggered when a next-state signal arrives for a previously logged decision
- Queries the Decision Trace Layer for the original decision context
- Produces per-head Binary RL rewards and OPD hints
- Supports **progressive re-evaluation**: an initial Tier 4 reward (like/skip) can be upgraded to Tier 1 (date) when stronger signals arrive days later

**Component 4: Training Engine**

- Accumulates samples from the PRM judge into a training buffer
- Applies gradient updates using the combined advantage: A_t = w_binary * r_final + w_opd * (teacher - student)
- Implements the two-speed architecture:
    - Fast updates: state encoder parameters refreshed from recent interaction patterns
    - Slow updates: head parameters and calibration refreshed from accumulated Tier 1-2 outcomes
- Produces updated model weights for graceful deployment to Component 1

### **4.2 How the Four Components Map to the Data Infrastructure Layers**

| RL component | Reads from | Writes to |
| --- | --- | --- |
| **Policy Serving** | State Snapshot Layer, current head models | Decision Trace Layer, Experimentation Layer |
| **Environment** | (user actions are external) | Event Layer |
| **PRM Judge** | Decision Trace Layer, Event Layer, State Snapshot Layer | Outcome & Label Layer (per-head rewards, OPD hints) |
| **Training Engine** | Outcome & Label Layer, Decision Trace Layer | Updated model weights to Policy Serving |

This mapping ensures every component reads from and writes to the **same Data Infrastructure** defined in the main document. There is no parallel data path. One coherent data contract across serving, judging, training, and evaluation.

---

## **5. RL-Trainable Agent Skills and Plugins**

### **5.1 Making Each Plugin RL-Trainable**

Each plugin decision is formalized as a **parameterized policy** with a learnable parameter vector updated via the RL training loop.

**Feasibility Threshold Gate:**

| Aspect | Current design | RL-trainable design |
| --- | --- | --- |
| Parameter | Fixed mutual feasibility threshold (e.g. 0.6) | Threshold is a function of user state: threshold(state) = f(receptivity, archetype_saturation, outcome_trajectory) |
| Action | Filter all candidates below threshold | Filter with state-adaptive threshold |
| Reward signal | Did the gating decision lead to better downstream outcomes? | PRM judge compares outcomes of gated vs non-gated candidates |
| Update | A/B test, quarterly | Continuous gradient updates from every gating decision |

Concrete example: A user in a convergent phase (strong recent outcomes, low archetype saturation) benefits from a **tighter** threshold (0.7) — only very strong candidates pass. A user in an exploratory phase (recent disappointments, high archetype saturation) benefits from a **looser** threshold (0.5) — more candidates pass, increasing discovery surface.

**Diversity Injector:**

| Aspect | Current design | RL-trainable design |
| --- | --- | --- |
| Parameter | Fixed: inject 1 non-top-ranked candidate per session | Injection rate and candidate selection are learned |
| Action | (a) inject or not, (b) which candidate to inject | Both decisions are part of the policy's action space |
| Reward signal | Did the injected candidate outperform a typical top-ranked candidate in similar sessions? | PRM judge compares outcomes of injected vs non-injected sessions |

The RL-trained diversity injector discovers that diversity helps most when the user's recent matches have been homogeneous AND recent outcomes have been flat, but hurts when the user is in a convergent phase with strong recent chemistry.

**Exploration Budget Controller:**

| Aspect | Current design | RL-trainable design |
| --- | --- | --- |
| Parameter | Fixed global exploration budget (e.g. 5%) | Per-user-state exploration budget |
| Action | Allocate fraction of exposures to non-top-ranked but feasible candidates | Same, but adapts to user state and recent exploration outcomes |
| Reward signal | Did exploration exposures lead to discovery or waste? | PRM judge tracks exploration-specific outcomes vs baseline |

**Readiness-Aware Delay:**

| Aspect | Current design | RL-trainable design |
| --- | --- | --- |
| Parameter | Fixed readiness threshold | Delay policy: delay(pair_quality, readiness_score, user_state) produces surface now / delay 1 session / delay 2-3 sessions |
| Action | Surface or delay | Surface / delay with learned hold duration |
| Reward signal | Did delayed pairs convert better when eventually surfaced? | PRM judge compares timing-specific outcomes |

### **5.2 Plugin Safety Constraints**

With RL-trainable plugins, the main document's anti-pattern — "plugins should not become a dumping ground for model weaknesses" — must be **enforced in the training objective**:

1. **Hard constraints on plugin parameter ranges**: feasibility threshold cannot drop below 0.3 (safety floor); exploration budget cannot exceed 20% (quality protection)
2. **Plugin KPI monitoring**: if a plugin's learned policy degrades its own KPI below the pre-RL baseline, training is paused and the plugin reverts to pre-RL parameters
3. **Kill-switch**: if plugin behavior causes unexpected regressions in system-level metrics, revert to last known-good parameter set within one serving cycle
4. **Audit trail**: every plugin parameter update is logged in the Experimentation Layer with exact training samples, rewards, and advantage values

---

## **6. Learnable Temporal State Encoder**

### **6.1 What Agentic RL Adds: Learnable Update Mechanics**

Currently, the temporal attenuation weights and guardrail thresholds are hand-designed. With agent-RL, these become learnable:

**Learnable attenuation weights:**

The current fixed attenuation becomes a **learned function** of user type:

![image.png](Ditto%20Matchmaking%20Engine%203%20x/image%2019.png)

- The 50-80% / 20-50% recency split becomes a per-user-state function
- Users with stable preferences learn a lower recency weight (historical data stays relevant longer)
- Users who shift frequently learn a higher recency weight (recency should dominate more)
- Learning signal: does the state encoder's output, when used by the four heads, lead to better match decisions?

**Learnable anomaly thresholds:**

- The anomaly detection currently uses fixed statistical thresholds
- With agent-RL, the **intent classifier** (Normal/Exploration/Noise/Manipulation) becomes trainable:
    - Input: session-level behavioral features (swipe velocity, acceptance rate, session duration, time-of-day)
    - Output: intent classification + confidence
    - Reward: was the classification correct? If a session classified as "Normal" was actually noise (subsequent behavior reverted, outcomes degraded), the classifier receives -1

**Learnable feedback loop dampening:**

- The dampening strength is learned:
    - Input: recent state trajectory (is receptivity declining over sessions?)
    - Action: increase dampening (break the cycle) or decrease dampening (the decline is genuine, not a loop)
    - Reward: did the intervention stabilize or worsen outcomes in subsequent sessions?

### **6.2 OPD for State Encoder Updates**

When a user provides explicit directive feedback — "I'm realizing I prefer someone with more ambition" — the OPD pipeline:

1. Extracts the directive hint from the user's message
2. Constructs an enhanced context: "if the state encoder had known this preference before the last 5 match decisions, what would it have produced?"
3. Computes the token-level advantage between the hint-enhanced state and the actual state at decision time
4. Updates the state encoder to produce state representations that would have led to better match decisions in hindsight

**Why this matters:** A rule-based preference update would simply flip a flag: "user now values ambition." OPD teaches the encoder **how** to represent ambition-preference in the context of this specific user's broader state — including how it interacts with their receptivity, their recent outcome trajectory, and their archetype exposure history. This is representation learning, not feature engineering.

---

## **7. Handling Matchmaking-Specific Challenges**

### **7.1 Delayed Rewards (Days to Weeks)**

**Step-Wise Process Rewards**

In long-horizon matchmaking interactions, outcome-only rewards provide gradient signal only at the terminal event (e.g. a date), leaving the vast majority of intermediate decisions (likes, conversations, contact exchanges) unsupervised. Process rewards solve this by assigning a reward to **each decision step** based on the next-state signal.

For a match decision at step t*t* within a user’s interaction trajectory, the **integrated reward** combines the outcome reward with the mean of step-wise PRM scores:

![image.png](Ditto%20Matchmaking%20Engine%203%20x/image%2020.png)

where o*o* is the final outcome reward (if available), and ri*ri* are independently assigned by the PRM judge. This ensures dense credit assignment throughout the interaction trajectory, not just at the terminal step.

For **step-wise advantage standardization**, actions at the same step index are grouped together, and advantages are standardized within each group — since matchmaking interactions do not have easily clusterable states, unlike structured tasks.

---

**Progressive reward assignment:**

- Day 0: Match decision made. Decision Trace logged.
- Day 0: User likes/skips. Tier 4 reward assigned immediately.
- Day 1: Conversation starts. Tier 3 reward; previous reward overridden for training.
- Day 3: Contact exchanged. Tier 2 reward; previous rewards overridden.
- Day 8: Date occurs. Tier 1 reward; all previous rewards overridden.
- Day 10: Post-date feedback. Tier 1 reward + OPD hint extracted.

Training entry for this decision is UPDATED at each stage. The effective reward at any point is:

![image.png](Ditto%20Matchmaking%20Engine%203%20x/image%2021.png)

Only the highest-tier reward reaches the slow-loop training batch. Fast-loop state updates use each signal as it arrives.

The training engine's batch constructor always uses the **latest available reward** for each decision, not the first one received. This is implemented via the Data Infrastructure's time-versioned state snapshots — the training engine joins on decision_id and takes the max-tier reward.

### **7.2 Two-Sided Evaluation**

The judge produces **separate rewards for each direction**:

- r(A to B): did User A benefit from this match exposure?
- r(B to A): did User B benefit from being surfaced to User A?
- r(pair): mutual outcome quality

For training:

- The **Likelihood Head** receives directional rewards (r_A_to_B and r_B_to_A separately)
- The **Intensity Head** receives the pair-level reward (r_pair)
- The **Chemistry Head** receives the pair-level reward vs baseline
- The **Readiness Head** receives directional rewards (timing may be right for A but wrong for B)

### **7.3 Exploration Safety**

With RL-trainable exploration, safety constraints must be **algorithmically enforced**:

1. **Feasibility floor**: No RL-learned policy can surface a candidate below the minimum mutual feasibility threshold. This is a hard constraint, not learnable.
2. **Chemistry safety bound**: Chemistry-promoted exploration can only increase a candidate's rank by at most K positions (e.g. K=3).
3. **Trust-aware gating**: The trust-aware plugin operates as a non-overridable filter. No RL gradient can promote a candidate flagged by the trust system.
4. **Exploration budget cap**: Hard ceiling (e.g. 20% of exposures). Even if RL learns more exploration is beneficial, it cannot exceed the cap without explicit manual override.

### **7.4 Cold-Start Users**

1. **No RL training on cold-start decisions**: Decisions made during the cold-start window (before state encoder reaches maturity threshold of N=10+ meaningful interactions) are used for evaluation only, not training.
2. **Accelerated OPD**: Cold-start users who provide explicit preference signals receive prioritized OPD processing.
3. **Cohort-level RL**: For cold-start users, RL rewards are aggregated at the cohort level rather than individual level.

---

## **8. Evaluation and Safety Monitoring Under RL Training**

### **8.1 Head-Level Monitoring Under RL Training**

For each head, track:

- Pre-RL baseline performance (from the A/B test gates in the rollout plan)
- RL-trained performance on the same held-out evaluation set
- Calibration stability under RL updates (Risk R7: Calibration Drift)
- Head independence: are the heads maintaining distinct decision semantics, or is RL training causing them to collapse? (Risk R4: Readiness Absorbs Everything)

**Automated guardrail**: if any head's held-out performance drops below its pre-RL baseline for two consecutive evaluation windows, RL training for that head is paused and last known-good weights are restored.

### **8.2 Decision-Level Monitoring**

- Exposure efficiency before and after RL training
- Counterfactual regret using logged candidate pools
- Plugin attribution: for each RL-trained plugin, what fraction of decisions were modified and what was the outcome difference?

### **8.3 Product-Level Monitoring**

- Funnel metrics (match-to-conversation, conversation-to-contact, contact-to-date) before and after RL training
- Post-date satisfaction trend
- Retention impact

### **8.4 System-Level Monitoring**

- Marketplace health: is RL training causing exposure concentration? (Risk R9: Feedback Loop Amplification)
- Fairness: are some cohorts benefiting more from RL improvements than others?
- Diversity: is the RL-trained exploration policy maintaining or improving archetype diversity?

## **9. Risks Specific to Agentic RL Training**

### **B-R1. Signal Sparsity**

**What breaks:** Dating outcomes are much sparser than conversational signals. A homework interaction produces feedback in minutes; a date produces feedback in weeks.

**How it manifests:**

- Tier 1 rewards arrive too slowly for meaningful gradient accumulation
- The model overfits to abundant Tier 3-4 proxies because Tier 1-2 signals are too rare
- OPD training is starved of directive hints because post-date feedback is sparse

**Mitigation:**

- Use the full Tier 1-4 hierarchy with hierarchical weighting
- Binary RL on Tier 3-4 provides continuous gradient even when Tier 1-2 is sparse
- OPD trades sample quantity for quality — designed to work with sparse, high-value signals
- Progressive reward assignment ensures early signals provide fast feedback while stronger signals override later
- Simulate accelerated feedback cycles during development using synthetic user models

### **B-R2. Reward Misspecification**

**What breaks:** The PRM judge may systematically misassign rewards, causing the model to learn the wrong behavior.

**How it manifests:**

- Judge assigns +1 to Chemistry-promoted matches that the user accepted but later regretted
- Judge cannot distinguish timing failures from compatibility failures
- Judge quality degrades for edge cases

**Mitigation:**

- Majority vote (m=3+) reduces single-judge errors
- Per-head judge prompts ensure the judge reasons about the right question for each head
- Track judge accuracy against ground-truth labels (manually validated subset)
- If judge error rate exceeds threshold, retrain or re-prompt the judge before continuing RL training
- Include head-level predictions in the judge context so it can distinguish head-specific errors

### **B-R3. Privacy in the Learning Loop**

**What breaks:** User interaction data flowing through PRM judging and OPD hint extraction could memorize individual users' match decisions, preferences, or feedback in the model weights.

**Mitigation:**

- Differential privacy noise applied to training gradients
- OPD hints are abstracted by the judge — raw user feedback is distilled into general principles, not specific details
- Training aggregates at cohort level, not individual level, for model weight updates
- Confidential API keys for environment communication
- Comply with the Data Infrastructure's privacy and access controls

### **B-R4. Exploration Harm**

**What breaks:** RL-driven exploration surfaces matches that are technically feasible but experientially poor.

**Mitigation:**

- Hard constraints: feasibility floor, Chemistry safety bound, trust-aware gating (cannot be overridden by RL)
- Exploration budget hard cap (20% of exposures max)
- Chemistry ablation: continuously measure system with vs without Chemistry-driven exploration
- The RL objective explicitly includes safety, fairness, and marketplace constraints

### **B-R5. Catastrophic Forgetting**

**What breaks:** Continuous online learning causes the model to forget previously learned patterns when the user distribution shifts.

**Mitigation:**

- KL penalty in the PPO objective constrains each update step to stay near the current policy
- Replay buffer of high-quality historical decisions mixed into training batches
- Evaluation guardrails: if held-out performance drops below pre-RL baseline, training is paused and weights are reverted
- State encoder's reversion-to-prior mechanism provides user-level protection against temporary distribution shifts

---

## **10. Strategic Conclusion**

Agentic RL is not a bolt-on feature. It is the **concrete mechanism** that turns the main document's self-improvement thesis from architecture into a running system.

The matchmaking document describes:

- Four heads that predict different aspects of match quality. The RL architecture provides **per-head training signals** (evaluative via Binary RL, directive via OPD).
- A temporal state encoder that adapts to users. agentic RL makes the **adaptation mechanism itself learnable**.
- Agent skills/plugins that govern decision policy. agentic RL makes **each plugin a trainable policy** that learns from outcomes.
- A closed-loop learning flywheel. The RL architecture provides the **four-component async infrastructure** (serving, environment, judging, training) to run this loop in production.
- A data infrastructure with six layers. agentic RL reads from and writes to **the same layers**, maintaining a single coherent data contract.
- A four-level evaluation framework. agentic RL training is **monitored and constrained** by this framework, with automated guardrails and revert mechanisms.

Every concept in this appendix references a specific section of the main document. Nothing is free-standing. The agent-RL integration extends the architecture — it does not replace any part of it.

The result: a matchmaking engine where **every swipe, every conversation, every date, and every piece of feedback makes the next match decision better** — not as an aspiration, but as a running system.

# Final Words

*Traditional systems ask: "Given profiles, who matches?"*

*Agentic systems ask: "Given what I BELIEVE about this user (uncertain), what I BELIEVE about candidates (uncertain), what I BELIEVE about my own model (might be wrong), and timing (maybe bad) — What action BEST advances this user toward connection while ALSO reducing my uncertainty?"*

A static ranking system can optimize local proxies. A true matchmaking engine can improve real outcomes.

A static ranking system can retrieve and sort. A true matchmaking engine can reason about viability, strength, bounded novelty, and timing as separate but connected dimensions.

A static ranking system can become more accurate within a narrow formulation. A true matchmaking engine can become more intelligent over time because it learns from the consequences of its own decisions.

That is the strategic shift.

Ditto is not merely trying to build a better matching surface. It is building a **stateful, governable, self-improving decision system for human relationships**.

![image.png](Ditto%20Matchmaking%20Engine%203%20x/image%2022.png)

Dating is the first domain. The larger opportunity is a category-defining **agentic social network** whose intelligence improves through real-world interaction, structured feedback, and controlled decision-making under uncertainty.

---

Over time, this architecture can support a more simulation-oriented control system, where replay, synthetic environments, counterfactual evaluation, and richer downstream supervision make the engine faster to improve and safer to evolve.

That future should be built on top of strong foundations, not used as a shortcut around them.

The immediate priority is clear:

**build the minimum coherent matchmaking engine that can reason separately about feasibility, strength, chemistry, and timing—and learn from real outcomes over time.**

![image.png](Ditto%20Matchmaking%20Engine%203%20x/image%2023.png)

# **Appendix A: Traditional Compatibility Frameworks as Chemistry Priors**

*This appendix provides detailed reference material on traditional compatibility systems and how they can inform Chemistry Head feature design. These frameworks are used as structured priors and hypothesis generators—not as deterministic rules or hard constraints. See the main document’s Chemistry section for architectural context.*

---

## **Ancient Wisdom Systems as Structured Priors for Chemistry**

Traditional cultures developed sophisticated matching frameworks over thousands of years. While these systems are not scientific in the modern experimental sense, they often encode recurring intuitions about human compatibility that are still relevant to modern matchmaking design.

For Ditto, the right way to use such systems is **not** as deterministic truth. It is as a source of **structured priors, hypothesis generation, and interpretable feature ideas** that may help the Chemistry Head reason about pair-level uplift, complementarity, and latent interaction dynamics.

In other words, these systems are useful **not because they should override modern learning**, but because they often point toward dimensions that a purely surface-level matcher would ignore.

### **Why they matter conceptually**

Across many traditional compatibility systems, several recurring ideas appear:

1. **Compatibility is multi-dimensional**
    
    Human matching should not be reduced to one scalar. Different aspects of fit matter differently.
    
2. **Complementarity is real**
    
    Many systems distinguish between harmful mismatch and constructive balance. This aligns with the modern idea that some forms of asymmetry can produce pair-level uplift.
    
3. **Energy / temperament patterns may matter**
    
    People often cluster into recurring behavioral or relational styles, and these styles may interact in meaningful ways.
    
4. **Weights matter**
    
    Not all dimensions of compatibility are equally important. Some factors may be foundational, while others are secondary modifiers.
    
5. **Compatibility may be context-dependent**
    
    Timing, cycles, pacing, and current state can affect whether a pair works well now.
    

These are not reasons to treat ancient frameworks as truth. They are reasons to treat them as **structured prior libraries** for feature ideation.

---

## **Vedic Ashtakoot as a Multi-Dimensional Compatibility Template**

One useful example is the **Vedic Ashtakoot system**, which models compatibility across eight distinct dimensions rather than reducing fit to one score.

In modernized language, the dimensions map surprisingly well onto concepts that a matchmaking system might care about:

- **Constitutional fit**
- **Emotional resonance**
- **Temperament compatibility**
- **Intellectual friendship**
- **Physical / intimacy style**
- **Lifestyle pace and wellbeing**
- **Power-dynamics balance**
- **Values or life philosophy alignment**

The key insight here is not astrology itself. The key insight is architectural:

> **compatibility may be composed of multiple dimensions with different weights, rather than one undifferentiated score.**
> 

That is directly relevant to chemistry modeling.

It suggests that the Chemistry Head should be comfortable representing multiple hidden dimensions of pair uplift rather than assuming one uniform form of spark.

It also suggests that some forms of fit may matter more than others. In modern terms, this means the system may learn that:

- emotional resonance matters more than superficial profile overlap
- intellectual friendship matters more than raw trait similarity for some users
- constitutional or pacing balance may matter more than shared hobbies

This is useful not as doctrine, but as a **feature-design intuition**.

---

## **Chinese Energy-Type Systems as Affinity and Clash Priors**

Chinese compatibility traditions, including BaZi-, zodiac-, and five-element-style reasoning, often organize people into recurring energy or temperament clusters.

Modernized, these systems point toward the idea that:

- some people cluster into recognizable relational styles
- some styles naturally stabilize one another
- some styles intensify one another
- some styles repeatedly clash

The important architectural takeaway is not whether any one taxonomy is “true.” It is that **pair interaction may depend on type-to-type dynamics**, not only user-level traits in isolation.

That is directly relevant to chemistry.

A Chemistry Head may benefit from modeling:

- affinity clusters
- tension patterns
- stabilizing pairings
- over-activation pairings
- complementary energy profiles
- repeated clash structures

This can be useful when generating pair-level features such as:

- energizing vs grounding
- reflective vs action-oriented
- intensity compatibility
- nurturing vs dominance balance
- emotional pacing fit

Again, the point is not to rank users by zodiac. The point is that traditional systems often preserved a valuable intuition:

> **compatibility may emerge from recurring interaction patterns between relational types, not just from static similarity.**
> 

---

## **Ayurvedic Dosha Logic as Balance and Over-Amplification**

Ayurvedic frameworks classify people according to broad constitutional styles and often reason about whether pairings balance one another or amplify one another excessively.

In modernized language, this suggests a useful chemistry principle:

> **some pairings stabilize; some pairings intensify; some pairings become volatile.**
> 

That is highly relevant to pair-level uplift.

For example, a chemistry-aware system may want to detect whether:

- one person’s intensity is balanced by another’s calm
- one person’s variability is grounded by another’s consistency
- both users amplify instability in one another
- both users create stagnation rather than momentum

This kind of thinking maps naturally onto features such as:

- emotional regulation compatibility
- pace and energy complementarity
- novelty-stability balance
- responsiveness asymmetry
- conflict-style interaction

The useful part is not the dosha labels themselves. The useful part is the structural idea that **good chemistry may sometimes come from balancing constitutions rather than maximizing sameness**.

---

## **What Ditto Can Learn from Ancient Systems**

If these systems are used carefully, they can contribute in four bounded ways.

### **1. Hypothesis generation**

They suggest candidate dimensions that modern systems may want to test, such as:

- emotional resonance
- constitutional balance
- intellectual friendship
- power-dynamics fit
- pacing compatibility
- temperament complementarity

### **2. Feature engineering**

They can inspire latent pair features or persona embeddings that are then validated against real outcomes.

### **3. Interpretability**

They can provide user-friendly language for why a match may feel resonant, balanced, grounding, energizing, or unusually aligned.

### **4. Cold-start priors**

They may offer weak initial structure before richer behavioral and downstream outcome data accumulate.

This is especially useful for chemistry because chemistry is one of the hardest signals to observe directly early on.

---

## **How Ancient Systems Should Be Used — and How They Should Not**

### **They may be used as**

- structured prior libraries
- exploratory feature families
- persona-clustering hypotheses
- interpretable compatibility vocabularies
- weak cold-start priors

### **They should not be used as**

- hard constraints
- deterministic compatibility rules
- unvalidated ranking overrides
- substitutes for user behavior and real outcomes
- unquestioned truth claims

The balanced principle is:

> **Ancient systems may provide useful priors about compatibility, complementarity, rhythm, and relational structure. But the modern matchmaking engine should treat them as hypotheses to test, not truths to obey.**
> 

In practice, this means the Chemistry Head may draw inspiration from these frameworks when forming pair-level features or priors, but final decisions should remain governed by:

- learned outcomes
- calibration
- feasibility constraints
- timing logic
- policy control

 **Pertinent Research Sources**

1. **People's stated preferences poorly predict actual attraction** (Eastwick & Finkel, 2008)
2. **Relationship effects (unique compatibility) matter more than partner effects (universal attractiveness)** (Kenny, 2019)
3. **Physical attraction is only 25% of chemistry** — Positive interaction (64%) matters more (Campbell & Campbell, 2023)
4. **Functional complementarity works** — Similar values + different skills, not "opposites attract" (Winch, 1958)
5. **Attachment style predicts relationship dynamics** — Avoid Anxious-Avoidant pairings (Fraley & Shaver, 2000)