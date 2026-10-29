# MLevel 發版指令

2026-10-29

只列更版時要下的指令。細節見 `mlevel/README.md` 的「發布新版（自動更新）」、`mlevelweb/README.md` 的「桌面程式的自動更新」。

## 順序

三個專案都要更新時，照這個順序：

1. **後端** `mlevelweb/functions/`：先部署，確認舊版 PC 端照常能驗證、心跳、檢查更新
2. **PC 端** `mlevel/`：發布新版
3. **網頁** `mlevelweb/`：部署前端

只改其中一個就只做那一段。

## 事前準備（只需做一次）

```powershell
npm install -g firebase-tools
firebase login                 # 有 mlevel-f575a 專案權限的帳號
gcloud auth login              # PC 端發版要用，需先裝 Google Cloud CLI
```

## 1. 後端 Functions

在 `D:\mLevelClaude\mlevelweb`：

```powershell
cd D:\mLevelClaude\mlevelweb
npm --prefix functions install   # 改過 functions/package.json 才需要
npm run deploy:functions         # = firebase deploy --only functions
npm run logs                     # 部署後看 ecpayApi 的 log
```

- 部署會帶上 `functions/.env`（不進版控），部署前確認裡面是正式環境的值：`ECPAY_API_URL` 指正式站、`ECPAY_ALLOW_PUBLIC_TEST_CREDENTIALS`、`ECPAY_ALLOW_SIMULATED_PAYMENT` 為 `false`。
- 新增 Secret 時（例如 `GMAIL_APP_PASSWORD`）要先設好再部署：`firebase functions:secrets:set GMAIL_APP_PASSWORD`
- 改了 `firestore.rules` 或 `firestore.indexes.json`：`firebase deploy --only firestore`

## 2. PC 端腳本

在 `D:\mLevelClaude\mlevel`：

1. 把 `version.py` 的 `__version__` 往上加（`X.Y.Z`，必須比線上新，否則 `build.ps1` 會擋下）。
2. commit 程式碼（沒 commit 的修改也會被打包，腳本只會提醒）。
3. 發版：

```powershell
cd D:\mLevelClaude\mlevel

# 先演練，只列出會上傳什麼
powershell -ExecutionPolicy Bypass -File .\build.ps1 -Release -Notes "這版改了什麼" -DryRun

# 經典版
powershell -ExecutionPolicy Bypass -File .\build.ps1 -Release -Notes "這版改了什麼"

# 楓星版（分開販售，要發就另外跑一次）
powershell -ExecutionPolicy Bypass -File .\build.ps1 -Edition worlds -Release -Notes "這版改了什麼"
```

- 加 `-MinVersion 1.2.0`：低於這版的使用者會被強制更新。
- 上傳前會要求輸入版號確認；輸入不符就取消，什麼都不傳。
- 上傳順序與位置（腳本自動處理）：

| 版本 | 更新包 | 最新版資訊 | 官網下載的完整安裝檔 |
| --- | --- | --- | --- |
| 經典版 | `releases/mlevel-<ver>-update.zip` | `releases/latest.json` | `mlevel.zip` |
| 楓星版 | `releases/star/mlevel-<ver>-update.zip` | `releases/star/latest.json` | `star/mlevel.zip` |

- 後端最晚 1 分鐘生效，不需重新部署 Functions。
- 只打包、不上傳：`.\build.ps1`（產出 `dist\mlevel\`）或 `.\build.ps1 -Zip`（另外產出完整包與更新包）。

### 使用手冊

`build.ps1` 不會上傳使用手冊，改了手冊要手動覆蓋 Storage 上的檔案：

```powershell
cd D:\mLevelClaude\mlevel
gcloud storage cp .\使用手冊.md "gs://mlevel-f575a.firebasestorage.app/使用手冊.md" --content-type="text/markdown; charset=utf-8" --custom-metadata="firebaseStorageDownloadTokens=$([guid]::NewGuid())"
gcloud storage cp .\使用手冊_楓星版.md "gs://mlevel-f575a.firebasestorage.app/star/使用手冊.md" --content-type="text/markdown; charset=utf-8" --custom-metadata="firebaseStorageDownloadTokens=$([guid]::NewGuid())"
```

下載 token 的用途：後端簽不出網址時靠它組下載連結（同 `build.ps1` 上傳安裝檔的做法）。

## 3. 販售網頁

在 `D:\mLevelClaude\mlevelweb`：

```powershell
cd D:\mLevelClaude\mlevelweb
npm install                       # 改過 package.json 才需要

npm run deploy:hosting            # 經典版網站（build → dist/ → hosting:classic）
npm run deploy:hosting:worlds     # 楓星版網站（build:worlds → dist-worlds/ → hosting:worlds）

npm run deploy                    # 全部：兩個網站 + Functions + Firestore 規則
```

- `.env`（前端，`VITE_` 開頭的變數）會在 build 時打進 bundle，部署前確認是正式值。
- 部署前可先本機預覽：`npm run build` 後 `npm run preview`。

## 發版後確認

- `npm run logs`：看有沒有錯誤，以及舊版 PC 端的 `/verify`、`/session/*`、`/update/*` 是否正常。
- 開一台舊版 PC 端，確認會跳出更新提示並能更新成功。
- 官網「找回授權」→ 下載，確認拿到的是新版安裝檔。
