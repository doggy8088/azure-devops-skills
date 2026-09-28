---
name: repos-pull-request-write
description: 以 Azure DevOps REST API 從目前的 git 分支建立 Pull Request（PR）：依 commit 與 PR 範本擬定標題與描述、連結工作項目、指定審查者或建立 draft；並修改自己的 PR：發布 draft、加入審查者、設定或取消 auto-complete、放棄或重新啟用。每個寫入動作都先確認。當使用者要開 PR、create PR、publish draft、加 reviewer 或設定自動完成時使用；審查、留言、投票等審查者端操作改用 `repos-pull-request-review`。
compatibility: 需要 curl、git、可連線至 dev.azure.com 與 vssps.dev.azure.com 的網路，以及 AZURE_DEVOPS_ORG 與 AZURE_DEVOPS_PAT 環境變數。
---

# Pull Request 建立與修改

這個 Skill 站在**作者**的角度處理 PR：從本機目前的分支建立 PR，以及修改既有的 PR。不提供手動完成（complete）或合併 PR；要合併時設定 auto-complete，或請使用者到網頁操作。審查、留言、回覆與投票改用 `repos-pull-request-review`。

## 儲存庫判斷

在 git 工作目錄中，從遠端網址取出組織、專案與儲存庫，並取得目前分支：

```bash
git remote get-url origin
git rev-parse --abbrev-ref HEAD
```

| 遠端網址格式 | 說明 |
| --- | --- |
| `https://{org}@dev.azure.com/{org}/{project}/_git/{repository}` | 從網頁複製的 HTTPS 網址 |
| `https://dev.azure.com/{org}/{project}/_git/{repository}` | 不含使用者名稱的 HTTPS 網址 |
| `https://{org}.visualstudio.com/[DefaultCollection/]{project}/_git/{repository}` | 舊網域；`DefaultCollection/` 可有可無 |
| `git@ssh.dev.azure.com:v3/{org}/{project}/{repository}` | SSH 網址 |

- 網址中的名稱可能已 URL 編碼（例如 `My%20App`）：放進 API 路徑時照用，顯示給使用者時先解碼。
- 遠端網址的組織與 `AZURE_DEVOPS_ORG` 不同時，詢問使用者一次要用哪一個。
- `rev-parse` 回傳 `HEAD` 代表 detached HEAD：請使用者先切換到要建立 PR 的分支，並停止。
- 目前目錄不是 git 儲存庫、遠端不是 Azure DevOps，或使用者要處理的是別的儲存庫時，請使用者提供專案與儲存庫名稱；仍未提供時，列出專案與儲存庫讓使用者選擇：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/projects?%24top=500&api-version=7.1"
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories?api-version=7.1"
  ```

  顯示 `value[].name`；`isDisabled` 為 `true` 的儲存庫標示為已停用。規則 1 必須在該儲存庫的本機工作目錄中執行。

## 連線與驗證

一律用 `curl` 直接呼叫 Azure DevOps REST API，每個指令都自帶驗證參數。

- **組織**：讀取環境變數 `AZURE_DEVOPS_ORG`（組織名稱，例如 `contoso`）。未設定時詢問使用者一次；使用者給的是網址時，從 `https://dev.azure.com/{org}` 或 `https://{org}.visualstudio.com` 取出組織名稱。
- **主機**：除了 `dev.azure.com`，以 email 或名稱查詢審查者身分的 Identities API 主機是 `vssps.dev.azure.com`。
- **權杖**：PAT 以 Basic 驗證送出：`-u ":$AZURE_DEVOPS_PAT"`；只設定了 `AZURE_DEVOPS_EXT_PAT`（Azure CLI azure-devops 擴充套件使用的變數）時，改用它。改用 Microsoft Entra ID 權杖時，換成 `-H "Authorization: Bearer $AZURE_DEVOPS_TOKEN"`。都未設定時，請使用者在自己的終端機設定 `AZURE_DEVOPS_PAT` 後重試，並停止。權杖只從環境變數讀取：不要請使用者把權杖貼進對話，也不要印出或寫入檔案。
- **PAT 權限範圍**：讀取所需：Code (Read)、Identity (Read)（查詢審查者）、Work Items (Read)（確認工作項目）；寫入所需：Code (Read & write)（建立與修改 PR）。
- **編碼**：路徑中的專案與儲存庫名稱、查詢字串中的分支名稱與審查者名稱要 URL 編碼（空白寫成 `%20`，中文與 `#` 等字元也要編碼）；查詢字串中的 `$` 寫成 `%24`（例如 `%24top`），避免被 shell 展開。
- **錯誤**：HTTP 401、403、203 或回應是 HTML 登入頁，代表權杖無效、過期或權限不足，回報後停止；HTTP 404 代表組織、專案、儲存庫或 PR ID 有誤。建立 PR 時回 HTTP 409（`TF401179`），代表同一組來源與目標分支已有進行中的 PR，改用規則 2 修改它。
- **Windows PowerShell**：呼叫 `curl.exe`（而非 `curl` 別名），環境變數寫成 `$env:AZURE_DEVOPS_PAT`；PowerShell 不支援 `<<'EOF'`，先把 JSON 主體寫入暫存檔，再以 `--data-binary "@body.json"` 送出。

## 目前使用者

檢查 PR 作者與設定 auto-complete 需要目前使用者的身分 ID（這個端點不帶 `api-version`）：

```bash
curl -sS -u ":$AZURE_DEVOPS_PAT" "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/connectionData"
```

使用 `authenticatedUser.id`；`authenticatedUser.providerDisplayName` 是顯示名稱。

## 規則

### 1. 從目前分支建立 PR

1. **確認來源分支已推送**：

   ```bash
   git status --porcelain
   git ls-remote --heads origin {branch}
   ```

   - 有未 commit 的變更時提醒使用者：這些變更不會進入 PR。
   - `ls-remote` 沒有輸出代表遠端沒有這個分支：詢問使用者是否執行 `git push -u origin {branch}`，同意後才推送。
   - 遠端已有分支，但 `git fetch origin {branch}` 後 `git rev-list --count origin/{branch}..HEAD` 大於 0（有未推送的 commit）時，同樣先詢問再 `git push`。

2. **決定目標分支**：使用者有指定就用；否則使用儲存庫的 `defaultBranch`（例如 `refs/heads/main`）。同時記下 `id` 與 `webUrl`：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories/{repository}?api-version=7.1"
   ```

   來源與目標分支相同時停止。

3. **檢查是否已有 PR**：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories/{repository}/pullrequests?searchCriteria.sourceRefName=refs/heads/{branch}&searchCriteria.status=active&api-version=7.1"
   ```

   同一個來源分支可能有多個進行中的 PR（目標分支不同）。有 `targetRefName` 等於目標分支的 PR 時，顯示它的 ID 與連結，詢問是否改為修改它（規則 2），不要重複建立。

4. **擬定標題與描述**：

   ```bash
   git fetch origin {targetBranch}
   git log --oneline --no-merges origin/{targetBranch}..HEAD
   ```

   - 沒有任何 commit 時停止：這個分支沒有要合併的變更。
   - 只有一個 commit：標題用它的第一行，描述用其餘內文（`git log -1 --format=%b`）。
   - 多個 commit：標題概括整體目的，描述以條列整理變更重點。
   - 使用者提供的標題或描述優先。描述上限 4000 字元，超過時精簡。

5. **套用 PR 範本**：只有在網頁建立 PR 時才會自動帶入範本；透過 API 建立時要自己讀取並填寫。範本一律取自**預設分支**，檔名與資料夾不分大小寫：

   ```bash
   git fetch origin {defaultBranch}
   git ls-tree -r --name-only origin/{defaultBranch} \
     | grep -i -E '^(\.azuredevops/|\.vsts/|docs/)?pull_request_template(\.md|\.txt|/)'
   git show "origin/{defaultBranch}:{templatePath}"
   ```

   依序在 `.azuredevops/`、`.vsts/`、`docs/`、儲存庫根目錄尋找，使用第一個符合的範本：
   1. **分支專屬範本**：`pull_request_template/branches/` 底下以目標分支命名的 `.md` 或 `.txt`。可以只對應第一層（`release.md` 套用到 `release` 與所有 `release/*`），也可以用巢狀資料夾對應多層（`release/october.md` 套用到 `release/october` 與 `release/october/*`）。
   2. **預設範本**：`pull_request_template.md` 或 `pull_request_template.txt`。

   `pull_request_template/`（不含 `branches/`）中的其他檔案是**選用範本**，只在使用者要求時附加。套用範本時保留它的標題與核取清單，把步驟 4 的變更摘要填進對應段落。

6. **連結工作項目**：從分支名稱與 commit 訊息找出工作項目 ID，例如 `AB#101`、`#101`、`feature/101-login`。把候選 ID 列給使用者確認，只連結確認過的項目。可以先查標題與狀態幫助確認（不存在或無權限的 ID 在 `value` 中是 `null`）：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/wit/workitems?ids=101,102&fields=System.Title,System.State,System.WorkItemType&errorPolicy=omit&api-version=7.1"
   ```

7. **解析審查者**：使用者以 email 或名稱指定審查者時，查出身分 ID：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://vssps.dev.azure.com/$AZURE_DEVOPS_ORG/_apis/identities?searchFilter=General&filterValue={emailOrName}&api-version=7.1"
   ```

   使用 `value[].id`；`providerDisplayName` 是顯示名稱，`properties.Mail.$value` 是 email。團隊或群組寫成 `[{project}]\{team}`（例如 `[Contoso]\Contoso Team`，URL 編碼後送出）。找到多筆時請使用者選擇；找不到時回報，不要猜。必要審查者設 `isRequired: true`。

8. **確認並建立**：把即將送出的內容列給使用者，確認一次後才執行：儲存庫、來源 → 目標分支、標題、完整描述、是否為 draft、審查者（必要／選用）、工作項目與標籤。

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -X POST -H "Content-Type: application/json" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories/{repository}/pullrequests?api-version=7.1" \
     --data-binary @- <<'EOF'
   {
     "sourceRefName": "refs/heads/feature/101-login",
     "targetRefName": "refs/heads/main",
     "title": "新增登入頁面",
     "description": "## 變更內容\n- 新增登入表單\n- 補上單元測試",
     "isDraft": false,
     "reviewers": [{"id": "{reviewerId}", "isRequired": true}],
     "workItemRefs": [{"id": "101"}],
     "labels": [{"name": "frontend"}]
   }
   EOF
   ```

   成功時回 HTTP 201，記下 `pullRequestId`。沒有審查者、工作項目或標籤時省略該欄位。描述含換行與引號時，先用 `jq -n --arg title "..." --arg description "..."` 產生 JSON，避免跳脫錯誤。

9. 建立時無法同時設定 auto-complete。使用者要自動完成時，建立後接著執行規則 2 的「設定 auto-complete」。

範例：「幫我把目前分支開 PR 到 main」、「開一個 draft PR，審查者是 someone@contoso.com」

### 2. 修改既有的 PR

先取得 PR（不需要知道儲存庫），記下 `repository.id`、`status`、`isDraft`、`createdBy.id`、`autoCompleteSetBy`、`completionOptions`、`targetRefName` 與 `reviewers[]`：

```bash
curl -sS -u ":$AZURE_DEVOPS_PAT" \
  "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/pullrequests/{pullRequestId}?api-version=7.1"
```

`createdBy.id` 不是目前使用者時，先提醒使用者這不是自己的 PR。`status` 為 `completed` 的 PR 不能修改。

每個修改之前，把 PR（ID 與標題）、要變更的欄位、目前值與新值列給使用者，確認一次後才執行。除了加入審查者，都用同一個 PATCH 端點，只送要變更的欄位：

```bash
curl -sS -u ":$AZURE_DEVOPS_PAT" -X PATCH -H "Content-Type: application/json" \
  "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories/{repositoryId}/pullrequests/{pullRequestId}?api-version=7.1" \
  -d '{"isDraft":false}'
```

| 動作 | 主體 |
| --- | --- |
| 發布 draft | `{"isDraft":false}` |
| 轉回 draft | `{"isDraft":true}` |
| 修改標題或描述 | `{"title":"...","description":"..."}` |
| 變更目標分支 | `{"targetRefName":"refs/heads/release"}` |
| 取消 auto-complete | `{"autoCompleteSetBy":{"id":"00000000-0000-0000-0000-000000000000"}}` |
| 放棄（abandon） | `{"status":"abandoned"}` |
| 重新啟用放棄的 PR | `{"status":"active"}` |

- Learn 文件列出的可更新欄位是 `status`、`title`、`description`（上限 4000 字元）、`completionOptions`、`mergeOptions`、`autoCompleteSetBy.id` 與 `targetRefName`（需啟用 PR retargeting 功能）；其他欄位可能被忽略或回錯誤。`isDraft` 不在清單中，但 Azure DevOps CLI 的 `az repos pr update --draft` 就是用這個 PATCH 送出 `isDraft`。送出後檢查回應的 `isDraft`；沒有變更時回報，並請使用者到網頁操作。
- 取消 auto-complete 用空 GUID，與 `az repos pr update --auto-complete false` 的作法相同；送出後重新取得 PR，確認 `autoCompleteSetBy` 已經不存在。
- 不提供 `"status":"completed"`（手動完成），也不提供 `bypassPolicy`（略過原則）。

**設定 auto-complete**：所有必要（blocking）原則通過後自動完成 PR。draft PR 不會完成，先發布。

1. 查詢目標分支允許的合併策略。`type.id` 為 `fa4e907d-c16b-4a4c-9dfa-4916e5d171ab`（Require a merge strategy）且 `isEnabled` 為 `true` 時，只有 `settings` 中 `allowNoFastForward`、`allowSquash`、`allowRebase`、`allowRebaseMerge` 為 `true` 的策略可用；沒有這個原則時全部可用：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/policy/configurations?repositoryId={repositoryId}&refName=refs/heads/main&api-version=7.1"
   ```

2. 和使用者確認合併策略、是否刪除來源分支、是否轉換工作項目狀態，再送出（`{myId}` 取自「目前使用者」）：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -X PATCH -H "Content-Type: application/json" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories/{repositoryId}/pullrequests/{pullRequestId}?api-version=7.1" \
     -d '{"autoCompleteSetBy":{"id":"{myId}"},"completionOptions":{"mergeStrategy":"squash","deleteSourceBranch":true,"transitionWorkItems":true}}'
   ```

   `mergeStrategy`：`noFastForward`（建立 merge commit）、`squash`（壓成單一 commit）、`rebase`（rebase 後 fast-forward）、`rebaseMerge`（rebase 後建立 merge commit）。需要自訂合併訊息時加 `mergeCommitMessage`。成功時回應的 `autoCompleteSetBy.id` 等於 `{myId}`。

**加入審查者**：依規則 1 步驟 7 查出身分 ID；已經在 `reviewers[]` 中的人不要再送，以免覆蓋既有的投票。每位審查者一次呼叫：

```bash
curl -sS -u ":$AZURE_DEVOPS_PAT" -X PUT -H "Content-Type: application/json" \
  "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories/{repositoryId}/pullRequests/{pullRequestId}/reviewers/{reviewerId}?api-version=7.1" \
  -d '{"vote":0,"isRequired":false}'
```

必要審查者把 `isRequired` 設為 `true`。移除審查者請使用者到網頁操作。

範例：「把 PR 101 發布」、「PR 101 加 someone@contoso.com 當必要審查者」、「PR 101 設定 squash 自動完成並刪除分支」、「取消 PR 101 的自動完成」、「放棄 PR 101」

## 呈現結果

PR 連結格式為 `{repository.webUrl}/pullrequest/{pullRequestId}`；分支名稱去掉 `refs/heads/`；日期顯示為 `YYYY/MM/DD`。

- **建立前的確認**：以清單列出將送出的每個欄位，描述完整顯示（不要只給摘要），並標明範本來源檔案與偵測到的工作項目。
- **建立後**：PR ID（可點擊連結）、標題（draft 加註 📝）、來源 → 目標分支、審查者（必要者標註「必要」）、連結的工作項目（`https://dev.azure.com/{organization}/{project}/_workitems/edit/{ID}`）。
- **已有 PR 時**：列出既有 PR 的 ID、標題、目標分支與連結，詢問是否改為修改它。
- **修改後**：PR 連結，以及每個欄位「修改前 → 修改後」；auto-complete 要列出合併策略、是否刪除來源分支與轉換工作項目。
- **失敗時**：顯示 HTTP 狀態碼與回應中的 `message`，不要自動重試寫入。
