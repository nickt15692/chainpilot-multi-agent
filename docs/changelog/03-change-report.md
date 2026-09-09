# ChainPilot — Change Report 3

## Overview

Two categories of change were made: wiring the autonomous monitor loop that was already written but never started, and rewriting the agent system prompts to give Claude real decision-making authority instead of scripted tool sequences. Both changes address the same root problem — the system was described as autonomous and agentic, but neither property was actually true at runtime.

---

## Change 1: Wire the Monitor Loop

**Files changed:** `backend/main.py`

### What changed

Added a FastAPI `lifespan` context manager that starts `monitor_loop()` as a background `asyncio.Task` when the server boots, and cancels it cleanly on shutdown.

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    global _monitor_running
    task = asyncio.create_task(monitor_loop())
    _monitor_running = True
    try:
        yield
    finally:
        task.cancel()
        _monitor_running = False

app = FastAPI(title="ChainPilot API", version="1.0.0", lifespan=lifespan)
```

`monitor_loop` is imported from `backend/monitor.py`, which already existed and was correctly written — it polls `detect_disruptions()` every 15 seconds and triggers the full pipeline for any new threshold breaches it finds.

Also removed the dead `POST /trigger` endpoint (it constructed an event dict and returned it, but was never called by the frontend or anything else) and the unused `HTTPException` import.

### Why it was made

`backend/monitor.py` had been written as the system's autonomy layer — its docstring even says "This is what makes ChainPilot an AUTONOMOUS agent: it acts without being asked." But it was never called from anywhere. `main.py` imported only `get_event_log` and `clear_active_events` from it; the loop itself was dead code. The `_monitor_running` flag was hardcoded to `False` and never updated, so the `/dashboard` endpoint was always reporting the monitor as inactive.

### What would happen without this change

The system would remain manual-only. Every pipeline run requires a user to click the "Simulate Disruption" button in the frontend. The monitor loop would sit in `monitor.py` indefinitely as dead code. The `/dashboard` `monitor_active` field would always return `False`. The system would have no ability to detect or respond to a disruption autonomously — it would be a demo tool, not an autonomous agent.

---

## Change 2: Remove Scripted Tool Sequences from Agent Prompts

**Files changed:** `backend/agent.py`

### What changed

Rewrote the system prompts for all three specialist agents and the orchestrator. Also updated the two user messages that were scripting tool call order alongside the system prompts.

**The pattern that was removed from every prompt:**

> "Execute in order: 1. tool_a() 2. tool_b() 3. tool_c() ... Always call all N tools."

**Replaced with** goal statements, tool descriptions, and reasoning guidance that let Claude decide what to call based on what it observes.

#### Procurement specialist

Before: `"Execute in order: 1. check_supplier_status() 2. check_price_feeds() 3. search_alternative_suppliers() 4. calculate_cost_impact() — Always call all 4 tools."`

After: Tool descriptions with judgment guidance. Claude is told to start with the tool most relevant to the disruption type (supplier status for a delay, price feeds for a price spike), to call `calculate_cost_impact` only once it has a specific alternative in mind, and that it does not need to call every tool if the situation doesn't require it.

#### Risk specialist

Before: `"Execute in order: 1. check_inventory_levels() 2. get_affected_customer_orders() — Always call both tools."`

After: Claude calls `check_inventory_levels` first and then decides whether `get_affected_customer_orders` is worth calling based on what it finds. If stock is healthy (above 50% with more than 168 hours of cover), it may conclude that customer exposure is negligible and skip the second tool.

#### Communications specialist

Before: `"Execute in order: 1. draft_rfq_email() 2. draft_rfq_email() 3. draft_slack_alert() 4. log_disruption_event()"`

After: Claude reads the procurement and risk context it was given and decides how many RFQ emails are warranted, derives severity from the actual `hours_to_stockout` value, and drafts communications accordingly. The Slack alert and audit log are still expected on every run (they're always appropriate), but the number and urgency of RFQ emails adapts to the situation.

#### Orchestrator

Before: `"Call them in this exact order: 1. run_procurement_analysis 2. run_risk_assessment 3. run_communications_draft 4. finalize_analysis — Call all 4 tools in order. Do not skip any."`

After: Claude is given the disruption event and a description of each specialist's purpose, and told to decide which ones are needed. Examples are given to anchor the reasoning: for a critical stockout, run risk assessment early so urgency informs procurement; for a minor price spike with healthy inventory, risk assessment may not add value. `finalize_analysis` is still required as the last step.

#### Orchestrator user message

The user message that triggers the orchestrator also contained an explicit sequence: `"Call run_procurement_analysis, then run_risk_assessment, then run_communications_draft, then finalize_analysis."` This would have overridden the latitude given in the system prompt, so it was replaced with a goal statement: `"Assess this disruption and coordinate the appropriate response."`

#### Communications user message

Same issue: the user message passed to the comms agent included `"Call draft_rfq_email for the recommended_supplier, then draft_slack_alert, then log_disruption_event."` Replaced with context handoff only.

### Why it was made

When a system prompt tells Claude exactly which tools to call in exactly what order, Claude is a script runner. It cannot adapt to what it finds, cannot skip steps that don't apply, and cannot sequence tools differently based on the type or severity of the disruption. The model's reasoning capability is entirely bypassed — it just follows the list.

A price spike disruption and a critical stockout have different information needs. A price spike is primarily a cost question; a stockout is primarily an urgency question. The old prompts ran the same 9-tool sequence regardless. The orchestrator had no basis to treat them differently because it was told to call all three specialists every time.

### What would happen without this change

Claude would call all tools on every pipeline run, in the same fixed order, regardless of disruption type or what it finds along the way. A price spike on a SKU with 3 months of inventory would still trigger a full risk assessment and customer exposure analysis — wasting API calls and adding latency for information that adds no value. The system would correctly describe itself as using Claude, but Claude would have no more decision-making authority than a for-loop. Tool call sequences across different disruption types would be identical, which is the clearest sign that the model is not making decisions.

---

## Summary

| Change | Problem solved | Without it |
|--------|---------------|------------|
| Wire monitor loop in `lifespan` | System required manual trigger for every pipeline run | Monitor loop sits as dead code; system is never autonomous |
| Remove `POST /trigger` endpoint | Dead API surface that was never called | No functional impact, but misleading |
| Rewrite specialist system prompts | Claude followed a fixed script instead of reasoning | All disruption types trigger identical tool sequences; model has no agency |
| Rewrite orchestrator system prompt | Orchestrator always delegated to all three specialists | Every pipeline runs full cost regardless of disruption type or what specialists find |
| Remove scripted user messages | System prompt latitude was overridden by the user message | Claude ignores the judgment guidance and follows the user message sequence instead |
