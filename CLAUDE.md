# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A demo project showing how to build a Claude-powered data analyst agent using the Managed Agents (beta) API, with optional Slack integration. The agent runs in a cloud environment, analyzes CSV data, and produces interactive HTML reports with Plotly charts.

## Running the Project

This is a Jupyter notebook project with no traditional build/test infrastructure. Run notebooks cell-by-cell or via:

```bash
jupyter notebook data_analyst_agent.ipynb   # Step 1: create agent + environment
jupyter notebook slack_data_bot.ipynb        # Step 2 (optional): Slack bot
```

**Required env vars** (stored in `.env` automatically after first run):
- `ANTHROPIC_API_KEY` — set before running
- `ANALYST_ENV_ID`, `ANALYST_AGENT_ID`, `ANALYST_AGENT_VERSION` — written by `data_analyst_agent.ipynb`
- `SLACK_BOT_TOKEN`, `SLACK_APP_TOKEN` — prompted by `slack_data_bot.ipynb`

Install dependencies:
```bash
pip install "anthropic>=0.91.0" python-dotenv slack_bolt requests markdown-to-mrkdwn pandas plotly
```

## Architecture

### Two-notebook design

**`data_analyst_agent.ipynb`** — provisions everything and runs a one-shot analysis:
1. Creates a named cloud environment (networking unrestricted, packages: pandas + plotly, tools: `agent_toolset_20260401` minus web_search/web_fetch)
2. Creates a versioned Claude agent with a system prompt that instructs professional data analysis, validation, and HTML report generation
3. Uploads a CSV to the Anthropic Files API and mounts it at `/mnt/session/uploads/`
4. Runs the agent session, streams events, and downloads `report.html` from `/mnt/session/outputs/`
5. Saves environment/agent IDs to `.env` for reuse

**`slack_data_bot.ipynb`** — long-running Slack Socket Mode bot:
- Triggers on `@mentions` in Slack
- Maps each Slack thread to a persistent agent session (session-per-thread pattern)
- Downloads file attachments from Slack → re-uploads to Anthropic Files API → mounts in session
- Streams agent events back to Slack in real time
- Loads agent/environment IDs from `.env` (no reprovisioning)

### Key design patterns

- **Agent/environment reuse**: IDs persisted in `.env` so neither is re-created on subsequent runs; only new sessions are created per conversation
- **Session-per-thread**: `thread_sessions` dict maps `thread_ts` → `session_id`; allows multi-turn follow-up in the same Slack thread
- **Streaming relay**: Agent events (tool calls, text deltas, errors) are streamed and translated to Slack `mrkdwn` format in real time

### Model

Uses `claude-sonnet-4-6`. The agent system prompt is the primary place to tune analysis behavior (validation rules, chart preferences, report structure).

### Sample data

`sample_data/fifa_wc_data.csv` — 48 teams × 24 columns (squad value, FIFA ranking, historical tournament performance) for FIFA World Cup 2026 predictions. Use this file to test the agent end-to-end.
