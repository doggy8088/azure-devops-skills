---
name: boards-team-active-work
description: 以 Azure DevOps REST API 取得整個團隊進行中的待辦項目，呈現相依關係、優先順序與 Sprint 歸屬：先列阻擋其他項目的工作，再列高優先與本 Sprint 到期的項目，並附父子階層、Iteration 名稱與結束日期。當使用者想掌握團隊目前進度、阻擋事項或本 Sprint 工作時使用。
compatibility: 需要 curl、可連線至 dev.azure.com 的網路，以及 AZURE_DEVOPS_ORG 與 AZURE_DEVOPS_PAT 環境變數。
---

# 團隊進行中工作與相依關係

這個 Skill 取得團隊需求層級（Requirements）的待辦清單，篩選出整個團隊**進行中**的項目，補上父子與相依資訊，再依阻擋狀態、優先順序與 Sprint 歸屬呈現。

## 專案選擇

- 使用者有提供專案名稱（例如「Contoso 專案」）就直接使用，不呼叫專案清單 API。
- 沒有提供時，請使用者提供一次。
- 仍未提供時，列出專案讓使用者選擇：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/projects?%24top=500&api-version=7.1"
  ```

- 使用者提供或選定專案之前不要繼續。

## 團隊選擇

- 使用者有提供團隊名稱就直接使用，不呼叫團隊清單 API。
- 沒有提供時，請使用者提供一次。
- 仍未提供時，列出所選專案的團隊讓使用者選擇：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/projects/{project}/teams?%24top=500&api-version=7.1"
  ```

- 使用者提供或選定團隊之前不要繼續。

專案清單的回應標頭（`curl -D -` 可看到）有 `x-ms-continuationtoken` 時，加上 `&continuationToken={token}` 取下一頁；團隊清單則以 `%24skip` 分頁。

## 連線與驗證

一律用 `curl` 直接呼叫 Azure DevOps REST API，每個指令都自帶驗證參數。

- **組織**：讀取環境變數 `AZURE_DEVOPS_ORG`（組織名稱，例如 `contoso`）。未設定時詢問使用者一次；使用者給的是網址時，從 `https://dev.azure.com/{org}` 或 `https://{org}.visualstudio.com` 取出組織名稱。
- **權杖**：PAT 以 Basic 驗證送出：`-u ":$AZURE_DEVOPS_PAT"`；只設定了 `AZURE_DEVOPS_EXT_PAT`（Azure CLI azure-devops 擴充套件使用的變數）時，改用它。改用 Microsoft Entra ID 權杖時，換成 `-H "Authorization: Bearer $AZURE_DEVOPS_TOKEN"`。都未設定時，請使用者在自己的終端機設定 `AZURE_DEVOPS_PAT` 後重試，並停止。權杖只從環境變數讀取：不要請使用者把權杖貼進對話，也不要印出或寫入檔案。
- **PAT 權限範圍**：Work Items (Read)、Project and Team (Read)。
- **編碼**：路徑中的專案與團隊名稱要 URL 編碼（空白寫成 `%20`）；查詢字串中的 `$` 寫成 `%24`（例如 `%24top`），避免被 shell 展開。
- **錯誤**：HTTP 401、403、203 或回應是 HTML 登入頁，代表權杖無效、過期或權限不足，回報後停止；HTTP 404 代表組織、專案或團隊名稱有誤。
- **Windows PowerShell**：呼叫 `curl.exe`（而非 `curl` 別名），環境變數寫成 `$env:AZURE_DEVOPS_PAT`；PowerShell 不支援 `<<'EOF'`，先把 JSON 主體寫入暫存檔，再以 `--data-binary "@body.json"` 送出。

## 步驟

1. 依上方規則決定專案與團隊。

2. 取得團隊的所有 Iteration（一次呼叫同時提供目前 Sprint 與各 Iteration 的結束日期）：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/{team}/_apis/work/teamsettings/iterations?api-version=7.1"
   ```

   `attributes.timeFrame` 為 `current` 的就是目前 Sprint。團隊還沒排下一個 Sprint 時，已結束的最後一個 Sprint 仍會標成 `current`：`finishDate` 早於今天時照樣使用它，但在呈現時註明「目前 Sprint 已於 {日期} 結束」。建立 `path` → `attributes.finishDate` 的對照表；`path` 的格式與工作項目的 `System.IterationPath` 相同（例如 `Contoso\Sprint 12`），可直接比對。

3. 取得團隊需求層級待辦清單的工作項目 ID：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/{team}/_apis/work/backlogs/Microsoft.RequirementCategory/workItems?api-version=7.1"
   ```

   ID 在 `workItems[].target.id`。清單為空時，直接進入步驟 5 的空結果訊息。

4. 用 WIQL 在伺服器端篩選出進行中的項目（整個團隊，不限指派對象）。把步驟 3 的 ID 填入 `IN (...)`（ID 超過 1,000 個時分批查詢）：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -X POST -H "Content-Type: application/json" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wit/wiql?api-version=7.1" \
     --data-binary @- <<'EOF'
   {"query": "SELECT [System.Id] FROM WorkItems WHERE [System.Id] IN (101, 102, 103) AND [System.State] IN ('Active', 'Committed', 'Doing')"}
   EOF
   ```

   「進行中」對應各內建流程的進行中狀態：Agile 與 CMMI 為 `Active`、Scrum 為 `Committed`、Basic 為 `Doing`。專案使用自訂流程時，改用該流程的進行中狀態名稱。

5. 回應的 `workItems` 為空時，說明這個團隊沒有進行中的待辦項目，然後停止。

6. 以批次呼叫取得篩選後項目的欄位與關聯（每次最多 200 個 ID）：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -X POST -H "Content-Type: application/json" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wit/workitemsbatch?api-version=7.1" \
     -d '{"ids":[101,102],"$expand":"Relations","errorPolicy":"Omit"}'
   ```

   `$expand` 不能與 `fields` 同時指定，所以這次會回傳所有欄位；只取用 `System.Id`、`System.Title`、`System.State`、`System.AssignedTo`、`System.WorkItemType`、`System.ChangedDate`、`System.IterationPath`、`Microsoft.VSTS.Common.Priority`、`System.Parent`。

7. 從每個項目的 `relations` 收集下列 ID（ID 是 `url` 的最後一段，`attributes.name` 是連結的顯示名稱）：
   - **父項目**：`System.LinkTypes.Hierarchy-Reverse`（Parent）
   - **子項目**：`System.LinkTypes.Hierarchy-Forward`（Child）
   - **被這個項目阻擋的項目**：`System.LinkTypes.Dependency-Forward`（Successor，對方要等這個項目完成）
   - **這個項目相依的項目**：`System.LinkTypes.Dependency-Reverse`（Predecessor，這個項目要等對方完成）
   - **相關項目**：`System.LinkTypes.Related`

8. 排除已取得的 ID 後，用一次批次呼叫取得上述所有項目的 `System.Id`、`System.Title`、`System.State`、`System.IterationPath`：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -X POST -H "Content-Type: application/json" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wit/workitemsbatch?api-version=7.1" \
     -d '{"ids":[201,202],"fields":["System.Id","System.Title","System.State","System.IterationPath"],"errorPolicy":"Omit"}'
   ```

9. 判斷每個項目的：
   - **是否阻擋其他項目**：有任何 `System.LinkTypes.Dependency-Forward`（Successor）關聯
   - **是否在目前 Sprint**：`System.IterationPath` 等於步驟 2 的目前 Iteration `path`
   - **優先順序**：`Microsoft.VSTS.Common.Priority`

10. 依下方格式呈現結果。

## 呈現結果

**先顯示目前 Sprint：**

> **目前 Sprint：** {Iteration 名稱}（{Iteration 結束日期，格式 YYYY/MM/DD} 結束）

團隊沒有目前 Sprint 時，改顯示「團隊目前沒有進行中的 Sprint」，並把第 3 組視為空組。

---

**接著依以下順序分組，每個項目只歸入第一個符合的組，組內依 Priority 由小到大排序（Priority 1 在最前）：**

1. **阻擋其他項目的項目**
2. **高優先項目**（Priority 1 或 2）
3. **本 Sprint 到期的項目**（Iteration 為目前 Sprint）
4. **其他項目**

每個項目顯示：

---

### [{ID}](https://dev.azure.com/{organization}/{project}/_workitems/edit/{ID}) — {標題}

{阻擋其他項目時顯示：**⚠️ 正在阻擋其他工作項目**}

| 欄位 | 值 |
|---|---|
| **狀態** | {System.State} |
| **優先順序** | {Microsoft.VSTS.Common.Priority} |
| **變更日期** | {System.ChangedDate，格式 YYYY/MM/DD} |
| **Iteration** | {System.IterationPath 的最後一段} |
| **Iteration 結束日期** | {步驟 2 對照表中該 Iteration 的 finishDate；未設定日期或團隊未訂閱該 Iteration 時顯示 *（未知）*} |
| **工作項目類型** | {System.WorkItemType} |

**父項目：**
- 有父項目：[`{ParentID}`](https://dev.azure.com/{organization}/{project}/_workitems/edit/{ParentID}) {父項目標題}
- 沒有父項目：*（無）*

**子項目：**

有子項目時以表格顯示：

| ID | 標題 | 狀態 |
|----|------|------|
| [{ChildID}](https://dev.azure.com/{organization}/{project}/_workitems/edit/{ChildID}) | {子項目標題} | {子項目狀態} |

沒有子項目時顯示：*（無）*

**相依關係：**

有 Successor 關聯（阻擋其他項目）時顯示：

**⚠️ 正在阻擋：**
- [{DependentID}](https://dev.azure.com/{organization}/{project}/_workitems/edit/{DependentID}) {被阻擋項目標題}（{被阻擋項目狀態}）

有 Predecessor 關聯（相依其他項目）時顯示：

**相依於：**
- [{DependencyID}](https://dev.azure.com/{organization}/{project}/_workitems/edit/{DependencyID}) {相依項目標題}（{相依項目狀態}）

兩者都沒有時顯示：*（無）*

---

列完所有項目後，顯示摘要：

> **摘要：**
> - **團隊進行中項目總數：** {N}
> - **阻擋其他項目的項目：** {M}
> - **高優先項目（1–2）：** {P}
> - **本 Sprint 到期的項目：** {Q}
>
> 團隊：**{團隊名稱}** | 專案：**{專案名稱}**
