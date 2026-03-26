# aiDAPTIVAppStore — 應用程式自動更新功能驗證說明

## 概述

本文件說明如何在驗證環境中測試 aiDAPTIVAppStore 的「應用程式自動更新」功能。
開發單位會提供**舊版安裝檔**（受測初始版本）、**新版安裝檔**（模擬升級目標版本）以及對應的 **`stable.ver` 版本描述檔**，驗證單位只需架設一個可下載這兩個檔案的伺服器，並修改設定檔，讓應用程式向驗證伺服器查詢更新。

---

## 開發單位提供物

| 項目 | 檔名 | 說明 |
|---|---|---|
| 舊版安裝檔 | `aiDAPTIVAppStore_release.exe`（標記 OLD） | 驗證機器的初始安裝用 |
| 新版安裝檔 | `aiDAPTIVAppStore_release.exe`（標記 NEW） | 模擬升版目標，供驗證伺服器提供下載 |
| 版本描述檔 | `stable.ver` | 應用程式查詢更新所需的版本資訊檔，**由開發單位產生提供** |
| 本說明文件 | `update_validate.md` | 驗證操作步驟 |

---

## 驗證環境架設流程

### Step 1 — 安裝舊版應用程式

使用**舊版安裝檔**在驗證機器上執行首次安裝（正常雙擊安裝，依精靈步驟完成）。

安裝完成後，應用程式會安裝至：

```
%LOCALAPPDATA%\aiDAPTIVAppStore\
```

---

### Step 2 — 架設驗證用下載伺服器

驗證單位需將開發單位提供的 **`stable.ver`** 與**新版安裝檔**上傳至可存取的伺服器，並確保以下兩個 URL 可正常下載：

| 資源 | 說明 |
|---|---|
| **`stable.ver` 下載連結** | 開發單位提供的版本描述檔（應用程式讀取純文字內容，不檢查 MIME Type） |
| **新版安裝檔下載連結** | 提供新版 `aiDAPTIVAppStore_release.exe` 的直接下載連結 |

**架設方式（擇一）：**

#### 方法 A — Python 簡易 HTTP Server（內網）

將 `stable.ver` 與新版 `aiDAPTIVAppStore_release.exe` 放在同一目錄，執行以下指令（**不需要撰寫任何 Python 檔案**，`http.server` 為 Python 內建模組）：

```powershell
# 在兩個檔案所在目錄執行
python -m http.server 8080
```

執行後，該目錄下所有檔案即可透過 HTTP 下載：

- `stable.ver` URL：`http://<驗證機IP>:8080/stable.ver`
- 安裝檔 URL：`http://<驗證機IP>:8080/aiDAPTIVAppStore_release.exe`

> 驗證機 IP 可透過 `ipconfig` 查詢（取 IPv4 位址），伺服器啟動後保持終端機視窗開啟即可。

#### 方法 B — IIS 或其他 Web Server

將兩個檔案上傳至 Web Server，確認安裝檔可直接下載（非預覽或串流）即可。`stable.ver` 的 MIME Type 不影響功能。

#### 方法 C — 雲端儲存（S3、Azure Blob、Google Drive 直連等）

上傳檔案後取得**直接下載連結（direct download link）**，確認：
- 安裝檔連結可直接下載 `.exe`（非 HTML 頁面）
- `stable.ver` 連結回傳的是純文字內容（非重導向至 HTML 登入頁）

---

### Step 3 — 修改 `aidaptiv_update.json`

應用程式安裝後，設定檔位於：

```
%LOCALAPPDATA%\aiDAPTIVAppStore\aidaptiv_update.json
```

以文字編輯器開啟，將 `StableEndpoint` 與 `StableInstallerUrl` 修改為 Step 2 架設的驗證伺服器連結：

**修改前（預設值）：**

```json
{
  "StableEndpoint": "https://phison-software-bucket.s3.ap-northeast-1.amazonaws.com/aiDAPTIVAppStore/Staged/aiDAPTIVAppStoreUpdate/stable.ver",
  "BetaEndpoint": "https://phison-software-bucket.s3.ap-northeast-1.amazonaws.com/aiDAPTIVAppStore/Staged/aiDAPTIVAppStoreUpdate/stable.ver",
  "StableInstallerUrl": "https://github.com/aiDAPTIV-Phison/aiDAPTIVAppStore/releases/latest/download/aiDAPTIVAppStore_release.exe",
  "BetaInstallerUrl": "https://github.com/aiDAPTIV-Phison/aiDAPTIVAppStore/releases/latest/download/aiDAPTIVAppStore_release.exe"
}
```

**修改後（填入驗證伺服器連結）：**

```json
{
  "StableEndpoint": "http://<驗證伺服器IP或網域>/stable.ver",
  "BetaEndpoint": "https://phison-software-bucket.s3.ap-northeast-1.amazonaws.com/aiDAPTIVAppStore/Staged/aiDAPTIVAppStoreUpdate/stable.ver",
  "StableInstallerUrl": "http://<驗證伺服器IP或網域>/aiDAPTIVAppStore_release.exe",
  "BetaInstallerUrl": "https://github.com/aiDAPTIV-Phison/aiDAPTIVAppStore/releases/latest/download/aiDAPTIVAppStore_release.exe"
}
```

> **注意：** `BetaEndpoint` 與 `BetaInstallerUrl` 不需修改，保持原值即可（驗證僅針對 Stable 頻道）。

---

### Step 4 — 執行驗證

1. **啟動** aiDAPTIVAppStore（舊版）。
2. 應用程式會在背景自動向 `StableEndpoint` 查詢版本（等待網路連線後立即開始，首次約在啟動後數秒內觸發）。
3. 系統偵測到 `stable.ver` 中的 BuildNumber（例如 `200`）大於目前安裝版本（`109`），開始從 `StableInstallerUrl` 下載新版安裝檔。
4. 下載進行中，主視窗頂部 **UpdatesBanner**（InfoBar）會顯示下載進度訊息。
5. 下載完成後，系統以 SHA256 驗證安裝檔完整性。
6. 驗證通過後，InfoBar 顯示「aiDAPTIVAppStore X 已準備好安裝」並出現 **Update now** 按鈕，同時發送 Windows Toast 通知。
7. 點擊 **Update now**，應用程式關閉，安裝程式以 `/UPDATE` 參數靜默執行（跳過 Welcome / KV Cache 設定 / Finish 頁面，保留現有模型路徑設定，更新應用程式檔案）。
8. 安裝完成後，應用程式自動重啟。

---

## 常見問題排查

| 問題 | 可能原因 | 解決方式 |
|---|---|---|
| 應用程式啟動後沒有 InfoBar | `aidaptiv_update.json` 未正確修改，或伺服器無法存取 | 確認 URL 可從驗證機器直接開啟（瀏覽器測試） |
| InfoBar 顯示「An error occurred when checking for updates」 | `stable.ver` 格式錯誤（分隔符、欄位數量） | 確認格式為 `BuildNumber////Hash////VersionName`（四個正斜線） |
| 下載完成但 InfoBar 顯示「安裝檔驗證失敗」 | `stable.ver` 中的 Hash 與實際安裝檔不符 | 通知開發單位確認 `stable.ver` 中的 SHA256 Hash 與提供的安裝檔是否對應 |
| 安裝程式啟動後畫面閃過又消失 | `/UPDATE` 模式為靜默安裝（正常行為） | 等待 10-30 秒，應用程式會自動重啟 |
| `stable.ver` 連結無法存取或回傳 HTML 頁面 | URL 錯誤，或雲端連結需登入才能下載 | 確認 URL 可直接在瀏覽器開啟並看到純文字內容（格式：`數字////hash////版本名稱`） |
| BuildNumber 不符（已是最新）| `stable.ver` 中的 BuildNumber 未大於現有版本（109） | 確認 `stable.ver` 第一欄位為 > 109 的整數 |

---

## 附錄

### `stable.ver` 格式說明

```
{BuildNumber}////{SHA256HexString}////{VersionName}
```

- **分隔符**：`////`（四個正斜線），每個欄位之間各一組
- **BuildNumber**：純整數，不含引號或空白
- **SHA256HexString**：64 位元十六進位字串（大小寫均可）
- **VersionName**：任意可讀字串
- **換行**：檔案末尾可有或無換行，不影響解析

### `aidaptiv_update.json` 欄位說明

| 欄位 | 說明 |
|---|---|
| `StableEndpoint` | 正式頻道的版本描述檔（`stable.ver`）下載 URL |
| `BetaEndpoint` | Beta 頻道的版本描述檔下載 URL（驗證時不需修改） |
| `StableInstallerUrl` | 正式頻道的安裝檔（`.exe`）下載 URL |
| `BetaInstallerUrl` | Beta 頻道的安裝檔下載 URL（驗證時不需修改） |

### 安裝程式 `/UPDATE` 模式行為說明

安裝程式以 `/UPDATE` 參數執行時（由應用程式自動觸發，驗證單位無需手動操作）：

- 跳過 Welcome 歡迎頁面
- 跳過 KV Cache 目錄設定頁面（保留現有設定）
- 備份 `aidaptiv_config.json` 中的 `model_path` 欄位
- 複製新版應用程式檔案（**保留** `Models/` 目錄、`logs/` 目錄不覆蓋）
- 還原 `model_path` 至新的設定檔
- 跳過 Finish 完成頁面
- 安裝完成後自動重啟應用程式
