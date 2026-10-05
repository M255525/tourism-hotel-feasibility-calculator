# CLAUDE.md

本檔案為 Claude Code 在此子資料夾工作時的指引。此資料夾**本身是獨立 git 儲存庫**，不受根目錄工作區規則約束（除語言等全域偏好）。

## 這是什麼

**觀光地區飯店／旅館的一般用途財務可行性試算工具**（2026-10-05 建立），比照姊妹專案 `資料儀表板/restaurant-feasibility-calculator` 的架構與外殼，主題換成觀光飯店。文案沿用餐飲版的教訓：用「你的觀光飯店」第二人稱，數字全是可調整起始值，不描述任何一家真實飯店。**單檔前端，無後端，無序號授權。**

飯店的推算鏈跟餐飲版不同，所以領域層是重寫的、不是改字：
房間數 → 各季（旺／平／淡）× 平日／假日 住房率 × 房價 → 客房營收 ＋ 附加營收 → 變動成本（備品洗滌、早餐、OTA 佣金、占營收比例成本）→ 固定成本（租金、人事、折舊）→ **逐月**淨利 → 全年加總。

## 架構

`index.html` 單一 IIFE `<script>`，無外部資料檔。外殼逐字沿用餐飲版（`buildGroupUI`/`buildVarCard`、`tweenNumber`、`healthState`、`AI_PROVIDERS`/`callLLM`/BYOK 面板、已儲存方案、`buildPrintReport`+`#pdfWatermark`、跑馬燈 IIFE、PWA 安裝腳本）——改其中一邊時考慮是否同步另一邊。

- `VAR_DEFS`／`GROUPS`：7 個分頁（客房與空間／房價與住房率／淡旺季月曆／通路與附加營收／變動成本／人事編制／CAPEX 與押金）。「淡旺季月曆」是 `custom:true` 的特殊分頁，不從 `VAR_DEFS` 產生滑桿，而是 12 顆點擊循環 旺→平→淡 的按鈕（`renderSeasonEditor`/`initSeasonEditor`）。
- `state.seasonMap`：長度 12 的 `"peak"|"shoulder"|"off"` 陣列，**不在 `VAR_DEFS` 裡**。所有寫入 state 的路徑（localStorage 還原、preset、載入已存方案）都走 `assignState()`，它會驗證 seasonMap 格式並 `slice()` 複製——直接 `Object.assign` 會讓 state 跟 preset 共用同一個陣列，點月份時把 preset 本身也改掉。重設時要另外還原 `DEFAULT_SEASON_MAP`。
- `calculate()`：逐月計算。每月固定以平日 22 天＋假日 8 天概算（全年 360 晚），不做真實行事曆。旺季房價 ×(1+旺季加價)、淡季 ×(1−淡季折扣)，假日再 ×(1+假日加價)。
  - **損益兩平住房率**用封閉解：所有變動項都跟住房率線性成正比，住房率等比例縮放 k 倍時淨利＝k×貢獻毛利−固定成本，所以 k*＝固定成本÷貢獻毛利，再乘上目前全年住房率。未處理住房率超過 100% 的上限（>100% 時 UI 會標「不可能達成」）。
  - 回本年數＝（每房造價×房數＋押金）÷（淨利＋折舊）。
- `BANDS`：票根判色門檻集中在這裡（租金 ≤20%、人事 22–32%、OTA 佣金 ≤9%、淨利率 ≥10%、回本 ≤8 年等）。
- Signature element：`renderRoomGrid`（每格一間房，依 `viewMonth`/`viewDayType` 點亮售出房數）＋ `renderMonthChart`（inline SVG 12 個月淨利正負長條，點長條同步切換客房圖月份）。

### BUSINESS_PRESETS（6 種型態）

海濱度假／溫泉／山林景觀小旅店／離島／古城觀光商旅／親子度假。每組覆寫房價、六組住房率、`seasonMap`、OTA、附加營收**與人事編制**。第一版人力配得太少，人事只占 14–18%、淨利率 22–32%，太樂觀（跟餐飲版踩過的「固定人事沒有跟著營收規模縮放」是同一類問題，方向相反）；已依房數調高人力與租金，用 node 重算後落在：人事 21–31%、淨利率 7–19%。**海濱、山林、離島三組刻意保留 8 個月虧損**（全年仍小幅獲利）——季節性強的觀光地區本來就是這樣，這是工具要呈現的重點，不要為了好看把虧損月份調掉。

驗證計算可以不開瀏覽器：用 node 讀 `index.html`，擷取主 script 裡 `var WD_DAYS` 到 `/* ---------- Build variable panel UI` 之間的片段，再用 `new Function` 包起來回傳 `calculate`/`assignState`/`BUSINESS_PRESETS`（這個區段不碰 DOM，只需要提供一個假的 `localStorage`）。

### localStorage（前綴 `hotelCalc*`，跟餐飲版同為 m255525.github.io 同源，不能共用 key）

- `hotelCalcState`：目前變數＋seasonMap
- `hotelCalcApiConfig`：`{provider, model, apiKey, extra}`
- `hotelCalcActivePreset`：目前選取中的型態 id
- `hotelCalcSavedPlans`：已儲存方案 `{id,name,savedAt,activePresetId,state}[]`
- `hotelCalcMarquee`：跑馬燈快取

## PDF／跑馬燈／PWA／手冊

- PDF：`buildPrintReport()` 組靜態報表（規模摘要、六組變數、12 個月逐月損益表、全年票根、AI 診斷）→ `window.print()`；`@media print` 用 `body > *{display:none}`，不要改用 `visibility:hidden`（餐飲版踩過，會多印空白頁）。浮水印 `<img id="wmImg">` 的 base64 是用 Python 從餐飲版 `index.html` 整行複製過來的（馬克老師品牌圖），不要經過對話視窗貼上。
- 跑馬燈：共用工作區同一顆 GAS 端點（`MARQUEE_CHECK_URL`），滑鼠移入時暫停。
- PWA：`manifest.json`、`service-worker.js`（cache 名 `hotelcalc-shell-v1`）、`icons/`（PIL＋msjhbd.ttc，深海藍 `#0E3B4E` 底＋米白「宿」字；產生腳本未進 repo）。
- `manual.html`：內容依飯店改寫（含淡旺季月曆、ADR/RevPAR/損益兩平名詞解釋、旅館業登記類別警語）；**創作者資料區塊逐字沿用**姊妹專案。
- 訪客計數器：`page_id=m255525.tourismhotelfeasibilitycalculator`。

## 部署

2026-10-05 經使用者同意後已推公開 GitHub repo：<https://github.com/M255525/tourism-hotel-feasibility-calculator>，並以 `.github/workflows/deploy-pages.yml`（Actions 部署模式，branch `master`，不是 legacy branch-source）啟用 GitHub Pages：<https://m255525.github.io/tourism-hotel-feasibility-calculator/>。push 到 master 就會自動重新部署。

## 指令

無建置步驟。預覽：port `8824`（`.claude/launch.json` 的 `tourism-hotel-feasibility-calculator`），或 `python -m http.server 8824 --directory 資料儀表板/tourism-hotel-feasibility-calculator` 暫時啟動、測完關掉。
