# FinAlly — AI 交易工作站

## 專案規格

## 1. 願景

FinAlly（Finance Ally）是一個視覺上令人驚豔的 AI 交易工作站，能串流即時市場資料，讓使用者交易模擬投資組合，並整合可分析持倉與代為執行交易的 LLM 聊天助理。它的外觀與操作感受就像現代版的 Bloomberg 終端機，並帶有 AI 副駕。

這是 agentic AI 程式設計課程的期末專案。整個專案完全由 Coding Agents 建構，展示經由協作的 AI 代理如何產出可用於實戰的全端應用程式。各代理透過 planning/ 目錄中的檔案互動。

## 2. 使用者體驗

### 初次啟動

使用者執行單一 Docker 指令（或提供的啟動腳本）。瀏覽器會開啟到 http://localhost:8000。不需要登入，也不需要註冊。使用者會立刻看到：

- 一個含有 10 個預設 ticker 的觀察清單，價格即時更新並以格狀呈現
- 10,000 美元的虛擬現金
- 深色、資訊密集的交易終端機風格
- 已準備好的 AI 聊天面板

### 使用者可以做什麼

- **觀看價格串流** — 價格上漲時閃綠、下跌時閃紅，並帶有柔和淡出的 CSS 動畫
- **查看 sparkline 小型圖表** — 每個 ticker 旁邊的價格走勢由前端根據頁面載入後收到的 SSE 串流累積而成（sparklines 會逐步填滿）
- **點選 ticker** 以在主圖表區看到更大型的詳細圖表
- **買進與賣出股票** — 只支援市價單，會以當前價格立即成交，沒有手續費，也沒有確認對話框
- **監控投資組合** — 以 heatmap（treemap）顯示持倉，區塊大小代表權重、顏色代表損益，並搭配追蹤總資產價值變化的 P&L 圖表
- **查看持倉表格** — ticker、數量、平均成本、目前價格、未實現損益、變動百分比
- **與 AI 助理聊天** — 詢問投資組合狀態、取得分析，並讓 AI 以自然語言執行交易與管理觀察清單
- **管理觀察清單** — 可手動新增/移除 ticker，或透過 AI 聊天完成

### 視覺設計

- **深色主題**：背景約為 #0d1117 或 #1a1a2e，搭配低飽和灰色邊框，不使用純黑
- **價格閃爍動畫**：價格變動時短暫出現綠/紅背景高亮，並透過 CSS transition 在約 500ms 內淡出
- **連線狀態指示器**：頁首顯示小型彩色圓點（綠色＝已連線、黃色＝重新連線中、紅色＝已中斷）
- **專業、資訊密集的版面**：靈感來自 Bloomberg/交易終端機，每個像素都有用途
- **響應式但以桌面優先**：針對寬螢幕最佳化，平板上仍可正常使用

### 配色方案

- 強調黃：#ecad0a
- 主藍：#209dd7
- 次要紫：#753991（提交按鈕）

## 3. 架構總覽

### 單容器、單埠

```
┌─────────────────────────────────────────────────┐
│  Docker Container (port 8000)                   │
│                                                 │
│  FastAPI (Python/uv)                            │
│  ├── /api/*          REST endpoints             │
│  ├── /api/stream/*   SSE streaming              │
│  └── /*              Static file serving        │
│                      (Next.js export)           │
│                                                 │
│  SQLite database (volume-mounted)               │
│  Background task: market data polling/sim       │
└─────────────────────────────────────────────────┘
```

- **前端**：Next.js + TypeScript，建置為靜態輸出（output: 'export'），由 FastAPI 當作靜態檔案提供
- **後端**：FastAPI（Python），以 uv 專案管理
- **資料庫**：SQLite，單一檔案位於 db/finally.db，透過 volume 掛載以保留資料
- **即時資料**：Server-Sent Events（SSE）— 比 WebSocket 更簡單、單向 server→client 推送、各瀏覽器都支援
- **AI 整合**：LiteLLM → OpenRouter（Cerebras 提供快速推論），並以 structured outputs 進行交易執行
- **市場資料**：由環境變數決定，預設使用模擬器；若提供金鑰則改用 Massive API 的真實資料

### 為什麼選這些方案

| 決策                 | 理由                                                                      |
| -------------------- | ------------------------------------------------------------------------- |
| SSE 取代 WebSockets  | 只需要單向推送；更簡單、沒有雙向複雜度、瀏覽器原生支援                    |
| 靜態 Next.js 輸出    | 單一來源，沒有 CORS 問題，一個埠、一個容器，部署簡單                      |
| SQLite 取代 Postgres | 沒有認證就沒有多使用者需求，也就不需要資料庫服務；自包含、零設定          |
| 單一 Docker 容器     | 學生只要執行一個指令；正式環境不需要 docker-compose 或額外服務編排        |
| Python 使用 uv       | 快速、現代化的 Python 專案管理；可重現的 lockfile；也是學生應該學習的工具 |
| 只支援市價單         | 移除委託簿、限價單與部分成交邏輯，大幅簡化投資組合計算                    |

---

## 4. 目錄結構

```
finally/
├── frontend/                 # Next.js TypeScript 專案（靜態輸出）
├── backend/                  # FastAPI uv 專案（Python）
│   └── db/                   # Schema 定義、種子資料、migration 邏輯
├── planning/                 # 提供代理使用的專案文件
│   ├── PLAN.md               # 本文件
│   └── ...                   # 其他代理參考文件
├── scripts/
│   ├── start_mac.sh          # 啟動 Docker 容器（macOS/Linux）
│   ├── stop_mac.sh           # 停止 Docker 容器（macOS/Linux）
│   ├── start_windows.ps1     # 啟動 Docker 容器（Windows PowerShell）
│   └── stop_windows.ps1      # 停止 Docker 容器（Windows PowerShell）
├── test/                     # Playwright E2E 測試 + docker-compose.test.yml
├── db/                       # volume 掛載目標（SQLite 檔案執行時會放在這裡）
│   └── .gitkeep              # 儲存庫中保留此目錄；finally.db 會被 gitignore
├── Dockerfile                # 多階段建置（Node → Python）
├── docker-compose.yml        # 便利包裝（E2E 測試必要）
├── .env                      # 環境變數（gitignore，.env.example 會提交）
└── .gitignore
```

### 主要邊界

- **frontend/** 是獨立的 Next.js 專案，完全不認識 Python。它透過 /api/_ 端點與 /api/stream/_ SSE 端點與後端溝通。內部結構由 Frontend Engineer 代理決定。
- **backend/** 是獨立的 uv 專案，擁有自己的 pyproject.toml。它負責所有伺服器邏輯，包括資料庫初始化、schema、種子資料、API 路由、SSE 串流、市場資料與 LLM 整合。內部結構由 Backend/Market Data 代理決定。
- **backend/db/** 放置 schema SQL 定義與種子邏輯。後端會在第一次請求時懶載入資料庫，若 SQLite 檔案不存在或資料表缺失，就建立 schema 並寫入預設資料。
- **db/** 的頂層目錄是執行期 volume 掛載點。SQLite 檔案（db/finally.db）會由後端建立在這裡，並透過 Docker volume 在容器重啟後持續保留。
- **planning/** 含有專案層級文件，包括這份計畫。所有代理都把這裡當作共享契約。
- **test/** 含有 Playwright E2E 測試與支援基礎設施（例如 docker-compose.test.yml）。單元測試則依各自框架慣例放在 frontend/ 與 backend/ 內。
- **scripts/** 放置包裝 Docker 指令的啟動/停止腳本。

---

## 5. 環境變數

```bash
# 必填：OpenRouter API key，用於 LLM 聊天功能
OPENROUTER_API_KEY=your-openrouter-api-key-here

# 選填：Massive（Polygon.io）API key，用於真實市場資料
# 若未設定，會使用內建市場模擬器（多數使用者建議如此）
MASSIVE_API_KEY=

# 選填：設為 "true" 時使用可預測的 mock LLM 回應（測試用）
LLM_MOCK=false
```

### 行為

- 若 MASSIVE_API_KEY 已設定且非空 → 後端使用 Massive REST API 取得市場資料
- 若 MASSIVE_API_KEY 未設定或為空 → 後端使用內建市場模擬器
- 若 LLM_MOCK=true → 後端回傳可預測的 mock LLM 回應（供 E2E 測試）
- 後端會從專案根目錄讀取 .env（可掛載進容器，或透過 docker --env-file 讀取）

---

## 6. 市場資料

### 兩種實作、一個介面

模擬器與 Massive 客戶端都實作相同的抽象介面。後端會依環境變數決定使用哪一個。所有下游程式碼（SSE 串流、價格快取、前端）都不需要知道資料來源。

### 模擬器（預設）

- 以幾何布朗運動（GBM）產生價格，且每個 ticker 都可設定漂移與波動度
- 約每 500ms 更新一次
- 不同 ticker 之間會有相關移動（例如科技股會一起波動）
- 偶爾出現隨機「事件」— 某個 ticker 突然發生 2-5% 的劇烈變動，增加戲劇效果
- 起始價格使用合理的種子值（例如 AAPL 約 190 美元、GOOGL 約 175 美元等）
- 以程序內背景任務執行，沒有外部依賴

### Massive API（選用）

- 使用 REST API 輪詢（不是 WebSocket）— 更簡單，也能支援所有方案等級
- 依照可配置的間隔輪詢所有被觀察 ticker 的聯集
- 免費方案（每分鐘 5 次）：每 15 秒輪詢一次
- 付費方案：依方案等級每 2 到 15 秒輪詢一次
- 將 REST 回應轉成與模擬器相同的格式

### 共用價格快取

- 單一背景任務（模擬器或 Massive 輪詢器）寫入記憶體中的價格快取
- 快取會保存每個 ticker 的最新價格、前一筆價格與時間戳
- SSE 串流從快取讀取並推送更新給已連線的客戶端
- 這個架構可在未來支援多使用者，而不必變更資料層

### SSE 串流

- 端點：GET /api/stream/prices
- 長連線 SSE；客戶端使用原生 EventSource API
- 伺服器以固定節奏（約 500ms）推送系統中所有 ticker 的價格更新；在單一使用者模型下，這等同於使用者的觀察清單
- 每個 SSE event 都包含 ticker、價格、前一筆價格、時間戳與變動方向
- 客戶端會自動處理重新連線（EventSource 內建 retry）
- **觀察清單同步**：使用者新增或移除 ticker 時，後端立即更新 SSE 推送的訂閱列表，不需要重新連線。前端收到移除事件後應清空該 ticker 已累積的 sparkline 資料。

---

## 7. 資料庫

### 延遲初始化的 SQLite

後端會在啟動時（或第一次請求時）檢查 SQLite 資料庫。如果檔案不存在或資料表缺失，就會建立 schema 並寫入預設資料。這代表：

- 不需要獨立的 migration 步驟
- 不需要手動建立資料庫
- 新的 Docker volume 會自動以乾淨且已初始化的資料庫啟動

### Schema

所有資料表都包含 user_id 欄位，預設為 "default"。目前這是硬編碼的單一使用者設計，但可在未來不需 schema migration 的情況下擴展成多使用者。

**users_profile** — 使用者狀態（現金餘額）

- id TEXT PRIMARY KEY（預設："default"）
- cash_balance REAL（預設：10000.0）
- created_at TEXT（ISO 時間戳）

**watchlist** — 使用者正在關注的 ticker

- id TEXT PRIMARY KEY（UUID）
- user_id TEXT（預設："default"）
- ticker TEXT
- added_at TEXT（ISO 時間戳）
- 在 (user_id, ticker) 上有 UNIQUE 約束

**positions** — 目前持倉（每個使用者每個 ticker 一列）

- id TEXT PRIMARY KEY（UUID）
- user_id TEXT（預設："default"）
- ticker TEXT
- quantity REAL（支援零碎股）
- avg_cost REAL（賣出時不變，只有買進時以加權平均更新）
- updated_at TEXT（ISO 時間戳）
- 在 (user_id, ticker) 上有 UNIQUE 約束
- 當 quantity 歸零時刪除該資料列（不保留 quantity=0 的空持倉）

**trades** — 交易歷史（只追加的紀錄）

- id TEXT PRIMARY KEY（UUID）
- user_id TEXT（預設："default"）
- ticker TEXT
- side TEXT（"buy" 或 "sell"）
- quantity REAL（支援零碎股）
- price REAL
- executed_at TEXT（ISO 時間戳）

**portfolio_snapshots** — 投資組合價值的時間序列（供 P&L 圖表使用）。由背景任務每 30 秒記錄一次，並在每筆交易執行後立即記錄。

- id TEXT PRIMARY KEY（UUID）
- user_id TEXT（預設："default"）
- total_value REAL
- recorded_at TEXT（ISO 時間戳）

**chat_messages** — 與 LLM 的對話歷史

- id TEXT PRIMARY KEY（UUID）
- user_id TEXT（預設："default"）
- role TEXT（"user" 或 "assistant"）
- content TEXT
- actions TEXT（JSON — 已執行交易、已做的觀察清單變更；使用者訊息為 null）
- created_at TEXT（ISO 時間戳）

### 預設種子資料

- 一個使用者資料：id="default"、cash_balance=10000.0
- 十個觀察清單項目：AAPL、GOOGL、MSFT、AMZN、TSLA、NVDA、META、JPM、V、NFLX

---

## 8. API 端點

### 市場資料

| 方法 | 路徑                        | 說明                                                      |
| ---- | --------------------------- | --------------------------------------------------------- |
| GET  | /api/stream/prices          | 即時價格更新的 SSE 串流                                   |
| GET  | /api/prices/history/{ticker} | 指定 ticker 的歷史價格序列（供主圖表與 sparkline 初始化） |

### 投資組合

| 方法 | 路徑                   | 說明                                    |
| ---- | ---------------------- | --------------------------------------- |
| GET  | /api/portfolio         | 目前持倉、現金餘額、總價值、未實現損益  |
| POST | /api/portfolio/trade   | 執行交易：{ticker, quantity, side}      |
| GET  | /api/portfolio/history | 投資組合價值時間序列（供 P&L 圖表使用） |

### 觀察清單

| 方法   | 路徑                    | 說明                           |
| ------ | ----------------------- | ------------------------------ |
| GET    | /api/watchlist          | 目前觀察清單 ticker 與最新價格 |
| POST   | /api/watchlist          | 新增 ticker：{ticker}          |
| DELETE | /api/watchlist/{ticker} | 移除 ticker                    |

### 聊天

| 方法 | 路徑      | 說明                                              |
| ---- | --------- | ------------------------------------------------- |
| POST | /api/chat | 傳送訊息，回傳完整 JSON 回應（訊息 + 已執行動作） |

### 系統

| 方法 | 路徑        | 說明                           |
| ---- | ----------- | ------------------------------ |
| GET  | /api/health | 健康檢查（供 Docker/部署使用） |

---

## 9. LLM 整合

撰寫 LLM 呼叫程式碼時，請使用 open-inference skill，透過 LiteLLM 與 OpenRouter，呼叫 openrouter/openai/gpt-oss-120b 模型，並以 Cerebras 作為推論供應商。結果應使用 Structured Outputs 來解析。

專案根目錄的 .env 檔中已包含 OPENROUTER_API_KEY。

### 運作方式

當使用者送出聊天訊息時，後端會：

1. 載入使用者目前的投資組合內容（現金、含損益的持倉、含即時價格的觀察清單、總資產價值）
2. 從 chat_messages 資料表載入最近 5 則對話（user + assistant 各算一則），以避免 prompt 超出模型上下文視窗
3. 使用系統訊息、投資組合內容、對話歷史與使用者新訊息組成 prompt
4. 透過 LiteLLM → OpenRouter 呼叫 LLM，並要求 structured output，使用 cerebras-inference skill
5. 解析完整的結構化 JSON 回應
6. 自動執行回應中指定的交易或觀察清單變更
7. 將訊息與已執行動作存入 chat_messages
8. 將完整 JSON 回應回傳給前端（不做 token-by-token 串流，因為 Cerebras 推論夠快，載入中指示器就足夠）

### Structured Output Schema

LLM 會被要求回傳符合以下 schema 的 JSON：

```json
{
  "message": "Your conversational response to the user",
  "trades": [{ "ticker": "AAPL", "side": "buy", "quantity": 10 }],
  "watchlist_changes": [{ "ticker": "PYPL", "action": "add" }]
}
```

- message（必填）：顯示給使用者的對話文字
- trades（選填）：要自動執行的交易陣列。每筆交易都會經過與手動交易相同的驗證（買進需有足夠現金，賣出需有足夠持股）
- watchlist_changes（選填）：觀察清單變更陣列

### 自動執行

LLM 指定的交易會自動執行，不會跳出確認對話框。這是刻意的設計選擇：

- 這是模擬環境，使用的是虛擬貨幣，所以沒有風險
- 這會帶來更流暢、令人印象深刻的示範體驗
- 這能展現 agentic AI 能力，也就是這門課程的核心主題

如果某筆交易驗證失敗（例如現金不足），錯誤會包含在聊天回應中，讓 LLM 能向使用者說明原因。

### 系統提示指引

LLM 應被設定為「FinAlly，一個 AI 交易助理」，並要求它：

- 分析投資組合組成、風險集中度與損益
- 提供帶有理由的交易建議
- 在使用者要求或同意時執行交易
- 主動管理觀察清單
- 回應要簡潔且以資料為導向
- 一律回傳有效的 structured JSON

### LLM Mock 模式

當 LLM_MOCK=true 時，後端會回傳可預測的 mock 回應，而不是呼叫 OpenRouter。這可用於：

- 快速、免費、可重現的 E2E 測試
- 不需要 API key 的開發流程
- CI/CD pipelines

Mock 回應應包含 `message`、`trades`（至少含一筆示範交易）與 `watchlist_changes`，以確保 E2E 測試能覆蓋交易執行與觀察清單變更的顯示邏輯。

---

## 10. 前端設計

### 版面

前端是一個單頁應用程式，採用資訊密集、終端機風格的版面。具體的元件架構與 layout 系統由 Frontend Engineer 決定，但 UI 至少應包含以下元素：

- **觀察清單面板** — 被觀察 ticker 的格狀/表格視圖，包含 ticker 符號、目前價格（變動時綠/紅閃爍）、日變動百分比（以當日第一筆收到的價格為基準；資料不足時不顯示），以及 sparkline 小圖表（初始資料從 GET /api/prices/history/{ticker} 載入，之後由 SSE 串流持續累積；從觀察清單移除時清空）
- **主圖表區** — 目前選擇 ticker 的大型圖表，初始歷史資料從 GET /api/prices/history/{ticker} 載入，之後由 SSE 串流即時延伸。點選觀察清單中的 ticker 會在這裡選取它。
- **投資組合 heatmap** — treemap 視覺化，每個矩形代表一個持倉，大小依投資組合權重決定，顏色依損益決定（綠＝獲利，紅＝虧損）
- **P&L 圖表** — 顯示總資產價值隨時間變化的折線圖，使用 portfolio_snapshots 資料
- **持倉表格** — 所有持倉的表格視圖：ticker、數量、平均成本、目前價格、未實現損益、變動百分比
- **交易列** — 簡單輸入區：ticker 欄位、數量欄位、買入按鈕、賣出按鈕。市價單、立即成交。
- **AI 聊天面板** — 停靠/可收合側邊欄。包含訊息輸入、可捲動對話紀錄，以及等待 LLM 回應時的載入指示器。交易執行與觀察清單變更會以內嵌確認的方式顯示。
- **頁首** — 投資組合總價值（即時更新）、連線狀態指示器、現金餘額

### 技術備註

- SSE 連線請使用 EventSource 連到 /api/stream/prices
- 圖表效能優先，建議使用 Canvas 型圖表函式庫（Lightweight Charts 或 Recharts）
- 價格閃爍效果：收到新價格時，短暫套用帶有背景色 transition 的 CSS class，然後移除
- 所有 API 呼叫都走同一來源（/api/\*），因此不需要 CORS 設定
- 使用 Tailwind CSS 搭配自訂深色主題做樣式

---

## 11. Docker 與部署

### 多階段 Dockerfile

```
Stage 1: Node 20 slim
  - Copy frontend/
  - pnpm install --frozen-lockfile && pnpm run build (produces static export)

Stage 2: Python 3.12 slim
  - Install uv
  - Copy backend/
  - uv sync (install Python dependencies from lockfile)
  - Copy frontend build output into a static/ directory
  - Expose port 8000
  - CMD: uvicorn serving FastAPI app
```

FastAPI 會在 8000 埠提供靜態前端檔案與所有 API 路由。

### Docker Volume

SQLite 資料庫會透過命名 Docker volume 持久化：

```bash
docker run -v finally-data:/app/db -p 8000:8000 --env-file .env finally
```

專案根目錄中的 db/ 目錄會對映到容器內的 /app/db。後端會把 finally.db 寫到這個路徑。

### 啟動/停止腳本

**scripts/start_mac.sh**（macOS/Linux）：

- 如果尚未建置鏡像（或有傳入 --build 旗標）就先建置 Docker image
- 以 volume 掛載、埠對應與 .env 檔執行容器
- 印出應用程式存取網址
- 可選擇自動開啟瀏覽器

**scripts/stop_mac.sh**（macOS/Linux）：

- 停止並移除正在執行的容器
- 不會移除 volume（資料會保留）

**scripts/start_windows.ps1** / **scripts/stop_windows.ps1**：Windows 的 PowerShell 對應版本。

所有腳本都應具備 idempotent 特性，也就是可以安全執行多次。

### 可選雲端部署

此容器設計可部署到 AWS App Runner、Render 或任何容器平台。App Runner 的 Terraform 設定可以作為進階目標放在 deploy/ 目錄中，但不屬於核心建置的一部分。

---

## 12. 測試策略

### 單元測試（放在 frontend/ 與 backend/ 內）

**後端（pytest）**：

- 市場資料：模擬器能產生有效價格、GBM 數學正確、Massive API 回應解析正常、兩種實作都符合抽象介面
- 投資組合：交易執行邏輯、P&L 計算、邊界情況（賣超過持有數量、現金不足時買進、虧損賣出）
- LLM：structured output 解析可處理所有合法 schema、可優雅處理格式錯誤的回應、聊天流程中的交易驗證
- API 路由：正確的 status code、回應格式與錯誤處理

**前端（React Testing Library 或類似工具）**：

- 使用 mock 資料進行元件渲染
- 價格變動時能正確觸發閃爍動畫
- 觀察清單 CRUD 操作
- 投資組合顯示計算
- 聊天訊息渲染與載入狀態

### E2E 測試（位於 test/）

**基礎架構**：在 test/ 中提供獨立的 docker-compose.test.yml，同時啟動應用容器與 Playwright 容器。如此可避免把瀏覽器相依套件放進 production image。

**環境**：預設以 LLM_MOCK=true 執行測試，以確保速度與可重現性。

**關鍵情境**：

- 全新啟動：出現預設觀察清單、顯示 1 萬美元餘額、價格持續串流
- 在觀察清單中新增與移除 ticker
- 買進股票：現金減少、出現持倉、投資組合更新
- 賣出股票：現金增加、持倉更新或消失
- 投資組合視覺化：heatmap 顯示正確顏色，P&L 圖表有資料點
- AI 聊天（mock）：送出訊息、收到回應、交易執行以內嵌方式顯示
- SSE 韌性：中斷連線並驗證能重新連線
