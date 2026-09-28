---
name: pipelines-run
description: 以 Azure DevOps REST API 執行 Pipeline：以指定分支、runtime parameters（templateParameters）、queue-time 變數與要略過的 stage 觸發 YAML 或 classic Pipeline，YAML 先以 previewRun 乾跑驗證，確認後才執行，接著追蹤到完成；也能在確認後取消執行中的建置。當使用者要 run、trigger、queue 一個 pipeline、帶參數部署某個分支，或要取消建置時使用；查看建置清單、狀態與失敗記錄檔改用 `pipelines-build-summary`，核准等待中的 stage 改用 `pipelines-approvals`。
compatibility: 需要 curl、可連線至 dev.azure.com 的網路，以及 AZURE_DEVOPS_ORG 與 AZURE_DEVOPS_PAT 環境變數。
---

# 執行 Pipeline

這個 Skill 負責**讓 Pipeline 跑起來**：觸發、追蹤、取消。觸發與取消都是寫入動作，一律先把內容列給使用者確認。Run ID 就是建置 ID，建置結果的細節（記錄檔、變更）交給 `pipelines-build-summary`。

## 專案選擇

- 使用者有提供專案名稱（例如「Contoso 專案」）就直接使用，不呼叫專案清單 API。
- 沒有提供時，請使用者提供一次。
- 仍未提供時，列出專案讓使用者選擇：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/projects?%24top=500&api-version=7.1"
  ```

- 使用者提供或選定專案之前不要繼續。

## Pipeline 選擇

- 使用者有提供 Pipeline 名稱時，以名稱篩選找出 ID（`name` 可用 `*` 萬用字元，例如 `CI*`；找到多個時請使用者選一個）：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/build/definitions?name={name}&api-version=7.1"
  ```

- 沒有提供時，請使用者提供一次；仍未提供時，列出所有 Pipeline（`value[].id`、`name`、`folder`）讓使用者選擇：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/pipelines?api-version=7.1"
  ```

- 判斷類型：取得單一 Pipeline，`configuration.type` 為 `yaml` 用規則 1，為 `designerJson`（classic）用規則 2：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/pipelines/{pipelineId}?api-version=7.1"
  ```

- 取得建置定義，檢查能否排入與哪些變數可在執行時覆寫。`process.type` 為 `2` 是 YAML、`1` 是 classic；`queueStatus` 為 `disabled` 時無法執行，`paused` 時會排入但不會開始，先告知使用者；`variables.{name}.allowOverride` 為 `true` 的變數才能在執行時指定：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/build/definitions/{pipelineId}?api-version=7.1"
  ```

## 連線與驗證

一律用 `curl` 直接呼叫 Azure DevOps REST API，每個指令都自帶驗證參數。

- **組織**：讀取環境變數 `AZURE_DEVOPS_ORG`（組織名稱，例如 `contoso`）。未設定時詢問使用者一次；使用者給的是網址時，從 `https://dev.azure.com/{org}` 或 `https://{org}.visualstudio.com` 取出組織名稱。
- **權杖**：PAT 以 Basic 驗證送出：`-u ":$AZURE_DEVOPS_PAT"`；只設定了 `AZURE_DEVOPS_EXT_PAT`（Azure CLI azure-devops 擴充套件使用的變數）時，改用它。改用 Microsoft Entra ID 權杖時，換成 `-H "Authorization: Bearer $AZURE_DEVOPS_TOKEN"`。都未設定時，請使用者在自己的終端機設定 `AZURE_DEVOPS_PAT` 後重試，並停止。權杖只從環境變數讀取：不要請使用者把權杖貼進對話，也不要印出或寫入檔案。
- **PAT 權限範圍**：讀取（列出 Pipeline、追蹤執行）需要 Build (Read)；觸發、`previewRun` 乾跑與取消需要 Build (Read & execute)。
- **編碼**：路徑與查詢字串中的專案、Pipeline 名稱要 URL 編碼（空白寫成 `%20`）；查詢字串中的 `$` 寫成 `%24`（例如 `%24top`），避免被 shell 展開。
- **錯誤**：HTTP 401、403、203 或回應是 HTML 登入頁，代表權杖無效、過期或權限不足，回報後停止；HTTP 404 代表組織、專案、Pipeline 或建置 ID 有誤。HTTP 400 且 `typeKey` 為 `PipelineValidationException` 代表 YAML、參數或分支驗證失敗，把 `message` 原文給使用者看。
- **Windows PowerShell**：呼叫 `curl.exe`（而非 `curl` 別名），環境變數寫成 `$env:AZURE_DEVOPS_PAT`；PowerShell 不支援 `<<'EOF'`，先把 JSON 主體寫入暫存檔，再以 `--data-binary "@body.json"` 送出。

## 規則

### 1. 執行 YAML Pipeline（寫入，先乾跑）

1. **整理執行內容**：
   - **分支**：寫成完整 ref（`refs/heads/main`）；使用者沒指定時用定義的 `repository.defaultBranch`，並在確認時說明。要指定 commit 時加上 `"version": "{commitSha}"`。
   - **參數**（`templateParameters`）：名稱與值由使用者提供。不確定有哪些參數時，看最近幾次執行曾帶過的 `templateParameters`（規則 3 的 runs 清單），或先不帶參數乾跑，從 `finalYaml` 與錯誤訊息判讀（例如 `Unexpected parameter 'x'` 代表名稱錯誤；缺少必填參數時訊息會點名）。不要試圖用 git 或 items API 讀 YAML 檔推敲參數：YAML 可能引用 template 或放在 GitHub。
   - **變數**（`variables`）：只能指定 `allowOverride` 為 `true` 的變數；只寫在 YAML 裡的變數無法在執行時覆寫，先告知使用者。機密值設 `"isSecret": true`，而且不要在對話中重述。
   - **略過的 stage**（`stagesToSkip`）：用 stage 的識別名稱，不是顯示名稱。可從最近一次執行的 timeline（`GET .../_apis/build/builds/{buildId}/timeline?api-version=7.1`）取 `type` 為 `Stage` 的紀錄的 `identifier`。

2. **乾跑**：送出 `"previewRun": true`。乾跑不會建立執行（回應 `id` 為 `-1`、`state` 為 `unknown`），`finalYaml` 是展開 template 後的完整 YAML：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -X POST -H "Content-Type: application/json" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/pipelines/{pipelineId}/runs?api-version=7.1" \
     --data-binary @- <<'EOF'
   {
     "previewRun": true,
     "resources": {"repositories": {"self": {"refName": "refs/heads/main"}}},
     "templateParameters": {"environment": "dev"},
     "variables": {"LogLevel": {"value": "Debug", "isSecret": false}},
     "stagesToSkip": []
   }
   EOF
   ```

   回應 HTTP 400 時，把錯誤訊息給使用者看，修正後重新乾跑。

3. **確認**：把即將送出的內容列給使用者，確認一次後才執行：Pipeline 名稱、分支（與 commit）、每個參數與值、覆寫的變數（機密值以 `***` 表示）、略過的 stage，以及從 `finalYaml` 整理出的 stage 與 job 清單（例如「會部署到 prod」）。

4. **執行**：送出與乾跑相同的主體，只拿掉 `previewRun`：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -X POST -H "Content-Type: application/json" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/pipelines/{pipelineId}/runs?api-version=7.1" \
     --data-binary @- <<'EOF'
   {
     "resources": {"repositories": {"self": {"refName": "refs/heads/main"}}},
     "templateParameters": {"environment": "dev"},
     "variables": {"LogLevel": {"value": "Debug", "isSecret": false}},
     "stagesToSkip": []
   }
   EOF
   ```

   回應的 `id` 是 Run ID（也是建置 ID），`name` 是建置編號，`_links.web.href` 是網頁連結。接著用規則 3 追蹤。

範例：「用 release/2.0 分支跑 Contoso 專案的 MyApp-CD，environment 設成 staging」、「跑 MyApp-CI 但略過 Deploy stage」

### 2. 執行 classic Pipeline（寫入）

classic Pipeline 不支援 Runs API 與乾跑，改用 Builds API。把 Pipeline、分支與變數列給使用者，確認一次後才執行。`parameters` 是**JSON 字串**（不是物件），只放 `allowOverride` 為 `true` 的變數：

```bash
curl -sS -u ":$AZURE_DEVOPS_PAT" -X POST -H "Content-Type: application/json" \
  "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/build/builds?api-version=7.1" \
  -d '{"definition":{"id":101},"sourceBranch":"refs/heads/main","parameters":"{\"BuildConfiguration\":\"Release\"}"}'
```

回應的 `id` 是建置 ID，`status` 通常是 `notStarted`。接著用規則 3 追蹤。

範例：「排一個 Contoso 專案 Legacy-Build 的建置，BuildConfiguration 用 Debug」

### 3. 追蹤執行到完成

- 取得執行狀態。`state` 為 `inProgress`、`canceling` 或 `completed`；完成後 `result` 為 `succeeded`、`failed` 或 `canceled`：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/pipelines/{pipelineId}/runs/{runId}?api-version=7.1"
  ```

  classic Pipeline 或不知道 Pipeline ID 時，改用 `GET .../_apis/build/builds/{buildId}?api-version=7.1`（`status`：`notStarted`、`inProgress`、`cancelling`、`completed`；`result` 另有 `partiallySucceeded`）。
- 列出某個 Pipeline 最近的執行（含每次的 `templateParameters`）：`GET .../_apis/pipelines/{pipelineId}/runs?api-version=7.1`。
- 大約每 30 秒查一次（兩次查詢之間 `sleep 30`），每次只回報狀態有變化的部分。累計等待超過約 15 分鐘時，先問使用者要不要繼續等；不繼續時給網頁連結後停止。
- 長時間停在 `inProgress` 時，取 timeline 檢查是否在等核准：`type` 為 `Checkpoint.Approval` 且 `state` 為 `inProgress` 的紀錄，就是等待中的核准，往上找 `parentId` 到 `type` 為 `Stage` 的紀錄即可得知是哪個 stage。告知使用者並改用 `pipelines-approvals`，不要空等。
- 結束時：
  - `succeeded`：回報耗時與連結。
  - `failed` 或 `partiallySucceeded`：改用 `pipelines-build-summary` 的「檢視失敗建置的記錄檔」，帶入建置 ID 找出失敗步驟。
  - `canceled`：說明是誰取消（build 的 `lastChangedBy.displayName`）。

範例：「剛剛那次執行跑完了嗎？」、「幫我盯著 run 12345 直到結束」

### 4. 取消執行中的建置（寫入）

1. 先用 `GET .../_apis/build/builds/{buildId}?api-version=7.1` 確認 `status` 是 `notStarted` 或 `inProgress`；已完成的建置不能取消，直接說明。
2. 把建置編號、Pipeline 名稱、分支、要求者（`requestedFor.displayName`）、開始時間與目前進度列給使用者，確認一次後才執行：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -X PATCH -H "Content-Type: application/json" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/build/builds/{buildId}?api-version=7.1" \
     -d '{"status":"cancelling"}'
   ```

3. 回應的 `status` 會是 `cancelling`；再查一次，直到 `status` 為 `completed` 且 `result` 為 `canceled`。

不提供刪除建置的指示；需要刪除時請使用者自行到網頁操作。

範例：「取消 Contoso 專案的 build 12345」

## 呈現結果

時間以使用者的當地時區顯示（格式 `YYYY/MM/DD HH:MM`），無法判斷時區時標明 UTC。建置連結格式為 `https://dev.azure.com/{organization}/{project}/_build/results?buildId={runId}`；分支名稱去掉 `refs/heads/`。

- **確認畫面**（執行前）：以表格列出 Pipeline、類型（YAML／classic）、分支、參數、變數、略過的 stage；YAML 另附 `finalYaml` 整理出的 stage 與 job 清單。
- **已觸發**：Run ID（可點擊連結）、建置編號、分支、觸發時間。
- **追蹤進度**：目前狀態、已執行時間；等待核准時標示 ⏸️ 與 stage 名稱。
- **結果**：✅ succeeded、❌ failed、⚠️ partiallySucceeded、🚫 canceled，加上耗時；失敗時附上改用 `pipelines-build-summary` 的建議。
