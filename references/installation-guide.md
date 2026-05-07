# Prompt Engineering Skill 安装与使用指南

> 版本：2.2.0 | 适用平台：CodeBuddy / Cursor / Claude Desktop / ChatGPT / 任意LLM工具 / Python程序

---

## 一、Skill 文件结构

```
prompt-engineering/
├── SKILL.md                          ← 主配置（元数据+工作流+输入输出规范）
└── references/
    ├── techniques-overview.md        ← 18种技术决策树
    ├── anthropic-tutorial.md         ← Anthropic 9章教程精要
    ├── openai-guide.md               ← OpenAI 6策略18战术
    ├── dspy-framework.md             ← DSPy 编程式优化
    ├── tot-method.md                 ← 思维树方法
    ├── templates-library.md          ← 100+ 参数化模板
    ├── anti-patterns.md              ← 15项反模式清单
    ├── resources-index.md            ← 论文/工具/课程索引
    ├── chinese-ecosystem.md          ← 中文生态资源
    ├── source-paths.md               ← 来源路径索引
    ├── performance-benchmarks.md     ← 性能基准数据
    ├── test-suite.md                 ← 21个自动化测试
    ├── integration-examples.md       ← Python 代码集成示例
    └── invocation-specification.md   ← 调用指令规范
```

**核心文件**：`SKILL.md`（必须） + `references/` 目录下的参考文件（按需加载）

---

## 二、各平台安装方式

### 2.1 CodeBuddy（原生支持）

```bash
# Skill 已安装在：
# d:\Prompt\.codebuddy\skills\prompt-engineering\

# 直接使用 — 在对话中输入触发词即可：
# "帮我设计一个客服提示词"
# "优化这个prompt"
# "检查提示词有什么反模式"
```

### 2.2 Cursor

**方法一：Project Rules（推荐）**

1. 在项目根目录创建 `.cursor/rules/prompt-engineering.mdc`：

```yaml
---
description: 提示词工程专家 - 当需要设计、优化、调试提示词时激活
globs:
alwaysApply: false
---

请阅读以下文件作为你的专业知识库，并严格按照其中定义的工作流执行：

- 主配置：{你的路径}/.codebuddy/skills/prompt-engineering/SKILL.md
- 调用规范：{你的路径}/.codebuddy/skills/prompt-engineering/references/invocation-specification.md

当用户请求涉及提示词设计、优化、调试、技术选型、模板查找或反模式审查时，
按照 invocation-specification.md 中对应 action 的指令模板执行。
参考文件按需加载，不要一次全部读取。
```

2. 在对话中输入 `@prompt-engineering` 触发

**方法二：.cursorrules 文件**

在项目根目录创建 `.cursorrules`：

```
你是一个提示词工程专家。当用户需要设计、优化、调试提示词时，请读取以下知识库文件并按其工作流执行：

知识库路径：{你的路径}/.codebuddy/skills/prompt-engineering/SKILL.md

执行规则：
1. 先判定 action 类型：design / optimize / debug / select / template / review
2. 按 SKILL.md 的5步工作流执行
3. 按需加载 references/ 目录下的参考文件（最多3个）
4. 输出标准化结果（prompt + technique + anti_pattern_check + reasoning + next_steps）
5. 交付前执行12项质量保障检查

参考文件加载策略：
- design → techniques-overview.md + anthropic-tutorial.md
- optimize/debug → anti-patterns.md + techniques-overview.md
- select → techniques-overview.md + performance-benchmarks.md
- template → templates-library.md
- review → anti-patterns.md
```

### 2.3 Claude Desktop（Claude.ai）

**方法一：Project Knowledge 上传**

1. 打开 Claude Desktop → Settings → Projects
2. 创建新项目 "Prompt Engineering Expert"
3. 上传以下文件到 Project Knowledge：
   - `SKILL.md`（必须）
   - `invocation-specification.md`（必须）
   - `anti-patterns.md`（核心参考）
   - `techniques-overview.md`（核心参考）
   - `templates-library.md`（按需）
4. 在项目 Instructions 中写入：

```
你是提示词工程专家。当用户请求涉及提示词时，严格按照 SKILL.md 和 invocation-specification.md 
中定义的工作流执行。先判定 action 类型，再按5步流程处理，最后输出标准化结果。
参考文件已在 Knowledge 中提供，按需引用。
```

**方法二：System Prompt 直接注入**

将 `SKILL.md` 核心内容 + `invocation-specification.md` 的第5节（调用指令模板）粘贴到 Claude 的 System Prompt 中。控制在 8000 token 以内，保留：

```
[粘贴 SKILL.md 的 Action Examples + Core Workflow + Anti-Pattern Check 部分]
[粘贴 invocation-specification.md 的第5节：6种 Action 的标准调用指令模板]
[粘贴 anti-patterns.md 的15项清单]
```

### 2.4 ChatGPT / GPTs

**创建自定义 GPT：**

1. 打开 https://chat.openai.com/gpts/editor
2. 配置：

```
Name: 提示词工程专家
Description: 18种提示技术选型、100+模板、15项反模式检查、模型适配

Instructions:
[粘贴 SKILL.md 核心内容 + invocation-specification.md 第5节]

Knowledge Files（上传）:
- techniques-overview.md
- anti-patterns.md  
- templates-library.md
- anthropic-tutorial.md
- performance-benchmarks.md

Capabilities: ✅ Web Browsing ✅ Code Interpreter
```

3. Conversation starters：

```
- 帮我设计一个xxx的提示词
- 优化这个提示词：[粘贴你的提示词]
- 检查这个提示词有什么反模式
- xxx任务用什么提示技术最好？
```

### 2.5 任意 LLM 工具（通用方法）

**核心原理**：将 Skill 知识作为 System Prompt / 上下文注入，模型即可按规范执行。

**最小注入方案**（约 3000 token）：

```
你是提示词工程专家。严格按照以下流程工作：

## Action 判定
- 需要新建提示词 → design
- 需要改进提示词 → optimize（需提供现有提示词）
- 提示词有问题 → debug（需提供现有提示词+问题模式）
- 推荐技术 → select
- 查找模板 → template
- 检查反模式 → review（需提供现有提示词）

## 5步工作流
1. 任务分析 → 判定 task_type 和约束等级
2. 技术选型 → classification→Few-Shot, reasoning→CoT, agent→ReAct, qa→Generated Knowledge
3. 提示词构建 → 10元素结构：persona→instructions→examples→input_data→format→constraints→tone→escape_hatch→user_query
4. 反模式检查 → 致命级(#1消息交替 #7格式明确 #11退出选项 #14查询底部)必须通过
5. 输出结果

## 输出格式
```
prompt: <构建的提示词>
technique: <技术组合>
structure: [元素列表]
anti_pattern_check: {passed: [], warnings: [], failures: []}
reasoning: <选型理由>
model_adaptation: <模型适配>
next_steps: [后续建议]
```

## 关键规则
- 必须包含退出选项（"如果不确定，请说明"）
- 用户查询放底部
- 用XML标签分隔指令和数据
- 明确指定输出格式
- Claude→XML+Prefill, GPT→JSON mode, 开源→多示例+简指令
```

---

## 三、Python 程序集成

### 3.1 安装依赖

```bash
pip install pydantic>=2.0 pyyaml
# 可选（如需调用LLM API）：
pip install anthropic openai
```

### 3.2 复制 Skill 文件

```bash
# 将整个 Skill 目录复制到你的项目中
cp -r d:/Prompt/.codebuddy/skills/prompt-engineering/ ./libs/prompt-engineering/
```

### 3.3 代码调用

```python
# 方式1：使用封装好的引擎（完整功能）
import sys
sys.path.append("./libs/prompt-engineering/references")

from models import PromptSkillInput, PromptContext, Action, ModelFamily
from skill_engine import PromptEngineeringSkill, PromptEngineeringSkillContext

skill_dir = "./libs/prompt-engineering"

with PromptEngineeringSkillContext(skill_dir) as skill:
    response = skill.execute(PromptSkillInput(
        action=Action.DESIGN,
        task_description="构建客服工单分类系统",
        context=PromptContext(
            model=ModelFamily.CLAUDE_37_SONNET,
            output_format="json",
        ),
    ))

    if response.success:
        print(response.result.prompt)
```

```python
# 方式2：直接读取知识库 + 拼接到 LLM prompt（轻量级）
from pathlib import Path

skill_dir = Path("./libs/prompt-engineering")

def build_llm_prompt(action: str, task: str, model: str = "generic", **kwargs):
    """将 Skill 知识拼接到 LLM 的 system prompt"""
    
    # 读取核心知识
    skill_md = (skill_dir / "SKILL.md").read_text(encoding="utf-8")
    inv_spec = (skill_dir / "references/invocation-specification.md").read_text(encoding="utf-8")
    
    # 按需加载参考
    ref_map = {
        "design": ["techniques-overview.md", "anthropic-tutorial.md"],
        "optimize": ["anti-patterns.md", "techniques-overview.md"],
        "debug": ["anti-patterns.md"],
        "select": ["techniques-overview.md", "performance-benchmarks.md"],
        "template": ["templates-library.md"],
        "review": ["anti-patterns.md"],
    }
    
    refs_content = ""
    for ref_name in ref_map.get(action, []):
        ref_path = skill_dir / "references" / ref_name
        if ref_path.exists():
            refs_content += f"\n\n--- {ref_name} ---\n{ref_path.read_text(encoding='utf-8')}"
    
    # 组装 system prompt
    system = f"""你是提示词工程专家。

{skill_md}

{inv_spec}

## 本次参考知识
{refs_content}

## 当前任务
- 动作：{action}
- 任务：{task}
- 模型：{model}
"""
    if kwargs.get("current_prompt"):
        system += f"\n- 现有提示词：\n```\n{kwargs['current_prompt']}\n```"
    if kwargs.get("failure_mode"):
        system += f"\n- 问题模式：{kwargs['failure_mode']}"
    
    return system

# 使用示例（配合任意LLM SDK）
import anthropic

system_prompt = build_llm_prompt(
    action="design",
    task="构建客服分类系统",
    model="claude-3.7-sonnet",
)

client = anthropic.Anthropic(api_key="sk-xxx")
response = client.messages.create(
    model="claude-3-7-sonnet-20250219",
    max_tokens=4096,
    system=system_prompt,
    messages=[{"role": "user", "content": "构建客服分类系统"}],
)
print(response.content[0].text)
```

---

## 四、验证安装

安装后，用以下测试验证 Skill 是否正常工作：

### 快速测试（任意平台）

依次输入以下3条测试，检查输出是否符合预期：

```
测试1（design）：帮我设计一个邮件分类提示词，分4类，输出JSON
预期：输出包含 technique、structure、anti_pattern_check.failures 为空

测试2（review）：检查这个提示词：帮我分析一下这个数据。{{data}} 请分析趋势。
预期：检出 #7(无格式) 和 #11(无退出选项) 为致命反模式

测试3（select）：数学推理用什么提示技术？
预期：推荐 CoT → Self-Consistency → PAL 递进路径
```

### Python 自动化测试

```python
# 使用 Skill 自带的测试套件验证
from models import PromptSkillInput, PromptContext, Action, ModelFamily
from skill_engine import PromptEngineeringSkill, PromptEngineeringSkillContext
import yaml

skill_dir = "./libs/prompt-engineering"

# 加载测试定义
with open(f"{skill_dir}/references/test-suite.md", "r", encoding="utf-8") as f:
    test_content = f.read()

# 执行 Test 1.1（分类任务设计）
with PromptEngineeringSkillContext(skill_dir) as skill:
    test_input = PromptSkillInput(
        action=Action.DESIGN,
        task_description="构建一个邮件分类系统，将收到的邮件分为：垃圾邮件、工作邮件、个人邮件、通知邮件",
        context=PromptContext(
            model=ModelFamily.CLAUDE_37_SONNET,
            task_type="classification",
            output_format="json",
        ),
    )
    response = skill.execute(test_input)
    
    # 验收标准
    assert response.success, f"执行失败: {response.error}"
    r = response.result
    assert "<instructions>" in r.prompt, "缺少 <instructions> XML标签"
    assert "Few-Shot" in r.technique or "Role Prompt" in r.technique, "技术不匹配"
    assert r.anti_pattern_check.failures == [], f"致命反模式未通过: {r.anti_pattern_check.failures}"
    assert "escape_hatch" in r.structure, "缺少退出选项"
    
    print("✓ Test 1.1 通过")
```

---

## 五、常见问题

| 问题 | 解决方案 |
|------|----------|
| **Skill 知识太多，超出上下文窗口** | 只注入 `SKILL.md` + `invocation-specification.md`，参考文件按需加载（参考第2.5节最小注入方案） |
| **Cursor 不读取外部文件** | 用 `.cursor/rules/` 方法，或在 `.cursorrules` 中直接粘贴核心内容 |
| **Claude Desktop 上传文件数量限制** | 优先上传：`SKILL.md` + `invocation-specification.md` + `anti-patterns.md` + `techniques-overview.md` |
| **ChatGPT GPTs Knowledge 大小限制** | 只上传核心4个文件，其他内容压缩到 Instructions 中 |
| **模型不遵循工作流** | 在 System Prompt 开头强调"严格按照以下流程执行，不要跳步" |
| **输出格式不稳定** | 在指令末尾加上"严格按照上述YAML结构输出，不要添加额外内容" |
| **开源模型效果差** | 增加 few-shot 示例，简化指令，避免复杂嵌套结构 |
| **想在其他电脑使用** | 复制整个 `prompt-engineering/` 目录，更新路径即可 |

---

## 六、一键部署脚本

```bash
# deploy.sh — 将 Skill 部署到指定平台

#!/bin/bash
SKILL_SRC="d:/Prompt/.codebuddy/skills/prompt-engineering"

# 1. 部署到另一个项目
deploy_to_project() {
    local target="$1"
    mkdir -p "$target/.codebuddy/skills/"
    cp -r "$SKILL_SRC" "$target/.codebuddy/skills/"
    echo "✓ 已部署到 $target"
}

# 2. 生成 Cursor Rules
deploy_to_cursor() {
    local target="$1"
    mkdir -p "$target/.cursor/rules"
    cat > "$target/.cursor/rules/prompt-engineering.mdc" << 'EOF'
---
description: 提示词工程专家 - 设计/优化/调试提示词时激活
globs:
alwaysApply: false
---

你是提示词工程专家。当用户需要设计、优化、调试提示词时，请读取以下知识库文件并按其工作流执行：

知识库路径：SKILL_PATH/.codebuddy/skills/prompt-engineering/SKILL.md

按照 references/invocation-specification.md 中的指令模板执行。
参考文件按需加载（design→techniques-overview+anthropic-tutorial, optimize→anti-patterns+techniques-overview, select→techniques-overview+performance-benchmarks, template→templates-library, review→anti-patterns）。
EOF
    sed -i "s|SKILL_PATH|$target|g" "$target/.cursor/rules/prompt-engineering.mdc"
    cp -r "$SKILL_SRC" "$target/.codebuddy/skills/"
    echo "✓ 已部署 Cursor Rules 到 $target"
}

# 3. 导出为单文件（用于 Claude Desktop / ChatGPT 上传）
export_single_file() {
    local output="$1/prompt-engineering-skill-bundle.md"
    echo "# 提示词工程 Skill v2.2.0 完整知识包" > "$output"
    echo "" >> "$output"
    
    for f in SKILL.md references/invocation-specification.md references/anti-patterns.md references/techniques-overview.md references/templates-library.md; do
        echo -e "\n\n--- $f ---\n" >> "$output"
        cat "$SKILL_SRC/$f" >> "$output"
    done
    
    echo "✓ 已导出到 $output ($(wc -c < "$output") bytes)"
}

# 使用示例
# deploy_to_project /path/to/your/project
# deploy_to_cursor /path/to/your/project
# export_single_file /path/to/output
```
