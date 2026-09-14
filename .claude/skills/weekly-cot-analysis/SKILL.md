---
name: weekly-cot-analysis
description: Weekly chart analysis focused on the forward curve and fundamentals for whatever symbol is on the weekly (1W) chart tab. Draws exactly 4 bullets on the daily (1D) chart tab: forward curve analysis, short-term fundamental, long-term structural bias, upcoming catalyst. Invoke when the user asks to "analyze tab 0", "run the weekly analysis", "analyze [symbol] weekly", or runs /weekly-cot-analysis.
---

# weekly-cot-analysis

Analyze the weekly (`1W`) chart — forward curve and fresh fundamental news. Deliver a concise written report, then draw exactly 4 bullet points on the daily (`1D`) chart.

## Steps

### 1. Locate the weekly and daily charts (by content, not tab index)

**Do not assume tab 0 is the weekly chart** — tabs can be reordered. Identify the charts by their content:

1. Call `tab_list` to enumerate the open tabs.
2. For each tab, `tab_switch` to it and call `chart_get_state`, then classify:
   - **Weekly analysis chart** = `resolution` is `1W` **and** `studies` include **`Future Forward Curve`** (usually alongside `COT Disaggregated`). Remember its index as **WEEKLY_TAB** — this is where you analyze (steps 2–4).
   - **Daily chart** = `resolution` is `1D` (the Daily-Default set: Fair Value Gap, Keltner Channel, Volume, Moving Average Ribbon). Remember its index as **DAILY_TAB** — this is where you draw (step 5).
   - You can stop scanning once both are found (there are typically only two tabs).
3. Make sure you're on **WEEKLY_TAB** (and it shows a `1W` chart with the Future Forward Curve) before continuing.
4. **If no `1W` chart with a Future Forward Curve is open**, stop and tell the user the expected weekly chart isn't available rather than analyzing the wrong chart.

### 2. Pull chart data and news in parallel

Fire all of these simultaneously:
- `quote_get` (no symbol arg) — current price and instrument description
- `data_get_pine_tables` with `study_filter: "Future Forward Curve"` — forward curve slope table
- `WebSearch` with query `"[INSTRUMENT_NAME] [SYMBOL] outlook fundamental news [MONTH] [YEAR]"` — use the `description` field from `quote_get` for instrument name (short-term fundamental read: weeks to months)
- `WebSearch` with query `"[INSTRUMENT_NAME] structural supply demand long-term outlook capacity"` — use the `description` field from `quote_get` for instrument name (long-term structural read: months up to 2 years)

If either search doesn't return sufficient information, run a second targeted search:
- Short-term: central bank policy, recent data prints, or a supply/demand driver specific to the commodity
- Long-term: capacity/production trends, multi-year demand drivers (e.g. demographics, policy shifts), or structural cost/regulatory changes

### 3. Write the analysis report

Structure:

```
## [Instrument] ([Symbol]) — Weekly Analysis
Current price | Exchange | Date

### Forward Curve
Table: Now | 1b ago | 4b ago | Δ vs 1b | Δ vs 4b
Interpretation — contango/backwardation, flattening/steepening direction, what it implies
for carry dynamics, supply tightness, or rate expectations

### Short-Term Fundamentals (Weeks to Months)
1-2 paragraphs covering near-term macro drivers, recent data prints,
central bank policy (for financials/FX), or commodity-specific news
that could move price over the next several weeks to a few months

### Long-Term Structural Bias (Months up to 2 Years)
1-2 paragraphs covering the underlying supply/demand structure —
production/capacity trends, multi-year demand drivers, demographics,
regulatory or policy shifts — disregarding short-term noise. State
whether the structural bias is bullish, bearish, or neutral and why

### Upcoming Catalysts (Next Several Weeks)
Bulleted list of specific dated or near-term events that could move price:
- Central bank meetings, economic data releases
- USDA reports, inventory data, earnings
- Geopolitical events, seasonal turning points
- Contract expirations or roll periods
```

### 4. Distill to exactly 4 bullets

Write exactly 4 tight bullets — one per theme, no more, no less:

1. **Forward curve:** Slope level, direction of change (flattening/steepening), and what it implies (supply tightness, carry cost, rate expectations)
2. **Short-term fundamental:** The single most important near-term (weeks to months) macro or supply/demand driver right now and its directional implication
3. **Long-term structural:** The structural (months up to 2 years) supply/demand bias — bullish, bearish, or neutral — and the one or two drivers behind it, disregarding short-term noise
4. **Catalyst:** The most significant specific upcoming event in the next several weeks that could move price, with approximate timing

### 5. Draw the 4 bullets on the daily chart (preserving any existing section text)

1. Call `tab_switch` with `index: DAILY_TAB` (the `1D` chart identified in step 1).
2. Call `quote_get` to get the current price on the daily chart.
3. **Check for an existing summary and carry over its sections.** Call `draw_list` on the daily chart. For **every** `text` shape, call `draw_get_properties` to read its content (the text is in `properties.text`). A shape is a prior analysis box if its text contains **most of the six section labels** (`Trend`, `Regime`, `Seasonal`, `Commercial`, `Managed Money`/`Managed money`, `Public`) followed by a colon — **regardless of header, label order, or capitalization**. Do **not** require a `… Weekly —` header: the user's own boxes may use a different header (e.g. `… — Key Factors for Next Move`), order the labels differently (e.g. `Regime`/`Trend`/`Seasonal`), or vary casing.
   - If one matches, **extract whatever the user typed after each of the six labels** and reuse it **verbatim** (match labels case-insensitively; normalize the label spelling to the canonical layout below when you rewrite, but keep the *value* exactly). If a label was blank, keep it blank.
   - **Also preserve any freeform note** — lines the user added that aren't one of your 4 bullets and aren't one of the six sections (e.g. a "Weekly in consolidation zone…" paragraph). Carry it over verbatim as a `Note:` line at the bottom.
   - **Do NOT preserve the old analysis/bullets/key-factors** — that is the "summary section" you are replacing with your fresh 4 bullets.
   - Note the matched box's `entity_id` for removal in step 5. (Leave non-analysis drawings — trend lines, positions, vertical lines — untouched.)
   - If no analysis box is found, **omit the six section labels entirely** and add no note — see "Only carry over what already existed" below.
4. Call `draw_shape` with:
   - `shape: "text"`
   - `point.time`: current bar time from the quote
   - `point.price`: current price + ~1% (so label sits just above price action)
   - `text`: symbol, date, the **freshly written** 4 numbered bullets, then — **only if they were present in the box you are replacing** — the carried-over sections and note from step 3.
   - **Separate every bullet with a blank line** — one after the header and one between each numbered bullet (`\n\n`, not `\n`). The bullets run long, so without blank lines the box renders as an unreadable wall of text.
5. **Remove the old summary.** If step 3 matched an existing analysis box, call `draw_remove_one` with its `entity_id` so only the updated summary remains. (Draw the new box first, then remove the old one, so a failed draw never leaves the chart with no summary — and never loses the carried-over sections/note.)

**Only carry over what already existed.** The four numbered bullets are the *only* thing you always write. The six section labels and any freeform note are user-owned content that you **preserve when present and omit when absent** — never introduce them yourself.

- **No prior box, or a prior box without the sections** → draw the header + 4 bullets and **nothing else**. Do not append empty `Trend:` / `Regime:` / `Seasonal:` / `Commercial:` / `Managed Money:` / `Public:` labels. A label the user never had is clutter, not a template to fill in.
- **Prior box had the sections** → reproduce exactly the labels it had, with their values verbatim. If the user had only some of the six, carry over only those. If a label existed but was blank, keep it blank.

Minimal layout (no prior sections — the common case on a fresh chart). Note the blank line after the header and between every bullet:

```
[SYMBOL] Weekly — [DATE]

1. Forward curve: ...

2. Short-term fundamental: ...

3. Long-term structural: ...

4. Catalyst: ...
```

Full layout — use this **only** when the box you are replacing already contained these sections. Keep the blank-line-separated 4 bullets, a blank line, the first section block, a blank line, the second section block, then an optional preserved note. The section blocks stay tightly grouped (no blank lines *within* a block) — only the bullets get blank lines between them:

```
[SYMBOL] Weekly — [DATE]

1. Forward curve: ...

2. Short-term fundamental: ...

3. Long-term structural: ...

4. Catalyst: ...

Trend: [carried over verbatim]
Regime: [carried over verbatim]
Seasonal: [carried over verbatim]

Commercial: [carried over verbatim]
Managed Money: [carried over verbatim]
Public: [carried over verbatim]

Note: [carried over freeform note — omit this line entirely if there was none]
```

**Only the four numbered bullets are (re)written by you.** Never fill in, infer, or guess a value for a section — if a carried-over label was blank, leave the label and colon with nothing after it.

## Notes & edge cases

- **Forward curve not present:** If `data_get_pine_tables` returns no studies, note it and skip that section. Do not call COT tools.
- **Do not pull COT data** — skip `data_get_study_values` and `data_get_indicator` entirely.
- **Data sync mismatch:** If `quote_get` returns a different symbol than `chart_get_state`, trust `chart_get_state` and explicitly pass the symbol to `quote_get`.
- **Tab 1 draw placement:** If `draw_shape` fails (e.g. price out of visible range), retry with `point.price` set closer to the current quote.
- **Stale data after symbol change:** Cross-check `quote_get` description against `chart_get_state` symbol before reporting.
- **Sections are user-owned — never introduce them:** The Trend / Regime / Seasonal and Commercial / Managed Money / Public labels belong to the user, not the template. If the box you are replacing didn't have them, the new box doesn't get them either — do **not** emit empty labels for the user to fill in. Only reproduce labels that were already there, and never populate, infer, or guess a value, even if you have data that would fit.
- **Preserve on re-run:** When updating an existing summary, the goal is to refresh *only* the 4 bullets (symbol, date, forward curve, short-term fundamental, long-term structural, catalyst) while keeping whatever user-edited sections the old box had. Read the old box, copy each section's text exactly, then replace the box.
- **Short-term vs. long-term must actually differ:** Don't let the long-term structural bullet restate the short-term one in different words. Short-term = weeks-to-months news/data drivers (tariff headlines, recent prints, near-term policy). Long-term = months-up-to-2-years supply/demand structure (capacity trends, multi-year demand shifts, demographics) that would hold even if the short-term driver reversed. If they point in opposite directions, say so explicitly rather than picking one.
- **Matching the existing summary:** Identify the prior box by **content, not header**. Either of these makes a `text` shape a prior analysis box: (a) it contains most of the six section labels (`Trend`/`Regime`/`Seasonal`/`Commercial`/`Managed Money`/`Public`) with a colon, in any order or casing; **or** (b) it is a previous run of this skill — a `… Weekly — [DATE]` header followed by numbered analysis bullets — even in an older format with fewer bullets and no sections at all. Case (b) still gets replaced and removed, it simply has no sections or note to carry over. The user's own boxes won't say `Weekly —` (e.g. they may use `— Key Factors for Next Move`) and may order labels `Regime`/`Trend`/`Seasonal` or write `Managed money`. Read each text shape's `properties.text` via `draw_get_properties` to check. Don't match by symbol — the daily contract code (e.g. `GCQ2026`) differs from the weekly root. If multiple analysis boxes match, use the most complete/recent and remove the stale one(s) too.
- **Drawings are symbol-specific:** Text boxes are bound to the chart symbol, so the daily chart only shows the current symbol's box — you won't see other symbols' summaries. This is why matching by content (not symbol) is safe: any analysis box present belongs to the symbol you're analyzing.
- **Preserve freeform notes, replace only the analysis:** The "summary section" you replace is the symbol/date + the analysis bullets (whatever their old format — numbered bullets, `•` key-factors, etc.). Everything else the user wrote — the six sections AND any freeform note/observations — is preserved verbatim. When in doubt about whether a line is old analysis (replace) or a user note (keep), keep it.
- **Blank line between every bullet (user preference, 2026-09-14):** In the drawn text box, put a blank line after the `[SYMBOL] Weekly — [DATE]` header and between each of the 4 numbered bullets — i.e. join them with `\n\n`, not `\n`. The bullets are long enough that single newlines render as one dense block that's hard to read on the chart. This applies to the drawn box only; the written report in chat keeps its normal markdown formatting. Carried-over section blocks are *not* blank-line separated internally — see the full layout above.
- **Draw-then-remove ordering:** Always draw the new box before calling `draw_remove_one` on the old one, so a failed draw never wipes the user's section text with nothing to replace it.
- **`draw_remove_one` can take other drawings with it (observed 2026-09-13):** a single remove call deleted three unrelated user callouts alongside the target. Before any removal, run `draw_get_properties` on **every** existing shape and keep the full output (points, text, style overrides) in your context as a backup. After removing, re-run `draw_list` and confirm the count changed by exactly one; if other shapes vanished, recreate them with `draw_shape` from the backup and tell the user. `draw_shape` passes `shape` straight through to `createMultipointShape`, so types beyond the documented five work — `callout` restores faithfully when given both points plus the original `overrides`. Chart `Ctrl+Z` does **not** undo CDP-driven changes, so the property snapshot is the only recovery path.
