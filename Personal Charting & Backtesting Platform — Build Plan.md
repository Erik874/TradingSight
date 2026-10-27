# Personal Charting & Backtesting Platform — Build Plan

Oct 27, 2026 · @Erik

## Goal and scope

Build a personal, local-first charting and backtesting app that works like TradingView for three jobs: writing strategies in a Pine-compatible editor, backtesting them with a realistic broker emulator, and reading charts with indicators to make decisions.

**In scope**

- Candlestick charts with timeframes, panes, crosshair, drawing tools and bar replay
- A built-in indicator library (50+ classics), each with a settings dialog
- A code editor for a Pine Script v6-compatible language, with highlighting, autocomplete and inline errors
- A Pine interpreter that runs `indicator()` and `strategy()` scripts on historical bars
- A Strategy Tester: performance summary, list of trades, equity curve, trade markers on the chart
- Extras TradingView lacks or charges for: parameter optimizer, walk-forward, Monte Carlo, lower-timeframe fill simulation

**Out of scope (for now)**

- Social features, idea publishing, public script library, screeners, paid exchange data feeds
- Live order execution with real money (paper trading can come later)

**Boundaries.** This is a functional clone, not a copy: use your own app name and visual identity, and never copy TradingView source code, icons, documentation text or scraped data. Implementing an interpreter for the Pine language from its observable behavior is the plan; the reference manual is used to learn semantics, not pasted into the repo. If you use TradingView's open-source Lightweight Charts library (Apache 2.0), keep its required attribution. Pine scripts copied from the public library carry their authors' licenses (often MPL 2.0) and should be treated that way.

## Architecture and tech stack

One TypeScript monorepo, so the same Pine engine and backtester run in the browser (in a Web Worker), in Node for tests, and in a desktop shell later. The engine packages never import UI code.

&#91;embedded content: system architecture · data, engine, chart, tester\]

Bars and script code flow into the engine; its plots go to the chart and its orders to the backtester, whose trades come back as chart markers.

| Layer | Choice | Why |
| --- | --- | --- |
| Repo and tooling | pnpm workspaces, TypeScript strict, Vitest, ESLint, Prettier | One language end to end; fast tests |
| Web app | React + Vite + Zustand | Simple state; fast dev loop |
| Chart rendering | Lightweight Charts v5 behind a `ChartAdapter` interface, plugins for drawings | Proven candlestick renderer; adapter lets you swap to a custom canvas later |
| Code editor | Monaco Editor with a custom Pine language (tokenizer, completion, hover, markers) | Same editor as VS Code |
| Pine engine | Hand-written lexer, parser, type checker and bar-by-bar runtime in TS, run in a Web Worker | Full control over Pine semantics; UI stays responsive |
| Backtester | TS broker emulator package, shared by UI and CLI | Same results in app and tests |
| Local server | Fastify (Node) for data download, caching and script storage | Avoids browser CORS limits on exchange APIs |
| Storage | SQLite (scripts, layouts, settings) + Parquet or DuckDB files (OHLCV) | Fast scans over millions of bars |
| Market data | Binance public klines and CCXT for crypto; CSV import; optional Alpaca, Polygon or Yahoo adapters for stocks | Free sources first; adapters behind one interface |
| Desktop (later) | Tauri | Small native app wrapping the same web build |

**Repo layout**

```text
apps/
  web/            React app: chart, editor, tester panels
  server/         Fastify: data providers, cache, storage API
  cli/            run a script/backtest headless (used by tests and the agent)
packages/
  core/           shared types: Bar, Symbol, Timeframe, Series
  data/           providers, resampling, sessions, timezones
  ta/             indicator math (pure functions, no Pine dependency)
  pine/           lexer, parser, AST, checker, runtime, builtins
  backtest/       broker emulator, order types, metrics
  chart/          ChartAdapter, panes, drawing tools, plot renderers
fixtures/         golden OHLCV data and expected outputs
docs/             PLAN.md, PROGRESS.md, COMPATIBILITY.md, decisions/
```

## Pine-compatible scripting engine

The engine is the hardest and most valuable part: a script that runs in TradingView should run here and give the same numbers, bar for bar. Target Pine v6 (accept `//@version=5` with documented differences).

**Pipeline**

1. **Lexer** — indentation-aware tokens (4-space blocks, line continuation), comments, `//@version` and annotations.
2. **Parser** — recursive descent into a typed AST with source positions for every node, so errors point at line and column.
3. **Checker** — scopes, type inference, and Pine's qualifiers (`const` < `input` < `simple` < `series`); rejects e.g. a series value where a simple one is required.
4. **Runtime** — executes the script once per bar, oldest to newest. Each series variable keeps a ring buffer so `x[n]` reads history; `var` keeps state across bars, `varip` is treated like `var` on historical data.
5. **Builtins** — namespaces implemented in order of value: `ta.*`, `math.*`, `input.*`, `plot*`/`hline`/`fill`/`bgcolor`, `strategy.*`, `str.*`, `array.*`, `map.*`, `matrix.*`, `label`/`line`/`box`/`table`/`polyline`, `request.security`, `timeframe.*`, `syminfo.*`, `time*`.
6. **Outputs** — a plain data object (plots, shapes, drawings, strategy orders, alerts) handed to the chart and the backtester; the engine never draws anything itself.

**Semantics that must be exact**

- `na` propagation through arithmetic and comparisons, `na()`, `nz()`, `fixnan()`
- Built-in `ta.*` functions keep their own internal state per call site, even inside `if` blocks (Pine warns about this; mirror the behavior)
- Warm-up: `ta.rma` and `ta.ema` seed values exactly as Pine does, or every RSI and ATR will drift
- `request.security` with `lookahead_off` by default and no access to future higher-timeframe bars
- `max_bars_back` limits, `barstate.*`, `bar_index`, `last_bar_index`
- Runtime errors (division semantics, array out of bounds) reported with the bar index where they happened

**How compatibility is proven**

- Golden tests: for each built-in and each sample script, store an OHLCV fixture plus expected values. Expected values come from (a) TA-Lib or pandas-ta for plain indicators and (b) TradingView itself: run the script on TradingView, use "Export chart data" to CSV, and commit that CSV as the reference.
- `docs/COMPATIBILITY.md` lists every built-in and language feature as supported, partial or missing, each with its test file.
- Tolerance: values match to 1e-8 relative after warm-up; trade lists match exactly (same entry bar, price and size).

## Backtester

The backtester is a broker emulator that reproduces TradingView's Strategy Tester rules by default, then offers stricter, more realistic modes as opt-in settings.

**Default fill rules (TradingView parity)**

- Orders created on a bar's close fill at the next bar's open, unless `process_orders_on_close = true`
- Stop and limit orders fill inside a bar using the assumed price path: if the high is closer to the open than the low, the path is open → high → low → close; otherwise open → low → high → close
- Gaps: a stop or limit crossed by a gap fills at the open
- Supports `strategy.entry`, `order`, `exit` (profit, loss, limit, stop, trail), `close`, `close_all`, `cancel`; pyramiding; OCA groups; `default_qty_type`; `initial_capital`; `commission_type`; `slippage`; margin calls

**Realism options (beyond parity)**

- Bar magnifier: resolve intrabar fills with lower-timeframe bars instead of the assumed path
- Spread and percentage slippage models; funding fees for perpetual futures
- Repainting guard: warn when a strategy uses `lookahead_on`, `security` on the current bar, or `varip`

**Report** — the Strategy Tester panel shows:

| Tab | Contents |
| --- | --- |
| Overview | Equity curve, drawdown, buy-and-hold comparison |
| Performance summary | Net profit, gross profit and loss, profit factor, max drawdown, win rate, average trade, Sharpe, Sortino, max consecutive losses, commission paid |
| List of trades | Entry and exit time, price, size, P&L, run-up, drawdown per trade, with click-to-jump on the chart |
| Properties | Every setting the run used, so results are reproducible |

**Research tools (later phase)** — grid and random parameter optimization run in parallel workers; walk-forward analysis with in-sample and out-of-sample windows; Monte Carlo trade reshuffling for drawdown confidence; batch runs across symbols with CSV export.

## Implementation phases

Ten phases, each ending in a gate the agent must pass before starting the next. Phases 3 and 4 can overlap; everything else is sequential. Start each phase in Claude Code's plan mode and break it into tasks of a few hours each.

1. **Phase 0 — Foundations**
   - Create the monorepo, strict TS config, Vitest, ESLint, Prettier, GitHub Actions CI
   - Add `CLAUDE.md` (rules below), `docs/PLAN.md` (this doc), `docs/PROGRESS.md`, `docs/decisions/` for ADRs
   - Gate: `pnpm typecheck && pnpm lint && pnpm test` pass in CI on an empty skeleton
2. **Phase 1 — Market data**
   - `DataProvider` interface; Binance klines provider; CSV importer; local cache with incremental updates
   - Resampling 1m → any timeframe, exchange sessions, timezone handling, gap detection
   - Gate: download 2 years of BTCUSDT 1h, resample to 4h and 1D, totals match the exchange's own 4h and 1D bars
3. **Phase 2 — Chart MVP**
   - Candles and volume, symbol search, timeframe switcher, crosshair with OHLC legend, pan and zoom, dark and light theme, watchlist
   - Gate: 100k bars pan and zoom at 60 fps; switching symbol or timeframe renders in under 300 ms from cache
4. **Phase 3 — Indicator library**
   - Pure functions in `packages/ta`: SMA, EMA, RMA, WMA, VWMA, HMA, RSI, MACD, Stochastic, Bollinger, Keltner, ATR, ADX/DMI, SuperTrend, Ichimoku, VWAP, OBV, MFI, CCI, pivots, and more
   - Overlay vs separate pane, settings dialog, style editor, indicator templates
   - Gate: every indicator matches its TA-Lib or TradingView-exported fixture within tolerance
5. **Phase 4 — Pine engine I (core language)**
   - Lexer, parser, checker, runtime; `indicator()`, inputs, `plot`, `hline`, `fill`, `bgcolor`, `plotshape`; `ta.*` wired to `packages/ta`
   - Gate: 30 sample indicator scripts produce plots identical to TradingView exports
6. **Phase 5 — Pine editor**
   - Monaco panel docked under the chart; highlighting, autocomplete, hover docs, inline errors from the checker; Add to chart and Update on save; script library with versions; auto-generated inputs dialog
   - Gate: typing a syntax error shows a marker within 200 ms; fixing it and saving updates the chart
7. **Phase 6 — Pine engine II (full language)**
   - User-defined types and methods, `switch`, loops, tuples, `array`/`map`/`matrix`, `label`/`line`/`box`/`table`/`polyline`, `request.security` (multi-timeframe), `alertcondition`, libraries (`import`)
   - Gate: `COMPATIBILITY.md` shows at least 90% of commonly used built-ins supported, each with a test
8. **Phase 7 — Strategies and the Strategy Tester**
   - `strategy()` settings and `strategy.*` functions; broker emulator with the parity fill rules; trade markers on chart; the four tester tabs
   - Gate: 20 reference strategies (MA cross, RSI reversion, breakout with trailing stop, pyramiding, OCA) match TradingView's list of trades exactly
9. **Phase 8 — Decision tools and UX**
   - Drawing tools: trend line, horizontal and vertical line, ray, Fibonacci retracement, rectangle, text, price range measure; magnets; persistence per symbol
   - Saved layouts, multi-chart grid, bar replay, price alerts (local notifications), keyboard shortcuts
   - Gate: a drawing survives reload and timeframe change at the correct price and time
10. **Phase 9 — Advanced backtesting and hardening**
    - Bar magnifier, optimizer, walk-forward, Monte Carlo, batch runs, HTML and CSV report export
    - Performance pass (engine in worker, incremental recompute on new bar), Tauri desktop build, user docs
    - Gate: optimizer runs 500 combinations of a 5-year 1h strategy in under 2 minutes on 8 cores

## Agent rules (CLAUDE.md)

Copy the block below into `CLAUDE.md` at the repo root. Claude Code reads it at the start of every session, so it is the agent's standing contract.

```markdown
# CLAUDE.md — Rules for working on this project

## Mission
Build a local-first charting + Pine-compatible backtesting app.
Correctness of indicator values and backtest results beats features and looks.
A wrong number on a chart can cost the user real money.

## Start of every session
1. Read docs/PLAN.md, docs/PROGRESS.md and the latest files in docs/decisions/.
2. State which phase and task you are on before writing code.
3. Work on ONE task at a time. Do not start the next phase until the current gate passes.
4. For a new phase or any task touching more than ~5 files, use plan mode first.

## Definition of done (every task)
- `pnpm typecheck && pnpm lint && pnpm test` all pass.
- New behaviour has tests; bug fixes start with a failing test that reproduces the bug.
- docs/PROGRESS.md updated: what changed, what is next, known gaps.
- docs/COMPATIBILITY.md updated if a Pine feature or builtin changed status.
- One focused commit, Conventional Commits style (feat:, fix:, test:, refactor:, docs:).

## Correctness rules (non-negotiable)
- No lookahead. A bar may only see data up to and including its own close.
- Deterministic: same data + same script + same settings = identical output.
- Never edit a golden fixture or expected value to make a test pass.
  If you believe a fixture is wrong, stop and explain why to the user.
- Never claim Pine compatibility for something without a test proving it.
- Use float64 everywhere; model `na` explicitly, never as NaN leaking into UI.
- Match TradingView's warm-up, seeding and fill rules by default;
  anything different must be an opt-in setting documented in an ADR.

## Architecture boundaries
- packages/core, ta, pine, backtest, data: pure TypeScript, no React, no DOM.
- packages/pine must not import chart code; it returns plain output objects.
- The UI talks to the engine only through the worker message API.
- Charting library is used only behind ChartAdapter in packages/chart.
- New runtime dependency? Write a short ADR (why, alternatives, licence) first.

## Code style
- TypeScript strict, no `any` without a comment explaining why.
- Files under ~400 lines; functions do one thing; names over comments.
- Errors carry context (script line/col, bar index, symbol, timeframe).
- Performance budgets live in tests (bench/*.bench.ts); do not regress them.

## Legal and safety
- Do not copy TradingView source code, icons, branding, or documentation text.
  Learn semantics from the Pine reference, then write our own code and docs.
- Keep Lightweight Charts attribution if that library is used.
- No real-money trading code. Exchange access is read-only market data.
- Never commit API keys or .env files; read secrets from environment only.

## When to stop and ask the user
- Trading semantics are ambiguous (fill price, commission, rounding, sessions).
- A gate cannot be met, or a test fails 3 times after different fixes.
- A change would alter results of existing reference strategies.
- Scope would grow beyond the current task.
When you stop, write what you tried, what you observed, and 2 options.

## Useful commands
- pnpm dev            # web app + server
- pnpm test           # all unit and golden tests
- pnpm test:golden    # Pine and indicator parity tests only
- pnpm cli run <script.pine> --symbol BTCUSDT --tf 1h   # headless run
- pnpm bench          # performance budgets
```

Keep the file short as the project grows: move detail into `docs/` and link it, so the rules stay visible in every session.

## Risks and how to start

The biggest risk is silent mismatch: results that look plausible but differ from TradingView. Golden tests from day one are the defence.

| Risk | Mitigation |
| --- | --- |
| Pine semantics are subtle and partly undocumented | Golden tests against TradingView CSV exports; COMPATIBILITY.md; ask before guessing |
| Scope explodes (Pine is a large language) | Strict phase gates; implement builtins in order of real use in your own scripts |
| Backtests look better than live trading | Parity mode first, then bar magnifier, slippage and walk-forward as defaults for decisions |
| Free data has gaps or differs from your broker's feed | Gap detection, provider choice per symbol, CSV import of broker data |
| Engine too slow for optimization | Worker threads, typed arrays for series, benchmarks in CI |
| Agent drifts across long sessions | CLAUDE.md, PROGRESS.md handoff notes, one task per session |

**First session prompt for Claude Code**

```text
Read docs/PLAN.md and CLAUDE.md. We are starting Phase 0.
Use plan mode: propose the monorepo skeleton, tooling and CI,
list the tasks for Phase 0, and wait for my approval before writing code.
```

Before Phase 3, export a handful of TradingView chart CSVs (BTCUSDT 1h with RSI, EMA, MACD, ATR, Bollinger added) and drop them into `fixtures/` — they become the reference the whole engine is measured against.
