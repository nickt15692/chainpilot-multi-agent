# ChainPilot — Change Report 4

## Overview

Three gaps were closed: specialist agents that were running sequentially now run in parallel when independent, RFQ emails that were drafted but never sent are now actually sent via SMTP on approval, and the all-or-nothing approval gate was replaced with per-action checkboxes so a human can approve individual emails and the Slack alert independently.

---

## Change 1: Parallel Specialist Execution

**Files changed:** `backend/agent.py`

### What changed

Two things were updated together.

**Orchestrator system prompt** (`_SYSTEM_ORCHESTRATOR_TEMPLATE`) — one line added to the guidelines:

> "run_procurement_analysis and run_risk_assessment are independent — you may call both in the same turn and they will execute simultaneously. This is preferred when both are needed."

**Tool execution loop in `_run_orchestrator`** — the `tool_use` branch was rewritten. Previously it was a `for` loop that ran each tool one at a time:

```python
for block in response.content:
    if block.type != "tool_use":
        continue
    fn = orchestrator_tool_map.get(block.name)
    specialist_result = fn(**block.input)  # blocks until done
    ...
```

Now it collects all `tool_use` blocks from the response first, submits all of them to a `ThreadPoolExecutor`, and waits for all to complete:

```python
tool_use_blocks = [b for b in response.content if b.type == "tool_use"]

def _execute_specialist(block):
    fn = orchestrator_tool_map.get(block.name)
    return block, fn(**block.input) if fn else {"error": ...}

with concurrent.futures.ThreadPoolExecutor() as pool:
    futures = [pool.submit(_execute_specialist, b) for b in tool_use_blocks]
    executed = [f.result() for f in concurrent.futures.as_completed(futures)]
```

`concurrent.futures` was also added to the top-level imports.

### Why it was made

The Procurement and Risk specialists look at completely different data — procurement checks supplier status, price feeds, and alternative suppliers; risk checks inventory levels and customer orders. Neither needs to wait for the other. Running them sequentially was pure overhead: if each takes 8 seconds, the old code took 16 seconds. With this change they take ~8 seconds total.

More importantly, the Anthropic API supports Claude returning multiple `tool_use` blocks in a single response. Without the system prompt hint, Claude had no reason to try calling both in the same turn. Without the parallel execution in the loop, even if Claude did return both simultaneously, the backend would have run them sequentially anyway.

### What would happen without this change

Every pipeline run executes Procurement, then waits, then executes Risk. The total wall-clock time is the sum of both agents' runtimes rather than the maximum. For disruptions that need both specialists (most of them), this doubles the waiting time unnecessarily. It also means the two agents can never share the response context from Claude's perspective — they will always be separate turns, never simultaneous.

---

## Change 2: Wire SMTP Email Sending

**Files changed:** `config.py`, `backend/main.py`

### What changed

**`config.py`** — four new config vars added:

```python
SMTP_PORT = int(os.getenv("SMTP_PORT", "587"))
SMTP_USER = os.getenv("SMTP_USER", "")
SMTP_PASS = os.getenv("SMTP_PASS", "")
SMTP_FROM = os.getenv("SMTP_FROM", "chainpilot@company.example.com")
```

`SMTP_SERVER` already existed in config but none of the other SMTP vars did, which meant a working SMTP connection was impossible even if `SMTP_SERVER` was set.

**`backend/main.py`** — `smtplib` and `email.mime.text.MIMEText` imported. New `_send_smtp_email(to, subject, body)` helper added:

- Connects to `SMTP_SERVER:SMTP_PORT`
- If `SMTP_USER` and `SMTP_PASS` are set, calls `starttls()` and logs in
- Sends the message and returns `{"sent": True, "to": to}`
- If `SMTP_SERVER` is empty, returns `{"sent": False, "skipped_reason": "SMTP_SERVER not configured"}` — graceful degradation, same pattern as Slack
- Any connection or auth error is caught and returned as `{"sent": False, "error": str(e)}`

The `/approve` endpoint was updated to call `_send_smtp_email` for each approved RFQ index (see Change 3).

### Why it was made

`draft_rfq_email()` in `tools.py` has always returned `{"draft_only": True}`. The system presents drafted emails to the user for approval, but before this change, clicking approve never actually sent them. `SMTP_SERVER` was loaded from the environment in `config.py` but never imported or used anywhere in the codebase — it was entirely dead config. RFQ emails are the primary output of the Communications agent, so leaving them unsent made the approval step effectively meaningless for email.

### What would happen without this change

RFQ emails remain display-only regardless of what the user approves. The supplier never receives the request for quotation. The only real action on approval is the Slack message. The drafted emails shown in the UI are misleading — they look like they will be sent, but nothing happens.

---

## Change 3: Granular Per-Action Approval

**Files changed:** `backend/main.py`, `frontend/src/App.jsx`

### What changed

**`backend/main.py` — `/approve` endpoint** now accepts a new request shape:

```json
{
  "approvals": {
    "slack": true,
    "rfq_emails": [0, 1]
  },
  "pipeline_result": { ... }
}
```

`rfq_emails` is a list of integer indices into `pipeline_result.drafts.rfq_emails`. Passing `[0]` sends only the first RFQ; passing `[]` sends none. The Slack alert is sent only if `approvals.slack` is `true`.

The old shape `{"approved": true, "pipeline_result": {...}}` still works — if the `"approvals"` key is absent, `approved: true` is treated as approving everything and `approved: false` as approving nothing.

The response was updated to return both `slack` and `emails` result arrays so the caller knows what was actually sent.

**`frontend/src/App.jsx` — `ApprovalGate` component** was rewritten. The single "Approve All Actions" / "Cancel" button pair was replaced with:

- A checkbox per RFQ email, showing the recipient address and subject line, with the first 220 characters of the email body visible
- A checkbox for the Slack alert, showing the full drafted message
- All checkboxes default to checked (opt-out model — approve all unless unchecked)
- An "Execute Selected Actions" button that is disabled if nothing is checked
- A "Cancel All" button that rejects everything

`handleApprove` in `App` was updated to accept the approvals object from the component and pass it to the backend instead of a flat `approved: true` boolean.

The done-state summary was updated to reflect what was actually approved: `"2 RFQ(s) sent · Slack posted · Audit logged"` rather than a hardcoded string. The "RFQs sent" stat in the summary card now shows `approved.rfq_emails.length` instead of the total number of drafted emails.

### Why it was made

The previous approval gate was binary. A disruption produces multiple independent actions — typically two RFQ emails (primary and backup supplier) and a Slack alert. These have different risk profiles and different consequences. A procurement manager might want to send the Slack immediately but hold the RFQ to a new supplier until they've made a phone call first, or send one RFQ but not the other if the second supplier has an unresolved quality issue. Forcing all-or-nothing approval removes this judgment.

### What would happen without this change

Every approval sends everything or nothing. There is no way to partially act on a disruption response — to send one email but not another, or to post Slack without committing to any supplier yet. The approval step has no granularity, which means it either over-executes (sends RFQs the human wasn't ready to send) or under-executes (human cancels everything because one action wasn't ready, even though the others were).

---

## Summary

| Change | Files | Problem solved | Without it |
|--------|-------|---------------|------------|
| Parallel specialist execution | `backend/agent.py` | Procurement and Risk ran sequentially despite being independent | Pipeline takes 2× as long; agents can never run simultaneously |
| SMTP email sending | `config.py`, `backend/main.py` | RFQ emails were drafted but never sent | Approval is meaningless for email; suppliers never receive RFQs |
| Granular approval checkboxes | `backend/main.py`, `frontend/src/App.jsx` | Approval was all-or-nothing across all drafted actions | No way to approve some actions without approving all; human loses judgment over individual communications |
