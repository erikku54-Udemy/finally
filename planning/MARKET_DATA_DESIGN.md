# MARKET_DATA_DESIGN.md — 市場資料後端完整設計

本文件是 FinAlly 市場資料後端的完整實作藍圖，整合 `MARKET_INTERFACE.md`、`MARKET_SIMULATOR.md`、`MASSIVE_API.md` 的所有設計，並補充原文件未涵蓋的細節（SSE 事件規格、錯誤處理、速率限制退避、相關性矩陣等）。

實作者應以本文件為最終參考，優先於其他三份拆分文件。

---

## 目錄

1. [架構總覽](#1-架構總覽)
2. [目錄結構](#2-目錄結構)
3. [資料模型](#3-資料模型)
4. [共享價格快取（PriceCache）](#4-共享價格快取pricecache)
5. [抽象介面（MarketDataProvider）](#5-抽象介面marketdataprovider)
6. [SimulatorProvider — GBM 模擬器](#6-simulatorprovider--gbm-模擬器)
7. [MassiveProvider — 真實市場資料](#7-massiveprovider--真實市場資料)
8. [工廠函式（create_market_provider）](#8-工廠函式create_market_provider)
9. [FastAPI 整合與生命週期](#9-fastapi-整合與生命週期)
10. [SSE 串流端點](#10-sse-串流端點)
11. [歷史價格 REST 端點](#11-歷史價格-rest-端點)
12. [觀察清單 API 整合](#12-觀察清單-api-整合)
13. [環境變數參考](#13-環境變數參考)
14. [測試策略](#14-測試策略)
15. [參數調整指南](#15-參數調整指南)

---

## 1. 架構總覽

### 資料流

```
┌──────────────────────────────────────────────────────────────┐
│                    FastAPI Process                           │
│                                                              │
│  ┌────────────────┐    write     ┌─────────────┐            │
│  │ MarketData     │ ──────────→  │  PriceCache │            │
│  │ Provider       │              │  (in-memory)│            │
│  │                │              └──────┬──────┘            │
│  │ SimulatorProv  │                     │ read              │
│  │   or           │              ┌──────▼──────┐            │
│  │ MassiveProv    │              │ SSE Stream  │ ──→ Browser │
│  └────────────────┘              │  Generator  │            │
│         ▲                        └─────────────┘            │
│         │ add/remove ticker                                  │
│  ┌──────┴──────┐                                            │
│  │  Watchlist  │                                            │
│  │  API Router │                                            │
│  └─────────────┘                                            │
└──────────────────────────────────────────────────────────────┘
```

### 設計原則

| 原則 | 說明 |
|------|------|
| **單一介面，兩種實作** | `MarketDataProvider` 抽象類別定義合約；`SimulatorProvider` 與 `MassiveProvider` 各自實作 |
| **工廠函式選擇實作** | `create_market_provider()` 依環境變數決定 Provider，呼叫方不需判斷 |
| **快取解耦推送** | Provider 只寫 `PriceCache`；SSE 串流只讀 `PriceCache`，兩者完全解耦 |
| **動態觀察清單** | Provider 支援執行期動態新增/移除 ticker，不需重啟 |
| **非同步優先** | 所有 IO 操作使用 `async/await`，不阻塞 FastAPI event loop |
| **異常隔離** | Provider 內部吸收錯誤並記錄 log；外部任何失敗都不應導致應用崩潰 |

---

## 2. 目錄結構

```
backend/
└── app/
    ├── main.py                    # FastAPI 應用、lifespan 管理
    ├── market/
    │   ├── __init__.py
    │   ├── models.py              # PriceUpdate, PriceBar 資料類別
    │   ├── cache.py               # PriceCache — 共享記憶體快取
    │   ├── base.py                # MarketDataProvider 抽象介面
    │   ├── simulator_provider.py  # SimulatorProvider（GBM 模擬器）
    │   ├── massive_provider.py    # MassiveProvider（Massive REST API）
    │   └── factory.py             # create_market_provider()
    └── routers/
        ├── stream.py              # GET /api/stream/prices (SSE)
        ├── market.py              # GET /api/prices/history/{ticker}
        └── watchlist.py           # GET/POST/DELETE /api/watchlist
```

---

## 3. 資料模型

```python
# backend/app/market/models.py

from dataclasses import dataclass, field
from datetime import datetime


@dataclass
class PriceUpdate:
    """
    單一 ticker 的一次價格更新。
    由 Provider 產生，存入 PriceCache，再由 SSE 串流推送給前端。
    """
    ticker: str
    price: float
    prev_price: float          # 上一次更新的價格（前端用來判斷閃爍方向）
    prev_close: float | None   # 前一交易日收盤價（計算日漲跌幅用）
    change_pct: float          # 相對前日收盤的漲跌幅（%）
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
        """
        工廠方法：自動計算 direction 與 change_pct。

        change_pct 優先使用 prev_close 計算（日漲跌幅）；
        若無 prev_close 則改用 prev_price（tick-to-tick 漲跌幅）。
        """
        direction = (
            "up" if price > prev_price
            else "down" if price < prev_price
            else "flat"
        )
        if prev_close and prev_close > 0:
            change_pct = (price - prev_close) / prev_close * 100
        elif prev_price > 0:
            change_pct = (price - prev_price) / prev_price * 100
        else:
            change_pct = 0.0

        return cls(
            ticker=ticker,
            price=round(price, 2),
            prev_price=round(prev_price, 2),
            prev_close=round(prev_close, 2) if prev_close else None,
            change_pct=round(change_pct, 4),
            timestamp=timestamp or datetime.utcnow(),
            direction=direction,
        )


@dataclass
class PriceBar:
    """
    單根 K 線（OHLCV），用於歷史圖表初始化與 sparkline。
    由 Provider.get_price_history() 回傳。
    """
    timestamp: datetime
    open: float
    high: float
    low: float
    close: float
    volume: float
```

---

## 4. 共享價格快取（PriceCache）

`PriceCache` 是 Provider 與 SSE 串流之間的唯一共享狀態。它是純記憶體快取，無持久化。

```python
# backend/app/market/cache.py

import asyncio
from datetime import datetime
from .models import PriceUpdate


class PriceCache:
    """
    asyncio-safe 記憶體價格快取。

    寫入方：MarketDataProvider（SimulatorProvider 或 MassiveProvider）
    讀取方：SSE 串流生成器（price_event_generator）

    設計說明：
    - asyncio.Lock 保護對 _prices dict 的讀寫，防止 race condition
    - asyncio.Event 讓 SSE 生成器以事件驅動方式等待更新，
      避免 busy-loop polling
    - wait_for_update() 帶 timeout，確保 SSE 連線每秒至少心跳一次
    """

    def __init__(self) -> None:
        self._prices: dict[str, PriceUpdate] = {}
        self._lock = asyncio.Lock()
        self._updated_event = asyncio.Event()

    async def set(self, ticker: str, update: PriceUpdate) -> None:
        """更新單一 ticker 的價格並通知等待者。"""
        async with self._lock:
            self._prices[ticker] = update
        self._updated_event.set()

    async def set_many(self, updates: dict[str, PriceUpdate]) -> None:
        """批次更新多個 ticker 的價格並通知等待者（一次性通知）。"""
        async with self._lock:
            self._prices.update(updates)
        self._updated_event.set()

    async def get_all(self) -> dict[str, PriceUpdate]:
        """取得所有快取價格的快照副本（不持有 lock）。"""
        async with self._lock:
            return dict(self._prices)

    async def get(self, ticker: str) -> PriceUpdate | None:
        """取得指定 ticker 的最新價格，不存在時回傳 None。"""
        async with self._lock:
            return self._prices.get(ticker.upper())

    async def remove(self, ticker: str) -> None:
        """從快取移除指定 ticker（用於觀察清單移除）。"""
        async with self._lock:
            self._prices.pop(ticker.upper(), None)

    async def wait_for_update(self, timeout: float = 1.0) -> bool:
        """
        等待任何價格更新事件。SSE 生成器用此方法做事件驅動推送。

        Args:
            timeout: 最長等待秒數（預設 1.0）。

        Returns:
            True — 有更新發生；False — timeout（可用於發送心跳）。
        """
        try:
            await asyncio.wait_for(self._updated_event.wait(), timeout=timeout)
            self._updated_event.clear()
            return True
        except asyncio.TimeoutError:
            return False

    @property
    def size(self) -> int:
        """目前快取中的 ticker 數量（非 async 屬性，用於 logging）。"""
        return len(self._prices)
```

---

## 5. 抽象介面（MarketDataProvider）

所有 Provider 必須實作此介面。下游程式碼（SSE 串流、觀察清單 API、測試）只依賴介面，不依賴具體實作。

```python
# backend/app/market/base.py

from abc import ABC, abstractmethod
from .models import PriceUpdate, PriceBar


class MarketDataProvider(ABC):
    """
    市場資料提供者的抽象介面。

    生命週期：
        provider = create_market_provider(cache)  # 建立（含初始 tickers）
        await provider.start()                    # 啟動背景任務（在 lifespan 中）
        ...                                       # 執行期：add/remove tickers
        await provider.stop()                     # 停止背景任務（在 lifespan 中）
    """

    # 子類別應宣告此屬性供工廠函式初始化
    _tickers: set[str]

    @abstractmethod
    async def start(self) -> None:
        """
        啟動背景任務（模擬迴圈或輪詢迴圈）。
        在 FastAPI lifespan 的 startup 階段呼叫。
        實作應在此初始化資源（HTTP client、asyncio Task 等）。
        """
        ...

    @abstractmethod
    async def stop(self) -> None:
        """
        停止背景任務，釋放所有資源。
        在 FastAPI lifespan 的 shutdown 階段呼叫。
        實作應確保 Task 被 cancel 且 await，HTTP client 被關閉。
        """
        ...

    @abstractmethod
    async def add_ticker(self, ticker: str) -> None:
        """
        動態新增 ticker 至觀察清單。
        呼叫後，Provider 的下一個 tick/poll 即會包含此 ticker。
        ticker 應轉為大寫後儲存。
        """
        ...

    @abstractmethod
    async def remove_ticker(self, ticker: str) -> None:
        """
        動態從觀察清單移除 ticker。
        除了停止模擬/輪詢外，還應從 PriceCache 移除該 ticker 的快取。
        """
        ...

    @abstractmethod
    async def get_latest_prices(self) -> dict[str, PriceUpdate]:
        """
        取得所有被觀察 ticker 的最新價格快照。
        實作通常直接委派給 PriceCache.get_all()。

        Returns:
            dict，key 為 ticker symbol（大寫），value 為 PriceUpdate。
        """
        ...

    @abstractmethod
    async def get_price_history(
        self,
        ticker: str,
        bars: int = 100,
    ) -> list[PriceBar]:
        """
        取得指定 ticker 的歷史 K 線資料，供圖表初始化使用。

        Args:
            ticker: 股票代號（不分大小寫，實作內部轉大寫）
            bars:   要取得的 K 線根數（預設 100 根日線）

        Returns:
            PriceBar 列表，時間由舊到新排序（index 0 最舊）。
            若資料不足，回傳能取到的數量；取得失敗則回傳空列表。
        """
        ...
```

---

## 6. SimulatorProvider — GBM 模擬器

### 6.1 數學基礎：幾何布朗運動（GBM）

GBM 是金融學標準的股票價格隨機模型，離散化公式為：

```
S(t+Δt) = S(t) × exp((μ - σ²/2) × Δt + σ × √Δt × Z)
```

| 符號 | 意義 | 說明 |
|------|------|------|
| `S(t)` | 當前價格 | 美元 |
| `μ` (mu) | 年化漂移率 | 期望報酬，如 0.08 = 8%/年 |
| `σ` (sigma) | 年化波動率 | 如 0.25 = 25%/年（科技股） |
| `Δt` | 時間步長（年） | `500ms / (252天 × 6.5小時 × 3600秒)` |
| `Z` | 標準常態隨機變數 | `N(0,1)` |

**時間步長計算**：

```python
UPDATE_INTERVAL_MS = 500
TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600  # ≈ 5,896,800 秒
DT = (UPDATE_INTERVAL_MS / 1000) / TRADING_SECONDS_PER_YEAR
# DT ≈ 8.48e-8 年（500ms 換算）
```

### 6.2 板塊相關性模型

同板塊股票使用共同隨機因子 `Z_sector`，混合公式：

```
Z_ticker = ρ × Z_sector + √(1 - ρ²) × Z_individual
```

| 板塊 | 相關係數 ρ | 說明 |
|------|------------|------|
| tech | 0.60 | 科技股聯動性高 |
| finance | 0.50 | 金融股中度聯動 |
| ev | 0.40 | 電動車板塊 |
| media | 0.40 | 媒體娛樂板塊 |

**直覺解釋**：ρ=0.6 時，60% 的價格變動來自板塊共同因子（整個科技板塊的漲跌），40% 來自個股因素（公司特定消息）。

### 6.3 隨機跳動事件

模擬突發新聞或財報效果：
- 每個 ticker 每次 tick 有 **0.5%** 機率觸發
- 跳動幅度：**±2% 至 ±5%**（均勻分布，方向隨機）

### 6.4 完整程式碼

```python
# backend/app/market/simulator_provider.py

import asyncio
import logging
import math
import random
from dataclasses import dataclass
from datetime import datetime, timedelta

from .base import MarketDataProvider
from .cache import PriceCache
from .models import PriceBar, PriceUpdate

logger = logging.getLogger(__name__)

# ── 模擬參數常數 ──────────────────────────────────────────────
UPDATE_INTERVAL_MS = 500                         # 更新間隔（毫秒）
TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600     # 每年交易秒數
DT = (UPDATE_INTERVAL_MS / 1000) / TRADING_SECONDS_PER_YEAR  # 時間步長（年）

SECTOR_CORRELATION: dict[str, float] = {
    "tech":    0.60,
    "finance": 0.50,
    "ev":      0.40,
    "media":   0.40,
}

JUMP_PROBABILITY = 0.005   # 每 tick 觸發跳動的機率
JUMP_MIN_PCT     = 0.02    # 跳動最小幅度（2%）
JUMP_MAX_PCT     = 0.05    # 跳動最大幅度（5%）


@dataclass
class TickerConfig:
    """單一 ticker 的 GBM 模擬參數。"""
    ticker:     str
    seed_price: float   # 起始價格（接近真實市場水準）
    mu:         float   # 年化漂移率
    sigma:      float   # 年化波動率
    sector:     str     # 板塊分類（用於相關性）


# ── 預設 10 個 Ticker 設定 ────────────────────────────────────
DEFAULT_TICKER_CONFIGS: dict[str, TickerConfig] = {
    "AAPL":  TickerConfig("AAPL",  190.0,  0.08, 0.25, "tech"),
    "GOOGL": TickerConfig("GOOGL", 175.0,  0.07, 0.27, "tech"),
    "MSFT":  TickerConfig("MSFT",  415.0,  0.09, 0.23, "tech"),
    "AMZN":  TickerConfig("AMZN",  195.0,  0.10, 0.28, "tech"),
    "TSLA":  TickerConfig("TSLA",  245.0,  0.05, 0.55, "ev"),
    "NVDA":  TickerConfig("NVDA",  875.0,  0.12, 0.50, "tech"),
    "META":  TickerConfig("META",  510.0,  0.09, 0.32, "tech"),
    "JPM":   TickerConfig("JPM",   205.0,  0.06, 0.20, "finance"),
    "V":     TickerConfig("V",     280.0,  0.07, 0.18, "finance"),
    "NFLX":  TickerConfig("NFLX",  680.0,  0.08, 0.38, "media"),
}


class SimulatorProvider(MarketDataProvider):
    """
    使用幾何布朗運動（GBM）模擬股票價格的市場資料提供者。

    特性：
    - 每 500ms 更新一次所有被觀察 ticker 的價格
    - 同板塊 ticker 之間有板塊相關性（科技股聯動）
    - 偶爾發生隨機跳動事件模擬突發新聞
    - 未知 ticker 自動套用合理預設參數（seed $100，sigma 30%）
    - get_price_history() 使用確定性回溯，相同 ticker 每次圖表一致
    """

    def __init__(self, cache: PriceCache) -> None:
        self._cache = cache
        self._tickers: set[str] = set()
        self._current_prices: dict[str, float] = {}   # 目前模擬價格
        self._day_open_prices: dict[str, float] = {}  # 當日開盤（近似前日收盤）
        self._task: asyncio.Task | None = None

    # ── 生命週期 ────────────────────────────────────────────────

    async def start(self) -> None:
        """初始化所有 ticker 起始價格，啟動背景模擬任務。"""
        self._initialize_prices()
        self._task = asyncio.create_task(self._simulation_loop())
        logger.info(
            f"SimulatorProvider 已啟動，監控 {len(self._tickers)} 個 ticker"
        )

    async def stop(self) -> None:
        """取消並等待背景任務結束。"""
        if self._task:
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        logger.info("SimulatorProvider 已停止")

    # ── 觀察清單管理 ─────────────────────────────────────────────

    async def add_ticker(self, ticker: str) -> None:
        """新增 ticker；若沒有預設設定則使用通用預設值。"""
        ticker = ticker.upper()
        self._tickers.add(ticker)
        if ticker not in self._current_prices:
            config = self._get_or_default_config(ticker)
            self._current_prices[ticker] = config.seed_price
            self._day_open_prices[ticker] = config.seed_price

    async def remove_ticker(self, ticker: str) -> None:
        """移除 ticker 並清除快取。"""
        ticker = ticker.upper()
        self._tickers.discard(ticker)
        self._current_prices.pop(ticker, None)
        self._day_open_prices.pop(ticker, None)
        await self._cache.remove(ticker)

    # ── 資料存取 ─────────────────────────────────────────────────

    async def get_latest_prices(self) -> dict[str, PriceUpdate]:
        return await self._cache.get_all()

    async def get_price_history(
        self, ticker: str, bars: int = 100
    ) -> list[PriceBar]:
        """
        確定性歷史回溯：以 hash(ticker) 為種子，向前回溯 bars 根日線。

        演算法：
        1. 取得目前模擬價格作為基準
        2. 用固定 seed 的 RNG 反向產生 bars 個價格（由最新往回推）
        3. 反轉為由舊到新的序列
        4. 為每根 K 線加上 open/high/low/volume 隨機偏移

        優點：相同 ticker 每次頁面載入都呈現一致的歷史走勢。
        """
        ticker = ticker.upper()
        config = self._get_or_default_config(ticker)
        current = self._current_prices.get(ticker, config.seed_price)

        rng = random.Random(hash(ticker) % (2 ** 32))
        dt_day = 1 / 252  # 一個交易日的時間步長（年）

        # 從當前價格向前回溯，產生 close 序列
        closes = [current]
        for _ in range(bars - 1):
            z = rng.gauss(0, 1)
            drift = (config.mu - 0.5 * config.sigma ** 2) * dt_day
            diffusion = config.sigma * math.sqrt(dt_day) * z
            # 向前回溯：除以 exp(drift + diffusion) 得到「前一天」價格
            closes.append(closes[-1] / math.exp(drift + diffusion))

        closes.reverse()  # 反轉：index 0 = 最舊

        result: list[PriceBar] = []
        base_time = datetime.utcnow() - timedelta(days=bars)
        day_sigma = config.sigma * math.sqrt(dt_day)

        for i, close in enumerate(closes):
            close = max(round(close, 2), 0.01)
            open_ = round(
                close * (1 + rng.gauss(0, day_sigma * 0.3)), 2
            )
            high = round(
                max(open_, close) * (1 + abs(rng.gauss(0, day_sigma * 0.2))), 2
            )
            low = round(
                min(open_, close) * (1 - abs(rng.gauss(0, day_sigma * 0.2))), 2
            )
            volume = round(abs(rng.gauss(10_000_000, 3_000_000)))

            result.append(PriceBar(
                timestamp=base_time + timedelta(days=i),
                open=max(open_, 0.01),
                high=max(high, open_, close),   # 確保 high ≥ open, close
                low=min(low, open_, close),      # 確保 low ≤ open, close
                close=close,
                volume=volume,
            ))

        return result

    # ── 內部實作 ─────────────────────────────────────────────────

    def _initialize_prices(self) -> None:
        """以種子價格初始化所有被觀察 ticker。"""
        for ticker in self._tickers:
            if ticker not in self._current_prices:
                config = self._get_or_default_config(ticker)
                self._current_prices[ticker] = config.seed_price
                self._day_open_prices[ticker] = config.seed_price

    async def _simulation_loop(self) -> None:
        """主模擬迴圈：每 UPDATE_INTERVAL_MS 執行一次 tick。"""
        while True:
            try:
                await self._tick()
            except asyncio.CancelledError:
                raise
            except Exception as e:
                logger.error(f"模擬器 tick 發生錯誤: {e}", exc_info=True)
            await asyncio.sleep(UPDATE_INTERVAL_MS / 1000)

    async def _tick(self) -> None:
        """
        執行一次 GBM 步進，批次更新所有 ticker 並寫入 PriceCache。

        流程：
        1. 為各板塊產生共同隨機因子 Z_sector
        2. 對每個 ticker 混合板塊因子與個別因子得到最終 Z
        3. 套用 GBM 公式計算新價格
        4. 以 JUMP_PROBABILITY 機率觸發跳動事件
        5. 批次寫入 PriceCache（一次 set_many 呼叫）
        """
        if not self._tickers:
            return

        # 步驟 1：各板塊共同因子
        sector_factors: dict[str, float] = {
            sector: random.gauss(0, 1)
            for sector in SECTOR_CORRELATION
        }

        updates: dict[str, PriceUpdate] = {}

        for ticker in list(self._tickers):
            config = self._get_or_default_config(ticker)
            prev_price = self._current_prices.get(ticker, config.seed_price)

            # 步驟 2：混合板塊與個別因子
            rho = SECTOR_CORRELATION.get(config.sector, 0.4)
            z_sector = sector_factors.get(config.sector, random.gauss(0, 1))
            z_individual = random.gauss(0, 1)
            z = rho * z_sector + math.sqrt(1 - rho ** 2) * z_individual

            # 步驟 3：GBM 更新
            drift = (config.mu - 0.5 * config.sigma ** 2) * DT
            diffusion = config.sigma * math.sqrt(DT) * z
            new_price = prev_price * math.exp(drift + diffusion)

            # 步驟 4：隨機跳動事件
            if random.random() < JUMP_PROBABILITY:
                jump_pct = random.uniform(JUMP_MIN_PCT, JUMP_MAX_PCT)
                direction = 1 if random.random() > 0.5 else -1
                new_price *= 1 + direction * jump_pct
                logger.debug(
                    f"[JUMP] {ticker}: {prev_price:.2f} → {new_price:.2f} "
                    f"({direction * jump_pct * 100:+.1f}%)"
                )

            new_price = max(round(new_price, 2), 0.01)  # 確保不低於 $0.01
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

    def _get_or_default_config(self, ticker: str) -> TickerConfig:
        """
        取得 ticker 設定。未知 ticker 使用保守預設值（seed $100，sigma 30%）。
        """
        return DEFAULT_TICKER_CONFIGS.get(
            ticker,
            TickerConfig(
                ticker=ticker,
                seed_price=100.0,
                mu=0.07,
                sigma=0.30,
                sector="tech",
            ),
        )
```

---

## 7. MassiveProvider — 真實市場資料

### 7.1 API 基本資訊

| 項目 | 內容 |
|------|------|
| 基礎網址 | `https://api.massive.com`（兼容 `https://api.polygon.io`） |
| 認證方式 | `Authorization: Bearer <KEY>` header |
| 安裝套件 | `uv add massive` 或 `uv add httpx` |
| 免費方案 | 每分鐘 5 次請求，15 分鐘延遲 |

### 7.2 核心端點：全市場快照

```
GET /v2/snapshot/locale/us/markets/stocks/tickers?tickers=AAPL,MSFT,...
```

**回應結構（關鍵欄位）：**

```json
{
  "status": "OK",
  "count": 2,
  "tickers": [
    {
      "ticker": "AAPL",
      "lastTrade":   { "p": 187.65 },
      "day":         { "o": 185.20, "h": 188.45, "l": 184.80, "c": 187.65, "v": 52341200 },
      "prevDay":     { "c": 185.92 },
      "todaysChangePerc": 0.93,
      "updated": 1705615200100000000
    }
  ]
}
```

**價格選取優先順序**：`lastTrade.p` → `day.c`（`lastTrade` 是即時性最高的成交價）。

### 7.3 速率限制策略

| 方案 | 請求限制 | 建議 POLL_INTERVAL_SEC |
|------|----------|------------------------|
| 免費 | 5 次/分鐘 | 15.0 秒 |
| Starter | 無限 | 10.0 秒 |
| Developer | 無限 | 5.0 秒 |
| Advanced/Business | 無限 | 2.0 秒 |

**退避策略**：`_poll_loop` 在連續錯誤時指數退避（最多退至 120 秒），避免在 API 異常時大量重試。

### 7.4 完整程式碼

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
DEFAULT_POLL_INTERVAL = 15.0  # 免費方案預設（秒）
MAX_BACKOFF = 120.0            # 最大退避時間（秒）


class MassiveProvider(MarketDataProvider):
    """
    使用 Massive REST API 取得真實市場資料的 Provider。

    策略：
    - 使用 httpx.AsyncClient 非同步輪詢全市場快照端點
    - 每次輪詢批次取得所有被觀察 ticker（一次 HTTP 請求）
    - 速率限制（429）觸發指數退避
    - 單一 ticker 解析失敗不影響其他 ticker
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

    # ── 生命週期 ────────────────────────────────────────────────

    async def start(self) -> None:
        """啟動 HTTP client 與背景輪詢任務。"""
        self._client = httpx.AsyncClient(
            base_url=BASE_URL,
            timeout=10.0,
            headers={"Authorization": f"Bearer {self._api_key}"},
        )
        self._task = asyncio.create_task(self._poll_loop())
        logger.info(
            f"MassiveProvider 已啟動，輪詢間隔 {self._poll_interval}s，"
            f"監控 {len(self._tickers)} 個 ticker"
        )

    async def stop(self) -> None:
        """取消輪詢任務並關閉 HTTP 連線。"""
        if self._task:
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        if self._client:
            await self._client.aclose()
        logger.info("MassiveProvider 已停止")

    # ── 觀察清單管理 ─────────────────────────────────────────────

    async def add_ticker(self, ticker: str) -> None:
        self._tickers.add(ticker.upper())

    async def remove_ticker(self, ticker: str) -> None:
        ticker = ticker.upper()
        self._tickers.discard(ticker)
        await self._cache.remove(ticker)

    # ── 資料存取 ─────────────────────────────────────────────────

    async def get_latest_prices(self) -> dict[str, PriceUpdate]:
        return await self._cache.get_all()

    async def get_price_history(
        self, ticker: str, bars: int = 100
    ) -> list[PriceBar]:
        """
        從 Massive 的 K 線聚合端點取得歷史日線資料。

        端點：GET /v2/aggs/ticker/{ticker}/range/1/day/{from}/{to}
        多取 bars*2 天以應對非交易日（週末、假日）。
        """
        if not self._client:
            return []

        ticker = ticker.upper()
        end = datetime.utcnow()
        start = end - timedelta(days=bars * 2)  # 多取以應對假日

        try:
            resp = await self._client.get(
                f"/v2/aggs/ticker/{ticker}/range/1/day"
                f"/{start.strftime('%Y-%m-%d')}/{end.strftime('%Y-%m-%d')}",
                params={"adjusted": "true", "sort": "asc", "limit": bars},
            )
            resp.raise_for_status()
            data = resp.json()

            result: list[PriceBar] = []
            for bar in data.get("results", [])[-bars:]:
                result.append(PriceBar(
                    timestamp=datetime.utcfromtimestamp(bar["t"] / 1000),
                    open=float(bar["o"]),
                    high=float(bar["h"]),
                    low=float(bar["l"]),
                    close=float(bar["c"]),
                    volume=float(bar.get("v", 0)),
                ))
            return result

        except httpx.HTTPStatusError as e:
            logger.error(
                f"取得 {ticker} 歷史資料失敗: HTTP {e.response.status_code}"
            )
            return []
        except Exception as e:
            logger.error(f"取得 {ticker} 歷史資料發生未預期錯誤: {e}")
            return []

    # ── 內部實作 ─────────────────────────────────────────────────

    async def _poll_loop(self) -> None:
        """
        背景輪詢迴圈，含指數退避策略。

        正常情況：每 poll_interval 秒輪詢一次。
        發生 429（速率限制）或連線錯誤時：
          - 退避時間從 poll_interval 開始，每次失敗加倍
          - 退避上限為 MAX_BACKOFF（120 秒）
          - 成功後重設退避計時器
        """
        backoff = self._poll_interval

        while True:
            try:
                if self._tickers:
                    await self._fetch_and_update()
                backoff = self._poll_interval  # 成功後重設退避

            except asyncio.CancelledError:
                raise

            except httpx.HTTPStatusError as e:
                if e.response.status_code == 429:
                    backoff = min(backoff * 2, MAX_BACKOFF)
                    logger.warning(
                        f"速率限制（429），退避 {backoff:.0f}s"
                    )
                elif e.response.status_code in (401, 403):
                    logger.error(
                        f"API Key 無效（{e.response.status_code}），停止輪詢"
                    )
                    return  # 無效 key，停止任務避免無謂重試
                else:
                    logger.error(f"HTTP 錯誤 {e.response.status_code}: {e}")
                    backoff = min(backoff * 2, MAX_BACKOFF)

            except (httpx.ConnectError, httpx.TimeoutException) as e:
                backoff = min(backoff * 2, MAX_BACKOFF)
                logger.warning(f"連線問題，退避 {backoff:.0f}s: {e}")

            except Exception as e:
                logger.error(f"輪詢發生未預期錯誤: {e}", exc_info=True)
                backoff = min(backoff * 2, MAX_BACKOFF)

            await asyncio.sleep(backoff)

    async def _fetch_and_update(self) -> None:
        """
        呼叫快照端點並將結果批次寫入 PriceCache。

        單一 ticker 解析失敗（缺少 price 欄位）時，跳過該 ticker，
        不影響其他 ticker 的更新。
        """
        ticker_param = ",".join(sorted(self._tickers))
        resp = await self._client.get(
            "/v2/snapshot/locale/us/markets/stocks/tickers",
            params={"tickers": ticker_param},
        )
        resp.raise_for_status()
        data = resp.json()

        existing = await self._cache.get_all()
        updates: dict[str, PriceUpdate] = {}
        now = datetime.utcnow()

        for t in data.get("tickers", []):
            symbol = t.get("ticker", "")
            if not symbol:
                continue

            try:
                # 價格優先順序：lastTrade.p → day.c
                last_trade = t.get("lastTrade") or {}
                day = t.get("day") or {}
                price = last_trade.get("p") or day.get("c")
                if not price:
                    logger.debug(f"跳過 {symbol}：無法取得價格")
                    continue

                prev_close = (t.get("prevDay") or {}).get("c")
                prev = existing.get(symbol)
                prev_price = prev.price if prev else float(price)

                updates[symbol] = PriceUpdate.from_prices(
                    ticker=symbol,
                    price=float(price),
                    prev_price=float(prev_price),
                    prev_close=float(prev_close) if prev_close else None,
                    timestamp=now,
                )

            except (KeyError, TypeError, ValueError) as e:
                logger.warning(f"解析 {symbol} 資料失敗，跳過: {e}")
                continue

        if updates:
            await self._cache.set_many(updates)
            logger.debug(f"Massive: 更新 {len(updates)}/{len(self._tickers)} 個 ticker")
```

---

## 8. 工廠函式（create_market_provider）

```python
# backend/app/market/factory.py

import logging
import os

from .base import MarketDataProvider
from .cache import PriceCache
from .massive_provider import MassiveProvider
from .simulator_provider import SimulatorProvider

logger = logging.getLogger(__name__)

# 預設觀察清單（與資料庫種子資料保持一致）
DEFAULT_TICKERS = [
    "AAPL", "GOOGL", "MSFT", "AMZN", "TSLA",
    "NVDA", "META", "JPM", "V", "NFLX",
]


def create_market_provider(cache: PriceCache) -> MarketDataProvider:
    """
    依環境變數決定使用 MassiveProvider 或 SimulatorProvider。

    決策邏輯：
        MASSIVE_API_KEY 已設定且非空 → MassiveProvider
        其他情況                     → SimulatorProvider

    注意：此函式回傳已設定初始 tickers 但尚未 start() 的 Provider。
    呼叫方（FastAPI lifespan）負責在適當時機呼叫 start() 與 stop()。

    若之後資料庫中的觀察清單與 DEFAULT_TICKERS 不同，
    watchlist router 在啟動時應呼叫 provider.add_ticker() 同步。

    Returns:
        已設定 _tickers 的 MarketDataProvider 實例。
    """
    api_key = os.environ.get("MASSIVE_API_KEY", "").strip()

    if api_key:
        poll_interval = float(os.environ.get("POLL_INTERVAL_SEC", "15.0"))
        provider: MarketDataProvider = MassiveProvider(
            api_key=api_key,
            cache=cache,
            poll_interval=poll_interval,
        )
        provider_name = f"MassiveProvider (poll={poll_interval}s)"
    else:
        provider = SimulatorProvider(cache=cache)
        provider_name = "SimulatorProvider"

    # 設定初始 tickers（工廠階段同步設定，start() 前完成）
    provider._tickers = set(DEFAULT_TICKERS)

    logger.info(
        f"市場資料提供者: {provider_name}，"
        f"初始 tickers: {sorted(DEFAULT_TICKERS)}"
    )
    return provider
```

---

## 9. FastAPI 整合與生命週期

```python
# backend/app/main.py

import logging
from contextlib import asynccontextmanager

from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles

from .market.cache import PriceCache
from .market.factory import create_market_provider
from .routers import stream, market, watchlist, portfolio, chat

logger = logging.getLogger(__name__)

# ── 應用程式層級的共享狀態 ────────────────────────────────────
# 這兩個物件在 lifespan 中初始化，透過 app.state 傳遞給 router

price_cache = PriceCache()
market_provider = create_market_provider(price_cache)


@asynccontextmanager
async def lifespan(app: FastAPI):
    """
    管理市場資料 Provider 的生命週期。

    startup：
    - 啟動 Provider 背景任務
    - 若資料庫觀察清單與預設不同，同步 tickers

    shutdown：
    - 確保 Provider 乾淨停止（取消任務、關閉 HTTP client）
    """
    # ── Startup ────────────────────────────────────────────────
    try:
        await market_provider.start()
        logger.info("市場資料 Provider 已啟動")
    except Exception as e:
        # Provider 啟動失敗時不崩潰，仍允許應用繼續服務靜態頁面
        logger.critical(f"市場資料 Provider 啟動失敗: {e}", exc_info=True)

    # 將共享物件附加到 app.state 供 router 存取
    app.state.price_cache = price_cache
    app.state.market_provider = market_provider

    yield  # ← 應用正在執行

    # ── Shutdown ───────────────────────────────────────────────
    try:
        await market_provider.stop()
        logger.info("市場資料 Provider 已停止")
    except Exception as e:
        logger.error(f"市場資料 Provider 停止時發生錯誤: {e}", exc_info=True)


app = FastAPI(
    title="FinAlly API",
    lifespan=lifespan,
)

# 掛載 API 路由
app.include_router(stream.router)
app.include_router(market.router)
app.include_router(watchlist.router)
app.include_router(portfolio.router)
app.include_router(chat.router)

# 掛載靜態前端（Next.js export），必須在 API 路由之後
app.mount("/", StaticFiles(directory="static", html=True), name="frontend")
```

---

## 10. SSE 串流端點

### 10.1 SSE 事件規格

所有 SSE 事件都是 `data: <JSON>\n\n` 格式的純文字，`EventSource` API 透過 `event.data` 存取。

#### 事件類型 A：價格批次更新（`price_batch`）

每 ~500ms 推送一次。Payload 為所有當前 ticker 的價格字典。

```
data: {"type":"price_batch","prices":{"AAPL":{"ticker":"AAPL","price":189.45,"prev_price":189.30,"change_pct":0.82,"direction":"up","timestamp":"2026-05-25T10:00:00.123Z"},"MSFT":{"ticker":"MSFT","price":415.20,"prev_price":415.50,"change_pct":-0.05,"direction":"down","timestamp":"2026-05-25T10:00:00.123Z"}}}\n\n
```

**Payload 結構（TypeScript 型別）：**

```typescript
interface PriceBatchEvent {
  type: "price_batch";
  prices: {
    [ticker: string]: {
      ticker: string;
      price: number;
      prev_price: number;
      change_pct: number;   // 相對前日收盤的百分比，如 0.82 = +0.82%
      direction: "up" | "down" | "flat";
      timestamp: string;    // ISO 8601 格式
    };
  };
}
```

#### 事件類型 B：Ticker 移除通知（`ticker_removed`）

當使用者從觀察清單移除 ticker 時，立即推送此事件。前端收到後應清空該 ticker 的 sparkline 資料並從 UI 移除。

```
data: {"type":"ticker_removed","ticker":"TSLA"}\n\n
```

**Payload 結構：**

```typescript
interface TickerRemovedEvent {
  type: "ticker_removed";
  ticker: string;
}
```

#### 心跳（每 ~1 秒，無更新時）

無事件時 SSE 生成器會傳送空行作為心跳，維持連線存活。這透過 `wait_for_update(timeout=1.0)` 的 timeout 機制自動發生——timeout 時 SSE 仍會推送當前快取，確保前端每秒至少收到一次資料。

### 10.2 SSE 生成器實作

```python
# backend/app/routers/stream.py

import json
import logging
from typing import AsyncIterator

from fastapi import APIRouter, Request
from fastapi.responses import StreamingResponse

from ..market.cache import PriceCache

router = APIRouter()
logger = logging.getLogger(__name__)


async def price_event_generator(
    cache: PriceCache,
    request: Request,
) -> AsyncIterator[str]:
    """
    SSE 事件生成器。

    每次 PriceCache 有更新（或最多每 1 秒）推送一次 price_batch 事件。
    使用 request.is_disconnected() 檢測客戶端斷線以乾淨退出。

    設計說明：
    - wait_for_update(timeout=1.0) 確保每秒最少一次推送（心跳兼更新）
    - 即使 timeout 也推送（可能有 watchlist 變動導致的增量）
    - 各 SSE 客戶端互相獨立，一個斷線不影響其他客戶端
    """
    logger.info("SSE 客戶端已連線")
    try:
        while True:
            # 檢測客戶端是否已中斷連線
            if await request.is_disconnected():
                logger.info("SSE 客戶端已中斷連線")
                break

            # 等待快取更新（最多 1 秒）
            await cache.wait_for_update(timeout=1.0)

            prices = await cache.get_all()
            if not prices:
                continue

            payload = {
                "type": "price_batch",
                "prices": {
                    ticker: {
                        "ticker": update.ticker,
                        "price": update.price,
                        "prev_price": update.prev_price,
                        "change_pct": update.change_pct,
                        "direction": update.direction,
                        "timestamp": update.timestamp.isoformat() + "Z",
                    }
                    for ticker, update in prices.items()
                },
            }
            yield f"data: {json.dumps(payload, separators=(',', ':'))}\n\n"

    except Exception as e:
        logger.error(f"SSE 生成器異常: {e}", exc_info=True)
    finally:
        logger.info("SSE 連線已關閉")


@router.get("/api/stream/prices")
async def stream_prices(request: Request) -> StreamingResponse:
    """
    GET /api/stream/prices

    長連線 SSE 端點。客戶端使用 EventSource API 連線。
    EventSource 內建斷線重連機制，不需額外處理。

    Headers：
    - Content-Type: text/event-stream
    - Cache-Control: no-cache（避免代理快取）
    - X-Accel-Buffering: no（停用 nginx 的 response 緩衝）
    """
    cache: PriceCache = request.app.state.price_cache
    return StreamingResponse(
        price_event_generator(cache, request),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "X-Accel-Buffering": "no",
            "Connection": "keep-alive",
        },
    )
```

### 10.3 前端 EventSource 整合（TypeScript 參考）

```typescript
// frontend/hooks/usePriceStream.ts（概念示意，非完整實作）

type PriceData = {
  price: number;
  prev_price: number;
  change_pct: number;
  direction: "up" | "down" | "flat";
  timestamp: string;
};

function usePriceStream() {
  useEffect(() => {
    const es = new EventSource("/api/stream/prices");

    es.onmessage = (event) => {
      const data = JSON.parse(event.data);

      if (data.type === "price_batch") {
        // 批次更新所有 ticker 的價格
        Object.entries(data.prices as Record<string, PriceData>).forEach(
          ([ticker, priceData]) => {
            updatePrice(ticker, priceData);
            appendSparklinePoint(ticker, priceData.price);
          }
        );
      } else if (data.type === "ticker_removed") {
        // 清除被移除 ticker 的所有資料
        clearTickerData(data.ticker);
      }
    };

    es.onerror = () => {
      // EventSource 會自動重連，此處可更新連線狀態指示器
      setConnectionStatus("reconnecting");
    };

    es.onopen = () => {
      setConnectionStatus("connected");
    };

    return () => es.close();
  }, []);
}
```

---

## 11. 歷史價格 REST 端點

```python
# backend/app/routers/market.py

from fastapi import APIRouter, HTTPException, Request
from pydantic import BaseModel
from typing import Optional

from ..market.base import MarketDataProvider
from ..market.models import PriceBar

router = APIRouter()


class PriceBarResponse(BaseModel):
    timestamp: str    # ISO 8601
    open: float
    high: float
    low: float
    close: float
    volume: float


class PriceHistoryResponse(BaseModel):
    ticker: str
    bars: list[PriceBarResponse]


@router.get(
    "/api/prices/history/{ticker}",
    response_model=PriceHistoryResponse,
    summary="取得歷史 K 線資料",
    description="供主圖表與 sparkline 初始化使用。回傳日線 K 線，時間由舊到新。",
)
async def get_price_history(
    ticker: str,
    request: Request,
    bars: int = 100,
) -> PriceHistoryResponse:
    """
    GET /api/prices/history/{ticker}?bars=100

    Args:
        ticker: 股票代號（大小寫均可）
        bars:   要取得的 K 線數量（預設 100，上限 500）

    Returns:
        PriceHistoryResponse，包含 ticker 與 bars 列表。

    Errors:
        404 — ticker 不在觀察清單中（可視需求改為允許查詢任意 ticker）
        422 — bars 超出範圍
    """
    if bars < 1 or bars > 500:
        raise HTTPException(
            status_code=422,
            detail="bars 必須在 1 到 500 之間",
        )

    provider: MarketDataProvider = request.app.state.market_provider
    ticker = ticker.upper()

    history: list[PriceBar] = await provider.get_price_history(
        ticker=ticker,
        bars=bars,
    )

    return PriceHistoryResponse(
        ticker=ticker,
        bars=[
            PriceBarResponse(
                timestamp=bar.timestamp.isoformat() + "Z",
                open=bar.open,
                high=bar.high,
                low=bar.low,
                close=bar.close,
                volume=bar.volume,
            )
            for bar in history
        ],
    )
```

---

## 12. 觀察清單 API 整合

觀察清單 router 在新增/移除 ticker 時需同步更新 Provider 狀態，以確保 SSE 串流即時反映變更。

```python
# backend/app/routers/watchlist.py（市場資料整合部分）

from fastapi import APIRouter, HTTPException, Request
from pydantic import BaseModel

router = APIRouter()


class WatchlistAddRequest(BaseModel):
    ticker: str


@router.post("/api/watchlist", status_code=201)
async def add_to_watchlist(
    body: WatchlistAddRequest,
    request: Request,
):
    """
    POST /api/watchlist
    Body: {"ticker": "AAPL"}

    1. 將 ticker 寫入 SQLite watchlist 資料表
    2. 呼叫 market_provider.add_ticker() 讓 Provider 立即開始追蹤
    3. Provider 的下一個 tick/poll 就會包含此 ticker 並推送 SSE 更新
    """
    ticker = body.ticker.upper().strip()
    if not ticker or not ticker.isalpha() or len(ticker) > 10:
        raise HTTPException(status_code=422, detail="無效的 ticker 格式")

    # TODO: 寫入資料庫（省略，由 DB 層實作）
    # await db.add_to_watchlist(user_id="default", ticker=ticker)

    # 同步更新 Provider
    provider = request.app.state.market_provider
    await provider.add_ticker(ticker)

    return {"ticker": ticker, "status": "added"}


@router.delete("/api/watchlist/{ticker}", status_code=200)
async def remove_from_watchlist(
    ticker: str,
    request: Request,
):
    """
    DELETE /api/watchlist/{ticker}

    1. 從 SQLite watchlist 資料表移除
    2. 呼叫 market_provider.remove_ticker()：停止追蹤並清除 PriceCache
    3. SSE 生成器的下一次推送中，此 ticker 不再出現
    4. 同時推送 ticker_removed SSE 事件（見下方）
    """
    ticker = ticker.upper()

    # TODO: 從資料庫移除
    # await db.remove_from_watchlist(user_id="default", ticker=ticker)

    provider = request.app.state.market_provider
    await provider.remove_ticker(ticker)

    # 廣播 ticker_removed 事件（透過 PriceCache 的特殊機制或獨立 channel）
    # 目前設計：price_cache 移除後，下一次 SSE 推送自動不包含該 ticker；
    # ticker_removed 事件由前端偵測「消失的 ticker」來清除 sparkline。
    # 若需即時通知，可在 PriceCache 中加入 removal_events 佇列。

    return {"ticker": ticker, "status": "removed"}
```

### 觀察清單同步啟動邏輯

若資料庫中的觀察清單與 `DEFAULT_TICKERS` 不同（使用者先前新增了自訂 ticker），需在應用啟動時同步：

```python
# backend/app/main.py（lifespan 補充）

@asynccontextmanager
async def lifespan(app: FastAPI):
    await market_provider.start()

    # 從資料庫載入觀察清單，同步 Provider
    # db_tickers = await db.get_watchlist_tickers(user_id="default")
    # for ticker in db_tickers:
    #     await market_provider.add_ticker(ticker)

    app.state.price_cache = price_cache
    app.state.market_provider = market_provider

    yield

    await market_provider.stop()
```

---

## 13. 環境變數參考

| 變數名稱 | 說明 | 預設值 | 必填 |
|---------|------|--------|------|
| `MASSIVE_API_KEY` | Massive API Key；設定後使用真實資料 | `""` | 否 |
| `POLL_INTERVAL_SEC` | Massive 輪詢間隔（秒）；建議：免費=15，付費=2-5 | `15.0` | 否 |
| `OPENROUTER_API_KEY` | OpenRouter API Key（LLM 功能） | — | 是 |
| `LLM_MOCK` | `"true"` 時使用 mock LLM 回應（E2E 測試） | `"false"` | 否 |

---

## 14. 測試策略

### 14.1 單元測試（backend/tests/）

```python
# backend/tests/test_simulator.py

import asyncio
import math
import pytest
from app.market.simulator_provider import SimulatorProvider, DEFAULT_TICKER_CONFIGS
from app.market.cache import PriceCache


@pytest.mark.asyncio
async def test_gbm_prices_always_positive():
    """GBM 模擬 100 個 tick，價格永遠大於 0。"""
    cache = PriceCache()
    sim = SimulatorProvider(cache)
    await sim.add_ticker("AAPL")
    await sim.start()
    await asyncio.sleep(2.0)  # 等待約 4 個 tick
    await sim.stop()

    prices = await cache.get_all()
    assert "AAPL" in prices
    assert prices["AAPL"].price > 0


@pytest.mark.asyncio
async def test_price_history_deterministic():
    """相同 ticker 每次呼叫 get_price_history 產生相同結果。"""
    cache = PriceCache()
    sim = SimulatorProvider(cache)
    await sim.add_ticker("AAPL")
    await sim.start()

    h1 = await sim.get_price_history("AAPL", bars=50)
    h2 = await sim.get_price_history("AAPL", bars=50)

    await sim.stop()

    assert [b.close for b in h1] == [b.close for b in h2]


@pytest.mark.asyncio
async def test_price_history_sorted_asc():
    """歷史 K 線時間由舊到新排序。"""
    cache = PriceCache()
    sim = SimulatorProvider(cache)
    await sim.add_ticker("MSFT")
    await sim.start()

    bars = await sim.get_price_history("MSFT", bars=30)
    await sim.stop()

    timestamps = [b.timestamp for b in bars]
    assert timestamps == sorted(timestamps)


@pytest.mark.asyncio
async def test_price_bar_ohlc_invariants():
    """每根 K 線滿足 low ≤ open, close ≤ high。"""
    cache = PriceCache()
    sim = SimulatorProvider(cache)
    await sim.add_ticker("NVDA")
    await sim.start()

    bars = await sim.get_price_history("NVDA", bars=50)
    await sim.stop()

    for bar in bars:
        assert bar.low <= bar.open, f"low > open: {bar}"
        assert bar.low <= bar.close, f"low > close: {bar}"
        assert bar.high >= bar.open, f"high < open: {bar}"
        assert bar.high >= bar.close, f"high < close: {bar}"


@pytest.mark.asyncio
async def test_sector_correlation():
    """
    同板塊 ticker（AAPL、MSFT）方向一致的比例應高於 50%
    （即相關係數顯著大於 0）。
    """
    cache = PriceCache()
    sim = SimulatorProvider(cache)
    for ticker in ["AAPL", "MSFT"]:
        await sim.add_ticker(ticker)
    await sim.start()

    samples = []
    for _ in range(50):
        await asyncio.sleep(0.5)
        snap = await cache.get_all()
        if "AAPL" in snap and "MSFT" in snap:
            samples.append(
                snap["AAPL"].direction == snap["MSFT"].direction
            )

    await sim.stop()

    if samples:
        same_pct = sum(samples) / len(samples)
        assert same_pct > 0.5, f"同方向比例 {same_pct:.2%} 未達預期"


def test_unknown_ticker_defaults():
    """未知 ticker 使用合理的預設設定。"""
    cache = PriceCache()
    sim = SimulatorProvider(cache)
    cfg = sim._get_or_default_config("UNKNOWN_XYZ")

    assert cfg.seed_price > 0
    assert 0 < cfg.sigma < 1
    assert cfg.mu > 0
    assert cfg.ticker == "UNKNOWN_XYZ"


@pytest.mark.asyncio
async def test_add_remove_ticker_dynamic():
    """動態新增/移除 ticker 正確影響 Provider 狀態與快取。"""
    cache = PriceCache()
    sim = SimulatorProvider(cache)
    await sim.add_ticker("AAPL")
    await sim.start()
    await asyncio.sleep(0.6)

    # 新增 ticker
    await sim.add_ticker("TSLA")
    await asyncio.sleep(0.6)
    snap = await cache.get_all()
    assert "TSLA" in snap

    # 移除 ticker
    await sim.remove_ticker("TSLA")
    snap = await cache.get_all()
    assert "TSLA" not in snap

    await sim.stop()
```

```python
# backend/tests/test_price_cache.py

import asyncio
import pytest
from app.market.cache import PriceCache
from app.market.models import PriceUpdate
from datetime import datetime


def make_update(ticker: str, price: float) -> PriceUpdate:
    return PriceUpdate.from_prices(ticker, price, price * 0.99)


@pytest.mark.asyncio
async def test_set_and_get():
    cache = PriceCache()
    update = make_update("AAPL", 190.0)
    await cache.set("AAPL", update)

    result = await cache.get("AAPL")
    assert result is not None
    assert result.price == 190.0


@pytest.mark.asyncio
async def test_remove():
    cache = PriceCache()
    await cache.set("AAPL", make_update("AAPL", 190.0))
    await cache.remove("AAPL")

    result = await cache.get("AAPL")
    assert result is None


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
async def test_wait_for_update_timeout():
    cache = PriceCache()
    result = await cache.wait_for_update(timeout=0.1)
    assert result is False
```

```python
# backend/tests/test_massive_provider.py

import pytest
from unittest.mock import AsyncMock, MagicMock, patch
from app.market.massive_provider import MassiveProvider
from app.market.cache import PriceCache


@pytest.mark.asyncio
async def test_parse_valid_snapshot():
    """正常快照回應應正確解析並更新快取。"""
    cache = PriceCache()
    provider = MassiveProvider(api_key="test", cache=cache)
    await provider.add_ticker("AAPL")

    mock_response = MagicMock()
    mock_response.json.return_value = {
        "status": "OK",
        "tickers": [{
            "ticker": "AAPL",
            "lastTrade": {"p": 189.50},
            "prevDay": {"c": 188.00},
        }],
    }
    mock_response.raise_for_status = MagicMock()

    with patch.object(provider, "_client") as mock_client:
        mock_client.get = AsyncMock(return_value=mock_response)
        await provider._fetch_and_update()

    result = await cache.get("AAPL")
    assert result is not None
    assert result.price == 189.50
    assert result.prev_close == 188.00


@pytest.mark.asyncio
async def test_missing_price_skipped():
    """缺少 price 的 ticker 應跳過，不影響其他 ticker。"""
    cache = PriceCache()
    provider = MassiveProvider(api_key="test", cache=cache)
    await provider.add_ticker("AAPL")
    await provider.add_ticker("MSFT")

    mock_response = MagicMock()
    mock_response.json.return_value = {
        "status": "OK",
        "tickers": [
            {"ticker": "AAPL", "lastTrade": {"p": 189.50}},
            {"ticker": "MSFT"},  # 缺少 price，應被跳過
        ],
    }
    mock_response.raise_for_status = MagicMock()

    with patch.object(provider, "_client") as mock_client:
        mock_client.get = AsyncMock(return_value=mock_response)
        await provider._fetch_and_update()

    aapl = await cache.get("AAPL")
    msft = await cache.get("MSFT")

    assert aapl is not None
    assert msft is None  # MSFT 因無 price 被跳過


@pytest.mark.asyncio
async def test_interface_conformance():
    """MassiveProvider 實作 MarketDataProvider 介面的所有方法。"""
    from app.market.base import MarketDataProvider
    cache = PriceCache()
    provider = MassiveProvider(api_key="test", cache=cache)
    assert isinstance(provider, MarketDataProvider)
```

---

## 15. 參數調整指南

### 模擬器調整

| 參數 | 影響 | 建議範圍 | 備註 |
|------|------|---------|------|
| `sigma` | 價格震盪程度 | 穩健股 0.15–0.25；科技股 0.25–0.45；高波動 0.45+ | 影響最大，優先調整 |
| `mu` | 長期趨勢偏向 | 0.04–0.12 | 短期展示影響不大 |
| `UPDATE_INTERVAL_MS` | 更新頻率 | 250–1000 | 低於 250 可能影響效能 |
| `JUMP_PROBABILITY` | 突發事件頻率 | 0.003–0.01 | 0.005 ≈ 每 3.3 分鐘/ticker 一次 |
| `JUMP_MIN/MAX_PCT` | 突發事件幅度 | 2%–5% | 過大會讓圖表失真 |
| `rho`（板塊相關） | 同類股聯動程度 | 0.4–0.7 | 超過 0.8 所有股票動作太相似 |

### Massive API 調整

| 環境變數 | 建議值 | 說明 |
|---------|--------|------|
| `POLL_INTERVAL_SEC=15` | 免費方案 | 每分鐘 4 次，留有餘裕 |
| `POLL_INTERVAL_SEC=5` | Starter/Developer | 更新更頻繁，UI 更流暢 |
| `POLL_INTERVAL_SEC=2` | Advanced/Business | 近似即時 |

### 效能考量

- `PriceCache` 使用 `asyncio.Lock` 保護，不阻塞 FastAPI event loop
- `set_many()` 一次更新所有 ticker，比逐一 `set()` 減少 lock 競爭
- SSE 生成器以事件驅動（`wait_for_update`）而非 busy-loop，CPU 使用率極低
- 每個 SSE 客戶端獨立連線，互不影響（FastAPI 的 async generator 天然並行）
