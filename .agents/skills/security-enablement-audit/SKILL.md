---
name: security-enablement-audit
description: 以 Azure DevOps REST API 盤點 GitHub Advanced Security for Azure DevOps 的啟用涵蓋率：哪些儲存庫尚未啟用 Secret Protection 或 Code Security、push protection 是否開啟、相依套件掃描與 CodeQL default setup 等功能是否啟用、新儲存庫是否自動啟用，並依專案彙總、附上網頁操作的啟用步驟；只讀取、不修改。當使用者要檢查 Advanced Security 啟用狀態、涵蓋率或 secret scanning 是否全面開啟時使用；要查看或摘要安全性警示改用 `security-alert-review`。
compatibility: 需要 curl、可連線至 dev.azure.com 與 advsec.dev.azure.com 的網路，以及 AZURE_DEVOPS_ORG 與 AZURE_DEVOPS_PAT 環境變數。
---

# Advanced Security 啟用稽核

這個 Skill 盤點組織或單一專案的 GitHub Advanced Security for Azure DevOps 啟用狀態，找出沒有被保護的儲存庫。它只讀取啟用設定，不讀取警示（警示改用 `security-alert-review`），也不啟用任何功能：啟用會開始計費，一律由使用者在網頁上操作。

## 範圍

- 預設稽核整個組織。
- 使用者指定專案（例如「Contoso 專案」）時只稽核那個專案；指定儲存庫時只查那些儲存庫。
- 不需要先列出專案：儲存庫清單已包含專案名稱。

## 連線與驗證

一律用 `curl` 直接呼叫 Azure DevOps REST API，每個指令都自帶驗證參數。

- **組織**：讀取環境變數 `AZURE_DEVOPS_ORG`（組織名稱，例如 `contoso`）。未設定時詢問使用者一次；使用者給的是網址時，從 `https://dev.azure.com/{org}` 或 `https://{org}.visualstudio.com` 取出組織名稱。
- **主機**：除了 `dev.azure.com`，Advanced Security 啟用設定 API 的主機是 `advsec.dev.azure.com`。
- **權杖**：PAT 以 Basic 驗證送出：`-u ":$AZURE_DEVOPS_PAT"`；只設定了 `AZURE_DEVOPS_EXT_PAT`（Azure CLI azure-devops 擴充套件使用的變數）時，改用它。改用 Microsoft Entra ID 權杖時，換成 `-H "Authorization: Bearer $AZURE_DEVOPS_TOKEN"`。都未設定時，請使用者在自己的終端機設定 `AZURE_DEVOPS_PAT` 後重試，並停止。權杖只從環境變數讀取：不要請使用者把權杖貼進對話，也不要印出或寫入檔案。
- **PAT 權限範圍**：Advanced Security (Read)、Code (Read)。這個 Skill 不需要任何寫入權限。
- **編碼**：路徑中的專案與儲存庫名稱要 URL 編碼（空白寫成 `%20`）；查詢字串中的 `$` 寫成 `%24`（例如 `%24top`），避免被 shell 展開。
- **錯誤**：HTTP 401、403、203 或回應是 HTML 登入頁，代表權杖無效、過期或權限不足，回報後停止；HTTP 404 代表組織、專案或儲存庫有誤。HTTP 400 且訊息含 `VssVersionOutOfRangeException` 代表 api-version 不被接受，改用「舊版回應」一節的版本。
- **Windows PowerShell**：呼叫 `curl.exe`（而非 `curl` 別名），環境變數寫成 `$env:AZURE_DEVOPS_PAT`；PowerShell 不支援 `<<'EOF'`，先把 JSON 主體寫入暫存檔，再以 `--data-binary "@body.json"` 送出。

## 欄位說明

依據 [Configure GitHub Advanced Security for Azure DevOps features](https://learn.microsoft.com/en-us/azure/devops/repos/security/configure-github-advanced-security-features) 與 REST 參考 [Org Enablement - Get](https://learn.microsoft.com/en-us/rest/api/azure/devops/advancedsecurity/org-enablement/get)。Advanced Security 現在分成兩個產品：**GitHub Secret Protection for Azure DevOps**（push protection、secret scanning 警示）與 **GitHub Code Security for Azure DevOps**（相依套件警示、CodeQL 掃描、第三方工具的安全性結果）。下表只描述 Learn 有記載的內容：

| 欄位 | 說明 |
|---|---|
| `secretProtectionFeatures.secretProtectionEnabled` | 是否啟用 Secret Protection。啟用後自動開始 push protection 與儲存庫 secret scanning |
| `secretProtectionFeatures.blockPushes` | 含機密的推送是否被阻擋（push protection）；只有帶 `includeAllProperties=true` 時才有值，否則為 `null` |
| `codeSecurityFeatures.codeSecurityEnabled` | 是否啟用 Code Security |
| `codeSecurityFeatures.dependencyScanningInjectionEnabled` | Dependency Scanning Injection：自動把相依套件掃描工作加入以預設分支為目標的 pipeline 與 PR 建置（UI 的 **Scan for vulnerable dependencies** / dependency scanning default setup） |
| `codeSecurityFeatures.codeQLEnabled` | 是否使用 CodeQL **default setup**；`false` 只代表沒有用 default setup，可能改用 pipeline 中的 advanced setup |
| `codeSecurityFeatures.autofixEnabled` | Copilot Autofix（Learn 標示為有限公開預覽） |
| `codeSecurityFeatures.malwareAlertsEnabled` | 是否針對相依套件中的惡意軟體產生警示 |
| `enablementOnCreateSettings.enableSecretProtectionOnCreate`、`enableCodeSecurityOnCreate` | 新建立的儲存庫是否自動啟用對應產品；子功能另有 `enableBlockPushesOnCreate`、`enableCodeQLOnCreate` 等旗標 |
| `*LastChangedDate`、`*ChangedBy` | 最後變更啟用狀態的時間與使用者 VSID |

- REST 文件說明布林欄位「Null is never explicitly set」：`null` 代表從未明確設定，視為未啟用，並在結果中註明「未設定」。
- 回應可能還有 `isBundledSKU` 與 `codeSecurityFeatures.dependabotEnabled`，REST 參考沒有定義它們的意義：只原樣列出，不據以判斷。

## 步驟

1. **取得儲存庫名稱對照**（整個組織一次取得）：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/git/repositories?api-version=7.1"
   ```

   以 `id` 對照 `name`、`project.id`、`project.name`、`webUrl`。`isDisabled` 為 `true` 標示「已停用」；沒有 `defaultBranch` 欄位的是空儲存庫，標示「空儲存庫」。這兩類不列入涵蓋率分母，另外列出。

2. **取得啟用狀態**。整個組織：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://advsec.dev.azure.com/$AZURE_DEVOPS_ORG/_apis/management/enablement?includeAllProperties=true&api-version=7.2-preview.3"
   ```

   只查一個專案：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://advsec.dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/management/enablement?includeAllProperties=true&api-version=7.2-preview.3"
   ```

   只查一個儲存庫（`{repository}` 可以是名稱或 ID）：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://advsec.dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/management/repositories/{repository}/enablement?includeAllProperties=true&api-version=7.2-preview.3"
   ```

   - 這些資源只有 preview 版本，必須帶 preview 版本號。
   - 組織與專案回應有 `enablementOnCreateSettings` 與 `reposEnablementStatus[]`；每筆有 `projectId`、`repositoryId`、`secretProtectionFeatures`、`codeSecurityFeatures`。儲存庫回應直接是一筆。
   - 專案回應的 `enablementOnCreateSettings` 可能是 `null`，代表專案沒有自己的自動啟用設定，改以組織層級的值判斷並註明。

3. **逐一判斷**（已停用與空儲存庫略過）：

   | # | 檢查 | ✅ | ❌ | ❓ |
   |---|---|---|---|---|
   | 1 | Secret Protection | `secretProtectionEnabled` 為 `true` | `false` 或 `null` | — |
   | 2 | Push protection | `blockPushes` 為 `true` | `false`（Secret Protection 未啟用時寫「—」） | `null`（沒有帶 `includeAllProperties=true`） |
   | 3 | Code Security | `codeSecurityEnabled` 為 `true` | `false` 或 `null` | — |
   | 4 | 相依套件掃描自動加入 | `dependencyScanningInjectionEnabled` 為 `true` | — | `false` 或 `null`：可能已在 pipeline 手動加入掃描工作，請確認 |
   | 5 | CodeQL default setup | `codeQLEnabled` 為 `true` | — | `false` 或 `null`：可能使用 advanced setup，請確認 |
   | 6 | Copilot Autofix、惡意軟體警示 | 只列出目前值，不判斷 | | |

   Code Security 未啟用時，第 4～6 項寫「—」。

4. **新儲存庫自動啟用**：組織層級（與使用者指定的專案）的 `enableSecretProtectionOnCreate`、`enableCodeSecurityOnCreate` 為 `true` 標 ✅，`false` 或 `null` 標 ❌。

5. **舊版回應**：`7.2-preview.3` 被拒絕時，改帶 `api-version=7.2-preview.1`。舊版回應沒有產品拆分，欄位是 `enableOnCreate`、`reposEnablementStatus[].advSecEnabled`、`advSecEnablementLastChangedDate`，以及 `advSecEnablementFeatures`（`dependencyScanningInjectionEnabled`、`codeQLEnabled` 等）；以 `advSecEnabled` 同時判斷第 1、3 項，第 2 項標 ❓，並在結果中註明使用舊版 API。

範例：「Contoso 組織有多少儲存庫還沒啟用 Advanced Security？」、「檢查 Contoso 專案的 secret scanning 與 push protection 是否全面開啟」、「新建立的儲存庫會自動啟用 Code Security 嗎？」

## 呈現結果

日期格式為 `YYYY/MM/DD`，時間轉成使用者當地時區，無法判斷時標明 UTC。依序呈現：

1. **概要**：稽核範圍、儲存庫總數（不含已停用與空儲存庫）、Secret Protection 與 Code Security 的啟用數與涵蓋率、push protection 開啟數、新儲存庫自動啟用狀態。

2. **依專案彙總**：

   | 專案 | 儲存庫 | Secret Protection | Push protection | Code Security | 相依套件自動掃描 | CodeQL default setup |
   |---|---|---|---|---|---|---|
   | Contoso | 12 | 3（25%） | 3 | 1（8%） | 1 | 0 |

3. **未啟用的儲存庫**：依專案分組，列出儲存庫名稱（連結 `webUrl`）與缺少的產品；全部啟用時明確說明。數量很多時先列出前 50 筆並告知總數。

4. **已啟用儲存庫的功能狀態**：每個已啟用任一產品的儲存庫一列，欄位為第 1～6 項檢查，另加最後變更日期。

5. **啟用步驟**（依 Learn；啟用前提醒使用者會開始依活躍提交者計費，**Begin billing** 由使用者自行確認）：
   - 單一儲存庫：**Project settings** > **Repos** > **Repositories** > 儲存庫，切換 **Secret Protection** 或 **Code Security**，再選 **Begin billing**；相依套件自動掃描在 **Options** 勾選 dependency scanning default setup。
   - 整個專案：**Project settings** > **Repos** > **Settings** 分頁 > **Enable all**，選擇產品後 **Begin billing**；另外切換 **Automatically enable Advanced Security for new repositories**。
   - 整個組織：**Organization settings** > **Repositories** > **Enable all**，選擇產品後 **Begin billing**；另外切換 **Automatically enable Advanced Security for new projects**。
   - Push protection：在儲存庫設定頁開啟 push protection；Learn 說明它只檢查啟用之後的推送，既有機密由儲存庫 secret scanning 找出。
   - 啟用後要把 Advanced Security 結果納入 PR 把關，見 `repos-branch-policy-audit` 的狀態檢查項目；要查看警示改用 `security-alert-review`。

   最後提醒這份報告沒有修改任何設定。
