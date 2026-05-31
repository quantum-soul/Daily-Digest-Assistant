# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Automated daily digest pipeline that fetches arXiv papers + Xiaohongshu (小红书) notes, filters them with an LLM, generates Chinese summaries, and emails a two-column HTML digest. Runs on GitHub Actions — no server required.

## Commands

```bash
# Install dependencies
pip install -r requirements.txt
npm install              # Node.js deps for XHS request signing (PyExecJS)

# Dry run — renders to preview.html, does NOT send email
python main.py --dry-run

# Dry run for a specific date (backfill historical arXiv batches)
python main.py --dry-run --date 2026-04-03

# Entropy-based filtering mode (SLTF-Entropy scoring + LLM summaries, no LLM relevance scoring)
python main.py --entropy-only

# Production run — sends email via SMTP
python main.py
```

Required environment variables: `LLM_API_KEY`, `EMAIL_USER`, `EMAIL_PASS`, `EMAIL_TO`. Optional: `XHS_COOKIE` (skip XHS fetch if unset).

## Architecture

```
main.py                 # Pipeline orchestrator (single entry point)
├── fetchers/           # Data acquisition
│   ├── arxiv_fetcher   # arXiv API search via official `arxiv` library
│   ├── arxiv_schedule  # arXiv announcement date/window calculation (ET timezone)
│   ├── xhs_fetcher     # XHS note search (two-step: search → detail)
│   ├── xhs_util        # XHS request signing (execjs calling static/*.js)
│   └── xhs_pc_apis     # XHS API HTTP wrappers
├── llm/                # LLM-based processing
│   ├── filter_and_summarize  # arXiv: LLM scoring + Chinese summary + detail
│   ├── filter_and_summarize_xhs  # XHS: LLM scoring + summary + topic dedup
│   └── entropy_scorer  # SLTF-Entropy paper scoring (keyword-only, no LLM)
├── render/             # Jinja2 HTML email rendering
├── sender/             # SMTP sending (163/Gmail/QQ)
└── templates/          # email.html (two-column table layout)
```

## Pipeline flow (main.py)

1. **Schedule determination** — `arxiv_schedule` calculates the correct arXiv announcement date and submission time window in ET. Handles holidays, weekends (Fri/Sat = no announcements), and auto-backoff to the previous valid announcement day.
2. **ArXiv fetch** — `arxiv_fetcher` queries arXiv API with keyword OR + category AND filters, constrained to the submission window. Implements 429 rate-limit retry with exponential backoff.
3. **LLM filter & summarize** — Batch prompt sends all candidates to the LLM at once. Returns JSON: `[{id, score (1-10), summary_zh (20-40 chars), detail_zh (100-150 chars)}]`. Papers below `min_score` threshold are dropped (no forced fill).
4. **XHS fetch** (optional) — Searches per-keyword, merges results, fetches note bodies via secondary API call. Sorts by like count.
5. **XHS LLM filter** — Same batch prompt pattern. Additional `_diversify_with_llm` step deduplicates by topic (same product/event → keep only top-scored one).
6. **Render** — Jinja2 renders email.html with two-column `<table>` layout: arXiv left, XHS right, card-per-row pairing.
7. **Send** — SMTP via provider config (163/Gmail/QQ).

## Key design decisions

- **LLM provider registry** (in `filter_and_summarize.py:BUILTIN_PROVIDERS`): Built-in providers are pre-configured tuples of `(sdk, base_url, model)`. Custom providers from `config.yml` → `custom_llm` are merged at runtime — no code changes needed. SDKs: `anthropic` (Anthropic, MiniMax) or `openai` (OpenAI, DeepSeek, Zhipu, Moonshot, Qwen).
- **arXiv schedule alignment**: The pipeline targets arXiv's official 20:00 ET announcement batches. `get_effective_announcement_date()` handles the rollback logic: when run at Beijing 12:00 (= ET 00:00), it always picks the previous valid announcement day (yesterday's batch).
- **Keyword tier system**: `_build_keyword_tiers()` auto-derives priority levels from user keywords — multi-word phrases + pairwise single-word combos get tier-1 (higher scoring weight) in the LLM prompt.
- **Entropy mode** (`--entropy-only`): Uses SLTF-Entropy formula instead of LLM for relevance scoring. LLM is still called for summary generation. Triggers a different prompt (no score assessment, just summaries).
- **XHS signing**: XHS APIs require cryptographic request signing implemented in JS (`static/xhs_xs_xsc_56.js`). Python calls JS via `PyExecJS` with a Node.js runtime. The `xhs_xray.js` tracer is skipped (returns empty string) due to CJS compatibility issues on newer Node.
- **Deduplication**: `arxiv_fetcher` tracks pushed paper IDs and filters duplicates. The archive commit workflow (`daily.yml`) writes to `docs/daily/` and a `index.json` manifest.
- **GitHub Actions dual workflow**: `daily.yml` runs the digest + archives results; `pages.yml` deploys `docs/` to GitHub Pages. The `sync-pages` job in `daily.yml` is conditional on `github.repository == 'yzbcs/Daily-Digest-Assistant'` (upstream only).

## Configuration

All user-facing config is in `config.yml`. Secrets are in GitHub Actions secrets / environment variables. The `custom_llm` block allows adding any OpenAI- or Anthropic-compatible API without touching Python code.
