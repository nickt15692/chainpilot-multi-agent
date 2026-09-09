# ChainPilot API Reference

FastAPI app served on **port 8002**. Interactive docs at
`http://localhost:8002/docs` while the backend is running.

CORS is open to `localhost:3000`, `localhost:3002` and `127.0.0.1:3002`.

---

## Dashboard and live state

### `GET /health`
Liveness check. Returns whether the autonomous monitor loop is running.

### `GET /dashboard`
Full dashboard state in one call — inventory, suppliers, price feeds, detected
events, and the most recent pipeline result. Used on initial page load.

### `GET /realtime/snapshot`
Lightweight snapshot of inventory and prices only. The frontend polls this every
3 seconds to animate the drifting figures without re-fetching everything.

### `GET /events`
The event log — every disruption detected so far this session, with its pipeline
result.

---

## Streams (Server-Sent Events)

### `GET /monitor/stream`
Pushes disruptions as the autonomous monitor detects them. Open for as long as
the dashboard is open; events arrive unprompted.

**Event payload:** `{ "event": {...}, "result": {...} }`

### `GET /pipeline/stream/{sku}`
Runs the full analysis pipeline for one SKU and streams progress step by step.

**Event types:**

| Event | Payload |
|---|---|
| `status` | `{ step, total, message, tool, detail }` — one per pipeline step |
| `complete` | Full pipeline result, approval requirement, trust stats |
| `error` | `{ message }` |

If the SKU has no currently detected disruption, a synthetic `low_stock` event is
constructed from its inventory record so the pipeline can still be demonstrated.

---

## Pipeline execution

### `POST /analyze`
Manually trigger the analysis pipeline (non-streaming equivalent of
`/pipeline/stream/{sku}`).

### `POST /approve`
**The only endpoint that transmits anything externally.** Sends the subset of
drafts the human approved.

```json
{
  "approvals": {
    "slack": true,
    "rfq_emails": [0, 2]
  },
  "event_id": "EVT-1A2B3C4D"
}
```

`rfq_emails` is an array of indices into the drafted RFQ list — approving one
email and rejecting another is supported. Omitted or `false` items are not sent.

**Demo mode:** when `SLACK_WEBHOOK_URL` / SMTP credentials are absent from
`.env`, this returns the content that *would* have been sent, marked as
demo, without transmitting. Real-send failures return the error alongside the
preserved message content.

---

## Trust engine

See [ARCHITECTURE.md](ARCHITECTURE.md#feature-2--confidence-calibrated-autonomy).

### `GET /trust`
Current threshold, decision counts, accuracy percentage, and the last 10 ledger
entries.

### `POST /trust/outcome`
Record how a recommendation actually turned out. This is what moves the
threshold.

```json
{
  "event_id": "EVT-1A2B3C4D",
  "sku": "SKU-4821",
  "recommended_supplier": "FastBear Inc.",
  "cost_exposure": 68400.0,
  "outcome": "good",
  "notes": "Delivered in 3 days as quoted."
}
```

`outcome` is `"good"` (+$8,000), `"bad"` (−$20,000) or `"neutral"` (no change).
Returns updated trust stats.

### `POST /trust/reset`
Reset the ledger to the $50,000 baseline. Intended for demos and testing.

---

## Supplier knowledge base

See [ARCHITECTURE.md](ARCHITECTURE.md#feature-3--uncertainty-modelling).

### `GET /knowledge`
The full supplier knowledge graph — uncertainty levels, interaction counts,
quality notes and known gaps.

### `POST /knowledge/annotate`
Add human knowledge about a supplier.

```json
{
  "supplier_name": "FastBear Inc.",
  "quality_note": "Two clean deliveries, no defects.",
  "uncertainty": "LOW"
}
```

`uncertainty` is `"HIGH"`, `"MEDIUM"` or `"LOW"`, and is optional — omit it to
add a note without changing the level. Setting `LOW` clears all recorded gaps.

### `POST /knowledge/reset`
Wipe the knowledge base. Intended for demos and testing.

---

## Demo controls

These manipulate the simulated data feeds and have no production equivalent.

| Endpoint | Effect |
|---|---|
| `POST /demo/pre-shock` | Reset feeds to a healthy all-green baseline |
| `POST /demo/inject-shock` | Apply crisis state — stock crash, 14-day supplier delay, price spike |
| `POST /demo/reset` | Clear active events and the event log so the same disruption can re-fire |

---

*ChainPilot — autonomous supply chain disruption response*
