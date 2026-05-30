# Swing Trader AI — Chapter-to-Phase Reading Map

> Read **Ch.00** once before anything else — it frames how to use your AI partner while building.

---

## Before Phase 1 — Foundations (Weeks 1–2)
*Goal: Fetch OHLCV, compute all 10 indicators, basic chart UI, daily refresh job, infra up.*

| Chapter | Why before Phase 1 |
|---|---|
| **Ch.01** — Function Calling | You're writing your first `get_ohlcv` and `compute_indicators` tools; understand the contract before you code it |
| **Ch.02** — The Agent Loop | `base-agent.ts` is the loop; read the theory before writing it |
| **Ch.03** — Tools as Contract | Type your tool definitions correctly from day one — harder to retrofit |
| **Ch.08** — State and Persistence | KiteSession, Redis token flag, job timestamps — you're building all of this in Phase 1 |
| **Ch.15** — Backend Infrastructure | Postgres + Redis + BullMQ setup *is* Phase 1 |

---

## Before Phase 2 — Agent Design (Weeks 3–6)
*Goal: Agents find setups, Vision ratifies, Plan Agent drafts hypothesis, you confirm in the evening (no GTT yet).*

| Chapter | Why before Phase 2 |
|---|---|
| **Ch.04** — Prompts, Context, Cache | Vision prompt engineering, context packing into Plan Agent, Redis caching market data |
| **Ch.07** — Memory Writing | Hypothesis gate is a memory-writing decision; understand curation before building the gate |
| **Ch.10** — Multi-Agent Delegation | You're building 7+ agents with distinct roles; understand delegation patterns before wiring them |
| **Ch.11** — Agent Harness | The EOD pipeline (Regime → Scanner → Vision → Briefing) is a harness; design it intentionally |
| **Ch.14** — Skills, MCP, Subagents | Vision as a skill invoked only on shortlist; Plan Agent as on-demand subagent |
| **Ch.17** — Cost, Latency, Model Strategy | Sonnet for text, Opus for Vision, cost guard in Redis — the two-tier strategy the chapter teaches |

---

## Before Phase 3 — Production Loop (Weeks 7–8)
*Goal: Morning readiness scoring drives GTT placement, full trade lifecycle end-to-end.*

| Chapter | Why before Phase 3 |
|---|---|
| **Ch.09** — Planning Patterns | Two-stage approval *is* the chapter's plan-then-execute pattern; read it before building the morning score gate |
| **Ch.12** — Human in the Loop | The philosophical center of your system — hard veto, `open_unprotected`, GTT placement gate |
| **Ch.13** — Connectors, MCP, IPC | Kite GTT postback webhook, Telegram, BullMQ as IPC |
| **Ch.18** — Safety and Adversarial Inputs | Checksum verification on postback, Zod validation on Vision output, hard scanner filters |
| **Ch.19** — Ops and Forward-Deployed | PM2 graceful shutdown, reconciliation safety net, holiday checks, loud early failures |
| **Ch.20** — Proactive Agents | News Agent 30-min polling, 7:30 AM token check, EOD trailing SL suggestions |

---

## Before Phase 4 — Intelligence Layer (Weeks 9–10)
*Goal: Weekly journal review, hypothesis integrity, discipline metrics, regime stats, chat widget, Strategy Lab.*

| Chapter | Why before Phase 4 |
|---|---|
| **Ch.05** — Short-term Memory | Chat Widget conversational context — understand in-run memory before building it |
| **Ch.06** — Long-term Recall | Journal Agent retrieves weeks of trade history; design the retrieval before writing the queries |
| **Ch.16** — Observability | Add structured tracing to agent runs; the chapter will expose the one real gap in your current health check |
| **Ch.21** — Self-Evolving Agents | Your discipline metric and regime stats need to *close the loop* — your known gap. Read before Phase 4, not after |
| **Ch.22** — Designing Your Own Agent | Read at the *end* of Phase 4 as a retrospective — compare your finished design against the canvas, decide what Phase 5 is |

---

## The Known Gap — Phase 5 (future)

Your blueprint collects all the right data (hypothesis integrity, regime breakdown, Vision accuracy per setup type) but stops at reporting. **Ch.21** shows the next move: automatic weight adjustment or prompt evolution based on accumulated outcomes. That is the natural Phase 5 after the core system works.
