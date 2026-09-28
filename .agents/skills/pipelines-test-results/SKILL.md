---
name: pipelines-test-results
description: 以 Azure DevOps REST API 檢視某次 Pipeline 建置發布的自動化測試結果：通過率與各 outcome 統計、各 test run 的數字、與上一次建置相比新增或已修復的失敗，以及失敗測試的錯誤訊息與 stack trace。當使用者問「build 12345 的測試結果」、「哪些測試失敗、為什麼」、「測試通過率多少」時使用；手動測試計畫（Test Plans）的執行進度改用 `test-plans-progress`，建置狀態與失敗步驟的記錄檔改用 `pipelines-build-summary`。
compatibility: 需要 curl、可連線至 dev.azure.com 與 vstmr.dev.azure.com 的網路，以及 AZURE_DEVOPS_ORG 與 AZURE_DEVOPS_PAT 環境變數。
---

# Pipeline 測試結果

這個 Skill 是唯讀的，以**建置 ID** 為中心，看的是 Pipeline 中測試工作（例如 `VSTest`、`PublishTestResults`）發布的自動化測試結果。

## 專案選擇

- 使用者有提供專案名稱（例如「Contoso 專案」）就直接使用，不呼叫專案清單 API。
- 沒有提供時，請使用者提供一次。
- 仍未提供時，列出專案讓使用者選擇：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/projects?%24top=500&api-version=7.1"
  ```

- 使用者提供或選定專案之前不要繼續。

## 建置選擇

- 使用者有提供建置 ID（例如「build 12345」）就直接使用。
- 只提供 Pipeline 名稱時，以 `GET .../_apis/build/definitions?name={name}&api-version=7.1` 找出定義 ID，再取最近一次完成的建置：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/build/builds?definitions={definitionId}&statusFilter=completed&%24top=1&queryOrder=finishTimeDescending&api-version=7.1"
  ```

  使用者有指定分支時加上 `branchName=refs/heads/main`。
- 都沒有提供時，請使用者提供一次建置 ID 或 Pipeline 名稱。

## 連線與驗證

一律用 `curl` 直接呼叫 Azure DevOps REST API，每個指令都自帶驗證參數。

- **組織**：讀取環境變數 `AZURE_DEVOPS_ORG`（組織名稱，例如 `contoso`）。未設定時詢問使用者一次；使用者給的是網址時，從 `https://dev.azure.com/{org}` 或 `https://{org}.visualstudio.com` 取出組織名稱。
- **主機**：除了 `dev.azure.com`，測試結果摘要 API 的主機是 `vstmr.dev.azure.com`。
- **權杖**：PAT 以 Basic 驗證送出：`-u ":$AZURE_DEVOPS_PAT"`；只設定了 `AZURE_DEVOPS_EXT_PAT`（Azure CLI azure-devops 擴充套件使用的變數）時，改用它。改用 Microsoft Entra ID 權杖時，換成 `-H "Authorization: Bearer $AZURE_DEVOPS_TOKEN"`。都未設定時，請使用者在自己的終端機設定 `AZURE_DEVOPS_PAT` 後重試，並停止。權杖只從環境變數讀取：不要請使用者把權杖貼進對話，也不要印出或寫入檔案。
- **PAT 權限範圍**：Test Management (Read)；用 Pipeline 名稱找建置另需 Build (Read)。
- **編碼**：路徑與查詢字串中的專案、Pipeline 名稱要 URL 編碼（空白寫成 `%20`）；查詢字串中的 `$` 寫成 `%24`（例如 `%24top`），避免被 shell 展開。
- **錯誤**：HTTP 401、403、203 或回應是 HTML 登入頁，代表權杖無效、過期或權限不足，回報後停止；HTTP 404 代表組織、專案、建置或 test run ID 有誤（`TestObjectNotFoundException` 代表 test run 已被刪除）。HTTP 400 且訊息為 `Maximum 200 test results can be processed in a single call` 時，把 `%24top` 降到 200。
- **Windows PowerShell**：呼叫 `curl.exe`（而非 `curl` 別名），環境變數寫成 `$env:AZURE_DEVOPS_PAT`；PowerShell 不支援 `<<'EOF'`，先把 JSON 主體寫入暫存檔，再以 `--data-binary "@body.json"` 送出。

## 步驟

### 1. 取得整個建置的測試摘要

```bash
curl -sS -u ":$AZURE_DEVOPS_PAT" \
  "https://vstmr.dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/testresults/resultsummarybybuild?buildId={buildId}&includeFailureDetails=true&api-version=7.1-preview.1"
```

- 這個資源只有 preview 版本，必須帶 `7.1-preview.1`，只寫 `7.1` 會回 HTTP 400。
- `aggregatedResultsAnalysis`：
  - `totalTests`、`duration`（`hh:mm:ss.fffffff`）。
  - `resultsByOutcome.{Outcome}.count`：各 outcome 的數量（例如 `Passed`、`Failed`、`NotExecuted`）。
  - `resultsDifference`：與上一次建置相比的增減（`increaseInTotalTests`、`increaseInFailures`、`increaseInPassedTests`、`increaseInDuration`）；比較對象是 `previousContext.build`，為 `null` 時代表沒有可比較的建置，不要呈現增減。
- `testFailures`（`includeFailureDetails=true` 才有）：`newFailures`（這次新失敗）、`existingFailures`（之前就失敗）、`fixedTests`（這次修好），各有 `count` 與 `testResults[]`（`testRunId`、`testResultId`），步驟 3 用它標示每個失敗是新的還是既有的。
- `totalTests` 為 `0` 時，說明這個建置沒有發布測試結果（Pipeline 可能沒有測試工作，或建置已被刪除），並停止。
- 通過率以 `Passed` ÷ `totalTests` 計算，四捨五入到小數一位；`NotExecuted` 等未執行的數量另外列出。

### 2. 列出建置的 test run

```bash
curl -sS -u ":$AZURE_DEVOPS_PAT" \
  "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/test/runs?buildUri=vstfs:///Build/Build/{buildId}&includeRunDetails=true&%24top=100&api-version=7.1"
```

- 每筆有 `id`、`name`、`state`（`Completed`、`InProgress`、`Aborted`…）、`totalTests`、`passedTests`、`unanalyzedTests`（尚未分析的失敗）、`notApplicableTests`（未執行等）、`incompleteTests`、`webAccessUrl`。加上 `includeRunDetails=true` 另有 `startedDate`、`completedDate` 與 `pipelineReference.stageReference.stageName`、`jobReference.jobName`，可看出是哪個 stage、job 發布的。
- 分頁用 `%24top` 與 `%24skip`；回傳筆數等於 `%24top` 時再取下一頁。
- `state` 是數字（例如 `255`）的 run 通常已被保留原則刪除，無法再取結果，標示後略過。
- 沒有任何 run 時，說明這個建置沒有 test run。

### 3. 取得失敗測試的細節

對 `totalTests` 大於 `passedTests + notApplicableTests` 的每個 run 取失敗結果：

```bash
curl -sS -u ":$AZURE_DEVOPS_PAT" \
  "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/test/Runs/{runId}/results?outcomes=Failed,Aborted,Timeout,Error&%24top=1000&api-version=7.1"
```

- `outcomes` 以逗號分隔多個值。不帶 `detailsToInclude` 時 `%24top` 最多 1000；帶 `detailsToInclude`（`Iterations`、`WorkItems`、`SubResults`、`Point`）時最多 200。分頁用 `%24skip`；回傳筆數等於 `%24top` 時再取下一頁。
- 預設回應就包含所需欄位，不需要再逐筆查詢：
  - `testCaseTitle`、`automatedTestName`（完整名稱）、`automatedTestStorage`（例如測試組件名稱）
  - `outcome`、`durationInMs`（可能不存在）、`computerName`
  - `errorMessage`、`stackTrace`（可能長達數千字，呈現時只取前段）
  - `failingSince.build.number` 與 `failingSince.date`：從哪次建置開始失敗
- 以步驟 1 的 `testFailures.newFailures.testResults[]` 比對 `testRun.id` 與結果 `id`，標示 🆕 新失敗；在 `existingFailures` 中的標示 🔁 持續失敗。
- 失敗超過 20 筆時，先依 `errorMessage` 的第一行分組（相同錯誤常是同一個原因），每組列出數量與前幾個測試名稱，再請使用者挑要看細節的組別。
- 需要測試附件或 console 輸出時，請使用者到 `webAccessUrl` 在網頁查看；要看失敗步驟的建置記錄檔改用 `pipelines-build-summary`。

範例：「Contoso 專案 build 12345 的測試結果如何？」、「MyApp-CI 最新一次建置有哪些測試失敗？」、「這些失敗是這次才出現的嗎？」

## 呈現結果

時間以使用者的當地時區顯示（格式 `YYYY/MM/DD HH:MM`），無法判斷時區時標明 UTC。建置測試頁面連結為 `https://dev.azure.com/{organization}/{project}/_build/results?buildId={buildId}&view=ms.vss-test-web.build-test-results-tab`。

1. **摘要**：建置編號（連結）、總數、✅ 通過、❌ 失敗、⏭️ 未執行、通過率、總耗時；有上一次建置可比較時加上增減（例如「失敗 +3」）與 🆕 新失敗、🔁 持續失敗、🎉 已修復的數量。
2. **Test run**：表格列出名稱、stage／job、狀態、總數、通過、失敗、未執行。
3. **失敗測試**：表格列出測試名稱（`testCaseTitle`，太長時縮短並保留類別與方法名）、新／持續、開始失敗的建置、錯誤訊息第一行。
4. **錯誤細節**：對每個（或每組）失敗，以程式碼區塊呈現 `errorMessage` 與 `stackTrace` 的前 20 行。
5. 沒有失敗時，明確說明所有測試都通過。
