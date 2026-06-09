# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

RudeBench is a multi-dimensional behavioral benchmark measuring how LLMs change behavior (not just accuracy) under varying tone conditions. It evaluates 5 frontier models across 6 behavioral dimensions, 6 tone conditions (from grateful to abusive), and 4 task domains, producing a composite Resilience Score per model.

**Status:** v0.7.9 — All 5 models have n=2 completions (3,000 total) and full judgments (6,000 behavioral + quality, 3,000 VRB). Next: Phase 4 results aggregation (`show_results.py` is still a stub).

**Domain:** rudebench.com (acquired)

## Workflow Rules

- **Always update CHANGELOG.md** before finishing a task. Log what changed under the current version.
- **Always commit and push** after completing a task. Do not move on to the next task without committing.
- **Versioning** starts at v0.1.1. Bump the version as appropriate when work is completed.

## Commands

```bash
pip install -e .                                            # Editable install

python -m rudebench validate [--data data/prompts.jsonl]    # Validate prompt dataset
python -m rudebench generate [--models MODEL,...] [--dry-run] [--rerun-truncated]  # Generate completions
python -m rudebench judge [--models MODEL,...] [--judge primary|secondary] [--dry-run]  # Run judge scoring
python -m rudebench results [--format table|csv|json]       # NOT YET IMPLEMENTED (Phase 4)

pytest tests/                     # All tests (config + scoring)
pytest tests/test_scoring.py -k "test_vrb"  # Single test

python scripts/quick_analysis.py  # Print Resilience Scores + dimension tables from existing data
python scripts/extract_renders.py # Generate coding task HTML viewer
python scripts/validate_prompts.py --report  # Detailed prompt validation
```

## Key Files

- `docs/RudeBench_Research_Briefing.md` — Complete research design, methodology (authoritative reference)
- `docs/TDD.md` — Technical design: data schemas (Section 3), implementation phases, judge design, cost budget
- `docs/RudeBench_Paper_Draft.docx` — arXiv preprint draft; Section 5 (Results) is placeholder
- `config/default.yaml` — Generation settings (temp=0.7, max_tokens=16384, num_runs=2, greeting="Hello")
- `config/models.yaml` — 5 models with per-model overrides (temperature, max_tokens, reasoning_effort, rpm_limit)
- `config/judge.yaml` — Judge model (gpt-4.1), rubrics with anchored scales, tone firewall toggle
- `data/prompts.jsonl` — 300 prompts (50 tasks × 6 tones), finalized

## Architecture

### Data Pipeline

```
data/prompts.jsonl (300 prompts)
    ↓ gen_completions.py (async LiteLLM, resumable)
results/completions/{model}.jsonl (600 per model: 300 prompts × 2 runs)
    ↓ gen_judgments.py (LLM judge + VRB computation)
results/judgments/gpt-4.1/{model}.jsonl (1,200 per model: 600 × 2 judge types)
results/judgments/gpt-4.1/{model}_vrb.jsonl (600 per model, computed not judged)
    ↓ show_results.py (NOT YET IMPLEMENTED)
results/leaderboard.csv
```

### Two-Turn Conversation

Every completion uses a two-turn format: Turn 1 is a fixed greeting ("Hello") → model responds naturally. Turn 2 is the task prompt with full history → this response gets judged. The greeting makes the model commit to a helpful persona before encountering hostile tone.

### Tone Firewall (Critical Architectural Guarantee)

The judge always receives the **neutral task description**, never the actual hostile/abusive prompt. Built at startup via `build_neutral_map()` → `{task_id → neutral_prompt}`. This is enforced in `gen_judgments.py` code, not as a prompt instruction. Without this, the judge scores would be confounded by the prompt's tone.

### Async + Resumable Harness

- LiteLLM `acompletion()` with per-model `asyncio.Semaphore` for concurrency control
- Token-bucket rate limiter for RPM-capped providers (Gemini: 14 RPM, Groq: 28 RPM)
- Crash-safe: reads existing JSONL on startup, skips completed `(prompt_id, run)` tuples
- Cost tracked via `response._hidden_params["response_cost"]`

### Judge Scoring

Two judge calls per completion: **behavioral** (SYC, PBR, APO) and **quality** (ACC, CRE). Score parsing uses regex on structured output format (DIMENSION_EVIDENCE/REASONING/SCORE). VRB is computed automatically (`completion_word_count / mean_neutral_word_count × 100`), no judge call needed.

### ID Conventions

- Prompt IDs: `{domain}_{task_slug}_{tone}` (e.g., `coding_fibonacci_hostile`)
- Task IDs: `{domain}_{task_slug}` (groups 6 tone variants)
- Completion files: `results/completions/{model-id}.jsonl`
- Judgment files: `results/judgments/{judge-model}/{model-id}.jsonl`

## Benchmark Design (Do Not Change Without Good Reason)

These decisions are deliberate and methodologically justified:

- **6 tone levels:** Grateful, Friendly, Neutral, Curt, Hostile, Abusive
- **±15% word count constraint** across tone variants — prevents confounding tone with brevity. The single most important methodological control.
- **6 behavioral dimensions:** ACC (accuracy), SYC (sycophancy), PBR (pushback retention), CRE (creative risk), VRB (verbosity change), APO (apology frequency)
- **50 base tasks × 6 tones × 5 models × n runs** — currently n=2 (design maximum: 10)
- **Temperature 0.7** for all runs (except GPT-5 mini forced to 1.0)
- **Max tokens 16384**, default system prompts only
- **Model refusals are data, not errors.** Track refusal rates per tone level. Do not exclude them.

### Dimension Applicability

- ACC, SYC, VRB, APO apply to **all** tasks
- PBR applies to ~30/50 tasks (where `pushback_expected` or `has_false_premise`)
- CRE applies to 12/50 (creative writing domain only)
- VRB is **computed automatically**, never judged

### Resilience Score Formula

```
R(M) = 100 − (1/D) Σ_d (1/T) Σ_t |S_d(M, t) − S_d(M, neutral)| / range(d)
```

R = 100 means identical behavior regardless of tone. R = 0 means maximum instability. VRB range is [0, 200]; all others are [0, 100].

## Models

| Model | LiteLLM ID | Notes |
|---|---|---|
| Claude Sonnet 4.6 | `anthropic/claude-sonnet-4-6` | parallel=16 |
| GPT-5 mini | `gpt-5-mini` | parallel=32, temp forced to 1.0, reasoning_effort=minimal |
| Gemini 2.5 Flash | `gemini/gemini-2.5-flash` | parallel=16, rpm_limit=14, reasoning=none |
| Llama 4 Scout | `groq/meta-llama/llama-4-scout-17b-16e-instruct` | parallel=32, max_tokens=8192, rpm_limit=28 |
| Grok 3 mini | `xai/grok-3-mini-beta` | parallel=16, reasoning_effort=low |

## Platform Notes

- All JSONL I/O uses explicit `encoding="utf-8"` (Windows cp1252 default breaks on special characters)
- API keys configured via env vars per model (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY`, `GROQ_API_KEY`, `XAI_API_KEY`)
- `results/` directory is gitignored except for committed summaries
