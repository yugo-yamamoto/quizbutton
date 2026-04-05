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

### 4. Embedded HTML/JS (`HTML` string)

A single-page app inlined as a Python string. The JS polling loop starts with a `GET /state` snapshot, then enters an infinite `poll()` loop. `applyState(data)` handles all UI rendering. The buzzer button is disabled immediately on press and re-enabled only on reset.
