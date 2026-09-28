---
name: boards-work-item-write
description: 以 Azure DevOps REST API 建立與修改工作項目（work item）：建立 User Story、Product Backlog Item、Bug、Task 並設定父項目，把一個工作項目拆解成多個子 Task，更新欄位、狀態（State）、Sprint 與指派對象，以及新增留言。建立與更新一律先以 validateOnly 乾跑，並經使用者確認後才寫入。當使用者要開單、拆 Task、改狀態、改指派、移到其他 Sprint 或在工作項目上留言時使用；只要讀取或摘要單一工作項目改用 `boards-work-item-summary`。
compatibility: 需要 curl、可連線至 dev.azure.com 的網路，以及 AZURE_DEVOPS_ORG 與 AZURE_DEVOPS_PAT 環境變數。
---

# 工作項目建立與修改

這個 Skill 負責**寫入**工作項目：建立、拆解、更新與留言。不提供刪除；需要刪除工作項目時，請使用者自行到網頁操作。永遠不要加 `bypassRules=true`：流程規則（必填欄位、允許的狀態）必須照常生效。

## 專案選擇

- 使用者有提供專案名稱（例如「Contoso 專案」）就直接使用，不呼叫專案清單 API。
- 沒有提供時，請使用者提供一次。
- 仍未提供時，列出專案讓使用者選擇：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/projects?%24top=500&api-version=7.1"
  ```

- 使用者提供或選定專案之前不要繼續。

更新、拆解或留言時，專案以工作項目的 `System.TeamProject` 為準，不必另外詢問。

## 連線與驗證

一律用 `curl` 直接呼叫 Azure DevOps REST API，每個指令都自帶驗證參數。

- **組織**：讀取環境變數 `AZURE_DEVOPS_ORG`（組織名稱，例如 `contoso`）。未設定時詢問使用者一次；使用者給的是網址時，從 `https://dev.azure.com/{org}` 或 `https://{org}.visualstudio.com` 取出組織名稱。
- **權杖**：PAT 以 Basic 驗證送出：`-u ":$AZURE_DEVOPS_PAT"`；只設定了 `AZURE_DEVOPS_EXT_PAT`（Azure CLI azure-devops 擴充套件使用的變數）時，改用它。改用 Microsoft Entra ID 權杖時，換成 `-H "Authorization: Bearer $AZURE_DEVOPS_TOKEN"`。都未設定時，請使用者在自己的終端機設定 `AZURE_DEVOPS_PAT` 後重試，並停止。權杖只從環境變數讀取：不要請使用者把權杖貼進對話，也不要印出或寫入檔案。
- **PAT 權限範圍**：讀取（工作項目、類型、狀態、流程設定）需要 Work Items (Read)、Project and Team (Read)；建立、更新與留言另需 Work Items (Read & write)。
- **編碼**：路徑中的專案、團隊與工作項目類型名稱要 URL 編碼（空白寫成 `%20`）；查詢字串中的 `$` 寫成 `%24`（例如 `%24expand`），避免被 shell 展開。
- **錯誤**：HTTP 401、403、203 或回應是 HTML 登入頁，代表權杖無效、過期或權限不足，回報後停止；HTTP 404 代表組織、專案、工作項目類型或 ID 有誤。HTTP 400 且 `typeKey` 為 `RuleValidationException`，代表欄位值不符合流程規則，把 `message` 原文給使用者看後停止，不要自行換值重試。
- **Windows PowerShell**：呼叫 `curl.exe`（而非 `curl` 別名），環境變數寫成 `$env:AZURE_DEVOPS_PAT`；PowerShell 不支援 `<<'EOF'`，先把 JSON 主體寫入暫存檔，再以 `--data-binary "@body.json"` 送出。

## 寫入流程（每個建立與更新都照做）

1. 先以 GET 取得最新資料（工作項目、類型的欄位與狀態），不要憑記憶猜欄位名稱或狀態名稱。
2. 組好 JSON Patch 主體，URL 加上 `validateOnly=true` **乾跑**。乾跑成功回 HTTP 200，回應有 `fields`、`_links`、`url`，但**沒有** `id` 與 `rev`，也不會建立或修改任何東西；沒指定的欄位會填入預設值（例如 `System.State` 顯示該類型的初始狀態）。
3. 把即將送出的內容列給使用者：目標（專案與類型，或工作項目 ID 與標題）、每個欄位的目前值 → 新值、影響（例如「將建立 4 個子 Task」），並附上乾跑回應中的 `System.State`、`System.AreaPath`、`System.IterationPath`。確認一次後才執行。
4. 拿掉 `validateOnly=true` 正式送出，回報結果。

JSON 主體一律用 `--data-binary @- <<'EOF'` heredoc；heredoc 內不會展開 shell 變數，組織名稱等值要直接寫入。路徑中的反斜線在 JSON 裡要寫成 `\\`（`Contoso\\Sprint 1`）。

## 欄位、類型與狀態

| 欄位 | 參考名稱 | 說明 |
|---|---|---|
| 標題 | `System.Title` | 必填 |
| 描述 | `System.Description` | HTML，例如 `<p>重設密碼流程</p>` |
| 指派給 | `System.AssignedTo` | 電子郵件或 unique name；`""` 代表取消指派 |
| 區域 | `System.AreaPath` | `Contoso\Web` |
| Iteration | `System.IterationPath` | `Contoso\Sprint 1`（不含 `Iteration` 這一層） |
| 標籤 | `System.Tags` | 以 `; ` 分隔：`api; backend`。寫入會取代整個欄位，要保留原標籤就先讀出再合併 |
| 優先順序 | `Microsoft.VSTS.Common.Priority` | 1–4，1 最高 |
| 驗收條件 | `Microsoft.VSTS.Common.AcceptanceCriteria` | HTML；Task 等類型沒有這個欄位 |

- **估計與工時欄位**依流程而異，從流程設定取得，不要猜：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/work/processconfiguration?api-version=7.1"
  ```

  `typeFields.Effort.referenceName` 是需求層級的估計欄位（Scrum 為 `Microsoft.VSTS.Scheduling.Effort`、Agile 為 `Microsoft.VSTS.Scheduling.StoryPoints`、CMMI 為 `Microsoft.VSTS.Scheduling.Size`，自訂流程依實際回應）；`typeFields.RemainingWork.referenceName` 是 Task 的剩餘工時；`typeFields.Activity.referenceName` 是 Task 的活動類別。`requirementBacklog.workItemTypes[].name` 是需求層級的類型，`taskBacklog.workItemTypes[].name` 是 Task 層級的類型。

- **類型清單**（`value[].name`；`isDisabled` 為 `true` 的不能建立）：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wit/workitemtypes?api-version=7.1"
  ```

- **類型的欄位**（確認欄位存在；`alwaysRequired` 為 `true` 的欄位建立時必須提供）：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wit/workitemtypes/{type}/fields?api-version=7.1"
  ```

- **類型的狀態**：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wit/workitemtypes/{type}/states?api-version=7.1"
  ```

  每筆有 `name` 與 `category`（`Proposed`、`InProgress`、`Resolved`、`Completed`、`Removed`）。使用者說「改成進行中」「改成完成」時，用 `category` 找對應的狀態名稱；同一個 `category` 有多個狀態（自訂流程常見）時請使用者選。目標狀態不在清單中就不要送出。

## 規則

### 1. 建立工作項目

1. 決定專案與類型（類型名稱以類型清單為準）。使用者沒指定區域與 Iteration 時，API 會放在專案根節點，可能不會出現在任何團隊的待辦清單上。要放到某個團隊時：
   - 區域用團隊預設區域：`GET https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/{team}/_apis/work/teamsettings/teamfieldvalues?api-version=7.1` 的 `defaultValue`。
   - Iteration 用團隊 Iteration 清單（`.../{team}/_apis/work/teamsettings/iterations?api-version=7.1`）中的 `path`；要目前 Sprint 時加上 `&%24timeframe=current`。
2. 乾跑（類型在路徑中寫成 `%24` 加上 URL 編碼的名稱，例如 `%24User%20Story`、`%24Product%20Backlog%20Item`、`%24Bug`）：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -X POST -H "Content-Type: application/json-patch+json" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wit/workitems/%24User%20Story?validateOnly=true&api-version=7.1" \
     --data-binary @- <<'EOF'
   [
     {"op": "add", "path": "/fields/System.Title", "value": "使用者可以重設密碼"},
     {"op": "add", "path": "/fields/System.AreaPath", "value": "Contoso\\Web"},
     {"op": "add", "path": "/fields/System.IterationPath", "value": "Contoso\\Sprint 1"},
     {"op": "add", "path": "/fields/Microsoft.VSTS.Common.AcceptanceCriteria", "value": "<ul><li>收到重設信後 30 分鐘內有效</li></ul>"},
     {"op": "add", "path": "/relations/-", "value": {"rel": "System.LinkTypes.Hierarchy-Reverse", "url": "https://dev.azure.com/contoso/_apis/wit/workItems/101"}}
   ]
   EOF
   ```

   最後一行把新項目設為工作項目 101 的子項目（`Hierarchy-Reverse` 指向父項目）；沒有父項目就省略。
3. 依「寫入流程」確認後，拿掉 `validateOnly=true` 正式送出。回應的 `id` 是新工作項目 ID。

範例：「在 Contoso 專案建立一個 User Story：使用者可以重設密碼，放在 Feature 101 底下」

### 2. 把工作項目拆解成子 Task

1. 取得父項目（含關聯）：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/wit/workitems/{id}?%24expand=relations&api-version=7.1"
   ```

   記下 `System.TeamProject`、`System.WorkItemType`、`System.Title`、`System.AreaPath`、`System.IterationPath`。`relations` 中 `rel` 為 `System.LinkTypes.Hierarchy-Forward` 的是既有子項目（ID 是 `url` 的最後一段），用 `POST _apis/wit/workitemsbatch` 取標題，避免建立重複的 Task。
2. 從流程設定取得 Task 層級的類型名稱與 `RemainingWork`、`Activity` 欄位的參考名稱。
3. 依父項目的描述與驗收條件規劃 Task 清單（標題，選填剩餘工時、活動類別、指派對象）。每個 Task 都繼承父項目的 `System.AreaPath` 與 `System.IterationPath`，並加上指向父項目的 `Hierarchy-Reverse` 關聯：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -X POST -H "Content-Type: application/json-patch+json" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wit/workitems/%24Task?validateOnly=true&api-version=7.1" \
     --data-binary @- <<'EOF'
   [
     {"op": "add", "path": "/fields/System.Title", "value": "實作重設密碼 API"},
     {"op": "add", "path": "/fields/System.AreaPath", "value": "Contoso\\Web"},
     {"op": "add", "path": "/fields/System.IterationPath", "value": "Contoso\\Sprint 1"},
     {"op": "add", "path": "/fields/Microsoft.VSTS.Scheduling.RemainingWork", "value": 6},
     {"op": "add", "path": "/relations/-", "value": {"rel": "System.LinkTypes.Hierarchy-Reverse", "url": "https://dev.azure.com/contoso/_apis/wit/workItems/101"}}
   ]
   EOF
   ```

4. 每個 Task 都乾跑，把整份清單（標題、工時、指派、繼承的區域與 Iteration）以表格列給使用者，確認一次後逐一正式建立。中途失敗就停止，回報已建立與未建立的 Task。

範例：「把 PBI 101 拆成前端、後端、測試三個 Task」

### 3. 更新欄位、狀態、Sprint 或指派

1. 取得最新版本，記下 `rev` 與要修改欄位的目前值：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/wit/workitems/{id}?api-version=7.1"
   ```

2. 改狀態時，先用該項目 `System.WorkItemType` 的狀態清單驗證目標狀態。
3. 乾跑。第一個操作一律是 `test` `/rev`（值為步驟 1 的 `rev`），做樂觀並行控制：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -X PATCH -H "Content-Type: application/json-patch+json" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wit/workitems/{id}?validateOnly=true&api-version=7.1" \
     --data-binary @- <<'EOF'
   [
     {"op": "test", "path": "/rev", "value": 7},
     {"op": "add", "path": "/fields/System.State", "value": "Committed"},
     {"op": "add", "path": "/fields/System.AssignedTo", "value": "alex@contoso.com"},
     {"op": "add", "path": "/fields/System.IterationPath", "value": "Contoso\\Sprint 2"}
   ]
   EOF
   ```

4. 依「寫入流程」列出目前值 → 新值，確認後拿掉 `validateOnly=true` 正式送出。
5. 回應 HTTP 412，或錯誤訊息指出 `test` 操作失敗，代表別人在這段期間改過這個項目：重新執行步驟 1，把對方的變更與你的變更一起列給使用者，以新的 `rev` 重新確認；不要直接覆寫。

- **換父項目**：用 `?%24expand=relations` 取得項目，找出 `rel` 為 `System.LinkTypes.Hierarchy-Reverse` 的索引 N，在同一個主體中先 `{"op": "remove", "path": "/relations/N"}`，再 `add` 新的父項目關聯（一個項目只能有一個父項目）。

範例：「把 101 改成進行中並指派給 Alex」、「把 101 移到下一個 Sprint」

### 4. 新增留言

留言沒有乾跑機制：先把留言全文與目標工作項目（ID 與標題）列給使用者，確認一次後才送出。

```bash
curl -sS -u ":$AZURE_DEVOPS_PAT" -X POST -H "Content-Type: application/json" \
  "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wit/workItems/{id}/comments?format=markdown&api-version=7.1-preview.4" \
  -d '{"text":"API 設計已完成，請協助 **review**。"}'
```

這個資源只有 preview 版本，必須帶 `7.1-preview.4`。`format` 決定 `text` 的格式：`markdown` 或 `html`；文件把它列為必要參數，一律明確指定。留言含單引號或多行時改用 heredoc。回應的 `id` 是留言 ID。

範例：「在 101 留言說 API 設計已完成」

## 呈現結果

- **乾跑結果**：標明「這是乾跑，尚未寫入」，以表格呈現「欄位 | 目前值 | 送出後的值」；建立時「目前值」寫 *（新項目）*。拆解時改列 Task 清單：標題、剩餘工時、指派、區域、Iteration。
- **寫入完成**：每個項目顯示 ID（連結 `https://dev.azure.com/{organization}/{project}/_workitems/edit/{ID}`）、類型、標題、狀態與新的 `rev`；拆解時以表格列出所有新建的子 Task，並附上父項目連結。
- 失敗時列出 HTTP 狀態碼與 `message`，說明哪些已寫入、哪些沒有。
