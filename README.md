# DCA Bot Parameterizer Codex

A skill that recommends optimized DCA (Dollar-Cost Averaging) bot parameters for OKX, driven by real-time market data and a strict six-step analytical workflow.

## What It Does

This skill analyzes live market conditions and generates ready-to-use parameter sets for OKX's Spot DCA and Contract DCA bots. Instead of guessing or copying generic settings, it reads real-time price, volatility, and structure data from OKX's MCP endpoints, classifies the current market state, and produces parameters tailored to that exact moment.

It outputs all seven core DCA parameters — `initOrdAmt`, `safetyOrdAmt`, `maxSafetyOrds`, `pxSteps`, `pxStepsMult`, `volMult`, and `tpPct` — plus a leverage recommendation for contract setups.

## Supported Modes

- **Spot DCA** — long-only martingale on any USDT spot pair
- **Contract DCA (Long / Short)** — leveraged martingale on USDT-margined perpetual swaps, with automatic direction selection

## How It Works

The skill follows a fixed six-step method:

1. **Define the trading scene** — collect symbol, mode, budget, timeframe, and risk preferences
2. **Classify market state** — compute EMA20 on 60 candles to determine Range, Mild Trend, or Strong Trend
3. **Measure volatility** — derive ATR% and compare it against the 20-bar average to classify the volatility regime (Low / Normal / High)
4. **Inspect structure** — locate the nearest swing-low support (for longs) or swing-high resistance (for shorts) from the last 20 candles
5. **Generate parameters** — apply market-state-specific bands from the reference table to produce each parameter
6. **Validate** — simulate all safety-order fills to check total capital usage, downside coverage depth, and structural alignment

## Safety Gates

Two gates protect users from poor setups:

- **Recommendation Gate** — the skill asks for explicit consent before generating any recommendations. Users can decline and set parameters manually.
- **Unsafe Setup Gate** — if the market is in a Strong Trend, the skill issues a warning and halts. It only continues after the user explicitly confirms, at which point it switches to a defensive configuration with reduced sizing, widened spacing, and minimum leverage.

## Market Classification

The skill classifies market state using EMA20 and ATR on the working timeframe:

| State | Criteria |
|---|---|
| **Strong Trend** | EMA20 clearly directional, price on the same side, 4+/5 closes aligned, distance ≥ 1×ATR |
| **Mild Trend** | EMA20 directional, price on the same side, 3+/5 closes aligned, distance < 1×ATR |
| **Range** | Anything that doesn't meet the trend conditions above |

## Contract Direction Logic

For contract DCA, the skill determines long or short automatically:

- **Trending market** — follows the working-timeframe trend direction
- **Range market** — inspects the next higher timeframe (1h→4H, 4h→1D, 1d→1W) to decide; defaults to long if the higher timeframe is also ranging

## Parameter Bands (Summary)

| Parameter | Range | Mild Trend | Strong Trend (Defensive) |
|---|---|---|---|
| `pxSteps` | ~1× ATR% | ~1.1–1.4× ATR% | Clearly wider |
| `pxStepsMult` | 1.05–1.20 | 1.15–1.35 | 1.30+ |
| `initOrdAmt` | 12–18% of budget | 8–12% | 5–8% |
| `volMult` | 1.15–1.35 | 1.05–1.20 | 1.0–1.10 |
| `tpPct` | ~ATR% | ~ATR% | ~ATR% |

Values shift further for high-volatility altcoins versus major coins (BTC, ETH).

## Contract Leverage Matrix

Leverage is recommended, never auto-set. The matrix considers asset type, market state, and volatility regime, with an internal hard cap of 4× and a floor of 1×:

| Asset | Range (Low/Normal Vol) | Range (High Vol) | Mild Trend | Strong Trend |
|---|---|---|---|---|
| Major coin | 3× | 2× | 2× (1× if High Vol) | 1× |
| High-vol altcoin | 2× | 1× | 1× | 1× |

## Data Source

All market data comes from OKX MCP endpoints in real time:

- `market_get_ticker` — current price
- `market_get_candles` — OHLCV candles (60 bars on the working timeframe)
- `market_get_instruments` — contract specs and max leverage

The skill uses the live OKX environment by default and falls back to the demo environment if a live call fails.

## Required Inputs

| Input | Required | Example |
|---|---|---|
| Symbol | Yes | `BTC`, `ETH`, `DOGE`, or full instId like `SOL-USDT-SWAP` |
| Mode | Yes | Spot DCA or Contract DCA |
| Budget | Yes | `500 USDT` |
| Timeframe | Yes | `1h`, `4h`, or `1d` |
| Risk preference | Optional | Max drawdown %, or "low risk" |

## Output Format

Every recommendation includes the following sections in order:

1. **Scene** — trading setup summary
2. **Recommendation Gate Result** — passed / declined
3. **Market State** — Range / Mild Trend / Strong Trend
4. **Higher Timeframe Bias** — included when the working timeframe is Range (contract only)
5. **Volatility** — ATR%, volatility regime, and comparison to average
6. **Structure** — nearest support or resistance with distance from current price
7. **Direction** — long or short (contract only)
8. **Decision** — rationale for the parameter choices
9. **Warning** — present when the setup is suboptimal
10. **Recommended Leverage** — contract only
11. **Parameters** — the full parameter table
12. **Validation** — capital usage, coverage depth, and structural checks

## Example Trigger Phrases

- `帮我开 DCA bot`
- `open a contract DCA on SOL`
- `推荐 BTC 现货 DCA 参数`
- `开合约 dca，ETH，1000u`
- `What pxSteps should I use for DOGE?`

## Skill Structure

```
dca-bot-parameterizer-codex/
├── SKILL.md                          # Main workflow and rules
└── references/
    └── parameter-bands.md            # Parameter bands, leverage matrix, validation checklist
```

## Limitations

- The skill recommends parameters only — it does not auto-create bots without user confirmation
- Leverage is recommended, never applied to the account directly
- Only three market indicators are used (EMA20, ATR%, swing structure); the skill intentionally avoids indicator overload
- The skill cannot predict future price action; recommendations reflect the current market snapshot
