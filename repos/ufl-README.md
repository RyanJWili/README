# ufl — Matchmaking Chat Analysis Pipeline

## Reference

> [User's Feedback Loop](https://www.notion.so/User-s-Feedback-Lifecycle-30bd2ecb07cc805982e3f66ec28b8846?source=copy_link)

---

## Overview

Analysis pipeline for Ditto's matchmaking chat data. Reads user conversations and match outcomes from PROD MongoDB Atlas, segments chat histories by topic via LLM agent, and produces structured quality metrics.

| Part | Directory | Description |
|------|-----------|-------------|
| 1 | `analysis/`  | Analyzer pipeline: signal extraction, LLM segmentation, grading, evaluation |
| 2 | `db/`        | MongoDB connection, indexes, and migrations |
| 3 | `config/`    | Global settings — loads `.env` for DB URIs and API keys |

---

## First-Time Setup

```bash
# 1. Copy environment template and fill in your keys
cp .env.example .env

# 2. Install dependencies (creates .venv/ automatically)
uv sync --dev

# 3. Start MongoDB (must be running before any pipeline step)
brew services start mongodb-community@8.0

# 4. Set your AI Gateway API key in .env
#    This single key routes to all LLM providers (OpenAI, Gemini, Claude, etc.)
#    via the Vercel AI Gateway. Configure the model in analysis/config/analyzer.toml.
AI_GATEWAY_API_KEY=your-key-here
```

---

## How to Run

The analysis pipeline reads from PROD MongoDB Atlas and writes to local MongoDB.
Run steps individually or all at once.

### Full pipeline (all steps)

```bash
uv run python -m analysis.pipeline
```

### Individual steps

```bash
uv run python -m analysis.pipeline --step signal     # Extract TP/FP signals from PROD matchings
uv run python -m analysis.pipeline --step segment    # LLM Analyzer: segment + classify chat histories
uv run python -m analysis.pipeline --step validate   # Deterministic graders (G0.3, G1.2, G1.4)
uv run python -m analysis.pipeline --step evaluate   # Quality metrics + run summary report
```

### Options

```bash
uv run python -m analysis.pipeline --limit 10        # Process only the first N users (fast testing)
uv run python -m analysis.pipeline --force            # Re-segment users already processed
uv run python -m analysis.pipeline --relabel          # Re-classify existing segments without re-segmenting
```

### Benchmark mode

Run the Analyzer across multiple LLMs for side-by-side comparison.
Configure models in `analysis/config/benchmark.toml`.

```bash
uv run python -m analysis.pipeline --benchmark
```

### Offline quality layer

Periodic embedding + HDBSCAN pass — not part of the regular pipeline.
Writes `clusterConfidence` back to segments and downgrades low-confidence auto-accepted segments.

```bash
uv run python -m analysis.pipeline --offline-quality
```

### Set up MongoDB indexes only (required once, safe to repeat)

```bash
uv run python -c "from db.client import setup_indexes, close_client; setup_indexes(); close_client()"
```

---

## Configuration

All tunable parameters live in **TOML/YAML config files** under `analysis/config/`.
Edit these files directly — no code changes needed.

### `analysis/config/analyzer.toml`

Controls the Analyzer pipeline (LLM model, segmentation, routing, collections).

| Section | Key | Default | Description |
|---|---|---|---|
| `[analyzer]` | `model` | `"openai/gpt-5.4-mini"` | LLM model via Vercel AI Gateway (`provider/model` format) |
| `[analyzer]` | `prompt_version` | `"v1"` | Prompt template version (`"v1"`, `"v2"`, or `"latest"`) |
| `[analyzer]` | `temperature` | `0.0` | LLM temperature |
| `[analyzer]` | `base_url` | `"https://ai-gateway.vercel.sh/v1"` | LLM endpoint — empty string = local Ollama |
| `[analyzer]` | `drop_existing` | `false` | Drop unreviewed `sms_chat_segments` docs before re-running |
| `[analyzer]` | `drop_reviewed` | `false` | Also drop human-reviewed segments — **DESTRUCTIVE** |
| `[analyzer]` | `force_rerun` | `false` | Re-segment users already processed (without global clear) |
| `[analyzer]` | `concurrency` | `4` | Parallel LLM calls |
| `[analyzer]` | `benchmark_mode` | `false` | When true, writes `analyzerMeta` to each segment for LLM comparison |
| `[analyzer]` | `window_size` | `200` | Max messages per LLM call; `0` = no windowing |
| `[routing]` | `cluster_confidence_threshold` | `0.6` | Confidence threshold for HDBSCAN cluster membership (offline quality) |
| `[pipeline]` | `user_limit` | `0` | Max users to process; `0` = all |
| `[pipeline]` | `relabel` | `false` | Re-classify existing segments without re-segmenting |
| `[collections]` | `input_*` | PROD names | Input collection names (`users`, `sms_chats`, `sms_chat_messages`, `matchings`) |
| `[collections]` | `output_*` | local names | Output collection names (`sms_chat_segments`, `matching_signals`) |

### `analysis/config/benchmark.toml`

Controls `--benchmark` mode — each `[[run]]` entry defines one LLM comparison pass.

| Key | Default | Description |
|---|---|---|
| `user_limit` | `2` | Users per benchmark run; `0` = all |
| `prompt_version` | `"v2"` | Global prompt version for all runs |
| `[[run]].run_id` | — | Unique identifier for this benchmark run |
| `[[run]].model` | — | LLM model (`provider/model` format) |
| `[[run]].temperature` | `0.0` | LLM temperature |
| `[[run]].base_url` | — | LLM endpoint URL |

### `analysis/config/clustering.toml`

Controls the offline quality layer (`--offline-quality`): embedding + UMAP + HDBSCAN.

| Section | Key | Default | Description |
|---|---|---|---|
| `[embedder]` | `model` | `"all-MiniLM-L6-v2"` | Sentence-transformer embedding model |
| `[embedder]` | `cache_dir` | `".cache/embeddings"` | Cached embeddings directory |
| `[embedder]` | `batch_size` | `64` | Segments per embedding batch |
| `[clusterer]` | `umap_n_components` | `10` | UMAP target dimensions |
| `[clusterer]` | `umap_n_neighbors` | `15` | UMAP neighbor count |
| `[clusterer]` | `umap_min_dist` | `0.05` | UMAP minimum distance |
| `[clusterer]` | `hdbscan_min_cluster_size` | `50` | Minimum segments to form a cluster |
| `[clusterer]` | `hdbscan_min_samples` | `3` | HDBSCAN min samples |

### `analysis/config/taxonomy.yaml`

Defines the canonical topic taxonomy — slug, description, and confirmed subtopics.
The pipeline reads this at startup to drive `isNewTopic` / `isNewSubTopic` flags.
Add confirmed subtopics here after human review.

---

## Project Structure

```
ufl/
├── pyproject.toml                        # uv dependencies and tool config
├── .env.example                          # Environment variable template
│
├── config/
│   └── settings.py                       # Pydantic BaseSettings — loads .env (DB URIs, API keys)
│
├── analysis/                             # Analyzer pipeline
│   ├── pipeline.py                       # Entry point: uv run python -m analysis.pipeline [--step X] [--limit N]
│   ├── config/
│   │   ├── loader.py                     # Reads *.toml + taxonomy.yaml → typed dataclasses
│   │   ├── analyzer.toml                 # ← EDIT: LLM model, concurrency, routing, collections
│   │   ├── benchmark.toml                # ← EDIT: multi-LLM comparison runs
│   │   ├── clustering.toml               # ← EDIT: offline quality (embedder, UMAP, HDBSCAN)
│   │   └── taxonomy.yaml                 # ← EDIT: canonical topic/subtopic pairs
│   ├── sampling/
│   │   └── signal_extractor.py           # --step signal: TP/FP from PROD matchings → matching_signals
│   ├── segmentation/
│   │   └── segmenter.py                  # --step segment: LLM Analyzer (segment + classify in one call)
│   ├── graders/
│   │   └── segmentation_graders.py       # --step validate: G0.3, G1.2, G1.4 deterministic checks
│   ├── evaluation/
│   │   ├── evaluator.py                  # --step evaluate: quality metrics
│   │   └── reporter.py                   # Timestamped run summary reports (JSON + Markdown)
│   ├── quality/
│   │   └── offline.py                    # --offline-quality: embed → UMAP → HDBSCAN → clusterConfidence
│   └── prompts/
│       └── analyzer/
│           ├── __init__.py               # PromptTemplate loader (parses versioned .md files)
│           ├── analyzer_prompt_v1.md      # v1: detailed rules + event keywords + Gen Z context
│           └── analyzer_prompt_v2.md      # v2 (current): leaner, rules in system section
│
├── db/
│   ├── client.py                         # get_db() (local output), get_input_db() (PROD input), setup_indexes()
│   ├── migrations/
│   │   ├── chat_segment_schema_v2.py     # Legacy: field renames + backfill chatEndedAt
│   │   └── merge_chat_sessions.py        # Legacy: merge multi-session chats into 1 per user
│   └── README.md
│
├── tests/
│   └── test_analyzer_v2.py               # Unit tests for deterministic functions (no LLM, no DB)
│
├── docs/
│   ├── erd.md                            # PROD schema ERD (Mermaid) + field reference
│   ├── GRADERS.md                        # 30 graders across 5 levels (L0–L4)
│   └── architecture/                     # Phase flowcharts (Eraser.io)
│
└── reports/                              # Timestamped run summary outputs (JSON + Markdown)
```

---

## Reference

- [Install MongoDB on macOS](https://www.mongodb.com/docs/v7.0/tutorial/install-mongodb-on-os-x/)
- [uv documentation](https://docs.astral.sh/uv/)
