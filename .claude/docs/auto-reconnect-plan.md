# 斷線自動重連：可行性分析與實作規劃

- 狀態：階段 A~D 已實作（2026-10-17），尚未實機驗證；實作與規劃的差異見文末「實作紀錄」
- 範圍：只改 PC 端 `mlevel/`；後端 `mlevelweb/functions/`、網頁 `mlevelweb/src/` 不動
- 目前版本：`mlevel/version.py` = 1.1.1，預計以 1.2.0 釋出
- 素材：使用者提供的截圖 `disconnect.png`、`login_1~5.png`、`startgame_1~4.png`（桌面，未進版控）

---

## 1. 結論

**可行，而且不需要繞 UAC、不需要存帳號密碼。**

| 問題 | 結論 | 依據 |
|------|------|------|
| 重開遊戲會跳 UAC？ | 不會。`mlevel.exe` 打包時就要求管理員（`mlevel.spec` 的 `uac_admin=True`），由它 `CreateProcess` 開遊戲，子行程沿用權限。現有的 `mlevel-updater.exe` 就是靠同一個原理不跳 UAC（`mlevel.spec` 中 updater 的註解） | `mlevel/mlevel.spec`、`mlevel/updater.py:342-351` |
| 要不要走官網「啟動遊戲」？ | 不要。那條路由瀏覽器觸發，才會跳 UAC。直接執行遊戲執行檔即可（待實測確認會自動開登入頁） | `login_1.png` |
| 要不要存帳密？ | 不用。gamapass 網頁只要依序點「Sign in with gamapass → 記住的帳號 → 遊戲帳號 → 繼續」 | `login_2~5.png` |
| 參考文件的四個方案 | 調低 UAC＋自動按 Alt+Y：不採用（降低使用者安全、會同意所有提權，且多半被 UIPI 擋）；管理員瀏覽器：不採用；硬體模擬：不適用；工作排程器：只當 mlevel 未提權時的備案，以目前打包方式用不到 | — |

**做不到／刻意不做的：**

1. gamapass 記住的登入過期，網頁改要密碼或 OTP → 停下通知，請使用者手動登入。
2. 重連後角色不在原本地圖 → 停下通知。路線只能在單一地圖內走（`mlevel.py:2801` 的步驟種類沒有傳送門／換圖）。
3. Windows 顯示縮放不是 100% → 第一版不支援自動點擊（程式非 DPI aware，只偵測縮放並警告，`mlevel.py:1812`、`mlevel.py:2261`）。
4. 頻道清單第一版只支援畫面上看得到的 ch.1~20（不做捲動）。

---

## 2. 現況分析（程式碼）

### 2.1 目前遇到斷線會發生什麼

| 情況 | 現在的行為 | 問題 |
|------|------------|------|
| 斷線木牌跳出、尚未按確認 | 主迴圈繼續送攻擊鍵；人物錨點被木牌擋住 5 秒後 `_check_covered` 響一次警報但不停（`mlevel.py:4349`） | 攻擊鍵若是 Enter／空白可能直接把「確認」按掉 |
| 遊戲已關閉 | 視窗只在 `Hunter.__init__` 找一次（`mlevel.py:1962`），`_is_foreground()` 永遠 False，迴圈卡在「Target window lost focus; pausing…」（`mlevel.py:4611-4617`） | **無限期暫停，沒有任何通知** |

程式碼裡沒有任何斷線、登入、滑鼠點擊的處理（grep 斷線／login／reconnect 只命中授權與更新器）。

### 2.2 可以直接沿用的

| 需要 | 既有實作 |
|------|----------|
| 找遊戲視窗、拉到前景 | `find_window`（`mlevel.py:1669`）、`bring_to_front`（`mlevel.py:1854`）；標題 `新楓之谷：經典版`（`mlevel.py:161`），選伺服器／選角畫面標題相同（`startgame_1.png`） |
| 截圖 | `grab(window)` 截外框含標題列（`mlevel.py:1720`）；`grab_region` 截螢幕任意區（`mlevel.py:1747`） |
| 範本比對 | `load_template`（`mlevel.py:852`，支援中文路徑與 alpha 遮罩）、`find_best`（`mlevel.py:938`） |
| 暫停打怪 → 處理事件 → 恢復 | `_check_minigame` / `_handle_minigame`（`mlevel.py:4272`、`mlevel.py:4398-4460`），恢復時的計時器歸零在 `mlevel.py:4452-4460` |
| 確認是不是原地圖 | `minimap_fingerprint`、`identify_map`（`mlevel.py:1457`、`mlevel.py:1533`） |
| 截圖工具 | 截繩子 `_snip_rope` / `_open_rope_snipper` / `_save_rope_crop`（`mlevel.py:6239`、`6269`、`6345`） |
| 圖片清單 UI | `_template_list`（`mlevel.py:5276`）、測謊區塊（`mlevel.py:5038-5071`） |
| 警報 | `minigame.alarm`（測謊用） |
| 授權失效立即停 | `App._check_license` 每 100ms 檢查，失效就 `stop_event.set()`（`mlevel.py:5389`） |

### 2.3 要新增的

| 需要 | 做法 |
|------|------|
| 滑鼠點擊 | 目前只有 `minigame.move_cursor`（`minigame.py:885`，SendInput 絕對座標移動），照同樣結構加 LEFTDOWN/LEFTUP |
| 判斷視窗還在不在 | ctypes `user32.IsWindow(hwnd)` |
| 取得遊戲執行檔路徑 | `GetWindowThreadProcessId` → `OpenProcess(PROCESS_QUERY_LIMITED_INFORMATION)` → `QueryFullProcessImageNameW` |
| 等遊戲行程結束 | `OpenProcess(SYNCHRONIZE)` + `WaitForSingleObject`，分段等以便檢查 `stop_event` |
| 依序點擊流程 | 新模組 `reconnect.py` |

全部用 ctypes，不新增第三方套件，`mlevel.spec` 不用加 hiddenimports（`reconnect.py` 由 import 自動帶入）。

---

## 3. 重連流程

| # | 畫面（截圖） | 動作 | 比對範圍 | 範本 | 完成判斷 | 預設等待 |
|---|------|------|------|------|------|------|
| D1 | 斷線木牌（`disconnect.png`） | 點「確認」 | 遊戲視窗 | 內附 | 遊戲行程結束 | 30 s |
| D2 | — | 等重試間隔、檢查維護時段與每小時上限 | — | — | — | 60 s |
| L0 | — | 用記下的路徑啟動遊戲 | — | — | 瀏覽器出現 W1 | 90 s |
| W1 | galaxy 登入頁（`login_2.png`） | Sign in with gamapass | 全螢幕 | 內附 | W2 或 W3 出現 | 20 s |
| W2 | 使用 gamapass 登入（`login_3.png`） | 點記住的帳號卡片 | 全螢幕 | **使用者自截** | W3 出現 | 20 s，可略過 |
| W3 | 選擇您的帳號（`login_4.png`） | 點遊戲帳號 | 全螢幕 | **使用者自截** | W4 變亮橘 | 20 s |
| W4 | 同頁（`login_5.png`） | 點「繼續」（需亮橘） | 全螢幕 | 內附＋顏色檢查 | 遊戲出現 G1 | 20 s |
| W5 | — | 對瀏覽器送 Ctrl+W 關登入分頁 | — | — | — | — |
| G1 | 選擇伺服器（`startgame_1.png`） | 點伺服器木牌（例：1 雪吉拉） | 遊戲視窗 | **使用者自截** | 頻道清單出現 | 60 s |
| G2 | 頻道清單（`startgame_2.png`） | 點頻道（例：ch.18） | 遊戲視窗 | **使用者自截**，只截「ch.18」文字，不含人數條 | 選擇角色畫面出現 | 30 s |
| G3 | 選擇角色（`startgame_3.png`） | 點角色名牌（例：手動a海盜） | 遊戲視窗 | **使用者自截**，黑底（選中）與灰底（未選）各一張 | 名牌為黑底 | 30 s，已選中可略過 |
| G4 | 同頁（`startgame_4.png`） | 開始遊戲 | 遊戲視窗 | 內附 | 小地圖出現 | 60 s |
| M | 遊戲內 | `identify_map` 比對小地圖 | 遊戲視窗 | 既有 maps 指紋 | 與開始打怪時同一張圖 | 20 s |

成功 → 恢復打怪。任何一步逾時 → 存畫面、警報、停止打怪，日誌寫明卡在哪一步。

**觸發條件（任一）：**

1. 畫面上偵測到斷線木牌（每秒檢查一次）。
2. 遊戲視窗已不存在（`IsWindow` 為 False）→ 跳過 D1，直接 D2。

---

## 4. 設計

### 4.1 新模組 `mlevel/reconnect.py`

和 `minigame.py` 一樣是獨立模組，不 import `mlevel.py`；需要的截圖、比對函式由呼叫端傳入，方便用 `tools/` 寫離線重播測試。

```python
@dataclass
class ClickStep:
    name: str                 # 日誌與 UI 顯示用，例如「W1 gamapass」
    templates: list[str]      # 任一張命中即可
    confidence: float = 0.85
    timeout: float = 20.0
    scope: str = "game"       # "game" = 遊戲視窗；"screen" = 主螢幕
    optional: bool = False    # 逾時視為已略過，繼續下一步
    need_color: tuple | None = None  # 命中後中心點 RGB 要接近這個色才點（「繼續」亮橘）
    settle: float = 0.8       # 點完等多久再看下一步

def click(x: int, y: int) -> None              # SendInput 移動＋左鍵按下／放開
def window_alive(hwnd: int) -> bool
def process_path(hwnd: int) -> str | None
def wait_process_exit(hwnd, timeout, stop_event) -> bool
def launch_game(path: str) -> None             # subprocess.Popen，cwd 設成遊戲資料夾
def in_maintenance(now: datetime) -> bool      # 週四 08:00~14:00

class Reconnector:
    def __init__(self, cfg, log, stop_event, grab, grab_screen, find_best, ...)
    def run(self, hwnd: int | None) -> ReconnectReport   # 跑 D1 → M
```

每一步的輪詢：截圖 → 所有範本 `find_best` 取最高分 → 過門檻（及顏色檢查）→ 換算螢幕座標 → `click` → `stop_event.wait(settle)`。

- `scope="game"`：螢幕座標 = `window.left/top` + 命中點（與 `minigame.Session._to_screen` 同一套，`minigame.py:1253`）。
- `scope="screen"`：抓主螢幕，命中點即螢幕座標。
- 所有等待都是 `stop_event.wait()`，按停止或授權失效時立即中止。
- 每步失敗時把當下畫面存到 `log/reconnect/<時間>/<步驟>.jpg`（張數設上限）。

### 4.2 掛進 `Hunter`

| 位置 | 改動 |
|------|------|
| `Hunter.__init__`（`mlevel.py:1962` 附近） | 記下 `self.window._hWnd`、遊戲執行檔路徑、開始時的地圖指紋；載入斷線木牌範本與 `reconnect_steps` |
| `Hunter.run()`，`_is_foreground()` 檢查之前（`mlevel.py:4611`） | `if not window_alive(hwnd): self._handle_disconnect(closed=True); continue` |
| `Hunter.run()`，`_check_minigame` 旁（`mlevel.py:4272` 呼叫處） | `_check_disconnect(gray)`：每秒最多一次，命中就 `_handle_disconnect(closed=False)` |
| 新增 `_handle_disconnect` | `_release_lock` → `release_all` → 警報 → 未啟用自動重連就 `stop_event.set()`；啟用則跑 `Reconnector.run()`，成功後重新 `find_window`、`bring_to_front`、`_reset_after_interrupt()`，失敗就 `stop_event.set()` |
| 抽出 `_reset_after_interrupt()` | 把 `_handle_minigame` 末段（`mlevel.py:4452-4460`）抽成共用，另外加上 `buff_due` 全部歸零（`mlevel.py:2020`）讓增益重新施放、`_anchor_blind()` |

**`run()` 的時限**：迴圈有 `deadline`（`mlevel.py:4606` 一帶），重連時間照算進去，不另外延長。

### 4.3 限流與保護

| 規則 | 預設 | 理由 |
|------|------|------|
| 每小時最多重連次數 | 3 | 伺服器不穩時避免反覆開遊戲 |
| 兩次重連最小間隔 | 60 s | 同上 |
| 維護時段（週四 08:00~14:00）不重連 | 開 | 維護中一定失敗（官網公告，`login_1.png`） |
| 等遊戲關閉逾時 | 只通知，不強殺 | 不替使用者結束遊戲行程 |
| 授權離線寬限 | 不變（30 分鐘） | 斷線若同時斷網，重連須在寬限內完成，否則 `SessionKeeper` 會停（`license_gate.py:59`） |
| 左上角中止手勢 | 重連中照樣有效 | 點擊座標不會落在 (0,0) |

---

## 5. 設定與介面

### 5.1 新設定鍵

全部加入 `DEFAULT_CONFIG`（`mlevel.py:160`）與 `CONFIG_NOTES`（`mlevel.py:271`）。`load_config` 會用預設值補齊舊設定檔沒有的鍵（`mlevel.py:4780`），`save_config` 只寫 `DEFAULT_CONFIG` 裡的鍵、保留其他鍵（`mlevel.py:4837`）。

| 鍵 | 型別 | 預設 | 說明 |
|----|------|------|------|
| `disconnect_watch_enabled` | bool | True | 偵測斷線；只停下＋警報，不重連 |
| `disconnect_alarm` | bool | True | 斷線時響警報 |
| `disconnect_templates` | list[str] | `["templates/reconnect/disconnect.png"]` | 斷線木牌範本 |
| `disconnect_confidence` | float | 0.85 | |
| `reconnect_enabled` | bool | False | 偵測到後自動重連 |
| `reconnect_game_path` | str | "" | 遊戲執行檔，開始打怪時自動填入 |
| `reconnect_max_per_hour` | int | 3 | |
| `reconnect_retry_delay_seconds` | float | 60 | |
| `reconnect_skip_maintenance` | bool | True | |
| `reconnect_steps` | list[dict] | 內建 W1~G4 樣板（使用者自截的步驟圖片為空） | 每步對應 `ClickStep` 的欄位 |

`load_config` 需比照 `minigame_templates`（`mlevel.py:4785-4793`）處理：

- `disconnect_templates` 不是 list 就改成 `[]`，路徑過 `stored_asset_path`。
- `reconnect_steps` 不是 list、或某一步缺欄位／型別錯，就丟掉那一步並寫日誌；圖片路徑過 `stored_asset_path`。
- 必要的自截步驟（W3、G1、G2、G3）沒有圖時，`reconnect_enabled` 在開始打怪時檢查並提示，不允許啟動自動重連（偵測斷線仍可用）。

> `disconnect_watch_enabled` 預設 True 是行為改變：遊戲關掉時從「無限期暫停」變成「停止＋警報」。這修正的是一個沒有提示的卡死狀態，不影響資料相容；若要求零行為改變，可改預設 False。

### 5.2 介面

新增分頁「斷線重連」（目前分頁在 `mlevel.py:4939-4942`），內容：

1. 勾選：偵測斷線、斷線警報、自動重連、維護時段不重連。
2. 遊戲執行檔：唯讀欄位（自動填入）＋「選擇…」按鈕。
3. 每小時上限、重試間隔。
4. 步驟清單（每步一列）：名稱、範本縮圖數、相似度、等待秒數、可略過、［截圖］［測試］。
   - **截圖**：仿截繩子工具，網頁步驟截全螢幕、遊戲步驟截遊戲視窗，框選後存到 `templates/reconnect/`。
   - **測試**：只找不點，回報最高相似度與位置，讓使用者在實際畫面確認。
5. 「整段試跑」按鈕：在使用者手動停在 G1 畫面時，從 G1 跑到 G4，用來驗證遊戲內步驟（不用真的斷線）。

---

## 6. 打包與更新

- 內附範本放 `mlevel/templates/reconnect/`：`disconnect.png`、`gamapass.png`、`continue.png`、`start_game.png`。
  - 打包時 `templates/` 會複製到 exe 旁（`mlevel.spec` 的 `data_dirs`、`build.ps1:109`）。
  - 自動更新時換檔程式只**補使用者缺的檔案**（`updater.py:9-10`），所以新範本會送到既有使用者手上，但之後若要改同名範本不會覆蓋 → **內附範本改版時要換檔名**（例如 `disconnect_v2.png`），並更新預設值。
  - 程式要容許範本檔不存在（舊版換檔程式或使用者刪掉）：載入失敗就關閉該功能並寫日誌，不能讓打怪啟動失敗。
- 不新增套件，`mlevel.spec` 不需修改。
- 版本號改 `mlevel/version.py`；後端 `/update/*` 的版本資訊照既有流程發布。

---

## 7. 相容性檢查

| 項目 | 影響 |
|------|------|
| 後端 API、金鑰、席位、綠界、Firestore | 無 |
| 網頁 | 無 |
| 舊設定檔 | 缺鍵補預設；自動重連預設關閉 |
| 舊版 PC 腳本 | 不受影響（純新功能，不改任何共用格式） |
| `SessionKeeper` | 重連期間照常心跳；同機不影響席位 |
| 部署順序 | 只有 PC 端，無後端前置 |

---

## 8. 實作階段

### 階段 A：偵測斷線、安全停下（可單獨釋出）

- `reconnect.py`：`window_alive`、`process_path`。
- `Hunter`：記 hwnd 與執行檔路徑；視窗消失偵測；斷線木牌偵測；`_handle_disconnect`（只停＋警報＋存畫面）。
- 設定：`disconnect_*` 四個鍵。
- 範本：`templates/reconnect/disconnect.png`（從 `disconnect.png` 裁出木牌標題或「確認」附近的穩定區塊，不含背景遊戲畫面）。
- 驗收：關遊戲 → 3 秒內停止並警報；人為斷線 → 偵測到木牌並停止；正常打怪 1 小時無誤判。

### 階段 B：點擊引擎與截圖工具

- `reconnect.py`：`click`、`ClickStep`、步驟輪詢、顏色檢查、失敗存圖。
- UI：「斷線重連」分頁、步驟清單、截圖、測試。
- 驗收：每個步驟的「測試」在實際畫面相似度 ≥ 門檻；在其他畫面不誤判。

### 階段 C：重開遊戲與網頁登入

- `wait_process_exit`、`launch_game`、W1~W5、限流、維護時段。
- 驗收：從斷線木牌開始，能自動到 G1 畫面；gamapass 登出狀態下停在 W2／W3 並通知。

### 階段 D：遊戲內選角與恢復打怪

- G1~G4、地圖確認、`_reset_after_interrupt`、增益重施。
- 驗收：完整斷線 → 回到原地圖繼續打怪；地圖不同時停下；連續斷線超過每小時上限時停下。

A 可先發 1.1.2；B~D 合併為 1.2.0。

---

## 9. 測試計畫

| 情境 | 做法 | 預期 |
|------|------|------|
| 真實斷線 | 打怪中關閉網路 | 木牌 → 確認 → 重連 → 恢復打怪 |
| 遊戲被關 | 工作管理員結束遊戲 | 跳過 D1，直接重開 |
| 攻擊鍵按掉確認 | 攻擊鍵設 Enter 再斷線 | 由「視窗消失」觸發 |
| gamapass 登出 | 先在瀏覽器登出 | 停在 W2/W3，警報 |
| 頻道滿人 | — | 待確認畫面（見 §10） |
| 維護時段 | 把系統時間設到週四 09:00 | 不重連，停止並通知 |
| 按停止 | 重連每一步中按停止 | 立即停下，不再點擊 |
| 授權被接手 | 另一台按「在這台裝置使用」 | 重連中止，主視窗顯示授權失效 |
| 舊設定檔 | 用 1.1.1 的 `auto_hunt_config.json` 啟動 | 行為與舊版相同 |
| 顯示縮放 125% | — | 提示不支援自動重連，偵測斷線仍可用 |

另寫 `tools/reconnect_replay.py`：對截圖資料夾逐張跑每個步驟的比對，輸出相似度表，用來調門檻與防誤判。

---

## 10. 待確認事項

| # | 問題 | 影響 |
|---|------|------|
| 1 | 以系統管理員直接執行遊戲執行檔，瀏覽器是否自動開 galaxy 登入頁？遊戲執行檔路徑？ | 階段 C 能否成立（不行的話改用官網按鈕＋工作排程器備案） |
| 2 | G2 點頻道後是直接進選角，還是要再點「前往選擇的伺服器」或雙擊？ | G2 是一步還是兩步 |
| 3 | G3 進來時是否已選中上次角色？有無第二組密碼？ | G3 是否可略過、是否多一步 |
| 4 | 按開始遊戲後是否有公告／活動彈窗要關？ | 是否要加 G5 |
| 5 | 頻道滿人或伺服器忙碌的畫面 | 換頻道重試或直接停 |
| 6 | gamapass 記住登入能維持多久？ | 是否需要提醒使用者定期手動登入 |
| 7 | 斷線偵測預設開啟可接受嗎？ | §5.1 的預設值 |

---

## 11. 實作紀錄（2026-10-17）

| 檔案 | 內容 |
|------|------|
| `mlevel/reconnect.py`（新） | 滑鼠點擊、`window_alive`／`process_path`／`wait_window_gone`／`launch`、維護時段、步驟定義 `ALL_STEPS`、`Sequencer` |
| `mlevel/mlevel.py` | 設定鍵與說明、`run_login_flow`（打怪中重連與試跑共用）、`Hunter._check_disconnect`／`_handle_disconnect`／`_reconnect`、`_reset_after_interrupt`（從 `_handle_minigame` 抽出）、「斷線重連」分頁 |
| `mlevel/templates/reconnect/` | 內附圖：`disconnect.png`、`disconnect_ok.png`、`web_gamapass.png`、`web_continue.png`、`game_start.png`（從使用者截圖裁出） |

與規劃不同之處：

- 設定鍵改為 `reconnect_images = {步驟: [圖檔]}` 一個 dict，取代 `disconnect_templates`／`reconnect_steps`；步驟的順序、逾時、可略過與否固定寫在 `reconnect.SEQUENCE`，使用者只管圖。
- `disconnect_watch_enabled` 預設 **False**（零行為改變）；勾「自動重連」時會一併開啟偵測。
- 斷線偵測只比對視窗中間 3/5 的範圍、每秒一次。
- 頻道步驟門檻至少 0.93（`min_confidence`），避免 ch.13／ch.18 互相誤認。
- 「繼續」按鈕的灰階在淡橘／亮橘時都是 1.000，必須靠顏色（`is_vivid`）分辨，已用 `login_4.png`／`login_5.png` 驗證。
- 流程可從任何畫面接著跑：還沒看到目前這一步、卻看到後面某一步，就直接跳過去。

離線驗證（用使用者截圖）：各內附圖在正確畫面 1.000、在其他畫面 0.31~0.50；`Sequencer` 依序點擊、跳步、逾時三條路徑正常；分頁可正常建立、`_collect` 驗證、還原預設不會改到 `DEFAULT_CONFIG`。

尚待實機驗證：§10 的 1~6 項，以及遊戲是否接受 SendInput 的滑鼠點擊。

### 2026-10-17 修正：遊戲改由官網登入流程開啟

實測直接執行 `Maplestory_Classic.exe` 不會出現任何畫面。查官網原始碼（`float-btns.js` 的 `gamePlay`、首頁的 `GalaxyLogin`、`platform.nexon.com/NGM/JS/npf_ngm.js`）：

1. 「啟動遊戲」= 開 `https://galaxy.games.gamania.com/webapi/view/login/mstc?redirect_url=https://maplestoryclassic.beanfun.com/Main`，302 到帶一次性 OTT 的 gamapass 登入頁；
2. 登入完導回官網，頁面用 OTT 換 `userSessionToken`，呼叫 `NGM.LaunchGameOnlyPassArgs` 開 `ngm://launch/...`；
3. `ngm` 協定由 `C:\ProgramData\Nexon\NGM\NGM64.exe "%1"` 處理（`HKCR\ngm`），由它開遊戲。Chrome 已記住此網站可開 `ngm`，不會再跳確認。

因此流程改為：用 `explorer.exe <網址>` 以一般權限開瀏覽器（不用 `webbrowser.open`，避免從管理員行程開出管理員瀏覽器）→ W1~W4 → 等遊戲自己出現（G1 等待放寬到 180 秒）→ G1~G4 → 進到地圖後才關掉官網分頁（按完「繼續」不能關，遊戲是那一頁開的）。新增設定 `reconnect_login_url`；`reconnect_game_path` 只用來辨認遊戲視窗。

待確認：NGM64.exe 由瀏覽器開遊戲時會不會跳 UAC。會的話全自動無法完成（UAC 在安全桌面，程式點不到），需另議。

### 2026-10-17 修正：從官網首頁開始、NGM 轉接避開 UAC

- 直接開 galaxy 登入網址在瀏覽器記得登入時會直接導回官網、不開遊戲，所以改成開官網首頁（`reconnect_site_url`）並新增第一步「網頁：官網『啟動遊戲』」（內附 `templates/reconnect/web_launch.png`）；網頁登入四步改成可略過。
- 實測 NGM 開遊戲會跳 UAC。`NGM64.exe`、`Maplestory_Classic.exe` 的 manifest 都是 `asInvoker`，是 NGM 執行時自己要權限。
- 新增可勾選的「轉接啟動」（`reconnect_ngm_relay`，預設關）：只在 HKCU 建 `Software\Classes\ngm`，指到 `%LOCALAPPDATA%\mlevel\ngm_relay.ps1`（`powershell -File`，網址只當參數不會被執行）。重連流程開始時寫 `ngm_armed.txt`（10 分鐘期限），腳本在期限內把網址寫進 `ngm_request.txt`，由已是管理員的 mlevel 執行 `NGM64.exe <網址>`，就不跳 UAC；期限外照 HKLM 原本的程式開（行為與未安裝相同）。取消勾選刪掉 HKCU 那一筆。勾選框顯示的是實際安裝狀態。
- 新的本機檔案（`%LOCALAPPDATA%\mlevel\ngm_relay.ps1`、`ngm_armed.txt`、`ngm_request.txt`）要補進契約文件的本機資料表。

### 2026-10-17 實機試跑通過與後續調整

- 試跑從官網一路進到遊戲成功（轉接啟動不跳 UAC）。
- 遊戲按鈕點了沒反應的原因：放開滑鼠後立刻把游標移開，遊戲在同一幀讀到的是「在別處放開」。`click` 放開後停 0.3 秒再移開；頻道、開始遊戲改連點兩下；點完把游標停到遊戲視窗左下角，免得遊戲游標蓋住「ch.18」。頻道門檻 0.97、只重點同一位置。
- 新增勾選「重連後自動開始打怪」（`reconnect_resume_hunt`，預設開）：打怪中重連成功就接著打（關掉則登入後停止）；分頁「試跑」成功後也直接按開始打怪。
