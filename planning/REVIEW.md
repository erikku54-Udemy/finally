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

---

# GitHub CI/CD Workflow 添加審查（Commit 5e61d15）

## 變更摘要

本次 commit 新增了 2 個 GitHub Actions 工作流程檔案，用於自動化 Claude Code 集成與代碼審查功能。

### 新增檔案

1. **`.github/workflows/claude-code-review.yml`** (44 行)
   - 自動在 PR 開啟/更新時觸發 Claude Code Review
   - 使用官方 `anthropics/claude-code-action@v1` 動作
   - 觸發事件：`opened`, `synchronize`, `ready_for_review`, `reopened`
   - 執行環境：`ubuntu-latest`
   - 必要權限：`contents`, `pull-requests`, `issues` 讀取權限，`id-token` 寫入
   - 搭配 `CLAUDE_CODE_OAUTH_TOKEN` 密碼進行身份驗證
   - 包含選擇性配置註釋（路徑篩選、PR 作者篩選）

2. **`.github/workflows/claude.yml`** (50 行)
   - 在 PR 評論、Issue 評論、新 Issue 或 PR 審查時觸發 Claude Code
   - 條件判斷：對話中或文本中需包含 `@claude` 標籤
   - 執行環境：`ubuntu-latest`
   - 必要權限：包含 `actions: read`（用於讀取 CI 結果）
   - 支援可選的自定義提示詞（prompt）與自定義參數（claude_args）
   - 包含選擇性工具限制示例註釋

## 品質評估

### ✅ 優點

1. **配置完善**
   - 使用官方認可的 `anthropics/claude-code-action@v1` 動作，版本穩定
   - 權限設置最小化原則（只授予必要的讀寫權限）
   - 明確記錄所有觸發事件與條件邏輯

2. **靈活的擴展空間**
   - 包含多個註釋化的可選配置（路徑篩選、作者篩選、自定義提示詞、工具限制）
   - 便於未來根據項目需求開啟或調整

3. **工作流程清晰**
   - `claude-code-review.yml` 專門用於自動 PR 審查
   - `claude.yml` 用於互動式評論觸發
   - 職責分離合理

### ⚠️ 需注意的項目

1. **密碼與身份驗證**
   - 工作流程依賴 `CLAUDE_CODE_OAUTH_TOKEN` 環境變數
   - **風險**：若此密碼未正確配置在 GitHub Secrets，工作流程將失敗
   - **建議**：確認此密鑰已在 repository settings > Secrets 中配置

2. **`claude.yml` 的 `@claude` 觸發機制**
   - 當前依賴評論/Issue 內容中包含 `@claude` 字串進行判斷
   - **限制**：若誤觸發（例如在評論中提及「@claude」作為名稱而非標籤），可能造成不必要的 workflow 執行
   - **建議**：若頻繁誤觸發，可考慮限制觸發範圍（如 `if: github.event.comment.user.type == 'User'` 等額外篩選）

3. **`claude-code-review.yml` 的檢查運行限制**
   - 工作流程無明確的文件路徑篩選（已被註釋），可能在所有 PR 上都執行 Claude 審查
   - 若項目中文件眾多，持續的自動審查可能增加 API 調用成本
   - **建議**：根據實際需求解除 `paths:` 註釋並設定適當的檔案模式

4. **缺少錯誤處理**
   - 工作流程未對 Claude Code Review 失敗的情況設置容錯（如失敗時自動重試）
   - **影響**：若 Claude Code Review 突然失敗，PR 檢查可能卡住
   - **建議**：可考慮新增 `continue-on-error: true`，或建立獨立的 issue 記錄失敗

## 安全考量

- ✅ 使用了 `id-token: write` 進行短期身份驗證，相對安全
- ⚠️ `CLAUDE_CODE_OAUTH_TOKEN` 是長期有效的密鑰，應定期輪換
- ✅ 未在 workflow 中硬編碼任何密鑰或敏感資訊

## 對後續開發的影響

| 層面 | 影響 | 建議 |
|------|------|------|
| **代碼審查流程** | PR 自動檢查可加速審查週期 | 監測首次執行的性能與成本 |
| **開發者體驗** | `@claude` 互動機制便於詢問建議 | 在團隊文檔中說明使用方式 |
| **成本控制** | 頻繁的自動審查可能增加 API 調用 | 根據實際使用情況調整觸發條件 |
| **密鑰管理** | 依賴 `CLAUDE_CODE_OAUTH_TOKEN` | 建立密鑰輪換政策 |

## 建議後續行動

1. **馬上做**
   - ✅ 確認 `CLAUDE_CODE_OAUTH_TOKEN` 已在 GitHub Secrets 中配置
   - ✅ 測試 `claude-code-review.yml` 在一個非關鍵 PR 上執行，驗證正常運作

2. **可選優化**
   - 在 `claude-code-review.yml` 中解除 `paths:` 註釋，針對特定檔案類型（如 `src/**/*.ts`）
   - 在 `claude.yml` 中新增額外的身份驗證過濾，減少誤觸發
   - 為兩個工作流程新增 `continue-on-error` 或重試邏輯

3. **文件更新**
   - 在 `PLAN.md` 或 `CLAUDE.md` 中記錄 Claude Code 工作流程的啟用與使用指南
   - 為團隊提供「如何在 PR 中使用 `@claude`」的說明文檔

---

# 綜合市場資料設計文件審查（Commit 67c8d74）

## 變更摘要

本次 commit 將先前分散的三份市場資料設計文件（`MARKET_INTERFACE.md`、`MARKET_SIMULATOR.md`、`MASSIVE_API.md`）整合為一份全面的實作藍圖：

### 新增檔案

**`planning/MARKET_DATA_DESIGN.md`** (1,796 行)
- 統一了市場資料架構、提供者實作、價格快取、SSE 流式傳輸和 REST API 的完整設計
- 15 個章節涵蓋從高層架構到參數調整的所有面向

## 章節結構與內容評估

### ✅ 架構與設計強度

| 章節 | 內容 | 評估 |
|------|------|------|
| **架構總覽** | ASCII 資料流圖，Provider → PriceCache → SSE 解耦設計 | 清晰的視覺化展示了系統流向 |
| **資料模型** | `PriceUpdate` / `PriceBar` 含完整欄位說明和 factory 方法 | 型別安全且易於擴展 |
| **PriceCache** | asyncio Lock + Event 的事件驅動快取，`wait_for_update(timeout)` | 避免 busy-loop，適合 asyncio 環境 |
| **MarketDataProvider** | 抽象介面定義生命週期合約（start/stop/add/remove ticker） | 清晰的 provider 契約，便於實作多個子類別 |
| **SimulatorProvider** | GBM 數學公式、板塊相關性矩陣、跳動事件、確定性回溯 | 數學基礎紮實，相關性模型具體化（√(1-ρ²) 公式） |
| **MassiveProvider** | httpx 非同步輪詢、指數退避策略（429 觸發，最大 120s）、單 ticker 隔離 | 生產級別的速率限制處理 |

### ⚠️ 需後續驗證的項目

1. **SSE 事件規格完整度**
   - 文件定義了 `price_batch` 和 `ticker_removed` event schema
   - ✅ 包含完整 JSON payload 範例與 TypeScript 型別
   - ✅ 提供了前端 `EventSource` 客戶端範例
   - **但需測試**：實際 SSE payload 大小、客戶端連線穩定性（特別是 timeout 重連）

2. **FastAPI lifespan 整合**
   - 文件現已包含 startup/shutdown try-except 異常隔離
   - ✅ provider startup 失敗時不會导致應用崩潰
   - **實作需確認**：
     - Provider 啟動失敗時是否應 fallback 到 simulator？
     - 日誌記錄策略（啟動失敗應記錄到何處）

3. **歷史價格 API (`GET /api/prices/history/{ticker}`)**
   - 文件定義了完整的 request/response schema
   - ✅ bars 上限驗證（預設 1,000）
   - ✅ 時間範圍查詢支援
   - **實作缺口**：
     - Massive 批次請求失敗時的恢復策略（部分 ticker 返回失敗）
     - 快取是否應用於歷史數據（防重複調用）

4. **觀察清單整合**
   - 文件說明了 add/remove ticker 時同步 Provider 的模式
   - ✅ 啟動時資料庫同步說明清晰
   - **實作需驗證**：
     - add_ticker 時是否應同步歷史資料（冷啟動延遲）
     - remove_ticker 是否應清除 cache 中該 ticker 的舊資料

5. **環境變數配置**
   - 提供了完整的變數表（預設值、說明、類型）
   - ✅ 包含 Massive API key、simulator 參數等
   - **風險**：未提及 API key 的加密存儲方式（應使用 GitHub Secrets + 運行時注入）

## 數學與演算法驗證

### SimulatorProvider

**GBM 實現** ✅
```
dS/S = μ·dt + σ·dW(t)
```
- 文件提供完整的時間步長計算：Δt = 1/252/market_hours
- 合理的預設參數（μ=0.1, σ=0.25）

**板塊相關性** ✅
- 公式 `ρ×Z_sector + √(1-ρ²)×Z_individual` 正確
- 10×10 相關性矩陣清晰定義（科技股之間 ρ≈0.8, 跨板塊 ≈0.3）

**跳動事件** ✅
- 每 ticker 每 tick 0.5% 機率 ±2%–5% 變動
- 合理的突發行情模擬

**確定性回溯** ✅
- 同 seed 下同 ticker 每次圖表一致
- OHLC 不等式保證：O ≤ min(C,H)，C ≤ max(O,L) 等

### MassiveProvider

**速率限制策略** ✅
- 指數退避：1s, 2s, 4s, ..., 最大 120s
- 合理的重試上限（防止無限重試）

**單 ticker 隔離** ✅
- 某 ticker 解析失敗不影響其他 ticker 的更新
- 可靠性強

## 與前期文件的對標

### 從 Session 207f4314 REVIEW 遺留問題

| 遺留問題 | 狀態 | 備註 |
|---------|------|------|
| SSE event schema 不清楚 | ✅ **已解決** | 文件第 10 章完整定義 `price_batch` / `ticker_removed` |
| lifespan 無異常處理 | ✅ **已解決** | 第 9 章包含 try-except，startup 失敗隔離 |
| 相關性矩陣定義不足 | ✅ **已解決** | 第 5 章明確給出 10×10 矩陣與 √(1-ρ²) 公式 |
| Massive backoff 策略未提 | ✅ **已解決** | 第 6 章詳述指數退避（1s→120s） |
| 批次錯誤恢復流程未提 | ⚠️ **部分** | 有單 ticker 隔離，但批次請求失敗的完整恢復流程未詳述 |

## 對後續實作的風險評估

### 🟢 低風險區域
- ✅ 資料模型設計（type-safe，易實作）
- ✅ Provider 抽象介面清晰
- ✅ GBM 與相關性演算法完備

### 🟡 中等風險區域
- ⚠️ SSE 客戶端連線穩定性（需實際負載測試）
- ⚠️ Massive 批次失敗時的恢復策略（部分 ticker 失敗的場景）
- ⚠️ Provider 切換時的資料連續性（如從 simulator 切換到 Massive）

### 🔴 高風險區域
- ❌ **暫無明顯高風險項**（整體設計紮實）

## 代碼範例質量

- ✅ 完整的 Python 3.10+ asyncio 程式碼
- ✅ 完整的 TypeScript 客戶端範例
- ✅ 完整的測試用例（GBM 不變量、確定性驗證、快取操作）
- ✅ 環境變數範例清晰

## 建議後續行動

### 🔴 必做（實作前）

1. **驗證 Massive 批次計數邏輯**
   - 免費方案 5 req/min，確認 `/v2/snapshot/.../tickers?tickers=AAPL,GOOGL,...` 是否計為 1 次請求
   - 若計為 N 次（tickers 數量），應調整輪詢間隔或批次大小

2. **補充 Provider 切換邏輯**
   - 若啟動時 Massive 失敗，是否應自動 fallback 到 simulator？
   - 需在工廠函式中明確定義此行為

3. **確定歷史資料快取策略**
   - GET `/api/prices/history/{ticker}` 應否快取？如何設定 TTL？
   - 與 PriceCache（實時更新）的關係如何？

### 🟡 可選優化

1. **WebSocket 方案評估**
   - 若 SSE 在生產環境中的連線穩定性不佳，考慮 WebSocket
   - 文件未提及重連邏輯（客戶端應實作指數退避重連）

2. **資料庫同步機制詳化**
   - 啟動時從 DB 同步觀察清單的完整程式碼範例
   - 如何防止啟動時同步的重複調用

3. **度量與監測**
   - 建議補充：SSE latency、Massive API 失敗率、cache hit ratio 等監測指標
   - 建議補充：日誌等級與重要事件的日誌點

### 📚 文件完善

1. **新增章節**
   - 「故障場景與恢復策略」（Massive 宕機、SSE 連線中斷、快取異常）
   - 「性能調優指南」（batch size、polling interval、cache size）
   - 「常見問題解答」（如何選擇 provider、如何調試相關性不符預期）

2. **交叉參考**
   - 將此文件與 `PLAN.md` 中的「市場資料模組」章節相互連結
   - 更新 `PLAN.md` 的「後端實作步驟」，引用此設計文件的對應章節

## 整體評分

| 維度 | 分數 | 備註 |
|------|------|------|
| **架構設計** | 9/10 | 解耦清晰，provider 模式靈活，唯缺 fallback 邏輯 |
| **完整度** | 8/10 | 覆蓋主流程，但故障恢復、監測指標未詳述 |
| **可實作性** | 9/10 | 程式碼範例完整，math 正確，型別清晰 |
| **安全性** | 7/10 | API key 配置說明不足，建議補充加密存儲 |
| **可維護性** | 8/10 | 文件結構清晰，但章節間的相互依賴需標註 |

**總體評價**：此文件為市場資料模組的實作奠定了堅實基礎。設計思路紮實，程式碼示例豐富，可直接用於開發。建議在實作過程中逐一驗證 Massive API 批次邏輯、provider 切換策略和 SSE 穩定性，並根據實際情況補充監測指標和故障恢復邏輯。
