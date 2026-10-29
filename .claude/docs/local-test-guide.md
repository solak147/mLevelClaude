# MLevel 本機測試環境指令

2026-10-29

只列架本機測試環境要下的指令。設計理由與踩過的坑見 `local-ecpay-test-plan.md`、`cvs-payment-email-plan.md`。

本機環境只連本機的 Functions／Firestore emulator，綠界用公開測試特店（payment-stage），不會動到正式資料。

```
vite dev（localhost:5173） ─┐
                            ├─> Functions emulator（127.0.0.1:5001） ─> Firestore emulator（127.0.0.1:8080）
PC 腳本（MLEVEL_API_BASE）─┘
```

## 事前準備（只需做一次）

```powershell
npm install -g firebase-tools
firebase login
gcloud auth application-default login     # emulator 讀正式 Storage（下載、自動更新）才需要
winget install Microsoft.OpenJDK.21        # Firestore emulator 需要 Java 21 以上

cd D:\mLevelClaude\mlevelweb
npm install
npm --prefix functions install

# PC 端從原始碼執行需要的套件
C:\Users\User\AppData\Local\Programs\Python\Python312\python.exe -m pip install pyautogui pydirectinput opencv-python numpy pillow
```

本機覆寫檔（都已建好，不進版控、不會被部署）：

| 檔案 | 用途 |
| --- | --- |
| `mlevelweb/functions/.env.local` | emulator 專用，覆寫 `.env`：綠界測試特店、回呼網址指向本機、寄信改 json |
| `mlevelweb/functions/.secret.local` | emulator 用的 `GMAIL_APP_PASSWORD`（平常是 `dummy`） |
| `mlevelweb/.env.development.local` | `vite dev` 專用：`VITE_ECPAY_API_URL` 指向本機 emulator |

**不要改 `functions/.env`**，那是正式部署用的。

## 1. 後端 emulator（視窗 1）

```powershell
cd D:\mLevelClaude\mlevelweb
firebase emulators:start --only functions,firestore --import=./emu-data --export-on-exit
```

- **一定要帶 `firestore`**。不要用 `functions/` 的 `npm run serve`（只有 functions），那樣測試訂單會寫進正式 Firestore。
- `--import` / `--export-on-exit` 用來保留測試資料；`emu-data` 資料夾還不存在時先拿掉 `--import=./emu-data`。
- 停止時在這個視窗按 `Ctrl+C`，資料才會正常匯出。
- Emulator UI：http://127.0.0.1:4000（看 Firestore 資料與 Functions log）
- API 位址：`http://127.0.0.1:5001/mlevel-f575a/asia-east1/ecpayApi`

## 2. 網頁（視窗 2）

```powershell
cd D:\mLevelClaude\mlevelweb
npm run dev            # 經典版，http://localhost:5173
npm run dev:worlds     # 楓星版
```

測試信用卡：`4311-9522-2222-2222`，有效期限填未來日期，CVV `222`，3D 驗證碼看綠界測試頁提示。

## 3. PC 端腳本（視窗 3）

```powershell
$env:MLEVEL_API_BASE = "http://127.0.0.1:5001/mlevel-f575a/asia-east1/ecpayApi"
$env:LOCALAPPDATA = "$env:TEMP\mlevel-localtest"
cd D:\mLevelClaude\mlevel
C:\Users\User\AppData\Local\Programs\Python\Python312\python.exe mlevel.py
```

- emulator 發的金鑰只存在本機 Firestore，所以 PC 端一定要指向本機。
- 改 `LOCALAPPDATA` 是為了不讓測試金鑰覆蓋這台電腦的正式授權快取。
- 測楓星版：另外加 `$env:MLEVEL_EDITION = "worlds"`（只對原始碼執行有效）。
- 環境變數只在這個 PowerShell 視窗有效，關掉就恢復。

## 4. 自動化測試腳本

emulator 啟動後，在 `D:\mLevelClaude\mlevelweb`：

```powershell
node scripts/renew-test.cjs       # 新購、續訂、冪等、席位、下載
node scripts/edition-test.cjs     # 經典版／楓星版分開販售
node scripts/cvs-mail-test.cjs    # 超商代碼付款 + 付款成功信（json，不真的寄出）
```

腳本自己簽綠界回呼，只打 `127.0.0.1`；`functions/.env.local` 不是 payment-stage 時會直接中止。

## 5. 用 Cloudflare 通道讓綠界打到本機

綠界伺服器的回呼（`/notify`、超商取號通知、模擬付款）連不到 `127.0.0.1`，要開通道。cloudflared 已裝在 `C:\Program Files (x86)\cloudflared\`（沒有的話 `winget install Cloudflare.cloudflared`）。

1. 開通道（視窗 4），記下印出來的 `https://<隨機>.trycloudflare.com`：

   ```powershell
   cloudflared tunnel --no-autoupdate --url http://127.0.0.1:5001
   ```

2. 改 `mlevelweb/functions/.env.local`：

   ```
   ECPAY_CALLBACK_BASE_URL=https://<隨機>.trycloudflare.com/mlevel-f575a/asia-east1/ecpayApi
   ECPAY_ALLOW_SIMULATED_PAYMENT=true
   ```

3. 視窗 1 按 `Ctrl+C`，重新啟動 emulator（指令同第 1 步）。
4. 在網頁**重新下單**（通知網址是建單時寫進訂單的，舊訂單不會變）。
5. 超商代碼訂單：到綠界測試站廠商後台 → 訂單 → 按「模擬付款」。
6. 測完還原並關閉通道（視窗 4 按 `Ctrl+C`）。通道開著時，任何人拿到網址都打得到本機 emulator：

   ```
   ECPAY_CALLBACK_BASE_URL=http://127.0.0.1:5001/mlevel-f575a/asia-east1/ecpayApi
   ECPAY_ALLOW_SIMULATED_PAYMENT=false
   ```

   還原後重啟 emulator。

## 6. 測信件

### 不真的寄出（預設）

`functions/.env.local` 預設是：

```
MLEVEL_MAIL_ENABLED=true
MLEVEL_MAIL_TRANSPORT=json
GMAIL_USER=test@example.com
```

付款成功信不會寄出，內容寫進 emulator 的 Firestore（outbox 文件）與 Functions log，到 Emulator UI 看。用 `node scripts/cvs-mail-test.cjs` 自動檢查。

### 正式環境的 Gmail 設定

本機要真的寄信時，可以沿用正式環境的寄件帳號與密碼。這份文件會進版控，所以密碼只寫存放位置。

| 項目 | 正式環境 |
| --- | --- |
| 寄件帳號 `GMAIL_USER` | `mlevelcpy@gmail.com`（寫在 `mlevelweb/functions/.env`） |
| 寄信開關 `MLEVEL_MAIL_ENABLED` | `true`（寫在 `mlevelweb/functions/.env`） |
| 應用程式密碼 `GMAIL_APP_PASSWORD` | Secret Manager：https://console.cloud.google.com/security/secret-manager/secret/GMAIL_APP_PASSWORD/versions?project=mlevel-f575a |
| 應用程式密碼的管理頁 | https://myaccount.google.com/apppasswords（用 `mlevelcpy@gmail.com` 登入） |
| 寄信的 Function | `sendOrderMail`（`asia-east1`），付款成功時 `mlevel_mail_outbox/{訂單}_paid` 建立就觸發 |

取出正式密碼（在 `D:\mLevelClaude\mlevelweb`）：

```powershell
firebase functions:secrets:access GMAIL_APP_PASSWORD
```

- 本機的 `.secret.local` 和正式的 Secret Manager 是兩份，改本機不會影響正式，反之亦然。
- 在管理頁撤銷這組應用程式密碼的話，正式環境會寄不出信。要換密碼：`firebase functions:secrets:set GMAIL_APP_PASSWORD`，再 `firebase deploy --only functions`。

### 用真的 Gmail 寄出

1. 改 `mlevelweb/functions/.env.local`：

   ```
   MLEVEL_MAIL_TRANSPORT=
   GMAIL_USER=mlevelcpy@gmail.com
   ```

2. 改 `mlevelweb/functions/.secret.local`，填上面 `secrets:access` 取出的值：

   ```
   GMAIL_APP_PASSWORD=<16 碼應用程式密碼，去掉空白>
   ```

3. 視窗 1 按 `Ctrl+C`，重新啟動 emulator。
4. 寄一封付款成功信（建一筆信用卡訂單、自簽付款回呼）：

   ```powershell
   cd D:\mLevelClaude\mlevelweb
   node scripts/mail-smoke.cjs 收件信箱@example.com
   ```

   要測完整流程的話，搭配第 5 節的通道，在網頁下一筆超商代碼訂單、到綠界後台按「模擬付款」，付款成功後就會寄信。

5. 測完還原，再重啟 emulator（沒還原前 `cvs-mail-test.cjs` 會拒絕執行）：

   ```
   # functions/.env.local
   MLEVEL_MAIL_TRANSPORT=json
   GMAIL_USER=test@example.com
   # functions/.secret.local
   GMAIL_APP_PASSWORD=dummy
   ```

信裡的下載連結指向本機 emulator（或通道網址），只有通道開著／在這台電腦上才點得開。
