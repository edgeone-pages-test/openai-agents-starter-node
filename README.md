# OpenAI Agents Starter

A full-stack EdgeOne Makers Agent template powered by the OpenAI Agents SDK (TypeScript). Demonstrates how to build a streaming chat Agent with custom tools, session memory, and real-time tool indicators.

## Deploy

[![Deploy with EdgeOne Pages](https://cdnstatic.tencentcs.com/edgeone/pages/deploy.svg)](https://edgeone.ai/makers/new?template=openai-agents-starter-node&from=within&fromAgent=1&agentLang=typescript)

## Features

- **SSE Streaming Chat** — Token-by-token `text_delta` push; `tool_called` events when tools are invoked
- **Session Memory** — Persists multi-turn conversation context via EdgeOne `context.store.openaiSession()`
- **Custom Agent Tools** — get_weather, get_clothing_advice, translate_text, text_statistics
- **Stop Generation** — Dual cancellation: frontend `AbortController` + backend `AbortSignal` truly interrupts the LLM call
- **Tool Indicators** — 4 animated lamps light up in real time when the Agent calls a tool

## Directory Structure

```text
openAI-agent-starter/
├── agents/                        # Stateful EdgeOne Makers Agent Functions (Node/TS)
│   ├── chat/
│   │   └── index.ts              # POST /chat — SSE streaming chat
│   ├── stop/
│   │   └── index.ts              # POST /stop — abort active agent run
│   ├── _logger.ts                # Logger utility (private module)
│   ├── _sse.ts                   # SSE helpers (private module)
│   └── _tools.ts                 # Agent tool definitions (private module)
├── cloud-functions/               # Stateless EdgeOne Pages Node Functions
│   ├── history/
│   │   └── index.ts              # POST /history — load conversation messages
│   └── _logger.ts                # Logger utility
├── src/                           # React frontend (Vite + TypeScript)
│   ├── App.tsx                    # Main app component
│   ├── api.ts                    # Backend API wrappers (SSE streaming)
│   ├── types.ts                  # Type definitions
│   └── components/               # UI components
│       ├── ChatWindow.tsx        # Chat window
│       ├── ChatBubble.tsx        # Message bubble (Markdown support)
│       ├── ChatInput.tsx         # Input box + presets + stop button
│       ├── CodeViewer.tsx        # Code display panel (CRT aesthetic)
│       ├── ToolIndicators.tsx    # Tool indicator container
│       └── ToolLamp.tsx          # Single tool indicator lamp
├── index.html                    # Entry HTML
├── package.json                  # Dependencies (includes @openai/agents)
├── vite.config.ts                # Vite config
├── tsconfig.json                 # TypeScript config
└── .gitignore                    # Git ignore rules
```

> Files prefixed with `_` are private modules — not mapped as public routes by EdgeOne.
>
> **Why two backend folders?** `agents/` holds long-running, stateful routes (active SSE streams, per-conversation abort signals); `cloud-functions/` holds short, stateless routes that just read `context.agent.store`. Splitting them keeps history requests from contending with an active chat for the per-conversation lock.

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `AI_GATEWAY_API_KEY` | Yes | LLM API key |
| `AI_GATEWAY_BASE_URL` | Yes | LLM API base URL (OpenAI-compatible) |
| `AI_GATEWAY_MODEL` | No | Model name (default: `@makers/hy3-preview`) |

## API Endpoints

| Endpoint | Method | Side | Description |
|----------|--------|------|-------------|
| `/chat` | POST | `agents/` | SSE streaming chat. Header: `makers-conversation-id` |
| `/stop` | POST | `agents/` | Abort the active agent run. Body: `{ "conversation_id": "..." }` |
| `/history` | POST | `cloud-functions/` | Get conversation history. Header: `makers-conversation-id` |

### SSE Events

```
event: text_delta     data: {"delta":"Hello"}
event: tool_called    data: {"tool":"get_weather"}
event: ping           data: {"ts":1710000000000}
event: error          data: {"message":"..."}
event: done           data: {"stopped":false}
```

## Architecture

### Backend (`agents/` + `cloud-functions/`)

`agents/` is where the stateful work happens — it owns the live SSE stream and the AbortSignal for the running model call:

1. **Inline LLM model config in `agents/chat/index.ts`** — Creates OpenAI Agents SDK model from environment config
2. **`createTools()`** — Returns the list of custom Agent tools (weather, clothing, translate, statistics)
3. **`context.store.openaiSession(conversationId)`** — Provides session persistence for multi-turn memory
4. **`run(agent, message, { session, stream, signal })`** — Launches the Agent with streaming output
5. **SSE output** — Yields `text_delta`, `tool_called`, `done`, `error` events

`cloud-functions/history` is the stateless `/history` route — it just reads `context.agent.store.getMessages()` to restore the chat after a page refresh, without spinning up an agent run.

### Frontend (`src/`)

- `App.tsx` — Orchestrates chat panel + code viewer, manages SSE stream
- `api.ts` — SSE parsing, dispatches `onTextDelta`, `onToolCalled`, `onDone`, `onError`
- `components/CodeViewer.tsx` — Static code display panel (amber CRT aesthetic) showing the agent flow
- `components/ToolIndicators.tsx` — Animated tool lamps that flash when tools are called

## Local Development

```bash
# Install dependencies
npm install

# Start EdgeOne local dev (frontend + backend)
edgeone makers dev
```
