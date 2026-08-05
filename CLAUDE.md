# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Single-file Python quiz buzzer server. Players open the page in a browser, enter their name, and press the big button. The first press wins. State is broadcast to all connected clients via long-polling.

**Constraints:** Python standard library only — no third-party packages.

## Running

```bash
python3 server.py          # listens on :8000
python3 server.py 9000     # custom port
```

Open `http://localhost:8000` in multiple browser tabs to test multiplayer behavior.

## Architecture

Everything lives in `server.py`. There are three layers:

### 1. Shared state (`_state`, `_lock`, `_condition`)

`_state` is a plain dict `{"winner": None | {name, buzzed_at}, "version": int}`.
All mutations go through `_buzz()` or `_reset()`, which hold `_lock` (a `threading.Condition`), modify `_state`, increment `version`, and call `_lock.notify_all()`.

### 2. Long-polling (`_wait_for_change`)

`GET /poll?version=N` blocks (up to 29 s) until `_state["version"] > N`, then returns the snapshot as JSON. Clients immediately re-issue the request after each response, creating a continuous stream of state updates. On timeout (no change), the same snapshot is returned and the client re-polls—this keeps connections from hanging forever behind proxies.

### 3. HTTP handler (`QuizHandler`)

Subclass of `http.server.BaseHTTPRequestHandler` served by `ThreadingHTTPServer` (one thread per connection, necessary for concurrent long-poll holds).

| Method | Path     | Purpose                        |
|--------|----------|--------------------------------|
| GET    | `/`      | Serve embedded HTML            |
| GET    | `/poll`  | Long-poll for state change     |
| GET    | `/state` | Snapshot (used on initial load)|
| POST   | `/buzz`  | `{"name": "..."}` → register buzz |
| POST   | `/reset` | Clear winner                   |

**Transport tuning — do not revert these three settings.** They were chosen from measurement, and each one is load-bearing:

- `QuizHandler.protocol_version = "HTTP/1.1"` — without keep-alive, every poll needs a fresh TCP connection and all clients reconnect simultaneously after each state change.
- `QuizHandler.disable_nagle_algorithm = True` — **required whenever keep-alive is on.** `_send_json()` writes headers and body separately (`wbufsize` is 0), so Nagle holds the body until the peer ACKs the headers, adding ~40 ms to *every* response. Enabling HTTP/1.1 without this makes the app ~20× slower than the original HTTP/1.0 code.
- `QuizServer.request_queue_size = 128` — the stdlib default of 5 drops the excess SYNs when more than a handful of players connect at once, stalling those clients ~1 s on the TCP SYN retransmit.

Measured delivery latency (buzz commit → other clients receive it, 8 clients, loopback): median 2.2 ms, p95 5.2 ms. Before the tuning: median 2.4 ms but p95 722 ms. An SSE rewrite measured 2.2 ms / 4.0 ms — i.e. no meaningful gain over the tuned long-polling, so long-polling was kept.

### 4. Embedded HTML/JS (`HTML` string)

A single-page app inlined as a Python string. The JS polling loop starts with a `GET /state` snapshot, then enters an infinite `poll()` loop. `applyState(data)` handles all UI rendering. The buzzer button is disabled immediately on press and re-enabled only on reset.
