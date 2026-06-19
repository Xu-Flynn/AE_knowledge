# Agents 目录

存放 Claude Code 的**子智能体 (Sub-Agent)** 定义。

## 命名规范
- 每个 Agent 一个 `.md` 文件
- 文件名描述 Agent 角色

## Agent 内容格式
```markdown
---
name: agent-name
description: Agent 角色描述
tools: read, edit, bash  # 可选：限定可用工具
---

# Agent 定义

详细的系统提示词、行为规范...
```

## 示例用途
- `doc-reviewer.md` — 文档审查 Agent，检查技术准确性
- `test-engineer.md` — 测试工程师 Agent，生成测试方案
- `chip-expert.md` — 芯片专家 Agent，解答技术问题
