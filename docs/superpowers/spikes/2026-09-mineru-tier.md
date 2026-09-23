# Spike C: MinerU 4.x API shape and tier `standard` on sm_120 (§13.1)

Date: 2026-09-24 · Host: Windows 11, RTX 5070 Ti 16 GB (sm_120), driver 591.86 · mineru 4.0.7,
torch 2.11.0+cu128, mineru-llama-cpp 0.1.2. Scripts: `.superpowers/spike/spikec*.py` (scratch, not committed).

## API (index A3)

```
mineru.parser.parse(path: str | Path, *, tier: Tier = 'standard', ocr_mode: Literal['auto','txt','ocr'] = 'auto',
                    image_analysis: bool = True, page_range: str = '', source_context=None, vlm_config=None) -> ParseResult
ParseResult.save(self, writer: DataWriter) -> None
mineru.parser.writer.FileBasedDataWriter(parent_dir: str = '')
ParseResult.markdown(*, add_markers=False, mode=None, asset_base_url='', image_renderer=None) -> str
ParseResult.structured_content(*, asset_base_url='') -> dict
ParseResult.export_pages() -> list[PageInfo]
```

- Library default tier is `standard`; the runner always passes the configured tier (default `basic`).
- `save(FileBasedDataWriter(dir))` writes `markdown.md`, `middle_json.json`, `structured_content.json`,
  `model_output.json` and `images/` (`page_{idx}_{type}_{n}.jpg`, including table snapshots that the markdown does
  not reference — the runner only reports images referenced by the markdown).
- `structured_content.json` (content list V2) = `{"pages": [{"page_idx", "blocks": [{"type", "bbox", "content",
  "image_source"?, "level"?}]}], "metadata": {"document": {"page_count"}}, ...}`. Block text is under `content`
  (not `text`), images under `image_source` (not `img_path`), blocks under `blocks`. The runner's
  `flatten_structured` / `_needles` handle both the planned and the real key names.
- `middle_json.json` = `{"pages": [{"page_idx", "blocks": [{"type", "bbox" (normalised 0..1), "content": [...]}]}]}`;
  tables carry nested `table_body` HTML → `table_edges` reads normalised bboxes and counts `<td>/<th>` of row 1.
- In MinerU 4.0.7 tables in `markdown.md` are already GFM; formulas are `$$…$$` blocks.
- `markdown(add_markers=True)` only inserts `---` between pages — not a reliable page marker, so the runner keeps
  the plan's `insert_page_markers` mapping (spec §3: markers recovered from the per-page content list).
- Windows gotcha: MinerU renders PDF pages in a `ProcessPoolExecutor` (spawn). A script fed through stdin fails with
  `BrokenProcessPool`; the runner is a real file with an `if __name__ == "__main__":` guard, which is required.
- `MINERU_HOME` and `MINERU_MODEL_SMALL_BACKEND` are honoured (mineru/config.py). There is no language parameter.
- MinerU 4 also parses `.docx` directly; routing still follows spec §4 (office → MarkItDown → Docling).

## Fixture change

The planned `formula.pdf` only contained math *symbols* as running text, which no engine turns into LaTeX, so the
planned `test_formula` (`"$" in markdown`) could not pass. The generator now typesets four display equations
(integral with limits and a fraction, a summation, `E = mc²`, the quadratic formula) with positioned glyphs and a
drawn fraction bar; the Symbol-font line and the symbol run are kept (probe `math_hint`). MinerU basic returns
them as `$$\int\limits_{0}^{1} x^{2} dx = \frac{1}{3}$$` etc. (10 LaTeX blocks over 2 pages).

## Tier `standard` vs `basic`

Method: one warm-up parse per tier, then each fixture in the same process; recall = character-multiset overlap with
the known fixture text; GPU sampled with `nvidia-smi` every 0.2 s (desktop baseline ≈ 9.0 GB).

| tier | file | s / page | recall | LaTeX blocks | GPU util max | GPU mem max (MB) |
|---|---|---|---|---|---|---|
| basic | (warm-up) | 5.5 s | | | | |
| basic | scanned_cht.pdf | 0.26 | 0.995 | 0 | 33 % | 10127 |
| basic | scanned_mixed.pdf | 0.25 | 1.000 | 0 | 33 % | 10127 |
| basic | page.png | 0.28 | 1.000 | 0 | 1 % | 9605 |
| basic | formula.pdf | 2.04 | 1.000 | 10 | 99 % | 10323 |
| standard | (warm-up) | 16.9 s | | | | |
| standard | scanned_cht.pdf | 0.54 | 1.000 | 0 | 69 % | 12071 |
| standard | scanned_mixed.pdf | 0.61 | 1.000 | 0 | 73 % | 12073 |
| standard | page.png | 0.47 | 1.000 | 0 | 1 % | 11739 |
| standard | formula.pdf | 0.87 | 1.000 | 10 | 59 % | 12073 |

(a) Runs without error on sm_120. The bundled `mineru-llama-cpp` 0.1.2 Windows build ships `ggml-vulkan.dll` +
`ggml-cpu.dll` (no `ggml-cuda`), so the GGUF VLM runs on the GPU through **Vulkan**, not CUDA — no `ggml_cuda` init
line exists; GPU use is evidenced by +1.9 GB VRAM and 70 % utilisation vs basic, and by sub-second pages.
(b) Time per page: 1.7–2.4× basic on scanned input (≤ 3× ✓); formula.pdf is faster (basic's formula recogniser
dominates there); model warm-up is 3× longer (16.9 s vs 5.5 s).
(c) Recall: identical on 3 of 4 fixtures, 0.995 → 1.000 on scanned_cht; same LaTeX output.

**Decision: keep `basic` as the default.** The plan's rule (c) asks for higher recall, and spec §13.1 (binding)
asks for 「品質明顯較好」(clearly better quality) before changing the default. A 0.005 gain on one fixture is not
clearly better, while standard costs ~2× per page, 3× warm-up and +1.9 GB VRAM. `aidoc.toml engines.mineru_tier`
stays `"basic"`; `standard` remains a supported setting (models already downloadable with
`mineru-kit models download --tier standard`), and the OOM downgrade `standard → basic` stays.
Re-evaluate with real scanned documents in `tests/fixtures/manual/`, where the VLM's layout/reading-order
advantages would show.
