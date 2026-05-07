# Prompt Engineering Skill 调用指令规范

> 版本：2.2.0 | 最后更新：2026-05-07
> 用途：定义如何精确调用 prompt-engineering Skill，确保大模型准确理解输入、执行工作流、输出标准化结果

---

## 1. 触发条件判定

在以下任一情况出现时，激活此 Skill：

| 信号类型 | 关键词/模式 | 对应 Action |
|----------|------------|-------------|
| 需要新建提示词 | "写一个提示词"、"设计prompt"、"帮我写system prompt" | `design` |
| 需要改进提示词 | "优化"、"改进"、"效果不好" + 提供了现有提示词 | `optimize` |
| 提示词有问题 | "格式不对"、"编造信息"、"不一致"、"拒绝回答" + 提供了现有提示词 | `debug` |
| 需要技术推荐 | "用什么技术"、"CoT和ToT哪个好"、"怎么选方法" | `select` |
| 需要模板 | "有没有模板"、"给我一个xxx的prompt" | `template` |
| 需要质量检查 | "检查这个提示词"、"有什么问题"、"反模式" + 提供了现有提示词 | `review` |

**判定规则**：
- 如果用户提供了现有提示词 → 只能是 `optimize` / `debug` / `review`
- 如果用户没提供现有提示词 → 只能是 `design` / `select` / `template`
- 如果用户意图模糊 → 优先匹配 `design`（最通用的入口）

---

## 2. 输入参数规范

### 2.1 必填参数

调用此 Skill 时，**必须**明确提供以下信息：

```yaml
action: <必填，枚举值>
  # design | optimize | debug | select | template | review
  # 决定执行哪种工作流

task_description: <必填，字符串>
  # 用自然语言描述用户想要完成什么
  # 不能少于10个字，越具体越好
  # 示例："构建一个客服工单分类系统，将反馈分为Bug/Feature/Question/Complaint四类"
```

### 2.2 条件必填参数

```yaml
current_prompt: <条件必填，字符串>
  # 当 action 为 optimize / debug / review 时必须提供
  # 用户现有的、需要被分析或改进的提示词原文
  # 必须是完整文本，不能只提供摘要
```

### 2.3 上下文参数（强烈推荐）

```yaml
context:
  model: <推荐，枚举值，默认 generic>
    # gpt-4 | gpt-4o | gpt-4.1 | claude-3.5-sonnet | claude-3.7-sonnet
    # deepseek-r1 | llama | qwen | generic
    # 影响模型适配策略（Claude用XML+Prefill，GPT用JSON mode等）

  task_type: <推荐，枚举值，默认自动推断>
    # classification | generation | reasoning | qa | code | agent | creative | multi_step
    # 影响技术选型路径
    # 如果无法确定，可以留空让 Skill 自动推断

  output_format: <按需，枚举值>
    # json | xml | markdown | table | code | natural_language
    # 当用户对输出格式有明确要求时提供
    # 留空则由 Skill 根据任务类型推荐最佳格式

  failure_mode: <条件推荐，枚举值>
    # hallucination | format_mismatch | inconsistency | verbosity | inaccuracy | refusal
    # 当 action 为 optimize / debug 时强烈推荐提供
    # 描述当前提示词的具体问题

  language: <可选，枚举值，默认 zh>
    # zh | en | bilingual
    # 输出提示词的语言

  complexity: <可选，枚举值，默认 moderate>
    # simple | moderate | complex | production
    # 生产级（production）会触发更严格的反模式检查和更多安全边界
```

### 2.4 参数提取规则

当用户的自然语言请求不包含明确的参数时，按以下规则推断：

| 用户说 | 推断规则 |
|--------|----------|
| "帮我写一个xx的提示词" | `action=design`, `task_description=用户描述` |
| "我用的Claude/GPT" | `context.model=claude-3.7-sonnet` 或 `gpt-4o` |
| "输出JSON" | `context.output_format=json` |
| "模型在编信息" | `context.failure_mode=hallucination` |
| "格式不对/格式乱了" | `context.failure_mode=format_mismatch` |
| "每次结果不一样" | `context.failure_mode=inconsistency` |
| "太啰嗦了" | `context.failure_mode=verbosity` |
| "给产品用的/线上用" | `context.complexity=production` |
| "随便写一个/简单就行" | `context.complexity=simple` |
| 未提及模型 | `context.model=generic`（保守策略） |
| 未提及任务类型 | `context.task_type=null`（自动推断） |

---

## 3. 执行工作流

调用此 Skill 后，必须严格按以下5步顺序执行：

```
┌─────────────────────────────────────────────┐
│  Step 1: 任务分析                            │
│  ├─ 识别 task_type（如未提供则推断）          │
│  ├─ 识别 constraint_level                    │
│  └─ 识别 failure_mode（如有）                │
├─────────────────────────────────────────────┤
│  Step 2: 技术选型                            │
│  ├─ 根据 task_type 查决策树                  │
│  ├─ 选择主技术 + 备选技术                    │
│  └─ 参考 references/techniques-overview.md   │
├─────────────────────────────────────────────┤
│  Step 3: 提示词构建                          │
│  ├─ 按10元素结构组织                         │
│  ├─ 按 model 适配格式                       │
│  └─ 参考 references/anthropic-tutorial.md    │
├─────────────────────────────────────────────┤
│  Step 4: 反模式检查                          │
│  ├─ 逐项检查15项清单                         │
│  ├─ 标记 致命/重要/建议 三级                 │
│  └─ 参考 references/anti-patterns.md         │
├─────────────────────────────────────────────┤
│  Step 5: 输出结果                            │
│  ├─ 组装标准化输出对象                       │
│  └─ 添加 model_adaptation 和 next_steps      │
└─────────────────────────────────────────────┘
```

### 各 Action 的特殊工作流

| Action | Step 1 差异 | Step 2 差异 | Step 3 差异 | Step 4 差异 |
|--------|------------|------------|------------|------------|
| `design` | 完整分析四维度 | 完整决策树 | 从零构建10元素 | 检查产出物 |
| `optimize` | 诊断原提示词缺陷 | 针对性选择修复技术 | 增量修改，保留可用部分 | 对比修改前后 |
| `debug` | 聚焦 failure_mode | 选择对应修复技术 | 仅修复问题部分 | 验证问题已消除 |
| `select` | 简化分析 | **核心步骤**，按复杂度递进推荐 | 不构建提示词 | 不执行 |
| `template` | 匹配需求到模板类 | 不选技术，选模板 | 填充模板 `{{变量}}` | 检查模板产出物 |
| `review` | 理解提示词意图 | 不选技术 | 不修改提示词 | **核心步骤**，完整15项检查 |

---

## 4. 输出格式规范

所有 Action 统一输出以下结构，确保可编程解析：

```yaml
result:
  # ─── 核心输出 ───
  prompt: str
    # design/optimize/debug/template: 完整的构建/优化后提示词文本
    # select/review: 空字符串（这两个action不产出提示词）

  technique: str
    # 应用的技术组合，用 " + " 连接
    # 示例："Role Prompt + Few-Shot + XML Structure + Escape Hatch"
    # 示例："CoT → Self-Consistency → PAL"（select的递进推荐路径）

  structure: list[str]
    # 提示词使用的元素列表，按出现顺序排列
    # 可选值：persona, instructions, examples, input_data, format,
    #         constraints, tone, escape_hatch, user_query
    # select/review: 空列表

  # ─── 反模式检查 ───
  anti_pattern_check:
    passed: list[str]
      # 通过的检查项，格式："#编号 描述"
      # 示例：["#1 消息交替", "#5 XML标签分隔", "#7 输出格式明确"]
    warnings: list[str]
      # 警告级问题（重要级+建议级）
      # 示例：["#8 考虑添加prefill", "#10 考虑选项顺序偏差"]
    failures: list[str]
      # 致命级问题（必须修复）
      # 示例：["#7 不指定输出格式", "#11 无不知道选项"]
      # 正常交付时此列表必须为空

  # ─── 决策解释 ───
  reasoning: str
    # 为什么选择这个技术/结构，包含：
    # - 任务类型判断依据
    # - 技术选型理由（引用决策树路径）
    # - 关键权衡说明

  model_adaptation: str
    # 模型特定适配说明
    # 空字符串表示未做特殊适配（generic模型）
    # 示例："Claude 3.7: 使用XML标签结构化，建议prefill '{' 强制JSON输出"

  next_steps: list[str]
    # 后续建议，按优先级排序
    # 示例：
    #   - "测试：提交10条真实数据验证准确率"
    #   - "优化：如准确率<95%，添加更多边界case示例"
    #   - "设置 temperature=0 提高事实性任务稳定性"
```

### 各 Action 的输出重点差异

| Action | `prompt` | `technique` 重点 | `anti_pattern_check` | `next_steps` 重点 |
|--------|----------|-----------------|---------------------|-------------------|
| `design` | 完整新提示词 | 设计技术组合 | 必须全部通过致命级 | 测试+迭代建议 |
| `optimize` | 优化后提示词 | 修复技术 | 对比原提示词改善项 | 验证改善效果 |
| `debug` | 修复后提示词 | 诊断+修复技术 | 验证问题已消除 | 防复发建议 |
| `select` | 空 | 递进推荐路径 | 空 | 实施步骤 |
| `template` | 参数化模板 | 模板+定制方法 | 检查模板完整性 | 定制+测试步骤 |
| `review` | 空 | 检查清单 | **核心输出**，完整15项 | 按优先级修复建议 |

---

## 5. 标准调用指令模板

以下是可以直接复制使用的调用指令，按6种 Action 分别给出：

### 5.1 Design — 从零设计

```
你现在以【提示词工程专家】身份执行任务。

## 任务
根据以下输入，设计一个完整的生产级提示词。

## 输入
- 动作：design
- 任务描述：{task_description}
- 目标模型：{model}
- 任务类型：{task_type}
- 输出格式：{output_format}
- 语言：{language}
- 复杂度：{complexity}

## 执行要求
1. 先分析任务类型和约束等级
2. 按 references/techniques-overview.md 的决策树选择技术
3. 按10元素结构构建提示词（persona → instructions → examples → input_data → format → constraints → tone → escape_hatch → user_query）
4. 执行15项反模式检查，确保致命级全部通过
5. 根据目标模型适配格式（Claude→XML+Prefill, GPT→JSON mode, 开源→多示例+简指令）

## 输出格式
严格按以下YAML结构输出：
```yaml
result:
  prompt: |
    <完整提示词文本>
  technique: "<技术组合，用+连接>"
  structure: [<元素列表>]
  anti_pattern_check:
    passed: [<通过项>]
    warnings: [<警告项>]
    failures: []  # 致命项必须为空
  reasoning: "<选型理由>"
  model_adaptation: "<模型适配说明>"
  next_steps:
    - "<建议1>"
    - "<建议2>"
```

## 关键约束
- 提示词必须包含退出选项（"如果不确定，请说明"）
- 用户查询必须放在提示词底部
- 必须用XML标签分隔指令和数据
- 必须明确指定输出格式
- 如果是Claude，优先用XML结构化和Prefill
- 如果是GPT，优先用JSON mode和system消息
```

### 5.2 Optimize — 优化提示词

```
你现在以【提示词工程专家】身份执行任务。

## 任务
诊断并优化以下提示词，解决 {failure_mode} 问题。

## 输入
- 动作：optimize
- 任务描述：{task_description}
- 现有提示词：
```
{current_prompt}
```
- 目标模型：{model}
- 问题模式：{failure_mode}

## 执行要求
1. 先用15项反模式清单审查现有提示词，识别导致 {failure_mode} 的根因
2. 针对性选择修复技术（幻觉→grounding+退出选项，格式→schema+prefill，不一致→few-shot，啰嗦→长度限制+去前言）
3. 增量修改，保留原提示词中有效的部分
4. 验证修改后反模式检查的致命项为空
5. 对比修改前后，解释每处修改的理由

## 输出格式
同 design 的标准YAML结构，额外在 reasoning 中包含：
- 原提示词的问题诊断
- 每处修改的具体理由
- 修改前后的关键差异
```

### 5.3 Debug — 诊断修复

```
你现在以【提示词工程专家】身份执行任务。

## 任务
诊断以下提示词的 {failure_mode} 问题并提供修复版本。

## 输入
- 动作：debug
- 问题描述：{task_description}
- 问题提示词：
```
{current_prompt}
```
- 目标模型：{model}
- 问题模式：{failure_mode}

## 执行要求
1. 定位问题根因：用15项反模式清单逐项排查，找出导致 {failure_mode} 的具体项
2. 按优先级修复：致命级 → 重要级 → 建议级
3. 仅修复问题部分，不做不必要的重构
4. 验证修复后问题已消除（anti_pattern_check.failures 不再包含相关项）

## 诊断输出格式
在 reasoning 中按以下格式输出诊断：
```
根因：#<编号> <反模式名称>
机制：<为什么这个反模式导致了当前问题>
修复：<具体修复措施>
验证：<如何确认修复有效>
```
```

### 5.4 Select — 技术推荐

```
你现在以【提示词工程专家】身份执行任务。

## 任务
为以下场景推荐最适合的提示技术，按复杂度递进排列。

## 输入
- 动作：select
- 问题：{task_description}
- 任务类型：{task_type}
- 失败模式（如有）：{failure_mode}

## 执行要求
1. 按 references/techniques-overview.md 的决策树确定技术路径
2. 按复杂度递进推荐：入门级 → 中级 → 高级 → 前沿
3. 每种技术说明：适用场景、预期效果、成本/复杂度代价
4. 给出明确的"从X开始，如不够则升级到Y"的实施路径
5. 引用 references/performance-benchmarks.md 的效果数据（如有）

## 输出格式
```yaml
result:
  prompt: ""
  technique: "<技术1> → <技术2> → <技术3>"
  structure: []
  anti_pattern_check:
    passed: []
    warnings: []
    failures: []
  reasoning: |
    <推荐逻辑>
    1. 基线方案：<技术1>，适用<场景>，预期<效果>
    2. 升级方案：<技术2>，适用<更高要求>，成本<代价>
    3. 极限方案：<技术3>，适用<最高要求>，成本<代价>
  model_adaptation: "<模型差异说明>"
  next_steps:
    - "第一步：<先用技术1验证基线>"
    - "第二步：<如不够升级到技术2>"
```
```

### 5.5 Template — 模板查找

```
你现在以【提示词工程专家】身份执行任务。

## 任务
从模板库中查找最匹配的提示词模板，并进行参数化定制。

## 输入
- 动作：template
- 需求描述：{task_description}
- 任务类型：{task_type}
- 输出格式：{output_format}

## 执行要求
1. 在 references/templates-library.md 中搜索匹配的模板
2. 展示完整模板内容
3. 说明所有 {{变量}} 的含义和推荐取值
4. 提供一个具体定制示例（替换变量为实际值）
5. 根据目标场景给出调整建议

## 输出格式
```yaml
result:
  prompt: |
    <参数化模板原文，保留{{变量}}标记>
  technique: "Template + Parameterization"
  structure: [<模板包含的元素>]
  anti_pattern_check:
    passed: [<模板已满足的项>]
    warnings: [<模板需要使用者补充的项，如escape_hatch>]
    failures: []
  reasoning: "<匹配理由和定制建议>"
  model_adaptation: ""
  next_steps:
    - "替换 {{变量1}} 为 <推荐值>"
    - "替换 {{变量2}} 为 <推荐值>"
    - "根据目标模型调整结构"
```
```

### 5.6 Review — 反模式审查

```
你现在以【提示词工程专家】身份执行任务。

## 任务
对以下提示词执行完整的15项反模式审查。

## 输入
- 动作：review
- 审查请求：{task_description}
- 待审查提示词：
```
{current_prompt}
```
- 目标模型：{model}
- 已知问题（如有）：{failure_mode}

## 执行要求
1. 按以下15项清单逐项检查，每项给出 通过/警告/失败 判定
2. 分级报告：致命级（#1,#7,#11,#14）→ 重要级（#2-#6,#9,#10,#12,#13）→ 建议级（#8,#15）
3. 每个非通过项给出具体修复建议
4. 按优先级排序修复建议（致命级优先）

## 反模式检查清单
致命级：
- #1 消息角色是否交替（user/assistant必须交替）
- #7 输出格式是否明确指定（JSON schema / XML tags / table）
- #11 是否包含"不知道"退出选项
- #14 用户查询是否放在提示词底部

重要级：
- #2 指令是否直接明确
- #3 是否要求去掉开场白
- #4 复杂推理是否分配了角色
- #5 是否用XML标签分隔数据与指令
- #6 提示词中是否有拼写/语法错误
- #9 思考过程是否可见（不能"思考但不输出"）
- #10 是否考虑了选项顺序偏差
- #12 是否先找证据再回答（反幻觉）
- #13 事实性任务temperature是否设为0

建议级：
- #8 是否用prefill控制输出开头
- #15 复查任务是否给了退出选项

## 输出格式
```yaml
result:
  prompt: ""
  technique: "Anti-Pattern Checklist (15 items)"
  structure: []
  anti_pattern_check:
    passed: [<通过项，格式"#编号 描述">]
    warnings: [<重要级+建议级问题>]
    failures: [<致命级问题>]
  reasoning: "<总体评价和关键问题概述>"
  model_adaptation: ""
  next_steps:
    - "[致命] 修复 #<编号>: <具体修复措施>"
    - "[重要] 修复 #<编号>: <具体修复措施>"
    - "[建议] 考虑 #<编号>: <改进建议>"
```
```

---

## 6. 参考文件加载策略

此 Skill 包含13个参考文件，**严禁一次性全部加载**。按以下规则按需加载：

| Action | 必须加载 | 按需加载 |
|--------|---------|---------|
| `design` | `techniques-overview.md`, `anthropic-tutorial.md` | `templates-library.md`（需模板时）, `performance-benchmarks.md`（需效果数据时） |
| `optimize` | `techniques-overview.md`, `anti-patterns.md` | `anthropic-tutorial.md`（需结构化参考时）, `performance-benchmarks.md` |
| `debug` | `anti-patterns.md`, `anthropic-tutorial.md` | `techniques-overview.md`（需重选技术时） |
| `select` | `techniques-overview.md` | `performance-benchmarks.md`（需效果数据时） |
| `template` | `templates-library.md` | 无 |
| `review` | `anti-patterns.md` | 无 |

**加载条件**：
- 当用户问题涉及具体技术细节时 → 加载对应的参考文件
- 当默认知识已足够回答时 → 不加载
- 每次最多加载3个参考文件（控制上下文长度）

---

## 7. 常见错误与纠正

| 错误做法 | 正确做法 | 原因 |
|----------|----------|------|
| 不指定 action 直接输出提示词 | 先判定 action 再执行工作流 | 不同 action 的处理逻辑完全不同 |
| 对所有任务都加 CoT | 分类/创意任务不需要 CoT | CoT 仅对推理任务有效，分类加 CoT 反而降低效果 |
| 不做反模式检查直接交付 | 必须执行 Step 4 反模式检查 | 致命级反模式会导致提示词完全不可用 |
| 一次性加载所有参考文件 | 按需加载，最多3个 | 控制上下文长度，避免信息过载 |
| output_format=json 时不给 schema | 必须给出具体的 JSON schema | "输出JSON" 太模糊，模型无法猜出结构 |
| 对 generic 模型不做适配 | generic 使用最保守策略 | 保守策略在所有模型上都安全 |
| review 只检查几项 | 必须完整检查15项 | 遗漏可能导致致命问题未被发现 |
| debug 时不提供 failure_mode | 至少推断一个 failure_mode | 不提供则诊断方向不明确 |
| 忽略模型差异 | 必须做模型适配 | Claude/GPT/开源模型的最佳实践差异很大 |
| select 只推荐一种技术 | 按复杂度递进推荐多种 | 用户可能不了解更高级的选项 |

---

## 8. 质量保障检查点

在输出结果前，执行以下最终检查：

```
□ action 是否正确匹配用户意图？
□ task_type 是否已明确（提供或推断）？
□ 输出 prompt 中是否包含退出选项（escape_hatch）？
□ 输出 prompt 中用户查询是否在底部？
□ 输出 prompt 中是否用XML标签分隔了数据和指令？
□ 输出格式是否明确指定（当 context.output_format 不为空时）？
□ anti_pattern_check.failures 是否为空？（design/optimize/debug/template 必须）
□ technique 字段是否与实际使用的技术一致？
□ reasoning 是否解释了选型逻辑？
□ model_adaptation 是否根据目标模型给出了适配说明？
□ next_steps 是否包含可执行的后续建议？
□ 参考文件是否按需加载（非一次性全加载）？
```

全部通过后方可输出结果。如有不通过项，返回 Step 4 补充修复。
