# ChainPilot — Change Report 7

## Overview

The monitor loop was already running autonomously in the background and firing full pipelines when disruptions were detected. The pipeline results were being stored in `_event_log`. The problem was that nothing communicated this to the frontend — the UI stayed on the idle dashboard until the user clicked a button. The "autonomous" part of the system was completely invisible. This change wires the monitor's pipeline results to the frontend via a new SSE stream so the approval gate appears automatically without any user action.

---

## Change 1: SSE Broadcast From Monitor Callback

**File changed:** `backend/main.py`

### What changed

A module-level list was added to hold one asyncio Queue per connected frontend client:

```python
_sse_monitor_clients: list = []
```

The lifespan `on_event` callback was changed from a no-op lambda to a real function that pushes the completed pipeline result to every connected client's queue. Because the callback is called from a `ThreadPoolExecutor` thread (inside `_run_pipeline_async` in monitor.py), `loop.call_soon_threadsafe` is required to safely hand data back to the asyncio event loop:

```python
# Before
task = asyncio.create_task(monitor_loop(on_event=lambda e, r: None))

# After
loop = asyncio.get_running_loop()

def _on_monitor_event(event, result):
    payload = {"event": event, "result": result}
    for q in list(_sse_monitor_clients):
        loop.call_soon_threadsafe(q.put_nowait, payload)

task = asyncio.create_task(monitor_loop(on_event=_on_monitor_event))
```

A new SSE endpoint was added. Each connecting client gets its own queue, receives events as they arrive, and is removed from the list on disconnect:

```python
@app.get("/monitor/stream")
async def monitor_event_stream():
    q = asyncio.Queue()
    _sse_monitor_clients.append(q)

    async def event_stream():
        try:
            while True:
                payload = await q.get()
                yield f"event: monitor_complete\ndata: {json.dumps(payload, default=str)}\n\n"
        except asyncio.CancelledError:
            pass
        finally:
            if q in _sse_monitor_clients:
                _sse_monitor_clients.remove(q)

    return StreamingResponse(event_stream(), media_type="text/event-stream",
                             headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"})
```

`demo_reset` was updated to also clear the event log so the monitor re-fires after reset:

```python
@app.post("/demo/reset")
def demo_reset():
    clear_active_events()
    clear_event_log()   # added
    return {"reset": True}
```

### Why it was made

The monitor's `on_event` callback receives the full pipeline result (`success`, `pipeline_result`, `needs_approval`, `cost_exposure`) after every autonomous pipeline run. Previously this was thrown away. The same data structure the frontend already knows how to render — the approval gate, the pipeline tracker — can be driven entirely from this result. The missing piece was a real-time channel from the backend callback to the frontend; SSE is the same mechanism used by the existing `/pipeline/stream/{sku}` endpoint, so no new infrastructure was needed.

`loop.call_soon_threadsafe` is necessary because `on_event` executes inside a thread pool (it is called from `_run_pipeline_async`, which uses `run_in_executor`). Calling asyncio primitives directly from a non-event-loop thread is unsafe; `call_soon_threadsafe` schedules the queue put on the correct event loop thread.

### What would happen without this change

The monitor fires pipelines silently. Results accumulate in `_event_log` and are accessible via `GET /events`, but nothing reads that endpoint. The frontend stays on the idle dashboard indefinitely. The entire autonomous detection and analysis capability is invisible to anyone watching the UI.

---

## Change 2: Add `clear_event_log`

**File changed:** `backend/monitor.py`

### What changed

A single function was added:

```python
def clear_event_log():
    _event_log.clear()
```

### Why it was made

`_active_events` (cleared by `clear_active_events`) controls whether the monitor re-detects a disruption it has already seen. Clearing it alone is not enough for a clean reset — `_event_log` still holds the result from the previous pipeline run. Without clearing it, the event log grows stale entries across resets. More practically, `demo_reset` needed a way to clear both so the monitor behaves identically after every reset.

### What would happen without this change

After reset, `_event_log` retains the previous run's result. `GET /events` returns stale data. Any future logic that checks the log for prior results would find false positives. For the demo it is a minor issue since the frontend does not poll the log — but it leaves the system in an inconsistent state between runs.

---

## Change 3: Frontend Subscribes to Monitor Stream

**File changed:** `frontend/src/App.jsx`

### What changed

An `autoDetected` state boolean was added:

```js
const [autoDetected, setAutoDetected] = useState(false);
```

A second `useEffect` opens an `EventSource` to `/monitor/stream` on mount. When a `monitor_complete` event arrives, it sets all pipeline state and transitions the phase — the same state updates that a manual pipeline trigger would make, but sourced from the already-completed background result:

```js
useEffect(() => {
    const monitorSse = new EventSource(`${API}/monitor/stream`);
    monitorSse.addEventListener("monitor_complete", (e) => {
        const data = JSON.parse(e.data);
        const result = data.result;
        if (!result?.success) return;
        setPipelineResult(result);
        setCostExposure(result.cost_exposure || 0);
        setToolsCalled(result.pipeline_result?.tools_called || TOOLS_ORDER);
        setActiveStep(10);
        setAutoDetected(true);
        setPhase(result.needs_approval ? "approval" : "done");
    });
    return () => monitorSse.close();
}, []);
```

`setAutoDetected(false)` was added to `reset()`.

`autoDetected` is passed to `ApprovalGate`, which uses it to change the subtitle text:

```jsx
# Before (hardcoded)
<div className="approval-sub">Exposure exceeds $50,000 threshold — select actions to execute</div>

# After (source-aware)
<div className="approval-sub">
    {autoDetected
        ? "Autonomously detected and analyzed by monitor"
        : `Exposure exceeds $${(50000).toLocaleString()} threshold — select actions to execute`}
</div>
```

### Why it was made

The frontend already had all the rendering logic for the approval gate and pipeline tracker. The only missing piece was a trigger to populate those components from a monitor-fired result rather than a user-initiated one. Setting `toolsCalled` to the tools from the pipeline result (rather than the `TOOLS_ORDER` constant) and setting `activeStep` to 10 causes the pipeline tracker to render with all tools green — the audience sees the completed analysis even though they didn't watch it run.

The `autoDetected` flag lets the two entry paths be visually distinguishable: a monitor-fired result shows "Autonomously detected and analyzed by monitor", a manually triggered one shows the cost threshold message. Without this distinction, a demo observer has no way to know the system acted on its own versus waiting to be told to act.

### What would happen without this change

The backend correctly fires pipelines and broadcasts results. The frontend receives the SSE events but ignores them. The approval gate never appears autonomously. The monitor could be removed entirely with no visible effect on the demo.

---

## Summary

| Change | Files | Problem solved | Without it |
|--------|-------|---------------|------------|
| SSE broadcast from monitor callback | `backend/main.py` | No real-time channel from completed background pipelines to the frontend | Monitor fires silently; results are stored but never surfaced; UI stays idle |
| `clear_event_log` on demo reset | `backend/monitor.py`, `backend/main.py` | Event log retained stale results across resets | Log accumulates old entries; system in inconsistent state between demo runs |
| Frontend subscribes to monitor stream | `frontend/src/App.jsx` | Frontend had no listener for monitor-fired results | SSE events arrive but are ignored; approval gate never appears without user action |
