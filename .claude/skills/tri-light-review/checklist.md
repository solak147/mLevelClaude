# 審查檢查清單

每項後方標示的燈號為**預設等級**，實際依觸發機率與影響範圍調整（見 SKILL.md 升降級原則）。
第 0 節是 mLevel 技術棧專屬項目，優先檢查；第 1–9 節是通用項目。與 `mlevel-context.md` 衝突時以 context 為準。

## 0. mLevel 技術棧專屬（優先檢查）

### PC 端（Python + Tkinter + PyInstaller）
- [ ] 打包後的路徑：讀寫 config、log、模板時以 `sys.executable` 的資料夾為準（`mlevel.py:51` 的做法），
      不可用 `Path(__file__)`，打包後它指向會被清掉的暫存資料夾 — 🔴
- [ ] 新增的資料夾有同步加到 `mlevel.spec` 的 `data_dirs` 與 `build.ps1` 的 `$dataDirs` — 🔴
- [ ] 在既有資料夾（`templates/`、`monster/`、`maps/`、`config/`）**修改**既有檔案，或在 `config/` 新增檔案時，要手動同步到
      `dist\mlevel\<資料夾>`：`build.ps1:139-158` 只在 dist 裡沒有該資料夾時才整個複製；資料夾已存在時，
      `maps/`、`monster/`、`templates/` 只補 dist 缺的新檔、不覆蓋已有的檔案，`config/` 完全不補 — 🔴
- [ ] Tkinter 元件只在主執行緒操作（包括 `after()` 本身）。背景執行緒只把結果放進 `queue.Queue`，
      主執行緒用 `after()` 定時取出（`license_gate.py:644-657` 的 `_work`／`_poll` 做法）— 🔴/🟡
- [ ] 讀 JSON 設定時，缺欄位用預設值、壞檔不崩潰，寫入時先寫暫存檔再改名 — 🟡
- [ ] HTTP 請求沿用標準函式庫 `urllib`（`license_gate.py` 的慣例），不引入 `requests` — 🟢
- [ ] 新增的 import 不在 `mlevel.spec` 的 `excludes` 清單內（目前排除 `sqlite3`、`setuptools`、`matplotlib`、`scipy`、`pandas` 等），
      否則打包後的 exe 會 `ModuleNotFoundError`；新增第三方套件要用打包版實際跑過 — 🔴

### 後端（Cloud Functions + Firestore）
- [ ] 讀取後再寫入的操作（發金鑰、搶席位）放在 `db.runTransaction` 裡 — 🔴
- [ ] 新增的查詢會不會增加 Firestore 讀寫次數（心跳、`/verify` 這類高頻端點特別注意成本）— 🟡
- [ ] 回應一律有明確的 HTTP 狀態碼與 JSON，PC 端會呼叫的端點一律帶 `valid` 欄位 — 🔴
- [ ] log 不記錄完整金鑰、email、卡號，比照 `describeCallback` 只記可公開的欄位 — 🔴

### 前端（Vue + Vite + Firebase Hosting）
- [ ] `VITE_*` 變數會打包進公開 JS，不可放任何機密 — 🔴
- [ ] 新增的外部網域有同步加進 `firebase.json` 的 CSP — 🔴
- [ ] 價格、商品資訊和後端 `PRODUCT` 一致 — 🟡

## 1. 正確性與資料安全

- [ ] 邏輯符合需求；邊界值（空集合、0、單一元素、極大量）處理正確 — 🔴/🟡
- [ ] 刪除、覆寫、移動、`rm -rf`、`Remove-Item -Recurse`、`DROP`、`git push --force` 等破壞性操作：路徑變數為空時不會變成操作根目錄（如 `rm -rf "$DIR/"*` 在 `$DIR` 為空時）— 🔴
- [ ] 破壞性操作有 dry-run／`-WhatIf`／確認機制或備份 — 🟡
- [ ] **冪等性**：重跑不會重複建立、重複寫入、重複扣款或累加錯誤狀態 — 🟡
- [ ] 寫檔採「寫入暫存檔 → 原子性改名」，避免中斷時留下半截檔案 — 🟡
- [ ] 時區、日期格式、浮點數、整數溢位、字串比對大小寫與文化特性（`-eq` vs `-ceq`、`ToLower()` 的 culture）— 🟡

## 2. 安全性（OWASP）

- [ ] **命令注入**：外部輸入未經處理就拼進 shell 指令、`eval`、`Invoke-Expression`、`subprocess(..., shell=True)`、`exec()` — 🔴
- [ ] **路徑穿越**：使用者提供的路徑未正規化與限制在允許目錄內（`../`）— 🔴
- [ ] **機密處理**：密碼、token、API key 不可寫死在程式碼、不可印到日誌或錯誤訊息、不可出現在命令列參數（會被 `ps` 看到）；應從環境變數、密碼庫或 credential manager 取得 — 🔴
- [ ] 下載內容（腳本、壓縮檔、更新包）有驗證 checksum／簽章，且不直接 `curl | sh` — 🔴/🟡
- [ ] 暫存檔使用安全 API（`mktemp`、`New-TemporaryFile`、`tempfile`），不用可預測檔名 — 🟡
- [ ] 最小權限：不要求不必要的管理員權限；外掛 manifest 只宣告用到的權限 — 🟡
- [ ] 反序列化不受信任資料（`pickle`、`yaml.load`、`BinaryFormatter`）— 🔴
- [ ] TLS 驗證未被關閉（`verify=False`、`-SkipCertificateCheck`、`NODE_TLS_REJECT_UNAUTHORIZED=0`）— 🔴

## 3. 錯誤處理與可恢復性

- [ ] **失敗即停**：
  - Bash：`set -euo pipefail`，且了解其在 `if`、`&&`、子 shell 中的例外
  - PowerShell：`$ErrorActionPreference = 'Stop'` 或關鍵 cmdlet 加 `-ErrorAction Stop`；注意原生程式（.exe）失敗不會觸發，要檢查 `$LASTEXITCODE`
  - Python／Node：不使用裸 `except:`／空 `catch {}` 吞掉錯誤
  — 🔴/🟡
- [ ] 結束碼正確：成功回 0、失敗回非 0，讓排程與 CI 能判斷 — 🟡
- [ ] 清理機制：`trap ... EXIT`、`try/finally`、`using`／`with`，中斷時釋放鎖、刪除暫存檔、還原狀態 — 🟡
- [ ] 網路與外部呼叫：有逾時設定、有限次數且帶退避的重試、區分可重試與不可重試錯誤 — 🟡
- [ ] 錯誤訊息包含足夠脈絡（哪個檔案、哪個步驟、原始錯誤）— 🟡
- [ ] 並行執行保護：同一排程重疊時有鎖（`flock`、mutex、lock file）— 🟡

## 4. 環境相容性

- [ ] **路徑**：含空白、中文、特殊字元的路徑有正確加引號；不寫死絕對路徑或使用者名稱；跨平台時用 `Join-Path`／`os.path.join`／`path.join` — 🔴/🟡
- [ ] **編碼**：讀寫檔明確指定編碼；Windows PowerShell 5.1 的 `Set-Content`／`Out-File` 預設編碼不同於 PowerShell 7；BOM 對其他工具的影響；主控台中文亂碼（`chcp 65001`、`[Console]::OutputEncoding`）— 🟡
- [ ] **版本相依**：使用的語法或 API 在目標執行環境可用（PowerShell 5.1 不支援 `&&`、`??`、三元運算子；Python 版本以 `build.ps1` 指定的 3.12 為準；Node 版本以 `functions/package.json` 的 `engines` 為準）— 🔴
- [ ] 外部指令存在性：使用前檢查（`command -v`、`Get-Command`），並給出清楚錯誤 — 🟡
- [ ] 不依賴目前工作目錄：以腳本自身位置為基準（`$PSScriptRoot`、`"$(dirname "$0")"`、`__dirname`、`Path(__file__)`）— 🟡
- [ ] 無互動環境（排程、CI）下不會卡在提示輸入 — 🔴

## 5. 外掛／擴充套件特有事項

- [ ] **宿主相容性**：宣告支援的宿主版本範圍；使用的 API 在範圍內皆存在；已棄用 API 的替代方案 — 🟡
- [ ] **生命週期**：在 activate／deactivate（或 load／unload）正確註冊與釋放事件監聽、計時器、檔案監看、子程序；不釋放會造成記憶體洩漏或重複觸發 — 🟡
- [ ] **不阻塞宿主**：耗時操作不在 UI／主執行緒同步執行；I/O 用非同步 API — 🔴/🟡
- [ ] **錯誤隔離**：外掛例外不會讓宿主當機；hook 失敗時有合理預設行為（放行或阻擋要明確設計）— 🔴
- [ ] **Hook 與 webhook**：遵守宿主規定的輸入輸出格式與結束碼語意；有執行時間上限；驗證 webhook 簽章 — 🔴/🟡
- [ ] **設定**：有預設值與 schema 驗證；設定變更時能即時套用或明確提示需重新載入 — 🟢/🟡
- [ ] 不污染全域：不修改宿主的全域物件、原型鏈或共用設定 — 🟡

## 6. 可觀測性

- [ ] 關鍵步驟有日誌，區分等級（info／warn／error）；支援 `--verbose`／`-Verbose` — 🟡/🟢
- [ ] 日誌輸出到 stderr，資料結果輸出到 stdout，方便管線串接 — 🟢
- [ ] 長時間作業有進度提示 — 🟢
- [ ] 結束時輸出摘要（處理幾筆、成功幾筆、失敗清單）— 🟢

## 7. 設定與介面（Twelve-Factor）

- [ ] 可變值（路徑、URL、帳號、門檻）外部化為參數、環境變數或設定檔，不寫死 — 🟡
- [ ] CLI 參數有驗證（必填、型別、範圍）與 `--help`／comment-based help — 🟢/🟡
- [ ] 參數命名一致且符合語言慣例（PowerShell 用 Verb-Noun 與核准動詞）— 🟢

## 8. 可維護性與可讀性（Google eng-practices）

- [ ] **設計**：職責分明，主流程易讀；過長函式（約 50 行以上）拆分 — 🟢/🟡
- [ ] **複雜度**：沒有過度設計，也沒有難以理解的單行技巧 — 🟢
- [ ] **命名**：變數與函式名稱表達意圖 — 🟢
- [ ] **註解**：說明「為什麼」而非「做什麼」；過時註解要刪 — 🟢
- [ ] **重複**：相同邏輯出現三次以上應抽出 — 🟢
- [ ] 魔術數字與字串改為具名常數 — 🟢
- [ ] 殘留的除錯輸出、註解掉的程式碼、TODO 未追蹤 — 🟢

## 9. 測試

- [ ] 核心邏輯可被獨立測試（與 I/O 分離）— 🟢
- [ ] 有對應測試（Pester、bats、pytest、Jest），涵蓋失敗路徑與邊界值 — 🟡/🟢
- [ ] 破壞性操作有在隔離環境（暫存目錄、mock）驗證 — 🟡
