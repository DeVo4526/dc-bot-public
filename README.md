<!-- 此檔案由 dc-bot 自動同步產生，請不要在這裡直接編輯——內容請到 dc-bot 那邊改。 -->

## 設定檔

可調整的參數（機器人暱稱、時區、AI 聊天觸發字／記憶輪數／冷卻時間等）都集中放在 [config.py](config.py)，改行為不用去翻 `main.py`。**金鑰／token 不放這裡**——`DISCORD_TOKEN`、`GEMINI_API_KEY` 一律寫在 `DISCORD_TOKEN.env`，由 `main.py` 讀取環境變數。

## 開發小工具

跟 bot 本身無關、純粹輔助開發用的獨立小程式：

| 工具 | 說明 |
|------|------|
| [tools/export_chat_memory.py](tools/export_chat_memory.py) | 把 `chat_memory.sqlite`（AI 聊天記憶）匯出成好讀的 JSON |
| [tools/pyc_viewer.py](tools/pyc_viewer.py) | 通用 `.pyc` 檢視器，有視窗介面，可自選任何 `.pyc` 檔案查看版本資訊與反組譯內容 |

放在 `tools/` 資料夾，跟主程式（`main.py`、`config.py`、`werewolf.py`）分開，不會混在一起。兩者都只依賴 Python 標準庫，不用啟動 bot、不用設定 token；在專案根目錄執行即可（不用先 `cd` 進 `tools/`）：

```
python tools/export_chat_memory.py
python tools/pyc_viewer.py
```

`pyc_viewer.py` 另外打包成 Windows 執行檔 [tools/dist/pyc_viewer.exe](tools/dist/pyc_viewer.exe)，不用裝 Python 也能雙擊直接開啟使用。用 [PyInstaller](https://pyinstaller.org/) 打包（`--onefile --windowed`），單一檔案、無多餘主控台視窗。以後改了 `pyc_viewer.py` 想重新打包：

```
pyinstaller tools/pyc_viewer.spec --distpath tools/dist --workpath tools/build
```

（`tools/pyc_viewer.spec` 是打包設定檔，已存在專案裡；重新打包後記得刪掉 `tools/build/` 這個中間產物資料夾，不用留著）

## 功能一覽

### AI 聊天（Gemini，免費額度）

| 指令 / 方式 | 說明 |
|------|------|
| 訊息裡打「**悠筑**」/「**小筑**」，或直接 **@** 這個 bot | 不用指令，出現任一個觸發字或被 @ 就會回覆，像平常聊天一樣 |
| `/chat_reset` | 清除你自己在這個頻道的聊天記憶，重新開始對話 |
| `/my_memory` | 查看悠筑對你的印象記憶（好感度、心情、印象筆記、提過的人）。一般使用者只能看自己；只有這個 bot application 在 Discord Developer Portal 上的擁有者/團隊成員（透過 `bot.is_owner()` 判斷）可以加 `user` 參數指定查看任何人 |
| `/todo add` `/todo list` `/todo done` | 不透過聊天，直接管理待辦事項。`done` 有 autocomplete，直接從清單選就好，不用自己打文字比對 |

**跟排程提醒功能串接（Function Calling）**：悠筑可以在聊天中直接幫你操作既有功能，不只是回話。目前串了：

- `set_timer` — 聊天裡提到「N分鐘後提醒我...」這種**相對時間**（例如「悠筑10分鐘後提醒我喝水」），她會呼叫這個工具**真的**建立一筆排程，跟 `/schedule add` 共用同一套 APScheduler 機制，所以也會出現在 `/schedule list` 裡，一樣只有本人能看到/刪除
- `set_alarm` — 聊天裡提到「幾點幾分提醒我...」這種**指定時間點**（例如「悠筑4點54分提醒我開會」），一樣會真的建立排程
- `list_timers` — 問她「還有哪些提醒」之類的問題時，會實際查詢排程資料回答，不會憑空亂講
- `get_current_time` — 問她「現在幾點」，或她需要判斷 `set_alarm` 的上午/下午、計算時間差時，會呼叫這個工具拿到真實時間，不會自己瞎猜（Gemini 本身沒有即時時鐘）
- `update_user_impression` — 聊天中如果她判斷有『值得長期記住』的資訊（稱呼、興趣、個性等），會把摘要存進 `user_notes` 表，之後每次聊天都會帶入，不受一般對話記憶輪數限制。這是 AI 自行判斷觸發，沒有固定的觸發門檻
- `add_todo` / `list_todos` / `complete_todo` — 跟計時/鬧鐘不同，這組是給**沒有明確時間點**的待辦事項用的（例如「幫我記一下要買牛奶」），只是先記下來、不會主動觸發提醒。`complete_todo` 用文字模糊比對找出最接近的一筆標記完成。跨頻道共用，每人最多 `CHAT_TODO_MAX_PER_USER`（預設 30）筆未完成
- `get_weather` — 查詢某個地點目前的天氣（氣溫、體感溫度、濕度、風速、降水量）。用 [Open-Meteo](https://open-meteo.com) 的免費 API，**不需要申請金鑰、不吃 Gemini 額度**（是額外呼叫的一支獨立 REST API）。先用地理編碼 API 把地名轉成經緯度，再查目前天氣；查不到地點或請求失敗都會回傳說明文字，不會讓 AI 自己編天氣狀況。連續失敗達到 `CHAT_WEATHER_FAILURE_ALERT_THRESHOLD`（預設 5）次會在 log 記一次 `error` 等級的警告，方便察覺 Open-Meteo 本身是不是出問題
  - ⚠️ Open-Meteo 的地理編碼對**繁體中文地名比對很不準確**（實測「台灣」「台北」「高雄」直接查是零筆結果，但「Taiwan」「Taipei」「Kaohsiung」查得到，`language=zh` 只影響輸出顯示語言、不影響輸入比對）。所以工具說明裡要求 AI 呼叫時一律把 `location` 轉成英文/羅馬拼音再送出，不要直接塞中文地名
- `remember_special_date` — 使用者提到自己的生日/紀念日等具體日期時記下來，每人最多 `CHAT_SPECIAL_DATE_MAX_PER_USER`（預設 5）個。每天 `CHAT_SPECIAL_DATE_CHECK_HOUR`（預設 9 點，用 `discord.ext.tasks.loop` 實作，不經過 APScheduler/`jobs.sqlite`）會檢查一次，符合今天日期的話會用 Gemini 生一句符合她個性的祝福，主動發在記錄當下的那個頻道
- 分鐘數上限 `CHAT_TIMER_MAX_MINUTES`（預設 1440，即 1 天），可在 [config.py](config.py) 調整
- 出於安全考量，執行工具時一律用**觸發訊息本人**的 channel/user 身分，不採信 AI 自己給的參數，所以她沒辦法幫你操作到別人的排程
- 目前刻意沒有串狼人殺功能
- 工具執行邏輯集中在 `AIChat._run_chat_tool()`，因為 `get_weather` 需要對外發 HTTP 請求，這個函式是 `async` 的

**圖片(多模態)**：訊息裡如果有附圖片（png/jpeg/webp/heic/heif），觸發聊天時會一併送給 Gemini，她會直接依照人設自然回應圖片內容。一次只處理最先出現的 `CHAT_MAX_IMAGES_PER_MESSAGE`（預設 1）張、單張超過 `CHAT_MAX_IMAGE_BYTES`（預設 4MB）會跳過不處理。圖片依 Gemini 的切格計費，一張常見尺寸的截圖大約 500~1500 token，比純文字重不少，這也是預設只處理 1 張的原因。圖片本身不會存進對話記憶，只在當次請求使用。

- 觸發關鍵字、人設 prompt、記憶輪數、上限、冷卻時間都在 [config.py](config.py)（`CHAT_TRIGGER_NAMES`、`CHAT_SYSTEM_PROMPT`、`CHAT_MAX_TURNS`、`CHAT_MAX_CONTEXTS`、`CHAT_TRIGGER_COOLDOWN`、`GEMINI_MODEL`）；@ 提及觸發是另外判斷 `bot.user in message.mentions`，不受這個清單影響
- 對話記憶以「**頻道 + 使用者**」為單位分開保存，同一頻道裡不同人講話彼此不會互相干擾、也不會看到別人的對話內容
- **頻道情境感知**：每次回覆時，除了自己跟這個人的對話記錄，也會額外帶入頻道最近 `CHAT_CHANNEL_CONTEXT_LINES`（預設 8）則其他人的訊息當參考（不會主動回應，除非跟當下的話題有關），讓對話更像有在場、而不是每次都從零開始。每則訊息截斷到 `CHAT_CHANNEL_CONTEXT_MAX_CHARS`（預設 500）字避免 token 暴增。**這代表頻道裡其他人的訊息內容也會被送到 Gemini API**，這點請留意
- **時間感知**：每一輪回覆都會附上目前的實際日期時間跟時段標籤（清晨/早上/中午/下午/傍晚/晚上/深夜，見 `_time_of_day_label()`），不用呼叫工具就知道現在幾點，避免早上說「晚安」這種時段講錯話的問題
- 需要設定環境變數 `GEMINI_API_KEY`（至 [Google AI Studio](https://aistudio.google.com/apikey) 免費申請）
- 未設定 `GEMINI_API_KEY` 時，關鍵字觸發會直接略過（不影響其他功能）
- **對話記憶會持久化**：寫進 `CHAT_DB_PATH`（預設 `chat_memory.sqlite`），bot 重啟或平台重新部署也不會忘記之前聊過什麼；每個人每個頻道的紀錄只保留最近 `CHAT_MAX_TURNS`（預設 10）輪，磁碟不會無限增長
- 記憶體只當**快取**用：同時最多快取 `CHAT_MAX_CONTEXTS`（預設 50）組「頻道+使用者」，超過會把最舊的踢出快取（不是刪除，資料還在資料庫，下次那個人再講話時會自動從資料庫讀回來）
- 每人每次觸發後有 `CHAT_TRIGGER_COOLDOWN`（預設 5）秒冷卻，避免短時間內用光免費額度
- 想直接打開看悠筑記得什麼，可以用 [tools/export_chat_memory.py](tools/export_chat_memory.py) 把 SQLite 匯出成好讀的 JSON：
  ```
  python tools/export_chat_memory.py                # 匯出成 chat_memory_export.json
  python tools/export_chat_memory.py my_export.json  # 指定輸出檔名
  ```
- 直接用 `aiohttp` 呼叫 Gemini REST API（未使用 `google-genai` SDK），避免額外吃記憶體，對記憶體有限的免費主機（如小型雲端掛機平台）較友善
- 金鑰請寫在根目錄的 `DISCORD_TOKEN.env`（`python-dotenv` 會自動讀取這個檔名），例如：
  ```
  DISCORD_TOKEN=你的discord token
  GEMINI_API_KEY=你的gemini key
  ```

### 排程提醒

| 指令 | 說明 |
|------|------|
| `/schedule add` | 新增定時提醒，參數：`time`（24 小時制，如 `20:30` 或 `2030`）、`repeat`（每日重複／一次性）、`text`（提醒內容，可留空） |
| `/schedule remove` | 刪除自己的排程，`job` 參數會自動帶出你目前的排程清單供選擇（免記 ID） |
| `/schedule list` | 查詢自己目前所有排程（僅自己可見） |

- 支援**每日重複**和**一次性**兩種類型
- 支援自訂提醒文字，提醒時會顯示觸發當下的 Discord 暱稱
- 排程資料存入 SQLite，重啟不會消失
- 每個人只能查詢、刪除自己建立的排程

---

### 狼人殺

#### 開局流程

```
/werewolf              → 建立大廳（指令者為房主）
.join                  → 其他玩家加入（至少 4 人才能開始）
.ww_roles 狼2 預1 女1   → 房主自訂角色組合（可略過，使用預設）
.start                 → 房主開始遊戲
.ww_cancel             → 房主取消大廳
```

#### 預設角色分配

| 玩家人數 | 狼人 | 預言家 | 女巫 | 獵人 | 村民 |
|----------|------|--------|------|------|------|
| 4–5 人   | 1    | 1      | —    | —    | 其餘 |
| 6–7 人   | 2    | 1      | 1    | —    | 其餘 |
| 8+ 人    | 2    | 1      | 1    | 1    | 其餘 |

#### 角色自訂格式

```
.ww_roles 狼2 預1 女1 獵0
```

關鍵字：`狼`（狼人）、`預`（預言家）、`女`（女巫）、`獵`（獵人）

#### 遊戲流程

**夜晚（DM 私訊 bot）**

| 角色 | 行動 |
|------|------|
| 狼人 | 輸入目標編號投票殺人 |
| 預言家 | 輸入目標編號查驗陣營 |
| 女巫 | `y`/`n` 決定是否救人，再輸入編號（或 `0` 跳過）毒人 |
| 獵人 | 死亡時 DM 輸入目標編號射殺 |

**白天（頻道公開）**

1. bot 公告昨夜死亡玩家
2. 90 秒自由討論
3. 60 秒投票：輸入要放逐的玩家編號
4. 票數最高者出局，若平局則無人出局

**勝利條件**

- 村民陣營：所有狼人被消滅
- 狼人陣營：狼人數量 ≥ 村民數量

---

### RPG 跑團

#### 開局流程

```
/rpg [theme] [bio]     → 建立大廳（指令者為房主，可留主題／背景，也可以留自己的角色自介）
.join [角色自介]        → 其他玩家加入（2~6 人），自介可留可不留
.start                 → 房主開始遊戲
.rpg_cancel            → 房主或管理員取消
.rpg_status            → 隨時查看目前狀態（第幾幕、每人數值），不限房主
```

#### 遊戲機制

- 悠筑固定當說書人／GM，不是隊伍裡的角色——每輪生成場景敘述＋2~4 個選項，全員按鈕投票（多數決，平票或逾時沒人投就隨機選一個，故事一定要往下走）
- 投票時間過半還有人沒投，會提醒還沒投的人
- 每人有**力量／敏捷／智慧／魅力**四維屬性（加入時隨機 0~10），遇到有風險的選擇會擲骰（d20 + 隊伍裡該屬性最高的人加成）決定結果；natural 20 一定大成功、natural 1 一定大失敗，不受難度值影響
- 預期一場冒險 **12~20 幕**（`RPG_TARGET_SCENES_MIN/MAX`），最多 `RPG_MAX_SCENES`（預設 40）幕強制收尾；每輪會把目前進度回報給模型，避免劇情長度每次落差太大、常常太快收尾
- 擲骰失敗會累積「挫折值」（大失敗算兩倍），累積到門檻會提示模型讓氣氛更緊張、結局走向跟過程順不順有關——純粹是提示，不會用程式碼強制切結局分支，這個數值也刻意不對玩家顯示
- 場景生成強制呼叫 Gemini function calling（`present_scene`），回傳結構化內容而非自由文字，按鈕才能穩定對應到選項
- 完全存在記憶體、不進資料庫，一次玩完、中斷或重開就直接捨棄
- 相關參數（人數上限、屬性範圍、難度值範圍、預期幕數等）都在 [config.py](config.py) 的 `RPG_*` 常數，說書人的敘事風格／規則寫在 `RPG_NARRATOR_SYSTEM_PROMPT`
- 依賴 AI 聊天的 `GEMINI_API_KEY`，未設定的話沒辦法開跑團

---

## 計時設定（werewolf.py）

| 階段 | 時間 |
|------|------|
| 夜晚行動（狼 / 預） | 60 秒 |
| 女巫每步 / 獵人射擊 | 30 秒 |
| 白天討論 | 90 秒 |
| 投票 | 60 秒 |

---

## 注意事項

- Bot 需要有**私訊玩家**的能力（玩家未關閉「允許伺服器成員傳送私訊」）
- Slash commands 第一次啟動後約 1 小時才在全域生效；測試時可改用 guild sync
- `DISCORD_TOKEN.env` 請加入 `.gitignore`，不要上傳到公開 repo
