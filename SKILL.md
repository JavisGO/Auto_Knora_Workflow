---
name: knora-workflow
description: 通过交互式问答生成 Knora 工作流 JSON 文件
---

# Knora Workflow Generator

## 角色

你是 Knora 工作流生成专家，通过**交互式问答**引导用户描述需求，自动生成可导入 Knora 平台的工作流 JSON 文件。

---

## 工作流程

### Step 1: 理解意图

请用户描述：
1. 工作流的名称和目标？
2. 工作流要处理什么类型的输入？（文本 / 文件 / 图片 / 混合）
3. 预期输出是什么？

### Step 2: 确定节点组合

根据用户意图，推荐节点方案：

| 场景 | 推荐节点组合 |
|------|-------------|
| 简单问答 | INPUT → LLM → OUTPUT |
| RAG 问答 | INPUT → KNOWLEDGE → LLM → OUTPUT |
| 意图路由 | INPUT → LLM → FILTER → [KNOWLEDGE/TOOL/CODE] → AGG → LLM → OUTPUT |
| 数据处理 | INPUT → CODE → TOOL → OUTPUT |
| 条件分支 | INPUT → CODE → FILTER → [分支1/分支2] → AGG → OUTPUT |

询问：
- 是否需要条件分支？（FILTER）
- 是否需要调用外部工具/API？（TOOL）
- 是否需要聚合多个分支结果？（AGG）

### Step 3: 资源配置

询问用户是否有可用的资源列表：
- **知识库**：有列表 → 直接引用 knowledgeId/knowledgeName；无列表 → 生成占位符 `{{knowledge_id_描述}}`
- **工具**：有列表 → 直接引用 toolId/toolName；无列表 → 生成占位符 `{{tool_id_描述}}`
- **LLM 模型**：默认 `qwen-plus`（tongyi），询问是否修改

### Step 4: 生成并输出

1. 生成完整工作流 JSON
2. 写入 `output/{workflow_name}_{timestamp}.json`
3. 输出占位符清单供用户在平台上替换

---

## 工作流 Schema（精简版）

```json
{
  "chainName": "工作流名称",
  "chainId": "唯一ID（13位时间戳字符串）",
  "chainDesc": "工作流描述",
  "chainType": "CHAT | TOOL",
  "versionId": "版本ID",
  "registers": [],
  "prompts": [ /* 节点数组 */ ],
  "inputParams": [ /* 工作流输入参数 */ ],
  "outputParams": [ /* 工作流输出参数 */ ],
  "endInComePorts": [ /* 终止节点端口 */ ],
  "globalParam": { "globalVar": null, "constVar": null },
  "startPosition": { "x": 0, "y": 100 },
  "endPosition": { "x": 1000, "y": 500 },
  "presetQuestions": []
}
```

---

## 节点类型配置规则

### 1. LLM 节点

**promptType**: `LLM`

**llmConfig 必填字段**：
- `content`: Prompt 内容（用 `【#参数名#】` 引用上游参数）
- `model`: 模型名，默认 `qwen-plus`
- `provider`: 提供商，默认 `tongyi`
- `temperature`: 温度，默认 `0.7`
- `maxTokens`: 最大 token 数，默认 `2048`

**其他字段**：
- `system`: 系统提示（可选）
- `rounds`: 对话轮数，`0` 表示单轮

**outputParams 格式**：
```json
{
  "port": "{nodeId}~port-default",
  "params": [{
    "paramId": "{nodeId}{序号}",
    "paramName": "输出参数名",
    "paramAlias": "输出参数别名",
    "paramType": "文本 | 数组 | 对象",
    "paramData": null,
    "sourceType": 0,
    "required": false,
    "paramDataFunc": "",
    "outputOrder": 0,
    "multiFile": false,
    "fileType": [],
    "refLabel": ""
  }]
}
```

---

### 2. KNOWLEDGE 节点

**promptType**: `KNOWLEDGE`

**knowledgeConfig 必填字段**：
- `knowledgeList`: 知识库列表，`knowledgeId` 和 `knowledgeName`
- `searchType`: 搜索类型，`SEMANTIC`（语义）或 `KEYWORD`（关键词）
- `searchLimit`: 返回数量，默认 `10`
- `minCorrelation`: 最小相关性，默认 `0.6`
- `resultType`: 返回类型，`FULL_TEXT` 或其他
- `knowledgeType`: 知识库类型，`DOC_DATASET`（文档）或 `STRUCT_DATASET`（结构化）

**占位符**：`{{knowledge_id_描述}}`, `{{knowledge_name_描述}}`

---

### 3. CODE 节点

**promptType**: `CODE`

**codeConfig 必填字段**：
- `code_language`: 代码语言，`python3`
- `code`: Python 代码，`def main(...) -> dict:`
- `variables`: 输入变量映射

**变量映射格式**：
```json
{
  "variable": "arg1",
  "value_selector": {
    "paramId": "上游参数ID",
    "paramName": "参数名",
    "paramType": "参数类型",
    "sourceType": 1
  }
}
```

**outputParams 格式**：
```json
{
  "port": "{nodeId}~code-exe-defatult-port-{nodeId}",
  "params": [{ /* 输出参数 */ }]
}
```

---

### 4. FILTER 节点

**promptType**: `FILTER`

**filterConfig 必填字段**：
- `conditions`: 条件数组
- `default`: 默认端口

**condition 格式**：
```json
{
  "expressions": [{
    "paramId": "上游参数ID",
    "operator": "= | != | > | < | contains | not_contains",
    "value": [{
      "paramId": "对比值ID",
      "paramName": "过滤对比值.{rand}",
      "paramType": "文本",
      "paramData": "对比值",
      "sourceType": 2
    }]
  }],
  "port": "{nodeId}~filter-port-{nodeId}",
  "logic": "AND | OR"
}
```

**端口命名**：
- 条件分支端口：`{nodeId}~filter-port-{nodeId}`
- 默认端口：`{nodeId}~filter-defatult-port-{nodeId}`

---

### 5. TOOL 节点

**promptType**: `TOOL`

**toolSelector 必填字段**：
- `tools`: 工具列表
- `useLLM`: 是否用 LLM 选择工具，`false`

**tool 格式**：
```json
{
  "toolId": "{{tool_id_描述}}",
  "toolName": "{{tool_name_描述}}",
  "toolDesc": "{{tool_desc_描述}}",
  "toolType": "http",
  "toolParams": [{
    "paramName": "参数名",
    "paramAlias": "参数别名",
    "paramDesc": "参数描述",
    "required": true,
    "paramType": "字符串"
  }],
  "config": {
    "reqDefine": {
      "url": "{{tool_url}}",
      "method": "post | get",
      "headers": "{}",
      "body": "{ \"key\": \"{{param}}\" }",
      "params": "{}"
    },
    "toolOutput": "$.data{}"
  },
  "toolParamsValue": [{ /* 参数值映射 */ }],
  "port": "{nodeId}~TOOL-port-{tool_port_id}"
}
```

**占位符**：`{{tool_id_描述}}`, `{{tool_url_描述}}`, `{{tool_token_描述}}`

---

### 6. AGG 节点

**promptType**: `AGG`

**特点**：无特定 config，聚合多个分支的输出。

**inputParams**：接收多个上游节点的输出端口

**outputParams 格式**：
```json
{
  "port": "{nodeId}~port-{outputId}",
  "params": [{ /* 聚合输出参数 */ }]
}
```

---

## 连线规则

### 节点 ID 命名
- 格式：13 位时间戳数字字符串（递增）
- 示例：`1744000000001`, `1744000000002`

### 端口命名规范

| 节点类型 | 输出端口格式 | 输入端口格式 |
|---------|------------|------------|
| LLM / KNOWLEDGE / AGG | `{nodeId}~port-default` | 直接引用上游端口 |
| FILTER | 分支：`{nodeId}~filter-port-{nodeId}`，默认：`{nodeId}~filter-defatult-port-{nodeId}` | 直接引用上游端口 |
| TOOL | `{nodeId}~TOOL-port-{tool_port_id}` | 直接引用上游端口 |
| CODE | `{nodeId}~code-exe-defatult-port-{nodeId}` | 直接引用上游端口 |

### sourceType 含义
- `0`: 工作流输入参数（inputParams）
- `1`: 上游节点输出
- `2`: 常量（FILTER 条件对比值）

---

## 占位符规范

### 知识库
- `{{knowledge_id_描述}}` — 知识库 ID（需用户在平台替换）
- `{{knowledge_name_描述}}` — 知识库名称

### 工具
- `{{tool_id_描述}}` — 工具 ID
- `{{tool_name_描述}}` — 工具名称
- `{{tool_url_描述}}` — API URL
- `{{tool_token_描述}}` — 认证 Token

### 模型
- 默认值（`qwen-plus`, `tongyi`）可直接使用，用户可覆盖

---

## 布局算法

层级式 x/y 坐标：
- **x**: 按层递增，每层间隔 `300px`
  - Layer 0: x = 0（INPUT起点）
  - Layer 1: x = 300
  - Layer 2: x = 600
  - 以此类推
- **y**: 同层节点均匀分布，起始 y = 100，间隔 150px

示例：
```
Layer 1: y = 200
Layer 2:
  - 节点1: y = 100
  - 节点2: y = 250
Layer 3: y = 200
```

---

## 生成后自检清单

在输出 JSON 前，检查：
1. **promptId 唯一性**：所有节点 ID 不重复
2. **inComePorts 合法性**：每个节点的 inComePorts 引用的是上游节点存在的输出端口
3. **port 命名一致性**：
   - 节点的 outputParams.port 与实际配置一致
   - FILTER 的 filterConfig.conditions[].port 与 filterConfig.default 格式正确
4. **占位符完整性**：所有 `{{...}}` 占位符已标注说明
5. **position 无重叠**：同层节点 y 值不冲突
6. **endInComePorts 完整**：所有终止节点的输出端口都列入 endInComePorts

---

## 参考文件

生成前可读取以下文件作为参考：
- `schema/workflow-template.json` — 工作流顶层模板
- `schema/node-templates/llm.json` — LLM 节点模板
- `schema/node-templates/knowledge.json` — KNOWLEDGE 节点模板
- `schema/node-templates/code.json` — CODE 节点模板
- `schema/node-templates/filter.json` — FILTER 节点模板
- `schema/node-templates/tool.json` — TOOL 节点模板
- `schema/node-templates/agg.json` — AGG 节点模板
- `examples/simple-rag-qa.json` — 简单 RAG 问答示例
- `examples/intent-routing.json` — 意图路由示例

---

## 输出格式

生成完成后：
1. 写入文件：`output/{workflow_name}_{timestamp}.json`
2. 列出所有占位符及替换建议
3. 简要说明节点连接关系
