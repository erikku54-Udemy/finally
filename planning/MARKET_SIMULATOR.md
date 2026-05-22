# 市場模擬器實作文件

當 `MASSIVE_API_KEY` 未設定時，FinAlly 使用內建模擬器產生擬真的股票價格資料。模擬器實作 `MarketDataProvider` 介面（見 `MARKET_INTERFACE.md`），下游程式碼無需任何變更即可切換資料來源。

---

## 1. 模擬方法：幾何布朗運動（GBM）

模擬器採用**幾何布朗運動**（Geometric Brownian Motion）產生股票價格。GBM 是金融學中最常用的股票價格模型，滿足以下特性：

- 價格永遠為正數（不會出現負值）
- 價格變動呈對數常態分布
- 支援設定各 ticker 的漂移率（drift）與波動率（volatility）

### GBM 離散化公式

```
S(t+Δt) = S(t) × exp((μ - σ²/2) × Δt + σ × √Δt × Z)
```

其中：
- `S(t)` — 當前價格
- `μ`（mu）— 年化漂移率（期望報酬），例如 0.05 代表 5%/年
- `σ`（sigma）— 年化波動率，例如 0.30 代表 30%/年
- `Δt` — 時間步長（以年為單位），例如 500ms = 500/(252×390×60×1000) 年
- `Z` — 標準常態分布隨機變數 N(0,1)

### 為什麼選 GBM

| 優點 | 說明 |
|------|------|
| 數學簡單 | 只需一個亂數乘法，計算成本極低 |
| 擬真度足夠 | 能產生視覺上合理的價格走勢 |
| 可設定個性 | 每支股票可有不同波動率，科技股波動較大、藍籌股較穩 |
| 易於理解 | 教學示範價值高 |

---

## 2. Ticker 設定

每個 ticker 有以下可設定參數：

```python
@dataclass
class TickerConfig:
    ticker: str
    seed_price: float      # 起始參考價格（接近真實市場水準）
    mu: float              # 年化漂移率（通常為小正數）
    sigma: float           # 年化波動率（科技股高、公用事業低）
    sector: str            # 用於相關性分組
```

### 預設 10 個 Ticker 設定

```python
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

## 3. 相關性模型

真實市場中，同類股票會一起漲跌（例如科技類股同步波動）。模擬器加入**板塊相關性**來重現這個效果：

```
Z_ticker = ρ × Z_sector + √(1 - ρ²) × Z_individual
```

其中：
- `Z_sector` — 板塊共同隨機因子（所有同板塊 ticker 共享）
- `Z_individual` — 個別 ticker 的獨立隨機因子
- `ρ`（rho）— 板塊相關係數（建議值：科技 0.6、金融 0.5、其他 0.4）

---

## 4. 隨機事件（Jumps）

為增加模擬的戲劇感，模擬器加入偶發的突然跳動事件：

- **觸發機率**：每次更新時，每個 ticker 有 0.5% 機率觸發跳動事件
- **跳動幅度**：價格突然變動 ±2% 至 ±5%（均勻分布），方向隨機
- **用途**：模擬突發新聞、財報公告等事件造成的劇烈波動

---

## 5. 完整程式碼實作

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

# 模擬器更新間隔（毫秒）
UPDATE_INTERVAL_MS = 500

# 每年的交易秒數（252 天 × 6.5 小時 × 3600 秒）
TRADING_SECONDS_PER_YEAR = 252 * 6.5 * 3600

# 時間步長（年）：500ms 換算為年
DT = (UPDATE_INTERVAL_MS / 1000) / TRADING_SECONDS_PER_YEAR

# 板塊相關係數
SECTOR_CORRELATION: dict[str, float] = {
    "tech": 0.6,
    "ev": 0.4,
    "finance": 0.5,
    "media": 0.4,
}

# 隨機跳動事件機率
JUMP_PROBABILITY = 0.005
JUMP_MIN_PCT = 0.02
JUMP_MAX_PCT = 0.05


@dataclass
class TickerConfig:
    """單一 ticker 的模擬參數。"""
    ticker: str
    seed_price: float   # 起始價格（接近真實市場）
    mu: float           # 年化漂移率
    sigma: float        # 年化波動率
    sector: str         # 板塊（用於相關性）


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
    - 同板塊 ticker 之間有相關性（科技股一起漲跌）
    - 偶爾發生隨機跳動事件模擬突發新聞
    - 未知 ticker 自動使用合理的預設參數
    """

    def __init__(self, cache: PriceCache) -> None:
        self._cache = cache
        self._tickers: set[str] = set()
        self._current_prices: dict[str, float] = {}   # 目前模擬價格
        self._day_open_prices: dict[str, float] = {}  # 當日開盤價（計算日漲跌幅）
        self._task: asyncio.Task | None = None

    async def start(self) -> None:
        """初始化價格並啟動背景模擬任務。"""
        self._initialize_prices()
        self._task = asyncio.create_task(self._simulation_loop())
        logger.info(f"SimulatorProvider 已啟動，監控 {len(self._tickers)} 個 ticker")

    async def stop(self) -> None:
        """停止模擬任務。"""
        if self._task:
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass
        logger.info("SimulatorProvider 已停止")

    async def add_ticker(self, ticker: str) -> None:
        """新增 ticker，若沒有設定則使用預設值。"""
        ticker = ticker.upper()
        self._tickers.add(ticker)
        if ticker not in self._current_prices:
            config = self._get_or_default_config(ticker)
            self._current_prices[ticker] = config.seed_price
            self._day_open_prices[ticker] = config.seed_price

    async def remove_ticker(self, ticker: str) -> None:
        ticker = ticker.upper()
        self._tickers.discard(ticker)
        self._current_prices.pop(ticker, None)
        self._day_open_prices.pop(ticker, None)
        await self._cache.remove(ticker)

    async def get_latest_prices(self) -> dict[str, PriceUpdate]:
        return await self._cache.get_all()

    async def get_price_history(
        self, ticker: str, bars: int = 100
    ) -> list[PriceBar]:
        """
        回溯模擬歷史 K 線（使用固定 seed 確保可重現性）。

        以目前價格為基準，向前推算 bars 根日線 K 線。
        """
        ticker = ticker.upper()
        config = self._get_or_default_config(ticker)
        current = self._current_prices.get(ticker, config.seed_price)

        # 用固定 seed 回溯，讓圖表看起來有歷史感
        rng = random.Random(hash(ticker) % (2**32))

        # 先產生 bars 根價格序列（由舊到新）
        prices = [current]
        for _ in range(bars - 1):
            z = rng.gauss(0, 1)
            drift = (config.mu - 0.5 * config.sigma ** 2) * (1 / 252)
            diffusion = config.sigma * math.sqrt(1 / 252) * z
            prices.append(prices[-1] * math.exp(drift + diffusion))

        prices.reverse()  # 由舊到新

        result = []
        base_time = datetime.utcnow() - timedelta(days=bars)
        for i, close in enumerate(prices):
            day_sigma = config.sigma * math.sqrt(1 / 252)
            open_ = close * (1 + rng.gauss(0, day_sigma * 0.3))
            high = max(open_, close) * (1 + abs(rng.gauss(0, day_sigma * 0.2)))
            low = min(open_, close) * (1 - abs(rng.gauss(0, day_sigma * 0.2)))
            volume = abs(rng.gauss(10_000_000, 3_000_000))
            result.append(PriceBar(
                timestamp=base_time + timedelta(days=i),
                open=round(open_, 2),
                high=round(high, 2),
                low=round(low, 2),
                close=round(close, 2),
                volume=round(volume),
            ))

        return result

    # ──────────────────────────────────────────
    # 內部實作
    # ──────────────────────────────────────────

    def _initialize_prices(self) -> None:
        """以種子價格初始化所有被觀察 ticker。"""
        for ticker in self._tickers:
            if ticker not in self._current_prices:
                config = self._get_or_default_config(ticker)
                self._current_prices[ticker] = config.seed_price
                self._day_open_prices[ticker] = config.seed_price

    async def _simulation_loop(self) -> None:
        """主模擬迴圈：每 500ms 更新一次所有 ticker 的價格。"""
        while True:
            try:
                await self._tick()
            except asyncio.CancelledError:
                raise
            except Exception as e:
                logger.error(f"模擬器 tick 錯誤: {e}")

            await asyncio.sleep(UPDATE_INTERVAL_MS / 1000)

    async def _tick(self) -> None:
        """
        執行一次 GBM 步進，更新所有 ticker 的價格並寫入快取。

        流程：
        1. 產生各板塊的共同隨機因子（Z_sector）
        2. 對每個 ticker 混合板塊因子與個別因子
        3. 套用 GBM 公式計算新價格
        4. 以 JUMP_PROBABILITY 機率觸發跳動事件
        5. 批次寫入 PriceCache
        """
        if not self._tickers:
            return

        # 步驟 1：各板塊隨機因子
        sector_factors: dict[str, float] = {
            sector: random.gauss(0, 1)
            for sector in SECTOR_CORRELATION
        }

        updates: dict[str, PriceUpdate] = {}

        for ticker in list(self._tickers):
            config = self._get_or_default_config(ticker)
            prev_price = self._current_prices.get(ticker, config.seed_price)

            # 步驟 2：混合板塊與個別隨機因子
            rho = SECTOR_CORRELATION.get(config.sector, 0.4)
            z_sector = sector_factors.get(config.sector, random.gauss(0, 1))
            z_individual = random.gauss(0, 1)
            z = rho * z_sector + math.sqrt(1 - rho ** 2) * z_individual

            # 步驟 3：GBM 價格更新
            drift = (config.mu - 0.5 * config.sigma ** 2) * DT
            diffusion = config.sigma * math.sqrt(DT) * z
            new_price = prev_price * math.exp(drift + diffusion)

            # 步驟 4：隨機跳動事件
            if random.random() < JUMP_PROBABILITY:
                jump_pct = random.uniform(JUMP_MIN_PCT, JUMP_MAX_PCT)
                direction = 1 if random.random() > 0.5 else -1
                new_price *= 1 + direction * jump_pct
                logger.debug(f"[跳動事件] {ticker}: {prev_price:.2f} → {new_price:.2f}")

            new_price = round(max(new_price, 0.01), 2)
            self._current_prices[ticker] = new_price

            prev_close = self._day_open_prices.get(ticker)  # 用開盤價近似前日收盤
            updates[ticker] = PriceUpdate.from_prices(
                ticker=ticker,
                price=new_price,
                prev_price=prev_price,
                prev_close=prev_close,
                timestamp=datetime.utcnow(),
            )

        await self._cache.set_many(updates)

    def _get_or_default_config(self, ticker: str) -> TickerConfig:
        """
        取得 ticker 設定。若不在預設清單中，使用通用預設值。

        未知 ticker 使用保守的中等波動率設定，起始價格 100 美元。
        """
        if ticker in DEFAULT_TICKER_CONFIGS:
            return DEFAULT_TICKER_CONFIGS[ticker]

        # 通用預設值
        return TickerConfig(
            ticker=ticker,
            seed_price=100.0,
            mu=0.07,
            sigma=0.30,
            sector="tech",
        )
```

---

## 6. 歷史資料回溯演算法

`get_price_history()` 使用**確定性回溯**（deterministic lookback）產生歷史 K 線：

1. 以 `hash(ticker)` 作為隨機種子（確保相同 ticker 每次產生相同歷史）
2. 以目前模擬價格為基準，往前推算 N 根日線
3. 每根 K 線：
   - close 由 GBM 決定
   - open/high/low 由 close 加上小幅隨機偏移產生
   - volume 由常態分布產生

**優點**：圖表在頁面重新載入後仍顯示一致的歷史走勢，不會每次都不同。

---

## 7. 測試

### 單元測試重點

```python
# backend/tests/test_simulator.py（提示）

import pytest
import asyncio
from app.market.simulator_provider import SimulatorProvider
from app.market.cache import PriceCache


@pytest.mark.asyncio
async def test_gbm_prices_always_positive():
    """GBM 產生的價格永遠為正數。"""
    cache = PriceCache()
    sim = SimulatorProvider(cache)
    await sim.add_ticker("AAPL")
    await sim.start()
    await asyncio.sleep(2)  # 等待幾個 tick
    await sim.stop()

    prices = await cache.get_all()
    assert "AAPL" in prices
    assert prices["AAPL"].price > 0


@pytest.mark.asyncio
async def test_price_history_deterministic():
    """相同 ticker 每次產生相同歷史。"""
    cache = PriceCache()
    sim = SimulatorProvider(cache)
    await sim.add_ticker("AAPL")
    await sim.start()

    history1 = await sim.get_price_history("AAPL", bars=50)
    history2 = await sim.get_price_history("AAPL", bars=50)

    await sim.stop()

    closes1 = [b.close for b in history1]
    closes2 = [b.close for b in history2]
    assert closes1 == closes2


@pytest.mark.asyncio
async def test_sector_correlation():
    """同板塊 ticker 的價格變動方向有相關性。"""
    cache = PriceCache()
    sim = SimulatorProvider(cache)
    for ticker in ["AAPL", "MSFT", "GOOGL"]:
        await sim.add_ticker(ticker)
    await sim.start()

    snapshots = []
    for _ in range(100):
        await asyncio.sleep(0.5)
        snapshot = await cache.get_all()
        snapshots.append({t: s.direction for t, s in snapshot.items()})

    await sim.stop()

    # 統計同板塊方向一致比例應高於 50%
    same_count = sum(
        1 for s in snapshots
        if s.get("AAPL") == s.get("MSFT")
    )
    assert same_count / len(snapshots) > 0.5


def test_unknown_ticker_uses_defaults():
    """未知 ticker 應使用合理的預設設定。"""
    cache = PriceCache()
    sim = SimulatorProvider(cache)
    config = sim._get_or_default_config("UNKNOWN_TICKER")
    assert config.seed_price > 0
    assert 0 < config.sigma < 1
    assert config.ticker == "UNKNOWN_TICKER"
```

---

## 8. 參數調整指南

| 參數 | 效果 | 建議範圍 |
|------|------|---------|
| `sigma`（波動率）| 越大越震盪 | 穩健股 0.15–0.25；科技股 0.25–0.40；高波動 0.40+ |
| `mu`（漂移）| 長期趨勢方向 | 通常 0.05–0.12；不影響短期展示效果 |
| `UPDATE_INTERVAL_MS` | 更新頻率 | 建議 500ms（與 SSE 推送節奏一致） |
| `JUMP_PROBABILITY` | 突發事件頻率 | 0.003–0.01（每 100–333 次 tick 發生一次）|
| `JUMP_MIN/MAX_PCT` | 突發事件幅度 | 2%–5%（視覺上引人注目但不誇張）|
| `rho`（板塊相關） | 同類股的聯動程度 | 0.4–0.7（超過 0.8 會讓所有股票動作太相似）|
