# Data Guide

(中文版见 数据使用说明.md)

## Layout

The sample bundle unpacks to a directory laid out exactly like the paid
archive, so anything written against a sample keeps working against a delivered
dataset unchanged:

```
predict-fun-data-samples/
  data/predict-fun/prices/BTCUSDT/BTCUSDT-feed<id>-predict-prices-<date>.csv.gz
  data/predict-fun/orderbook/BTC-5M/BTC-5M-predict-orderbook-<date>.jsonl.gz
  data/predict-fun/markets/predict-markets-<date>.jsonl.gz
  data/predict-fun/klines/BTCUSDT/1m/BTCUSDT-feed<id>-1m-<date>.csv.gz
```

The directory segments carry meaning, and tooling reads them:
`data/predict-fun/<dataset>/<series>/`, where the series key is the symbol for
prices and klines (`BTCUSDT`) and `<ASSET>-<INTERVAL>` for the order book
(`BTC-5M`, `ETH-15M`, `BNB-DAILY`). The markets table is one file per day
covering every asset and interval, so it has no series directory. A full
purchased dataset has the same shape with more assets, more intervals, all 13
kline periods and more days under it.

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
series and no trade tape. Book state at time t = the market's latest snapshot
with `recv_ms <= t`.

Throttling (disclosed), which changed and matters if you span the date:

- through 2026-08-24: kept at most 1 snapshot per market per second
- **from 2026-08-25: no throttling at all — every snapshot the upstream pushed
  is archived**

The earlier throttle dropped a large majority of upstream snapshots, so a day
from 2026-08-25 onward carries roughly ten times the book detail of an earlier
one. Compare for yourself: the sample day holds 679,335 snapshots for BTC-5M
where 2026-07-24 held 68,398.

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
| start_price | the strike — Up must close strictly above it |
| end_price | the settlement price |
| status | the market state as of our **last read** of it upstream — not a settlement flag, see below |

Settlement rule, three outcomes: `end_price > start_price` → Up wins;
`end_price < start_price` → Down wins; `end_price == start_price` → the slot is
a **push**, where the venue resolves both sides as won and stakes are returned.
A tie is not an Up win here — the opposite of Polymarket — and about 1 in 100
five-minute slots closes flat, so a binary `>=` predicate will misscore them.
Both values come from the upstream market object, so a market's outcome is
verifiable from this file alone; the price files let you audit the path in
between.

Do **not** use `status` to decide whether a slot has settled. We stop re-reading
a market once it has an `end_price`, and at that moment the venue very often
still reports it as `OPEN` — so `status` freezes at whatever it was then. Most 5m/15m slots do read `RESOLVED`, but **fewer than 1% of
settled daily slots ever do** — filtering on `status == 'RESOLVED'` silently
drops nearly every daily slot. The reliable test is whether `end_price` is
present; in our whole history no row carries `RESOLVED` without one.

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
