# ChainPilot — Multi-Agent Architecture Change Report

## Overview

The single monolithic agentic loop was replaced with an orchestrator + three specialist agents pattern. Previously, one Claude instance handled everything — supplier lookup, cost analysis, risk assessment, and drafting communications — in a single 25-turn conversation with all 9 tools available. Now, an orchestrator agent coordinates three specialists, each with a focused tool set and system prompt, and delegates work to them as sub-calls.

The external interface (`run_disruption_pipeline` signature, `pipeline_result` structure, `on_progress` callback) is unchanged. `main.py`, `monitor.py`, and the frontend required zero changes.

---

## Change 1 — Tool subset lists in `tools.py`

**File:** `backend/tools.py`

**What changed:** Three named subsets were appended after `TOOL_MAP`:

```python
PROCUREMENT_TOOLS = [t for t in TOOLS if t["name"] in {
    "check_supplier_status", "check_price_feeds",
    "search_alternative_suppliers", "calculate_cost_impact",
}]

RISK_TOOLS = [t for t in TOOLS if t["name"] in {
    "check_inventory_levels", "get_affected_customer_orders",
}]

COMMS_TOOLS = [t for t in TOOLS if t["name"] in {
    "draft_rfq_email", "draft_slack_alert", "log_disruption_event",
}]
```

No existing code was touched. `TOOLS` and `TOOL_MAP` remain exactly as-is.

**Why:** Each specialist agent should only see the tools relevant to its job. A procurement agent that can also call `draft_rfq_email` might use it prematurely, before the risk assessment context is available. Restricting each agent's tool set enforces the separation of concerns and prevents Claude from taking shortcuts across role boundaries. Filtering from the existing `TOOLS` list avoids duplicating schema definitions.

**Impact if absent:** Every specialist agent would need the full `TOOLS` list passed to it, giving each Claude instance access to all 10 tools regardless of role. The procurement agent could draft Slack messages; the communications agent could re-run cost calculations. This undermines the specialist/orchestrator split entirely — Claude would likely use whatever tool gets to the answer fastest, collapsing the multi-agent pattern back into a single-agent one.

---

## Change 2 — Specialist system prompts

**File:** `backend/agent.py`

**What changed:** Three module-level constants were added — `_SYSTEM_PROCUREMENT`, `_SYSTEM_RISK`, and `_SYSTEM_COMMS` — each a focused system prompt for one specialist role.

- `_SYSTEM_PROCUREMENT` instructs the agent to call its 4 tools in order (supplier status → price feeds → alt search → cost calc) and respond with a JSON summary containing `recommended_supplier`, `alt_unit_cost`, `estimated_30d_exposure`, and `switching_premium`.
- `_SYSTEM_RISK` instructs the agent to call its 2 tools (inventory levels → customer orders) and respond with a JSON summary containing `hours_to_stockout`, `customers_affected`, `total_orders_at_risk`, and `7day_penalty_exposure`.
- `_SYSTEM_COMMS` instructs the agent to use the procurement and risk context it receives to draft RFQ emails, a Slack alert, and an audit log entry — in that order, all with `draft_only=True`.

**Why:** Without a focused system prompt, a Claude instance with only 2–4 tools available would still need to infer its entire role from the user message alone, which is brittle and verbose. A dedicated system prompt establishes role identity, expected output format, and tool execution order once — at the system level where it is cached — rather than repeating those instructions in every user message. The prompts are also intentionally narrow: they name exactly which tools to call and in what sequence, rather than leaving ordering to the model's discretion.

**Impact if absent:** Each specialist would need its full role instructions injected into the user message on every pipeline run, re-billing those tokens on every API call (no caching benefit). More critically, without a system-level mandate on tool order and output format, the specialists may call tools out of sequence or produce inconsistently structured summaries that the orchestrator cannot parse reliably.

---

## Change 3 — Orchestrator system prompt and per-type decision rules

**File:** `backend/agent.py`

**What changed:** A `_SYSTEM_ORCHESTRATOR_TEMPLATE` string was added, containing a `{disruption_type_rules}` placeholder filled at runtime by `_build_orchestrator_prompt(disruption_type)`. An `_ORCH_RULES_BY_TYPE` dict provides the three type-specific rule blocks:

- `low_stock`: prioritise fastest lead time; use CRITICAL if < 24 hours
- `supplier_delay`: compare delay_days vs. stock cover days; calculate switching break-even
- `price_spike`: focus on 30-day cost exposure; RFQ only if alternative is materially cheaper

The orchestrator system prompt tells Claude exactly which 4 meta-tools to call in which order, and mandates `finalize_analysis()` as the final step.

**Why:** The orchestrator needs to know the decision rules for each disruption type so it can correctly classify severity and set `needs_human_approval` when it calls `finalize_analysis`. But those rules must not live in the specialist prompts — the procurement agent shouldn't be making severity decisions, that belongs to the orchestrator. Injecting them via a template placeholder keeps the rules where they logically belong (at the coordination layer) while still enabling prompt caching on the stable template text. Extracting per-type rules from the previous `PLAYBOOKS` dict into a shorter decision-rules format removes the step-by-step tool instructions that are now redundant (the specialists handle that ordering themselves).

**Impact if absent:** The orchestrator has no disruption-type context when calling `finalize_analysis`. It classifies all disruptions using the same generic logic regardless of whether it's a 13-hour stockout or a 15% price spike, producing incorrect severity labels and wrong `needs_human_approval` decisions.

---

## Change 4 — `ORCHESTRATOR_TOOLS` list

**File:** `backend/agent.py`

**What changed:** A new `ORCHESTRATOR_TOOLS` list was defined at module level containing schemas for four tools: `run_procurement_analysis`, `run_risk_assessment`, `run_communications_draft`, and `finalize_analysis` (the last one is referenced directly from the existing `TOOLS` list to avoid duplication). These are the only tools the orchestrator agent sees.

**Why:** The orchestrator's job is to coordinate specialists and synthesize findings — it should not be calling raw tools like `check_inventory_levels` or `draft_rfq_email` directly. Giving it only meta-tool schemas enforces this boundary. Each meta-tool schema also describes what the specialist returns, so Claude can reason about what data will be available to pass to subsequent specialists (e.g., knowing it needs to pass `procurement_summary` and `risk_summary` to `run_communications_draft` before calling it).

**Impact if absent:** There is no way to configure the orchestrator as a Claude agent — `client.messages.create()` requires a `tools` list. Without a dedicated orchestrator tool list, the orchestrator would either use the full `TOOLS` list (bypassing all specialist logic) or have no tools at all (unable to delegate).

---

## Change 5 — `_run_procurement_agent()` specialist function

**File:** `backend/agent.py`

**What changed:** A new function `_run_procurement_agent(sku, quantity_needed, on_progress=None)` was added. It is a 10-turn-max agentic loop that:
- Sends a focused user message asking for procurement analysis for the given SKU and quantity
- Uses `PROCUREMENT_TOOLS` and `_SYSTEM_PROCUREMENT` as its system prompt with prompt caching
- Captures the `calculate_cost_impact` result into `result["cost_analysis_raw"]`
- Captures the top alternative supplier name and unit cost from `search_alternative_suppliers`
- Fires `on_progress(4, "Finding alternative suppliers...", "search_alternative_suppliers")` when that tool is called
- Parses the agent's final `end_turn` JSON summary and merges it into the result dict
- Falls back to the captured raw tool results if JSON parsing fails
- Returns a result dict consumed by the orchestrator

**Why:** Isolating procurement logic into its own function and conversation thread means the procurement agent's context window contains only the supplier/cost data it needs — not customer order data, not draft emails, not the full event history. A smaller, focused context leads to more reliable tool-calling behavior and faster convergence. The 10-turn limit is sufficient because the agent has only 4 tools and a clear completion criterion (a JSON summary).

**Impact if absent:** The orchestrator has no way to delegate procurement work to a specialist. The `run_procurement_analysis` meta-tool in `ORCHESTRATOR_TOOLS` would have no implementation to call, causing a `KeyError` in `orchestrator_tool_map` and a failed pipeline.

---

## Change 6 — `_run_risk_agent()` specialist function

**File:** `backend/agent.py`

**What changed:** A new function `_run_risk_agent(sku, on_progress=None)` was added following the same mini-loop pattern. It uses `RISK_TOOLS` and `_SYSTEM_RISK`, captures the `get_affected_customer_orders` raw result into `result["customer_impact_raw"]`, fires `on_progress(6, ...)`, and returns the risk summary dict.

**Why:** Same rationale as the procurement agent — focused context, smaller tool set, faster convergence. The risk agent's conversation never needs to know about suppliers or emails; it only needs to answer two questions: how long until stockout, and which customers are affected.

**Impact if absent:** Same as above — the `run_risk_assessment` meta-tool has no implementation. The orchestrator cannot assess customer exposure or hours-to-stockout, so it cannot classify severity accurately or provide meaningful data to the communications agent.

---

## Change 7 — `_run_communications_agent()` specialist function

**File:** `backend/agent.py`

**What changed:** A new function `_run_communications_agent(sku, event_id, procurement_summary, risk_summary, on_progress=None)` was added. It differs from the other two specialists in one key way: it receives the full JSON result strings from both the procurement and risk agents injected into its first user message. This gives the communications agent the context it needs — recommended supplier name, cost exposure, hours-to-stockout, affected customers — to draft accurate emails, Slack messages, and audit log entries.

The agent accumulates RFQ emails into a list (to support drafting to two suppliers), captures the Slack draft dict, and captures the audit log. It fires `on_progress(8, ...)` for both `draft_rfq_email` and `draft_slack_alert` calls.

**Why:** Communications cannot be drafted in a vacuum — the RFQ email needs to name the right supplier, the Slack alert needs the correct cost exposure and severity, and the audit log needs a meaningful action list. Rather than re-running all the procurement and risk tools a second time inside this agent, the orchestrator passes the previously computed results as input context. This is the key data-passing mechanism of the multi-agent pattern: each agent's output becomes the next agent's input.

**Impact if absent:** The `run_communications_draft` meta-tool has no implementation. No RFQ emails, Slack alerts, or audit logs are ever drafted — `pipeline_result["drafts"]` remains empty, and the approval gate in the frontend has nothing to display or send.

---

## Change 8 — `_run_orchestrator()` function

**File:** `backend/agent.py`

**What changed:** A new function `_run_orchestrator(trigger_event, event_id, system_prompt, on_progress=None)` was added. It is an 8-turn-max agentic loop that:
- Uses `ORCHESTRATOR_TOOLS` (the four meta-tools)
- Closes over `on_progress` when building its `orchestrator_tool_map`, so all specialist agents receive the same callback
- Maps each meta-tool call to a progress step (3, 5, 7, 9) via `orch_step_map`
- After each specialist call, merges the result into `pipeline_result` at the right key (`cost_analysis`, `customer_impact`, `drafts`, `structured_summary`)
- Accumulates all tool names from all specialist runs into one flat `tools_called` list (so the frontend tracker still shows individual tool completions)
- Detects `incomplete` state if `end_turn` arrives before `finalize_analysis` is called, or if the 8-turn limit is exhausted

**Why:** The orchestrator is the coordination layer — it doesn't do the work itself, it decides what work needs to be done and in what order, then assembles the results. The `on_progress` closure threading is critical: it means specialist agents can fire their own step callbacks (4, 6, 8) directly, giving the frontend a continuous 1-through-10 progress sequence even though it crosses three separate agent conversations. Merging results into `pipeline_result` inside the orchestrator keeps all assembly logic in one place rather than scattered across the specialist functions.

**Impact if absent:** There is no coordination layer. The three specialist functions exist but are never called in sequence, results are never merged into `pipeline_result`, and `run_disruption_pipeline` has nothing to delegate to.

---

## Change 9 — Updated `run_disruption_pipeline()` public function

**File:** `backend/agent.py`

**What changed:** The function body was replaced with a thin wrapper that:
1. Generates the event ID
2. Fires progress steps 1 and 2
3. Selects the orchestrator system prompt via `_build_orchestrator_prompt(disruption_type)`
4. Calls `_run_orchestrator(...)` and receives the full `pipeline_result`
5. Computes `needs_approval` from `cost_analysis.estimated_30d_exposure` exactly as before
6. Returns the same dict shape as the previous implementation

The function signature, return structure, and `on_progress` behavior are identical to before. `main.py`, `monitor.py`, and `detect_disruptions()` required no changes.

**Why:** The public interface is the contract between the agent layer and everything above it (the API server, the monitor loop, the SSE stream). Keeping it unchanged means the multi-agent refactor is entirely internal — it can be validated and rolled back without touching any other layer. The wrapper pattern also keeps the public function readable: its job is setup and teardown, not the mechanics of multi-agent coordination.

**Impact if absent:** Without this wrapper, there is no entry point that assembles the orchestrator and returns a result in the shape the rest of the system expects. `main.py` would call a function that no longer exists or returns the wrong structure.

---

## Turn Limits and API Call Budget

| Agent | Turn limit | Typical turns |
|-------|-----------|--------------|
| Orchestrator | 8 | 4–5 |
| Procurement Specialist | 10 | 5–6 |
| Risk Specialist | 10 | 3–4 |
| Communications Specialist | 10 | 5–6 |
| **Total max** | **38** | **~18–21** |

The previous single-loop maximum was 25 turns. The realistic new ceiling is lower because each specialist converges faster with a smaller tool set and a tightly scoped task.

---

## Summary Table

| File | Change |
|------|--------|
| `backend/tools.py` | Added `PROCUREMENT_TOOLS`, `RISK_TOOLS`, `COMMS_TOOLS` subsets after `TOOL_MAP` |
| `backend/agent.py` | Added `_SYSTEM_PROCUREMENT`, `_SYSTEM_RISK`, `_SYSTEM_COMMS`, `_SYSTEM_ORCHESTRATOR_TEMPLATE`, `_ORCH_RULES_BY_TYPE`, `ORCHESTRATOR_TOOLS`; added `_run_procurement_agent`, `_run_risk_agent`, `_run_communications_agent`, `_run_orchestrator`, `_build_orchestrator_prompt`; replaced `run_disruption_pipeline` body with thin wrapper |
| `backend/main.py` | No changes |
| `backend/monitor.py` | No changes |
| `frontend/src/App.jsx` | No changes |

---

## Non-Technical Summary

Previously, ChainPilot worked like one very capable employee handling an emergency entirely on their own — checking suppliers, calculating costs, assessing customer risk, writing emails, and filing paperwork, all in a single unbroken stream of thought.

The new version works like a small team. When a disruption is detected, a manager (the orchestrator) takes charge and assigns work to three specialists:

- The **Procurement Specialist** contacts supplier records and figures out who can fill the order and at what cost.
- The **Risk Analyst** checks how long production can run before running out and which customers will be affected.
- The **Communications Specialist** takes the findings from both and writes the RFQ emails, the Slack alert, and the audit record.

The manager reviews all three specialists' work and writes the final report.

Each specialist only has access to the information and tools relevant to their job — the Procurement Specialist can't accidentally draft emails, and the Communications Specialist can't re-run cost calculations. The manager decides the order of work and makes the final judgment calls (severity, whether human approval is needed).

From the outside — from the perspective of the dashboard and approval gate — the system works identically to before. The same progress bar fills, the same approval screen appears, the same emails and Slack messages are drafted. The difference is entirely internal: the reasoning is now distributed across focused experts rather than concentrated in one generalist, which makes each step more reliable and the overall system easier to extend.