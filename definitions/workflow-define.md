# Workflow 3.0 定义

我现在希望实现一个 workflow copilot ，用于生成和编辑 workflow，我们的 workflow 相比传统的 workflow 最大的区别是，我们的 workflow 是一组 agent 的串联，通过上下文 + prompt + tools + 模型来解决问题，而并非传统的 workflow 要有 code、if/else、while 等，我们的每个节点是一个 agent，agent 内部可以做 if/else 路由，做 while 的拆分使用，写代码解决问题，这样我们的 workflow 搭建可以更简单，是第三代 AI Native 的 Workflow

## Workflow 架构对比示例

### Pre Workflow (传统工作流)

#### 传统 DSL 工作流示例 - 销售线索处理 (类似 n8n/dify)

```json
{
  "meta": {
    "instanceId": "lead-processing-workflow"
  },
  "nodes": [
    {
      "id": "trigger",
      "name": "文件上传触发器",
      "type": "n8n-nodes-base.manualTrigger",
      "position": [250, 300],
      "parameters": {}
    },
    {
      "id": "file-parser",
      "name": "文件解析器",
      "type": "n8n-nodes-base.spreadsheetFile",
      "position": [450, 300],
      "parameters": {
        "operation": "read",
        "options": {
          "headerRow": true
        }
      }
    },
    {
      "id": "data-validation",
      "name": "数据验证",
      "type": "n8n-nodes-base.function",
      "position": [650, 300],
      "parameters": {
        "functionCode": "// 验证邮箱格式\nconst emailRegex = /^[^\\s@]+@[^\\s@]+\\.[^\\s@]+$/;\nfor (let i = 0; i < items.length; i++) {\n  if (!emailRegex.test(items[i].json.email)) {\n    items[i].json.status = 'invalid';\n  }\n  if (!items[i].json.company_name) {\n    items[i].json.company_name = 'Unknown';\n  }\n}\nreturn items;"
      }
    },
    {
      "id": "company-enrichment",
      "name": "企业信息补全",
      "type": "n8n-nodes-base.httpRequest",
      "position": [850, 300],
      "parameters": {
        "url": "https://api.company-data.com/lookup",
        "method": "POST",
        "sendBody": true,
        "bodyParameters": {
          "company_name": "={{$json.company_name}}"
        }
      }
    },
    {
      "id": "score-calculation",
      "name": "评分计算",
      "type": "n8n-nodes-base.function",
      "position": [1050, 300],
      "parameters": {
        "functionCode": "for (let i = 0; i < items.length; i++) {\n  let score = 0;\n  const data = items[i].json;\n  \n  if (data.revenue > 1000000) score += 30;\n  if (data.employees > 100) score += 20;\n  if (['tech', 'finance'].includes(data.industry)) score += 25;\n  \n  items[i].json.score = score;\n}\nreturn items;"
      }
    },
    {
      "id": "route-decision",
      "name": "路由决策",
      "type": "n8n-nodes-base.switch",
      "position": [1250, 300],
      "parameters": {
        "conditions": {
          "number": [
            {
              "value1": "={{$json.score}}",
              "operation": "largerEqual",
              "value2": 70
            },
            {
              "value1": "={{$json.score}}",
              "operation": "largerEqual", 
              "value2": 40
            }
          ]
        }
      }
    },
    {
      "id": "high-priority-email",
      "name": "高优先级邮件",
      "type": "n8n-nodes-base.emailSend",
      "position": [1450, 200],
      "parameters": {
        "fromEmail": "sales@company.com",
        "toEmail": "manager@company.com",
        "subject": "高价值客户线索: {{$json.company_name}}",
        "text": "发现高价值客户线索，评分: {{$json.score}}"
      }
    },
    {
      "id": "medium-priority-task",
      "name": "中等优先级任务",
      "type": "n8n-nodes-base.httpRequest",
      "position": [1450, 300],
      "parameters": {
        "url": "https://api.crm.com/tasks",
        "method": "POST",
        "sendBody": true,
        "bodyParameters": {
          "title": "跟进客户: {{$json.company_name}}",
          "priority": "medium",
          "assigned_to": "sales_team"
        }
      }
    },
    {
      "id": "low-priority-nurture",
      "name": "低优先级培育",
      "type": "n8n-nodes-base.httpRequest",
      "position": [1450, 400],
      "parameters": {
        "url": "https://api.marketing.com/campaigns",
        "method": "POST",
        "sendBody": true,
        "bodyParameters": {
          "email": "={{$json.email}}",
          "campaign_type": "nurturing",
          "frequency": "weekly"
        }
      }
    }
  ],
  "connections": {
    "trigger": {
      "main": [
        [
          {
            "node": "file-parser",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "file-parser": {
      "main": [
        [
          {
            "node": "data-validation",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "data-validation": {
      "main": [
        [
          {
            "node": "company-enrichment",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "company-enrichment": {
      "main": [
        [
          {
            "node": "score-calculation",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "score-calculation": {
      "main": [
        [
          {
            "node": "route-decision",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "route-decision": {
      "main": [
        [
          {
            "node": "high-priority-email",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "medium-priority-task",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "low-priority-nurture",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}
```

**传统 DSL 工作流的问题：**

- **配置复杂**：需要配置大量节点、连接和参数，JSON 结构冗长难读
- **功能节点固化**：只能使用预定义的节点类型，扩展性有限
- **错误处理困难**：缺乏智能的异常处理和恢复机制
- **逻辑分散**：业务逻辑分散在多个节点的 `functionCode` 中，难以理解整体流程
- **调试困难**：需要逐个节点调试，难以追踪数据流转
- **维护成本高**：修改流程需要重新配置多个节点和连接关系
- **缺乏智能性**：无法根据业务场景自动调整处理策略
- **技术门槛**：仍需要一定的技术背景来理解节点配置和表达式语法

### Post Workflow (AI Native 3.0)

#### AI Native Workflow 3.0 示例 - 智能销售线索管理系统

```yaml
workflow_name: "CRM 线索智能评分与跟进自动化系统"
description: "自动处理客户线索，完成信息补全、优先级评分和个性化跟进方案"

start_parameters:
  leads_file: "file"          # 上传的客户线索文件
  company_info: "string"      # 公司背景和产品信息
  sales_criteria: "object"    # 评分标准配置
  team_config: "object"       # 团队结构和通知配置

agents:
  - name: "数据解析与清洗专家"
    context: "表格结构、数据质量标准、字段映射规则"
    prompt: "解析上传的线索文件，提取并标准化客户基础信息，处理数据质量问题"
    tools: ["文件解析器", "数据清洗工具", "格式转换", "重复数据检测"]
    output: "标准化客户数据结构、数据质量报告"

  - name: "信息补全调研员"
    context: "客户基础信息、企业数据源、行业知识库"
    prompt: "基于客户公司名称等信息，智能补全详细的企业背景和联系人信息"
    tools: ["企查查API", "LinkedIn API", "网络搜索", "邮箱验证"]
    output: "完整的客户画像数据、企业财务状况、决策人信息"

  - name: "销售评分分析师"
    context: "完整客户信息、历史转化数据、行业标准、竞品案例"
    prompt: "基于多维度客户信息和业务标准，智能计算销售优先级分数和转化概率"
    tools: ["评分算法", "机器学习模型", "业务规则引擎"]
    output: "客户评分（0-100分）、评分依据说明、预期转化时间"

  - name: "个性化营销专家"
    context: "高分客户信息、邮件模板库、产品卖点、成功案例"
    prompt: "为高价值客户（分数≥70）生成个性化跟进邮件和营销方案"
    tools: ["邮件模板引擎", "个性化内容生成", "A/B测试"]
    output: "个性化邮件内容、发送时机建议、跟进计划"
    condition: "customer_score >= 70"

  - name: "团队协作通知员"
    context: "中等分数客户信息、销售团队结构、任务分配规则"
    prompt: "为潜在客户（分数40-70）发送团队提醒和任务分配"
    tools: ["Slack API", "企业微信", "任务管理系统"]
    output: "团队通知消息、任务分配确认、跟进提醒"
    condition: "customer_score >= 40 && customer_score < 70"

  - name: "培育计划管理员"
    context: "低分客户信息、长期培育策略、自动化营销平台"
    prompt: "将低分客户（分数<40）归档到培育列表，设置自动化营销序列"
    tools: ["CRM API", "营销自动化平台", "定期任务调度"]
    output: "归档确认、培育计划、定期触达策略"
    condition: "customer_score < 40"

  - name: "综合报告生成员"
    context: "所有处理结果、报表模板、数据可视化要求"
    prompt: "生成包含客户分析、行动建议、ROI预测的综合管理报告"
    tools: ["表格生成器", "数据可视化工具", "PDF生成"]
    output: "完整的客户管理表格、可视化分析报告、执行计划"

final_output: "完整的客户管理表格、个性化跟进方案、团队任务分配、综合分析报告"
```

## AI Native Workflow 3.0 的核心优势

### 1. **自然语言描述 vs 复杂节点配置**

- **传统 DSL**：需要配置大量 JSON 节点，如 `"type": "n8n-nodes-base.function"`, `"functionCode": "复杂JS代码"`
- **AI Native 3.0**：直接用自然语言描述任务，如 `"解析上传的线索文件，提取并标准化客户基础信息"`

### 2. **智能Agent推理 vs 固定节点逻辑**

- **传统 DSL**：预定义节点类型，逻辑固化在 switch/function 节点中
- **AI Native 3.0**：每个Agent具有推理能力，可以根据上下文灵活处理各种情况

### 3. **业务语言配置 vs 技术实现配置**

- **传统 DSL**：关注 "怎么做"（HTTP请求、邮件发送、数据库查询等技术细节）
- **AI Native 3.0**：关注 "做什么"（智能补全信息、个性化营销、综合评估等业务目标）

### 4. **单一能力节点 vs 复合能力Agent**

- **传统 DSL**：一个节点只能做一件事，需要多个节点串联完成复杂任务
- **AI Native 3.0**：一个Agent可以完成一整个业务环节，内部自主调用多种工具

### 5. **静态流程 vs 动态适应**

- **传统 DSL**：流程路径固定，只能通过预设的 switch 条件分支
- **AI Native 3.0**：Agent可以根据实际情况动态选择处理策略和工具

### 6. **技术视角 vs 业务视角**

- **传统 DSL**：从技术实现角度构建流程，需要了解各种节点类型和API参数
- **AI Native 3.0**：从业务流程角度设计工作流，业务人员可以直接参与配置

## 直观对比总结

| 对比维度 | 传统 DSL 工作流 (n8n/dify) | AI Native Workflow 3.0 |
|---------|---------------------------|------------------------|
| **配置复杂度** | 217行 JSON，8个节点，复杂连接关系 | 简洁的 YAML，7个Agent描述 |
| **可读性** | 技术性强，需要理解节点类型和参数 | 业务语言描述，非技术人员也能理解 |
| **扩展性** | 受限于预定义节点类型 | Agent可以自由组合工具和能力 |
| **智能性** | 静态规则，需要预设所有条件 | 动态推理，可以处理未预见的情况 |
| **维护成本** | 修改需要重新配置多个节点 | 修改Agent的prompt即可调整逻辑 |
| **调试难度** | 需要逐个节点检查数据流转 | 每个Agent有明确的输入输出和职责 |
| **错误处理** | 手工配置错误分支和重试逻辑 | Agent内置智能异常处理能力 |
| **用户门槛** | 需要技术背景，理解API和表达式 | 业务人员可以用自然语言配置 |

这种设计让非技术用户也能构建复杂的业务流程，同时保持足够的灵活性和智能化程度，真正实现了第三代AI Native Workflow的愿景。
