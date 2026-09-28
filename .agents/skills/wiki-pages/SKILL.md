---
name: wiki-pages
description: 以 Azure DevOps REST API 讀取與編輯 Wiki：列出專案的 Wiki（project wiki 與 code wiki）、瀏覽頁面樹、讀取頁面內容、以關鍵字搜尋 Wiki 頁面，並在確認後建立新頁面，或先顯示 diff 再更新既有頁面。當使用者要查 Wiki 文件、找某個主題寫在哪一頁、新增或修改 Wiki 頁面時使用。
compatibility: 需要 curl、git、可連線至 dev.azure.com 與 almsearch.dev.azure.com 的網路，以及 AZURE_DEVOPS_ORG 與 AZURE_DEVOPS_PAT 環境變數。
---

# Wiki 頁面

這個 Skill 讀取 Wiki，並在使用者確認後建立或更新頁面。不提供刪除、移動或重新命名頁面；需要時請使用者到網頁操作。

## 專案選擇

- 使用者有提供專案名稱（例如「Contoso 專案」）就直接使用，不呼叫專案清單 API。
- 使用者只要「在整個組織搜尋 Wiki」時，規則 4 改用不含專案的 URL，不需要選專案。
- 其他情況沒有提供時，請使用者提供一次。
- 仍未提供時，列出專案讓使用者選擇：

  ```bash
  curl -sS -u ":$AZURE_DEVOPS_PAT" \
    "https://dev.azure.com/$AZURE_DEVOPS_ORG/_apis/projects?%24top=500&api-version=7.1"
  ```

- 使用者提供或選定專案之前不要繼續。

## Wiki 選擇

- 使用者有提供 Wiki 名稱就直接使用；`{wikiIdentifier}` 可以是 Wiki 的 `id` 或 `name`（名稱要 URL 編碼）。
- 沒有提供時，以規則 1 列出專案的 Wiki：只有一個就直接使用，有多個時請使用者選擇。
- 使用者給的是頁面網址 `https://dev.azure.com/{org}/{project}/_wiki/wikis/{wikiName}/{pageId}/{slug}` 時，取出專案、Wiki 名稱與 `pageId`，以規則 3 依 ID 讀取。

## 連線與驗證

一律用 `curl` 直接呼叫 Azure DevOps REST API，每個指令都自帶驗證參數。

- **組織**：讀取環境變數 `AZURE_DEVOPS_ORG`（組織名稱，例如 `contoso`）。未設定時詢問使用者一次；使用者給的是網址時，從 `https://dev.azure.com/{org}` 或 `https://{org}.visualstudio.com` 取出組織名稱。
- **主機**：除了 `dev.azure.com`，Wiki 搜尋 API 的主機是 `almsearch.dev.azure.com`。
- **權杖**：PAT 以 Basic 驗證送出：`-u ":$AZURE_DEVOPS_PAT"`；只設定了 `AZURE_DEVOPS_EXT_PAT`（Azure CLI azure-devops 擴充套件使用的變數）時，改用它。改用 Microsoft Entra ID 權杖時，換成 `-H "Authorization: Bearer $AZURE_DEVOPS_TOKEN"`。都未設定時，請使用者在自己的終端機設定 `AZURE_DEVOPS_PAT` 後重試，並停止。權杖只從環境變數讀取：不要請使用者把權杖貼進對話，也不要印出或寫入檔案。
- **PAT 權限範圍**：讀取所需：Wiki (Read)（已包含 Wiki 搜尋）；寫入所需：Wiki (Read & write)（建立與更新頁面）。
- **編碼**：路徑中的專案與 Wiki 名稱、查詢字串中的頁面路徑與 `comment` 要 URL 編碼（空白寫成 `%20`，規則見「頁面路徑」）；查詢字串中的 `$` 寫成 `%24`（例如 `%24top`），避免被 shell 展開。
- **錯誤**：HTTP 401、403、203 或回應是 HTML 登入頁，代表權杖無效、過期或權限不足，回報後停止；HTTP 404 代表組織、專案、Wiki 或頁面路徑有誤（頁面不存在時 `typeKey` 為 `WikiPageNotFoundException`）。code wiki 指定不存在的分支時回 HTTP 500（`TF401175`）。
- **Windows PowerShell**：呼叫 `curl.exe`（而非 `curl` 別名），環境變數寫成 `$env:AZURE_DEVOPS_PAT`；PowerShell 不支援 `<<'EOF'`，先把 JSON 主體寫入暫存檔，再以 `--data-binary "@body.json"` 送出。

## 頁面路徑

- **頁面路徑**（API 的 `path`）以 `/` 開頭，由頁面標題組成，保留空白，例如 `/Release Notes/v1.0`；根目錄是 `/`。code wiki 的頁面路徑相對於 `mappedPath`。
- **標題限制**：不能含 `/`、`\`、`#`，不能以 `.` 開頭或結尾；含儲存庫網址的完整路徑上限 235 字元，單頁上限 18 MB。
- **放進查詢字串**：整段 URL 編碼（空白 `%20`、`&` `%26`、`?` `%3F`、`%` `%25`、`+` `%2B`，中文以 UTF-8 編碼；`/` 可保留或寫成 `%2F`）。可用 `jq -rn --arg p "/Release Notes/v1.0" '$p|@uri'` 產生。
- **`gitItemPath`** 是頁面在 Wiki 儲存庫中的檔案路徑：標題中的空白變 `-`，原本的 `-` 變 `%2D`，`:`、`*`、`?`、`|`、`"`、`<`、`>` 也以 `%XX` 表示，最後加上 `.md`；code wiki 會加上 `mappedPath` 前綴。
- **由 `gitItemPath` 還原頁面路徑**：去掉 `mappedPath` 前綴與 `.md`，把 `-` 換成空白，最後做 URL 解碼（順序不能反，否則 `%2D` 會被誤換成空白）。

## 規則

### 1. 列出 Wiki

```bash
curl -sS -u ":$AZURE_DEVOPS_PAT" \
  "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wiki/wikis?api-version=7.1"
```

每筆有 `id`、`name`、`type`（`projectWiki` 是專案 Wiki；`codeWiki` 是由儲存庫資料夾發布的 Wiki）、`repositoryId`、`mappedPath`（code wiki 對應的資料夾）、`versions[].version`（project wiki 固定一個分支，例如 `wikiMaster`；code wiki 是發布的分支）與 `remoteUrl`。沒有任何 Wiki 時明確說明，並停止。

範例：「Contoso 專案有哪些 Wiki？」

### 2. 瀏覽頁面樹

```bash
curl -sS -u ":$AZURE_DEVOPS_PAT" \
  "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wiki/wikis/{wikiIdentifier}/pages?path=/&recursionLevel=oneLevel&api-version=7.1"
```

- `recursionLevel`：`oneLevel` 只取直接子頁面，`full` 取得全部。頁面很多時先用 `oneLevel`，再依使用者需要把 `path` 換成子頁面逐層展開。
- `subPages[]` 每筆有 `path`、`order`（同層排序）、`isParentPage`（有子頁面）與 `gitItemPath`。
- **code wiki**：省略 `versionDescriptor` 時讀取預設的發布版本。`versions` 有多個分支時，加上 `&versionDescriptor.version={branch}&versionDescriptor.versionType=branch` 指定；`{branch}` 只用 `versions[].version` 中的值（其他分支也可能回 200，但那不是發布的內容）。project wiki 不需要這個參數。

範例：「列出 Contoso.wiki 的頁面結構」

### 3. 讀取頁面

```bash
curl -sS -u ":$AZURE_DEVOPS_PAT" -D - \
  "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wiki/wikis/{wikiIdentifier}/pages?path={pagePath}&includeContent=true&api-version=7.1"
```

- `-D -` 會先印出回應標頭：`ETag` 是頁面目前的版本（含雙引號），規則 6 更新時要用。只讀取時可以省略 `-D -`。
- 回應有 `content`（Markdown）、`id`（永久的頁面 ID）、`path`、`gitItemPath`、`remoteUrl`（網頁連結）。
- 已知頁面 ID 時改用 `.../wiki/wikis/{wikiIdentifier}/pages/{pageId}?includeContent=true&api-version=7.1`，同樣回傳 `ETag`。
- 內容中的 `[[_TOC_]]`、`:::mermaid`、`/.attachments/...` 圖片連結是 Wiki 專用語法，呈現時照原樣保留或說明。

範例：「讀 Contoso.wiki 的 /Release Notes/v1.0」

### 4. 搜尋 Wiki

```bash
curl -sS -u ":$AZURE_DEVOPS_PAT" -X POST -H "Content-Type: application/json" \
  "https://almsearch.dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/search/wikisearchresults?api-version=7.1" \
  -d '{"searchText":"deployment checklist","$skip":0,"$top":25,"filters":{"Project":["Contoso"],"Wiki":["Contoso.wiki"]},"includeFacets":true}'
```

- `searchText` 支援關鍵字（預設 `AND`）、大寫的 `AND`、`OR`、`NOT`、雙引號片語與結尾的 `*` 萬用字元，不分大小寫。
- `filters.Wiki` 用 Wiki 的**名稱**（用 `id` 會找不到結果）；不需要的篩選整個省略。搜尋整個組織時，URL 去掉 `/{project}`，`filters` 也不放 `Project`。結果同時包含 project wiki 與 code wiki。
- `$top` 必須在 1 到 1000 之間；`count` 是總筆數，大於已取得的筆數時以 `$skip` 取下一頁。`includeFacets` 為 `true` 時，`facets.Wiki[]` 列出各 Wiki 的符合數。
- 每筆有 `fileName`、`path`（`gitItemPath` 形式，含 `.md`）、`wiki.name`、`wiki.id`、`wiki.mappedPath`、`wiki.version`、`project.name`，以及 `hits[]`：`fieldReferenceName` 是 `fileNames` 或 `content`，`highlights[]` 是以 `<highlighthit>` 標記符合處的片段。
- 依「頁面路徑」把 `path` 還原成頁面路徑，再用規則 3 讀取需要的頁面。

範例：「Wiki 裡哪一頁有寫部署流程？」

### 5. 建立頁面（寫入）

1. 確認頁面不存在：以規則 3 讀取，回 HTTP 404（`WikiPageNotFoundException`）才繼續；已存在時改用規則 6。
2. 確認上層頁面存在（例如 `/Release Notes/v1.0` 的上層是 `/Release Notes`）；不存在時告知使用者，詢問是否先建立上層頁面。
3. 把即將送出的內容列給使用者，確認一次後才執行：Wiki、頁面路徑、完整 Markdown 內容、修改說明（`comment`）。
4. 建立時**不帶** `If-Match`：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -X PUT -H "Content-Type: application/json" \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wiki/wikis/{wikiIdentifier}/pages?path={pagePath}&comment={comment}&api-version=7.1" \
     --data-binary @- <<'EOF'
   {"content": "# v1.0\n\n- 新增登入功能\n- 修正匯出錯誤"}
   EOF
   ```

   成功時回 HTTP 201。內容含換行與引號時，先把 Markdown 寫入暫存檔，再用 `jq -Rs '{content: .}' page.md` 產生 JSON 主體。

範例：「在 Contoso.wiki 的 /Release Notes 底下新增 v1.0 頁面」

### 6. 更新頁面（寫入）

1. 讀取頁面，記下 `ETag`，把目前內容存成暫存檔 `old.md`（`jq -j` 不會在結尾多加換行）：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -D headers.txt -o page.json \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wiki/wikis/{wikiIdentifier}/pages?path={pagePath}&includeContent=true&api-version=7.1"
   grep -i '^etag:' headers.txt
   jq -j '.content' page.json > old.md
   ```

2. 依使用者要求產生新內容 `new.md`：只改使用者要求的段落，其餘內容與格式保持原樣。
3. 顯示差異，並把 Wiki、頁面路徑、修改說明與 diff 列給使用者，確認一次後才執行：

   ```bash
   git diff --no-index --unified=3 old.md new.md
   ```

4. 更新時**必須**帶 `If-Match`，值照 `ETag` 標頭原樣（含雙引號；範例中的 `{etag}` 是雙引號內的版本字串）：

   ```bash
   curl -sS -u ":$AZURE_DEVOPS_PAT" -X PUT -H "Content-Type: application/json" \
     -H 'If-Match: "{etag}"' \
     "https://dev.azure.com/$AZURE_DEVOPS_ORG/{project}/_apis/wiki/wikis/{wikiIdentifier}/pages?path={pagePath}&comment={comment}&api-version=7.1" \
     --data-binary "@body.json"
   ```

   `body.json` 以 `jq -Rs '{content: .}' new.md > body.json` 產生。成功時回 HTTP 200，回應標頭的 `ETag` 是新版本。

5. 失敗且訊息指出版本不符時，代表頁面在這段期間被別人修改：從步驟 1 重新開始，重新顯示 diff 並確認。不要拿掉 `If-Match` 或沿用舊的 `ETag` 重送。
6. 暫存檔用完即刪。

**code wiki 的寫入**：建立與更新會直接 commit 到發布的分支（`versions` 有多個分支時，加上規則 2 的 `versionDescriptor` 參數指定）。分支設有必須經過 PR 的原則時，寫入可能被拒絕：改為在本機修改 `mappedPath` 底下的 `.md` 檔，再用 `repos-pull-request-write` 開 PR。

範例：「把 Contoso.wiki 的 /Onboarding 頁面中的 VPN 設定步驟更新成新版」

## 呈現結果

日期顯示為 `YYYY/MM/DD`，時間轉成使用者當地時區，無法判斷時標明 UTC。

- **Wiki 清單**：表格呈現名稱（連結 `remoteUrl`）、類型（專案 Wiki／code wiki）、code wiki 的儲存庫資料夾（`mappedPath`）與發布分支。
- **頁面樹**：以縮排清單呈現頁面標題（路徑最後一段），有子頁面但尚未展開的加註「（有子頁面）」。
- **頁面內容**：先列頁面路徑與網頁連結，再呈現 Markdown 內容；內容很長時先摘要重點，詢問是否顯示全文。
- **搜尋結果**：先說明總筆數，再以表格呈現頁面路徑（還原後，可點擊）、Wiki 名稱與符合片段（把 `<highlighthit>` 標記改成粗體）。沒有結果時明確說明。
- **寫入前**：建立時列出完整內容；更新時列出 diff 與修改說明。
- **寫入後**：頁面路徑、網頁連結、建立或更新的結果；失敗時顯示 HTTP 狀態碼與回應中的 `message`，不要自動重試寫入。
