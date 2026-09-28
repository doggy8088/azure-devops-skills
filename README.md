# Azure DevOps Agent Skills

這個儲存庫收錄一組處理 Azure DevOps 工作的通用 Agent Skills。每個 Skill 是一個獨立資料夾，內含一份 `SKILL.md`，教 AI 代理以一致、可重複的流程完成某項 Azure DevOps 任務。

設計原則：

- **不綁定任何 AI 工具**：遵循 [Agent Skills](https://agentskills.io) 開放格式，任何能讀取 `SKILL.md` 並執行 shell 指令的代理都能使用。
- **直接呼叫 REST API**：所有操作都用 `curl` 呼叫 [Azure DevOps REST API](https://learn.microsoft.com/rest/api/azure/devops/)，不需要 MCP 伺服器、IDE 擴充套件或 Azure CLI。
- **每個 Skill 自成一體**：連線方式、API 端點、流程與呈現格式都寫在同一份 `SKILL.md`，可以單獨複製使用。

---

## 📁 目錄結構

```text
.agents/
└── skills/
    └── <skill-name>/
        └── SKILL.md    # Skill 的中繼資料（frontmatter）與操作指示
```

---

## 🧩 可用的 Skills

| Skill | 用途 | 需要的 PAT 權限範圍 |
|---|---|---|
| `boards-backlog-summary` | 取得團隊需求層級待辦清單中，指派給目前使用者的進行中項目，附父子階層並依優先順序排序 | Work Items (Read)、Project and Team (Read) |
| `boards-my-work` | 列出指派給目前使用者、尚未完成的工作項目，依類型分組 | Work Items (Read)、Project and Team (Read) |
| `boards-team-active-work` | 取得整個團隊進行中的項目，呈現阻擋關係、優先順序與 Sprint 歸屬 | Work Items (Read)、Project and Team (Read) |
| `boards-work-item-summary` | 摘要單一工作項目，含連結、Pull Request 與留言 | Work Items (Read)、Code (Read) |
| `pipelines-build-summary` | 列出、檢視與排查 Pipeline 建置，含失敗步驟記錄檔與變更清單 | Build (Read)、Project and Team (Read) |
| `security-alert-review` | 列出與檢視 Advanced Security 警示（相依套件弱點、機密外洩、程式碼掃描） | Advanced Security (Read)、Code (Read)、Project and Team (Read) |
| `work-iterations` | 列出、建立 Iteration，並指派給團隊 | Work Items (Read & write)、Project and Team (Read) |

---

## 🔐 前置需求

1. 安裝 `curl`（macOS、Linux 與 Windows 10 以上都已內建；在 Windows PowerShell 請呼叫 `curl.exe`）。
2. 到 `https://dev.azure.com/{organization}/_usersSettings/tokens` 建立 Personal Access Token（PAT），勾選上表所需的權限範圍。
3. 在啟動 AI 代理的終端機設定環境變數：

   ```bash
   # macOS / Linux
   export AZURE_DEVOPS_ORG="contoso"          # 組織名稱，對應 https://dev.azure.com/contoso
   export AZURE_DEVOPS_PAT="<your-pat>"
   ```

   ```powershell
   # Windows PowerShell
   $env:AZURE_DEVOPS_ORG = "contoso"
   $env:AZURE_DEVOPS_PAT = "<your-pat>"
   ```

已經為 Azure CLI 的 azure-devops 擴充套件設定過 `AZURE_DEVOPS_EXT_PAT` 的話，Skills 會在沒有 `AZURE_DEVOPS_PAT` 時改用它。

若組織改用 Microsoft Entra ID 權杖，可改設 `AZURE_DEVOPS_TOKEN`，Skills 會以 `Authorization: Bearer` 送出。例如以 Azure CLI 取得：

```bash
export AZURE_DEVOPS_TOKEN="$(az account get-access-token --resource 499b84ac-1321-427f-aa17-267ca6975798 --query accessToken -o tsv)"
```

> ⚠️ 請把權杖放在環境變數或密碼管理工具中，不要貼進與 AI 的對話，也不要提交到版本控制。

---

## 💻 使用方式

1. 讓你的 AI 代理載入 `.agents/skills/` 底下的 Skills：
   - 支援專案層級 `.agents/skills/` 的代理，直接在此儲存庫（或複製了這個資料夾的專案）中啟動即可。
   - 其他代理請把需要的 Skill 資料夾複製或建立 symlink 到該工具的 Skills 目錄。
2. 確認已設定上一節的環境變數。
3. 用自然語言描述需求，例如：
   - 「列出 Contoso 專案的 Iteration」
   - 「摘要工作項目 115770」
   - 「Contoso 專案的 build 12345 為什麼失敗？」
   - 「給我 Contoso 專案 MyApp 儲存庫的安全性警示總覽」
4. 代理會依 Skill 的 `description` 選出合適的 Skill，並依 `SKILL.md` 的步驟呼叫 REST API。

---

## 🛠️ 新增 Skill

1. **建立資料夾**：在 `.agents/skills/<skill-name>/` 建立 `SKILL.md`，資料夾名稱必須與 `name` 相同。
2. **填寫 frontmatter**：
   - `name`：小寫英數字與連字號組成的唯一識別名稱。
   - `description`：說明這個 Skill 做什麼、何時該使用；代理靠它判斷是否觸發。
   - `compatibility`：列出執行環境需求（`curl`、網路、環境變數）。
3. **說明觸發時機**：描述哪些使用者需求該觸發這個 Skill（例如 Iteration 規劃、看板查詢、建置摘要）。
4. **寫出 API 呼叫與流程**：
   - 沿用現有 Skills 的「連線與驗證」段落，並列出這個 Skill 需要的 PAT 權限範圍。
   - 針對每個常見任務，寫出要呼叫哪些 REST API、呼叫順序，以及結果如何呈現；每個呼叫附上可直接執行的 `curl` 範例。
   - 寫入類操作（建立、指派、更新）先把變更內容列給使用者確認。
5. **保持指示精簡、明確**：
   - 優先使用編號步驟與明確的判斷條件。
   - 寫清楚邊界情況（缺少專案或團隊、結果為空、錯誤回應）時代理該怎麼做。

現有的 Skills 可作為結構、語氣與細節程度的參考。

---

## 🤝 貢獻

- 讓指示聚焦在 Azure DevOps 工作流程，並只引用 REST API 實際提供的端點與參數。
- 讓行為可預期且有防護（例如缺少專案或團隊時只詢問一次，再退回清單 API）。
- 偏好小而可組合的 Skill，而不是大而全的 Skill。
- 送出前，實際以 PAT 呼叫一次 Skill 中的 `curl` 範例，確認端點、參數與 `api-version` 正確。

---

## 📜 來源與授權

本專案改寫自 [microsoft/azure-devops-skills](https://github.com/microsoft/azure-devops-skills)（MIT License，Copyright (c) Microsoft Corporation）。原專案的 Skills 依賴 GitHub Copilot 與 Azure DevOps MCP 伺服器；本專案改為直接呼叫 REST API，並改放在 `.agents/skills/`。散布或修改時，請保留原專案的 MIT 授權聲明。
