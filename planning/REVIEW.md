# PLAN.md 審查回饋

## 主要問題

1. **LLM 金鑰需求與「無需設定即可開發/測試」目標衝突**  
   `OPENROUTER_API_KEY` 被標為必填（`planning/PLAN.md:125`），但後面又說 `LLM_MOCK=true` 可支援「不需要 API key 的開發流程」（`planning/PLAN.md:355-359`）。如果啟動流程在缺少 key 時直接失敗，mock 模式與 E2E 都會被破壞。建議明確規定：只有 `LLM_MOCK=false` 且聊天端點需要真實 LLM 時才要求 `OPENROUTER_API_KEY`；mock 模式與純市場資料/交易功能不應阻擋啟動。

2. **LLM 指令引用的 skill 名稱不一致且不可驗證**  
   第 9 節先要求使用 `open-inference skill`（`planning/PLAN.md:299`），流程第 4 步又寫 `cerebras-inference skill`（`planning/PLAN.md:310`）。如果代理依文件分工，這會造成不同實作者採用不同整合方式。建議統一成一個明確名稱，並補上 LiteLLM provider/model 設定範例，例如 model id、OpenRouter headers、Cerebras routing 參數，以及 mock 模式如何繞過該整合。

3. **SSE 觀察清單同步規格不足，前後端可能做出不相容協定**  
   文件要求新增/移除 ticker 後「不需要重新連線」且前端收到移除事件後清空 sparkline（`planning/PLAN.md:177-182`），但沒有定義 SSE event type、payload schema、移除事件格式、錯誤事件、heartbeat 或 reconnect 後的初始快照。這是前後端高風險交界。建議補上至少三種事件：`snapshot`、`price`、`watchlist_removed`，並明確 JSON 欄位。

4. **`/api/prices/history/{ticker}` 沒有可支撐的資料來源定義**  
   API 要提供歷史價格（`planning/PLAN.md:262-265`），前端也依賴它初始化主圖與 sparkline（`planning/PLAN.md:371-372`），但資料庫 schema 沒有價格歷史表，市場資料章節只描述記憶體最新價格快取（`planning/PLAN.md:168-173`）。若使用模擬器，重啟後或剛加入 ticker 時如何回傳歷史資料未定義；若使用 Massive，也未說要取哪個 endpoint 與時間窗。建議定義 history endpoint 的來源、資料點數、時間粒度，以及無資料時回傳空陣列還是合成 seed series。

5. **交易與投資組合 API 缺少回應/錯誤契約，會拖慢前後端整合**  
   `/api/portfolio/trade` 只列出 request shape（`planning/PLAN.md:269-273`），但沒有定義成功回應、錯誤碼、數量/價格精度、ticker 正規化、零碎股最小數量、是否允許賣出非 watchlist ticker、以及交易時取不到價格時的處理。這些都會影響 UI 狀態更新與測試。建議新增 API schema 小節，至少定義 `TradeRequest`、`TradeResult`、`PortfolioResponse`、`ApiError`。

6. **自動執行 LLM 交易的規則彼此拉扯**  
   文件同時說 LLM 可以「代為執行交易」（`planning/PLAN.md:30`）、會自動執行 structured output 中的交易（`planning/PLAN.md:312-334`），又在 prompt 指引中要求「在使用者要求或同意時執行交易」（`planning/PLAN.md:348`）。目前沒有可執行的判斷規則，模型可能在一般分析問題中產生交易。建議後端加一道 deterministic guard：只有當最新使用者訊息明確要求交易/管理觀察清單時才執行 actions；否則只顯示建議並忽略 action arrays。

7. **Docker volume 說明有命名 volume 與專案目錄掛載混用的歧義**  
   文件說 SQLite 位於專案根目錄 `db/finally.db`（`planning/PLAN.md:68`、`planning/PLAN.md:115`），但 Docker 範例使用命名 volume `finally-data:/app/db`（`planning/PLAN.md:414-418`）。這兩者行為不同：命名 volume 不會把資料寫回 repo 的 `db/`。建議明確區分開發 bind mount 與正式/示範命名 volume，並規定 start script 採用哪一種。

8. **Next.js static export 與後端 API/static fallback 需要更精確規格**  
   架構要求 Next.js `output: 'export'` 並由 FastAPI 提供 `/*` 靜態檔案（`planning/PLAN.md:66`、`planning/PLAN.md:408`），但沒有說明輸出目錄是 `out/`、Docker 複製到哪裡、FastAPI 如何避免 `/api/*` 被 static catch-all 吃掉，以及 SPA fallback 是否需要回傳 `index.html`。建議補上路由掛載順序與 Docker copy path，避免建置完成後根頁或 API 404。

## 次要問題與改善建議

- `watchlist` 缺少 ticker 格式規範。建議統一 uppercase、去空白，並限制符號字元，避免 `aapl` 與 `AAPL` 繞過 UNIQUE。
- 金錢與股數使用 SQLite `REAL`（`planning/PLAN.md:203`、`planning/PLAN.md:219-232`）對課程專案可接受，但應規定顯示與計算四捨五入策略，否則測試容易不穩。
- `portfolio_snapshots` 每 30 秒記錄一次（`planning/PLAN.md:235`）可能讓 E2E 在短時間內沒有足夠 P&L 圖表資料。建議啟動 seed 一筆、每次價格更新可節流記錄，或測試模式縮短 snapshot interval。
- Massive API 免費方案「每分鐘 5 次、每 15 秒輪詢一次」（`planning/PLAN.md:164`）數學上是 4 次/分鐘，並且所有 ticker 聯集若不能 batch 會超額。建議確認 API endpoint 是否支援批次報價，並把輪詢策略寫清楚。
- Mock LLM 被要求至少產生一筆示範交易（`planning/PLAN.md:361`），但若每次 E2E 都執行同一筆交易，重跑或狀態未清空時可能現金/持倉不穩。建議 mock 根據輸入固定但可預測，或測試前重置 DB。
- 啟動腳本「可選擇自動開啟瀏覽器」（`planning/PLAN.md:427`）需要環境差異處理。建議預設只印 URL，另提供 `--open`。

## 建議補充的契約

- API request/response JSON schema 與錯誤格式。
- SSE event types 與完整 payload 範例。
- 價格 history 的資料來源、時間窗、點數與空狀態行為。
- LLM action execution guard 與失敗 action 的回傳格式。
- Docker/start script 的實際資料持久化模式。
- 測試模式的 deterministic seed、DB reset 策略與 mock LLM 行為。

## 總結

這份計畫的產品方向清楚，技術選型也適合課程期末專案。主要風險不在範圍過大，而在跨代理邊界的契約還不夠具體：SSE、history API、交易 API、LLM actions 與 Docker 資料路徑都需要更精準，否則前端、後端、測試代理會各自補假設，最後整合成本會升高。
