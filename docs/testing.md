---
layout: default
---

# WMP Interop Testing Guide

## Wallet Messaging Protocol — Manual Interop Testing with Reference Tools

**Status:** Draft  
**Version:** 0.1.0  
**Date:** 2026-07-24

## 1. Overview

This document describes how to perform basic interoperability and regression tests for WMP using the reference tools maintained by the SIROS project:

| Tool | Repository | Role |
|------|------------|------|
| [wmp-cli](https://github.com/sirosfoundation/go-wmp/tree/main/cmd/wmp-cli) | `sirosfoundation/go-wmp` | Interactive command-line WMP peer |
| [httpsse-relay](https://github.com/sirosfoundation/go-wmp/tree/main/examples/httpsse-relay) | `sirosfoundation/go-wmp` | Standalone HTTP+SSE relay for firewall-friendly deployments |
| [wmp-inspector](https://github.com/sirosfoundation/wmp-inspector) | `sirosfoundation/wmp-inspector` | Web-based observer that joins or resumes WMP sessions and logs traffic |

The tests below exercise:

- Session creation and resumption over HTTPS+SSE.
- Plaintext `wmp.message.deliver` notifications.
- MLS group setup and encrypted messaging.
- Multi-observer fan-out (wmp-cli self-echo + wmp-inspector).
- Invitation-based session joining.

## 2. Prerequisites

### 2.1 Build wmp-cli

```bash
cd go-wmp
go build -o /tmp/wmp-cli ./cmd/wmp-cli
```

The CLI is self-contained. It supports two transports:

- `ws` (WebSocket) — default.
- `httpsse` (HTTP request/response + Server-Sent Events).

Use `--transport=httpsse` (or `WMP_TRANSPORT=httpsse`) for the relay tests below.

### 2.2 Deployed Reference Services

A public relay and inspector are continuously deployed from `main`:

| Service | URL |
|---------|-----|
| HTTPSSE relay | `https://httpsse-relay.fly.dev` |
| WMP inspector | `https://wmp-inspector.fly.dev` |

For local development, run the relay from the `go-wmp` repo:

```bash
cd go-wmp/examples/httpsse-relay
go run main.go
```

The relay listens on `http://localhost:8080` by default.

### 2.3 Optional: Run wmp-inspector Locally

```bash
cd wmp-inspector
npm install
npm run build
PORT=3000 SELF_ID=https://wmp-inspector.fly.dev npm start
```

## 3. Quick Start — Plaintext Messaging

This is the simplest end-to-end test. It creates a session on the relay, sends a plaintext message, and observes it in the inspector.

### 3.1 Create a Session with wmp-cli

```bash
/tmp/wmp-cli -transport httpsse
```

Inside the REPL:

```text
/join https://httpsse-relay.fly.dev
/create
/status
```

Expected output:

```text
Connected.
Creating session...
Session created: sess-<id>
Resumption token: <token>
SSE event stream connected.
Connected:  yes
Endpoint:   https://httpsse-relay.fly.dev
Transport:  httpsse
Session:    sess-<id>
Resume:     <token>
```

Note the **Session** and **Resume** values; they are needed to resume the session in the inspector.

### 3.2 Resume the Session in wmp-inspector

From another terminal:

```bash
curl -s -X POST https://wmp-inspector.fly.dev/api/resume \
  -H "Content-Type: application/json" \
  -d '{
    "relayUrl": "https://httpsse-relay.fly.dev",
    "sessionId": "sess-<id>",
    "resumptionToken": "<token>"
  }'
```

Expected response:

```json
{"sessionId":"sess-<id>"}
```

### 3.3 Send a Plaintext Message

Back in the wmp-cli REPL:

```text
/send hello inspector
```

### 3.4 Verify Delivery in the Inspector

```bash
curl -s "https://wmp-inspector.fly.dev/api/sessions/sess-<id>" | jq
```

The session log should contain a `wmp.message.deliver` entry with `content_type: text/plain` and `body: "hello inspector"`.

## 4. Testing MLS Encrypted Messaging

The reference deployments use a **noop/insecure MLS provider** so that manual tests can run without real key packages. This is suitable for interop testing of the WMP MLS message flow, but **must not be used in production**.

### 4.1 Create a Session and Set Up MLS

```text
/join https://httpsse-relay.fly.dev
/create
/setup-mls
```

Expected output:

```text
Session created: sess-<id>
Resumption token: <token>
SSE event stream connected.
MLS group created: wmp-cli-<pid>-mls-<epoch> (epoch 0)
```

### 4.2 Resume the Inspector and Send Encrypted Messages

Resume the inspector as in Section 3.2, then send several MLS messages:

```text
/send-mls hello inspector 1
/send-mls hello inspector 2
/send-mls hello inspector 3
```

### 4.3 Verify Fan-Out

Check the inspector log:

```bash
curl -s "https://wmp-inspector.fly.dev/api/sessions/sess-<id>" | jq '.log[] | select(.method == "wmp.message.deliver") | .raw.body'
```

Expected result — **all three** messages appear:

```text
"hello inspector 1"
"hello inspector 2"
"hello inspector 3"
```

> **Regression note:** Before the SSE fan-out fix, a resumed inspector reading the same session as the wmp-cli self-echo received only every other message (e.g. messages 2 and 4). If you see missing messages today, verify that the relay image was built from a commit that includes the fan-out changes in `pkg/wmp/httpsse/httpsse.go`.

The wmp-cli REPL also decrypts and prints incoming MLS ciphertext:

```text
[MLS decrypted epoch 0] hello inspector 1
```

## 5. Inviting Multiple Endpoints

The `/invite-endpoints` command creates invitation URIs and delivers them as messages. The intended receiver accepts the invitation and joins the session on the same relay.

### 5.1 Invite an Endpoint

```text
/join https://httpsse-relay.fly.dev
/create
/invite-endpoints inspector
```

The CLI prints an invitation URI and sends it into the session. Another peer (or the inspector via `/api/invite`) can accept it.

### 5.2 Accept an Invitation in the Inspector

```bash
curl -s -X POST https://wmp-inspector.fly.dev/api/invite \
  -H "Content-Type: application/json" \
  -d '{"uri":"wmp://invite?data=<base64url>"}'
```

The inspector joins the session as a new participant. Messages sent afterward are delivered to both the inviter and the inspector.

## 6. wmp-cli Command Reference

| Command | Purpose |
|---------|---------|
| `/connect <url>` | Open a transport without creating a session |
| `/join <url>` | Connect and create a session in one step |
| `/create` | Create a session on the current transport |
| `/resume <token>` | Resume a session using its resumption token |
| `/rejoin <sid> <token>` | Resume with explicit session id |
| `/setup-mls` | Create a local (noop/insecure) MLS group for the current session |
| `/invite-endpoints <provider>…` | Build invitation URIs and deliver them into the session |
| `/send <text>` | Send a plaintext message |
| `/send-mls <text>` | Send an MLS-encrypted message |
| `/deliver <json>` | Send a raw `wmp.message.deliver` notification |
| `/raw` | Toggle printing of all incoming raw JSON |
| `/status` | Print session/transport status |
| `/close` | Close the session |
| `/quit` | Exit the CLI |

Environment variables:

- `WMP_TRANSPORT` — default transport (`ws` or `httpsse`).
- `WMP_INSECURE=1` — allow `http://` / `ws://` and skip TLS verification.
- `WMP_SENDER` — override the sender identity.

## 7. Regression Test Checklist

Use this checklist when changing the relay, inspector, or CLI:

1. **Plaintext fan-out.** Create a session, resume the inspector, send three `/send` messages. Inspector log contains all three.
2. **MLS fan-out.** Create a session, `/setup-mls`, resume the inspector, send three `/send-mls` messages. Inspector log contains all three and the CLI self-echo prints all three plaintexts.
3. **Session resume.** Close the CLI, start a new CLI, `/connect` to the relay, `/resume <token>`; messages sent from the resumed session are still delivered to the inspector.
4. **Invitation flow.** Use `/invite-endpoints` and `/api/invite` to add the inspector; messages reach the invited peer.
5. **Raw JSON monitoring.** Toggle `/raw` and confirm incoming JSON-RPC envelopes are printed.

## 8. Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| `No MLS group. Use /setup-mls first.` | `/send-mls` called before `/setup-mls` | Run `/setup-mls` after `/create` |
| Inspector receives only some messages | Race condition (inspector not yet connected) or stale relay image | Ensure inspector resume returned `sessionId`, then retry; redeploy relay from `main` |
| `Not connected. Use /connect first.` | No transport established | Run `/join <url>` or `/connect <url>` |
| `method not allowed` from relay | Wrong URL path | Use the relay root URL, e.g. `https://httpsse-relay.fly.dev` |
| TLS errors | Self-signed cert or `http://` without `-insecure` | Use `-insecure` flag or `WMP_INSECURE=1` |

## 9. Automated Tests

The `go-wmp` repository contains unit and integration tests that cover the fan-out fix:

```bash
cd go-wmp
go test ./pkg/wmp/httpsse -v -run TestSSEMultipleClientsReceiveBroadcast
```

Run the full suite before deploying:

```bash
cd go-wmp
go test ./...
```

The `wmp-inspector` repository uses Vitest for its backend and frontend. Run:

```bash
cd wmp-inspector
npm test
```
