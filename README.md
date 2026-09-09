# ChainPilot

**An autonomous multi-agent system for supply chain disruption response.**

ChainPilot watches inventory, supplier and price feeds continuously. When it
detects a disruption it runs a six-agent analysis — including a structured debate
between an agent arguing *for* action and one arguing *against* — and presents a
recommendation at a human approval gate whose height rises and falls with the
system's own track record.

Built on the Anthropic API with FastAPI and React.

---

## What makes it different

Three ideas, each addressing a common failure mode in agentic systems:

**1 · Adversarial deliberation.** Most agent systems emit one confident
recommendation, and you cannot tell whether the downside was considered and
rejected or never considered at all. ChainPilot runs an **Advocate** and a
**Skeptic** in parallel — neither sees the other's argument — and has an
**Arbiter** synthesise them. The Arbiter must name the strongest point on each
side and produce a `swing_condition`: the one thing that would need to be true for
the losing side to win. That turns a verdict into a falsifiable verdict. The whole
debate is shown in the UI, not just the conclusion.

**2 · Confidence-calibrated autonomy.** The auto-execute threshold is not
hardcoded. It starts at $50,000 and moves with outcomes: **+$8,000** when a
recommendation proves good, **−$20,000** when it proves bad. The asymmetry is
deliberate — trust should be slow to build and quick to lose. Hard bounds at
$10,000 and $200,000 mean no accumulation of good outcomes ever unlocks unlimited
autonomy.

**3 · Uncertainty modelling.** The system tracks what it doesn't know. When it
recommends a supplier it has no quality history with, it says so — specifically
("No quality history on file", "Delivery performance unknown") — right at the
approval gate. Uncertainty decays as interactions accumulate and as humans
annotate suppliers.

Underpinning all three: **agents draft, humans send.** Every communication tool
is `draft_only`. Nothing leaves the system without explicit approval.

---

## Architecture at a glance

```
Orchestrator
 ├── parallel → Procurement Specialist   (suppliers, prices, alternatives, cost)
 ├── parallel → Risk Specialist          (customer orders, exposure, stockout)
 ↓  both complete
 ├── Adversarial Deliberation
 │    ├── parallel → Advocate            (argues for immediate action)
 │    ├── parallel → Skeptic             (argues for caution)
 │    └── Arbiter                        (verdict + confidence + swing_condition)
 ↓
 ├── Communications Specialist           (RFQ emails, Slack alert, audit log)
 ↓
 └── finalize_analysis()                 → recommendation card + approval gate
```

Six agents, each a separate Claude call with its own system prompt and tool
subset. Three layers of parallelism. Full detail in
**[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)**.

---

## Quick start

**Prerequisites:** Python 3.9+, Node 18+, and an
[Anthropic API key](https://console.anthropic.com/settings/keys).

```bash
# 1. Configure
cp .env.example .env
#    then edit .env and replace your_api_key_here with your Anthropic key

# 2. Backend  (terminal 1)
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
uvicorn backend.main:app --port 8002

# 3. Frontend (terminal 2)
cd frontend
npm install && npm run dev
```

Dashboard at **http://localhost:3002**, API at **http://localhost:8002**
(interactive docs at `/docs`).

Slack and email are optional — leave those `.env` entries blank and the system
runs in demo mode, rendering approved messages in the UI instead of transmitting
them.

### macOS one-click

Double-click `mac/setup.command` once, then `mac/start.command` whenever you want
to run it. See [mac/README.md](mac/README.md). If ports stay busy after a crash,
`mac/stop.command` clears them.

> **Apple Silicon note:** if `npm run dev` fails with
> `Cannot find module @rollup/rollup-darwin-arm64`, run
> `cd frontend && rm -rf node_modules package-lock.json && npm install`.
> This is a known npm bug with optional native dependencies on ARM Macs.

---

## Walkthrough

1. Open the dashboard — everything green, Trust Meter at the $50,000 baseline.
   The figures drift continuously; the monitor is polling every 15 seconds
   whether or not anyone is watching.
2. Click **⚡ Inject Supply Chain Shock**. Inventory crashes, a supplier goes
   14 days late, prices spike. The monitor detects it and fires the pipeline on
   its own — no further clicks.
3. Watch the pipeline stream through its steps, including the
   **Advocate ⚔ Skeptic** debate.
4. The **Recommendation Card** shows the Arbiter's verdict, its confidence, and
   the `swing_condition` that would flip the decision.
5. At the **approval gate**, open the Deliberation panel to read the full debate,
   and note the Uncertainty badge flagging gaps on the recommended supplier.
6. Approve individual RFQs and the Slack alert independently, then
   **Execute Selected**.
7. Rate the outcome with ✓ / ✗ and watch the Trust Meter move.

---

## Project structure

```
chainpilot/
├── config.py                    Model, thresholds, integration config
├── mock_data.py                 Simulated ERP/supplier/price feeds + drift engine
├── .env.example                 Template — copy to .env
├── backend/
│   ├── agent.py                 Orchestrator + six agents
│   ├── tools.py                 Ten tool functions and their schemas
│   ├── trust_engine.py          Dynamic approval threshold
│   ├── uncertainty_tracker.py   Supplier knowledge graph
│   ├── monitor.py               Async polling loop — the always-on watcher
│   └── main.py                  FastAPI: REST endpoints + SSE streams
├── frontend/src/
│   ├── App.jsx                  Dashboard, pipeline tracker, deliberation, gate
│   └── App.css
├── mac/                         Double-clickable launchers for macOS
└── docs/
    ├── ARCHITECTURE.md          How it works and why
    ├── API.md                   Endpoint reference
    └── changelog/               Eleven development change reports
```

Two files are created at runtime and are not in version control:
`backend/trust_ledger.json` (autonomy history) and
`backend/supplier_knowledge.json` (supplier knowledge graph). Both are per-install
state; delete them to start fresh.

---

## Documentation

| Document | Contents |
|---|---|
| **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)** | Agent hierarchy, the three features in depth, execution and concurrency model, extension points |
| **[docs/API.md](docs/API.md)** | Every endpoint, with request/response shapes |
| **[docs/changelog/](docs/changelog/)** | How the system evolved, including two candid bug post-mortems |
| **[mac/README.md](mac/README.md)** | macOS one-click setup and troubleshooting |

---

## Configuration

Detection thresholds and model settings live in `config.py`:

| Setting | Default | Meaning |
|---|---|---|
| `MODEL` | `claude-sonnet-4-6` | Model used by all six agents |
| `POLL_INTERVAL_SECONDS` | `15` | Monitor loop cadence |
| `STOCK_THRESHOLD_PERCENT` | `0.20` | Below this fires `low_stock` |
| `PRICE_SPIKE_THRESHOLD` | `0.15` | Above baseline fires `price_spike` |
| `SUPPLIER_DELAY_DAYS` | `3` | At or above fires `supplier_delay` |

Autonomy policy — the boost, penalty and bounds — lives at the top of
`backend/trust_engine.py`.

---

## Note on the data

All inventory, supplier, customer and price data is simulated in `mock_data.py`.
Supplier and customer names are fictional and all email addresses use reserved
`.example.com` domains. To connect real systems, replace the tool functions in
`backend/tools.py` while preserving their return shapes — nothing upstream needs
to change.

---

*ChainPilot — autonomous supply chain disruption response*

## License

Released under the [MIT License](LICENSE).
