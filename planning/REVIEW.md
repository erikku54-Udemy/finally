# 自上次 commit 以來的變更審查

## Findings

未發現需要修正的問題。

## 審查範圍

- `.gitignore` 新增 `node_modules/` 忽略規則。
- `planning/PLAN.md` 將 LLM 整合流程中的 skill 名稱統一為 `open-inference skill`。
- `planning/PLAN.md` 補充根目錄 `package.json` 的用途說明。
- 原本的 `planning/REVIEW.md` 被刪除；本檔為本次審查重新產生的回饋。

## Notes

- `.gitignore` 的 `node_modules/` 規則合理，會避免本機安裝輸出被納入版本控制。
- LLM skill 名稱現在和同一節前文一致，修正了先前 `open-inference` 與 `cerebras-inference` 混用的規格風險。
- 根目錄 `package.json` 目前只宣告 `packageManager`，新增說明與實際內容一致。

## Residual Risk

- 本次變更是文件與 ignore 規則更新，沒有可執行程式碼變更；未執行測試。

---

# 完整變更審查（Session 207f4314）

## 變更摘要

本次 session 產生了 **3 份新的市場資料設計文件**（1,403 行），並對 `PLAN.md` 進行了 2 項微調修正。

### 新增檔案

1. **`planning/MASSIVE_API.md`** (407 行)
   - Massive API（前 Polygon.io）基本資訊與免費方案限制說明
   - Python 套件安裝指導
   - 核心端點：全市場快照（`GET /v2/snapshot/.../tickers`）、單一 ticker、統一快照
   - K 線與歷史資料端點說明
   - 完整的錯誤處理程式碼範例
   - 免費方案限制（5 req/min）→ 15 秒輪詢策略

2. **`planning/MARKET_INTERFACE.md`** (527 行)
   - 市場資料層抽象設計
   - `MarketDataProvider` 抽象類別合約
   - `PriceUpdate` / `PriceBar` 資料模型（含完整程式碼）
   - `PriceCache` 共享快取實作（Provider 寫入、SSE 讀取）
   - `MassiveProvider` 完整非同步實作
   - `create_market_provider()` 工廠函式
   - FastAPI lifespan 與 SSE 串流整合範例
   - 目錄結構 `backend/app/market/`

3. **`planning/MARKET_SIMULATOR.md`** (469 行)
   - 幾何布朗運動（GBM）數學公式與推導
   - 時間步長計算詳解
   - 10 個預設 ticker 設定（種子價格接近真實市場）
   - **板塊相關性模型**（科技股聯動）
   - **隨機跳動事件**（每 ticker 每 tick 0.5% 機率 ±2%–5% 劇烈變動）
   - `SimulatorProvider` 完整程式碼
   - 確定性歷史回溯演算法（同 ticker 每次圖表一致）
   - 測試範例 + 參數調整指南

### 修改檔案

1. **`.gitignore`**
   - 新增 `node_modules/` 忽略規則（適用於根目錄最小化 `package.json`）
   - 合理且無副作用

2. **`planning/PLAN.md`**
   - **行 310**：將 `cerebras-inference skill` 改為 `open-inference skill`（與行 299 保持一致）
   - **行 410–411**：新增根目錄 `package.json` 用途說明（僅宣告 `packageManager`，不包含依賴）

## 品質評估

### ✅ 優點

1. **文件完整度高**
   - 三份文件涵蓋 API、抽象層、模擬器，互相參照清晰
   - 包含完整程式碼片段與錯誤處理
   - 提供數學公式與設定參數表

2. **設計思路健全**
   - 工廠模式分離 Provider 實作，支援無縫切換
   - 共享快取與 SSE 解耦，符合反應式架構
   - GBM 模型兼具數學嚴謹性與計算效率

3. **實踐指導具體**
   - 包含環境變數配置、class 簽名、API endpoint
   - 測試範例直接可用

### ⚠️ 需後續補充

1. **`MASSIVE_API.md` 遺漏的細節**
   - Massive REST client 的全市場快照端點完整 request/response schema（只有 fragment）
   - 批次更新時錯誤處理邏輯（某個 ticker 失敗時是否 retry、fallback）
   - 速率限制達到時的重試策略（backoff 參數）

2. **`MARKET_INTERFACE.md` 的整合假設**
   - FastAPI lifespan 範例假設 provider startup 不拋出異常（無 try-except）
   - SSE event serialization format 未詳述（是否直接 JSON 或需 wrapper）
   - `PriceCache` 的 dict 併發存取是否需鎖（asyncio 環境下可能安全，但文件未明確說明）

3. **`MARKET_SIMULATOR.md` 的實作缺口**
   - 板塊相關性（"同一板塊的 ticker 大概率同方向"）的 correlation matrix 定義不足
     - 文件說「科技股一起漲跌」但沒有 matrix
     - 如何決定 ticker A 漲時 ticker B 的機率
   - 種子（seed）機制與確定性回溯的完整演算法流程（pseudo code）未提供

### 🎯 對後續實作的影響

| 層面 | 風險 | 緩解 |
|------|------|------|
| **後端實作** | Provider startup 在 lifespan 中未做異常處理，可能導致應用啟動失敗 | 應在 `create_market_provider()` 和 lifespan context 中加 try-finally |
| **SSE 客戶端** | event type 與 JSON schema 未在文件中明確定義 | 需補充 `snapshot`、`price`、`watchlist_removed` event 的完整 payload |
| **模擬器精度** | 相關性模型不夠具體，不同實作者可能差異大 | 應提供 correlation matrix 與決策樹，或提供參考實作 |
| **Massive 集成** | 免費方案 5 req/min，但未說明多 ticker 批次是否計一次請求 | 需測試 `/v2/snapshot/.../tickers?tickers=AAPL,GOOGL,...` 的實際計數 |

## 對標 PLAN.md 的補完情況

**原 PLAN.md 遺留問題**（見 20250101 版本 REVIEW.md）中的進度：

- ✅ **#2：LLM skill 名稱混用** → 已統一為 `open-inference`
- ✅ **#4：歷史價格 API 資料來源** → `MARKET_INTERFACE.md` 和 `MASSIVE_API.md` 都涵蓋了 history endpoint
- ✅ **#8：靜態檔案路由** → `PLAN.md` 已補充 `package.json` 說明，與根目錄掛載策略更清晰
- ⚠️ **#3：SSE 事件規格** → `MARKET_INTERFACE.md` 提到 SSE，但事件類型與 payload 仍需細化
- ⚠️ **#5：交易 API 契約** → 未在新文件中涵蓋，應補充 `TradeRequest`、`TradeResult` 的完整 schema

## 建議後續行動

1. **馬上做**
   - 在 `MARKET_INTERFACE.md` 補充 SSE event types（`snapshot`, `price`, `watchlist_removed`）的完整 JSON schema
   - 測試 Massive `/v2/snapshot/...` 的批次 request 計數方式，確認免費方案 QPS 計算

2. **實作前做**
   - 在 `MARKET_SIMULATOR.md` 補充相關性 matrix（10×10，科技 vs 金融 vs 消費）與決策邏輯
   - 補充 `MARKET_INTERFACE.md` lifespan exception handling 的完整範例
   - 提供 `MASSIVE_API.md` 的批次錯誤恢復流程

3. **文件統整**
   - 考慮將三份市場文件合併為單一 `planning/MARKET_DESIGN.md`，分三個小節
   - 或新增 `planning/API_CONTRACTS.md` 統一定義所有 request/response/event schema
