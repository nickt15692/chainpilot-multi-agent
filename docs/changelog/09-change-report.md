# ChainPilot — Change Report 9

## Overview

Four operational issues were fixed before the demo. First, double-clicking `start.command` opened the browser to the wrong port — Vite's default 5173 instead of the configured 3002 — so the app appeared as a blank page. Second, `start.command` and Vite both tried to open the browser independently, resulting in two tabs every time. Third, running the pipeline triggered a `429 RateLimitError` from the Anthropic API: three Claude agents firing simultaneously (orchestrator, procurement specialist, risk specialist) exceeded the 30,000 input-token-per-minute free-tier cap. A retry wrapper with exponential backoff was added so the pipeline recovers automatically without crashing. The README was also updated to reflect the current demo flow, pipeline structure, and agent architecture.

---

## Change 1: Fix Browser Port in `start.command`

**File changed:** `start.command`

### What changed

The final `open` command was corrected from Vite's default port to the port configured in `vite.config.js`:

```bash
# Before
open "http://localhost:5173"

# After
open "http://localhost:3002"
```

### Why it was made

`vite.config.js` sets `server: { port: 3002 }`, so the dev server always binds to 3002. The `start.command` script launched both servers correctly but then opened the browser to 5173, which had nothing listening on it.

### What would happen without this change

Double-clicking `start.command` would open a blank browser tab. The user would need to manually navigate to `http://localhost:3002` to see the app — a confusing failure mode during a demo setup.

---

## Change 2: Add Rate Limit Retry Logic to Agent Pipeline

**File changed:** `backend/agent.py`

### What changed

A `_create_with_retry` wrapper was added that catches `anthropic.RateLimitError` and retries with exponential backoff before re-raising:

```python
import time

def _create_with_retry(max_retries=5, initial_wait=15, **kwargs):
    """Call client.messages.create with exponential backoff on rate limit errors."""
    wait = initial_wait
    for attempt in range(max_retries):
        try:
            return client.messages.create(**kwargs)
        except anthropic.RateLimitError:
            if attempt == max_retries - 1:
                raise
            time.sleep(wait)
            wait = min(wait * 2, 60)
```

All four `client.messages.create(...)` call sites — the orchestrator, procurement specialist, risk specialist, and communications agent — were replaced with `_create_with_retry(...)`:

```python
# Before (all four agents)
response = client.messages.create(
    model=MODEL,
    max_tokens=MAX_TOKENS,
    system=[...],
    tools=...,
    messages=messages,
    extra_headers={"anthropic-beta": "prompt-caching-2024-07-31"}
)

# After
response = _create_with_retry(
    model=MODEL,
    max_tokens=MAX_TOKENS,
    system=[...],
    tools=...,
    messages=messages,
    extra_headers={"anthropic-beta": "prompt-caching-2024-07-31"}
)
```

### Why it was made

The free-tier rate limit is 30,000 input tokens per minute. The pipeline runs three Claude agents — orchestrator, procurement specialist, and risk specialist — where the two specialists execute in parallel via `ThreadPoolExecutor`. Each agent sends its full system prompt plus tool schemas on every turn. When the specialists fire simultaneously their combined token usage can exceed 30k TPM in the same minute window, causing a `429 RateLimitError` that crashed the entire pipeline with an ASGI exception.

The initial wait of 15 seconds is chosen because the token bucket at 30k TPM refills at ~500 tokens per second — a 5,000-token overage clears in 10 seconds. The backoff doubles to 30s then caps at 60s, which covers the worst case of multiple agents all retrying at once. The `prompt-caching-2024-07-31` beta header means subsequent turns within the same pipeline run use cached prompt tokens at 10% of the normal cost, so the rate limit typically only applies to the first pipeline run per session.

### What would happen without this change

Any API call that exceeds the token rate limit raises `anthropic.RateLimitError`, which propagates uncaught up through the thread pool, through the FastAPI streaming handler, and crashes the ASGI application with a 500 error. The frontend receives an `error` SSE event and the pipeline fails completely with no recovery.

---

## Change 3: Update README

**File changed:** `README.md`

### What changed

The README was rewritten to reflect the current state of the project:

- **Setup section** now leads with the `setup.command` / `start.command` double-click workflow for Mac, with manual terminal commands as a fallback
- **Demo flow** updated from the old mid-crisis starting state to the current three-act sequence: healthy baseline → inject shock → recommendation card → approval gate → reset
- **Pipeline** expanded from 9 to 10 tools and updated with the shocked-state values (12% stock, 14-day delay) that the audience will actually see
- **Agent architecture** section added explaining the orchestrator → parallel specialists → communications agent structure and why `ThreadPoolExecutor` is used
- **Demo state endpoints** table added (`/demo/pre-shock`, `/demo/inject-shock`, `/demo/reset`) replacing the old curl reset example
- **Key technical points** updated to include the healthy baseline, multi-agent coordination, and the `RecommendationCard` as the structured output of `finalize_analysis`

### Why it was made

The old README described a single-agent pipeline starting at crisis state with a "Simulate Disruption" button — none of which matched the current implementation. A presenter reading the README before the demo would be briefed on the wrong flow.

### What would happen without this change

The README would describe a demo that no longer exists: a single agent, a mid-crisis starting state, and a 9-tool pipeline without `finalize_analysis`. Anyone using the README as a reference during setup or presentation would be confused by the mismatch.

---

## Change 4: Remove Duplicate Browser Open in Vite Config

**File changed:** `frontend/vite.config.js`

### What changed

`open: true` was removed from the Vite server config:

```js
// Before
export default defineConfig({ plugins: [react()], server: { port: 3002, open: true } })

// After
export default defineConfig({ plugins: [react()], server: { port: 3002 } })
```

### Why it was made

`start.command` already calls `open "http://localhost:3002"` after a 4-second delay to give both servers time to start. With `open: true` in the Vite config, Vite also opens a browser tab the moment the dev server is ready — typically 1–2 seconds after startup. The result was two browser tabs opening on every `start.command` run: one from Vite immediately, one from the script 4 seconds later.

Removing `open: true` makes `start.command` the single authority on when the browser opens. The 4-second delay in the script is intentional — it ensures the backend is also ready before the frontend loads, avoiding a dashboard fetch that fails because the API isn't up yet.

### What would happen without this change

Every `start.command` launch opens two browser tabs. The first (from Vite) may load before the backend is ready, showing a network error in the dashboard. The second (from the script) loads correctly. During a demo setup this is confusing and leaves an extra tab to close.

---

## Summary

| Change | Files | Problem solved | Without it |
|--------|-------|---------------|------------|
| Fix browser port in start.command | `start.command` | Script opened wrong port (5173 instead of 3002) | Browser opens to blank page after double-clicking start.command |
| Remove duplicate browser open | `frontend/vite.config.js` | Vite and start.command both opened the browser, causing two tabs | Two tabs every launch; first tab may show a network error before backend is ready |
| Rate limit retry logic | `backend/agent.py` | Parallel agents exceeded 30k TPM free-tier cap, crashing the pipeline | `429 RateLimitError` crashes the ASGI app; pipeline fails with no recovery |
| README rewrite | `README.md` | README described the old single-agent, mid-crisis-start demo | Presenter briefed on wrong demo flow, wrong tool count, wrong starting state |
