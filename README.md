---
title: AI Automation Stack
emoji: 🤖
colorFrom: blue
colorTo: purple
sdk: docker
app_port: 7860
---

# AI Automation Stack

A self-contained automation platform combining **n8n** (workflow engine), **Claude Code** (AI coding agent), **Ollama + Qwen** (local LLMs), **FastAPI** (agent API), **MCP** (automation control), and **Supabase Postgres** (external persistence).

---

## Services

| Service | Port | Description |
|---|---|---|
| n8n | 7860 | Workflow engine — main UI and REST API |
| FastAPI Agent | 8000 | AI agent REST API |
| MCP Server | stdio | Automation control layer for Claude Code |
| Ollama | 11434 | Local LLM runtime (internal only) |

---

## Quick Start

### 1. Required HF Space Secrets

Go to your Space → **Settings → Variables and Secrets** and add:

| Secret | Example | Required |
|---|---|---|
| `N8N_OWNER_EMAIL` | `you@example.com` | Yes |
| `N8N_OWNER_PASSWORD` | `StrongPass123!` | Yes |
| `DATABASE_URL` | `postgresql://postgres.xxx:PASS@aws-0-us-east-1.pooler.supabase.com:6543/postgres` | Recommended |
| `N8N_API_KEY` | *(leave empty on first boot — auto-generated)* | Optional |
| `OLLAMA_MODEL` | `qwen3:1.7b` | Optional |
| `OLLAMA_CODER_MODEL` | `qwen2.5-coder:1.5b` | Optional |

> **First boot without `N8N_API_KEY`:** the container auto-generates a key, logs it to `/var/log/n8n.log`, and writes it to `/root/.n8n/.env`. Copy it and set it as a Secret so it survives restarts.

### 2. Supabase Postgres (recommended)

Without a `DATABASE_URL`, n8n uses SQLite and **workflows reset on every restart**.

1. Create a free project at [supabase.com](https://supabase.com)
2. Go to **Project → Settings → Database → Connection Pooler** (Transaction mode, port 6543)
3. Copy the **URI** and set it as the `DATABASE_URL` secret

### 3. First boot timeline

| Time | What's happening |
|---|---|
| 0–30s | n8n starts, port 7860 becomes live |
| 30–60s | FastAPI agent and MCP server start |
| 60–180s | Ollama pulls `qwen3:1.7b` (~1.1 GB) and `qwen2.5-coder:1.5b` (~0.9 GB) in background |
| After model pull | Qwen Local node becomes usable in workflows |

The UI is accessible as soon as port 7860 is live — you don't need to wait for models.

---

## API Endpoints (FastAPI Agent — port 8000)

| Method | Path | Description |
|---|---|---|
| `GET` | `/` | Status and service URLs |
| `GET` | `/healthz` | Liveness probe |
| `POST` | `/agent` | Run prompt through Ollama chat model |
| `POST` | `/claude` | Run prompt through Claude Code CLI |
| `POST` | `/workflow/create` | Create n8n workflow from explicit JSON |
| `POST` | `/workflow/from-prompt` | Generate and save workflow from plain English |
| `GET` | `/workflow/list` | List all n8n workflows |
| `POST` | `/hf` | Run a `huggingface-cli` command |

Example — create a workflow from a prompt:
```bash
curl -X POST http://your-space.hf.space:8000/workflow/from-prompt \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Fetch JSON from an API every hour and log the result"}'
```

---

## MCP Tools (for Claude Code)

Claude Code connects via `.claude/mcp.json` (already configured). Available tools:

| Tool | Description |
|---|---|
| `list_workflows` | List all workflows in n8n |
| `read_workflow(workflow_id)` | Get full JSON of a workflow |
| `create_workflow(name, nodes, connections)` | Create a validated workflow |
| `delete_workflow(workflow_id)` | Delete a workflow |
| `create_workflow_from_prompt(prompt)` | Generate and save a workflow from plain English |
| `get_n8n_info` | Check n8n health and API key status |

---

## Custom Nodes

### Qwen Local (`qwenLocal`)

Runs local Qwen models via Ollama directly inside an n8n workflow.

**Properties:**
- **Prompt** — text to send (supports n8n expressions like `{{ $json.text }}`)
- **Model** — `qwen3:1.7b` (chat), `qwen2.5-coder:1.5b` (coding), or Custom
- **Custom Model Name** — any Ollama model name (if you pulled additional models)

**Output fields:** `response`, `model`, `prompt`

---

## Models

| Model | Size | Used for |
|---|---|---|
| `qwen3:1.7b` | ~1.1 GB | Chat, summarize, classify — `/agent` endpoint and Qwen Local node |
| `qwen2.5-coder:1.5b` | ~0.9 GB | Workflow JSON generation — `/workflow/from-prompt` and MCP |

---

## Troubleshooting

**n8n shows "license expired" or "license invalid"**
This is fixed in this build. `N8N_LICENSE_RENEW_ON_INIT=true` forces renewal on every restart. If it persists, check that the container has outbound internet access to `license.n8n.io`.

**Qwen Local node not found in n8n**
The custom node loads on n8n startup. If it's missing, check `/var/log/n8n.log` for errors. Make sure `N8N_CUSTOM_EXTENSIONS=/root/.n8n/custom-nodes` is set (already in Dockerfile).

**Workflows not persisting after restart**
Set `DATABASE_URL` to a Supabase Postgres URL. SQLite is ephemeral on HF Spaces.

**API calls return 401 Unauthorized**
The `N8N_API_KEY` is empty or wrong. Check `/root/.n8n/.env` inside the container for the auto-generated key, or set it manually in HF Space Secrets.

**Ollama models not loading**
Check `/var/log/ollama.log`. Models are pulled in the background after startup — the first request to the Qwen Local node may time out if models are still downloading. Wait ~3 minutes after first boot.

---

## File Structure

```
/
├── agent/
│   ├── server.py            ← FastAPI agent server
│   └── mcp_n8n_server.py    ← MCP server (Claude Code integration)
├── tools/
│   ├── n8n_utils.py         ← n8n REST API helpers + workflow generation
│   └── create_workflow.py   ← CLI tool for creating workflows from JSON files
├── skills/
│   └── workflow_builder.md  ← Claude Code skill: build n8n workflows from prompts
├── root/.n8n/
│   ├── custom-nodes/
│   │   └── qwen-node/       ← Qwen Local n8n node package
│   └── workflows/           ← n8n workflow storage (SQLite mode)
└── start.sh                 ← Container entrypoint
```
