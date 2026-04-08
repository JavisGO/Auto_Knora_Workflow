# Auto Knora Workflow Generator

## 项目说明

本项目是一个基于 Claude Code Skill 的 Knora 工作流生成器。用户通过交互式问答描述需求，自动生成可导入 Knora 平台的工作流 JSON 文件。

## 使用方式

### 启动工作流生成

在 Claude Code 中输入：
```
/knora-workflow
```

或者直接描述你的需求，Claude 会引导你完成整个工作流构建过程。

### 交互式问答流程

1. **理解意图** — 描述工作流目标和输入输出类型
2. **确定节点** — 推荐节点组合（LLM / KNOWLEDGE / CODE / FILTER / TOOL / AGG）
3. **资源配置** — 配置知识库、工具、模型
4. **生成输出** — 生成 JSON 并输出占位符清单

## 文件结构

```
Auto_Knora_Workflow/
├── .claude/
│   └── skills/
│       └── knora-workflow.md      # 核心 Skill（交互式问答 + 生成）
├── schema/
│   ├── workflow-template.json     # 工作流顶层模板
│   └── node-templates/            # 6 种节点模板
│       ├── llm.json
│       ├── knowledge.json
│       ├── code.json
│       ├── filter.json
│       ├── tool.json
│       └── agg.json
├── examples/                      # 示例工作流
│   ├── simple-rag-qa.json         # 简单 RAG 问答
│   └── intent-routing.json        # 意图路由
├── output/                         # 生成输出目录
├── reference/                     # 参考资料（只读）
│   ├── knora4.0-docs/
│   └── workflow_output/
├── CLAUDE.md
└── README.md
```

## 节点类型说明

| 节点类型 | 用途 |
|---------|------|
| LLM | 大语言模型调用 |
| KNOWLEDGE | 知识库检索 |
| CODE | Python 代码执行 |
| FILTER | 条件分支过滤 |
| TOOL | 外部 API/工具调用 |
| AGG | 多分支结果聚合 |

## 占位符说明

生成的 JSON 中可能包含以下占位符，需在 Knora 平台上手动替换：

| 占位符 | 说明 |
|-------|------|
| `{{knowledge_id_描述}}` | 知识库 ID |
| `{{knowledge_name_描述}}` | 知识库名称 |
| `{{tool_id_描述}}` | 工具 ID |
| `{{tool_url_描述}}` | API URL |
| `{{tool_token_描述}}` | 认证 Token |
