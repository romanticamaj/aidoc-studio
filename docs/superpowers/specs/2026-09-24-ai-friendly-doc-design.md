# AI-Friendly Doc 轉換流程 — 設計文件

- 日期：2026-09-24
- 狀態：待審核

## 1. 目標

把任何類型的文件轉成 AI-friendly 格式（Markdown + 結構化中繼資料），同時服務三種用途：

1. **臨時餵給 LLM**：單檔轉成乾淨 Markdown。
2. **RAG／知識庫**：批次轉換 + 帶頁碼與標題路徑的切段輸出。
3. **Claude Code skill**：對話中說「轉一下這份檔案」，自動挑工具轉換。

三種用途共用同一個核心（CLI `aidoc`），判斷邏輯只存在一處。

### 支援的輸入

Office（docx/pptx/xlsx）、電子 PDF、掃描 PDF／圖片（需 OCR）、含公式或複雜表格的學術技術文件、HTML、EPUB、CSV、音訊等 MarkItDown 支援的格式。

### OCR 語言

繁體中文 + 英文（含中英混排）為預設；`--lang en` 可切純英文模型。

### 成功標準

- 每種 fixture 類型都能產出品質檢查為 `ok` 的 Markdown。
- 單一檔案失敗不中斷批次作業。
- 三個工具可獨立安裝、獨立升級，互不破壞。

## 2. 使用的工具

| 工具 | 定位 |
|---|---|
| [MarkItDown](https://github.com/microsoft/markitdown) | 輕量；Office／HTML／EPUB／CSV／音訊等 |
| [Docling](https://github.com/docling-project/docling) | 電子 PDF 的版面、表格、閱讀順序 |
| [MinerU](https://github.com/opendatalab/mineru) | 掃描檔、OCR、公式、複雜版面（GPU） |

**Marker（未來可選）**：功能與 Docling／MinerU 重疊，且為 GPL-3.0 + 模型權重有商用限制，第一版不實作。若 fixture 實測發現某類文件兩者皆處理不佳，再以 `engines/marker.py` + `envs/marker/` 加入為該類備援。

## 3. 架構

```
ai-friendly-doc/
├─ src/aidoc/                 ← 主程式（輕量，不裝 torch）
│  ├─ cli.py                  ← aidoc convert / batch / chunk / setup
│  ├─ probe.py                ← 檔案偵測
│  ├─ router.py               ← 決定主要工具與備援順序
│  ├─ engines/
│  │  ├─ base.py              ← Engine.convert(input, outdir, opts) -> RawResult
│  │  ├─ markitdown.py
│  │  ├─ docling.py
│  │  └─ mineru.py
│  ├─ quality.py              ← 品質評分
│  ├─ normalize.py            ← 統一輸出格式
│  └─ chunk.py                ← RAG 切段
├─ envs/                      ← 各工具獨立 uv 環境
│  ├─ markitdown/pyproject.toml
│  ├─ docling/pyproject.toml  (torch cu128)
│  └─ mineru/pyproject.toml   (torch cu128)
├─ skill/SKILL.md
└─ tests/
   ├─ unit/
   └─ fixtures/
```

### 環境隔離

- 主程式依賴僅 PyMuPDF 與少量工具套件，偵測與判斷都很快。
- 每個 engine 以 `envs/<工具>/.venv` 的 Python 透過 subprocess 執行（各 engine 附一支小 runner 腳本，在該環境內呼叫工具的 Python API，把結果寫到指定目錄）。
- 硬體：RTX 5070 Ti（Blackwell，16GB），torch 需 CUDA 12.8 以上版本，版本鎖在各 env 的 `pyproject.toml`。
- 某工具未安裝時視為不可用，router 跳到備援順序的下一個，並提示 `aidoc setup <工具>`。

### Engine 介面

```python
class Engine(Protocol):
    name: str
    def available(self) -> bool: ...
    def convert(self, src: Path, workdir: Path, opts: ConvertOptions) -> RawResult: ...
```

`RawResult` 包含：Markdown 字串、圖片檔路徑列表、頁界資訊（若工具有提供）、工具原始輸出路徑。`normalize.py` 負責把 `RawResult` 轉成統一格式。

## 4. 偵測與判斷

### probe（PDF 每頁取樣，最多 20 頁）

- `text_ratio`：有文字層的頁數比例
- `image_cover`：圖片佔頁面面積比例（取樣頁平均）
- `math_hint`：出現數學字型（CMMI、CMSY、Symbol、Cambria Math 等）或數學符號密度高
- `layout_hint`：多欄版面（文字區塊 x 座標分布呈雙峰）
- `has_table_lines`：頁面有構成格線的線段

### 判斷規則（依序比對，第一條符合即採用）

| 條件 | 主要 → 備援 |
|---|---|
| 非 PDF，MarkItDown 支援的格式 | MarkItDown → Docling（僅 docx/pptx/html） |
| 圖片（png/jpg/tiff…） | MinerU → Docling |
| PDF 且 `text_ratio < 0.5` 或 `image_cover > 0.6` | MinerU → Docling |
| PDF 且 `math_hint` | MinerU → Docling |
| 其他 PDF | Docling → MinerU → MarkItDown |

`--engine <名稱>` 可強制指定，此時不走備援。

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

`<檔名>.json`：

```json
{
  "source": "report.pdf",
  "sha256": "…",
  "pages": 42,
  "engine": "mineru",
  "tried": [{"engine": "mineru", "score": 0.93, "reasons": []}],
  "probe": {"text_ratio": 0.1, "image_cover": 0.85, "math_hint": false},
  "quality": {"score": 0.93, "level": "ok", "reasons": []},
  "lang": "chinese_cht+en",
  "elapsed_s": 38.2,
  "aidoc_version": "0.1.0"
}
```

### 批次

`aidoc batch <輸入目錄> -o <輸出目錄> [--force] [--engine X] [--lang en]`

- 以 sha256 快取：`out/<檔名>/<檔名>.json` 的 sha256 相同即跳過；`--force` 重新轉換。
- 同名檔案衝突時輸出目錄加上 sha256 前 8 碼。
- MarkItDown 任務以 process pool 平行；Docling／MinerU 共用 GPU，序列執行。
- 輸出 `out/_manifest.jsonl`：每檔一行 `{source, status, engine, score, level, error?}`。

### RAG 切段

`aidoc chunk <輸出目錄> [--max-tokens 800]`

- 依標題階層切段，超過上限時在段落邊界再切；表格與公式區塊不切開（單一表格超長時整塊保留並標記 `oversized: true`）。
- token 以 `tiktoken` 的 `cl100k_base` 估算。
- 輸出 `chunks.jsonl`：`{id, source, heading_path, page_start, page_end, text}`。
- 不做向量化（由使用者自行選擇向量資料庫）。

## 6. Claude Code skill

`skill/SKILL.md`：

- 觸發詞：「轉成 markdown」「把這份文件轉給 AI 讀」「convert this doc」等。
- 行為：執行 `aidoc convert <檔案> -o <暫存或指定目錄>`，讀出 `.md`；`level: "low"` 時主動告知並附上 `reasons`。
- 判斷邏輯不寫在 skill 裡，只呼叫 CLI。

## 7. 安裝

`aidoc setup [markitdown|docling|mineru|all]`

- 以 uv 在 `envs/<工具>/` 建立 `.venv` 並安裝鎖定的版本。
- 預先下載模型（Docling 版面／表格模型、MinerU 模型）。
- 最後執行自我檢查：轉換一個內建小樣本，並確認 GPU 可用（`torch.cuda.is_available()`）。

## 8. 錯誤處理

- 工具未安裝：跳過並提示安裝指令。
- subprocess 逾時：預設每頁 60 秒，最少 120 秒，可用 `--timeout` 覆寫；逾時視為該工具失敗，走下一個備援。
- 單檔失敗：記入 manifest 的 `error`，批次繼續。
- GPU 顯存不足（OOM）：MinerU 失敗時自動以較小 batch 重試一次，仍失敗則走備援。

## 9. 測試

- **單元測試**（不需 GPU）：`probe`、`router`、`quality`、`normalize`、`chunk`。
- **整合測試**（`@pytest.mark.slow`）：`tests/fixtures/` 每類一個小樣本——docx、pptx、xlsx、電子 PDF、繁中掃描 PDF、中英混排掃描 PDF、公式 PDF、png。斷言：引擎選擇正確、品質為 `ok`、產出檔案齊全。

## 10. 不在範圍內（第一版）

- Marker 引擎（見第 2 節）
- 向量化與向量資料庫整合
- 網頁 UI 或 HTTP API
- LLM 輔助修正輸出
