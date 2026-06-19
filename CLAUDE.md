# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is the **AE (Application Engineer) Knowledge Documentation Repository** for an AE team covering analog/mixed-signal chip products. No source code, build system, or test framework — purely documentation in two formats:

| Format | Location | Chapters | Features |
|--------|----------|----------|----------|
| **Markdown** (source) | `docs/` | 6 chapters | ASCII diagrams, tables |
| **HTML** (generated) | `html/` | 9+1 chapters | SVG diagrams, sticky nav, color-coded domains |

## Product Lines

Five product families, each with its own directory under `html/` (and most under `docs/`):

| # | Product Line | Theme Color | Products |
|---|-------------|-------------|----------|
| 01 | 光通信 (Optical) | Blue `#1a56db` | oDSP, TIA/Driver, Retimer, SerDes |
| 02 | 无线通信 (Wireless) | Red `#e74c3c` | 基站/卫星/WiFi 射频收发 |
| 03 | 工业汽车 (Industrial) | Green `#27ae60` | 储能/车规 BMS AFE |
| 04 | ADC | Green `#27ae60` | SAR, Sigma-Delta, Pipeline, Time-Interleaved |
| 05 | POWER (Power Mgmt) | Green `#27ae60` | BUCK, BOOST, BUCK-BOOST converters |

- `04_ADC/` and `05_POWER/` exist only under `html/` — they have no `docs/` markdown counterparts. The `docs/README.md` lists only 3 product lines (光通信/无线通信/工业汽车).
- `05_POWER` documents are named by converter topology (e.g., `BUCK_Converter.html`), not the `{number}_{description}.html` convention used in other families.

## HTML Document System (`html/`)

### Generation via Skill
Use the **`ae-knowledge-doc` skill** (`.claude/skills/ae-knowledge-doc.md`) to generate or regenerate HTML product documents. Invoke it when the user asks to "introduce/explain a chip", generate "AE training material", or update existing HTML docs.

The skill produces self-contained HTML files with:
- Inline CSS using a 5-color theme system (see table above)
- Embedded SVG diagrams (no external images)
- Sticky navigation bar
- 10 standard sections (`#sec1`–`#sec9` + `#sec2b`)

### 9+1 Chapter Structure (HTML)
1. **背景与定位** (`#sec1`) — positioning, applications, technology roadmap table
2. **工作原理** (`#sec2`) — system-level signal flow SVG + processing steps + formulas
3. **🔬 核心模块深度解析** (`#sec2b`) — per-submodule deep dive, amber/gold header
4. **电路框图** (`#sec3`) — complete internal architecture SVG with color-coded domains
5. **核心对比** (`#sec4`) — cross-architecture/generational comparison tables
6. **核心指标** (`#sec5`) — spec tables + metric-card deep-dives for key parameters
7. **影响因素** (`#sec6`) — factor network table with colored chip tags
8. **测试方法** (`#sec7`) — test principles, equipment (with models), SVG setup diagrams, pass/fail criteria
9. **注意事项** (`#sec8`) — caution-grid warning cards + danger-box TOP 5 errors
10. **选型指南** (`#sec9`) — compare-grid decision guide + scenario selection table

### SVG Requirements
- Every HTML doc must have at minimum 3–4 embedded SVGs: Section 2 signal flow, Section 3 architecture, Section 7 test setup(s)
- viewBox width fixed at 820, height per content
- Domain coloring: analog=`#fee2e2`/`#e74c3c`, digital=`#dbeafe`/`#1a56db`, clock=`#fef3c7`/`#f59e0b`, power=`#fef2f2`/`#fca5a5`
- Modules: rounded rects (`rx="6"`), bold titles at `font-size="10"`, params at `font-size="7-8"`
- **CRITICAL**: SVG `<text>` elements do NOT support HTML tags (`<sub>`, `<sup>`, `<strong>`). Use plain text (e.g., `VIN` not `V<sub>IN</sub>`, `2^N` not `2<sup>N</sup>`).
- Define arrow markers in `<defs><marker>` and reference via `marker-end="url(#id)"`

### Index Page
`html/index.html` is the entry point with product cards linking to all HTML docs. Update it when adding new HTML documents.

## Markdown Document System (`docs/`)

### 6-Chapter Structure (Markdown)
1. **一、产品概述** — positioning, applications, technology roadmap
2. **二、工作原理** — system-level signal chain, processing flow
3. **三、芯片框图** — ASCII box-drawing diagrams (`┌─┐│└─┘├┤┬┴┼`), no images
4. **四、模块详解** — per-submodule principles and key parameters
5. **五、关键指标** — spec tables with physical meaning explanations
6. **六、测试方法** — know-how: principles, factors, equipment, steps

Optional **七、AE知识体系** section with formula references and troubleshooting.

### Content Conventions
- **Language**: Simplified Chinese (zh-CN). English for standard acronyms (PAM4, BER, EVM).
- **Acronyms**: Expand on first use, e.g., "oDSP (Optical Digital Signal Processor)".
- **Process nodes**: Always specify (e.g., "7nm FinFET CMOS").
- **Uncertain specs**: Mark with `[待确认]`.
- **Testing equipment**: Reference real models from `docs/附录/测试设备与通用方法.md`.

## Adding/Updating Content

### New Product Document (Markdown)
1. Create `.md` in appropriate `docs/` product line folder
2. Follow 6-chapter template and content conventions above
3. Add row to `docs/README.md` table + update product tree

### New/Updated HTML Document
1. Invoke the `ae-knowledge-doc` skill with the product specification
2. Place output in `html/<product-line>/` matching the `docs/` structure naming
3. For `04_ADC` (HTML-only), follow the same 9+1 chapter structure and green theme
4. Update `html/index.html` with a new product card

### Cross-Reference
- `docs/README.md` links both `.md` and `.html` versions
- `html/index.html` links all HTML docs grouped by product line

## Project Configuration (`.claude/`)

| File | Purpose |
|------|---------|
| `settings.json` | Project config: language=`zh-CN`, doc root=`docs/` |
| `settings.local.json` | Gitignored personal overrides (model preferences, etc.) |
| `rules/ae-docs.md` | Content standards: accuracy-first, EE-undergrad audience, naming conventions |
| `skills/ae-knowledge-doc.md` | HTML doc generation skill definition (trigger + template) |
| `agents/` | Sub-agent definitions (doc-review, test-engineer, chip-expert skeletons) |

### Personal Workspace
`CLAUDE.local.md` (gitignored) is a per-user scratchpad for shortcuts, TODO lists, and personal notes. It supplements but never replaces `CLAUDE.md`.

## Document Naming & Versioning

From `.claude/rules/ae-docs.md`:
- **Markdown filenames**: `{domain}_{function}.md` — e.g., `01_400G_800G_oDSP.md`, `02_车规BMS_AFE.md`
- **Version annotations**: `v0.1-draft` (creation) → `v1.0` (reviewed) → increment on major revision
- **Uncertain specs**: Always mark `[待确认]` until verified against real datasheets
- **HTML footer**: Every HTML doc must display version (currently `v2.0`) and generation date

## Repository Rules

See `.claude/rules/ae-docs.md` for content principles (accuracy-first, appropriate depth, actionable testing, EE-undergrad audience level). These apply to both Markdown and HTML content generation.
