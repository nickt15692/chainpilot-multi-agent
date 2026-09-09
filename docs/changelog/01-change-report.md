# ChainPilot — Change Report

## Overview

Four areas of improvement were implemented: backend reliability, agent intelligence, agent expansion, and real integrations. The changes are described below in the order they were applied, since each layer builds on the previous one.

---

## Area 1: Backend Reliability

### 1a — Fixed deprecated `asyncio.get_event_loop()`

**Files:** `backend/main.py`, `backend/monitor.py`

**What changed:** Replaced `asyncio.get_event_loop()` with `asyncio.get_running_loop()` in both files.

**Why:** `asyncio.get_event_loop()` is deprecated in Python 3.10 and raises a `DeprecationWarning` when called from inside a coroutine. In Python 3.12 it can raise outright in certain contexts. Both call sites were already inside `async` functions, which means the event loop is guaranteed to be running — `asyncio.get_running_loop()` is the correct API for that situation. It also makes the intent clearer: you're fetching the loop that is currently executing, not creating or fetching a global one.

**Impact if absent:** On Python 3.10–3.11, a `DeprecationWarning` fires on every pipeline request. On Python 3.12+, the call can raise a `RuntimeError` mid-stream, crashing the SSE response and leaving the frontend stuck showing a running pipeline with no completion event.

---

### 1b — Thread-safe `progress_holder`

**File:** `backend/main.py`

**What changed:** Added a `threading.Lock` around all reads and writes to the `progress_holder` dictionary inside `event_stream()`. Writes happen in the `on_progress` callback (called from a `ThreadPoolExecutor` thread); reads happen in the async polling loop (running on the event loop thread). Before this fix, those two threads accessed the same dict concurrently without any synchronization.

The read side now takes a full snapshot of the dict under the lock before using the values, rather than reading individual keys separately:

```python
with _lock:
    snapshot = dict(progress_holder)
current_step = snapshot.get("step", 0)
```

**Why:** Even though CPython's GIL prevents partial writes to individual dict values, it does not prevent the reader from observing an inconsistent state across three separate `__setitem__` calls (step, message, and tool are written one at a time). Without a lock, the polling loop could read a new `step` paired with an old `message` from the previous callback invocation. The lock ensures the reader always sees a consistent set of values from a single `on_progress` call. Taking a snapshot rather than holding the lock during the `yield` also avoids holding the lock across an `await`, which would block the writer thread unnecessarily.

**Impact if absent:** The frontend occasionally receives a mismatched step/message pair — e.g. "Step 5: Checking inventory levels..." when the agent is actually finding alternative suppliers. The pipeline tracker visually desynchronises from the agent's actual progress. Rare under low load but increasingly likely under concurrent requests.

---

### 1c — Removed dead `_last_result` global

**File:** `backend/main.py`

**What changed:** Removed the `_last_result = {}` module-level declaration and the two lines inside `event_stream()` that wrote to it (`global _last_result` / `_last_result = result`).

**Why:** The variable was set after every pipeline run but never read — no endpoint returned it, nothing imported it. It was dead code that could mislead a future reader into thinking there was a way to retrieve the last result via this variable, when there wasn't. Removing it eliminates that false signal and reduces the amount of global mutable state in the module.

**Impact if absent:** No runtime breakage, but the global holds a reference to the last full pipeline result in memory indefinitely. Under concurrent load, it is also a silent write race — multiple pipeline runs writing to the same global with no synchronisation.

---

### 1d — Fixed hardcoded timestamp in `log_disruption_event`

**File:** `backend/tools.py`

**What changed:** Replaced the hardcoded string `"2026-04-13T10:00:00Z"` with a live UTC timestamp:

```python
from datetime import datetime, timezone
"timestamp": datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")
```

**Why:** The hardcoded value was left in from initial development. Every audit log entry was being written with the same fixed date regardless of when the pipeline actually ran, making the audit trail useless for any real troubleshooting or compliance purpose. The fix uses the stdlib `datetime` module (no new dependency) and generates an accurate UTC timestamp at the moment the tool is called.

**Impact if absent:** Every audit log entry carries `2026-04-13T10:00:00Z`. Sorting or filtering events by time returns nonsense. Any compliance, debugging, or forensic use of the audit trail is impossible — all entries are indistinguishable by time.

---

## Area 2: Agent Intelligence

### 2a — Replaced fragile regex with a `finalize_analysis` tool

**Files:** `backend/tools.py`, `backend/agent.py`

**What changed:** The original code tried to extract a structured summary from Claude's free-text response using a regex: `re.search(r'\{[^{}]*"severity"[^{}]*\}', text, re.DOTALL)`. This was replaced by adding a new `finalize_analysis` tool that Claude is required to call as its final step. The tool accepts all summary fields as typed parameters and returns them as a clean dictionary, which the pipeline loop captures directly into `pipeline_result["structured_summary"]`.

The new tool was added to `tools.py` as both a Python function and a JSON schema entry in `TOOLS` and `TOOL_MAP`. The agent loop in `agent.py` was updated to capture its result:

```python
elif tool_name == "finalize_analysis":
    pipeline_result["structured_summary"] = result
```

The system prompt was updated to explicitly require this tool as the final step ("Your FINAL MANDATORY step is ALWAYS finalize_analysis()"), and the old "provide a JSON summary" instruction was removed.

**Why:** The regex approach had two fundamental problems. First, it only matched flat JSON objects — any nested JSON in Claude's response (which is common when Claude elaborates on cost breakdowns or customer lists) would cause the match to fail silently, leaving `structured_summary` as `None`. Second, free-text extraction is inherently brittle: the format of Claude's response can vary between model versions. Making structured output a tool call converts it from a best-effort parse into a guaranteed, typed, schema-validated operation. Claude cannot end its turn without calling `finalize_analysis` (the prompt enforces this), so the structured summary is always present on a completed pipeline run.

**Impact if absent:** `pipeline_result["structured_summary"]` is `None` on any run where Claude includes nested JSON in its response, which is the common case. The approval gate in the frontend cannot display structured cost, severity, or customer data — it falls back to empty values. Any downstream system consuming the structured summary silently receives nothing, with no error raised.

---

### 2b — Added prompt caching

**File:** `backend/agent.py`

**What changed:** The `client.messages.create()` call was updated to pass the system prompt as a structured content block with `cache_control`:

```python
system=[{
    "type": "text",
    "text": system_prompt,
    "cache_control": {"type": "ephemeral"}
}],
extra_headers={"anthropic-beta": "prompt-caching-2024-07-31"}
```

**Why:** A single pipeline run makes up to 25 API calls, each sending the full system prompt (which is now ~500–700 tokens per playbook). Without caching, the same prompt tokens are billed and processed on every single turn. With the `ephemeral` cache control marker, Anthropic caches the system prompt after the first call and serves it from cache on subsequent turns within the same session. This reduces both cost and latency for every turn after the first. The `anthropic-beta` header is required by the current SDK to activate prompt caching.

**Impact if absent:** Every API call in a pipeline run re-bills the full system prompt. Across a 10-turn pipeline with a 600-token prompt, that is ~6,000 extra input tokens billed at full price per run. At scale (many concurrent pipelines) this compounds significantly. Each turn also takes slightly longer as the model re-processes the prompt rather than serving it from cache.

---

### 2c — Incomplete pipeline detection

**File:** `backend/agent.py`

**What changed:** Two new failure modes are now detected explicitly:

1. If Claude reaches `end_turn` without ever calling `finalize_analysis`, `pipeline_result["incomplete"] = True` is set.
2. If the 25-turn loop exhausts without a `break` (i.e., Claude never reached `end_turn`), the `else` clause on the `for` loop fires and sets both `pipeline_result["incomplete"] = True` and `pipeline_result["incomplete_reason"]`.

**Why:** Previously, both failure modes were silent — the pipeline would return `success: True` with a `None` structured summary and no indication that anything had gone wrong. This made debugging and monitoring impossible. Explicitly flagging incomplete pipelines allows the frontend, monitoring systems, or operators to distinguish "pipeline ran and finished" from "pipeline ran but hit the turn limit" — two very different situations with very different remediation paths.

**Impact if absent:** A pipeline that hits the 25-turn limit returns `{"success": True}` with a `None` structured summary — indistinguishable from a successful run. Operators cannot tell whether a real disruption was handled or silently dropped. The approval gate receives no data to display, and no one is alerted that the agent failed to complete its response.

---

## Area 3: Agent Expansion

### 3a — Per-disruption-type system prompts (playbooks)

**File:** `backend/agent.py`

**What changed:** The single `SYSTEM_PROMPT` constant was replaced with a `PLAYBOOKS` dictionary containing three distinct system prompts — one for each disruption type (`low_stock`, `supplier_delay`, `price_spike`) — plus a shared `_BASE_RULES` string appended to all three. At the start of `run_disruption_pipeline`, the correct playbook is selected based on `trigger_event["type"]`:

```python
disruption_type = trigger_event.get("type", "low_stock")
system_prompt = PLAYBOOKS.get(disruption_type, PLAYBOOKS["low_stock"])
```

Each playbook has a different tool execution order tailored to the disruption:

- **`low_stock`**: Starts with `check_inventory_levels` to confirm hours-to-stockout, then immediately finds alternative suppliers and drafts urgent RFQs.
- **`supplier_delay`**: Starts with `check_supplier_status` to confirm the delay, then checks how long current stock can cover production before escalating.
- **`price_spike`**: Starts with `check_price_feeds` to confirm the spike magnitude, then checks buffer stock and whether switching suppliers is cost-effective. `draft_rfq_email` is marked optional — only draft it if a cheaper alternative exists.

**Why:** The original single prompt was written for a stockout scenario. When a supplier delay or price spike was detected and fed through the same prompt, Claude would follow the wrong playbook: starting by confirming inventory levels (irrelevant for a price spike) and applying stockout-specific thresholds and urgency framing to situations that warranted different responses. A supplier delay with 11 days of runway is not the same as a 13-hour stockout, and the agent's reasoning and communications should reflect that. Separate playbooks give Claude the right context and decision rules for each situation, resulting in more accurate severity classifications, better-targeted tool calls, and more appropriate draft communications.

**Impact if absent:** A supplier delay with 11 days of stock cover is classified as CRITICAL and triggers urgent dual RFQs — the same response as a 13-hour stockout. A price spike is treated as an inventory emergency. Draft communications, severity labels, and recommended actions are all wrong for any non-stockout event.

---

### 3b — All detected disruptions processed (not just the first)

**File:** `backend/agent.py`

**What changed:** The `__main__` block was changed from `run_disruption_pipeline(events[0])` to iterating over all detected events:

```python
for event in events:
    result = run_disruption_pipeline(event)
```

**Why:** The monitor loop already correctly processes all detected events in parallel — the bug was only in the standalone `__main__` entrypoint used for CLI testing and debugging. With the mock data, three disruptions are simultaneously active (SKU-4821 low stock, SUP-001 supplier delay, SKU-7703 price spike). Running the script from the command line would only process the first one, giving an incomplete picture. The fix makes the CLI behaviour consistent with the monitor loop.

**Impact if absent:** Running `python3 -m backend.agent` for CLI testing silently drops two of the three active disruptions. Any test or demo run from the command line gives a false impression that the system only detected one event, masking whether the supplier delay and price spike pipelines work correctly.

---

### 3c — Stream endpoint detects actual disruption type

**File:** `backend/main.py`

**What changed:** The `/pipeline/stream/{sku}` endpoint previously hardcoded `"type": "low_stock"` regardless of which SKU was requested. It now calls `detect_disruptions()` and looks up the real event for the given SKU:

```python
all_events = detect_disruptions()
event = next((e for e in all_events if e.get("sku") == sku), None)
```

If no active disruption is found for the SKU (e.g., a manual demo trigger for a healthy SKU), it falls back to constructing a `low_stock` event as before.

**Why:** This was a direct consequence of the playbook work. Without this fix, even though agent.py now has the right playbook for each disruption type, the event object passed to it from the API always said `"type": "low_stock"` — so the correct playbook would never be selected from the frontend. The endpoint needs to know the real disruption type to pass it to the pipeline so the right playbook is chosen.

**Impact if absent:** The three playbooks added in 3a have no effect on any frontend-triggered pipeline run. Every run — regardless of whether it was triggered by a stockout, a supplier delay, or a price spike — uses the stockout playbook. The entire playbook expansion is silently bypassed from the UI.

---

## Area 4: Real Integrations

### 4a — Slack webhook wired on approval

**File:** `backend/main.py`

**What changed:** The `/approve` endpoint was rewritten from a stub that returned a hardcoded string into an endpoint that actually sends the drafted Slack message to the configured webhook URL when the user approves:

```python
if approved and slack_message and SLACK_WEBHOOK_URL:
    payload = json.dumps({
        "text": slack_message,
        "username": "ChainPilot",
        "icon_emoji": ":robot_face:"
    }).encode("utf-8")
    req = urllib.request.Request(SLACK_WEBHOOK_URL, data=payload,
                                 headers={"Content-Type": "application/json"})
    with urllib.request.urlopen(req, timeout=5) as resp:
        slack_result = {"sent": True, "status_code": resp.status}
```

The endpoint gracefully handles three cases: webhook sent successfully, webhook URL not configured (returns `skipped_reason`), and send failure (returns the error). `urllib.request` from the standard library is used so no new dependency is needed. The endpoint now accepts `pipeline_result` in the POST body so it has access to the drafted Slack message that Claude wrote during the pipeline.

**Why:** `SLACK_WEBHOOK_URL` was already present in `config.py` and read from the environment, but nothing in the codebase ever called it. The entire approval flow — generating the Slack draft, presenting it to the user, asking for approval — existed purely to send nothing. Wiring the actual send completes the human-in-the-loop cycle: the agent drafts the message, a human reviews and approves it, and the system sends it. The graceful fallback when the URL is not configured means the feature degrades cleanly for demo environments that haven't set up a Slack workspace.

**Impact if absent:** Clicking "Approve" produces no external effect. The procurement team is never notified. The human-in-the-loop approval flow — framed as the core safety mechanism of the system — completes without any real-world consequence, making it a UI prop rather than a functional control.

---

### 4b — Frontend passes `pipeline_result` in the approval POST

**File:** `frontend/src/App.jsx`

**What changed:** `handleApprove` was updated to include the pipeline result in the POST body:

```javascript
body: JSON.stringify({
    approved: true,
    pipeline_result: pipelineResult?.pipeline_result || {}
})
```

**Why:** This is a required companion to the backend change above. The backend's `/approve` endpoint needs the pipeline result to access the drafted Slack message (stored in `pipeline_result.drafts.slack.message`). The frontend already held this data in the `pipelineResult` state variable — it simply wasn't being sent. Without this change, the backend would always see an empty `pipeline_result` and would never be able to find the Slack draft to send.

**Impact if absent:** The backend receives `pipeline_result: {}` on every approval, finds no Slack draft, and returns `skipped_reason: "No Slack draft in pipeline result"` — making 4a completely non-functional even when a webhook URL is configured. The Slack integration silently does nothing with no visible error in the UI.

---

## Summary Table

| File | Changes |
|------|---------|
| `backend/tools.py` | Live UTC timestamp in `log_disruption_event`; added `finalize_analysis` function, schema, and TOOL_MAP entry |
| `backend/agent.py` | Full replacement: three playbooks replacing one static prompt; per-type dispatch; prompt caching; `finalize_analysis` capture; regex removed; incomplete pipeline detection; all-events `__main__` |
| `backend/main.py` | `get_running_loop`; `threading.Lock` on progress_holder; removed dead `_last_result`; stream endpoint detects real disruption type; `/approve` sends Slack webhook |
| `backend/monitor.py` | `get_running_loop` |
| `frontend/src/App.jsx` | `handleApprove` passes `pipeline_result` in POST body |
