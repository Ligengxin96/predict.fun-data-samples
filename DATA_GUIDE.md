# Data Guide

(中文版见 数据使用说明.md)

Predict.fun is the prediction-market venue reachable from the Binance Wallet
front end (a BNB-chain venue). This dataset is collected independently from our
Polymarket dataset and is **never mixed with it** — different venue, different
settlement source, separate files.

## <SYMBOL>-feed<id>-predict-prices-<date>.csv.gz — settlement price ticks

| column | meaning |
|---|---|
| price_feed_id | upstream feed id (1 = BTCUSDT, 2 = ETHUSDT, 3 = BNBUSDT) |
| symbol | feed symbol, e.g. BTCUSDT |
| provider | source the upstream attributes this tick to: `CHAINLINK` or `BINANCE` — see the note below |
| publish_time | price event time (unix **seconds**) |
| server_ts | upstream message timestamp (unix **seconds**) |
| price | price as float64 |
| recv_ms | collector receive time (unix **milliseconds**) |

Note on `provider`: the upstream feed carries a per-tick source attribution and
it is **not constant** — on 2026-07-24 roughly 90% of ticks on each feed were
attributed to `BINANCE` and 10% to `CHAINLINK`. We archive the field verbatim
rather than normalising it, so you can filter or weight by source yourself. Do
not assume a single provider for the whole file.

Note on precision: unlike our Polymarket settlement feed, Predict.fun publishes
this price as a float64 only — there is no full-precision integer string
upstream, so none is archived. Ticks are keyed by
`(price_feed_id, publish_time, price)`, which absorbs the duplicate frames the
upstream WebSocket sends for each publish time.

## <ASSET>-<INTERVAL>-predict-orderbook-<date>.jsonl.gz — order-book snapshots

Series key is `<ASSET>-<INTERVAL>`, e.g. `BTC-5M`, `ETH-15M`, `BNB-DAILY`.

| field | meaning |
|---|---|
| market_id | upstream market id |
| category_slug | market slug; suffix = slot start (unix sec) |
| update_ts_ms | upstream book update time (ms) |
| recv_ms | collector receive time (ms) |
| payload | the upstream snapshot object, stored verbatim |

`payload` contains `bids` / `asks` as `[price, size]` pairs, plus the upstream's
own `version`, `marketId`, `orderCount`, `lastOrderSettled` and
`settlementsPending` fields exactly as received.

Note: these are **full snapshots only**. Predict.fun's stream does not publish
order-book deltas, so unlike our Polymarket dataset there is no `price_change`
series and no trade tape. Snapshots are kept-first throttled to 1 per second
per market (disclosed); book state at time t = the market's latest snapshot
with `recv_ms <= t`.

## predict-markets-<date>.jsonl.gz — market metadata and settlement outcome

One file per day covering every asset and interval.

| field | meaning |
|---|---|
| category_slug | market id; suffix = slot start (unix sec) |
| asset | btc / eth / bnb |
| interval_label | 5m / 15m / daily |
| market_id | upstream market id (joins to the order-book files) |
| price_feed_id / price_feed_symbol | which feed settles this market |
| price_feed_provider | settlement source for this market: `CHAINLINK` for 5m and 15m, `BINANCE` for daily |
| condition_id | on-chain condition id |
| start_sec / end_sec | slot boundaries (unix sec) |
| start_price | the strike (price to beat) |
| end_price | the settlement price |
| status | e.g. RESOLVED |

Settlement rule: Up wins when `end_price >= start_price`. Both values come from
the upstream market object, so a market's outcome is verifiable from this file
alone; the price files let you audit the path in between.

## <SYMBOL>-feed<id>-<period>-<date>.csv.gz — klines

| column | meaning |
|---|---|
| ts_ms | bucket open time (ms) |
| open / high / low / close | float64 prices within the bucket |
| ticks | number of ticks aggregated into the bucket |

Periods 1s..1d are per-day files; 3d / 1w / 1mo ship as full-history snapshots
named `-thru-<date>` and contain completed buckets only. Klines are derived
from the price ticks above — you can always recompute them yourself.

## manifest.json — file inventory with per-file row counts and sha256 checksums

Every sample file here is byte-identical to the corresponding file in the paid
dataset; the sha256 values match the archive's own checksum index.
