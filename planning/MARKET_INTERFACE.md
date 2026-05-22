# 市場資料統一介面設計

本文件定義 FinAlly 後端的市場資料抽象層。所有下游程式碼（SSE 串流、投資組合快照、API 路由）都只依賴這個介面，不需要知道資料來源是模擬器還是 Massive API。

---

## 1. 設計原則

- **單一介面，兩種實作**：`MarketDataProvider` 抽象類別定義合約，`SimulatorProvider` 與 `MassiveProvider` 各自實作。
- **工廠函式決定實作**：後端啟動時呼叫 `create_market_provider()`，依環境變數自動選擇。
- **價格快取集中管理**：Provider 不直接推送到 SSE；它們只負責更新共享的 `PriceCache`，SSE 串流從快取讀取。
- **觀察清單動態更新**：Provider 接受動態新增/移除 ticker，不需要重啟背景任務。
- **非同步優先**：所有 IO 操作使用 `async/await`，避免阻塞 FastAPI event loop。

---

## 2. 資料模型

```python
# backend/app/market/models.py

from dataclasses import dataclass, field
from datetime import datetime


@dataclass
class PriceUpdate:
    """單一 ticker 的一次價格更新。"""
    ticker: str
    price: float
    prev_price: float          # 上一次更新的價格（用於閃爍方向判斷）
    prev_close: float | None   # 前一交易日收盤價（用於計算日漲跌幅）
    change_pct: float          # 相對前日收盤的漲跌幅（%）；模擬器用累積估算
    timestamp: datetime
    direction: str             # "up" | "down" | "flat"

    @classmethod
    def from_prices(
        cls,
        ticker: str,
        price: float,
        prev_price: float,
        prev_close: float | None = None,
        timestamp: datetime | None = None,
    ) -> "PriceUpdate":
        direction = "up" if price > prev_price else ("down" if price < prev_price else "flat")
        if prev_close and prev_close > 0:
            change_pct = (price - prev_close) / prev_close * 100
        else:
            change_pct = (price - prev_price) / prev_price * 100 if prev_price > 0 else 0.0
        return cls(
            ticker=ticker,
            price=price,
            prev_price=prev_price,
            prev_close=prev_close,
            change_pct=change_pct,
            timestamp=timestamp or datetime.utcnow(),
            direction=direction,
        )


@dataclass
class PriceBar:
    """單根 K 線（用於歷史圖表與 sparkline）。"""
    timestamp: datetime
    open: float
    high: float
    low: float
    close: float
    volume: float
```

---

## 3. 抽象介面

```python
# backend/app/market/base.py

from abc import ABC, abstractmethod
from .models import PriceUpdate, PriceBar


class MarketDataProvider(ABC):
    """
    市場資料提供者的抽象介面。

    所有實作必須：
    - 在 start() 中啟動背景輪詢/模擬任務
    - 在 stop() 中乾淨地結束背景任務
    - 支援動態新增/移除 ticker
    - 透過 get_latest_prices() 回傳當前所有 ticker 的最新價格
    - 透過 get_price_history() 提供歷史 K 線（用於圖表初始化）
    """

    @abstractmethod
    async def start(self) -> None:
        """啟動背景任務（輪詢或模擬）。"""
        ...

    @abstractmethod
    async def stop(self) -> None:
        """停止背景任務，釋放資源。"""
        ...

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """將 ticker 加入觀察清單。"""
        ...

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """從觀察清單移除 ticker。"""
        ...

    @abstractmethod
    async def get_latest_prices(self) -> dict[str, PriceUpdate]:
        """
        取得所有被觀察 ticker 的最新價格快照。
        
        Returns:
            dict，key 為 ticker symbol，value 為 PriceUpdate。
        """
        ...

    @abstractmethod
    async def get_price_history(
        self,
        ticker: str,
        bars: int = 100,
    ) -> list[PriceBar]:
        """
        取得指定 ticker 的歷史 K 線，用於圖表初始化。

        Args:
            ticker: 股票代號
            bars: 要取得的 K 線根數（預設 100）

        Returns:
            PriceBar 列表，時間由舊到新排序。
        """
        ...
```

---

## 4. 共享價格快取

```python
# backend/app/market/cache.py

import asyncio
from datetime import datetime
from .models import PriceUpdate


class PriceCache:
    """
    執行緒安全的記憶體價格快取。

    Provider 寫入快取，SSE 串流讀取快取。
    這個設計將資料來源與推送邏輯解耦，未來可輕鬆支援多使用者。
    """

    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._lock = asyncio.Lock()
        self._updated_event = asyncio.Event()

    async def set(self, ticker: str, update: PriceUpdate) -> None:
        """更新單一 ticker 的價格。"""
        async with self._lock:
            self._prices[ticker] = update
        self._updated_event.set()

    async def set_many(self, updates: dict[str, PriceUpdate]) -> None:
        """批次更新多個 ticker 的價格。"""
        async with self._lock:
            self._prices.update(updates)
        self._updated_event.set()

    async def get_all(self) -> dict[str, PriceUpdate]:
        """取得所有快取的價格（快照）。"""
        async with self._lock:
            return dict(self._prices)

    async def remove(self, ticker: str) -> None:
        """從快取移除 ticker。"""
        async with self._lock:
            self._prices.pop(ticker, None)

    async def wait_for_update(self, timeout: float = 1.0) -> bool:
        """
        等待任何價格更新。SSE 串流用此方法做長輪詢。

        Returns:
            True 表示有更新，False 表示 timeout。
        """
        try:
            await asyncio.wait_for(self._updated_event.wait(), timeout=timeout)
            self._updated_event.clear()
            return True
        except asyncio.TimeoutError:
            return False
```

---

## 5. Massive API 實作

```python
# backend/app/market/massive_provider.py

import asyncio
import logging
from datetime import datetime, timedelta

import httpx

from .base import MarketDataProvider
from .cache import PriceCache
from .models import PriceUpdate, PriceBar

logger = logging.getLogger(__name__)

# 免費方案：每分鐘 5 次請求 → 每 15 秒一次
# 付費方案：可設為 2 秒
DEFAULT_POLL_INTERVAL = 15.0
BASE_URL = "https://api.massive.com"


class MassiveProvider(MarketDataProvider):
    """
    使用 Massive REST API 取得真實市場資料的實作。

    以固定間隔輪詢全市場快照端點，批次取得所有被觀察 ticker 的價格。
    """

    def __init__(
        self,
        api_key: str,
        cache: PriceCache,
        poll_interval: float = DEFAULT_POLL_INTERVAL,
    ) -> None:
        self._api_key = api_key
        self._cache = cache
        self._poll_interval = poll_interval
        self._tickers: set[str] = set()
        self._task: asyncio.Task | None = None
        self._client: httpx.AsyncClient | None = None

    async def start(self) -> None:
        """啟動 HTTP 客戶端與輪詢背景任務。"""
        self._client = httpx.AsyncClient(
            base_url=BASE_URL,
            timeout=10.0,
            headers={"Authorization": f"Bearer {self._api_key}"},
        )
        self._task = asyncio.create_task(self._poll_loop())
        logger.info(f"MassiveProvider 已啟動，輪詢間隔 {self._poll_interval}s")

    async def stop(self) -> None:
        """停止輪詢任務並關閉 HTTP 連線。"""
        if self._task:
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        if self._client:
            await self._client.aclose()
        logger.info("MassiveProvider 已停止")

    async def add_ticker(self, ticker: str) -> None:
        self._tickers.add(ticker.upper())

    async def remove_ticker(self, ticker: str) -> None:
        self._tickers.discard(ticker.upper())
        await self._cache.remove(ticker.upper())

    async def get_latest_prices(self) -> dict[str, PriceUpdate]:
        return await self._cache.get_all()

    async def get_price_history(
        self, ticker: str, bars: int = 100
    ) -> list[PriceBar]:
        """從 Massive 取得日線 K 線資料（最近 bars 根）。"""
        if not self._client:
            return []

        end = datetime.utcnow()
        start = end - timedelta(days=bars * 2)  # 多取一些以應對假日

        try:
            resp = await self._client.get(
                f"/v2/aggs/ticker/{ticker}/range/1/day"
                f"/{start.strftime('%Y-%m-%d')}/{end.strftime('%Y-%m-%d')}",
                params={"adjusted": "true", "sort": "asc", "limit": bars},
            )
            resp.raise_for_status()
            data = resp.json()

            result = []
            for bar in data.get("results", [])[-bars:]:
                result.append(PriceBar(
                    timestamp=datetime.utcfromtimestamp(bar["t"] / 1000),
                    open=bar["o"],
                    high=bar["h"],
                    low=bar["l"],
                    close=bar["c"],
                    volume=bar["v"],
                ))
            return result

        except Exception as e:
            logger.error(f"取得 {ticker} 歷史資料失敗: {e}")
            return []

    async def _poll_loop(self) -> None:
        """背景輪詢迴圈：定期呼叫快照端點更新快取。"""
        while True:
            try:
                if self._tickers:
                    await self._fetch_and_update()
            except asyncio.CancelledError:
                raise
            except Exception as e:
                logger.error(f"輪詢發生錯誤: {e}")

            await asyncio.sleep(self._poll_interval)

    async def _fetch_and_update(self) -> None:
        """呼叫快照端點並將結果寫入 PriceCache。"""
        ticker_param = ",".join(sorted(self._tickers))
        resp = await self._client.get(
            "/v2/snapshot/locale/us/markets/stocks/tickers",
            params={"tickers": ticker_param},
        )
        resp.raise_for_status()
        data = resp.json()

        updates: dict[str, PriceUpdate] = {}
        existing = await self._cache.get_all()

        for t in data.get("tickers", []):
            symbol = t["ticker"]

            # 取最新價格：優先用 lastTrade，其次用 day.c
            last_trade = t.get("lastTrade") or {}
            day = t.get("day") or {}
            price = last_trade.get("p") or day.get("c")
            if not price:
                continue

            prev_close = (t.get("prevDay") or {}).get("c")
            prev_price = existing[symbol].price if symbol in existing else price

            updates[symbol] = PriceUpdate.from_prices(
                ticker=symbol,
                price=float(price),
                prev_price=float(prev_price),
                prev_close=float(prev_close) if prev_close else None,
                timestamp=datetime.utcnow(),
            )

        if updates:
            await self._cache.set_many(updates)
            logger.debug(f"更新 {len(updates)} 個 ticker 的價格")
```

---

## 6. 工廠函式

```python
# backend/app/market/factory.py

import os
from .base import MarketDataProvider
from .cache import PriceCache
from .massive_provider import MassiveProvider
from .simulator_provider import SimulatorProvider  # 見 MARKET_SIMULATOR.md

# 預設觀察清單（與資料庫種子資料一致）
DEFAULT_TICKERS = [
    "AAPL", "GOOGL", "MSFT", "AMZN", "TSLA",
    "NVDA", "META", "JPM", "V", "NFLX",
]


def create_market_provider(cache: PriceCache) -> MarketDataProvider:
    """
    依環境變數決定使用 Massive API 或模擬器。

    - MASSIVE_API_KEY 已設定且非空 → MassiveProvider
    - 其他情況 → SimulatorProvider

    Returns:
        已設定初始 tickers 但尚未 start() 的 MarketDataProvider 實例。
        呼叫方需在 FastAPI lifespan 中負責呼叫 start() 與 stop()。
    """
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()

    if api_key:
        poll_interval = float(os.environ.get("POLL_INTERVAL_SEC", "15.0"))
        provider = MassiveProvider(
            api_key=api_key,
            cache=cache,
            poll_interval=poll_interval,
        )
        provider_name = "MassiveProvider"
    else:
        provider = SimulatorProvider(cache=cache)
        provider_name = "SimulatorProvider"

    # 非同步 add_ticker 在工廠中以同步方式初始化
    # 實際的非同步初始化在 lifespan 的 start() 呼叫中完成
    provider._tickers = set(DEFAULT_TICKERS)

    import logging
    logging.getLogger(__name__).info(
        f"市場資料提供者: {provider_name}，初始 tickers: {sorted(DEFAULT_TICKERS)}"
    )
    return provider
```

---

## 7. FastAPI 整合

```python
# backend/app/main.py（相關片段）

from contextlib import asynccontextmanager
from fastapi import FastAPI
from .market.cache import PriceCache
from .market.factory import create_market_provider

price_cache = PriceCache()
market_provider = create_market_provider(price_cache)


@asynccontextmanager
async def lifespan(app: FastAPI):
    """管理市場資料提供者的生命週期。"""
    await market_provider.start()
    yield
    await market_provider.stop()


app = FastAPI(lifespan=lifespan)
```

---

## 8. SSE 串流端點整合

```python
# backend/app/routers/stream.py（相關片段）

import json
import asyncio
from fastapi import APIRouter
from fastapi.responses import StreamingResponse
from ..market.cache import PriceCache

router = APIRouter()


async def price_event_generator(cache: PriceCache):
    """
    SSE 事件產生器：每次快取更新（或最多每 500ms）推送一次。
    """
    while True:
        await cache.wait_for_update(timeout=0.5)
        prices = await cache.get_all()

        if prices:
            payload = {
                ticker: {
                    "ticker": update.ticker,
                    "price": update.price,
                    "prev_price": update.prev_price,
                    "change_pct": update.change_pct,
                    "direction": update.direction,
                    "timestamp": update.timestamp.isoformat(),
                }
                for ticker, update in prices.items()
            }
            yield f"data: {json.dumps(payload)}\n\n"


@router.get("/api/stream/prices")
async def stream_prices(cache: PriceCache):
    return StreamingResponse(
        price_event_generator(cache),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",
        },
    )
```

---

## 9. 目錄結構

```
backend/app/market/
├── __init__.py
├── base.py              # MarketDataProvider 抽象類別
├── cache.py             # PriceCache
├── factory.py           # create_market_provider()
├── massive_provider.py  # MassiveProvider 實作
├── models.py            # PriceUpdate, PriceBar
└── simulator_provider.py # SimulatorProvider 實作（見 MARKET_SIMULATOR.md）
```

---

## 10. 環境變數參考

| 變數名稱 | 說明 | 預設值 |
|---------|------|--------|
| `MASSIVE_API_KEY` | Massive API Key；設定後使用真實資料 | （空，使用模擬器）|
| `POLL_INTERVAL_SEC` | Massive 輪詢間隔（秒） | `15.0` |
