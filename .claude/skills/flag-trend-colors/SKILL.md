---
name: flag-trend-colors
description: Flag every ticker in the "Stocks Watchlist" INDEX section with a color (green/yellow/red/pink) based on trend rules using SMA200/EMA8/EMA21. Use when the user asks to "flag the index tickers", "update the trend flags", or runs /flag-trend-colors.
---

# Flag INDEX Tickers by Trend Color

Colors each ticker in the watchlist's INDEX section based on where price sits
relative to the 200 SMA and the EMA8/EMA21 relationship. Uses the user's
*existing* chart indicators (no new indicators added, no bulk OHLCV history
pulled) to keep this cheap on tokens.

## Hardcoded ticker list (INDEX section, watchlist "Stocks Watchlist")

```
SPCFD:SPX
AMEX:SPY
NASDAQ:NDX
BCBA:QQQ
DJCFD:DJT
AMEX:XLB
AMEX:XLRE
AMEX:XLU
AMEX:XLI
AMEX:XLC
AMEX:XLY
AMEX:XLF
AMEX:XLV
AMEX:XLK
```

This list was read once from the live watchlist and hardcoded here since the
user confirmed it doesn't change. If the user says they added/removed a
ticker from the INDEX section, re-read it with `watchlist_get` (or by
inspecting the DOM per the "list all tickers" approach used previously) and
update this list.

## Prerequisites (assumed already configured on the user's chart)

- **Moving Average Ribbon** indicator with SMA200 and EMA21 among its lines
- **Bollinger Bands** indicator with basis set to EMA(8)

If `data_get_study_values` doesn't show a study called "Moving Average Ribbon"
or "Bollinger Bands", stop and ask the user — don't fall back to computing
from raw OHLCV history without checking with them first (that path is much
more expensive and was only used once, before this skill existed).

## Color rules

- **GREEN**: close > SMA200 AND EMA8 > EMA21 AND today's close > EMA21 AND yesterday's close > EMA21
- **YELLOW**: close > SMA200 but NOT all of the other two GREEN conditions
- **RED**: close < SMA200 AND EMA8 < EMA21 AND today's close < EMA21 AND yesterday's close < EMA21
- **PINK**: close < SMA200 but NOT all of the other two RED conditions

## Step 1: For each ticker in the list

1. `chart_set_symbol` — switch to the ticker (this drives which symbol
   `data_get_study_values` / `data_get_ohlcv` read from)
2. `data_get_study_values` — read current SMA200 and EMA21 from "Moving
   Average Ribbon", and EMA8 from "Bollinger Bands" basis
3. `data_get_ohlcv` with `count: 2` — get today's and yesterday's close only
   (cheap; do NOT pull large history for this)
4. Back-derive yesterday's EMA21 from today's EMA21 — no need for a long
   bar history just for one prior value:

   ```
   k = 2 / (21 + 1)
   EMA21_yesterday = (EMA21_today - close_today * k) / (1 - k)
   ```

   (Verified against a full from-scratch EMA computation on 2026-09-26 — this
   back-derivation matches exactly.)
5. Evaluate the color rules above using: close_today, SMA200, EMA8, EMA21_today,
   EMA21_yesterday, close_yesterday.

## Step 2: Apply the flag on the watchlist row

The chart symbol doesn't need to match the watchlist row to flag it — flagging
acts on the watchlist regardless of what's on the chart. Do this once per
ticker, right after computing its color (or batch all computations first,
then flag all — either order works).

1. Find the row and its center point:
   ```js
   (function(){
     var el = document.querySelector('[data-symbol-full="EXCHANGE:TICKER"]');
     if(!el) return {error:'not found — is it visible/scrolled into view in the watchlist?'};
     var rect = el.getBoundingClientRect();
     return {midX: rect.x+rect.width/2, midY: rect.y+rect.height/2};
   })()
   ```
   via `ui_evaluate`. If `error: not found`, the row may be scrolled out of
   view in the watchlist panel — scroll the watchlist panel and retry.
2. `ui_mouse_click` with `button: "right"` at that midX/midY to open the
   "Flag/Unflag TICKER" context menu.
3. Find the correct color swatch's real coordinates from the DOM — **do not
   guess coordinates from a screenshot**; TradingView Desktop runs at a
   devicePixelRatio that doesn't match screenshot pixel coordinates 1:1, and
   screenshot-based clicks have landed on the wrong element before. Use
   `ui_evaluate`:
   ```js
   (function(){
     var all = document.querySelectorAll('*');
     var target = null;
     for (var i=0;i<all.length;i++){
       var el = all[i];
       if (el.children.length===0 && el.textContent && el.textContent.trim().indexOf('Flag/Unflag')===0){
         target = el; break;
       }
     }
     if(!target) return {error:'menu not open'};
     var menu = target.closest('[class*="menu"]') || target.parentElement.parentElement;
     var colorMap = {
       green:  'rgb(129, 199, 132)',
       yellow: 'rgb(251, 192, 45)',
       red:    'rgb(255, 82, 82)',
       pink:   'rgb(244, 143, 177)',
     };
     var wantBg = colorMap['COLOR_NAME_HERE'];
     var out = null;
     Array.from(menu.querySelectorAll('*')).forEach(function(e){
       if (getComputedStyle(e).backgroundColor === wantBg) {
         var r = e.getBoundingClientRect();
         out = {x:r.x+r.width/2, y:r.y+r.height/2};
       }
     });
     return out || {error:'swatch not found — menu may not have opened'};
   })()
   ```
4. `ui_mouse_click` (left, default) at the returned x/y.
5. Verify by checking the flag actually applied (don't trust the click
   succeeded just because no error was thrown):
   ```js
   (function(){
     var el = document.querySelector('[data-symbol-full="EXCHANGE:TICKER"]');
     var flag = el.querySelector('.flag-ALH_8bJI, [class*="uiMarker"]');
     var cs = flag ? getComputedStyle(flag) : null;
     return {flagDisplay: cs ? cs.display : null, flagColor: cs ? cs.color : null};
   })()
   ```
   `flagDisplay` should be `"flex"` and `flagColor` should match the intended
   swatch's rgb value. If `flagDisplay` is `"none"`, the click missed —
   reopen the context menu and retry with freshly read coordinates (don't
   reuse coordinates from a previous ticker's menu; the menu position depends
   on where the row is on screen).

## Step 3: Report

Give the user a compact table: ticker, close, SMA200, EMA8, EMA21 (today),
color assigned. Flag any ticker where the numbers were borderline/ambiguous.
