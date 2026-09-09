# ChainPilot — Change Report 10

## Overview

Three non-obvious features were added to transform ChainPilot from a competent agentic demo into something that genuinely rethinks how autonomous AI systems should reason and earn trust. Additionally, a persistent Slack and email delivery bug was diagnosed and fixed. Each feature addresses a real critique of the original architecture.

---

## Fix: Slack and Email Delivery

**Files changed:** `backend/main.py`

### Problem

Two related bugs caused Slack and email delivery to silently fail:

1. **Email (SMTP)**: The original `_send_smtp_email` used a bare `smtplib.SMTP` context manager without calling `ehlo()` before `starttls()`. Most SMTP servers (including Mailtrap's sandbox) require the EHLO handshake before accepting TLS negotiation. The result was a `SMTPHeloError` or silent connection drop that Python swallowed.

2. **Slack**: The webhook URL in `.env` was a real webhook URL, but the original code had no error detail on failure — it caught all exceptions and returned `{"sent": False, "error": str(e)}` with no access to the HTTP response body. When Slack returns `invalid_payload` or `channel_not_found`, the caller had no way to diagnose it.

3. **Demo mode detection**: The original code checked `if not SMTP_SERVER` to detect demo mode, but when SMTP credentials were present and the send failed, it returned `{"sent": False}` with no preserved content. The email body was lost.

### What changed

Both functions were rewritten in `backend/main.py`:

- `_send_smtp_email`: Now calls `server.ehlo()` before and after `starttls()`. On failure, returns `{"sent": False, "error": ..., "body": ..., "note": "..."}` so the email content is always preserved and visible in the UI regardless of transmission outcome.

- `_send_slack`: Now catches `urllib.error.HTTPError` separately from generic exceptions, reads the response body, and surfaces the exact Slack error string. Demo mode (no webhook configured) returns `{"sent": True, "demo": True}` with the full message content preserved.

- **Frontend**: The Sent Items outbox now renders `outbox-error` and `outbox-note` for failed items, and the email/Slack content is always shown even when transmission fails. A `failed-badge` style distinguishes failed sends from successful ones.

### What would happen without this change

Slack and email would appear to execute (checkboxes submitted, `/approve` returns 200) but nothing would actually be sent or shown. The demo would show "0 RFQs sent · Slack skipped" with no explanation.

---

## Feature 1: Adversarial Deliberation

**Files changed:** `backend/agent.py`, `frontend/src/App.jsx`, `frontend/src/App.css`

### What it is

Instead of the agent producing one recommendation, it now forces two sub-agents to argue opposite sides of the decision before a third agent synthesizes them:

- **Advocate**: Argues for switching suppliers immediately. Focuses on stockout risk, customer penalties, and lead time advantage.
- **Skeptic**: Argues for caution. Focuses on unknown quality risk of alternatives, whether the cost premium is justified, and whether the preferred supplier might recover faster than modeled.
- **Arbiter**: Reads both arguments, identifies the strongest point from each side, produces a final recommendation, and assigns a confidence level (`HIGH`, `MEDIUM`, `LOW`) with a `swing_condition` — the one thing that would change the verdict.

### Architecture

The Advocate and Skeptic run in parallel via `ThreadPoolExecutor`. The Arbiter runs after both complete. This adds approximately 5–10 seconds to pipeline time but produces a qualitatively richer output.

The Orchestrator now has a `run_adversarial_deliberation` meta-tool in its schema, which it is instructed to always call after `run_specialist_analysis` and before `run_communications_draft`. The pipeline step is visible in the Pipeline Tracker as "Advocate ⚔ Skeptic debate."

```
Orchestrator
 ├── run_specialist_analysis (parallel Procurement + Risk)
 ├── run_adversarial_deliberation (parallel Advocate + Skeptic → Arbiter)
 ├── run_communications_draft (Communications agent)
 └── finalize_analysis
```

### Why it is non-obvious

Most multi-agent systems build agents that cooperate toward a shared goal. Adversarial deliberation is the opposite: agents are explicitly prompted to disagree, with different priors and different objectives. This is closer to how real procurement decisions are made (finance vs. operations vs. supply chain all push in different directions) and produces a fundamentally different kind of output — one that shows the human *why* the decision is uncertain rather than just what the decision is.

The `swing_condition` field is particularly useful for a live demo: it forces the Arbiter to articulate *what would need to be true for the opposing view to win*, which is exactly the kind of structured uncertainty that distinguishes serious decision support from a magic 8-ball.

### Frontend

The `DeliberationPanel` component renders the debate in a collapsible card showing:
- Arbiter verdict and `final_action` (`immediate_switch` / `partial_order` / `wait_and_monitor` / `emergency_spot_buy`)
- Confidence badge
- Strongest point from each side
- `swing_condition` — the sentence that would flip the verdict
- Expandable `<details>` blocks with full arguments from Advocate and Skeptic

---

## Feature 2: Confidence-Calibrated Autonomy Threshold

**Files changed:** `backend/trust_engine.py` (new), `backend/agent.py`, `backend/main.py`, `config.py`, `frontend/src/App.jsx`, `frontend/src/App.css`

### What it is

The approval threshold is no longer a hardcoded `$50,000` constant. It is a dynamic value managed by `backend/trust_engine.py` and stored in `trust_ledger.json`. The agent earns the right to act autonomously by making recommendations that humans validate as good; it loses autonomy after bad outcomes.

**Rules:**
- Starting threshold: `$50,000`
- Per good outcome: threshold rises `+$8,000` (up to `$200,000` maximum)
- Per bad outcome: threshold drops `-$20,000` (down to `$10,000` minimum)
- The asymmetry is intentional: bad outcomes cost more than good ones earn, because the cost of a wrong autonomous action exceeds the cost of an unnecessary approval gate.

### Storage

`trust_ledger.json` persists across server restarts. Each entry in `history` records: event ID, SKU, recommended supplier, cost exposure, outcome, threshold before and after, delta description, and timestamp.

### New API endpoints

| Endpoint | Method | What it does |
|---|---|---|
| `GET /trust` | GET | Return current trust stats and threshold |
| `POST /trust/outcome` | POST | Record outcome: `{event_id, sku, outcome, notes}` |
| `POST /trust/reset` | POST | Reset to baseline $50,000 |

### Frontend

The **Trust Meter** on the left panel shows:
- A horizontal bar ranging from `$10K` (minimum, cautious) to `$200K` (maximum, trusted)
- A vertical marker at the `$50K` baseline for reference
- The current auto-execute threshold, color-coded (green = trusted, yellow = moderate, red = cautious)
- Past decision count, good/bad split, and accuracy percentage
- "No past decisions yet — starting at baseline" for a fresh run

After a pipeline executes and actions are approved, the **Outcome Feedback** widget appears:
- Three buttons: ✓ Good call / ~ Neutral / ✗ Bad call
- Optional note field
- Submits to `POST /trust/outcome`
- Updates the Trust Meter live after submission

### Why it is non-obvious

Every existing agentic system uses either a fixed threshold or no threshold at all (fully autonomous). Neither is right: a fixed threshold treats all agents as equally trustworthy regardless of track record, and full autonomy is obviously dangerous for financial decisions.

The non-obvious insight is that the threshold should be a *function of the agent's past performance* — and that this function should be asymmetric, because trust is hard to earn and easy to lose. This is how trust actually works between people in organizations. An agent that has made five good calls in a row has earned the right to act on more expensive decisions without asking; an agent that made one catastrophic call should be watched more closely for a while.

---

## Feature 3: Agent Uncertainty Modeling

**Files changed:** `backend/uncertainty_tracker.py` (new), `backend/agent.py`, `backend/main.py`, `frontend/src/App.jsx`, `frontend/src/App.css`

### What it is

The agent explicitly tracks what it does not know about the suppliers it recommends. Instead of producing a recommendation with equal confidence regardless of whether it has seen a supplier before, it now:

1. Checks `supplier_knowledge.json` for past interaction history with the recommended supplier
2. Attaches an `uncertainty` assessment to the pipeline result: `HIGH`, `MEDIUM`, or `LOW`
3. Lists specific `known_gaps` ("No quality history on file", "Delivery performance unknown")
4. Produces a `recommendation_caveat` that is surfaced in the UI

As the system is used over time, uncertainty decreases:
- First time an RFQ is sent to a supplier → recorded in the knowledge base (first data point)
- Human annotates a supplier via `POST /knowledge/annotate` → uncertainty drops
- After human marks a supplier as `LOW` uncertainty → gaps are cleared

### Storage

`supplier_knowledge.json` persists across restarts. Each supplier entry contains: uncertainty level, interaction count, quality notes (with timestamps), known gaps, and RFQ history.

### New API endpoints

| Endpoint | Method | What it does |
|---|---|---|
| `GET /knowledge` | GET | Return full supplier knowledge base |
| `POST /knowledge/annotate` | POST | `{supplier_name, quality_note, uncertainty}` |
| `POST /knowledge/reset` | POST | Reset for demo/testing |

### Frontend

The `UncertaintyBadge` component appears in the Approval Gate when `uncertainty_level` is `HIGH` or `MEDIUM`. It shows:
- Uncertainty level badge (color-coded red/yellow/green)
- Bulleted list of specific knowledge gaps
- Recommendation caveat text

The badge is deliberately only shown when uncertainty is non-trivial — `LOW` uncertainty means the agent knows what it's doing and no warning is needed.

### Why it is non-obvious

Standard agentic systems treat their recommendations as equally valid regardless of how much evidence supports them. A system that recommends FastBear Inc. — a supplier it has never worked with — with the same confidence as one it has used fifty times is making an epistemically dishonest claim.

The non-obvious move is to make the agent *model its own ignorance* and surface that explicitly to the human decision-maker. This is the difference between a system that produces outputs and a system that helps humans reason. The human reviewing the Approval Gate now sees not just "recommend FastBear" but "recommend FastBear — HIGH uncertainty, no quality history on file" — which is exactly what they need to make a good decision about whether to add a second RFQ to a known backup.

---

## Summary of New Files

| File | What it does |
|---|---|
| `backend/trust_engine.py` | Dynamic approval threshold — rises with good outcomes, falls with bad ones |
| `backend/uncertainty_tracker.py` | Supplier knowledge graph — agent models what it doesn't know |
| `trust_ledger.json` | Auto-created on first run — persists trust history |
| `supplier_knowledge.json` | Auto-created on first run — persists supplier knowledge |

## Summary of Modified Files

| File | What changed |
|---|---|
| `backend/agent.py` | Added Advocate/Skeptic/Arbiter agents; trust threshold lookup; uncertainty assessment; RFQ recording |
| `backend/main.py` | Fixed SMTP/Slack delivery; added `/trust/*` and `/knowledge/*` endpoints; expose trust stats on `/dashboard` |
| `config.py` | `AUTO_EXECUTE_BELOW_USD` now reference-only; live threshold managed by trust engine |
| `frontend/src/App.jsx` | TrustMeter, DeliberationPanel, UncertaintyBadge, OutcomeFeedback components; approval threshold display |
| `frontend/src/App.css` | Styles for all new components |

---

*ChainPilot — autonomous supply chain disruption response*
