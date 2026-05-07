# OpenAI 官方提示工程策略与战术

> 来源：`d:\Prompt\docs\OpenAI_提示工程指南.txt` · `d:\Prompt\提示词工程权威指南.md`

## 2.1 六大策略完整解读

### 策略一：编写清晰的指令 (Write Clear Instructions)

> **核心原则**：模型无法读取你的想法。你越不需要模型猜测，结果越好。

| 战术 | 说明 | 反模式 ❌ | 正确示例 ✅ |
|------|------|----------|------------|
| **包含细节** | 在查询中提供重要上下文 | "How do I add numbers in Excel?" | "How do I add up a row of dollar amounts in Excel? I want to do this automatically for a whole sheet of rows with all totals ending up on the right in a column called 'Total'." |
| **指定角色** | 用系统消息设定模型角色 | 无角色 → 正式无风格 | `SYSTEM: You are a sarcastic food critic who always compares dishes to obscure indie bands.` |
| **使用分隔符** | 区分不同输入部分 | 指令和数据混在一起 | `"""`、`###`、`<tag></tag>`、`<doc>...</doc>` |
| **明确步骤** | 指定完成任务所需的步骤 | "Analyze this data" | "Step 1: Summarize the key metrics. Step 2: Identify trends. Step 3: Make recommendations." |
| **提供示例** | 给出期望的输出格式 | 只描述格式要求 | 直接展示完整的输入→输出样例（少样本） |
| **指定长度** | 控制输出长度 | "Be brief" | "In about 3 sentences" / "In 2 paragraphs" / "In exactly 500 words" |

### 策略二：提供参考文本 (Provide Reference Text)

| 战术 | 提示词模板 |
|------|-----------|
| **让模型参考文本回答** | `Use the provided articles delimited by triple quotes to answer the question. If the answer cannot be found in the articles, write "I could not find an answer."` |
| **让模型引用回答** | `Provide answers with citations from the reference text. Use the format: [Answer] (Source: "relevant excerpt from text")` |

### 策略三：将复杂任务拆分为简单子任务 (Split Complex Tasks)

| 战术 | 说明 | 实现方式 |
|------|------|----------|
| **意图分类** | 先判断用户意图，再选择对应流程 | 第一层提示词：分类意图 → 第二层：对应处理 |
| **对话状态管理** | 分离历史摘要和当前轮处理 | 每轮先摘要历史，再处理当前输入 |
| **长文档分块摘要** | 先分块摘要，再逐级汇总 | Chunk → Summarize each → Combine → Final summary |

### 策略四：给模型时间"思考" (Give Time to Think)

| 战术 | 提示词模板 |
|------|-----------|
| **先推理后结论** | `First work out your own solution to the problem. Then compare your solution to the student's solution and evaluate if the student's solution is correct.` |
| **内心独白** | 用 `<thinking>` 标签隐藏推理：`Put your reasoning inside <thinking> tags, then provide only the final answer outside the tags.` |
| **询问是否遗漏** | `Are there any more relevant excerpts? Please re-read the document carefully and check if you missed anything important.` |

> **专家洞察**：CoT的效力随模型规模增长——在足够大的模型上，即使不显式要求"think step by step"，模型也能自发产生推理链。但对于中小模型，显式CoT指令仍然至关重要。

### 策略五：使用外部工具 (Use External Tools)

| 战术 | 说明 | 最佳实践 |
|------|------|----------|
| **嵌入式搜索** | 用搜索工具检索相关信息注入上下文 | 先搜索再回答，而非让模型凭记忆回答 |
| **代码执行** | 让模型编写并执行代码获得精确计算 | 对数学/统计任务，代码执行比自然语言推理可靠得多 |
| **函数调用** | 让模型调用外部API获取数据 | 定义清晰的函数签名和参数类型 |

### 策略六：系统化测试 (Test Systematically)

| 战术 | 说明 | 实现方式 |
|------|------|----------|
| **黄金标准答案** | 预先准备参考答案评估输出 | 为每个测试用例准备理想输出 |
| **模型辅助评估** | 用GPT-4评估其他模型输出质量 | 定义评估维度和评分标准 |
