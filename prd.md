# Product Requirements Document: Chat-to-Workflow Routing System

A generic, modular architecture connecting OpenWebUI chat interfaces to n8n automation workflows with external worker execution capabilities. This PRD provides complete technical specifications for building an intelligent routing system that processes user requests, executes complex agentic workflows, and delivers results asynchronously.

## Executive summary

This system enables users to submit natural language requests through OpenWebUI that are routed to n8n workflows for orchestration, with compute-intensive tasks delegated to containerized workers running OpenCode CLI with local Ollama models. The architecture supports the specific use case of React Native to ReactJS code migration while remaining generic enough for other agentic workflows. **Key technical foundations** include n8n webhooks with streaming response mode, Redis Streams for reliable job queuing with at-least-once delivery, and OpenWebUI pipe functions for async polling patterns.

---

## Part 1: Generic infrastructure specifications

### 1.1 n8n webhook configuration

The n8n Webhook node provides the entry point for all external requests. Understanding the **responseMode** parameter is critical for async workflow patterns.

#### Response mode options

| Option | Parameter Value | Behavior |
|--------|----------------|----------|
| Immediately | `onReceived` | Returns HTTP 200 with "Workflow got started" instantly |
| When Last Node Finishes | `lastNode` | Waits for entire workflow, returns last node output |
| Using Respond to Webhook Node | `responseNode` | Custom response from Respond to Webhook node anywhere in flow |
| Streaming | `streaming` | Real-time SSE streaming (requires streaming-capable nodes like AI Agent) |

#### HTTP 202 Accepted pattern for async workflows

To implement proper async acknowledgment, use `responseNode` mode with an early Respond to Webhook node:

```json
{
  "nodes": [
    {
      "parameters": {
        "path": "async-workflow",
        "httpMethod": "POST",
        "responseMode": "responseNode",
        "options": {}
      },
      "type": "n8n-nodes-base.webhook",
      "name": "Webhook"
    },
    {
      "parameters": {
        "respondWith": "json",
        "responseBody": "={{ { \"status\": \"accepted\", \"jobId\": $execution.id, \"statusUrl\": \"/webhook/status/\" + $execution.id } }}",
        "options": { "responseCode": 202 }
      },
      "type": "n8n-nodes-base.respondToWebhook",
      "name": "Respond 202 Accepted"
    }
  ]
}
```

The workflow continues executing after the 202 response is sent, enabling true async processing. The client receives the execution ID for status polling or callback correlation.

#### Header authentication configuration

Configure header-based authentication by setting `authentication` to `headerAuth` in the webhook parameters. The actual token validation occurs through n8n's credential system, which stores the expected header name (commonly `X-API-Key` or `Authorization`) and value securely.

```json
{
  "parameters": {
    "authentication": "headerAuth",
    "path": "secure-endpoint",
    "httpMethod": "POST"
  }
}
```

Additional webhook security options include `options.ipWhitelist` for IP filtering and `options.allowedOrigins` for CORS configuration.

### 1.2 Execute Workflow node async execution

The Execute Workflow node (for calling sub-workflows) provides the critical **waitForSubWorkflow** parameter:

- **Parameter name:** `waitForSubWorkflow` (located in Options)
- **Default value:** `true`
- **When OFF:** Main workflow continues immediately without waiting for sub-workflow completion

This enables fire-and-forget patterns where the parent workflow dispatches long-running tasks and proceeds to send responses or perform other operations.

```json
{
  "parameters": {
    "source": "database",
    "workflowId": "{{ $json.targetWorkflowId }}",
    "mode": "once"
  },
  "options": {
    "waitForSubWorkflow": false
  },
  "type": "n8n-nodes-base.executeWorkflow"
}
```

### 1.3 Redis Streams integration

Redis Streams provide the optimal data structure for execution state management, offering persistence, consumer groups, and message acknowledgment that simple pub/sub lacks.

#### Core stream commands

**XADD** - Adding entries with automatic ID generation and optional stream capping:
```redis
XADD execution:{exec_id}:events MAXLEN ~ 1000 * 
  status "started" 
  phase "ingestion" 
  timestamp "1734264000"
```

The `*` generates a timestamp-based ID (`{milliseconds}-{sequence}`), while `MAXLEN ~` provides approximate capping for performance (exact trimming with `=` is slower).

**XREAD** - Reading with optional blocking for real-time updates:
```redis
# Non-blocking poll from beginning
XREAD COUNT 10 STREAMS execution:{exec_id}:events 0

# Blocking wait for new entries (5 second timeout)
XREAD BLOCK 5000 STREAMS execution:{exec_id}:events $
```

**XRANGE** - Retrieving historical entries:
```redis
# All entries
XRANGE execution:{exec_id}:events - +

# Last 10 entries (reverse)
XREVRANGE execution:{exec_id}:events + - COUNT 10
```

#### Consumer groups for reliable processing

Consumer groups enable distributed processing with acknowledgment:
```redis
# Create group starting from new messages
XGROUP CREATE jobs:stream workers $ MKSTREAM

# Worker reads from group
XREADGROUP GROUP workers worker-1 COUNT 1 BLOCK 5000 STREAMS jobs:stream >

# Acknowledge successful processing
XACK jobs:stream workers {message_id}

# Claim abandoned messages after 60s idle
XAUTOCLAIM jobs:stream workers worker-2 60000 0 COUNT 1
```

#### Recommended key schema

```
# Job queues
jobs:{queue_name}:pending          # Pending work stream
jobs:{queue_name}:completed        # Completed work stream
jobs:dlq                           # Dead letter queue

# Execution tracking
execution:{exec_id}:events         # Event log stream
execution:{exec_id}:state          # Current state (use JSON if Redis Stack)

# Session tracking
session:{session_id}:events        # Per-session event stream

# Job status (Redis Hash)
job:{job_id}                       # HSET status, progress, worker, started_at
```

#### TTL and cleanup strategies

| Strategy | Implementation | Best For |
|----------|---------------|----------|
| Inline MAXLEN | `XADD ... MAXLEN ~ 1000 *` | High-throughput streams |
| Inline MINID | `XADD ... MINID ~ {timestamp}-0 *` | Time-based retention |
| Key EXPIRE | `EXPIRE session:{id}:events 86400` | Session-scoped streams |
| Periodic XTRIM | Scheduled `XTRIM` via cron/n8n | Complex cleanup rules |

**Critical note:** The built-in n8n Redis node does NOT support Redis Streams commands (XADD, XREAD). Use the HTTP Request node with a Redis REST proxy like Upstash, or implement a Code node with the Redis library.

#### Redis Stack for enhanced capabilities

Redis Stack adds RedisJSON and RediSearch modules, enabling:
- **RedisJSON:** Store complex execution state with path-based updates: `JSON.SET execution:{id}:state $.current_step 3`
- **RediSearch:** Index and query executions by status, workflow, or metadata

Docker deployment: `redis/redis-stack:latest` (includes RedisInsight GUI on port 8001)

### 1.4 OpenWebUI pipe function architecture

Pipe functions extend OpenWebUI by creating custom "models" that route requests to external systems.

#### Class structure

```python
from pydantic import BaseModel, Field
from typing import Callable, Awaitable, Optional

class Pipe:
    class Valves(BaseModel):
        N8N_WEBHOOK_URL: str = Field(default="http://n8n:5678/webhook/chat")
        N8N_API_KEY: str = Field(default="")
        POLL_INTERVAL: float = Field(default=2.0)
        TIMEOUT: int = Field(default=300)
    
    def __init__(self):
        self.type = "pipe"
        self.id = "n8n_router"
        self.name = "N8N Workflow Router"
        self.valves = self.Valves()
    
    async def pipe(
        self,
        body: dict,
        __user__: Optional[dict] = None,
        __event_emitter__: Callable[[dict], Awaitable[None]] = None,
    ) -> str:
        # Implementation here
        pass
```

**Valves** define persistent, admin-configurable settings using Pydantic models. The `pipe` method receives the chat body (including messages array and model info) plus optional context parameters.

#### Event emitter patterns

**emit_status** - Progress updates (works in all function calling modes):
```python
await __event_emitter__({
    "type": "status",
    "data": {
        "description": "Processing step 2 of 4...",
        "done": False
    }
})
```

**emit_message** - Content streaming (Default mode only):
```python
await __event_emitter__({
    "type": "message",
    "data": {"content": "Partial response chunk..."}
})
```

**Compatibility note:** The `message` event type is broken in Native function calling mode. Use `status` events for progress and return the final content from the pipe method.

#### Async polling implementation

```python
async def pipe(self, body: dict, __event_emitter__=None, __user__=None):
    import httpx
    import asyncio
    
    user_id = __user__["id"] if __user__ else "anonymous"
    messages = body.get("messages", [])
    question = messages[-1]["content"] if messages else ""
    
    # Submit job
    async with httpx.AsyncClient() as client:
        response = await client.post(
            self.valves.N8N_WEBHOOK_URL,
            json={
                "sessionId": f"owui_{user_id}",
                "chatInput": question,
                "callbackUrl": None  # Using polling instead
            },
            headers={"Authorization": f"Bearer {self.valves.N8N_API_KEY}"}
        )
        job = response.json()
        job_id = job.get("jobId")
    
    # Poll for completion
    elapsed = 0
    while elapsed < self.valves.TIMEOUT:
        await __event_emitter__({
            "type": "status",
            "data": {"description": f"Processing... ({elapsed}s)", "done": False}
        })
        
        async with httpx.AsyncClient() as client:
            status_response = await client.get(
                f"{self.valves.N8N_WEBHOOK_URL}/status/{job_id}"
            )
            status = status_response.json()
        
        if status.get("completed"):
            await __event_emitter__({
                "type": "status",
                "data": {"description": "Complete!", "done": True}
            })
            return status.get("result", "No result")
        
        await asyncio.sleep(self.valves.POLL_INTERVAL)
        elapsed += self.valves.POLL_INTERVAL
    
    return "Error: Request timed out"
```

**HTTP client recommendation:** Use `httpx.AsyncClient()` for async operations with proper connection management. The `aiohttp` library is also supported and available via `open_webui.env.AIOHTTP_CLIENT_TIMEOUT`.

### 1.5 n8n AI Agent configuration

The AI Agent node connects to LLMs and executes tools in an agentic loop.

#### Ollama integration

Use the **Ollama Chat Model** node (not the basic Ollama Model node, which doesn't support tools):

```json
{
  "parameters": {
    "model": "qwen2.5-coder:7b"
  },
  "credentials": {
    "ollamaApi": {
      "baseUrl": "http://ollama:11434"
    }
  },
  "type": "@n8n/n8n-nodes-langchain.lmChatOllama"
}
```

For Docker deployments, use `http://host.docker.internal:11434` if Ollama runs on the host, or `http://ollama:11434` if containerized on the same network.

#### System prompt and agent configuration

```json
{
  "parameters": {
    "promptType": "define",
    "text": "={{ $json.userMessage }}",
    "options": {
      "systemMessage": "You are a code migration assistant specializing in React Native to ReactJS conversions. Always provide complete, working code with TypeScript types.",
      "maxIterations": 10,
      "returnIntermediateSteps": true
    }
  },
  "type": "@n8n/n8n-nodes-langchain.agent"
}
```

#### Memory node configuration

**Window Buffer Memory** provides session-based conversation history:

```json
{
  "parameters": {
    "sessionIdType": "fromInput",
    "sessionKey": "={{ $json.sessionId }}",
    "contextWindowLength": 10
  },
  "type": "@n8n/n8n-nodes-langchain.memoryBufferWindow"
}
```

For persistent memory across restarts, use **Redis Chat Memory** or **Postgres Chat Memory** nodes.

---

## Part 2: External worker architecture

### 2.1 Docker worker container with GPU support

#### Ollama Docker deployment with NVIDIA GPU

**Prerequisites:** NVIDIA drivers and Container Toolkit installed on host.

```bash
# Install NVIDIA Container Toolkit (Ubuntu/Debian)
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -fsSL https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

#### Docker Compose multi-container setup

```yaml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD}
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - worker-network

  ollama:
    image: ollama/ollama:latest
    ports:
      - "11434:11434"
    volumes:
      - ollama_data:/root/.ollama
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    environment:
      - OLLAMA_KEEP_ALIVE=24h
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:11434/"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - worker-network

  opencode-worker:
    build:
      context: ./worker
      dockerfile: Dockerfile
    environment:
      - REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379
      - OLLAMA_HOST=http://ollama:11434
      - WORKER_ID=worker-1
    volumes:
      - ./workspace:/workspace
      - opencode_config:/root/.config/opencode
      - opencode_data:/root/.local/share/opencode
    depends_on:
      redis:
        condition: service_healthy
      ollama:
        condition: service_healthy
    networks:
      - worker-network

networks:
  worker-network:
    driver: bridge

volumes:
  redis_data:
  ollama_data:
  opencode_config:
  opencode_data:
```

#### OpenCode worker Dockerfile

```dockerfile
FROM node:20-alpine

# Install dependencies
RUN apk add --no-cache git curl bash

# Install OpenCode
RUN npm install -g opencode

# Create directories
WORKDIR /workspace
RUN mkdir -p /root/.config/opencode /root/.local/share/opencode

# Copy configuration
COPY opencode.json /root/.config/opencode/opencode.json
COPY AGENTS.md /workspace/AGENTS.md

# Entry point for headless mode
ENTRYPOINT ["opencode"]
CMD ["serve", "--port", "4096"]
```

### 2.2 OpenCode configuration

#### opencode.json schema

The configuration file supports project-local (`./opencode.json`) or global (`~/.config/opencode/opencode.json`) placement:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "ollama/qwen2.5-coder:7b",
  "small_model": "ollama/qwen2.5-coder:7b",
  
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama Local",
      "options": {
        "baseURL": "http://ollama:11434/v1",
        "apiKey": "ollama"
      },
      "models": {
        "qwen2.5-coder:7b": {
          "name": "Qwen2.5 Coder 7B",
          "tools": true,
          "options": {
            "num_ctx": 16384
          }
        }
      }
    }
  },
  
  "tools": {
    "write": true,
    "bash": true,
    "edit": true
  },
  
  "permission": {
    "edit": "always",
    "bash": "always"
  },
  
  "agent": {
    "migration": {
      "mode": "primary",
      "model": "ollama/qwen2.5-coder:7b",
      "tools": { "write": true, "edit": true, "bash": true }
    }
  },
  
  "instructions": ["AGENTS.md", ".opencode/skills/*/SKILL.md"]
}
```

**Key configuration notes:**
- OpenCode connects to Ollama via its OpenAI-compatible `/v1` endpoint
- The `apiKey` field is required but unused for local Ollama
- Set `num_ctx` explicitly since Ollama defaults to 4096 tokens

#### AGENTS.md file format

Place at project root or `~/.config/opencode/AGENTS.md`:

```markdown
# React Native to ReactJS Migration Agent

This project performs automated migration of React Native codebases to ReactJS web applications.

## Project Structure
- `input/` - Source React Native project
- `output/` - Generated ReactJS project
- `skills/` - Migration transformation skills

## Code Standards
- Use TypeScript with strict mode
- Convert StyleSheet to Tailwind CSS classes
- Use React Router DOM v6 for navigation
- Prefer functional components with hooks

## Validation Requirements
- All output must pass ESLint
- All output must pass TypeScript compilation
- Generated tests must achieve >80% coverage

## Commands
- Lint: `npm run lint`
- Type check: `npx tsc --noEmit`
- Test: `npm test`
```

#### Skills system configuration

Skills provide reusable instruction sets for specific tasks. Directory structure:

```
.opencode/skills/
├── react-native-migration/
│   ├── SKILL.md
│   ├── references/
│   │   └── component-mappings.md
│   └── scripts/
│       └── validate-output.sh
```

**SKILL.md frontmatter format:**

```markdown
---
name: react-native-migration
description: Transforms React Native components and patterns to ReactJS web equivalents with Tailwind CSS styling
license: MIT
allowed-tools:
  - read
  - write
  - edit
  - bash
metadata:
  version: "1.0"
  category: "code-transformation"
---

# React Native Migration Skill

## Component Transformation Rules

### View → div
Replace `<View>` with `<div>`. Add `className` with flex-col (RN default direction).

### Text → span
Replace `<Text>` with `<span>`. Note: Text styling doesn't cascade in RN but does in web.

### TouchableOpacity → button
Replace with `<button>` and add CSS for opacity transition on active state.

[Additional instructions...]
```

#### Non-interactive mode (opencode run)

For automated/CI execution:

```bash
# Single prompt execution
opencode run "Migrate the Button component from React Native to ReactJS"

# With specific model
opencode run -m ollama/qwen2.5-coder:7b "Convert App.tsx"

# Continue existing session
opencode run -c "Now add tests for the migrated components"

# Attach files for context
opencode run -f ./src/components/Button.tsx "Migrate this component"

# JSON output for parsing
opencode run --format json "List all React Native imports"

# Headless server mode (for worker integration)
opencode serve --port 4096

# Submit to running server
opencode run --attach http://localhost:4096 "Process migration task"
```

### 2.3 Async job queue pattern

#### Job payload structure

```json
{
  "job_id": "job-550e8400-e29b-41d4-a716-446655440000",
  "type": "code_migration",
  "created_at": "2025-12-15T10:30:00Z",
  "priority": 1,
  "timeout_ms": 300000,
  "retry_count": 0,
  "max_retries": 3,
  "callback_url": "https://n8n.example.com/webhook/job-complete",
  "idempotency_key": "migration-repo-abc-branch-feature",
  "metadata": {
    "workflow_id": "wf-migration-123",
    "execution_id": "exec-456",
    "session_id": "owui_user123_abc",
    "correlation_id": "corr-789"
  },
  "payload": {
    "repository_url": "https://github.com/user/react-native-app",
    "source_branch": "main",
    "target_branch": "feature/web-migration",
    "files": ["src/components/*.tsx", "src/screens/*.tsx"],
    "options": {
      "styling": "tailwind",
      "navigation": "react-router-dom",
      "generate_tests": true
    }
  }
}
```

#### Status reporting structure

```json
{
  "job_id": "job-550e8400-e29b-41d4-a716-446655440000",
  "status": "processing",
  "phase": "execution",
  "progress_percent": 65,
  "current_step": "transforming_components",
  "total_steps": 4,
  "message": "Converting StyleSheet to Tailwind classes...",
  "phases_completed": ["ingestion", "verification"],
  "updated_at": "2025-12-15T10:35:00Z"
}
```

#### Webhook callback implementation

n8n webhook endpoint receives completion notifications:

```json
{
  "job_id": "job-550e8400-e29b-41d4-a716-446655440000",
  "status": "completed",
  "result": {
    "files_processed": 15,
    "output_branch": "feature/web-migration",
    "pr_url": "https://github.com/user/repo/pull/42",
    "validation": {
      "eslint_passed": true,
      "typescript_passed": true,
      "tests_passed": true,
      "coverage": 85.2
    }
  },
  "metrics": {
    "duration_ms": 125000,
    "tokens_used": 45000,
    "files_created": 20,
    "files_modified": 15
  }
}
```

#### Error handling and retry with exponential backoff

```javascript
const calculateBackoff = (attempt, baseMs = 1000, maxMs = 30000) => {
  const exponentialDelay = baseMs * Math.pow(2, attempt);
  const jitter = Math.random() * 1000;
  return Math.min(exponentialDelay + jitter, maxMs);
};

// Retry delays: ~1s, ~2s, ~4s, ~8s, ~16s (capped at 30s)
```

---

## Part 3: React Native to ReactJS migration specifications

### 3.1 Technical transformation requirements

#### Component mapping table

| React Native | ReactJS/HTML | Transformation Notes |
|--------------|--------------|---------------------|
| `<View>` | `<div>` | Add `flex flex-col` (RN default direction) |
| `<Text>` | `<span>` | Wrap in semantic tags (`<p>`, `<h1>`) where appropriate |
| `<ScrollView>` | `<div className="overflow-auto">` | Handle `contentContainerStyle` separately |
| `<FlatList>` | `map()` or `react-window` | Use virtualization for large lists |
| `<TouchableOpacity>` | `<button>` | Add `active:opacity-70 transition-opacity` |
| `<Pressable>` | `<button>` | Map `onPressIn`/`onPressOut` to mouse events |
| `<Image source={{uri}}/>` | `<img src="" />` | Add `alt` attribute, handle `require()` imports |
| `<TextInput>` | `<input>` or `<textarea>` | Map `onChangeText` to `onChange={e => fn(e.target.value)}` |
| `<Switch>` | `<input type="checkbox">` | Style as toggle with CSS |
| `<SafeAreaView>` | `<div>` | Use `env(safe-area-inset-*)` CSS |
| `<Modal>` | `<dialog>` or portal | Handle overlay and body scroll lock |

#### StyleSheet.create() to Tailwind conversion

```javascript
// React Native Input
const styles = StyleSheet.create({
  container: {
    flex: 1,
    flexDirection: 'column',
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#ffffff',
    padding: 16,
    borderRadius: 8,
  },
  title: {
    fontSize: 24,
    fontWeight: 'bold',
    color: '#1f2937',
    marginBottom: 12,
  }
});
```

```jsx
// ReactJS Output with Tailwind
<div className="flex-1 flex flex-col justify-center items-center bg-white p-4 rounded-lg">
  <span className="text-2xl font-bold text-gray-800 mb-3">Title</span>
</div>
```

**Key flexbox differences:** React Native defaults to `flexDirection: 'column'`; CSS defaults to `row`. Always add `flex-col` when converting View containers.

**Dimension conversion:** RN uses unitless numbers (density-independent pixels); add `px` units or use Tailwind spacing scale (4px increments: `p-4` = 16px).

#### React Navigation to React Router DOM

```jsx
// React Native Navigation
import { NavigationContainer } from '@react-navigation/native';
import { createStackNavigator } from '@react-navigation/stack';

const Stack = createStackNavigator();

function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator>
        <Stack.Screen name="Home" component={HomeScreen} />
        <Stack.Screen name="Details" component={DetailsScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}

// Navigation usage
navigation.navigate('Details', { itemId: 42 });
const { itemId } = route.params;
```

```jsx
// React Router DOM
import { BrowserRouter, Routes, Route, useNavigate, useParams } from 'react-router-dom';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<HomeScreen />} />
        <Route path="/details/:itemId" element={<DetailsScreen />} />
      </Routes>
    </BrowserRouter>
  );
}

// Navigation usage
const navigate = useNavigate();
navigate(`/details/${itemId}`);
const { itemId } = useParams();
```

#### Platform-specific code handling

```javascript
// React Native - Remove or convert
if (Platform.OS === 'ios') { /* iOS specific */ }
Platform.select({ ios: value1, android: value2, default: value3 })

// Web - Use feature detection or remove entirely
if (typeof window !== 'undefined') { /* Browser specific */ }
```

Remove `.ios.js` and `.android.js` file suffixes; consolidate to single `.tsx` files.

#### Common library mappings

| React Native Library | Web Equivalent |
|---------------------|----------------|
| `@react-native-async-storage/async-storage` | `localStorage` or `idb` (IndexedDB) |
| `react-native-gesture-handler` | `@use-gesture/react` |
| `react-native-reanimated` | `framer-motion` |
| `react-native-svg` | Native `<svg>` JSX |
| `expo-linear-gradient` | CSS `background: linear-gradient(...)` |
| `expo-blur` | CSS `backdrop-filter: blur(...)` |

### 3.2 qwen2.5-coder:7b specifications

#### Model architecture

| Specification | Value |
|---------------|-------|
| **Total Parameters** | 7.61 billion |
| **Architecture** | Transformer (Qwen2) with GQA |
| **Layers** | 28 |
| **Hidden Size** | 3,584 |
| **Attention Heads** | 28 (Q), 4 (KV) |
| **Context Window** | **32,768 tokens** (default) |
| **Max Context (YaRN)** | 131,072 tokens |
| **Vocabulary** | 151,646 tokens |
| **Training Data** | 5.5 trillion tokens (70% code, 20% text, 10% math) |
| **Languages** | 92 programming languages |

#### VRAM requirements by quantization

| Quantization | Model Size | VRAM Required |
|--------------|------------|---------------|
| Q4_K_M (default) | 4.7 GB | **6-8 GB** |
| Q8_0 | 8.1 GB | 10-12 GB |
| FP16 | 15 GB | 17-20 GB |

**Hardware recommendation:** RTX 3060 (12GB) or better for comfortable Q4 inference with extended context.

#### Ollama configuration

```bash
# Pull model
ollama pull qwen2.5-coder:7b

# Create variant with extended context
ollama run qwen2.5-coder:7b
>>> /set parameter num_ctx 16384
>>> /save qwen2.5-coder:7b-16k
>>> /bye
```

**Modelfile for custom configuration:**

```dockerfile
FROM qwen2.5-coder:7b

PARAMETER num_ctx 16384
PARAMETER temperature 0.3
PARAMETER top_p 0.9
PARAMETER repeat_penalty 1.05

SYSTEM """You are a code migration expert specializing in React Native to ReactJS conversions. Generate clean, type-safe TypeScript code with Tailwind CSS styling."""
```

#### Ollama HTTP API

**Chat endpoint (recommended for conversations):**
```bash
curl http://localhost:11434/api/chat -d '{
  "model": "qwen2.5-coder:7b",
  "messages": [
    {"role": "system", "content": "You are a code migration assistant."},
    {"role": "user", "content": "Convert this View to a div..."}
  ],
  "stream": false,
  "options": {
    "num_ctx": 16384,
    "temperature": 0.3
  }
}'
```

**Response structure:**
```json
{
  "model": "qwen2.5-coder:7b",
  "message": {
    "role": "assistant",
    "content": "Here's the converted component..."
  },
  "done": true,
  "total_duration": 5000000000,
  "eval_count": 150
}
```

### 3.3 Validation and build pipeline

#### ESLint configuration (eslint.config.js)

```javascript
import js from '@eslint/js';
import tseslint from 'typescript-eslint';
import reactHooks from 'eslint-plugin-react-hooks';
import reactRefresh from 'eslint-plugin-react-refresh';

export default tseslint.config(
  { ignores: ['dist', 'node_modules'] },
  {
    extends: [js.configs.recommended, ...tseslint.configs.recommended],
    files: ['**/*.{ts,tsx}'],
    plugins: {
      'react-hooks': reactHooks,
      'react-refresh': reactRefresh,
    },
    rules: {
      ...reactHooks.configs.recommended.rules,
      '@typescript-eslint/no-explicit-any': 'error',
      '@typescript-eslint/no-unused-vars': 'error',
    },
  }
);
```

#### TypeScript configuration (tsconfig.json)

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noEmit": true,
    "skipLibCheck": true,
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["src"]
}
```

#### Validation commands

```bash
# TypeScript type checking
npx tsc --noEmit

# ESLint
npm run lint

# Build verification (Vite)
npm run build

# Test execution
npm test -- --coverage --passWithNoTests
```

#### Jest + React Testing Library setup

```javascript
// jest.config.js
export default {
  preset: 'ts-jest',
  testEnvironment: 'jsdom',
  setupFilesAfterEnv: ['<rootDir>/jest.setup.ts'],
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/src/$1',
    '\\.(css|less|scss)$': 'identity-obj-proxy',
  },
};

// jest.setup.ts
import '@testing-library/jest-dom';
```

---

## Part 4: Integration specifications

### 4.1 n8n to Docker worker communication

#### HTTP job submission flow

```
OpenWebUI → n8n Webhook → Respond 202 → Submit to Redis → Worker Consumes
                                    ↓
                              Worker Polls Redis Stream
                                    ↓
                              Worker Completes
                                    ↓
                              Webhook Callback to n8n
                                    ↓
                              n8n Updates Status / Notifies User
```

#### n8n workflow pattern for async jobs

1. **Webhook Trigger** (responseMode: `responseNode`)
2. **Respond to Webhook** (202 Accepted with job ID)
3. **HTTP Request** (POST job to worker API or XADD to Redis via REST)
4. **Wait Node** (On Webhook Call - waits for callback)
5. **Process Result** (handle completion/failure)

### 4.2 GitHub integration

#### n8n GitHub node operations

**Directly supported:** Create/Edit/Delete/Get File, Create Issue, Create Release, Dispatch Workflow

**Requires HTTP Request node:**
- Create Branch: `POST /repos/{owner}/{repo}/git/refs`
- Create Pull Request: `POST /repos/{owner}/{repo}/pulls`

#### GitHub API rate limits

| Authentication | Rate Limit |
|---------------|------------|
| Unauthenticated | 60/hour |
| Personal Access Token | 5,000/hour |
| GitHub App | 5,000+/hour (scales) |
| GITHUB_TOKEN (Actions) | 1,000/hour per repo |

#### Branch and PR creation via API

```json
// n8n HTTP Request - Create Branch
{
  "method": "POST",
  "url": "https://api.github.com/repos/{{ $json.owner }}/{{ $json.repo }}/git/refs",
  "authentication": "predefinedCredentialType",
  "nodeCredentialType": "githubApi",
  "body": {
    "ref": "refs/heads/feature/web-migration-{{ $json.jobId }}",
    "sha": "{{ $json.baseSha }}"
  }
}

// n8n HTTP Request - Create PR
{
  "method": "POST",
  "url": "https://api.github.com/repos/{{ $json.owner }}/{{ $json.repo }}/pulls",
  "body": {
    "title": "feat: React Native to ReactJS Migration",
    "head": "feature/web-migration-{{ $json.jobId }}",
    "base": "main",
    "body": "## Automated Migration\\n\\nThis PR was generated by the migration workflow.\\n\\n- Files converted: {{ $json.filesConverted }}\\n- Validation: {{ $json.validationStatus }}"
  }
}
```

### 4.3 State machine workflow phases

#### Phase definitions

| Phase | Actions | Success Criteria | Failure Handling |
|-------|---------|------------------|------------------|
| **Ingestion** | Clone repo, parse request, validate inputs | Files accessible, valid config | Immediate fail with validation errors |
| **Verification** | Lint source, check dependencies | Source code parseable | Fail fast, return diagnostics |
| **Execution** | Run transformations, generate code | All files transformed | Retry with backoff, capture logs |
| **Output** | Run tests, commit, push, create PR | Tests pass, PR created | Retry push, manual intervention flag |

#### State transitions in Redis

```redis
# Update phase state
HSET job:{job_id} 
  phase "execution" 
  phase_started_at "1734264000"
  ingestion_completed_at "1734263900"
  verification_completed_at "1734263950"

# Log phase transition event
XADD job:{job_id}:events * 
  type "phase_transition" 
  from "verification" 
  to "execution" 
  timestamp "1734264000"
```

#### Rollback with saga compensation

```javascript
const compensations = {
  'branch_created': async (job) => {
    await github.deleteRef(`refs/heads/${job.branchName}`);
  },
  'files_committed': async (job) => {
    await github.revertCommit(job.commitSha);
  },
  'pr_created': async (job) => {
    await github.closePR(job.prNumber);
  }
};

// Execute in reverse order on failure
for (const step of completedSteps.reverse()) {
  if (compensations[step]) {
    await compensations[step](job);
  }
}
```

---

## Appendix: Complete system diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              OpenWebUI                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐│
│  │                     Pipe Function                                    ││
│  │  - Valves (N8N_URL, API_KEY, TIMEOUT)                               ││
│  │  - emit_status for progress                                          ││
│  │  - Async polling or webhook callback                                 ││
│  └──────────────────────────┬──────────────────────────────────────────┘│
└─────────────────────────────┼───────────────────────────────────────────┘
                              │ HTTP POST
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                               n8n                                        │
│  ┌───────────┐  ┌──────────────┐  ┌──────────┐  ┌───────────────────┐  │
│  │  Webhook  │─▶│ Respond 202  │─▶│ XADD to  │─▶│   Wait for        │  │
│  │  Trigger  │  │  + Job ID    │  │  Redis   │  │   Callback        │  │
│  └───────────┘  └──────────────┘  └──────────┘  └─────────┬─────────┘  │
│                                                            │            │
│  ┌───────────────────────────────────────────────────────┐│            │
│  │  AI Agent (optional preprocessing)                     ││            │
│  │  - Ollama Chat Model (qwen2.5-coder:7b)               ││            │
│  │  - Window Buffer Memory                                ││            │
│  │  - System prompt for routing decisions                 ││            │
│  └───────────────────────────────────────────────────────┘│            │
│                                                 Callback──┘            │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          Redis Streams                                   │
│  jobs:migration:pending ─────────▶ Consumer Groups ────▶ jobs:completed │
│                                          │                               │
│  job:{id} (Hash: status, progress)       │                               │
│  job:{id}:events (Stream: audit log)     │                               │
└──────────────────────────────────────────┼──────────────────────────────┘
                                           │
                              ┌────────────┴────────────┐
                              ▼                         ▼
┌────────────────────────────────────┐  ┌────────────────────────────────┐
│         OpenCode Worker            │  │           Ollama                │
│  ┌──────────────────────────────┐  │  │  ┌──────────────────────────┐  │
│  │  opencode.json               │  │  │  │  qwen2.5-coder:7b        │  │
│  │  - Ollama provider config    │  │  │  │  - 32K context           │  │
│  │  - Agent definitions         │  │◀─│  │  - Q4_K_M quantization   │  │
│  │  - Tool permissions          │  │  │  │  - 6-8GB VRAM            │  │
│  └──────────────────────────────┘  │  │  └──────────────────────────┘  │
│  ┌──────────────────────────────┐  │  │                                │
│  │  AGENTS.md                   │  │  │  GPU: NVIDIA with Container   │
│  │  - Project instructions      │  │  │       Toolkit                  │
│  │  - Code standards            │  │  └────────────────────────────────┘
│  │  - Validation commands       │  │
│  └──────────────────────────────┘  │
│  ┌──────────────────────────────┐  │
│  │  Skills                      │  │
│  │  - react-native-migration/   │  │
│  │    - SKILL.md (frontmatter)  │  │
│  │    - Component mappings      │  │
│  │    - Validation scripts      │  │
│  └──────────────────────────────┘  │
│                                    │
│  Phases: Ingestion → Verification  │
│          → Execution → Output      │
│                                    │
│  Callback: POST to n8n webhook     │
└────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           GitHub                                         │
│  - Create feature branch                                                 │
│  - Commit transformed files                                              │
│  - Create pull request with migration report                             │
│  Rate limit: 5,000 req/hr (PAT)                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Conclusion

This PRD provides a complete technical foundation for implementing a chat-to-workflow routing system. The architecture prioritizes **reliability** through Redis Streams with at-least-once delivery, **observability** through structured status reporting and event logging, and **extensibility** through OpenCode's skills system and n8n's workflow capabilities.

Key implementation priorities should be:

1. **Start with the OpenWebUI pipe function** - the user-facing entry point
2. **Implement the n8n async webhook pattern** - 202 responses with job IDs
3. **Deploy Redis Streams job queue** - reliable work distribution
4. **Configure OpenCode worker** - the execution engine
5. **Add GitHub integration** - output delivery mechanism

The React Native to ReactJS migration use case demonstrates the system's capabilities while the generic architecture supports arbitrary agentic workflows through configuration changes rather than code modifications.