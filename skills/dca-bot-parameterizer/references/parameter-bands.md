# Parameter Bands

Use this file only after the main skill workflow has:

1. passed the recommendation gate,
2. classified the market state,
3. measured current `ATR%`,
4. identified the relevant structure,
5. determined whether the setup is spot or contract.

## OKX MCP Fetch Rules

Use OKX public market data in this sequence:

1. `okx-live.market_get_ticker`
2. `okx-live.market_get_candles`
3. `okx-live.market_get_instruments` when contract leverage metadata is needed
4. if a live market request fails, retry the same request with the `okx-demo` market tool

Use 60 candles on the working timeframe. Use 60 candles on the higher timeframe only when contract DCA needs higher-timeframe bias.

## Market-State Policy

- `Range`
  - allow standard DCA configuration
  - favor normal-sized `initOrdAmt`
  - keep `pxStepsMult` low
  - allow a moderate `volMult`

- `Mild Trend`
  - allow DCA with more conservative spacing
  - reduce `initOrdAmt`
  - raise `pxSteps` modestly above `ATR%`
  - keep `volMult` lower than range conditions

- `Strong Trend`
  - default to warning and stop
  - continue only after explicit user insistence
  - if forced to continue, use defensive mode

## Direction and Structure Rules

- spot DCA
  - always `long`
  - use nearest support

- contract DCA in `Mild Trend` or `Strong Trend`
  - rising `EMA20` with price mostly above it => `long`
  - falling `EMA20` with price mostly below it => `short`

- contract DCA in `Range`
  - inspect the next higher timeframe
  - higher timeframe uptrend => `long`
  - higher timeframe downtrend => `short`
  - higher timeframe also range => default `long`

- structure by side
  - `long` => nearest valid support
  - `short` => nearest valid resistance

## Simple Parameter Bands

These are starting bands, not automatic outputs. Always validate budget and cumulative coverage.

### `pxSteps`

- `Range`
  - major coin: around `0.9x` to `1.1x` current `ATR%`
  - high-volatility altcoin: around `1.0x` to `1.25x` current `ATR%`

- `Mild Trend`
  - major coin: around `1.05x` to `1.25x` current `ATR%`
  - high-volatility altcoin: around `1.15x` to `1.4x` current `ATR%`

- `Strong Trend` with forced continuation
  - use clearly wider spacing than `Mild Trend`

### `pxStepsMult`

- `Range`: `1.05` to `1.20`
- `Mild Trend`: `1.15` to `1.35`
- `Strong Trend` with forced continuation: `1.30+`

### `initOrdAmt`

Use it as a share of total budget:

- strong entry quality: `12%` to `18%`
- neutral entry quality: `8%` to `12%`
- weak entry quality: `5%` to `8%`

Entry quality is stronger when:

- market is in `Range`
- relevant structure is near
- volatility is not elevated for that asset

### `safetyOrdAmt`

- usually keep it near `initOrdAmt`
- common starting point: `0.8x` to `1.2x` of `initOrdAmt`

### `volMult`

- `Range`: `1.15` to `1.35`
- `Mild Trend`: `1.05` to `1.20`
- `Strong Trend` with forced continuation: keep it restrained, usually near `1.0` to `1.10`

### `tpPct`

Anchor it to `ATR%` and cost:

- lower-volatility conditions: slightly below or around `ATR%`
- medium-volatility conditions: around `ATR%`
- higher-volatility conditions: around `1.0x` to `1.4x` `ATR%`

Never place `tpPct` so low that fees and slippage erase the trade's edge.

## Contract Leverage Matrix

Recommend leverage only. Do not modify account settings.

First classify asset type:

- major coin
- high-volatility altcoin

Then classify volatility regime from current `ATR%` versus the 20-bar average `ATR%` on the same timeframe:

- `Low`: current `ATR% < 0.9x` average `ATR%`
- `Normal`: `0.9x` to `1.2x`
- `High`: `> 1.2x`

Use this exact matrix before applying caps:

- major coin, `Range`, `Low` or `Normal` volatility => `3x`
- major coin, `Range`, `High` volatility => `2x`
- major coin, `Mild Trend`, `Low` or `Normal` volatility => `2x`
- major coin, `Mild Trend`, `High` volatility => `1x`
- major coin, `Strong Trend` after explicit confirmation => `1x`
- high-volatility altcoin, `Range`, `Low` or `Normal` volatility => `2x`
- high-volatility altcoin, `Range`, `High` volatility => `1x`
- high-volatility altcoin, `Mild Trend`, any volatility => `1x`
- high-volatility altcoin, `Strong Trend` after explicit confirmation => `1x`

After selecting a leverage value, cap it by:

- the OKX instrument maximum leverage from `market_get_instruments`
- an internal hard cap of `4x`

If the user explicitly states low risk tolerance or a tight maximum drawdown, reduce the recommendation by one step with a floor of `1x`.

## Validation Checklist

After drafting parameters, always check:

1. How much `USDT` is consumed if every safety order fills.
2. How far from entry the final safety order lands.
3. Whether the final safety order lands too far beyond the relevant structure.
4. Whether the setup still matches the user's risk tolerance.
5. For contract DCA, whether the recommended leverage is still conservative relative to state and volatility.

## Warning Template

Use this warning before giving parameters in unsuitable conditions:

`This market state is not suitable for standard DCA under this method. The move is too one-sided, which increases the chance of filling safety orders into continued trend pressure. Wait for a better range structure or a cleaner reaction at support or resistance. If you still want to continue, say so explicitly and I will switch to a defensive configuration.`
