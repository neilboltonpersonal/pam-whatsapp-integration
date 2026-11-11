# Pam on WhatsApp: How We Built It

A phone-first channel for a PM agent. Neil messages Pam from WhatsApp, Pam answers with full access to the auto-PM repo and MCP tools, and pushes urgent items back unprompted.

Built 2 to 3 May 2026. Removed 12 June 2026 when Pam moved fully into Cursor. The code is no longer on `main`. Recover it with `git show e0aed64~1:remote/<file>`.

## What it did

**Two directions.** Inbound: Neil sends a text or voice note, Pam replies as the same agent that runs in the IDE. Outbound: the hourly pulse sends one consolidated message when something crosses the urgency bar (stakeholder escalation, blocked ticket, deadline within 24 hours, partner Sev1).

**Quick commands.** "brief me" returned a morning summary. "pulse" returned what changed in the last hour. "status" returned the action tracker summary.

**Queue for the IDE.** Work that needs Neil at a keyboard (canvases, email composition, gated approvals) was written to `remote/cursor-queue.jsonl` and presented at the next Cursor session.

## Architecture

```
WhatsApp
  -> Meta WhatsApp Cloud API (webhook)
  -> Cloudflare Tunnel (public HTTPS URL to the laptop)
  -> webhook-server.js (port 3000)
  -> api-server.js (port 4000)
  -> Cursor CLI / SDK running Pam as a local agent
  -> reply
  -> webhook-server.js -> Meta send API -> WhatsApp
```

Voice notes take a detour before the API server:

```
WhatsApp audio (OGG/Opus)
  -> download from Meta media endpoint
  -> ffmpeg (OGG to 16kHz mono PCM)
  -> whisper-cpp-node, local ggml-base.en model
  -> text -> normal pipeline
```

No external transcription API. If any step fails, Pam asks Neil to type instead.

## Components

| File | Role |
|---|---|
| `remote/webhook-server.js` | Express app. Meta verification handshake (`GET /webhook`), signed message intake (`POST /webhook`), allow-list check, voice handling, concurrency queue, reply send. |
| `remote/whatsapp-client.js` | Thin Meta Graph API client: `sendText`, `markRead`, `downloadMedia`. Truncates at 4,096 characters. |
| `remote/api-server.js` | Express app that runs Pam. Endpoints: `/message`, `/send`, `/cmd`, `/queue`, `/health`, `/dashboard`. Builds the prompt from `CLAUDE.md` plus the Pam agent definition, then calls the Cursor agent. |
| `remote/conversation-store.js` | In-memory history: last 20 turns, 4-hour TTL. |
| `remote/pipeline-monitor.js` | Tracks each request through `received, transcribing, calling-api, sending-reply, complete`. Exposed at `GET /pipeline`. |
| `remote/service-watchdog.js` | 30-second checks on `cloudflared` and the API server, auto-restart, recovery log. |
| `remote/health-check.js` | One-shot health report, feeds the dashboard canvas. |
| `remote/transcribe.js` | ffmpeg plus Whisper wrapper. |
| `remote/send-whatsapp.sh` | Shell entry point so Pam in the IDE can push a message. |
| `remote/start-all.sh` | Pre-flight checks (`.env`, ffmpeg, credentials, Whisper model), then starts services. |
| `skills/comms-pa/whatsapp-channel.md` | The skill: when to send, message format, parity table, resilience rules. |

## Key design decisions

**Meta Cloud API, not a consumer-app bridge.** Official, signed webhooks, no unofficial client to break. Setup needs a Meta developer app, a WhatsApp Business phone number ID, a permanent access token, and an app secret.

**Single-user allow-list.** Every inbound message is checked against one allowed number. Everything else is dropped. Every webhook payload is verified with the `x-hub-signature-256` HMAC before processing.

**Run the real Pam, not a lookalike.** The API server injects `CLAUDE.md` and the Pam definition into the prompt and runs the agent with repo access. Skills, context files, the action tracker and the other agents are all reachable. Jira, Drive and GitHub come through the user-level MCP config.

**Snapshots where live access was impossible.** The SDK could not use Slack or Gmail OAuth tokens. IDE Pam writes `slack-snapshot.md`, `gmail-snapshot.md` and `jira-team-snapshot.md` during each hourly pulse. WhatsApp Pam reads them and states the timestamp. Data is up to one hour old.

**n8n removed from the path.** First version routed through an n8n workflow. It added a hop and a failure point, so the webhook server now calls the API server directly (commit `ced7c25`).

**Phone-sized output.** Target 500 characters, ceiling 800. Answer first, one idea per message, native WhatsApp formatting only (single-asterisk bold, underscore italic, no tables, no headers, no URLs). Long replies split into multiple bubbles.

**Outbound is rationed.** The skill defines send-now, batch-on-next-pulse and never-send lists. Awareness-only items stay out. "No changes" stays in Cursor.

## Resilience

| Control | Setting |
|---|---|
| Send timeout | 15s |
| Media download timeout | 30s |
| ffmpeg timeout | 30s |
| Whisper timeout | 60s |
| Agent run timeout | 5 min |
| Webhook to API timeout | 5.5 min |
| Circuit breaker | Opens after 3 consecutive API failures, 60s cooldown, one half-open probe |
| Concurrency | Max 2 messages in parallel, rest queue in order |
| Watchdog | Warn at 3 min stuck, force-complete at 10 min |
| Process safety | Global `uncaughtException` and `unhandledRejection` handlers keep the server alive |

## Cloud variant

A second build collapsed the webhook and API servers into one `remote/cloud/server.js`, deployed on Fly.io (about $2 a month), running Pam as a Cursor Cloud Agent against a fresh clone of the repo. It removed the laptop and tunnel dependency. Gaps: no voice, no Granola, action tracker writes lost on VM teardown, MCP access limited to team config.

## Environment variables

Names only. Values live in `remote/.env`, which is gitignored.

`VERIFY_TOKEN`, `APP_SECRET`, `PHONE_NUMBER_ID`, `META_ACCESS_TOKEN`, `ALLOWED_WA_NUMBER`, `AUTO_PM_API_URL`, `AUTO_PM_API_PORT`, `AUTO_PM_API_SECRET`, `CURSOR_API_KEY`.

## Rebuild checklist

1. Create a Meta developer app with the WhatsApp product. Get phone number ID, permanent token and app secret.
2. Stand up a public HTTPS endpoint (Cloudflare Tunnel locally, or Fly.io).
3. Implement the webhook: GET verification handshake, POST with HMAC check, allow-list, text and audio types.
4. Implement the agent runner with a prompt built from your master context file and persona definition.
5. Add conversation history with a TTL.
6. Add timeouts, circuit breaker, concurrency cap and a watchdog before using it daily.
7. Write snapshot files for any source the agent cannot reach live.
8. Write the outbound rules down first. Rationing is what keeps the channel useful.

## What we learned

**Resilience was most of the work.** The happy path took an afternoon. Timeouts, breakers and restarts took the rest of the build.

**Snapshots beat no access.** A timestamped one-hour-old Slack summary was good enough for a phone triage.

**Rationed outbound matters more than clever inbound.** One message that earns the interruption beats ten that do not.

**Why it was removed.** Pam became IDE-first, and the always-on stack (tunnel, two servers, watchdog) cost more to keep alive than the phone channel returned.

## Source history

| Commit | Change |
|---|---|
| `eb05c46` | Initial WhatsApp channel, API server, n8n workflow |
| `ced7c25` | Bypass n8n, fix agent dispose crash |
| `29c765a` | Voice messages, resilience, watchdog, tunnel auto-start |
| `e0aed64` | Integration removed (use `e0aed64~1` to read the code) |
