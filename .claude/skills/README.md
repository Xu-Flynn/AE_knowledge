# Skills 目录

存放 Claude Code 可调用的**可复用技能脚本**。

## 命名规范
- 每个 Skill 一个 `.md` 文件（如 `test-report.md`）
- 文件名用英文小写 + 连字符 (kebab-case)

## Skill 内容格式
```markdown
---
name: skill-name
description: 技能描述
---

# 技能标题

技能的详细指令、步骤、模板...
```

## 已有 Skills

| Skill | 文件 | 用途 |
|-------|------|------|
| AE 知识文档生成器 | `ae-knowledge-doc.md` | 为特定芯片/技术领域生成结构化的 AE 理论知识 HTML 文档（9+1 章节，SVG 框图） |
| 项目初始化 | `claude-project-init.md` | 一键初始化 Claude Code 标准项目结构 |

## 示例用途
- `generate-datasheet.md` — 从指标表生成芯片规格书
- `test-report.md` — 生成标准化测试报告
- `troubleshoot-ber.md` — BER 问题排查流程
