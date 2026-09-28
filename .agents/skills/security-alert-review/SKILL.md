---
name: security-alert-review
description: 以 Azure DevOps REST API 列出與檢視儲存庫的 Advanced Security 警示：相依套件弱點、機密外洩與程式碼掃描結果，可依嚴重性、狀態與警示類型篩選，也能產生總覽。當使用者想查看、篩選、摘要安全性警示，或深入了解某一則警示時使用。要檢查哪些儲存庫尚未啟用 Advanced Security 改用 `security-enablement-audit`。
compatibility: 需要 curl、可連線至 dev.azure.com 與 advsec.dev.azure.com 的網路，以及 AZURE_DEVOPS_ORG 與 AZURE_DEVOPS_PAT 環境變數；儲存庫需已啟用 Advanced Security。
---

# 安全性警示檢視

這個 Skill 在**專案**與**儲存庫**的範圍內運作，兩者都必須有才能取得警示。

## 專案選擇

- 使用者有提供專案名稱（例如「Contoso 專案」）就直接使用，不呼叫專案清單 API。
- 沒有提供時，請使用者提供一次。
- 仍未提供時，列出專案讓使用者選擇：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/projects?%24top=500&api-version=7.1"
  ```

## 儲存庫選擇

- 使用者有提供儲存庫名稱就直接使用。
- 沒有指定時，詢問一次儲存庫名稱。
- 仍未提供時，列出專案中的儲存庫讓使用者選擇：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories?api-version=7.1"
  ```

  顯示 `value[].name`。

## 連線與驗證

一律用 `curl` 直接呼叫 Azure DevOps REST API，每個指令都自帶驗證參數。

- **組織**：讀取環境變數 `AZURE_DEVOPS_ORG`（組織名稱，例如 `contoso`）。未設定時詢問使用者一次；使用者給的是網址時，從 `https://dev.azure.com/{org}` 或 `https://{org}.visualstudio.com` 取出組織名稱。
- **主機**：除了 `dev.azure.com`，Advanced Security 警示 API 的主機是 `advsec.dev.azure.com`。
- **權杖**：PAT 以 Basic 驗證送出：`-u ":$AZURE_DEVOPS_PAT"`；只設定了 `AZURE_DEVOPS_EXT_PAT`（Azure CLI azure-devops 擴充套件使用的變數）時，改用它。改用 Microsoft Entra ID 權杖時，換成 `-H "Authorization: Bearer $AZURE_DEVOPS_TOKEN"`。都未設定時，請使用者在自己的終端機設定 `AZURE_DEVOPS_PAT` 後重試，並停止。權杖只從環境變數讀取：不要請使用者把權杖貼進對話，也不要印出或寫入檔案。
- **PAT 權限範圍**：Advanced Security (Read)、Code (Read)、Project and Team (Read)。
- **編碼**：路徑中的專案與儲存庫名稱要 URL 編碼（空白寫成 `%20`）；查詢字串中的 `$` 寫成 `%24`（例如 `%24top`），避免被 shell 展開。
- **錯誤**：HTTP 401、403、203 或回應是 HTML 登入頁，代表權杖無效、過期或權限不足，回報後停止；HTTP 404 代表組織、專案或儲存庫名稱有誤。回應訊息含 `VS2150009` 代表這個儲存庫沒有啟用 Advanced Security，告知使用者後停止。
- **Windows PowerShell**：呼叫 `curl.exe`（而非 `curl` 別名），環境變數寫成 `$env:AZURE_DEVOPS_PAT`；PowerShell 不支援 `<<'EOF'`，先把 JSON 主體寫入暫存檔，再以 `--data-binary "@body.json"` 送出。

## 查詢參數

警示清單 API 以 `criteria.*` 查詢參數篩選，多個值用逗號分隔，值不分大小寫：

| 參數 | 可用值 | 範例 |
|---|---|---|
| `criteria.severities` | `critical`、`high`、`medium`、`low`、`note`、`warning`、`error`、`undefined` | 「重大警示」→ `critical` |
| `criteria.states` | `active`、`dismissed`、`fixed`、`autoDismissed` | 「作用中的警示」→ `active` |
| `criteria.alertType` | `dependency`、`secret`、`code`（單一值；另有較新的 `license`、`aiCode`） | 「相依套件警示」→ `dependency` |
| `criteria.confidenceLevels` | `high`、`other` | 預設帶 `high,other` |
| `criteria.onlyDefaultBranch` | `true`、`false` | 預設帶 `true` |
| `top` | 每頁筆數 | `top=100` |
| `continuationToken` | 上一頁回應標頭 `x-ms-continuationtoken` 的值 | 取下一頁時帶上 |

- 每次呼叫都帶 `criteria.confidenceLevels=high,other`，除非使用者明確要求其他信賴度篩選。未帶這個參數時，機密外洩警示只回傳高信賴度的結果。
- 帶了 `criteria.confidenceLevels=high,other` 卻回傳 0 筆時，拿掉這個參數再查一次；若因此查到警示，在結果中註明「只包含高信賴度的機密外洩警示」。
- 使用者沒有指定篩選條件時，顯示預設分支上所有作用中的警示：`criteria.onlyDefaultBranch=true&criteria.states=active&criteria.confidenceLevels=high,other`。

## 規則

### 1. 列出儲存庫的警示

- 使用者要**列出警示**、**顯示安全性警示**或**檢視警示**時，依使用者的要求組合篩選參數：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://advsec.dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/alert/repositories/{repository}/alerts?criteria.onlyDefaultBranch=true&criteria.states=active&criteria.confidenceLevels=high,other&top=100&api-version=7.2-preview.1"
  ```

- 以表格呈現結果。
- 沒有任何警示時，明確說明這個儲存庫沒有符合條件的警示。

範例：「顯示 Contoso 專案 MyApp 儲存庫的安全性警示」、「列出 MyApp 的重大相依套件警示」、「顯示 MyApp 所有作用中的機密外洩警示」

### 2. 取得特定警示的細節

- 使用者問起**特定警示**（例如「alert 42」、「說明一下 alert 42」）時：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://advsec.dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/alert/repositories/{repository}/alerts/{alertId}?api-version=7.2-preview.1"
  ```

- 取細節時只帶 `api-version`：`expand=ValidationFingerprint` 會讓回應包含明文機密，這個 Skill 不使用它。
- 呈現所有可取得的細節欄位，包括受影響的檔案、行號、說明、修補建議與規則資訊。

範例：「顯示 Contoso 專案 MyApp 儲存庫 alert 42 的細節」、「alert 42 是什麼問題？」

### 3. 總覽

- 使用者要**摘要**或**總覽**警示時，呼叫規則 1 的 API，不帶嚴重性與類型篩選，只帶 `criteria.states=active&criteria.confidenceLevels=high,other`，並依下列方式彙總：
  1. 依**警示類型**（Dependency、Secret、Code）計數。
  2. 在每個類型內依**嚴重性**（Critical、High、Medium、Low、Other）計數。
- 以精簡表格呈現彙總結果，最後附上總數。
- 呼叫時加上 `-D -` 以看到回應標頭。標頭有 `x-ms-continuationtoken` 時，代表還有更多警示：告訴使用者這份總覽只根據第一批警示，並另有其他警示存在。

範例：「給我 MyApp 儲存庫的安全性總覽」、「摘要 Contoso 專案 MyApp 儲存庫的警示」

## 呈現結果

日期格式為 `YYYY/MM/DD`。

警示清單以表格呈現：

- **警示 ID**（`alertId`）
- **標題**（`title`，或規則名稱）
- **嚴重性**（`severity`），加上 emoji：🔴 Critical、🟠 High、🟡 Medium、🟢 Low
- **狀態**（`state`：Active、Dismissed、Fixed、AutoDismissed）
- **警示類型**（`alertType`：Dependency、Secret、Code）
- **規則**（`tools[].rules[].friendlyName`，沒有時用 `opaqueId`）
- **首次發現**（`firstSeenDate`）

警示細節除清單欄位外，再加上：

- **說明**（`tools[].rules[].description`）：這則警示代表什麼問題的完整說明。
- **檔案路徑**與**行號**（若有）：`physicalLocations[].filePath` 與 `physicalLocations[].region.lineStart`。
- **修補建議**（`tools[].rules[].helpMessage` 與 `resources`，有提供時）：如何修正。
- **信賴度**（`confidence`）：High 或 Other（機密外洩警示）。
- **有效性**（`validityDetails.validityStatus`）：Active、Inactive 或 Unknown（機密外洩警示）。
- **工具名稱**（`tools[].name`）：發現這則警示的掃描工具。

總覽以下表呈現：

| 警示類型 | 🔴 Critical | 🟠 High | 🟡 Medium | 🟢 Low | Other | 總計 |
|------------|------------|---------|----------|--------|-------|-------|
| Dependency | 數量 | 數量 | 數量 | 數量 | 數量 | 數量 |
| Secret     | 數量 | 數量 | 數量 | 數量 | 數量 | 數量 |
| Code       | 數量 | 數量 | 數量 | 數量 | 數量 | 數量 |
| **總計**   | 數量 | 數量 | 數量 | 數量 | 數量 | 數量 |

**Other** 欄包含 Critical、High、Medium、Low 以外的嚴重性（例如 Note、Warning、Error、Undefined）。出現 Dependency、Secret、Code 以外的警示類型時，在表中多加一列。
