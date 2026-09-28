---
name: pipelines-release-notes
description: 以 Azure DevOps REST API 產生兩次建置之間的 Release Notes（Markdown）：比對使用者指定的兩個建置，或某個建置與同一 Pipeline、同一分支上一次成功的建置，整理期間的 commit、合併的 PR、連結的工作項目（功能、Bug、其他）與貢獻者。當使用者要寫 release notes、changelog、版本說明，或問「這次部署比上次多了什麼」時使用；只看單一建置的狀態、記錄檔或它自己的 commit 清單改用 `pipelines-build-summary`。
compatibility: 需要 curl、可連線至 dev.azure.com 的網路，以及 AZURE_DEVOPS_ORG 與 AZURE_DEVOPS_PAT 環境變數。
---

# Release Notes

這個 Skill 是唯讀的：以**兩個建置**界定範圍（不含起點建置、含終點建置），把期間的變更整理成可以直接貼上的 Markdown。

## 專案選擇

- 使用者有提供專案名稱（例如「Contoso 專案」）就直接使用，不呼叫專案清單 API。
- 沒有提供時，請使用者提供一次。
- 仍未提供時，列出專案讓使用者選擇：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/projects?%24top=500&api-version=7.1"
  ```

- 使用者提供或選定專案之前不要繼續。

## 連線與驗證

一律用 `curl` 直接呼叫 Azure DevOps REST API，每個指令都自帶驗證參數。

- **組織**：讀取環境變數 `AZURE_DEVOPS_ORG`（組織名稱，例如 `contoso`）。未設定時詢問使用者一次；使用者給的是網址時，從 `https://dev.azure.com/{org}` 或 `https://{org}.visualstudio.com` 取出組織名稱。
- **權杖**：PAT 以 Basic 驗證送出：`-u ":$AZURE_DEVOPS_PAT"`；只設定了 `AZURE_DEVOPS_EXT_PAT`（Azure CLI azure-devops 擴充套件使用的變數）時，改用它。改用 Microsoft Entra ID 權杖時，換成 `-H "Authorization: Bearer $AZURE_DEVOPS_TOKEN"`。都未設定時，請使用者在自己的終端機設定 `AZURE_DEVOPS_PAT` 後重試，並停止。權杖只從環境變數讀取：不要請使用者把權杖貼進對話，也不要印出或寫入檔案。
- **PAT 權限範圍**：Build (Read)、Code (Read)、Work Items (Read)。
- **編碼**：路徑與查詢字串中的專案、Pipeline 名稱要 URL 編碼（空白寫成 `%20`）；查詢字串中的 `$` 寫成 `%24`（例如 `%24top`），避免被 shell 展開。
- **錯誤**：HTTP 401、403、203 或回應是 HTML 登入頁，代表權杖無效、過期或權限不足，回報後停止；HTTP 404 代表組織、專案、建置 ID 或 API 版本有誤。
- **Windows PowerShell**：呼叫 `curl.exe`（而非 `curl` 別名），環境變數寫成 `$env:AZURE_DEVOPS_PAT`；PowerShell 不支援 `<<'EOF'`，先把 JSON 主體寫入暫存檔，再以 `--data-binary "@body.json"` 送出。

## 步驟

### 1. 決定比較範圍

- **終點建置**（`to`）：使用者有給建置 ID 就用它，取得建置以得知 `definition.id`、`sourceBranch`、`finishTime`、`buildNumber`、`repository.name`：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/build/builds/{toBuildId}?api-version=7.1"
  ```

  使用者只給 Pipeline 名稱時，以 `GET .../_apis/build/definitions?name={name}&api-version=7.1` 找出定義 ID，終點用下面查詢的第一筆（最新一次成功的建置）。
- **起點建置**（`from`）：使用者有給就用它（兩個建置要屬於同一個儲存庫；起點必須較早，給反了就對調）。沒有給時，找同一定義、同一分支上一次成功的建置：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/build/builds?definitions={definitionId}&branchName=refs/heads/main&resultFilter=succeeded&%24top=2&queryOrder=finishTimeDescending&api-version=7.1"
  ```

  `value[0]` 是終點、`value[1]` 是起點。終點不是最新的成功建置時，加上 `&maxTime={toFinishTime}`（包含該時間），結果一樣是 `[終點, 起點]`。只找到一筆時，說明沒有更早的成功建置可比較，請使用者指定起點。
- 把範圍（兩個建置的編號、分支、完成時間）先告訴使用者，再繼續。

### 2. 取得期間的 commit

```bash
curl -sS -u ":$AZURE_DEVOPS_PAT" \
  "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/build/changes?fromBuildId={fromBuildId}&toBuildId={toBuildId}&%24top=200&api-version=7.1-preview.2"
```

- Microsoft Learn 只列出這個資源的 preview 版本（`7.1-preview.2`）。
- 範圍不含起點建置本身的 commit，含終點建置的 commit。
- 沒有分頁：回傳筆數等於 `%24top` 時可能被截斷，加大 `%24top`（例如 `1000`）重查。
- 每筆有 `id`（完整 SHA）、`message`（`messageTruncated` 為 `true` 時不完整）、`type`（`TfsGit` 是 Azure Repos、`GitHub` 是 GitHub）、`author.displayName`、`timestamp`、`location`（Azure Repos 為 `.../git/repositories/{repositoryId}/commits/{sha}`，可從中取出儲存庫 ID）。
- 沒有任何 commit 時，說明兩次建置之間沒有程式碼變更（可能是重跑同一個 commit），並停止。

### 3. 取得連結的工作項目

```bash
curl -sS -u ":$AZURE_DEVOPS_PAT" \
  "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/build/workitems?fromBuildId={fromBuildId}&toBuildId={toBuildId}&%24top=200&api-version=7.1-preview.2"
```

- 同樣只有 preview 版本。回傳 `value[].id`（字串）；單一建置的工作項目改用 `GET .../_apis/build/builds/{buildId}/workitems?api-version=7.1`。
- 以 `workitemsbatch` 取欄位（每次最多 200 個 ID，`errorPolicy` 設 `Omit` 讓無權限或已刪除的項目回傳 `null` 而不是整批失敗）：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" -X POST -H "Content-Type: application/json" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wit/workitemsbatch?api-version=7.1" \
    -d '{"ids":[101,102],"fields":["System.Id","System.Title","System.WorkItemType","System.State"],"errorPolicy":"Omit"}'
  ```

- 工作項目來自 commit 與 PR 的連結，包含 commit 訊息裡的 `#123` 寫法；標題看起來與這批變更無關的項目（例如很小的 ID、多年前的項目）另外列出，請使用者確認是否保留。

### 4. 找出合併的 PR

- **Azure Repos**（`type` 為 `TfsGit`）：把步驟 2 的所有 SHA 一次送出查詢，`lastMergeCommit` 找出以這些 commit 合併的 PR。這是唯讀查詢（成功時回 HTTP 201），SHA 必須是完整 40 碼，一個查詢可放上百個 SHA：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" -X POST -H "Content-Type: application/json" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories/{repositoryId}/pullrequestquery?api-version=7.1" \
    -d '{"queries":[{"type":"lastMergeCommit","items":["{commitSha1}","{commitSha2}"]}]}'
  ```

  `results[0]` 是 `{ "<sha>": [PR, ...] }` 的對照表，只包含找得到 PR 的 SHA。PR 有 `pullRequestId`、`title`、`createdBy.displayName`、`sourceRefName`、`targetRefName`。功能分支先合併到中間分支（例如 `develop`）、再整批合併到發布分支時，兩層 PR 都會出現：以目標不是建置分支的 PR 作為功能清單，目標是建置分支的 PR 註明為「發布合併」。
- **GitHub**（`type` 為 `GitHub`）：無法查 Azure Repos 的 PR，從 commit 訊息解析 `Merge pull request #123` 或 squash merge 標題結尾的 `(#123)`，連結寫成 `https://github.com/{owner}/{repo}/pull/123`（`{owner}/{repo}` 取自建置的 `repository.id`）。
- Azure Repos 的合併訊息通常是 `Merged PR 123: 標題`，查詢失敗時可用它當備援。

### 5. 分類與整理

- **✨ 新功能**：`System.WorkItemType` 為 Feature、Epic、User Story、Product Backlog Item、Requirement 的工作項目。
- **🐛 Bug 修正**：Bug（與 Issue，視流程範本而定）。
- **🔧 其他**：Task 等其他類型。
- 每個工作項目列出相關的 PR（若可對應）。沒有連結工作項目的 PR 放在「其他變更」。
- **重要 commit**：沒有被任何 PR 涵蓋、也不是合併 commit（訊息以 `Merge` 或 `Merged PR` 開頭）的直接 commit，挑出有實質內容的列出（例如 `feat:`、`fix:` 前綴）；數量很多時只列前 20 筆並註明總數。
- **貢獻者**：commit `author.displayName` 去重後依 commit 數排序。
- 工作項目狀態不是完成類（例如 New、Active）時加註，提醒可能尚未完成驗收。

範例：「幫 Contoso 專案 MyApp-CI 最新的建置產生 release notes」、「build 12300 到 12345 之間有哪些變更？」

## 呈現結果

時間以使用者的當地時區顯示（格式 `YYYY/MM/DD`），無法判斷時區時標明 UTC。工作項目連結為 `https://dev.azure.com/{organization}/{project}/_workitems/edit/{ID}`；Azure Repos PR 連結為 `https://dev.azure.com/{organization}/{project}/_git/{repositoryName}/pullrequest/{pullRequestId}`（`{repositoryName}` 取自建置的 `repository.name`，查詢結果的 PR 不含 `webUrl`）；建置連結為 `https://dev.azure.com/{organization}/{project}/_build/results?buildId={buildId}`。

以一個 Markdown 區塊輸出，方便使用者複製：

```markdown
# MyApp 20260915.3 Release Notes

比較範圍：[20260910.1](建置連結) → [20260915.3](建置連結)（main，2026/09/10 – 2026/09/15）
共 23 個 commit、6 個 PR、5 個工作項目，4 位貢獻者。

## ✨ 新功能
- [#101](連結) 會員可以匯出訂單（PR [!12345](連結)）

## 🐛 Bug 修正
- [#102](連結) 修正結帳金額四捨五入錯誤（PR [!12346](連結)）

## 🔧 其他變更
- PR [!12347](連結) 升級 Node.js 版本

## 貢獻者
Contoso Dev A、Contoso Dev B
```

沒有資料的段落省略。最後在區塊外補充：可能被截斷的清單、無法對應的 PR、需要使用者確認的工作項目。
