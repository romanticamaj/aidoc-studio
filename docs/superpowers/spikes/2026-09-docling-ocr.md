# Spike: Docling per-page export (A) and EasyOCR vs RapidOCR (B)

Date: 2026-09-24 · Host: Windows 11, RTX 5070 Ti 16 GB (sm_120), driver 591.86 ·
docling 2.130.0, torch 2.11.0+cu128, easyocr 1.7.2, rapidocr (docling[rapidocr]), onnxruntime-gpu 1.25.1.
Scripts: `.superpowers/spike/spikea.py`, `spikeb.py` (scratch, not committed).

## A. Per-page `export_to_markdown(page_no=n)` — duplication / loss

Method: convert with the runner's `build_converter` (EasyOCR, layout + tableformer), compare the multiset of
non-empty, non-marker lines of the concatenated per-page exports with the whole-document
`export_to_markdown()`.

| file | pages | per-page lines | whole-doc lines | extra in per-page | missing in per-page | ruled table |
|---|---|---|---|---|---|---|
| text.pdf | 3 | 11 | 11 | 0 | 0 | 1 GFM table, on page 2 only |
| big.pdf | 45 | 90 | 90 | 0 | 0 | — |

**Decision:** no duplication and no loss → keep index A2 as written (per-page export, PLACEHOLDER images replaced
by pictures saved per page). The `self_ref` dedupe fallback is not needed.

Note: `tests/integration/test_engine_docling.py::test_no_duplicate_lines_per_page` as planned compared lines
across the whole document, but `text.pdf` repeats identical paragraphs on every page by design, so it failed on
legitimate repeats. The test now checks duplicates within each page; cross-page duplication is ruled out by
the table above.

## B. EasyOCR vs RapidOCR (§13.5)

Method: `DoclingEngine.convert` on the three scanned fixtures with `docling_ocr="easyocr"` and `"rapidocr"`
(`full_page_ocr` on, as for scanned input). Recall = character-multiset overlap with the known fixture text
(ZH/EN strings from `make_fixtures.py`). GPU sampled with `nvidia-smi` every 0.2 s (baseline 9.0 GB used by
other desktop apps). Wall time includes runner start + model load (≈ 7 s of each run).

| OCR | file | wall s | recall | GPU util max | GPU mem max (MB) |
|---|---|---|---|---|---|
| easyocr | scanned_cht.pdf | 9.7 | 0.918 | 69 % | 14766 |
| easyocr | scanned_mixed.pdf | 9.5 | 0.975 | 42 % | 14772 |
| easyocr | page.png | 8.5 | 0.975 | 33 % | 14765 |
| rapidocr (CPU ORT, first try) | scanned_cht.pdf | 9.2 | 1.000 | 15 % | 9987 |
| rapidocr (CPU ORT, first try) | scanned_mixed.pdf | 8.4 | 1.000 | 15 % | 9994 |
| rapidocr (CPU ORT, first try) | page.png | 7.5 | 1.000 | 5 % | 9987 |
| rapidocr (onnxruntime-gpu 1.25.1) | scanned_cht.pdf | 51.5* | 1.000 | 57 % | 11656 |
| rapidocr (onnxruntime-gpu 1.25.1) | scanned_mixed.pdf | 10.9 | 1.000 | 22 % | 11657 |
| rapidocr (onnxruntime-gpu 1.25.1) | page.png | 8.8 | 1.000 | 7 % | 10881 |

\* first CUDA session on a fresh ORT install (kernel/cudnn cache warm-up); 21.8 s with 1.23.2, later runs ≈ 9–11 s.

EasyOCR did use the GPU on Windows (docling issue #2727 did not reproduce with torch cu128).

**Decision rule:** RapidOCR if its recall ≥ EasyOCR recall − 0.02 **or** EasyOCR did not use the GPU.
RapidOCR recall is 1.000 on every fixture vs EasyOCR 0.918–0.975 → **RapidOCR becomes the default**:
`aidoc.toml engines.docling_ocr = "rapidocr"`, `ConvertOptions.docling_ocr` / `Engines.docling_ocr` default
`"rapidocr"`, and `docling[rapidocr]` + `onnxruntime-gpu` moved into the env's main dependencies.
EasyOCR stays installed and downloaded; `docling_ocr = "easyocr"` switches back without re-running setup.

### Packaging findings (applied to `envs/docling/pyproject.toml`)

1. `docling[rapidocr]` depends on the CPU `onnxruntime`, which installs into the same `onnxruntime/` package as
   `onnxruntime-gpu` and wins → RapidOCR logged "CUDAExecutionProvider is available, but ... available lists are
   ['CPUExecutionProvider']". Fixed with `[tool.uv] override-dependencies = ["onnxruntime; sys_platform == 'never'"]`.
   (When switching an existing env, reinstall: `uv sync --project envs/docling --reinstall-package onnxruntime-gpu`,
   because uninstalling `onnxruntime` deletes files shared with `onnxruntime-gpu`.)
2. `onnxruntime-gpu` 1.30.0 (latest) is built for CUDA 13 (`cublas64_13.dll`, `cudart64_13.dll`), but torch here
   is cu128 and supplies CUDA 12 DLLs. 1.23.2 and 1.25.1 both run on the torch-loaded CUDA 12 runtime → pinned
   `onnxruntime-gpu>=1.23,<1.26`. Moving the env to cu130 (envs/README.md) should lift this pin.
3. RapidOCR models: `docling-tools models download rapidocr --rapidocr-backend-lang onnxruntime:chinese_cht
   --rapidocr-backend-lang onnxruntime:en` fetches PP-OCRv6 det/rec small + cls; `chinese_cht` resolves to the
   PP-OCRv6 `ch` recogniser (covers 繁中/簡中/English). `aidoc setup docling` now downloads both OCR model sets.
