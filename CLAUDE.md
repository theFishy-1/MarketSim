# MarketSim — guide for future sessions

Order-book–driven stock-market **simulation**, **zero build** (modular classic `<script>` files, no bundler), Polish UI.
Graded university "programowania komputerów" project. Inspired by the YouTube video
*"I made a Market Simulation to see if Patterns are Real"* — patterns (support/resistance) are the
"market's memory" = accumulated resting orders, **not** a random walk of price.

## Files
The app is split into page + styles + ordered JS modules (no bundler; classic `<script src>` so it still runs by double-click under `file://`). **Load order matters** — it's the dependency order; top-level `const`/`class`/`let` are shared across classic scripts via the global lexical scope (e.g. `_orderId` lives in `order-book.js`, reassigned by `Simulation.reset`).
- `market_simulation.html` — page structure only; links `styles.css` and the `js/*` modules (Chart.js CDN stays in `<head>`).
- `styles.css` — all CSS.
- `js/config.js` — `CFG` (all tunables + `PACE_PRESETS`). `js/utils.js` — helpers (`fmt`/`clamp`/`snapToTick`/`expSample`/`compactProf`/`fmtRate`/`fmtMarketTime`).
- `js/order-book.js` — `_orderId`, `Order`, `OrderBook`. `js/engine.js` — `MatchingEngine`. `js/agents.js` — `Agent`+`MarketMaker`/`NoiseTrader`/`TrendFollower`/`Whale`.
- `js/simulation.js` — `Simulation`. `js/renderer.js` — `SUPPORT_CENTS`/`RESISTANCE_CENTS`+`Renderer`. `js/main.js` — bootstrap IIFE (loop, UI wiring, `applyPace`/`applySimSpeed`, global error handler; sets `window.__sim`/`__renderer`).
- Load order in HTML: config → utils → order-book → engine → agents → simulation → renderer → main.
- `diagram.html` — 4 Mermaid diagrams (main loop, matching engine, agents, UML class), Polish.
- `README.md` — Polish project doc; **keep it in sync** when behaviour changes (we have, every time).
- `_*.png` / `_*.html` / `_*.js` are throwaway verification artifacts → gitignored. **Always delete them after use.**
- Code is intentionally comment-light (one-line file headers only); the rationale/invariants live in this file.

## How to run / verify (do this — don't guess)
Node 24, Chrome and Edge are installed. The app is browser JS; verify with **headless Chrome**:

```
"/c/Program Files/Google/Chrome/Application/chrome.exe" --headless=new --disable-gpu --no-sandbox \
  --screenshot="$(pwd -W)/_shot.png" --window-size=1500,1080 --virtual-time-budget=8000 \
  "file:///d:/Repo/MarketSim/market_simulation.html"
```
- Use `--headless=new`. Plain `--headless` sometimes exits non-zero with no file.
- `--dump-dom` (to stdout) to read KPI values / catch errors. Errors surface in `#errbar` (and `window.onerror` writes there) — grep for `id="errbar"[^>]*>[^<]+`.
- **JS syntax check:** `node -e` that reads each `js/*.js` and `new Function(body)` (free vars across modules are fine at parse time).
- **Read screenshots** with the Read tool to eyeball the chart.

### Verification gotchas (important)
- **rAF does NOT fast-forward under `--virtual-time-budget`.** It fires at the real frame cadence, so the running sim only advances a few ticks in a virtual-time window. To test the sim **at scale**, inject a `<script>` (replace `</body>`) that sets `window.__sim.running=false` then drives `window.__sim.step()` in a loop, calls `window.__renderer.render()`, and writes results to a `#out` div; dump-dom it. `window.__sim` and `window.__renderer` are intentional debug handles — keep them.
- **`performance.now()` is frozen during synchronous execution under `--virtual-time-budget`.** To measure real per-step cost, run a synchronous timed loop **without** `--virtual-time-budget` (it was ~7 µs/step).
- For a screenshot that needs injected state to apply first, you MUST pass `--virtual-time-budget` so the injected `setTimeout` fires before the capture (the synchronous batch then runs regardless).

## Core conventions
- **Prices are INTEGER CENTS** everywhere internally. `fmt(c)=c/100`. Don't introduce float prices (it fragments liquidity "walls").
- All tunables live in the `CFG` object in `js/config.js`.
- Classes: `Order`, `OrderBook` (two sorted arrays, price-time priority), `MatchingEngine` (`executeOrder` = the single walk-the-book path for agents/whale/user), `Agent` + `MarketMaker`/`NoiseTrader`/`TrendFollower`/`Whale`, `Simulation`, `Renderer`.

## TWO-CLOCK time model (easy to break — read before touching speed)
1. `CFG.marketSecPerTick` — how much **market time** one tick/step represents (realism). Default **0.5** (`Tempo rynku` = Płynny). Presets 0.5 / 1 / 3 s/tick.
2. `CFG.simSpeed` — **compression**: market-seconds replayed per wall-second (`Tempo symulacji`).

Run loop is a **requestAnimationFrame accumulator** (NOT one step per timer):
`steps/frame = floor(simSpeed*dtWall / marketSecPerTick)`, **capped at `CFG.maxStepsPerFrame` (1800)**, render **once per frame**.
- The speed slider is logarithmic and its **max is pace-aware & achievable**: `speedMax = maxStepsPerFrame*60*marketSecPerTick`. **NEVER** map the slider to a fixed huge compression (the old 604800× = "a week per second" is uncomputable → ~20000 steps/frame → freeze/jank). Per-step ≈7 µs, so 1800/frame ≈ 12.6 ms = smooth.
- Changing `Tempo rynku`: the `#marketPace` handler calls **`applyPace(value)`** then clears `baseCandles`/`_curBase`, rebuilds the timeframe ladder, and re-applies the speed slider. Does NOT reset the sim.
- **`Tempo rynku` is a full LIQUIDITY PROFILE, not just a clock.** Each preset in `CFG.PACE_PRESETS` (keyed by the `<option>` value) sets `marketSecPerTick` **and** the MM book shape: `mmTargetDepth` (depth/level), `mmLevels` (ladder levels), `mmInnerTicks` (best level's distance from mid = half-spread in ticks). `applyPace` writes all four into `CFG` + recomputes `staleTicks`; it's called both at init and on change, so CFG's MM defaults must stay equal to the `'0.5'` preset. Rationale: real illiquidity = fewer trades/sec **+ thinner book + wider spread + bigger price impact** — not just the same flow stretched over more time. Verified: presets give spread ≈ 8/12/31¢ and ≈ 6/3.5/1.3 trades/s for 0.5/1/3 s/tick.
- `MarketMaker.replenish` stacks levels at `(mmInnerTicks + k)*tickCents` for `k=0..mmLevels-1` (k starts at 0). `mmInnerTicks=1` ⇒ distances 1..mmLevels ticks = the old behaviour; keep it backward-compatible. `ladderSpan` in `_renderHeatmap` includes `mmInnerTicks`.
- `staleSeconds=60` → `staleTicks=120` at 0.5 pace (preserves the original wall-decay look).
- KPI shows market clock (`marketTimeSec()`); not the raw tick.

## Timeframe ladder
Derived from `marketSecPerTick`, do **not** hardcode option values. `TF_DEFS = [[label, seconds]]`; option value (ticks) = `round(sec/marketSecPerTick)`. Range: 1 tick, 1s, 5s, 15s, 1m, 5m, 15m, 1h, 4h, 1D.

## CANDLE BUILDING — invariants (this caused MANY bugs; protect them)
- **Bucket by ABSOLUTE tick/minute**: `floor(tickNo/tf)` (sub-minute) or `floor(m/tfMin)` (≥1m). **NEVER** bucket by array index or render-frame — that reintroduces "crawling / morphing candles".
- **Completed candles must be FROZEN.** Only the forming (rightmost) candle updates.
- **Skip the incomplete OLDEST bucket** — start at the first *whole* bucket boundary, in BOTH builders. Otherwise the leftmost candle "pełza"/morphs as the buffer slides. This was fixed once per path (`_buildCandlesFromTicks` and `_buildCandlesFromBase`) — don't regress. Keep the partial only if it's the single/forming candle.
- **Gapless candles:** open = previous bucket's close.
- Two builders, dispatched by `tf` vs `baseTicks = round(60/marketSecPerTick)`:
  - `tf < baseTicks` → `_buildCandlesFromTicks` (raw price ring, `maxHistory=3000` ticks ≈ 25 min @0.5 — a scrolling window).
  - `tf >= baseTicks` → `_buildCandlesFromBase` (aggregates **persistent 1-minute base candles**, `maxBaseCandles=60000` ≈ 42 days — so 1m/15m/1h/4h/1D **fill** the chart instead of starving).
- Sparse high-TF views **right-align** with a capped slot (≤30 px) so they don't float in the middle.

## ORDER-BOOK HISTORY overlay (the video-matching feature)
- Each base candle carries a **compact** depth snapshot (`compactProf` = near-price ±`bookNearCents` flat arrays `[p,q,...]`), computed once per market-minute on completion. The forming candle's prof refreshes on snapshot frames.
- `_drawBookHistory` reads these per-candle profs → overlay **covers the whole chart and persists** (do NOT drive it from the short `sim.heatmap` buffer — that's why it used to "disappear fast"). Opacity ∝ resting **size** (`q/mmTargetDepth`); lines thinner than the bin gap so gaps show (like the video).
- Memory: profs kept only for the last `bookProfWindow=6000` base candles; older candles keep OHLC only. `_bookByBucket` iterates only that tail.
- Bucket index of a base-candle prof equals the candle's `b` (`floor(m/tfMin)=floor(tick/tf)`), so overlay and candles **align**.
- The **heatmap panel** ("Mapa płynności w czasie") is a SEPARATE viz using `sim.heatmap` (full profiles, throttled 1/frame, cap 600). Don't conflate the two.

## Performance throttling (don't undo)
- `step(doSnapshot)`: the expensive `depthProfile` + S/R + `heatmap.push` run only when `doSnapshot` (last step of a frame). Everything **price-determining** runs every tick. Base-candle prof is computed once per minute (cheap), independent of doSnapshot.
- Do **not** reset `lastUserFill`/`eventBanner`/`noLiquidity` at the top of `step()` — they'd be wiped mid-batch; the Renderer clears them **after** display.

## Faithfulness / model facts
- Maker/taker split is what makes it work: `MarketMaker` replenishes a ladder both sides (level count/depth/spread per the active liquidity profile — see `PACE_PRESETS`); ~80% of noise + 90% trend + 100% whale orders are **MARKET** orders that walk the book. Don't make most orders limit (price would barely move).
- Healthy taker/maker ratio ≈ 0.4–1.6 (the KPI badge).
- Price is a **driftless random walk** amplified by trend followers → **no mean reversion**, so it can wander over very long runs; clamped to [10, 1000]. That's expected, not a bug.

## diagram.html (Mermaid) gotchas
- `<<abstract>>` inside a `<pre>` must be HTML-escaped (`&lt;&lt;abstract&gt;&gt;`) or the HTML parser eats `<abstract>` as a tag → classDiagram syntax error.
- Class-diagram arrays: use generics `List~T~`, not `T[]` (incl. method return types).
- We import Mermaid dynamically, so `startOnLoad` won't fire — call `await mermaid.run()` explicitly.
- Diagram process/method names should match the real code; text is Polish (tak/nie).

## Don't do
- Don't promise uncomputable speeds (fixed huge slider max). Keep the slider max = cap-sustainable, pace-aware.
- Don't bucket candles by array position / frame.
- Don't store full depth profiles for all base candles (memory) — compact + `bookProfWindow`.
- Don't run rendering or full-rate `depthProfile` every tick at high speed.
- Don't overwrite a screenshot/temp `_*` file into git; clean them up.
