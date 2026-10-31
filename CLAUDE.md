# 參數相框（Photo_Frame）開發說明

替照片加上外框與拍攝參數（EXIF）的單頁 PWA。網址：https://roysph1013.github.io/Photo_Frame/

## 與使用者合作的方式

- 使用者以繁體中文溝通；所有文字一律寫「臺灣」，不寫「台灣」（含臺海、對臺等）。
- 重大的介面或架構改動，先提出方案並取得同意再動手；小修正可直接做。
- 每次修改後要實際用 Playwright 開頁面驗證（截圖、檢查 console 錯誤），再推送。
- 使用者喜歡簡約、類似 Apple 的設計美學，介面與操作邏輯必須一致。
- 主要使用情境：Android 手機（Chrome、Samsung 網際網路）安裝成 PWA；也要顧及 iPhone Safari。
- 使用者的相機：Sony a6700（之前用 ZV-E10 II）。

## 檔案結構

- `index.html`：整個 App（HTML、CSS、JS 全部在一個檔，約 1.3 MB，內含 base64 的內建 Logo 圖）。
- `sw.js`：Service Worker。主頁面網路優先（`cache:'no-cache'`），字體與 cdn.jsdelivr.net 快取優先。
- `manifest.webmanifest`、`icon-*.png`、`apple-touch-icon.png`：PWA 安裝用。
- `.nojekyll`：**必須保留**。少了它，GitHub Pages 會用 Jekyll 處理大檔案，建置會卡住超過 10 分鐘並被取消。

## 每次發佈必做

1. `index.html` 裡的 `const APP_VER = 'vNN'` 加一（點畫面標題會顯示版本）。
2. `sw.js` 的 `const CACHE = 'photo-frame-vNN'` 同步加一。
3. 推送後用 `gh run list -R roysph1013/Photo_Frame` 確認 pages build 成功。
4. 連續推送會取消前一次建置，最後一次成功即可。

## 介面結構（v26 起）

- 頂部：左「選照片」、中「參數相框」（點擊顯示版本）、右「使用說明（問號）、復原、重做」。
- 使用說明：`helpHTML()` 產生的底部彈出頁，iOS「新功能」式圖示列表＋手勢示意圖；新增功能時要同步更新說明內容。
- 底部第一層：五個圓形按鈕「版型、照片、外觀、資訊、保存」。
- 點進去是第二層：分段控制切換細項，右側橘色圓形打勾鈕回到第一層。
  - 版型：版型（星號最愛排前面）／我的樣式
  - 照片：比例・裁切（裁切只用手指：拖曳、雙指縮放，不放滑桿）／調色
  - 外觀：邊框／Logo／字型／位置（文字位置與大小用手指調整，只保留邊距、間距、行距滑桿）
  - 資訊：上方欄位膠囊列＋下方單一輸入框（型號、鏡頭、焦段、光圈、快門、ISO、時間、地點、署名、引言）
  - 保存：保存、分享、全部保存（逐張存檔，不打包 ZIP）；大小滑桿（IG 用→1600→2048→3072→4096→原圖，下方以「萬畫素・檔案大小」顯示，不放大超過原圖）、格式、EXIF、GPS；JPG 品質固定 0.92
- 面板最高 34vh，內容多時在面板內捲動，預覽區要維持夠大。
- 首次開啟顯示引導畫面（`#empty`），內部仍保留範例照片供「先用範例看看」。

## 設計系統規則

- 純黑底，灰階 `--fill / --fill2 / --fill3`，唯一強調色 `--accent:#f0a35e`。
- 文字選項一律用膠囊 `.chip`（選中填橘色），單排可左右滑動，不折行。
- 圖像選項（版型縮圖、Logo、款式、字體、顏色）一律用外圈光環 `box-shadow:var(--ring)`。
- 設定列一律「左標籤 56px、右控制」`.row`；滑桿數值點兩下歸零（`data-reset`）。
- 說明文字盡量不用，只保留看不出來的操作提示。
- 提示訊息（toast）顯示在畫面上方。

## 重要邏輯

- 設定存在 `localStorage['exif-frame-settings']`（物件 `S`，預設值 `DEF`）；Logo 與資料夾權限存在 IndexedDB。
- 每張照片各自的設定：`data`（文字欄位）、`adj`（調色）、`zoom/panX/panY`（裁切）、`brandKey`。其餘為全域。
- 版型定義在 `TPL`，每個版型回傳 `{W,H,photo(),info()}`；比例固定時外框不變、照片自動裁切。
- Logo 大小：`opticalScale(aspect) = 1.18 * aspect^-0.36`，再乘 `BUILTIN[x].weight` 個別修正
  （GR 0.78、SONY 純字 0.9、富士圓徽與盒標 1.15）。新增 Logo 時視覺上要與其他品牌一致。
- 內建 Logo：`BUILTIN`（light／dark，Apple 另有 gray、color）；多款式品牌用 `VARIANTS`（SONY：α／純字；富士 5 款）。
- 相機型號易讀名稱：`friendlyModel()`（ILCE-6700→α6700、ZV-E10M2→ZV-E10 II 等）。
- 輸出：`exportBlob()` → 依尺寸縮放 → `withExif()` 把原始 EXIF 寫回（方向設 1、移除 IFD1 縮圖、GPS 依設定清空）。
- HEIC：先用瀏覽器原生解碼，失敗才載入 heic2any；HEIC 的 EXIF 用 exifr 讀（皆從 cdn.jsdelivr.net）。
- GPS 地名：OpenStreetMap Nominatim 反查，手動按鈕觸發，連續查詢間隔 1.1 秒。
- 復原／重做：`histPush()` 記錄 `S` 與目前照片的個別設定；`save()` 會自動呼叫。

## 測試素材建議

- 用 Pillow 產生帶 EXIF（含 GPS、Orientation 6）的 JPG 測試 EXIF 保留與方向。
- 真實 iPhone HEIC 範例：exifr repo 的 `test/fixtures/heic-iphone.heic`。
- 測試時攔截 `fonts.googleapis.com`（沙箱連不到）；CDN 可用 npm 安裝的本地檔案代替。

## 尚未決定的事

- GPS 目前預設不保留（隱私考量），使用者正在考慮是否改為預設保留並加上提示。
