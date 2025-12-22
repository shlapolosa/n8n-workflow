# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Note

Always start by reading; prd.md and use the docker-compose.yml as a guide to reference what services will be deployed. claude cli will be deployed on the same server as ollama for the external agents.

## Project Overview

This is a **Chat-to-Workflow Routing System** that connects OpenWebUI chat interfaces to n8n automation workflows with external worker execution. The architecture enables:

- Natural language requests through OpenWebUI routed to n8n workflows
- Compute-intensive tasks delegated to containerized workers running OpenCode CLI with local Ollama models
- Async job processing with Redis Streams for reliable queuing
- The primary use case is React Native to ReactJS code migration

## Architecture

```
OpenWebUI (Pipe Function) → n8n Webhooks → Redis Streams → OpenCode Worker → GitHub
```

### Key Components

1. **OpenWebUI Pipe Function**: Custom Python function that routes chat requests to n8n, implements async polling
2. **n8n Workflows**: Webhook triggers with 202 async response pattern, orchestrates job flow
3. **Redis Streams**: Job queue with consumer groups for at-least-once delivery (`jobs:{queue}:pending`, `job:{id}:events`)
4. **OpenCode Worker**: Docker container running OpenCode CLI with Ollama for code transformations
5. **Ollama**: Local LLM inference using `qwen2.5-coder:7b` model

### Job Processing Phases

1. **Ingestion**: Clone repo, parse request, validate inputs
2. **Verification**: Lint source, check dependencies
3. **Execution**: Run transformations via OpenCode
4. **Output**: Run tests, commit, push, create PR

## Key Technical Details

### n8n Webhook Response Modes
- `responseNode`: For 202 async patterns - workflow continues after early response
- Use `waitForSubWorkflow: false` for fire-and-forget sub-workflow calls

### Redis Streams (not supported by n8n Redis node)
Use HTTP Request node with Redis REST proxy (e.g., Upstash) for XADD/XREAD commands.

### OpenCode Configuration
- Config file: `opencode.json` (project root or `~/.config/opencode/`)
- Instructions file: `AGENTS.md`
- Skills directory: `.opencode/skills/*/SKILL.md`
- Ollama uses OpenAI-compatible `/v1` endpoint

### OpenWebUI Pipe Function Notes
- Use `emit_status` for progress (works in all modes)
- `emit_message` is broken in Native function calling mode
- Use `httpx.AsyncClient()` for async HTTP requests

## Environment Setup

Copy `.env.example` to `.env` and configure:
- `REDIS_PASSWORD`, `N8N_API_KEY`, `N8N_ENCRYPTION_KEY` (required secrets)
- `GITHUB_TOKEN` with repo permissions
- `N8N_HOST`, `GITHUB_OWNER`, `GITHUB_REPO`

Generate secrets: `openssl rand -base64 32`

## React Native → ReactJS Migration

Key transformations:
- `View` → `div` (add `flex flex-col` - RN defaults to column)
- `TouchableOpacity` → `button`
- `StyleSheet.create()` → Tailwind classes
- React Navigation → React Router DOM v6
- Remove Platform-specific code (`.ios.js`, `.android.js`)

## Ollama Model: qwen2.5-coder:7b

- Context: 32K tokens (configure with `num_ctx`)
- VRAM: 6-8GB (Q4_K_M quantization)
- Set `num_ctx` explicitly (Ollama defaults to 4096)
