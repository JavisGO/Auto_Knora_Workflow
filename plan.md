# Auto Knora Workflow Generator - 最终实施方案



## Context



### 为什么做这个

Knora 平台创建工作流需手动拖拽配置每个节点，复杂工作流效率很低。本项目构建一个基于 Claude Code Skill 的工作流生成器，用户通过**交互式问答**描述需求，自动生成可导入 Knora 平台的工作流 JSON。



### 用户确认的关键决策

| 决策项 | 选择 |

|--------|------|

| 形态 | 纯 Claude Code Skill（先验证效果，后续按需加代码辅助） |

| 工具资源 | 混合模式：有列表时直接引用，无列表时生成占位符+建议 |

| 输入方式 | 交互式问答引导（意图→节点→配置逐步明确） |

| 资源占位 | `{{placeholder}}` 语法 |

| 输出 | JSON 文件导出到 `output/` 目录 |



---



## 文件结构



```

Auto_Knora_Workflow/

├── .claude/

│   └── skills/

│       └── knora-workflow.md              # 核心 Skill（交互式问答 + 生成）

├── schema/

│   ├── workflow-template.json             # 工作流顶层空壳模板

│   └── node-templates/                    # 6 种节点模板

│       ├── llm.json

│       ├── knowledge.json

│       ├── code.json

│       ├── filter.json

│       ├── tool.json

│       └── agg.json

├── examples/                              # 验证用示例工作流

│   ├── simple-rag-qa.json

│   └── intent-routing.json

├── output/                                # 生成输出目录

├── reference/                             # 已有参考资料（不动）

│   ├── knora4.0-docs/

│   └── workflow_output/

├── CLAUDE.md

└── README.md

```



---



## 分阶段实施



### Phase 1: Schema 模板提取

**目标**：从 3 个样例 JSON 提取标准模板



**创建文件：**



1. **`schema/workflow-template.json`** — 工作流顶层空壳

   - 包含所有顶层字段的默认值

   - 关键字段：chainId, chainName, chainDesc, chainType, prompts[], inputParams[], outputParams[], endInComePorts[], globalParam, startPosition, endPosition, registers[], presetQuestions[]

   - chainType 默认 "TOOL"，isMcp/isTool/publish 默认 false



2. **`schema/node-templates/*.json`** — 6 种节点模板，每个包含：

   - 完整的节点字段结构（promptId, promptName, promptType, position, inputParams, outputParams, inComePorts）

   - 该类型的活跃 config 字段有示例值，其他 config 字段设为 null

   - 关键节点配置细节：

     - **LLM**: llmConfig（system, content, temperature, model, provider, maxTokens, streamMode, rounds, errorRetryLimit）

     - **KNOWLEDGE**: knowledgeConfig（knowledgeList[{knowledgeId,knowledgeName}], searchType, searchLimit, minCorrelation, rerankerModel, resultType, usedParams）

     - **CODE**: codeConfig（code_language, code, variables[{variable, value_selector}], dependencies）

     - **FILTER**: filterConfig（conditions[{expressions[{paramId, operator, value}], port, logic}], default）

     - **TOOL**: toolSelector（tools[{toolId, toolName, toolDesc, toolType, config.reqDefine{url,method,headers,body,params}, toolOutput}], useLLM, prompt）

     - **AGG**: actionConfig（变量聚合配置）



**输入**：`reference/workflow_output/*.json`（3 个样例）



### Phase 2: Skill Prompt 工程（核心）

**目标**：编写 `.claude/skills/knora-workflow.md`



Skill 设计为**交互式问答引导**流程：



#### 问答流程设计

```

Step 1: 理解意图

  → "请描述你想创建的工作流目标？"

  → "工作流要处理什么类型的输入？（文本/文件/图片/混合）"



Step 2: 确定节点组合

  → 根据意图推荐节点方案（如：RAG = KNOWLEDGE + LLM）

  → "是否需要条件分支？"

  → "是否需要调用外部工具/API？"

  → 展示推荐的节点流程图（文本形式）



Step 3: 资源配置

  → "你有可用的工具/知识库列表吗？（提供 JSON 或描述，没有也可以）"

  → 有列表：直接引用 toolId/knowledgeId

  → 无列表：生成 {{placeholder}} 并附带配置建议

  → LLM 模型默认 qwen-plus/tongyi，询问是否修改



Step 4: 生成并输出

  → 生成完整工作流 JSON

  → 写入 output/{workflow_name}_{timestamp}.json

  → 输出占位符清单供用户在平台上替换

```



#### Skill Prompt 核心内容

1. **角色定义**：Knora 工作流生成专家，交互式引导用户

2. **内联 Schema 摘要**：精简版工作流结构说明（非完整 schema，控制 prompt 长度）

3. **6 种节点配置规则**：各节点的必填字段、使用场景、典型配置

4. **连线规则**：

   - 节点 ID：13 位时间戳数字字符串（递增）

   - 端口命名规范：

     - 默认端口：`{nodeId}~port-default`

     - FILTER 分支端口：`{nodeId}~filter-port-{nodeId}`

     - FILTER 默认端口：`{nodeId}~filter-default-port-{nodeId}`

     - TOOL 端口：`{nodeId}~TOOL-port-{toolId}`

     - CODE 端口：`{nodeId}~code-exe-default-port-{newId}`

     - AGG 端口：`{nodeId}~port-{newId}`

   - sourceType：0=工作流输入参数, 1=上游节点输出, 3=常量

5. **占位符规范**：

   - 知识库：`{{knowledge_id_描述}}`, `{{knowledge_name_描述}}`

   - 工具：`{{tool_id_描述}}`, `{{tool_url_描述}}`, `{{tool_token_描述}}`

   - 模型：默认值可直接使用，用户可覆盖

6. **布局算法**：层级式 x/y 坐标（x 按层递增 300px，y 按同层节点数均匀分布）

7. **验证清单**（生成后自检）：

   - promptId 唯一性

   - inComePorts 引用合法性

   - port 命名与 inComePorts 一致性

   - 占位符标记完整性

   - position 无重叠

8. **参考指令**：告诉 Claude 读取 `schema/node-templates/` 和 `examples/` 作为生成参考



### Phase 3: 示例工作流制作

**目标**：手工制作 2 个示例，验证模板正确性



1. **`examples/simple-rag-qa.json`** — 简单 RAG 问答（4 节点）

   - 输入(文本) → KNOWLEDGE(知识库检索) → LLM(生成回答) → 输出

   - 演示：基础连线、占位符、inputParams/outputParams



2. **`examples/intent-routing.json`** — 意图路由（8 节点）

   - 输入 → LLM(意图识别) → FILTER(分支) → [KNOWLEDGE+LLM / TOOL / CODE] → AGG → 输出

   - 演示：FILTER 多端口、AGG 聚合、TOOL 占位符



### Phase 4: 项目配置

1. **`CLAUDE.md`**：

   - 项目说明

   - `/knora-workflow` Skill 使用方式

   - Schema 和模板文件位置

   - 生成输出目录说明



2. **`README.md`** 更新



### Phase 5: 端到端验证

1. 调用 `/knora-workflow` Skill，走完交互式问答流程

2. 对比生成 JSON 与 `reference/workflow_output/` 样例的结构一致性

3. 验证占位符完整标记

4. 检查连线规则正确性（inComePorts、port 引用）



---



## 关键设计决策



| 决策 | 选择 | 原因 |

|------|------|------|

| 增强方式 | 纯 Skill，暂不加代码 | 先验证 prompt 工程效果，避免过度设计 |

| Schema 位置 | Skill 内联摘要 + 外部模板文件 | Skill 无法 import，但指令 Claude 读取外部文件作为参考 |

| 工具资源 | 混合模式 | 灵活适配有/无资源列表的场景 |

| 输入方式 | 交互式问答 | 比纯自然语言更精确，逐步引导减少歧义 |

| 节点 ID | 13 位时间戳字符串，递增 | 与样例一致 |

| 空 config | null | 样例中非活跃 config 字段都是 null |

| 复杂度控制 | 问答中推荐简单方案优先 | 用户可在平台上细化 |



---



## 风险与缓解



| 风险 | 缓解措施 |

|------|----------|

| Prompt 过长 | 内联精简摘要，完整信息放外部文件让 Claude 按需读取 |

| 端口连线复杂 | 在 Skill 中详细列出各节点端口命名规则 + 示例 |

| 未覆盖节点类型 | 暂只支持 6 种核心类型，后续按需扩展（Loop, Batch, Register 等） |

| JSON 兼容性 | 示例工作流基于真实样例结构制作，作为生成参考 |

| 交互式引导可能啰嗦 | 允许用户跳过问答直接提供完整描述 |



---



## 关键文件清单



| 文件 | 操作 | 说明 |

|------|------|------|

| `.claude/skills/knora-workflow.md` | 新建 | 核心 Skill 定义 |

| `schema/workflow-template.json` | 新建 | 工作流顶层模板 |

| `schema/node-templates/llm.json` | 新建 | LLM 节点模板 |

| `schema/node-templates/knowledge.json` | 新建 | KNOWLEDGE 节点模板 |

| `schema/node-templates/code.json` | 新建 | CODE 节点模板 |

| `schema/node-templates/filter.json` | 新建 | FILTER 节点模板 |

| `schema/node-templates/tool.json` | 新建 | TOOL 节点模板 |

| `schema/node-templates/agg.json` | 新建 | AGG 节点模板 |

| `examples/simple-rag-qa.json` | 新建 | RAG 示例工作流 |

| `examples/intent-routing.json` | 新建 | 意图路由示例工作流 |

| `CLAUDE.md` | 新建 | 项目配置 |

| `README.md` | 更新 | 项目说明 |

| `reference/workflow_output/*.json` | 只读 | 参考样例 |

