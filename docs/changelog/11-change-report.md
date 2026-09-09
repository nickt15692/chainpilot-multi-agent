# ChainPilot — Change Report 11

## Overview

This report covers three closely related improvements that together make the demo feel like a real, always-on system instead of a one-shot script:

1. **Real-time data simulation.** The mock data feeds (`mock_data.py`) now drift continuously between polls, so the dashboard shows live-feeling movement even when no shock is active. Stock burns down, prices oscillate around their baseline, and the shocked phase accelerates the decay.
2. **RFQ delivery resolution.** A class of silent-failure bugs in Slack and SMTP delivery — where approvals appeared to succeed but no message was actually sent — was diagnosed and fixed. The "0 RFQs sent" mystery is gone; failures now surface in the UI with the full message body preserved.
3. **Frontend phase-hijack guard.** The monitor SSE channel could overwrite the user's current screen if a *second* shock arrived while the user was still reviewing the first. The `monitor_complete` handler now respects user state and never hijacks the approval gate or the post-execution success view.

Together these three changes make the demo robust to repeated shocks, a flaky email server, or a slow human reviewer — all of which used to break the previous build.

---

## Change 1: Real-Time Mock Data Simulation

**File changed:** `mock_data.py`

### Problem

Until now the mock data was static. `INVENTORY`, `SUPPLIERS`, and `PRICE_FEEDS` were dicts that only ever changed when somebody clicked **Inject Shock** or **Reset**. Between polls the dashboard numbers were frozen. That was fine for a click-through demo, but it betrayed the "always-on monitor" framing: anyone watching the screen for a few seconds before the shock would see no movement at all and conclude that nothing was happening.

### What changed

Two new pieces of state and one new function were added:

```python
# Tracks the current phase so the drift engine knows how to behave.
# Phases: "healthy" → "pre_shock" → "shocked"
_sim_phase: str = "healthy"
_sim_tick: int = 0          # increments every drift_tick() call
```

`apply_pre_shock()` resets the phase to `"healthy"` and zeros the tick counter. `apply_shock()` flips the phase to `"shocked"`. The new `drift_tick()` function is invoked by the monitor loop every `POLL_INTERVAL_SECONDS` (≈ 15 s) and applies small, realistic fluctuations to every SKU:

- **Stock burn.** Each SKU consumes a fraction of its `daily_production_need` per tick, with ±10% random noise. The shocked phase burns 2.5× faster than the healthy phase. `hours_to_stockout` and `stock_pct` are recomputed from the new stock count, so all downstream tools see internally consistent values.
- **Price drift.** In the healthy phase, prices follow a tiny sine wave plus low-amplitude white noise — the curve looks organic without trending. In the shocked phase, the affected SKUs (`SKU-4821`, `SKU-7703`) get a slow upward creep proportional to the tick counter, simulating a tightening market. `current_price` is recomputed from `change_pct` so consumers always see a consistent pair.

### Why this matters

- The dashboard now has visible motion at idle. Stock counters tick down by a few units between polls, prices wobble by a fraction of a percent. To a viewer this reads as "the system is watching live data."
- Because the drift respects the same data structures the tools already read, no tool code had to change. `check_inventory_levels()` simply sees a slightly different number every time it's called.
- Drift is bounded. `change_pct` is clamped to `[-30%, +50%]` and stock can never go negative. The shock injection still produces the same final crisis values regardless of how long the system has been drifting, because `apply_shock()` writes absolute patch values, not deltas.

### What would happen without this change

The dashboard would feel like a screenshot. Reviewers would not understand that the monitor is actually polling — they would assume the "Inject Shock" button is what makes the system run, instead of seeing it as an interventional event on top of an already-running loop.

---

## Change 2: RFQ Delivery Resolution

**Files changed:** `backend/main.py`, `frontend/src/App.jsx`, `frontend/src/App.css`

### Problem

When the user clicked **Execute Selected** on the approval gate, the UI reported success and the pipeline finalized — but no email or Slack message was actually sent. The Sent Items panel showed "0 RFQs sent · Slack skipped" with no error explaining why.

Three distinct bugs were stacked on top of each other:

1. **SMTP handshake.** The original `_send_smtp_email` opened an `smtplib.SMTP` context manager and called `starttls()` directly. Most servers — Mailtrap's sandbox included — require an `EHLO` exchange before TLS can be negotiated. Without it, `starttls()` raised an exception that was swallowed by a bare `except Exception:` block, and the caller got back `{"sent": False}` with no useful detail.
2. **Slack error opacity.** `_send_slack` caught all exceptions but never read the HTTP response body. When Slack rejected a payload (`invalid_payload`, `channel_not_found`), the caller saw only `"HTTP Error 400"` instead of the actual reason. Debugging required `tcpdump`.
3. **Demo-mode false positive.** The original code checked `if not SMTP_SERVER` to decide whether to run in demo mode. When SMTP credentials *were* present but the send failed, it returned `{"sent": False}` and threw away the email body. The user lost both the message they had drafted and the explanation for the failure.

### What changed

Both helpers were rewritten in `backend/main.py`:

- **`_send_smtp_email`.** Now calls `server.ehlo()` before *and* after `starttls()`. On any failure, it returns `{"sent": False, "error": "<exception>", "body": "<full email>", "note": "Email content preserved"}`. The body is never lost.
- **`_send_slack`.** Catches `urllib.error.HTTPError` separately, reads the response body, and surfaces the exact Slack error string. Demo mode (no webhook configured) now returns `{"sent": True, "demo": True}` with the full message content embedded so the UI can render the would-have-been-sent payload.
- **Frontend.** The Sent Items outbox now renders `outbox-error` and `outbox-note` blocks for failed items. A `failed-badge` style distinguishes a transmission failure from a successful demo-mode preview. The email or Slack body is shown in both cases.

### Why this matters

For a demo that brags about *autonomous execution*, having execution silently fail is the worst possible outcome. The fix turns failures into visible, debuggable events that still preserve the agent's work product. A reviewer who configures Mailtrap incorrectly now sees: "RFQ to FastBear — failed to send (SMTPHeloError: 503), but here is the exact email the agent drafted." That is a much better story than "0 RFQs sent."

### What would happen without this change

The pipeline would look broken even though it isn't. Approval would succeed, `/approve` would return 200, and the dashboard would claim execution finished — but nothing would land in any inbox or channel. There would be no log line, no error toast, no preserved draft. To diagnose it you would need to read the source code.

---

## Change 3: Frontend Phase-Hijack Guard

**File changed:** `frontend/src/App.jsx`

### Problem

The `/monitor/stream` SSE endpoint was added in change report 7 to push autonomous-pipeline results to the frontend so the approval gate would appear without anyone clicking a button. That worked perfectly for the *first* shock. But the monitor loop is always on. If the demo presenter triggered a second shock — or if the autonomous monitor itself fired a fresh pipeline while the human was still reading the previous approval gate — the inbound `monitor_complete` event would unconditionally call:

```javascript
setPhase(result.needs_approval ? "approval" : "done");
```

This would *replace* whatever the user was looking at. If they were halfway through reading the Advocate-vs-Skeptic deliberation panel, it would reset to a fresh approval gate for the new event. If they had just executed the previous recommendation and were on the success screen, it would jump them back to a new approval state. Both are user-hostile.

### What changed

The `setPhase` call inside the `monitor_complete` handler now runs through a functional updater that reads the previous phase and refuses to overwrite a phase the user is actively engaged with:

```javascript
monitorSse.addEventListener("monitor_complete", (e) => {
  const data = JSON.parse(e.data);
  const result = data.result;
  if (!result?.success) return;

  setPipelineResult(result);
  setCostExposure(result.cost_exposure || 0);
  setApprovalThreshold(result.approval_threshold || 50000);
  if (result.trust_stats) setTrustStats(result.trust_stats);
  setToolsCalled(result.pipeline_result?.tools_called || TOOLS_ORDER);
  setActiveStep(10);
  setAutoDetected(true);

  // Don't hijack if user is already in approval or done
  setPhase(prev => {
    if (prev === "approval" || prev === "done") return prev;
    return result.needs_approval ? "approval" : "done";
  });
});
```

All the *data* updates (recommendation, cost exposure, trust stats, tools called) still happen — the underlying state stays current. Only the *navigation* (phase transition) is suppressed when the user is mid-task. As soon as they click **Execute Selected** or **Reset Demo**, they will see the most up-to-date state.

### Why this matters

This is the difference between a demo that survives a curious reviewer and one that doesn't. Reviewers *will* click Inject Shock twice in a row. They *will* leave the approval gate open while reading the deliberation panel. The previous behavior produced visible glitches in both scenarios. The fix makes the autonomy invisible when the human is actively engaged, which is exactly what good autonomy should do.

### What would happen without this change

A second shock during the first approval gate would silently replace the entire screen with new content, with no transition or warning — the reviewer would think the app had a bug or that the previous recommendation had been canceled. After execution, the success screen could be replaced by a fresh approval gate before the user had read the success message. Both scenarios looked like instability.

---

## Summary of Modified Files

| File | What changed |
|---|---|
| `mock_data.py` | Added `_sim_phase`, `_sim_tick`, and `drift_tick()`; healthy/shocked drift behavior; price oscillation |
| `backend/main.py` | Rewrote `_send_smtp_email` (EHLO + body preservation) and `_send_slack` (typed errors, demo mode); `monitor_loop` now calls `drift_tick()` each iteration |
| `frontend/src/App.jsx` | `monitor_complete` handler uses functional setter to avoid hijacking `approval` and `done` phases; outbox renders failed-send details with preserved body |
| `frontend/src/App.css` | New styles for `outbox-error`, `outbox-note`, and `failed-badge` |

---

*ChainPilot — autonomous supply chain disruption response*
