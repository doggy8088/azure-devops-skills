---
name: boards-my-work
description: 以 Azure DevOps REST API 列出目前使用者在某個專案中被指派、尚未完成的工作項目，依類型分組、依最近變更排序，並附可點擊連結。當使用者想看「我的工作」、指派給自己的工作項目或手上的待辦時使用。
compatibility: 需要 curl、可連線至 dev.azure.com 的網路，以及 AZURE_DEVOPS_ORG 與 AZURE_DEVOPS_PAT 環境變數。
---

# 我的工作項目

## 專案選擇

- 使用者有提供專案名稱（例如「Contoso 專案」）就直接使用，不呼叫專案清單 API。
- 沒有提供時，請使用者提供一次。
- 仍未提供時，列出專案讓使用者選擇：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/projects?%24top=500&api-version=7.1"
  ```

  顯示 `value[].name`。回應標頭（`curl -D -` 可看到）有 `x-ms-continuationtoken` 時代表還有更多專案，加上 `&continuationToken={token}` 取下一頁。
- 使用者提供或選定專案之前不要繼續。

## 連線與驗證

一律用 `curl` 直接呼叫 Azure DevOps REST API，每個指令都自帶驗證參數。

- **組織**：讀取環境變數 `AZURE_DEVOPS_ORG`（組織名稱，例如 `contoso`）。未設定時詢問使用者一次；使用者給的是網址時，從 `https://dev.azure.com/{org}` 或 `https://{org}.visualstudio.com` 取出組織名稱。
- **權杖**：PAT 以 Basic 驗證送出：`-u ":$AZURE_DEVOPS_PAT"`；只設定了 `AZURE_DEVOPS_EXT_PAT`（Azure CLI azure-devops 擴充套件使用的變數）時，改用它。改用 Microsoft Entra ID 權杖時，換成 `-H "Authorization: Bearer $AZURE_DEVOPS_TOKEN"`。都未設定時，請使用者在自己的終端機設定 `AZURE_DEVOPS_PAT` 後重試，並停止。權杖只從環境變數讀取：不要請使用者把權杖貼進對話，也不要印出或寫入檔案。
- **PAT 權限範圍**：Work Items (Read)、Project and Team (Read)。
- **編碼**：路徑中的專案名稱要 URL 編碼（空白寫成 `%20`）；查詢字串中的 `$` 寫成 `%24`（例如 `%24top`），避免被 shell 展開。
- **錯誤**：HTTP 401、203 或回應是 HTML 登入頁，代表權杖無效、過期或權限不足，回報後停止；HTTP 404 代表組織或專案名稱有誤。
- **Windows PowerShell**：呼叫 `curl.exe`（而非 `curl` 別名），環境變數寫成 `$env:AZURE_DEVOPS_PAT`。

## 步驟

1. 依上方規則決定專案。

2. 用 WIQL 查出指派給目前使用者、尚未完成的工作項目。`@Me` 會解析為權杖所屬的使用者，`@project` 會解析為網址中的專案：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -X POST -H "Content-Type: application/json" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wit/wiql?%24top=200&api-version=7.1" \
     --data-binary @- <<'EOF'
   {"query": "SELECT [System.Id] FROM WorkItems WHERE [System.TeamProject] = @project AND [System.AssignedTo] = @Me AND [System.State] NOT IN ('Closed', 'Done', 'Removed') ORDER BY [System.ChangedDate] DESC"}
   EOF
   ```

   排除的狀態涵蓋 Agile、Scrum、CMMI 與 Basic 內建流程的完成與移除狀態。使用者要求包含已完成的項目時，拿掉 `[System.State]` 條件；專案使用自訂流程且有其他完成狀態時，把它們加進 `NOT IN`。

3. 取出回應 `workItems[].id`。沒有任何項目時，說明這個專案中沒有指派給使用者的進行中工作項目，然後停止。

4. 以批次呼叫取得細節（每次最多 200 個 ID，超過就分批）。使用者沒有指定欄位時，使用下方預設欄位：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -X POST -H "Content-Type: application/json" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wit/workitemsbatch?api-version=7.1" \
     -d '{"ids":[101,102],"fields":["System.Id","System.Title","System.State","System.AssignedTo","System.WorkItemType","System.CreatedDate","System.ChangedDate","System.Tags","Microsoft.VSTS.Common.Priority"],"errorPolicy":"Omit"}'
   ```

5. 依工作項目類型（`System.WorkItemType`）分組，組內依 `System.ChangedDate` 由新到舊排序。

6. 依下方格式呈現。

## 呈現結果

每個類型一個標題與一張表格，欄位為：

- **ID**：可點擊的超連結，格式為 `[{ID}](https://dev.azure.com/{organization}/{project}/_workitems/edit/{ID})`（`{project}` 需 URL 編碼）
- **標題**
- **狀態**
- **優先順序**

分組依以下順序，沒有項目的分組直接略過：

1. Epic
2. Feature
3. 需求類（User Story、Product Backlog Item、Requirement、Issue）
4. Bug
5. Task
6. Test Case
7. 其他工作項目類型
