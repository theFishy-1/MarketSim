# MarketSim — guide for future sessions

Order-book–driven stock-market **simulation**, **zero build** (modular classic `<script>` files, no bundler), Polish UI.
Graded university "programowania komputerów" project. Inspired by the YouTube video
*"I made a Market Simulation to see if Patterns are Real"* — patterns (support/resistance) are the
"market's memory" = accumulated resting orders, **not** a random walk of price.

## Files
The app is split into page + styles + ordered JS modules (no bundler; classic `<script src>` so it still runs by double-click under `file://`). **Load order matters** — it's the dependency order; top-level `const`/`class`/`let` are shared across classic scripts via the global lexical scope (e.g. `_orderId` lives in `order-book.js`, reassigned by `Simulation.reset`).
- `market_simulation.html` — page structure only; links `styles.css` and the `js/*` modules (Chart.js CDN stays in `<head>`).
- `styles.css` — all CSS.
- `js/config.js` — `CFG` (all tunables + `PACE_PRESETS`). `js/utils.js` — helpers (`fmt`/`clamp`/`snapToTick`/`expSample`/`compactProf`/`compactMap`/`fmtRate`/`fmtMarketTime`).
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
1. `CFG.marketSecPerTick` — how much **market time** one tick/step represents (realism). Default **0.05** (`Tempo rynku` = Bardzo płynny). Presets 0.05 / 0.1 / 0.5 s/tick (a "tick" is a sim STEP — one round of all agents — not "time between trades"; many trades happen per tick).
2. `CFG.simSpeed` — **compression**: market-seconds replayed per wall-second (`Tempo symulacji`).

Run loop is a **requestAnimationFrame accumulator** (NOT one step per timer):
`steps/frame = floor(simSpeed*dtWall / marketSecPerTick)`, **capped at `CFG.maxStepsPerFrame` (1800)**, render **once per frame**.
- The speed slider is logarithmic and its **max is pace-aware & achievable**: `speedMax = maxStepsPerFrame*60*marketSecPerTick`. **NEVER** map the slider to a fixed huge compression (the old 604800× = "a week per second" is uncomputable → ~20000 steps/frame → freeze/jank). Per-step ≈7 µs, so 1800/frame ≈ 12.6 ms = smooth.
- Changing `Tempo rynku`: the `#marketPace` handler calls **`applyPace(value)`** then clears `baseCandles`/`_curBase`, rebuilds the timeframe ladder, and re-applies the speed slider. Does NOT reset the sim.
- **`Tempo rynku` is a full LIQUIDITY PROFILE, not just a clock.** Each preset in `CFG.PACE_PRESETS` (keyed by the `<option>` value) sets `marketSecPerTick` **and** the MM book shape: `mmTargetDepth` (depth/level), `mmLevels` (ladder levels), `mmInnerTicks` (best level's distance from mid = half-spread in ticks). `applyPace` writes all four (+ `tvpScale` for the traded-volume overlay) into `CFG` + recomputes `staleTicks`; it's called both at init and on change, so CFG's MM defaults must stay equal to the `'0.05'` preset (the default). Rationale: real illiquidity = fewer trades/sec **+ thinner book + wider spread + bigger price impact** — not just the same flow stretched over more time. **Liquid ⇒ faster ticks AND deeper book** (deeper book absorbs the higher flow so price stays calm; just shrinking `marketSecPerTick` alone makes price 3–4× choppier per market-minute — measured). Verified ladder (1800 s market each): `0.05` (d35×18,in1)→ ~20 ticks/s, ~254 szt/s, spread ≈ 5¢, range ≈ 79¢/min; `0.1` (d18×13,in1)→ ~10 ticks/s, ~137 szt/s, spread ≈ 5¢, range ≈ 193¢/min; `0.5` (d8×8,in3)→ ~2 ticks/s, ~36 szt/s, spread ≈ 20¢, range ≈ 770¢/min (thin book + fine ticks ⇒ very volatile). (`speedMax` ⇒ 1.5 / 3 / 15 h/s respectively.)
- `MarketMaker.replenish` stacks levels at `(mmInnerTicks + k)*tickCents` for `k=0..mmLevels-1` (k starts at 0). `mmInnerTicks=1` ⇒ distances 1..mmLevels ticks = the old behaviour; keep it backward-compatible. `ladderSpan` in `_renderHeatmap` includes `mmInnerTicks`.
- `staleSeconds=60` → `staleTicks` recomputed per pace (1200 / 600 / 120 at 0.05 / 0.1 / 0.5 — always 60 s of wall-decay).
- KPI shows market clock (`marketTimeSec()`); not the raw tick.

## Timeframe ladder
Derived from `marketSecPerTick`, do **not** hardcode option values. `TF_DEFS = [[label, seconds]]`; option value (ticks) = `round(sec/marketSecPerTick)`. Range: 1 tick, 1s, 5s, 15s, 1m, 5m, 15m, 1h, 4h, 1D, **1T (tydzień=604800 s), 1Mc (miesiąc=2592000 s ≈ 30 d)**. Weekly/monthly fill only as far as `maxBaseCandles` allows (~4.6 months) and otherwise right-align sparsely.
- **Dedup intervals that collapse to the same tick count** (`seen` Set in `rebuildTimeframeOptions`): at a coarse pace a labelled interval can round down to a finer one's tick count (e.g. at 2 s/tick `1s → round(1/2)=1 tick`, duplicating + mislabelling "1 tick"); drop the duplicate. (The current 0.05/0.1/0.5 presets don't trigger it — finest is 0.5 where `1s`=2 ticks — but it's kept as a safety net.) Selection is preserved across pace changes by the canonical label in `option.dataset.tf` (not by index, since the option count can vary). The "1 tick" option shows its real duration, e.g. `1 tick (0.5 s)`.
- **Default/fallback interval = first interval with ticks ≥ `baseTicks` (i.e. "1m")** (init + when the previously-selected interval is deduped away). Two reasons: (a) a candle needs several ticks for a visible body — defaulting to "1 tick" looked like a flat line; (b) the order-book/volume overlay is **gated to ≥ 1m** (see overlay section), so defaulting to 1m means the overlay is visible from the start without smearing. All paces therefore default to "1m" (proper candles + accurate overlay). Sub-minute intervals stay available (manual) for a clean live price view. (Don't fall back to `index 0`; that reintroduces the line-look.)
- **"1 tick" interval renders as a LINE even in świece mode** (`_renderCandles`, `if (this.timeframe === 1)`): a 1-tick "candle" is just one price (open≈close, no body), so candlesticks are meaningless there — draw a direction-coloured polyline, like a tick chart on any real platform. Coarser intervals (≥2 ticks) draw normal OHLC bodies.

## CANDLE BUILDING — invariants (this caused MANY bugs; protect them)
- **Bucket by ABSOLUTE tick/minute**: `floor(tickNo/tf)` (sub-minute) or `floor(m/tfMin)` (≥1m). **NEVER** bucket by array index or render-frame — that reintroduces "crawling / morphing candles".
- **Completed candles must be FROZEN.** Only the forming (rightmost) candle updates.
- **Skip the incomplete OLDEST bucket** — start at the first *whole* bucket boundary, in BOTH builders. Otherwise the leftmost candle "pełza"/morphs as the buffer slides. This was fixed once per path (`_buildCandlesFromTicks` and `_buildCandlesFromBase`) — don't regress. Keep the partial only if it's the single/forming candle.
- **Gapless candles:** open = previous bucket's close.
- Two builders, dispatched by `tf` vs `baseTicks = round(60/marketSecPerTick)`:
  - `tf < baseTicks` → `_buildCandlesFromTicks` (raw price ring, `maxHistory=15000` ticks ≈ 12.5 min @0.05 default pace — a scrolling window).
  - `tf >= baseTicks` → `_buildCandlesFromBase` (aggregates **persistent 1-minute base candles**, `maxBaseCandles=200000` ≈ 139 days / ~4.6 months — so 1m…1D **fill** the chart, and weekly/monthly show a handful of candles instead of one; bumped from 60000 to make 1T/1Mc usable. The per-minute `shift()` on this array is O(n) but cheap — a memmove ~once per market-minute).
- Sparse high-TF views **right-align** with a capped slot (≤30 px) so they don't float in the middle.
- **Zoom:** both builders take `this.visibleBars` (not `CFG.maxVisibleBars`) as the count of newest candles to show — `Renderer.zoom(factor)` scales it, clamped to `[CFG.minVisibleBars=40, CFG.maxVisibleBars=400]`, default `CFG.defaultVisibleBars=150`. Wired to mouse-wheel over `.chart-wrap` (up=in) and the `#zoomIn`/`#zoomOut` buttons. Right-anchored (always the latest bars) — no horizontal pan. `maxVisibleBars` is now just the zoom-out limit / data-window bound.

## ORDER-BOOK HISTORY overlay (the video-matching feature)
- Each base candle carries a **compact** depth snapshot (`compactProf` = near-price ±`bookNearCents` flat arrays `[p,q,...]`), computed once per market-minute on completion. The forming candle's prof refreshes on snapshot frames.
- `_drawBookHistory` reads these per-candle profs → overlay **covers the whole chart and persists** (do NOT drive it from the short `sim.heatmap` buffer — that's why it used to "disappear fast"). Opacity ∝ resting **size** (`q/mmTargetDepth`). Line height `th = max(1, gap*0.9)` — i.e. **fills ~90 % of the bin gap** (`gap` = px per `depthBinCents`), leaving only a hairline so the heatmap reads as continuous at any zoom. (Was capped at 2.5 px, which left huge holes when zoomed in — tight price range ⇒ large `gap` ⇒ tiny lines with big gaps; the user reported this. Don't reintroduce a fixed px cap.)
- Memory: profs kept only for the last `bookProfWindow=6000` base candles; older candles keep OHLC only. `_bookByBucket` iterates only that tail.
- Bucket index of a base-candle prof equals the candle's `b` (`floor(m/tfMin)=floor(tick/tf)`), so overlay and candles **align**.
- The **heatmap panel** ("Mapa płynności w czasie") is a SEPARATE viz using `sim.heatmap` (full profiles, throttled 1/frame, cap 600). Don't conflate the two. **It is NOT gated** — so even when the chart overlay is hidden at sub-minute (below), the live resting liquidity is still visible here.
- **GATED to tf ≥ baseTicks (≥ 1 min) — BOTH modes** (`_drawBookHistory` top guard, `if (this.timeframe < baseTicks)`): overlay data (resting `prof` + `tvp`) is captured per market-minute, so below 1m it smears one minute's profile across every sub-candle → staircase blocks (the worse the more volatile the pace → tragic at the thin-book preset). Below 1m it draws a one-line hint instead. Paired with the "default = 1m" rule above, the default view always shows the overlay correctly; sub-minute is a clean price chart. (Earlier only `volume` was gated and `rest` smeared into solid blocks at the illiquid preset — user reported it as "tragiczne".)

### TRADED-VOLUME (`tvp`) — realised orders, the second overlay lens
The overlay has two modes via the `#bookMode` select ("Nakładka"), `Renderer.bookMode` (`'rest'` default | `'volume'`):
- **`rest`** — resting walls as above, PLUS a cream/amber **consumption accent** (`rgba(255,240,170,a)`) on the *resting-wall bins that actually traded* (looks up `tvp.get(p)` per wall bin). Answers "which walls are real vs the MM's phantom ladder." Accent only on bins present in the end-of-minute prof snapshot — a wall fully eaten during the minute is gone from the snapshot, so it won't accent (shows in `volume` mode instead).
- **`volume`** — pure **traded-volume-at-price** (volume profile / VPVR-over-time), neutral gold, opacity ∝ `q/tvpScale`. No bid/ask side (a fill has no side once executed). (Both modes gated to ≥ 1m — see the overlay section above.)
- **Capture path:** `MatchingEngine.executeOrder(order, tradeSink)` & `clearCrossedBook(refCents, tradeSink)` take an optional `Map(binPrice→qty)` and record each crossed level's `tq` at `floor(price/depthBinCents)*depthBinCents`. Recorded **once per crossed level** (no maker+taker double-count). `Simulation` owns `_tvpTick` (per-tick, `.clear()`ed at top of `step`, reused → no per-tick alloc) and `_tvpBucket` (per-base-candle accumulator).
- **Boundary-tick correctness:** at minute rollover the OLD candle's `tvp` is snapshotted from `_tvpBucket` (which excludes the current tick), then `_tvpBucket.clear()`; only AFTER the candle block is `_tvpTick` merged into `_tvpBucket`, so the boundary tick's trades land in the NEW candle (mirrors how this tick's price opens the new candle). Forming candle's `tvp` refreshed in the `doSnapshot` block via `compactMap` centered on the candle's mid.
- **`compactMap(map, center, W)`** (utils) = single-side version of `compactProf` (which now delegates to it) → flat `[p,q,...]` trimmed to ±`bookNearCents`. Stored as `baseCandle.tvp` / `_curBase.tvp`.
- **High-TF aggregation (both `_bookByBucket` and `_tvpByBucket`).** When several base-minutes merge into one chart candle (tf > baseTicks), both **SUM** their bins per bucket AND count `n` minutes → return `{bids:Map, asks:Map, n}` (book) / `{map:Map, n}` (tvp). The drawers normalise by `n`: resting depth → **average depth/price** over the period (`q/n / mmTargetDepth`), traded volume → **avg volume/min** (`q/n / tvpScale`), so opacity stays in the per-minute range at any TF. (Previously `_bookByBucket` *overwrote* — kept only the last minute's snapshot → on high TFs the bands clustered at each candle's close instead of covering its range; user reported "wygląda dziwnie".) At 1m (tfMin=1) `n=1` ⇒ identical to a raw snapshot. Sub-minute TF (gated, not drawn) still maps one base candle's profile to its spanned sub-buckets (`n=1` each).
- **Memory:** `tvp` evicted alongside `prof` beyond `bookProfWindow`; same `maxBaseCandles` cap. `tvpScale` is the opacity normaliser and is **per-preset** (in `PACE_PRESETS`, written by `applyPace` like `mmTargetDepth`): **3600 / 900 / 75** for `0.05 / 0.1 / 0.5` pace, each ≈ that preset's p90 per-bin-per-minute traded volume (faster+deeper book ⇒ much more volume/bin, so a single global scale would saturate). Re-measure if a preset's pace/depth changes.

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
