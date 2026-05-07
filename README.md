# 🎯 Prompt Engineering Expert Skill

> 系统化提示词工程专家技能 — 覆盖18种提示技术选型、OpenAI/Anthropic官方最佳实践、DSPy编程式优化、ToT思维树方法、100+生产级模板和反模式避坑清单

[![Skill Version](https://img.shields.io/badge/version-2.2.0-blue.svg)](https://github.com/)
[![License](https://img.shields.io/badge/license-MIT--0-green.svg)](LICENSE)
[![CodeBuddy](https://img.shields.io/badge/platform-CodeBuddy-orange.svg)](https://www.codebuddy.ai)

---

## 📖 概述

本技能将提示词编写从"试错式手艺"提升为**有纪律的工程过程**，提供：

- **6种核心动作**：design / optimize / debug / select / template / review
- **18种提示技术**：Zero-shot → CoT → ToT → DSPy → Meta-Prompting 完整覆盖
- **反模式避坑**：10种致命反模式检测 + 修复方案
- **模型适配**：GPT-4o / Claude 3.7 / Gemini 2.5 等主流模型差异处理
- **生产级模板**：100+ 可直接使用的提示词模板

## 🗂️ 项目结构

```
prompt-engineering-skill/
├── SKILL.md                          # 技能核心定义（元数据 + 动作规范 + 输出格式）
├── README.md                         # 本文件
├── LICENSE                           # MIT-0 许可证
├── .gitignore                        # Git 忽略规则
└── references/                       # 知识参考文件（16个）
    ├── anthropic-tutorial.md         # Anthropic 官方提示词教程
    ├── anti-patterns.md              # 10种致命反模式 + 修复方案
    ├── chinese-ecosystem.md          # 中文提示词生态资源
    ├── dspy-framework.md             # DSPy 编程式优化框架
    ├── installation-guide.md         # 6大平台安装指南
    ├── integration-examples.md       # Python 代码集成示例
    ├── invocation-specification.md   # 精确调用指令规范
    ├── openai-guide.md               # OpenAI 官方提示词指南
    ├── performance-benchmarks.md     # 技术×模型效果基准
    ├── resources-index.md            # 外部资源索引
    ├── skillhub-upload-guide.md      # SkillHub 上传指南
    ├── source-paths.md               # 源文件路径映射
    ├── techniques-overview.md        # 18种提示技术详解
    ├── templates-library.md          # 100+ 提示词模板库
    ├── test-suite.md                 # 21条验收测试用例
    └── tot-method.md                 # 思维树(ToT)方法
```

## 🚀 快速开始

### 在 CodeBuddy 中使用

本技能已可直接使用，触发词示例：

```
帮我设计一个客服工单分类的提示词
优化这个prompt，它输出太啰嗦
我的提示词导致模型幻觉，怎么修
我应该用什么提示技术来做QA任务
给我一个代码审查的提示词模板
帮我审查这个提示词的质量
```

### 在其他工具中安装

详见 [`references/installation-guide.md`](references/installation-guide.md)，支持：

| 平台 | 安装方式 |
|------|---------|
| Cursor | `.cursor/rules/` 或 `.cursorrules` |
| Claude Desktop | Project Knowledge 上传 |
| ChatGPT / GPTs | 自定义 GPT + Knowledge Files |
| 任意 LLM | System Prompt 注入 |
| Python 程序 | 导入引擎模块 |

### Python 集成示例

```python
from prompt_engineering_skill import PromptEngineeringSkill, PromptSkillInput, Action

skill = PromptEngineeringSkill()
response = skill.execute(PromptSkillInput(
    action=Action.DESIGN,
    task_description="构建客服工单分类系统",
))

if response.success:
    print(response.result.prompt)     # 生成的提示词
    print(response.result.technique)  # 使用的技术
```

详见 [`references/integration-examples.md`](references/integration-examples.md)。

## ⚡ 6种核心动作

| 动作 | 用途 | 必填参数 |
|------|------|---------|
| **`design`** | 从零设计新提示词 | `task_description` |
| **`optimize`** | 优化现有提示词 | `task_description` + `current_prompt` |
| **`debug`** | 诊断修复提示词问题 | `task_description` + `current_prompt` |
| **`select`** | 推荐合适的提示技术 | `task_description` |
| **`template`** | 提供参数化模板 | `task_description` |
| **`review`** | 审查提示词质量 | `current_prompt` |

## 📊 提示技术覆盖

```
基础层:  Zero-shot │ Few-shot │ System Prompt │ Role Prompting
推理层:  CoT │ Self-Consistency │ ToT │ GoT
优化层:  APE │ DSPy │ Meta-Prompting │ Self-Refine
结构层:  XML Tags │ Markdown │ JSON Schema │ Separator
控制层:  Conditioning │ Precognition │ Step-Back │ Active-Prompt
```

## 🛡️ 反模式检测

自动检测10种致命反模式并给出修复方案：

| 反模式 | 严重程度 | 典型表现 |
|--------|---------|---------|
| 指令矛盾 | 🔴 致命 | "简洁"和"详细"同时出现 |
| 角色混淆 | 🔴 致命 | System/User 角色指令冲突 |
| 缺少输出格式 | 🟡 重要 | 未指定 JSON/Markdown 等格式 |
| 超长提示词 | 🟡 重要 | >4000 token 未分段 |
| ... | ... | 完整列表见 anti-patterns.md |

## 📋 调用指令规范

每个 Action 都有标准化的调用指令模板，详见 [`references/invocation-specification.md`](references/invocation-specification.md)。

**最简调用示例：**

```
动作：design
任务描述：构建一个客服工单分类系统
目标模型：claude-3.7-sonnet
输出格式：json
```

## 🔧 开发指南

### 本地开发

```bash
# 克隆仓库
git clone https://github.com/<your-username>/prompt-engineering-skill.git
cd prompt-engineering-skill

# 验证结构
ls -la SKILL.md references/
```

### 运行测试

参考 [`references/test-suite.md`](references/test-suite.md) 中的 21 条验收测试用例。

### 发布新版本

1. 更新 `SKILL.md` 中的 `version` 和 `last_updated`
2. 更新兼容性矩阵
3. 提交并打 Tag：`git tag v2.3.0`

## 📤 上传到 SkillHub

详见 [`references/skillhub-upload-guide.md`](references/skillhub-upload-guide.md)，支持 SkillHub / ClawHub / 私有化部署。

## 📄 许可证

本项目采用 [MIT-0 License](LICENSE) — 无条件免费使用、修改和分发。

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

1. Fork 本仓库
2. 创建功能分支：`git checkout -b feature/your-feature`
3. 提交更改：`git commit -m 'Add your feature'`
4. 推送分支：`git push origin feature/your-feature`
5. 创建 Pull Request

## 📮 联系方式

- 问题反馈：[GitHub Issues](https://github.com/<your-username>/prompt-engineering-skill/issues)
- 讨论交流：[GitHub Discussions](https://github.com/<your-username>/prompt-engineering-skill/discussions)
