# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Official starter kit for the KDD Cup 2026 DataAgent-Bench challenge. Implements a ReAct-style baseline agent that reads tasks from `data/public/input/` and writes predictions for evaluation.

## Common Commands

```bash
# Install dependencies
uv sync

# Check dataset status
uv run dabench status --config configs/react_baseline.example.yaml

# Inspect a single task
uv run dabench inspect-task task_1 --config configs/react_baseline.local.yaml

# Run baseline on one task
uv run dabench run-task task_1 --config configs/react_baseline.local.yaml

# Run baseline on all tasks (with optional limit)
uv run dabench run-benchmark --config configs/react_baseline.local.yaml --limit 10

# Run tests
uv run pytest

# Lint code
uv run ruff check src/
```

## Architecture

**Entry point**: `src/data_agent_baseline/cli.py` — Typer CLI with commands: `status`, `inspect-task`, `run-task`, `run-benchmark`.

**ReAct Agent Loop** (`src/data_agent_baseline/agents/react.py`):
- System prompt + task question → model → JSON action response
- Parse `{thought, action, action_input}` from fenced JSON block
- Execute tool → observation → append to conversation → repeat
- Terminates when `answer` tool is called or `max_steps` reached

**Tool Registry** (`src/data_agent_baseline/tools/registry.py`):
- `list_context`, `read_csv`, `read_json`, `read_doc` — file inspection
- `inspect_sqlite_schema`, `execute_context_sql` — SQLite queries
- `execute_python` — arbitrary Python execution in task context (30s timeout)
- `answer` — terminal action that submits final table

**Model Adapter** (`src/data_agent_baseline/agents/model.py`):
- `OpenAIModelAdapter` — uses OpenAI-compatible API
- `ScriptedModelAdapter` — for testing with pre-scripted responses

**Runner** (`src/data_agent_baseline/run/runner.py`):
- `run_single_task` — executes one task with timeout via subprocess
- `run_benchmark` — parallel execution with `max_workers` threads
- Outputs: `artifacts/runs/<run_id>/<task_id>/trace.json` + `prediction.csv`

## Configuration

YAML config file (`configs/*.yaml`) structure:
- `dataset.root_path` — path to task input directory
- `agent.model`, `agent.api_base`, `agent.api_key` — OpenAI-compatible model settings
- `agent.max_steps` — maximum ReAct steps per task
- `run.max_workers` — parallel worker count
- `run.task_timeout_seconds` — wall-clock timeout per task (0 disables)

## Data Flow

Each task directory (`data/public/input/task_<id>/`) contains:
- `task.json` — `{task_id, difficulty, question}`
- `context/` — data files (CSV, JSON, SQLite, text)

Agent outputs:
- `trace.json` — full ReAct step history with observations
- `prediction.csv` — final answer table (columns + rows)

## Key Files to Modify

- `src/data_agent_baseline/agents/prompt.py` — system prompt and response examples
- `src/data_agent_baseline/agents/react.py` — ReAct loop logic and parsing
- `src/data_agent_baseline/tools/registry.py` — add new tools
- `src/data_agent_baseline/agents/model.py` — model adapter protocol