# Push delivery: SSE vs WebSocket

Both [`sse-server`](sse-server/) and [`broadcast-server`](broadcast-server/) in
this lab are **push-based** — the server proactively sends data, the client
never polls. The real choice is not "push or not"; it is **how** you push:

- **SSE** — server → client only, over one long-lived plain HTTP response
- **WebSocket** — full-duplex, over an upgraded socket

## The one question that decides it

> Does the client need to send back to the server in real time, continuously?

- **No / rarely** → **SSE**
- **Yes, low-latency, continuous** → **WebSocket**

Everything below is a consequence of this.

## Use SSE when

| Situation | Why SSE wins |
|---|---|
| Live feed, notifications, dashboard, progress bar, log streaming | Only need server→client; SSE is enough and simpler |
| LLM token streaming (ChatGPT-style) | The reason SSE is popular again — one request, stream text back |
| Want minimal infrastructure | Plain HTTP: zero libraries, proxies/CDNs/load-balancers already understand it |
| Want auto-reconnect for free | SSE spec has built-in `retry:` + `Last-Event-ID`; the browser reconnects itself |
| Client is a browser | `new EventSource(url)` — one line, no library |

## Use WebSocket when

| Situation | Why a socket is needed |
|---|---|
| Chat, multiplayer game, collaborative editing | Bidirectional, low-latency, both sides send constantly |
| Client emits high-frequency events (cursor, keystrokes, position) | SSE would force a separate POST per event — too costly |
| Binary payloads (audio/video frames, protobuf) | SSE is text/UTF-8 only; WebSocket has binary frames |
| Complex interactive protocol, request/response over one connection | A single socket carries both directions, no split endpoint |

## SSE gotchas to know

1. **HTTP/1.1 caps ~6 connections per domain.** Many tabs exhaust the pool.
   HTTP/2 fixes this via multiplexing. WebSocket is not subject to this limit.
2. **Text/UTF-8 only.** Binary must be base64-encoded → ~33% larger.
3. **Client→server goes a different path.** In this lab, `sse-server`'s
   `connect` must `POST /publish` separately, while `broadcast-server` sends
   over the same socket it receives on.

## Rule of thumb

> Default to **SSE**. Upgrade to **WebSocket** only when you genuinely need
> real-time client→server push or binary frames.

Choosing WebSocket reflexively and never using the reverse channel is a common
form of over-engineering.

## Side by side in this lab

| | [`sse-server`](sse-server/) | [`broadcast-server`](broadcast-server/) |
|---|---|---|
| Transport | Plain HTTP, one long response | Protocol upgrade |
| Direction | Server → client only | Bidirectional |
| Dependencies | None (stdlib) | `gorilla/websocket` |
| Sending from client | Separate `POST /publish` | Same socket |
| Auto-reconnect | Built into the SSE spec (`retry:`) | You build it |
| Browser API | `new EventSource(url)` | `new WebSocket(url)` |
