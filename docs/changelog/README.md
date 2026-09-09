# Development changelog

Eleven change reports written as the system was built, in order. Each explains
what changed, *why*, and what would break if the change were absent.

They are kept because the reasoning is often more interesting than the diff —
several of these document architectural reversals, and reports 05 and 09 are
candid post-mortems of bugs introduced by earlier reports.

| # | Focus |
|---|---|
| [01](01-change-report.md) | Backend reliability, agent intelligence, first real integrations |
| [02](02-change-report.md) | **Monolith → orchestrator + specialists.** One 25-turn loop with 9 tools became an orchestrator coordinating three focused agents |
| [03](03-change-report.md) | Starting the autonomous monitor loop that existed but was never wired up; giving agents real decision authority instead of scripted tool sequences |
| [04](04-change-report.md) | Parallel specialists, live SMTP sending, per-action approval checkboxes replacing all-or-nothing |
| [05](05-change-report.md) | Post-mortem: two bugs introduced by report 04 |
| [06](06-change-report.md) | **Guaranteeing parallelism.** Two tools merged into one with boolean flags, so Python dispatches specialists concurrently rather than hoping Claude emits both calls in one turn |
| [07](07-change-report.md) | Wiring monitor results to the frontend via SSE — the autonomy existed but was invisible until a user clicked something |
| [08](08-change-report.md) | Healthy baseline + one-click shock injection; rendering the `finalize_analysis` output that was computed but never displayed |
| [09](09-change-report.md) | Post-mortem: wrong browser port, duplicate tabs, and rate-limit retry with exponential backoff |
| [10](10-change-report.md) | **The three headline features** — adversarial deliberation, trust-calibrated autonomy, uncertainty modelling |
| [11](11-change-report.md) | Continuous data drift so the dashboard is never frozen between polls |

For the resulting architecture rather than the path to it, see
[ARCHITECTURE.md](../ARCHITECTURE.md).

---

*ChainPilot — autonomous supply chain disruption response*
