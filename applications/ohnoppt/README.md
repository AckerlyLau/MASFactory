# OhNoPPT

This directory contains the MASFactory-based **OhNoPPT** paper-to-deck pipeline. It prepares source assets from paper files, generates a slide outline, renders an HTML deck and validation PDF, and then reconstructs an editable PPTX with review loops between stages.

## Experience

- Hosted experience: `https://ppt.masfactory.dev`

## Overview

**OhNoPPT** is a MASFactory-based **Paper2PPT** application for turning research papers into presentation decks. Users upload a paper, describe the audience and reporting goal, and the system runs a multi-agent pipeline that converts the source material into an editable `.pptx` file instead of a static slide export.

At the end of each major stage, users can inspect the current result and request revisions, which makes the generation process much more controllable for real research talks, project demos, and group meetings.

## Workflow Design

The workflow breaks AI PPT generation into three stages: outline generation, HTML slide generation, and PPTX reconstruction. Each stage combines agent drafting, programmatic validation, and human-in-the-loop review, and the pipeline iterates until it can deliver an editable deck.

<p align="center">
  <img src="assets/ohnoppt-workflow.png" alt="OhNoPPT workflow in MASFactory" width="920" />
</p>

## Product Preview

The web app lets users review intermediate slides, compare revisions, and approve the final editable presentation directly in the browser.

<p align="center">
  <img src="assets/ohnoppt-preview-en.png" alt="OhNoPPT product preview" width="980" />
</p>

## Layout

```text
applications/ohnoppt/
├── tools/
│   └── external_pipeline_tools.py   # PDF / Office / Playwright helpers
└── workflows/
    ├── agents/                      # prompt markdown files
    ├── common/                      # shared prompt / asset helpers
    ├── outline_stage/               # requirements -> outline loop
    ├── html_stage/                  # outline -> HTML/PDF loop
    ├── pptx_stage/                  # PDF -> python-pptx -> PPTX loop
    ├── main.py                      # Entry point
    └── workflow.py                  # RootGraph definition
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
export MODEL_NAME="gpt-4o-mini"
```

Python-side packages used by this app include `Pillow` and `python-pptx`.

Required system binaries:

- Poppler tools: `pdfinfo`, `pdftotext`, `pdftoppm`, `pdfimages`
- `pdfunite`
- `soffice` (LibreOffice)
- `node`
- `npm`
- `npx`

Notes:

- Playwright Chromium is installed on first use into a local `playwright_env/` directory.
- `tectonic` is also supported by the shared external tools module, but it is not required for the default paper-to-PPT workflow path.

## Run

Commands below assume the working directory is `applications/ohnoppt/`.

```bash
cd applications/ohnoppt

uv run python workflows/main.py \
  --paper /path/to/paper.pdf \
  --user-requirements "Create a 12-slide research-group presentation for this paper." \
  --audience "Chinese research group meeting" \
  --page-budget 12 \
  --presentation-goal "Research paper group meeting presentation" \
  --style-brief "Clear, structured, suitable for technical presentations" \
  --model "${MODEL_NAME:-gpt-4o-mini}" \
  --api-key "${OPENAI_API_KEY}" \
  --base-url "${BASE_URL:-https://api.openai.com/v1}"
```

You can pass multiple source files by repeating `--paper`.

## Outputs

Each run creates `runs_text/<run_name>/` with the following structure:

```text
runs_text/<run_name>/
├── assets/                          # page previews and extracted images
├── 00_sources/                      # normalized source text / asset manifests
├── 01_outline/                      # outline.md, source_map.md, outline_review.md
├── 02_html/                         # slides.html, structure_notes.md, html_review.md
├── 03_pdf/                          # slides.pdf, preview/, export_report.md
└── 04_pptx/                         # recreate_pdf_to_pptx.py, recreated_from_pdf.pptx, recreated_from_pdf.pdf
```

The entrypoint prints a compact JSON summary including:

- `run_dir`
- `outline_path`
- `html_path`
- `rendered_pdf_path`
- `recreated_pptx_path`
- `recreated_validation_pdf_path`
- `ppt_pipeline_success`

## Notes

- This workflow contains `HumanChatVisual` review nodes in the outline, HTML/PDF, and PPTX stages. The hosted product at `ppt.masfactory.dev` provides the full interactive review experience; local runs are best paired with MASFactory Visualizer or another runtime capable of handling human-in-the-loop nodes.
- HTML-to-PDF export supports both `slide_screenshots_to_pdf` and `print_pdf`; the default is `slide_screenshots_to_pdf`.
