---
name: prompt-engineering
version: "2.2.0"
last_updated: "2026-05-06"
description: >
  提示词工程专家技能。当用户需要设计、优化、调试提示词(prompt)，选择提示技术，
  构建Agent提示系统，或解决提示词相关问题时触发。覆盖18种提示技术选型、OpenAI/Anthropic
  官方最佳实践、DSPy编程式优化、ToT思维树方法、100+生产级模板和反模式避坑清单。
  触发词：提示词、prompt、提示工程、提示优化、角色提示、少样本、思维链、CoT、
  ToT、DSPy、提示词模板、反模式、幻觉、Agent提示。
compatibility:
  min_codebuddy_version: "1.0"
  reference_files_version: "2.2.0"
  breaking_changes: []
---

# Prompt Engineering Expert Skill

## Purpose / 目标

Provide systematic prompt engineering knowledge and executable workflows for designing, optimizing, and debugging prompts. Transform ad-hoc prompt writing into a disciplined engineering process with technique selection, template application, and anti-pattern avoidance.

提供系统化的提示词工程知识和可执行工作流，用于设计、优化和调试提示词。将临时性的提示词编写转化为有纪律的工程过程，包含技术选型、模板应用和反模式规避。

## When to Use This Skill / 适用场景

- Designing new prompts for LLM applications or agents
- Optimizing existing prompts that produce unsatisfactory outputs
- Selecting appropriate prompting techniques for specific task types
- Building multi-step or agentic prompt systems
- Debugging prompt issues (hallucination, format mismatch, inconsistency)
- Converting natural language prompts to structured, production-ready format

## Action Examples (触发示例)

Each action type has characteristic user intents. Map the user's request to the closest action:

### `design` — 从零设计新提示词
- "帮我写一个客服机器人的提示词"
- "我需要一个代码审查的system prompt"
- "设计一个用于文档摘要的提示词"

### `optimize` — 优化现有提示词
- "这个提示词效果不好，帮我改进" + 提供当前提示词
- "我的prompt输出太啰嗦了"
- "优化这个提示词让它更准确"

### `debug` — 诊断和修复提示词问题
- "模型输出格式不对"
- "Claude总是在编造信息"
- "提示词有时有效有时无效"
- "输出不一致"

### `select` — 推荐提示技术
- "什么提示技术适合数学推理？"
- "CoT和ToT哪个好？"
- "如何选择提示词方法？"

### `template` — 查找和定制模板
- "有没有代码审查的提示词模板？"
- "给我一个Agent的提示词模板"
- "找一个SQL助手的prompt"

### `review` — 审查反模式
- "帮我检查这个提示词有什么问题" + 提供提示词
- "这个prompt有什么反模式？"
- "审查我的提示词质量"

## Core Workflow / 核心工作流

### Step 1: Task Analysis / 任务分析

Analyze the user's task along four dimensions to determine the optimal approach:

```
Input:
  task_description: str    — What the user wants to accomplish
  task_type: enum          — One of: classification | generation | reasoning | qa | code | agent | creative
  constraint_level: enum   — One of: strict (must follow exact format) | moderate | flexible
  failure_mode: enum|null  — Current problem if optimizing: hallucination | format_mismatch | inconsistency | verbosity | inaccuracy | null
```

Map `task_type` to recommended techniques using the decision tree in `references/techniques-overview.md`.

### Step 2: Technique Selection / 技术选型

Follow the decision tree in `references/techniques-overview.md` Section 1.3. Key mapping:

| Task Type | Primary Technique | Fallback |
|-----------|-------------------|----------|
| Simple classification | Zero-shot | Few-shot (2-3 examples) |
| Reasoning (single-step) | CoT ("Think step by step") | Role prompt + CoT |
| Reasoning (high confidence) | Self-Consistency | ToT |
| Reasoning (exploration needed) | ToT | ReAct |
| Tool-assisted reasoning | ReAct / PAL | DSPy ReAct |
| Knowledge-intensive QA | Generated Knowledge + citation | RAG |
| Prompt itself needs optimization | APE / DSPy | MIPROv2 |
| Agent system | ReAct + Reflexion | Agent governance framework |
| Format-critical output | Prefill + XML structure | Few-shot with format examples |

### Step 3: Prompt Construction / 提示词构建

Use the 10-element structure from `references/anthropic-tutorial.md` Section "Ch9". Order matters:

```
1. Personas (角色定义)
2. Task Context (任务描述)
3. Instructions (指令步骤) — use <instructions> XML tags
4. Examples (示例) — use <examples><example> XML tags, 2-5 examples
5. Input Data (输入数据) — use <document> or domain-specific XML tags
6. Formatting (格式要求) — specify exact output schema
7. Constraints (约束条件)
8. Tone/Style (语气风格)
9. Escape Hatches (退出条件) — always include "I don't know" option
10. User Query (用户查询) — place LAST for maximum attention
```

### Step 4: Anti-Pattern Check / 反模式检查

Before finalizing, verify against the 15-item checklist below (detailed version in `references/anti-patterns.md`):

**致命级（必须修复）：**
- [ ] #1 消息角色是否交替（user/assistant必须交替，不能连续两条user）
- [ ] #7 输出格式是否明确指定（JSON schema / XML tags / table）
- [ ] #11 是否包含"不知道"退出选项（"If unsure, say I'm not sure"）
- [ ] #14 用户查询是否放在提示词底部（LLM对末尾注意力最强）

**重要级（优先修复）：**
- [ ] #5 是否用XML标签分隔数据与指令
- [ ] #4 复杂推理是否分配了角色
- [ ] #9 思考过程是否可见（不能"思考但不输出"）
- [ ] #12 是否先找证据再回答（反幻觉）
- [ ] #13 事实性任务temperature是否设为0
- [ ] #10 是否考虑了选项顺序偏差
- [ ] #2 指令是否直接明确
- [ ] #3 是否要求去掉开场白
- [ ] #6 提示词中是否有拼写/语法错误

**建议级（锦上添花）：**
- [ ] #8 是否用prefill控制输出开头
- [ ] #15 复查任务是否给了退出选项

### Step 5: Optimization / 优化（如需要）

For prompts that need systematic optimization rather than manual tuning, refer to `references/dspy-framework.md`:

- **<10 samples**: Use `BootstrapFewShot` — auto-search few-shot examples
- **50+ samples**: Use `BootstrapFewShotWithRandomSearch` — random search + bootstrap
- **Instruction optimization**: Use `MIPROv2` (0-shot mode)
- **200+ samples**: Use `MIPROv2` (full mode) — optimize both instructions and examples
- **Production efficiency**: Use `BootstrapFinetune` — distill to weight updates

## Model Adaptation Strategy / 模型适配策略

Different models have different prompt design characteristics. Adapt accordingly:

| Model Family | Key Differences | Adaptation |
|-------------|-----------------|------------|
| **Claude (3.5/3.7 Sonnet)** | XML标签高度敏感；Prefill极其有效；选项顺序偏差明显 | 优先用XML结构化；善用prefill；交换选项测试 |
| **GPT-4/4o/4.1** | 系统消息权重高；JSON mode可靠；函数调用规范 | 重要指令放system；用JSON mode约束格式 |
| **DeepSeek-R1** | 自生长CoT；不需要显式"think step by step"；reasoning_content独立 | 不要强制CoT指令；利用其自发推理能力 |
| **开源模型 (Llama/Qwen)** | 指令遵循能力弱于闭源；对格式敏感；few-shot更有效 | 提供更多示例；指令更明确更短；避免复杂嵌套结构 |
| **通用适配** | 不确定模型时 | 使用最保守策略：XML标签 + few-shot + 明确格式 + temperature=0 |

## Error Handling & Degradation / 错误处理与降级策略

When the ideal approach isn't available, follow these degradation paths:

| 理想方案不可用 | 降级方案 | 说明 |
|--------------|---------|------|
| ToT（太复杂/太贵） | Self-Consistency (3-5次CoT+投票) | 保留多路径评估，降低成本 |
| DSPy优化（无训练数据） | 手动Few-Shot + APE指令搜索 | 人工构造3-5示例+指令迭代 |
| Prefill（API不支持） | 明确格式指令 + Few-Shot示例 | 用示例替代prefill的格式引导 |
| XML标签（输出被干扰） | Markdown标题+代码块分隔 | 降低结构化程度但保持清晰 |
| 多步管道（延迟太高） | 合并为单次提示 + CoT | 牺牲精度换速度 |
| 角色提示（输出过于角色化） | 弱化角色为"专业视角" | "Think from the perspective of an expert" 替代 "You are an expert" |

## Input/Output Specifications / 输入输出规范

### Input Schema

```yaml
action:
  type: enum
  required: true
  values:
    - design       # Design a new prompt from scratch
    - optimize     # Optimize an existing prompt
    - debug        # Diagnose and fix prompt issues
    - select       # Recommend technique(s) for a task
    - template     # Find and customize a template
    - review       # Review prompt for anti-patterns

task_description:
  type: str
  required: true
  description: What the prompt should accomplish

current_prompt:
  type: str
  required: false  # Required for optimize/debug/review actions
  description: The existing prompt to improve or analyze

context:
  type: object
  required: false
  properties:
    model:
      type: enum
      values: [gpt-4, gpt-4o, gpt-4.1, claude-3.5-sonnet, claude-3.7-sonnet, deepseek-r1, llama, qwen, generic]
      default: generic
    task_type:
      type: enum
      values: [classification, generation, reasoning, qa, code, agent, creative, multi_step]
      default: null       # Auto-detect if not provided
    output_format:
      type: enum
      values: [json, xml, markdown, table, code, natural_language]
      default: null
    failure_mode:
      type: enum
      values: [hallucination, format_mismatch, inconsistency, verbosity, inaccuracy, refusal, null]
      default: null
    language:
      type: enum
      values: [zh, en, bilingual]
      default: zh
    complexity:
      type: enum
      values: [simple, moderate, complex, production]
      default: moderate
```

### Output Schema

```yaml
result:
  type: object
  properties:
    prompt:
      type: str
      description: The constructed or optimized prompt (full text)
    technique:
      type: str
      description: Primary technique applied (e.g., "CoT + Few-Shot + Prefill")
    structure:
      type: list
      description: Ordered list of prompt elements used
      items:
        type: str  # e.g., ["persona", "instructions", "examples", "format", "escape_hatch"]
    anti_pattern_check:
      type: object
      description: Checklist results
      properties:
        passed: list[str]    # Items that pass
        warnings: list[str]  # Items with warnings
        failures: list[str]  # Items that fail (must fix)
    reasoning:
      type: str
      description: Why this technique/structure was chosen
    model_adaptation:
      type: str
      description: Model-specific adaptations applied (if any)
    next_steps:
      type: list[str]
      description: Suggested follow-up actions (e.g., "Test with temperature=0", "Add DSPy optimization")
```

### Complete Input/Output Example

**Input (design action):**
```yaml
action: design
task_description: "构建一个客服工单分类Agent，将用户反馈分为：Bug、Feature Request、Question、Complaint，并提取优先级和关键信息"
context:
  model: claude-3.7-sonnet
  task_type: classification
  output_format: json
  language: zh
  complexity: production
```

**Output:**
```yaml
result:
  prompt: |
    你是一位专业的客服工单分析专家，擅长从用户反馈中精准提取关键信息并进行分类。

    <instructions>
    1. 仔细阅读用户反馈内容
    2. 在 <analysis> 标签中，分析反馈的核心诉求、情绪倾向和紧急程度
    3. 在 <output> 标签中，按指定JSON格式输出分类结果
    </instructions>

    <classification_categories>
    - Bug: 用户报告的功能故障、错误行为、崩溃问题
    - Feature Request: 用户提出的新功能需求或改进建议
    - Question: 用户对产品功能的咨询或使用疑问
    - Complaint: 用户表达不满、投诉或要求赔偿
    </classification_categories>

    <priority_criteria>
    - P0_Critical: 安全问题、数据丢失、服务完全不可用
    - P1_High: 核心功能受损、大量用户受影响
    - P2_Medium: 次要功能异常、少量用户受影响
    - P3_Low: 界面瑕疵、体验优化建议
    </priority_criteria>

    <examples>
    <example>
    <feedback>你们的支付功能又坏了！我客户付不了款，这已经是第三次了！</feedback>
    <output>{"category": "Bug", "priority": "P1_High", "key_info": "支付功能故障，重复发生，影响客户付款", "sentiment": "angry", "suggested_action": "立即转交支付团队紧急修复"}</output>
    </example>
    <example>
    <feedback>能不能加一个批量导出功能？每次一个个导出太麻烦了</feedback>
    <output>{"category": "Feature Request", "priority": "P2_Medium", "key_info": "需要批量导出功能，当前效率低", "sentiment": "neutral", "suggested_action": "加入产品需求池评估"}</output>
    </example>
    </examples>

    <feedback>
    {{USER_FEEDBACK}}
    </feedback>

    如果反馈内容不足以确定分类，请在 output 中将 category 设为 "Unclear"，并在 key_info 中说明缺少什么信息。
  technique: "Role Prompt + Few-Shot + XML Structure + Escape Hatch"
  structure: ["persona", "instructions", "classification_rules", "priority_criteria", "examples", "input_data", "escape_hatch"]
  anti_pattern_check:
    passed: ["#1 消息交替", "#5 XML标签分隔", "#7 输出格式明确", "#11 退出选项", "#14 查询在底部"]
    warnings: ["#8 考虑添加prefill引导JSON输出: assistant prefill = '{'"]
    failures: []
  reasoning: "客服工单分类是典型分类任务，zero-shot即可，但生产环境加few-shot确保格式稳定。XML标签分离分类规则与输入数据，避免混淆。Claude 3.7对XML和prefill敏感，适配效果最佳。"
  model_adaptation: "Claude 3.7: 使用XML标签结构化（而非Markdown），建议prefill '{' 强制JSON输出"
  next_steps:
    - "测试：提交10条真实工单验证分类准确率"
    - "优化：如准确率<95%，考虑添加更多边界case示例"
    - "增强：如需处理多语言反馈，添加语言检测步骤"
```

## Core Parameters / 核心参数

### technique_selector

Maps task characteristics to prompting techniques. Values from `references/techniques-overview.md`.

```yaml
parameters:
  complexity:
    type: enum
    values: [entry, intermediate, advanced, frontier]
    description: Prompting technique complexity tier

  reasoning_depth:
    type: enum
    values: [none, single_step, multi_step, tree_search, graph]
    description: Required reasoning depth

  automation_level:
    type: enum
    values: [manual, semi_auto, full_auto]
    description: How much of the prompt should be auto-optimized

  risk_tolerance:
    type: enum
    values: [high, medium, low]
    description: Tolerance for hallucination/incorrect output
    note: Low → use Self-Consistency or ToT; High → standard CoT sufficient
```

### format_controller

Controls output format specifications.

```yaml
parameters:
  output_schema:
    type: str
    description: JSON schema, XML tag structure, or format description
    example: |
      {
        "analysis": {"summary": "str", "key_points": ["str"]},
        "decision": {"action": "enum[ESCALATE,NORMAL]", "confidence": "float"}
      }

  prefill:
    type: str
    description: Text to prefill in assistant message (controls output start)
    examples:
      - "{"          # Force JSON output
      - "| Col |"    # Force table output
      - "```python"  # Force code block
      - "根据分析"    # Force Chinese output

  xml_tags:
    type: list[object]
    description: XML tag definitions for structured output
    item_schema:
      tag: str           # Tag name
      content_type: str  # Expected content type
      required: bool     # Whether this tag is mandatory
```

### hallucination_guard

Anti-hallucination configuration for factual tasks.

```yaml
parameters:
  grounding:
    type: enum
    values: [strict, moderate, none]
    description: How strictly to ground answers in provided context
    strict: "Answer ONLY from provided document. If not found, say 'Not enough information.'"
    moderate: "Prefer provided context but use general knowledge if clearly needed."
    none: "No grounding constraint."

  citation:
    type: bool
    default: false
    description: Require source citations for claims

  confidence_reporting:
    type: enum
    values: [none, simple, detailed]
    none: No confidence reporting
    simple: "Append confidence level: HIGH/MEDIUM/LOW"
    detailed: "Rate confidence 1-5 and explain what information is missing if < 3"

  temperature:
    type: float
    description: Model temperature setting
    default: 0.0  # For factual tasks
    note: Set to 0.0 for factual/analytical tasks; 0.7+ for creative tasks
```

## Reference Files / 参考文件（渐进式加载）

Load reference files ONLY when the relevant topic arises — do not load all at once.

**核心文件（大多数场景需要）：**

| File | When to Load / 加载时机 | Content / 内容 |
|------|------------------------|----------------|
| `references/techniques-overview.md` | Task analysis or technique selection / 任务分析或技术选型 | 18 techniques matrix, decision tree, classification |
| `references/anti-patterns.md` | Debugging or reviewing prompts / 调试或审查提示词 | 15 anti-patterns with severity and fixes |
| `references/anthropic-tutorial.md` | Building structured prompts or debugging / 构建结构化提示词 | 9-chapter tutorial, 10-element structure, prefill, anti-hallucination |

**扩展文件（按需加载）：**

| File | When to Load / 加载时机 | Content / 内容 |
|------|------------------------|----------------|
| `references/openai-guide.md` | Following OpenAI best practices / 遵循OpenAI最佳实践 | 6 strategies, 18 tactics, complete examples |
| `references/dspy-framework.md` | Programmatic prompt optimization / 编程式提示词优化 | Signature, Module, Optimizer, ChatAdapter format |
| `references/tot-method.md` | Complex reasoning with search / 复杂推理搜索 | ToT operations, BFS/DFS, 3 task templates |
| `references/templates-library.md` | Need production-ready templates / 需要生产级模板 | 100+ templates with `{{variable}}` interpolation, indexed by 20 categories |
| `references/resources-index.md` | Research or tool recommendations / 研究或工具推荐 | Papers, tools, courses, books, models |
| `references/chinese-ecosystem.md` | Chinese-language prompt resources / 中文提示词资源 | Chinese prompt communities, tools, guides |
| `references/source-paths.md` | Trace back to original sources / 追溯原始来源 | All local repository paths |
| `references/test-suite.md` | Validating skill behavior / 验证技能行为 | 21 test cases for 6 actions with acceptance criteria |
| `references/performance-benchmarks.md` | Technique effectiveness data / 技术效果数据 | Model×technique benchmarks, cost analysis, selection matrix |
| `references/integration-examples.md` | Integrating Skill in code / 代码集成调用 | Data models, engine, 6-action examples, batch/pipeline, error handling, LLM API integration, CLI |
| `references/invocation-specification.md` | Calling this Skill accurately / 精确调用此Skill | Trigger rules, input params, workflow, output format, 6 action instruction templates, error correction, quality checklist |
| `references/installation-guide.md` | Installing in other tools / 在其他工具中安装 | 6 platforms (CodeBuddy/Cursor/Claude Desktop/ChatGPT/Generic LLM/Python), verification, one-click deploy |
| `references/skillhub-upload-guide.md` | Uploading to SkillHub / 上传到SkillHub | 3 platforms (SkillHub/ClawHub/Private), metadata, packaging, review, versioning, error solutions, pack script |

## Version Compatibility Matrix / 版本兼容性矩阵

| Component | Version | Compatible With | Breaking Changes |
|-----------|---------|-----------------|------------------|
| SKILL.md | 2.2.0 | All reference files v2.2.0 | None from v2.1.0 |
| techniques-overview.md | 2.2.0 | SKILL.md v2.1.0+ | None |
| openai-guide.md | 2.2.0 | SKILL.md v2.1.0+ | None |
| anthropic-tutorial.md | 2.2.0 | SKILL.md v2.1.0+ | None |
| dspy-framework.md | 2.2.0 | SKILL.md v2.1.0+ | None |
| tot-method.md | 2.2.0 | SKILL.md v2.1.0+ | None |
| templates-library.md | 2.2.0 | SKILL.md v2.2.0+ | Added `{{variable}}` interpolation (v2.1.0 templates still work) |
| anti-patterns.md | 2.2.0 | SKILL.md v2.1.0+ | None |
| resources-index.md | 2.2.0 | SKILL.md v2.1.0+ | None |
| chinese-ecosystem.md | 2.2.0 | SKILL.md v2.1.0+ | None |
| source-paths.md | 2.2.0 | SKILL.md v2.1.0+ | None |
| test-suite.md | 2.2.0 | SKILL.md v2.2.0+ | New file |
| performance-benchmarks.md | 2.2.0 | SKILL.md v2.2.0+ | New file |
| integration-examples.md | 2.2.0 | SKILL.md v2.2.0+ | New file |
| invocation-specification.md | 2.2.0 | SKILL.md v2.2.0+ | New file |
| installation-guide.md | 2.2.0 | SKILL.md v2.2.0+ | New file |
| skillhub-upload-guide.md | 2.2.0 | SKILL.md v2.2.0+ | New file |

**升级指南：**
- v2.1.0 → v2.2.0：无破坏性变更。新增 `test-suite.md` 和 `performance-benchmarks.md`，`templates-library.md` 新增参数化支持（旧模板格式仍兼容）。
- v2.0.0 → v2.1.0：补全8个缺失参考文件，新增 action 触发示例、模型适配策略、降级方案。

## Execution Rules / 执行规则

1. **Always start with task analysis** — Classify the task type before jumping to solution
2. **Prefer structured prompts** — Use XML tags, explicit formatting, and the 10-element structure over free-form prompts
3. **Include escape hatches** — Every prompt must have a "I don't know" / "Not enough information" path
4. **Place user query at bottom** — Claude/LLMs attend most to prompt-end content
5. **Check anti-patterns before delivering** — Run through the 15-item checklist (致命级 first)
6. **Specify concrete output format** — Never leave output format ambiguous when programmatic extraction is needed
7. **Match technique to complexity** — Don't use ToT for simple classification; don't use zero-shot for complex reasoning
8. **Adapt to target model** — Use model-specific strategies from the Model Adaptation table
9. **Degrade gracefully** — If ideal approach isn't available, follow the degradation paths
10. **Reference the source** — When using templates or techniques, cite which reference file they come from
11. **Language alignment** — Match prompt language to user's language; use prefill to enforce output language
12. **Iterate with data** — For production prompts, recommend DSPy optimization over manual tuning when sample data is available
