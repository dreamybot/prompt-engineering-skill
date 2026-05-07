# 提示工程资源导航与论文索引

> 来源：`d:\Prompt\repos\promptslab_Awesome-Prompt-Engineering\README.md`

## 新手入门路径（5步）

1. **学习基础** → [ChatGPT Prompt Engineering for Developers](https://www.deeplearning.ai/short-courses/chatgpt-prompt-engineering-for-developers/)（免费，90分钟）
2. **阅读指南** → [Prompt Engineering Guide by DAIR.AI](https://www.promptingguide.ai/)（开源，全面）
3. **学习供应商文档** → [OpenAI 指南](https://platform.openai.com/docs/guides/prompt-engineering) · [Anthropic 指南](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview)
4. **了解领域方向** → [Anthropic: Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
5. **阅读研究** → [The Prompt Report](https://arxiv.org/abs/2406.06608)（1500+论文，58+提示技术）

## 核心论文索引

### 主要综述论文

| 论文 | 年份 | 核心贡献 |
|------|------|----------|
| [The Prompt Report](https://arxiv.org/abs/2406.06608) | 2024 | 58种文本+40种多模态提示技术，1500+论文 |
| [A Systematic Survey of PE](https://arxiv.org/abs/2402.07927) | 2024 | 44种技术跨应用领域 |
| [Demystifying Chains, Trees, Graphs of Thoughts](https://arxiv.org/abs/2401.14295) | 2024 | 多提示推理拓扑统一框架 |
| [Towards Reasoning Era](https://arxiv.org/abs/2503.09567) | 2025 | 区分o1/R1时代长CoT与短CoT |

### 提示词优化论文

| 论文 | 年份 | 核心贡献 |
|------|------|----------|
| [OPRO](https://arxiv.org/abs/2309.03409) | 2023 | LLM作为优化器，元提示优化，优于人类设计最高50% |
| [DSPy](https://arxiv.org/abs/2310.03714) | 2023 | 编程（非提示）LLM框架，自动提示优化 |
| [MIPRO](https://arxiv.org/abs/2406.11695) | 2024 | 贝叶斯优化多阶段LM程序，最高13%精度提升 |
| [TextGrad](https://arxiv.org/abs/2406.07496) | 2024 | 文本反馈作为梯度，Nature发表 |
| [EvoPrompt](https://arxiv.org/abs/2309.08532) | 2023 | 进化算法自动优化离散提示 |

### 推理进展论文

| 论文 | 年份 | 核心贡献 |
|------|------|----------|
| [Scaling Test-Time Compute](https://arxiv.org/abs/2408.03314) | 2024 | 最优测试时计算分配可超越14x大模型 |
| [DeepSeek-R1](https://arxiv.org/abs/2501.12948) | 2025 | 纯RL训练推理模型匹配o1，开源 |
| [s1: Simple Test-Time Scaling](https://arxiv.org/abs/2501.19393) | 2025 | 1000样本+预算强制=推理模型 |

### 提示词压缩论文

| 论文 | 年份 | 核心贡献 |
|------|------|----------|
| [LLMLingua-2](https://arxiv.org/abs/2403.12968) | 2024 | 比LLMLingua快3-6倍，ACL 2024 |
| [LongLLMLingua](https://arxiv.org/abs/2310.06839) | 2023 | 问题感知压缩，4x少token提升21.4% |

## 核心工具分类

| 类别 | 工具 | 用途 |
|------|------|------|
| **提示词管理/测试** | PromptPerfect, Promptfoo, Parea AI | 版本管理、A/B测试、回归测试 |
| **LLM评估** | LM Evaluation Harness, HELM, Open LLM Leaderboard | 模型能力基准测试 |
| **智能体框架** | LangChain, CrewAI, AutoGen, DSPy | Agent开发框架 |
| **提示词优化** | DSPy Optimizer, TextGrad, GEPA, MIPRO | 自动化提示词调优 |
| **红队测试** | Garak, PyRIT, PromptInject | 安全性测试 |
| **MCP** | Model Context Protocol, Claude Desktop | 模型上下文协议 |
| **AI编码** | Cursor, Windsurf, GitHub Copilot, Cline, Claude Code | AI辅助编码 |

## 前沿模型（2025-2026）

| 类别 | 模型 |
|------|------|
| **前沿模型** | GPT-4.1, Claude 3.7 Sonnet, Gemini 2.5 Pro, Grok 3 |
| **推理模型** | o3, o4-mini, DeepSeek-R1, Claude 3.7 Sonnet (extended thinking) |
| **开源模型** | Llama 4, DeepSeek-V3, Qwen3, Mistral Large |

## 推荐书籍

| 书名 | 作者 | 主题 |
|------|------|------|
| The Prompt Report | White et al. | 58+提示技术综述 |
| Prompt Engineering for Generative AI | James Phoenix | 提示工程实践 |
| Building LLM Apps | Valentina Alto | LLM应用开发 |
| AI Agents in Action | Micheal Lanham | AI智能体 |

## 推荐课程

| 课程 | 平台 | 费用 |
|------|------|------|
| ChatGPT Prompt Engineering for Developers | DeepLearning.AI | 免费 |
| Prompt Engineering for Vision Models | DeepLearning.AI | 免费 |
| Building AI Applications with LangChain | Coursera | 免费 |
| Learn Prompting | learnprompting.org | 免费 |
