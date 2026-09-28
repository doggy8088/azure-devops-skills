---
name: boards-query
description: 以 Azure DevOps REST API 執行 WIQL 查詢或已儲存的查詢（saved query），並依查詢的欄位呈現結果：臨時撰寫 WIQL（含 @Me、@Today、@CurrentIteration 等巨集）、列出或搜尋「共用查詢」與「我的查詢」、以名稱、路徑或 ID 執行已儲存的查詢，以及父子樹狀（tree）與直接連結（one-hop）查詢。當使用者要用 WIQL 或自訂條件篩選工作項目，或要執行某個已儲存的查詢時使用；只要看指派給自己、尚未完成的工作改用 `boards-my-work`。
compatibility: 需要 curl、可連線至 dev.azure.com 的網路，以及 AZURE_DEVOPS_ORG 與 AZURE_DEVOPS_PAT 環境變數。
---

# 工作項目查詢（WIQL 與已儲存的查詢）

這個 Skill 只讀取：執行使用者給的條件或已儲存的查詢，再以查詢本身的欄位呈現結果。固定格式的「我的工作」報表改用 `boards-my-work`；要修改查到的工作項目改用 `boards-work-item-write`。

## 專案選擇

- 使用者有提供專案名稱（例如「Contoso 專案」）就直接使用，不呼叫專案清單 API。
- 沒有提供時，請使用者提供一次。
- 仍未提供時，列出專案讓使用者選擇：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/projects?%24top=500&api-version=7.1"
  ```

- 使用者提供或選定專案之前不要繼續。

**團隊**是選填的：只有查詢用到 `@CurrentIteration`，且使用者指的不是專案的預設團隊時，才把團隊放進 URL（`{project}/{team}/_apis/...`）。使用者沒提到團隊時就省略，這時 `@CurrentIteration` 以專案的預設團隊解析。

## 連線與驗證

一律用 `curl` 直接呼叫 Azure DevOps REST API，每個指令都自帶驗證參數。

- **組織**：讀取環境變數 `AZURE_DEVOPS_ORG`（組織名稱，例如 `contoso`）。未設定時詢問使用者一次；使用者給的是網址時，從 `https://dev.azure.com/{org}` 或 `https://{org}.visualstudio.com` 取出組織名稱。
- **權杖**：PAT 以 Basic 驗證送出：`-u ":$AZURE_DEVOPS_PAT"`；只設定了 `AZURE_DEVOPS_EXT_PAT`（Azure CLI azure-devops 擴充套件使用的變數）時，改用它。改用 Microsoft Entra ID 權杖時，換成 `-H "Authorization: Bearer $AZURE_DEVOPS_TOKEN"`。都未設定時，請使用者在自己的終端機設定 `AZURE_DEVOPS_PAT` 後重試，並停止。權杖只從環境變數讀取：不要請使用者把權杖貼進對話，也不要印出或寫入檔案。
- **PAT 權限範圍**：Work Items (Read)、Project and Team (Read)。
- **編碼**：路徑中的專案、團隊名稱與查詢路徑（含中文的資料夾與查詢名稱），以及查詢搜尋的關鍵字，都要 URL 編碼（空白寫成 `%20`）；查詢字串中的 `$` 寫成 `%24`（例如 `%24top`），避免被 shell 展開。
- **錯誤**：HTTP 401、403、203 或回應是 HTML 登入頁，代表權杖無效、過期或權限不足，回報後停止；HTTP 404 代表組織、專案、團隊或查詢路徑有誤。HTTP 400 的 `message` 會指出 WIQL 的問題（例如 `TF51005` 欄位不存在、`VS403721` 團隊沒有排定目前的 Iteration），把原文給使用者看並協助修正。
- **Windows PowerShell**：呼叫 `curl.exe`（而非 `curl` 別名），環境變數寫成 `$env:AZURE_DEVOPS_PAT`；PowerShell 不支援 `<<'EOF'`，先把 JSON 主體寫入暫存檔，再以 `--data-binary "@body.json"` 送出。

## WIQL 速查

| 需求 | 寫法 |
|---|---|
| 目前使用者 | `[System.AssignedTo] = @Me` |
| URL 中的專案 | `[System.TeamProject] = @project` |
| 相對日期 | `[System.ChangedDate] >= @Today - 7`；另有 `@StartOfWeek`、`@StartOfMonth` |
| 團隊目前、上一個 Sprint | `[System.IterationPath] = @CurrentIteration`、`@CurrentIteration - 1`（團隊由 URL 決定） |
| 團隊的區域 | `[System.AreaPath] IN (@TeamAreas('[Contoso]\\Contoso Team'))` |
| 階層以下 | `[System.AreaPath] UNDER 'Contoso\\Web'` |
| 需求層級類型 | `[System.WorkItemType] IN GROUP 'Microsoft.RequirementCategory'` |
| 文字 | `[System.Title] CONTAINS WORDS 'login'`；標籤 `[System.Tags] CONTAINS 'api'` |

- **連結查詢**用 `FROM WorkItemLinks`，條件分成 `[Source].`、`[Target].` 與 `[System.Links.LinkType]`，最後加模式：
  - `MODE (Recursive)`：樹狀（`queryType` 為 `tree`），只能搭配 `System.LinkTypes.Hierarchy-Forward`。
  - `MODE (MustContain)`：直接連結（`queryType` 為 `oneHop`），只留下有符合條件之目標的來源。
- WIQL 裡的字串用單引號，JSON 裡的反斜線寫成 `\\`；主體一律用 heredoc。
- 查詢結果上限 20,000 筆，超過會被**靜默截斷**、不回錯誤；WIQL 本身上限 32,000 字元。筆數剛好是上限時，提醒使用者結果可能不完整並建議加條件。

## 規則

### 1. 執行臨時 WIQL

1. 把使用者的需求寫成 WIQL，`SELECT` 列出要顯示的欄位，並加 `ORDER BY`。執行前把 WIQL 給使用者看。
2. 執行（有團隊時 URL 改成 `{project}/{team}/_apis/wit/wiql`）：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -X POST -H "Content-Type: application/json" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wit/wiql?%24top=200&api-version=7.1" \
     --data-binary @- <<'EOF'
   {"query": "SELECT [System.Id], [System.Title], [System.State], [System.AssignedTo] FROM WorkItems WHERE [System.TeamProject] = @project AND [System.WorkItemType] = 'Bug' AND [System.ChangedDate] >= @Today - 7 ORDER BY [System.ChangedDate] DESC"}
   EOF
   ```

   `%24top=200` 先取前 200 筆；使用者要全部時拿掉它（仍受 20,000 筆上限限制）。連結查詢的 `%24top` 計算的是 `workItemRelations` 的列數，會把樹切斷，除非結果很大否則不要加。
3. 依下方「取得欄位值」補齊資料並呈現。

範例：「查 Contoso 專案最近 7 天變更過的 Bug」、「列出 Feature 101 底下所有子項目的樹」

連結查詢範例（每個 Feature 與它所有層級的子項目）：

```json
{"query": "SELECT [System.Id], [System.Title], [System.State] FROM WorkItemLinks WHERE ([Source].[System.TeamProject] = @project AND [Source].[System.WorkItemType] = 'Feature') AND ([System.Links.LinkType] = 'System.LinkTypes.Hierarchy-Forward') AND ([Target].[System.TeamProject] = @project) MODE (Recursive)"}
```

### 2. 找出並執行已儲存的查詢

1. 使用者給的是完整路徑（例如「Shared Queries/Sprint/進行中的 Bug」）時直接取用；路徑每一段都要 URL 編碼：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wit/queries/Shared%20Queries/{folder}/{name}?%24expand=wiql&api-version=7.1"
   ```

   回應有 `id`、`queryType`（`flat`、`tree`、`oneHop`）與 `wiql`。根資料夾是 `Shared Queries`（共用查詢）與 `My Queries`（我的查詢，只看得到權杖擁有者自己的）。
2. 只給名稱或關鍵字時，以名稱搜尋：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wit/queries?%24filter={keyword}&%24expand=minimal&%24top=50&api-version=7.1"
   ```

   找到多個時列出 `path` 讓使用者選。
3. 要瀏覽清單時取得查詢樹（`%24depth` 最大 2；`hasChildren` 為 `true` 但沒有 `children` 的資料夾，再以該資料夾路徑加 `?%24depth=2` 展開）：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wit/queries?%24depth=2&%24expand=minimal&api-version=7.1"
   ```

   `isFolder` 為 `true` 的是資料夾，其餘是查詢。
4. 以 ID 執行查詢（查詢用到 `@CurrentIteration` 且使用者指定團隊時，URL 加上 `{team}`）：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wit/wiql/{queryId}?api-version=7.1"
   ```

   也可以加 `&%24top=200`。回應格式與規則 1 相同。
5. 依下方「取得欄位值」補齊資料並呈現；標題註明查詢的 `path`。

範例：「執行共用查詢『進行中的 Bug』」、「列出 Contoso 專案有哪些共用查詢」

## 取得欄位值

WIQL 只回傳 ID，欄位值要另外取：

1. 收集 ID：`queryResultType` 為 `workItem` 時取 `workItems[].id`；為 `workItemLink` 時取 `workItemRelations[]` 的 `source.id` 與 `target.id`（去除重複）。沒有任何 ID 時，說明沒有符合條件的工作項目，然後停止。
2. 欄位用回應 `columns[].referenceName`（就是查詢的 `SELECT` 欄位），每次最多 200 個 ID、超過就分批：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -X POST -H "Content-Type: application/json" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wit/workitemsbatch?api-version=7.1" \
     -d '{"ids":[101,102],"fields":["System.Id","System.Title","System.State","System.AssignedTo"],"errorPolicy":"Omit"}'
   ```

   值為空的欄位不會出現在 `fields` 中，顯示為空白。身分欄位（例如 `System.AssignedTo`）是物件，顯示 `displayName`。
3. 依 WIQL 回傳的順序排列（那是 `ORDER BY` 的結果）；批次回應會依 ID 排序，不保留要求的順序。

## 呈現結果

- 先顯示執行的 WIQL 或查詢路徑，以及筆數（有 `%24top` 或達 20,000 筆上限時註明可能還有更多）。
- **一般查詢**：以表格呈現，欄位與順序照 `columns[]`（表頭用 `columns[].name`）。ID 是可點擊的連結 `[{ID}](https://dev.azure.com/{organization}/{project}/_workitems/edit/{ID})`（`{project}` 需 URL 編碼）；日期顯示 `YYYY/MM/DD`，時間轉成使用者當地時區，無法判斷時標明 UTC。
- **樹狀查詢**：`source` 為 `null` 的列是根節點，其後 `source.id` 指向它的列是子項目；依 `workItemRelations[]` 的順序輸出，在標題欄前以縮排或 `└─` 表示層級。
- **直接連結查詢**：每個來源一列，下方縮排列出符合條件的目標，註明連結類型（`rel`）。
