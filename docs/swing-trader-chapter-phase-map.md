# Swing Trader AI — Chapter-to-Phase Reading Map

> Read **Ch.00** once before anything else — it frames how to use your AI partner while building.

---

## Mental Model First

Before Phase 1, internalize two things from the v5 blueprint (§1):

**Three Stock Groups** — open positions (Group 1, excluded from setup detection), active watchlist (Group 2, scanned nightly Mon–Fri), NSE 500 universe (Group 3, scanned weekly on Sundays only). Stocks move between groups automatically.

**Two Rhythms** — Weekly (Sunday scanner runs NSE 500, produces proposals, you curate watchlist) + Daily (Mon–Fri evening scan on Group 2 only → morning GTT placement). The entire system is designed around these two cadences. Every agent fits into one or both.

---

## Before Phase 1 — Data + Indicators (Weeks 1–2)
*Goal: Fetch OHLCV for Group 2 watchlist (~50 stocks), compute all 10 indicators, basic chart UI, daily refresh job, infra up.*

| Chapter | Why before Phase 1 |
|---|---|
| **Ch.01** — Function Calling | Writing first `get_ohlcv` and `compute_indicators` tools; understand the contract before you code it |
| **Ch.02** — The Agent Loop | `base-agent.ts` is the loop; read the theory before writing it |
| **Ch.03** — Tools as Contract | Type tool definitions, metadata flags, and result envelopes correctly from day one — harder to retrofit |
| **Ch.08** — State and Persistence | KiteSession, Redis token flag, BullMQ job timestamps — you're building all of this in Phase 1 |
| **Ch.15** — Backend Infrastructure | Postgres + Redis + BullMQ setup *is* Phase 1 |

---

## Before Phase 2 — Scanner + Vision + Plan Agent + Hypothesis (Weeks 3–6)
*Goal: Sunday scanner proposes NSE 500 additions to watchlist. Weekday scanner finds setups in Group 2, each scored by the §15 indicator confidence formula. Vision ratifies (veto-first). Plan Agent drafts hypothesis. You confirm in the evening (no GTT yet).*

| Chapter | Why before Phase 2 |
|---|---|
| **Ch.04** — Prompts, Context, Cache | Vision prompt engineering; packing indicator snapshots into Plan Agent context; Redis caching market data |
| **Ch.07** — Memory Writing | Hypothesis gate is a memory-writing decision; understand curation before building the gate |
| **Ch.10** — Multi-Agent Delegation | Two fan-outs live here: Sunday scanner across NSE 500 (500 parallel invocations) and weekday scanner across Group 2. Understand delegation patterns before wiring them |
| **Ch.11** — Agent Harness | The EOD pipeline (Regime → Scanner → Vision → Briefing) and Sunday pipeline are harnesses; design them intentionally |
| **Ch.14** — Skills, MCP, Subagents | Vision as a skill invoked only on shortlist; Plan Agent as on-demand subagent per setup |
| **Ch.17** — Cost, Latency, Model Strategy | Vision model evaluation (Gemini Flash vs GPT-4o vs Claude Sonnet) is a Ch.17 decision; Sonnet for text agents, selected model for Vision, cost guard in Redis. v5 keeps Vision **veto-first** (40% weight is un-backtested) until Phase 4 measures its accuracy per score decile |

---

## Before Phase 3 — Morning Score + GTT Lifecycle + Briefing (Weeks 7–8)
*Goal: Morning readiness scoring drives GTT placement, gated by regime (risk-off blocks new longs). Dual score columns (finalScore + morningReadinessScore). 9:15 AM gap check cancels stale GTTs. Staged exit (breakeven / partial / trail). Full trade lifecycle end-to-end.*

| Chapter | Why before Phase 3 |
|---|---|
| **Ch.09** — Planning Patterns | Two-stage approval (evening hypothesis confirm → morning GTT placement) *is* the chapter's plan-then-execute pattern. v5 adds the **regime hard-gate** as a plan precondition — risk-off blocks placement entirely |
| **Ch.12** — Human in the Loop | Philosophical center of your system — hard veto, `open_unprotected` red banner, GTT placement gate, SL GTT failure escalation, staged-exit approvals (breakeven / partial / trail) |
| **Ch.13** — Connectors, MCP, IPC | Kite GTT postback webhook, Telegram alerts, BullMQ as IPC between workers |
| **Ch.18** — Safety and Adversarial Inputs | Checksum verification on postback, Zod validation on Vision output, hard scanner filters (pledge %, F&O ban, ex-dividend). v5: the **Kite-GTT-is-LIMIT-on-trigger** correctness fix — buffered SL limit so a gap-down doesn't leave the stop unfilled |
| **Ch.19** — Ops and Forward-Deployed | PM2 graceful shutdown, holiday checks, loud early failures. v5: News-Agent **30-min SL-presence assertion** is the *primary* protection (unprotected window ≤30 min); the 3:35 PM reconciliation is now the backstop, not the front line |
| **Ch.20** — Proactive Agents | News Agent 30-min BSE polling, 7:30 AM token health check, 9:15 AM gap check (cancel entry GTT if opened outside zone), EOD **staged-exit** suggestions (breakeven at +1R, partial at T1, ATR/swing trail, time stop), mid-swing results alert |

---

## Before Phase 4 — Journal + Intelligence (Weeks 9–10)
*Goal: Weekly journal review, hypothesis integrity scoring, discipline metrics, regime-correlated stats, chat widget, Strategy Lab — which now also **calibrates the v5 backtest-seed constants** (indicator sub-score weights, regime thresholds, exit config).*

| Chapter | Why before Phase 4 |
|---|---|
| **Ch.05** — Short-term Memory | Chat Widget needs conversational context within a session — understand in-run memory before building it |
| **Ch.06** — Long-term Recall | Journal Agent retrieves weeks of trade history; design the retrieval before writing the queries |
| **Ch.16** — Observability | Add structured tracing to all agent runs; exposes the one real gap in your current health check |
| **Ch.21** — Self-Evolving Agents | Discipline metric and regime stats need to *close the loop* — your known gap. Strategy Lab is the first move: it must **earn the v5 seed constants** (indicator weights, regime thresholds, exit config) via walk-forward backtests before they can be trusted live. Read before Phase 4, not after |
| **Ch.22** — Designing Your Own Agent | Read at the *end* of Phase 4 as a retrospective — compare your finished design against the canvas, decide what Phase 5 is |

---

## The Known Gap — Phase 5 (future)

Your blueprint collects all the right data: hypothesis integrity, regime breakdown, Vision accuracy per setup type, discipline score. But it stops at reporting. **Ch.21** shows the next move: automatic weight adjustment or prompt evolution based on accumulated outcomes. Strategy Lab (Phase 4) seeds the data; Phase 5 closes the loop.

**v5 made this gap concrete.** Every constant introduced in v5 — the indicator sub-score weights, the regime classification thresholds, the 0.6/0.4 Vision blend, the MR ≥ 50 bar, the exit config — is a hand-set *seed*, not a measured value. Phase 4's Strategy Lab calibrates them once via walk-forward backtest; Phase 5 is where the system **re-calibrates them automatically** from live outcomes. Until then, treat every v5 number as a hypothesis under test, not a tuned parameter.
