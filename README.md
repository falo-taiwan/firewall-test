# FALO Enterprise Deployment Assessment & Compatibility Matrix
> 💼 **企業環境部署評估與網關相容性矩陣工具**

這是一款專為售前顧問、架構師與項目經理（PM）設計的標準評估工具。協助顧問在導入 AI 代理、自動化 Workflow、企業知識庫、RPA 等方案時，迅速核對客戶端的網路安全環境、白名單放行策略與 Gateway 架構相容性。

本專案支援 **100% 離線運行**，並針對行動裝置進行了**手機閱讀與操作導向（Mobile-First RWD）**的介面優化，非常適合顧問在企業訪談現場使用手機或平板進行快速勾選與檢測。

---

## ⚡ 一鍵線上評估 (GitHub Pages)

您可以直接將此儲存庫部署至 **GitHub Pages**，即可獲得專屬的線上評估網址。
* 顧問在現場可直接分享網址給客戶 IT 窗口自主填寫。
* 客戶檢測完成後，將評估報告匯回給顧問，形成高效的無伺服器協作閉環。

---

## 🌟 核心功能特點

1. **📱 手機導向之響應式 UI (Mobile-First RWD)**
   * 使用底部「Dock 浮動式導航列」切換不同功能分頁，操作直覺。
   * 以「自適應卡片流（Card Grid）」取代複雜表格，手機上自動垂直堆疊排列，平板與電腦版自動分流成多欄佈局。
   * 頂部加入可單指橫向滑動的「分類標籤列 (Category Swiper)」，點擊一鍵快速過濾平台。

2. **⚡ 白名單網路連線檢測 (Reachability Engine)**
   * 卡片中直接整合 `mode: 'no-cors'` 跨域網路探測技術。
   * 顧問或客戶連線至公司 Wi-Fi 後，點擊「一鍵檢測連線」，系統會在 10 秒內跑完 50 個熱門網域的 TCP 握手檢測。
   * 檢測結果會即時回報為「通暢 ✅」或「受阻 ❌」，並自動預設標記為「可用」或「不可用」，免去手動勾選的痛苦。
   * **併發控制**：測試採用背景佇列（最大並行數 = 5），防止瀏覽器 Socket 阻塞。

3. **📋 技術能力摺疊細節 (Information Progressive Disclosure)**
   * 卡片預設隱藏繁雜的能力標籤，僅保留推薦等級與評估狀態。
   * 點擊摺疊按鈕後，才會展開顯示「7 大部署技術相容性（Static, Dynamic API, Webhook, OAuth, Reverse Proxy, Custom Domain, Fixed IP）」與「導入難度評估分值（1~5 分）」。

4. **🔄 數據雙向交流 (Data Exchange Closing Loop)**
   * **匯出 CSV**：一鍵導出標準 CSV，已做 UTF-8 BOM 處理，保障 Excel 開啟中文字元絕無亂碼。
   * **匯出 JSON**：匯出完整的當前矩陣狀態。
   * **匯出 HTML 報告**：匯出包含圖表統計與「一鍵複製網管放行名單」的精美 HTML 報告。
   * **雙模匯入還原**：工具不僅支援上載 `.json`，更支援**直接將剛才匯出的 `.html` 報告上傳回工具**，工具會自動解析其中隱藏的加密數據包並完美還原評估矩陣，實現真正的無縫溝通！

---

## 🛠️ 技術架構

* **核心技術**：HTML5, CSS3 (Vanilla CSS), JavaScript (ES6)
* **字型資源**：Google Fonts (Plus Jakarta Sans & Outfit)
* **依賴關係**：0 外部 JS 框架依賴（無 jQuery, React 或 Vue），無需伺服器或後端資料庫，100% 客戶端運行，保障企業資安隱私。

---

## 📖 使用指南與協作流程

1. **顧問設定 (或直接發送網址)**：
   * 顧問直接將發布的 Pages 網址傳送給客戶的資訊部門窗口。
2. **客戶現場盲測 (網路檢測分頁)**：
   * 客戶在受訪網絡環境下點開連結，填入公司名稱，並點擊「一鍵檢測連線」，測試完畢後在報告導出頁點擊「產出 HTML 報告」並回寄給顧問。
3. **顧問專業評估 (相容矩陣分頁)**：
   * 顧問在自己的設備上點擊「載入評估報告」，上傳剛才客戶的 HTML 報告。
   * 工具會瞬間還原客戶在該網絡環境的測試結果。顧問可逐一核對受阻網域，展開技術細節，為客戶標記解決方案（如使用特定 Proxy、改用其他備選平台）並填寫專業顧問備忘。
4. **輸出決策報告**：
   * 顧問最終再次導出正式版 HTML 報告，交給客戶 IT 部門進行防火牆配置。

---

## 🌐 本地開發與內網穿透 (轉 Port / Tunnel 服務)

本系統為純前端靜態架構，通常直接雙擊網頁檔案或透過 GitHub Pages 即可運行。然而，若顧問在**本地修改代碼但尚未 Push 到 GitHub**，或者**需要將本地伺服器臨時展示給外網的客戶**時，可以使用轉 Port 工具進行「內網穿透」。

本專案支援並推薦以下 9 種熱門的內網穿透工具。在啟動前，請先於本地專案目錄啟動一個網頁伺服器（例如使用 Python）：
```bash
python3 -m http.server 8000
```

隨後您可以選擇以下任一方案對外分享您的本地伺服器：

### 1. Cloudflare Tunnel (cloudflared) -【企業首選，無訪客警告頁】
完全免費且連線極度穩定。
```bash
# 安裝 (Mac/Homebrew)
brew install cloudflared

# 啟動臨時 Tunnel 並獲取專屬網址
cloudflared tunnel --url http://localhost:8000
```

### 2. ngrok -【老牌穿透工具】
成熟普及，但免費版在訪客開啟網址時會出現一次性 ngrok 警告頁面。
```bash
ngrok http 8000
```

### 3. Localtunnel -【免註冊、最快速】
支援自訂二級域名，開箱即用。
```bash
npx localtunnel --port 8000 --subdomain falo-test
```

### 4. SSH 反向隧道 (serveo.net / localhost.run) -【本機零安裝黑科技】
電腦只要有 `ssh` 指令即可運行，完全不需要安裝額外軟體：
```bash
# 方案 A：使用 Serveo.net
ssh -R 80:localhost:8000 serveo.net

# 方案 B：使用 Localhost.run
ssh -R 80:localhost:8000 nob@localhost.run
```

### 5. VS Code 內建埠轉送 (Port Forwarding)
若您使用 VS Code 開發，可以直接在下方的 **「Ports (連接埠)」** 面板中點選 **「Add Port」** 輸入 `8000`，並將存取權限設為 **Public (公開)**。

### 6. 其他支援方案
專案同時相容 **Bore** (`bore local 8000 --to bore.pub`)、**Tailscale Funnel**、**Pinggy** (`ssh -t a:localhost:8000+ssl@connect.pinggy.io`) 及企業自建的 **frp (Fast Reverse Proxy)**，詳情與資安原理請參見 [TEACHING_NOTES.md](file:///Users/force/Google_Antigravity/firewall_ip_website/TEACHING_NOTES.md#單元九顧問現場調試與內網穿透技術-port-forwarding--tunneling)。
