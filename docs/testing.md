---
layout: default
title: Interop testing guide
description: Manual interoperability and regression testing for WMP using the reference wmp-cli, httpsse-relay, and wmp-inspector tools.
permalink: /testing/
og_type: article
---

<section class="hero hero--compact">
  <div class="hero__grid" aria-hidden="true"></div>
  <div class="container hero__inner">
    <div class="eyebrow"><span class="eyebrow__bar"></span>Guide · v0.1.0 · Draft</div>
    <h1>Interop testing guide</h1>
    <p class="lede">
      Manual interoperability and regression tests for WMP using the reference tools
      maintained by the SIROS project — <code>wmp-cli</code>, <code>httpsse-relay</code>,
      and <code>wmp-inspector</code>.
    </p>
  </div>
</section>

<section class="section">
  <div class="container">
    <div class="testing-layout">
      <aside class="toc" aria-label="On this page">
        <div class="toc__label">On this page</div>
        <ul>
          <li><a href="#overview">01 — Overview</a></li>
          <li><a href="#prerequisites">02 — Prerequisites</a></li>
          <li><a href="#quick-start">03 — Quick start</a></li>
          <li><a href="#mls">04 — MLS messaging</a></li>
          <li><a href="#invitations">05 — Invitations</a></li>
          <li><a href="#reference">06 — CLI reference</a></li>
          <li><a href="#checklist">07 — Regression checklist</a></li>
          <li><a href="#troubleshooting">08 — Troubleshooting</a></li>
          <li><a href="#automated">09 — Automated tests</a></li>
        </ul>
      </aside>

      <article class="testing-body wmp-prose">

<section id="overview" markdown="1">

## 01 — Overview

This document describes how to perform basic interoperability and regression
tests for WMP using the reference tools maintained by the SIROS project.

| Tool | Repository | Role |
|---|---|---|
| [`wmp-cli`](https://github.com/sirosfoundation/go-wmp/tree/main/cmd/wmp-cli) | `sirosfoundation/go-wmp` | Interactive command-line WMP peer. |
| [`httpsse-relay`](https://github.com/sirosfoundation/go-wmp/tree/main/examples/httpsse-relay) | `sirosfoundation/go-wmp` | Standalone HTTP+SSE relay for firewall-friendly deployments. |
| [`wmp-inspector`](https://github.com/sirosfoundation/wmp-inspector) | `sirosfoundation/wmp-inspector` | Web observer that joins or resumes sessions and logs traffic. |

The tests below exercise:

- Session creation and resumption over HTTPS+SSE.
- Plaintext `wmp.message.deliver` notifications.
- MLS group setup and encrypted messaging.
- Multi-observer fan-out (`wmp-cli` self-echo + `wmp-inspector`).
- Invitation-based session joining.

</section>

<section id="prerequisites" markdown="1">

## 02 — Prerequisites

### Build wmp-cli

```
cd go-wmp
go build -o /tmp/wmp-cli ./cmd/wmp-cli
```

The CLI is self-contained. It supports two transports:

- `ws` (WebSocket) — default.
- `httpsse` (HTTP request/response + Server-Sent Events).

Use `--transport=httpsse` (or `WMP_TRANSPORT=httpsse`) for the relay tests below.

### Deployed reference services

A public relay and inspector are continuously deployed from `main`:

- HTTPSSE relay — <https://httpsse-relay.fly.dev>
- WMP inspector — <https://wmp-inspector.fly.dev>

For local development, run the relay from the `go-wmp` repo:

```
cd go-wmp/examples/httpsse-relay
go run main.go
```

The relay listens on `http://localhost:8080` by default.

### Optional: run wmp-inspector locally

```
cd wmp-inspector
npm install
npm run build
PORT=3000 SELF_ID=https://wmp-inspector.fly.dev npm start
```

</section>

<section id="quick-start" markdown="1">

## 03 — Quick start — plaintext messaging

The simplest end-to-end test. It creates a session on the relay, sends a plaintext
message, and observes it in the inspector.

<div class="step" markdown="1">
<div class="step__head"><span class="step__n">1</span> **Create a session with wmp-cli**</div>

```
/tmp/wmp-cli -transport httpsse
```

Inside the REPL:

```
/join https://httpsse-relay.fly.dev
/create
/status
```

Expected output:

```
Connected.
Creating session...
Session created: sess-<id>
Resumption token: <token>
SSE event stream connected.
Connected: yes
Endpoint:  https://httpsse-relay.fly.dev
Transport: httpsse
Session:   sess-<id>
Resume:    <token>
```

Note the **Session** and **Resume** values; they are needed to resume the session in the inspector.
</div>

<div class="step" markdown="1">
<div class="step__head"><span class="step__n">2</span> **Resume the session in wmp-inspector**</div>

From another terminal:

```
curl -s -X POST https://wmp-inspector.fly.dev/api/resume \
  -H "Content-Type: application/json" \
  -d '{
    "relayUrl": "https://httpsse-relay.fly.dev",
    "sessionId": "sess-<id>",
    "resumptionToken": "<token>"
  }'
```

Expected response:

```
{"sessionId":"sess-<id>"}
```
</div>

<div class="step" markdown="1">
<div class="step__head"><span class="step__n">3</span> **Send a plaintext message**</div>

Back in the wmp-cli REPL:

```
/send hello inspector
```
</div>

<div class="step" markdown="1">
<div class="step__head"><span class="step__n">4</span> **Verify delivery in the inspector**</div>

```
curl -s "https://wmp-inspector.fly.dev/api/sessions/sess-<id>" | jq
```

The session log should contain a `wmp.message.deliver` entry with
`content_type: text/plain` and `body: "hello inspector"`.
</div>

</section>

<section id="mls" markdown="1">

## 04 — MLS encrypted messaging

<div class="callout callout--warn" markdown="1">
The reference deployments use a **noop/insecure MLS provider** so manual tests
can run without real key packages. This is suitable for interop testing of the
WMP MLS message flow, but **must not be used in production**.
</div>

<div class="step" markdown="1">
<div class="step__head"><span class="step__n">1</span> **Create a session and set up MLS**</div>

```
/join https://httpsse-relay.fly.dev
/create
/setup-mls
```

Expected output:

```
Session created: sess-<id>
Resumption token: <token>
SSE event stream connected.
MLS group created: wmp-cli-<id>-mls-<n> (epoch 0)
```
</div>

<div class="step" markdown="1">
<div class="step__head"><span class="step__n">2</span> **Resume the inspector and send encrypted messages**</div>

Resume the inspector as in the quick-start section, then send several MLS messages:

```
/send-mls hello inspector 1
/send-mls hello inspector 2
/send-mls hello inspector 3
```
</div>

<div class="step" markdown="1">
<div class="step__head"><span class="step__n">3</span> **Verify fan-out**</div>

Check the inspector log:

```
curl -s "https://wmp-inspector.fly.dev/api/sessions/sess-<id>" \
  | jq '.log[] | select(.method == "wmp.message.deliver") | .raw.body'
```

Expected result — **all three** messages appear:

```
"hello inspector 1"
"hello inspector 2"
"hello inspector 3"
```

<div class="callout" markdown="1">
**Regression note.** Before the SSE fan-out fix, a resumed inspector reading the
same session as the wmp-cli self-echo received only every other message
(e.g. messages 2 and 4). If you see missing messages today, verify that the
relay image was built from a commit that includes the fan-out changes in
`pkg/wmp/httpsse/httpsse.go`.
</div>

The wmp-cli REPL also decrypts and prints incoming MLS ciphertext:

```
[MLS decrypted epoch 0] hello inspector 1
```
</div>

</section>

<section id="invitations" markdown="1">

## 05 — Inviting multiple endpoints

The `/invite-endpoints` command creates invitation URIs and delivers them as
messages. The intended receiver accepts the invitation and joins the session
on the same relay.

<div class="step" markdown="1">
<div class="step__head"><span class="step__n">1</span> **Invite an endpoint**</div>

```
/join https://httpsse-relay.fly.dev
/create
/invite-endpoints inspector
```

The CLI prints an invitation URI and sends it into the session. Another peer
(or the inspector via `/api/invite`) can accept it.
</div>

<div class="step" markdown="1">
<div class="step__head"><span class="step__n">2</span> **Accept an invitation in the inspector**</div>

```
curl -s -X POST https://wmp-inspector.fly.dev/api/invite \
  -H "Content-Type: application/json" \
  -d '{"uri":"wmp://invite?data=<invitation>"}'
```

The inspector joins the session as a new participant. Messages sent afterward
are delivered to both the inviter and the inspector.
</div>

</section>

<section id="reference" markdown="1">

## 06 — wmp-cli command reference

| Command | Purpose |
|---|---|
| `/connect <url>` | Open a transport without creating a session. |
| `/join <url>` | Connect and create a session in one step. |
| `/create` | Create a session on the current transport. |
| `/resume <token>` | Resume a session using its resumption token. |
| `/rejoin <sess> <token>` | Resume with explicit session id. |
| `/setup-mls` | Create a local (noop/insecure) MLS group for the current session. |
| `/invite-endpoints …` | Build invitation URIs and deliver them into the session. |
| `/send <text>` | Send a plaintext message. |
| `/send-mls <text>` | Send an MLS-encrypted message. |
| `/deliver <json>` | Send a raw `wmp.message.deliver` notification. |
| `/raw` | Toggle printing of all incoming raw JSON. |
| `/status` | Print session/transport status. |
| `/close` | Close the session. |
| `/quit` | Exit the CLI. |

### Environment variables

| Variable | Description |
|---|---|
| `WMP_TRANSPORT` | Default transport (`ws` or `httpsse`). |
| `WMP_INSECURE=1` | Allow `http://` / `ws://` and skip TLS verification. |
| `WMP_SENDER` | Override the sender identity. |

</section>

<section id="checklist" markdown="1">

## 07 — Regression test checklist

Use this checklist when changing the relay, inspector, or CLI:

1. **Plaintext fan-out** — Create a session, resume the inspector, send three `/send` messages. Inspector log contains all three.
2. **MLS fan-out** — Create a session, `/setup-mls`, resume the inspector, send three `/send-mls` messages. Inspector log contains all three and the CLI self-echo prints all three plaintexts.
3. **Session resume** — Close the CLI, start a new CLI, `/connect` to the relay, `/resume <token>`. Messages from the resumed session still reach the inspector.
4. **Invitation flow** — Use `/invite-endpoints` and `/api/invite` to add the inspector; messages reach the invited peer.
5. **Raw JSON monitoring** — Toggle `/raw` and confirm incoming JSON-RPC envelopes are printed.

</section>

<section id="troubleshooting" markdown="1">

## 08 — Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `No MLS group. Use /setup-mls first.` | `/send-mls` called before `/setup-mls`. | Run `/setup-mls` after `/create`. |
| Inspector receives only some messages | Race condition (inspector not yet connected) or stale relay image. | Ensure inspector resume returned `sessionId`, then retry; redeploy relay from main. |
| `Not connected. Use /connect first.` | No transport established. | Run `/join <url>` or `/connect <url>`. |
| `method not allowed from relay` | Wrong URL path. | Use the relay root URL, e.g. `https://httpsse-relay.fly.dev`. |
| TLS errors | Self-signed cert or `http://` without `-insecure`. | Use `-insecure` flag or `WMP_INSECURE=1`. |

</section>

<section id="automated" markdown="1">

## 09 — Automated tests

The [go-wmp](https://github.com/sirosfoundation/go-wmp) repository contains unit
and integration tests that cover the fan-out fix:

```
cd go-wmp
go test ./pkg/wmp/httpsse -v -run TestSSEMultipleClientsReceiveBroadcast
```

Run the full suite before deploying:

```
cd go-wmp
go test ./...
```

The [wmp-inspector](https://github.com/sirosfoundation/wmp-inspector) repository
uses Vitest for its backend and frontend:

```
cd wmp-inspector
npm test
```

</section>

      </article>
    </div>
  </div>
</section>
