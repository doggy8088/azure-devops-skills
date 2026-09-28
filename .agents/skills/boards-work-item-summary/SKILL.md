---
name: boards-work-item-summary
description: 以 Azure DevOps REST API 摘要單一工作項目（work item）：欄位、父子與相關連結、連結的 Pull Request 與留言。當使用者提供工作項目 ID，想了解它的內容、狀態、關聯或討論時使用。
compatibility: 需要 curl、可連線至 dev.azure.com 的網路，以及 AZURE_DEVOPS_ORG 與 AZURE_DEVOPS_PAT 環境變數。
---

# 工作項目摘要

這個 Skill 一律針對**一個特定的工作項目 ID**。工作項目 ID 在整個組織內唯一，因此不需要先選專案。

- 使用者有提供 ID（例如「115770」或「work item 115770」）就直接使用。
- 沒有提供時，請使用者提供一次。
- 提示後仍沒有 ID，就回覆「需要工作項目 ID」並停止，不呼叫任何 API。

## 連線與驗證

一律用 `curl` 直接呼叫 Azure DevOps REST API，每個指令都自帶驗證參數。

- **組織**：讀取環境變數 `AZURE_DEVOPS_ORG`（組織名稱，例如 `contoso`）。未設定時詢問使用者一次；使用者給的是網址時，從 `https://dev.azure.com/{org}` 或 `https://{org}.visualstudio.com` 取出組織名稱。
- **權杖**：PAT 以 Basic 驗證送出：`-u ":$AZURE_DEVOPS_PAT"`；只設定了 `AZURE_DEVOPS_EXT_PAT`（Azure CLI azure-devops 擴充套件使用的變數）時，改用它。改用 Microsoft Entra ID 權杖時，換成 `-H "Authorization: Bearer $AZURE_DEVOPS_TOKEN"`。都未設定時，請使用者在自己的終端機設定 `AZURE_DEVOPS_PAT` 後重試，並停止。權杖只從環境變數讀取：不要請使用者把權杖貼進對話，也不要印出或寫入檔案。
- **PAT 權限範圍**：Work Items (Read)；要讀取 Pull Request 細節另需 Code (Read)。
- **編碼**：路徑中的專案名稱要 URL 編碼（空白寫成 `%20`）；查詢字串中的 `$` 寫成 `%24`（例如 `%24expand`），避免被 shell 展開。
- **錯誤**：HTTP 401、403、203 或回應是 HTML 登入頁，代表權杖無效、過期或權限不足，回報後停止；HTTP 404 代表組織名稱或 ID 有誤。
- **Windows PowerShell**：呼叫 `curl.exe`（而非 `curl` 別名），環境變數寫成 `$env:AZURE_DEVOPS_PAT`；PowerShell 不支援 `<<'EOF'`，先把 JSON 主體寫入暫存檔，再以 `--data-binary "@body.json"` 送出。

## 步驟

1. 一次取回工作項目的欄位、關聯與連結：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/wit/workitems/{id}?%24expand=all&api-version=7.1"
   ```

   記下 `fields["System.TeamProject"]`，後續組網址與呼叫留言 API 時使用。

2. 從 `relations` 陣列收集所有連結的工作項目 ID：`url` 形如 `.../_apis/wit/workItems/{linkedId}` 的關聯，取網址最後一段就是 ID；`attributes.name` 是連結類型的顯示名稱（Parent、Child、Related…）。父項目（`System.LinkTypes.Hierarchy-Reverse`）也在其中，不需要另外呼叫。

3. 以一次批次呼叫取得所有連結項目的細節（每次最多 200 個 ID，超過就分批）：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -X POST -H "Content-Type: application/json" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/wit/workitemsbatch?api-version=7.1" \
     -d '{"ids":[101,102],"fields":["System.Id","System.Title","System.State","System.WorkItemType","System.TeamProject"],"errorPolicy":"Omit"}'
   ```

   `errorPolicy: Omit` 讓已刪除或無權限的項目以 `null` 回傳，整批呼叫不會因此失敗。

4. 找出連結的 Pull Request：`relations` 中 `rel` 為 `ArtifactLink` 且 `attributes.name` 為 `Pull Request` 的項目，`url` 形如 `vstfs:///Git/PullRequestId/{projectId}%2F{repositoryId}%2F{pullRequestId}`（分隔符號可能是 `%2F` 或 `%2f`，解析時不分大小寫）。其他 `ArtifactLink`（Commit、Build、Release）不列入摘要。對每個 PR 呼叫：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{projectId}/_apis/git/repositories/{repositoryId}/pullrequests/{pullRequestId}?api-version=7.1"
   ```

   取 `title`、`status`（`active`、`completed`、`abandoned`）與 `repository.webUrl`。權杖缺少 Code (Read) 而取不到時，仍列出 PR ID，並註明無法取得細節。

5. 一次取得原始工作項目的留言：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wit/workItems/{id}/comments?api-version=7.1-preview.4"
   ```

   `totalCount` 是留言總數，`comments[].text` 是 HTML。回應帶有 `continuationToken` 時，代表還有更多留言；需要時加上 `&continuationToken={token}` 再取一次。

## 呈現結果

- **主項目欄位**：ID、標題、狀態、指派對象（只顯示 `displayName`，不顯示 email）、工作項目類型、建立日期（`YYYY/MM/DD`）、優先順序（`Microsoft.VSTS.Common.Priority`）、標籤（`System.Tags`）。

- **連結**表格，每個連結的工作項目一列：
  - **ID**：可點擊的超連結，格式固定為 `[{ID}](https://dev.azure.com/{organization}/{project}/_workitems/edit/{ID})`，`{project}` 取該項目的 `System.TeamProject`（URL 編碼）
  - **連結類型**（Parent、Child、Related，或原始關聯名稱）
  - **工作項目類型**
  - **標題**
  - **狀態**

  依序排列：**Parent** → **Child** → **Related** → 其他連結類型。

- 有連結的 Pull Request 時，另列 **Pull Requests** 表格，至少包含 PR ID、標題、狀態，以及可點擊的連結 `{repository.webUrl}/pullrequest/{pullRequestId}`。

- 用簡短白話**摘要**工作項目的描述（`System.Description`）、系統資訊（`Microsoft.VSTS.TCM.SystemInfo`）與重現步驟（`Microsoft.VSTS.TCM.ReproSteps`），只摘要實際存在的欄位。這些欄位是 HTML，閱讀時略過標籤。聚焦在核心意圖、範圍，以及關鍵的技術或重現細節。

- **留言**：
  - 有留言時，說明**留言總數**，並簡述主要議題或已做出的決定。
  - 沒有留言時，明確說明這個工作項目沒有留言。
