---
description: "从需求到部署，端到端地编排功能开发"
argument-hint: "<功能描述> [--methodology tdd|bdd|ddd] [--complexity simple|medium|complex]"
---

# 功能开发编排器

## 关键行为规则

你必须严格遵守以下规则。违反任何一条都是失败。

1. **按顺序执行步骤。** 不要跳过、重新排序或合并步骤。
2. **写入输出文件。** 每个步骤必须在下一步开始之前将其输出文件写入 `.feature-dev/`。读取上一步的文件——不要依赖上下文窗口记忆。
3. **在检查点停止。** 当你到达 `阶段检查点` 时，必须停止并等待用户的明确批准后再继续。使用 AskUserQuestion 工具并提供清晰的选项。
4. **失败时暂停。** 如果任何步骤失败（agent 错误、测试失败、缺少依赖），立即停止。呈现错误并询问用户如何继续。不要静默继续。
5. **仅使用本地 agent。** 所有 `subagent_type` 引用使用本插件捆绑的 agent 或 `general-purpose`。不要有跨插件依赖。
6. **遵循本命令内置的执行计划。** 本命令自身已定义分阶段执行流程，无需额外进入计划模式。仅在用户明确要求时才使用 EnterPlanMode。

## 飞行前检查

开始前，执行以下检查：

### 1. 检查现有会话

检查 `.feature-dev/state.json` 是否存在：

- 如果存在且 `status` 为 `"in_progress"`：读取它，显示当前步骤，并询问用户：

  ```
  发现正在进行的功能开发会话：
  功能：[state 中的 name]
  当前步骤：[state 中的 step]

  1. 从上次离开的地方继续
  2. 重新开始（归档现有会话）
  ```

- 如果存在且 `status` 为 `"complete"`：询问是否归档并开始新的。

### 2. 初始化状态

创建 `.feature-dev/` 目录和 `state.json`：

```json
{
  "feature": "$ARGUMENTS",
  "status": "in_progress",
  "methodology": "traditional",
  "complexity": "medium",
  "current_step": 1,
  "current_phase": 1,
  "completed_steps": [],
  "files_created": [],
  "started_at": "ISO_TIMESTAMP",
  "last_updated": "ISO_TIMESTAMP"
}
```

解析 `$ARGUMENTS` 中的 `--methodology` 和 `--complexity` 标志。如未指定则使用默认值。

### 3. 解析功能描述

从 `$ARGUMENTS` 中提取功能描述（标志之前的所有内容）。这会在下文的提示中作为 `$FEATURE` 引用。

---

## 第一阶段：发现（步骤 1–2）—— 交互式

### 步骤 1：需求收集

通过交互式问答收集需求。一次只问一个问题，使用 AskUserQuestion 工具。不要一次性问所有问题。

**按顺序提问：**

1. **问题陈述**："这个功能解决什么问题？用户是谁，他们的痛点是什么？"
2. **验收标准**："关键的验收标准是什么？这个功能什么时候算'完成'？"
3. **范围边界**："这个功能明确排除在外的范围是什么？"
4. **技术约束**："有什么技术约束？（例如，必须使用现有认证系统、特定数据库、延迟要求）"
5. **依赖关系**："这个功能是否依赖或影响其他功能/服务？"

收集答案后，写入需求文档：

**输出文件：** `.feature-dev/01-requirements.md`

```markdown
# 需求：$FEATURE

## 问题陈述

[来自 Q1]

## 验收标准

[来自 Q2 — 格式化为复选框]

## 范围

### 范围内

[从答案推导]

### 范围外

[来自 Q3]

## 技术约束

[来自 Q4]

## 依赖关系

[来自 Q5]

## 方法论：[tdd|bdd|ddd|traditional]

## 复杂度：[simple|medium|complex]
```

更新 `state.json`：设置 `current_step` 为 2，将 `"01-requirements.md"` 添加到 `files_created`，将步骤 1 添加到 `completed_steps`。

### 步骤 2：架构与安全设计

读取 `.feature-dev/01-requirements.md` 以加载需求上下文。

使用 Task 工具启动架构 agent：

```
Task:
  subagent_type: "backend-architect"
  description: "为 $FEATURE 设计架构"
  prompt: |
    为此功能设计技术架构。

    ## 需求
    [插入 .feature-dev/01-requirements.md 的完整内容]

    ## 交付物
    1. **服务/组件设计**：需要什么组件、它们的职责和边界
    2. **API 设计**：端点、请求/响应模式、错误处理
    3. **数据模型**：数据库表/集合、关系、需要的迁移
    4. **安全考虑**：认证需求、输入验证、数据保护、OWASP 关注点
    5. **集成点**：如何连接到现有服务/系统
    6. **风险评估**：技术风险和缓解策略

    将完整的架构设计写成单个 markdown 文档。
```

将 agent 的输出保存到 `.feature-dev/02-architecture.md`。

更新 `state.json`：设置 `current_step` 为 "checkpoint-1"，将步骤 2 添加到 `completed_steps`。

---

## 阶段检查点 1 —— 需要用户批准

你必须在此处停止并呈现架构以供审查。

显示 `.feature-dev/02-architecture.md` 的摘要（关键组件、API 端点、数据模型概览）并询问：

```
架构设计已完成。请审查 .feature-dev/02-architecture.md

1. 批准 —— 继续实施
2. 请求修改 —— 告诉我需要调整什么
3. 暂停 —— 保存进度并在此处停止
```

在用户选择选项 1 之前不要进入第二阶段。如果他们选择选项 2，修改架构并重新检查点。如果选项 3，更新 `state.json` 状态并停止。

---

## 第二阶段：实施（步骤 3–5）

### 步骤 3：后端实现

读取 `.feature-dev/01-requirements.md` 和 `.feature-dev/02-architecture.md`。

使用 Task 工具启动后端架构师进行实施：

```
Task:
  subagent_type: "backend-architect"
  description: "实现 $FEATURE 的后端"
  prompt: |
    基于已批准的架构实现此功能的后端。

    ## 需求
    [插入 .feature-dev/01-requirements.md 的内容]

    ## 架构
    [插入 .feature-dev/02-architecture.md 的内容]

    ## 指令
    1. 按设计实现 API 端点、业务逻辑和数据访问层
    2. 包含架构中指定的数据层组件（模型、迁移、仓库）
    3. 添加输入验证和错误处理
    4. 遵循项目现有的代码模式和约定
    5. 如果方法论是 TDD：先写失败的测试，再实现
    6. 仅在逻辑不明显的地方添加内联注释

    编写所有代码文件。报告创建/修改了哪些文件。
```

将实施的摘要保存到 `.feature-dev/03-backend.md`（创建/修改的文件列表、关键决策、与架构的任何偏差）。

更新 `state.json`：设置 `current_step` 为 4，将步骤 3 添加到 `completed_steps`。

### 步骤 4：前端实现

读取 `.feature-dev/01-requirements.md`、`.feature-dev/02-architecture.md` 和 `.feature-dev/03-backend.md`。

使用 Task 工具：

```
Task:
  subagent_type: "general-purpose"
  description: "实现 $FEATURE 的前端"
  prompt: |
    你是一位前端开发者。为此功能实现前端组件。

    ## 需求
    [插入 .feature-dev/01-requirements.md 的内容]

    ## 架构
    [插入 .feature-dev/02-architecture.md 的内容]

    ## 后端实现
    [插入 .feature-dev/03-backend.md 的内容]

    ## 指令
    1. 构建与后端 API 端点集成的 UI 组件
    2. 实现状态管理、表单处理和错误状态
    3. 在适当的地方添加加载状态和乐观更新
    4. 遵循项目现有的前端模式和组件约定
    5. 确保响应式设计和基础无障碍（语义化 HTML、ARIA 标签、键盘导航）

    编写所有代码文件。报告创建/修改了哪些文件。
```

将摘要保存到 `.feature-dev/04-frontend.md`。

**注意：** 如果功能没有前端组件（纯后端/API），跳过此步骤——在 `04-frontend.md` 中写一个简短的说明解释为什么跳过，然后继续。

更新 `state.json`：设置 `current_step` 为 5，将步骤 4 添加到 `completed_steps`。

### 步骤 5：测试与验证

读取 `.feature-dev/03-backend.md` 和 `.feature-dev/04-frontend.md`。

使用多个 Task 工具并行启动三个 agent：

**5a. 测试套件创建：**

```
Task:
  subagent_type: "test-automator"
  description: "为 $FEATURE 创建测试套件"
  prompt: |
    为此功能创建全面的测试套件。

    ## 已实现的内容
    ### 后端
    [插入 .feature-dev/03-backend.md 的内容]

    ### 前端
    [插入 .feature-dev/04-frontend.md 的内容]

    ## 指令
    1. 为所有新后端函数/方法编写单元测试
    2. 为 API 端点编写集成测试
    3. 如适用，编写前端组件测试
    4. 覆盖：正常路径、边界情况、错误处理、边界条件
    5. 遵循项目现有的测试模式和框架
    6. 新代码目标 80%+ 覆盖率

    编写所有测试文件。报告创建了哪些测试文件以及它们覆盖什么。
```

**5b. 安全审查：**

```
Task:
  subagent_type: "security-auditor"
  description: "$FEATURE 的安全审查"
  prompt: |
    对此功能实施进行安全审查。

    ## 架构
    [插入 .feature-dev/02-architecture.md 的内容]

    ## 后端实现
    [插入 .feature-dev/03-backend.md 的内容]

    ## 前端实现
    [插入 .feature-dev/04-frontend.md 的内容]

    审查：OWASP Top 10、认证/授权缺陷、输入验证缺口、
    数据保护问题、依赖漏洞、以及任何安全反模式。

    提供发现项，附严重性、位置和具体修复建议。
```

**5c. 性能审查：**

```
Task:
  subagent_type: "performance-engineer"
  description: "$FEATURE 的性能审查"
  prompt: |
    审查此功能实现的性能。

    ## 架构
    [插入 .feature-dev/02-architecture.md 的内容]

    ## 后端实现
    [插入 .feature-dev/03-backend.md 的内容]

    ## 前端实现
    [插入 .feature-dev/04-frontend.md 的内容]

    审查：N+1 查询、缺失索引、未优化查询、内存泄漏、
    缺失缓存机会、大载荷、慢渲染路径。

    提供发现项，附影响估算和具体优化建议。
```

三个全部完成后，将结果整合到 `.feature-dev/05-testing.md`：

```markdown
# 测试与验证：$FEATURE

## 测试套件

[来自 5a 的摘要——创建的文件、覆盖区域]

## 安全发现项

[来自 5b 的摘要——按严重性分类的发现项]

## 性能发现项

[来自 5c 的摘要——按影响分类的发现项]

## 行动项

[列出交付前需要解决的任何关键/高优先级发现项]
```

如果安全或性能审查有关键或高严重性的发现项，立即修复。应用修复并重新验证。

更新 `state.json`：设置 `current_step` 为 "checkpoint-2"，将步骤 5 添加到 `completed_steps`。

---

## 阶段检查点 2 —— 需要用户批准

显示 `.feature-dev/05-testing.md` 的测试和验证结果摘要并询问：

```
测试和验证完成。请审查 .feature-dev/05-testing.md

测试覆盖率：[摘要]
安全发现项：[X 关键, Y 高, Z 中]
性能发现项：[X 关键, Y 高, Z 中]

1. 批准 —— 继续部署和文档
2. 请求修改 —— 告诉我修复什么
3. 暂停 —— 保存进度并在此处停止
```

在用户批准之前不要进入第三阶段。

---

## 第三阶段：交付（步骤 6–7）

### 步骤 6：部署与监控

读取 `.feature-dev/02-architecture.md` 和 `.feature-dev/05-testing.md`。

使用 Task 工具：

```
Task:
  subagent_type: "general-purpose"
  description: "为 $FEATURE 创建部署配置"
  prompt: |
    你是一位部署工程师。为此功能创建部署和监控配置。

    ## 架构
    [插入 .feature-dev/02-architecture.md 的内容]

    ## 测试结果
    [插入 .feature-dev/05-testing.md 的内容]

    ## 指令
    1. 为新代码创建或更新 CI/CD 流水线配置
    2. 如果功能应逐步推出，添加功能标志配置
    3. 为新服务/端点定义健康检查和就绪探针
    4. 为关键指标（错误率、延迟、吞吐量）创建监控告警
    5. 编写包含回滚步骤的部署运维手册
    6. 遵循项目现有的部署模式

    编写所有配置文件。报告创建/修改了什么。
```

将输出保存到 `.feature-dev/06-deployment.md`。

更新 `state.json`：设置 `current_step` 为 7，将步骤 6 添加到 `completed_steps`。

### 步骤 7：文档与交接

读取所有之前的 `.feature-dev/*.md` 文件。

使用 Task 工具：

```
Task:
  subagent_type: "general-purpose"
  description: "为 $FEATURE 编写文档"
  prompt: |
    你是一位技术文档撰写者。为此功能创建文档。

    ## 功能上下文
    [插入 .feature-dev/01-requirements.md 的内容]

    ## 架构
    [插入 .feature-dev/02-architecture.md 的内容]

    ## 实施摘要
    ### 后端：[插入 .feature-dev/03-backend.md 的内容]
    ### 前端：[插入 .feature-dev/04-frontend.md 的内容]

    ## 部署
    [插入 .feature-dev/06-deployment.md 的内容]

    ## 指令
    1. 为新端点编写 API 文档（请求/响应示例）
    2. 如适用，更新或创建面向用户的文档
    3. 编写简短的架构决策记录（ADR），解释关键设计选择
    4. 创建交接摘要：构建了什么、如何测试、已知限制

    编写文档文件。报告创建/修改了什么。
```

将输出保存到 `.feature-dev/07-documentation.md`。

更新 `state.json`：设置 `current_step` 为 "complete"，将步骤 7 添加到 `completed_steps`。

---

## 完成

更新 `state.json`：

- 设置 `status` 为 `"complete"`
- 设置 `last_updated` 为当前时间戳

呈现最终摘要：

```
功能开发完成：$FEATURE

## 创建的文件
[列出所有 .feature-dev/ 输出文件]

## 实施摘要
- 需求：.feature-dev/01-requirements.md
- 架构：.feature-dev/02-architecture.md
- 后端：.feature-dev/03-backend.md
- 前端：.feature-dev/04-frontend.md
- 测试：.feature-dev/05-testing.md
- 部署：.feature-dev/06-deployment.md
- 文档：.feature-dev/07-documentation.md

## 下一步
1. 审查所有生成的代码和文档
2. 运行完整测试套件以验证全部通过
3. 创建包含实现的拉取请求
4. 使用 .feature-dev/06-deployment.md 中的运维手册进行部署
```
