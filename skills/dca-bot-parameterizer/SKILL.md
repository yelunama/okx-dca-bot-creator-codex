---
name: dca-bot-parameterizer
description: Recommend OKX-backed DCA bot parameters for spot or contract setups from a strict market-state workflow. Use when the user asks to open or review a DCA bot, asks for DCA parameter recommendations, asks how to set `initOrdAmt`, `safetyOrdAmt`, `maxSafetyOrds`, `pxSteps`, `pxStepsMult`, `volMult`, `tpPct`, or contract leverage, or mentions opening spot or contract DCA in English or Chinese, including phrases such as `open DCA`, `contract DCA`, `swap DCA`, `开单dca`, `开dca`, or `开合约dca`.
---

# DCA Bot Parameterizer

Follow this workflow exactly. Keep the skill body in English. Do not add extra indicators unless the user explicitly overrides the method.

## Core Method

Use only these market inputs unless the user explicitly provides more:

- trend: `EMA20` plus price location relative to that average
- volatility: `ATR% = ATR / current price`
- structure:
  - spot DCA or contract `long`: nearest valid support
  - contract `short`: nearest valid resistance

Use this six-step method:

1. define the trading scene
2. classify market state
3. measure volatility
4. inspect structure
5. generate parameters
6. validate capital usage and coverage

## Recommendation Gate

When the user expresses intent to open a DCA bot, do not jump straight into recommendations.

1. Ask whether they want parameter recommendations.
2. If they decline, stop the recommendation flow and state that they can set custom parameters themselves.
3. Only continue if they explicitly want recommendations.

The gate result must appear in the final output as `Recommendation Gate Result`.

## Required Inputs

Collect or infer only what is necessary:

- symbol or `instId`
- mode: spot DCA or contract DCA
- asset type: major coin or high-volatility altcoin
- timeframe: `1h`, `4h`, or `1d`
- total budget in `USDT`
- optional risk input:
  - maximum drawdown or target coverage depth
  - explicit low risk tolerance

If the user provides a full OKX instrument ID, use it directly. Otherwise resolve:

- spot: `<BASE>-USDT`
- contract: `<BASE>-USDT-SWAP`

## OKX MCP Data Source

Use OKX MCP market data, not guesswork.

### Primary source

Use `okx-live` public market endpoints first:

- `market_get_ticker` for current price
- `market_get_candles` for candles
- `market_get_instruments` for contract leverage cap metadata

### Fallback

If an `okx-live` market call fails or returns unusable data, retry the same market request with `okx-demo`.

### Candle windows

Fetch:

- 60 candles on the working timeframe
- 60 candles on the higher timeframe only when contract DCA needs higher-timeframe bias

Use the fixed higher-timeframe mapping:

- `1h -> 4H`
- `4h -> 1D`
- `1d -> 1W`

## Market Classification

Use `EMA20` as the default moving average.

### Working-timeframe state

Derive the state from the last 60 candles on the working timeframe.

- `Strong Trend`
  - `EMA20` is clearly rising or falling
  - current price is on the same side of `EMA20`
  - at least 4 of the last 5 closes are on the same side of `EMA20`
  - current price is at least `1 x current ATR` away from `EMA20`

- `Mild Trend`
  - `EMA20` is rising or falling
  - current price is on the same side of `EMA20`
  - at least 3 of the last 5 closes are on the same side of `EMA20`
  - current price is less than `1 x current ATR` away from `EMA20`

- `Range`
  - any setup that does not meet the trend conditions above

State the market as one of:

- `Range`
- `Mild Trend`
- `Strong Trend`

## Contract Direction Logic

Spot DCA stays `long` only.

Contract DCA must output a direction.

### If the working timeframe is trending

Follow the working-timeframe trend:

- price mostly above a rising `EMA20` => `long`
- price mostly below a falling `EMA20` => `short`

### If the working timeframe is `Range`

Inspect the next higher timeframe:

- higher timeframe trending up => `long`
- higher timeframe trending down => `short`
- higher timeframe also `Range` => default `long`

When higher-timeframe bias is used, include it in the final output as `Higher Timeframe Bias`.

## Structure Logic

Use structure only after trend and volatility are known.

- spot DCA and contract `long`
  - use the nearest valid support
  - prefer the most recent visible swing low below current price from the last 20 closed candles
  - if no clean swing low exists, fall back to the lowest low of the last 20 closed candles

- contract `short`
  - use the nearest valid resistance
  - prefer the most recent visible swing high above current price from the last 20 closed candles
  - if no clean swing high exists, fall back to the highest high of the last 20 closed candles

Check:

- whether current price is reasonably close to the relevant structure
- whether planned safety orders land near meaningful structure instead of far beyond it

## Volatility Logic

Use only `ATR%`.

- current `ATR%` drives `pxSteps`
- current `ATR%` plus trading cost drives `tpPct`

For contract leverage, also compute a volatility regime on the working timeframe by comparing current `ATR%` with the 20-bar average `ATR%`:

- `Low`: current `ATR% < 0.9x` average `ATR%`
- `Normal`: current `ATR%` is `0.9x` to `1.2x` average `ATR%`
- `High`: current `ATR% > 1.2x` average `ATR%`

## Unsafe Setup Gate

Apply this gate before giving parameters for a poor setup.

- spot DCA in `Strong Trend`: warn first and stop unless the user explicitly insists on continuing
- contract DCA in `Strong Trend`: warn first and stop unless the user explicitly insists on continuing, even if direction follows trend

Accept clear confirmations such as:

- `continue`
- `continue anyway`
- `继续`
- `继续执行`
- equivalent explicit insistence

If the user insists on continuing in `Strong Trend`, switch to defensive mode:

- reduce `initOrdAmt`
- widen `pxSteps`
- increase `pxStepsMult`
- lower `volMult`
- tighten capital-usage and coverage checks
- for contract DCA, set recommended leverage to `1x`

## Parameter Rules

Apply these rules strictly.

### `pxSteps`

- set from current `ATR%`
- in `Range`, keep it near current `ATR%`
- in `Mild Trend`, set it slightly above current `ATR%`
- in `Strong Trend` after explicit confirmation, widen it clearly above `Mild Trend`
- widen it further for high-volatility altcoins

### `pxStepsMult`

- low in `Range`
- medium in `Mild Trend`
- high in `Strong Trend` only after explicit confirmation

### `initOrdAmt`

Base it on entry quality, not a formula.

- larger when the setup is ranging, structure is near, and volatility is not elevated
- smaller when the setup is trending, structure is far, or volatility is elevated

### `safetyOrdAmt` and `volMult`

- keep `safetyOrdAmt` close to `initOrdAmt` in most cases
- use a higher `volMult` in `Range`
- use a lower `volMult` in trend conditions
- keep `volMult` restrained in defensive mode

### `maxSafetyOrds`

- do not choose it first
- derive it only after spacing and cumulative coverage are drafted
- keep only as many safety orders as the budget and target coverage allow

### `tpPct`

- derive it from current `ATR%` and trading cost
- keep it above fees and slippage
- keep it realistic for the timeframe's usual rebound size

### Contract leverage

Recommend leverage only. Do not read or modify account leverage settings.

Use the exact leverage matrix from [references/parameter-bands.md](references/parameter-bands.md), then cap it by:

- the OKX instrument maximum leverage
- an internal hard cap of `4x`

If the user explicitly states low risk tolerance or a tight maximum drawdown, reduce the recommendation by one leverage step, with a floor of `1x`.

## Validation

Always validate:

- total capital usage if every safety order fills
- total downside coverage from entry to the final safety order
- whether the final safety order lands too far beyond the relevant structure
- whether the setup still fits the user's stated risk tolerance

If validation fails:

- increase `maxSafetyOrds` or `pxStepsMult` when coverage is too shallow
- increase `pxSteps` when the bot would fill too easily
- reduce `volMult` or `maxSafetyOrds` when capital usage is too heavy

## Output Format

Produce the final answer in this order:

1. `Scene`
2. `Recommendation Gate Result`
3. `Market State`
4. `Higher Timeframe Bias` when used
5. `Volatility`
6. `Structure`
7. `Direction` for contract DCA only
8. `Decision`
9. `Warning` when needed
10. `Recommended Leverage` for contract DCA only
11. `Parameters`
12. `Validation`

## Reference

For parameter bands, contract direction, leverage rules, and OKX MCP fallback behavior, read [references/parameter-bands.md](references/parameter-bands.md).
