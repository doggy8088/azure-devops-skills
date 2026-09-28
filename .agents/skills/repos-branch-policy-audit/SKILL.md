---
name: repos-branch-policy-audit
description: 以 Azure DevOps REST API 稽核儲存庫預設分支的 branch policy，比對 Microsoft 建議基準（至少兩位審查者、建立者投票不計入、推送後重新核准、連結工作項目、建置驗證、留言須解決），列出沒有任何原則的儲存庫，並附上網頁操作的修正步驟；只讀取、不修改。當使用者要稽核分支保護、檢查 branch policy 是否符合最佳實務，或找出沒有保護的儲存庫時使用；要查單一 PR 為何被擋改用 `repos-pull-request-review`。
compatibility: 需要 curl、可連線至 dev.azure.com 的網路，以及 AZURE_DEVOPS_ORG 與 AZURE_DEVOPS_PAT 環境變數。
---

# Branch Policy 稽核

這個 Skill 以**專案**為範圍，逐一檢查每個儲存庫的**預設分支**套用了哪些 branch policy，對照 Microsoft Learn 的安全基準產生檢查表。全程只呼叫 GET，不建立、不修改任何原則；修正一律提供網頁操作步驟，由使用者自行處理。

## 專案選擇

- 使用者有提供專案名稱（例如「Contoso 專案」）就直接使用，不呼叫專案清單 API。
- 沒有提供時，請使用者提供一次。
- 仍未提供時，列出專案讓使用者選擇：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/projects?%24top=500&api-version=7.1"
  ```

- 使用者要「所有專案」時，對清單中的每個專案重複步驟；專案超過 20 個時先告知數量並確認一次。
- 使用者提供或選定專案之前不要繼續。

使用者指定儲存庫時只稽核那些儲存庫；沒有指定時稽核專案內所有儲存庫。

## 連線與驗證

一律用 `curl` 直接呼叫 Azure DevOps REST API，每個指令都自帶驗證參數。

- **組織**：讀取環境變數 `AZURE_DEVOPS_ORG`（組織名稱，例如 `contoso`）。未設定時詢問使用者一次；使用者給的是網址時，從 `https://dev.azure.com/{org}` 或 `https://{org}.visualstudio.com` 取出組織名稱。
- **權杖**：PAT 以 Basic 驗證送出：`-u ":$AZURE_DEVOPS_PAT"`；只設定了 `AZURE_DEVOPS_EXT_PAT`（Azure CLI azure-devops 擴充套件使用的變數）時，改用它。改用 Microsoft Entra ID 權杖時，換成 `-H "Authorization: Bearer $AZURE_DEVOPS_TOKEN"`。都未設定時，請使用者在自己的終端機設定 `AZURE_DEVOPS_PAT` 後重試，並停止。權杖只從環境變數讀取：不要請使用者把權杖貼進對話，也不要印出或寫入檔案。
- **PAT 權限範圍**：Code (Read)；列出專案另需 Project and Team (Read)。這個 Skill 不需要任何寫入權限。
- **編碼**：路徑中的專案名稱要 URL 編碼（空白寫成 `%20`）；查詢字串中的 `$` 寫成 `%24`（例如 `%24top`），避免被 shell 展開。
- **錯誤**：HTTP 401、403、203 或回應是 HTML 登入頁，代表權杖無效、過期或權限不足，回報後停止；HTTP 404 代表組織、專案或儲存庫有誤。
- **Windows PowerShell**：呼叫 `curl.exe`（而非 `curl` 別名），環境變數寫成 `$env:AZURE_DEVOPS_PAT`；PowerShell 不支援 `<<'EOF'`，先把 JSON 主體寫入暫存檔，再以 `--data-binary "@body.json"` 送出。

## 稽核基準

依據 [Secure repositories and pull requests](https://learn.microsoft.com/en-us/azure/devops/repos/git/secure-repositories-pull-requests) 的「Security baseline for most teams」與 [Branch policies](https://learn.microsoft.com/en-us/azure/devops/repos/git/branch-policies)。以原則類型 GUID 比對（`type.id`），不要用 `displayName` 比對。

**基準項目**（未通過標 ❌）：

| # | 檢查 | 原則類型（GUID） | 通過條件 |
|---|---|---|---|
| 1 | 至少兩位審查者 | Minimum number of reviewers（`fa4e907d-c16b-4a4c-9dfa-4906e5d171dd`） | 啟用且 blocking，`minimumApproverCount` ≥ 2 |
| 2 | 建立者投票不計入 | 同上 | `creatorVoteCounts` 為 `false`（UI：**Allow requestors to approve their own changes** 關閉） |
| 3 | 推送後需重新核准 | 同上 | `resetOnSourcePush`、`requireVoteOnLastIteration` 或 `requireVoteOnEachIteration` 任一為 `true` |
| 4 | 連結工作項目 | Work item linking（`40e92b44-2fe1-4dd6-b3d8-74a9c21d0c6e`） | 啟用且 blocking |
| 5 | 建置驗證 | Build（`0609b952-1397-4640-95ec-e00a01b2c241`） | 至少一筆啟用且 blocking |
| 6 | 留言須解決 | Comment requirements（`c6a1889d-b943-4856-b76f-9e46bb6b0df2`） | 啟用且 blocking |

**參考項目**（依團隊需求決定，不算不合格；未設定標 ❌ 並註明「參考」）：

| # | 檢查 | 原則類型（GUID） | 判斷方式 |
|---|---|---|---|
| 7 | 最近推送者不能核准 | Minimum number of reviewers | `blockLastPusherVote` 為 `true`（UI：**Prohibit the most recent pusher from approving their own changes**；Learn 建議需要職責分離時開啟） |
| 8 | 限制合併類型 | Require a merge strategy（`fa4e907d-c16b-4a4c-9dfa-4916e5d171ab`） | 列出允許的類型：`allowNoFastForward`（Basic merge）、`allowSquash`（Squash）、`allowRebase`（Rebase and fast-forward）、`allowRebaseMerge`（Semi-linear merge）。Learn 說明這是歷史治理設定，不是直接的安全控制 |
| 9 | 敏感路徑的必要審查者 | Required reviewers（`fd2167ab-b0be-447a-8ec8-39368250530e`） | 啟用且 blocking，`settings.filenamePatterns` 有值（沒有路徑篩選代表套用到所有 PR） |
| 10 | Advanced Security 狀態檢查 | Status（`cbdc66da-9728-4af8-aada-9a5a32e4a226`） | `settings.statusGenre` 為 `AdvancedSecurity`，`settings.statusName` 為 `NewHighAndCritical` 或 `AllHighAndCritical`；儲存庫未啟用 Advanced Security 時標 ❓ |

判斷原則：

- 忽略 `isDeleted` 為 `true` 的原則；`isEnabled` 為 `false` 的原則視同不存在。`isBlocking` 為 `false`（UI 的 Optional）不算通過基準項目。
- 舊的原則可能沒有較新的設定欄位（例如 `requireVoteOnLastIteration`、`blockLastPusherVote`），欄位不存在時視為 `false`（未勾選）。
- 同一類型有多筆原則時，取最嚴格且啟用、blocking 的那一筆判斷。
- Build 原則的 `settings.manualQueueOnly` 為 `true`（Trigger 為 Manual）或 `settings.filenamePatterns` 有值（只有特定路徑才觸發）時，仍算通過，但在細節中註明。
- `allowDownvotes` 為 `true` 代表有人投拒絕仍可完成，在細節中註明。

## 步驟

1. **確認原則類型 GUID**。取得專案可用的原則類型，確認上表的 GUID 都存在；某個 GUID 不在清單中時，該欄標 ❓：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/policy/types?api-version=7.1"
   ```

   `value[].id` 是 GUID，`value[].displayName` 只用於顯示。

2. **列出儲存庫**：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/repositories?api-version=7.1"
   ```

   - `isDisabled` 為 `true`：標示「已停用」，略過。
   - 回應中沒有 `defaultBranch` 欄位（空儲存庫不會回傳這個欄位）：標示「空儲存庫」，略過。
   - 其餘記下 `id`、`name`、`defaultBranch`（例如 `refs/heads/main`）與 `webUrl`。

3. **取得套用在預設分支的原則**。每個儲存庫呼叫一次，帶 `repositoryId` 與 `refName`；回應包含只設在這個分支的原則，也包含專案層級（跨儲存庫）的原則：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -D - \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/git/policy/configurations?repositoryId={repositoryId}&refName={defaultBranch}&api-version=7.1"
   ```

   - `-D -` 用來看回應標頭：有 `x-ms-continuationtoken` 時，把它的值加上 `&continuationToken={token}` 再取下一頁，直到沒有這個標頭。
   - 每筆原則有 `type.id`、`isEnabled`、`isBlocking`、`isDeleted` 與 `settings`；`settings.scope[]` 中 `repositoryId` 為 `null` 的是專案層級原則，在細節中標註「專案層級」。
   - 分支名稱含特殊字元時，`refName` 要 URL 編碼。
   - 回傳 0 筆代表這個預設分支沒有任何原則，列入「沒有任何原則的儲存庫」。

4. **評估**。依「稽核基準」逐項判斷每個儲存庫：✅ 通過、❌ 未通過、❓ 無法判斷（例如原則類型不存在、儲存庫未啟用 Advanced Security）。

5. **需要人工確認的項目**。API 無法可靠地判斷誰有繞過原則的權限，請使用者到網頁檢查並在結果中列出：
   - **Bypass policies when completing pull requests** 與 **Bypass policies when pushing**：只保留給少數受信任的管理員。
   - **Force push (rewrite history, delete branches and tags)**、**Edit policies**、**Manage permissions**。
   - 位置：**Project settings** > **Repositories** > 儲存庫 > **Security**（儲存庫層級），或 **Repos** > **Branches** > 分支的 **…** > **Branch security**（分支層級）。
   - 依據：[Secure repositories and pull requests](https://learn.microsoft.com/en-us/azure/devops/repos/git/secure-repositories-pull-requests) 的「Limit branch permissions」與 [Branch policies](https://learn.microsoft.com/en-us/azure/devops/repos/git/branch-policies) 的「Allow policy bypass when needed」。

範例：「稽核 Contoso 專案所有儲存庫的 branch policy」、「MyApp 的 main 分支保護符合 Microsoft 建議嗎？」、「哪些儲存庫的預設分支完全沒有原則？」

## 呈現結果

分支名稱去掉 `refs/heads/`；儲存庫名稱連結到 `webUrl`。依序呈現：

1. **概要**：專案、稽核的儲存庫數、全部基準項目通過的數量、沒有任何原則的數量、略過的空儲存庫與已停用儲存庫數量。

2. **基準檢查表**（每個儲存庫一列）：

   | 儲存庫 | 預設分支 | 審查者 ≥2 | 建立者不計 | 推送後重核 | 工作項目 | 建置驗證 | 留言解決 |
   |---|---|---|---|---|---|---|---|
   | [MyApp](https://dev.azure.com/contoso/Contoso/_git/MyApp) | main | ✅ | ❌ | ✅ | ✅ | ❌ | ✅ |

   審查者欄可附上實際人數，例如 `❌（1）`。

3. **參考項目表**：儲存庫、最近推送者不能核准、允許的合併類型、路徑審查者、Advanced Security 狀態檢查。

4. **沒有任何原則的儲存庫**：列出名稱與預設分支；沒有時明確說明「所有儲存庫的預設分支都至少有一項原則」。另列出空儲存庫與已停用的儲存庫。

5. **修正步驟**：依未通過的項目分組，列出受影響的儲存庫與網頁操作：
   - 開啟原則設定：**Repos** > **Branches** > 預設分支的 **…** > **Branch policies**，或 **Project settings** > **Repositories** > 儲存庫 > **Policies** > **Branch Policies** > 分支。
   - 審查者：開啟 **Require a minimum number of reviewers**，**Minimum number of reviewers** 設為 `2`，取消 **Allow requestors to approve their own changes**，在 **When new changes are pushed** 選 **Require at least one approval on the last iteration** 或 **Reset all approval votes**。
   - 工作項目：開啟 **Check for linked work items**，選 **Required**。
   - 建置驗證：在 **Build validation** 按 **+**，選擇驗證用的 pipeline，Trigger 選 **Automatic**，**Policy requirement** 選 **Required**。
   - 留言：開啟 **Check for comment resolution**，選 **Required**。
   - Advanced Security 狀態檢查：先完成 Advanced Security 掃描與建置驗證，再於 **Status checks** 按 **+**，**Status to check** 填 `AdvancedSecurity/NewHighAndCritical`。
   - 多個儲存庫都缺同一項時，建議設定跨儲存庫原則：**Project settings** > **Repositories** > **Policies** 分頁 > **Branch Policies** 的 **+** > **Protect the default branch of each repository**（見 [Set Git repository settings](https://learn.microsoft.com/en-us/azure/devops/repos/git/repository-settings)）。

   每組修正附上對應的 Microsoft Learn 連結；最後提醒這份報告沒有修改任何設定。

6. **需要人工確認**：列出步驟 5 的繞過權限檢查項目。
