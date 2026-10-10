# mLevel 工作區

本工作區有三個互相依賴的專案。已售出的舊版 PC 腳本仍在使用者電腦上執行、無法強制更新，所以任何回答與修改都要從三個專案一起看。

| 專案 | 路徑 | 技術 | 角色 |
|------|------|------|------|
| PC 端腳本 | `mlevel/` | Python（Tkinter），PyInstaller 打包（`build.ps1`、`mlevel.spec`） | 販售的自動化程式。授權與席位在 `license_gate.py`，自動更新在 `updater.py`、`updater_client.py` |
| 販售網頁 | `mlevelweb/` | Vue 3 + Vite，Firebase Hosting | 購買、查詢訂單、找回授權、下載。`src/lib/ecpay.js` 呼叫後端 |
| 後端 Functions | `mlevelweb/functions/` | Node 22，firebase-functions，`asia-east1` | 單一 HTTP function `ecpayApi`（`index.js`）：綠界金流、金鑰（`license.js`）、驗證、席位、更新、下載 |

資料流：網頁 →（`/create`、`/order`、`/lookup`）→ `ecpayApi` ← 綠界（`/notify`、`/result`）；PC 腳本 →（`/verify`、`/session/*`、`/update/*`）→ `ecpayApi`。

## 回答問題前

- 問題涉及功能、API、授權、席位、付款、更新、設定檔、打包或部署時，**先讀相關程式碼再回答**，不要憑記憶或猜測。
- 跨專案的問題（例如某個 API 欄位、金鑰、價格、下載流程），**三個專案都要查**：呼叫端（`mlevel/` 或 `mlevelweb/src/`）和服務端（`mlevelweb/functions/`）都要確認，再說明兩邊目前的行為。
- 只涉及單一專案的問題，也要確認是否有其他專案依賴它（例如改 `functions/` 的回應，要查 `license_gate.py` 和 `src/lib/ecpay.js` 怎麼讀）。
- 回答時用 `檔案路徑:行號` 標出依據。程式碼和文件不一致時，以程式碼為準，並指出不一致的地方。
- 純觀念或與程式碼無關的問題，不需要讀程式碼。

## 修改程式時

- 不可破壞三個專案的既有功能；新功能必須相容舊版 PC 腳本與既有使用者資料。
- API 路由與欄位只增不改不刪；不動金鑰格式、雜湊、簽章演算法；本機設定檔讀取要容許舊格式；部署順序先後端、再用戶端。
- 完整契約（API 路由表、金鑰格式、綠界規則、Firestore、本機檔案、安全常數）見 `.claude/skills/tri-light-review/mlevel-context.md`，修改前先讀。
- 審查程式碼使用 `tri-light-review` skill。
