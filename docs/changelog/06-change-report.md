# ChainPilot — Change Report 6

## Overview

The Procurement and Risk specialist agents were previously two separate tools that Claude could call one at a time. Even though the Python backend was already capable of running them in parallel, whether that happened depended entirely on Claude choosing to return both tool calls in the same response turn — which it rarely did due to its step-by-step reasoning style. This change makes parallel execution guaranteed rather than optional: the two tools were merged into one tool with boolean flags, so Python always dispatches whichever specialists Claude selects simultaneously.

---

## Change 1: Merge Two Specialist Tools Into One With Boolean Flags

**File changed:** `backend/agent.py`

### What changed

The two entries in `ORCHESTRATOR_TOOLS` — `run_procurement_analysis` and `run_risk_assessment` — were replaced with a single `run_specialist_analysis` tool:

```python
# Before: two separate tools
{
    "name": "run_procurement_analysis",
    "input_schema": {
        "properties": {
            "sku": {"type": "string"},
            "quantity_needed": {"type": "integer"}
        },
        "required": ["sku", "quantity_needed"]
    }
},
{
    "name": "run_risk_assessment",
    "input_schema": {
        "properties": {
            "sku": {"type": "string"}
        },
        "required": ["sku"]
    }
},

# After: one tool with scope flags
{
    "name": "run_specialist_analysis",
    "input_schema": {
        "properties": {
            "sku": {"type": "string"},
            "quantity_needed": {"type": "integer"},
            "run_procurement": {"type": "boolean"},
            "run_risk": {"type": "boolean"}
        },
        "required": ["sku", "quantity_needed", "run_procurement", "run_risk"]
    }
}
```

The orchestrator system prompt was updated to describe the new tool and clarify when to set each flag to False (price spike with healthy inventory → `run_risk=False`). The note about "calling both in the same turn" was removed — it is no longer relevant.

### Why it was made

The previous design had a structural problem: parallelism depended on Claude's internal tool-calling behavior, not on Python execution. Claude can return multiple `tool_use` blocks in one response, which is what would trigger parallel execution. But Claude's step-by-step reasoning style means it tends to make one call, wait for the result, decide on the next call, and so on — particularly when the system prompt doesn't strongly force otherwise. Even with a hint in the system prompt ("you may call both in the same turn"), this was a suggestion Claude could ignore.

The merged tool removes the ambiguity. Claude makes one decision about scope — `run_procurement=True, run_risk=True` for most disruptions — and Python always runs both simultaneously. The agentic behavior is preserved because Claude still decides which specialists are needed based on the disruption type. It just no longer controls *when* they run relative to each other.

### What would happen without this change

Pipeline runtime for standard disruptions (the majority) would be roughly double what it needs to be. Each specialist takes 8–12 seconds. Running them sequentially means 16–24 seconds just for the analysis phase. With the merged tool, both run simultaneously, so the analysis phase takes ~8–12 seconds regardless of how many specialists are selected. In a demo context where a 30-second pipeline is already stretching attention, the difference between 25 seconds and 15 seconds is visible.

Additionally, Claude's choice of whether to call both tools in the same turn is an unpredictable variable — the pipeline could run fast one time and slow the next depending on how Claude structured the response. The merged tool makes runtime consistent.

---

## Change 2: Add Parallel Dispatcher in the Orchestrator

**File changed:** `backend/agent.py`

### What changed

A `_run_parallel_specialists` helper function was added inside `_run_orchestrator`. It is used as the implementation of `run_specialist_analysis` in `orchestrator_tool_map`:

```python
def _run_parallel_specialists(sku, quantity_needed, run_procurement, run_risk):
    if run_procurement:
        log(3, "Running Procurement Agent — finding alternatives...", "run_procurement_analysis")
    if run_risk:
        log(5, "Running Risk Agent — assessing customer exposure...", "run_risk_assessment")
    with concurrent.futures.ThreadPoolExecutor() as pool:
        futures = {}
        if run_procurement:
            futures["procurement"] = pool.submit(
                _run_procurement_agent, sku=sku, quantity_needed=quantity_needed, on_progress=on_progress
            )
        if run_risk:
            futures["risk"] = pool.submit(_run_risk_agent, sku=sku, on_progress=on_progress)
        return {k: f.result() for k, f in futures.items()}
```

The result-merging loop was updated to unpack the combined result dict:

```python
# Before: two separate branches
if block.name == "run_procurement_analysis":
    pipeline_result["cost_analysis"] = specialist_result.get("cost_analysis_raw", {})
    ...
elif block.name == "run_risk_assessment":
    pipeline_result["customer_impact"] = specialist_result.get("customer_impact_raw", {})
    ...

# After: one branch, two keys
if block.name == "run_specialist_analysis":
    if "procurement" in specialist_result:
        p = specialist_result["procurement"]
        pipeline_result["cost_analysis"] = p.get("cost_analysis_raw", {})
        ...
    if "risk" in specialist_result:
        r = specialist_result["risk"]
        pipeline_result["customer_impact"] = r.get("customer_impact_raw", {})
        ...
```

Progress logging inside the helper uses the original tool names (`run_procurement_analysis`, `run_risk_assessment`) so the pipeline tracker in the frontend continues to mark those steps correctly.

### Why it was made

The parallel dispatch logic needed to live inside the tool handler, not in the outer orchestrator loop. The outer loop already handled parallelism when Claude returned multiple tool_use blocks, but now that there is only one tool_use block per turn (a single `run_specialist_analysis` call), parallel execution must happen *within* the handler for that single call.

Defining the helper as a closure inside `_run_orchestrator` gives it access to `on_progress` (for progress logging) and `log` without needing to thread them through as additional parameters.

### What would happen without this change

The merged tool would be wired but would run specialists sequentially (calling `_run_procurement_agent` then `_run_risk_agent` one at a time), defeating the purpose of the merge. The tool interface would be cleaner but performance would be unchanged.

---

## Summary

| Change | File | Problem solved | Without it |
|--------|------|---------------|------------|
| Merge two specialist tools into one with boolean flags | `backend/agent.py` | Whether agents ran in parallel depended on Claude's response structure, not Python execution | Parallel execution unpredictable; may not happen at all depending on Claude's reasoning path |
| Parallel dispatcher inside the merged tool handler | `backend/agent.py` | Parallelism must happen within the single tool call, not across multiple tool_use blocks | Merged tool would still run specialists sequentially, with no performance improvement |
