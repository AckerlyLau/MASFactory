# NowWhat

This directory contains the MASFactory-based **NowWhat** daily AI paper digest workflow. It ingests a local directory of PDFs, extracts previews and text, summarizes papers in batches, scores them along novelty / rigor / impact axes, and writes a ranked daily brief in JSON and Markdown.

## Experience

- Hosted experience: `https://what.masfactory.dev`

## Overview

**NowWhat** stands for "Now what are my peers working on?" It is an information-filtering app for AI researchers and developers. The product turns a stream of recent AI papers into a readable daily briefing, helping users quickly see which papers are worth attention and why.

## Workflow Design

During development, we built a full multi-agent review workflow with MASFactory. The system first reads and distills each paper, then routes it through three reviewer groups focused on **Novelty**, **Rigor**, and **Impact**. Their results are aggregated into a ranked shortlist and a structured daily digest.

<p align="center">
  <img src="assets/nowwhat-workflow.png" alt="NowWhat workflow in MASFactory" width="920" />
</p>

## Product Preview

The hosted interface presents the generated report as a browsable daily brief, with at-a-glance takeaways on the left and highlighted papers on the right.

<p align="center">
  <img src="assets/nowwhat-preview-en.png" alt="NowWhat product preview" width="980" />
</p>

## Layout

```text
applications/nowwhat/
├── tools/
│   ├── llm_tools.py                  # OpenAI-compatible helpers
│   └── pdf_tools.py                  # PDF extraction / rendering tools
└── workflows/
    └── daily_digest/
        ├── main.py                   # Entry point
        ├── prompts.py                # Agent prompts
        ├── tools.py                  # Batch preparation / persistence helpers
        └── workflow.py               # RootGraph definition
```

## Setup

Run dependency installation from the repo root:

```bash
uv sync
```

Environment variables commonly used by this app:

```bash
export OPENAI_API_KEY="..."
export BASE_URL="https://api.openai.com/v1"
export MODEL_NAME="gpt-5.2"
```

Required system binaries:

- `pdfinfo`
- `pdftotext`
- `pdftoppm`
- `pdftocairo`
- `pdfimages`

These commands are typically provided by Poppler.

## Run

Commands below assume the working directory is `applications/nowwhat/`.

```bash
cd applications/nowwhat

uv run python workflows/daily_digest/main.py \
  --pdf-dir /path/to/paper_pdfs \
  --output-dir ./runs \
  --run-name 2026-03-23_digest \
  --date-label 2026-03-23 \
  --model "${MODEL_NAME:-gpt-5.2}" \
  --api-key "${OPENAI_API_KEY}" \
  --base-url "${BASE_URL:-https://api.openai.com/v1}"
```

Useful local-testing flags:

- `--max-papers 10`
- `--summary-batch-size 2`
- `--scoring-batch-size 4`

## Outputs

Each run creates `runs/<run_name>/` with the following structure:

```text
runs/<run_name>/
├── 00_sources/                      # paper inventory JSON
├── 01_extracted/                    # extracted text and first-page previews
├── 02_summaries/                    # summary JSON
├── 03_scores/                       # novelty / rigor / impact / aggregate scores
├── 04_brief/                        # final daily_brief.json + daily_brief.md
└── 05_tool_assets/                  # tool-generated images and extracted assets
```

The entrypoint prints a compact JSON summary including:

- `run_dir`
- `paper_inventory_path`
- `paper_summaries_path`
- `aggregated_scores_path`
- `daily_brief_json_path`
- `daily_brief_markdown_path`

## Notes

- This application is designed around **local PDF corpora**. The hosted demo may include extra service glue, but the workflow here expects an input directory of PDFs.
- The workflow exposes PDF inspection tools to agents, so richer models generally produce better briefings on figure-heavy papers.
