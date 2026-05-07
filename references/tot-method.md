# Tree of Thoughts (ToT) 思维树方法

> 来源：`d:\Prompt\repos\princeton-nlp_tree-of-thought-llm\`
> 论文：*Tree of Thoughts: Deliberate Problem Solving with Large Language Models* (NeurIPS 2023)

## 核心概念

ToT将推理组织成一棵**思维树**，每个节点=一个中间推理步骤（"思维"）。

**与CoT的本质区别：**
- CoT：一条线性推理链，无法回溯
- ToT：树状结构，支持**探索→评估→回溯→选择**

**四大能力：** 1.探索多种推理路径 2.评估各思维状态 3.回溯到之前节点 4.通过BFS/DFS系统性搜索

## 三大核心操作

| 操作 | 说明 | 模式1 | 模式2 | 选型建议 |
|------|------|-------|-------|----------|
| **思维生成(Generate)** | 生成候选的下一步思维 | `sample`：独立采样多次 | `propose`：一次生成所有候选 | 创意任务→sample；搜索任务→propose |
| **状态评估(Evaluate)** | 评估当前思维状态的价值 | `value`：独立打分(sure/likely/impossible) | `vote`：多候选投票比较 | 离散状态→value；质量比较→vote |
| **状态选择(Select)** | 选择保留哪些思维状态 | `sample`：按概率采样 | `greedy`：贪心选Top-k | 需要多样性→sample；追求最优→greedy |

## Game of 24 提示词模板

**标准提示词（5-shot）：**
```
Use numbers and basic arithmetic operations (+ - * /) to obtain 24.
Input: 4 4 6 8
Answer: (4 + 8) * (6 - 4) = 24
Input: 2 9 10 12
Answer: 2 * 12 * (10 - 9) = 24
Input: 4 9 10 13
Answer: (13 - 9) * (10 - 4) = 24
Input: 1 4 8 8
Answer: (8 / 4 + 1) * 8 = 24
Input: 5 5 5 9
Answer: 5 + 5 + 5 + 9 = 24
Input: {input}
```

**CoT提示词：**
```
Use numbers and basic arithmetic operations (+ - * /) to obtain 24. Each step, you are only allowed to choose two of the remaining numbers to obtain a new number.
Input: 4 4 6 8
Steps:
4 + 8 = 12 (left: 4 6 12)
6 - 4 = 2 (left: 2 12)
2 * 12 = 24 (left: 24)
Answer: (6 - 4) * (4 + 8) = 24
Input: {input}
```

**ToT思维提议提示词（核心！）：**
```
Input: 2 8 8 14
Possible next steps:
2 + 8 = 10 (left: 8 10 14)
8 / 2 = 4 (left: 4 8 14)
14 + 2 = 16 (left: 8 8 16)
2 * 8 = 16 (left: 8 14 16)
8 - 2 = 6 (left: 6 8 14)
14 - 8 = 6 (left: 2 6 8)
14 / 2 = 7 (left: 7 8 8)
14 - 2 = 12 (left: 8 8 12)
Input: {input}
Possible next steps:
```

**ToT状态评估提示词（核心！）：**
```
Evaluate if given numbers can reach 24 (sure/likely/impossible)
10 14
10 + 14 = 24
sure
11 12
11 + 12 = 23
12 - 11 = 1
11 * 12 = 132
11 / 12 = 0.91
impossible
4 4 10
4 + 4 + 10 = 8 + 10 = 18
4 * 10 - 4 = 40 - 4 = 36
(10 - 4) * 4 = 6 * 4 = 24
sure
{input}
```

**值映射：** `{'impossible': 0.001, 'likely': 1, 'sure': 20}`

## Creative Writing 提示词模板

| 类型 | 模板 |
|------|------|
| **标准** | `Write a coherent passage of 4 short paragraphs. The end sentence of each paragraph must be: {input}` |
| **CoT** | 标准指令 + `Make a plan then write. Your output should be: Plan: ... Passage: ...` |
| **投票评估** | `Given an instruction and several choices, decide which choice is most promising. Analyze each choice in detail, then conclude in the last line "The best choice is {s}"` |
| **比较评估** | `Briefly analyze the coherency of the following two passages. Conclude in the last line "The more coherent passage is 1" / "2"` |
| **打分评估** | `Analyze the following passage, then at the last line conclude "Thus the coherency score is {s}", where s is an integer from 1 to 10` |

## Mini Crosswords 提示词模板

**ToT思维提议提示词：**
```
Let's play a 5 x 5 mini crossword, where each word should have exactly 5 letters.
{input}
Given the current status, list all possible answers for unfilled or changed words, and your confidence levels (certain/high/medium/low), using the format "h1. apple (medium)". Use "certain" cautiously and only when you are 100% sure.
```

**置信度映射：** `{'certain': 1, 'high': 0.5, 'medium': 0.2, 'low': 0.1}`

## BFS搜索算法核心代码

```python
def solve(args, task, idx, to_print=True):
    x = task.get_input(idx)
    ys = ['']
    infos = []
    for step in range(task.steps):
        # 步骤1：思维生成
        if args.method_generate == 'sample':
            new_ys = [get_samples(task, x, y, args.n_generate_sample, args.prompt_sample, stop) for y in ys]
        elif args.method_generate == 'propose':
            new_ys = [get_proposals(task, x, y) for y in ys]

        # 步骤2：状态评估
        if args.method_evaluate == 'vote':
            values = get_votes(task, x, new_ys, args.n_evaluate_sample)
        elif args.method_evaluate == 'value':
            values = get_values(task, x, new_ys, args.n_evaluate_sample)

        # 步骤3：状态选择
        if args.method_select == 'sample':
            select_ids = np.random.choice(ids, size=n_select_sample, p=ps)
        elif args.method_select == 'greedy':
            select_ids = sorted(ids, key=lambda x: values[x], reverse=True)[:n_select_sample]

        ys = select_new_ys
    return ys, {'steps': infos}
```

## 快速启动配置

```python
# Game of 24 — propose + value + greedy
args = argparse.Namespace(backend='gpt-4', temperature=0.7, task='game24',
    method_generate='propose', method_evaluate='value', method_select='greedy',
    n_generate_sample=1, n_evaluate_sample=3, n_select_sample=5)

# Creative Writing — sample + vote + greedy
args = argparse.Namespace(backend='gpt-4', temperature=0.7, task='text',
    method_generate='sample', method_evaluate='vote', method_select='greedy',
    n_generate_sample=5, n_evaluate_sample=5, n_select_sample=1)

# Mini Crosswords — propose + value + DFS
args = argparse.Namespace(backend='gpt-4', temperature=0.7, task='crosswords',
    method_generate='propose', method_evaluate='value', method_select='greedy',
    n_generate_sample=6, n_evaluate_sample=3, n_select_sample=3)
```
