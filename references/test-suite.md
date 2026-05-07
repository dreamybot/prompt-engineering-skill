# 自动化测试用例套件

> 用途：验证 Skill 的6种 action 是否按预期工作。每个测试包含输入、预期行为和验收标准。

## 测试总览

| Action | 测试数 | 覆盖场景 |
|--------|--------|----------|
| design | 4 | 分类/推理/Agent/创意 |
| optimize | 4 | 啰嗦/幻觉/格式/不一致 |
| debug | 4 | 幻觉/格式错误/不一致/拒答 |
| select | 3 | 推理/知识/Agent |
| template | 3 | 代码审查/SQL/Agent |
| review | 3 | 致命反模式/重要反模式/综合审查 |

---

## 1. design（从零设计）

### Test 1.1：分类任务提示词设计

```yaml
input:
  action: design
  task_description: "构建一个邮件分类系统，将收到的邮件分为：垃圾邮件、工作邮件、个人邮件、通知邮件"
  context:
    model: claude-3.7-sonnet
    task_type: classification
    output_format: json
    language: zh
    complexity: production

expected_behavior:
  - 技术选择：Role Prompt + Few-Shot（分类任务首选零样本，但生产级加few-shot保稳）
  - 必须包含：XML标签分隔指令与数据、JSON输出格式明确指定、退出选项
  - 10元素结构至少包含：persona, instructions, examples, format, escape_hatch
  - anti_pattern_check 的 failures 列表必须为空

acceptance_criteria:
  - 输出 prompt 中有 <instructions> XML标签
  - 输出 prompt 中有 <examples> XML标签
  - 输出 prompt 包含 JSON schema 定义
  - technique 字段包含 "Role Prompt" 或 "Few-Shot"
  - anti_pattern_check.failures == []
  - structure 列表包含 "escape_hatch"
```

### Test 1.2：推理任务提示词设计

```yaml
input:
  action: design
  task_description: "设计一个逻辑推理助手，能解答类似LSAT的逻辑推理题"
  context:
    model: gpt-4.1
    task_type: reasoning
    output_format: natural_language
    language: zh
    complexity: complex

expected_behavior:
  - 技术选择：CoT + Role Prompt（推理任务首选CoT）
  - 必须包含：角色分配（逻辑专家）、逐步思考指令、思考过程可见
  - 用户查询放在提示词底部

acceptance_criteria:
  - technique 字段包含 "CoT" 或 "Chain-of-Thought"
  - prompt 中有角色定义（如 "你是一位逻辑推理专家"）
  - prompt 中有逐步思考指令（"Think step by step" 或 "逐步分析"）
  - structure 列表包含 "persona"
  - anti_pattern_check 中 #14(查询位置) 通过
  - anti_pattern_check 中 #9(思考可见) 通过
```

### Test 1.3：Agent系统提示词设计

```yaml
input:
  action: design
  task_description: "设计一个客服Agent的system prompt，需要能查询订单、处理退款、转接人工客服"
  context:
    model: claude-3.5-sonnet
    task_type: agent
    output_format: json
    language: zh
    complexity: production

expected_behavior:
  - 技术选择：ReAct + Agent Governance（Agent系统需要推理+行动框架）
  - 必须包含：工具调用格式定义、多步推理结构、安全边界
  - 退出条件包含转人工路径

acceptance_criteria:
  - technique 字段包含 "ReAct"
  - prompt 中有工具定义或函数调用格式
  - prompt 中有安全边界或约束条件
  - structure 列表包含 "escape_hatch"
  - escape_hatch 包含转人工选项
  - model_adaptation 字段提及 Claude 适配策略
```

### Test 1.4：创意写作提示词设计

```yaml
input:
  action: design
  task_description: "设计一个科幻短篇小说写作助手，需要能生成有深度的故事大纲和角色设定"
  context:
    model: generic
    task_type: creative
    output_format: markdown
    language: zh
    complexity: moderate

expected_behavior:
  - 技术选择：Role Prompt + Directional Stimulus（创意任务用角色+方向引导）
  - 不应使用 CoT（创意任务不需要逐步推理）
  - temperature 应建议 > 0.7
  - 格式灵活，不需要严格JSON

acceptance_criteria:
  - technique 字段不包含 "CoT"
  - technique 字段包含 "Role Prompt"
  - prompt 中有角色定义（如科幻作家）
  - next_steps 中有关于 temperature 的建议
```

---

## 2. optimize（优化现有提示词）

### Test 2.1：优化啰嗦的提示词

```yaml
input:
  action: optimize
  task_description: "这个提示词输出太啰嗦，需要更简洁"
  current_prompt: |
    你是一个帮助别人的助手，用户会问你问题，你需要回答他们的问题。
    回答的时候请尽量详细一些，把你想到的都写出来，不要遗漏任何细节。
    如果你觉得答案很长，那也没关系，继续写就好了。
    你的回答应该尽可能的长和全面。
  context:
    model: generic
    failure_mode: verbosity

expected_behavior:
  - 识别反模式：不直接说要什么、不要求去前言
  - 修复：明确长度限制、加 "Skip the preamble"、指定具体格式
  - technique 应提及 Direct Instruction

acceptance_criteria:
  - 优化后 prompt 中有明确的长度限制（如 "3段以内" 或 "不超过300字"）
  - 优化后 prompt 中有 "Skip the preamble" 或 "去掉开场白"
  - anti_pattern_check 中 #3(去前言) 改善
  - reasoning 字段解释了为什么原提示词导致啰嗦
```

### Test 2.2：优化有幻觉问题的提示词

```yaml
input:
  action: optimize
  task_description: "模型在回答文档问题时经常编造信息"
  current_prompt: |
    根据以下内容回答用户的问题：
    {document}
    问题：{question}
  context:
    model: claude-3.7-sonnet
    failure_mode: hallucination

expected_behavior:
  - 识别反模式：不给"不知道"选项、不让先找证据、无温度控制
  - 修复：加退出选项、加引用要求、建议temperature=0、加XML标签
  - 引用 anti-patterns.md 的 #11, #12, #13

acceptance_criteria:
  - 优化后 prompt 中有 "不知道" 或 "I'm not sure" 退出选项
  - 优化后 prompt 中有先引用后回答的指令
  - anti_pattern_check.failures 原始包含 #11, #12；优化后这些通过
  - next_steps 中有 "设置 temperature=0" 建议
  - prompt 中有 <document> XML标签
```

### Test 2.3：优化格式不稳定的提示词

```yaml
input:
  action: optimize
  task_description: "模型输出格式不统一，有时返回JSON有时返回纯文本"
  current_prompt: |
    分析用户反馈并返回分类结果。
  context:
    model: gpt-4o
    failure_mode: format_mismatch
    output_format: json

expected_behavior:
  - 识别反模式：#7 不指定输出格式
  - 修复：明确JSON schema、加prefill引导、加few-shot示例
  - GPT-4o 适配：使用 JSON mode

acceptance_criteria:
  - 优化后 prompt 中有明确的 JSON schema 定义
  - technique 包含 "Prefill" 或 "Few-Shot"
  - anti_pattern_check 中 #7 通过
  - model_adaptation 提及 GPT-4o JSON mode
```

### Test 2.4：优化输出不一致的提示词

```yaml
input:
  action: optimize
  task_description: "同一个提示词每次执行结果差异很大"
  current_prompt: |
    帮我写一段产品介绍，要有吸引力。
  context:
    model: generic
    failure_mode: inconsistency
    task_type: creative

expected_behavior:
  - 识别问题：缺少具体约束、无格式定义、创意任务但也需锚点
  - 修复：加 few-shot 示例锚定风格、加结构化格式、明确tone/audience

acceptance_criteria:
  - 优化后 prompt 中有 few-shot 示例或风格参考
  - 优化后 prompt 中有明确的 tone/audience 定义
  - reasoning 字段解释了不一致的根因
  - next_steps 中有测试建议（多次运行验证一致性）
```

---

## 3. debug（诊断修复）

### Test 3.1：诊断幻觉问题

```yaml
input:
  action: debug
  task_description: "Claude总是在编造信息"
  current_prompt: |
    你是一位知识渊博的助手。请回答用户的问题。
    问题：{question}
  context:
    model: claude-3.7-sonnet
    failure_mode: hallucination

expected_behavior:
  - 诊断出：缺少 grounding 约束、无退出选项、查询位置不佳
  - 反模式命中：#11(无不知道选项), #12(不先找证据), #14(查询在顶部)
  - 给出分优先级的修复建议

acceptance_criteria:
  - anti_pattern_check.failures 包含 #11
  - anti_pattern_check.failures 包含 #12 或 #12 在 warnings 中
  - 给出至少3条分优先级的修复建议
  - 修复后的 prompt 包含 grounding 指令和退出选项
```

### Test 3.2：诊断格式错误

```yaml
input:
  action: debug
  task_description: "模型输出格式不对，需要JSON但返回了Markdown"
  current_prompt: |
    请分析这封邮件的意图和紧急程度。
    邮件内容：{email}
  context:
    model: gpt-4o
    failure_mode: format_mismatch
    output_format: json

expected_behavior:
  - 诊断出：#7(不指定输出格式)
  - 修复：加JSON schema + prefill + few-shot
  - GPT-4o适配：使用 JSON mode

acceptance_criteria:
  - anti_pattern_check.failures 包含 #7
  - 修复建议中包含 JSON schema 定义
  - 修复建议中提及 prefill 或 JSON mode
  - model_adaptation 字段提及 GPT-4o 策略
```

### Test 3.3：诊断不一致问题

```yaml
input:
  action: debug
  task_description: "提示词有时有效有时无效，结果不稳定"
  current_prompt: |
    分析用户反馈的情感。
    {feedback}
  context:
    model: generic
    failure_mode: inconsistency

expected_behavior:
  - 诊断出：缺少 few-shot、格式定义不明确、无约束条件
  - 建议添加 few-shot 示例锚定行为、明确分类标准和格式

acceptance_criteria:
  - 修复建议中包含添加 few-shot 示例
  - 修复建议中包含明确输出格式
  - reasoning 字段分析了不一致的根因
  - next_steps 中有验证一致性的方法
```

### Test 3.4：诊断拒答问题

```yaml
input:
  action: debug
  task_description: "模型经常拒绝回答合理的请求"
  current_prompt: |
    你是一位安全的助手。你必须确保所有回答都是安全无害的。
    回答用户问题：{question}
  context:
    model: generic
    failure_mode: refusal

expected_behavior:
  - 诊断出：安全约束过于笼统、缺少边界定义
  - 修复：明确安全边界（什么可以回答）、加分级处理策略
  - 不是移除安全约束，而是精确定义

acceptance_criteria:
  - 修复建议不是简单移除安全约束
  - 修复后的 prompt 有更精确的安全边界定义
  - 修复后的 prompt 有分级处理策略（如 "如果不在安全范围内，解释原因并建议替代方案"）
  - 保留 escape_hatch
```

---

## 4. select（技术推荐）

### Test 4.1：数学推理技术推荐

```yaml
input:
  action: select
  task_description: "什么提示技术适合数学推理？"
  context:
    task_type: reasoning

expected_behavior:
  - 推荐路径：CoT → Self-Consistency → PAL
  - 按复杂度递进推荐
  - 给出每种技术的适用场景和论文引用

acceptance_criteria:
  - 推荐列表包含 "CoT" 或 "Chain-of-Thought"
  - 推荐列表包含 "Self-Consistency" 或 "PAL"
  - 每种技术有适用场景说明
  - reasoning 字段解释了推荐顺序
```

### Test 4.2：知识密集型QA技术推荐

```yaml
input:
  action: select
  task_description: "需要让模型基于文档回答问题，不能编造信息"
  context:
    task_type: qa
    failure_mode: hallucination

expected_behavior:
  - 推荐路径：Generated Knowledge + citation → RAG
  - 强调反幻觉技术：grounding、先引用后回答、退出选项
  - 引用 anthropic-tutorial.md Ch8 的反幻觉技术

acceptance_criteria:
  - 推荐列表包含 grounding 或引用验证相关技术
  - 推荐列表包含 "Generated Knowledge" 或 "RAG"
  - 包含反幻觉的具体提示策略
```

### Test 4.3：Agent系统技术推荐

```yaml
input:
  action: select
  task_description: "构建一个能调用工具的自主Agent"
  context:
    task_type: agent

expected_behavior:
  - 推荐路径：ReAct → Reflexion → Agent Governance
  - 提及 DSPy.ReAct 的编程式方案
  - 提及多Agent治理框架

acceptance_criteria:
  - 推荐列表包含 "ReAct"
  - 推荐列表包含 "Reflexion" 或 "Agent Governance"
  - reasoning 字段解释了从单Agent到治理框架的进阶逻辑
```

---

## 5. template（模板查找）

### Test 5.1：代码审查模板

```yaml
input:
  action: template
  task_description: "有没有代码审查的提示词模板？需要安全审计功能"
  context:
    task_type: code
    language: zh

expected_behavior:
  - 匹配到 Code Reviewer Security 模板
  - 展示模板内容并说明参数化方式
  - 如需中文版本，提供翻译适配

acceptance_criteria:
  - 返回的模板是安全代码审查类
  - 模板包含 OWASP 分类和安全检查项
  - 提供模板定制建议
```

### Test 5.2：SQL助手模板

```yaml
input:
  action: template
  task_description: "找一个SQL助手的prompt"
  context:
    task_type: code
    output_format: code

expected_behavior:
  - 匹配到 SQL Assistant 模板
  - 展示完整模板
  - 建议添加特定数据库方言的约束

acceptance_criteria:
  - 返回的模板是 SQL 助手类
  - 模板包含 CTE、窗口函数等最佳实践
  - next_steps 中有定制建议
```

### Test 5.3：Agent模板

```yaml
input:
  action: template
  task_description: "给我一个Agent的提示词模板"
  context:
    task_type: agent
    complexity: production

expected_behavior:
  - 匹配到 Agentic Coder 或 Agent Skill Designer 模板
  - 展示完整模板和参数化说明
  - 推荐结合 ReAct 框架使用

acceptance_criteria:
  - 返回的模板是 Agent 系统类
  - 模板包含核心原则和安全检查项
  - 提供与 ReAct 框架结合的建议
```

---

## 6. review（反模式审查）

### Test 6.1：致命级反模式审查

```yaml
input:
  action: review
  task_description: "帮我检查这个提示词有什么问题"
  current_prompt: |
    帮我分析一下这个数据。

    这是数据：{{data}}

    请分析数据中的趋势。
  context:
    model: claude-3.7-sonnet

expected_behavior:
  - 检出致命级：#7(不指定输出格式)、#11(无不知道选项)
  - 检出重要级：#5(应用XML标签分隔)、#14(查询位置——但此处已是底部)
  - 分级报告：致命→重要→建议

acceptance_criteria:
  - anti_pattern_check.failures 包含 #7
  - anti_pattern_check.failures 包含 #11
  - failures 列表仅包含致命级项
  - warnings 列表包含重要级项
  - 给出每个反模式的具体修复建议
```

### Test 6.2：重要级反模式审查

```yaml
input:
  action: review
  task_description: "这个prompt有什么反模式？"
  current_prompt: |
    你是一位助手。用户会问你数学问题，请你思考后回答。
    但是思考过程不要显示出来，直接给答案就好。
    问题：{question}
  context:
    model: generic

expected_behavior:
  - 检出：#9(思考但不输出)、#4(推理任务不给角色)
  - 解释为什么"思考但不输出"是反模式

acceptance_criteria:
  - anti_pattern_check 中 #9 标记为 failure 或 warning
  - anti_pattern_check 中 #4 标记为 warning
  - reasoning 字段解释了"思考必须可见"的原理
  - 修复建议不是简单移除"不要显示"指令，而是改为标签化输出
```

### Test 6.3：综合审查

```yaml
input:
  action: review
  task_description: "审查我的提示词质量"
  current_prompt: |
    请帮我回答这个问题。如果你知道答案，请告诉我。
    如果不知道，也没关系，可以猜一下。
    问题是：{question}
  context:
    model: generic
    failure_mode: hallucination

expected_behavior:
  - 检出：#11(猜一下=鼓励幻觉)、#7(无输出格式)、#2(不直接)
  - 综合15项清单逐项检查
  - 给出优先修复顺序

acceptance_criteria:
  - anti_pattern_check 中 #11 标记为 failure
  - reasoning 字段解释了"猜一下"如何导致幻觉
  - 修复后的 prompt 移除了"猜一下"并替换为安全的退出选项
  - 至少检查10项以上的反模式
```

---

## 回归测试检查清单

每次修改 Skill 后，运行以下最小回归测试：

1. **design action 可执行**：Test 1.1 通过 → 5步工作流完整
2. **anti-pattern check 有效**：Test 6.1 通过 → 致命级检出准确
3. **model adaptation 触发**：Test 1.3 通过 → Claude/GPT 适配生效
4. **参考文件可加载**：Test 4.1 通过 → techniques-overview.md 引用有效
5. **降级策略触发**：删除 ToT 推荐，验证 Self-Consistency 降级是否生效
