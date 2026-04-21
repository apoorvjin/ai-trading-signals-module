# AI Trading Signals — Integration Module Specification

> **Module purpose**: A self-contained, production-grade real-time market intelligence module that delivers live prices, AI-generated trade signals (with three pluggable strategies), interactive candlestick charts, walk-forward backtesting, news-sentiment analysis, and risk-managed entry/stop/target levels for **39 globally diversified assets** spanning commodities, equity indices, and crypto.
>
> This document is the authoritative integration spec. An engineering agent can read this end-to-end and embed the module into any host application (web, mobile, desktop) without ambiguity.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Functional Capabilities](#2-functional-capabilities)
3. [System Architecture](#3-system-architecture)
4. [Asset Universe (39 Instruments)](#4-asset-universe-39-instruments)
5. [Backend API Specification](#5-backend-api-specification)
6. [Live Data Pipeline](#6-live-data-pipeline)
7. [Signal Generation Strategies](#7-signal-generation-strategies)
8. [News Sentiment Engine](#8-news-sentiment-engine)
9. [Risk Management (Entry / SL / TP)](#9-risk-management-entry--sl--tp)
10. [Backtesting Engine](#10-backtesting-engine)
11. [Frontend Module (React Native / Expo)](#11-frontend-module-react-native--expo)
12. [Type Contracts (TypeScript)](#12-type-contracts-typescript)
13. [Integration Guide](#13-integration-guide)
14. [Configuration & Secrets](#14-configuration--secrets)
15. [Tech Stack](#15-tech-stack)
16. [Operational Notes](#16-operational-notes)
17. [Repository Layout](#17-repository-layout)

---

## 1. Executive Summary

### What it is
A two-tier system:

- **API server** (Node.js + Express + TypeScript) — exposes a REST surface for live quotes, OHLCV history, AI signals, news with sentiment, and backtest results. Maintains a real-time in-memory price store fed by Yahoo Finance polling and a Finnhub WebSocket stream.
- **Mobile client** (Expo / React Native + TypeScript) — consumes the API via auto-generated React Query hooks; renders a Markets dashboard, an AI Signals screen, an Asset Detail screen with five sub-tabs (Chart, Signal, Indicators, Backtest, News), and a Strategy selector.

### Why it exists (business)
Retail and prosumer traders need a single pane of glass that:
- Shows **truly live prices** for cross-asset macro markets (no 30-second stale caches),
- Generates **explainable, multi-strategy signals** rather than a single black-box recommendation,
- Quantifies **risk per trade** via stop-loss / take-profit / R:R ratios,
- Validates strategies via **walk-forward backtests** on real historical data,
- Anchors signals in **breaking news sentiment** when relevant.

### How it integrates into a larger app
Embed it as a **microservice + UI module**. The host app:
1. Runs the API server (or proxies to a hosted instance) — see [§13](#13-integration-guide).
2. Imports the React Native screens **or** the API client hooks for a custom UI.
3. Navigates to the module via a single route / deep link (e.g. `/trading`).
4. Shares user identity / auth via a host-provided token (the module is auth-agnostic by default).

---

## 2. Functional Capabilities

| # | Capability | Description |
|---|---|---|
| F1 | Live quotes | All 39 assets, refreshed every ≤10s; crypto via WebSocket (sub-second). |
| F2 | Multi-timeframe candlestick charts | 1m, 5m, 1h, 4h, 1d with crosshair, OHLCV inspector, line/candle toggle. |
| F3 | AI signals | BUY / HOLD / SELL with confidence (0–94), per-strategy reasoning bullets. |
| F4 | Three strategies | S1 (technical), S2 (multi-factor AI), S3 (technical + news sentiment). |
| F5 | Entry / SL / TP card | Visual price ladder with risk %, target %, R:R ratio. |
| F6 | Walk-forward backtest | 60 / 30 / 10 splits, returns win-rate, P&L, max DD, Sharpe. |
| F7 | News + sentiment | Live Yahoo Finance news per asset, scored -100 ↔ +100. |
| F8 | Search & filter | By symbol, name, country, category, signal direction, timeframe. |
| F9 | Currency-correct display | Euro indices in EUR, JPY in JPY, etc. (locale-aware formatting). |
| F10 | Strategy persistence | User's chosen strategy persists via AsyncStorage (`@trading_strategy`). |

---

## 3. System Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                         HOST APP                                 │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │   AI TRADING SIGNALS MOBILE MODULE (Expo / React)       │     │
│  │  ┌───────────┐ ┌───────────┐ ┌──────────────┐           │     │
│  │  │  Markets  │ │  Signals  │ │ Asset Detail │  (Tabs)   │     │
│  │  └───────────┘ └───────────┘ └──────────────┘           │     │
│  │           ▼ React Query (auto-generated hooks)          │     │
│  └─────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
                                 │ HTTPS / JSON
                                 ▼
┌──────────────────────────────────────────────────────────────────┐
│                       API SERVER (Express)                       │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │  /quotes  /signals/:s  /history/:s  /backtest/:s  /news/:s │  │
│  └────────────────────────────────────────────────────────────┘  │
│                          ▲                                       │
│   ┌──────────────────────┴──────────────────────┐                │
│   │     latestPrices  Map<symbol, LiveQuote>    │  (in-memory)   │
│   └──────────────────────▲──────────────────────┘                │
│       ▲                  │                  ▲                    │
│       │ 10s poll         │ on tick          │ 5min cache         │
│       │                  │                  │                    │
│  ┌────┴──────┐    ┌──────┴──────┐    ┌──────┴───────┐            │
│  │   Yahoo   │    │   Finnhub   │    │  Yahoo News  │            │
│  │  Finance  │    │  WebSocket  │    │   + chart    │            │
│  │ (REST)    │    │  (crypto)   │    │   history    │            │
│  └───────────┘    └─────────────┘    └──────────────┘            │
└──────────────────────────────────────────────────────────────────┘
```

### Data flow guarantees
- **Quotes**: served from `latestPrices` map; entries older than **60 seconds** are excluded (stale-feed protection).
- **Signals**: cached **30 s** per `(symbol, timeframe)`; cache always uses the *current* price from `latestPrices`, so signal confidence/levels stay aligned with live market.
- **History (OHLCV)**: cached **5 min** per `(symbol, interval)` — historical candles are rarely revised.
- **News + sentiment**: cached **15 min** per asset — headlines do not change minute-to-minute.
- **Backtest**: cached **10 min** per `(symbol, timeframe)` — re-running every request is wasteful.

All caches are bypassable client-side via a `?fresh=<nonce>` query param.

---

## 4. Asset Universe (39 Instruments)

Defined in two synchronized files (must stay identical):
- `artifacts/api-server/src/routes/trading.ts` — `ASSETS`
- `artifacts/trading-app/utils/assets.ts` — `ASSETS_META` (adds `country`, `currency`)

### Commodities (14)
| Symbol | Name | Category |
|---|---|---|
| GC=F | Gold | commodity |
| SI=F | Silver | commodity |
| CL=F | Crude Oil | commodity |
| NG=F | Natural Gas | commodity |
| HG=F | Copper | commodity |
| PL=F | Platinum | commodity |
| PA=F | Palladium | commodity |
| ZW=F | Wheat | commodity |
| ZC=F | Corn | commodity |
| ZS=F | Soybeans | commodity |
| KC=F | Coffee | commodity |
| CC=F | Cocoa | commodity |
| SB=F | Sugar | commodity |
| CT=F | Cotton | commodity |

### Indices (15)
| Symbol | Name | Country | Currency |
|---|---|---|---|
| ^GSPC | S&P 500 | USA | USD |
| ^IXIC | NASDAQ | USA | USD |
| ^DJI | Dow Jones | USA | USD |
| ^RUT | Russell 2000 | USA | USD |
| ^GDAXI | DAX | Germany | EUR |
| ^FCHI | CAC 40 | France | EUR |
| ^FTSE | FTSE 100 | UK | GBP |
| ^N225 | Nikkei 225 | Japan | JPY |
| ^HSI | Hang Seng | Hong Kong | HKD |
| ^AXJO | ASX 200 | Australia | AUD |
| ^BSESN | BSE SENSEX | India | INR |
| ^NSEI | NIFTY 50 | India | INR |
| ^NSEBANK | NIFTY Bank | India | INR |
| ^KS11 | KOSPI | South Korea | KRW |
| ^GSPTSE | TSX Composite | Canada | CAD |

### Crypto (10)
| Yahoo Symbol | Name | Finnhub Stream Symbol |
|---|---|---|
| BTC-USD | Bitcoin | BINANCE:BTCUSDT |
| ETH-USD | Ethereum | BINANCE:ETHUSDT |
| BNB-USD | BNB | BINANCE:BNBUSDT |
| SOL-USD | Solana | BINANCE:SOLUSDT |
| XRP-USD | XRP | BINANCE:XRPUSDT |
| ADA-USD | Cardano | BINANCE:ADAUSDT |
| AVAX-USD | Avalanche | BINANCE:AVAXUSDT |
| DOGE-USD | Dogecoin | BINANCE:DOGEUSDT |
| DOT-USD | Polkadot | BINANCE:DOTUSDT |
| LINK-USD | Chainlink | BINANCE:LINKUSDT |

---

## 5. Backend API Specification

**Base URL**: configurable via the API client (`setBaseUrl`). Default in dev: `http://localhost:8080`.

All endpoints return `application/json`. All errors return `{ error: string }` with HTTP 4xx / 5xx.

### 5.1 `GET /api/quotes`
Live quotes for all 39 assets.

**Query params**: `fresh?` — any truthy value triggers an upstream refresh and a stale-protection re-check.

**Response** `200 OK` — array of `LiveQuote`:
```ts
type LiveQuote = {
  symbol: string;          // e.g. "GC=F"
  name: string;            // e.g. "Gold"
  category: "commodity" | "index" | "crypto";
  price: number;           // last trade
  change: number;          // absolute day change
  changePercent: number;   // % day change
  high: number;            // day high
  low: number;             // day low
  volume: number;          // session volume
  timestamp: string;       // ISO of upstream tick
};
```

### 5.2 `GET /api/history/:symbol`
OHLCV candles for charting.

**Path**: `symbol` (e.g. `GC=F`, URL-encoded if it contains `^`)
**Query**: `interval` ∈ `1m | 5m | 1h | 4h | 1d` (default `1d`)

Lookback windows:
| Interval | Window |
|---|---|
| 1m | 1 day |
| 5m | 5 days |
| 1h | 5 days |
| 4h | 30 days (aggregated from 1h) |
| 1d | 120 days |

**Response** `200 OK` — array of `Candle`:
```ts
type Candle = {
  time: number;   // epoch ms (UTC)
  open: number;
  high: number;
  low: number;
  close: number;
  volume: number;
};
```

### 5.3 `GET /api/news/:symbol`
Up to 8 recent headlines with sentiment.

**Response** `200 OK`:
```ts
type NewsResponse = {
  symbol: string;
  items: {
    title: string;
    publisher: string;
    publishedAt: string;     // ISO
    link: string;
    sentiment: number;       // -1 ↔ +1 per article
  }[];
  aggregateSentiment: number; // -100 ↔ +100 mean across items
};
```

### 5.4 `GET /api/signals/:symbol`
Multi-strategy AI signal.

**Query**:
- `timeframe` ∈ `1m | 1h | 4h | 1d` (default `1d`)
- `fresh?` — busts the 30-second signal cache and the quotes cache.

**Response** `200 OK` — `SignalResult` (see [§12](#12-type-contracts-typescript)).

### 5.5 `GET /api/backtest/:symbol`
Walk-forward backtest of all three strategies.

**Query**: `timeframe` ∈ `1h | 4h | 1d` (default `1d`)

**Response** `200 OK` — `BacktestResult`:
```ts
type BacktestResult = {
  symbol: string;
  timeframe: string;
  trades: number;
  wins: number;
  losses: number;
  winRate: number;          // 0–100
  totalReturnPct: number;   // cumulative %
  avgReturnPct: number;     // per trade
  maxDrawdownPct: number;
  sharpeRatio: number;
  perStrategy: {            // same shape per strategy id
    "1": StrategyStats;
    "2": StrategyStats;
    "3": StrategyStats;
  };
};
```

### 5.6 `GET /api/health`
`200 OK` → `{ status: "ok" }`. Used by deployment health checks.

---

## 6. Live Data Pipeline

### 6.1 The `latestPrices` store
A single `Map<string, LiveQuote & { updatedAt: number }>` lives in `trading.ts`. **All read endpoints read from it; nothing reads Yahoo on the hot request path.**

### 6.2 Yahoo Finance polling (always-on)
```ts
async function refreshAllPrices(): Promise<void> {
  await Promise.all(ASSETS.map(async (a) => {
    try {
      const q = await yahooFinance.quote(a.symbol, {}, { validateResult: false });
      latestPrices.set(a.symbol, { ...mapQuote(q, a), updatedAt: Date.now() });
    } catch { /* keep prior entry; updatedAt stays stale */ }
  }));
}
setInterval(refreshAllPrices, 10_000);
refreshAllPrices(); // immediate on boot
```

### 6.3 Finnhub WebSocket (sub-second crypto)
Started **only if `FINNHUB_API_KEY` is set**. Falls back to Yahoo polling otherwise.
```ts
const ws = new WebSocket(`wss://ws.finnhub.io?token=${process.env.FINNHUB_API_KEY}`);
ws.on("open", () => {
  for (const [yahoo, finnhub] of CRYPTO_SYMBOL_MAP) {
    ws.send(JSON.stringify({ type: "subscribe", symbol: finnhub }));
  }
});
ws.on("message", (raw) => {
  const msg = JSON.parse(raw.toString());
  if (msg.type !== "trade") return;
  for (const t of msg.data) {
    const yahoo = REVERSE_MAP.get(t.s);
    if (!yahoo) continue;
    const prev = latestPrices.get(yahoo);
    latestPrices.set(yahoo, { ...prev!, price: t.p, updatedAt: Date.now() });
  }
});
// Reconnect with exponential backoff: 5s → 10s → 20s → 40s → 60s (cap)
```

### 6.4 Stale-feed guard
On `GET /quotes`, entries with `Date.now() - updatedAt > 60_000` are excluded from the response. If the entire feed dies, the client sees an empty array and surfaces an "Updating…" state instead of stale prices.

---

## 7. Signal Generation Strategies

All three strategies share the same indicator pipeline (RSI, MACD, EMA12/26/50/200, Bollinger Bands, ATR, ROC) but combine outputs differently. The signal output is normalized to `BUY | HOLD | SELL` plus a confidence in `[55, 94]`.

### 7.1 Strategy 1 — Standard Technical (S1)
Score in `[-90, +90]`:
| Component | Weight | Rule |
|---|---|---|
| RSI | ±30 | Below oversold → +30; above overbought → -30. Thresholds vary by timeframe (1m: 38/62, 1h: 35/65, 4h: 32/68, 1d: 30/70). |
| MACD histogram | ±25 | Sign + magnitude vs ATR. |
| EMA cross | ±20 | EMA12 > EMA26 → +20. |
| Bollinger | ±15 | Close < lower band → +15 (mean reversion); > upper → -15. |

**Decision**:
- `score ≥ 30` → BUY
- `score ≤ -30` → SELL
- else HOLD

**Confidence**: `clamp(55 + (|score| / 90) * 39, 55, 94)`.

### 7.2 Strategy 2 — Multi-Factor AI Pattern (S2)
Weighted score (sum to 100%):
| Factor | Weight |
|---|---|
| RSI | 20% |
| MACD | 15% |
| Volume confirmation (volume > 1.5× 20-bar avg) | 15% |
| AI pattern (RSI divergence OR 4-of-5 directional bars) | 15% |
| Sentiment proxy (ROC5 + ROC20) | 10% |
| EMA trend (12 vs 26) | 10% |
| Bollinger position | 10% |
| Fundamental proxy (EMA50 vs EMA200 — golden / death cross) | 5% |

**Decision threshold**: ±25.

### 7.3 Strategy 3 — Technical + News Sentiment Hybrid (S3)
```
s3Score = 0.6 × technicalScore_S1 + 0.4 × newsSentiment100
```
Where `newsSentiment100` is the aggregate sentiment from [§8](#8-news-sentiment-engine) scaled to `[-100, +100]`.

**Decision threshold**: ±25.

### 7.4 Reasoning bullets
Each strategy returns a `reasoning: string[]` of human-readable bullets that explain *why* the signal fired (e.g. `"RSI 27.4 — oversold bounce setup"`, `"News sentiment +42 — bullish coverage"`).

---

## 8. News Sentiment Engine

### 8.1 Source
`yahooFinance.search(query, { newsCount: 10 })` where `query` is the asset's display name (e.g. "Gold", "Bitcoin"). Results are mapped to `{ title, publisher, publishedAt, link }`.

### 8.2 Keyword scorer
A lexicon of ~40 positive and ~40 negative keywords (e.g. positive: `surge, rally, breakout, bullish, soar, beat, upgrade`; negative: `slump, crash, bearish, miss, downgrade, recession, inflation`).

For each headline:
```
positiveCount = matches(positive lexicon)
negativeCount = matches(negative lexicon)
articleScore  = (positiveCount - negativeCount) / max(1, positiveCount + negativeCount)
              ∈ [-1, +1]
```

### 8.3 Aggregate
```
aggregateSentiment = mean(articleScores) × 100   // → [-100, +100]
```

This value feeds Strategy 3 directly and is surfaced in the News tab as a colored badge.

---

## 9. Risk Management (Entry / SL / TP)

Computed inside the signal endpoint and returned as part of `SignalResult` (per strategy: `entryPrice`, `stopLoss`, `takeProfit`; for S2/S3: `s2EntryPrice` etc.).

### 9.1 Risk parameters by timeframe
| Timeframe | stopPct | R:R ratio |
|---|---|---|
| 1m | 0.3% | 2.0 |
| 1h | 0.8% | 2.2 |
| 4h | 1.5% | 2.5 |
| 1d | 2.5% | 2.5 |

### 9.2 BUY signal
```
entryPrice = currentPrice
stopLoss   = max(bollingerLower, currentPrice × (1 - stopPct))
takeProfit = entryPrice + (entryPrice - stopLoss) × RR
```

### 9.3 SELL signal
```
entryPrice = currentPrice
stopLoss   = min(bollingerUpper, currentPrice × (1 + stopPct))
takeProfit = entryPrice - (stopLoss - entryPrice) × RR
```

### 9.4 HOLD signal
Returns `support` and `resistance` (recent swing low/high) instead of trade levels — the UI pivots to a "Key Levels" view.

---

## 10. Backtesting Engine

Walk-forward, no look-ahead bias.

1. Pull `N = 500` candles for the requested `(symbol, timeframe)` from Yahoo.
2. For each candle `i` from `60 → N-10`:
   - Use candles `[i-60, i]` as the indicator window.
   - Generate a signal as if we were at candle `i`.
   - On BUY/SELL, simulate entry at `candles[i].close`, then walk forward up to 10 bars or until SL/TP hits.
   - Record P&L (% of entry).
3. Aggregate per strategy → win rate, total return %, avg return %, max drawdown %, Sharpe ratio (`mean / stddev × √annualizer`).

Result is cached for **10 minutes**.

---

## 11. Frontend Module (React Native / Expo)

### 11.1 Navigation tree (Expo Router)
```
app/
├─ _layout.tsx            // Stack + StrategyContext provider + QueryClient
├─ +not-found.tsx
├─ (tabs)/
│  ├─ _layout.tsx         // Tab bar
│  ├─ index.tsx           // Markets
│  ├─ signals.tsx         // AI Signals
│  ├─ settings.tsx        // Strategy selector
│  └─ watchlist.tsx       // (hidden tab — accessible via deep link)
└─ asset/
   └─ [symbol].tsx        // Detail screen with Chart / Signal / Indicators / Backtest / News
```

### 11.2 Screen specs

#### Markets (`(tabs)/index.tsx`)
- Header with "LIVE" / "UPDATING" badge.
- BUY / HOLD / SELL summary tiles (counts across all 39 assets).
- `SearchBar` → filters by symbol / name / country.
- Category chips (All / Commodities / Indices / Crypto).
- Vertical list of `AssetCard` rows: name, symbol, price (currency-correct), %change, mini sparkline, signal badge, confidence bar.
- Pull-to-refresh sends `?fresh=<nonce>`.
- React Query: `staleTime: 10s`, `refetchInterval: 10s`.

#### Signals (`(tabs)/signals.tsx`)
- Animated **Refresh** button + active strategy badge in the header.
- Timeframe segmented control (`1m | 1h | 4h | 1d`).
- `SearchBar`, category chips, signal-direction chips (`ALL | BUY | HOLD | SELL`).
- Per-asset `SignalCard`: 3-panel score breakdown (Technical / AI Pattern / Sentiment), confidence bar, top reasoning bullet.
- `staleTime: 30s`.

#### Asset Detail (`asset/[symbol].tsx`)
Sub-tabs:
1. **Chart** — `CandlestickChart` (SVG, pan/crosshair, line/candle toggle, OHLCV inspector, timeframe selector).
2. **Signal** — `EntryExitCard` (visual ladder: Entry / TP / SL with R:R, risk %, target %, profit & risk zones), reasoning bullets, indicator score chips.
3. **Indicators** — RSI / MACD / EMA / Bollinger / ATR raw values with mini-explanations.
4. **Backtest** — win rate, total return, max DD, Sharpe; bar chart of trade returns.
5. **News** — list of headlines with publisher, time-ago, sentiment badge, "Open" button.

### 11.3 Strategy context
```ts
// context/StrategyContext.tsx
type Strategy = "1" | "2" | "3";
const StrategyContext = createContext<{
  strategy: Strategy;
  setStrategy: (s: Strategy) => void;
}>(...);

// Persisted to AsyncStorage key "@trading_strategy".
```

### 11.4 API client hooks
Auto-generated from an OpenAPI spec via Orval. Re-exported from `@workspace/api-client-react`:
- `useGetQuotes(params?: { fresh?: string })`
- `useGetSignal(symbol, params: { timeframe: string; fresh?: string })`
- `useGetHistory(symbol, params: { interval: string })`
- `useGetBacktest(symbol, params: { timeframe: string })`
- `useGetNews(symbol)`

Plus configuration helpers:
```ts
import { setBaseUrl, setAuthTokenGetter } from "@workspace/api-client-react";
setBaseUrl("https://your-api.example.com");
setAuthTokenGetter(async () => await getHostAppToken()); // optional
```

### 11.5 Currency formatting (`utils/currency.ts`)
```ts
formatPrice("^GDAXI", 22456.7) // → "22.456,70 €"
formatPrice("^N225", 39280)     // → "¥39,280"
formatPrice("BTC-USD", 67890.1) // → "$67,890"
formatPrice("ZW=F", 5.6234)     // → "$5.6234"
```
Decimal rules:
- ≥ 100,000 → 0 decimals
- 1,000–100,000 → 0 decimals + thousands sep
- 1–1,000 → 2 decimals
- < 1 → 4 decimals

---

## 12. Type Contracts (TypeScript)

Source of truth: `lib/api-client-react/src/generated/api.schemas.ts` (regenerated from OpenAPI).

```ts
export type Strategy = "1" | "2" | "3";
export type SignalDirection = "BUY" | "HOLD" | "SELL";

export interface Indicators {
  rsi: number;
  macd: { value: number; signal: number; histogram: number };
  ema12: number;
  ema26: number;
  ema50: number;
  ema200: number;
  bollinger: { upper: number; middle: number; lower: number };
  atr: number;
  roc5: number;
  roc20: number;
}

export interface SignalResult {
  symbol: string;
  name: string;
  timeframe: string;
  currentPrice: number;
  signal: SignalDirection;          // primary (per active strategy on client)
  confidence: number;               // 55–94
  reasoning: string[];
  indicators: Indicators;
  technicalScore: number;           // S1 score
  // Strategy 1
  s1Signal: SignalDirection;
  s1Confidence: number;
  s1EntryPrice: number;
  s1StopLoss: number;
  s1TakeProfit: number;
  // Strategy 2
  s2Signal: SignalDirection;
  s2Confidence: number;
  s2Score: number;
  s2EntryPrice: number;
  s2StopLoss: number;
  s2TakeProfit: number;
  // Strategy 3
  s3Signal: SignalDirection;
  s3Confidence: number;
  s3Score: number;
  s3NewsSentimentRaw: number;       // -100 ↔ +100
  s3EntryPrice: number;
  s3StopLoss: number;
  s3TakeProfit: number;
  // Levels for HOLD
  support: number;
  resistance: number;
  generatedAt: string;              // ISO
}
```

---

## 13. Integration Guide

### 13.1 Run the API server standalone
```bash
# Prereqs: Node >= 20, pnpm
pnpm install
pnpm --filter @workspace/api-server dev
# → listens on $PORT (default 8080)
```
Mount behind your gateway at `/trading/api/*` or expose directly.

### 13.2 Embed the mobile screens in an Expo host app
```bash
pnpm --filter @workspace/trading-app dev
```
Or copy the `app/(tabs)`, `app/asset/`, `components/`, `context/`, `utils/` directories into the host Expo project. Wrap your root with the providers:
```tsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { StrategyProvider } from "@/context/StrategyContext";
import { setBaseUrl } from "@workspace/api-client-react";

setBaseUrl(process.env.EXPO_PUBLIC_TRADING_API_URL!);
const qc = new QueryClient({ defaultOptions: { queries: { staleTime: 60_000, retry: 2 } } });

export default function Root() {
  return (
    <QueryClientProvider client={qc}>
      <StrategyProvider>{/* host nav incl. /trading route */}</StrategyProvider>
    </QueryClientProvider>
  );
}
```

### 13.3 Use the client hooks in a custom UI (web or mobile)
```tsx
import { useGetSignal } from "@workspace/api-client-react";

function GoldSignal() {
  const { data, isLoading } = useGetSignal("GC=F", { timeframe: "1d" });
  if (isLoading) return <Spinner />;
  return <SignalCard data={data!} />;
}
```

### 13.4 Auth (optional)
The module is auth-agnostic. To attach a host app token to every request:
```ts
import { setAuthTokenGetter } from "@workspace/api-client-react";
setAuthTokenGetter(async () => hostSession.getAccessToken());
```
Server-side, add a middleware in `artifacts/api-server/src/index.ts` that validates the token and rejects with 401 otherwise.

### 13.5 Deep-linking from the host app
Recommended scheme:
- `app://trading` → Markets tab
- `app://trading/signals?timeframe=1d&category=crypto` → Signals tab pre-filtered
- `app://trading/asset/BTC-USD` → Asset detail
- `app://trading/asset/^GSPC?tab=backtest` → Asset detail, Backtest sub-tab

---

## 14. Configuration & Secrets

| Name | Required | Purpose |
|---|---|---|
| `PORT` | yes | API server bind port. |
| `NODE_ENV` | yes | `development` / `production`. |
| `FINNHUB_API_KEY` | optional | Enables crypto WebSocket stream. Without it, all assets fall back to 10s Yahoo polling. Free tier sufficient. |
| `SESSION_SECRET` | optional | Reserved for future auth middleware. |
| `EXPO_PUBLIC_TRADING_API_URL` | yes (mobile) | Base URL the Expo client points to. |

**Never** commit secrets. Use the host platform's secret manager.

---

## 15. Tech Stack

| Layer | Technology | Notes |
|---|---|---|
| Server runtime | Node.js 20+ | ESM, TypeScript |
| Web framework | Express 4 | Lightweight, no DB required |
| Logging | Pino | Structured JSON, dev pretty-printer |
| Bundler | esbuild | Single-file output (`build.mjs`) |
| Market data — REST | `yahoo-finance2` v3 | All 39 assets |
| Market data — WS | `ws` + Finnhub WebSocket API | Crypto sub-second |
| News | `yahoo-finance2.search` | Sentiment scored locally |
| Mobile framework | Expo (React Native) | Expo Router file-based nav |
| State / data fetching | `@tanstack/react-query` v5 | Auto-generated hooks via Orval |
| Charts | `react-native-svg` | Hand-rolled candlestick + crosshair |
| Animations | `react-native-reanimated` v3 | Refresh button, transitions |
| Persistent state | `@react-native-async-storage/async-storage` | Strategy selection |
| Monorepo | pnpm workspaces | `artifacts/` + `lib/` |
| API contract | OpenAPI 3.1 → Orval codegen | `lib/api-spec/`, `lib/api-client-react/` |

---

## 16. Operational Notes

- **Yahoo Finance reliability**: free tier with no SLA. Rate limits are unofficial; stay under ~1 request/asset every 10 s. Always handle 429 / 5xx with backoff.
- **Finnhub free tier**: 60 calls / minute REST, unlimited WebSocket subscribers. Crypto symbols use the `BINANCE:*USDT` convention.
- **Server cold start**: first `/quotes` request after boot may return an empty array for ~1 s while `refreshAllPrices()` populates `latestPrices`. Clients should treat empty as "loading", not "error".
- **Time zones**: all timestamps are UTC (epoch ms internally, ISO on the wire). Render in user locale on the client.
- **Indices closed**: equity indices (e.g. `^N225`, `^GDAXI`) only update during local market hours. Their `change`/`changePercent` will be flat overnight — surface a "Market closed" hint in the UI when `Date.now() - regularMarketTime > 4h`.
- **Backtest realism**: assumes immediate fills at candle close, no slippage, no commission. Adjust if integrating with a live broker.
- **Signal disclaimer**: outputs are educational, not financial advice. The host app must surface a disclaimer before any "trade" CTA.

---

## 17. Repository Layout

```
.
├─ artifacts/
│  ├─ api-server/                  # Express API
│  │  ├─ src/
│  │  │  ├─ index.ts               # bootstrap, middleware, route mount
│  │  │  └─ routes/
│  │  │     └─ trading.ts          # ALL endpoints + signal logic
│  │  ├─ build.mjs                 # esbuild config (yahoo-finance2 + ws external)
│  │  └─ package.json
│  ├─ trading-app/                 # Expo mobile app
│  │  ├─ app/                      # Expo Router screens
│  │  ├─ components/               # CandlestickChart, EntryExitCard, etc.
│  │  ├─ context/StrategyContext.tsx
│  │  ├─ utils/{assets,currency}.ts
│  │  ├─ babel.config.js           # MUST include react-native-reanimated/plugin
│  │  └─ package.json
│  └─ mockup-sandbox/              # design preview server (optional)
├─ lib/
│  ├─ api-spec/                    # OpenAPI 3.1 source
│  ├─ api-client-react/            # generated React Query hooks
│  └─ api-zod/                     # generated zod schemas
├─ pnpm-workspace.yaml
└─ package.json
```

---

## License & Attribution

This module wraps free-tier public data sources (Yahoo Finance, Finnhub). Verify each provider's terms before commercial deployment. The signal logic, sentiment engine, backtester, and UI components are original to this project and may be re-licensed by the integrator.

---
*Module spec version 1.0 — generated from production codebase. Keep this file in sync with `artifacts/api-server/src/routes/trading.ts` and `artifacts/trading-app/`.*
