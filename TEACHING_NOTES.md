# FALO 企業環境部署評估矩陣：專案建置與技術教學講義
> 🎓 **Force 課程教學專用講義與實戰踩坑筆記**

本專案是一個非常經典的 **「純前端去中心化協作（Serverless Client-Side Collaboration）」** 實戰案例。本講義旨在為學生拆解這款評估矩陣工具的技術建置過程、在真實網路環境中碰到的核心問題（如 **CORS 跨域防護**、**DNS 假性阻擋**）以及最終的解決方案與黑科技。

---

## 學習目標
1. 理解如何利用單一 HTML + 響應式卡片流 (Mobile-First RWD) 滿足現場訪談的機動性。
2. 掌握如何利用 `mode: 'no-cors'` 探針繞過瀏覽器跨域限制，測試客戶端防火牆放行。
3. **（核心踩坑點）** 理解雲端架構的 DNS 解析機制，學會排除「DNS 網域不存在 (NXDOMAIN)」造成的假性網路阻擋。
4. 掌握「HTML 報告數據內嵌與回載技術」，在無資料庫情況下達成完整的數據交流閉環。

---

## 單元一：為什麼選擇「單一 HTML」與「行動優先 (Mobile-First)」？

### 1. 痛點分析
售前顧問進入企業客戶會議室時，網路往往極受限制，且需要快速查閱各平台指標。
* **無後端/無資料庫**：免去繁瑣的伺服器架構部署，100% 保障客戶企業的資安隱私，網頁下載下來直接離線即開即用。
* **行動優先 (Mobile-First RWD)**：
  - 傳統的「大矩陣表格」在手機上會被迫產生水平滾動條，極難閱讀。
  - 我們改用 **「卡片流 (Card Grid)」** 設計。手機上是垂直排列的溫潤水彩圓角卡片，觸控面積大；當螢幕拉寬到平板和電腦時，再利用 CSS Grid 自動擴展為雙欄或三至四欄佈局。
  - 頂部使用可單指滑動的分類列（Category Swiper），點擊快速篩選，大幅降低滾動頁面的疲勞度。

---

## 單元二：瀏覽器沙盒下的網路連線檢測 (Reachability Engine)

### 1. CORS 限制與跨域沙盒
當我們在 `file://` 或非目標伺服器的網域下，嘗試用瀏覽器對 `https://api.openai.com` 發送一般的 `fetch()` 請求時，瀏覽器會直接因為 **CORS (跨來源資源共用)** 安全政策而**攔截回應內容**，並在 Console 報出跨域錯誤。

### 2. 解決手段：無 CORS 連線探針 (no-cors)
為了解決這個限制，我們在 `fetch` 參數中加入了：
```javascript
fetch(url, {
  method: "GET",
  mode: "no-cors",  // 繞過 CORS 回傳讀取限制
  cache: "no-store",
  signal: controller.signal // 超時中止控制
})
```
* **原理**：`mode: 'no-cors'` 告訴瀏覽器：「我只是想把請求發出去建立連線，我不需要、也**不讀取**你回傳的具體 JSON 回應（此時回應類型會被瀏覽器封裝為 Opaque）」。
* **檢測邏輯**：
  - **連線成功**：只要 TCP/TLS 三向交握成功（不論 HTTP 狀態碼，因為無跨域權限，取得的狀態碼一律為 0），代表客戶網路**並未阻擋該網域**。
  - **受阻 (Blocked)**：如果客戶網路被防火牆（如 Fortinet, Palo Alto）阻擋或 DNS 遭劫持，Promise 會直接進入 `catch` 分支，此時便判定為受阻 ❌。
  - **超時 (Timeout)**：如果客戶內網把該網域丟棄（Drop Packet）導致連線無限期等待，系統會以 `AbortController` 於 5 秒後強制中止，並判定為受阻。

---

## 單元三：（核心重點）真實連線測試下的「DNS NXDOMAIN」與假性阻擋

### 1. 踩坑現況描述
在開發第一版時，顧問即使在**完全沒有防火牆限制的家用網路**中進行檢測，以下幾個網域依然顯示 **「受阻 ❌」**：
* `lambda.amazonaws.com`
* `execute-api.amazonaws.com`
* `bedrock.amazonaws.com`
* `openai.azure.com`
* `www.hinet.net`
* `fly.dev`

### 2. 原因剖析：DNS 解析失敗 (NXDOMAIN) 與安全阻擋
我們在終端機執行 `nslookup` (網域解析工具) 後，發現了根本原因：

#### A. 雲端服務的「非實體根網域」
* AWS 的服務端點是**區域型（Region-specific）**的。例如 AWS Lambda 在 DNS 中根本沒有 `lambda.amazonaws.com` 這個「根」紀錄（查詢會得到 `NXDOMAIN`，即網域不存在）。真正的實體解析端點必須是 `lambda.us-east-1.amazonaws.com`。
* AWS Bedrock 也是如此，根域名 `bedrock.amazonaws.com` 是 NXDOMAIN，必須帶有區域。
* Azure OpenAI 的網域必須包含客戶自己申請的資源前綴（如 `your-resource.openai.azure.com`），直接 ping `openai.azure.com` 是無 IP 回應的。

#### B. 通配域名安全政策與過期網域
* `fly.dev` 是 Fly.io 提供給成千上萬開發者的通配後綴（如 `appName.fly.dev`），其根網域 `https://fly.dev` 因為安全考量，拒絕了直接的網頁請求。
* `www.hinet.net` 在 DNS 中已不復存在，目前中華電信主業務網域已全數移轉。

因為 DNS 找不到這些域名，瀏覽器會直接回報 `ERR_NAME_NOT_RESOLVED` 錯誤，這是一種**假性的防火牆阻擋**！

### 3. 解決方案：更換為「實體可解析且通暢」的公共網域
為了解決此問題，我們對測試網域進行了修正，對照表如下，這些新網域皆能在沒有防火牆的網路下 100% 通暢解析：

| 平台類別 | 原配置網域 (假性阻擋) | 修正後網域 (正確連通) | 修正原因說明 |
| :--- | :--- | :--- | :--- |
| **AWS Lambda** | `lambda.amazonaws.com` | `lambda.us-east-1.amazonaws.com` | AWS 端點為區域型，補上 `us-east-1` 取得 DNS A 紀錄。 |
| **AWS API Gateway** | `execute-api.amazonaws.com` | `apigateway.us-east-1.amazonaws.com` | 根域名無解析。改用區域型 API Gateway 管理端點。 |
| **AWS Bedrock** | `bedrock.amazonaws.com` | `bedrock.us-east-1.amazonaws.com` | 補上區域，防止 DNS NXDOMAIN 錯誤。 |
| **Azure OpenAI** | `openai.azure.com` | `management.azure.com` | 改用 Azure 雲資源管理入口 (ARM API) 全球公共解析端點。 |
| **Fly.io** | `fly.dev` | `fly.io` | `fly.dev` 根網域限制連線，改用 Fly.io 官網主域。 |
| **HiNet 虛擬主機** | `www.hinet.net` | `www.cht.com.tw` | 原網域廢棄。改用中華電信公共官網。 |
| **HiNet 主機代管** | `colocation.hinet.net` | `hicloud.hinet.net` | 伺服器拒絕空連線，改用 hicloud 企業雲網域。 |

> 💡 **課堂教學提示**：可以引導學生在終端機輸入 `nslookup lambda.amazonaws.com` 來親自體驗 `NXDOMAIN` 的回報，並對比 `nslookup lambda.us-east-1.amazonaws.com` 成功返回 IP 的差別，這能讓學生深刻理解 DNS A 紀錄在網路開發中的底層原理。

---

## 單元四：去中心化雙向交流黑科技 —— HTML 數據內嵌技術

這套工具最酷的地方在於：**顧問發出去的 HTML 評估報告，不僅能用來閱讀列印，它自己本身也是一個備份數據包！**

### 1. 數據是如何被隱藏內嵌的？
在顧問點擊「產生 HTML 報告」時，JavaScript 會將當前記憶體中的 `platforms` 資料陣列與客戶名稱封裝為 JSON，並進行以下處理：
```javascript
// 1. JSON 字串化
const jsonStr = JSON.stringify(backupPayload);
// 2. 避免中文 UTF-8 編碼錯誤的萬用瀏覽器編碼套路
const latinStr = unescape(encodeURIComponent(jsonStr));
// 3. 轉為 Base64
const serializedData = btoa(latinStr);
```
隨後，我們將這段 Base64 字串寫入導出報告最底部的隱藏標籤中：
```html
<div id="falo-embedded-data" style="display:none;" data-payload="[Base64加密數據]"></div>
```

### 2. 如何反向還原 (HTML Parser)？
當顧問拿到客戶發回的 `Report.html` 報告並上傳時，我們在純前端使用瀏覽器內建的 **`DOMParser`** 進行解析：
```javascript
// 1. 將讀取的 HTML 純文字轉換為 DOM 物件
const parser = new DOMParser();
const doc = parser.parseFromString(htmlText, "text/html");
// 2. 尋找隱藏的資料 DIV
const div = doc.getElementById("falo-embedded-data");
// 3. 取得 Payload 並還原解碼
const encodedPayload = div.getAttribute("data-payload");
const decodedJson = decodeURIComponent(escape(atob(encodedPayload)));
const parsedPayload = JSON.parse(decodedJson);
```
這套黑科技免去了「既要傳 HTML 報告、又要傳 JSON 資料」的雙檔案麻煩，達成了**「報告即資料，資料即報告」**的無縫協作體驗，這在前端開發中是非常高級且實用的交互設計。

---

## 單元五：課堂思辨與技術盲點反思（本案例之核心靈魂）

在 Force 老師帶領學生進行這個專案的演練時，以下兩個核心觀念是極具啟發性的「工程思維」教材，能讓學生跳脫單純的寫 Code 框架，站在架構師與資深顧問的維度思考：

### 概念一：優先排除「軟體配置缺陷」，再評估「環境網路阻擋」
在實務開發或顧問現場，當我們看到一個連線顯示「受阻 ❌」時，工程師非常容易陷入直覺偏誤：**「一定是客戶那邊的防火牆或安全閘道把連線擋掉了。」**
* **本案例的教訓**：
  在第一版的網域測試中，我們在**完全沒有防火牆限制**的網路環境下測試，卻有 7 個網域回報失敗。這並非環境問題，而是我們代碼中「測試網址配置錯誤」（填寫了無 DNS 解析的虛擬根網域）。
* **課堂思維引導**：
  如果顧問沒有經過本地解析驗證，就直接把這份錯誤的清單丟給客戶的資安窗口說：「請幫我們在防火牆放行 `lambda.amazonaws.com` 與 `openai.azure.com`」。
  1. 客戶的網管工程師在設定時會發現「這些網域根本沒有 IP 紀錄，無法配置防火牆策略」，直接退件。
  2. 這會立刻讓客戶質疑 FALO 顧問團隊的技術專業度，並在無意義的溝通中白白浪費數天時間。
* **結論**：工程師必須建立**「先驗證本地配置正確性（如用 nslookup 確保 DNS 解析正常），再指控網路環境阻擋」**的標準排查思維。

### 概念二：AI 生成內容不等於「正確」或「符合實戰」
在生成式 AI (Generative AI) 時代，像本專案這類包含 50 個平台與大量 JSON 欄位的程式碼，交給 AI 輔助生成（AI-Assisted Coding）可以在 10 秒內完成，大幅縮短開發時間。然而，這也帶來了關鍵的盲點：
* **AI 的盲點**：
  AI 是基於概率生成文字與代碼的。當你讓它生成 AWS 或 Azure 平台的測試網址時，它會按照語言邏輯，吐出「看起來極度合理、拼字完全正確」但**實際上根本不存在於真實 DNS 系統中**的網址（如 `openai.azure.com`）。
* **語法正確 vs. 實務合用**：
  AI 生成的這段程式碼，其語法結構是 100% 正確的，瀏覽器執行也不會報出 JavaScript 程式語法錯誤。但當它被放到真實網路環境下運作時，它卻是失效的。
* **給學生的啟示**：
  AI 是極佳的「加速器」，但它無法替工程師做「真實物理世界的狀態驗證」。
  這說明了：**在 AI 時代，工程師的基本功（如掌握 DNS 解析、CORS 原理、TCP 握手與 Chrome 開發者工具排查）不僅沒有貶值，反而變得更加珍貴。** 只有具備紮實底層知識的開發者，才能進行「人肉審查」與「實地 Debug」，將 AI 生成的「半成品模型」修正為真正符合企業實戰的穩定方案。

---

## 單元六：網頁版權防護黑科技 —— 多層次隱形浮水印 (Watermark)

在發佈開源教學工具或將評估表提供給第三方客戶自填時，如何防範剽竊、抄襲與商用代碼剝離？本專案在前端設計了**「多層次隱形浮水印 (Multi-layered Invisible Watermarks)」**防護機制：

### 1. HTML 註解浮水印 (Comment Watermark)
* **實作**：在 HTML 頂部和核心函數內嵌開發版權與發佈日期 `Falo x Force Cheng 2026/6/22`。
* **效果**：防範初階開發者使用「檢視網頁原始碼」直接複製 HTML，並留存原始開發印記。

### 2. DOM 隱身浮水印 (CSS Invisible DOM Watermark)
* **原理**：利用 CSS 的遮罩與絕對裁切技術（Absolute Clipping），建立一個既在 DOM 樹中存在，但又完全不佔用版面、不被視覺察覺的 DOM 節點。
* **代碼實作**：
  ```css
  .falo-invisible-watermark {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0); /* 舊版瀏覽器相容裁切 */
    white-space: nowrap;
    border: 0;
    pointer-events: none; /* 穿透點擊，防止滑鼠選取 */
    user-select: none;
    color: rgba(0, 0, 0, 0.001); /* 極限近乎透明 */
    background: transparent;
  }
  ```
* **效果**：該節點對訪客不可見，但當抄襲者直接複製 DOM 程式碼，或者爬蟲抓取網頁內容時，該浮水印文字將會被一併複製，成為侵權鐵證。

### 3. JavaScript 全域唯讀浮水印 (JS Read-Only Watermark)
* **原理**：利用 ES5 的 `Object.defineProperty()` 註冊全域變數，並將 `writable` 與 `configurable` 設為 `false`，防範第三方指令碼惡意覆寫或刪除版權。
* **代碼實作**：
  ```javascript
  Object.defineProperty(window, '__FALO_WATERMARK__', {
    value: "Falo x Force Cheng 2026/6/22",
    writable: false,      // 防範覆寫：window.__FALO_WATERMARK__ = 'new value' 將會失效
    enumerable: true,     // 可枚舉，便於自動化稽核
    configurable: false   // 防範刪除：delete window.__FALO_WATERMARK__ 將會失效
  });
  ```
* **教學提示**：可讓學生在瀏覽器開發者工具的 Console 中嘗試執行 `window.__FALO_WATERMARK__ = 'test'`，觀察其值是否依然保持不變，理解 JavaScript 元程式設計（Metaprogramming）在安全防護上的實踐。

---

## 單元七：智慧型搜尋優化與 AI 友善爬取 (GEO)

隨著 Perplexity、Gemini、ChatGPT Search 等 AI 搜尋引擎的普及，傳統的 SEO（搜尋引擎優化）已轉變為 **GEO (Generative Engine Optimization，生成式引擎優化)**。如何讓我們的教學網頁更易被 AI 搜尋並準確生成解答？

### 1. JSON-LD 語意化 Schema 結構化數據
在 `<head>` 中置入 `application/ld+json` 格式，主動向 AI 提供本軟體的屬性：
* 宣告 `@type: "SoftwareApplication"`，並指明 `name`、`author` (Force Cheng) 與 `datePublished` (2026-06-22)。
* 這使得 AI 爬蟲無須進行語意猜测，即可 100% 準確地將 Force Cheng 歸類為此工具的原創作者。

### 2. 結構化問答定義區 (FAQ Schema)
AI 搜尋引擎傾向於直接摘錄有明確「問答對應 (Q&A)」的段落。我們在「評估指南」分頁中建立了 FAQ 段落：
* 提供明確的標題與條理清晰的答覆（如：繞過 CORS、處理 DNS NXDOMAIN、偵測 AdBlocker 的具體技術實現）。
* **GEO 優勢**：當使用者向 AI 提問「如何用純前端偵測 AdBlock？」或「為什麼 Azure OpenAI 直接檢測網域會 NXDOMAIN？」時，AI 搜尋引擎能精準定位並提取我們網頁的 FAQ，給予高權重排名並引用為資料來源。

---

## 單元八：瀏覽器指紋追蹤與環境曝光 (Cleartext Diagnostics)

本專案特意將收集到的客戶端特徵以**「明碼 (Cleartext) / 明文」**呈現在環境診斷面板與導出的 HTML 報告中。這包含了深層的資安教學意義：

### 1. 偵測原理與隱私邊界 (Privacy Boundaries)
* **AdBlocker 廣告防護偵測**：
  - **原理**：瀏覽器的 AdBlock 插件是基於「名單攔截（Blacklist rule-matching）」工作的。當偵測到載入 Google AdSense 主腳本 `adsbygoogle.js` 時，插件會直接將其 Drop。
  - **資安手法**：我們透過前端 `fetch(adTestUrl, { mode: 'no-cors' })` 模擬請求該檔案，一旦連線失敗，即可判定訪客啟用了 AdBlocker。這展示了如何利用「異常連線反向推導訪客軟體特徵」。
* **其他設備指紋蒐集**：
  - `navigator.connection` (Network Info API) 暴露頻寬與延遲。
  - `navigator.deviceMemory` 與 `hardwareConcurrency` 暴露 CPU 與記憶體規格。
  - `Intl.DateTimeFormat().resolvedOptions().timeZone` 暴露訪客的精確地理時區。
  - `navigator.getBattery()` (Battery API) 暴露設備剩餘電量。

### 2. 資安教育思維：為什麼明碼顯示？
* **消除技術神秘感**：許多用戶以為靜態網頁（Static HTML）沒有後端資料庫，就不會收集或傳送個人隱私。我們以明碼將這些數據直接拍在用戶眼前，警告用戶：「只要你打開一個網頁，你的 IP、位置、電量和硬體指紋就已經被靜態代碼瞭若指掌了。」
* **明碼傳輸風險示範**：當這份報告被匯出並在顧問與客戶間傳遞時，所有系統特徵均在 HTML 中明文保存。這可用於示範「敏感特徵（Metadata）未加密傳輸的資安風險」，提升學生對於數據傳輸安全（Data-in-Transit Security）的警覺度。
