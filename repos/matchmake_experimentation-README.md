# LLM Matchmaker Simulation

An experiment to predict dating app accept/reject decisions using LLM personas. The project includes two main systems:

1. **Match Prediction Simulation**: Each user profile becomes an AI agent that evaluates potential matches based on personality, preferences, and writing style
2. **Photo Analysis Pipeline**: A 3-agent system that deeply analyzes profile photos for attractiveness, vibe/aesthetic, and lifestyle signals

> **Note**: This project uses [OpenRouter](https://openrouter.ai/), which provides unified API access to hundreds of AI models including GPT-5-mini, GPT-4o, Claude, Gemini, and more. OpenRouter offers better pricing and model flexibility compared to using OpenAI directly.

## 🆕 MCP Server Available!

You can now run the photo analysis pipeline through a **Model Context Protocol (MCP) server**, allowing access to the tools/server via Cursor. 


## Overview

### Match Prediction System

This system:
1. Reads match pairs from `out.json` (which users were matched)
2. Looks up detailed profile data from `profiles.json` 
3. Simulates each user's decision using an LLM persona
4. Compares predictions to actual decisions from `out.json`
5. Reports accuracy metrics

### Photo Analysis Pipeline

A specialized 3-agent system that analyzes profile photos:
1. **Attractiveness Agent**: Evaluates physical attractiveness, facial features, body composition, grooming, and dating market positioning
2. **Vibe & Aesthetic Agent**: Identifies archetypes (e.g., "Girl Next Door", "Gym Bro"), fashion brands, cultural signaling, and aesthetic cohesion
3. **Lifestyle Agent**: Detects socioeconomic signals, social capital, hobbies, and lifestyle patterns from visual cues

## Setup

### Prerequisites

- Bun runtime (or Node.js)
- OpenRouter API key ([Get one here](https://openrouter.ai/))

### Installation

```bash
# Install dependencies
bun install

### Configuration

Create a `.env` file with your OpenRouter API key:

```bash
# Single API key for all models
OPENROUTER_API_KEY=your_api_key_here

# Model configuration (OpenRouter format: provider/model-name)
TEXT_MODEL=openai/gpt-5-mini              # For match prediction simulation
VISION_MODEL=google/gemini-3-pro-preview  # For photo analysis pipeline

# Match Prediction - Scoring weights (must sum to 1.0, default values shown)
WEIGHT_PHYSICAL=0.50      # Physical attraction (50%)
WEIGHT_LIFESTYLE=0.30     # Lifestyle compatibility (30%)
WEIGHT_PERSONALITY=0.15   # Personality match (15%)
WEIGHT_INTENTIONS=0.05    # Dating intentions alignment (5%)

# Match Prediction - Sample configuration
SAMPLE_SIZE=10            # Number of matches to test (default: 10)
SAMPLE_OFFSET=0           # Starting index in matches array (default: 0)

# Photo Analysis - Test configuration
TEST_COUNT=5              # Number of matches to analyze (default: 5)
TEST_OFFSET=0             # Starting index for photo analysis (default: 0)

# MCP Server - Data paths (optional)
DATA_DIR=.                # Directory containing data files (default: current dir)
PROFILES_PATH=./profiles.json  # Path to profiles.json
MATCHES_PATH=./out.json        # Path to out.json
```

## Usage

### Run Match Prediction Simulation

```bash
bun index.ts
```

This will:
- Load profiles and matches
- Simulate decisions for the specified sample size
- Display results in the console
- Save detailed results to `simulation_results.json`

### Run Photo Analysis Pipeline

```bash
bun testAnalysis.ts
```

This will:
- Load profiles and matches
- Extract unique profiles from match pairs
- Run 3-agent analysis on each profile (Attractiveness, Vibe & Aesthetic, Lifestyle)
- Display detailed analysis for each profile
- Save results to `analysis_test_results.json`

### Run via MCP Server

The MCP server provides 4 tools accessible from AI assistants:
- `analyze_profile`: Analyze a single profile by userId
- `run_batch_analysis`: Batch analyze multiple profiles from matches
- `get_profile_info`: Get profile information without running analysis
- `list_matches`: List match pairs with filtering


## Project Structure

```
.
├── types.ts                      # TypeScript interfaces for profiles and matches
├── schemas.ts                    # Zod schemas for photo analysis validation
├── prompts.ts                    # LLM prompts for both systems
│
├── index.ts                      # Match prediction simulation engine
├── results.ts                    # Metrics calculation and reporting
├── validate.ts                   # Data validation script (no API calls)
│
├── analyzer.ts                   # Photo analysis pipeline (3 agents)
├── testAnalysis.ts               # Photo analysis test runner
├── imageProcessor.ts             # Image processing utilities
│
├── mcp-server.ts                 # Model Context Protocol server
├── test-mcp.ts                   # MCP server testing script
│
├── profiles.json                 # User profile data (8K+ profiles)
├── out.json                      # Match interactions (3.5K+ matches)
├── simulation_results.json       # Match prediction output (generated)
├── analysis_test_results.json    # Photo analysis output (generated)
│
├── prompts/                      # Detailed analysis prompts
│   ├── attractivenessanalysisprompt.md
│   ├── vibe.md
│   └── lifestyleprompt.md
│
├── README.md                     # This file
├── MCP_SERVER_README.md          # MCP server documentation
├── MCP_QUICK_REFERENCE.md        # MCP quick reference
└── USAGE.txt                     # Usage instructions
```

## How It Works

### Data Flow

1. **Input**: `out.json` provides match pairs and actual decisions
2. **Lookup**: `profiles.json` provides detailed profile data indexed by `userId`
3. **Simulate**: For each match, create two LLM personas (one per user)
4. **Compare**: Compare predicted decisions to actual decisions

### Persona Simulation

Each user becomes an LLM agent with:
- Demographics (age, gender, height, ethnicity)
- Preferences (gender, age range, ethnicity, physical attraction)
- Personality (green flags, red flags, political/religious beliefs)
- Writing style (analyzed from text length and tone)

### Multi-Dimensional Scoring

The system evaluates 4 compatibility dimensions:

1. **Physical Attraction** (50% weight): Visual appeal, style, photos
2. **Lifestyle Compatibility** (30% weight): Hobbies, activities, interests
3. **Personality Match** (15% weight): Vibe, energy, communication style
4. **Intention Alignment** (5% weight): Dating goals (casual vs serious)

Each dimension scored 0.0-1.0, then combined via weighted average.

### Dual-Track Evaluation

**Text Track**: Text-only LLM reads profile descriptions and scores all 4 dimensions based on written content.

**Vision Track**: Vision-capable LLM (Gemini) analyzes actual profile photos and scores dimensions based on visual cues.

**Final Score**: Average of text and vision scores for each dimension, then weighted sum (≥0.5 = ACCEPT).

### Hard Filters

Before LLM evaluation, the system checks:
- Gender preferences (`expectedGender`)
- Age range preferences (`ageRange`)
- Ethnicity preferences (`expectedEthnicity`)

If hard filters fail → instant reject (saves API costs).

## Key Features

- **Multi-dimensional scoring**: 4 compatibility categories (Physical, Lifestyle, Personality, Intentions)
- **Dual-track evaluation**: Text-only LLM + Vision LLM analyze each candidate independently
- **Vision analysis**: Gemini evaluates actual profile photos, not just text descriptions
- **Persona-based prediction**: LLM adopts user's personality/preferences
- **Weighted aggregation**: Configurable weights for each compatibility dimension
- **Continuous scoring (0-1)**: Nuanced confidence scores for each category
- **Dual reasoning display**: See both text-based reasoning and visual analysis
- **Category breakdown**: Understand exactly why matches succeed or fail
- **Writing style analysis**: Matches casual vs serious users
- **Score separation metrics**: Measures model's ability to distinguish accept/reject
- **UserId debugging**: Display actual userIds instead of redacted names
- **Comprehensive metrics**: Accuracy, confusion matrix, acceptance rates, per-category analysis

## Tuning

To improve accuracy:

1. **Adjust scoring weights** in `.env`:
   - Increase `WEIGHT_PHYSICAL` if physical attraction matters more
   - Increase `WEIGHT_LIFESTYLE` if hobby alignment is key
   - Weights must sum to 1.0

2. **Change models** in `.env`:
   - **Text models**: `openai/gpt-5-mini`, `anthropic/claude-3.5-sonnet`
   - **Vision models**: `google/gemini-pro-1.5`, `google/gemini-flash-1.5`, `openai/gpt-4o`
   - See [OpenRouter models](https://openrouter.ai/models) for full list

3. **Adjust prompts** in `prompts.ts`:
   - Make personas more picky/lenient
   - Emphasize different evaluation criteria

4. **Modify evaluation strategy** in `index.ts`:
   - Change how text/vision scores are merged (currently simple average)
   - Add minimum thresholds for specific categories

