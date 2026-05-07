# Anthropic 交互式教程深度精要

> 来源：`d:\Prompt\repos\anthropics_prompt-eng-interactive-tutorial\Anthropic 1P\`

## 课程架构

| 难度 | 章节 | 核心技能 |
|------|------|----------|
| 入门 | Ch1 基本结构 / Ch2 清晰直接 / Ch3 角色分配 | API调用、指令清晰度、角色提示 |
| 中级 | Ch4 分离数据与指令 / Ch5 格式化输出 / Ch6 逐步思考 / Ch7 少样本提示 | XML标签、Prefill、CoT、Few-Shot |
| 高级 | Ch8 避免幻觉 / Ch9 复杂提示词构建 | 反幻觉技术、综合模板 |
| 专家 | 附录: 链式提示 / 工具使用 / 搜索检索 | 多步管道、工具集成、RAG |

## Ch1: 基本提示词结构

**API调用模板：**
```python
import anthropic

client = anthropic.Anthropic(api_key=API_KEY)

def get_completion(prompt: str, system_prompt=""):
    message = client.messages.create(
        model=MODEL_NAME,
        max_tokens=2000,
        temperature=0.0,
        system=system_prompt,
        messages=[{"role": "user", "content": prompt}]
    )
    return message.content[0].text
```

**反模式：** ❌ 连续两条user消息 ❌ 不以user角色开头 ❌ 在messages中混入system消息

## Ch2: 清晰直接

> 黄金法则：把提示词给同事看，如果他们困惑，Claude也会困惑。

| 意图 | 模糊 ❌ | 清晰 ✅ |
|------|---------|---------|
| 不要前言 | "Write a haiku about robots" | "Write a haiku about robots. Skip the preamble; go straight into the poem." |
| 做出选择 | "Who is the best basketball player?" | "Who is the best basketball player? If you absolutely had to pick one, who would it be?" |
| 更长输出 | "Tell me about X" | "Write a detailed, multi-paragraph explanation of X." |
| 特定格式 | "List the pros and cons" | "Create a markdown table with two columns: 'Pros' and 'Cons'." |

**反模式：** ❌ 不直接说出你要什么 ❌ 不要求去掉开场白 ❌ 不要求Claude做决定

## Ch3: 分配角色

**关键发现：** 角色提示不仅能改变风格，还能**显著提升逻辑推理准确率**。

```
# 无角色 → 答错
USER: Jack is looking at Anne. Anne is looking at George. Jack is married, George is not, and we don't know if Anne is married. Is a married person looking at an unmarried person?

# 有角色 → 答对
SYSTEM: You are a logic bot designed to answer complex logic problems step by step.
USER: [同上问题]
```

**反模式：** ❌ 不给Claude分配角色处理复杂逻辑/数学问题 ❌ 角色过于简单 ❌ 只设角色不给目标受众

## Ch4: 分离数据与指令

**XML 标签结构化模板：**
```
You are a customer service agent. Analyze the following email and determine if it requires urgent escalation.

<instructions>
1. Check if the email mentions any safety concerns
2. Check if the customer is requesting a refund over $500
3. Check if the email contains threats of legal action
</instructions>

<email>
{{EMAIL_CONTENT}}
</email>

Output your decision as either "ESCALATE" or "NORMAL" with a brief reason.
```

**反模式：** ❌ 不使用XML标签分隔输入数据与指令 ❌ 替换变量后边界模糊 ❌ 拼写和语法错误 ❌ 列表格式与指令格式混同

## Ch5: 格式化输出与Prefill

**三大格式控制技巧：**

1. **明确指定输出格式**
```
Output your answer as a JSON object with the following schema:
{
  "name": "string",
  "species": "string",
  "age": number
}
```

2. **用XML标签包裹输出**
```
Output your analysis in the following format:
<summary>One paragraph summary</summary>
<key_points>
- Point 1
- Point 2
</key_points>
<recommendation>Specific recommendation</recommendation>
```

3. **Prefill技巧 — 替Claude说话**
```python
# 引导 JSON 输出
messages=[
    {"role": "user", "content": "Output the animal info as JSON"},
    {"role": "assistant", "content": "{"}
]

# 引导表格输出
messages=[
    {"role": "user", "content": "Compare these options"},
    {"role": "assistant", "content": "| Option | Pros | Cons |\n|--------|------|------|"}
]

# 引导中文输出
messages=[
    {"role": "user", "content": "Explain quantum computing"},
    {"role": "assistant", "content": "量子计算是"}
]
```

**反模式：** ❌ 不指定输出格式 ❌ 不使用prefill控制输出开头

## Ch6: 预认知（逐步思考）

**进阶技巧 — 标签化思考：**
```
Analyze whether the following list of numbers is sorted in ascending order.

<instructions>
1. First, inside <analysis> tags, compare each adjacent pair of numbers
2. Then, inside <answer> tags, state "YES" or "NO"
</instructions>

<numbers>
3, 7, 12, 5, 18, 21
</numbers>
```

**反模式：** ❌ 不让Claude先思考再回答 ❌ 要求"思考但不输出思考过程"——思考只有在出声时才算数！ ❌ 忽略Claude对选项顺序的敏感性——Claude更倾向于选择两个选项中的后者

## Ch7: 少样本提示

**高质量少样本模板：**
```
You are a document reviewer. Analyze each review and provide a one-sentence summary.

<examples>
<example>
<doc_review>
The product exceeded my expectations in terms of performance, but the packaging was damaged during shipping.
</doc_review>
<summary>
Positive product sentiment with negative packaging feedback.
</summary>
</example>

<example>
<doc_review>
While the customer service was responsive, the actual product quality didn't match what was advertised.
</doc_review>
<summary>
Mixed experience: responsive service but poor product quality compared to advertising.
</summary>
</example>
</examples>

Now review this document:
<doc_review>
{{REVIEW}}
</doc_review>
```

**最佳实践：** 2-5个示例，覆盖边界情况，格式绝对一致，使用`<example>`标签界定边界

**反模式：** ❌ 不提供期望行为的示例 ❌ 只描述格式不给实际示例

## Ch8: 避免幻觉（5大核心技术）

| # | 技术 | 提示词模板 | 适用场景 |
|---|------|-----------|----------|
| 1 | 只使用提供的信息 | `Answer based only on the following document. If the document doesn't contain the answer, say "Not enough information."` | 知识问答 |
| 2 | 先引用后回答 | `First, find the relevant quotes from the document. Then, based only on those quotes, answer the question. Cite your sources.` | 文档QA |
| 3 | 给"不知道"选项 | `If you're not sure about the answer or the information isn't provided, say "I'm not sure" rather than guessing.` | 所有场景 |
| 4 | 温度设为0 | `temperature=0.0` | 事实准确性场景 |
| 5 | 让Claude自我评估 | `After providing your answer, rate your confidence level from 1-5. If below 3, explain what information you're missing.` | 关键决策 |

**反幻觉完整模板：**
```
You are a precise research assistant. Answer the question using ONLY the provided document.

<instructions>
1. Search the document for relevant information
2. Quote the exact passages you're using
3. If the document doesn't contain enough information, say "The document does not provide sufficient information to answer this question."
4. Do NOT use any outside knowledge or make inferences beyond what is explicitly stated
</instructions>

<document>
{{DOCUMENT}}
</document>

Question: {{QUESTION}}

Answer format:
<evidence>[relevant quotes]</evidence>
<answer>[your answer based ONLY on the evidence]</answer>
<confidence>[HIGH/MEDIUM/LOW]</confidence>
```

**反模式：** ❌ 不给Claude"不知道"选项 ❌ 不让Claude先找证据再回答 ❌ 温度设置过高

## Ch9: 从零构建复杂提示词

**10元素结构（按最佳顺序）：**

```
# 1. 角色定义 (Personas)
You are an expert {ROLE} with deep expertise in {DOMAIN}.

# 2. 任务描述 (Task Context)
Your task is to {TASK_DESCRIPTION}.

# 3. 指令步骤 (Instructions)
<instructions>
1. {STEP_1}
2. {STEP_2}
3. {STEP_3}
</instructions>

# 4. 示例 (Examples)
<examples>
<example>
Input: {INPUT_EXAMPLE}
Output: {OUTPUT_EXAMPLE}
</example>
</examples>

# 5. 输入数据 (Input Data)
<document>
{SOURCE_DATA}
</document>

# 6. 格式要求 (Formatting)
Output your response in {FORMAT} format.

# 7. 约束条件 (Constraints)
- {CONSTRAINT_1}
- {CONSTRAINT_2}

# 8. 语气风格 (Tone/Style)
Use a {TONE} tone. Write for an audience of {AUDIENCE}.

# 9. 退出条件 (Escape Hatches)
If you cannot complete the task, say "I need more information about {SPECIFIC_MISSING_INFO}."

# 10. 用户查询 (User Query — 放在最后！)
{USER_QUESTION}
```

> **关键原则**：用户查询放在提示词**底部**，而非顶部。LLM对提示词末尾的内容注意力最强。

## 附录: 链式提示

**设计原则：** 每步只做一件事，有明确输入/输出格式，可独立测试，前一步输出=后一步输入

**反模式：** 让Claude复查自己的答案时，不给"退出"选项——只说"找出不对的词"会导致改掉正确答案。应加 "If all the words are real words, return the original list"。

## 附录: 工具使用

```python
response = client.messages.create(
    model="claude-3-5-sonnet-20241022",
    tools=[{
        "name": "get_weather",
        "description": "Get the current weather in a location",
        "input_schema": {
            "type": "object",
            "properties": {
                "location": {"type": "string", "description": "City name"},
            },
            "required": ["location"]
        }
    }],
    messages=[{"role": "user", "content": "What's the weather in Tokyo?"}]
)
```
