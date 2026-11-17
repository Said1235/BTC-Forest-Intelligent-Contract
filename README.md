# BTC Forecast

A daily, **non-financial** prediction game on **Binance Spot BTCUSDT 1d candles (UTC)**, implemented as a single GenLayer Intelligent Contract (`BTCForecast`).

Users predict `UP`, `DOWN` or `FLAT` before a daily candle opens. After the candle closes, anyone can resolve the market through a validator-checked, two-stage process. A correct prediction earns one point, an incorrect one earns zero.

There are no bets, deposits, withdrawals, tokens, payments, administrators or manual overrides, and **no LLM is used anywhere**: the result is a mathematical comparison of two exact decimal prices.

> **Status: not production-ready.** The contract passes its Direct Mode test suite, but it has **not** been run on a real GenLayer Studio or network. Several security-relevant points are still unverified. Read [`docs/BTC_FORECAST_SECURITY_AUDIT.md`](docs/BTC_FORECAST_SECURITY_AUDIT.md) before deploying anything.

## How it works

- **Market identity:** a market is the UTC opening timestamp of a daily candle (`market_open_time_ms`, aligned to 86,400,000 ms). Exchange, symbol, interval and timezone are fixed constants. There is no `create_market` and no registry.
- **Predictions:** `submit_prediction(t, "UP" | "DOWN" | "FLAT")`. The owner is the transaction sender. A prediction can be edited until the candle opens (`now < t`), and only for the next seven daily markets.
- **Resolution (permissionless, two stages):**
  1. Once `now >= t + 86,400,000 + 300,000` ms (candle closed + 5 min), anyone calls `resolve_market(t)`. Validators independently fetch the candle; the accepted evidence is stored as a **provisional candidate** (not a settlement).
  2. At least 5 minutes later, another call fetches again. If **every** evidence field matches the candidate, the market is finalized. If anything differs, the candidate is replaced and the 5-minute timer restarts.
  3. A resolved market returns its stored result and is never refetched or changed.
- **Failures never become results:** HTTP errors, timeouts, malformed or missing data fail the transaction with no state change; a later call retries.
- **Claims:** `claim_points(t)` works only after final settlement, once per user and market.

Outcome rule: `UP` if close > open, `DOWN` if close < open, `FLAT` if equal, using exact decimal arithmetic (no floats), so `1.0 == 1.000`.

## Public interface

| Method | Kind | Notes |
|---|---|---|
| `submit_prediction(market_open_time_ms, prediction)` | write | caller is the owner |
| `resolve_market(market_open_time_ms)` | write | returns a status dict: `CANDIDATE_RECORDED`, `CANDIDATE_REPLACED` or `RESOLVED` |
| `claim_points(market_open_time_ms)` | write | one-time per user and market |
| `get_prediction(market_open_time_ms, user_address)` | view | |
| `get_market_candidate(market_open_time_ms)` | view | provisional evidence and `confirmable_at_ms` |
| `get_market_result(market_open_time_ms)` | view | final settlement only |
| `get_user_points(user_address)` | view | |

The constructor takes no arguments.

## Repository layout

```
contracts/btc_forecast.py                    the Intelligent Contract
tests/direct/btc_helpers.py                  shared fixtures, strict mocks, validator-vote model
tests/direct/test_btc_forecast.py            functional and exact-arithmetic tests (158)
tests/direct/test_btc_forecast_security.py   adversarial, lifecycle and validator tests (182)
deploy/deployScript.ts                       GenLayer deploy script (see "Deployment")
docs/BTC_FORECAST_CONTRACT_DESIGN.md         design, storage model, invariants, limitations
docs/BTC_FORECAST_SECURITY_AUDIT.md          findings, evidence, unresolved items
```

## Getting started

Requires Python 3.12.

```bash
pip install -r requirements.txt

# lint and validate the contract
genvm-lint check contracts/btc_forecast.py --json

# run the Direct Mode suite (340 tests)
python3 -m pytest tests/direct -q
```

Direct Mode uses **mocked** Binance responses and runs the leader path only; validator behavior is exercised through `vm.run_validator()` and a helper that reproduces the SDK's validator-sandbox rules. It does not prove real multi-validator consensus.

### Known tooling issue

`genlayer-test 0.29.2` asks for the GenVM bundle `v0.3.0-rc7`, whose download currently returns HTTP 404. The tests in this repository ran against the `v0.2.16` bundle that `genvm-lint` downloads. If direct tests fail while fetching the SDK, run `genvm-lint check contracts/btc_forecast.py` once, then copy the bundle into the test runner's cache:

```bash
mkdir -p ~/.cache/gltest-direct
cp ~/.cache/genvm-linter/genvm-universal-v0.2.16.tar.xz ~/.cache/gltest-direct/
```

## Deployment

`deploy/deployScript.ts` is the standard GenLayer deploy script pointed at `contracts/btc_forecast.py` with no constructor arguments. It is meant to be run through the GenLayer CLI from the repository root after selecting a network. **It has not been executed**: nothing in this repository has been deployed to any network, and the script depends on the GenLayer CLI / `genlayer-js` tooling, which this repository does not vendor.

The contract pins the runner `py-genlayer:1jb45aa8ynh2a9c9xn3b7qqh8sm5q93hwfp7jqmwsfhh8jpz09h6` (the older SDK generation). Confirm that your target network serves a compatible runner before deploying.

## Trust assumptions and limitations (summary)

- **Single source:** the only data source is `https://data-api.binance.vision/api/v3/klines`. Validators agreeing on Binance's data does not prove Binance is right, and an accepted settlement is never rewritten if Binance later corrects history.
- **Finality is policy-based:** the REST kline has no closed-candle flag, so protection is a 5-minute grace period plus two matching observations at least 5 minutes apart.
- **Still unverified:** how the transaction timestamp is set (it drives the prediction deadline), real multi-validator consensus, GenVM access to Binance, and Binance's `startTime`/`endTime` inclusivity.
- **Response size:** the 8,192-byte limit is applied after the response body is received; the SDK exposes no transport-level download cap.
- **Product choice to confirm:** resolution requires at least one prediction for the market.

Full details: [`docs/BTC_FORECAST_CONTRACT_DESIGN.md`](docs/BTC_FORECAST_CONTRACT_DESIGN.md) and [`docs/BTC_FORECAST_SECURITY_AUDIT.md`](docs/BTC_FORECAST_SECURITY_AUDIT.md).
