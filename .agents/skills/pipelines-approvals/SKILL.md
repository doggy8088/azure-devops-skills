---
name: pipelines-approvals
description: 以 Azure DevOps REST API 處理 Pipeline 核准（approval）：列出等待我核准的部署，包含指派給我所屬群組的，對應到 Pipeline、執行、分支與 stage，並在確認後核准或拒絕並附上意見。當使用者問「有哪些部署在等我核准」、要 approve 或 reject 某個 stage、或問某次執行為何卡在等待核准時使用；環境與 checks 的設定不在此範圍，檢視改用 `pipelines-security-audit`；觸發或追蹤 Pipeline 改用 `pipelines-run`。核准 Pull Request 改用 `repos-pull-request-review`。
compatibility: 需要 curl、可連線至 dev.azure.com 與 vssps.dev.azure.com 的網路，以及 AZURE_DEVOPS_ORG 與 AZURE_DEVOPS_PAT 環境變數。
---

# Pipeline 核准

這個 Skill 站在**核准者**的角度處理 YAML Pipeline 在 environment 或資源上的核准（Approvals and Checks）：找出等我核准的項目、看懂它要部署什麼、回應它。新增或修改核准者、checks 的設定請使用者到網頁操作。

## 專案選擇

- 使用者有提供專案名稱（例如「Contoso 專案」）就直接使用，不呼叫專案清單 API。
- 沒有提供時，請使用者提供一次。
- 仍未提供時，列出專案讓使用者選擇：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/projects?%24top=500&api-version=7.1"
  ```

- 核准 API 沒有組織層級的端點，一定要指定專案。使用者要「所有專案」時，對清單中的每個專案重複規則 1；專案超過 20 個時先告知數量並確認一次。
- 使用者提供或選定專案之前不要繼續。

## 連線與驗證

一律用 `curl` 直接呼叫 Azure DevOps REST API，每個指令都自帶驗證參數。

- **組織**：讀取環境變數 `AZURE_DEVOPS_ORG`（組織名稱，例如 `contoso`）。未設定時詢問使用者一次；使用者給的是網址時，從 `https://dev.azure.com/{org}` 或 `https://{org}.visualstudio.com` 取出組織名稱。
- **主機**：除了 `dev.azure.com`，查詢群組成員的 Identities API 主機是 `vssps.dev.azure.com`。
- **權杖**：PAT 以 Basic 驗證送出：`-u ":$AZURE_DEVOPS_PAT"`；只設定了 `AZURE_DEVOPS_EXT_PAT`（Azure CLI azure-devops 擴充套件使用的變數）時，改用它。改用 Microsoft Entra ID 權杖時，換成 `-H "Authorization: Bearer $AZURE_DEVOPS_TOKEN"`。都未設定時，請使用者在自己的終端機設定 `AZURE_DEVOPS_PAT` 後重試，並停止。權杖只從環境變數讀取：不要請使用者把權杖貼進對話，也不要印出或寫入檔案。
- **PAT 權限範圍**：列出與對應執行需要 Build (Read)、Identity (Read)；核准或拒絕另需 Build (Read & execute) 與 Pipeline Resources (Use)（Microsoft Learn 的 Approvals - Update 列出這兩個範圍）。
- **編碼**：路徑中的專案名稱要 URL 編碼（空白寫成 `%20`）；查詢字串中的 `$` 寫成 `%24`（例如 `%24expand`），避免被 shell 展開。
- **錯誤**：HTTP 401、403、203 或回應是 HTML 登入頁，代表權杖無效、過期或權限不足，回報後停止；HTTP 404 代表組織、專案或核准 ID 有誤。核准或拒絕失敗時，把錯誤訊息原文給使用者看（常見原因：我不是該 step 的核准者、在 `blockedApprovers` 中，或核准已不是 `pending`）。
- **Windows PowerShell**：呼叫 `curl.exe`（而非 `curl` 別名），環境變數寫成 `$env:AZURE_DEVOPS_PAT`；PowerShell 不支援 `<<'EOF'`，先把 JSON 主體寫入暫存檔，再以 `--data-binary "@body.json"` 送出。

## 目前使用者

規則 1 需要目前使用者的身分 ID（這個端點不帶 `api-version`）：

```bash
curl -sS -u ":$AZURE_DEVOPS_PAT" "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/connectionData"
```

使用 `authenticatedUser.id`；`authenticatedUser.providerDisplayName` 是顯示名稱。

## 規則

### 1. 列出等待我核准的項目

1. 取得專案中等待中的核准。一定要帶 `%24expand=steps`，否則 `steps` 是空陣列；加上 `permissions` 可知道我能不能處理：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/pipelines/approvals?state=pending&%24expand=steps,permissions&api-version=7.1"
   ```

   - 每筆有 `id`、`status`、`instructions`（核准說明）、`minRequiredApprovers`、`executionOrder`（`anyOrder` 或 `inSequence`）、`blockedApprovers`、`createdOn`，以及 `pipeline`（見規則 2）。
   - `steps[]` 是每位核准者：`assignedApprover`（`id`、`displayName`、`descriptor`）、`status`、`permissions`、`comment`。
   - 限制筆數的參數是 `top`（沒有 `$`）；不帶 `state` 會回傳所有狀態。不要依賴 `userIds` 篩選，一律在本機比對。

2. 判斷哪些是「我的」：
   - **直接指派**：`steps[].assignedApprover.id` 等於我的 ID，且該 step 的 `status` 為 `pending`。
   - **透過群組**：`assignedApprover.descriptor` 以 `vssgp.` 或 `aadgp.` 開頭（或 `isContainer` 為 `true`）的是群組。取得我的直接群組成員資格（`memberOf` 是群組描述子清單）：

     ```bash
     curl -sS -u ":$AZURE_DEVOPS_PAT" \
       "https://vssps.dev.azure.com/$AZURE_DEVOPS_ORG/_apis/identities?identityIds={myId}&queryMembership=Direct&api-version=7.1"
     ```

     再以 `identityIds={groupId1},{groupId2}` 一次查出這些群組的 `descriptor`，出現在我的 `memberOf` 中的就算「透過 {群組名稱} 待我核准」。
   - `steps[].permissions` 或整筆的 `permissions` 含 `update`，代表我目前可以處理；只有 `view` 時列出但標示「無法處理」。
   - 我出現在 `blockedApprovers` 中（例如不允許核准自己觸發的執行）時，標示「你被排除，無法核准」。
   - `executionOrder` 為 `inSequence` 時，只有 `order` 最小且仍 `pending` 的 step 可以處理；`status` 為 `uninitiated` 的 step 還沒輪到。
   - `assignedApprover.displayName` 為 `null` 時，身分無法解析，列出 ID 並標示「無法辨識的核准者」。

3. 對每筆「我的」核准做規則 2，補上 Pipeline、執行與 stage。
4. 沒有任何等待中的核准時，明確說明沒有等我核准的項目。

範例：「Contoso 專案有哪些部署在等我核准？」、「所有專案裡有沒有等我 approve 的 stage？」

### 2. 對應 Pipeline、執行與 stage

- 核准物件的 `pipeline` 欄位（Microsoft Learn 的模型沒有列出，但實際會回傳）：`pipeline.id` 與 `pipeline.name` 是 Pipeline，`pipeline.owner.id` 是執行（建置）ID，`pipeline.owner.name` 是建置編號，`pipeline.owner._links.web.href` 是網頁連結。
- 需要分支與要求者時，取得建置（`sourceBranch`、`requestedFor.displayName`、`sourceVersion`）：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/build/builds/{buildId}?api-version=7.1"
  ```

- 找出 stage：取得該建置的 timeline，`records[]` 中 `id` 等於核准 ID 的紀錄，`type` 是 `Checkpoint.Approval`（等待中時 `state` 為 `inProgress`）。它的 `parentId` 指向 `type` 為 `Checkpoint` 的紀錄，再往上一層的 `parentId` 就是 `type` 為 `Stage` 的紀錄，`name` 是 stage 顯示名稱，`identifier` 是 stage 識別名稱：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/build/builds/{buildId}/timeline?api-version=7.1"
  ```

  同一個 `Checkpoint` 底下的 `Checkpoint.TaskCheck`、`Checkpoint.ExtendsCheck` 等紀錄是其他 checks；它們 `state` 還是 `pending` 時，說明核准後仍要等這些 checks 通過。
- 核准物件沒有 `pipeline` 時，列出執行中的建置（`GET .../_apis/build/builds?statusFilter=inProgress&api-version=7.1`），逐一取 timeline 找 `id` 等於核准 ID 的紀錄；都找不到時標示「無法對應到執行」，不要猜測。
- 從執行找核准（使用者問「build 12345 為什麼卡住」）：反過來在 timeline 找 `type` 為 `Checkpoint.Approval` 且 `state` 為 `inProgress` 的紀錄，以它的 `id` 查核准：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/pipelines/approvals/{approvalId}?%24expand=steps&api-version=7.1"
  ```

範例：「build 12345 卡在哪個 stage 等誰核准？」

### 3. 核准或拒絕（寫入）

1. 把即將送出的內容列給使用者，確認一次後才執行：Pipeline 名稱、建置編號（連結）、分支與 commit、stage、核准說明（`instructions` 全文）、要求者、決定（核准或拒絕）與意見全文。使用者沒給意見時詢問一次，仍沒有就留空。拒絕會讓該 stage 失敗，要特別提醒。
2. 送出（`status` 為 `approved` 或 `rejected`；陣列可一次處理多筆，但每筆都要先經過確認）：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -X PATCH -H "Content-Type: application/json" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/pipelines/approvals?api-version=7.1" \
     -d '[{"approvalId":"{approvalId}","status":"approved","comment":"已確認變更內容，同意部署。"}]'
   ```

3. 檢查回應中該核准的 `status`：仍是 `pending` 代表還需要其他人（`minRequiredApprovers` 大於已核准人數），說明還差幾位；變成 `approved` 或 `rejected` 代表已生效。之後要追蹤執行時改用 `pipelines-run`。

改派核准者（`reassignTo`）與延後核准不在此範圍，請使用者到網頁操作。

範例：「核准 MyApp-CD 在 Production stage 的部署」、「拒絕 build 12345 的部署，原因是還沒通過 QA」

## 呈現結果

時間以使用者的當地時區顯示（格式 `YYYY/MM/DD HH:MM`），無法判斷時區時標明 UTC。建置連結格式為 `https://dev.azure.com/{organization}/{project}/_build/results?buildId={buildId}`；分支名稱去掉 `refs/heads/`。

**等待核准清單**以表格呈現：

- **Pipeline**（`pipeline.name`）與**建置編號**（可點擊連結）
- **分支**、**Stage**（顯示名稱）
- **指派方式**：👤 直接指派給我，或 👥 透過 {群組名稱}
- **進度**：已核准人數／`minRequiredApprovers`，`inSequence` 時註明順序
- **等待時間**：由 `createdOn` 計算（例如「3 小時」）
- **說明**：`instructions` 的第一行
- **能否處理**：✅ 可以處理、👀 只能檢視、⛔ 被排除

**核准後**回報：✅ approved 或 ❌ rejected、我的意見、核准的最終狀態，以及還需要幾位核准者。
