# Predict.fun Crypto Up/Down — Market Data

Tick-level market data for **Predict.fun** crypto Up/Down markets — the
prediction-market venue reachable from the Binance Wallet front end (BNB
chain) — collected 24/7 directly from the source feeds. This repository hosts
**free samples and documentation**; the full dataset is sold by subscription or
by range.

本仓库提供 **Predict.fun**（币安钱包前端接入的预测市场，BNB 链）加密 Up/Down
市场数据的**免费样例与文档**；完整数据集按订阅或按区间出售。

> This is a **separate venue** from Polymarket, with a different settlement
> source. It is collected and sold independently and is never mixed into our
> Polymarket dataset. For that one, see
> [`polymarket-data-samples`](https://github.com/Ligengxin96/polymarket-data-samples).
>
> 本数据集与 Polymarket 是**不同场子、不同结算源**，独立采集、独立出售，绝不与
> Polymarket 数据混装。

## What the full dataset contains / 完整数据集内容

| dataset | description |
|---|---|
| `prices` | The settlement price feed, tick-by-tick (~1Hz per symbol), with the upstream's own per-tick `provider` attribution and three timestamps |
| `orderbook` | Full order-book snapshots, up to 1/sec per market |
| `markets` | Per-market metadata with **strike** (`start_price`) and **settlement price** (`end_price`) — every slot's outcome is verifiable from this file alone |
| `klines` | 13 intraday periods (1s..1d) plus 3d / 1w / 1mo full-history snapshots, derived from the price ticks |

- Assets: BTC, ETH, BNB × intervals 5m / 15m / daily
- History from 2026-06-12, growing daily
- Every file ships with row counts + SHA-256 in the manifest, so you can verify
  what you received; the price files carry per-tick timestamps, so feed
  continuity is auditable directly from the data

## Samples / 样例

**Easiest download: [Releases → samples-v1](https://github.com/Ligengxin96/predict.fun-data-samples/releases/tag/samples-v1)**
(the same files also live in `samples/` for browsing).

One real, unmodified UTC day — **2026-07-24** — of the BTC 5-minute series plus
the settlement price feed, the full market/settlement table and one kline
period:

| file | rows | what |
|---|---|---|
| `BTCUSDT-feed1-predict-prices-2026-07-24.csv.gz` | 82,523 | settlement price ticks |
| `BTC-5M-predict-orderbook-2026-07-24.jsonl.gz` | 68,398 | order-book snapshots |
| `predict-markets-2026-07-24.jsonl.gz` | 1,245 | every market that day (all assets and intervals) with strike + settlement price |
| `BTCUSDT-feed1-1m-2026-07-24.csv.gz` | 1,440 | 1-minute klines (one of 13 periods in the full set) |

Field-level documentation: [`DATA_GUIDE.md`](DATA_GUIDE.md) (English) /
[`数据使用说明.md`](数据使用说明.md) (中文).

### Verify it yourself / 自行验证

Every settled market in `predict-markets-2026-07-24.jsonl.gz` carries both
`start_price` (the strike) and `end_price` (the settlement price), so you can
check the Up/Down outcome directly: **Up wins when `end_price >= start_price`.**
On 2026-07-24 every 5m and 15m slot of all three assets is present and settled
(288 five-minute + 96 fifteen-minute slots per asset). The price files let you
replay the path between the two.

`predict-markets-*.jsonl.gz` 里每个已结算市场都同时带 `start_price`（strike）与
`end_price`（结算价），可直接核对 Up/Down 结果：**`end_price >= start_price` 时
Up 赢**。2026-07-24 当天三个资产的 5m / 15m 全部槽位齐全且已结算（每资产 288 个
5 分钟局 + 96 个 15 分钟局）。价格文件可用于复盘中间过程。

## Data quality on the sample day / 样例当天的数据质量

- Settlement feed: all three feeds carried a report for **82,523 of the day's
  86,400 seconds (95.5%)**, with **no gap longer than 6 seconds** across the
  whole 24 hours — zero outages over 60s.
- Markets: every 5m and 15m slot of all three assets present and settled
  (288 + 96 per asset).
- Collection point sits next to the venue's own infrastructure in Tokyo.

You can check the feed-continuity claim yourself: `publish_time` in the price
file is a unix second, so the gap distribution is computable straight from the
sample.

结算价流当天三条 feed 均覆盖 86,400 秒中的 **82,523 秒（95.5%）**，**最大间隔 6
秒**，全天无超过 60 秒的断档；三个资产的 5m / 15m 槽位全部齐全且已结算（每资产
288 + 96 个）；采集点部署在东京，紧邻场方基础设施。价格文件的 `publish_time` 是
unix 秒，间隔分布可直接从样例自行核算。

## Known limits — stated up front / 已知限制（先说清楚）

- Prices are **float64 only**. The upstream publishes no full-precision integer
  string for this venue, so none is archived.
- Order book is **snapshots only** — the upstream stream carries no deltas, and
  there is no trade tape. (Our Polymarket dataset does have both.)
- The per-tick `provider` field is **mixed** — see `DATA_GUIDE.md`. Market-level
  settlement source is `CHAINLINK` for 5m/15m and `BINANCE` for daily.

价格仅 float64（上游无全精度串）；盘口**只有快照**，上游不发增量，也没有成交
流水（我们的 Polymarket 数据集两者都有）；逐条 tick 的 `provider` 字段是**混合**
的，详见说明书——市场层面的结算源 5m/15m 为 `CHAINLINK`、daily 为 `BINANCE`。

## Buy / 购买

Telegram: **@hankson_level** — delivery is an expiring private download link
(tar bundle with checksums and the data guide), scoped to exactly the assets,
data types and date range you purchase.
