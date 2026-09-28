---
name: boards-sprint-metrics
description: 以 Azure DevOps Analytics（OData）產生團隊最近幾個已結束 Sprint 的交付指標：每個 Sprint 的 Velocity（完成的 Story Points／Effort／Size 總和）、Throughput（完成項目數）、Carry-over（Sprint 結束時仍未完成的項目數），以及完成項目的 Cycle time 與 Lead time（平均、中位數、P85）。只讀取。當使用者問團隊速度、交付量、Sprint 完成率、週期時間或要準備 Sprint Review／回顧的數據時使用；要看目前 Sprint 進行中的工作改用 `boards-team-active-work`。
compatibility: 需要 curl、可連線至 dev.azure.com 與 analytics.dev.azure.com 的網路，以及 AZURE_DEVOPS_ORG 與 AZURE_DEVOPS_PAT 環境變數。
---

# Sprint 交付指標

這個 Skill 以 Analytics 在伺服器端彙總資料，產生團隊最近 N 個（預設 6 個）**已結束** Sprint 的指標。只讀取，不修改任何資料。

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

## 連線與驗證

一律用 `curl` 直接呼叫 Azure DevOps REST API，每個指令都自帶驗證參數。

- **組織**：讀取環境變數 `AZURE_DEVOPS_ORG`（組織名稱，例如 `contoso`）。未設定時詢問使用者一次；使用者給的是網址時，從 `https://dev.azure.com/{org}` 或 `https://{org}.visualstudio.com` 取出組織名稱。
- **主機**：除了 `dev.azure.com`，Analytics OData 的主機是 `analytics.dev.azure.com`。
- **權杖**：PAT 以 Basic 驗證送出：`-u ":$AZURE_DEVOPS_PAT"`；只設定了 `AZURE_DEVOPS_EXT_PAT`（Azure CLI azure-devops 擴充套件使用的變數）時，改用它。改用 Microsoft Entra ID 權杖時，換成 `-H "Authorization: Bearer $AZURE_DEVOPS_TOKEN"`。都未設定時，請使用者在自己的終端機設定 `AZURE_DEVOPS_PAT` 後重試，並停止。權杖只從環境變數讀取：不要請使用者把權杖貼進對話，也不要印出或寫入檔案。
- **PAT 權限範圍**：Analytics (Read)；讀取流程設定與團隊設定另需 Work Items (Read)、Project and Team (Read)。
- **編碼**：路徑中的專案與團隊名稱要 URL 編碼（空白寫成 `%20`）；查詢字串中的 `$` 寫成 `%24`（例如 `%24top`），避免被 shell 展開。OData 參數改用 `-G --data-urlencode` 交給 curl 編碼（見下方）。
- **錯誤**：HTTP 401、403、203 或回應是 HTML 登入頁，代表權杖無效、過期或權限不足，回報後停止；HTTP 404 代表組織或專案名稱有誤。Analytics 回 401 或 403 時，也可能是 PAT 缺少 Analytics (Read) 或使用者沒有「View analytics」權限。HTTP 410 代表這個 OData 版本已不提供。HTTP 400 的 `error.message` 會指出問題，例如 `VS403522` 代表屬性在這個專案不存在（常見於估計欄位，見步驟 2）。
- **Windows PowerShell**：呼叫 `curl.exe`（而非 `curl` 別名），環境變數寫成 `$env:AZURE_DEVOPS_PAT`；PowerShell 不支援 `<<'EOF'`，先把 JSON 主體寫入暫存檔，再以 `--data-binary "@body.json"` 送出。

## Analytics 查詢規則

- 基底網址：`https://analytics.dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_odata/v4.0-preview/`。`v4.0-preview` 回 404 或 410 時依序改用 `v3.0-preview`、`v2.0`（本 Skill 的查詢在這些版本語法相同）。
- 以 `curl -G` 搭配 `--data-urlencode` 傳 OData 參數；參數寫在雙引號中，`$filter`、`$apply`、`$count` 的 `$` 寫成 `\$`（PowerShell 改用反引號跳脫：`` `$filter ``）。
- **同一個查詢不要同時使用 `$filter` 與 `$apply`**：要彙總就把條件寫進 `$apply=filter(...)/groupby(...)`；要逐筆資料就用 `$filter` 加 `$select`。能在伺服器端彙總的一律用 `$apply`，不要把整批工作項目拉回本機計算。
- 字串用單引號，名稱中的單引號寫成兩個（`'O''Brien Team'`）；GUID（例如 `IterationSK`）不加引號。
- 以團隊篩選：`Teams/any(t: t/TeamName eq 'Contoso Team')`（依團隊的區域路徑判斷）。
- 回應有 `@odata.nextLink` 時代表還有下一頁，照該網址再取一次。`@vsts.warnings` 中的 `VS403507` 只是提醒查詢沒有 `$select` 或 `$apply`，不影響結果。
- 日期是組織時區的時間（例如 `2026-01-16T23:59:59.999+08:00`）；`...DateSK` 是 `YYYYMMDD` 整數。

## 步驟

1. 依上方規則決定專案與團隊。N 預設為 6，使用者指定時照用。

2. 決定工作項目類型與估計屬性：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/work/processconfiguration?api-version=7.1"
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/{team}/_apis/work/teamsettings?api-version=7.1"
   ```

   - 類型：`requirementBacklog.workItemTypes[].name`；團隊設定的 `bugsBehavior` 為 `asRequirements` 時，再加上 `bugWorkItems.workItemTypes[].name`。
   - 估計屬性：把 `typeFields.Effort.referenceName` 對應成 Analytics 屬性：`Microsoft.VSTS.Scheduling.StoryPoints` → `StoryPoints`、`Microsoft.VSTS.Scheduling.Effort` → `Effort`、`Microsoft.VSTS.Scheduling.Size` → `Size`；自訂欄位把 `.` 換成 `_`（`Custom.Estimate` → `Custom_Estimate`）。查詢回 `VS403522` 時，拿掉估計屬性，只報 Throughput，並說明原因。

3. 取得團隊最近 N 個已結束的 Iteration。以 Analytics 的 `IsEnded` 判斷，不要用 REST 的 `timeFrame`（沒有新 Sprint 時，REST 可能把已結束的最後一個 Sprint 標成 `current`）：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -G \
     "https://analytics.dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_odata/v4.0-preview/Iterations" \
     --data-urlencode "\$filter=Teams/any(t: t/TeamName eq 'Contoso Team') and IsEnded eq true" \
     --data-urlencode "\$select=IterationSK,IterationPath,IterationName,StartDate,EndDate" \
     --data-urlencode "\$orderby=EndDate desc" \
     --data-urlencode "\$top=6"
   ```

   沒有任何結果時，說明這個團隊沒有已結束的 Iteration（或 Iteration 沒有設定日期），然後停止。

4. **Velocity 與 Throughput**（一次查詢；`IterationSK in (...)` 填入步驟 3 的所有 `IterationSK`）：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -G \
     "https://analytics.dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_odata/v4.0-preview/WorkItems" \
     --data-urlencode "\$apply=filter(Teams/any(t: t/TeamName eq 'Contoso Team') and StateCategory eq 'Completed' and WorkItemType in ('Product Backlog Item','Bug') and IterationSK in ({iterationSK1},{iterationSK2}))/groupby((Iteration/IterationPath, Iteration/EndDate), aggregate(\$count as Count, Effort with sum as TotalEstimate))"
   ```

   `Count` 是 Throughput，`TotalEstimate` 是 Velocity。依工作項目**目前**的 Iteration 歸屬計算：完成後才被移到其他 Sprint 的項目，會算在新的 Sprint。沒有出現在回應中的 Iteration 代表完成 0 個；`TotalEstimate` 缺少或為 `null` 代表完成的項目都沒有估計。

5. **Cycle time 與 Lead time**（逐筆資料，由你計算統計值；日期範圍從最早 Iteration 的 `StartDate` 到最新 Iteration 的 `EndDate`）：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -G \
     "https://analytics.dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_odata/v4.0-preview/WorkItems" \
     --data-urlencode "\$filter=Teams/any(t: t/TeamName eq 'Contoso Team') and StateCategory eq 'Completed' and WorkItemType in ('Product Backlog Item','Bug') and CompletedDateSK ge 20260105 and CompletedDateSK le 20260327" \
     --data-urlencode "\$select=WorkItemId,Title,WorkItemType,CycleTimeDays,LeadTimeDays,CompletedDateSK"
   ```

   `CycleTimeDays` 是進入進行中到完成的天數，`LeadTimeDays` 是建立到完成的天數。分別計算平均、中位數與 P85（由小到大排序，取第 ⌈0.85 × n⌉ 筆）；值為 `null` 的項目（例如沒經過進行中就直接完成）排除並註明筆數。分布偏斜（平均遠大於中位數）時，列出最長的 3 筆作為離群值。

6. **Carry-over**：每個 Iteration 結束當天，仍在該 Iteration 且未完成的項目數。用 `WorkItemSnapshot` 取結束日（`EndDate` 的日期部分寫成 `DateSK`）的每日快照：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -G \
     "https://analytics.dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_odata/v4.0-preview/WorkItemSnapshot" \
     --data-urlencode "\$apply=filter(Teams/any(t: t/TeamName eq 'Contoso Team') and WorkItemType in ('Product Backlog Item','Bug') and StateCategory ne 'Completed' and StateCategory ne 'Removed' and ((IterationSK eq {iterationSK1} and DateSK eq 20260116) or (IterationSK eq {iterationSK2} and DateSK eq 20260130)))/groupby((Iteration/IterationPath, DateSK), aggregate(\$count as Count))"
   ```

   沒有出現在回應中的 Iteration 代表 Carry-over 為 0。使用者要看某個 Iteration 延後了哪些項目時，改用 `$filter`（同樣的條件，只留一組 `IterationSK` 與 `DateSK`）加 `\$select=WorkItemId,Title,State`。完成率＝Throughput ÷（Throughput ＋ Carry-over）。

7. 依下方格式呈現。

## 呈現結果

1. **標題**：專案、團隊、涵蓋的 Iteration 數與日期範圍、估計屬性（例如 `StoryPoints`）與納入的工作項目類型。
2. **每個 Sprint 的指標**（由舊到新）：

   | Sprint | 期間 | Velocity | Throughput | Carry-over | 完成率 |
   |---|---|---|---|---|---|
   | Sprint 12 | 2026/01/05–2026/01/16 | 21 | 8 | 2 | 80% |

   最後一列顯示平均。沒有估計時 Velocity 欄顯示 *（無估計）*。
3. **Cycle time 與 Lead time**：平均、中位數、P85（天，取到小數一位）與樣本數；有離群值時列出 ID（連結 `https://dev.azure.com/{organization}/{project}/_workitems/edit/{ID}`）與標題。
4. **觀察**：用 2–4 點說明趨勢（例如 Velocity 波動、Carry-over 偏高、估計不足），標明這是 AI 產生的解讀，數字以上表為準。
5. 日期顯示 `YYYY/MM/DD`；Analytics 的日期已是組織時區，直接取日期部分。
