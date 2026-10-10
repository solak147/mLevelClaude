# mLevel 專案脈絡與相容性契約

審查 `D:\mLevelClaude` 下任何程式碼前先讀本檔。三個專案互相依賴，而且**已售出的舊版 PC 腳本仍在使用者電腦上執行，除了業主決定啟用的淘汰機制外無法強制更新**，所以任何介面變動都要以「舊版用戶端不壞」為前提。

> 本檔記錄的是「哪些東西是契約」，數值僅供參考。審查時一律以程式碼為準；
> 發現本檔和程式碼不一致時，在報告的綠燈段落提醒更新本檔。

## 三個專案

| 專案 | 路徑 | 技術 | 角色 |
|------|------|------|------|
| PC 端腳本 | `mlevel/` | Python（Tkinter GUI），PyInstaller 打包（`mlevel.spec`、`build.ps1`）→ `dist/` | 實際販售的自動化程式。`mlevel.py` 主程式、`auto_hunt.py`、`minigame.py`、`desktop_visual_trigger.py`；`license_gate.py` 負責授權驗證與席位 |
| 販售網頁 | `mlevelweb/` | Vue 3 + Vite，Firebase Hosting | 購買、查詢訂單、找回授權、下載程式。`src/lib/ecpay.js` 呼叫後端 |
| 後端 Functions | `mlevelweb/functions/` | Node 22，firebase-functions v2，region `asia-east1` | 單一 HTTP function `ecpayApi`（`index.js`）處理綠界金流、發放金鑰、授權驗證、席位租約、下載。`ecpay.js` 簽章、`license.js` 金鑰格式 |

## 跨專案契約（不可破壞）

### 1. 後端 API：`https://asia-east1-mlevel-f575a.cloudfunctions.net/ecpayApi`

| 路由 | 呼叫者 | App Check | 備註 |
|------|--------|:---:|------|
| `GET /` | 健康檢查 | | |
| `POST /create` | 網頁 | ✔ | 建立訂單、回傳綠界表單欄位 |
| `POST /notify` | **綠界伺服器**（ReturnURL） | | 必須驗 CheckMacValue，成功回純文字 `1\|OK`，失敗 `0\|...` |
| `GET/POST /result` | **綠界導回瀏覽器**（OrderResultURL） | | |
| `GET /order` | 網頁 | ✔ | |
| `POST /lookup` | 網頁 | ✔ | 找回授權 |
| `POST /renew` | 網頁 | ✔ | 續訂：body `email, licenseKey`，回傳格式同 `/create` 另加 `currentExpiresAt`。續訂訂單 `orderType: 'renewal'`、`licenseKey` 必須是空字串（金鑰存 `renewsLicenseKey`），付款後由 `applyRenewal` 延長**原始訂單**的 `licenseExpiresAt` |
| `POST /verify` | **舊版 PC 腳本**的啟動檢查（`index.js:58`），不佔席位 | | 不可移除或改回應格式；唯一例外是業主啟用 `MLEVEL_REQUIRE_SESSION=true`，讓它回 `426 UPGRADE_REQUIRED`（見 SKILL.md「有規劃的淘汰」） |
| `POST /session/acquire` | PC 腳本 | | body：`licenseKey, deviceId, deviceLabel, force` |
| `POST /session/heartbeat` | PC 腳本 | | body：`licenseKey, sessionId`；409 = 席位被搶 |
| `POST /session/release` | PC 腳本 | | 失敗被用戶端忽略 |
| `POST /update/check` | PC 腳本（1.0.0 起，自動更新） | | body：`licenseKey, sessionId, version`；要有成立中的租約（`checkSessionLease`，和心跳同一套）。**刻意不回 `valid`**：用戶端以 `require_valid=False` 呼叫、看 `ok` 判斷，不會把付費用戶誤判成無效 |
| `POST /update/download` | PC 腳本（1.0.0 起） | | body：`licenseKey, sessionId`；回 Storage 短效網址與 `sha256`。同樣不回 `valid` |
| `GET/HEAD /download` | 網頁（`/order`、`/lookup` 回傳的 `downloadUrl`、`manualUrl`） | | query：`orderId, token, file`（`app`／`manual`） |

App Check 是否真的阻擋請求，取決於 `functions/.env` 的 `MLEVEL_APPCHECK_ENFORCE`（`index.js:262`）；未開啟時只記 log。評估「繞過網頁直接打 API」的風險時要考慮這點。

**規則：**
- 路由路徑、HTTP 方法、request 欄位名稱、response 欄位名稱與型別、HTTP 狀態碼語意，一律**只增不改不刪**
- PC 端 `_post` 以 response 有無 `valid` 欄位判斷「後端錯誤 vs 金鑰無效」——任何會被 PC 端呼叫的端點都必須保留 `valid`，否則付費用戶會被誤判
- `/session/acquire` 回傳 `heartbeatSeconds`（目前後端 `SESSION_HEARTBEAT_SECONDS` 是 360 秒，租約 `SESSION_LEASE_MS` 是 15 分鐘），用戶端下限 30 秒、後端沒回時用戶端退回 180 秒；改動時要確認舊版用戶端在租約時間內一定會送出心跳
- PC 端寫死 `DEFAULT_API_BASE`（可被 `MLEVEL_API_BASE` 環境變數覆寫）：**不可更改 function 名稱、region 或專案 ID**
- 新增欄位時，舊版用戶端會忽略它；新版用戶端讀取時必須容許欄位不存在（舊後端或快取資料）
- App Check 不可加到 PC 端會呼叫的路由（桌面程式送不了 App Check token）

### 2. 授權金鑰格式

- `MLV-XXXX-XXXX-XXXX-XXXX`（16 字、去除易混淆字元 0/O/1/I/L），PC 端 `KEY_PATTERN` 與後端 `license.js` `normalizeLicenseKey` 必須一致
- 已發出的金鑰永遠要能驗證；不可改變正規化規則讓舊金鑰失效
- `mlevel_sessions` 文件 ID 是 `hashLicenseKey(licenseKey)`：雜湊演算法變了，既有席位會全部失聯

### 3. 綠界金流

- `ecpay.js` 的 CheckMacValue 演算法（SHA256、URL encode 規則）照綠界規範，**不可隨意調整**
- 驗章採「綠界送什麼欄位就全部納入計算」，不可改成白名單，否則綠界新增欄位就驗章失敗
- 回呼可能重送：`/notify` 處理必須冪等（同一筆 MerchantTradeNo 不可重複發金鑰或重複入帳）
- 金額必須比對訂單記錄（縱深防禦）
- 正式 HashKey、HashIV、MerchantID 不可寫死、不可記入日誌，必須從 `functions/.env` 讀取
- `ecpay.js` 內建的綠界公開測試金鑰是刻意保留的；只有在 `ECPAY_ALLOW_PUBLIC_TEST_CREDENTIALS=true` 且結帳網址是 `payment-stage` 時才可使用。這道保護不可移除或放寬（否則漏帶 .env 時，任何人都能偽造 /notify 拿到金鑰）
- `firebase.json` CSP 的 `form-action` 與 `connect-src` 綁著綠界與 Functions 網域；改網域要同步改 CSP

### 售價

- 後端 `PRODUCT` 售價（`index.js` 的 `PRODUCT`）是權威價格，`/create` 不採用前端傳入的金額
- 前端 `HeroSection.vue`、`CheckoutModal.vue` 顯示的價格必須同步修改，否則使用者看到的價格和實際扣款不同

### 4. Firestore

- 集合：`mlevel_orders`（訂單、金鑰、email、accessToken）、`mlevel_sessions`（席位租約）
- 金鑰的到期日只存在**原始訂單**（`licenseKey` 欄位等於該金鑰的那張）上；續訂訂單的 `licenseKey` 一律空字串，否則 `findPaidOrder` 可能撈到續訂訂單。`mlevel_sessions.licenseExpiresAt` 是副本，續訂時同步更新，心跳判定到期前也會回頭查訂單（`refreshSessionExpiry`）
- `firestore.rules` 全部拒絕前端存取，只能透過 Functions（Admin SDK）存取——**不可放寬**
- 欄位只增不刪；讀取時要處理舊文件缺少新欄位的情況（不可假設欄位一定存在）
- 新增需要的查詢要同步更新 `firestore.indexes.json`

### 5. PC 端本機資料（使用者升級後必須能沿用）

| 檔案 | 位置 | 相容性要求 |
|------|------|------|
| `license.json` | `%LOCALAPPDATA%\mlevel\`（退路 `%APPDATA%`） | 含 HMAC 簽章 `sig`，金鑰為 `mlevel-license-cache-v1\|COMPUTERNAME\|USERNAME`。改簽章方式或欄位會讓所有用戶的快取失效、要重新輸入金鑰 |
| `device.json` | 同上 | 裝置 ID 退路；`device_id()` 演算法改變 = 所有用戶被視為新裝置 |
| `auto_hunt_config.json` | 打包版：`<exe 所在資料夾>\config\`（`mlevel.py:51`）；原始碼版：`mlevel/config/` | 使用者設定；新增欄位要有預設值，讀到舊版檔案不可報錯 |
| `routes.json` | 同上 | 有 `ROUTE_FILE_VERSION`（目前 3）；格式變更要遞增版本並提供舊版遷移 |
| `rope_memory.json` | 同上 | 同上：缺欄位用預設值、無法解析時不可崩潰 |
| `visual_trigger_settings.json` | 舊程式 `desktop_visual_trigger.py` 專用，不出貨（`build.ps1:67`） | 不屬於產品契約 |

- `config_file()`（`mlevel.py:85`）會把舊版放在根目錄的設定檔自動搬進 `config/`，這段遷移邏輯不可移除
- 打包版啟動時會 `os.chdir` 到 exe 所在資料夾，設定裡的模板路徑是相對路徑
- `templates/`、`maps/`、`monster/` 等資源檔的路徑直接寫在使用者設定裡（例如 `monster/blue-snail.png`），不可改名或搬移
- **更新流程**：zip 內含 `config/`（`build.ps1:61-69`），舊用戶解壓覆蓋會蓋掉自己的設定與路線。任何調整打包內容的變更，都要確認不會覆蓋使用者既有的 `config/`、`maps/`；使用手冊目前沒有更新步驟，需確認
- **自動更新（1.0.0 起）**：更新包 `mlevel-<ver>-update.zip` 只含 `mlevel.exe`、`_internal`、`mlevel-updater.exe` 與 `defaults/`（monster、templates；maps 不放，地圖清單在 routes.json，補圖沒用）。
  `updater.py` 只用 `defaults/` 補使用者缺的檔案，不覆蓋；`config/`、`log/` 永遠不碰。改更新包格式時，要確認已裝 1.0.0 的 updater 還能處理新的包（updater 是跟著舊版一起出貨、由舊版啟動的）

### 6. 安全常數

- `MASTER_KEY_SHA256`：只存雜湊，不可把原文寫進程式碼或日誌
- `OFFLINE_GRACE_MINUTES`（30 分鐘）：放寬會削弱「一金鑰一席」的限制，縮短會讓後端維護時把付費用戶擋在外面——變動需說明理由

## 相容性審查判定

| 情況 | 燈號 |
|------|:---:|
| 刪除或改名已存在的 API 路由、欄位、狀態碼語意 | 🔴 |
| 修改金鑰格式、正規化、雜湊、裝置 ID、快取簽章演算法，而沒有提供舊格式相容路徑 | 🔴 |
| 本機設定檔格式變更後，讀取舊版檔案會崩潰或遺失使用者設定 | 🔴 |
| 綠界回呼失去冪等性、驗章變寬鬆、金額未比對 | 🔴 |
| 放寬 `firestore.rules` 或在 PC 端路由加上 App Check | 🔴 |
| 變更 function 名稱、region、專案 ID、`DEFAULT_API_BASE` | 🔴 |
| 業主明確決定，透過 `MLEVEL_REQUIRE_SESSION` 等既有機制淘汰舊版 | 🟡（需業主確認） |
| 改後端售價但前端顯示價格沒同步 | 🟡 |
| 出貨的 zip 帶上開發者的路線、繩索記憶，或更新時會覆蓋使用者的 config | 確認使用者會解壓覆蓋時 🔴；更新方式未確認時 🟡（需確認） |
| 新增欄位但讀取端假設它一定存在（舊資料或舊後端會壞） | 🟡 |
| 新功能依賴新後端，但新版 PC 端遇到舊後端回應時沒有退路 | 🟡 |
| 新增 API 或設定檔欄位，沒有在 README 或註解說明版本差異 | 🟢 |

## 新功能的相容性設計原則

1. **後端先行，用戶端後上**：先部署能同時服務新舊用戶端的後端，確認舊版 PC 端正常，再發布新版 PC 端與網頁
2. **加法優先**：新增路由或欄位，不修改既有的；真要淘汰，先標註 deprecated、保留至少一個以上的發布週期，並確認已無舊版流量（看 `firebase functions:log`）
3. **功能偵測，不靠版本假設**：新版用戶端用「回應中是否有某欄位」決定行為，缺少時退回舊行為
4. **本機檔案遷移**：讀取時自動升級舊格式，寫入新格式前先備份（例如 `.bak`），不可直接覆寫導致降版後無法使用
5. **部署順序寫進 PR 說明**：哪個專案先部署、回滾時要注意什麼
