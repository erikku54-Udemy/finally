# 市場資料後端 — 程式碼審查報告

**審查日期：** 2026-05-27
**審查範圍：** `backend/app/market/`（8 個原始檔）與 `backend/tests/market/`（6 個測試檔）
**審查分支：** `market-data-review`
**Python 版本：** 3.12.3
**pytest 版本：** 9.0.3

---

## 1. 測試結果摘要

**共收集 73 個測試，68 個通過，5 個失敗。**

```
PASSED  tests/market/test_cache.py           13/13
PASSED  tests/market/test_factory.py          7/7
FAILED  tests/market/test_massive.py          8/13  ← 5 failures
PASSED  tests/market/test_models.py          11/11
PASSED  tests/market/test_simulator.py       19/19
PASSED  tests/market/test_simulator_source.py  9/9
```

### 1.1 失敗測試與根本原因分析

所有 5 個失敗都在 `test_massive.py` 中，分為**兩個獨立的根本原因**：

#### 根本原因 A：`_client` 未初始化（影響 3 個測試）

**失敗的測試：**
- `test_poll_updates_cache`
- `test_malformed_snapshot_skipped`
- `test_timestamp_conversion`

**原因：** 這三個測試直接設定 `source._tickers`，然後呼叫 `source._poll_once()` 而不先呼叫 `start()`。`_client` 屬性在建構時初始化為 `None`，只有在 `start()` 中才被賦值。`_poll_once()` 開頭有防護條件：

```python
# massive_client.py:93
async def _poll_once(self) -> None:
    if not self._tickers or not self._client:  # ← _client is None → 直接 return
        return
```

因此方法在呼叫 `_fetch_snapshots` 之前就已返回，即使 `_fetch_snapshots` 已被 mock，快取也不會更新。

**修正方式：** 在呼叫 `_poll_once()` 前，在測試中加入 `source._client = MagicMock()`：
```python
source._client = MagicMock()  # 加入這行
with patch.object(source, "_fetch_snapshots", return_value=mock_snapshots):
    await source._poll_once()
```

#### 根本原因 B：模組層級無法 patch 的惰性匯入（影響 2 個測試）

**失敗的測試：**
- `test_stop_cancels_task`
- `test_start_immediate_poll`

**原因：** 兩個測試皆使用 `patch("app.market.massive_client.RESTClient")`，但 `RESTClient` 是在 `start()` 方法內部惰性匯入的（`from massive import RESTClient`），不存在於模組層級命名空間，因此 `patch()` 拋出 `AttributeError`：

```
AttributeError: <module 'app.market.massive_client'> does not have the attribute 'RESTClient'
```

**修正方式：** 有兩個可行方案：

方案 1（最小改動）：在 patch 呼叫中加入 `create=True`：
```python
with patch("app.market.massive_client.RESTClient", create=True):
```

方案 2（更清晰）：改為在匯入源頭 patch：
```python
with patch("massive.RESTClient"):
```

### 1.2 覆蓋率報告

```
Name                           Stmts   Miss  Cover   Missing
------------------------------------------------------------
app/__init__.py                    0      0   100%
app/market/__init__.py             6      0   100%
app/market/cache.py               39      0   100%
app/market/factory.py             15      0   100%
app/market/interface.py           13      0   100%
app/market/models.py              26      0   100%
app/market/seed_prices.py          9      0   100%
app/market/simulator.py          137      3    98%   L145, L264-265
app/market/massive_client.py      68     30    56%   L42-51, 59-63, 87-89, 96-121, 127-129
app/market/stream.py              35     24    31%   L25-47, 61-86
------------------------------------------------------------
TOTAL                            348     57    84%
```

核心業務邏輯（models、cache、factory、interface、seed_prices）達到 100% 覆蓋率。`massive_client.py` 的 56% 與 `stream.py` 的 31% 屬於預期結果（分別需要真實 API 連線與執行中的 ASGI 伺服器）。

---

## 2. 架構評估

市場資料子系統的整體設計品質良好，採用清晰的策略模式：

```
MarketDataSource (ABC)
├── SimulatorDataSource  ← GBM 模擬器
└── MassiveDataSource    ← Polygon.io REST 輪詢器
        │
        ▼
   PriceCache（共用、執行緒安全）
        │
        ▼
   SSE 串流 → 前端
```

**主要優點：**
- 8 個聚焦的模組，關注點分離清晰
- 工廠模式搭配惰性匯入：只有在設定 `MASSIVE_API_KEY` 時才需要 `massive` 套件
- 以 `PriceCache` 作為單一事實來源，生產者與消費者解耦
- `PriceUpdate` 使用 `frozen=True, slots=True` 的不可變 dataclass，正確且高效
- GBM 數學實作正確：透過 `exp((mu - 0.5*sigma²)*dt + sigma*sqrt(dt)*Z)` 生成對數正態價格路徑
- 使用 Cholesky 分解實現相關移動，對模擬真實感是加分的做法
- 所有背景任務均可正確取消，`stop()` 具有等冪性

---

## 3. 發現的問題

### 3.1 建置設定錯誤（嚴重性：高）

`pyproject.toml` 缺少 hatchling 套件探索設定。執行 `uv sync` 會失敗：

```
ValueError: Unable to determine which files to ship inside the wheel
The most likely cause is that there is no directory that matches
the name of your project (finally_backend).
```

**必要的修正：** 在 `pyproject.toml` 中加入：
```toml
[tool.hatch.build.targets.wheel]
packages = ["app"]
```

此問題會阻擋 Docker 建置與任何全新的 `uv sync`，直到修正為止。目前的臨時解決方案是 `uv sync --no-install-project`，但這在 CI 和 Docker 環境中行不通。

---

### 3.2 MassiveDataSource 測試不健全（嚴重性：中）

如第 1.1 節詳述，5 個測試因為兩個不同的測試設定問題而失敗。這些是測試本身的問題，而非生產程式碼的邏輯錯誤：

**根本原因 A（3 個測試）：** 未在測試中初始化 `_client`，導致 `_poll_once()` 提前返回。

**根本原因 B（2 個測試）：** `patch("app.market.massive_client.RESTClient")` 因為惰性匯入而失敗。

這些修正都很直觀，預計不需要更改生產程式碼。

---

### 3.3 `_generate_events` 回傳型別標注錯誤（嚴重性：低）

`stream.py:54` 宣告回傳型別為 `-> None`，但該函式是非同步生成器（使用了 `yield`）。正確的標注應為：

```python
# 現況（錯誤）
async def _generate_events(...) -> None:

# 應改為
from collections.abc import AsyncGenerator
async def _generate_events(...) -> AsyncGenerator[str, None]:
```

這不會導致執行期問題，但對型別檢查器和開發者有誤導性。

---

### 3.4 `version` 屬性未在鎖定下讀取（嚴重性：低）

`PriceCache.version`（`cache.py:64`）在讀取 `self._version` 時未取得 `self._lock`：

```python
@property
def version(self) -> int:
    return self._version  # ← 無鎖定
```

在使用 GIL 的 CPython 上，讀取單一整數是原子操作，不會產生損毀。然而這與類別其餘部分的寫法不一致；若未來此專案在無 GIL 的 Python 建置（PEP 703，Python 3.13t+）上執行，可能成為競爭條件。

---

### 3.5 `SimulatorDataSource.get_tickers()` 存取私有狀態（嚴重性：低）

`simulator.py:254`：

```python
def get_tickers(self) -> list[str]:
    return list(self._sim._tickers) if self._sim else []
                       ^^^^^^^^^^  ← 存取 GBMSimulator 的私有屬性
```

`SimulatorDataSource` 直接存取 `GBMSimulator._tickers`。`GBMSimulator` 應公開一個 `get_tickers()` 方法或 `tickers` 屬性，以保持物件邊界的清晰性。

---

### 3.6 模組層級 router 實例（嚴重性：低）

`stream.py:16` 建立了一個模組層級的 `router` 物件，而 `create_stream_router()` 透過閉包在其上註冊路由：

```python
router = APIRouter(...)   # 模組層級

def create_stream_router(price_cache: PriceCache) -> APIRouter:
    @router.get("/prices")    # 每次呼叫都在同一個 router 上加入路由
    async def stream_prices(...):
        ...
    return router
```

若 `create_stream_router` 被呼叫兩次（例如在測試中），`/prices` 路由會被重複註冊在同一個 router 上。在實際應用中，此函式僅在應用啟動時呼叫一次，所以不會觸發，但這是一個潛在的問題。

---

### 3.7 `DEFAULT_CORR` 常數未被引用（嚴重性：微小）

`seed_prices.py:48` 定義了 `DEFAULT_CORR = 0.3`，但 `GBMSimulator._pairwise_correlation()` 從未引用它。程式碼中的後備路徑使用的是 `CROSS_GROUP_CORR`（也是 0.3），因此行為是正確的，但命名上有誤導性：`DEFAULT_CORR` 看起來是為「不在任何分組中的 ticker」設計的，但實際上程式碼對所有不匹配的配對都回傳 `CROSS_GROUP_CORR`。

---

### 3.8 測試中未使用的匯入（嚴重性：微小）

以下 lint 警告（`ruff` 可偵測）：

| 檔案 | 未使用的匯入 |
|---|---|
| `test_cache.py` | `pytest` |
| `test_factory.py` | `pytest` |
| `test_massive.py` | `asyncio` |
| `test_simulator.py` | `math`, `pytest` |

---

### 3.9 `conftest.py` 使用已棄用的 `event_loop_policy` fixture（嚴重性：微小）

`conftest.py` 定義了 `event_loop_policy` fixture，而 pytest-asyncio 在每次測試執行時都會警告：

```
PytestDeprecationWarning: Overriding the "event_loop_policy" fixture is deprecated
and will be removed in a future version of pytest-asyncio.
Use the "pytest_asyncio_loop_factories" hook to customize event loop creation.
```

---

## 4. 設計觀察

### 4.1 做得好的部分

- **GBM 參數調整用心。** TSLA 的 sigma=0.50 對比 V 的 0.17，反映了真實世界的波動性差異。衝擊事件系統（每個 tick 約 0.1% 機率，每約 50 秒產生一次可見移動）增加了視覺戲劇性而不破壞價格穩定性。

- **使用 Cholesky 分解實現相關移動** 是數學上正確的方式。以 sector 為基礎的相關性結構（科技股 0.6、金融股 0.5、跨產業 0.3）合理。

- **兩個資料源都有防禦性錯誤處理。** `_run_loop`（模擬器）和 `_poll_once`/`_poll_loop`（Massive）都會捕獲異常並繼續執行，這對長時間運行的背景服務至關重要。

- **SSE 實作整潔。** 基於版本號的變更偵測避免發送重複的 payload。`retry: 1000\n\n` 指令確保瀏覽器自動重連。主動禁用 nginx 緩衝。

- **啟動時即將種子價格寫入快取**，這意味著前端在第一次 SSE 輪詢時就能獲得資料，沒有可見的延遲。

- **使用 `Lock` 的執行緒安全快取** 是正確的選擇，因為 Massive 客戶端透過 `asyncio.to_thread` 運行 API 呼叫。

### 4.2 缺失的測試

- **SSE 串流（`stream.py`）** 覆蓋率僅 31%，沒有專用測試。測試 SSE 需要 ASGI 測試客戶端（例如 `httpx.AsyncClient` 搭配 app）。此為 `PriceCache` 的主要消費者，即使是基本的整合測試也能增加信心。

- **PriceCache 沒有並行/執行緒安全測試。** 鎖的使用從審查看起來正確，但一個多執行緒同時寫入的測試能做實證驗證。

- **沒有使用全部 10 個預設 ticker 的 `GBMSimulator` 測試。** 測試僅使用 1-2 個 ticker。一個確認 Cholesky 分解在完整 10 ticker 預設集合中成功的測試，將能捕獲相關矩陣問題（例如非正定矩陣）。

---

## 5. 各模組快速檢視

| 模組 | 狀態 | 備註 |
|---|---|---|
| `models.py` | ✅ 良好 | 不可變 dataclass，100% 測試覆蓋 |
| `cache.py` | ✅ 良好 | 執行緒安全，100% 覆蓋；`version` 屬性可選擇加鎖 |
| `interface.py` | ✅ 良好 | 清晰的 ABC 合約，文件完整 |
| `seed_prices.py` | ✅ 良好 | 數值合理；`DEFAULT_CORR` 命名可改善 |
| `factory.py` | ✅ 良好 | 選擇邏輯簡潔正確 |
| `simulator.py` | ✅ 良好 | GBM 數學正確；`get_tickers()` 跨越私有邊界 |
| `massive_client.py` | ⚠️ 測試問題 | 生產邏輯正確；5 個測試因設定問題失敗 |
| `stream.py` | ⚠️ 標注問題 | 邏輯正確；回傳型別標注錯誤；模組層級 router 問題 |

---

## 6. 結論

市場資料後端結構扎實、設計良好。GBM 模擬器、價格快取、抽象介面、工廠模式和 SSE 串流均能正確運作並遵循良好實踐。此架構將能與應用程式其餘部分順暢整合。

### 繼續推進前必須修正

1. **修正 `pyproject.toml` 建置設定**（`[tool.hatch.build.targets.wheel] packages = ["app"]`）  
   → 否則 Docker 建置與 `uv sync` 將在任何乾淨環境中失敗。

### 應當修正

2. **修正 `test_massive.py` 中的 5 個失敗測試：**
   - 3 個「`_client` 未初始化」測試：加入 `source._client = MagicMock()`
   - 2 個「惰性匯入無法 patch」測試：改用 `patch("massive.RESTClient")` 或加入 `create=True`

3. **修正 `_generate_events` 的回傳型別標注**（從 `-> None` 改為 `-> AsyncGenerator[str, None]`）

4. **移除測試檔案中未使用的匯入**（`test_cache.py`、`test_factory.py`、`test_massive.py`、`test_simulator.py`）

### 優化方向

5. 在 `GBMSimulator` 上新增公開的 `get_tickers()` 方法，以避免 `SimulatorDataSource` 存取私有屬性

6. 新增至少一個 SSE 整合測試（使用 `httpx.AsyncClient` 搭配 `TestClient`）

7. 新增使用全部 10 個 ticker 的 `GBMSimulator` 測試，確保 Cholesky 分解的穩定性

8. 統一 `DEFAULT_CORR` 與 `CROSS_GROUP_CORR` 的命名（或移除未使用的常數）

9. 更新 `conftest.py`，移除已棄用的 `event_loop_policy` fixture
