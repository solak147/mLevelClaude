# MLevel 續訂功能規劃

2026-10-11

**狀態：已上線（2026-10-11）。** 後端、網頁、PC 1.1.0 都已依序部署，部署後檢查正常。本機 emulator 測試通過（後端 48 項、PC 腳本新舊版 14 項），三燈 code review 的修正已套用。還沒在正式環境實際刷卡續訂過，網頁也還沒在瀏覽器完整操作過。實作與規劃的差異見文末「實作紀錄」。

## 目標

已經買過金鑰的使用者，可以在官網用**同一組金鑰**續訂 30 天，不必換金鑰、不必在程式裡重新輸入。

- 到期前、到期後都能續。到期前續訂時，天數接在原本的到期日後面，可疊加、不設上限
- 舊版 PC 腳本（1.0.0 與更早）不用更新就能用續訂後的金鑰
- 既有訂單、金鑰、席位資料不需要搬移

## 現況

目前沒有續訂功能，到期後只能重新購買，拿到一組新金鑰。

| 端 | 現況 | 依據 |
| --- | --- | --- |
| 後端 | `/create` 只收 email，每張訂單付款後都產生新金鑰，到期日 = 付款時間 + 30 天 | `functions/index.js:775`、`:510`、`:68` |
| 後端 | 已付款的訂單不會再更新，沒有任何延長到期日的路由 | `functions/index.js:493` |
| 後端 | 授權有效與否看「`licenseKey` 欄位等於這組金鑰的那張已付款訂單」上的 `licenseExpiresAt` | `findPaidOrder`，`functions/index.js:625` |
| 後端 | 席位文件 `mlevel_sessions` 在取得席位時**複製一份到期日**，之後心跳只看這份，不回頭查訂單 | `functions/index.js:1178`、`:671` |
| 後端 | 到期訊息寫「重新購買後即可繼續使用」 | `functions/index.js:678`、`:1047`、`:1129`、`:1417` |
| PC 腳本 | 收到 `EXPIRED` 會清掉本機快取的金鑰，下次啟動要重新輸入 | `mlevel/license_gate.py:409`、`:735` |
| PC 腳本 | 會優先顯示後端的 `error` 文字；後端沒給時才用自己的「到官網續訂後可以繼續使用」 | `mlevel/license_gate.py:383-384` |
| 網頁 | 查到過期授權只顯示「重新購買後即可再次下載」，沒有續訂入口 | `src/components/LicenseLookup.vue:143`、`LicenseResult.vue:110` |

另外，正式環境的 `MLEVEL_REQUIRE_SESSION=true`，`/verify` 一律回 426（見 `local-ecpay-test-plan.md`），所以實際上判斷到期的是 `/session/acquire` 和心跳。

## 設計

### 核心決定：原始訂單是到期日的唯一來源

金鑰第一次發出的那張訂單（以下稱**原始訂單**）繼續保存這組金鑰的 `licenseExpiresAt`。續訂付款成功時，延長原始訂單上的到期日。

這樣做的好處是 `/session/acquire`、`/download`、`/lookup`、`/verify` 都不用改查詢方式：它們本來就去讀原始訂單，到期日變了自然就生效。

**續訂訂單的 `licenseKey` 必須保持空字串。** `findPaidOrder` 和 `/lookup` 用 `licenseKey == 金鑰` 查詢，取第一張已付款的（`functions/index.js:625-627`、`:962`）。如果續訂訂單也填上同一組金鑰，查詢可能拿到續訂訂單，讀到錯的到期日。所以續訂訂單改把金鑰存在新欄位 `renewsLicenseKey`。

### 流程

```mermaid
flowchart TD
    A[官網「續訂」：輸入 Email + 金鑰] --> B[POST /lookup<br/>顯示目前到期日與續訂後到期日]
    B --> C[POST /renew<br/>驗證 Email 與金鑰相符，建立續訂訂單]
    C --> D[綠界付款]
    D --> E[/notify 或 /result → markOrderPaid]
    E --> F{orderType == renewal?}
    F -- 否 --> G[原本的發金鑰流程，不變]
    F -- 是 --> H[交易內：延長原始訂單到期日<br/>同步席位文件的到期日<br/>續訂訂單標記 paid]
    H --> I[導回官網，顯示「續訂完成，金鑰不變」與新到期日]
    I --> J[PC 腳本：執行中的下次心跳就拿到新到期日<br/>已停掉的重新啟動即可]
```

### 到期日怎麼算

```
新到期日 = max(現在, 原始訂單目前的到期日) + PRODUCT.licenseDays
```

- 已過期：從付款當下起算 30 天
- 未過期：接在原本的到期日後面，剩下的天數不會浪費

## 實作項目

共六項：後端三項、網頁兩項、PC 腳本一項，這次一起做。

### 1. 後端：新增 `POST /renew`（`functions/index.js`）

只新增路由，`/create` 完全不動。

**Request**：`{ email, licenseKey, amount?, redirectBaseUrl? }`

**處理步驟：**

1. 驗證 email 格式、`normalizeLicenseKey(licenseKey)`，失敗回 400
2. 用 `tooManyLookups(clientIp)` 限流（`functions/index.js:339`）。這支和 `/lookup` 一樣能用來猜「金鑰 + email 對不對」，所以共用同一個限流
3. `findPaidOrder(licenseKey)` 找原始訂單，比對 email（不分大小寫）。找不到或 email 不符都回同一個 404：「找不到符合的授權，請確認 Email 與金鑰是否正確。」，不區分是哪一項錯了
4. 建立續訂訂單：

```js
{
  orderId, email, amount: PRODUCT.amount, currency: 'TWD',
  productName: RENEWAL_PRODUCT.name, productType: RENEWAL_PRODUCT.type,
  provider: 'ecpay', paymentMethod: 'credit_onetime',
  status: 'pending', accessToken, redirectBaseUrl,
  licenseKey: '',                    // 一定要空字串，理由見「核心決定」
  orderType: 'renewal',              // 新欄位；舊訂單沒有這個欄位，一律視為新購
  renewsLicenseKey: licenseKey,      // 新欄位
  renewsOrderId: 原始訂單 orderId,    // 新欄位
  createdAt: serverTimestamp(),
}
```

5. 用 `buildEcpayOrder` 產生綠界表單，回傳格式和 `/create` 一樣（`ok, merchantTradeNo, accessToken, action, params`），另外加 `currentExpiresAt`

**其他：**

- `RENEWAL_PRODUCT = { ...PRODUCT, type: 'mlevel_monthly_renewal', name: 'MLevel 月訂閱續訂（30 天）' }`。金額和天數直接用 `PRODUCT` 的值，調價時只要改一個地方
- `/renew` 加進 `APPCHECK_PATHS`（`functions/index.js:350`）。這支只有網頁會呼叫，PC 腳本不會
- `GET /` 的 routes 清單加上 `'POST /renew'`

### 2. 後端：`markOrderPaid` 分支處理續訂（`functions/index.js:460`）

前面的 MerchantID、`SimulatePaid` 檢查照舊。進入交易、讀到訂單之後，**在 `data.status === 'paid' && data.licenseKey` 判斷（`:493`）之前**先分支：

```js
if (data.orderType === 'renewal') {
  return applyRenewal(tx, ref, data, payload);
}
```

> ⚠️ 一定要在 `:493` 和 `:510` 之前分支。續訂訂單的 `licenseKey` 是空字串，走到 `:510` 會被 `generateLicenseKey()` 發一組新金鑰。

`applyRenewal` 在同一個交易裡做（Firestore 交易要先讀完再寫）：

1. **讀**：續訂訂單已經是 `paid` 就直接回傳，綠界重送回呼時不會重複延長
2. **讀**：比對 `TradeAmt` 和訂單金額，跟現有邏輯一樣（`:497-507`）
3. **讀**：用 `renewsOrderId` 讀原始訂單，確認 `status === 'paid'` 而且 `licenseKey === renewsLicenseKey`。對不上就記 error log、回 `null`（`/result` 會走 review 畫面，讓使用者拿訂單編號找客服）
4. **讀**：席位文件 `mlevel_sessions/{hashLicenseKey(key)}`
5. **算**：`newExpiresAt = max(now, 原始訂單到期日) + PRODUCT.licenseDays`
6. **寫原始訂單**：`licenseExpiresAt = newExpiresAt`、`renewalCount: increment(1)`、`lastRenewedAt`、`lastRenewalOrderId`
7. **寫席位文件**（存在時才寫）：`{ licenseExpiresAt: newExpiresAt }`，用 merge
8. **寫續訂訂單**：`status: 'paid'`、`paidAt`、`transactionId`、`ecpayRawResult: pickCallbackFields(payload)`、`previousExpiresAt`、`renewedExpiresAt: newExpiresAt`

續訂訂單**不寫 `licenseExpiresAt`**，避免有人誤以為那是金鑰目前的到期日。只存 `renewedExpiresAt` 當作歷史紀錄。

### 3. 後端：讀取端配合

**(a) 心跳遇到到期時回頭查一次訂單（`checkSessionLease`，`functions/index.js:671`）**

席位文件的到期日是取得席位時抄下來的副本。第 2 項會在續訂交易裡同步更新它，不過 `/session/acquire` 是先在交易外讀訂單、再進交易寫席位（`:1114` 讀、`:1178` 寫）。如果續訂剛好在這兩步之間完成，席位又會被寫回舊的到期日。

修法是 `checkSessionLease` 判斷到期時，先用 `session.orderId` 回頭讀原始訂單：

- 訂單的到期日比較晚：更新席位文件的 `licenseExpiresAt`，照常放行
- 訂單也到期了：才回 `EXPIRED`

這段只在「看起來到期」時才多讀一次，平常的心跳成本不變。這也是一道保險：任何原因造成的席位到期日過時，都會在這裡修正。

**(b) `orderPublicView` 認得續訂訂單（`functions/index.js:714`）**

`/order` 拿續訂訂單的 `orderId + token` 查詢時（付款回跳的結果頁），要顯示金鑰和新的到期日：

- 新增 `async resolveLicenseView(data)`：續訂訂單就依 `renewsOrderId` 讀原始訂單，拿它的 `licenseExpiresAt`；其他訂單照舊
- `licenseKey`：改成 `data.licenseKey || data.renewsLicenseKey || ''`
- 新增回應欄位 `orderType`（`'purchase'` 或 `'renewal'`）、`renewedExpiresAt`。舊欄位全部保留、型別不變

**(c) `/download`（`functions/index.js:1392`）**

續訂訂單的下載連結會帶續訂訂單的 `orderId + token`，所以 `/download` 也要用 (b) 的方式讀原始訂單的到期日。不然續訂訂單沒有 `licenseExpiresAt`，`readExpiresAt` 回 `null`，會被當成「沒有期限」永遠放行。

**(d) 到期訊息改成指向續訂**

把 `:678`、`:1047`、`:1129` 的 `error` 改成：

> 授權已到期。到官網「續訂」可以用同一組金鑰再續 30 天，完成後重新輸入這組金鑰即可。

`:1417`（下載頁）改成「授權已到期，到官網續訂後即可再次下載。」

這只改文字，`reason`、狀態碼、欄位都不變。舊版 PC 腳本會優先顯示後端的 `error`（`license_gate.py:383`），所以**不更新程式也能看到續訂的指引**。訊息特別寫「重新輸入這組金鑰」，是因為舊版收到 `EXPIRED` 會清掉快取（`license_gate.py:409`）。

### 4. 網頁：續訂入口（`mlevelweb/src/`）

- `lib/ecpay.js` 新增 `createRenewalOrder(payload)`，呼叫 `POST /renew`，寫法照 `createEcpayOrder`
- 新增 `components/RenewModal.vue`，分兩步：
  1. 輸入 Email + 金鑰 → 呼叫現有的 `lookupLicense`，顯示「目前到期日」和「續訂後約到 X」（用第 2 項的公式在前端估算，標示「實際以付款時間計算」）
  2. 勾選同意條款 → `createRenewalOrder` → `sessionStorage` 存 `mlevel:lastOrder` → `submitEcpayForm`。跟 `CheckoutModal.vue` 一樣
- 入口：
  - `LicenseLookup.vue` 找到授權後（不管有沒有過期）加一顆「續訂 30 天」按鈕，帶入剛剛輸入的 Email 和金鑰開 `RenewModal`
  - `SiteHeader.vue` 在「找回授權」旁邊加「續訂」
  - `App.vue` 讀網址的 `#renew=MLV-...`，有的話直接開 `RenewModal` 並帶入金鑰（給第 6 項 PC 腳本的按鈕用）
- 價格顯示：`RenewModal` 的價格跟 `CheckoutModal.vue:7` 的 `PRICE` 共用同一個常數，避免調價時漏改

### 5. 網頁：結果頁與到期文字

- `LicenseResult.vue`：`order.orderType === 'renewal'` 時，標題改成「續訂完成」，說明「金鑰不變，已延長到 X。程式執行中會自動更新到期日，已停止的話重新啟動即可。」
- `LicenseLookup.vue:143`、`LicenseResult.vue:110`：「重新購買」改成「續訂」，並附上續訂按鈕
- `FaqSection.vue` 加一題「到期了怎麼續訂？」

### 6. PC 腳本：保留金鑰與續訂按鈕（`mlevel/`）

這一項和後端、網頁同一次上線。舊版 PC 腳本沒有這一項也能續訂（靠第 3(d) 項的訊息）；新版可以少一步重新輸入。

- `license_gate.py:409`：`EXPIRED` 不再清快取，把 `"EXPIRED"` 從清除清單移掉。過期的快取不會被誤放行：離線寬限本來就會檢查 `expiresAt`（`license_gate.py:224`）
- `_GateWindow`（`license_gate.py:595`）：`reason == "EXPIRED"` 時多一顆「到官網續訂」按鈕，開 `f"{SITE_URL}/#renew={key}"`。用 `#` 而不是 `?`，金鑰不會送到 Hosting 的存取紀錄
- `license_gate.py:384`、`updater_client.py:200` 的預設訊息跟第 3(d) 項統一
- （選做，可留到下一版）`SessionKeeper` 發現 `daysLeft <= 3` 時，主視窗頂端顯示「授權將在 X 到期，點此續訂」，沿用自動更新提示列的寫法
- 透過自動更新發布（`version.py` 升到 1.1.0）

## 相容性檢查

| 項目 | 結果 |
| --- | --- |
| API 路由 | 只新增 `/renew`；`/create`、`/notify`、`/result`、`/order`、`/lookup`、`/verify`、`/session/*`、`/update/*`、`/download` 的路徑、欄位、狀態碼都不變 |
| 回應欄位 | `/order`、`/lookup` 只新增 `orderType`、`renewedExpiresAt`；`licenseKey` 欄位還在，型別不變 |
| `valid` 欄位 | PC 會呼叫的端點回應不變 |
| 金鑰格式、雜湊、簽章 | 不動 |
| Firestore | 只新增欄位；舊訂單沒有 `orderType` 時一律當新購處理；不需要新索引（`/renew` 用的是現有的 `licenseKey ==` 查詢，續訂訂單用 doc id 讀） |
| `firestore.rules` | 不動 |
| App Check | 只加在 `/renew`（網頁專用），PC 會呼叫的路由不加 |
| CSP | 沒有新網域，不用改 `firebase.json` |
| 綠界 | 驗章、冪等、金額比對、`SimulatePaid` 檢查全部沿用；續訂訂單也走同一個 `markOrderPaid` 入口 |
| 舊版 PC 腳本 | 續訂後原始訂單的到期日延長，`/session/acquire` 讀到就放行；執行中的程式靠心跳拿到新到期日（第 2、3(a) 項）；到期訊息由後端提供，舊版直接顯示 |
| 本機檔案 | `license.json` 格式不變；第 6 項只改「什麼時候清快取」 |

## 部署順序

1. **後端**（第 1–3 項）：部署後，舊網頁、舊 PC 腳本都照常運作。新購流程不受影響，可以先用 `/create` 買一筆確認
2. **網頁**（第 4–5 項）：`npm run build` 後部署 Hosting
3. **PC 腳本**（第 6 項）：網頁上線後，打包 1.1.0，上傳到 Storage 的 `releases/`，更新 `latest.json`

**回滾注意：** 後端退回舊版時，還沒付款的續訂訂單如果之後才付款，舊版 `markOrderPaid` 會把它當新購，發一組**新金鑰**（`:510`）並給 30 天。使用者不會白付錢，但會拿到另一組金鑰。回滾前先看 Firestore 有沒有 `orderType == 'renewal' && status == 'pending'` 的訂單，有的話記下來人工處理。

## 測試

在本機 emulator 上測，環境見 `local-ecpay-test-plan.md`。綠界回呼用測試特店 HashKey 自己簽 CheckMacValue。

**後端**

- [ ] `/renew`：email 錯、金鑰錯都回同一個 404；格式錯回 400；連打會被限流
- [ ] 已過期的金鑰續訂 → 到期日 = 付款時間 + 30 天
- [ ] 還有 10 天的金鑰續訂 → 到期日 = 原到期日 + 30 天
- [ ] 同一筆續訂的 `/notify` 和 `/result` 都打、各打兩次 → 只延長一次
- [ ] 續訂訂單 `TradeAmt` 不符、`SimulatePaid=1` → 不延長
- [ ] 續訂後原始訂單的 `licenseKey`、`accessToken` 沒變；續訂訂單的 `licenseKey` 仍是空字串
- [ ] `/lookup` 用原本的 Email + 金鑰查，回傳原始訂單，而且到期日是新的
- [ ] `/order` 用續訂訂單的 token 查，看得到金鑰、`orderType: 'renewal'`、新到期日
- [ ] `/download` 用續訂訂單的連結：未過期可下載；把原始訂單到期日改成過去後被擋
- [ ] 新購流程（`/create` → 付款 → 發新金鑰）完全不變

**席位**

- [ ] 程式執行中續訂 → 席位文件的 `licenseExpiresAt` 同步更新
- [ ] 把席位文件的到期日手動改成過去、原始訂單是未來 → 心跳照常放行，而且席位文件被修正（第 3(a) 項）
- [ ] 原始訂單和席位都過期 → 心跳回 `EXPIRED`

**PC 腳本（用 `MLEVEL_API_BASE` 指向 emulator）**

- [ ] **舊版 1.0.0**：過期 → 看到新的續訂訊息 → 續訂 → 重新輸入同一組金鑰 → 可以啟動
- [ ] 舊版 1.0.0 執行中續訂 → 越過原本的到期時間後還能繼續跑
- [ ] 新版 1.1.0：過期後重開，金鑰還在輸入框裡；按「到官網續訂」開的網址帶入金鑰；續訂完按「驗證並啟動」直接通過

**網頁**

- [ ] 找回授權 → 續訂 → 綠界測試站付款 → 結果頁顯示「續訂完成」與新到期日
- [ ] `#renew=MLV-...` 開啟續訂視窗並帶入金鑰

## 已決定事項（2026-10-11）

1. **提前續訂可疊加，不設上限。** 新到期日一律是 `max(現在, 原到期日) + 30 天`，`/renew` 不檢查原到期日離現在多遠
2. **續訂限定原購買 Email。** 跟 `/lookup` 一致，`/renew` 也不會變成探測金鑰的工具
3. **續訂價格和新購相同（NT$399）。** 金額直接用 `PRODUCT.amount`，`RENEWAL_PRODUCT` 只改 `type` 和 `name`
4. **PC 腳本（第 6 項）這次一起做。** 照部署順序最後上線：後端 → 網頁 → 1.1.0 自動更新
5. **到期提醒信不在這次範圍內**：目前後端沒有寄信功能

## 實作紀錄（2026-10-11）

### 改動的檔案

| 專案 | 檔案 | 內容 |
| --- | --- | --- |
| functions | `index.js` | `RENEWAL_PRODUCT`、`extendExpiresAt`、`EXPIRED_MESSAGE`；`POST /renew`；`markOrderPaid` 在發金鑰前分出 `applyRenewal`；`paidAmountMatches` 抽出共用；`checkSessionLease` 到期前呼叫 `refreshSessionExpiry`；`licenseSourceFor` 給 `/order`、`/download` 用；`findOwnedPaidOrder` 由 `/lookup`、`/renew` 共用；`orderPublicView` 新增 `orderType`、`renewedExpiresAt` |
| 網頁 | `src/lib/product.js`（新） | `PRICE`、`LICENSE_DAYS`，`CheckoutModal` 與 `RenewModal` 共用 |
| 網頁 | `src/lib/ecpay.js` | `createRenewalOrder` |
| 網頁 | `src/components/RenewModal.vue`（新） | 兩步驟：`/lookup` 確認 → 顯示目前與續訂後到期日 → `/renew` → 綠界 |
| 網頁 | `App.vue`、`SiteHeader.vue`、`SiteFooter.vue`、`LicenseLookup.vue`、`LicenseResult.vue`、`FaqSection.vue` | 續訂入口（頁首、頁尾、找回授權、到期的結果頁）、`#renew=金鑰`、續訂完成畫面、FAQ |
| PC | `license_gate.py` | `EXPIRED` 不清快取；`GateResult.expired`；`renew_url()`；授權視窗「到官網續訂」按鈕；`SessionKeeper.expired()` |
| PC | `mlevel.py` | 執行中到期時詢問是否開官網續訂 |
| PC | `updater_client.py` | 到期訊息改成續訂 |
| 工作區 | `.claude/skills/tri-light-review/mlevel-context.md` | 路由表加 `/renew`、Firestore 段落加「到期日只存在原始訂單」 |

### 和規劃不同的地方

- **沒有到期日的原始訂單不給續訂**：`/renew` 回 409，`applyRenewal` 也擋一次。理由：沒有到期日的授權不會過期，延長反而會把它縮成 30 天。目前的 `markOrderPaid` 一定會寫到期日，正常資料不會碰到
- **續訂訂單的 email 用原始訂單的值**，不用使用者輸入的大小寫，`/lookup` 比對才一致
- **頁尾也加了「續訂」**：頁首的選單在 720px 以下會隱藏，手機只能從頁尾進入
- **主程式執行中到期**：除了授權視窗，主視窗的「授權已失效」也改成問「要現在打開官網續訂嗎？」
- `/download` 遇到找不到原始訂單的續訂連結，回 404 `LICENSE_NOT_FOUND`（新的錯誤代碼，只出現在 HTML 錯誤頁）
- **收了錢卻續不上時，在續訂訂單寫 `renewalError`**（`MISSING_REFERENCE`／`TARGET_MISMATCH`／`NO_EXPIRY`）、`renewalErrorAt`、`transactionId`、`ecpayRawResult`，訂單仍是 `pending`。客服在 Firestore 查 `renewalError` 就能找出這些訂單。
  `/notify` 已經回 `1|OK`，綠界**不會重送**；我們也不存 `CheckMacValue`，沒辦法重打回呼（自己用正式 HashKey 簽一份等於偽造綠界通知，不可以這樣做）。所以要人工處理：
  1. 先到綠界後台用 `transactionId` 確認款項確實入帳
  2. 找到原始訂單（`licenseKey` 等於續訂訂單的 `renewsLicenseKey`），把 `licenseExpiresAt` 設成 `max(現在, 原到期日) + 30 天`，`renewalCount` 加 1，`lastRenewedAt` 設成現在，`lastRenewalOrderId` 設成續訂訂單編號
  3. 續訂訂單改成 `status: 'paid'`，補上 `paidAt`、`previousExpiresAt`、`renewedExpiresAt`，刪掉 `renewalError`、`renewalErrorAt`
  4. 席位文件（`mlevel_sessions`，文件 ID 是金鑰的雜湊）的 `licenseExpiresAt` 建議一起改成新日期。沒改也不會中斷使用：等席位上的舊到期日過了，下一次心跳就會由 `refreshSessionExpiry` 從原始訂單同步；只是在那之前，程式顯示的還是舊日期

  程式裡清掉 `renewalError` 的分支只在一種情況會走到：`/notify` 續不上之後，使用者的 `/result` 才進來，而且資料剛好已經修好。
- 到期訊息改成「完成後用這組金鑰重新驗證即可」，新版（保留金鑰）和舊版（清掉金鑰）都說得通

### 測試

整合測試放在 `mlevelweb/scripts/renew-test.cjs`，用法見 `local-ecpay-test-plan.md` 的「自動化：續訂功能整合測試」。PC 端測試沒有放進 repo：它需要把舊版 `license_gate.py` 從 git 取出來比對，而且要改 `LOCALAPPDATA`，比較適合照測試清單手動測。

### 部署紀錄（2026-10-11）

| 順序 | 項目 | 結果 |
| --- | --- | --- |
| 1 | functions（`mlevelweb` `2f69576`） | revision `ecpayapi-00017-niy`。`GET /` 出現 `POST /renew`；`/renew`、`/order` 沒帶 App Check 回 401；`/verify` 仍回 426；`/session/acquire`、`/session/heartbeat` 行為不變；log 無錯誤 |
| 2 | 網頁 Hosting | 打包檔 `index-B8RC9ouX.js` 指向正式 API、沒有本機網址，含續訂功能；CSP `connect-src` 允許 Functions 網域 |
| 3 | PC 1.1.0（`mlevel` `b44e802`） | `releases/latest.json` 為 1.1.0；`releases/mlevel-1.1.0-update.zip`、`mlevel.zip` 的 MD5 與本機一致，更新包 SHA-256 與 `latest.json` 相同；沒有設 `minVersion`（只提示、不強制） |

第一次發版時 `build.ps1` 在繁中 Windows（主控台 cp950）讀 gcloud 輸出失敗：線上 `latest.json` 的中文 notes 被讀成亂碼，`ConvertFrom-Json` 中止（沒有上傳任何東西）。已修正為呼叫 gcloud 期間暫時切成 UTF-8（`mlevel` `13325fd`），重發成功。

### 尚未完成

- [ ] 在正式環境用自己的金鑰實際續訂一次（付款 → 「續訂完成，金鑰不變」→ 到期日延長 30 天；`firebase functions:log` 有 `License renewed`）
- [ ] 在瀏覽器完整操作網頁（到期結果頁 → 續訂時背景不可捲動、`#renew=` 帶入金鑰、手機寬度的頁尾入口）
- [ ] `firebase-functions` 套件升級（部署時 CLI 提示版本過舊；升級後要重跑 `scripts/renew-test.cjs`）
