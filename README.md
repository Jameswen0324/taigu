# 台股擇時挑戰

一個在瀏覽器裡玩的台股擇時小遊戲：系統從 30 檔台股（20 檔個股、10 檔 ETF）中隨機抽一檔，再隨機抽一段至少 250 個交易日的真實歷史走勢，標的名稱與日期全程隱藏；你在 23 個時間點決定買進、賣出或不操作，結算時才揭曉是哪一檔、哪段期間，並和「每次入金就全額買進、從不賣出」的對照組比較報酬，附上操作風格點評。整個遊戲是單一 HTML 頁面加一份 `prices.json`，不需要後端，可直接放上 GitHub Pages。

## 玩法

- 開局給 NT$1,000,000 現金，之後每輪再入金 NT$100,000；共 23 輪，每輪看到截至當下的走勢，選擇買進、賣出或不操作，金額可用 25%／50%／全額快速填入，或自訂。
- 對照組「持續買進組」拿到同樣的錢，每次入金都立即全額買入、從不賣出。
- 第 23 輪後以區間最後一天收盤價結算，揭曉標的、期間與 Yahoo 股市連結，比較你和持續買進組的期末總資產與報酬率（詳細數據含 XIRR），並給出玩家類型、操作點評與投資建議。
- 走勢採每日還原權息收盤價，股息視為再投入；不計交易成本與現金利息。頁尾「標的與資料說明」列出標的池與每檔的資料期間。

## 部署到 GitHub Pages

1. 在 GitHub 建立一個新的 repository（例如 `tw-timing-game`，Public）。
2. 把這個資料夾的內容推上 `main` 分支的根目錄（`index.html`、`prices.json`、`.nojekyll` 都要在根目錄）。
3. 到 repository 的 Settings → Pages，Source 選 Deploy from a branch，Branch 選 `main`、資料夾選 `/ (root)`，按 Save。
4. 等一兩分鐘部署完成，網址為 `https://<你的帳號>.github.io/<repository 名稱>/`。

`tools/data/raw/`（原始 API 快取，約 73 MB）已列在 `.gitignore`，不會被推上去；頁面只需要 `index.html` 和 `prices.json`。

## 本機預覽

在專案根目錄啟動任一靜態伺服器，再用瀏覽器開啟它印出的網址：

```
npx serve .
```

或

```
python -m http.server 8000
```

直接雙擊 `index.html` 用 `file://` 開啟會失敗，因為頁面用 `fetch` 讀取 `prices.json`，瀏覽器在 `file://` 下會擋掉這個請求，畫面會停在資料載入失敗。

## 修改頁面

`src/game.html` 是遊戲的原始碼（claude.ai Artifact 格式的頁面片段），`index.html` 由它產生，不要直接改 `index.html`。改完 `src/game.html` 後執行：

```
node tools/build.js
```

它會補上 `<!doctype html>`、`<head>`（字元編碼、viewport、基本重設樣式）與 `<body>`，寫出完整的 `index.html`。

## 更新股價資料

1. 重建 `prices.json`（預設使用 `tools/data/raw/` 的快取；加 `--refetch` 會忽略快取、重新向 Yahoo Finance 與 FinMind 抓取）：

   ```
   node tools/data/build_prices.js [--refetch]
   ```

2. 驗證資料（交易日、還原比例、分割減資等公司行動），結尾印出 PASS 才算通過：

   ```
   node tools/data/validate_prices.js
   ```

3. 重新產生頁面：`node tools/build.js`。

資料截止日寫在 `build_prices.js`、`validate_prices.js`、`fetch_corp_actions.js` 三支腳本的 `AS_OF`，更新到新日期時三處一起改，頁尾「標的與資料說明」的日期文字也要在 `src/game.html` 同步。分割、減資、面額變更的資料由 `node tools/data/fetch_corp_actions.js` 抓取，它只補抓快取裡缺少的檔案（`--refetch` 不會重抓這些）；換新截止日時先刪掉 `tools/data/raw/` 裡的 `fm_TaiwanStockSplitPrice_*`、`fm_TaiwanStockCapitalReductionReferencePrice_*`、`fm_TaiwanStockParValueChange_ALL.json`，再執行它，然後才跑 `build_prices.js`。FinMind 免費（不帶 token）使用有每小時請求次數上限，腳本在需要連網時每檔之間會間隔 1.5 秒；`--refetch` 全部重抓時若被限流，等額度恢復後再跑，已成功抓到的回應會留在快取裡。`build_prices.js` 會把各檔的處理紀錄寫到 `tools/data/build_report.json`，頁面上的「起點後移」標記與 e2e 測試都以它為準。`tools/data/` 另有挑選標的池時用過的輔助腳本（`ytd.js`、`official.js`、`size.js`、`inspect.js`，以資料夾路徑為參數），一般更新股價不需要執行。

## 測試

單元測試（抽題邏輯、價格解碼、點評規則，只需要 Node.js）：

```
node tests/pickround.test.js
```

端對端測試（以桌機 1280×900 與手機 390×844 兩種視窗實際玩完整局，檢查頁尾致謝連結、AI 點評按鈕在沒有 `window.claude` 時隱藏、無 console 錯誤等；截圖存到 `tests/shots/`）：

```
node tools/build.js
node tests/e2e.js
```

e2e 需要 Playwright 與 Chromium，腳本從 `c:\ClaudeUse\playwright\node_modules\playwright` 載入；換到其他電腦時，先 `npm install playwright` 並執行 `npx playwright install chromium`，再把 `tests/e2e.js` 開頭的 `require` 路徑改成 `playwright`。

## 資料來源與注意事項

- 股價：Yahoo Finance 每日還原權息收盤價（adjclose）。
- 交易日曆與公司行動：證交所交易日，透過 FinMind 開放資料取得（TaiwanStockPrice、分割、減資、面額變更資料集），用來核對還原價的跳動。
- 本專案僅供個人學習與教育用途；遊戲結果與點評都不是投資建議，也不是對任何個股或 ETF 的買賣推薦。歷史走勢不代表未來表現。

## 「請 Claude 深入點評」按鈕

結算畫面的「請 Claude 深入點評」按鈕只有在 claude.ai 的 Artifact 裡開啟時才會出現，因為它需要 Artifact 執行環境提供的 `window.claude`；在 GitHub Pages 或本機預覽時這個按鈕會自動隱藏，其餘功能（含內建的玩家類型與投資建議）照常運作。

## 致謝

玩法參考自清流君的〈[你能買在最低點？賣在最高點嗎？](https://buydip.peacefulring4.chatgpt.site/)〉。
