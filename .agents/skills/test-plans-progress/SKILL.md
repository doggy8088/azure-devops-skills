---
name: test-plans-progress
description: 以 Azure DevOps REST API 產生 Azure Test Plans 手動測試的進度報告：列出進行中的測試計畫，依測試套件統計測試點的通過、失敗、阻擋與未執行數量、執行率與通過率，並列出失敗或被阻擋的測試案例與負責測試人員；只讀取、不修改。當使用者問起測試計畫進度、test plan 或 test suite 的執行狀況、哪些手動測試還沒跑或失敗時使用；要看某次建置或 pipeline 的自動化測試結果改用 `pipelines-test-results`。
compatibility: 需要 curl、可連線至 dev.azure.com 的網路，以及 AZURE_DEVOPS_ORG 與 AZURE_DEVOPS_PAT 環境變數。
---

# 測試計畫進度

這個 Skill 在**專案**的範圍內讀取 Azure Test Plans 的測試計畫、測試套件與測試點（test point，也就是「測試案例 × 組態」的執行單位），彙總手動測試的進度。它不執行測試、不修改結果；某次建置的自動化測試結果改用 `pipelines-test-results`。

## 專案選擇

- 使用者有提供專案名稱（例如「Contoso 專案」）就直接使用，不呼叫專案清單 API。
- 沒有提供時，請使用者提供一次。
- 仍未提供時，列出專案讓使用者選擇：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/projects?%24top=500&api-version=7.1"
  ```

- 使用者提供或選定專案之前不要繼續。

## 測試計畫選擇

- 使用者有提供測試計畫 ID 或名稱時，只報告那些計畫；給的是名稱時，用步驟 1 的清單比對 `name`，找到多個時請使用者選一個。
- 沒有指定時，報告步驟 1 列出的所有進行中計畫；超過 10 個時先列出清單，請使用者挑選或確認全部。

## 連線與驗證

一律用 `curl` 直接呼叫 Azure DevOps REST API，每個指令都自帶驗證參數。

- **組織**：讀取環境變數 `AZURE_DEVOPS_ORG`（組織名稱，例如 `contoso`）。未設定時詢問使用者一次；使用者給的是網址時，從 `https://dev.azure.com/{org}` 或 `https://{org}.visualstudio.com` 取出組織名稱。
- **權杖**：PAT 以 Basic 驗證送出：`-u ":$AZURE_DEVOPS_PAT"`；只設定了 `AZURE_DEVOPS_EXT_PAT`（Azure CLI azure-devops 擴充套件使用的變數）時，改用它。改用 Microsoft Entra ID 權杖時，換成 `-H "Authorization: Bearer $AZURE_DEVOPS_TOKEN"`。都未設定時，請使用者在自己的終端機設定 `AZURE_DEVOPS_PAT` 後重試，並停止。權杖只從環境變數讀取：不要請使用者把權杖貼進對話，也不要印出或寫入檔案。
- **PAT 權限範圍**：Test Management (Read)；列出專案另需 Project and Team (Read)。
- **編碼**：路徑中的專案名稱要 URL 編碼（空白寫成 `%20`）；查詢字串中的 `$` 寫成 `%24`（例如 `%24top`），避免被 shell 展開。
- **錯誤**：HTTP 401、403、203 或回應是 HTML 登入頁，代表權杖無效、過期或權限不足，回報後停止；HTTP 404 代表組織、專案、測試計畫或測試套件 ID 有誤。
- **Windows PowerShell**：呼叫 `curl.exe`（而非 `curl` 別名），環境變數寫成 `$env:AZURE_DEVOPS_PAT`；PowerShell 不支援 `<<'EOF'`，先把 JSON 主體寫入暫存檔，再以 `--data-binary "@body.json"` 送出。

## 結果分類

測試點的 `results.outcome` 依 [Test Point - Get Points List](https://learn.microsoft.com/en-us/rest/api/azure/devops/testplan/test-point/get-points-list) 的 `Outcome` 列舉分類：

| 分類 | `results.outcome` |
|---|---|
| 通過 | `passed` |
| 失敗 | `failed`、`timeout`、`aborted`、`error` |
| 阻擋 | `blocked` |
| 不適用 | `notApplicable` |
| 進行中 | `inProgress`、`paused` |
| 未執行 | `unspecified`、`none`、`notExecuted` |
| 其他 | `inconclusive`、`warning`、`notImpacted` 與其他值 |

- 從未執行過的測試點回傳 `outcome` 為 `unspecified`、`results.state` 為 `ready`，且沒有 `lastTestRunId`（REST 文件把 `unspecified` 描述為更新時保留原值用，實際讀取時代表尚無結果）。
- `results.state`：`ready` 需要執行、`completed` 已通過不需重跑、`inProgress` 執行中、`notReady` 需要執行但無法執行。
- **執行率** = （總數 − 未執行）÷ 總數；**通過率** = 通過 ÷ 總數，與網頁 **Test Plans** > **Progress Report** 的 Passed% 算法一致。阻擋、不適用與暫停會計入執行率，但不計入通過或失敗，所以通過率加失敗率可能小於執行率（見 [Progress report](https://learn.microsoft.com/en-us/azure/devops/test/progress-report)）。
- `isAutomated` 為 `true` 的測試點是關聯自動化測試的測試案例，照樣計入，並在結果中註明數量。

## 步驟

1. **列出進行中的測試計畫**：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -D - \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/testplan/plans?filterActivePlans=true&includePlanDetails=true&api-version=7.1"
   ```

   - 記下 `id`、`name`、`state`、`iteration`、`areaPath`、`startDate`、`endDate`、`owner.displayName`、`rootSuite.id`。
   - `-D -` 用來看回應標頭：有 `x-ms-continuationtoken` 時，加上 `&continuationToken={token}` 取下一頁，直到沒有這個標頭。
   - 沒有任何計畫時，明確說明這個專案沒有進行中的測試計畫；使用者要包含非進行中的計畫時，拿掉 `filterActivePlans=true`。

2. **取得套件階層**：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -D - \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/testplan/Plans/{planId}/suites?asTreeView=true&api-version=7.1"
   ```

   - 回應的第一層是根套件（與計畫同名），子套件在 `children[]` 內逐層巢狀；每個套件有 `id`、`name`、`suiteType`（`staticTestSuite`、`requirementTestSuite`、`dynamicTestSuite`）與 `hasChildren`。
   - 不帶 `asTreeView` 時回傳平面清單，每個套件以 `parentSuite.id` 指向上層，適合直接建立對照表。
   - 回應標頭有 `x-ms-continuationtoken` 時同樣以 `continuationToken` 取下一頁。

3. **取得整個計畫的測試點**。以根套件 ID 搭配 `isRecursive=true` 一次取回所有子套件的測試點，不需要逐一呼叫每個套件：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -D - \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/testplan/Plans/{planId}/Suites/{rootSuiteId}/TestPoint?includePointDetails=true&isRecursive=true&api-version=7.1"
   ```

   - 每頁最多 200 筆；回應標頭有 `x-ms-continuationtoken` 時，加上 `&continuationToken={token}` 取下一頁，直到沒有這個標頭。
   - 每個測試點使用：`testSuite.id` 與 `testSuite.name`（所屬套件）、`testCaseReference.id` 與 `testCaseReference.name`、`configuration.name`、`tester.displayName`、`isActive`、`isAutomated`，以及 `results` 下的 `outcome`、`state`、`lastTestRunId`、`lastRunBuildNumber`、`lastResultDetails.dateCompleted`。
   - `lastResultDetails.dateCompleted` 為 `0001-01-01T00:00:00` 或沒有 `lastTestRunId` 時，表示沒有執行紀錄；`lastRunBuildNumber` 為空或 `0` 時表示沒有關聯建置。
   - 使用者只要某個套件時，把 `{rootSuiteId}` 換成該套件 ID。

4. **彙總**：依「結果分類」把測試點依 `testSuite.id` 計數；上層套件的小計包含所有子孫套件（用步驟 2 的階層加總），計畫總計等於根套件小計。一個測試案例在多個組態下會有多個測試點，計數以測試點為準（與網頁一致）。

5. **列出需要處理的測試案例**：分類為失敗、阻擋的測試點，另外把進行中但 `lastResultDetails.dateCompleted` 超過 7 天前的列為「可能卡住」。

範例：「Contoso 專案的測試計畫進度如何？」、「測試計畫 101 還有多少測試沒跑？」、「列出 Sprint 5 測試計畫中失敗或被阻擋的測試案例」

## 呈現結果

日期格式為 `YYYY/MM/DD`，時間轉成使用者當地時區，無法判斷時標明 UTC。每個測試計畫依序呈現：

1. **計畫概要**：名稱（連結 `https://dev.azure.com/{organization}/{project}/_testPlans/define?planId={planId}&suiteId={rootSuiteId}`）、反覆項目、期間（`startDate` ～ `endDate`）、負責人、測試點總數、執行率、通過率；關聯自動化測試的測試點另外註明數量。

2. **套件進度表**：依階層縮排列出套件（子套件前加 `└`），根套件列為計畫總計：

   | 套件 | 總數 | ✅ 通過 | ❌ 失敗 | 🚫 阻擋 | ⏳ 未執行 | 其他 | 執行率 | 通過率 |
   |---|---|---|---|---|---|---|---|---|
   | Contoso 發行測試 | 40 | 25 | 3 | 2 | 8 | 2 | 80% | 63% |
   | └ 登入功能 | 12 | 10 | 1 | 0 | 1 | 0 | 92% | 83% |

   「其他」欄合計不適用、進行中與其他分類；有值時在表下註明組成。

3. **失敗與阻擋的測試案例**：

   | 測試案例 | 套件 | 組態 | 結果 | 測試人員 | 最後執行 | 建置 |
   |---|---|---|---|---|---|---|
   | [12345](https://dev.azure.com/contoso/Contoso/_workitems/edit/12345) 登入失敗顯示錯誤訊息 | 登入功能 | Windows 11 | ❌ failed | Contoso Team | 2026/09/01 | MyApp_20260901.1 |

   測試案例 ID 連結格式為 `https://dev.azure.com/{organization}/{project}/_workitems/edit/{testCaseId}`。沒有失敗或阻擋時明確說明。

4. **尚未執行**：依測試人員彙總未執行的測試點數量（`tester.displayName`；沒有指派時寫「未指派」），方便追蹤誰還有測試要跑；接著列出「可能卡住」的測試點。

最後提醒：REST API 回傳的是即時狀態；網頁的 Progress Report 由 Analytics 產生，約有 15 分鐘延遲，兩者數字短時間內可能不同。
