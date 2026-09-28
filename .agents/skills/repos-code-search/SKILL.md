---
name: repos-code-search
description: 以 Azure DevOps REST API 搜尋 Azure Repos 的程式碼與 commit：用 Code Search 語法（ext:、file:、path:、class:、def:、ref:、AND/OR/NOT）跨組織、專案或儲存庫找程式碼，讀取指定分支或 commit 的檔案內容、列出目錄，並依作者、路徑、日期查詢 commit 歷史。當使用者要在遠端儲存庫找某段程式碼、某個類別的定義或引用、看某個檔案的內容，或查誰在什麼時候改過某個檔案時使用；要查 commit 是哪個 PR 帶進來的改用 `repos-pull-request-review`。
compatibility: 需要 curl、可連線至 dev.azure.com 與 almsearch.dev.azure.com 的網路，以及 AZURE_DEVOPS_ORG 與 AZURE_DEVOPS_PAT 環境變數。
---

# 程式碼搜尋

這個 Skill 只讀取：搜尋程式碼、讀取檔案與目錄、查詢 commit 歷史，不修改任何內容。要找出某個 commit 是由哪個 PR 帶進來的，改用 `repos-pull-request-review`（規則 3「由 commit 反查 PR」）。

## 專案選擇

- 使用者有提供專案名稱（例如「Contoso 專案」）就直接使用，不呼叫專案清單 API。
- 使用者要搜尋「整個組織」或「所有專案」的程式碼時，規則 1 改用不含專案的 URL，不需要選專案。
- 其他情況沒有提供時，請使用者提供一次。
- 仍未提供時，列出專案讓使用者選擇：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/projects?%24top=500&api-version=7.1"
  ```

- 使用者提供或選定專案之前不要繼續。

## 儲存庫選擇

規則 1 的儲存庫是選填的（省略代表整個專案）；規則 2 至規則 4 必須指定儲存庫。

- 使用者有提供儲存庫名稱就直接使用。
- 沒有指定時，詢問一次儲存庫名稱。
- 仍未提供時，列出專案中的儲存庫讓使用者選擇：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories?api-version=7.1"
  ```

  顯示 `value[].name`；`isDisabled` 為 `true` 的儲存庫標示為已停用。`defaultBranch` 是預設分支（例如 `refs/heads/main`），`webUrl` 用來組合網頁連結。

## 連線與驗證

一律用 `curl` 直接呼叫 Azure DevOps REST API，每個指令都自帶驗證參數。

- **組織**：讀取環境變數 `AZURE_DEVOPS_ORG`（組織名稱，例如 `contoso`）。未設定時詢問使用者一次；使用者給的是網址時，從 `https://dev.azure.com/{org}` 或 `https://{org}.visualstudio.com` 取出組織名稱。
- **主機**：除了 `dev.azure.com`，程式碼搜尋 API 的主機是 `almsearch.dev.azure.com`。
- **權杖**：PAT 以 Basic 驗證送出：`-u ":$AZURE_DEVOPS_PAT"`；只設定了 `AZURE_DEVOPS_EXT_PAT`（Azure CLI azure-devops 擴充套件使用的變數）時，改用它。改用 Microsoft Entra ID 權杖時，換成 `-H "Authorization: Bearer $AZURE_DEVOPS_TOKEN"`。都未設定時，請使用者在自己的終端機設定 `AZURE_DEVOPS_PAT` 後重試，並停止。權杖只從環境變數讀取：不要請使用者把權杖貼進對話，也不要印出或寫入檔案。
- **PAT 權限範圍**：Code (Read)（已包含程式碼搜尋）。
- **編碼**：路徑中的專案與儲存庫名稱、查詢字串中的檔案路徑、分支名稱與作者要 URL 編碼（空白寫成 `%20`，中文也要編碼）；查詢字串中的 `$` 寫成 `%24`（例如 `%24top`），避免被 shell 展開。JSON 主體中的名稱不需要編碼。
- **錯誤**：HTTP 401、403、203 或回應是 HTML 登入頁，代表權杖無效、過期或權限不足，回報後停止；HTTP 404 代表組織、專案、儲存庫或檔案路徑有誤（檔案不存在時 `message` 含 `TF401174`，分支或 commit 不存在時含 `TF401175`）。
- **Windows PowerShell**：呼叫 `curl.exe`（而非 `curl` 別名），環境變數寫成 `$env:AZURE_DEVOPS_PAT`；PowerShell 不支援 `<<'EOF'`，先把 JSON 主體寫入暫存檔，再以 `--data-binary "@body.json"` 送出。

## 搜尋語法

`searchText` 使用與網頁 Code Search 相同的語法，一律不分大小寫。

| 功能 | 寫法 | 說明 |
| --- | --- | --- |
| 關鍵字 | `validate revisit` | 多個關鍵字之間預設是 `AND` |
| 布林運算 | `A AND B`、`A OR B`、`A NOT B`、`(A NOT B) OR "c d"` | 運算子必須大寫；可用括號分組 |
| 完整片語 | `"Client not found"` | 以雙引號包住 |
| 萬用字元 | `Browser*`、`alpha?version` | `*` 任意多字元、`?` 單一字元 |
| 鄰近 | `A NEAR B`、`A BEFORE B`、`A AFTER B` | 預設距離 5 個 token；運算子必須大寫 |
| 特殊字元 | `"flatten()"`、`"\"react-redux\""` | `( ) [ ] : * ?` 放進雙引號片語；片語內的 `"` 寫成 `\"` |
| 位置 | `proj:Contoso`、`repo:MyApp`、`path:src/Services`、`file:Program*`、`ext:cs` | `path:` 含子目錄，可用 `**` 跨層比對；值含空白時用雙引號 |
| 程式碼類型 | `class:`、`def:`、`ref:`、`method:`、`interface:`、`enum:`、`field:`、`prop:`、`namespace:`、`type:`、`comment:`、`strlit:`、`basetype:`、`decl:`、`macro:` | 只支援 C#、C、C++、Java、Visual Basic.NET；例如 `def:Main`、`comment:todo` |

`func:`、`ctor:`、`methoddef:` 等已併入 `method:`，`struct:`、`typedef:` 已併入 `type:`，`classdef:`、`classdecl:` 已併入 `class:`。

## 規則

### 1. 搜尋程式碼

```bash
curl -sS -u ":$AZURE_DEVOPS_PAT" -X POST -H "Content-Type: application/json" \
  "https://almsearch.dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/search/codesearchresults?api-version=7.1" \
  --data-binary @- <<'EOF'
{
  "searchText": "def:GetOrder ext:cs",
  "$skip": 0,
  "$top": 50,
  "filters": {"Project": ["Contoso"], "Repository": ["MyApp"], "Path": ["/src"], "Branch": ["main"]},
  "includeFacets": false
}
EOF
```

- `filters` 的值是名稱陣列：`Project`、`Repository`、`Path`（含子目錄，不分大小寫）、`Branch`。不需要的篩選整個省略。只給 `Repository` 不給 `Project` 會回 HTTP 400。
- 搜尋整個組織時，URL 去掉 `/{project}`，`filters` 也不放 `Project`。
- 程式碼類型請寫在 `searchText`（例如 `class:Order`）。實測 `filters.CodeElement` 不會縮小結果，只會多回傳 `CodeElement` facet。
- `$top` 必須在 1 到 1000 之間；`count` 是符合的檔案總數，大於已取得的筆數時以 `$skip` 取下一頁。
- **分支**：只有已建立索引的分支能搜尋，預設只有預設分支；其他分支要到專案設定 > Repositories > 儲存庫的 Options 加入可搜尋的分支。指定未建立索引的分支時會回 HTTP 200、`count` 為 0，不會報錯：先以 `"includeFacets": true` 搜尋，`facets.Branch[].name` 就是已建立索引的分支。
- **`infoCode`**：`0` 正常；`1`、`2`、`6`、`7`、`9` 代表組織或分支仍在建立索引，結果可能不完整，要告知使用者；`8` 代表 `$top` 被截斷。
- **Code Search 未安裝**：程式碼搜尋需要組織安裝 Code Search 擴充套件（Marketplace 的 `ms.vss-code-search`）。回應不是 HTTP 200，或 `message` 提到 search 服務或擴充套件無法使用時，回報原始訊息、請組織管理員安裝，並改用規則 3 列出目錄、規則 2 逐檔讀取，或在本機 clone 後用 `git grep`。
- 每筆結果：`fileName`、`path`、`repository.name`、`project.name`、`versions[].branchName` 與 `versions[].changeId`（該分支的 commit）、`matches.content[]`（符合位置）、`matches.fileName[]`（檔名符合）。
- `matches.content[]` 中可用的只有 `charOffset`、`length` 與 `type`（`content`、`class`、`ref`、`comment`、`strlit` 等）；`line` 與 `column` 一律是 `0`，`codeSnippet` 即使設 `includeSnippet: true` 也是 `null`。要顯示符合的程式碼行時，用規則 2 以 `versions[0].changeId` 讀取檔案，把 `charOffset` 之前的換行數加 1 當作行號（位移以字元計算，不含檔案開頭的 BOM）。每個檔案最多顯示前 3 處。

範例：「在 Contoso 專案找 `GetOrder` 的定義」、「整個組織哪裡用到 `ConnectionString`？」、「MyApp 的 src 底下有哪些 TODO 註解？」

### 2. 讀取檔案

```bash
curl -sS -u ":$AZURE_DEVOPS_PAT" \
  "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories/{repository}/items?path={filePath}&versionDescriptor.version={branch}&versionDescriptor.versionType=branch&api-version=7.1"
```

- 不帶 `%24format` 時回傳檔案的原始內容（純文字）。
- 省略 `versionDescriptor.*` 代表預設分支；讀取特定 commit 或 tag 時，把 `versionType` 換成 `commit` 或 `tag`。
- 檔案不存在時回 HTTP 404（`TF401174`）；二進位檔只回報檔案類型，不要顯示內容。大檔案用 `-o` 存到暫存目錄，只顯示使用者需要的段落，用完即刪。

範例：「讀 MyApp 的 `/src/appsettings.json`（release 分支）」

### 3. 列出目錄

```bash
curl -sS -u ":$AZURE_DEVOPS_PAT" \
  "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories/{repository}/items?scopePath={folderPath}&recursionLevel=OneLevel&versionDescriptor.version={branch}&versionDescriptor.versionType=branch&api-version=7.1"
```

- `value[0]` 是目錄本身，其餘是子項目。每筆有 `path`、`isFolder`（檔案沒有這個欄位）與 `gitObjectType`：`tree` 是目錄、`blob` 是檔案、`commit` 是 submodule。
- 要列出整棵子樹時改用 `recursionLevel=Full`；項目很多時先告知筆數，只列出與使用者問題相關的部分。

範例：「MyApp 根目錄有哪些檔案？」

### 4. 查詢 commit 歷史

```bash
curl -sS -u ":$AZURE_DEVOPS_PAT" \
  "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories/{repository}/commits?searchCriteria.itemVersion.version={branch}&searchCriteria.author={author}&searchCriteria.itemPath={path}&searchCriteria.fromDate=2025-01-01T00:00:00Z&searchCriteria.toDate=2025-02-01T00:00:00Z&searchCriteria.%24top=50&api-version=7.1"
```

| 參數 | 說明 |
| --- | --- |
| `searchCriteria.itemVersion.version` | 分支名稱；省略代表預設分支 |
| `searchCriteria.author` | 作者名稱或 email，部分字串也會比對 |
| `searchCriteria.itemPath` | 檔案或目錄路徑，只回傳動到它的 commit |
| `searchCriteria.fromDate`、`searchCriteria.toDate` | ISO 8601 日期時間；使用者給當地日期時先換算成 UTC |
| `searchCriteria.%24top`、`searchCriteria.%24skip` | 分頁；預設最多 100 筆，回傳筆數等於 `%24top` 時再取下一頁 |

- 不需要的條件整個省略。
- **陷阱**：`searchCriteria.fromCommitId` 與 `searchCriteria.toCommitId` 是依 commit ID **字母順序**篩選的上下界，不是「兩個 commit 之間的歷史」。要比較兩個版本之間的 commit，在本機 clone 後用 `git log {from}..{to}`。
- 這個 API 不能依 commit 訊息搜尋：要找訊息中的關鍵字時，先用日期、作者或路徑縮小範圍，再比對 `comment`。`commentTruncated` 為 `true` 時，以 `GET .../commits/{commitId}?api-version=7.1` 取得完整訊息。
- 每筆有 `commitId`、`author.name`、`author.email`、`author.date`、`comment`、`changeCounts`（`Add`、`Edit`、`Delete` 的數量）。

範例：「誰最近改過 `/src/Services/OrderService.cs`？」、「列出 someone@contoso.com 上個月在 main 的 commit」

## 呈現結果

日期顯示為 `YYYY/MM/DD`，時間轉成使用者當地時區，無法判斷時標明 UTC。

- **程式碼搜尋**：先說明搜尋條件（語法、專案、儲存庫、分支）與符合的檔案總數。以表格呈現：檔案路徑（可點擊，連結 `https://dev.azure.com/{organization}/{project}/_git/{repository}?path={path}&version=GB{branch}`）、儲存庫、分支、符合處數。有讀取檔案時，在表格下方逐檔列出行號與該行程式碼。還有下一頁時告知剩餘數量；沒有結果時明確說明，並提醒分支可能未建立索引或 `infoCode` 的狀況。
- **檔案內容**：以程式碼區塊呈現，依副檔名標示語言，並註明分支或 commit。
- **目錄**：先列目錄再列檔案，目錄名稱後加 `/`，submodule 標註「submodule」。
- **commit 歷史**：表格呈現短 SHA（前 7 碼，連結 `https://dev.azure.com/{organization}/{project}/_git/{repository}/commit/{commitId}`）、作者、日期、訊息第一行、變更檔案數。沒有 commit 時明確說明。
