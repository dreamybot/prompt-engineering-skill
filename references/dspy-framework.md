# DSPy 编程式提示词优化框架

> 来源：`d:\Prompt\repos\stanfordnlp_dspy\`

## 核心理念

DSPy = **Declarative Self-improving Python** — 编程而非提示（Programming, not Prompting）。

**传统方法问题：** 手工提示词脆弱，换模型就失效；调优靠试错，不可复现；架构与措辞耦合

**DSPy 解决方案：** 用签名声明输入/输出行为 → 用模块组合管道 → 用优化器自动调优

## 签名（Signature）

**内联签名：**
```python
"question -> answer"
"sentence -> sentiment: bool"
"document -> summary"
"context: list[str], question: str -> answer"
"question, choices: list[str] -> reasoning: str, selection: int"
```

**带指令的签名：**
```python
dspy.Signature("comment -> toxic: bool",
    instructions="Mark as 'toxic' if the comment includes insults, harassment, or sarcastic derogatory remarks.")
```

**类签名（推荐用于生产）：**
```python
from typing import Literal

class Emotion(dspy.Signature):
    """Classify emotion."""
    sentence: str = dspy.InputField()
    sentiment: Literal['sadness', 'joy', 'love', 'anger', 'fear', 'surprise'] = dspy.OutputField()

class CheckCitationFaithfulness(dspy.Signature):
    """Verify that the text is based on the provided context."""
    context: str = dspy.InputField(desc="facts here are assumed to be true")
    text: str = dspy.InputField()
    faithfulness: bool = dspy.OutputField()
    evidence: dict[str, list[str]] = dspy.OutputField(desc="Supporting evidence for claims")
```

> **专家洞察**：不要过早手动调优签名字段名——让DSPy编译器做优化。字段名是语义角色声明，不是提示词文本。

## 模块（Module）

| 模块 | 功能 | 适用场景 |
|------|------|----------|
| `dspy.Predict` | 基础预测器 | 简单任务 |
| `dspy.ChainOfThought` | 自动注入reasoning字段 | 推理任务 |
| `dspy.ProgramOfThought` | LM输出代码→执行→返回结果 | 数学/计算任务 |
| `dspy.ReAct` | 推理+行动循环+工具调用 | Agent任务 |
| `dspy.MultiChainComparison` | 比较多个CoT输出 | 自我一致性 |
| `dspy.majority` | 多数投票 | 高置信度需求 |

**RAG管道示例：**
```python
class RAG(dspy.Module):
    def __init__(self, num_passages=3):
        super().__init__()
        self.retrieve = dspy.Retrieve(k=num_passages)
        self.generate_answer = dspy.ChainOfThought("context, question -> answer")

    def forward(self, question):
        context = self.retrieve(question).passages
        prediction = self.generate_answer(context=context, question=question)
        return dspy.Prediction(context=context, answer=prediction.answer)
```

**多跳搜索示例：**
```python
class Hop(dspy.Module):
    def __init__(self, num_docs=10, num_hops=4):
        self.generate_query = dspy.ChainOfThought('claim, notes -> query')
        self.append_notes = dspy.ChainOfThought('claim, notes, context -> new_notes: list[str], titles: list[str]')

    def forward(self, claim: str) -> list[str]:
        notes, titles = [], []
        for _ in range(self.num_hops):
            query = self.generate_query(claim=claim, notes=notes).query
            context = search(query, k=self.num_docs)
            prediction = self.append_notes(claim=claim, notes=notes, context=context)
            notes.extend(prediction.new_notes)
            titles.extend(prediction.titles)
        return dspy.Prediction(notes=notes, titles=list(set(titles)))
```

## 优化器（Optimizer）

| 数据量 | 推荐优化器 | 说明 |
|--------|-----------|------|
| ~10样本 | `BootstrapFewShot` | 自动搜索少样本示例 |
| 50+样本 | `BootstrapFewShotWithRandomSearch` | 随机搜索 + Bootstrap |
| 0-shot指令优化 | `MIPROv2` (0-shot) | 数据感知指令生成 |
| 200+样本 | `MIPROv2` | 同时优化指令和示例（最强） |
| 需要高效推理 | `BootstrapFinetune` | 蒸馏为权重更新 |
| 需要反思优化 | `GEPA` | 利用LM反思轨迹修复提示词 |

```python
from dspy.teleprompt import BootstrapFewShot, MIPROv2

# 1. BootstrapFewShot
optimizer = BootstrapFewShot(metric=answer_exact_match, max_bootstrapped_demos=4)
optimized = optimizer.compile(RAG(), trainset=trainset)

# 2. MIPROv2（推荐）
optimizer = MIPROv2(metric=answer_exact_match, num_threads=4)
optimized = optimizer.compile(RAG(), trainset=trainset,
    max_bootstrapped_demos=3, max_labeled_demos=3)

# 3. 保存和加载
optimized.save("optimized_rag.json")
RAG().load("optimized_rag.json")
```

## 速查表

| 操作 | 代码 |
|------|------|
| 配置 LM | `lm = dspy.LM('openai/gpt-4o-mini')` |
| 设置默认 | `dspy.configure(lm=lm)` |
| 基础预测 | `dspy.Predict("q -> a")` |
| 链式思考 | `dspy.ChainOfThought("q -> a")` |
| ReAct | `dspy.ReAct("q -> a", tools=[...])` |
| 检索 | `dspy.Retrieve(k=5)` |
| 编译优化 | `optimizer.compile(module, trainset=...)` |
| 保存/加载 | `optimized.save("m.json")` / `RAG().load("m.json")` |
| 查看提示 | `dspy.inspect_trace()` |

## ChatAdapter 提示词生成格式

DSPy默认的ChatAdapter将签名转换为以下格式：

```
System: Your input fields are:
1. `question` (str)
Your output fields are:
1. `answer` (str)
All interactions will be structured in the following way:

[[ ## question ## ]]
{question}

[[ ## answer ## ]]
{answer}

[[ ## completed ## ]]

In adhering to this structure, your objective is:
Given the fields `question`, produce the fields `answer`.

User: [[ ## question ## ]]
What is 2+2?

Assistant: [[ ## answer ## ]]
4

[[ ## completed ## ]]
```
