# Swing Trader AI — Complete Architecture Blueprint

> **Target:** Personal use · Zerodha Kite · Next.js + TypeScript · Claude API
> **Philosophy:** Solid foundation, built to expand. Human approves, agent executes.
> **Signal approach:** Hybrid — 10 orthogonal scalar indicators (fast, free, runs on all 500 stocks) ratified by Vision AI chart analysis (slow, paid, runs only on shortlisted setups).

---

## Table of Contents

1. [Big Picture Mental Model](#1-big-picture-mental-model)
2. [Technology Stack](#2-technology-stack)
3. [Repository Structure](#3-repository-structure)
4. [System Architecture](#4-system-architecture)
5. [Database Schema](#5-database-schema)
6. [The Agent System](#6-the-agent-system)
7. [All Agents — Detailed Specification](#7-all-agents--detailed-specification)
8. [Workflows](#8-workflows)
9. [API Layer](#9-api-layer)
10. [Frontend Pages and Components](#10-frontend-pages-and-components)
11. [Data Pipeline](#11-data-pipeline)
12. [Indicator Engine — 10 Orthogonal Indicators](#12-indicator-engine--10-orthogonal-indicators)
13. [Vision AI Ratification Layer](#13-vision-ai-ratification-layer)
14. [Zerodha Kite Integration](#14-zerodha-kite-integration)
15. [Authentication and Security](#15-authentication-and-security)
16. [Environment Variables](#16-environment-variables)
17. [Build and Run](#17-build-and-run)
18. [Phase-wise Development Plan](#18-phase-wise-development-plan)
19. [Key Design Decisions and Rationale](#19-key-design-decisions-and-rationale)

---

## 1. Big Picture Mental Model

Before any code, understand the **core loop** this app automates:

```
EVERY EVENING (after 3:30 PM IST)
  Token Health Check — verify Kite access_token is valid
  Data Agent pulls OHLCV for NSE 500 stocks
  Indicator Engine computes 10 orthogonal indicators per stock   ← fast, free
  Regime Agent classifies market condition
  Scanner Agent scores every stock → shortlist (~15–25 setups)
  Vision Agent renders charts for shortlist → ratifies each setup ← slow, paid
  Plan Agent drafts 2–3 line hypothesis for each setup           ← NEW
  Briefing Agent writes your evening summary
  Watchlist Score recomputed for all watchlist stocks            ← NEW

YOU review Vision-ratified shortlist → read agent hypothesis → edit if needed
→ approve trade plans

NEXT MORNING (8:45 AM IST, before market open)
  Token Health Check — alert if re-auth needed
  Briefing Agent fetches global cues, events, FII data
  Pre-Market Staleness Check on all pending setups               ← NEW
  Morning Readiness Score computed for each candidate            ← NEW
  You review morning brief + readiness scores

DURING MARKET HOURS
  9:15 AM — News Agent first pass: opening gap check on gtt_placed setups
             Cancel GTT + mark expired if stock opened outside entry zone
  Every 30 min — News Agent: BSE announcements for open positions,
                              gtt_placed setups, watchlist stocks
  Real-time — Kite GTT postback webhook: entry/SL execution → DB update
              + immediate SL GTT placement on entry + Telegram alert
  3:35 PM — Order Reconciliation: safety net for missed postbacks

WEEKLY (Sunday)
  Journal Agent reviews all closed trades
  Generates win rate, R:R analysis, mistake patterns
  Evaluates: was the hypothesis still intact at entry/exit?      ← NEW
```

**The human is always in the approval seat.** The agent never places orders without your confirmation. This is both a safety design and compliant with SEBI's retail algo trading rules.

---

## 2. Technology Stack

| Layer | Technology | Why |
|---|---|---|
| Frontend | Next.js 14 (App Router) | React + API routes in one repo, SSR for fast loads |
| Language | TypeScript | Type safety across frontend and backend |
| Backend API | Next.js Route Handlers | No separate Express needed for personal use |
| Background Jobs | BullMQ + Redis | Reliable job queues for scheduled scans |
| Database | PostgreSQL (via Prisma ORM) | Relational, typed, great for trade/journal data |
| Cache | Redis | Market data cache, job queue, WebSocket state |
| AI Agent (text) | Claude API — claude-sonnet-4-5 | Tool-use + reasoning for agentic workflows |
| AI Agent (vision) | Claude API — claude-opus-4-5 | Vision model for chart pattern ratification |
| Agent Framework | Custom tool-call loop (TypeScript) | Lightweight, no heavy framework needed |
| Indicator Engine | technicalindicators (npm) | Pure TS, covers all standard indicators |
| Chart Renderer | chartjs-node-canvas (npm) | Server-side PNG generation for Vision input |
| Broker API | kiteconnect (npm) | Official Zerodha SDK |
| WebSocket | Kite Ticker (via kiteconnect) | Real-time price feed |
| Charts | TradingView Lightweight Charts | Professional-grade, free |
| Job Scheduler | node-cron | Triggers BullMQ jobs on IST schedule |
| Notifications | Telegram Bot API | Alerts to your phone |
| Auth | NextAuth.js (single user) | Simple, secure, local login |
| Deployment | Local machine or VPS (PM2) | Personal use, no cloud cost needed |

---

## 3. Repository Structure

```
swing-trader/
├── prisma/
│   └── schema.prisma              # Database models
│
├── src/
│   ├── app/                       # Next.js App Router
│   │   ├── (auth)/
│   │   │   └── login/page.tsx
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx           # Home dashboard
│   │   │   ├── scanner/page.tsx
│   │   │   ├── watchlist/page.tsx
│   │   │   ├── positions/page.tsx
│   │   │   ├── journal/page.tsx
│   │   │   ├── briefing/page.tsx
│   │   │   └── strategy-lab/page.tsx
│   │   └── api/
│   │       ├── agent/
│   │       │   ├── scan/route.ts
│   │       │   ├── vision/route.ts
│   │       │   ├── plan/route.ts
│   │       │   ├── review/route.ts
│   │       │   ├── chat/route.ts
│   │       │   └── morning-score/route.ts   # NEW
│   │       ├── kite/
│   │       │   ├── auth/route.ts
│   │       │   ├── callback/route.ts
│   │       │   ├── orders/route.ts
│   │       │   └── positions/route.ts
│   │       ├── market/
│   │       │   ├── ohlcv/route.ts
│   │       │   └── quote/route.ts
│   │       ├── jobs/
│   │       │   └── trigger/route.ts
│   │       └── health/route.ts             # NEW
│   │
│   ├── agents/
│   │   ├── base-agent.ts
│   │   ├── scanner-agent.ts
│   │   ├── vision-agent.ts
│   │   ├── plan-agent.ts                   # UPDATED — drafts hypothesis
│   │   ├── monitor-agent.ts
│   │   ├── news-agent.ts
│   │   ├── briefing-agent.ts
│   │   ├── journal-agent.ts                # UPDATED — evaluates hypothesis
│   │   ├── regime-agent.ts
│   │   └── watchlist-score-agent.ts        # NEW
│   │
│   ├── tools/
│   │   ├── market-tools.ts
│   │   ├── kite-tools.ts
│   │   ├── db-tools.ts
│   │   ├── news-tools.ts
│   │   └── screener-tools.ts
│   │
│   ├── indicators/
│   │   ├── engine.ts
│   │   ├── trend.ts
│   │   ├── momentum.ts
│   │   ├── volatility.ts
│   │   ├── volume.ts
│   │   └── structure.ts
│   │
│   ├── scoring/                            # NEW — scoring engines
│   │   ├── watchlist-score.ts             # Composite watchlist score (0–100)
│   │   └── morning-readiness-score.ts     # Staleness-adjusted morning score (0–100)
│   │
│   ├── vision/
│   │   ├── chart-renderer.ts
│   │   ├── vision-prompt.ts
│   │   └── vision-types.ts
│   │
│   ├── services/
│   │   ├── kite.ts
│   │   ├── ticker.ts
│   │   ├── telegram.ts
│   │   ├── cache.ts
│   │   └── token-manager.ts               # NEW — daily token lifecycle
│   │
│   ├── jobs/
│   │   ├── queue.ts
│   │   ├── workers/
│   │   │   ├── eod-scan.worker.ts         # UPDATED — includes EOD position review
│   │   │   ├── morning-brief.worker.ts    # UPDATED — staleness check + MR score + GTT placement
│   │   │   ├── news-agent.worker.ts       # 30-min intraday + 9:15 AM gap check (replaces monitor)
│   │   │   ├── journal.worker.ts
│   │   │   ├── token-check.worker.ts      # NEW
│   │   │   └── order-reconciliation.worker.ts  # NEW
│   │   └── scheduler.ts
│   │
│   ├── lib/
│   │   ├── prisma.ts
│   │   ├── redis.ts
│   │   ├── claude.ts
│   │   └── utils.ts
│   │
│   └── types/
│       ├── market.ts
│       ├── agent.ts
│       ├── vision.ts
│       ├── trade.ts
│       └── scoring.ts                     # NEW — WatchlistScore, MorningReadinessScore types
│
├── .env.local
├── package.json
├── tsconfig.json
└── docker-compose.yml
```

---

## 4. System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     YOUR BROWSER                            │
│              Next.js Frontend (React)                       │
│   Dashboard │ Scanner │ Positions │ Journal │ Briefing      │
└──────────────────────┬──────────────────────────────────────┘
                       │ HTTP / Server-Sent Events
┌──────────────────────▼──────────────────────────────────────┐
│                  Next.js API Layer                           │
│            (Route Handlers — same process)                  │
│    /api/agent/*   /api/kite/*   /api/market/*  /api/health  │
└────┬──────────────────┬───────────────────┬─────────────────┘
     │                  │                   │
     ▼                  ▼                   ▼
┌─────────┐    ┌────────────────┐   ┌───────────────┐
│  Agent  │    │   BullMQ Jobs  │   │  Kite Connect │
│  Layer  │    │   (Redis queue)│   │  REST + WS    │
│         │    │                │   │               │
│ Claude  │    │ EOD Scan       │   │ Historical    │
│ API     │    │ Morning Brief  │   │ OHLCV         │
│ Tool    │    │ Monitor        │   │ Live Ticker   │
│ Calls   │    │ Journal Review │   │ Order Mgmt    │
│         │    │ Token Check    │   │               │
│         │    │ Order Recon    │   │               │
└────┬────┘    └───────┬────────┘   └──────┬────────┘
     │                 │                   │
     └────────┬────────┘                   │
              ▼                            │
┌─────────────────────────┐               │
│      PostgreSQL          │◄──────────────┘
│      (via Prisma)        │
│                         │
│  instruments  stocks    │
│  candles      trades    │
│  positions    journal   │
│  setups       plans     │
│  briefings    alerts    │
│  market_regime          │  ← NEW
│  kite_sessions          │  ← NEW
│  trade_exits            │  ← NEW
└─────────────────────────┘
              │
              ▼
┌─────────────────────────┐
│         Redis           │
│  BullMQ queues          │
│  Market data cache      │
│  WebSocket tick cache   │
│  Vision daily counter   │  ← NEW
│  Audit log stream       │  ← NEW
└─────────────────────────┘
              │
              ▼
┌─────────────────────────┐
│      Telegram Bot       │
│  Alerts to your phone   │
└─────────────────────────┘
```

---

## 5. Database Schema

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ── Market Data ──────────────────────────────────────────────

model Instrument {
  id            String   @id // NSE:RELIANCE
  symbol        String
  exchange      String
  name          String
  sector        String?
  isFnO         Boolean  @default(false)
  lotSize       Int?
  candles       Candle[]
  watchlistItems WatchlistItem[]
  setups        Setup[]
  createdAt     DateTime @default(now())
}

model Candle {
  id           Int        @id @default(autoincrement())
  instrumentId String
  instrument   Instrument @relation(fields: [instrumentId], references: [id])
  timeframe    String     // "day", "week", "15minute"
  timestamp    DateTime
  open         Float
  high         Float
  low          Float
  close        Float
  volume       Int
  @@unique([instrumentId, timeframe, timestamp])
  @@index([instrumentId, timeframe])
}

// ── Market Regime ─────────────────────────────────────────────
// NEW: Persists every evening's regime classification so the
// Journal Agent can correlate trade outcomes with market conditions.

model MarketRegime {
  id           Int      @id @default(autoincrement())
  date         DateTime @unique
  trend        String   // "up" | "down" | "sideways"
  strength     Float    // ADX value of Nifty
  vixLevel     Float
  breadthScore Float    // advance/decline ratio
  summary      String   // plain English paragraph from Regime Agent
  createdAt    DateTime @default(now())
}

// ── Kite Session ──────────────────────────────────────────────
// NEW: Persists the daily Kite access_token so it survives PM2
// restarts. The token-check worker writes here after each login.

model KiteSession {
  id           Int      @id @default(autoincrement())
  accessToken  String
  refreshedAt  DateTime @default(now())
  expiresAt    DateTime // always next day 6:00 AM IST
}

// ── Watchlist ─────────────────────────────────────────────────

model WatchlistItem {
  id               Int        @id @default(autoincrement())
  instrumentId     String
  instrument       Instrument @relation(fields: [instrumentId], references: [id])
  addedAt          DateTime   @default(now())
  isActive         Boolean    @default(true)

  // Structural thesis — written and owned by you (1–3 sentences).
  // Explains WHY this stock is on your watchlist at the macro/
  // fundamental level, independent of any specific setup.
  hypothesis       String?
  hypothesisUpdatedAt DateTime?

  // Composite watchlist score (0–100), recomputed every evening.
  // Components: weekly trend strength (40%) + RS vs Nifty (30%)
  // + setup readiness proximity (30%).
  watchlistScore        Float?     // 0–100
  watchlistScoreDetail  Json?      // breakdown of sub-scores
  watchlistScoredAt     DateTime?
}

// ── Setups and Plans ─────────────────────────────────────────

model Setup {
  id              Int        @id @default(autoincrement())
  instrumentId    String
  instrument      Instrument @relation(fields: [instrumentId], references: [id])
  detectedAt      DateTime   @default(now())
  setupType       String     // "breakout", "pullback_ema", "inside_bar", etc.
  timeframe       String
  confidenceScore Float      // 0–100, from scalar indicators
  visionScore     Float?     // 0–100, from Vision AI ratification
  finalScore      Float?     // 0.6 * indicatorScore + 0.4 * visionScore

  // Morning Readiness Score — computed at 8:45 AM the next day.
  // Separate from finalScore so you can always see the original
  // evening signal vs. how conditions look at market open.
  morningReadinessScore  Float?   // 0–100
  morningScoreDetail     Json?    // penalty breakdown (gap, news, VIX, Nifty gap)
  morningScoreComputedAt DateTime?

  // Agent-drafted hypothesis bridging your watchlist thesis to
  // this specific setup. Drafted by the Plan Agent using:
  // your watchlist hypothesis + indicator snapshot + Vision verdict
  // + any overnight BSE/news. You must edit or confirm before
  // approving a plan. Plan approval is blocked until non-empty.
  setupHypothesis        String?
  hypothesisEditedByUser Boolean  @default(false)

  visionVerdict   Json?
  agentReasoning  String
  indicatorData   Json
  weeklySnapshot  Json?      // NEW: weekly indicator context
  chartImagePath  String?
  status          String     @default("new")
  // Statuses:
  //   new                  — detected by scanner, Vision pending
  //   vision_pending       — in Vision Agent queue
  //   vision_done          — Vision ratified, finalScore set, awaiting your review
  //   vision_vetoed        — Vision score < 40, excluded from Candidates
  //   plan_generated       — Plan Agent has run, hypothesis drafted
  //   plan_confirmed       — you confirmed/edited hypothesis (evening gate passed)
  //                          Morning Readiness Score not yet computed
  //   gtt_placed           — morning score reviewed, entry GTT live at Kite
  //   expired              — morning veto OR stock opened outside entry zone at 9:15 AM
  //   stale_pending_review — morning score dropped > 20 pts, needs manual review
  //   traded               — entry GTT triggered, linked Trade record created
  //   rejected             — you explicitly skipped this setup

  plan            TradePlan?
}

model TradePlan {
  id              Int      @id @default(autoincrement())
  setupId         Int      @unique
  setup           Setup    @relation(fields: [setupId], references: [id])
  createdAt       DateTime @default(now())
  entryPrice      Float
  entryZoneLow    Float
  entryZoneHigh   Float
  stopLoss        Float
  target1         Float
  target2         Float?
  target3         Float?
  quantity        Int
  riskAmount      Float
  riskRewardRatio Float
  agentReasoning  String
  status          String   @default("pending")
  // Statuses:
  //   pending          — plan generated, hypothesis not yet confirmed
  //   plan_confirmed   — hypothesis confirmed/edited, waiting for morning score
  //                      NO GTT placed yet. This is the evening approval gate.
  //   gtt_placed       — morning score reviewed, GTT entry order live at Kite
  //   rejected         — you chose to skip this setup
  //   expired          — morning score vetoed or stock opened outside entry zone
  //   executed         — entry GTT triggered, trade is now open
  trade           Trade?
}

// ── Trades and Positions ──────────────────────────────────────

model Trade {
  id              Int       @id @default(autoincrement())
  planId          Int?      @unique
  plan            TradePlan? @relation(fields: [planId], references: [id])
  instrumentId    String
  entryGttId      String?   // Kite GTT order ID for entry leg
  slGttId         String?   // Kite GTT order ID for SL leg (set after entry fires)
  entryDate       DateTime
  entryPrice      Float
  quantity        Int
  side            String    // "BUY" or "SELL"
  exitDate        DateTime?
  exitPrice       Float?
  exitReason      String?   // "target1" | "target2" | "stoploss" | "manual" | "trailing_sl"
  realizedPnl     Float?
  status          String    @default("open")
  // Statuses:
  //   open               — entry triggered, SL GTT is live
  //   open_unprotected   — entry triggered but SL GTT placement FAILED
  //                        requires immediate manual action
  //   closed             — trade fully exited
  //   partial            — partially exited (some quantity remains open)
  stopLoss        Float
  currentSL       Float     // updated as trailing SL moves
  target1         Float
  target2         Float?
  journalEntry    JournalEntry?
  exits           TradeExit[]
}

// NEW: Records each individual exit leg — supports scaled exits
// (50% at target1, trail the rest). Without this, the Trade model
// can only store a single exit price, breaking partial-exit accounting.

model TradeExit {
  id          Int      @id @default(autoincrement())
  tradeId     Int
  trade       Trade    @relation(fields: [tradeId], references: [id])
  exitDate    DateTime
  exitPrice   Float
  quantity    Int
  exitReason  String   // "target1" | "target2" | "stoploss" | "manual" | "trailing_sl"
  realizedPnl Float
  createdAt   DateTime @default(now())
}

// ── Journal ───────────────────────────────────────────────────

model JournalEntry {
  id              Int      @id @default(autoincrement())
  tradeId         Int      @unique
  trade           Trade    @relation(fields: [tradeId], references: [id])
  createdAt       DateTime @default(now())
  setupCorrect    Boolean?
  entryCorrect    Boolean?
  exitCorrect     Boolean?
  mistakeTag      String?  // "early_entry" | "ignored_sl" | "missed_news" | "thesis_invalid_at_entry" | etc.

  // NEW: Was the hypothesis still valid at the time of entry?
  // The Journal Agent re-reads the setup hypothesis and compares
  // it against what actually happened at open. This single question
  // across 50+ trades reveals whether losses stem from bad setups
  // or from trading against your own conviction.
  hypothesisIntactAtEntry  Boolean?
  hypothesisIntactAtExit   Boolean?
  hypothesisBreakReason    String?  // what specifically invalidated it

  agentReview     String
  manualNote      String?
}

// ── Briefings and Alerts ─────────────────────────────────────

model Briefing {
  id          Int      @id @default(autoincrement())
  type        String   // "morning" | "evening"
  date        DateTime
  content     String
  createdAt   DateTime @default(now())
}

model Alert {
  id           Int      @id @default(autoincrement())
  type         String   // "sl_hit" | "trailing_sl" | "news" | "setup_found" | "token_expired" | "stale_setup" | "order_mismatch"
  message      String
  instrumentId String?
  tradeId      Int?     // NEW: FK to Trade for SL/target/trailing alerts
  createdAt    DateTime @default(now())
  isRead       Boolean  @default(false)
  sentToTelegram Boolean @default(false)
}
```

---

## 6. The Agent System

Each agent follows the same loop — this is the agentic pattern:

```
1. Build a system prompt (role + context)
2. Give Claude a set of "tools" it can call
3. Send user message to Claude
4. Claude responds with either:
   a. A text answer → done
   b. A tool_call → run the tool, send result back to Claude
5. Repeat step 4 until Claude gives a final text answer
```

### Base Agent (shared by all agents)

```typescript
// src/agents/base-agent.ts

import Anthropic from "@anthropic-ai/sdk";
import { Tool, ToolResult } from "@/types/agent";

const claude = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

export async function runAgent({
  systemPrompt,
  userMessage,
  tools,
  toolHandlers,
  model = "claude-sonnet-4-5",
  maxIterations = 10,
}: {
  systemPrompt: string;
  userMessage: string;
  tools: Anthropic.Tool[];
  toolHandlers: Record<string, (input: unknown) => Promise<unknown>>;
  model?: string;
  maxIterations?: number;
}): Promise<string> {
  const messages: Anthropic.MessageParam[] = [
    { role: "user", content: userMessage },
  ];

  for (let i = 0; i < maxIterations; i++) {
    const response = await claude.messages.create({
      model,
      max_tokens: 4096,
      system: systemPrompt,
      tools,
      messages,
    });

    if (response.stop_reason === "end_turn") {
      const textBlock = response.content.find((b) => b.type === "text");
      return textBlock?.text ?? "";
    }

    if (response.stop_reason === "tool_use") {
      messages.push({ role: "assistant", content: response.content });

      const toolResults: Anthropic.ToolResultBlockParam[] = [];
      for (const block of response.content) {
        if (block.type === "tool_use") {
          const handler = toolHandlers[block.name];
          if (!handler) throw new Error(`No handler for tool: ${block.name}`);
          const result = await handler(block.input);
          toolResults.push({
            type: "tool_result",
            tool_use_id: block.id,
            content: JSON.stringify(result),
          });
        }
      }

      messages.push({ role: "user", content: toolResults });
    }
  }

  throw new Error("Agent exceeded max iterations");
}
```

---

## 7. All Agents — Detailed Specification

### Agent 1: Regime Agent
**Runs:** Evening, before scanner
**Purpose:** Classifies current market condition — trending up, trending down, ranging, or distribution. All other agents use this output to calibrate their logic.

**Tools it uses:**
- `get_nifty_ohlcv(timeframe, days)`
- `get_sector_performance()`
- `get_advance_decline()`
- `get_india_vix()`

**Output:** A `RegimeReport` persisted to the `MarketRegime` table:
```typescript
{
  trend: "up" | "down" | "sideways",
  strength: number,      // Nifty ADX
  vixLevel: number,
  breadthScore: number,  // advance/decline
  summary: string        // plain English paragraph
}
```

---

### Agent 2: Scanner Agent
**Runs:** Every evening after 3:45 PM IST
**Purpose:** Scans NSE 500 + watchlist for valid swing setups using 10 orthogonal indicators. Returns a shortlist ranked by indicator confidence score.

**Tools it uses:**
- `get_instrument_list(universe)`
- `get_ohlcv(symbol, timeframe, days)`
- `compute_indicators(symbol, timeframe)`
- `check_fno_ban(symbol)`
- `check_corporate_events(symbol, days_ahead)`
- `get_delivery_data(symbol)`
- `get_liquidity_data(symbol)` — **NEW:** filters stocks below ₹30 Cr avg daily turnover
- `get_promoter_pledge(symbol)` — **NEW:** flags stocks with promoter pledge > 50%
- `check_index_rebalancing_risk(symbol)` — **NEW:** skips stocks facing imminent Nifty 500 exclusion
- `save_setup(setupData)`

**Setup types it detects:**
- Breakout from consolidation (price + volume)
- Pullback to rising EMA 20 in uptrend
- Inside bar after a strong move
- Higher high / higher low structure resumption
- Relative strength breakout vs. Nifty

**Additional filters applied (hard exclusions):**
- Average daily turnover < ₹30 Cr → skip
- Promoter pledge > 50% → skip (structural risk overrides technical signal)
- F&O ban → skip
- Corporate results in < 5 days → skip
- Index exclusion in next rebalancing → skip

---

### Agent 2b: Vision Agent
**Runs:** Immediately after Scanner Agent (still part of the evening scan job)
**Purpose:** Renders each shortlisted setup's chart as a PNG and sends it to Claude Opus for qualitative pattern assessment. Acts as a quality gate.

**Chart renderer now produces candlestick + volume:**
The chart sent to Vision must include:
- Candlestick bars (not a line chart) — pattern recognition requires candle bodies
- Volume histogram subplot below the price chart
- EMA 20, EMA 50, EMA 200 overlays
- 120 daily candles (~6 months)

```typescript
// src/agents/vision-agent.ts
export async function runVisionAgent(
  symbol: string,
  candles: OHLCV[],
  setupType: string,
  indicatorSnapshot: IndicatorSnapshot
): Promise<VisionVerdict> {

  // Step 1: Render candlestick + volume chart to PNG
  const pngBase64 = await renderChartToPng(candles, {
    chartType: "candlestick",   // CHANGED from "line"
    overlays: ["ema20", "ema50", "ema200"],
    showVolume: true,           // NEW: volume subplot required
    height: 700,                // taller to accommodate volume panel
    width: 1000,
    periods: 120,
  });

  // Step 2: Call Claude Opus (vision) — single pass, no tool loop
  const response = await claude.messages.create({
    model: "claude-opus-4-5",
    max_tokens: 1024,
    messages: [{
      role: "user",
      content: [
        { type: "image", source: { type: "base64", media_type: "image/png", data: pngBase64 } },
        { type: "text", text: buildVisionPrompt(symbol, setupType, indicatorSnapshot) },
      ],
    }],
  });

  // Step 3: Parse and validate with Zod
  const raw = response.content[0].type === "text" ? response.content[0].text : "";
  const parsed = VisionVerdictSchema.safeParse(JSON.parse(raw));

  if (!parsed.success) {
    // Retry once with explicit correction instruction
    const retryResponse = await claude.messages.create({
      model: "claude-opus-4-5",
      max_tokens: 1024,
      messages: [{
        role: "user",
        content: `Your previous response was not valid JSON matching the required schema.
Errors: ${JSON.stringify(parsed.error.issues)}
Respond ONLY with a corrected JSON object matching the schema exactly.
Previous response: ${raw}`,
      }],
    });
    const retryRaw = retryResponse.content[0].type === "text" ? retryResponse.content[0].text : "";
    return VisionVerdictSchema.parse(JSON.parse(retryRaw));
  }

  return parsed.data;
}
```

**VisionVerdict schema (with Zod validation):**

```typescript
import { z } from "zod";

export const VisionVerdictSchema = z.object({
  pattern_confirmed: z.boolean(),
  pattern_type: z.enum([
    "flat_base", "cup_handle", "ascending_triangle", "bull_flag",
    "inside_bar", "pullback_ema", "breakout", "none"
  ]),
  pattern_quality: z.number().min(1).max(10),
  consolidation_tightness: z.number().min(1).max(10),
  trendline_integrity: z.number().min(1).max(10),
  volume_character: z.enum(["contracting", "expanding", "neutral"]),
  character: z.enum(["momentum_setup", "mean_reversion_setup", "unclear"]),
  key_concern: z.string().nullable(),
  vision_score: z.number().min(0).max(100),
  one_line_verdict: z.string().max(100),
});

export type VisionVerdict = z.infer<typeof VisionVerdictSchema>;
```

**Score blending (unchanged):**

```typescript
function computeFinalScore(indicatorScore: number, visionScore: number): number {
  if (visionScore < 40) return visionScore * 0.5;
  return (indicatorScore * 0.6) + (visionScore * 0.4);
}
```

---

### Agent 2c: Watchlist Score Agent (NEW)
**Runs:** Every evening after Vision Agent completes (part of EOD scan job)
**Purpose:** Computes a composite score (0–100) for every stock on your watchlist, measuring structural strength and proximity to a tradeable setup. Gives you a ranked view of which watchlist stocks are heating up.

**Score components:**

| Component | Weight | What it measures |
|---|---|---|
| Weekly trend strength | 40% | EMA alignment on weekly + weekly ADX |
| RS vs Nifty | 30% | Relative strength vs. index over 3 months |
| Setup readiness proximity | 30% | How close is price to forming a valid setup |

**Setup readiness proximity logic:**
- Price within 2% of EMA 20 with contraction → high proximity
- BB width in lowest quartile of last 6 months → high proximity
- RSI cooling from > 60 to 45–55 range in an uptrend → high proximity

**Tools it uses:**
- `get_ohlcv(symbol, "week", 52)` — 52 weeks of weekly data
- `compute_indicators(symbol, "week")` — weekly indicator snapshot
- `compute_indicators(symbol, "day")` — daily indicator snapshot
- `get_nifty_ohlcv("week", 52)` — for RS calculation
- `update_watchlist_score(instrumentId, score, detail)` — persists to DB

**Output per watchlist stock:**
```typescript
interface WatchlistScoreDetail {
  weeklyTrendScore: number;     // 0–100
  rsVsNiftyScore: number;       // 0–100
  setupProximityScore: number;  // 0–100
  compositeScore: number;       // 0–100 (weighted blend)
  scoredAt: Date;
  weeklyTrend: "up" | "down" | "sideways";
  weeklyAdx: number;
  rsValue: number;              // e.g. 1.14 = 14% better than Nifty
  proximityReason: string;      // human-readable reason for proximity score
}
```

---

### Agent 3: Plan Agent
**Runs:** On-demand (when you click "Generate Plan" on a setup)
**Purpose:** Takes a detected setup and produces a complete, executable trade plan — **including a 2–3 sentence setup hypothesis that bridges your watchlist thesis to the current technical opportunity.**

**Tools it uses:**
- `get_ohlcv(symbol, timeframe, days)`
- `compute_indicators(symbol, timeframe)`
- `get_account_balance()`
- `get_open_positions()`
- `get_open_risk()`
- `get_portfolio_heat_by_sector()` — **NEW:** checks sector concentration
- `calculate_position_size(entry, sl, riskPercent, capital)`
- `get_watchlist_hypothesis(symbol)` — **NEW:** reads your structural thesis from WatchlistItem
- `get_recent_bse_news(symbol, days)` — **NEW:** checks last 48h announcements
- `save_trade_plan(planData)`

**Hypothesis drafting logic:**

The Plan Agent's system prompt instructs it to draft a `setupHypothesis` in exactly 2–3 sentences using the following inputs, in priority order:

1. Your watchlist hypothesis (the structural reason you've been watching this stock)
2. What the indicator snapshot shows right now (RSI cooldown, volume contraction, EMA proximity)
3. What the Vision verdict found (pattern type, quality, key concern)
4. Any relevant BSE/news context from the last 48 hours

The hypothesis must answer two questions in plain language:
- **"Why is this setup consistent with (or a risk to) my original thesis?"**
- **"What would invalidate this trade tomorrow morning?"**

**Example output:**

```
Setup: Pullback to EMA 20 — TATAMOTORS

Hypothesis (agent-drafted):
  "CV cycle recovery remains intact — delivery data shows sustained FII
  accumulation over the last 3 weeks, and the stock continues to outperform
  Nifty by 14% over 3 months (RS: 1.14). Daily RSI has cooled from 72 to 52
  during the pullback on contracting volume, which is exactly the setup the
  watchlist thesis was waiting for. Invalidated if price opens below ₹798
  (original swing low) or if Nifty gaps down more than 1.5% tomorrow."

Entry Zone: ₹820 – ₹828
Stop Loss:  ₹798 (below Aug 14 swing low, 2.8% risk)
Target 1:   ₹865 (1:1.6 R:R)
Target 2:   ₹910 (1:3.4 R:R)
Quantity:   36 shares
Risk:       ₹1,080 (1% of ₹1,08,000 capital)

Portfolio check:
  Current Auto sector exposure: 18% of deployed capital
  Adding this trade: 21% — within 25% sector cap. OK.
  Portfolio heat: 4.2% — within 6% threshold. OK.
```

**Sector concentration guard:**

```typescript
const sectorExposure = await toolHandlers.get_portfolio_heat_by_sector({});
const currentSectorPct = sectorExposure[instrument.sector] ?? 0;
const newSectorPct = currentSectorPct + (riskAmount / totalCapital) * 100;

if (newSectorPct > SECTOR_CAP_PCT) {
  // Recommend half-size, flag for manual review
  quantity = Math.floor(quantity / 2);
  agentReasoning += `\n⚠ Sector cap warning: ${instrument.sector} exposure
    would reach ${newSectorPct.toFixed(1)}% with full size. Recommending
    half-size (${quantity} shares). Override by editing quantity before approval.`;
}
```

**UI gate — hypothesis required before plan approval:**

The Plan Approval UI (`TradePlanCard.tsx`) blocks the "Approve" button until `setupHypothesis` is non-empty. The hypothesis field is editable inline. If you agree with the agent's draft, leave it as-is. If not, edit it to reflect your actual conviction. The `hypothesisEditedByUser` flag records whether you changed it.

---

### Agent 4: News Agent (sole intraday agent)
**Runs:** Every 30 minutes during market hours (9:15 AM – 3:30 PM IST)
**Purpose:** The only agent running during market hours. Monitors BSE/NSE
announcements and news for open positions, gtt_placed setups, and watchlist
stocks. SL/target execution is handled by the Kite GTT postback webhook —
not by any polling agent. Trailing SL decisions are made at EOD on the daily
chart — not intraday.

**Tools it uses:**
- `fetch_bse_announcements(symbols)` — uses last trading session close as boundary
- `search_financial_news(symbol)`
- `classify_news_impact(headline, symbol)`
- `get_open_trade_symbols()` — open trades only
- `get_gtt_placed_symbols()` — setups with entry GTT live
- `get_watchlist_symbols()` — active watchlist
- `create_alert(type, message, symbol, tradeId)` — tradeId populated for trade alerts
- `cancel_entry_gtt(entryGttId)` — only called at 9:15 AM first pass for expired setups

**Logic — 9:15 AM first pass (opening gap check):**
```
For each setup with status = "gtt_placed":
  Fetch opening price via kite.getLTP(symbol)
  If opening price is OUTSIDE entry zone:
    Cancel entry GTT → kite.deleteGTT(entryGttId)
    Setup status → "expired"
    Telegram: "[symbol] opened outside entry zone. GTT cancelled."
```

**Logic — every 30 min pass:**
```
Fetch BSE announcements for all three symbol groups
Classify each announcement
HIGH impact on open trade → immediate Telegram alert with tradeId
HIGH impact on gtt_placed setup → Telegram alert, status → "stale_pending_review"
                                   (GTT NOT auto-cancelled — you decide)
MEDIUM/LOW → log to DB, include in evening briefing
```

**Filters out:** AGM notices, routine filings, standard board meeting intimations.
**Flags:** Pledge changes, earnings releases, promoter activity, SEBI orders,
block deals > 1% of equity.

---

*(Agent numbering note: News Agent is now Agent 4 — the sole intraday agent. Agents 5–7 below are renumbered accordingly.)*

---

### Agent 6: Briefing Agent
**Runs:** 8:45 AM IST (morning) and 4:00 PM IST (evening)
**Purpose:** Writes your daily market briefing. The morning brief now also triggers the Pre-Market Staleness Check and computes the Morning Readiness Score for each pending candidate.

**Morning tools:**
- `get_sgx_nifty()`
- `get_global_indices()`
- `get_crude_gold_usd()`
- `get_todays_events()`
- `get_fii_dii_data()`
- `get_open_trades()`
- `get_pending_setups()` — **UPDATED:** returns setups with `vision_done` or `planned` status
- `get_india_vix()` — **NEW:** for VIX-based staleness check
- `get_bse_overnight_announcements(symbols)` — **NEW:** checks news since 3:30 PM yesterday
- `compute_morning_readiness_score(setupId)` — **NEW:** see scoring section

**Evening tools:**
- `get_nifty_eod()`
- `get_sector_performance()`
- `get_scanner_results()`
- `get_trade_updates()`
- `check_upcoming_events()`
- `get_vision_verdict_summary()` — includes Vision ratification summary

---

### Agent 7: Journal Agent
**Runs:** Every Sunday at 9:00 AM
**Purpose:** Reviews all trades closed in the past week. Now evaluates whether the setup hypothesis was intact at entry and exit — the single most important question for improving decision quality.

**Tools it uses:**
- `get_closed_trades(startDate, endDate)`
- `get_ohlcv(symbol, timeframe, days)`
- `get_setup_data(tradeId)` — includes `setupHypothesis`
- `get_market_regime(date)` — **NEW:** fetches `MarketRegime` record for trade date
- `compute_statistics(trades)`
- `save_journal_entries(entries)`

**Hypothesis evaluation logic:**

For each closed trade, the Journal Agent:
1. Re-reads the `setupHypothesis` that was set at plan approval time
2. Looks at what actually happened at market open (gap, volume, first-hour price action from OHLCV)
3. Determines: was the hypothesis's invalidation condition triggered at open?
4. Records `hypothesisIntactAtEntry` (Boolean) and `hypothesisBreakReason` (String)

**Output per trade (extended):**
```
Trade: HDFCBANK — Closed at loss (−0.8R)
Hypothesis at entry:
  "Banking sector rotation continues with FII net buying for 5 consecutive
  sessions. RSI pullback to 52 on falling volume near EMA 20 — the setup
  the thesis was waiting for. Invalidated if Nifty opens down > 1% or
  stock gaps below ₹1,628."

Hypothesis intact at entry? NO
  Reason: Nifty opened down 1.3% that morning. The invalidation condition
  in the hypothesis was triggered before market open. The trade should not
  have been entered. This is a DISCIPLINE error, not a SETUP error.

Setup was valid:     YES
Entry execution:     POOR — entered above plan zone (₹1,642 vs ₹1,630–1,638)
Exit execution:      CORRECT — respected SL at ₹1,598
Mistake tag:         HYPOTHESIS_IGNORED / EARLY_ENTRY
Regime at entry:     Sideways (ADX 19, VIX 14.2)
```

**Weekly summary extended metrics:**
- Win rate by regime type (trending vs. sideways)
- % of losing trades where hypothesis was NOT intact at entry (discipline metric)
- % of winning trades where hypothesis WAS intact at entry (validation metric)

---

## 8. Workflows

### Workflow 1: Evening Scan (Daily, Automated)

```
[node-cron: 3:45 PM IST]
        │
        ▼
[BullMQ: eod-scan job]
        │
        ├─► Token Health Check
        │       Call kite.getProfile() — if 403, alert + abort
        │       Check redis kite:token:valid flag
        │
        ├─► Market Holiday Check
        │       If NSE holiday → log + abort
        │
        ├─► Fetch OHLCV for NSE 500 via Kite Historical API
        │       (batch requests, 300ms delay, daily + weekly candles)
        │
        ├─► Store candles in PostgreSQL
        │
        ├─► Run Indicator Engine on all 500 symbols
        │       (10 indicators, daily + weekly snapshots)
        │
        ├─► Run Regime Agent
        │       → persist RegimeReport to MarketRegime table
        │
        ├─► Run Scanner Agent
        │       (apply all hard filters including liquidity, pledge, rebalancing)
        │       (score every symbol → shortlist ~15–25 setups)
        │       (save each with status = "vision_pending")
        │
        ├─► Vision Cost Guard
        │       Check Redis key vision:calls:YYYY-MM-DD
        │       If daily count + shortlist.length > VISION_MAX_SETUPS_PER_RUN → abort Vision
        │
        ├─► Run Vision Agent on each shortlisted setup
        │       For each setup:
        │         render candlestick + volume chart PNG
        │         call Claude Opus vision model
        │         validate VisionVerdict with Zod (retry once on failure)
        │         increment vision:calls:YYYY-MM-DD in Redis
        │         compute finalScore = 0.6 * indicatorScore + 0.4 * visionScore
        │         if visionScore < 40 → mark "vision_vetoed"
        │         else → mark status = "vision_done"
        │
        ├─► Run Watchlist Score Agent
        │       For each active watchlist stock:
        │         compute composite score (weekly trend 40% + RS 30% + proximity 30%)
        │         persist to WatchlistItem.watchlistScore + watchlistScoreDetail
        │
        ├─► EOD Position Review (for all open trades)
        │       For each open trade:
        │         fetch daily OHLCV → detect new daily swing low above currentSL
        │         if found → create pending TrailingSlSuggestion record
        │         check if target1 was hit today (close >= target1)
        │         if hit → create pending PartialExitSuggestion record
        │       All suggestions surfaced on Positions page and Dashboard
        │       Included in evening Telegram briefing
        │
        ├─► Run Briefing Agent (evening mode)
        │       (includes Vision summary + top watchlist score movers
        │        + pending trailing SL and exit suggestions)
        │
        └─► Send Telegram:
                "Scan done. X setups passed Vision. Y vetoed.
                 Top watchlist movers: [symbol list with scores]
                 [If any] Trailing SL suggestions: [symbol list]
                 [If any] Target 1 hit today: [symbol list]"
```

---

### Workflow 2: Morning Briefing + Staleness Check + Readiness Score + GTT Placement

```
[node-cron: 8:45 AM IST]
        │
        ▼
[BullMQ: morning-brief job]
        │
        ├─► Token Health Check
        │       If expired → Telegram alert with re-auth URL → abort
        │
        ├─► Market Holiday Check
        │       If NSE holiday → skip
        │
        ├─► Briefing Agent fetches global cues, events, FII data
        │
        ├─► Pre-Market Staleness Check
        │   Runs on every setup with status = "plan_confirmed"
        │   (only setups where you have already confirmed the hypothesis
        │    in the evening session are eligible for GTT placement)
        │   │
        │   For each plan_confirmed setup:
        │     ├─► Check SGX Nifty gap direction vs. setup direction
        │     │       Bullish setup + Nifty gap down > 0.8% → penalty −15 pts
        │     │
        │     ├─► Check overnight BSE/NSE announcements for that symbol
        │     │       Material announcement found → HARD VETO (−30 pts)
        │     │       Hard veto = never auto-place GTT, always requires
        │     │       manual review regardless of final score
        │     │
        │     └─► Check India VIX spike overnight
        │             VIX move > 15% → penalty −10 pts across all setups
        │
        ├─► Compute Morning Readiness Score for each plan_confirmed setup
        │       morningReadinessScore = finalScore − totalPenalty
        │       totalPenalty = sum of applicable penalties (max −40 pts)
        │       Persist to Setup.morningReadinessScore + morningScoreDetail
        │
        ├─► GTT Placement Decision (per setup):
        │   │
        │   ├─► HARD VETO (BSE material announcement present)?
        │   │       → Do NOT place GTT
        │   │       → Setup status = "stale_pending_review"
        │   │       → Telegram: "GTT NOT placed for [symbol].
        │   │                    Reason: BSE announcement overnight.
        │   │                    Review manually before entering."
        │   │
        │   ├─► morningReadinessScore >= 50 AND no hard veto?
        │   │       → Auto-place single-leg ENTRY GTT at Kite
        │   │           trigger: price >= entryZoneHigh
        │   │           order: BUY LIMIT, CNC, quantity from plan
        │   │       → Save entryGttId to TradePlan
        │   │       → Setup status = "gtt_placed"
        │   │       → Telegram: "GTT placed for [symbol].
        │   │                    Entry: ₹[price]. MR Score: [score]/100."
        │   │
        │   └─► morningReadinessScore < 50 AND no hard veto?
        │           → Do NOT auto-place GTT
        │           → Setup status = "stale_pending_review"
        │           → Telegram: "GTT NOT placed for [symbol].
        │                        MR Score: [score]/100 (below threshold).
        │                        Review or skip."
        │
        │   Note: actual opening gap check (did stock open outside entry
        │   zone?) runs at 9:15 AM via the News Agent's first pass — not
        │   here. Market has not opened yet at 8:45 AM.
        │
        ├─► Fallback — if morning-brief job fails before 9:00 AM:
        │       → Telegram: "⚠ Morning brief failed. GTTs not placed.
        │                    Review approved plans manually before entering."
        │       → All plan_confirmed setups remain in that status
        │       → Dashboard shows "Place pending GTTs" manual trigger button
        │
        ├─► Save morning Briefing to DB (includes readiness score table
        │   and GTT placement outcomes per setup)
        │
        └─► Telegram: morning brief summary
                "GTTs placed (2): TATAMOTORS ✓ 71pts, INFY ✓ 68pts
                 GTTs held (1):   HDFCBANK ✗ BSE announcement — review
                 Expired (1):     CIPLA ✗ 43pts — below threshold"
```

**Morning Readiness Score — penalty weights:**

All three conditions use data available before market open at 9:15 AM IST.
Penalties are subtracted from finalScore (not multiplied). Max combined penalty: −40 pts.

| Condition | Penalty | Veto type | Data source |
|---|---|---|---|
| Nifty gap direction contradicts setup | −15 pts | Soft — GTT placed if score still ≥ 50 | SGX Nifty pre-open futures (labelled as estimate in UI) |
| Material overnight BSE/NSE announcement | −30 pts | Hard — GTT never auto-placed | BSE filing feed (since last trading session close, not fixed 48h) |
| India VIX spike overnight > 15% | −10 pts | Soft — applied to all setups | India VIX prior-session close from MarketRegime table |

Example: finalScore 78, BSE announcement → morningReadinessScore 48, hard veto, GTT not placed.
Example: finalScore 78, Nifty gap contradicts → morningReadinessScore 63, soft penalty, GTT placed.
Example: finalScore 78, Nifty gap + VIX spike → morningReadinessScore 53, GTT placed (score ≥ 50).
Example: finalScore 62, Nifty gap + VIX spike → morningReadinessScore 37, GTT not placed (score < 50).

**What does NOT belong here — actual opening gap check:**
Once market opens at 9:15 AM, the News Agent's first polling pass checks whether each
gtt_placed setup's stock opened outside the entry zone (via Kite live quote). If so, the
entry GTT is cancelled via `kite.deleteGTT(entryGttId)`, setup is marked `expired`,
and a Telegram alert is sent. This is a real price check, not a pre-market estimate.

---

### Workflow 3: Trade Plan — Two-Stage Approval

The approval process is split across two sessions — evening and morning.
No GTT is placed in the evening. The GTT is only placed after you have
seen the Morning Readiness Score and the system has validated overnight
conditions. This is the correct decision sequence for a swing trader.

#### Stage 1 — Evening: Plan Review and Hypothesis Confirmation

```
[Evening — you open a setup with status = "vision_done"]
        │
        ▼
[You click "Generate Plan"]
        │
        ▼
[POST /api/agent/plan]
        │
        ▼
[Plan Agent runs]
        │   reads: indicators + Vision verdict + watchlist hypothesis
        │         + BSE news since last session close
        │   drafts: setupHypothesis (2–3 sentences)
        │   checks: sector exposure + portfolio heat
        │
        ▼
[Trade Idea Detail page — plan displayed with full reasoning]
        │
        ├─► Score row:
        │       Evening Final Score (indicator sub-score + Vision sub-score)
        │       Morning Readiness Score: "Computed tomorrow at 8:45 AM" ⏳
        │
        ├─► Hypothesis shown in editable text field
        │       (agent-drafted, provenance label shown)
        │       Confirm/Edit hypothesis to proceed
        │
        ├─► Risk panel: entry zone, SL, targets, quantity, sector check
        │
        └─► "Confirm Plan" button
                DISABLED until hypothesis field is non-empty
                Label: "Confirm Plan for Tomorrow"
                (NOT "Approve" — no GTT is placed yet)
                Sub-label: "GTT will be placed tomorrow morning
                            after the readiness check at 8:45 AM"

[You read hypothesis → edit if needed → click "Confirm Plan"]
        │
        ▼
[Setup status → "plan_confirmed"]
[TradePlan status → "plan_confirmed"]
[No Kite API call made — no GTT placed]
        │
        ▼
[Telegram: "Plan confirmed for [symbol].
            GTT will be placed tomorrow at 8:45 AM
            if Morning Readiness Score passes threshold."]
```

#### Stage 2 — Morning: Readiness Review and GTT Placement

```
[8:45 AM — Morning Readiness Score computed for all plan_confirmed setups]
        │
        ▼
[You receive Telegram morning brief with GTT placement outcomes]
        │
        ├─► GTT placed automatically for setups that passed
        │       (morningReadinessScore ≥ 50, no hard veto)
        │
        └─► For setups with hard veto or score below threshold:
                → App shows "Review Required" card on Dashboard
                → You can:
                    a) Manually trigger GTT after reviewing the reason
                       [POST /api/kite/gtt/place] — manual override
                    b) Skip the setup → status = "rejected"

[Morning: if you want to review before GTT fires]
        │
        ▼
[Dashboard → Today's Plan section]
        │
        ├─► Each plan_confirmed setup now shows:
        │       Evening Final Score (unchanged from last night)
        │       Morning Readiness Score (just computed)
        │       GTT status: "Placed ✓" or "Held — review" or "Expired"
        │       Penalty breakdown on hover
        │
        └─► No action required for auto-placed GTTs
            Manual review required only for held/stale setups
```

---

### Workflow 4: Intraday Monitoring (Lean — Push-Based)

There is no 5-minute polling loop. GTT orders execute automatically at
the exchange level without server involvement. The only intraday jobs
are the News Agent (30-min polling) and the Kite GTT postback webhook
(push-based). This keeps the system consistent with the EOD philosophy.

#### 4a: Opening Gap Check (9:15 AM — first News Agent pass)

```
[node-cron: 9:15 AM IST — first News Agent pass after market open]
        │
        ▼
[For each setup with status = "gtt_placed":]
        │
        ├─► Fetch actual opening price via kite.getLTP(symbol)
        │
        ├─► If opening price is OUTSIDE entry zone:
        │       Cancel entry GTT → kite.deleteGTT(entryGttId)
        │       Setup status → "expired"
        │       Telegram: "[symbol] opened outside entry zone.
        │                  Entry GTT cancelled. Setup expired."
        │
        └─► If opening price is INSIDE entry zone or below entry:
                GTT remains live, no action needed
```

#### 4b: News Agent (every 30 min, 9:15 AM – 3:30 PM IST)

```
[node-cron: every 30 min during market hours]
        │
        ▼
[BullMQ: news-agent job]
        │
        ├─► fetch_bse_announcements for:
        │       all open trade symbols (status = "open")
        │       all gtt_placed setup symbols
        │       all active watchlist symbols
        │
        ├─► classify_news_impact for each announcement
        │
        ├─► HIGH impact on open trade:
        │       Telegram alert immediately (with tradeId)
        │       Log Alert with tradeId FK
        │
        ├─► HIGH impact on gtt_placed setup:
        │       Telegram alert: "[symbol] material news while GTT live.
        │                        Review before entry executes."
        │       Setup status → "stale_pending_review"
        │       Note: GTT is NOT automatically cancelled — you decide
        │
        └─► MEDIUM/LOW: log to DB, include in evening briefing
```

#### 4c: GTT Postback Webhook (real-time, push-based)

```
[Kite executes entry GTT → POST /api/kite/postback]
        │
        ▼
[Verify postback checksum]
        │
        ▼
[Identify trade by entryGttId → match against TradePlan]
        │
        ▼
[STEP 1 — IMMEDIATE: Place single-leg SL GTT]
        kite.placeGTT({
          trigger_type: "single",
          trigger_values: [trade.stopLoss],
          order: SELL MARKET, CNC, quantity
        })
        │
        ├─► SL GTT placed successfully:
        │       Save slGttId to Trade record
        │       Trade status → "open"
        │       TradePlan status → "executed"
        │       Setup status → "traded"
        │
        └─► SL GTT placement FAILED:
                Trade status → "open_unprotected"
                Telegram: "⚠ CRITICAL: [symbol] entry triggered at
                           ₹[price] but SL GTT FAILED to place.
                           Place stop loss manually NOW at ₹[SL price]."
                Dashboard: persistent red banner until SL is confirmed
                [This is the highest-priority failure state in the system]
        │
        ▼
[STEP 2: Update DB]
        TradePlan status → "executed" (if SL placed)
        Trade.entryDate = now(), Trade.entryPrice = filled price
        Recalculate portfolio heat
        │
        ▼
[STEP 3: Send Telegram confirmation]
        "TATAMOTORS entry triggered at ₹828.
         SL GTT placed at ₹798. Risk: ₹1,080."

[Kite executes SL GTT → POST /api/kite/postback]
        │
        ▼
[Identify trade by slGttId]
        │
        ├─► Update Trade: status = "closed", exitPrice, exitDate, exitReason = "stoploss"
        ├─► Create TradeExit record
        ├─► Recalculate portfolio heat
        └─► Telegram: "[symbol] SL hit at ₹[price]. −[R]R. Trade closed."
```

---

### Workflow 5: Order Reconciliation (Daily Safety Net)

The GTT postback webhook is the primary trade lifecycle update mechanism.
This job is a safety net only — it catches discrepancies if a postback
was missed due to server downtime or network failure. It runs after
market close, before the evening scan, so DB state is correct when
the Plan Agent calculates portfolio heat for new positions.

```
[node-cron: 3:35 PM IST — after market close]
        │
        ▼
[BullMQ: order-reconciliation job]
        │
        ▼
[Fetch all trades from DB where status = "open" or "open_unprotected"]
        │
        ▼
[Fetch all GTT orders from kite.getGTTs()]
[Fetch all regular orders from kite.getOrders() for today]
        │
        ▼
[For each DB trade — reconcile against Kite:]
        │
        ├─► DB status = "open", slGttId present in Kite as active GTT?
        │       → All clear, log pass
        │
        ├─► DB status = "open" but SL GTT not found in Kite?
        │       → Webhook likely missed SL execution
        │       → Check kite.getOrders() for a SELL order on this symbol today
        │       → If SELL order found: close trade in DB, record exitPrice + exitReason
        │       → Alert: "[symbol] SL execution detected via reconciliation
        │                  (postback missed). Trade closed."
        │
        ├─► DB status = "open_unprotected"?
        │       → Attempt to place SL GTT now if market still open
        │       → If market closed: flag for manual review tomorrow morning
        │       → Telegram: "⚠ [symbol] still unprotected at EOD. 
        │                     Place SL GTT manually before tomorrow's open."
        │
        ├─► DB status = "gtt_placed" but entry GTT not found in Kite?
        │       → Entry may have triggered and postback was missed
        │       → Check kite.getOrders() for a BUY order on this symbol today
        │       → If BUY found: create Trade record, attempt SL GTT placement
        │       → Alert: "Entry detected via reconciliation for [symbol]."
        │
        ├─► Partial fill detected (filled qty < plan qty)?
        │       → Update Trade.quantity to actual filled quantity
        │       → Recalculate Trade.riskAmount
        │       → Alert: "[symbol] partial fill: [X] of [Y] shares filled."
        │
        └─► All clear → log reconciliation pass, timestamp to Redis
```

---

### Workflow 6: Weekly Journal Review

```
[node-cron: Sunday 9:00 AM IST]
        │
        ▼
[Journal Agent fetches all closed trades (past 7 days)]
        │
        ▼
[For each trade:]
        │   re-examine entry, exit, plan vs actual
        │   re-read setupHypothesis
        │   fetch MarketRegime for trade date
        │   evaluate hypothesisIntactAtEntry + hypothesisIntactAtExit
        │
        ▼
[Generate per-trade critique + weekly stats]
        │   including: discipline metric (% losses where hypothesis was ignored)
        │
        ▼
[Save JournalEntry records to DB]
        │
        ▼
[Telegram: "Weekly review ready.
  Win rate: X% | Profit factor: Y
  Discipline score: Z% (trades where hypothesis was intact at entry)
  Regime breakdown: X% wins in trending, Y% in sideways"]
```

---

### Workflow 7: Kite Token Lifecycle (NEW, Daily)

```
[node-cron: 7:30 AM IST — every trading day]
        │
        ▼
[BullMQ: token-check job]
        │
        ▼
[Call kite.getProfile() with stored access_token]
        │
        ├─► 200 OK → token valid → log + proceed
        │
        └─► 403 / error → token expired
                │
                ▼
            [Alert via Telegram]
                "Kite token expired. Re-authenticate before 9:15 AM.
                 Link: https://kite.zerodha.com/connect/login?api_key=YOUR_KEY"
                │
                ▼
            [Set flag in Redis: kite:token:valid = false]
            [All EOD/monitor jobs check this flag before running]

[After you re-authenticate via /api/kite/callback]
        │
        ▼
[Persist new access_token to KiteSession table]
        │
        ▼
[Set Redis flag: kite:token:valid = true]
        │
        ▼
[Telegram: "Kite token refreshed successfully."]
```

---
## 9. API Layer

| Method | Route | Purpose |
|---|---|---|
| `GET` | `/api/market/ohlcv?symbol=NSE:RELIANCE&tf=day&days=365` | Fetch candle data |
| `GET` | `/api/market/quote?symbols=NSE:RELIANCE,NSE:TCS` | Live quotes |
| `POST` | `/api/agent/scan` | Trigger scanner agent manually |
| `POST` | `/api/agent/vision` | Trigger Vision ratification for a setup |
| `POST` | `/api/agent/plan` | Generate trade plan + hypothesis for a setup |
| `POST` | `/api/agent/plan/:id/confirm` | Evening gate — confirm hypothesis, status → plan_confirmed. No GTT placed. |
| `POST` | `/api/agent/chat` | Chat with agent about any stock |
| `GET` | `/api/agent/review` | Get latest journal review |
| `POST` | `/api/agent/morning-score` | Recompute Morning Readiness Score on demand |
| `GET` | `/api/kite/auth` | Initiate Kite login flow |
| `GET` | `/api/kite/callback` | Handle Kite OAuth callback + persist token |
| `POST` | `/api/kite/gtt/place` | Place entry GTT manually (override or fallback) |
| `DELETE` | `/api/kite/gtt/:gttId` | Cancel a GTT (e.g. setup expired at open) |
| `POST` | `/api/kite/postback` | Kite GTT execution webhook — primary trade lifecycle handler |
| `GET` | `/api/kite/positions` | Fetch open positions from Kite |
| `POST` | `/api/jobs/trigger` | Manually trigger any background job |
| `GET` | `/api/health` | System health: DB, Redis, Kite token, last job timestamps |

---

## 10. Frontend Pages and Components

### Pages

| Route | Page | Description |
|---|---|---|
| `/` | Dashboard | P&L summary, open positions, latest alerts, market regime badge |
| `/briefing` | Daily Briefing | Morning/evening briefing with Morning Readiness Score table |
| `/scanner` | Scanner Results | Setups with finalScore, morningReadinessScore, hypothesis preview |
| `/scanner/:id` | Setup Detail | Full chart + indicators + Vision verdict + editable hypothesis + Plan button |
| `/watchlist` | Watchlist Manager | Stocks with watchlistScore, hypothesis field, score breakdown |
| `/positions` | Open Positions | Open trades with live P&L, current SL, pending agent suggestions |
| `/journal` | Trade Journal | Closed trades with agent review, hypothesis integrity, performance stats |
| `/strategy-lab` | Strategy Lab | Backtest a setup rule against historical data |

### Key Components

```
components/
├── charts/
│   ├── CandleChart.tsx          # TradingView Lightweight Charts wrapper
│   └── IndicatorPanel.tsx       # RSI, MACD, Volume below main chart
├── agents/
│   ├── AgentReasoningCard.tsx   # Claude's reasoning
│   ├── VisionVerdictCard.tsx    # Vision score, pattern type, key concern
│   ├── TradePlanCard.tsx        # Plan with hypothesis editor + Approve/Reject
│   └── ChatWidget.tsx           # Ask Claude anything about a stock
├── scoring/
│   ├── WatchlistScoreCard.tsx   # Composite score with sub-score breakdown  ← NEW
│   ├── MorningReadinessCard.tsx # Morning score with penalty breakdown       ← NEW
│   └── ScoreBadge.tsx           # Reusable coloured score pill (0–100)       ← NEW
├── hypothesis/
│   ├── HypothesisEditor.tsx     # Inline editable text field + save          ← NEW
│   └── HypothesisGate.tsx       # Blocks plan approval if hypothesis empty   ← NEW
├── positions/
│   ├── PositionRow.tsx          # Single position with live P&L
│   └── SLUpdatePrompt.tsx       # Agent SL suggestion with approve button
├── scanner/
│   ├── SetupCard.tsx            # Setup card with finalScore + morningScore   ← UPDATED
│   └── SetupFilters.tsx         # Filter by setup type, sector, score, vision
├── layout/
│   ├── Sidebar.tsx
│   ├── AlertBell.tsx
│   └── RegimeBadge.tsx
└── shared/
    ├── ConfirmDialog.tsx
    └── RiskMeter.tsx
```

### Scanner Results Page — Score Display

The `/scanner` page shows two score columns side by side for each setup:

```
┌──────────┬────────────┬─────────────────────────────────────────────┐
│  Symbol  │ Eve Score  │ Morning Score │ Hypothesis preview           │
├──────────┼────────────┼───────────────┼──────────────────────────────┤
│ TATAMOTORS│    75     │    75  ✓      │ "CV cycle recovery intact…"  │
│ HDFCBANK │    71     │    46  ↓ -25  │ "Banking FII inflow thesis…" │
│ INFY     │    68     │    68  ✓      │ "IT sector rotation play…"   │
└──────────┴────────────┴───────────────┴──────────────────────────────┘
  ✓ = no staleness penalties   ↓ -25 = gap-outside-entry-zone penalty
```

### Watchlist Page — Score Display

```
┌──────────┬──────────────┬──────────────────────────────────────────┐
│  Symbol  │ Wlist Score  │ Your hypothesis (editable)               │
├──────────┼──────────────┼──────────────────────────────────────────┤
│ TATAMOTORS│  82  ↑      │ "CV cycle recovery, FII accumulation…"  │
│ CIPLA    │  74          │ "Pharma export recovery underway…"       │
│ IDFCFIRSTB│ 61  ↓      │ "Waiting for NIM improvement signal…"   │
└──────────┴──────────────┴──────────────────────────────────────────┘
  ↑/↓ = score moved vs. last week's computation
```

---

## 11. Data Pipeline

### Getting OHLCV Data from Kite

```typescript
// src/services/kite.ts

import KiteConnect from "kiteconnect";

export function getKiteClient() {
  return new KiteConnect({ api_key: process.env.KITE_API_KEY! });
}

export async function fetchOHLCV(
  kite: KiteConnect,
  symbol: string,
  days: number = 365,
  timeframe: "day" | "week" | "15minute" = "day"
) {
  const to = new Date();
  const from = new Date();
  from.setDate(from.getDate() - days);

  return kite.getHistoricalData(
    await getInstrumentToken(symbol),
    timeframe,
    from,
    to,
    false
  );
}
```

### Rate Limiting Strategy

```typescript
async function fetchWithRateLimit<T>(
  tasks: (() => Promise<T>)[],
  delayMs = 300
): Promise<T[]> {
  const results: T[] = [];
  for (const task of tasks) {
    results.push(await task());
    await new Promise(r => setTimeout(r, delayMs));
  }
  return results;
}
```

### Kite Login Flow + Token Persistence

```typescript
// src/app/api/kite/callback/route.ts

export async function GET(request: Request) {
  const { searchParams } = new URL(request.url);
  const requestToken = searchParams.get("request_token");

  const kite = getKiteClient();
  const session = await kite.generateSession(
    requestToken!,
    process.env.KITE_API_SECRET!
  );

  // Persist to DB — survives PM2 restarts
  await prisma.kiteSession.create({
    data: {
      accessToken: session.access_token,
      refreshedAt: new Date(),
      expiresAt: getNextDayAt6AM(),   // tokens expire at 6 AM IST
    }
  });

  // Also set in Redis for fast checks
  await redis.set("kite:access_token", session.access_token);
  await redis.set("kite:token:valid", "true");

  await sendTelegram("Kite token refreshed successfully.");
  return Response.redirect("/");
}
```

---

## 12. Indicator Engine — 10 Orthogonal Indicators

**Design philosophy:** Every indicator answers a distinct question. Beyond 10 orthogonal representatives, you are recomputing the same answer with different smoothing. Redundant indicators produce correlated false confidence.

**What was cut and why:**
- Stochastic, Williams %R → both are normalised momentum oscillators like RSI. Same information.
- OBV → cumulative, lags significantly. Volume ratio captures the same signal more directly.
- Keltner Channel → shares the volatility band role with Bollinger. Pick one.
- VWAP → useful intraday, not meaningful on daily/weekly swing timeframe.
- ROC → MACD histogram already encodes acceleration. Redundant.

```typescript
// src/indicators/engine.ts

export interface IndicatorSnapshot {
  symbol: string;
  timestamp: Date;
  timeframe: string;   // "day" | "week"

  // 1–3: Trend
  trend: {
    ema20: number;
    ema50: number;
    ema200: number;
    adx: number;
    plusDI: number;
    minusDI: number;
    macdLine: number;
    macdSignal: number;
    macdHist: number;
    supertrend: { value: number; direction: "up" | "down" };
  };

  // 4: Momentum
  momentum: {
    rsi14: number;
    rsVsNifty: number;   // > 1.0 = outperforming index
  };

  // 5–6: Volatility
  volatility: {
    atr14: number;
    atrPercent: number;
    bbUpper: number;
    bbMiddle: number;
    bbLower: number;
    bbWidth: number;   // (upper-lower)/middle — low = compression
  };

  // 7: Volume
  volume: {
    volumeRatio: number;          // today / 20d avg
    deliveryPercent: number | null;
  };

  // 8: Structure
  structure: {
    lastSwingHigh: number;
    lastSwingLow: number;
    nearestResistance: number;
    nearestSupport: number;
    priceVsEma20Pct: number;
  };
}
```

The `computeAllIndicators` function accepts a `timeframe` parameter. The EOD scan now calls it twice per symbol — once with daily candles (`"day"`) and once with weekly candles (`"week"`). The weekly snapshot is stored in `Setup.weeklySnapshot` and passed to the Plan Agent for additional context.

---

## 13. Vision AI Ratification Layer

### Chart Renderer — Candlestick + Volume

The chart renderer **must produce candlestick bars and a volume subplot**, not a line chart. Pattern recognition (inside bar, bull flag, cup-and-handle) is meaningless on a line chart.

```typescript
// src/vision/chart-renderer.ts

import { ChartJSNodeCanvas } from "chartjs-node-canvas";
import { OHLCV } from "@/types/market";

const renderer = new ChartJSNodeCanvas({
  width: 1000,
  height: 700,   // taller to fit volume subplot
  backgroundColour: "#111"
});

export async function renderChartToPng(
  candles: OHLCV[],
  options: {
    ema20: number[];
    ema50: number[];
    ema200: number[];
    chartType: "candlestick";
    showVolume: boolean;
  }
): Promise<string> {
  const labels = candles.map(c =>
    new Date(c.timestamp).toLocaleDateString("en-IN")
  );

  // Price panel (top 70% of canvas)
  // Volume panel (bottom 30% of canvas)
  // Both rendered via chartjs-chart-financial plugin

  const config = {
    type: "candlestick" as const,
    data: {
      labels,
      datasets: [
        {
          label: "Price",
          data: candles.map(c => ({
            x: new Date(c.timestamp).getTime(),
            o: c.open, h: c.high, l: c.low, c: c.close,
          })),
          color: {
            up: "#26A69A",    // teal for bullish candles
            down: "#EF5350",  // red for bearish candles
            unchanged: "#888",
          },
        },
        { label: "EMA 20",  data: options.ema20,  borderColor: "#F6C90E", type: "line", borderWidth: 1, pointRadius: 0 },
        { label: "EMA 50",  data: options.ema50,  borderColor: "#26A69A", type: "line", borderWidth: 1, pointRadius: 0 },
        { label: "EMA 200", data: options.ema200, borderColor: "#EF5350", type: "line", borderWidth: 1, pointRadius: 0 },
      ],
    },
    options: {
      plugins: { legend: { labels: { color: "#CCC" } } },
      scales: {
        x: { ticks: { color: "#999", maxTicksLimit: 12 } },
        y: { ticks: { color: "#999" }, position: "right" },
        volume: {
          type: "linear",
          position: "left",
          max: Math.max(...candles.map(c => c.volume)) * 4, // compress volume to bottom 25%
          ticks: { color: "#666" },
          grid: { display: false },
        },
      },
    },
  };

  const buffer = await renderer.renderToBuffer(config);
  return buffer.toString("base64");
}
```

### Vision Prompt

```typescript
// src/vision/vision-prompt.ts

export function buildVisionPrompt(
  symbol: string,
  setupType: string,
  indicators: IndicatorSnapshot
): string {
  return `
You are a professional technical analyst reviewing an NSE equity chart.
The chart shows candlestick price bars (not a line chart) with a volume histogram below.

Symbol: ${symbol}
Setup type detected by scanner: ${setupType}
Indicator context:
  - Trend: EMA 20 ${indicators.trend.ema20 > indicators.trend.ema50 ? "above" : "below"} EMA 50
  - RSI 14: ${indicators.momentum.rsi14.toFixed(1)}
  - ADX: ${indicators.trend.adx.toFixed(1)}
  - Volume ratio vs 20d avg: ${indicators.volume.volumeRatio.toFixed(2)}x

Analyse the chart image carefully and respond with ONLY a valid JSON object.
No preamble, no markdown fences, no explanation outside the JSON.

{
  "pattern_confirmed": true | false,
  "pattern_type": "flat_base | cup_handle | ascending_triangle | bull_flag | inside_bar | pullback_ema | breakout | none",
  "pattern_quality": 1-10,
  "consolidation_tightness": 1-10,
  "trendline_integrity": 1-10,
  "volume_character": "contracting | expanding | neutral",
  "character": "momentum_setup | mean_reversion_setup | unclear",
  "key_concern": "string — single biggest risk visible in the chart, or null",
  "vision_score": 0-100,
  "one_line_verdict": "max 20 words"
}

Scoring guide for vision_score:
  80–100: Textbook setup, clean structure, high conviction
  60–79:  Good setup with minor flaws
  40–59:  Mixed signals, proceed with caution
  0–39:   Poor quality, invalidates indicator signal
`;
}
```

### Setup Detail Page — Score Display

The `/scanner/:id` page shows three cards:

```
┌─────────────────────────┐  ┌─────────────────────────┐
│  INDICATOR SIGNALS       │  │  VISION VERDICT          │
│                         │  │                         │
│  Trend:    ▲ Uptrend    │  │  Pattern: Bull flag  ✓  │
│  ADX:      31           │  │  Quality:     8 / 10    │
│  MACD:     +ve          │  │  Tightness:   7 / 10    │
│  RSI:      54           │  │  Volume: Contracting     │
│  ATR:      ₹18 (1.9%)   │  │  Concern: None          │
│  BB Width: 0.04         │  │                         │
│  Vol ratio: 1.8x        │  │  Vision score: 78/100   │
│  RS vs Nifty: 1.14      │  │  "Clean bull flag,      │
│                         │  │   strong vol contraction"│
│  Indicator score: 72    │  └─────────────────────────┘
└─────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  HYPOTHESIS                                                  │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ CV cycle recovery remains intact — FII accumulation   │  │
│  │ visible in delivery data for 3 weeks, RS vs Nifty     │  │
│  │ 1.14. RSI cooled to 52 on contracting volume near     │  │
│  │ EMA 20 — the setup this thesis was waiting for.       │  │
│  │ Invalidated if open below ₹798 or Nifty gaps -1.5%.   │  │
│  └────────────────────────────────────────────────────────┘  │
│  [Edit]  ✎ Agent drafted · not yet edited by you            │
│                                                              │
│  Eve score: 75/100    Morning score: 75/100  ✓              │
│                                                              │
│  [Generate Plan →]  (enabled — hypothesis present)          │
└──────────────────────────────────────────────────────────────┘
```

---

## 14. Zerodha Kite Integration

### Order Types Used

| Order Type | When Used | How |
|---|---|---|
| `GTT` (Good-Till-Triggered) | Entry orders placed after market hours | `kite.placeGTT()` |
| `GTT` | Stop-loss orders | `kite.placeGTT()` — one-cancels-other |
| `LIMIT` | Intraday entry during market hours | `kite.placeOrder()` |
| `SL-M` | Panic exit if SL breached with no GTT | `kite.placeOrder()` |

### GTT Structure — Two Separate Single-Leg Orders

**Why not two-leg GTT?**
Kite's `trigger_type: "two-leg"` is designed for an already-open position
where you want a target above and a stop below your current holding.
For a pending entry, the lower leg (SELL MARKET at stopLoss) would attempt
to short-sell shares you do not yet own — which is not permitted for CNC
delivery trades on NSE. The correct approach is two sequential single-leg GTTs.

#### Step 1 — Morning (8:45 AM, after readiness check passes)

Place a single-leg entry GTT. Called from the morning-brief worker.

```typescript
// src/services/kite.ts

async function placeEntryGTT(plan: TradePlan, currentPrice: number): Promise<string> {
  const response = await kite.placeGTT({
    trigger_type: "single",
    tradingsymbol: plan.instrument.symbol,
    exchange: "NSE",
    trigger_values: [plan.entryZoneHigh],
    last_price: currentPrice,
    orders: [{
      transaction_type: "BUY",
      quantity: plan.quantity,
      product: "CNC",
      order_type: "LIMIT",
      price: plan.entryZoneHigh,
    }],
  });
  // Returns the GTT trigger ID — save as TradePlan.entryGttId
  return response.trigger_id;
}
```

#### Step 2 — On Entry Execution (postback webhook fires)

Called immediately as the first action inside the postback handler,
before any DB update or Telegram notification. An open position without
a stop loss is the only unrecoverable failure mode in the system.

```typescript
// src/app/api/kite/postback/route.ts

async function handleEntryExecution(plan: TradePlan, filledPrice: number) {

  // STEP 1 — FIRST: place SL GTT before anything else
  let slGttId: string | null = null;
  try {
    const response = await kite.placeGTT({
      trigger_type: "single",
      tradingsymbol: plan.instrument.symbol,
      exchange: "NSE",
      trigger_values: [plan.stopLoss],
      last_price: filledPrice,
      orders: [{
        transaction_type: "SELL",
        quantity: plan.quantity,
        product: "CNC",
        order_type: "MARKET",
        price: 0,
      }],
    });
    slGttId = response.trigger_id;
  } catch (error) {
    // SL GTT failed — this is a critical state requiring immediate human action
    await prisma.trade.create({
      data: {
        planId: plan.id,
        instrumentId: plan.instrumentId,
        entryGttId: plan.entryGttId,
        slGttId: null,
        entryDate: new Date(),
        entryPrice: filledPrice,
        quantity: plan.quantity,
        side: "BUY",
        status: "open_unprotected",   // highest priority alert state
        stopLoss: plan.stopLoss,
        currentSL: plan.stopLoss,
        target1: plan.target1,
        target2: plan.target2 ?? null,
      }
    });
    await sendTelegram(
      `⚠ CRITICAL: ${plan.instrument.symbol} entry triggered at ₹${filledPrice} ` +
      `but SL GTT FAILED to place. ` +
      `Place stop loss manually NOW at ₹${plan.stopLoss}.`
    );
    return; // exit early — DB and Telegram handled above
  }

  // STEP 2: SL GTT placed successfully — update DB
  await prisma.trade.create({
    data: {
      planId: plan.id,
      instrumentId: plan.instrumentId,
      entryGttId: plan.entryGttId,
      slGttId,
      entryDate: new Date(),
      entryPrice: filledPrice,
      quantity: plan.quantity,
      side: "BUY",
      status: "open",
      stopLoss: plan.stopLoss,
      currentSL: plan.stopLoss,
      target1: plan.target1,
      target2: plan.target2 ?? null,
    }
  });
  await prisma.tradePlan.update({
    where: { id: plan.id },
    data: { status: "executed" }
  });

  // STEP 3: Recalculate portfolio heat
  await recalculatePortfolioHeat();

  // STEP 4: Telegram confirmation
  await sendTelegram(
    `${plan.instrument.symbol} entry triggered at ₹${filledPrice}. ` +
    `SL GTT placed at ₹${plan.stopLoss}. Risk: ₹${plan.riskAmount}.`
  );
}
```

#### Trailing SL Update (EOD, after your approval)

When you approve a trailing SL suggestion from the EOD position review,
cancel the existing SL GTT and place a new one at the higher level.

```typescript
async function updateTrailingSL(trade: Trade, newSL: number, currentPrice: number) {
  // Cancel existing SL GTT
  await kite.deleteGTT(trade.slGttId!);

  // Place new SL GTT at higher trailing level
  const response = await kite.placeGTT({
    trigger_type: "single",
    tradingsymbol: trade.instrument.symbol,
    exchange: "NSE",
    trigger_values: [newSL],
    last_price: currentPrice,
    orders: [{
      transaction_type: "SELL",
      quantity: trade.quantity,
      product: "CNC",
      order_type: "MARKET",
      price: 0,
    }],
  });

  // Update DB
  await prisma.trade.update({
    where: { id: trade.id },
    data: {
      currentSL: newSL,
      slGttId: response.trigger_id,
    }
  });

  await sendTelegram(
    `Trailing SL updated for ${trade.instrument.symbol}: ` +
    `₹${trade.currentSL} → ₹${newSL}.`
  );
}
```

### GTT Postback Webhook

Configure the postback URL in your Kite app dashboard:
`https://yourdomain.com/api/kite/postback`

Kite signs every postback with a checksum. Always verify before processing:

```typescript
// src/app/api/kite/postback/route.ts

export async function POST(request: Request) {
  const body = await request.json();

  // 1. Verify Kite checksum
  const checksum = generateChecksum(body, process.env.KITE_API_SECRET!);
  if (checksum !== body.checksum) {
    return Response.json({ error: "Invalid checksum" }, { status: 401 });
  }

  const triggerId = body.trigger_id;
  const status = body.status; // "triggered" | "cancelled" | "expired"

  if (status !== "triggered") return Response.json({ ok: true });

  // 2. Identify which GTT fired — entry or SL
  const plan = await prisma.tradePlan.findFirst({
    where: { entryGttId: triggerId },
    include: { instrument: true }
  });

  if (plan) {
    // Entry GTT fired
    await handleEntryExecution(plan, body.meta.filled_price);
    return Response.json({ ok: true });
  }

  const trade = await prisma.trade.findFirst({
    where: { slGttId: triggerId },
    include: { instrument: true }
  });

  if (trade) {
    // SL GTT fired
    await handleSLExecution(trade, body.meta.filled_price);
    return Response.json({ ok: true });
  }

  // Unknown GTT — log for reconciliation
  console.error("Unknown GTT postback:", triggerId);
  return Response.json({ ok: true });
}
```

### Graceful WebSocket Shutdown

```typescript
// src/services/ticker.ts

process.on("SIGTERM", () => {
  console.log("SIGTERM received — disconnecting Kite Ticker cleanly");
  ticker.disconnect();
  process.exit(0);
});

process.on("SIGINT", () => {
  ticker.disconnect();
  process.exit(0);
});
```

---

## 15. Scoring Reference

### Watchlist Score (0–100, computed every evening)

Answers: **"How strong is this stock's structural position and how close is it to setting up?"**

| Component | Weight | Signal used |
|---|---|---|
| Weekly trend strength | 40% | Weekly EMA alignment (20 > 50 > 200) + weekly ADX |
| RS vs Nifty (3-month) | 30% | RS value; 1.0 = inline, 1.1 = 10% outperformance |
| Setup readiness proximity | 30% | BB compression + RSI cooling in uptrend + price vs EMA 20 |

```typescript
// src/scoring/watchlist-score.ts

export function computeWatchlistScore(
  weeklySnapshot: IndicatorSnapshot,
  dailySnapshot: IndicatorSnapshot,
): WatchlistScoreDetail {
  // Weekly trend strength (0–100)
  const emaAligned =
    weeklySnapshot.trend.ema20 > weeklySnapshot.trend.ema50 &&
    weeklySnapshot.trend.ema50 > weeklySnapshot.trend.ema200;
  const weeklyTrendScore = emaAligned
    ? Math.min(100, (weeklySnapshot.trend.adx / 40) * 100)
    : Math.min(40, (weeklySnapshot.trend.adx / 40) * 40); // penalise misaligned trend

  // RS vs Nifty (0–100)
  // RS 1.2 = outperforming by 20% → maps to ~100; RS 0.8 = 20% under → maps to 0
  const rsVsNiftyScore = Math.min(100, Math.max(0, (dailySnapshot.momentum.rsVsNifty - 0.7) / 0.6 * 100));

  // Setup readiness proximity (0–100)
  const bbCompressed = dailySnapshot.volatility.bbWidth < 0.06;  // tight consolidation
  const rsiCooling = dailySnapshot.momentum.rsi14 >= 45 && dailySnapshot.momentum.rsi14 <= 58;
  const nearEma20 = Math.abs(dailySnapshot.structure.priceVsEma20Pct) < 2.0;
  const proximitySignals = [bbCompressed, rsiCooling, nearEma20].filter(Boolean).length;
  const setupProximityScore = proximitySignals * 33;   // 0, 33, 66, or 99

  const compositeScore =
    weeklyTrendScore * 0.4 +
    rsVsNiftyScore   * 0.3 +
    setupProximityScore * 0.3;

  return {
    weeklyTrendScore,
    rsVsNiftyScore,
    setupProximityScore,
    compositeScore,
    scoredAt: new Date(),
    weeklyTrend: weeklySnapshot.trend.ema20 > weeklySnapshot.trend.ema50 ? "up" : "down",
    weeklyAdx: weeklySnapshot.trend.adx,
    rsValue: dailySnapshot.momentum.rsVsNifty,
    proximityReason: [
      bbCompressed && "BB compressed",
      rsiCooling   && "RSI cooling",
      nearEma20    && "near EMA 20",
    ].filter(Boolean).join(", ") || "no proximity signals",
  };
}
```

---

### Morning Readiness Score (0–100, computed at 8:45 AM)

Answers: **"Given what changed overnight, how actionable is this setup today?"**

This is a **separate score alongside `finalScore`** — it never overwrites the original evening signal. Displaying both allows you to see exactly what changed and why.

```typescript
// src/scoring/morning-readiness-score.ts

export interface StalenessPenalty {
  condition: string;
  penalty: number;
  detail: string;
}

export function computeMorningReadinessScore(
  finalScore: number,
  penalties: StalenessPenalty[]
): { morningReadinessScore: number; totalPenalty: number; breakdown: StalenessPenalty[] } {
  const totalPenalty = Math.min(40, penalties.reduce((sum, p) => sum + p.penalty, 0));
  const morningReadinessScore = Math.max(0, finalScore - totalPenalty);
  return { morningReadinessScore, totalPenalty, breakdown: penalties };
}

// Penalty weights — all three conditions use data available before 9:15 AM open
export const STALENESS_PENALTIES = {
  NIFTY_GAP_CONTRADICTS:     15,  // Soft — GTT placed if score still >= 50
  BSE_MATERIAL_ANNOUNCEMENT: 30,  // HARD VETO — GTT never auto-placed regardless of score
  VIX_SPIKE:                 10,  // Soft — applied to all setups simultaneously
} as const;

// Max combined penalty: 40 pts. BSE_MATERIAL_ANNOUNCEMENT is a hard veto —
// GTT not auto-placed even if score stays above 50.
// The actual opening gap check (price opened outside entry zone) is handled by
// the News Agent first pass at 9:15 AM using real Kite LTP data — not here.
```

---

## 16. Authentication and Security

```typescript
// src/app/api/auth/[...nextauth]/route.ts

providers: [
  CredentialsProvider({
    credentials: { password: { type: "password" } },
    authorize(credentials) {
      if (credentials.password === process.env.APP_PASSWORD) {
        return { id: "1", name: "Trader" };
      }
      return null;
    }
  })
]
```

Protect all API routes:
```typescript
const session = await getServerSession();
if (!session) return Response.json({ error: "Unauthorized" }, { status: 401 });
```

### Health Check Endpoint

```typescript
// src/app/api/health/route.ts

export async function GET() {
  const checks = {
    postgres:     await checkPostgres(),
    redis:        await checkRedis(),
    kiteToken:    await redis.get("kite:token:valid") === "true",
    lastEodScan:  await getLastJobTimestamp("eod-scan"),
    lastMorningBrief: await getLastJobTimestamp("morning-brief"),
  };

  const healthy = Object.values(checks).every(Boolean);
  return Response.json(checks, { status: healthy ? 200 : 503 });
}
```

---

## 17. Environment Variables

```bash
# .env.local

# App
APP_PASSWORD=your_secure_password
NEXTAUTH_SECRET=generate_with_openssl_rand_-base64_32
NEXTAUTH_URL=http://localhost:3000

# Database
DATABASE_URL=postgresql://postgres:password@localhost:5432/swingtrader

# Redis
REDIS_URL=redis://localhost:6379

# Claude AI
ANTHROPIC_API_KEY=sk-ant-...
# Text agents:   claude-sonnet-4-5
# Vision agent:  claude-opus-4-5

# Zerodha Kite
KITE_API_KEY=your_kite_api_key
KITE_API_SECRET=your_kite_api_secret
KITE_REDIRECT_URL=http://localhost:3000/api/kite/callback

# Telegram
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_personal_chat_id

# Vision cost guard
VISION_MAX_SETUPS_PER_RUN=30

# Scoring thresholds (configurable without code changes)
WATCHLIST_SCORE_SECTOR_CAP_PCT=25       # max % of deployed capital in one sector
PORTFOLIO_HEAT_MAX_PCT=6                # max total risk across all open trades
LIQUIDITY_MIN_TURNOVER_CR=30           # min avg daily turnover in crores
PROMOTER_PLEDGE_MAX_PCT=50             # hard filter: skip stocks above this pledge %
MORNING_READINESS_STALE_THRESHOLD=20   # flag as stale if morningScore drops > this vs finalScore
VIX_SPIKE_THRESHOLD_PCT=15             # VIX overnight change that triggers staleness penalty
NIFTY_GAP_STALENESS_PCT=0.8            # Nifty gap % that triggers direction-contradiction check
```

---

## 18. Build and Run

### Prerequisites
- Node.js 20+
- Docker Desktop (for PostgreSQL + Redis)

### Step 1: Start infrastructure
```bash
docker-compose up -d
```

```yaml
# docker-compose.yml
version: "3.8"
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: swingtrader
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

### Step 2: Install and migrate
```bash
npm install
npx prisma migrate dev --name init
npx prisma generate
```

### Step 3: Seed instrument list
```bash
npx tsx scripts/seed-instruments.ts
```

### Step 4: Run development
```bash
npm run dev
```

### Step 5: Production
```bash
npm run build
pm2 start ecosystem.config.js
```

```javascript
// ecosystem.config.js
module.exports = {
  apps: [
    {
      name: "swing-trader",
      script: "node_modules/.bin/next",
      args: "start",
      env: { NODE_ENV: "production", PORT: 3000 }
    },
    {
      name: "swing-trader-worker",
      script: "src/jobs/run-workers.ts",
      interpreter: "npx tsx",
      kill_timeout: 5000,   // allows graceful WebSocket disconnect
    }
  ]
};
```

---

## 19. Phase-wise Development Plan

### Phase 1 — Data + Indicators (Weeks 1–2)
Goal: Fetch data, compute 10 indicators, see them working correctly.

- [ ] Set up Next.js project, Prisma, PostgreSQL, Redis
- [ ] Kite authentication flow + KiteSession persistence
- [ ] OHLCV fetch for a single symbol (daily + weekly)
- [ ] Indicator engine — all 10 indicators, both timeframes
- [ ] Simple UI: enter a symbol, see chart with indicators
- [ ] Background job: daily OHLCV refresh for 50 watchlist stocks
- [ ] `/api/health` endpoint

**Milestone:** Any NSE stock shows all 10 indicators correctly on daily and weekly timeframes.

---

### Phase 2 — Scanner + Vision + Plan Agent + Hypothesis (Weeks 3–6)
Goal: Agent finds setups, Vision ratifies them, hypothesis is drafted, you confirm in the evening.

- [ ] Scanner Agent with 3 setup types + all hard filters (liquidity, pledge, ex-dividend, rebalancing)
- [ ] Setup storage and scanner results page
- [ ] Candlestick + volume chart renderer (model-agnostic PNG)
- [ ] Vision model evaluation — label 30–50 setups manually, test Gemini Flash, GPT-4o, Claude Sonnet
- [ ] Vision Agent — selected model → Zod-validated VisionVerdict with retry
- [ ] Score blending — finalScore = 0.6 × indicator + 0.4 × vision; hard veto < 40
- [ ] Setup detail page — indicator signals + Vision verdict + hypothesis editor
- [ ] Plan Agent — entry/SL/target/quantity + agent-drafted hypothesis (2–3 sentences)
- [ ] HypothesisGate component — blocks "Confirm Plan" until hypothesis non-empty
- [ ] Evening plan confirmation UI — "Confirm Plan for Tomorrow" button
  (NO GTT placed yet — status → "plan_confirmed")
- [ ] Watchlist Score Agent + WatchlistScoreCard component
- [ ] Telegram alerts for scan complete with watchlist score movers
- [ ] Token lifecycle worker + daily token check at 7:30 AM
- [ ] Monthly loss cap enforcement — block plan confirmation when cap reached

**Milestone:** Evening scan produces Vision-ratified setups with agent-drafted hypotheses.
You can confirm plans in the evening. No GTT is placed at this stage.

---

### Phase 3 — Morning Score + GTT Lifecycle + Briefing (Weeks 7–8)
Goal: Morning readiness scoring drives GTT placement. Complete trade lifecycle end-to-end.

- [ ] Pre-Market Staleness Check in morning-brief worker
- [ ] Morning Readiness Score computation + MorningReadinessCard component
- [ ] Hard veto / soft penalty distinction (BSE announcement = hard, others = soft)
- [ ] Auto GTT placement in morning-brief worker for setups passing threshold (score ≥ 50, no hard veto)
- [ ] Single-leg entry GTT placement — `placeEntryGTT()` function
- [ ] Manual GTT placement override — POST /api/kite/gtt/place for held/stale setups
- [ ] Kite GTT postback webhook — /api/kite/postback
  - SL GTT placed immediately on entry (first action, before DB update)
  - open_unprotected status + critical Telegram alert on SL GTT failure
  - Trade record created with entryGttId + slGttId
- [ ] Opening gap check at 9:15 AM in News Agent first pass
  - Cancel entry GTT + mark expired if stock opened outside entry zone
- [ ] News Agent — 30-min BSE announcement polling during market hours
- [ ] Order reconciliation job at 3:35 PM (safety net for missed postbacks)
  - Partial fill detection and quantity correction
  - open_unprotected end-of-day handling
- [ ] EOD position review in evening scan — trailing SL + target suggestions
- [ ] Trailing SL update — cancel old SL GTT + place new one at higher level
- [ ] Morning + evening briefing agent
- [ ] Scanner results page updated with dual score columns
- [ ] Alert center in UI with tradeId linkage
- [ ] Dashboard: open_unprotected persistent red banner
- [ ] Dashboard: "Place pending GTTs" manual trigger button (morning brief fallback)

**Milestone:** Complete trade lifecycle end-to-end. Evening confirmation → morning readiness
check → GTT placed automatically → entry fires → SL GTT placed immediately → news
monitored intraday → trade closed via postback → EOD trailing SL suggestions.

---

### Phase 4 — Journal + Intelligence (Weeks 9–10)
Goal: The app helps you improve as a trader.

- [ ] Journal Agent — weekly review with hypothesis integrity evaluation
- [ ] Journal page with per-trade review, hypothesis intact flag, mistake tags
- [ ] Performance statistics (win rate, R:R, profit factor)
- [ ] Regime-correlated stats (win rate in trending vs. sideways markets)
- [ ] Discipline metric: % of losses where hypothesis was not intact at entry
- [ ] Track Vision accuracy — did high-Vision-score setups outperform?
- [ ] TradeExit records for partial exits
- [ ] Chat widget — ask Claude anything about any stock
- [ ] Strategy Lab — basic backtest of setup rules against historical data

**Milestone:** Complete trading system with hypothesis-driven accountability, regime-aware analysis, and automated improvement feedback.

---

## 20. Key Design Decisions and Rationale

**Why 10 indicators, not 25?**
Every indicator is derived from price and/or volume. Beyond 10 orthogonal representatives — one per distinct question — you are recomputing the same answer with different smoothing. Redundant indicators produce correlated false confidence.

**Why Vision as a ratification layer, not the primary scanner?**
Vision cannot run on 500 stocks efficiently. Cost (~₹550/month at 20 setups/day) and latency (~80 seconds for 20 calls) are both acceptable on a shortlist but not on the full universe. Vision adds qualitative judgment indicators cannot — pattern quality, consolidation tightness, gestalt shape.

**Why hard-veto Vision scores below 40?**
A Vision score below 40 means chart quality is poor enough that the indicator signal is likely a false positive — a sloppy pattern that numerically scored well but visually looks wrong.

**Why two score columns (finalScore + morningReadinessScore) instead of one?**
Overwriting the evening finalScore with the morning adjustment destroys information. The evening score represents the quality of the technical setup. The morning score represents how much overnight context degraded it. Showing both lets you see exactly what changed and make an informed decision — a 75-point setup dropping to 50 due to a news announcement is very different from a 75-point setup staying at 75 with no staleness flags.

**Why is the hypothesis required before plan approval?**
The hypothesis gate forces a 30-second deliberation before each trade. It eliminates impulsive approvals driven by score numbers alone. Over time, the quality of your hypotheses — and whether you actually follow them — becomes the most valuable data in the system.

**Why does the Journal Agent evaluate hypothesis integrity, not just P&L?**
P&L tells you outcomes. Hypothesis integrity tells you process quality. A trade that was right process but bad outcome (setup was valid, hypothesis was intact, market moved against you) is very different from a trade that was bad process (hypothesis invalidated at open, you entered anyway). The discipline metric — % of losses where hypothesis was not intact at entry — directly measures whether you're trading your thesis or just trading signals.

**Why candlestick charts for Vision, not line charts?**
Candlestick pattern recognition is the primary value Vision adds over indicators. Inside bars, doji candles, bullish engulfing, hammer formations — all are invisible on a line chart. Sending Vision a line chart is like asking a doctor to diagnose from a blurred X-ray.

**Why the watchlist score runs every evening, not just on Sundays?**
A stock's setup proximity changes daily. A stock that was 40 points on the watchlist score yesterday may jump to 78 today because BB compression hit its tightest point in 6 months. Catching this the evening it happens — not a week later — is the entire value of the watchlist score.

**Why GTT orders over WebSocket-triggered orders?**
SEBI's position on retail algo trading is nuanced — placing orders automatically via API without an exchange-approved algorithm creates regulatory exposure. GTT orders are exchange-managed and widely used by retail traders without requiring algo registration.

**Why are GTTs placed at 8:45 AM and not at plan approval time (evening)?**
Approving a plan in the evening and immediately placing a GTT means the GTT sits live overnight with no awareness of what changes between evening and the next morning open — BSE announcements, global events, VIX spikes. By the time the trader wakes up, the GTT may be armed for a setup whose thesis has been invalidated overnight. Separating plan confirmation (evening) from GTT placement (morning, after readiness check) ensures the order only goes live when overnight conditions have been validated. This also makes the Morning Readiness Score operationally meaningful — it is a decision gate that determines whether the GTT is placed, not just an informational score.

**Why two sequential single-leg GTTs instead of one two-leg GTT?**
Kite's two-leg GTT is designed for an already-open position: one leg protects the downside (stop loss), one captures the upside (target). For a pending entry, the lower leg (SELL MARKET) would attempt to short-sell shares not yet owned — which is not permitted in CNC delivery trades on NSE cash equities. The correct structure is: (1) single-leg entry GTT placed at 8:45 AM after readiness check; (2) single-leg SL GTT placed immediately inside the postback handler when the entry triggers. The SL GTT only exists after the position is open.

**Why is SL GTT placement the first action in the postback handler?**
An open position without a stop loss is the only genuinely unrecoverable failure state in the entire system. Every other failure can be corrected at EOD. An unprotected position during market hours cannot. The postback handler places the SL GTT before updating the database, before sending Telegram, before recalculating portfolio heat. If the SL GTT placement fails, the trade is immediately marked `open_unprotected` and a critical Telegram alert fires — this is the highest-priority UI state in the product.

**Why Telegram over in-app notifications only?**
During market hours you may not have the app open. Telegram delivers to your phone instantly with no app needed.

**Why the token lifecycle worker is critical?**
Zerodha access tokens expire every day at 6:00 AM IST. Without an automated check, the system goes blind every single morning. Every job that touches Kite checks the Redis flag `kite:token:valid` before executing. The system is designed to fail loudly and early rather than silently operate on stale data.
