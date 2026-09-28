---
name: pipelines-build-summary
description: 以 Azure DevOps REST API 列出、檢視與排查 Pipeline 建置：顯示某個 Pipeline 定義的近期建置、單一建置的狀態與結果、失敗步驟的記錄檔，以及建置包含的變更。當使用者問起建置清單、建置狀態、建置為何失敗或建置內含哪些 commit 時使用。要觸發 Pipeline 改用 `pipelines-run`，看自動化測試結果改用 `pipelines-test-results`，產生版本說明改用 `pipelines-release-notes`。
compatibility: 需要 curl、可連線至 dev.azure.com 的網路，以及 AZURE_DEVOPS_ORG 與 AZURE_DEVOPS_PAT 環境變數。
---

# Pipeline 建置摘要

這個 Skill 在**專案**的範圍內運作。是否需要 **Pipeline 定義**或**建置 ID**，取決於使用者的問題。

## 專案選擇

- 使用者有提供專案名稱（例如「Contoso 專案」）就直接使用，不呼叫專案清單 API。
- 沒有提供時，請使用者提供一次。
- 仍未提供時，列出專案讓使用者選擇：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/projects?%24top=500&api-version=7.1"
  ```

## Pipeline 定義選擇

- 使用者有提供 Pipeline 或定義名稱時，以名稱篩選找出定義 ID：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/build/definitions?name={name}&api-version=7.1"
  ```

  取 `value[].id` 與 `value[].name`。`name` 可用 `*` 萬用字元（例如 `CI*`）；找到多個時請使用者選一個。
- 使用者沒有指定 Pipeline，而問題需要它（例如「列出近期建置」）時，請使用者提供一次。
- 仍未提供時，不帶 `name` 呼叫上述 API，列出所有定義讓使用者選擇。

## 連線與驗證

一律用 `curl` 直接呼叫 Azure DevOps REST API，每個指令都自帶驗證參數。

- **組織**：讀取環境變數 `AZURE_DEVOPS_ORG`（組織名稱，例如 `contoso`）。未設定時詢問使用者一次；使用者給的是網址時，從 `https://dev.azure.com/{org}` 或 `https://{org}.visualstudio.com` 取出組織名稱。
- **權杖**：PAT 以 Basic 驗證送出：`-u ":$AZURE_DEVOPS_PAT"`；只設定了 `AZURE_DEVOPS_EXT_PAT`（Azure CLI azure-devops 擴充套件使用的變數）時，改用它。改用 Microsoft Entra ID 權杖時，換成 `-H "Authorization: Bearer $AZURE_DEVOPS_TOKEN"`。都未設定時，請使用者在自己的終端機設定 `AZURE_DEVOPS_PAT` 後重試，並停止。權杖只從環境變數讀取：不要請使用者把權杖貼進對話，也不要印出或寫入檔案。
- **PAT 權限範圍**：Build (Read)、Project and Team (Read)。
- **編碼**：路徑與查詢字串中的專案、Pipeline 名稱要 URL 編碼（空白寫成 `%20`）；查詢字串中的 `$` 寫成 `%24`（例如 `%24top`），避免被 shell 展開。
- **錯誤**：HTTP 401、403、203 或回應是 HTML 登入頁，代表權杖無效、過期或權限不足，回報後停止；HTTP 404 代表組織、專案或建置 ID 有誤。
- **Windows PowerShell**：呼叫 `curl.exe`（而非 `curl` 別名），環境變數寫成 `$env:AZURE_DEVOPS_PAT`；PowerShell 不支援 `<<'EOF'`，先把 JSON 主體寫入暫存檔，再以 `--data-binary "@body.json"` 送出。

## 規則

### 1. 列出某個 Pipeline 的近期建置

- 使用者要**列出建置**或**顯示近期建置**時，以定義 ID 篩選：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/build/builds?definitions={definitionId}&%24top=20&queryOrder=queueTimeDescending&api-version=7.1"
  ```

- 使用者有指定條件時加上篩選參數（例如「main 上失敗的建置」）：
  - 分支：`branchName=refs/heads/main`
  - 狀態：`statusFilter=completed`（`inProgress`、`notStarted`、`cancelling`…）
  - 結果：`resultFilter=failed`（`succeeded`、`partiallySucceeded`、`canceled`）
- 使用者沒有要求時，不要抓記錄檔或變更。
- 以表格呈現結果。
- 沒有任何建置時，明確說明這個 Pipeline 沒有建置。

範例：「列出 Contoso 專案 CI-Main pipeline 的近期建置」

### 2. 依 ID 取得建置狀態

- 使用者問起**特定建置**（例如「build 12345」）時：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/build/builds/{buildId}?api-version=7.1"
  ```

- 顯示建置編號、狀態、結果、來源分支、開始與結束時間、要求者與定義名稱。
- 建置失敗時，建議使用者查看記錄檔。

範例：「Contoso 專案的 build 12345 狀態如何？」

### 3. 檢視失敗建置的記錄檔

- 使用者要**看記錄檔**或問**建置為何失敗**時，先取得建置的 Timeline，找出失敗的步驟：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/build/builds/{buildId}/timeline?api-version=7.1"
  ```

  在 `records[]` 中找 `result` 為 `failed` 的紀錄，優先看 `type` 為 `Task` 的步驟；`issues[]` 已含錯誤摘要，`log.id` 是該步驟的記錄檔 ID。
- 對每個失敗步驟取得記錄檔內容（純文字），只保留最後一段：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/build/builds/{buildId}/logs/{logId}?api-version=7.1" | tail -n 200
  ```

  需要特定範圍時改用 `&startLine={n}&endLine={m}`。
- 以程式碼區塊呈現記錄檔內容，方便閱讀。
- 聚焦在失敗步驟與各記錄檔的最後一段；其餘步驟的記錄檔保持不取。

範例：「顯示 Contoso 專案 build 12345 的記錄檔」、「build 12345 為什麼失敗？」

### 4. 列出建置的變更

- 使用者要**看變更內容**或**列出 commit** 時：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/build/builds/{buildId}/changes?%24top=50&api-version=7.1"
  ```

- 以表格呈現 commit ID、作者與訊息。
- 沒有任何變更時，明確說明這個建置沒有關聯的變更。

範例：「build 12345 包含哪些變更？」、「列出 Contoso 專案 build 12345 的 commit」

## 呈現結果

時間以使用者的當地時區顯示，無法判斷時區時標明 UTC。

建置清單以表格呈現下列欄位：

- **建置 ID**：可點擊的超連結，格式為 `[{buildId}](https://dev.azure.com/{organization}/{project}/_build/results?buildId={buildId})`
- **定義名稱**（`definition.name`）
- **狀態**（`status`：completed、inProgress、cancelling、notStarted）
- **結果**（`result`），加上 emoji：✅ succeeded、❌ failed、⚠️ partiallySucceeded、🚫 canceled
- **來源分支**（`sourceBranch`）
- **開始時間**（`startTime`，格式 `YYYY/MM/DD HH:MM`）
- **要求者**（`requestedFor.displayName`，只顯示名稱）

單一建置的狀態，除上述欄位外再加上：

- **結束時間**（`finishTime`，格式 `YYYY/MM/DD HH:MM`）
- **建置編號**（`buildNumber`）
- **耗時**（由開始與結束時間計算）

變更清單以表格呈現：

- **Commit ID**（`id` 的前 8 碼）
- **作者**（`author.displayName`）
- **訊息**（`message` 的第一行）
- **時間**（`timestamp`）
