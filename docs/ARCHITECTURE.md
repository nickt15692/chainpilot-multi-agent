# ChainPilot Architecture

How the system is put together, and why it is built this way.

---

## The problem being modelled

A supply chain disruption is a decision under time pressure with incomplete
information. A supplier slips fourteen days; you have eight hours of stock; the
alternative supplier is faster but 23% more expensive and you have never bought
from them. Somebody has to decide, quickly, and be able to justify the decision
afterwards.

ChainPilot models that as an **autonomous monitoring loop** that detects
disruptions without being asked, runs a **multi-agent analysis** over them, and
stops at a **human approval gate** whose height moves with the system's track
record.

The design goal throughout is *legible autonomy*: the system should act on its
own, but a human should be able to see exactly what it concluded, what the
counter-argument was, and what it did not know.

---

## Component map

| Layer | File | Responsibility |
|---|---|---|
| Watcher | `backend/monitor.py` | Async poll loop; drifts the simulation, detects disruptions, fires pipelines |
| Agents | `backend/agent.py` | Orchestrator + six specialist agents |
| Capabilities | `backend/tools.py` | The ten tool functions agents can call, plus their JSON schemas |
| Autonomy | `backend/trust_engine.py` | Dynamic approval threshold, persisted to `trust_ledger.json` |
| Epistemics | `backend/uncertainty_tracker.py` | Supplier knowledge graph, persisted to `supplier_knowledge.json` |
| Transport | `backend/main.py` | FastAPI app: REST endpoints + two SSE streams |
| Data | `mock_data.py` | Simulated ERP/supplier/price feeds with a live drift engine |
| UI | `frontend/src/App.jsx` | Dashboard, pipeline tracker, deliberation viewer, approval gate |

---

## The agent hierarchy

Six agents, arranged in three tiers. Every one is a separate Claude call with
its own system prompt; none of them share a conversation.

```
                        ┌─────────────────┐
                        │  ORCHESTRATOR   │  decides which specialists to run
                        └────────┬────────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          │                      │                      │
    ┌─────▼──────┐        ┌──────▼─────┐                │
    │ PROCUREMENT│        │    RISK    │   ← run in parallel
    │ Specialist │        │ Specialist │
    └─────┬──────┘        └──────┬─────┘
          └───────────┬──────────┘
                      │  both summaries feed the debate
        ┌─────────────▼─────────────┐
        │  ADVERSARIAL DELIBERATION │
        │  ┌──────────┐ ┌─────────┐ │
        │  │ ADVOCATE │ │ SKEPTIC │ │   ← run in parallel, argue opposite sides
        │  └────┬─────┘ └────┬────┘ │
        │       └─────┬──────┘      │
        │        ┌────▼────┐        │
        │        │ ARBITER │        │   ← reads both, issues the verdict
        │        └────┬────┘        │
        └─────────────┼─────────────┘
                      │
            ┌─────────▼─────────┐
            │  COMMUNICATIONS   │  drafts RFQs, Slack alert, audit log
            │    Specialist     │
            └─────────┬─────────┘
                      │
              finalize_analysis()  → structured recommendation → UI
```

### Tier 1 — Orchestrator

`_run_orchestrator()` in `backend/agent.py`. It is a tool-use loop (max 10
turns) where the "tools" are the other agents. It never touches inventory data
directly; it decides *who to ask* and *in what order*.

Its system prompt is assembled at runtime by `_build_orchestrator_prompt()` from
two moving parts:

- **Disruption-type rules** (`_ORCH_RULES_BY_TYPE`) — a `price_spike` needs
  different reasoning than a `low_stock`, so the prompt swaps in rules specific
  to the triggering event type.
- **The current trust threshold** — injected as a number, so the orchestrator
  knows what dollar figure requires human sign-off *on this run*.

Its four meta-tools are `run_specialist_analysis`, `run_adversarial_deliberation`,
`run_communications_draft`, and `finalize_analysis`. When Claude returns several
`tool_use` blocks in one turn, they are executed concurrently in a thread pool.

### Tier 2 — Domain specialists

Both are given a narrow tool subset and are asked to return **only JSON**, so
their output is machine-consumable by the next tier.

- **Procurement Specialist** — supplier status, price feeds, alternative
  suppliers, cost impact. Returns recommended supplier, unit cost, 30-day
  exposure, switching premium.
- **Risk Specialist** — inventory levels, affected customer orders. Returns
  hours-to-stockout, customers affected, penalty exposure, top customer.

They run simultaneously because neither depends on the other's output. Note that
both prompts explicitly grant *discretion to skip tools* — the Risk Specialist is
told it may skip customer-order lookup when stock is healthy. The agents are not
required to fan out over every tool available to them.

### Tier 3 — Adversarial deliberation

This is the part that makes ChainPilot more than a pipeline. See below.

### Communications Specialist

Receives both specialist summaries and drafts the outbound artefacts. Everything
is drafted with `draft_only=True` — **no agent can send anything**. Transmission
happens only after a human approves, through `POST /approve`.

---

## Feature 1 — Adversarial deliberation

`_run_adversarial_deliberation()` in `backend/agent.py`.

Most agent systems produce a single confident recommendation. The failure mode is
that the reasoning that *didn't* happen is invisible: you cannot tell whether the
agent considered the downside and rejected it, or never considered it.

ChainPilot forces the argument into the open by running three agents:

| Agent | Prompt instruction | Output |
|---|---|---|
| **Advocate** | Argue FOR switching immediately. "Do NOT hedge. You are an advocate." | 3–5 bullets + `VERDICT: Act immediately — …` |
| **Skeptic** | Argue AGAINST switching immediately. "Do NOT hedge. You are a skeptic." | 3–5 bullets + `VERDICT: Wait / proceed cautiously — …` |
| **Arbiter** | Weigh both; acknowledge the strongest point from each side | Structured JSON verdict |

The Advocate and Skeptic run **in parallel** and never see each other's
arguments. This matters: they are producing independent strongest-cases, not
converging on a compromise. Only the Arbiter reads both.

The Arbiter returns:

```json
{
  "arbiter_recommendation": "...",
  "strongest_advocate_point": "...",
  "strongest_skeptic_point": "...",
  "swing_condition": "...",
  "confidence": "HIGH | MEDIUM | LOW",
  "final_action": "immediate_switch | partial_order | wait_and_monitor | emergency_spot_buy"
}
```

The field worth dwelling on is **`swing_condition`** — the one thing that would
need to be true for the losing side to win. It converts a verdict into a
*falsifiable* verdict. A human reading "switch now, unless the preferred supplier
confirms a revised ETA under 5 days" knows precisely what to go check. A human
reading "switch now, confidence HIGH" does not.

`confidence` is defined relative to the debate, not to the model's own certainty:
HIGH means the Advocate clearly won, MEDIUM means it was close, LOW means genuine
unresolved uncertainty.

The full debate — both arguments and the synthesis — is surfaced in the UI's
`DeliberationPanel`, not just the conclusion.

---

## Feature 2 — Confidence-calibrated autonomy

`backend/trust_engine.py`, persisted to `trust_ledger.json`.

A fixed "auto-execute below $50K" rule is arbitrary and static: it does not care
whether the system has been right forty times or wrong four times. ChainPilot
makes the threshold a function of track record.

```
start                   $50,000
good outcome            +$8,000     (floor $10,000)
bad outcome            −$20,000     (ceiling $200,000)
```

Two properties are deliberate:

- **Asymmetry.** A bad outcome costs 2.5× what a good outcome earns. Trust should
  be slow to build and quick to lose — an agent that gets ten routine calls right
  has not earned the right to make one catastrophic call unsupervised.
- **Hard bounds.** Below `$10,000` there is always *some* oversight; above
  `$200,000` there is always human sign-off, regardless of how good the record is.
  No accumulation of good outcomes can unlock unlimited autonomy.

The threshold is read at the *start* of every pipeline run and injected into the
orchestrator's system prompt, so the agent knows the standard it is being held to
while it reasons. Humans close the loop with `POST /trust/outcome` (the
`OutcomeFeedback` widget's ✓/✗ buttons), and the Trust Meter updates live.

All writes are guarded by a `threading.Lock` because the monitor loop and request
handlers touch the ledger from different threads.

---

## Feature 3 — Uncertainty modelling

`backend/uncertainty_tracker.py`, persisted to `supplier_knowledge.json`.

The system is asked to recommend suppliers it has often never transacted with.
Rather than let that gap disappear behind a confident recommendation, it is
tracked explicitly and surfaced at the approval gate.

Every supplier carries an uncertainty level — `HIGH` (the default for anyone
unseen), `MEDIUM`, or `LOW` — plus a list of specific `known_gaps`, e.g.
*"No quality history on file"*, *"Delivery performance unknown"*.

Uncertainty decays through two channels:

1. **Automatically**, as interactions accumulate. `record_rfq_sent()` logs each
   RFQ and drops the "no past RFQ responses" gap after first contact.
2. **Manually**, through human annotation. `POST /knowledge/annotate` lets a
   procurement person write a quality note and set the level directly — setting
   `LOW` clears all gaps.

`assess_recommendation_uncertainty()` runs after the Procurement Specialist
returns and evaluates both the recommended supplier *and* every alternative that
was considered, so the gate can show "we picked A, but we know nothing about
A, B, or C either."

The result is a caveat attached to the recommendation and an `UncertaintyBadge`
rendered in the approval gate. The design claim: an agent that says
*"switch to FastBear — but I have no quality history on them"* is more useful than
one that just says *"switch to FastBear."*

---

## Execution model

### The autonomous loop

`monitor_loop()` polls every `POLL_INTERVAL_SECONDS` (default 15s). Each tick:

1. `mock_data.drift_tick()` — nudges inventory and prices so the dashboard is
   never frozen between polls.
2. `detect_disruptions()` — threshold checks across three feeds:
   - stock below `STOCK_THRESHOLD_PERCENT` (20%) → `low_stock`
   - supplier delay ≥ `SUPPLIER_DELAY_DAYS` (3) → `supplier_delay`
   - price above `PRICE_SPIKE_THRESHOLD` (15%) over 30-day baseline → `price_spike`
3. Deduplicate against `_active_events` — keyed on `type:sku`, so a persistent
   disruption fires one pipeline, not one per poll.
4. Dispatch new events to the pipeline in a thread pool, so a multi-minute
   analysis never blocks the next poll.

This loop starts with the FastAPI app via its `lifespan` handler. It runs whether
or not a browser is open — that is what makes the system autonomous rather than
request-driven.

### Concurrency

Three layers of parallelism, all `ThreadPoolExecutor` over blocking SDK calls:

- Procurement ‖ Risk
- Advocate ‖ Skeptic
- Any orchestrator turn returning multiple `tool_use` blocks

Shared state is protected by locks in both persistence modules.

### Resilience

- `_create_with_retry()` wraps every Claude call with exponential backoff on
  `RateLimitError` — 15s doubling to a 60s cap, 5 attempts. With six agents and
  three parallel fan-outs, rate limits are a routine condition, not an exception.
- Every system prompt is sent with `cache_control: ephemeral`. Prompts are long
  and static across runs, so caching cuts both cost and latency materially.
- The orchestrator loop is bounded at 10 turns; exceeding it marks the result
  `incomplete` with a reason rather than looping forever.
- Arbiter JSON that fails to parse degrades to a MEDIUM-confidence result built
  from the raw text instead of crashing the pipeline.

### Streaming to the UI

Two Server-Sent Event streams:

- `GET /monitor/stream` — disruptions as the autonomous monitor finds them.
- `GET /pipeline/stream/{sku}` — step-by-step progress for one pipeline run.

Progress is threaded through the agents as an `on_progress(step, msg, tool,
detail)` callback, which is why the UI can show *"Adversarial deliberation —
Advocate arguing for action…"* while it is happening rather than after.

---

## The human approval gate

The one invariant worth stating plainly: **agents draft, humans send.**

Every communication tool takes `draft_only=True`. Nothing leaves the system
during analysis. After the pipeline completes, `ApprovalGate` presents the drafts
with per-item checkboxes; `POST /approve` is the only path to real SMTP or Slack
transmission, and it only sends the items that were explicitly approved.

When credentials are absent from `.env`, the system runs in **demo mode** —
approved content is rendered in the UI rather than transmitted. Failures during
real sending surface the error in the UI and preserve the message content rather
than silently dropping it.

Whether the gate is *mandatory* depends on the trust threshold: exposure at or
above the current threshold forces human approval. Below it, the system is
permitted to proceed on its own.

---

## Extending the system

**Real data.** Replace `mock_data.py` with live ERP/supplier-portal calls. The
tool functions in `backend/tools.py` are the seam — keep their return shapes and
nothing upstream changes.

**A new specialist.** Add its system prompt and runner function in
`backend/agent.py`, add a meta-tool schema to `ORCHESTRATOR_TOOLS`, register it in
`orchestrator_tool_map`, and give it a step number in `orch_step_map` so progress
streams to the UI.

**A new disruption type.** Add detection to `detect_disruptions()` and a rules
entry to `_ORCH_RULES_BY_TYPE` so the orchestrator reasons about it correctly.

**Retuning autonomy.** The constants at the top of `backend/trust_engine.py`
(`_GOOD_OUTCOME_BOOST`, `_BAD_OUTCOME_PENALTY`, and the bounds) are the entire
policy.

---

*ChainPilot — autonomous supply chain disruption response*
