# ChainPilot — Change Report 8

## Overview

The demo started with data already in a crisis state — inventory at 12%, suppliers delayed — with no "before" to contrast against. An audience watching the demo had no way to see the shock *happen*; they just arrived to a system already on fire. This change adds a healthy baseline state and a one-click shock injection that visually transitions the dashboard from green to red, then immediately triggers the agent pipeline. A second gap was also closed: the agent's `finalize_analysis` output — its structured recommendation, the "why" behind its decision — was computed and transmitted to the frontend but never rendered. A new Recommendation Card surfaces that output prominently before the approval gate so the audience sees the agent's reasoned conclusion, not just draft emails.

---

## Change 1: Add Healthy and Shocked State Patches to Mock Data

**File changed:** `mock_data.py`

### What changed

Two patch dicts and two mutating functions were added. The patch dicts contain only the fields that differ between states — `stock_units`, `stock_pct`, `hours_to_stockout` for inventory, and `status`, `delay_days`, `delay_reason` for suppliers:

```python
_SHOCKED_INVENTORY_PATCH = {
    "SKU-4821": {"stock_units": 42,   "stock_pct": 0.12, "hours_to_stockout": 8.5},
    "SKU-7703": {"stock_units": 1100, "stock_pct": 0.22, "hours_to_stockout": 26.4},
    "SKU-5541": {"stock_units": 68,   "stock_pct": 0.31, "hours_to_stockout": 74},
}
_HEALTHY_INVENTORY_PATCH = {
    "SKU-4821": {"stock_units": 1326, "stock_pct": 0.78, "hours_to_stockout": 220},
    "SKU-7703": {"stock_units": 4050, "stock_pct": 0.81, "hours_to_stockout": 243},
    "SKU-5541": {"stock_units": 187,  "stock_pct": 0.85, "hours_to_stockout": 204},
}

def apply_pre_shock():
    for sku, patch in _HEALTHY_INVENTORY_PATCH.items():
        INVENTORY[sku].update(patch)
    for sid, patch in _HEALTHY_SUPPLIERS_PATCH.items():
        SUPPLIERS[sid].update(patch)

def apply_shock():
    for sku, patch in _SHOCKED_INVENTORY_PATCH.items():
        INVENTORY[sku].update(patch)
    for sid, patch in _SHOCKED_SUPPLIERS_PATCH.items():
        SUPPLIERS[sid].update(patch)
```

The live `INVENTORY` and `SUPPLIERS` dicts were updated to start at healthy values so the server starts in a clean state. Both suppliers that were previously hardcoded as `DELAYED` start as `ACTIVE`.

### Why it was made

Mutating the shared dicts in place keeps the approach compatible with all existing tools and endpoints — `check_inventory_levels`, `check_supplier_status`, `/dashboard`, and every agent tool read directly from these module-level dicts. Swapping values with `update()` means there is no indirection or conditional at the point of read; callers see whichever state was last applied. Two separate patch dicts (rather than full copies of the dicts) keep the change minimal and make it clear exactly what values differ between states.

### What would happen without this change

The server would start already in crisis state. The audience has no reference point for what "normal" looks like, so the visual drop when the shock is injected cannot happen. The demo starts mid-emergency rather than telling the story of an emergency unfolding.

---

## Change 2: Add `/demo/pre-shock` and `/demo/inject-shock` Endpoints

**File changed:** `backend/main.py`

### What changed

Two new POST endpoints were added. `import mock_data` was added (alongside the existing `from mock_data import ...`) so the module-level functions can be called:

```python
import mock_data

@app.post("/demo/pre-shock")
def demo_pre_shock():
    mock_data.apply_pre_shock()
    clear_active_events()
    clear_event_log()
    return {"reset": True, "state": "pre-shock"}

@app.post("/demo/inject-shock")
def demo_inject_shock():
    mock_data.apply_shock()
    clear_active_events()
    return {"injected": True, "state": "shocked"}
```

### Why it was made

`/demo/pre-shock` clears both `_active_events` and `_event_log` in addition to resetting data — this ensures the monitor treats the post-reset state as fresh and will re-detect the disruption after `/demo/inject-shock` is called. `/demo/inject-shock` only clears `_active_events` (not the log) because the pipeline is about to run fresh; there is no stale log entry to clear yet.

The frontend handles triggering the pipeline after injection, which keeps the backend endpoints single-purpose and avoids needing the backend to initiate a pipeline run from an HTTP handler.

### What would happen without this change

There is no server-side mechanism to transition between states. The frontend could not trigger a shock or reset to healthy without the host manually restarting the server.

---

## Change 3: Pre-Shock Phase, Inject Shock Button, and Reset Flow

**File changed:** `frontend/src/App.jsx`

### What changed

`phase` was given a new starting value `"pre-shock"`. On mount, the app calls `/demo/pre-shock` before the first dashboard fetch to guarantee the server is in healthy state regardless of prior runs:

```js
// Before
const [phase, setPhase] = useState("monitor");
useEffect(() => {
    fetchDashboard();
    pollRef.current = setInterval(fetchDashboard, 8000);
    ...
}, []);

// After
const [phase, setPhase] = useState("pre-shock");
useEffect(() => {
    fetch(`${API}/demo/pre-shock`, { method: "POST" }).finally(() => {
        fetchDashboard();
        setPhase("pre-shock");
    });
    pollRef.current = setInterval(fetchDashboard, 8000);
    ...
}, []);
```

An `injectShock` handler was added that sequences three steps: post to inject the shock, refresh the dashboard (so bars visually update), then immediately start the pipeline stream:

```js
const injectShock = async () => {
    await fetch(`${API}/demo/inject-shock`, { method: "POST" });
    await fetchDashboard();
    setPhase("monitor");
    triggerPipeline("SKU-4821");
};
```

An "⚡ Inject Supply Chain Shock" button renders in the left panel when `phase === "pre-shock"`. The idle state in the right panel shows "Supply chain nominal — All systems healthy" in this phase instead of the generic watching message.

`reset()` was updated to call `/demo/pre-shock` and set `phase` back to `"pre-shock"` so each demo run starts identically:

```js
// Before
await fetch(`${API}/demo/reset`, { method: "POST" });
setPhase("monitor");

// After
await fetch(`${API}/demo/pre-shock`, { method: "POST" });
setPhase("pre-shock");
```

The header Reset button's visibility condition was changed from `phase !== "monitor"` to `phase !== "pre-shock"` so it appears as soon as the shock is injected.

### Why it was made

Sequencing the dashboard refresh between the inject POST and the pipeline start is the key design decision. `fetchDashboard` is async and updates the React state, so by the time `triggerPipeline` fires, the inventory bars and supplier status in the UI already reflect the shocked values. The audience sees the crisis appear on the left panel at the same moment the right panel starts showing the pipeline running — both halves of the demo are live simultaneously.

Calling `/demo/pre-shock` on mount rather than relying on server startup state means the demo is repeatable without a server restart. Any prior state from a previous run is cleared automatically.

### What would happen without this change

The bars would only update on the next 8-second poll, not immediately when the shock is injected, so there would be a visible lag between the button click and the visual change. The connection between "shock happened" and "agent is responding" would be broken.

---

## Change 4: RecommendationCard Component

**Files changed:** `frontend/src/App.jsx`, `frontend/src/App.css`

### What changed

A `RecommendationCard` component was added that reads `structured_summary` from the pipeline result — the output of `finalize_analysis` — and displays the agent's conclusion:

```jsx
function RecommendationCard({ summary, exposure }) {
  if (!summary) return null;
  const { severity, confidence, recommended_action, recommended_supplier } = summary;
  const borderColor = SEV_BORDER[severity] || "#6B7280";
  return (
    <div className="recommendation-card" style={{ borderLeftColor: borderColor }}>
      <div className="rec-header">
        <span className="rec-title">Agent Recommendation</span>
        <div className="rec-badges">
          <span className="rec-severity-badge" style={{ background: borderColor }}>{severity}</span>
          <span className="rec-confidence">{confidence} confidence</span>
        </div>
      </div>
      <p className="rec-action-text">"{recommended_action}"</p>
      <div className="rec-meta">
        <span>Recommended supplier: <strong>{recommended_supplier}</strong></span>
        {exposure > 0 && <span>30-day exposure: <strong>${Math.round(exposure).toLocaleString()}</strong></span>}
      </div>
    </div>
  );
}
```

It is rendered in two places: at the top of `ApprovalGate` (above the stats row, before the action checkboxes) and in the done state (above the tool-count stats).

In `ApprovalGate`, `structured_summary` is read from the result prop:

```js
const summary = result?.pipeline_result?.structured_summary || null;
```

CSS was added for the card, its severity badge, and the action text:

```css
.recommendation-card { border-left: 4px solid #DC2626; background: var(--card);
  border-radius: var(--radius); padding: 14px 16px; margin-bottom: 16px; }
.rec-action-text { font-size: 0.88rem; color: var(--text); line-height: 1.5;
  font-style: italic; margin-bottom: 10px; }
```

### Why it was made

`structured_summary` was already being sent to the frontend in the SSE `complete` event — it was computed by the orchestrator and stored in `pipeline_result["structured_summary"]` — but the frontend never read it. The `recommended_action` field contains the agent's natural-language reasoning for why it chose a particular supplier and urgency level. This is the most important output of the entire pipeline for an audience: not the draft email, but the decision and its justification. Placing the card above the approval checkboxes means the first thing a viewer reads is the agent's conclusion, with the supporting actions below it.

The border color is keyed to severity so the card is immediately scannable: red for CRITICAL, orange for HIGH. The `if (!summary) return null` guard means the component is safe to include in both `ApprovalGate` and the done state — it renders nothing if `finalize_analysis` was not called or its result was not stored.

### What would happen without this change

The agent's structured recommendation is calculated, stored in `pipeline_result`, transmitted over SSE, and then silently discarded by the frontend. An audience watching the approval gate sees cost numbers and draft emails but no explanation of *why* the agent chose that supplier or rated the situation CRITICAL. The "what it decides and why" part of the demo story is completely missing.

---

## Summary

| Change | Files | Problem solved | Without it |
|--------|-------|---------------|------------|
| Healthy/shocked state patches and apply functions | `mock_data.py` | No visual transition between normal and crisis state | Demo starts mid-crisis; shock cannot be shown happening |
| `/demo/pre-shock` and `/demo/inject-shock` endpoints | `backend/main.py` | No server-side mechanism to switch between states | Frontend cannot trigger or reset the shock without a server restart |
| Pre-shock phase, inject button, and reset flow | `frontend/src/App.jsx` | Shock injection was not wired to the UI; reset returned to mid-demo state | No "before" view; bars don't update immediately on shock; reset doesn't restore healthy baseline |
| `RecommendationCard` component | `frontend/src/App.jsx`, `frontend/src/App.css` | `finalize_analysis` output was computed and transmitted but never displayed | Audience sees draft emails but no explanation of the agent's decision or reasoning |
