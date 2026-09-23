# AI-Friendly Doc 轉換流程 — 設計文件

- 日期：2026-09-24
- 狀態：待審核（v2：加入 Web 應用、可靠性設計）

## 1. 目標

把任何類型的文件轉成 AI-friendly 格式（Markdown + 結構化中繼資料），同時服務三種用途：

1. **臨時餵給 LLM**：單檔轉成乾淨 Markdown。
2. **RAG／知識庫**：批次轉換 + 帶頁碼與標題路徑的切段輸出。
3. **Claude Code skill**：對話中說「轉一下這份檔案」，自動挑工具轉換。

另有 **Web 應用**（SaaS 級外觀與操作）做為主要操作介面。CLI、skill、Web 共用同一個核心 pipeline，判斷邏輯只存在一處。

### 使用情境

個人／團隊內部工具（非多租戶 SaaS）。預設只綁 `127.0.0.1`；`aidoc serve --host 0.0.0.0 --token <token>` 可開放內網（例如 Tailscale）並啟用 token 驗證。

### 支援的輸入

Office（docx/pptx/xlsx）、電子 PDF、掃描 PDF／圖片（需 OCR）、含公式或複雜表格的學術技術文件、HTML、EPUB、CSV、音訊等 MarkItDown 支援的格式。

### OCR 語言

繁體中文 + 英文（含中英混排）為預設；`--lang en` 可切純英文模型。

### 成功標準

- 每種 fixture 類型都能產出品質檢查為 `ok` 的 Markdown。
- 單一檔案失敗不中斷批次作業。
- server 或 engine 在任何時間點被中斷，重啟後能從中斷處續轉（大型 PDF 以段為單位），且不留下半套輸出。
- 三個工具可獨立安裝、獨立升級，互不破壞。

## 2. 使用的工具

| 工具 | 定位 |
|---|---|
| [MarkItDown](https://github.com/microsoft/markitdown) | 輕量；Office／HTML／EPUB／CSV／音訊等 |
| [Docling](https://github.com/docling-project/docling) | 電子 PDF 的版面、表格、閱讀順序 |
| [MinerU](https://github.com/opendatalab/mineru) | 掃描檔、OCR、公式、複雜版面（GPU） |

**Marker（未來可選）**：功能與 Docling／MinerU 重疊，且為 GPL-3.0 + 模型權重有商用限制，第一版不實作。若 fixture 實測發現某類文件兩者皆處理不佳，再以 `engines/marker.py` + `envs/marker/` 加入為該類備援。

**授權提醒**：本專案定位為內部工具。若未來改為對外提供服務，需重新檢視各工具（尤其 MinerU、Marker）的授權條款。

## 3. 架構

```
ai-friendly-doc/
├─ src/aidoc/                 ← 主程式（輕量，不裝 torch）
│  ├─ cli.py                  ← aidoc convert / batch / chunk / setup / serve
│  ├─ pipeline.py             ← 單一 task 的執行流程（CLI 與 server 共用）
│  ├─ probe.py                ← 檔案偵測
│  ├─ router.py               ← 決定主要工具與備援順序
│  ├─ segment.py              ← 大型 PDF 分段與合併
│  ├─ engines/
│  │  ├─ base.py              ← Engine 介面
│  │  ├─ runner/              ← 在各 env 內執行的小腳本
│  │  ├─ markitdown.py
│  │  ├─ docling.py
│  │  └─ mineru.py
│  ├─ quality.py              ← 品質評分
│  ├─ normalize.py            ← 統一輸出格式
│  ├─ chunk.py                ← RAG 切段
│  ├─ store.py                ← SQLite 狀態儲存
│  └─ server/
│     ├─ app.py               ← FastAPI app、靜態檔
│     ├─ jobs.py              ← 佇列、worker、啟動恢復
│     ├─ uploads.py           ← 分塊上傳與續傳
│     ├─ sse.py
│     └─ api/*.py
├─ web/                       ← Vite + React + TS + Tailwind + shadcn/ui
├─ envs/                      ← 各工具獨立 uv 環境
│  ├─ markitdown/pyproject.toml
│  ├─ docling/pyproject.toml  (torch cu128)
│  └─ mineru/pyproject.toml   (torch cu128)
├─ data/                      ← aidoc.db、uploads/、work/（執行期產生，不進版控）
├─ skill/SKILL.md
└─ tests/
   ├─ unit/
   ├─ reliability/            ← 假 engine 的中斷／續傳情境
   └─ fixtures/
```

### 環境隔離

- 主程式依賴僅 PyMuPDF、FastAPI、uvicorn、tiktoken 等輕量套件，偵測與判斷都很快。
- 每個 engine 以 `envs/<工具>/.venv` 的 Python 透過 subprocess 執行 `engines/runner/<工具>_runner.py`，在該環境內呼叫工具的 Python API，把結果寫到指定目錄。
- 硬體：RTX 5070 Ti（Blackwell，16GB），torch 需 CUDA 12.8 以上版本，版本鎖在各 env 的 `pyproject.toml`。
- 某工具未安裝時視為不可用，router 跳到備援順序的下一個，並提示 `aidoc setup <工具>`。

### Engine 介面

```python
class Engine(Protocol):
    name: str
    def available(self) -> bool: ...
    def convert(self, src: Path, workdir: Path, opts: ConvertOptions,
                pages: tuple[int, int] | None,
                on_progress: Callable[[float, str], None]) -> RawResult: ...
```

- `pages`：大型 PDF 分段時的頁碼範圍（含頭含尾，從 1 起算）；`None` 代表整份。
- `on_progress`：解析 subprocess stderr（tqdm／log）回報段內進度與 log 行，僅供顯示。
- `RawResult`：Markdown 字串、圖片檔路徑列表、頁界資訊（若工具有提供）、工具原始輸出路徑。`normalize.py` 把它轉成統一格式。

## 4. 偵測與判斷

### probe（PDF 每頁取樣，最多 20 頁）

- `text_ratio`：有文字層的頁數比例
- `image_cover`：圖片佔頁面面積比例（取樣頁平均）
- `math_hint`：出現數學字型（CMMI、CMSY、Symbol、Cambria Math 等）或數學符號密度高
- `layout_hint`：多欄版面（文字區塊 x 座標分布呈雙峰）
- `has_table_lines`：頁面有構成格線的線段
- 同時偵測損毀、加密的 PDF → 直接判定 `input` 類失敗

### 判斷規則（依序比對，第一條符合即採用）

| 條件 | 主要 → 備援 |
|---|---|
| 非 PDF，MarkItDown 支援的格式 | MarkItDown → Docling（僅 docx/pptx/html） |
| 圖片（png/jpg/tiff…） | MinerU → Docling |
| PDF 且 `text_ratio < 0.5` 或 `image_cover > 0.6` | MinerU → Docling |
| PDF 且 `math_hint` | MinerU → Docling |
| 其他 PDF | Docling → MinerU → MarkItDown |

`--engine <名稱>` 可強制指定，此時不走備援。
大型 PDF 分段時，**整份文件用同一個 engine**；若某段品質不合格，整份改用下一個備援重轉（已完成的段作廢），避免同一份文件混用不同 engine 的輸出風格。

### OCR 語言設定

- MinerU：`lang=chinese_cht`（PaddleOCR 繁中模型，含英數）；`--lang en` → `en`
- Docling：EasyOCR `["ch_tra", "en"]`；`--lang en` → `["en"]`

### 品質檢查

以下任一條件成立即判定失敗、改用下一個備援工具：

- 平均每頁字數 < 50（排除 probe 判定為空白的頁面）
- 亂碼比例 > 5%（`U+FFFD`、私有使用區字元、無意義符號連串）
- probe 的 `has_table_lines` 為真，但輸出中沒有任何表格

分數 `score ∈ [0,1]` 由上述指標加權計算，`level` 為 `ok` / `low`。
全部工具都失敗時保留分數最高的結果，標記 `level: "low"`。門檻為初始值，依 fixture 實測調整。

## 5. 輸出格式

### 單檔

```
out/<檔名>/
├─ <檔名>.md
├─ <檔名>.json
└─ assets/
```

- Markdown：標題階層、GFM 表格、公式 `$...$` / `$$...$$`，圖片以相對路徑 `assets/…` 引用。
- 頁界以 `<!-- page: N -->` 註解標示（工具有提供頁資訊時）。
- 同名檔案衝突時輸出目錄加上 sha256 前 8 碼。

`<檔名>.json`：

```json
{
  "source": "report.pdf",
  "sha256": "…",
  "pages": 42,
  "engine": "mineru",
  "tried": [{"engine": "mineru", "score": 0.93, "reasons": []}],
  "segments": 2,
  "probe": {"text_ratio": 0.1, "image_cover": 0.85, "math_hint": false},
  "quality": {"score": 0.93, "level": "ok", "reasons": []},
  "lang": "chinese_cht+en",
  "elapsed_s": 38.2,
  "aidoc_version": "0.1.0"
}
```

### 批次（CLI）

`aidoc batch <輸入目錄> -o <輸出目錄> [--force] [--engine X] [--lang en]`

- 以 sha256 快取：輸出 JSON 的 sha256 相同即跳過；`--force` 重新轉換。
- CLI 批次同樣寫入 SQLite（第 8 節），Web 上看得到 CLI 啟動的工作；CLI 中斷後再執行同一指令即續轉。
- 輸出 `out/_manifest.jsonl`：每檔一行 `{source, status, engine, score, level, error?}`。

### RAG 切段

`aidoc chunk <輸出目錄> [--max-tokens 800]`

- 依標題階層切段，超過上限時在段落邊界再切；表格與公式區塊不切開（單一表格超長時整塊保留並標記 `oversized: true`）。
- token 以 `tiktoken` 的 `cl100k_base` 估算。
- 輸出 `chunks.jsonl`：`{id, source, heading_path, page_start, page_end, text}`。
- 不做向量化（由使用者自行選擇向量資料庫）。

## 6. Web 應用

### 技術

- 前端：Vite + React + TypeScript + Tailwind CSS + shadcn/ui；API 狀態用 TanStack Query，即時更新用 SSE（`EventSource`，斷線自動重連）。
- Markdown 渲染：react-markdown + remark-gfm + rehype-katex；PDF 預覽：pdf.js。
- 後端：FastAPI + uvicorn。開發時 Vite proxy 到 FastAPI；正式使用時 `pnpm build` 產出靜態檔由 FastAPI 提供，`aidoc serve` 一個指令啟動。
- 版面：左側導覽（Convert / Jobs / Library / Chunks / Settings）、頂部狀態列（GPU、佇列數）、深淺色模式。

### API（前綴 `/api`）

| 方法 | 路徑 | 用途 |
|---|---|---|
| POST | `/uploads` | 建立上傳：`{filename, size, sha256}` → `upload_id` |
| PUT | `/uploads/{id}?offset=N` | 上傳一塊（8MB） |
| HEAD | `/uploads/{id}` | 查詢已上傳位置（續傳用） |
| POST | `/jobs` | 建立工作：`{inputs: [upload_id \| 本機路徑], engine, lang, force}` |
| GET | `/jobs` · `/jobs/{id}` | 工作清單與詳情（含 tasks、segments） |
| POST | `/jobs/{id}/cancel` | 取消（結束 subprocess） |
| POST | `/tasks/{id}/retry` | 重試單一 task（含「用新版本重轉」） |
| GET | `/events` | **SSE**：所有 job／task／segment 狀態變化與 log 行 |
| POST | `/queue/pause` · `/queue/resume` | 暫停／繼續佇列 |
| GET | `/documents` · `/documents/{id}` | 已轉換文件 |
| GET | `/documents/{id}/markdown` · `/documents/{id}/assets/{path}` · `/documents/{id}/source` · `/documents/{id}/download.zip` | 預覽與下載 |
| DELETE | `/documents/orphaned` | 清除 orphaned 紀錄 |
| POST | `/chunks` | 對指定文件切段，回傳 `chunks.jsonl` |
| GET | `/system` | 各工具安裝狀態、GPU 名稱與顯存、佇列長度、磁碟剩餘空間 |
| POST | `/system/setup/{engine}` | 背景執行安裝，進度經 SSE |

所有回應帶 `workspace: "default"`，為日後多工作區預留；第一版不實作多工作區與帳號。

### 頁面

1. **Convert**：拖放區（分塊上傳、顯示上傳進度、斷線自動續傳）、本機資料夾路徑輸入；進階選項收合（工具、語言、強制重轉）。送出後跳到該工作頁。
2. **Jobs**：工作清單，每列有進度條與狀態徽章。詳情頁是 **task 時間軸**（probe 結果 → 選用工具 → 備援切換與原因 → 品質分數），大型 PDF 顯示段落進度（「第 120/300 頁」），下方即時 log。取消、重試、暫停／繼續佇列按鈕。
3. **Library**：已轉換文件的卡片／表格檢視，可依檔名、工具、品質等級篩選。
4. **Document 檢視**：左右兩欄——左側原始檔預覽（PDF 用 pdf.js、圖片直接顯示；原始檔已移除時顯示提示）；右側三分頁：Markdown 渲染（KaTeX、GFM 表格）、原始碼、JSON。**捲動時依 `<!-- page: N -->` 左右同步**。右上：複製 Markdown、下載 zip、送去切段。
5. **Chunks**：選文件、設定 max tokens，預覽切段結果（標題路徑、頁碼範圍），下載 jsonl。
6. **Settings**：三個工具的安裝狀態卡片（一鍵安裝、安裝 log）、GPU 資訊、預設語言、輸出目錄。

## 7. Claude Code skill

`skill/SKILL.md`：

- 觸發詞：「轉成 markdown」「把這份文件轉給 AI 讀」「convert this doc」等。
- 行為：執行 `aidoc convert <檔案> -o <暫存或指定目錄>`，讀出 `.md`；`level: "low"` 時主動告知並附上 `reasons`。
- 判斷邏輯不寫在 skill 裡，只呼叫 CLI。

## 8. 批次、進度、續傳與錯誤處理

### 8.1 狀態儲存：SQLite

- 標準函式庫 `sqlite3`，WAL 模式，檔案 `data/aidoc.db`。
- 三張表：
  - `jobs(id, created_at, options_json, status)`
  - `tasks(id, job_id, source_path, work_path, sha256, size, mtime, engine, attempt, status, error_kind, error_msg, quality_json, output_dir, pid)`
  - `segments(id, task_id, idx, page_start, page_end, status, attempt, output_path)`
- 每次狀態改變**先寫資料庫、再推 SSE**。資料庫是唯一依據，SSE 只是通知。
- CLI 與 server 用同一個資料庫；同一時間只允許一個佇列執行者（以 `data/aidoc.lock` 檔案鎖保證），server 執行中時 CLI `batch` 改為把工作送進 server 佇列。

### 8.2 狀態機

```
task: queued → probing → converting → checking → done | low | failed | cancelled | skipped
                            ↑ 失敗且還有備援 ─┘
segment: queued → converting → done | failed
```

- 每個轉換記錄 `attempt`、`engine`、`reason`，對應 Web 時間軸。
- **失敗分類**：
  - `transient`（逾時、GPU 顯存不足、process 被結束）：自動重試，最多 2 次，退避 5s → 30s；顯存不足時以較小 batch 重試。
  - `engine`（品質不合格、工具回報錯誤）：換下一個備援工具。
  - `input`（檔案消失、被修改、損毀、加密）：直接 `failed`，不重試。
- `skipped`：sha256 快取命中。

### 8.3 大型 PDF 分段轉換

- 超過 **40 頁**的 PDF 切成每段 40 頁，每段一筆 `segment`。
- 中斷後重跑只轉尚未完成的段。
- 進度 = 已完成頁數 ÷ 總頁數；段內進度來自 `on_progress`，僅供參考。
- 所有段完成後依序合併：頁碼標記加上偏移、圖片檔名加段落前綴避免衝突。
- **跨段表格**：若段落 N 結尾與段落 N+1 開頭都是表格，且欄數相同，合併為一個表格（省略第二個表頭，若其與第一個表頭相同）。
- 品質檢查在合併後對整份文件執行；不合格則整份換備援（見第 4 節）。

### 8.4 啟動恢復

server（或 CLI batch）啟動時：

1. `probing`／`converting`／`checking` 的 task 改回 `queued`；分段 task 只重設未完成的段。
2. 結束上次留下的孤兒 subprocess（依資料庫記錄的 `pid` 檢查，且確認命令列屬於 aidoc runner 才結束）。
3. 清除 `out/.tmp/` 下未完成的暫存目錄。

「暫停佇列」會等目前的段完成後才停止接新工作。

### 8.5 檔案消失或被修改

| 時間點 | 處理 |
|---|---|
| 建立工作時 | 立即計算 sha256，記錄大小與 mtime；檔案不存在直接回報錯誤 |
| 開始轉換前 | 將來源檔複製到 `data/work/<task_id>/`（上傳檔本來就在工作區）並比對 sha256：不存在 → `failed(input: source_missing)`；內容不同 → `failed(input: source_changed)`，Web 提供「用新版本重轉」 |
| 轉換過程中 | engine 只讀工作區副本，原始檔被移走不受影響 |
| 瀏覽 Library 時 | 原始檔與工作區副本都不存在時，結果照常顯示，原始檔預覽顯示「原始檔已移除」 |
| 輸出目錄被刪除 | 列出文件與啟動時以磁碟實際內容為準；資料庫中對不上的紀錄標記 `orphaned`，可一鍵清除 |

工作區副本在 task 成功完成後保留 7 天（供 Document 檢視預覽），之後清除；可在 Settings 調整。

### 8.6 原子寫入

- 轉換結果先寫到 `out/.tmp/<task_id>/`，全部完成後 rename 成正式目錄（同一磁碟分割區，rename 為原子操作）。
- `_manifest.jsonl`、`chunks.jsonl` 同樣先寫暫存檔再改名。
- 因此快取判斷（第 5 節）只會看到完整的輸出。

### 8.7 上傳續傳

- 前端把檔案切成 8MB 分塊，依序 `PUT /uploads/{id}?offset=N`；server 只接受 `offset` 等於目前已收大小的分塊。
- 斷線後前端以 `HEAD /uploads/{id}` 取得已上傳位置，從該處續傳。
- 全部收完後比對建立時提供的 sha256，不符則刪除並回報錯誤。
- 未完成的上傳 24 小時後自動清除。

### 8.8 資源保護

- 建立工作前檢查磁碟空間：剩餘空間需 ≥ 輸入總大小 × 3，不足則拒絕並提示。
- 單檔上傳上限預設 2GB，可調整。
- subprocess 逾時：每頁 60 秒、最少 120 秒（分段時以段的頁數計算），可用 `--timeout` 覆寫。
- `/system` 顯示 GPU 顯存使用量。

## 9. 安裝

`aidoc setup [markitdown|docling|mineru|all]`（Web Settings 頁可一鍵執行）

- 以 uv 在 `envs/<工具>/` 建立 `.venv` 並安裝鎖定版本。
- 預先下載模型（Docling 版面／表格模型、MinerU 模型）。
- 最後執行自我檢查：轉換一個內建小樣本，並確認 GPU 可用（`torch.cuda.is_available()`）。

## 10. 測試

- **單元測試**（不需 GPU）：`probe`、`router`、`quality`、`normalize`、`segment`（分段與合併、跨段表格）、`chunk`、`store`（狀態轉換）。
- **可靠性測試**（假 engine，不需 GPU）：
  - 轉換中結束 server → 重啟後從未完成的段續轉，已完成的段不重轉
  - 轉換前刪除／修改來源檔 → `source_missing`／`source_changed`
  - 上傳到一半中斷 → 續傳成功且 sha256 相符
  - 假 engine 模擬逾時、顯存不足、品質不合格 → 分別重試、重試、換備援
  - 殺掉 engine process → 視為 transient 重試
  - 寫入中途失敗 → 不留下正式輸出目錄
- **API 測試**：FastAPI TestClient 覆蓋各端點與 SSE 事件順序。
- **前端**：Vitest 測關鍵元件邏輯（上傳續傳、SSE 狀態合併）；Playwright 跑一條端到端流程（上傳 → 轉換 → 預覽）。
- **整合測試**（`@pytest.mark.slow`，需要 GPU）：`tests/fixtures/` 每類一個小樣本——docx、pptx、xlsx、電子 PDF、繁中掃描 PDF、中英混排掃描 PDF、公式 PDF、png，以及一份 > 40 頁的 PDF 驗證分段。斷言：引擎選擇正確、品質為 `ok`、產出檔案齊全。

## 11. 不在範圍內（第一版）

- Marker 引擎（見第 2 節）
- 帳號系統、多工作區、多租戶
- 向量化與向量資料庫整合
- LLM 輔助修正輸出
- 設計精修動畫
