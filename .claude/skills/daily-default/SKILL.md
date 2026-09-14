---
name: daily-default
description: Restore the user's default daily TradingView chart setup. Invoke when the user wants their standard daily indicators back, runs /daily-default, or asks to "reset/restore my chart" / "go back to my default chart". Loads the saved "Daily Default" layout (all standard indicators with their tuned settings) while keeping the symbol the user is currently viewing.
---

# daily-default

Restore the user's standard daily chart setup — all of their default indicators, on Daily / Candles — **without changing the symbol they are currently viewing**.

## What "default" is

The source of truth is the saved TradingView layout **`Daily Default`** (created from the user's working chart). It contains, with their tuned settings:

- **Timeframe:** Daily (`D`) · **Chart type:** Candles
- **Indicators:**
  1. Fair Value Gap [LuxAlgo]
  2. Keltner Channel signals
  3. Volume
  4. Moving Average Ribbon
  5. Future Forward Curve

A saved layout is used (rather than re-adding indicators by name) because it is the only mechanism that restores **every** indicator with its exact tuned settings (e.g. the Future Forward Curve's lookbacks, rebase, DTE spacing, and filters) — and any favorited/community scripts that aren't in the user's saved Pine scripts and can't be re-added by name.

## Steps

1. **Remember the current symbol.** Call `chart_get_state` and save its `symbol` as `CURRENT_SYMBOL`.
2. **Load the layout.** Call `layout_switch` with name `Daily Default`. This restores all indicators, the timeframe, and the chart type — but it will switch the chart to the layout's *saved* symbol.
3. **Restore the user's symbol.** Call `chart_get_state` again; if its `symbol` differs from `CURRENT_SYMBOL`, call `chart_set_symbol` with `CURRENT_SYMBOL`. (The indicators persist across the symbol change — they are layout-level.)
4. **Verify and report.** Call `chart_get_state` and confirm all 5 indicators above are present on `CURRENT_SYMBOL`. Briefly report the final symbol + indicator list.

## Notes & edge cases

- **Keep the current symbol** — the whole point is to restore the *indicator set / timeframe / chart type*, not to change the ticker. Only fall back to the layout's symbol if `CURRENT_SYMBOL` could not be read.
- **Custom indicator settings** (e.g., the Future Forward Curve's lookbacks, rebase, DTE spacing, filters) come from the layout/saved Pine script — do not reset them.
- **Missing layout:** if `layout_switch` cannot find `Daily Default` (e.g., it was deleted), tell the user and offer to recreate it: open the chart with the desired indicators, then TradingView layout menu → "Make a copy…" → name it `Daily Default`. Do not silently re-add indicators by name (tuned settings and any favorited/community scripts would be lost).
- **Unsaved-changes prompt:** `layout_switch` auto-dismisses the "unsaved changes" dialog. If the user had unsaved edits on their previous layout, mention they may have been discarded.
- After running, the active layout becomes `Daily Default`. That is expected.

## To update the default later

If the user changes their standard indicator set and wants `/daily-default` to match, re-save the layout: with the new setup on screen, TradingView layout menu → "Make a copy…" (overwrite/replace `Daily Default`) or save over it. Then this skill will restore the new set.
