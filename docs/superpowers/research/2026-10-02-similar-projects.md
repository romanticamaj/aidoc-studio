# 類似 Doc4AI Studio 的開源專案調查

- **日期：** 2026-10-02
- **範圍：** GitHub 上與 Doc4AI Studio（`romanticamaj/doc4ai-studio`）功能重疊的開源專案，分四類：A 最接近的整合型平台、B 單一引擎轉換器與其包裝、C 文件 → MCP server、D 文件解析能力強的 RAG 平台（只看解析側）。
- **方法：** GitHub REST API（`gh api repos/...`、`search/repositories`、topic `pdf-to-markdown`／`document-parsing`）取星數、最後 push 日、授權、主要語言、最新 release；逐一讀 README（必要時讀原始碼，例如 Tianshu 的 `backend/mcp_server.py`）；WebSearch 補 RAGFlow／Open WebUI／Dify 的解析選項；HN Algolia 查 2025-09 之後的 Show HN。
- **數字口徑：** ★ = 2026-10-02 當下星數；「最後活動」= `pushed_at`；release = 最新 GitHub Release（沒有則註明）。授權照 GitHub 偵測結果，`NOASSERTION` 代表自訂授權（另註明內容）。
- **未逐項驗證的部分**會標「（未驗證）」，不要當事實引用。

> **一句話結論：** 「多引擎 + 自動路由 + 逐頁品質評分與自動 fallback + 精緻 Web UI + 有權限控管的遠端 MCP」這個**組合**，目前沒有任何一個開源專案完整做到。但每一個單點都有人做得比我們好或至少一樣好：**pdfmux** 已經是「逐頁路由 + 自我稽核 + 重抽」的同概念產品（無 UI）；**MinerU Tianshu** 是最像的整合型平台（Vue UI + 佇列 + 多使用者 + MCP，但單引擎為主、無品質 fallback）；**MinerU 4.0 本身**已內建文件庫、分頁定位讀取、搜尋、Router 與 Gradio WebUI，侵蝕我們的 MCP 讀取差異化；**anymd／pdf-mcp** 在「大文件分頁讀取 + token 預算 + 搜尋」的 MCP 體驗上更成熟、安裝更簡單。

---

## 0. Doc4AI Studio 自己的定位（對照基準）

| 面向 | Doc4AI Studio 現況（README 與 specs） |
|---|---|
| 引擎 | MarkItDown 0.1.8（Office/HTML/EPUB）、Docling 2.x（數位 PDF）、MinerU 4.x（掃描、圖片、公式、複雜版面），各自獨立 `uv` env、subprocess 呼叫 |
| 路由 | 規則式：探測文字層、影像覆蓋率、公式、表格 → 選引擎 |
| 品質 | 逐頁評分（字元/頁、garbage ratio、缺表格），失敗換下一個引擎；**逐頁偵測壞文字層（無 ToUnicode 字型）並以 OCR 逐頁修復後接回原位**（Docling pypdfium FULL_PAGE + RapidOCR，退而 MinerU `ocr_mode=ocr`） |
| 語言 | 繁中 + 英文 OCR、混排 |
| 大文件 | >40 頁分段轉換、中斷續跑、跨段表格接合；1000+ 頁實測 |
| RAG | 依標題層級切段、保留頁碼範圍、不切斷表格 |
| Web | React：可續傳分塊上傳、SSE 即時進度佇列、Library、PDF/Markdown 左右對照且頁同步捲動、品質徽章、chunk 預覽、引擎安裝頁、深淺色 |
| 介面 | CLI `aidoc`、Claude Code skill、MCP server（Streamable HTTP、規格 2026-07-28、PAT + scopes、後台看連線與呼叫紀錄、頁範圍讀取＋token 預算、逐頁 FTS 搜尋） |
| 授權 | AGPL-3.0；GitHub 0 ★（2026-10-01 剛公開） |

---

## A. 最接近的整合型競品（多引擎／UI／API、可自架）

| # | 專案 | ★ | 最後活動 / release | 授權 | 語言 |
|---|---|---|---|---|---|
| A1 | [magicyuan876/mineru-tianshu](https://github.com/magicyuan876/mineru-tianshu)（天樞） | 832 | 2026-09-28 / v2.1.0（09-13） | Apache-2.0 | Python + Vue |
| A2 | [NameetP/pdfmux](https://github.com/NameetP/pdfmux) | 82 | 2026-09-12 / v1.8.7（07-27） | MIT | Python |
| A3 | [CHEN010325/paddleocr-local](https://github.com/CHEN010325/paddleocr-local) | 133 | 2026-09-10 / 無 release | Apache-2.0 | Python |
| A4 | [KingsleyOWO/Semark](https://github.com/KingsleyOWO/Semark) | 28 | 2026-08-14 / v0.3.1（08-12） | Apache-2.0 | Python |
| A5 | [scub-france/docling-Studio](https://github.com/scub-france/docling-Studio) | 263 | 2026-09-28 / v0.7.2（09-22） | MIT | Python + Vue |
| A6 | [AeternaLabsHQ/pullmd](https://github.com/AeternaLabsHQ/pullmd) | 485 | 2026-09-21 / v3.12.0 | AGPL-3.0 | JavaScript |
| A7 | [magicrew/doc7](https://github.com/magicrew/doc7) | 1,152 | 2026-08-07 / v0.1.2 | MIT | Go |
| A8 | [SylphxAI/anymd](https://github.com/SylphxAI/anymd)（前身 pdf-reader-mcp） | 1,016 | 2026-10-02 / v8.4.0 | MIT（另有付費 Pro） | Rust |
| A9 | [xberg-io/xberg](https://github.com/xberg-io/xberg)（前身 Goldziher/kreuzberg） | 9,363 | 2026-10-02 / v1.3.2 | MIT | Rust |
| A10 | [ibrahimqureshae/mdflux](https://github.com/ibrahimqureshae/mdflux) | 541 | 2026-09-07 | MIT | Python（桌面 App） |
| A11 | [drmingler/docling-api](https://github.com/drmingler/docling-api) | 1,689 | 2026-08-04 | MIT | Python |
| A12 | [opendatalab/MinerU](https://github.com/opendatalab/MinerU) 4.0（本身已變成平台，詳見 B3） | 80,973 | 2026-09-30 / 4.0.10（09-29） | MinerU Open Source License | Python |

**A1 MinerU Tianshu（天樞）— 最像我們的整合型平台。**
「企業級 AI 資料預處理平台」：Vue 3 前端 + FastAPI + LitServe GPU 負載均衡 + Redis 佇列 + SQLite(WAL)。引擎：MinerU 3.4.5（pipeline／vlm／hybrid）、MarkItDown（EPUB 等）、外掛式格式引擎（FASTA/GenBank），另有音訊（SenseVoice）、影片（關鍵幀 OCR）、浮水印去除；v2.0 起**移除 PaddleOCR 引擎、「引擎聚焦 MinerU」**。大文件：>500 頁自動拆成父子任務並行處理、按頁碼合併（未提續跑）。企業功能：JWT、多使用者資料隔離、RBAC、API Key、SSO 預留（OIDC/SAML）、Webhook（HMAC 簽章、重試、死信）、稽核日誌、ZIP 批次、多模態模型自動產圖說、RustFS/S3 物件儲存、Docker 一鍵部署。MCP：`backend/mcp_server.py` 使用**舊式 HTTP+SSE transport**（`/sse` + `/messages`），以 `MCP_API_KEYS` 環境變數做 Bearer／X-API-Key 常數時間比對；tools 只有 `parse_document`、`get_task_status`、`list_tasks`、`get_queue_stats`——**沒有分頁讀取、搜尋、scopes、後台呼叫紀錄**。品質 fallback：README 未見。
→ **重疊**：Web 平台 + 佇列 + 大文件拆分 + MCP + MinerU/MarkItDown。**差異**：它強在多使用者／企業整合／多模態；我們強在多引擎路由、逐頁品質 fallback 與壞文字層修復、MCP 讀取體驗與權限模型、繁中。

**A2 pdfmux — 概念上最接近「路由 + 品質評分 + 自動重抽」的專案。**
「Self-healing PDF extraction」：逐頁把頁面路由到 7 個後端（PyMuPDF、OpenDataLoader、RapidOCR、Docling、Surya、Marker、Mistral OCR）+ BYOK LLM（Gemini/Claude/GPT-4o/Ollama），抽完以「4 訊號動態信心分數」（字元密度、OCR 雜訊比、表格完整度、標題結構）稽核，**失敗頁用更強的後端重抽，修不好就標出而非靜默丟掉**。另有 `pdfmux verify` 可稽核**任何**引擎（Reducto、LlamaParse、Docling…）的輸出，找出被靜默丟掉的頁。成本模式（`--budget`）、schema 抽取、`--chunk --max-tokens`、NDJSON 串流、資料夾 watch、hash 快取、`doctor` 檢查缺哪個 extra、MCP server（stdio 或 HTTP，7 個 tools）、LangChain/LlamaIndex loader。只處理 PDF；無自架 Web UI（雲端版 app.pdfmux.com 另售）。README 稱「patent-pending method」。
→ **重疊度極高**（逐頁路由、品質分數、fallback、chunk、MCP）。**差異**：pdfmux 是函式庫/CLI、純 PDF、引擎在同一 Python env；我們有完整 Web App、Office/EPUB、引擎隔離 env、繁中 OCR、超長書分段續跑。它的 `verify`（稽核外部引擎）與成本預估是我們沒有的。

**A3 PaddleOCR Local — 「多模型 + WebUI + 單卡顯存管理」。**
一套 WebUI 跑五種 OCR/解析模型：PaddleOCR-VL 1.6、PP-OCRv6、OvisOCR2、HPD-Parsing、NaviDC-OCR（另有外部 API 模式）。**同時只啟動一個模型、切換時釋放顯存、顯存預檢推薦模型**。逐頁進度、原文對照、表格/公式渲染、PDF 按頁或按批解析並**從中斷處恢復、失敗批次單獨重試**、CLI、Watch Folder、API Token、Windows 一鍵、macOS MLX、CPU compose。模型選擇是手動，未見自動路由或品質 fallback；未見 MCP。
→ **重疊**：本機 Web 工作台、續跑、原文對照、Windows 友善。**差異**：它是「手動選 VLM OCR 模型」，我們是「自動路由 + 品質 fallback」；它的顯存預檢/單模型常駐管理值得學。

**A4 Semark — 台灣作者、繁中優先的「語意 Markdown」。**
MinerU 解析 + VLM/LLM「reviewer」重寫：表格格線修復（重讀表格影像重建欄位）、截圖轉語意描述與選單路徑、流程圖轉結構化條件分支、表單轉固定 RAG 模板、長文件自動拆附件、個資遮罩、source map、chunks、quality gate、Web UI + API、Docker；**繁中/英雙語輸出**，示例多為台灣公部門文件。
→ **重疊**：繁中、MinerU、RAG chunk、Web UI。**差異**：它用 LLM 改寫成「語意」Markdown（非逐字），我們追求忠實轉換 + 品質保證。兩者可互補（Semark 當我們的後處理選項），也是在台灣受眾上的直接比較對象。

**A5 Docling Studio — Docling 的視覺化工作室。**
Vue 3 + FastAPI + SQLite：PDF 檢視器 + bounding box 疊圖、右側結果與當前頁同步、可設定 Docling pipeline（OCR、表格、公式、圖片分類/描述）、chunking（hierarchical/hybrid/page）可行內編輯、一鍵 Docling → chunk → embedding → OpenSearch、Neo4j 文件圖、上傳上限、限流、深淺色、FR/EN。單引擎，無 MCP。
→ **重疊**：頁同步檢視、chunk 預覽、深淺色。**它領先**：bbox 疊圖、chunk 行內編輯、直接灌向量庫。

**A6 PullMD — 有 OAuth 2.1 的 URL/文件 → Markdown 服務。**
主打網頁（Readability/Trafilatura/Playwright，Reddit/HN 專用管線），v3 擴到 PDF/Office/EPUB（MarkItDown sidecar）、選用 OCR tier（Mistral OCR 等，失敗**自動退回免費路徑**）、圖片說明與音訊轉寫、YouTube 字幕、frontmatter 帶品質與用量。PWA、REST、**MCP 在 `/mcp`（Streamable HTTP、stateless）**、Claude Code skill。認證：disabled／single-admin／multi-user，API key（`pmd_` 前綴、SHA-256 雜湊、只顯示一次），**完整 OAuth 2.1（DCR、PKCE、RFC 8414/9728、refresh token 輪替與重放偵測）可直接接 claude.ai 自訂 connector**；SSRF 防護；`?query=` + `max_tokens` 只回相關段落。
→ **重疊**：自架、AGPL、Streamable HTTP MCP + PAT、skill。**它領先**：OAuth 2.1（我們 Phase 2 才做）、query-scoped 擷取。PDF 能力遠弱於我們。

**A7 doc7 — 純 VLM 路線。**
Go 單一執行檔：把每頁渲染成圖丟給**你自己的 OpenAI 相容多模態模型**（LM Studio/Ollama/遠端），不需要 OCR 堆疊。有 CLI、批次、**逐頁 manifest、只重跑失敗頁（`--resume`，SHA-256 校驗）**、context 超限時降解析度重試、grounding 檢查、MCP（stdio）、Go SDK、非同步 HTTP 服務。自家 benchmark 在純點陣頁面上勝過 MarkItDown OCR 與 Docling 標準管線。
→ **差異**：doc7 是「一個 VLM 打天下」，我們是「多專用引擎 + 規則路由」。它的逐頁重跑設計與我們的分段續跑相近。

**A8 anymd — 最成熟的「文件 → Agent」MCP 體驗。**
Rust 原生、npm/pip/Docker/MCPB 一鍵、`npx @sylphx/anymd setup` 自動加進所有 MCP client。PDF 以字形位置重建閱讀順序與無框表格；OCR 用本機 doc-VLM 或 tesseract；音影片轉寫。MCP tools：`read`（`pages`、`max_tokens` 預設 20000、`cursor` 續讀、`<!-- page N -->` 錨點）、`inspect`（render_page、ocr_pages、structure、compare、cite_check）、`search`（找不到精確字串時退 BM25）。自建 AgentDocBench（含它輸的類別）。stdio only，`--allow-dir` 限縮。
→ **重疊**：分頁讀取 + token 預算 + 頁錨點 + 搜尋，這正是我們 MCP 的賣點，而它更輕、裝更快。**差異**：無 Web UI、無遠端多使用者權限、PDF 轉換靠自家規則引擎（掃描/複雜版面不如 MinerU/Docling）。

**A9 xberg（原 kreuzberg）— 多語言綁定的抽取核心。**
Rust 核心 + Python/Node/WASM/Java/Go/.NET/PHP/Ruby/Elixir/Dart 綁定；OCR 後端 Tesseract/PaddleOCR/Candle/VLM，**fallback chains 與信心分數**；CLI 14 個指令、REST（`xberg serve`）、MCP（stdio）、Docker/Helm、內容 hash 快取、chunk、embed。無 UI。
→ 可視為「函式庫層的競品/可整合對象」，不是終端產品。

**A10 MDFlux** — 本機桌面 App（Windows/Linux），PDF/Office/EPUB/影像/音訊 → Markdown，掃描 PDF OCR、資料夾批次、選用 LLM 清理；主打比 VLM 省 6 倍 token。無 MCP、單機。
**A11 docling-api** — Docling 包成可擴充後端（FastAPI + Celery/Redis 佇列、Docker GPU），單引擎、無 UI、無 MCP；2024 年起的老牌包裝。
**A12 MinerU 4.0** — 見 B3；重點是它已具備「文件庫 + 分頁定位讀取 + 搜尋 + Router + WebUI」。

其他小型同類（★ < 150，僅列名）：[ZHYX91/docwen](https://github.com/ZHYX91/docwen)（AGPL，本機 GUI+CLI 轉換/OCR/校對）、[ber-ners/mineru-textbook-to-md](https://github.com/ber-ners/mineru-textbook-to-md)（MinerU 教材轉 MD、品質報告、斷點續跑）、[TylerMorrison21/paperflow](https://github.com/TylerMorrison21/paperflow)（Marker/MinerU/PyMuPDF/Docling 輸出的後處理 + 自架 UI）、[thomas-villani/all2md](https://github.com/thomas-villani/all2md)、[hwdsl2/docker-docling](https://github.com/hwdsl2/docker-docling)。

---

## B. 單一引擎轉換器與其包裝

| # | 專案 | ★ | 最後活動 / release | 授權 | 路線 | UI/API/MCP | CJK/繁中 |
|---|---|---|---|---|---|---|---|
| B1 | [microsoft/markitdown](https://github.com/microsoft/markitdown) | 187,933 | 10-02 / v0.1.8（09-21） | MIT | 格式轉換器；`markitdown-ocr` 外掛用 LLM Vision OCR | CLI；官方 `markitdown-mcp`（stdio/Streamable HTTP/SSE） | 依 LLM |
| B2 | [firecrawl/anydoc](https://github.com/firecrawl/anydoc) | 22,394 | 08-28 | MIT | Rust，Office/ODF/RTF/EPUB/CSV/PDF → GFM，毫秒級；掃描走 Firecrawl 雲端 | CLI、Node/Python、WASM、Agent Skill | 掃描需雲端 |
| B3 | [opendatalab/MinerU](https://github.com/opendatalab/MinerU) | 80,973 | 09-30 / 4.0.10 | MinerU OSL（Apache 2.0 + 附加條件） | 4 個 tier（flash/basic/standard/advanced）；pipeline 小模型 ONNX/Torch + VLM（llama.cpp/vLLM/LMDeploy）；DocVortex 原生解析 Office/EPUB/OFD/HTML | Python SDK、V1 API、**Router**、**Gradio WebUI**、**文件庫 + 分頁/區塊定位讀取 + 搜尋**；官方 MCP 在 MinerU-Ecosystem | 109 語 OCR，含繁中 |
| B4 | [docling-project/docling](https://github.com/docling-project/docling) | 68,289 | 10-02 / v2.132.0 | MIT | 版面 + TableFormer + 可選 OCR（EasyOCR/RapidOCR/Tesseract）+ VLM pipeline | 函式庫/CLI | 依 OCR 後端 |
| B4' | [docling-project/docling-serve](https://github.com/docling-project/docling-serve) | 1,845 | 10-01 / v1.36.0 | MIT | Docling API 服務（同步/非同步、chunk） | REST v1 + `/ui` playground；cu128/cu130 映像 | 同上 |
| B5 | [datalab-to/marker](https://github.com/datalab-to/marker) | 40,159 | 09-13 / v2.0.0（07-20） | 程式 Apache-2.0；**模型 OpenRAIL-M 修改版（商用有營收門檻）** | v2：版面/OCR/表格統一由 surya VLM（vLLM 或 llama.cpp server）；`balanced` 模式「任何嵌入文字壞掉就整頁重 OCR」、`fast` 模式做區塊級修復；`--use_llm` 跨頁表格合併 | CLI、多 worker、多 GPU | 90+ 語 |
| B5' | [datalab-to/surya](https://github.com/datalab-to/surya) | 21,444 | 09-11 | Apache-2.0（模型同上） | OCR/版面/閱讀順序/表格 | 函式庫 | 90+ 語 |
| B5'' | [datalab-to/chandra](https://github.com/datalab-to/chandra) | 12,370 | 06-26 | 程式 Apache-2.0；**商用自架需授權** | Chandra 2（2026-03）OCR VLM，表格/表單/手寫 | HF 或 vLLM | 90+ 語 |
| B6 | [allenai/olmocr](https://github.com/allenai/olmocr) | 19,687 | 03-25 / v0.4.27（03-12） | Apache-2.0 | olmOCR-2-7B VLM 線性化 PDF，vLLM；olmOCR-Bench | CLI、Docker、可接外部 vLLM | 以英文為主（未驗證中文品質） |
| B7 | [PaddlePaddle/PaddleOCR](https://github.com/PaddlePaddle/PaddleOCR) | 90,505 | 09-16 / v3.7.0（06-11） | Apache-2.0 | PaddleOCR-VL-1.6（0.9B，OmniDocBench v1.6 96.3%）、PP-StructureV3、PP-OCRv6（50 語單模型）、HPD-Parsing | CLI/SDK、serving、**官方 MCP server**（FastMCP；本機推論／自架服務／AI Studio） | 中文最強之一（含繁中） |
| B8 | [opendataloader-project/opendataloader-pdf](https://github.com/opendataloader-project/opendataloader-pdf) | 29,453 | 10-02 / v2.5.12 | Apache-2.0 | Java；確定性本地模式（XY-Cut++、bbox）+ **hybrid 模式把複雜頁路由到 AI 後端**；自稱 benchmark 0.907 第一 | CLI/SDK | hybrid 模式 80+ 語 OCR |
| B9 | [run-llama/liteparse](https://github.com/run-llama/liteparse) | 12,761 | 10-02 | Apache-2.0 | PDFium 空間文字 + 選擇性 OCR（內建 Tesseract 或任何 OCR HTTP server）、**複雜度偵測**（解析前判斷要不要 OCR/重解析）、頁面截圖 | CLI/函式庫 | 依 OCR |
| B10 | [Unstructured-IO/unstructured](https://github.com/Unstructured-IO/unstructured) | 15,527 | 10-02 / 0.27.10 | Apache-2.0 | 元素化 ETL，hi_res（版面模型 + OCR）/fast 策略 | 函式庫；API 為商用 | Tesseract 語言包 |
| B11 | [pymupdf/pymupdf4llm](https://github.com/pymupdf/pymupdf4llm) | 2,225 | 10-01 / v0.3.4（02-14） | AGPL-3.0 | 數位 PDF 快速轉 MD，可選 OCR | 函式庫 | 依 Tesseract |
| B12 | [studio-dots-ai/dots.ocr](https://github.com/studio-dots-ai/dots.ocr)（→ dots.mocr） | 9,158 | 03-24 | MIT | 1.7B 單一 VLM 版面 + 辨識；2026-03 改名 [dots.mocr](https://github.com/studio-dots-ai/dots.mocr)（353★） | vLLM | 多語強 |
| B13 | [deepseek-ai/DeepSeek-OCR](https://github.com/deepseek-ai/DeepSeek-OCR) / [OCR-2](https://github.com/deepseek-ai/DeepSeek-OCR-2) | 23,928 / 3,443 | 01-27 / 02-03 | MIT / Apache-2.0 | 「光學壓縮」VLM，vLLM 上游支援 | 模型 | 中文佳 |
| B14 | [zai-org/GLM-OCR](https://github.com/zai-org/GLM-OCR) | 7,483 | 04-21 | Apache-2.0 | 智譜 OCR VLM | 模型 | 中文佳 |
| B15 | [Yuliang-Liu/MonkeyOCR](https://github.com/Yuliang-Liu/MonkeyOCR) / [bytedance/Dolphin](https://github.com/bytedance/Dolphin) | 6,652 / 9,058 | 07-20 / 03-25 | Apache-2.0 / 自訂 | 輕量 LMM 文件解析 | 模型 + demo | 中英 |
| B16 | [getomni-ai/zerox](https://github.com/getomni-ai/zerox) | 12,261 | 2025-05-20（最後 release 2024-12） | MIT | 頁面轉圖丟雲端 VLM | SDK | 依模型；**已近停更** |
| B17 | [facebookresearch/nougat](https://github.com/facebookresearch/nougat) / [GOT-OCR2.0](https://github.com/Ucas-HaoranWei/GOT-OCR2.0) | 10,083 / 8,228 | 2025-02 | MIT / 無 | 學術論文 OCR / 通用 OCR-2.0 | 模型 | **已停更**，被 olmOCR/MinerU-VLM 取代 |
| B18 | [Filimoa/open-parse](https://github.com/Filimoa/open-parse)、[NanoNets/docstrange](https://github.com/NanoNets/docstrange)、[QuivrHQ/megaparse](https://github.com/The-Vibe-Company/megaparse)、[adithya-s-k/omniparse](https://github.com/adithya-s-k/omniparse) | 3,154 / 1,574 / 7,415 / 7,932 | 2024-11～2025-12 | MIT / MIT / Apache / GPL-3.0 | 各式 LLM 友善解析 | — | **活躍度低**，不建議整合 |
| B19 | [emcf/thepipe](https://github.com/emcf/thepipe)、[oomol-lab/pdf-craft](https://github.com/oomol-lab/pdf-craft) | 1,530 / 6,341 | 09-28 / 09-30 | MIT | thepipe：VLM 抽取；pdf-craft：掃描書 → MD/EPUB（章節、註腳、目錄、翻譯） | 函式庫 | pdf-craft 中文書友善 |

**對我們的意義：**
- MarkItDown、Docling、MinerU 是我們的上游，三者都很活躍；**MinerU 4.0 的 tier/Router/文件庫**讓它本身越來越像平台（見 §5）。
- 「VLM OCR」這條線（PaddleOCR-VL-1.6、MinerU2.5-Pro VLM、olmOCR-2、Chandra 2、dots.mocr、DeepSeek-OCR-2、GLM-OCR）是 2025-26 品質提升的主戰場，**我們目前沒有獨立的 VLM OCR 引擎**（只透過 MinerU 的 tier 間接用到）。
- Marker v2 的「嵌入文字壞掉就整頁重 OCR」與我們的壞文字層修復是同一想法；它做到區塊級（`fast` 模式），我們是頁級。

---

## C. 文件 → MCP server

| # | 專案 | ★ | 最後活動 | 授權 | Transport | 認證 | 多引擎 | 大文件分頁 |
|---|---|---|---|---|---|---|---|---|
| C1 | [markitdown-mcp](https://github.com/microsoft/markitdown/tree/main/packages/markitdown-mcp)（markitdown repo 內） | （隨 markitdown） | 10-02 | MIT | stdio、Streamable HTTP、SSE（預設綁 localhost） | **無**（官方明說只給本機可信 agent） | 否；單一 `convert_to_markdown(uri)` | 否 |
| C2 | [docling-project/docling-mcp](https://github.com/docling-project/docling-mcp) | 765 | 10-01 / v3.2.1 | MIT | stdio、SSE（另有 streamable-http 選項，未驗證） | 遠端模式用 docling-serve API key | Docling 本地/遠端/hybrid 自動 fallback | 有文件生成/操作 tools；分頁讀取未見 |
| C3 | [opendatalab/MinerU-Ecosystem](https://github.com/opendatalab/MinerU-Ecosystem) `mcp/` | 218 | 09-22 | Apache-2.0 | （未驗證） | MinerU Open API key（雲端） | 否 | — |
| C4 | [opendatalab/MinerU-Document-Explorer](https://github.com/opendatalab/MinerU-Document-Explorer) | 637 | 04-26 | MIT | MCP tools | — | MinerU | **有**：目錄、章節讀取、文件內搜尋、BM25/向量/混合 + rerank、LLM Wiki |
| C5 | PaddleOCR MCP（[docs](https://github.com/PaddlePaddle/PaddleOCR/blob/main/docs/version3.x/integrations/mcp_server.en.md)） | （隨 PaddleOCR） | 09-16 | Apache-2.0 | FastMCP v2 | 依部署 | OCR / PP-StructureV3 / VL | 否 |
| C6 | [SylphxAI/anymd](https://github.com/SylphxAI/anymd) | 1,016 | 10-02 | MIT | stdio | 目錄白名單 | 自家引擎 + doc-VLM/tesseract OCR | **有**：`pages`、`max_tokens`、`cursor`、頁錨點、BM25 搜尋 |
| C7 | [jztan/pdf-mcp](https://github.com/jztan/pdf-mcp) | 147 | 10-01 / v3.4.0 | MIT | stdio（.mcpb 一鍵裝 Claude Desktop） | 本機 | PyMuPDF + Tesseract | **有**：只讀需要的頁、BM25+語意混合搜尋、整個資料夾 triage、SQLite 快取、CJK/直書閱讀順序、**隱藏/注入文字標為 untrusted** |
| C8 | [zcaceres/markdownify-mcp](https://github.com/zcaceres/markdownify-mcp) | 2,997 | 09-25 | MIT | stdio | 無 | 包 MarkItDown | 否 |
| C9 | [chrisryugj/kordoc](https://github.com/chrisryugj/kordoc) | 2,328 | 10-02 | MIT | stdio（CLI + MCP） | — | 自家 HWP/HWPX/PDF/Office 解析 | 韓國公文表格無損、表單自動填寫、新舊對照 |
| C10 | [DocSlicer/DocSlicer](https://github.com/DocSlicer/DocSlicer) | 35 | 09-01 | AGPL-3.0 | MCP | — | 確定性解析（無 ML） | **vectorless RAG**：取大綱 → 選章節 → 只讀該段 |
| C11 | Tianshu / PullMD / pdfmux / xberg / doc7 | 見 A 類 | | | SSE / **Streamable HTTP + OAuth 2.1** / stdio+HTTP / stdio / stdio | API key / **PAT + OAuth** / ? / — / — | | PullMD 有 query + max_tokens |
| C12 | [FutureUnreal/mcp-pdf2md](https://github.com/FutureUnreal/mcp-pdf2md)、[KorigamiK/markitdown_mcp_server](https://github.com/KorigamiK/markitdown_mcp_server)、[neosun100/mineru-mcp-server](https://github.com/neosun100/mineru-mcp-server)、[zanetworker/mcp-docling](https://github.com/zanetworker/mcp-docling) | 35 / 88 / 28 / 19 | 2025-03～2026-05 | 多為 MIT | stdio | 無 | 單引擎包裝 | 否 |

**觀察：**
1. 絕大多數文件 MCP 是 **stdio + 本機 + 無認證**。有「遠端 HTTP + 帳號/token 管理」的只有 Tianshu（SSE + 靜態 API key 清單）與 PullMD（Streamable HTTP + API key + OAuth 2.1）。**Streamable HTTP（2026-07-28 stateless）+ 每把 PAT 有 scopes + 後台看每次呼叫**這一組合在文件 MCP 領域確實少見——這是我們真正的差異化之一。
2. 但「大文件分頁讀取 + token 預算 + 搜尋」已經不稀奇：anymd、pdf-mcp、MinerU 4.0（CLI 定位器）、MinerU Document Explorer、DocSlicer 都有，而且多數**安裝只要一行**（npx / uv tool / .mcpb）。
3. pdf-mcp 把「PDF 內隱藏或注入的文字標為 untrusted」做成功能——對我們的 MCP（把文件內容回給 agent）是值得抄的安全設計。

---

## D. 解析能力強的 RAG 平台（只看解析側）

| # | 專案 | ★ | 最後活動 | 授權 | 解析方式 |
|---|---|---|---|---|---|
| D1 | [infiniflow/ragflow](https://github.com/infiniflow/ragflow) | 91,594 | 10-01（1.0.0-rc1 於 09-29） | Apache-2.0 | **DeepDoc**（版面 DLR + OCR + 表格 TSR），1.0 改以 Go 實作、CPU 推論；**可切換 PDF parser：DeepDoc／MinerU（v0.22 起）／Docling／PaddleOCR／TCADP／OpenDataLoader／Naive**；依文件類型的切段模板（論文、書、法規、表格、問答…）；chunk 可視化與人工修正 |
| D2 | [langgenius/dify](https://github.com/langgenius/dify) | 157,716 | 10-02 | Dify OSL（Apache 2.0 + 條件） | Knowledge Pipeline：FILE → 解析節點（內建抽取或 **MinerU 等外掛**）→ Chunker → 知識庫 |
| D3 | [open-webui/open-webui](https://github.com/open-webui/open-webui) | 153,774 | 10-02 | Open WebUI License（BSD-3 + 品牌條款） | 可切換「content extraction engine」：預設、Tika、**Docling（docling-serve）**、**MinerU**、Mistral OCR、Datalab、Document Intelligence、PaddleOCR 等；已知 issue：Docling/Tika 不寫頁碼 metadata（[#14011](https://github.com/open-webui/open-webui/issues/14011)） |
| D4 | [Cinnamon/kotaemon](https://github.com/Cinnamon/kotaemon) | 25,798 | 07-14 | Apache-2.0 | File loader 可選：預設、Unstructured、Azure Document Intelligence、Adobe PDF Extract、**Docling**、**PaddleOCR**；PDF 檢視器顯示引用來源 |
| D5 | [onyx-dot-app/onyx](https://github.com/onyx-dot-app/onyx) | 32,313 | 10-02 | 自訂（MIT 核心 + EE） | 內建檔案處理，可選 Unstructured API（未逐項驗證） |
| D6 | [Zipstack/unstract](https://github.com/Zipstack/unstract) | 7,270 | 10-01 | AGPL-3.0 | Text extractor 可選 LLMWhisperer / Unstructured.io；Prompt Studio 做結構化抽取；有 MCP server |
| D7 | [HKUDS/RAG-Anything](https://github.com/HKUDS/RAG-Anything) | 23,475 | 09-15 | MIT | 多模態 RAG，parser 可選 MinerU / Docling（未逐項驗證最新選項） |

**對我們的意義：** 主流 RAG 平台都已做成「parser 可插拔」，**MinerU 與 Docling 是共同選項**。我們不該跟它們比 RAG，而是**當它們的上游**：提供 docling-serve 或 Open WebUI「external extraction engine」相容的 API，讓 Open WebUI/RAGFlow/Dify 使用者直接把 Doc4AI 當解析器（帶品質 fallback 與繁中）。Open WebUI 的「頁碼 metadata 遺失」issue 正好是我們 `<!-- page: N -->` + chunk 頁碼範圍能解決的痛點。

---

## 5. 功能比較矩陣（最接近的 10 個 vs Doc4AI Studio）

圖例：✅ 有　◐ 部分／有條件　✗ 無　? 未驗證

| 功能 | **Doc4AI** | Tianshu | pdfmux | MinerU 4.0 | Docling Studio / serve | PaddleOCR Local | Semark | PullMD | anymd | doc7 | xberg |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 多個轉換引擎 | ✅ 3 個 | ◐ MinerU 為主 + MarkItDown | ✅ 7 + LLM | ◐ 自家多 tier | ✗ Docling | ✅ 5 模型 | ✗ MinerU + LLM | ◐ MarkItDown + OCR API | ✗ 自家 | ✗ 任一 VLM | ◐ 多 OCR 後端 |
| 依文件探測自動路由 | ✅ | ◐ 依副檔名 | ✅ 逐頁 | ◐ 依格式／tier | ✗ | ✗ 手動 | ✗ | ◐ | ◐ | ✗ | ◐ |
| 逐頁品質分數 | ✅ | ? | ✅ | ? | ✗ | ✗ | ◐ quality gate | ◐ frontmatter | ✗ | ◐ grounding | ◐ 信心分數 |
| 自動換引擎 fallback | ✅ | ✗ | ✅ 逐頁重抽 | ◐ 需使用者同意 | ✗（docling-mcp 有 local/remote fallback） | ✗ | ✗ | ◐ OCR 失敗退回 | ✗ | ◐ 降解析度重試 | ◐ OCR chain |
| 壞文字層逐頁 OCR 修復 | ✅ | ? | ✅ | ◐ `auto` 自偵測 | ✗ | n/a | ✗ | ✗ | ✗ | n/a | ? |
| 稽核外部引擎輸出 | ✗ | ✗ | ✅ `verify` | ✗ | ✗ | ✗ | ✗ | ✗ | ◐ `cite_check` | ✗ | ✗ |
| 繁中 OCR | ✅ 實測 | ◐ 依 MinerU | ◐ RapidOCR | ✅ | ◐ 依 OCR | ✅ | ✅ 繁中優先 | ◐ 依 API | ◐ | ◐ 依模型 | ◐ |
| 1000+ 頁分段 + 續跑 | ✅ | ◐ 500 頁拆分並行，續跑? | ◐ 串流 | ◐ 分頁視窗 | ◐ 頁數上限 | ✅ 按批續跑 | ◐ 拆附件 | ✗ | ◐ 逐頁 | ✅ 失敗頁重跑 | ◐ |
| RAG chunk + 頁碼來源 | ✅ | ✗ | ✅ | ◐ 定位器 | ✅ + 灌向量庫 | ✗ | ✅ source map | ✗ | ◐ 頁錨點 | ◐ | ✅ |
| Web UI | ✅ React | ✅ Vue（多使用者） | ✗（雲端另售） | ◐ Gradio | ✅ Vue | ✅ | ✅ | ✅ PWA | ✗ | ✗ | ✗ |
| 原文/結果左右對照、頁同步 | ✅ | ? | ✗ | ◐ Gradio 對照 | ✅ + bbox 疊圖 | ✅ 原文對照 | ? | ✗ | ✗ | ✗ | ✗ |
| 可續傳上傳 + 即時進度 | ✅ SSE | ◐ 佇列監控 | ✗ | ? | ◐ | ✅ 逐頁進度 | ? | ✗ | ✗ | ◐ HTTP job | ✗ |
| 多使用者 / RBAC | ✗（單一管理者） | ✅ JWT/RBAC/SSO 預留 | ✗ | ✗ | ✗ | ◐ API Token | ? | ✅ multi-user | ✗ | ✗ | ✗ |
| MCP transport | ✅ Streamable HTTP（2026-07-28） | ◐ 舊 SSE | stdio/HTTP | ◐ Ecosystem（雲端 API） | ✗ / docling-mcp stdio,SSE | ✗ | ? | ✅ Streamable HTTP | stdio | stdio | stdio |
| MCP 認證 | ✅ PAT + scopes + 撤銷 + 限流 | ◐ 靜態 key 清單 | ? | — | — | — | — | ✅ API key + **OAuth 2.1** | 目錄白名單 | — | — |
| MCP 呼叫紀錄後台 | ✅ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ◐ 歷史 | ✗ | ✗ | ✗ |
| 分頁讀取 + token 預算 | ✅ | ✗ | ◐ | ✅ 定位器 | ✗ | ✗ | ✗ | ◐ query | ✅ | ◐ | ✗ |
| 文件內／Library 搜尋 | ✅ 逐頁 FTS | ✗ | ✗ | ✅ | ◐ 名稱搜尋 | ✗ | ✗ | ✗ | ✅ 精確 + BM25 | ✗ | ✗ |
| 一行安裝 | ✗（uv + pnpm + 數 GB 模型） | ◐ setup.sh/Docker | ✅ pip | ✅ uv | ✅ pip/容器 | ◐ 一鍵腳本 | ◐ Docker | ✅ Docker | ✅ npx | ✅ 安裝腳本 | ✅ 多套件庫 |
| Docker 映像 | ✗ | ✅ | ? | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ◐ | ✅ |
| 公開 benchmark | ✗ | ✗ | ✅ | ✅ OmniDocBench | ✅ | ✗ | ◐ demo | ✗ | ✅ AgentDocBench | ✅ 小型 | ◐ |
| 授權 | AGPL-3.0 | Apache-2.0 | MIT | MinerU OSL | MIT | Apache-2.0 | Apache-2.0 | AGPL-3.0 | MIT（+Pro） | MIT | MIT |
| ★ | 0 | 832 | 82 | 80,973 | 263 / 1,845 | 133 | 28 | 485 | 1,016 | 1,152 | 9,363 |

---

## 6. 我們真正的差異化（誠實版）

1. **「多個重量級引擎 + 自動路由 + 逐頁品質 fallback」放進一個可用的 Web App。** pdfmux 有路由與 fallback 但沒 UI、只做 PDF；Tianshu 有 UI 但不做品質 fallback；PaddleOCR Local 多模型但要手動選。三者同時具備的開源專案，本次調查沒有找到。
2. **壞文字層（無 ToUnicode 字型）逐頁偵測 + OCR 修復 + 接回原位 + 修不好就標記。** 明確做這件事的只有 pdfmux（逐頁重抽）與 Marker v2（`balanced` 整頁重 OCR／`fast` 區塊修復）；在 Docling 為主引擎的文件上用同引擎 FULL_PAGE OCR 修，以維持 Markdown 風格一致，這個細節是我們獨有的。
3. **引擎各自隔離在 `uv` env、用 subprocess 呼叫。** 能同時跑 MinerU 4 + Docling 2 + MarkItDown 而不互相衝突版本（torch/onnxruntime/transformers）；多數競品不是單引擎，就是把多引擎塞在同一個 env（pdfmux 的 extras）。這是工程上的實質優勢，但使用者看不見，需要在 README 講清楚。
4. **超長文件（1000+ 頁）分段、續跑、跨段表格接合，並有實測。** Tianshu 有 500 頁拆分並行、PaddleOCR Local 有按批續跑、doc7 有失敗頁重跑，但「跨段表格接合」與實測 1192 頁書的報告沒看到別人做。
5. **遠端可用、權限完整的 MCP：** Streamable HTTP（2026-07-28 stateless）+ 每把 PAT 的 scopes（read/convert/convert:local/manage）+ 撤銷/過期 + 限流 + 後台即時看連線與每次呼叫。文件 MCP 幾乎全是 stdio 本機；有遠端認證的只有 Tianshu（舊 SSE + 靜態 key）與 PullMD（有 OAuth，但 PDF 能力弱）。
6. **繁中（台灣）優先**：README 雙語、繁中 OCR 實測。Semark 是唯一同樣以繁中為主的對手，但路線不同（LLM 語意改寫 vs 忠實轉換）。

**不算差異化的（別過度宣傳）：**
- 「分頁讀取 + token 預算 + 搜尋」：anymd、pdf-mcp、MinerU 4.0、MinerU Document Explorer、DocSlicer 都有。
- 「PDF/Markdown 左右對照」：Docling Studio（還有 bbox 疊圖）、PaddleOCR Local、MinerU Gradio 都有。
- 「RAG chunk 帶頁碼」：pdfmux、Docling、Semark、xberg 都有。
- 「Claude Code skill」：MinerU、anydoc、PullMD、anymd 都有官方 skill。

---

## 7. 對手明顯領先、值得學或整合的地方

| 領域 | 誰領先 | 具體做法 | 我們差在哪 |
|---|---|---|---|
| 轉換品質（掃描/複雜版面） | PaddleOCR-VL-1.6、MinerU2.5-Pro VLM、Chandra 2、olmOCR-2、dots.mocr | 0.9B～7B 文件 VLM，OmniDocBench/olmOCR-Bench 名列前茅 | 我們只透過 MinerU tier 間接使用 VLM；沒有獨立 VLM OCR 引擎可當最後一道 fallback |
| 品質稽核 | pdfmux | 4 訊號信心分數、`verify` 稽核任何引擎、`--strict --min-confidence` CI 模式、成本預估 | 我們的品質訊號只對自家轉換用；沒有「拿外部輸出來稽核」 |
| 解析前分流 | LiteParse、opendataloader hybrid | 便宜的「複雜度偵測」→ 只把難頁送重引擎 | 我們以文件為單位路由，頁級混用引擎只在修復時發生 |
| 遠端 MCP 認證 | PullMD | OAuth 2.1（DCR、PKCE、PRM、refresh 輪替與重放偵測），claude.ai 自訂 connector 直接可用 | 我們排在 Phase 2 |
| MCP 安裝體驗 | anymd、pdf-mcp、markitdown-mcp | `npx … setup` 自動寫入所有 client、`.mcpb` 一鍵裝 Claude Desktop、MCP Registry 上架 | 我們要先架伺服器、建 token、手貼設定 |
| MCP 安全 | pdf-mcp | 把 PDF 隱藏/注入文字標為 untrusted | 我們未處理文件內 prompt injection |
| 企業整合 | Tianshu | 多使用者 + RBAC + SSO、Webhook（HMAC、重試、死信）、稽核日誌、物件儲存 | 單一管理者、無 webhook |
| 部署 | 幾乎所有對手 | Docker/compose、GPU/CPU 映像、Windows 一鍵 | 沒有 Docker 映像；首次安裝要 uv + pnpm + 數 GB 模型 |
| 視覺化除錯 | Docling Studio | bbox 疊圖、chunk 行內編輯、一鍵灌 OpenSearch/Neo4j | 只有頁同步，無區塊層級對照 |
| 顯存管理 | PaddleOCR Local、Marker v2 | 顯存預檢推薦模型、單模型常駐自動釋放；依 GPU 容量自動分配並行 | 我們靠 subprocess 結束釋放，無預檢 |
| Benchmark / 可信度 | anymd（AgentDocBench）、opendataloader、doc7、pdfmux | 公開 benchmark，連輸的類別也列 | 沒有公開數字，0 ★，外界無從判斷品質 |
| 生態整合 | MinerU、Docling | LangChain/LlamaIndex loader、Dify/RAGFlow/Open WebUI 外掛 | 沒有任何 RAG 平台能直接接我們 |

---

## 8. 具體建議

### 8.1 引擎（依投報率排序）

1. **加一個「VLM OCR」引擎作為掃描頁與修復的最後一道 fallback。** 首選 **PaddleOCR-VL-1.6**（Apache-2.0、0.9B、中文/繁中強、有 vLLM/serving、官方 MCP 可參考）；次選 **olmOCR-2**（Apache-2.0，英文強，7B 較重）。**避開 Chandra/Marker 模型權重**：OpenRAIL-M 修改版或商用自架需授權，與我們 AGPL 自架散佈相衝。實作上沿用隔離 env + subprocess，或更好：**接「任何 OpenAI 相容 VLM 端點」**（學 doc7），使用者自帶 LM Studio/Ollama/vLLM。
2. **頁級混用引擎（不只修復時）：** 學 LiteParse/opendataloader hybrid/pdfmux，先做便宜的逐頁複雜度分類，只把難頁送 MinerU/VLM，其餘走 Docling 或 PyMuPDF。對 1000+ 頁書可大幅縮短時間。
3. **評估 opendataloader-pdf（Apache-2.0、Java）作為快速數位 PDF 引擎**、**anydoc（MIT、Rust）作為 MarkItDown 的 Office 替代/備援**。兩者都輕、快、活躍，可作為新的 fallback 層。
4. **不建議**整合：Zerox、Nougat、GOT-OCR、Open Parse、MegaParse、OmniParse（停更或已被取代）。

### 8.2 值得借用的點子

1. **`aidoc verify`**（學 pdfmux）：拿任何引擎的輸出對照原 PDF，逐頁報告漏頁/亂碼——把我們的品質評分變成獨立賣點，也能稽核 Open WebUI/RAGFlow 已經轉好的庫。
2. **一行安裝 MCP**：發一個 `npx`/`uvx` 的薄 client（或 `.mcpb`）幫使用者寫入 Claude Code/Cursor/VS Code 設定；上架 [MCP Registry](https://registry.modelcontextprotocol.io)。
3. **文件內容 untrusted 標記**（學 pdf-mcp）：`read_document` 回傳時偵測白字、極小字、頁外文字並標註，降低 prompt injection 風險。
4. **OAuth 2.1 提前**（學 PullMD）：claude.ai／ChatGPT connector 無法帶自訂 header，PAT 只能服務 CLI/IDE；PullMD 證明單人專案也能做完整 OAuth。
5. **Docker 映像**（GPU 與 CPU 兩版）+ 首頁顯存預檢（學 PaddleOCR Local）。
6. **bbox/區塊層級對照**（學 Docling Studio）：MinerU middle_json 與 Docling 都有座標，可在對照檢視點區塊跳原文。
7. **Webhook 完成通知**（學 Tianshu）：讓 n8n/Dify 等流程串接。
8. **相容 API**：提供 docling-serve `/v1/convert/file` 相容端點，Open WebUI、Kotaemon 等就能直接把 Doc4AI 當「Docling server」用，順便解掉 Open WebUI 頁碼遺失問題。

### 8.3 定位與 README 措辭

- **主打一句話（建議）：**「**不知道該用 MarkItDown、Docling 還是 MinerU？Doc4AI Studio 逐頁替你選、檢查結果、壞了自動換引擎重做——再透過 Web 與 MCP 交給你的 AI。**」把「品質保證／自動 fallback」放第一位，因為這是 MarkItDown/Docling/MinerU 本身都不做的事。
- README 加「**和其他工具的關係**」一節，誠實寫：我們不是新的解析模型，是**編排層**（orchestrator）；何時該直接用 MinerU 4.0 / docling-serve（單一引擎就夠、要 Docker）、何時用 anymd/pdf-mcp（只要讓 agent 讀 PDF、要一行安裝）、何時用我們（混雜格式的大量文件、要品質保證、要團隊共用的遠端 MCP、繁中）。這會比宣稱「最強」更可信。
- 加一張**實測表**（就算只有 10 份文件）：我們的路由結果 vs 單用各引擎，含繁中掃描、壞字型 PDF、1000+ 頁書。沒有數字的「quality fallback」很難說服人。
- 授權提醒放顯眼處：MinerU OSL 對大規模/線上服務有附加條件；若加 Marker/Chandra 模型，權重授權另有限制。

### 8.4 可能的合作／整合

- **pdfmux**：理念最接近（MIT）。可以把 pdfmux 當一個「engine」接進來（PDF 數位頁），或在其 `verify` 上貢獻繁中/CJK 訊號；反過來讓 pdfmux 把 Doc4AI 當高品質後端。
- **Semark**（台灣、Apache-2.0）：作為選用「語意後處理」步驟（表格修復、截圖描述），或聯合做繁中 benchmark。
- **Open WebUI / RAGFlow / Dify**：以「external extraction engine」或外掛形式提供 Doc4AI，取得使用者入口。
- **MinerU 團隊**：我們已用 MinerU 4.0；回報 Windows/Blackwell（CUDA 12.8）與繁中實測問題，有機會被列入其生態清單。
- **PaddleOCR**：其 `awesome_projects.md` 收錄社群專案；整合 PaddleOCR-VL 後可申請列入。

---

## 9. 風險與注意

- **MinerU 4.0 的擴張**是最大威脅：tier 品質分級 + Router + 文件庫 + 分頁定位讀取 + 搜尋 + Gradio WebUI + 官方 skill，已覆蓋我們一半的賣點，且有 8 萬星與團隊維護。我們的護城河要放在「跨引擎」與「品質保證」，不是「讀取體驗」。
- 星數落差：我們 0 ★，同類專案中小型者也有 100～1,000 ★；短期內要靠 benchmark 與清楚定位，而非功能數量。
- AGPL-3.0 對企業整合者（Tianshu、Dify 使用者）是門檻；這是取捨，不一定要改，但 README 應說明「以網路服務提供需公開修改」。

---

## 10. 來源

**A 類**
- https://github.com/magicyuan876/mineru-tianshu（README、`backend/mcp_server.py`）
- https://github.com/NameetP/pdfmux
- https://github.com/CHEN010325/paddleocr-local
- https://github.com/KingsleyOWO/Semark
- https://github.com/scub-france/docling-Studio
- https://github.com/AeternaLabsHQ/pullmd
- https://github.com/magicrew/doc7
- https://github.com/SylphxAI/anymd
- https://github.com/xberg-io/xberg
- https://github.com/ibrahimqureshae/mdflux
- https://github.com/drmingler/docling-api
- https://github.com/ZHYX91/docwen 、https://github.com/ber-ners/mineru-textbook-to-md 、https://github.com/TylerMorrison21/paperflow 、https://github.com/thomas-villani/all2md 、https://github.com/hwdsl2/docker-docling

**B 類**
- https://github.com/microsoft/markitdown（含 `packages/markitdown-mcp`、`packages/markitdown-ocr`）
- https://github.com/firecrawl/anydoc
- https://github.com/opendatalab/MinerU
- https://github.com/docling-project/docling 、https://github.com/docling-project/docling-serve
- https://github.com/datalab-to/marker 、https://github.com/datalab-to/surya 、https://github.com/datalab-to/chandra
- https://github.com/allenai/olmocr
- https://github.com/PaddlePaddle/PaddleOCR
- https://github.com/opendataloader-project/opendataloader-pdf
- https://github.com/run-llama/liteparse
- https://github.com/Unstructured-IO/unstructured
- https://github.com/pymupdf/pymupdf4llm
- https://github.com/studio-dots-ai/dots.ocr 、https://github.com/studio-dots-ai/dots.mocr
- https://github.com/deepseek-ai/DeepSeek-OCR 、https://github.com/deepseek-ai/DeepSeek-OCR-2
- https://github.com/zai-org/GLM-OCR 、https://github.com/Yuliang-Liu/MonkeyOCR 、https://github.com/bytedance/Dolphin
- https://github.com/getomni-ai/zerox 、https://github.com/facebookresearch/nougat 、https://github.com/Ucas-HaoranWei/GOT-OCR2.0
- https://github.com/Filimoa/open-parse 、https://github.com/NanoNets/docstrange 、https://github.com/The-Vibe-Company/megaparse 、https://github.com/adithya-s-k/omniparse
- https://github.com/emcf/thepipe 、https://github.com/oomol-lab/pdf-craft
- https://github.com/opendatalab/OmniDocBench

**C 類**
- https://github.com/docling-project/docling-mcp
- https://github.com/opendatalab/MinerU-Ecosystem 、https://github.com/opendatalab/MinerU-Document-Explorer
- https://github.com/PaddlePaddle/PaddleOCR/blob/main/docs/version3.x/integrations/mcp_server.en.md
- https://github.com/jztan/pdf-mcp
- https://github.com/zcaceres/markdownify-mcp
- https://github.com/chrisryugj/kordoc
- https://github.com/DocSlicer/DocSlicer
- https://github.com/FutureUnreal/mcp-pdf2md 、https://github.com/KorigamiK/markitdown_mcp_server 、https://github.com/neosun100/mineru-mcp-server 、https://github.com/zanetworker/mcp-docling

**D 類**
- https://github.com/infiniflow/ragflow 、https://ragflow.io/docs/select_pdf_parser 、https://github.com/infiniflow/ragflow/pull/14097
- https://github.com/langgenius/dify 、https://marketplace.dify.ai/plugin/langgenius/mineru
- https://github.com/open-webui/open-webui 、https://docs.openwebui.com/features/chat-conversations/rag/document-extraction/docling/ 、https://github.com/open-webui/open-webui/issues/14011
- https://github.com/Cinnamon/kotaemon
- https://github.com/onyx-dot-app/onyx
- https://github.com/Zipstack/unstract
- https://github.com/HKUDS/RAG-Anything

**其他**
- GitHub topic：https://github.com/topics/pdf-to-markdown 、https://github.com/topics/document-parsing 、https://github.com/topics/mineru
- HN（2025-09 之後 PDF→Markdown 相關 Show HN，聲量都很小，最高 10 分）：https://hn.algolia.com/?query=pdf%20to%20markdown
- 比較文章：https://jimmysong.io/blog/pdf-to-markdown-open-source-deep-dive/
