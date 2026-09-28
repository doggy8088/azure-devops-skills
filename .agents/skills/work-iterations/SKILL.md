---
name: work-iterations
description: 以 Azure DevOps REST API 列出、建立 Iteration（Sprint），並把 Iteration 指派給團隊。當使用者要查看專案或團隊的 Iteration、依既有節奏建立新 Sprint，或設定團隊訂閱哪些 Iteration 時使用。
compatibility: 需要 curl、可連線至 dev.azure.com 的網路，以及 AZURE_DEVOPS_ORG 與 AZURE_DEVOPS_PAT 環境變數。
---

# Iteration 管理

這個 Skill 一律在**專案**的範圍內運作。**團隊**是選填的，只有在指派 Iteration 或列出特定團隊的 Iteration 時才需要。

## 專案選擇

- 使用者有提供專案名稱（例如「Contoso 專案」）就直接使用，不呼叫專案清單 API。
- 沒有提供時，請使用者提供一次。
- 仍未提供時，列出專案讓使用者選擇：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/projects?%24top=500&api-version=7.1"
  ```

## 團隊選擇

- 沒有提供團隊時，只執行**專案層級**的動作（列出或建立專案的 Iteration），指派 Iteration 要等到有團隊才進行。
- 使用者要求**指派 Iteration** 但沒有提供團隊時，詢問一次團隊名稱。
- 仍未提供時，列出所選專案的團隊讓使用者選擇：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/projects/{project}/teams?%24top=500&api-version=7.1"
  ```

## 連線與驗證

一律用 `curl` 直接呼叫 Azure DevOps REST API，每個指令都自帶驗證參數。

- **組織**：讀取環境變數 `AZURE_DEVOPS_ORG`（組織名稱，例如 `contoso`）。未設定時詢問使用者一次；使用者給的是網址時，從 `https://dev.azure.com/{org}` 或 `https://{org}.visualstudio.com` 取出組織名稱。
- **權杖**：PAT 以 Basic 驗證送出：`-u ":$AZURE_DEVOPS_PAT"`；只設定了 `AZURE_DEVOPS_EXT_PAT`（Azure CLI azure-devops 擴充套件使用的變數）時，改用它。改用 Microsoft Entra ID 權杖時，換成 `-H "Authorization: Bearer $AZURE_DEVOPS_TOKEN"`。都未設定時，請使用者在自己的終端機設定 `AZURE_DEVOPS_PAT` 後重試，並停止。權杖只從環境變數讀取：不要請使用者把權杖貼進對話，也不要印出或寫入檔案。
- **PAT 權限範圍**：列出只需 Work Items (Read)、Project and Team (Read)；建立與指派需要 Work Items (Read & write)。
- **編碼**：路徑中的專案、團隊與 Iteration 名稱要 URL 編碼（空白寫成 `%20`）；查詢字串中的 `$` 寫成 `%24`（例如 `%24depth`），避免被 shell 展開。
- **錯誤**：HTTP 401、403、203 或回應是 HTML 登入頁，代表權杖無效、過期或權限不足，回報後停止；HTTP 404 代表組織、專案或團隊名稱有誤。
- **Windows PowerShell**：呼叫 `curl.exe`（而非 `curl` 別名），環境變數寫成 `$env:AZURE_DEVOPS_PAT`；PowerShell 不支援 `<<'EOF'`，先把 JSON 主體寫入暫存檔，再以 `--data-binary "@body.json"` 送出。

## Iteration 的兩種路徑

- **專案 Iteration**（分類節點）的 `path` 含 `Iteration` 這一層：`\Contoso\Iteration\Sprint 12`。
- **團隊 Iteration** 與工作項目 `System.IterationPath` 的路徑則沒有：`Contoso\Sprint 12`。
- 分類節點有兩個 ID：整數的 `id`，以及 GUID 的 `identifier`。指派給團隊時要用 `identifier`。

## 規則

### 1. 列出專案的 Iteration

- 使用者**只要求列出專案的 Iteration** 時，取得 Iteration 分類樹：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wit/classificationnodes/Iterations?%24depth=10&api-version=7.1"
  ```

  根節點是專案本身；遞迴展開 `children`，每個節點取 `name`、`path`、`attributes.startDate`、`attributes.finishDate`。
- 這個情境只讀取，不建立或指派 Iteration。
- 以清單呈現，包含 Iteration 名稱與起訖日期。
- 沒有任何 Iteration 時，明確說明這個專案沒有 Iteration。

範例：「列出 Contoso 專案的 Iteration」

### 2. 列出專案中某個團隊的 Iteration

- 使用者要**列出特定團隊的 Iteration** 時：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/{team}/_apis/work/teamsettings/iterations?api-version=7.1"
  ```

  每筆取 `name`、`path`、`attributes.startDate`、`attributes.finishDate`、`attributes.timeFrame`（`past`、`current`、`future`）。只要目前 Sprint 時加上 `&%24timeframe=current`。
- 這個情境只讀取，不建立或指派 Iteration。
- 以清單呈現，包含 Iteration 名稱與起訖日期。
- 沒有任何 Iteration 時，明確說明這個團隊沒有 Iteration。

範例：「列出 Contoso 專案 Contoso Team 團隊的 Iteration」

### 3. 為專案建立 Iteration

1. 先依規則 1 取得完整的 Iteration 樹，從中推算既有的節奏：每個 Iteration 的長度、Iteration 之間的間隔、命名模式（例如 `Sprint 12` → `Sprint 13`），以及新 Iteration 應放在哪個父節點底下。
2. 依這個節奏規劃新的 Iteration，略過名稱已存在的項目。建立前把規劃好的清單（名稱、起訖日期、父節點）列給使用者確認一次。
3. 確認後逐一建立。放在根層級時省略 `{parentPath}`；否則填入父節點相對於 Iteration 根的路徑（例如 `2026` 或 `Release%201`）：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -X POST -H "Content-Type: application/json" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wit/classificationnodes/Iterations/{parentPath}?api-version=7.1" \
     -d '{"name":"Sprint 13","attributes":{"startDate":"2026-01-05T00:00:00Z","finishDate":"2026-01-16T00:00:00Z"}}'
   ```

   回應是新建的節點；記下 `identifier`，後續指派給團隊時使用。
4. 以清單呈現新建立的 Iteration，包含名稱與起訖日期。
5. 沒有建立任何 Iteration 時，明確說明這個專案沒有建立新的 Iteration。

範例：

- 「在最後一個 Iteration 之後，用同樣的節奏建立到 2026 年底」
  - 依規則 1 取得完整的 Iteration 樹並推算節奏。
  - 依節奏規劃新的 Iteration，經使用者確認後逐一建立。
- 「在最後一個 Iteration 之後，用同樣的節奏建立到 2026 年底，然後把這些新 Iteration 全部指派給 Contoso Team」
  - 依上一個範例建立 Iteration。
  - 用建立時回傳的 `identifier`，依規則 4 把它們指派給 Contoso Team。
  - 呈現新建立的 Iteration，並註明已指派給該團隊。

### 4. 把 Iteration 指派給團隊

- 依上方規則決定專案與團隊。
- 找出每個要指派的 Iteration 的 `identifier`（GUID）：剛建立的直接用建立回應；既有的從規則 1 的 Iteration 樹依名稱或日期找出。
- 每個 Iteration 呼叫一次：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" -X POST -H "Content-Type: application/json" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/{team}/_apis/work/teamsettings/iterations?api-version=7.1" \
    -d '{"id":"a589a806-bf11-4d4f-a031-c19813331553"}'
  ```

- 以清單呈現已指派的 Iteration，包含名稱與起訖日期。
- 沒有指派任何 Iteration 時，明確說明這個團隊沒有被指派 Iteration。

範例：

- 「把 Iteration 1、Iteration 2、Iteration 3 指派給 Contoso Team」
  - 從 Iteration 樹找出這三個 Iteration 的 `identifier`，逐一指派給團隊。
- 「把 2025 與 2026 年的所有 Iteration 指派給 Contoso Team」
  - 依規則 1 取得完整的 Iteration 樹。
  - 找出起訖日期落在 2025 與 2026 年的 Iteration。
  - 逐一指派給 Contoso Team，並以清單呈現。

## 呈現結果

日期格式為 `YYYY/MM/DD`。API 回傳的日期是 UTC 午夜（例如 `2026-01-05T00:00:00Z`），直接取日期部分，不做時區換算。`startDate`、`finishDate` 為 `null` 時顯示 *（未設定日期）*。推算節奏時只採用有日期的 Iteration。
