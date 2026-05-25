# FinAlly 市場資料後端設計文件

本文件是市場資料層的**完整實作藍圖**，整合並細化了 `MARKET_INTERFACE.md`、`MARKET_SIMULATOR.md`、`MASSIVE_API.md` 的內容。後端工程師應以本文件為主要參考，其餘三份文件為補充背景。

---

## 目錄

1. [設計總覽](#1-設計總覽)
2. [目錄結構](#2-目錄結構)
3. [資料模型](#3-資料模型)
4. [抽象介面](#4-抽象介面)
5. [共享價格快取](#5-共享價格快取)
6. [市場模擬器](#6-市場模擬器)
7. [Massive API 實作](#7-massive-api-實作)
8. [工廠函式](#8-工廠函式)
9. [FastAPI 整合](#9-fastapi-整合)
10. [SSE 串流端點](#10-sse-串流端點)
11. [歷史資料端點](#11-歷史資料端點)
12. [觀察清單動態同步](#12-觀察清單動態同步)
13. [測試策略](#13-測試策略)
14. [設定參數參考](#14-設定參數參考)

---

## 1. 設計總覽

### 1.1 系統架構圖

```
┌─────────────────────────────────────────────────────────────────┐
│  FastAPI Application                                            │
│                                                                 │
│  ┌──────────────────┐     ┌──────────────────────────────────┐  │
│  │  MarketDataProvider  │     │  PriceCache (memory)          │  │
│  │  (抽象介面)       │────▶│  dict[ticker → PriceUpdate]     │  │
│  │                  │     │  asyncio.Lock + Event            │  │
│  │  ┌────────────┐  │     └──────────────┬───────────────────┘  │
│  │  │ Simulator  │  │                    │                       │
│  │  │ Provider   │  │                    ▼                       │
│  │  └────────────┘  │     ┌──────────────────────────────────┐  │
│  │  ┌────────────┐  │     │  SSE Stream  /api/stream/prices  │  │
│  │  │  Massive   │  │     │  EventSource → Browser           │  │
│  │  │  Provider  │  │     └──────────────────────────────────┘  │
│  │  └────────────┘  │                                           │
│  └──────────────────┘                                           │
│                                                                 │
│  create_market_provider() ← MASSIVE_API_KEY env var            │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 設計原則

| 原則 | 說明 |
|------|------|
| **單一介面、兩種實作** | `SimulatorProvider` 與 `MassiveProvider` 都實作 `MarketDataProvider`，下游程式碼無需知道資料來源 |
| **快取解耦** | Provider 只寫 `PriceCache`；SSE 串流只讀 `PriceCache`，兩者完全解耦 |
| **動態觀察清單** | `add_ticker()` / `remove_ticker()` 立即生效，不重啟背景任務 |
| **非同步優先** | 所有 IO 使用 `async/await`，不阻塞 FastAPI event loop |
| **容錯設計** | Provider 內部錯誤不應讓整個應用崩潰；lifespan 需有例外處理 |

---

## 2. 目錄結構

```
backend/
├── app/
│   ├── main.py                    # FastAPI 應用入口、lifespan、路由掛載
│   ├── market/
│   │   ├── __init__.py
│   │   ├── models.py              # PriceUpdate, PriceBar, TickerConfig
│   │   ├── base.py                # MarketDataProvider 抽象類別
│   │   ├── cache.py               # PriceCache
│   │   ├── simulator_provider.py  # SimulatorProvider（GBM 模擬器）
│   │   ├── massive_provider.py    # MassiveProvider（Massive REST API）
│   │   └── factory.py             # create_market_provider()
│   └── routers/
│       ├── stream.py              # GET /api/stream/prices (SSE)
│       ├── market.py              # GET /api/prices/history/{ticker}
│       └── watchlist.py           # GET/POST/DELETE /api/watchlist
├── tests/
│   ├── test_simulator.py
│   ├── test_massive_provider.py
│   ├── test_cache.py
│   └── test_stream.py
└── pyproject.toml                 # uv 專案設定
```

---

## 3. 資料模型

```python
# backend/app/market/models.py

from __future__ import annotations

import math
from dataclasses import dataclass, field
from datetime import datetime


@dataclass
class PriceUpdate:
    """
    單一 ticker 的一次價格更新事件。

    這是流經整個市場資料層的核心資料結構：
    - Provider 產生 PriceUpdate 並寫入 PriceCache
    - SSE 串流從 PriceCache 讀取並序列化成 JSON 推送給瀏覽器
    - 前端用 direction 決定閃爍顏色（綠/紅）
    """

    ticker: str
    price: float          # 最新價格（2 位小數）
    prev_price: float     # 上次更新的價格（用於 direction 判斷）
    prev_close: float | None  # 前一交易日（或模擬開盤）收盤價，用於計算 change_pct
    change_pct: float     # 相對 prev_close 的漲跌幅（%）；若 prev_close 為 None 則用 prev_price
    timestamp: datetime   # UTC 時間戳
    direction: str        # "up" | "down" | "flat"

    @classmethod
    def from_prices(
        cls,
        ticker: str,
        price: float,
        prev_price: float,
        prev_close: float | None = None,
        timestamp: datetime | None = None,
    ) -> PriceUpdate:
        """
        工廠方法：自動計算 direction 與 change_pct。

        change_pct 計算優先順序：
        1. 有 prev_close → (price - prev_close) / prev_close * 100
        2. 有 prev_price 且非零 → (price - prev_price) / prev_price * 100
        3. 其他 → 0.0
        """
        if price > prev_price:
            direction = "up"
        elif price < prev_price:
            direction = "down"
        else:
            direction = "flat"

        if prev_close and prev_close > 0:
            change_pct = (price - prev_close) / prev_close * 100
        elif prev_price > 0:
            change_pct = (price - prev_price) / prev_price * 100
        else:
            change_pct = 0.0

        return cls(
            ticker=ticker,
            price=price,
            prev_price=prev_price,
            prev_close=prev_close,
            change_pct=round(change_pct, 4),
            timestamp=timestamp or datetime.utcnow(),
            direction=direction,
        )

    def to_dict(self) -> dict:
        """序列化為可直接 JSON 化的 dict（供 SSE 使用）。"""
        return {
            "ticker": self.ticker,
            "price": self.price,
            "prev_price": self.prev_price,
            "prev_close": self.prev_close,
            "change_pct": self.change_pct,
            "direction": self.direction,
            "timestamp": self.timestamp.isoformat() + "Z",
        }


@dataclass
class PriceBar:
    """
    單根 K 線資料（OHLCV）。

    用途：
    - GET /api/prices/history/{ticker} 的回傳值
    - 前端主圖表的初始歷史資料
    - sparkline 初始資料（前端只用 close）
    """

    timestamp: datetime
    open: float
    high: float
    low: float
    close: float
    volume: float

    def to_dict(self) -> dict:
        return {
            "timestamp": self.timestamp.isoformat() + "Z",
            "open": self.open,
            "high": self.high,
            "low": self.low,
            "close": self.close,
            "volume": self.volume,
        }


@dataclass
class TickerConfig:
    """
    模擬器用的單一 ticker 參數。

    不用於 MassiveProvider（真實資料不需要模擬參數）。
    """

    ticker: str
    seed_price: float    # 起始參考價格（接近真實市場水準）
    mu: float            # 年化漂移率（期望報酬）；建議 0.05–0.12
    sigma: float         # 年化波動率；建議 0.15–0.55
    sector: str          # 板塊（用於相關性分組）


# ──────────────────────────────────────
# 預設 10 個 ticker 設定
# ──────────────────────────────────────
DEFAULT_TICKER_CONFIGS: dict[str, TickerConfig] = {
    "AAPL":  TickerConfig("AAPL",  seed_price=190.0,  mu=0.08, sigma=0.25, sector="tech"),
    "GOOGL": TickerConfig("GOOGL", seed_price=175.0,  mu=0.07, sigma=0.27, sector="tech"),
    "MSFT":  TickerConfig("MSFT",  seed_price=415.0,  mu=0.09, sigma=0.23, sector="tech"),
    "AMZN":  TickerConfig("AMZN",  seed_price=195.0,  mu=0.10, sigma=0.28, sector="tech"),
    "TSLA":  TickerConfig("TSLA",  seed_price=245.0,  mu=0.05, sigma=0.55, sector="ev"),
    "NVDA":  TickerConfig("NVDA",  seed_price=875.0,  mu=0.12, sigma=0.50, sector="tech"),
    "META":  TickerConfig("META",  seed_price=510.0,  mu=0.09, sigma=0.32, sector="tech"),
    "JPM":   TickerConfig("JPM",   seed_price=205.0,  mu=0.06, sigma=0.20, sector="finance"),
    "V":     TickerConfig("V",     seed_price=280.0,  mu=0.07, sigma=0.18, sector="finance"),
    "NFLX":  TickerConfig("NFLX",  seed_price=680.0,  mu=0.08, sigma=0.38, sector="media"),
}
```

---

## 4. 抽象介面

```python
# backend/app/market/base.py

from abc import ABC, abstractmethod
from .models import PriceUpdate, PriceBar


class MarketDataProvider(ABC):
    """
    市場資料提供者的抽象介面。

    合約要求：
    - start() 啟動背景任務（GBM 模擬迴圈或 REST 輪詢迴圈）
    - stop() 必須能被多次呼叫而不拋出例外（idempotent）
    - add_ticker() / remove_ticker() 即時生效，不需重啟背景任務
    - get_latest_prices() 直接從 PriceCache 讀取，不觸發網路請求
    - get_price_history() 可能觸發網路請求（Massive）或 CPU 計算（Simulator）

    生命週期：
        provider = create_market_provider(cache)
        await provider.start()     # FastAPI lifespan 啟動時
        ...
        await provider.stop()      # FastAPI lifespan 關閉時
    """

    @abstractmethod
    async def start(self) -> None:
        """
        啟動背景任務。

        實作者必須：
        - 建立並儲存 asyncio.Task
        - 在任務中捕捉所有非 CancelledError 例外（避免任務靜默失敗）
        """
        ...

    @abstractmethod
    async def stop(self) -> None:
        """
        停止背景任務，釋放資源。

        實作者必須：
        - cancel() 任務後 await 它（捕捉 CancelledError）
        - 關閉 HTTP 連線（如有）
        - 可被多次呼叫而不拋出例外
        """
        ...

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """
        將 ticker 加入觀察清單（全大寫），立即生效。
        若 ticker 已存在，不做任何事（idempotent）。
        """
        ...

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """
        從觀察清單移除 ticker，同時從 PriceCache 刪除。
        若 ticker 不存在，不做任何事（idempotent）。
        """
        ...

    @abstractmethod
    async def get_latest_prices(self) -> dict[str, PriceUpdate]:
        """
        取得目前所有被觀察 ticker 的最新價格快照。
        直接代理 PriceCache.get_all()，不做額外 IO。
        """
        ...

    @abstractmethod
    async def get_price_history(
        self,
        ticker: str,
        bars: int = 100,
    ) -> list[PriceBar]:
        """
        取得指定 ticker 的歷史 K 線（由舊到新）。

        Args:
            ticker: 股票代號（大小寫不敏感，內部會轉大寫）
            bars:   要取得的 K 線根數

        Returns:
            PriceBar 列表，時間升冪排序（index 0 = 最舊）。
            若取得失敗，回傳空列表（不拋例外）。
        """
        ...
```

---

## 5. 共享價格快取

```python
# backend/app/market/cache.py

import asyncio
from datetime import datetime
from .models import PriceUpdate


class PriceCache:
    """
    執行緒安全的記憶體價格快取。

    並發安全性說明：
    - Python 的 asyncio 是單執行緒事件迴圈（GIL）
    - 所有操作都在同一個執行緒中以協程方式執行
    - asyncio.Lock 防止兩個協程同時修改 _prices（例如 set_many + get_all 競爭）
    - asyncio.Event 作為輕量的訂閱/通知機制：Provider 寫入後 set()，
      SSE 的 wait_for_update() 被喚醒，讀取快照後 clear()

    注意：由於 asyncio 單執行緒性質，asyncio.Lock 在技術上不是必需的，
    但保留它是為了：
    1. 程式碼的意圖明確（批次操作應視為原子操作）
    2. 若未來切換到多執行緒模型（如 threadpool executor），有正確的保護
    """

    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._lock = asyncio.Lock()
        self._updated_event = asyncio.Event()

    async def set(self, ticker: str, update: PriceUpdate) -> None:
        """更新單一 ticker 的價格，並通知等待的消費者。"""
        async with self._lock:
            self._prices[ticker] = update
        self._updated_event.set()

    async def set_many(self, updates: dict[str, PriceUpdate]) -> None:
        """
        批次更新多個 ticker（原子操作）。

        模擬器和 Massive Provider 的主要寫入路徑——每個 tick 一次批次更新。
        """
        if not updates:
            return
        async with self._lock:
            self._prices.update(updates)
        self._updated_event.set()

    async def get_all(self) -> dict[str, PriceUpdate]:
        """取得所有快取價格的淺拷貝（防止呼叫方修改快取）。"""
        async with self._lock:
            return dict(self._prices)

    async def get(self, ticker: str) -> PriceUpdate | None:
        """取得單一 ticker 的最新價格，不存在回傳 None。"""
        async with self._lock:
            return self._prices.get(ticker)

    async def remove(self, ticker: str) -> None:
        """
        從快取移除 ticker（觀察清單移除時呼叫）。
        前端收到 SSE 的 ticker_removed 事件後會清空 sparkline；
        快取移除確保下一次 get_all() 不再包含此 ticker。
        """
        async with self._lock:
            self._prices.pop(ticker, None)
        # 通知 SSE：有更新（即使是移除也需通知，供 watchlist_removed event 使用）
        self._updated_event.set()

    async def wait_for_update(self, timeout: float = 0.5) -> bool:
        """
        等待任何價格更新。SSE 事件產生器用此方法做高效長輪詢。

        Returns:
            True  → 有更新（含新價格或 ticker 移除）
            False → 超時（應照常推送心跳或當前快照）

        使用方式：
            while True:
                updated = await cache.wait_for_update(timeout=0.5)
                prices = await cache.get_all()
                yield sse_event(prices)
        """
        try:
            await asyncio.wait_for(self._updated_event.wait(), timeout=timeout)
            self._updated_event.clear()
            return True
        except asyncio.TimeoutError:
            return False

    @property
    def ticker_count(self) -> int:
        """目前快取中的 ticker 數量（無需 lock，僅供監控/日誌使用）。"""
        return len(self._prices)
```

---

## 6. 市場模擬器

### 6.1 幾何布朗運動（GBM）數學

模擬器使用**離散化 GBM**產生每個 tick 的價格：

```
S(t + Δt) = S(t) × exp( (μ - σ²/2) × Δt  +  σ × √Δt × Z )
```

| 符號 | 含義 | 數值範例 |
|------|------|---------|
| `S(t)` | 當前價格 | 190.00 |
| `μ` (mu) | 年化漂移率 | 0.08（8% / 年）|
| `σ` (sigma) | 年化波動率 | 0.25（25% / 年）|
| `Δt` | 時間步長（年）| 500ms ÷ (252天 × 6.5h × 3600s × 1000ms) ≈ 8.48×10⁻⁸ 年 |
| `Z` | 標準常態分布亂數 N(0,1) | 由 `random.gauss(0, 1)` 產生 |

**Δt 計算**：

```python
UPDATE_INTERVAL_MS = 500
TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600   # ≈ 5,896,800 秒/年
DT = (UPDATE_INTERVAL_MS / 1000) / TRADING_SECONDS_PER_YEAR
# DT ≈ 8.48e-8 年（極短時間步長，確保每次變動幅度極小，模擬合理）
```

### 6.2 板塊相關性模型

真實市場的同類股會同向波動。模擬器使用**因子模型**重現此效果：

```
Z_final = ρ × Z_sector  +  √(1 - ρ²) × Z_individual
```

**板塊相關係數（ρ）**：

| 板塊 | ρ | 包含的 ticker | 說明 |
|------|---|--------------|------|
| `tech` | 0.6 | AAPL, GOOGL, MSFT, AMZN, NVDA, META | 科技股強聯動 |
| `finance` | 0.5 | JPM, V | 金融股中度聯動 |
| `ev` | 0.4 | TSLA | 電動車較獨立 |
| `media` | 0.4 | NFLX | 媒體較獨立 |

**10×10 相關矩陣（近似值，供參考）**：

```
         AAPL GOOGL MSFT AMZN TSLA NVDA META  JPM    V  NFLX
AAPL  [  1.00  0.36  0.36  0.36  0.16  0.36  0.36  0.00  0.00  0.00 ]
GOOGL [  0.36  1.00  0.36  0.36  0.16  0.36  0.36  0.00  0.00  0.00 ]
MSFT  [  0.36  0.36  1.00  0.36  0.16  0.36  0.36  0.00  0.00  0.00 ]
AMZN  [  0.36  0.36  0.36  1.00  0.16  0.36  0.36  0.00  0.00  0.00 ]
TSLA  [  0.16  0.16  0.16  0.16  1.00  0.16  0.16  0.00  0.00  0.00 ]
NVDA  [  0.36  0.36  0.36  0.36  0.16  1.00  0.36  0.00  0.00  0.00 ]
META  [  0.36  0.36  0.36  0.36  0.16  0.36  1.00  0.00  0.00  0.00 ]
JPM   [  0.00  0.00  0.00  0.00  0.00  0.00  0.00  1.00  0.25  0.00 ]
V     [  0.00  0.00  0.00  0.00  0.00  0.00  0.00  0.25  1.00  0.00 ]
NFLX  [  0.00  0.00  0.00  0.00  0.00  0.00  0.00  0.00  0.00  1.00 ]
```

> 相關係數推導：同板塊兩支 ticker 都含有 ρ 的板塊因子，相關性 ≈ ρ²（tech: 0.6² = 0.36, finance: 0.5² = 0.25, ev/media: 0.4² = 0.16）。

### 6.3 隨機跳動事件

```
每次 tick，每個 ticker 有 0.5% 機率觸發跳動事件：
  跳動幅度 = ±2% 至 ±5%（均勻分布），方向各 50%
```

跳動事件模擬財報公告、重大新聞等突發資訊的影響，增加視覺戲劇性。

### 6.4 確定性歷史回溯演算法

`get_price_history()` 使用**確定性回溯**，確保相同 ticker 在頁面重新載入後圖表歷史一致：

```
1. seed = hash(ticker) % 2^32（確定性種子）
2. rng = random.Random(seed)（獨立的亂數產生器）
3. 以目前模擬價格為「今日收盤」
4. 往前推 N 天，每天套用 GBM（Δt = 1/252 年）
5. prices[] 由「過去」堆到「今日」（最後 reverse）
6. 每根 K 線的 open/high/low 由 close 加小幅隨機偏移產生
```

**為什麼用 hash(ticker) 作種子**：
- 同一個 ticker 每次都產生完全相同的歷史
- 不同 ticker 的歷史走勢不同（避免所有股票圖形一模一樣）
- 即使 Docker 容器重啟，歷史圖表不會突然改變（UX 一致性）

### 6.5 完整程式碼

```python
# backend/app/market/simulator_provider.py

import asyncio
import logging
import math
import random
from datetime import datetime, timedelta

from .base import MarketDataProvider
from .cache import PriceCache
from .models import DEFAULT_TICKER_CONFIGS, PriceBar, PriceUpdate, TickerConfig

logger = logging.getLogger(__name__)

# ──────────────────────────────────────
# 常數
# ──────────────────────────────────────

UPDATE_INTERVAL_MS = 500
TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600
DT = (UPDATE_INTERVAL_MS / 1000) / TRADING_SECONDS_PER_YEAR

SECTOR_CORRELATION: dict[str, float] = {
    "tech": 0.6,
    "finance": 0.5,
    "ev": 0.4,
    "media": 0.4,
}

JUMP_PROBABILITY = 0.005       # 每 tick 觸發機率：0.5%
JUMP_MIN_PCT = 0.02            # 跳動最小幅度：2%
JUMP_MAX_PCT = 0.05            # 跳動最大幅度：5%

# 未知 ticker 的預設設定（使用者手動新增的不在清單中的股票）
DEFAULT_UNKNOWN_CONFIG = TickerConfig(
    ticker="UNKNOWN",
    seed_price=100.0,
    mu=0.07,
    sigma=0.30,
    sector="tech",
)


class SimulatorProvider(MarketDataProvider):
    """
    使用幾何布朗運動（GBM）模擬股票價格的市場資料提供者。

    每 500ms 更新一次所有被觀察 ticker 的價格，並將結果批次寫入 PriceCache。
    """

    def __init__(self, cache: PriceCache) -> None:
        self._cache = cache
        self._tickers: set[str] = set()
        self._current_prices: dict[str, float] = {}
        self._day_open_prices: dict[str, float] = {}  # 用於計算 change_pct
        self._task: asyncio.Task | None = None

    # ──────────────────────────────────────
    # MarketDataProvider 介面實作
    # ──────────────────────────────────────

    async def start(self) -> None:
        """初始化種子價格並啟動模擬背景任務。"""
        self._initialize_prices()
        self._task = asyncio.create_task(self._simulation_loop(), name="simulator_loop")
        logger.info(
            f"SimulatorProvider 已啟動，監控 {len(self._tickers)} 個 ticker: "
            f"{sorted(self._tickers)}"
        )

    async def stop(self) -> None:
        """停止模擬任務（idempotent）。"""
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
            self._task = None
        logger.info("SimulatorProvider 已停止")

    async def add_ticker(self, ticker: str) -> None:
        """新增 ticker。若尚無種子價格，以設定的 seed_price 初始化。"""
        ticker = ticker.upper()
        self._tickers.add(ticker)
        if ticker not in self._current_prices:
            config = self._get_config(ticker)
            self._current_prices[ticker] = config.seed_price
            self._day_open_prices[ticker] = config.seed_price
            logger.debug(f"新增 ticker {ticker}，起始價格 {config.seed_price}")

    async def remove_ticker(self, ticker: str) -> None:
        """移除 ticker，同時從快取刪除。"""
        ticker = ticker.upper()
        self._tickers.discard(ticker)
        self._current_prices.pop(ticker, None)
        self._day_open_prices.pop(ticker, None)
        await self._cache.remove(ticker)
        logger.debug(f"移除 ticker {ticker}")

    async def get_latest_prices(self) -> dict[str, PriceUpdate]:
        return await self._cache.get_all()

    async def get_price_history(
        self, ticker: str, bars: int = 100
    ) -> list[PriceBar]:
        """
        確定性歷史回溯：以 hash(ticker) 作種子，確保可重現。

        回溯流程：
        1. 取目前模擬價格（或種子價格）作為「今日收盤」
        2. 用固定亂數種子往前推 bars 根日線（Δt = 1/252）
        3. 根據 close 估算 open/high/low/volume
        4. 回傳由舊到新的 K 線列表
        """
        ticker = ticker.upper()
        config = self._get_config(ticker)
        current = self._current_prices.get(ticker, config.seed_price)

        # 確定性亂數產生器（同 ticker 每次相同）
        rng = random.Random(hash(ticker) % (2**32))

        # 步驟 1：向前回溯 bars 個價格點（GBM 逆推）
        dt_day = 1.0 / 252  # 一個交易日
        prices = [current]
        for _ in range(bars - 1):
            z = rng.gauss(0, 1)
            drift = (config.mu - 0.5 * config.sigma ** 2) * dt_day
            diffusion = config.sigma * math.sqrt(dt_day) * z
            # 逆推：從「今日」往前推（除以 exp(drift+diffusion)）
            prices.append(prices[-1] / math.exp(drift + diffusion))

        prices.reverse()  # 現在從最舊到最新

        # 步驟 2：產生 OHLCV K 線
        result: list[PriceBar] = []
        base_time = datetime.utcnow() - timedelta(days=bars)
        day_sigma = config.sigma * math.sqrt(dt_day)  # 日波動率

        for i, close in enumerate(prices):
            close = round(max(close, 0.01), 2)

            # open 接近 close（小幅偏移）
            open_ = round(
                max(close * (1 + rng.gauss(0, day_sigma * 0.3)), 0.01), 2
            )
            # high 在 open/close 最大值之上
            high = round(
                max(open_, close) * (1 + abs(rng.gauss(0, day_sigma * 0.4))), 2
            )
            # low 在 open/close 最小值之下
            low = round(
                min(open_, close) * (1 - abs(rng.gauss(0, day_sigma * 0.4))), 2
            )
            # 確保 high >= max(open, close) 且 low <= min(open, close)
            high = max(high, open_, close)
            low = min(low, open_, close)

            volume = round(abs(rng.gauss(10_000_000, 3_000_000)))

            result.append(PriceBar(
                timestamp=base_time + timedelta(days=i),
                open=open_,
                high=high,
                low=low,
                close=close,
                volume=volume,
            ))

        return result

    # ──────────────────────────────────────
    # 內部實作
    # ──────────────────────────────────────

    def _initialize_prices(self) -> None:
        """以種子價格初始化所有被觀察 ticker（已存在的不覆蓋）。"""
        for ticker in self._tickers:
            if ticker not in self._current_prices:
                config = self._get_config(ticker)
                self._current_prices[ticker] = config.seed_price
                self._day_open_prices[ticker] = config.seed_price

    async def _simulation_loop(self) -> None:
        """
        主模擬迴圈。

        內部例外會被記錄但不終止迴圈（避免單次 tick 錯誤讓整個模擬停止）。
        """
        while True:
            try:
                await self._tick()
            except asyncio.CancelledError:
                raise  # CancelledError 必須重新拋出（任務取消機制）
            except Exception as e:
                logger.error(f"模擬器 tick 錯誤（已跳過）: {e}", exc_info=True)

            await asyncio.sleep(UPDATE_INTERVAL_MS / 1000)

    async def _tick(self) -> None:
        """
        執行一次 GBM 步進，更新所有 ticker 的價格。

        完整流程：
        1. 對每個有效板塊產生共同隨機因子 Z_sector
        2. 對每個 ticker 混合板塊與個別因子得到 Z_final
        3. 套用 GBM 公式：S_new = S_old × exp(drift + diffusion × Z_final)
        4. 以 JUMP_PROBABILITY 機率觸發跳動事件（±2%–5%）
        5. 批次寫入 PriceCache
        """
        if not self._tickers:
            return

        # 步驟 1：各板塊共同隨機因子（每次 tick 重新抽）
        sectors_in_use = {
            self._get_config(t).sector for t in self._tickers
        }
        sector_factors: dict[str, float] = {
            sector: random.gauss(0, 1) for sector in sectors_in_use
        }

        updates: dict[str, PriceUpdate] = {}

        for ticker in list(self._tickers):
            config = self._get_config(ticker)
            prev_price = self._current_prices.get(ticker, config.seed_price)

            # 步驟 2：混合板塊與個別因子
            rho = SECTOR_CORRELATION.get(config.sector, 0.4)
            z_sector = sector_factors.get(config.sector, random.gauss(0, 1))
            z_individual = random.gauss(0, 1)
            z_final = rho * z_sector + math.sqrt(1 - rho ** 2) * z_individual

            # 步驟 3：GBM 公式
            drift = (config.mu - 0.5 * config.sigma ** 2) * DT
            diffusion = config.sigma * math.sqrt(DT) * z_final
            new_price = prev_price * math.exp(drift + diffusion)

            # 步驟 4：隨機跳動事件
            if random.random() < JUMP_PROBABILITY:
                jump_pct = random.uniform(JUMP_MIN_PCT, JUMP_MAX_PCT)
                jump_direction = 1 if random.random() > 0.5 else -1
                new_price *= 1 + jump_direction * jump_pct
                logger.info(
                    f"[跳動事件] {ticker}: {prev_price:.2f} → {new_price:.2f} "
                    f"({'+' if jump_direction > 0 else ''}{jump_direction * jump_pct * 100:.1f}%)"
                )

            new_price = round(max(new_price, 0.01), 2)
            self._current_prices[ticker] = new_price

            updates[ticker] = PriceUpdate.from_prices(
                ticker=ticker,
                price=new_price,
                prev_price=prev_price,
                prev_close=self._day_open_prices.get(ticker),
                timestamp=datetime.utcnow(),
            )

        # 步驟 5：批次寫入快取
        await self._cache.set_many(updates)

    def _get_config(self, ticker: str) -> TickerConfig:
        """
        取得 ticker 設定。未知 ticker 使用通用預設值（不拋例外）。
        """
        if ticker in DEFAULT_TICKER_CONFIGS:
            return DEFAULT_TICKER_CONFIGS[ticker]
        # 用 ticker 名稱建立個性化預設（不同 ticker 有略微不同的 seed_price）
        seed = hash(ticker) % 1000 + 50  # 50–1049 美元
        return TickerConfig(
            ticker=ticker,
            seed_price=float(seed),
            mu=DEFAULT_UNKNOWN_CONFIG.mu,
            sigma=DEFAULT_UNKNOWN_CONFIG.sigma,
            sector=DEFAULT_UNKNOWN_CONFIG.sector,
        )
```

---

## 7. Massive API 實作

### 7.1 API 端點概覽

本專案只用到兩個 Massive 端點：

| 端點 | 用途 | 呼叫時機 |
|------|------|---------|
| `GET /v2/snapshot/locale/us/markets/stocks/tickers` | 批次取得多個 ticker 的即時快照 | 每 15s（免費）/每 2s（付費）輪詢 |
| `GET /v2/aggs/ticker/{ticker}/range/1/day/{from}/{to}` | 取得歷史日線 K 線 | `GET /api/prices/history/{ticker}` 被呼叫時 |

### 7.2 速率限制策略

| 方案 | req/min | 建議輪詢間隔 | 資料延遲 |
|------|---------|------------|---------|
| 免費（預設）| 5 | **15 秒** | 15 分鐘（延遲） |
| 付費 Starter | 無限 | 5 秒 | 15 分鐘 |
| Advanced/Business | 無限 | **2 秒** | **即時** |

> **免費方案的限制**：資料有 15 分鐘延遲，且免費方案不保證即時。對本課程示範而言，延遲資料仍能展示 SSE 串流的架構，只是不反映真實市場現況。若要真實即時資料，需 Advanced 以上方案。

### 7.3 完整程式碼

```python
# backend/app/market/massive_provider.py

import asyncio
import logging
from datetime import datetime, timedelta

import httpx

from .base import MarketDataProvider
from .cache import PriceCache
from .models import PriceBar, PriceUpdate

logger = logging.getLogger(__name__)

BASE_URL = "https://api.massive.com"
DEFAULT_POLL_INTERVAL = 15.0   # 免費方案：5 req/min → 15 秒間隔


class MassiveProvider(MarketDataProvider):
    """
    使用 Massive REST API 取得真實市場資料的實作。

    採用輪詢（非 WebSocket）策略：
    - 批次取得所有觀察 ticker 的快照（一次請求）
    - 失敗時記錄錯誤並等待下一個輪詢週期（不崩潰）
    - 支援動態新增/移除 ticker（下次輪詢時生效）
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

    # ──────────────────────────────────────
    # MarketDataProvider 介面實作
    # ──────────────────────────────────────

    async def start(self) -> None:
        """啟動 httpx 非同步客戶端與輪詢背景任務。"""
        self._client = httpx.AsyncClient(
            base_url=BASE_URL,
            timeout=10.0,
            headers={"Authorization": f"Bearer {self._api_key}"},
            # 自動重試 503/504（伺服器暫時不可用）
            transport=httpx.AsyncHTTPTransport(retries=2),
        )
        self._task = asyncio.create_task(self._poll_loop(), name="massive_poll_loop")
        logger.info(
            f"MassiveProvider 已啟動，輪詢間隔 {self._poll_interval}s，"
            f"監控 {len(self._tickers)} 個 ticker"
        )

    async def stop(self) -> None:
        """停止輪詢任務並關閉 HTTP 連線（idempotent）。"""
        if self._task and not self._task.done():
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
            self._task = None

        if self._client:
            await self._client.aclose()
            self._client = None

        logger.info("MassiveProvider 已停止")

    async def add_ticker(self, ticker: str) -> None:
        """新增 ticker（下次輪詢時自動包含在請求中）。"""
        self._tickers.add(ticker.upper())

    async def remove_ticker(self, ticker: str) -> None:
        """移除 ticker，同時從快取刪除。"""
        ticker = ticker.upper()
        self._tickers.discard(ticker)
        await self._cache.remove(ticker)

    async def get_latest_prices(self) -> dict[str, PriceUpdate]:
        return await self._cache.get_all()

    async def get_price_history(
        self, ticker: str, bars: int = 100
    ) -> list[PriceBar]:
        """
        從 Massive 取得日線 K 線。

        策略：
        - 請求 bars × 2 天的範圍（應對假日、無交易日）
        - 取最後 bars 根（最近的）
        - 失敗回傳空列表（不拋例外，前端應優雅處理）
        """
        if not self._client:
            logger.warning(f"get_price_history({ticker}) 呼叫時 client 尚未初始化")
            return []

        ticker = ticker.upper()
        end = datetime.utcnow()
        start = end - timedelta(days=bars * 2)

        try:
            resp = await self._client.get(
                f"/v2/aggs/ticker/{ticker}/range/1/day"
                f"/{start.strftime('%Y-%m-%d')}/{end.strftime('%Y-%m-%d')}",
                params={
                    "adjusted": "true",
                    "sort": "asc",
                    "limit": bars,
                },
            )
            resp.raise_for_status()
            data = resp.json()

            results = data.get("results", [])
            if not results:
                logger.warning(f"Massive 回傳 {ticker} 的歷史資料為空")
                return []

            bars_data: list[PriceBar] = []
            for bar in results[-bars:]:
                try:
                    bars_data.append(PriceBar(
                        timestamp=datetime.utcfromtimestamp(bar["t"] / 1000),
                        open=float(bar["o"]),
                        high=float(bar["h"]),
                        low=float(bar["l"]),
                        close=float(bar["c"]),
                        volume=float(bar.get("v", 0)),
                    ))
                except (KeyError, ValueError, TypeError) as e:
                    logger.warning(f"解析 {ticker} K 線資料失敗（已跳過此根）: {e}")
                    continue

            logger.debug(f"取得 {ticker} 歷史資料：{len(bars_data)} 根 K 線")
            return bars_data

        except httpx.HTTPStatusError as e:
            self._handle_http_error(ticker, e)
            return []
        except httpx.TimeoutException:
            logger.error(f"取得 {ticker} 歷史資料超時（>10s）")
            return []
        except Exception as e:
            logger.error(f"取得 {ticker} 歷史資料時發生未預期錯誤: {e}", exc_info=True)
            return []

    # ──────────────────────────────────────
    # 內部實作
    # ──────────────────────────────────────

    async def _poll_loop(self) -> None:
        """
        輪詢主迴圈。

        每次迴圈：
        1. 若有 tickers → 呼叫快照端點，更新快取
        2. 等待 poll_interval 秒
        3. 若發生錯誤 → 記錄後繼續（不終止迴圈）
        """
        while True:
            try:
                if self._tickers:
                    await self._fetch_and_update()
                else:
                    logger.debug("觀察清單為空，跳過本次輪詢")
            except asyncio.CancelledError:
                raise
            except Exception as e:
                logger.error(f"輪詢迴圈發生未預期錯誤: {e}", exc_info=True)

            await asyncio.sleep(self._poll_interval)

    async def _fetch_and_update(self) -> None:
        """
        呼叫 Massive 快照端點並批次更新 PriceCache。

        批次錯誤恢復策略：
        - 整個請求失敗（網路錯誤/429）→ 記錄，跳過本次更新，下次重試
        - 單個 ticker 解析失敗（資料格式異常）→ 記錄，跳過此 ticker，其他繼續
        - 某個 ticker 不在回應中（Massive 無資料）→ 靜默跳過（快取保留舊值）
        """
        tickers_snapshot = list(self._tickers)  # 複製一份，防止並發修改
        ticker_param = ",".join(sorted(tickers_snapshot))

        try:
            resp = await self._client.get(
                "/v2/snapshot/locale/us/markets/stocks/tickers",
                params={"tickers": ticker_param},
            )
            resp.raise_for_status()
        except httpx.HTTPStatusError as e:
            self._handle_http_error("batch_snapshot", e)
            return
        except httpx.TimeoutException:
            logger.error("快照端點請求超時，略過本次更新")
            return
        except httpx.ConnectError:
            logger.error("無法連線到 Massive API，略過本次更新")
            return

        data = resp.json()
        ticker_items = data.get("tickers", [])

        if not ticker_items:
            logger.warning("Massive 快照回傳空資料")
            return

        # 取得現有快取（用於計算 prev_price）
        existing = await self._cache.get_all()
        updates: dict[str, PriceUpdate] = {}
        missing = set(tickers_snapshot)

        for t in ticker_items:
            symbol = t.get("ticker")
            if not symbol:
                continue

            missing.discard(symbol)

            try:
                # 優先用最新成交價，其次用今日收盤價
                last_trade = t.get("lastTrade") or {}
                day = t.get("day") or {}
                price_raw = last_trade.get("p") or day.get("c")

                if not price_raw:
                    logger.warning(f"Massive 回傳 {symbol} 無有效價格，略過")
                    continue

                price = float(price_raw)
                prev_day = t.get("prevDay") or {}
                prev_close_raw = prev_day.get("c")
                prev_close = float(prev_close_raw) if prev_close_raw else None

                # prev_price：取快取中的上一筆，若無則用 prev_close，否則同 price
                if symbol in existing:
                    prev_price = existing[symbol].price
                elif prev_close:
                    prev_price = prev_close
                else:
                    prev_price = price

                updates[symbol] = PriceUpdate.from_prices(
                    ticker=symbol,
                    price=price,
                    prev_price=prev_price,
                    prev_close=prev_close,
                    timestamp=datetime.utcnow(),
                )

            except (KeyError, ValueError, TypeError) as e:
                logger.warning(f"解析 {symbol} 快照資料失敗（已略過）: {e}")
                continue

        if missing:
            logger.debug(f"本次快照缺少以下 ticker 的資料: {missing}")

        if updates:
            await self._cache.set_many(updates)
            logger.debug(f"更新 {len(updates)}/{len(tickers_snapshot)} 個 ticker 的價格")

    def _handle_http_error(self, context: str, error: httpx.HTTPStatusError) -> None:
        """統一處理 HTTP 錯誤回應，提供明確的診斷訊息。"""
        status = error.response.status_code
        if status == 401 or status == 403:
            logger.error(
                f"[{context}] API Key 無效或已達方案限制（HTTP {status}）"
                "，請檢查 MASSIVE_API_KEY 環境變數"
            )
        elif status == 429:
            logger.warning(
                f"[{context}] 超過速率限制（HTTP 429），"
                f"建議增大 POLL_INTERVAL_SEC（目前 {self._poll_interval}s）"
            )
        elif status >= 500:
            logger.error(f"[{context}] Massive API 伺服器錯誤（HTTP {status}）")
        else:
            logger.error(f"[{context}] HTTP 錯誤 {status}: {error}")
```

---

## 8. 工廠函式

```python
# backend/app/market/factory.py

import logging
import os

from .base import MarketDataProvider
from .cache import PriceCache
from .massive_provider import MassiveProvider
from .simulator_provider import SimulatorProvider

logger = logging.getLogger(__name__)

# 預設觀察清單（與 DB schema 種子資料一致）
DEFAULT_TICKERS: list[str] = [
    "AAPL", "GOOGL", "MSFT", "AMZN", "TSLA",
    "NVDA", "META", "JPM", "V", "NFLX",
]


def create_market_provider(cache: PriceCache) -> MarketDataProvider:
    """
    依環境變數建立並設定市場資料提供者。

    選擇邏輯：
    - MASSIVE_API_KEY 已設定且非空 → MassiveProvider
    - 其他情況（含空字串）→ SimulatorProvider

    注意：
    - 回傳的 provider 已設定初始 tickers（來自 DEFAULT_TICKERS）
    - 尚未呼叫 start()；呼叫方（FastAPI lifespan）負責 start()/stop()
    - _tickers 直接賦值而非 await add_ticker()，因為 start() 尚未執行

    Returns:
        已設定但尚未啟動的 MarketDataProvider 實例。
    """
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()

    if api_key:
        poll_interval = float(os.environ.get("POLL_INTERVAL_SEC", "15.0"))
        provider: MarketDataProvider = MassiveProvider(
            api_key=api_key,
            cache=cache,
            poll_interval=poll_interval,
        )
        provider_name = f"MassiveProvider (間隔={poll_interval}s)"
    else:
        provider = SimulatorProvider(cache=cache)
        provider_name = "SimulatorProvider"

    # 直接設定初始 ticker 集合（繞過非同步 add_ticker）
    provider._tickers = set(DEFAULT_TICKERS)

    logger.info(
        f"已建立市場資料提供者: {provider_name}，"
        f"初始觀察清單: {sorted(DEFAULT_TICKERS)}"
    )
    return provider
```

---

## 9. FastAPI 整合

### 9.1 應用入口與 Lifespan

```python
# backend/app/main.py

import logging
from contextlib import asynccontextmanager

from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles

from .market.cache import PriceCache
from .market.factory import create_market_provider
from .routers import market, stream, watchlist, portfolio, chat

logger = logging.getLogger(__name__)

# ──────────────────────────────────────
# 全域單例（在 lifespan 中初始化）
# ──────────────────────────────────────
price_cache = PriceCache()
market_provider = create_market_provider(price_cache)


@asynccontextmanager
async def lifespan(app: FastAPI):
    """
    FastAPI lifespan context manager。

    啟動順序：
    1. 啟動市場資料 Provider（開始更新 PriceCache）
    2. yield（應用開始接受請求）
    3. 停止 Provider（清理背景任務與 HTTP 連線）

    例外處理原則：
    - Provider 啟動失敗 → 記錄錯誤後繼續（應用仍可運行，只是沒有市場資料）
    - Provider 停止失敗 → 記錄但不重新拋出（避免 shutdown 程序中斷）
    """
    # 啟動
    try:
        await market_provider.start()
        logger.info("市場資料 Provider 已成功啟動")
    except Exception as e:
        logger.error(f"市場資料 Provider 啟動失敗（應用繼續運行）: {e}", exc_info=True)

    yield  # 應用運行中

    # 關閉
    try:
        await market_provider.stop()
        logger.info("市場資料 Provider 已成功停止")
    except Exception as e:
        logger.error(f"市場資料 Provider 停止時發生錯誤（已忽略）: {e}", exc_info=True)


# ──────────────────────────────────────
# FastAPI 應用
# ──────────────────────────────────────
app = FastAPI(
    title="FinAlly API",
    lifespan=lifespan,
)

# 依賴注入：讓路由能取得 price_cache 與 market_provider
app.state.price_cache = price_cache
app.state.market_provider = market_provider

# 路由掛載
app.include_router(stream.router)
app.include_router(market.router)
app.include_router(watchlist.router)
app.include_router(portfolio.router)
app.include_router(chat.router)

# 靜態前端（Next.js export，最後掛載避免遮蔽 API 路由）
app.mount("/", StaticFiles(directory="static", html=True), name="static")
```

### 9.2 依賴注入輔助函式

```python
# backend/app/dependencies.py

from fastapi import Request
from .market.cache import PriceCache
from .market.base import MarketDataProvider


def get_price_cache(request: Request) -> PriceCache:
    """從 app.state 取得 PriceCache 單例。"""
    return request.app.state.price_cache


def get_market_provider(request: Request) -> MarketDataProvider:
    """從 app.state 取得 MarketDataProvider 單例。"""
    return request.app.state.market_provider
```

---

## 10. SSE 串流端點

### 10.1 SSE 事件規格

SSE 串流在單一 `GET /api/stream/prices` 連線中傳送所有市場相關事件。每個事件使用標準 SSE 格式：

```
event: <type>\n
data: <json>\n
\n
```

#### 事件類型一覽

| event | 觸發時機 | data 結構 |
|-------|---------|----------|
| `snapshot` | 連線建立後立即推送一次（含所有 ticker 的當前價格） | 見下方 |
| `price` | 每次 PriceCache 更新後（約 500ms 一次） | 見下方 |
| `ticker_removed` | 觀察清單移除 ticker 時 | `{"ticker": "AAPL"}` |
| `heartbeat` | 每 30 秒一次（無價格更新時防止連線超時） | `{"ts": "2026-05-25T12:00:00Z"}` |

#### `snapshot` 事件 payload

連線後立即推送，供前端初始化顯示所有 ticker 的當前價格：

```json
{
  "AAPL": {
    "ticker": "AAPL",
    "price": 191.23,
    "prev_price": 190.88,
    "prev_close": 189.50,
    "change_pct": 0.91,
    "direction": "up",
    "timestamp": "2026-05-25T08:30:01.234Z"
  },
  "MSFT": { ... },
  ...
}
```

#### `price` 事件 payload

每次 PriceCache 被更新後推送，包含本次更新的所有 ticker：

```json
{
  "AAPL": {
    "ticker": "AAPL",
    "price": 191.45,
    "prev_price": 191.23,
    "prev_close": 189.50,
    "change_pct": 1.03,
    "direction": "up",
    "timestamp": "2026-05-25T08:30:01.734Z"
  }
}
```

> **注意**：每次 `price` 事件包含**本次批次更新的所有 ticker**，而非單一 ticker。模擬器每次 tick 更新全部 ticker，因此每個 price 事件通常包含所有 ticker；Massive 輪詢同樣批次更新。

#### `ticker_removed` 事件 payload

```json
{
  "ticker": "AAPL"
}
```

前端收到此事件後應：
1. 從觀察清單 UI 移除該 ticker
2. 清空該 ticker 的 sparkline 資料

### 10.2 SSE 端點實作

```python
# backend/app/routers/stream.py

import asyncio
import json
import logging
from datetime import datetime

from fastapi import APIRouter, Depends, Request
from fastapi.responses import StreamingResponse
from sse_starlette.sse import EventSourceResponse  # pip install sse-starlette

from ..dependencies import get_price_cache
from ..market.cache import PriceCache

logger = logging.getLogger(__name__)
router = APIRouter(prefix="/api")

# 無更新時的心跳間隔（防止 Nginx/Load Balancer 超時關閉連線）
HEARTBEAT_INTERVAL_SEC = 30.0


async def price_event_generator(request: Request, cache: PriceCache):
    """
    SSE 事件產生器。

    流程：
    1. 立即推送 `snapshot` 事件（目前所有 ticker 的最新價格）
    2. 進入迴圈：等待快取更新（最多 500ms），推送 `price` 事件
    3. 若超過 HEARTBEAT_INTERVAL_SEC 秒沒有更新，推送 `heartbeat`
    4. 檢測客戶端斷線後乾淨退出（不留懸空協程）
    """

    # 步驟 1：初始快照
    prices = await cache.get_all()
    if prices:
        yield {
            "event": "snapshot",
            "data": json.dumps({t: u.to_dict() for t, u in prices.items()}),
        }

    last_heartbeat = datetime.utcnow()

    # 步驟 2：持續推送
    while True:
        # 檢測客戶端是否已斷線（避免資源洩漏）
        if await request.is_disconnected():
            logger.debug("SSE 客戶端已斷線，結束事件產生器")
            break

        updated = await cache.wait_for_update(timeout=0.5)

        if updated:
            prices = await cache.get_all()
            if prices:
                yield {
                    "event": "price",
                    "data": json.dumps({t: u.to_dict() for t, u in prices.items()}),
                }
            last_heartbeat = datetime.utcnow()
        else:
            # 超時：檢查是否需要心跳
            now = datetime.utcnow()
            if (now - last_heartbeat).total_seconds() >= HEARTBEAT_INTERVAL_SEC:
                yield {
                    "event": "heartbeat",
                    "data": json.dumps({"ts": now.isoformat() + "Z"}),
                }
                last_heartbeat = now


@router.get("/stream/prices")
async def stream_prices(
    request: Request,
    cache: PriceCache = Depends(get_price_cache),
):
    """
    即時價格 SSE 串流。

    客戶端使用標準 EventSource API 連線：
        const es = new EventSource('/api/stream/prices');
        es.addEventListener('snapshot', (e) => { ... });
        es.addEventListener('price', (e) => { ... });
        es.addEventListener('ticker_removed', (e) => { ... });
        es.addEventListener('heartbeat', () => {});  // 可忽略
    """
    return EventSourceResponse(
        price_event_generator(request, cache),
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",  # 禁用 Nginx 緩衝
        },
    )
```

### 10.3 觀察清單移除時通知 SSE

觀察清單路由在移除 ticker 時，除了更新 DB 和 PriceCache，也需要通知 SSE 推送 `ticker_removed` 事件。

實作方式：在 `PriceCache` 中加入一個 `removed_tickers` 佇列，SSE 產生器讀取後推送事件：

```python
# backend/app/market/cache.py 補充

class PriceCache:
    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._lock = asyncio.Lock()
        self._updated_event = asyncio.Event()
        self._removed_tickers: list[str] = []  # 新增

    async def remove(self, ticker: str) -> None:
        """移除 ticker，並記錄移除事件供 SSE 推送。"""
        async with self._lock:
            self._prices.pop(ticker, None)
            self._removed_tickers.append(ticker)
        self._updated_event.set()

    async def pop_removed_tickers(self) -> list[str]:
        """
        取出並清空待通知的移除事件列表（供 SSE 產生器呼叫）。
        """
        async with self._lock:
            removed = list(self._removed_tickers)
            self._removed_tickers.clear()
        return removed
```

SSE 產生器相應更新：

```python
# 在 price_event_generator 的迴圈中加入：

if updated:
    # 推送移除事件
    removed = await cache.pop_removed_tickers()
    for ticker in removed:
        yield {
            "event": "ticker_removed",
            "data": json.dumps({"ticker": ticker}),
        }
    
    # 推送價格更新
    prices = await cache.get_all()
    if prices:
        yield {
            "event": "price",
            "data": json.dumps({t: u.to_dict() for t, u in prices.items()}),
        }
```

---

## 11. 歷史資料端點

```python
# backend/app/routers/market.py

from fastapi import APIRouter, Depends, HTTPException

from ..dependencies import get_market_provider
from ..market.base import MarketDataProvider

router = APIRouter(prefix="/api")


@router.get("/prices/history/{ticker}")
async def get_price_history(
    ticker: str,
    bars: int = 100,
    provider: MarketDataProvider = Depends(get_market_provider),
):
    """
    取得指定 ticker 的歷史 K 線。

    Response 範例：
    {
        "ticker": "AAPL",
        "bars": [
            {
                "timestamp": "2026-02-15T00:00:00Z",
                "open": 185.20,
                "high": 188.45,
                "low": 184.80,
                "close": 187.65,
                "volume": 52341200
            },
            ...
        ]
    }

    bars 參數：預設 100，最大 500。
    若 ticker 不存在或資料取得失敗，回傳空陣列（不回傳 404），
    前端應優雅地顯示「暫無資料」而非崩潰。
    """
    ticker = ticker.upper()
    bars = min(max(bars, 1), 500)  # 限制在 1–500

    history = await provider.get_price_history(ticker=ticker, bars=bars)

    return {
        "ticker": ticker,
        "bars": [bar.to_dict() for bar in history],
    }
```

---

## 12. 觀察清單動態同步

觀察清單的 CRUD 操作需要同步更新三個地方：

1. **SQLite 資料庫**（持久化）
2. **MarketDataProvider**（決定哪些 ticker 被模擬/輪詢）
3. **PriceCache**（移除時清除快取，新增時等待下次 tick 自然填入）

```python
# backend/app/routers/watchlist.py（片段）

from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel

from ..dependencies import get_market_provider, get_price_cache
from ..market.base import MarketDataProvider
from ..market.cache import PriceCache
from ..db import database  # 假設 db 模組提供 DB 操作

router = APIRouter(prefix="/api")


class AddTickerRequest(BaseModel):
    ticker: str


@router.get("/watchlist")
async def get_watchlist(
    provider: MarketDataProvider = Depends(get_market_provider),
):
    """取得觀察清單，附上最新價格。"""
    prices = await provider.get_latest_prices()
    # TODO: 從 DB 取 watchlist，合併最新價格後回傳
    ...


@router.post("/watchlist", status_code=201)
async def add_to_watchlist(
    body: AddTickerRequest,
    provider: MarketDataProvider = Depends(get_market_provider),
):
    """
    新增 ticker 到觀察清單。

    步驟：
    1. 驗證 ticker 格式（1–5 個大寫字母）
    2. 寫入 DB（watchlist 表）
    3. 通知 Provider 開始追蹤此 ticker
    4. 回傳 201 + 新增的 ticker
    """
    ticker = body.ticker.strip().upper()
    if not ticker or not ticker.isalpha() or len(ticker) > 10:
        raise HTTPException(status_code=400, detail="無效的 ticker 格式")

    # TODO: 寫入 DB，處理 UNIQUE 約束（若已存在回傳 200 而非 409）

    await provider.add_ticker(ticker)
    return {"ticker": ticker, "added": True}


@router.delete("/watchlist/{ticker}", status_code=200)
async def remove_from_watchlist(
    ticker: str,
    provider: MarketDataProvider = Depends(get_market_provider),
    cache: PriceCache = Depends(get_price_cache),
):
    """
    從觀察清單移除 ticker。

    步驟：
    1. 從 DB 刪除（watchlist 表）
    2. 通知 Provider 停止追蹤
       - Provider 內部會呼叫 cache.remove()
       - cache.remove() 會記錄移除事件（_removed_tickers）
    3. SSE 事件產生器在下次喚醒時推送 `ticker_removed` 事件
    4. 前端收到後清空此 ticker 的 sparkline 資料

    注意：Provider.remove_ticker() 已內含 cache.remove()，
    不需要在路由中重複呼叫 cache.remove()。
    """
    ticker = ticker.upper()

    # TODO: 從 DB 刪除

    await provider.remove_ticker(ticker)  # 包含 cache.remove()
    return {"ticker": ticker, "removed": True}
```

---

## 13. 測試策略

### 13.1 模擬器單元測試

```python
# backend/tests/test_simulator.py

import asyncio
import math
import pytest
from app.market.simulator_provider import SimulatorProvider, DT, JUMP_PROBABILITY
from app.market.cache import PriceCache


@pytest.fixture
def sim_with_aapl():
    cache = PriceCache()
    sim = SimulatorProvider(cache)
    sim._tickers = {"AAPL"}
    sim._initialize_prices()
    return sim, cache


@pytest.mark.asyncio
async def test_gbm_prices_always_positive(sim_with_aapl):
    """GBM 產生的價格永遠為正數。"""
    sim, cache = sim_with_aapl
    await sim.start()
    await asyncio.sleep(1.5)  # 約 3 個 tick
    await sim.stop()

    prices = await cache.get_all()
    assert "AAPL" in prices
    assert prices["AAPL"].price > 0


@pytest.mark.asyncio
async def test_price_direction_matches_change(sim_with_aapl):
    """direction 與實際價格變動方向一致。"""
    sim, cache = sim_with_aapl
    await sim.start()
    await asyncio.sleep(1.0)
    await sim.stop()

    prices = await cache.get_all()
    update = prices.get("AAPL")
    if update:
        if update.price > update.prev_price:
            assert update.direction == "up"
        elif update.price < update.prev_price:
            assert update.direction == "down"
        else:
            assert update.direction == "flat"


@pytest.mark.asyncio
async def test_price_history_deterministic():
    """相同 ticker 每次產生相同歷史（確定性回溯）。"""
    cache = PriceCache()
    sim = SimulatorProvider(cache)
    sim._tickers = {"AAPL"}
    sim._initialize_prices()

    history1 = await sim.get_price_history("AAPL", bars=50)
    history2 = await sim.get_price_history("AAPL", bars=50)

    assert len(history1) == 50
    assert [b.close for b in history1] == [b.close for b in history2]


@pytest.mark.asyncio
async def test_price_history_ordered_ascending():
    """歷史 K 線由舊到新排序。"""
    cache = PriceCache()
    sim = SimulatorProvider(cache)
    sim._tickers = {"MSFT"}
    sim._initialize_prices()

    history = await sim.get_price_history("MSFT", bars=20)
    timestamps = [b.timestamp for b in history]
    assert timestamps == sorted(timestamps)


@pytest.mark.asyncio
async def test_price_history_ohlc_valid():
    """每根 K 線的 high >= max(open, close) 且 low <= min(open, close)。"""
    cache = PriceCache()
    sim = SimulatorProvider(cache)
    sim._tickers = {"TSLA"}
    sim._initialize_prices()

    history = await sim.get_price_history("TSLA", bars=30)
    for bar in history:
        assert bar.high >= bar.open, f"high={bar.high} < open={bar.open}"
        assert bar.high >= bar.close, f"high={bar.high} < close={bar.close}"
        assert bar.low <= bar.open, f"low={bar.low} > open={bar.open}"
        assert bar.low <= bar.close, f"low={bar.low} > close={bar.close}"


@pytest.mark.asyncio
async def test_sector_correlation_tech_stocks():
    """科技股（AAPL, MSFT）方向應有高於隨機的同向率（> 50%）。"""
    cache = PriceCache()
    sim = SimulatorProvider(cache)
    sim._tickers = {"AAPL", "MSFT"}
    sim._initialize_prices()
    await sim.start()

    same_direction_count = 0
    samples = 100
    for _ in range(samples):
        await asyncio.sleep(0.5)
        prices = await cache.get_all()
        aapl = prices.get("AAPL")
        msft = prices.get("MSFT")
        if aapl and msft and aapl.direction != "flat" and msft.direction != "flat":
            if aapl.direction == msft.direction:
                same_direction_count += 1

    await sim.stop()
    # 科技股 ρ=0.6，相關係數 ≈ 0.36，同向率應顯著高於 50%
    assert same_direction_count / samples > 0.5


def test_unknown_ticker_gets_valid_default_config():
    """未知 ticker 使用有效的預設設定（正種子價格、合理波動率）。"""
    cache = PriceCache()
    sim = SimulatorProvider(cache)
    config = sim._get_config("XYZABC")
    assert config.seed_price > 0
    assert 0 < config.sigma < 1
    assert config.ticker == "XYZABC"


@pytest.mark.asyncio
async def test_add_remove_ticker_dynamic():
    """動態新增/移除 ticker 不影響其他 ticker 的模擬。"""
    cache = PriceCache()
    sim = SimulatorProvider(cache)
    sim._tickers = {"AAPL", "MSFT"}
    sim._initialize_prices()
    await sim.start()

    # 新增 ticker
    await sim.add_ticker("NVDA")
    await asyncio.sleep(0.6)  # 等待 1–2 個 tick
    prices = await cache.get_all()
    assert "NVDA" in prices

    # 移除 ticker
    await sim.remove_ticker("MSFT")
    await asyncio.sleep(0.6)
    prices = await cache.get_all()
    assert "MSFT" not in prices
    assert "AAPL" in prices  # 其他 ticker 不受影響

    await sim.stop()
```

### 13.2 快取單元測試

```python
# backend/tests/test_cache.py

import asyncio
import pytest
from datetime import datetime
from app.market.cache import PriceCache
from app.market.models import PriceUpdate


def make_update(ticker: str, price: float) -> PriceUpdate:
    return PriceUpdate.from_prices(ticker=ticker, price=price, prev_price=price - 0.1)


@pytest.mark.asyncio
async def test_set_and_get():
    cache = PriceCache()
    update = make_update("AAPL", 190.0)
    await cache.set("AAPL", update)
    all_prices = await cache.get_all()
    assert "AAPL" in all_prices
    assert all_prices["AAPL"].price == 190.0


@pytest.mark.asyncio
async def test_set_many_atomic():
    """set_many 應是原子操作（所有 ticker 同時可見）。"""
    cache = PriceCache()
    updates = {t: make_update(t, 100.0) for t in ["AAPL", "MSFT", "GOOGL"]}
    await cache.set_many(updates)
    all_prices = await cache.get_all()
    assert set(all_prices.keys()) == {"AAPL", "MSFT", "GOOGL"}


@pytest.mark.asyncio
async def test_remove():
    cache = PriceCache()
    await cache.set("AAPL", make_update("AAPL", 190.0))
    await cache.remove("AAPL")
    all_prices = await cache.get_all()
    assert "AAPL" not in all_prices


@pytest.mark.asyncio
async def test_wait_for_update_returns_true_on_set():
    cache = PriceCache()

    async def delayed_set():
        await asyncio.sleep(0.1)
        await cache.set("AAPL", make_update("AAPL", 190.0))

    asyncio.create_task(delayed_set())
    result = await cache.wait_for_update(timeout=1.0)
    assert result is True


@pytest.mark.asyncio
async def test_wait_for_update_returns_false_on_timeout():
    cache = PriceCache()
    result = await cache.wait_for_update(timeout=0.1)
    assert result is False


@pytest.mark.asyncio
async def test_pop_removed_tickers():
    """remove() 應記錄移除事件，pop_removed_tickers() 應消費並清空。"""
    cache = PriceCache()
    await cache.set("AAPL", make_update("AAPL", 190.0))
    await cache.remove("AAPL")

    removed = await cache.pop_removed_tickers()
    assert "AAPL" in removed

    # 第二次呼叫應為空
    removed2 = await cache.pop_removed_tickers()
    assert removed2 == []
```

### 13.3 Massive Provider 測試（Mock）

```python
# backend/tests/test_massive_provider.py

import json
import pytest
from unittest.mock import AsyncMock, MagicMock, patch

import httpx
from app.market.massive_provider import MassiveProvider
from app.market.cache import PriceCache


MOCK_SNAPSHOT_RESPONSE = {
    "status": "OK",
    "count": 2,
    "tickers": [
        {
            "ticker": "AAPL",
            "lastTrade": {"p": 191.23},
            "prevDay": {"c": 189.50},
            "todaysChangePerc": 0.91,
        },
        {
            "ticker": "MSFT",
            "lastTrade": None,
            "day": {"c": 415.50},
            "prevDay": {"c": 412.00},
            "todaysChangePerc": 0.85,
        },
    ],
}


@pytest.fixture
def mock_provider():
    cache = PriceCache()
    provider = MassiveProvider(api_key="test_key", cache=cache, poll_interval=999)
    provider._tickers = {"AAPL", "MSFT"}
    return provider, cache


@pytest.mark.asyncio
async def test_fetch_and_update_parses_response(mock_provider):
    """_fetch_and_update() 正確解析快照回應並更新快取。"""
    provider, cache = mock_provider

    mock_response = MagicMock()
    mock_response.json.return_value = MOCK_SNAPSHOT_RESPONSE
    mock_response.raise_for_status = MagicMock()

    provider._client = AsyncMock()
    provider._client.get = AsyncMock(return_value=mock_response)

    await provider._fetch_and_update()

    prices = await cache.get_all()
    assert "AAPL" in prices
    assert prices["AAPL"].price == 191.23
    assert prices["AAPL"].prev_close == 189.50

    assert "MSFT" in prices
    assert prices["MSFT"].price == 415.50  # fallback 到 day.c


@pytest.mark.asyncio
async def test_fetch_handles_missing_ticker(mock_provider):
    """Massive 回應缺少某個 ticker 時，快取保留舊值（不刪除）。"""
    provider, cache = mock_provider

    from app.market.models import PriceUpdate
    from datetime import datetime
    old_update = PriceUpdate.from_prices("GOOGL", 175.0, 174.5)
    await cache.set("GOOGL", old_update)

    partial_response = {
        "status": "OK",
        "tickers": [{"ticker": "AAPL", "lastTrade": {"p": 191.0}, "prevDay": {"c": 189.0}}],
    }
    mock_response = MagicMock()
    mock_response.json.return_value = partial_response
    mock_response.raise_for_status = MagicMock()

    provider._client = AsyncMock()
    provider._client.get = AsyncMock(return_value=mock_response)

    await provider._fetch_and_update()

    prices = await cache.get_all()
    # GOOGL 應保留舊值（未被刪除）
    assert "GOOGL" in prices
    assert prices["GOOGL"].price == 175.0


@pytest.mark.asyncio
async def test_fetch_handles_rate_limit(mock_provider, caplog):
    """429 速率限制應記錄警告，不拋例外。"""
    provider, cache = mock_provider

    mock_response = MagicMock()
    mock_response.status_code = 429
    http_error = httpx.HTTPStatusError("429", request=MagicMock(), response=mock_response)

    provider._client = AsyncMock()
    provider._client.get = AsyncMock(side_effect=http_error)

    import logging
    with caplog.at_level(logging.WARNING):
        await provider._fetch_and_update()  # 不應拋例外

    assert "429" in caplog.text or "速率限制" in caplog.text


@pytest.mark.asyncio
async def test_stop_is_idempotent(mock_provider):
    """stop() 可被多次呼叫而不拋例外。"""
    provider, _ = mock_provider
    await provider.stop()  # 未 start 就 stop
    await provider.stop()  # 第二次 stop
```

---

## 14. 設定參數參考

### 14.1 環境變數

| 變數 | 說明 | 預設值 | 範例 |
|------|------|--------|------|
| `MASSIVE_API_KEY` | Massive API Key；設定後使用真實資料 | （空，使用模擬器）| `abc123...` |
| `POLL_INTERVAL_SEC` | Massive 輪詢間隔（秒）| `15.0` | `2.0`（付費方案）|

### 14.2 模擬器調整指南

| 參數 | 位置 | 效果 | 建議範圍 |
|------|------|------|---------|
| `sigma`（波動率）| `DEFAULT_TICKER_CONFIGS` | 越大越震盪 | 穩健股 0.15–0.25；科技股 0.25–0.50 |
| `mu`（漂移）| `DEFAULT_TICKER_CONFIGS` | 長期趨勢 | 0.05–0.12（短期示範影響不大）|
| `UPDATE_INTERVAL_MS` | `simulator_provider.py` | 更新頻率 | 建議 500ms |
| `JUMP_PROBABILITY` | `simulator_provider.py` | 突發事件頻率 | 0.003–0.01 |
| `JUMP_MIN/MAX_PCT` | `simulator_provider.py` | 突發幅度 | 2%–5% |
| `SECTOR_CORRELATION` | `simulator_provider.py` | 板塊聯動程度 | 0.4–0.7 |

### 14.3 pyproject.toml 依賴項

```toml
[project]
name = "finally-backend"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.30.0",
    "sse-starlette>=2.1.0",    # SSE 支援
    "httpx>=0.27.0",           # Massive API HTTP 客戶端
    "python-dotenv>=1.0.0",    # .env 檔案讀取
]

[project.optional-dependencies]
test = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.23.0",
    "respx>=0.21.0",           # httpx mock 測試
]
```

> **注意**：不需安裝 `massive` Python SDK（我們直接用 `httpx` 呼叫 REST API），原因：
> 1. `httpx` 已在專案中使用，減少依賴
> 2. 更精確控制請求格式與錯誤處理
> 3. 避免 SDK 版本不相容的問題

---

## 附錄：資料流完整時序圖

```
使用者瀏覽器                  FastAPI                    背景任務
     │                          │                            │
     │── GET /api/stream/prices ─▶                           │
     │                          │── SSE 連線建立             │
     │                          │                            │
     │                          │◀─── cache.wait_for_update()│
     │                          │              │             │
     │                          │         (500ms 等待)       │
     │                          │              │             │
     │                          │              │     SimulatorProvider._tick()
     │                          │              │     │── GBM 計算新價格
     │                          │              │     │── cache.set_many()
     │                          │              │     └── cache._updated_event.set()
     │                          │              │             │
     │                          │◀─────────────┘             │
     │                          │── 讀取 cache.get_all()     │
     │◀── SSE event: "price" ───│                            │
     │    { AAPL: {...}, ... }  │                            │
     │                          │                            │
     │── DELETE /api/watchlist/AAPL ─▶                       │
     │                          │── provider.remove_ticker("AAPL")
     │                          │   └── cache.remove("AAPL")
     │                          │       ├── 從 _prices 刪除  
     │                          │       ├── _removed_tickers.append("AAPL")
     │                          │       └── _updated_event.set()
     │◀── 200 OK ───────────────│                            │
     │                          │                            │
     │                          │◀─ wait_for_update 喚醒     │
     │◀── SSE event: "ticker_removed" ─│                     │
     │    { "ticker": "AAPL" } │                            │
     │ (清空 AAPL sparkline)    │                            │
```
