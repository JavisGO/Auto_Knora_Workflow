# Auto Knora Workflow Generator

> 基于 Claude Code Skill 的 Knora 工作流自动生成工具

## 功能特性

- **交互式问答**：通过引导式问答收集需求，无需记忆复杂 Schema
- **6 种节点支持**：LLM、KNOWLEDGE、CODE、FILTER、TOOL、AGG
- **占位符机制**：支持无资源列表时生成占位符，后续平台替换
- **端到端生成**：直接输出可导入 Knora 平台的 JSON 文件

## 快速开始

### 1. 启动生成

在 Claude Code 中执行：
```
/knora-workflow
```

### 2. 描述需求

按照引导描述：
- 工作流名称和目标
- 输入输出类型
- 需要的节点类型
- 可用的知识库/工具列表（如有）

### 3. 替换占位符

生成 JSON 后，替换所有 `{{placeholder}}` 为实际值

### 4. 导入 Knora

将生成的 JSON 文件导入 Knora 平台

## 项目结构

```
.
├── .claude/skills/          # Claude Code Skill
├── schema/                  # Schema 模板
│   ├── workflow-template.json
│   └── node-templates/      # 节点模板
├── examples/                # 示例工作流
├── output/                  # 生成输出
└── reference/               # 参考资料
```

## 示例工作流

- `examples/simple-rag-qa.json` — 简单 RAG 问答（INPUT → KNOWLEDGE → LLM → OUTPUT）
- `examples/intent-routing.json` — 意图路由（INPUT → LLM → FILTER → [分支] → AGG → LLM → OUTPUT）

## 节点类型

| 类型 | 说明 |
|------|------|
| LLM | 调用大语言模型 |
| KNOWLEDGE | 知识库向量检索 |
| CODE | 执行 Python 代码 |
| FILTER | 条件分支判断 |
| TOOL | HTTP API 调用 |
| AGG | 聚合多分支结果 |
