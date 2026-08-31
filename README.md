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
| `orderbook` | Full order-book snapshots. Throttled to 1/sec per market through 2026-08-24; **unthrottled from 2026-08-25**, so every snapshot the venue pushed is archived — roughly ten times the detail |
| `markets` | Per-market metadata with **strike** (`start_price`) and **settlement price** (`end_price`) — every slot's outcome is verifiable from this file alone |
| `klines` | 13 intraday periods (1s..1d) plus 3d / 1w / 1mo full-history snapshots, derived from the price ticks |

- Assets: BTC, ETH, BNB × intervals 5m / 15m / daily
- History from 2026-06-12, growing daily. Note that **2026-06-12 is a partial
  day** (collection started mid-day; ~66% of the day's seconds) — the **first
  complete UTC day is 2026-06-13**. Early days also carry fewer ETH/BNB slots,
  which only had daily markets at the time.
- Every file ships with row counts + SHA-256 in the manifest, so you can verify
  what you received; the price files carry per-tick timestamps, so feed
  continuity is auditable directly from the data

## Samples / 样例

**[Download the sample bundle](https://github.com/Ligengxin96/predict.fun-data-samples/releases/latest/download/predict-fun-data-samples.tar.gz)**
— one real, unmodified UTC day (**2026-08-25**) of the BTC 5-minute series plus
the settlement price feed, the full market/settlement table and one kline period.

> **The data is in the Release, not in the git tree.** Clicking *Code → Download
> ZIP* gets you the documentation and nothing else. This is deliberate: a
> half-populated archive would replay without errors and give wrong answers, so
> the repository ships either the whole day or none of it.
>
> **数据在 Release 里，不在 git 仓库里。** 点 *Code → Download ZIP* 只会拿到文档。
> 这是有意的：残缺的归档跑起来不报错却会给出错误结果，所以要么给完整一天，要么不给。

```bash
curl -L https://github.com/Ligengxin96/predict.fun-data-samples/releases/latest/download/predict-fun-data-samples.tar.gz | tar xz
```

It unpacks to `predict-fun-data-samples/`, laid out exactly like the paid
archive, so it replays directly:

```bash
# The order book is unthrottled from 2026-08-25 — 679,335 full snapshots in this
# one day — so give the replay room. The default Node heap is not enough.
NODE_OPTIONS=--max-old-space-size=8192 ot run . --data ./predict-fun-data-samples
```

| file (inside the bundle) | rows | what |
|---|---|---|
| `data/predict-fun/prices/BTCUSDT/BTCUSDT-feed1-predict-prices-2026-08-25.csv.gz` | 82,568 | settlement price ticks |
| `data/predict-fun/orderbook/BTC-5M/BTC-5M-predict-orderbook-2026-08-25.jsonl.gz` | 679,335 | order-book snapshots, unthrottled |
| `data/predict-fun/markets/predict-markets-2026-08-25.jsonl.gz` | 1,245 | every market that day (all assets and intervals) with strike + settlement price |
| `data/predict-fun/klines/BTCUSDT/1m/BTCUSDT-feed1-1m-2026-08-25.csv.gz` | 1,440 | 1-minute klines (one of 13 periods in the full set) |

This day is the first full one with the order book unthrottled: 679,335
snapshots against 68,398 on a throttled day, roughly ten times the book detail.

本样例日是盘口不再限流后的第一个完整日：679,335 条快照，而限流时期的日子只有
68,398 条，盘口细节约为此前的十倍。

Each file is also attached to the Release individually, for anyone who only
wants one of them. Row counts are data rows (CSV headers excluded); every file
carries its SHA-256 in [`samples/manifest.json`](samples/manifest.json). Those
checksums are the archive's own, so a sample verifies byte for byte against a
delivered bundle.

Field-level documentation: [`DATA_GUIDE.md`](DATA_GUIDE.md) (English) /
[`数据使用说明.md`](数据使用说明.md) (中文).

### Verify it yourself / 自行验证

The venue publishes each market's own `start_price` (the strike) and
`end_price` (the settlement price). What matters about our data is whether the
price stream we sell is the one those numbers came from — so that is what this
check asks: **does our archived tick at those exact seconds reproduce the
venue's published prices?**

On 2026-08-25, across the 291 BTC 5-minute markets in the sample:

| | count | result |
|---|---|---|
| boundary seconds we hold a tick for | 555 | **555 of 555 reproduce the venue's published price** |
| boundary seconds absent from our feed | 27 | not independently provable — reported as undetermined |

Prices are compared at the precision the venue itself printed. Our files carry
the full float64 expansion (`64944.024999999994`), the venue prints the same
number as `64944.025`; those are one value, not a disagreement.

The venue's own settlement rule is then plain arithmetic on the same file:
**Up wins when `end_price >= start_price`** — 145 Up and 146 Down that day.

场方为每个市场发布自己的 `start_price`（strike）与 `end_price`（结算价）。对我们的
数据而言真正要紧的是：**我们卖的这条价格流，是不是它结算时用的那条**。所以这里核对的
是「我方在那两个精确秒归档的 tick，能否复现场方发布的价格」。

2026-08-25 当天样例中的 291 个 BTC 5 分钟局：我方持有边界秒 tick 的 **555 个，
555/555 全部与场方发布价一致**；另有 27 个边界秒不在我方 feed 中，按不可独立证明
处理、不作判定。比较按场方自己打印的精度进行——我方文件存的是 float64 的完整展开
（`64944.024999999994`），场方打印为 `64944.025`，二者是同一个数、不是分歧。

场方的结算规则本身则可直接在同一文件上算：**`end_price >= start_price` 时 Up 赢**，
当天 145 Up / 146 Down。

## Data quality on the sample day / 样例当天的数据质量

- Settlement feed: **82,568 of the day's 86,400 seconds (95.6%)** carried a
  report, with **no gap longer than 5 seconds** across the whole 24 hours.
- Order book: 679,335 snapshots for BTC-5M alone, unthrottled.
- Capture latency for the order book, measured on this day:
  **p50 63 ms, p95 199 ms** (`recv_ms − update_ts_ms`).
- Collection point sits next to the venue's own infrastructure in Tokyo.

You can check the feed-continuity claim yourself: `publish_time` in the price
file is a unix second, so the gap distribution is computable straight from the
sample.

结算价流当天覆盖 86,400 秒中的 **82,568 秒（95.6%）**，**最大间隔 5 秒**；仅
BTC-5M 一个系列就有 679,335 条盘口快照（未限流）；盘口采集延迟当日实测
**p50 63 毫秒、p95 199 毫秒**（`recv_ms − update_ts_ms`）；采集点部署在东京，紧邻
场方基础设施。价格文件的 `publish_time` 是 unix 秒，间隔分布可直接从样例自行核算。

## Known limits — stated up front / 已知限制（先说清楚）

- Prices are **float64 only**. The upstream publishes no full-precision integer
  string for this venue, so none is archived.
- Order book is **snapshots only** — the upstream stream carries no deltas, and
  there is no trade tape. (Our Polymarket dataset does have both.)
- The per-tick `provider` field is **mixed** — see `DATA_GUIDE.md`. Market-level
  settlement source is `CHAINLINK` for 5m/15m and `BINANCE` for daily.
- The price feed's upstream timestamps (`publish_time`, `server_ts`) are **whole
  seconds**, so a millisecond-level capture latency cannot be derived for it —
  the quantisation is larger than the latency being measured. `recv_ms` is our
  own millisecond receive time, and the order book does carry a millisecond
  upstream timestamp, which is why the latency figure above is quoted for the
  book only. We would rather omit a number than publish one its inputs cannot
  support.

价格仅 float64（上游无全精度串）；盘口**只有快照**，上游不发增量，也没有成交
流水（我们的 Polymarket 数据集两者都有）；逐条 tick 的 `provider` 字段是**混合**
的，详见说明书——市场层面的结算源 5m/15m 为 `CHAINLINK`、daily 为 `BINANCE`。
价格流的上游时间戳（`publish_time`、`server_ts`）**只到整秒**，因此无法据此给出
毫秒级采集延迟——量化误差比要测的延迟本身还大；`recv_ms` 是我方毫秒级接收时间，
而盘口带有毫秒级上游时间戳，所以上面的延迟数字只给盘口。**宁可不给一个数字，
也不给一个它的输入撑不起来的数字。**
历史起点 2026-06-12，但**该日为部分覆盖日**（当天中途开采，约占全天 66%），
**首个完整 UTC 日为 2026-06-13**；早期若干天 ETH/BNB 槽位较少（当时只有 daily 市场）。

## Buy / 购买

**[outcometick.com](https://outcometick.com)** — subscribe or buy a date range,
then pull the data with your API key. Predict.fun is sold as its own venue tier,
or bundled with Polymarket. It is the same archive these samples come from,
scoped to exactly the assets, data types and dates you purchase.

Questions, or something not working:
**[Telegram group](https://t.me/+TcK_hJOzOz5mZTk1)** ·
**support@outcometick.com**

**[outcometick.com](https://outcometick.com)** —— 按订阅或按日期区间购买，用 API
key 自助取数。Predict.fun 可单独作为一个场子购买，也可与 Polymarket 打包。样例即
取自同一份归档，购买后按你选择的币种、数据类型与日期范围开放。

有疑问或遇到问题：**[Telegram 群](https://t.me/+TcK_hJOzOz5mZTk1)** ·
**support@outcometick.com**
