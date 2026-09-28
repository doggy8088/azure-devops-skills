---
name: pipelines-security-audit
description: 以 Azure DevOps REST API 稽核專案的 Pipeline 安全設定：job 授權範圍、儲存庫存取保護、佇列時變數限制、停用 Classic pipeline 等專案設定，服務連線的驗證方式與閒置狀態，以及服務連線、變數群組、安全檔案、代理程式佇列、環境是否開放給所有 pipeline、正式環境資源是否有核准或分支控制檢查，並附上網頁操作的修正步驟；只讀取、不修改。當使用者要做 pipeline 安全檢查、稽核 service connection 或 protected resources、檢查 approvals and checks 時使用。
compatibility: 需要 curl、可連線至 dev.azure.com 的網路，以及 AZURE_DEVOPS_ORG 與 AZURE_DEVOPS_PAT 環境變數。
---

# Pipeline 安全稽核

這個 Skill 以**專案**為範圍，對照 Microsoft Learn 的 Azure Pipelines 安全建議產生檢查表。全程只呼叫 GET，不修改任何設定；修正一律提供網頁操作步驟，由使用者自行處理。

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
- **PAT 權限範圍**：Project and Team (Read)、Service Connections (Read)、Build (Read)、Variable Groups (Read)、Agent Pools (Read)、Secure Files (Read)、Code (Read)；列出環境另需 Environment (Read & manage)（選用，見步驟 3）。各檢查對應的範圍見「檢查項目」。這個 Skill 不需要任何寫入權限。
- **編碼**：路徑中的專案名稱要 URL 編碼（空白寫成 `%20`）；查詢字串中的 `$` 寫成 `%24`（例如 `%24top`），避免被 shell 展開。
- **錯誤**：HTTP 401、403、203 或回應是 HTML 登入頁，代表權杖無效、過期或權限不足，回報後停止；HTTP 404 代表組織、專案或資源 ID 有誤。例外：專案設定已成功取得後，某個檢查的 API 回 403 時，只把該檢查標 ❓ 並註明缺少的權限範圍，繼續其他檢查。
- **Windows PowerShell**：呼叫 `curl.exe`（而非 `curl` 別名），環境變數寫成 `$env:AZURE_DEVOPS_PAT`；PowerShell 不支援 `<<'EOF'`，先把 JSON 主體寫入暫存檔，再以 `--data-binary "@body.json"` 送出。

## 檢查項目

| 檢查 | 使用的 API | PAT 權限範圍 | 依據（Microsoft Learn） |
|---|---|---|---|
| A. 專案的 Pipeline 設定 | `build/generalsettings` | Project and Team (Read) | [Job access tokens](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/access-tokens)、[Secure Azure Pipelines](https://learn.microsoft.com/en-us/azure/devops/pipelines/security/overview)、[Approach](https://learn.microsoft.com/en-us/azure/devops/pipelines/security/approach) |
| B. 服務連線的驗證方式、範圍與閒置 | `serviceendpoint/endpoints`、`executionhistory` | Service Connections (Read) | [Overview](https://learn.microsoft.com/en-us/azure/devops/pipelines/security/overview)、[Misc](https://learn.microsoft.com/en-us/azure/devops/pipelines/security/misc) |
| C. 受保護資源是否開放給所有 pipeline | `pipelines/pipelinepermissions` | Build (Read)，另需各資源清單的範圍 | [Resources](https://learn.microsoft.com/en-us/azure/devops/pipelines/security/resources) |
| D. 正式環境資源的核准與檢查 | `pipelines/checks/configurations` | Build (Read) | [Resources](https://learn.microsoft.com/en-us/azure/devops/pipelines/security/resources)、[Overview](https://learn.microsoft.com/en-us/azure/devops/pipelines/security/overview) |

判斷符號：✅ 符合建議、❌ 不符合、❓ 無法判斷（欄位不存在、權限不足或需要人工確認）。

## 步驟

### 1. 專案的 Pipeline 設定（A）

```bash
curl -sS -u ":$AZURE_DEVOPS_PAT" \
  "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/build/generalsettings?api-version=7.1"
```

每個欄位的建議值都是 `true`；欄位不存在時標 ❓，不要當成 `false`：

| 欄位 | UI 設定名稱 | 依據 |
|---|---|---|
| `enforceJobAuthScope` | Limit job authorization scope to current project for non-release pipelines | access-tokens |
| `enforceJobAuthScopeForReleases` | Limit job authorization scope to current project for release pipelines | access-tokens |
| `enforceReferencedRepoScopedToken` | Protect access to repositories in YAML pipelines | access-tokens |
| `enforceSettableVar` | Limit variables that can be set at queue time | overview |
| `enableShellTasksArgsSanitizing` | Enable shell tasks arguments parameter validation | overview |
| `disableClassicBuildPipelineCreation` | Disable creation of classic build pipelines | approach |
| `disableClassicReleasePipelineCreation` | Disable creation of classic release pipelines | approach |
| `enforceNoAccessToSecretsFromForks` | 不讓 fork 的建置取得 secrets（見 overview 的「Don't provide secrets to fork builds」） | overview、misc |

- 組織層級的設定沒有 REST API（`{org}/_apis/build/generalsettings` 回 404）。組織層級開啟時會套用到所有專案，因此專案層級為 `false` 的項目標 ❌，並請使用者到 **Organization settings** > **Pipelines** > **Settings**（`https://dev.azure.com/{org}/_settings/pipelinessettings`）確認是否已在組織層級開啟。Learn 建議在組織層級限制 job 授權範圍。
- 公開專案的 job 授權範圍一律限制在專案內，與設定值無關，在結果中註明。

### 2. 服務連線（B）

1. 列出服務連線：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/serviceendpoint/endpoints?api-version=7.1"
   ```

   記下 `id`、`name`、`type`、`authorization.scheme`、`isShared`、`isReady`、`isDisabled`、`serviceEndpointProjectReferences`。`type` 大小寫不一（例如 `azurerm` 與 `AzureRM`），比對時不分大小寫。清單不含任何憑證內容，也不要嘗試取得。

2. 逐一判斷：
   - **驗證方式**（`type` 為 `azurerm`）：`authorization.scheme` 為 `WorkloadIdentityFederation` 標 ✅；`ServicePrincipal`、`PublishProfile` 等使用秘密的方式標 ❌；`ManagedServiceIdentity` 不存放秘密，但需確認 agent 身分的權限，標 ❓。
   - **範圍**（`azurerm`）：`authorization.parameters.scope` 含 `/resourcegroups/` 標 ✅；只到訂用帳戶或管理群組標 ❌（Learn 建議選擇特定資源群組）；沒有這個欄位（多為手動建立）標 ❓。
   - **就緒**：`isReady` 為 `false` 標 ❌；`isDisabled` 為 `true` 標示「已停用」，略過其餘檢查。
   - **跨專案共用**：`isShared` 為 `true` 或 `serviceEndpointProjectReferences` 超過一筆時，註明共用的專案數，提醒確認每個專案都需要它。

3. 閒置檢查：取最近一筆使用紀錄（依時間由新到舊排序）：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/serviceendpoint/{endpointId}/executionhistory?top=1&api-version=7.1"
   ```

   `value[0].data.finishTime`（沒有時用 `startTime`）距今超過 100 天標 ❌；`value` 為空代表查不到使用紀錄，標 ❓ 並建議確認是否仍需要。`data.planType`（例如 `Build`、`Release`）與 `data.definition.name` 可顯示最後使用它的 pipeline。依據：Learn 建議定期檢視 **Usage history**，並說明 Azure Pipelines 開始自動停用閒置 100 天的服務連線。

### 3. 列出其他受保護資源

依序取得清單；某個清單回 403 時，該類資源標 ❓（缺少的權限範圍寫在括號內）：

```bash
curl -sS -u ":$AZURE_DEVOPS_PAT" \
  "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/distributedtask/variablegroups?api-version=7.1"
curl -sS -u ":$AZURE_DEVOPS_PAT" \
  "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/distributedtask/securefiles?api-version=7.1-preview.1"
curl -sS -u ":$AZURE_DEVOPS_PAT" \
  "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/distributedtask/queues?api-version=7.1"
curl -sS -u ":$AZURE_DEVOPS_PAT" \
  "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories?api-version=7.1"
```

- 變數群組（Variable Groups (Read)）：記下 `id`、`name`、`type`（`Vsts` 或 `AzureKeyVault`），以及是否有 `variables.*.isSecret` 為 `true` 的變數；不要輸出任何變數值。
- 安全檔案（Secure Files (Read)）：Learn 文件只列出 preview 版本，這裡帶 `7.1-preview.1`；記下 `id`、`name`。
- 代理程式佇列（Agent Pools (Read)）：記下 `id`、`name`、`pool.isHosted`（`true` 是 Microsoft 代管）。
- 儲存庫（Code (Read)）：記下 `id` 與 `project.id`。
- 環境（選用）：列出環境的 API 只接受 Environment (Read & manage) 這個具管理權限的範圍，沒有唯讀範圍可用。PAT 有這個範圍時才呼叫；沒有時略過清單並標 ❓，或請使用者從環境網址（`/_environments/{id}`）提供環境 ID，步驟 4、5 對環境只需要 Build (Read)。

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/distributedtask/environments?%24top=100&api-version=7.1"
  ```

### 4. 開放存取（C）

對每個服務連線、變數群組、安全檔案、佇列、儲存庫與環境查詢 pipeline 權限（只有 preview 版本，必須帶 `7.1-preview.1`，帶 `7.1` 會回 400）：

```bash
curl -sS -u ":$AZURE_DEVOPS_PAT" \
  "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/pipelines/pipelinepermissions/{resourceType}/{resourceId}?api-version=7.1-preview.1"
```

| 資源 | `resourceType` | `resourceId` |
|---|---|---|
| 服務連線 | `endpoint` | 服務連線 `id` |
| 變數群組 | `variablegroup` | 變數群組 `id` |
| 安全檔案 | `securefile` | 安全檔案 `id` |
| 代理程式佇列 | `queue` | 佇列 `id` |
| 環境 | `environment` | 環境 `id` |
| 儲存庫 | `repository` | `{projectId}.{repositoryId}`（只給儲存庫 ID 會回 400） |

- `allPipelines.authorized` 為 `true`：UI 的 **Open access**（所有 pipeline 都能使用），標 ❌。Learn 建議不要開啟，只明確授權給信任的 pipeline。含 secret 的變數群組、服務連線與正式環境優先處理。
- 沒有 `allPipelines` 欄位：只有 `pipelines[]` 列出的 pipeline 能使用，標 ✅，並附上授權的 pipeline 數。
- Microsoft 代管的佇列（`pool.isHosted` 為 `true`）一律回傳 `authorized: true`，且 `authorizedOn` 是查詢當下的時間；這是預設行為，只列為參考，不標 ❌。

### 5. 正式環境資源的核准與檢查（D）

1. 找出正式環境類的資源：名稱含 `prod`、`prd`、`production`、`live`、`正式` 的服務連線、環境、變數群組、自行代管的佇列；把清單給使用者看，請使用者增減一次。
2. 對每個資源查詢檢查設定（只有 preview 版本，必須帶 `7.1-preview.1`）；`resourceType` 與 `resourceId` 同步驟 4：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/pipelines/checks/configurations?resourceType={resourceType}&resourceId={resourceId}&%24expand=settings&api-version=7.1-preview.1"
   ```

3. 判斷：略過 `isDisabled` 為 `true` 的檢查。
   - 有核准（`type.id` 為 `8c6f20a7-a545-4486-9777-f762fafe0d4d`，`type.name` 為 `Approval`），或有分支控制（`type.name` 為 `Task Check` 且 `settings.definitionRef.name` 為 `evaluatebranchProtection`，`settings.displayName` 為 `Branch control`）標 ✅；兩者都沒有標 ❌。
   - 其他檢查以 `type.name` 顯示（例如 `ExtendsCheck` 是 Required template、`ExclusiveLock` 是 Exclusive lock；其餘 `Task Check` 用 `settings.displayName`）。
   - 核准：顯示 `settings.approvers[].displayName` 的人數與 `settings.minRequiredApprovers`；分支控制：顯示 `settings.inputs.allowedBranches`，`settings.inputs.ensureProtectionOfBranch` 不是 `true` 時註明「未要求分支受保護」。

範例：「稽核 Contoso 專案的 pipeline 安全設定」、「哪些 service connection 還在用 service principal 秘密？」、「有哪些資源開放給所有 pipeline？」

## 呈現結果

時間以使用者的當地時區顯示，無法判斷時區時標明 UTC；日期格式為 `YYYY/MM/DD`。依序呈現：

1. **概要**：專案、各類檢查的 ✅/❌/❓ 數量，以及因權限不足而略過的檢查與所需範圍。
2. **專案設定表**：UI 設定名稱、目前值、判斷、依據連結。
3. **服務連線表**：名稱、類型、驗證方式、範圍、就緒、最後使用日期、開放存取、核准或分支控制（非正式環境寫「—」）。
4. **其他受保護資源表**：類型、名稱、開放存取、授權的 pipeline 數、核准或分支控制、備註（例如「含 secret」、「Microsoft 代管，僅供參考」）。
5. **修正步驟**：依未通過的項目分組，列出受影響的資源與網頁操作：
   - 專案設定：**Project settings** > **Pipelines** > **Settings** 開啟對應的設定；建議在 **Organization settings** > **Pipelines** > **Settings** 一次開啟。
   - 服務連線驗證方式：**Project settings** > **Service connections** > 選擇連線，依 [Use an Azure Resource Manager service connection](https://learn.microsoft.com/en-us/azure/devops/pipelines/library/connect-to-azure) 改用 workload identity federation，並把範圍縮小到特定資源群組。
   - 閒置連線：在同一頁的 **Usage history** 確認後，停用或移除（由使用者自行操作）。
   - 開放存取：開啟資源的 **Security** 頁，在 **Pipeline permissions** 關閉 **Open access**，改為只加入信任的 pipeline。位置：服務連線在 **Project settings** > **Pipelines** > **Service connections** > 連線 > **More actions** > **Security**；變數群組與安全檔案在 **Pipelines** > **Library** > 資源 > **Security**；代理程式集區在 **Project settings** > **Pipelines** > **Agent pools** > 集區 > **Security**；環境在 **Pipelines** > **Environments** > 環境 > **More actions** > **Security**；儲存庫在 **Project settings** > **Repositories** > 儲存庫 > **Security**。
   - 核准與檢查：在同一個資源頁面的 **More actions** > **Approvals and checks** 加入 **Approvals** 或 **Branch control**（見 [Define approvals and checks](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/approvals)）。

   最後提醒這份報告沒有修改任何設定。
