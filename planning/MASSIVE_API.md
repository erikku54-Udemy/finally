# Massive API 說明文件

> Massive（前身為 Polygon.io，2025 年 10 月 30 日更名）提供美國股票市場的完整即時與歷史資料，涵蓋全美 19 個主要交易所、暗池及 OTC 市場。現有 API key 與整合皆可繼續使用，`api.polygon.io` 在更名後仍長期支援。

---

## 1. 基本資訊

| 項目 | 內容 |
|------|------|
| API 基礎網址 | `https://api.massive.com` |
| 舊版相容網址 | `https://api.polygon.io`（仍有效） |
| 認證方式 | Query param `?apiKey=<KEY>` 或 `Authorization: Bearer <KEY>` header |
| 資料格式 | JSON |
| 免費方案限制 | 每分鐘 **5 次**請求，15 分鐘延遲資料 |
| 付費方案 | 無限請求次數，進階/商業方案提供即時資料 |

---

## 2. Python 套件安裝

```bash
# 使用 uv（專案建議方式）
uv add massive

# 或使用 pip
pip install -U massive
```

**需求**：Python 3.9 或以上。

---

## 3. 認證初始化

```python
from massive import RESTClient

client = RESTClient(api_key="YOUR_MASSIVE_API_KEY")
```

---

## 4. 核心端點：即時與收盤價

### 4.1 全市場快照（一次取得多個 ticker）

這是本專案最重要的端點，可一次取得多個 ticker 的即時資料。

**端點：** `GET /v2/snapshot/locale/us/markets/stocks/tickers`

**Query 參數：**

| 參數 | 型別 | 說明 |
|------|------|------|
| `tickers` | string | 逗號分隔的 ticker 列表（如 `AAPL,MSFT,GOOGL`）；空白代表全市場 |
| `include_otc` | boolean | 是否包含 OTC 股票，預設 `false` |
| `apiKey` | string | API Key |

**回應結構：**

```json
{
  "status": "OK",
  "count": 3,
  "tickers": [
    {
      "ticker": "AAPL",
      "day": {
        "o": 185.20,
        "h": 188.45,
        "l": 184.80,
        "c": 187.65,
        "v": 52341200,
        "vw": 186.92
      },
      "min": {
        "o": 187.40,
        "h": 187.80,
        "l": 187.20,
        "c": 187.65,
        "v": 123400,
        "vw": 187.55
      },
      "prevDay": {
        "o": 183.10,
        "h": 186.20,
        "l": 182.50,
        "c": 185.92,
        "v": 48200000,
        "vw": 184.65
      },
      "lastTrade": {
        "p": 187.65,
        "s": 100,
        "t": 1705615200000000000,
        "x": 4
      },
      "lastQuote": {
        "P": 187.66,
        "S": 2,
        "p": 187.65,
        "s": 3,
        "t": 1705615200100000000
      },
      "todaysChange": 1.73,
      "todaysChangePerc": 0.93,
      "updated": 1705615200100000000
    }
  ]
}
```

**欄位說明：**

| 欄位 | 說明 |
|------|------|
| `day.c` | 今日目前（或收盤）價格 |
| `day.o` / `day.h` / `day.l` | 今日開/高/低價 |
| `day.v` | 今日成交量 |
| `lastTrade.p` | 最新成交價格（即時性最高） |
| `lastQuote.p` / `lastQuote.P` | 最新 bid / ask 價格 |
| `prevDay.c` | 前一交易日收盤價 |
| `todaysChangePerc` | 相對前日收盤的漲跌幅（%） |
| `updated` | 最後更新的 Unix nanosecond 時間戳 |

**Python 範例：**

```python
from massive import RESTClient

client = RESTClient(api_key="YOUR_API_KEY")

# 取得指定 tickers 的快照
tickers = ["AAPL", "MSFT", "GOOGL", "AMZN", "TSLA"]
snapshot = client.get_snapshot_all_tickers(
    locale="us",
    market_type="stocks",
    tickers=tickers
)

for ticker_data in snapshot.tickers:
    price = ticker_data.last_trade.price if ticker_data.last_trade else ticker_data.day.close
    print(f"{ticker_data.ticker}: ${price:.2f} ({ticker_data.todays_change_perc:+.2f}%)")
```

---

### 4.2 單一 Ticker 快照

**端點：** `GET /v2/snapshot/locale/us/markets/stocks/tickers/{stocksTicker}`

```python
snapshot = client.get_snapshot_ticker("stocks", "AAPL")
ticker = snapshot.ticker

price = ticker.last_trade.price       # 最新成交價
prev_close = ticker.prev_day.close    # 前日收盤價
change_pct = ticker.todays_change_perc
```

---

### 4.3 統一快照（跨資產類別，最多 250 個 ticker）

**端點：** `GET /v3/snapshot`

可一次查詢多種資產類別（股票、選擇權、外匯、加密貨幣）。

**Query 參數：**

| 參數 | 說明 |
|------|------|
| `ticker.any_of` | 逗號分隔，最多 250 個 ticker |
| `type` | 資產類別過濾：`stocks`、`options`、`fx`、`crypto` |
| `limit` | 每頁筆數，預設 10，最大 250 |

```python
# 直接使用 requests 呼叫（適合自行控制分頁）
import requests

API_KEY = "YOUR_API_KEY"
tickers = "AAPL,MSFT,GOOGL,NVDA,TSLA,AMZN,META,JPM,V,NFLX"

resp = requests.get(
    "https://api.massive.com/v3/snapshot",
    params={
        "ticker.any_of": tickers,
        "type": "stocks",
        "limit": 250,
        "apiKey": API_KEY,
    },
    timeout=10,
)
resp.raise_for_status()
data = resp.json()

for result in data["results"]:
    ticker = result["ticker"]
    price = result["last_trade"]["price"]
    session = result["session"]
    print(f"{ticker}: ${price:.2f}, 今日變動: {session.get('change_percent', 0):+.2f}%")
```

---

### 4.4 單日開收盤（歷史）

**端點：** `GET /v1/open-close/{ticker}/{date}`

適合取得指定日期的 OHLC 資料。

```python
import requests

def get_daily_ohlc(ticker: str, date: str, api_key: str) -> dict:
    """
    取得指定 ticker 在 date（YYYY-MM-DD）的日線資料。
    """
    resp = requests.get(
        f"https://api.massive.com/v1/open-close/{ticker}/{date}",
        params={"adjusted": "true", "apiKey": api_key},
        timeout=10,
    )
    resp.raise_for_status()
    return resp.json()

# 範例
data = get_daily_ohlc("AAPL", "2025-01-09", "YOUR_API_KEY")
print(f"AAPL 2025-01-09: 開={data['open']}, 收={data['close']}, 高={data['high']}, 低={data['low']}")
```

**回應範例：**
```json
{
  "symbol": "AAPL",
  "from": "2025-01-09",
  "status": "OK",
  "open": 185.20,
  "close": 187.65,
  "high": 188.45,
  "low": 184.80,
  "volume": 52341200,
  "preMarket": 184.50,
  "afterHours": 188.10
}
```

---

### 4.5 K 線聚合資料（歷史 OHLC）

**端點：** `GET /v2/aggs/ticker/{ticker}/range/{multiplier}/{timespan}/{from}/{to}`

適合取得 sparkline 與圖表的歷史資料。

```python
from massive import RESTClient

client = RESTClient(api_key="YOUR_API_KEY")

# 取得 AAPL 最近 30 天的日線資料
aggs = []
for bar in client.list_aggs(
    ticker="AAPL",
    multiplier=1,
    timespan="day",
    from_="2025-04-01",
    to="2025-05-01",
    limit=50000,
    adjusted=True,
):
    aggs.append({
        "timestamp": bar.timestamp,
        "open": bar.open,
        "high": bar.high,
        "low": bar.low,
        "close": bar.close,
        "volume": bar.volume,
    })

print(f"取得 {len(aggs)} 根 K 線")
```

---

### 4.6 最新成交價

**端點：** `GET /v2/last/trade/{ticker}`

```python
trade = client.get_last_trade("AAPL")
print(f"最新成交價: ${trade.results.price}")
print(f"成交量: {trade.results.size} 股")
print(f"時間: {trade.results.sip_timestamp}")
```

---

## 5. 本專案使用建議

### 5.1 輪詢策略

本專案採 REST 輪詢（非 WebSocket），利用快照端點批次取得所有被觀察 ticker 的價格：

| 方案 | 免費（5 req/min） | 付費（無限制） |
|------|-----------------|--------------|
| 輪詢間隔 | 每 **15 秒**一次（1 次/15s < 5 次/min） | 每 **2 秒**一次 |
| 每次請求 | 全部觀察清單 tickers（一個請求） | 全部觀察清單 tickers |

### 5.2 推薦呼叫方式

```python
import asyncio
import requests
from datetime import datetime

async def poll_prices(tickers: list[str], api_key: str, interval_sec: float = 15.0):
    """持續輪詢所有 tickers 的最新價格。"""
    while True:
        try:
            ticker_param = ",".join(tickers)
            resp = requests.get(
                "https://api.massive.com/v2/snapshot/locale/us/markets/stocks/tickers",
                params={"tickers": ticker_param, "apiKey": api_key},
                timeout=10,
            )
            resp.raise_for_status()
            data = resp.json()

            prices = {}
            for t in data.get("tickers", []):
                # 優先用最新成交價，其次用今日收盤/現價
                price = None
                if t.get("lastTrade"):
                    price = t["lastTrade"].get("p")
                if price is None and t.get("day"):
                    price = t["day"].get("c")
                if price:
                    prices[t["ticker"]] = {
                        "price": price,
                        "prev_close": t.get("prevDay", {}).get("c"),
                        "change_pct": t.get("todaysChangePerc", 0.0),
                        "updated_at": datetime.utcnow().isoformat(),
                    }

            yield prices

        except requests.RequestException as e:
            print(f"[Massive API] 輪詢失敗: {e}")

        await asyncio.sleep(interval_sec)
```

### 5.3 錯誤處理

```python
import requests
from requests.exceptions import HTTPError, Timeout, ConnectionError

def safe_fetch_snapshot(tickers: list[str], api_key: str) -> dict:
    """帶錯誤處理的快照取得。"""
    try:
        resp = requests.get(
            "https://api.massive.com/v2/snapshot/locale/us/markets/stocks/tickers",
            params={"tickers": ",".join(tickers), "apiKey": api_key},
            timeout=10,
        )
        resp.raise_for_status()
        return resp.json()

    except Timeout:
        raise RuntimeError("Massive API 請求超時（>10s）")
    except HTTPError as e:
        if e.response.status_code == 403:
            raise RuntimeError("API Key 無效或已達到方案限制")
        elif e.response.status_code == 429:
            raise RuntimeError("超過請求頻率限制，請降低輪詢頻率")
        raise
    except ConnectionError:
        raise RuntimeError("無法連線到 Massive API")
```

---

## 6. 方案限制摘要

| 方案 | 每分鐘請求數 | 資料延遲 | 歷史資料 |
|------|------------|---------|---------|
| 免費 | 5 次 | 15 分鐘 | 2 年 |
| Starter | 無限 | 15 分鐘 | 2 年 |
| Developer | 無限 | 15 分鐘 | 全部 |
| Advanced | 無限 | **即時** | 全部 |
| Business | 無限 | **即時** | 全部 + FMV |

> **本專案設計**：以免費方案（5 req/min）為基準，每 15 秒輪詢一次。若使用付費方案，可調低 `POLL_INTERVAL_SEC` 環境變數以提高更新頻率。

---

## 7. 參考資源

- [Massive API 文件](https://massive.com/docs)
- [REST API 快速入門](https://massive.com/docs/rest/quickstart)
- [股票 REST 概覽](https://massive.com/docs/rest/stocks/overview)
- [全市場快照端點](https://massive.com/docs/rest/stocks/snapshots/full-market-snapshot)
- [Python 客戶端 GitHub](https://github.com/massive-com/client-python)
- [方案與限制](https://massive.com/pricing)
