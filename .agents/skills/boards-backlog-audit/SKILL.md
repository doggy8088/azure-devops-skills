---
name: boards-backlog-audit
description: 以 Azure DevOps REST API 依 Microsoft 的 Azure Boards 最佳實務，健檢團隊需求層級的待辦清單（Product Backlog Item、User Story、Requirement、Issue）：找出缺少估計、缺少驗收條件、沒有父項目（Feature／Epic）、進行中卻未指派、進行中卻沒有連結 branch／commit／PR 的項目，以及未來 Sprint 不足與待辦清單過大等問題，附修正建議。只讀取、不修改。當使用者要檢查 backlog 品質、做 backlog refinement 前的健檢或問待辦清單有哪些問題時使用；只要列出進行中工作改用 `boards-team-active-work`。
compatibility: 需要 curl、可連線至 dev.azure.com 的網路，以及 AZURE_DEVOPS_ORG 與 AZURE_DEVOPS_PAT 環境變數。
---

# 待辦清單健檢

這個 Skill 讀取一個團隊需求層級（Requirements）的待辦清單，逐項對照 Microsoft Learn 的建議，列出問題與修正方式。**只讀取、不修改**：要修正欄位時改用 `boards-work-item-write`（每筆都需使用者確認），要補 Sprint 時改用 `work-iterations`。

依據的 Microsoft Learn 文件：

- [Best practices for Agile product management](https://learn.microsoft.com/en-us/azure/devops/boards/best-practices-agile-project-management)
- [Create and manage your product backlog](https://learn.microsoft.com/en-us/azure/devops/boards/backlogs/create-your-backlog)
- [Drive Git development from work items](https://learn.microsoft.com/en-us/azure/devops/boards/backlogs/connect-work-items-to-git-dev-ops)

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
- **權杖**：PAT 以 Basic 驗證送出：`-u ":$AZURE_DEVOPS_PAT"`；只設定了 `AZURE_DEVOPS_EXT_PAT`（Azure CLI azure-devops 擴充套件使用的變數）時，改用它。改用 Microsoft Entra ID 權杖時，換成 `-H "Authorization: Bearer $AZURE_DEVOPS_TOKEN"`。都未設定時，請使用者在自己的終端機設定 `AZURE_DEVOPS_PAT` 後重試，並停止。權杖只從環境變數讀取：不要請使用者把權杖貼進對話，也不要印出或寫入檔案。
- **PAT 權限範圍**：Work Items (Read)、Project and Team (Read)。
- **編碼**：路徑中的專案、團隊與工作項目類型名稱要 URL 編碼（空白寫成 `%20`）；查詢字串中的 `$` 寫成 `%24`（例如 `%24expand`），避免被 shell 展開。
- **錯誤**：HTTP 401、403、203 或回應是 HTML 登入頁，代表權杖無效、過期或權限不足，回報後停止；HTTP 404 代表組織、專案或團隊名稱有誤。
- **Windows PowerShell**：呼叫 `curl.exe`（而非 `curl` 別名），環境變數寫成 `$env:AZURE_DEVOPS_PAT`；PowerShell 不支援 `<<'EOF'`，先把 JSON 主體寫入暫存檔，再以 `--data-binary "@body.json"` 送出。

## 步驟

1. 依上方規則決定專案與團隊。

2. 取得流程設定。估計欄位**一律**用 `typeFields.Effort.referenceName`（例如 `Microsoft.VSTS.Scheduling.Effort`、`Microsoft.VSTS.Scheduling.StoryPoints`、`Microsoft.VSTS.Scheduling.Size`），不要依流程名稱猜：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/work/processconfiguration?api-version=7.1"
   ```

   `portfolioBacklogs[].workItemTypes[].name` 是 Feature、Epic 等組合層級的類型。

3. 取得待辦清單的工作項目 ID（依待辦清單順序）：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/{team}/_apis/work/backlogs/Microsoft.RequirementCategory/workItems?api-version=7.1"
   ```

   ID 在 `workItems[].target.id`。清單為空時，說明這個團隊的待辦清單沒有項目，只執行步驟 7 的 Iteration 檢查。

4. 以批次呼叫取得所有欄位與關聯（每次最多 200 個 ID，超過就分批）。`$expand` 不能與 `fields` 同時指定，所以不帶 `fields`：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -X POST -H "Content-Type: application/json" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wit/workitemsbatch?api-version=7.1" \
     -d '{"ids":[101,102],"$expand":"Relations","errorPolicy":"Omit"}'
   ```

   值為空的欄位不會出現在 `fields` 中。

5. 對回應中出現的每個 `System.WorkItemType`（例如 Bug 也在待辦清單上時會有兩種）各取一次狀態與欄位：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wit/workitemtypes/{type}/states?api-version=7.1"
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wit/workitemtypes/{type}/fields?api-version=7.1"
   ```

   **進行中**的狀態是 `category` 為 `InProgress` 的 `name`（不要寫死 `Active`、`Committed` 等名稱）。欄位清單 `value[].referenceName` 用來判斷這個類型有沒有估計欄位與 `Microsoft.VSTS.Common.AcceptanceCriteria`；沒有該欄位的類型（例如自訂流程的 Issue）跳過那一項檢查，並在報告中註明。

6. 逐項檢查（「進行中」＝`System.State` 屬於步驟 5 的進行中狀態）：

   | 檢查 | 判斷方式 |
   |---|---|
   | 缺少估計 | 類型有估計欄位，但 `fields` 中沒有該欄位 |
   | 缺少驗收條件 | 類型有 `Microsoft.VSTS.Common.AcceptanceCriteria`，但沒有值，或去掉 HTML 標籤與空白後是空字串 |
   | 沒有父項目 | `fields` 中沒有 `System.Parent` |
   | 進行中但未指派 | 進行中，且沒有 `System.AssignedTo` |
   | 進行中但沒有開發連結 | 進行中，且 `relations[]` 中沒有 `rel` 為 `ArtifactLink`、`url` 開頭是下列之一的關聯 |
   | 待辦清單過大 | 步驟 3 的項目數超過 999 |

   開發連結依 `url` 前綴判斷，不要依 `attributes.name` 這類顯示名稱：`vstfs:///Git/Commit/`（commit）、`vstfs:///Git/Ref/`（branch）、`vstfs:///Git/PullRequestId/`（PR）、`vstfs:///GitHub/Commit/`、`vstfs:///GitHub/PullRequest/`、`vstfs:///GitHub/Branch/`。`vstfs:///Build/Build/`（Integrated in build）不算，因為它不代表有人在開發。

   有 `System.Parent` 時，可選擇再以一次批次呼叫取父項目的 `System.WorkItemType`，父項目不是組合層級類型（例如掛在另一個 PBI 底下）時另外列出。

7. 取得團隊的 Iteration：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/{team}/_apis/work/teamsettings/iterations?api-version=7.1"
   ```

   計算 `attributes.timeFrame` 為 `future` 的數量，少於 6 個就列為問題。`timeFrame` 為 `current` 的 Iteration，若 `attributes.finishDate` 早於今天，代表團隊沒有涵蓋今天的 Sprint，也列為問題。`startDate`、`finishDate` 可能是 `null`，這類 Iteration 另外註明「未設定日期」。

8. 依下方格式呈現。

## 修正建議

| 檢查 | 建議 |
|---|---|
| 缺少估計 | 在 backlog refinement 時用相對大小估計；Velocity 與 Forecast 都需要估計。只想依數量預測時，Learn 建議每個項目填 1。 |
| 缺少驗收條件 | 每個項目都要有清楚的驗收條件與完成定義，讓團隊能估計與測試。 |
| 沒有父項目 | 把未歸屬的需求對應到適當的 Feature（待辦清單的 **Mapping** 窗格，或以 `boards-work-item-write` 設定父項目）。 |
| 進行中但未指派 | 每個項目只指派給一位負責人；多人分工時拆成多個項目。 |
| 進行中但沒有開發連結 | 從工作項目的 **Development** 區塊建立 branch，或在 commit 與 PR 中引用工作項目 ID，讓變更可以追溯。 |
| 未來 Sprint 不足 | Learn 建議至少建立六個 Iteration，涵蓋未來三到六個月；以 `work-iterations` 依既有節奏建立並指派給團隊。 |
| 待辦清單過大 | create-your-backlog 文件說明每個待辦清單最多顯示 999 個項目（Services 的物件上限頁面寫 10,000），建議新增團隊並移出部分項目，或關閉過時項目。 |

## 呈現結果

1. **標題**：專案、團隊、待辦清單項目數、估計欄位的參考名稱、檢查時間（使用者當地時區）。
2. **摘要表**：

   | 檢查 | 問題數 | 狀態 |
   |---|---|---|
   | 缺少估計 | 12 | ⚠️ |
   | 未來 Sprint | 3 個（建議 ≥ 6） | ❌ |

   沒有問題標 ✅、有問題標 ⚠️、Iteration 不足或待辦清單過大標 ❌；跳過的檢查標 ➖ 並註明原因（例如「Issue 類型沒有驗收條件欄位」）。
3. **各項明細**：每個有問題的檢查一個小節，先寫修正建議，再以表格列出項目：ID（連結 `https://dev.azure.com/{organization}/{project}/_workitems/edit/{ID}`）、類型、標題、狀態、指派給。依步驟 3 的待辦清單順序排列（批次回應會依 ID 排序）；超過 20 筆時只列前 20 筆並寫出總數，詢問是否列出其餘項目。
4. **Iteration**：列出目前與未來的 Iteration 名稱與起訖日期（`YYYY/MM/DD`）。
5. 最後一行：共檢查 **{N}** 個項目，**{M}** 個項目至少有一個問題。
