---
name: repos-pull-request-review
description: 以 Azure DevOps REST API 審查 Pull Request（PR）：列出待我審查或我建立的 PR，摘要單一 PR 的變更檔案與 diff、討論串、審查者投票、連結工作項目與被阻擋的原因（branch policy），由 commit 反查 PR，並在確認後留言、回覆、解決討論或投票。當使用者要 review PR、查看 PR 狀態或回應 PR 討論時使用；要建立或修改自己的 PR 改用 `repos-pull-request-write`。核准 Pipeline 部署改用 `pipelines-approvals`。
compatibility: 需要 curl、git、可連線至 dev.azure.com 與 vssps.dev.azure.com 的網路，以及 AZURE_DEVOPS_ORG 與 AZURE_DEVOPS_PAT 環境變數。
---

# Pull Request 審查

這個 Skill 站在**審查者**的角度處理 PR：找出要看的 PR、讀懂它、回應它。建立 PR、發布 draft、設定 auto-complete 等作者端操作改用 `repos-pull-request-write`。

## 專案選擇

- 使用者有提供專案名稱（例如「Contoso 專案」）就直接使用，不呼叫專案清單 API。
- 沒有提供時，請使用者提供一次。
- 仍未提供時，列出專案讓使用者選擇：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/projects?%24top=500&api-version=7.1"
  ```

- 使用者要「所有專案」時，對清單中的每個專案重複規則 1；專案超過 20 個時先告知數量並確認一次。
- 使用者提供或選定專案之前不要繼續。

儲存庫是選填的：列出 PR 時省略代表整個專案；使用者指定儲存庫時，改用 `_apis/git/repositories/{repository}/pullrequests`。

## 連線與驗證

一律用 `curl` 直接呼叫 Azure DevOps REST API，每個指令都自帶驗證參數。

- **組織**：讀取環境變數 `AZURE_DEVOPS_ORG`（組織名稱，例如 `contoso`）。未設定時詢問使用者一次；使用者給的是網址時，從 `https://dev.azure.com/{org}` 或 `https://{org}.visualstudio.com` 取出組織名稱。
- **主機**：除了 `dev.azure.com`，查詢群組成員的 Identities API 主機是 `vssps.dev.azure.com`。
- **權杖**：PAT 以 Basic 驗證送出：`-u ":$AZURE_DEVOPS_PAT"`；只設定了 `AZURE_DEVOPS_EXT_PAT`（Azure CLI azure-devops 擴充套件使用的變數）時，改用它。改用 Microsoft Entra ID 權杖時，換成 `-H "Authorization: Bearer $AZURE_DEVOPS_TOKEN"`。都未設定時，請使用者在自己的終端機設定 `AZURE_DEVOPS_PAT` 後重試，並停止。權杖只從環境變數讀取：不要請使用者把權杖貼進對話，也不要印出或寫入檔案。
- **PAT 權限範圍**：Code (Read)、Work Items (Read)、Identity (Read)；留言、回覆、解決討論與投票另需 Code (Read & write)。
- **編碼**：路徑中的專案與儲存庫名稱、查詢字串中的檔案路徑要 URL 編碼（空白寫成 `%20`）；查詢字串中的 `$` 寫成 `%24`（例如 `%24top`），避免被 shell 展開。
- **錯誤**：HTTP 401、403、203 或回應是 HTML 登入頁，代表權杖無效、過期或權限不足，回報後停止；HTTP 404 代表組織、專案、儲存庫或 PR ID 有誤。
- **Windows PowerShell**：呼叫 `curl.exe`（而非 `curl` 別名），環境變數寫成 `$env:AZURE_DEVOPS_PAT`；PowerShell 不支援 `<<'EOF'`，先把 JSON 主體寫入暫存檔，再以 `--data-binary "@body.json"` 送出。

## 目前使用者

規則 1 與規則 4 需要目前使用者的身分 ID（這個端點不帶 `api-version`）：

```bash
curl -sS -u ":$AZURE_DEVOPS_PAT" "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/connectionData"
```

使用 `authenticatedUser.id`；`authenticatedUser.providerDisplayName` 是顯示名稱。

## 規則

### 1. 列出待我審查或我建立的 PR

- **待我審查**：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/pullrequests?searchCriteria.reviewerId={myId}&searchCriteria.status=active&%24top=100&api-version=7.1"
  ```

  `reviewerId` 只找得到**直接**指派給我的 PR。要涵蓋指派給我所屬團隊或群組的 PR：
  1. 不帶 `reviewerId` 列出 `searchCriteria.status=active` 的 PR，挑出 `reviewers[]` 中 `isContainer` 為 `true` 的審查者。
  2. 取得我的直接群組成員資格（`memberOf` 是群組描述子清單）：

     ```bash
     curl -sS -u ":$AZURE_DEVOPS_PAT" \
       "https://vssps.dev.azure.com/$AZURE_DEVOPS_ORG/_apis/identities?identityIds={myId}&queryMembership=Direct&api-version=7.1"
     ```

  3. 以 `identityIds={groupId1},{groupId2}` 一次查出這些群組審查者的 `descriptor`，出現在我的 `memberOf` 中的就算「透過 {群組名稱} 待我審查」。
- **我建立的**：把 `reviewerId` 換成 `searchCriteria.creatorId={myId}`。
- 其他篩選：`searchCriteria.targetRefName=refs/heads/main`、`searchCriteria.status=completed|abandoned|all`。
- 分頁用 `%24top` 與 `%24skip`；回傳筆數等於 `%24top` 時再取下一頁。
- 清單 API 的 `description` 會被截斷成 400 字；需要完整描述時用規則 2 取單一 PR。
- 沒有任何 PR 時，明確說明沒有符合條件的 PR。

範例：「列出 Contoso 專案待我審查的 PR」、「我在 Contoso 開了哪些 PR 還沒合併？」

### 2. 審查單一 PR

1. 取得 PR（不需要知道儲存庫）：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/pullrequests/{pullRequestId}?api-version=7.1"
   ```

   記下 `repository.id`、`repository.project.id`（專案 GUID）、`repository.webUrl`、`isDraft`、`mergeStatus`、`reviewers[].vote`、`autoCompleteSetBy`。

2. 取得 iterations，使用最後一筆的 `id`、`sourceRefCommit.commitId` 與 `commonRefCommit.commitId`（merge base）：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories/{repositoryId}/pullRequests/{pullRequestId}/iterations?api-version=7.1"
   ```

3. 取得整個 PR 相對於 merge base 的變更檔案（`%24compareTo=0`）：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories/{repositoryId}/pullRequests/{pullRequestId}/iterations/{iterationId}/changes?%24compareTo=0&%24top=2000&api-version=7.1"
   ```

   `changeEntries[]` 有 `item.path`、`changeType`（add、edit、delete、rename）與 `changeTrackingId`（行內留言會用到）。回應有 `nextSkip` 時用 `%24skip={nextSkip}` 取下一頁。

4. 產生 diff：伺服器不提供 diff，要取兩個版本的檔案內容後在本機比較。檔案超過 20 個時，先列出清單，請使用者挑選或依風險挑重點檔案。

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -o base.txt \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories/{repositoryId}/items?path={filePath}&versionDescriptor.version={commonRefCommit}&versionDescriptor.versionType=commit&api-version=7.1"
   curl -sS -u ":$AZURE_DEVOPS_PAT" -o source.txt \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories/{repositoryId}/items?path={filePath}&versionDescriptor.version={sourceRefCommit}&versionDescriptor.versionType=commit&api-version=7.1"
   git diff --no-index --unified=3 base.txt source.txt
   ```

   不帶 `%24format` 時回傳純文字內容。新增的檔案在 base 會回 404（`TF401174`），刪除的檔案在 source 會回 404：把該側視為空檔。二進位檔只標示「二進位檔已變更」。暫存檔放在暫存目錄，用完即刪。

5. 取得討論串，只保留 `comments[0].commentType` 不是 `system`、且 `isDeleted` 不為 `true` 的串：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories/{repositoryId}/pullRequests/{pullRequestId}/threads?api-version=7.1"
   ```

   `status` 為 `active` 或 `pending` 的是未解決；`threadContext.filePath` 與 `threadContext.rightFileStart.line` 是行內留言的位置。

6. 取得連結的工作項目（回傳 `value[].id`），再以 `POST _apis/wit/workitemsbatch` 取 `System.Title`、`System.State`、`System.WorkItemType`：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories/{repositoryId}/pullRequests/{pullRequestId}/workitems?api-version=7.1"
   ```

7. 取得 branch policy 評估結果。`artifactId` 必須用**專案 GUID**（步驟 1 的 `repository.project.id`），不能用專案名稱：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/policy/evaluations?artifactId=vstfs:///CodeReview/CodeReviewId/{projectGuid}/{pullRequestId}&api-version=7.1-preview.1"
   ```

   這個資源只有 preview 版本。`configuration.isBlocking` 為 `true` 且 `status` 不是 `approved` 或 `notApplicable` 的，就是 PR 被擋住的原因；`configuration.type.displayName` 是原則名稱。

範例：「幫我 review Contoso 專案的 PR 1234」、「PR 1234 為什麼還不能合併？」

### 3. 由 commit 反查 PR

```bash
curl -sS -u ":$AZURE_DEVOPS_PAT" -X POST -H "Content-Type: application/json" \
  "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories/{repository}/pullrequestquery?api-version=7.1" \
  -d '{"queries":[{"type":"commit","items":["{commitSha}"]}]}'
```

這是唯讀查詢（成功時回 HTTP 201）。`type: commit` 找出包含該 commit 的 PR；`type: lastMergeCommit` 找出以該 commit 合併的 PR。`results[0]` 是 `{ "<sha>": [PR, ...] }` 的對照表。commit SHA 必須是完整 40 碼。

範例：「commit 1a2b3c4 是哪個 PR 帶進來的？」

### 4. 留言、回覆、解決討論、投票（寫入）

每個寫入動作之前，把即將送出的內容（PR、檔案與行號、留言全文、投票值或狀態）列給使用者，確認一次後才執行。自己產生的審查意見先呈現給使用者，由使用者決定要不要貼上。

- **新增一般留言**：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" -X POST -H "Content-Type: application/json" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories/{repositoryId}/pullRequests/{pullRequestId}/threads?api-version=7.1" \
    -d '{"comments":[{"parentCommentId":0,"content":"建議補上單元測試。","commentType":1}],"status":1}'
  ```

- **新增行內留言**：在上述主體加上位置資訊。`changeTrackingId` 取自規則 2 步驟 3，`secondComparingIteration` 是最後一個 iteration 的 `id`：

  ```json
  "threadContext": {"filePath": "/src/app.cs", "rightFileStart": {"line": 42, "offset": 1}, "rightFileEnd": {"line": 42, "offset": 1}},
  "pullRequestThreadContext": {"changeTrackingId": 3, "iterationContext": {"firstComparingIteration": 1, "secondComparingIteration": 2}}
  ```

- **回覆討論串**：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" -X POST -H "Content-Type: application/json" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories/{repositoryId}/pullRequests/{pullRequestId}/threads/{threadId}/comments?api-version=7.1" \
    -d '{"parentCommentId":1,"content":"已修正，謝謝。","commentType":1}'
  ```

- **變更討論串狀態**（例如標為已解決）：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" -X PATCH -H "Content-Type: application/json" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories/{repositoryId}/pullRequests/{pullRequestId}/threads/{threadId}?api-version=7.1" \
    -d '{"status":2}'
  ```

  狀態值：`1` active、`2` fixed（已解決）、`3` wontFix、`4` closed、`5` byDesign、`6` pending。

- **投票**（`{myId}` 取自「目前使用者」；我還不是審查者時會自動加入）：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" -X PUT -H "Content-Type: application/json" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories/{repositoryId}/pullRequests/{pullRequestId}/reviewers/{myId}?api-version=7.1" \
    -d '{"vote":10}'
  ```

  投票值：`10` 核准、`5` 核准但有建議、`0` 未投票（撤回）、`-5` 等待作者、`-10` 拒絕。

## 呈現結果

投票以 emoji 呈現：✅ 核准（10）、☑️ 核准但有建議（5）、⏳ 未投票（0）、⏸️ 等待作者（-5）、❌ 拒絕（-10）。PR 連結格式為 `{repository.webUrl}/pullrequest/{pullRequestId}`；分支名稱去掉 `refs/heads/`。

**PR 清單**以表格呈現：PR ID（可點擊連結）、標題（draft 加註 📝）、儲存庫、作者、目標分支、我的投票、建立日期。「透過群組待我審查」的 PR 在標題後註明群組名稱。

**單一 PR 審查**依序呈現：

1. **概要**：標題、作者、來源 → 目標分支、狀態（含 draft、`mergeStatus` 是否有衝突）、auto-complete 是否已設定。
2. **阻擋原因**：列出所有 blocking 且未通過的原則；全部通過時明確說明可以合併。
3. **審查者**：每位審查者與投票、是否為必要審查者（`isRequired`）。
4. **變更檔案**：表格列出路徑與變更類型，以及檔案總數。
5. **變更重點**：依 diff 逐檔摘要重要變更、可能的風險（錯誤處理、安全性、相容性、缺少測試）與具體建議；標明這是 AI 產生的審查意見。
6. **未解決的討論串**：作者、檔案與行號、內容摘要；沒有時明確說明。
7. **連結的工作項目**：ID（連結 `https://dev.azure.com/{organization}/{project}/_workitems/edit/{ID}`）、類型、標題、狀態；沒有時提醒 PR 應連結工作項目。
