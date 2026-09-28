# Azure DevOps Services 最佳實踐與 REST API 研究筆記

- **日期**：2026-09-28
- **目的**：決定這個儲存庫要新增哪些 Agent Skills，並確認每個 Skill 依據的官方最佳實踐與 REST API 細節。
- **方法**：只採用第一手來源（Microsoft Learn、Microsoft 官方 GitHub 儲存庫），並以唯讀呼叫實測 REST API（寫入類只做 `validateOnly`、`previewRun` 乾跑）。
- **文件網址慣例**：REST API 參考頁以 `D:<path>` 表示 `https://learn.microsoft.com/en-us/rest/api/azure/devops/<path>?view=azure-devops-rest-7.1`。

---

## 1. Azure Repos

來源：[Secure repositories and pull requests](https://learn.microsoft.com/en-us/azure/devops/repos/git/secure-repositories-pull-requests)、[Git branching guidance](https://learn.microsoft.com/en-us/azure/devops/repos/git/git-branching-guidance)、[About pull requests](https://learn.microsoft.com/en-us/azure/devops/repos/git/about-pull-requests)、[Repository settings and policies](https://learn.microsoft.com/en-us/azure/devops/repos/git/repository-settings)

- `main` 與 release 分支至少 2 位審查者；關閉「允許要求者核准自己的變更」；推送新變更後重設投票；可選擇啟用「禁止最後推送者核准自己的變更」。
- 「Check for linked work items」、「Build validation」、「Check for comment resolution」設為 Required。
- 在敏感路徑（基礎架構、驗證、pipeline 範本）自動加入審查者。
- 「Bypass policies when completing pull requests / when pushing」只給少數管理者群組；Force push、Edit policies、Manage permissions 要審慎授權。
- 加上 `AdvancedSecurity/NewHighAndCritical` 狀態檢查，警示清完後改為 `AllHighAndCritical`。
- 「Limit merge types」用來控制歷史，官方說明它「不是直接的安全控制」。
- 每個變更用 feature 分支，命名如 `users/`、`feature/`、`bugfix/`、`hotfix/`，可用「Require branch folders」強制；只能透過 PR 合併到 `main`，並要求建置成功。
- 審查者以 2 位為佳，避免同一批人包辦大多數 PR；PR 描述要足夠詳細，可用 PR 範本；範圍外的好建議轉成新的工作項目。
- 所有 branch policy 預設關閉；只要設定任一原則，該分支就必須透過 PR 修改且無法刪除。專案層級可啟用「Protect the default branch of each repository」。

## 2. Azure Pipelines 安全

來源：[Securing Azure Pipelines](https://learn.microsoft.com/en-us/azure/devops/pipelines/security/overview)、[Job access tokens](https://learn.microsoft.com/en-us/azure/devops/pipelines/process/access-tokens)、[Security approach](https://learn.microsoft.com/en-us/azure/devops/pipelines/security/approach)、[Protect resources](https://learn.microsoft.com/en-us/azure/devops/pipelines/security/resources)、[Templates for security](https://learn.microsoft.com/en-us/azure/devops/pipelines/security/templates)、[Secrets](https://learn.microsoft.com/en-us/azure/devops/pipelines/security/secrets)、[Other security considerations](https://learn.microsoft.com/en-us/azure/devops/pipelines/security/misc)

- 使用 YAML，不用 Classic；啟用「Disable creation of classic build/release pipelines」（Sprint 226 起新組織預設開啟）。
- 在組織層級啟用兩個「Limit job authorization scope to current project…」設定，以及「Protect access to repositories in YAML pipelines」。
- 啟用「Enable shell tasks arguments validation」與「Limit variables that can be set at queue time」（搭配 `settableVariables`）。
- 不要開啟「Make secrets available to builds of forks」；fork 的建置考慮只允許手動觸發，並在 Microsoft-hosted agent 上執行。
- 服務連線使用 workload identity federation，範圍限縮到特定資源群組，並加上 branch control check；避免 classic 服務連線。閒置 100 天的服務連線會開始被自動停用。
- 受保護資源（服務連線、變數群組、agent pool、secure file、環境、儲存庫）只授權給特定 pipeline，不要開啟「Open access」；加上核准、branch control 與 required template 等 checks。
- 使用固定 tag 或分支的 `extends` 範本，並以 required-template check 強制。
- 機密放在 Key Vault，不寫進 YAML；不要用 logging command 設定機密，改用環境變數對應；定期檢視與輪替。
- 保留原則只在專案層級設定；用 lease 保留特定 run。

以 REST API 讀取設定：`GET {org}/{project}/_apis/build/generalsettings?api-version=7.1`（只有專案層級；組織層級的同名路徑回 404，實測，組織層級設定需到網頁確認），欄位包括 `enforceJobAuthScope`、`enforceJobAuthScopeForReleases`、`enforceReferencedRepoScopedToken`、`enforceSettableVar`、`enableShellTasksArgsSanitizing`、`disableClassicBuildPipelineCreation`、`disableClassicReleasePipelineCreation`、`enforceNoAccessToSecretsFromForks`（D:build/general-settings/get）。

## 3. Azure Boards

來源：[Best practices for Agile project management](https://learn.microsoft.com/en-us/azure/devops/boards/best-practices-agile-project-management)、[Scrum best practices](https://learn.microsoft.com/en-us/azure/devops/boards/sprints/best-practices-scrum)、[Create your product backlog](https://learn.microsoft.com/en-us/azure/devops/boards/backlogs/create-your-backlog)、[Link work items to Git](https://learn.microsoft.com/en-us/azure/devops/boards/backlogs/connect-work-items-to-git-dev-ops)

- 團隊約 5–9 人，各有自己的 area path 與預設 area／iteration；Sprint 長度 1–4 週，**至少預先排好 6 個 iteration**。
- Feature backlog 保持排序；需求大小要能在一個 iteration 內完成；每個需求都要有父 Feature。
- 允收條件與完成定義要清楚（Acceptance Criteria「定義了 Done」）；各團隊用一致的方式估點；每個項目指派給一位使用者。
- 相依關係用 Predecessor/Successor 連結，並搭配一致的 `dependency` 標籤。
- 設定 Sprint 目標、每天更新狀態；看板欄位設 WIP 限制，追蹤 cycle time 與累積流程圖。
- Sprint 規劃不超過 4 小時；寫下 Bug 分級準則；Sprint 開始後避免追加工作。
- 從工作項目建立分支以自動連結；官方範例檢查項目是「進行中但沒有開發連結的 Story」。
- [Create your product backlog](https://learn.microsoft.com/en-us/azure/devops/boards/backlogs/create-your-backlog) 寫 backlog 最多顯示 999 個項目，但 Services 的 [Work tracking object limits](https://learn.microsoft.com/en-us/azure/devops/organizations/settings/work/object-limits) 列的是 10,000；999 是 Azure DevOps Server 的數字。

## 4. 安全性與組織管理

來源：[Make your Azure DevOps secure](https://learn.microsoft.com/en-us/azure/devops/organizations/security/security-overview)、[Use personal access tokens](https://learn.microsoft.com/en-us/azure/devops/organizations/accounts/use-personal-access-tokens-to-authenticate)、[Manage PATs with policies](https://learn.microsoft.com/en-us/azure/devops/organizations/accounts/manage-pats-with-policies-for-administrators)、[Auditing](https://learn.microsoft.com/en-us/azure/devops/organizations/audit/azure-devops-auditing)、[Configure GitHub Advanced Security features](https://learn.microsoft.com/en-us/azure/devops/repos/security/configure-github-advanced-security-features)

- 優先使用 Microsoft Entra 權杖、服務主體或受控識別，而不是 PAT；啟用 Conditional Access；不需要時關閉外部來賓存取。
- PAT 只用組織範圍、每個工具一個、最小權限、最短期限；**全域 PAT 於 2026-12-01 停止運作**。
- PAT Lifecycle、Profiles 等 API 需要 Entra 權杖，無法用 PAT 呼叫。
- 移除閒置使用者並撤銷其 PAT；權限給群組而非個人；以 PIM 做 just-in-time 存取。
- 稽核預設關閉，要啟用「Log Audit Events」；事件保留 90 天，需要更久要串流出去。
- 「Automatically revoke leaked personal access tokens」預設開啟，保持開啟。
- Advanced Security：啟用「對新儲存庫／專案自動啟用」；啟用後 push protection 自動開啟；開啟相依套件弱點掃描；程式碼掃描建議從 CodeQL 預設設定開始。
- 公開專案已退場：無法再建立，既有公開專案將於 2027 年轉為私人。

## 5. Artifacts、Test Plans、Wiki

- Artifacts（[Best practices](https://learn.microsoft.com/en-us/azure/devops/artifacts/concepts/best-practices)、[Upstream sources](https://learn.microsoft.com/en-us/azure/devops/artifacts/concepts/upstream-sources)）：每個儲存庫一個 feed，外部套件透過 upstream source 取得；以 view 當作發行通道（`@local` → `@prerelease` → `@release`）；及早啟用保留原則。
- Test Plans（[Create a test plan](https://learn.microsoft.com/en-us/azure/devops/test/create-a-test-plan)）：每個 Sprint 或里程碑一個 test plan，在 backlog 定義後建立；重複使用 test case。
- Wiki（[Manage wiki permissions](https://learn.microsoft.com/en-us/azure/devops/project/wiki/manage-readme-wiki-permissions)）：權限來自 wiki 背後的 Git 儲存庫，Contributors 預設可編輯；個別頁面無法單獨設權限。

---

## 6. REST API 查證重點

「實測」表示已以唯讀呼叫或乾跑驗證；其餘依 Microsoft Learn 文件。

### 共通

- **分頁方式因 API 而異**：
  - `$top`/`$skip`：PR、commits、test runs/results、teams。
  - 回應本文 `nextSkip`：PR iteration changes。
  - 回應標頭 `x-ms-continuationtoken`：projects、builds、definitions、policy configurations、test plans/suites/points、advsec alerts。
  - 回應本文 `continuationToken`：WIT comments、audit log。
- **preview 限定**：WIT Comments（`7.1-preview.4`，帶 `7.1` 會回 400，實測）、Policy Evaluations、Check Configurations、Pipeline Permissions（`7.1-preview.1`）、builds 之間的 changes／work items（`7.1-preview.2`）、Advanced Security（`7.2-preview.1`，實測）。
- **目前使用者**：`GET {org}/_apis/connectionData`（不帶 api-version）→ `authenticatedUser.id`（實測 PAT 可用）；群組成員資格用 `vssps.dev.azure.com/{org}/_apis/identities?identityIds={id}&queryMembership=Direct` → `memberOf` 描述子清單（實測）。

### Work items

- `workitemsbatch` 同時指定 `fields` 與 `$expand` 會回 HTTP 400「The expand parameter can not be used with the fields parameter」（實測）。每次最多 200 個 ID。
- 建立：`POST .../wit/workitems/${type}`，`Content-Type: application/json-patch+json`；`validateOnly=true` 回 200、沒有 `id`、預設欄位會補上，不會建立項目（實測）；無效值回 400 `RuleValidationException`（實測）。
- 更新：PATCH 前先送 `{"op":"test","path":"/rev","value":N}` 做樂觀並行控制（D:wit/work-items/update）。
- Team iterations 的 `timeFrame`：團隊沒有排下一個 Sprint 時，已結束的最後一個 Sprint 仍標為 `current`（實測）；Analytics 的 `Iterations.IsEnded` 較可靠。
- WIQL 支援 `@Me`、`@project`（實測）；結果上限 20,000 筆且會無聲截斷；查詢字串上限 32,000 字元（[Work tracking object limits](https://learn.microsoft.com/en-us/azure/devops/organizations/settings/work/object-limits)）。
- 連結方向：`System.LinkTypes.Dependency-Forward` = Successor（對方要等這個項目）、`Dependency-Reverse` = Predecessor；`Hierarchy-Forward` = Child、`Hierarchy-Reverse` = Parent（[Link type reference](https://learn.microsoft.com/en-us/azure/devops/boards/queries/link-type-reference)）。

### Pull requests

- 專案層級清單 `GET {org}/{project}/_apis/git/pullrequests`；**沒有組織層級清單**；清單中的 `description` 截斷為 400 字（實測）。
- `searchCriteria.reviewerId` 只比對直接指派的審查者，指派給群組的 PR 要另外比對成員資格。
- 變更檔案：`iterations/{id}/changes?$compareTo=0` 以 merge base 比較（實測）；伺服器不提供 diff，要以 `git/items` 取 `commonRefCommit` 與 `sourceRefCommit` 兩版內容後本機比較（實測；不存在的檔案回 404 `TF401174`）。
- Policy evaluations 的 `artifactId` 為 `vstfs:///CodeReview/CodeReviewId/{projectGuid}/{prId}`，必須是專案 GUID（實測）。
- `pullrequestquery` 以 commit 反查 PR，成功時回 201，SHA 必須是完整 40 碼（實測）。
- Update PR 文件列出的可更新欄位：status、title、description（≤4000）、completionOptions、mergeOptions、`autoCompleteSetBy.id`、targetRefName；其他欄位會被忽略（D:git/pull-requests/update）。
- `searchCriteria.fromCommitId`/`toCommitId` 是字母順序的上下界，不是祖先範圍（D:git/commits/get-commits）。

### Branch policies

- `GET {org}/{project}/_apis/git/policy/configurations?repositoryId=&refName=` 回傳套用在該分支的原則（D:git/policy-configurations/get）。
- 以 policy type GUID 比對，不要看 displayName（Learn 部分範例把類型標錯）：最少審查者 `fa4e907d-c16b-4a4c-9dfa-4906e5d171dd`、建置驗證 `0609b952-1397-4640-95ec-e00a01b2c241`、連結工作項目 `40e92b44-2fe1-4dd6-b3d8-74a9c21d0c6e`、留言需解決 `c6a1889d-b943-4856-b76f-9e46bb6b0df2`、合併策略 `fa4e907d-c16b-4a4c-9dfa-4916e5d171ab`、必要審查者 `fd2167ab-b0be-447a-8ec8-39368250530e`（來源：D:policy/configurations/create、[azure-devops-cli-extension policy.py](https://github.com/Azure/azure-devops-cli-extension/blob/master/azure-devops/azext_devops/dev/repos/policy.py)）。

### Pipelines

- `POST {org}/{project}/_apis/pipelines/{id}/runs`：`previewRun:true` 回 200，`id` 為 -1、`state` 為 unknown，並回傳展開後的 `finalYaml`，不會建立 run（實測）；未知的 template parameter 回 400 `PipelineValidationException`（實測）。
- Runs API 只適用 YAML pipeline；`GET pipelines/{id}` 的 `configuration.type` 為 `yaml` 或 `designerJson`（Classic，實測），Classic 要改用 `POST build/builds`。
- Approvals：`GET {org}/{project}/_apis/pipelines/approvals?state=pending&$expand=steps`；更新用 PATCH（D:approvalsandchecks/approvals）。
- Environments 清單在文件中要求 `vso.environment_manage`，即使只是讀取（D:distributedtask/environments/list）。

### 其他主機

- Code／Work item／Wiki search：`https://almsearch.dev.azure.com/{org}/{project}/_apis/search/{code|workitem|wiki}searchresults`（D:search）；程式碼搜尋需要安裝 Code Search 擴充套件。
- Analytics OData：`https://analytics.dev.azure.com/{org}/{project}/_odata/v4.0-preview/`，scope Analytics (Read)；`WorkItems` 有 `CycleTimeDays`、`LeadTimeDays`、`StateCategory`；不可在同一查詢混用 `$filter` 與 `$apply`（[Lead/cycle time sample](https://learn.microsoft.com/en-us/azure/devops/report/powerbi/sample-boards-leadcycletime)、[Aggregated data](https://learn.microsoft.com/en-us/azure/devops/report/extend-analytics/aggregated-data-analytics)）。
- Advanced Security 啟用狀態：`https://advsec.dev.azure.com/{org}[/{project}]/_apis/management/enablement`。Learn 目前的版本是 `7.2-preview.3`（搭配 `includeAllProperties=true`，新結構分為 Secret Protection 與 Code Security）；`7.2-preview.1` 仍回舊結構（`advSecEnabled`、`enableOnCreate`），`7.2-preview.2` 回 400（皆實測，PAT 可用）。儲存庫未啟用時警示 API 回 `VS2150009`（實測）。
- Pipeline permissions：`resourceType` 支援 endpoint、queue、variablegroup、securefile、environment、repository；repository 的 `resourceId` 必須是 `{projectId}.{repoId}`，且只接受 preview 版本（實測）。

---

## 7. 能力盤點與缺口

- Microsoft 的 [Azure DevOps MCP Server](https://github.com/microsoft/azure-devops-mcp/blob/main/docs/TOOLSET.md) 涵蓋 core、work、work items、repositories、pipelines、wiki、test plans、search、advanced security；但沒有 pipeline 核准、環境、變數群組、branch policy、Artifacts feed。
- MCP 儲存庫中反應最多的功能需求：PR diff、建立 PR 討論串、PR commits、以參數觸發 pipeline（[issues](https://github.com/microsoft/azure-devops-mcp/issues)）。
- 上游 [microsoft/azure-devops-skills](https://github.com/microsoft/azure-devops-skills) 仍只有 7 個 Skills；綁定第三方擴充套件的提案被婉拒（PR #12），顯示偏好通用型 Skills。
- 本儲存庫原本的 7 個 Skills 完全沒有 Azure Repos／PR，而 PR 是所有來源中需求最高的領域。

## 8. 新增 Skills 與依據

| Skill | 依據 |
|---|---|
| `repos-pull-request-review`、`repos-pull-request-write` | §1 PR 流程、§7 需求最高的缺口 |
| `repos-code-search` | §7 缺口（程式碼搜尋、讀檔、commit 搜尋） |
| `boards-work-item-write`、`boards-query` | §7 缺口；§6 Work items |
| `pipelines-run`、`pipelines-approvals` | §2 approvals and checks；§7 缺口 |
| `wiki-pages` | §5 Wiki；§7 缺口 |
| `repos-branch-policy-audit` | §1 分支原則基準 |
| `pipelines-security-audit` | §2 Pipeline 安全設定與受保護資源 |
| `boards-backlog-audit` | §3 Backlog 衛生 |
| `security-enablement-audit` | §4 Advanced Security 啟用 |
| `pipelines-release-notes`、`pipelines-test-results`、`test-plans-progress`、`boards-sprint-metrics` | §3 cycle time／velocity、§5 Test Plans、§7 缺口 |

## 9. 評估後未納入

- **PAT 稽核**：PAT Lifecycle API 需要 Entra 權杖，與「只靠 PAT」的設計衝突。
- **組織原則稽核**：沒有正式文件化的 API。
- **Artifacts feed 稽核**、**稽核記錄檢視**：本次未選入，可作為後續候選。
