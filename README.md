# Cloudflare Workers AI Chat

An AI-powered chat app on Cloudflare Workers using Durable Objects for state and a tiny static UI. Ships with production-ready defaults plus a local mock for quick iteration.
https://red-bush-877e.stockton1229.workers.dev/
## Contents

- Features
- Project Structure
- Quickstart
- Configuration
- Model Selection
- API (with curl)
- Local vs Remote Dev
- Troubleshooting / FAQ
- Architecture
- Roadmap

## Features

- LLM via Workers AI (configurable model with fallbacks)
- Per-session memory using a Durable Object (SQLite migration for free plan)
- Minimal chat UI served as Worker assets
- Local dev mock when AI isn’t available (`--local`)

## Project Structure

- `wrangler.toml` — Worker config, AI and DO bindings, assets, env vars
- `src/worker.ts` — Routes `POST /api/chat` to the Durable Object
- `src/ChatRoom.ts` — Calls Workers AI and persists rolling history
- `frontend/` — Minimal UI (`index.html`, `app.js`)
- `package.json` — Scripts for dev and deploy

## Quickstart

1) Install Wrangler and log in

```bash
npm i -g wrangler
wrangler login
```

2) Dev options

```bash
# Local (no AI; mocked replies)
wrangler dev --local

# Remote (real Workers AI)
wrangler dev
```

3) Deploy

```bash
wrangler deploy
```

## Configuration

- AI binding: `[ai] binding = "AI"` in `wrangler.toml`
- Durable Object binding: `CHAT_ROOM` with SQLite migration (`new_sqlite_classes`)
- Static assets: `[assets].directory = "./frontend"`
- Default model: `[vars].MODEL` (overridable per deploy)

## Model Selection

Default in `wrangler.toml`:

```toml
[vars]
MODEL = "@cf/meta/llama-3.1-70b-instruct"
```

Override per run or deploy:

```bash
wrangler dev --var MODEL="@cf/meta/llama-3.3-70b-instruct-fp8-fast"
wrangler deploy --var MODEL="@cf/meta/llama-3.3-70b-instruct-fp8-fast"
```

If a model isn’t available in your account/region, the app falls back to:

- `@cf/meta/llama-3.1-70b-instruct` (default)
- `@cf/meta/llama-3.1-8b-instruct`
- `@cf/meta/llama-3.2-11b-vision-instruct`
- `@cf/meta/llama-3.2-3b-instruct`
- `@cf/mistral/mistral-7b-instruct-v0.2`

List and test models:

```bash
wrangler ai models list
wrangler ai run @cf/meta/llama-3.1-70b-instruct --input "hello"
```

## API (with curl)

- `POST /api/chat`

Request:

```json
{
  "sessionId": "uuid-or-string",
  "message": "Hello",
  "system": "Optional system prompt"
}
```

Response:

```json
{ "reply": "…assistant text…" }
```

Curl example (replace URL with your worker):

```bash
curl -s https://<name>.<subdomain>.workers.dev/api/chat \
  -H 'content-type: application/json' \
  -d '{"sessionId":"demo","message":"Hello"}' | jq
```

## Local vs Remote Dev

- Local: `wrangler dev --local`
  - Uses a mock assistant reply; Durable Object state still works.

- Remote: `wrangler dev`
  - Requires a workers.dev subdomain; uses real Workers AI.

## Troubleshooting / FAQ

- Remote dev asks for a subdomain
  - Register one in Dashboard → Workers & Pages → Overview, or use `--local`.

- “No such model / task” (error 5007)
  - List models: `wrangler ai models list`
  - Test: `wrangler ai run <model-id> --input "hello"`
  - Set a working model via `MODEL` or rely on fallbacks.

- Free plan Durable Objects
  - This repo uses `new_sqlite_classes` in the first migration as required.

- How much context is kept?
  - The DO stores the last 30 user/assistant turns per session.

## Architecture

```
Frontend → /api/chat (Worker) → Durable Object (per session)
                              ↳ Workers AI (model)
                              ↳ DO storage (rolling memory)
```

## Roadmap

- Streaming replies (SSE) from Workers AI
- Voice input (STT endpoint) → same DO flow
- RAG with Vectorize for long‑term memory
