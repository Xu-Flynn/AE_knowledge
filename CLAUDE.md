# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

**AE (Application Engineer) Knowledge Documentation** for analog/mixed-signal chips. No source code, build system, or test framework — purely documentation:

| Format | Location | Chapters | Features |
|--------|----------|----------|----------|
| **Markdown** (source) | `docs/` | 6 | ASCII diagrams, tables |
| **HTML** (generated) | `html/` | 10 (9+1) | SVG diagrams, sticky nav, color-coded domains |

## Product Lines

Five product families (theme colors drive CSS/SVG domain coloring):

| # | Product Line | Color | Key Products |
|---|-------------|-------|----------|
| 01 | 光通信 (Optical) | Blue `#1a56db` | oDSP, TIA/Driver, Retimer, SerDes |
| 02 | 无线通信 (Wireless) | Red `#e74c3c` | 基站/卫星/WiFi 射频收发 |
| 03 | 工业汽车 (Industrial) | Green `#27ae60` | 储能/车规 BMS AFE |
| 04 | ADC | Green `#27ae60` | SAR, Sigma-Delta, Pipeline, Time-Interleaved |
| 05 | POWER | Green `#27ae60` | BUCK, BOOST, BUCK-BOOST converters |

- `04_ADC/` and `05_POWER/` are **HTML-only** (no `docs/` counterpart).
- `05_POWER` uses topology names (`BUCK_Converter.html`), not `{number}_{description}.html`.

## Document Systems

### HTML (`html/`)
Generate via the **`ae-knowledge-doc` skill** (`.claude/skills/ae-knowledge-doc.md`). Self-contained files with inline CSS, embedded SVG, sticky nav. Standard sections: 背景与定位→工作原理→🔬核心模块深度解析→电路框图→核心对比→核心指标→影响因素→测试方法→注意事项→选型指南 (`#sec1`–`#sec9` + `#sec2b`).

**Critical SVG rule**: `<text>` elements do NOT support HTML tags. Use plain text (`VIN`, `2^N`), never `<sub>`/`<sup>`. Define arrow markers in `<defs><marker>`.

`html/index.html` is the entry point — update it when adding/removing HTML docs.

### Markdown (`docs/`)
Standard 6-chapter structure: 产品概述→工作原理→芯片框图(ASCII)→模块详解→关键指标→测试方法. Optional 七、AE知识体系 appendix.

## Content Conventions

- **Language**: Simplified Chinese (zh-CN); English for standard acronyms (PAM4, BER, EVM)
- **Acronyms**: Expand on first use — "oDSP (Optical Digital Signal Processor)"
- **Process nodes**: Always specify (e.g., "7nm FinFET CMOS")
- **Uncertain specs**: Mark with `[待确认]`
- **Testing equipment**: Reference real models from `docs/附录/测试设备与通用方法.md`
- **Accuracy-first**: EE-undergrad audience level, actionable testing guidance (see `.claude/rules/ae-docs.md`)

## Adding Content

| Task | Action |
|------|--------|
| New Markdown doc | Create `docs/<product-line>/{number}_{description}.md`, follow 6-chapter template, update `docs/README.md` |
| New/Update HTML doc | Invoke `ae-knowledge-doc` skill → output to `html/<product-line>/`, update `html/index.html` |

## Naming & Versioning

- **Markdown files**: `{number}_{description}.md` (e.g., `01_400G_800G_oDSP.md`)
- **HTML footprint**: `v2.0` + date; version increments on major revision
- **Draft mark**: `v0.1-draft` → `v1.0` (reviewed)

## Configuration

`.claude/settings.json` sets `language=zh-CN`, doc root. `CLAUDE.local.md` (gitignored) for personal notes — supplements but never replaces this file.
