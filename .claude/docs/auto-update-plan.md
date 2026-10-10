# MLevel 自動更新功能規劃

2026-10-10 · 線上版：https://claude.ai/code/artifact/068fc666-5868-42a1-8775-d8984fec0faf

**狀態：已完成（2026-10-10）。** 八個實作項目與第 6 步測試都做完；後端已部署，並在正式環境實測過 1.0.0 → 1.0.1（`D:` 與 Program Files 各一次）。剩下的是收尾，見文末。

## 目標

有新版本時，主程式提示使用者；點一下就自動下載、驗證並換上新版，全程有進度條。失敗時彈出錯誤視窗，讓使用者選擇重新更新、到官網下載，或先不更新繼續用目前版本。

- 使用者的設定、地圖、怪物圖、模板、路線與日誌，更新後完整保留
- 任何一步失敗都能退回原版，不會讓程式變成打不開
- 更新檔只給持有有效授權的人下載

## 現況與限制

程式執行中無法覆蓋自己的 exe 與 DLL，所以換檔必須交給另一支程式。以下是從目前程式碼看到的影響。

| 現況 | 對更新的影響 |
| --- | --- |
| PyInstaller onedir：`mlevel.exe` + `_internal\`（`mlevel.spec`） | 執行中檔案被鎖，需要獨立的 `updater.exe` 換檔 |
| `config/ maps/ monster/ templates/` 放在 exe 旁邊 | 更新絕不碰這四個資料夾與 `log/` |
| `uac_admin=True` | 主程式帶起的 updater 沿用管理員權限，可寫入 Program Files |
| GUI 用 tkinter、網路用 urllib | 不需加新套件 |
| 後端 `/download` 用訂單 `orderId + token` 驗證 | 程式只有金鑰與 `sessionId`，需要另開更新端點 |
| 目前沒有版本號 | 第一步先補上 |

## 整體流程

```mermaid
flowchart TD
    A[主視窗開啟後，背景檢查新版本] --> B[頂端提示「有新版本，點此更新」]
    B --> C[下載 → 驗證 SHA-256 → 解壓（進度條）]
    C --> D[釋放席位 → 啟動 updater.exe → 結束]
    D --> E[updater：舊版改名備份 → 換上新版]
    E --> F[啟動新版，通過授權後刪除備份]
    C -- 失敗 --> X[錯誤視窗<br/>重新更新 / 前往官網下載 / 先不更新]
    X -- 重新更新 --> C
    E -- 換檔失敗 --> R[還原備份，啟動舊版]
    R -- 帶 --update-failed 重開 --> X
```

下載或驗證失敗時，程式還沒動到任何檔案；換檔失敗時，updater 把備份改回原名再開舊版。兩種情況最後都會停在錯誤視窗，由使用者決定下一步。

## 實作項目

共八項，橫跨 mlevel 與 mlevelweb 兩個專案。

### 1. 版本號（mlevel）

- 新增 `version.py`：`__version__ = "1.4.0"`，`mlevel.py` 讀它並顯示在標題列
- `build.ps1` 讀同一個版本號，產出 `mlevel-1.4.0.zip`

### 2. 後端端點（mlevelweb/functions）

- `POST /update/check`：傳 `{key, sessionId, version}`，回 `{latest, minVersion, notes, size, sha256, publishedAt}`
- `POST /update/download`：傳 `{key, sessionId}`，回短效期的 Storage 簽名網址。要有有效席位才拿得到
- 版本資訊（manifest）放 Storage 的 `releases/latest.json`（實作時選定，不用 Firestore）
- Storage 另放 `releases/mlevel-<ver>-update.zip`：`mlevel.exe`、`_internal`、`mlevel-updater.exe`，加上 `defaults/`（monster、templates 預設檔）。原本的 `mlevel.zip` 照舊給新使用者完整安裝

### 3. 檢查更新（新檔 `updater_client.py`）

- 主視窗開啟後在背景執行緒檢查一次，之後每 24 小時一次；連不上就安靜跳過
- 有新版時，主視窗頂端顯示「有新版本 v1.5.0，點此更新」
- 打怪進行中只提示、不能按更新，要先停止
- 「略過此版」記在 `%LOCALAPPDATA%\mlevel\update.json`
- 低於 `minVersion`（強制更新）時，只留「更新」與「前往官網」

### 4. 更新視窗與進度條

- tkinter Toplevel + `ttk.Progressbar`，文字如「下載中 12.3 / 45.6 MB　1.2 MB/s　剩約 28 秒」
- 階段：下載 → 驗證 SHA-256 → 解壓 → 準備安裝
- 下載在背景執行緒分塊讀，經 `queue` + `after()` 更新畫面，和 `_GateWindow` 同一套做法
- 有「取消」按鈕，取消時刪掉下載到一半的檔案
- 下載前檢查磁碟空間，需有 zip 大小的 3 倍

### 5. 更新失敗的錯誤視窗

自訂 Toplevel（messagebox 放不了三個自訂按鈕），按鈕為：**重新更新**、**前往官網下載**（`webbrowser.open(SITE_URL)`）、**先不更新，繼續使用**。詳細錯誤寫進 `log/update.log`。

| 錯誤 | 顯示的訊息 |
| --- | --- |
| 連線失敗 / 逾時 | 網路連線中斷，請確認網路後重試 |
| SHA-256 不符 | 下載的檔案不完整或已損毀 |
| 403 授權到期 | 授權已到期，續約後才能更新 |
| 磁碟空間不足 | 剩餘空間不足，需要約 N MB |
| 檔案被鎖住 | 有程式正在使用檔案（可能是防毒軟體），請稍後重試 |

### 6. updater.exe（換檔程式）

- 在 `mlevel.spec` 多編一支 onefile 小 exe，不包 cv2/numpy，只用 tkinter 顯示小進度窗
- 執行前先複製到 `%LOCALAPPDATA%\mlevel\updates`，因為它本身也在要被換掉的安裝資料夾裡
- 參數：`--pid <主程式PID> --install <安裝資料夾> --staging <解壓好的新版> --old-version 1.4.0`
- 主程式啟動 updater 前先釋放席位（`keeper.stop()`），再結束自己

換檔步驟：

1. 等主程式 PID 結束，最多 30 秒
2. 把 `mlevel.exe` 與 `_internal` 改名成 `_backup_1.4.0\`（改名比複製快，也好還原）
3. 把新版搬進安裝資料夾
4. 啟動新版 `mlevel.exe --updated-from 1.4.0`

任一步失敗：把備份改回原名，啟動舊版並帶 `--update-failed "<原因>"`，主程式看到就彈出第 5 項的錯誤視窗。新版正常啟動並通過授權後，才刪掉 `_backup_*`。

實作時另外加上的：

- 換檔前先把 `defaults/` 裡使用者缺的模板補進去（已有的不覆蓋）
- 新版啟動後觀察 10 秒，一開就異常結束就還原舊版；新版等 20 秒才刪備份，確保觀察期間備份還在
- 換檔途中被中斷（停電、強制關機）：換檔前寫 `_update_journal.json` 並在 `RunOnce` 登記 `--recover`，下次登入自動照紀錄還原舊版

### 7. 使用者資料

- `config/ maps/ monster/ templates/ log/` 一律不覆蓋
- 新版附帶的新預設模板，只補上原本不存在的檔案
- 新設定欄位由程式預設值補上，不改使用者的 json

### 8. 發版流程（`build.ps1 -Release`）

1. 打包
2. 產出精簡更新包並計算 SHA-256
3. 上傳 Storage
4. 最後才更新 manifest。順序不能反，否則使用者會拿到還沒上傳完的檔案

用 `gcloud storage` 依序上傳 更新包 → `latest.json` → `mlevel.zip`，任何一步失敗就停。另外會擋版號沒有比 Storage 上新、上傳前要輸入版號確認、上傳後讀回核對，`-DryRun` 只演練。「真的上傳成功」這段要等第一次正式發版（版號至少 1.0.2）才會跑到。

## 開發順序與測試

原計畫分六個 PR；實際照指示每一步直接 commit 到 `main`。後端沒有用 emulator，改用 Node 腳本模擬 Firestore 與 Storage 測過，再部署到正式環境驗證。

1. 版本號 + `build.ps1` 產出更新包與 SHA-256
2. 後端兩支端點 + manifest（先用 emulator，`MLEVEL_API_BASE` 已支援切換）
3. 檢查更新 + 提示 UI
4. 下載視窗、進度條、錯誤視窗（這階段先不實際換檔）
5. `updater.exe` 換檔與還原
6. 實測失敗情境

第 6 步要測的情境：

- [x] 下載中斷網（本機模擬）
- [x] SHA-256 不符（本機模擬）
- [x] 檔案被鎖（真實 Windows 檔案鎖 + 打包好的 exe）
- [x] 換檔中途強制關閉 updater（5 個中斷點模擬 + 打包好的 exe 實際修復）
- [x] 授權到期時按更新（本機模擬）
- [x] 安裝在 Program Files 的情況（正式環境實測）
- [x] 更新後 config、路線、模板完整保留（正式環境實測兩次）

## 待決定事項

- [x] **已決定**：做強制更新，但預設不強制；發版時加 `-MinVersion` 才生效
- [x] **已決定**：更新包綁授權，放在 Firebase Storage。只有持有有效金鑰、正在使用中的人拿得到下載網址；使用者更新後不必重新輸入金鑰
- [x] **已決定**：提示只出現在主視窗頂端，授權視窗不提示

## 收尾

- [ ] 刪掉 Firestore 的測試訂單 `TEST20261010`（測試金鑰隨之失效）
- [ ] 恢復管理者模式：`%LOCALAPPDATA%\mlevel\license.master-backup.json` 改回 `license.json`
- [ ] 刪掉測試資料夾：`C:\Program Files\MLevelTest`、`D:\mLevelClaude\mlevel-test-1.0.0`、`mlevel\dist\upload-1.0.1`
- [ ] 兩個 repo push 到 GitHub
- [ ] 第一次正式發版的版號至少 1.0.2：`releases/latest.json` 目前指向測試版 1.0.1
