# Skill 集成调用示例

> 本文档提供完整的代码示例，展示如何在主程序或其他模块中调用 prompt-engineering Skill。
> 覆盖所有6种 action、错误处理、资源释放和高级场景。

---

## 1. 核心概念

Skill 的调用遵循统一模式：

```
输入（action + task_description + context）→ Skill 处理 → 输出（result 对象）
```

**关键依赖**：
- Python ≥ 3.10（使用 `match-case`、类型注解）
- `pydantic ≥ 2.0`（数据验证与序列化）
- `pyyaml`（YAML 格式输入解析，可选）
- `anthropic` / `openai`（仅运行生成结果时需要，非 Skill 自身依赖）

---

## 2. 数据模型定义

### 2.1 输入模型

```python
# models.py — Skill 输入输出数据模型

from __future__ import annotations
from enum import Enum
from typing import Optional
from pydantic import BaseModel, Field


# ─── 枚举类型 ───

class Action(str, Enum):
    DESIGN = "design"
    OPTIMIZE = "optimize"
    DEBUG = "debug"
    SELECT = "select"
    TEMPLATE = "template"
    REVIEW = "review"


class TaskType(str, Enum):
    CLASSIFICATION = "classification"
    GENERATION = "generation"
    REASONING = "reasoning"
    QA = "qa"
    CODE = "code"
    AGENT = "agent"
    CREATIVE = "creative"
    MULTI_STEP = "multi_step"


class ModelFamily(str, Enum):
    GPT4 = "gpt-4"
    GPT4O = "gpt-4o"
    GPT41 = "gpt-4.1"
    CLAUDE_35_SONNET = "claude-3.5-sonnet"
    CLAUDE_37_SONNET = "claude-3.7-sonnet"
    DEEPSEEK_R1 = "deepseek-r1"
    LLAMA = "llama"
    QWEN = "qwen"
    GENERIC = "generic"


class OutputFormat(str, Enum):
    JSON = "json"
    XML = "xml"
    MARKDOWN = "markdown"
    TABLE = "table"
    CODE = "code"
    NATURAL_LANGUAGE = "natural_language"


class FailureMode(str, Enum):
    HALLUCINATION = "hallucination"
    FORMAT_MISMATCH = "format_mismatch"
    INCONSISTENCY = "inconsistency"
    VERBOSITY = "verbosity"
    INACCURACY = "inaccuracy"
    REFUSAL = "refusal"


class Language(str, Enum):
    ZH = "zh"
    EN = "en"
    BILINGUAL = "bilingual"


class Complexity(str, Enum):
    SIMPLE = "simple"
    MODERATE = "moderate"
    COMPLEX = "complex"
    PRODUCTION = "production"


# ─── 输入模型 ───

class PromptContext(BaseModel):
    """Skill 上下文参数，全部可选，提供越多信息输出越精准"""
    model: ModelFamily = Field(
        default=ModelFamily.GENERIC,
        description="目标模型，影响适配策略"
    )
    task_type: Optional[TaskType] = Field(
        default=None,
        description="任务类型，None 时自动推断"
    )
    output_format: Optional[OutputFormat] = Field(
        default=None,
        description="期望的输出格式"
    )
    failure_mode: Optional[FailureMode] = Field(
        default=None,
        description="当前问题模式（optimize/debug/review 时必填）"
    )
    language: Language = Field(
        default=Language.ZH,
        description="输出语言"
    )
    complexity: Complexity = Field(
        default=Complexity.MODERATE,
        description="提示词复杂度等级"
    )


class PromptSkillInput(BaseModel):
    """Skill 统一输入模型"""
    action: Action = Field(..., description="执行的动作类型")
    task_description: str = Field(..., min_length=1, description="任务描述")
    current_prompt: Optional[str] = Field(
        default=None,
        description="现有提示词（optimize/debug/review 时必填）"
    )
    context: PromptContext = Field(
        default_factory=PromptContext,
        description="上下文参数"
    )

    def validate_action_requirements(self) -> None:
        """验证 action 特定的必填字段"""
        if self.action in (Action.OPTIMIZE, Action.DEBUG, Action.REVIEW):
            if not self.current_prompt:
                raise ValueError(
                    f"action={self.action.value} 需要 current_prompt 参数"
                )
```

### 2.2 输出模型

```python
# models.py（续）

class AntiPatternCheck(BaseModel):
    """反模式检查结果"""
    passed: list[str] = Field(default_factory=list, description="通过的项目")
    warnings: list[str] = Field(default_factory=list, description="警告项")
    failures: list[str] = Field(default_factory=list, description="必须修复的致命项")


class PromptSkillResult(BaseModel):
    """Skill 统一输出模型"""
    prompt: str = Field(..., description="构建或优化后的完整提示词")
    technique: str = Field(..., description="应用的主要技术组合")
    structure: list[str] = Field(
        default_factory=list,
        description="提示词元素结构列表"
    )
    anti_pattern_check: AntiPatternCheck = Field(
        default_factory=AntiPatternCheck,
        description="反模式检查结果"
    )
    reasoning: str = Field(..., description="技术选型理由")
    model_adaptation: str = Field(default="", description="模型适配说明")
    next_steps: list[str] = Field(
        default_factory=list,
        description="后续建议"
    )


class SkillResponse(BaseModel):
    """Skill 完整响应包装"""
    success: bool = Field(..., description="是否成功")
    action: Action = Field(..., description="执行的 action")
    result: Optional[PromptSkillResult] = Field(
        default=None, description="成功时的结果"
    )
    error: Optional[str] = Field(default=None, description="失败时的错误信息")
    references_loaded: list[str] = Field(
        default_factory=list,
        description="本次加载的参考文件列表"
    )
```

---

## 3. Skill 核心引擎

```python
# skill_engine.py — Skill 核心引擎，负责加载参考文件、执行工作流、返回结果

from __future__ import annotations
import os
from pathlib import Path
from typing import Optional

from models import (
    Action, PromptSkillInput, PromptContext,
    PromptSkillResult, AntiPatternCheck, SkillResponse
)


class PromptEngineeringSkill:
    """
    提示词工程 Skill 核心引擎 v2.2.0

    使用方式：
        skill = PromptEngineeringSkill(skill_dir="/path/to/skill")
        response = skill.execute(input_data)
    """

    VERSION = "2.2.0"

    # 参考文件加载映射：action → 需要加载的参考文件
    REFERENCE_MAP: dict[Action, list[str]] = {
        Action.DESIGN: [
            "references/techniques-overview.md",
            "references/anthropic-tutorial.md",
            "references/templates-library.md",
        ],
        Action.OPTIMIZE: [
            "references/techniques-overview.md",
            "references/anti-patterns.md",
            "references/anthropic-tutorial.md",
        ],
        Action.DEBUG: [
            "references/anti-patterns.md",
            "references/anthropic-tutorial.md",
        ],
        Action.SELECT: [
            "references/techniques-overview.md",
            "references/performance-benchmarks.md",
        ],
        Action.TEMPLATE: [
            "references/templates-library.md",
        ],
        Action.REVIEW: [
            "references/anti-patterns.md",
        ],
    }

    def __init__(self, skill_dir: str | Path):
        """
        初始化 Skill 引擎。

        Args:
            skill_dir: Skill 根目录路径，包含 SKILL.md 和 references/ 目录
        """
        self.skill_dir = Path(skill_dir)
        self._reference_cache: dict[str, str] = {}

        # 验证 Skill 目录结构
        self._validate_structure()

    def _validate_structure(self) -> None:
        """验证 Skill 目录完整性"""
        skill_md = self.skill_dir / "SKILL.md"
        refs_dir = self.skill_dir / "references"

        if not skill_md.exists():
            raise FileNotFoundError(f"SKILL.md 不存在: {skill_md}")
        if not refs_dir.exists():
            raise FileNotFoundError(f"references/ 目录不存在: {refs_dir}")

    def load_reference(self, ref_path: str) -> str:
        """
        按需加载参考文件（带缓存）。

        Args:
            ref_path: 相对于 skill_dir 的路径，如 "references/anti-patterns.md"

        Returns:
            参考文件内容

        Raises:
            FileNotFoundError: 文件不存在
        """
        if ref_path in self._reference_cache:
            return self._reference_cache[ref_path]

        full_path = self.skill_dir / ref_path
        if not full_path.exists():
            raise FileNotFoundError(f"参考文件不存在: {full_path}")

        content = full_path.read_text(encoding="utf-8")
        self._reference_cache[ref_path] = content
        return content

    def _load_references_for_action(self, action: Action) -> list[str]:
        """根据 action 加载所需的参考文件，返回加载的文件名列表"""
        loaded = []
        for ref_path in self.REFERENCE_MAP.get(action, []):
            try:
                self.load_reference(ref_path)
                loaded.append(ref_path)
            except FileNotFoundError:
                # 降级：缺少参考文件不阻断流程
                pass
        return loaded

    def clear_cache(self) -> None:
        """清除参考文件缓存，释放内存"""
        self._reference_cache.clear()

    def execute(self, input_data: PromptSkillInput) -> SkillResponse:
        """
        执行 Skill 工作流。

        这是主入口方法，按照 5 步工作流处理：
        1. 输入验证
        2. 加载参考文件
        3. 任务分析
        4. 执行 action 逻辑
        5. 反模式检查 + 返回结果

        Args:
            input_data: 标准化输入对象

        Returns:
            SkillResponse 包含结果或错误信息
        """
        try:
            # Step 1: 输入验证
            input_data.validate_action_requirements()

            # Step 2: 渐进式加载参考文件
            refs_loaded = self._load_references_for_action(input_data.action)

            # Step 3-5: 分发到具体 action 处理器
            match input_data.action:
                case Action.DESIGN:
                    result = self._handle_design(input_data)
                case Action.OPTIMIZE:
                    result = self._handle_optimize(input_data)
                case Action.DEBUG:
                    result = self._handle_debug(input_data)
                case Action.SELECT:
                    result = self._handle_select(input_data)
                case Action.TEMPLATE:
                    result = self._handle_template(input_data)
                case Action.REVIEW:
                    result = self._handle_review(input_data)

            return SkillResponse(
                success=True,
                action=input_data.action,
                result=result,
                references_loaded=refs_loaded,
            )

        except ValueError as e:
            return SkillResponse(
                success=False,
                action=input_data.action,
                error=f"输入验证失败: {e}",
            )
        except FileNotFoundError as e:
            return SkillResponse(
                success=False,
                action=input_data.action,
                error=f"参考文件缺失: {e}",
            )
        except Exception as e:
            return SkillResponse(
                success=False,
                action=input_data.action,
                error=f"执行异常: {type(e).__name__}: {e}",
            )

    # ─── Action 处理器（示意实现，实际由 Skill 知识驱动） ───

    def _handle_design(self, inp: PromptSkillInput) -> PromptSkillResult:
        """design action: 从零设计提示词"""
        # 实际实现中，这里会结合参考文件知识构建提示词
        # 以下展示完整的返回结构
        return PromptSkillResult(
            prompt="<constructed prompt>",
            technique="Role Prompt + Few-Shot + XML Structure",
            structure=["persona", "instructions", "examples", "format", "escape_hatch"],
            anti_pattern_check=AntiPatternCheck(
                passed=["#1 消息交替", "#5 XML标签分隔", "#7 输出格式明确"],
                warnings=["#8 考虑添加prefill"],
                failures=[],
            ),
            reasoning="分类任务首选 zero-shot，生产级加 few-shot 确保格式稳定",
            model_adaptation="Claude 3.7: 使用XML标签结构化，建议prefill引导JSON输出",
            next_steps=[
                "测试：提交10条真实数据验证准确率",
                "优化：如准确率<95%，添加更多边界case示例",
            ],
        )

    def _handle_optimize(self, inp: PromptSkillInput) -> PromptSkillResult:
        """optimize action: 优化现有提示词"""
        return PromptSkillResult(
            prompt="<optimized prompt>",
            technique="Direct Instruction + Few-Shot + Prefill",
            structure=["persona", "instructions", "examples", "format", "escape_hatch"],
            anti_pattern_check=AntiPatternCheck(
                passed=["#3 去前言", "#5 XML标签", "#7 输出格式明确", "#11 退出选项"],
                warnings=[],
                failures=[],
            ),
            reasoning="原提示词缺少明确格式约束和退出选项，导致输出不稳定",
            model_adaptation="",
            next_steps=["设置 temperature=0", "多次运行验证一致性"],
        )

    def _handle_debug(self, inp: PromptSkillInput) -> PromptSkillResult:
        """debug action: 诊断修复提示词问题"""
        return PromptSkillResult(
            prompt="<fixed prompt>",
            technique="Grounding + Citation + Escape Hatch",
            structure=["persona", "instructions", "format", "escape_hatch"],
            anti_pattern_check=AntiPatternCheck(
                passed=["#11 退出选项", "#12 先找证据"],
                warnings=["#8 考虑prefill"],
                failures=[],
            ),
            reasoning="幻觉根因：无 grounding 约束、无退出选项、查询位置不佳",
            model_adaptation="",
            next_steps=["设置 temperature=0", "添加引用验证逻辑"],
        )

    def _handle_select(self, inp: PromptSkillInput) -> PromptSkillResult:
        """select action: 推荐提示技术"""
        return PromptSkillResult(
            prompt="",  # select 不返回 prompt
            technique="CoT → Self-Consistency → PAL",
            structure=[],
            anti_pattern_check=AntiPatternCheck(passed=[], warnings=[], failures=[]),
            reasoning="推理任务推荐路径：CoT为基础，Self-Consistency提升可靠性，PAL处理计算",
            model_adaptation="DeepSeek-R1: 不需要显式CoT指令",
            next_steps=[
                "先用CoT验证基线效果",
                "如需更高可靠性，升级到Self-Consistency(5次投票)",
                "参考 performance-benchmarks.md 查看具体效果数据",
            ],
        )

    def _handle_template(self, inp: PromptSkillInput) -> PromptSkillResult:
        """template action: 查找和定制模板"""
        return PromptSkillResult(
            prompt="<template with {{variables}}>",
            technique="Template + Parameterization",
            structure=["persona", "instructions", "examples", "format"],
            anti_pattern_check=AntiPatternCheck(
                passed=["#5 XML标签", "#7 输出格式"],
                warnings=["#11 模板需根据场景添加退出选项"],
                failures=[],
            ),
            reasoning="匹配到 Code Reviewer Security 模板，已参数化 {{LANGUAGE}}、{{FRAMEWORK}}、{{SECURITY_FOCUS}}",
            model_adaptation="",
            next_steps=["替换 {{变量}} 为实际值", "根据目标模型调整结构"],
        )

    def _handle_review(self, inp: PromptSkillInput) -> PromptSkillResult:
        """review action: 审查反模式"""
        return PromptSkillResult(
            prompt="",  # review 不修改 prompt，只检查
            technique="Anti-Pattern Checklist (15 items)",
            structure=[],
            anti_pattern_check=AntiPatternCheck(
                passed=["#1 消息交替", "#6 无拼写错误"],
                warnings=["#5 建议用XML标签分隔", "#9 思考过程应可见", "#4 推理任务应分配角色"],
                failures=["#7 不指定输出格式", "#11 无不知道选项", "#14 查询不在底部"],
            ),
            reasoning="3项致命级反模式需立即修复：#7格式缺失、#11无退出选项、#14查询位置",
            model_adaptation="",
            next_steps=[
                "[致命] 修复 #7: 添加明确的JSON/XML输出格式定义",
                "[致命] 修复 #11: 添加'如果不确定，请说明'退出选项",
                "[致命] 修复 #14: 将用户查询移至提示词底部",
                "[重要] 修复 #5: 用<instructions>和<document>标签分隔",
            ],
        )


# ─── 上下文管理器支持 ───

class PromptEngineeringSkillContext:
    """
    上下文管理器封装，确保资源正确释放。

    用法:
        with PromptEngineeringSkillContext(skill_dir) as skill:
            result = skill.execute(input_data)
    """

    def __init__(self, skill_dir: str | Path):
        self.skill_dir = skill_dir
        self._skill: Optional[PromptEngineeringSkill] = None

    def __enter__(self) -> PromptEngineeringSkill:
        self._skill = PromptEngineeringSkill(self.skill_dir)
        return self._skill

    def __exit__(self, exc_type, exc_val, exc_tb) -> None:
        if self._skill:
            self._skill.clear_cache()
        self._skill = None
```

---

## 4. 调用示例 — 6种 Action

### 4.1 Design — 从零设计提示词

```python
# example_design.py

from models import PromptSkillInput, PromptContext, Action, ModelFamily, TaskType, OutputFormat, Complexity, Language
from skill_engine import PromptEngineeringSkill, PromptEngineeringSkillContext


def example_design_classification():
    """示例：设计客服工单分类提示词"""

    # ─── 构建输入 ───
    input_data = PromptSkillInput(
        action=Action.DESIGN,
        task_description="构建一个客服工单分类Agent，将用户反馈分为：Bug、Feature Request、Question、Complaint，并提取优先级和关键信息",
        context=PromptContext(
            model=ModelFamily.CLAUDE_37_SONNET,
            task_type=TaskType.CLASSIFICATION,
            output_format=OutputFormat.JSON,
            language=Language.ZH,
            complexity=Complexity.PRODUCTION,
        ),
    )

    # ─── 执行（带资源管理） ───
    skill_dir = r"d:\Prompt\.codebuddy\skills\prompt-engineering"

    with PromptEngineeringSkillContext(skill_dir) as skill:
        response = skill.execute(input_data)

        if response.success:
            result = response.result
            print(f"技术: {result.technique}")
            print(f"结构: {result.structure}")
            print(f"反模式检查 - 通过: {len(result.anti_pattern_check.passed)}")
            print(f"反模式检查 - 致命: {len(result.anti_pattern_check.failures)}")
            print(f"理由: {result.reasoning}")
            print(f"模型适配: {result.model_adaptation}")
            print(f"\n生成提示词:\n{result.prompt}")
            print(f"\n后续建议: {result.next_steps}")
        else:
            print(f"执行失败: {response.error}")


def example_design_agent():
    """示例：设计Agent系统提示词"""

    input_data = PromptSkillInput(
        action=Action.DESIGN,
        task_description="设计一个客服Agent的system prompt，需要能查询订单、处理退款、转接人工客服",
        context=PromptContext(
            model=ModelFamily.CLAUDE_35_SONNET,
            task_type=TaskType.AGENT,
            output_format=OutputFormat.JSON,
            language=Language.ZH,
            complexity=Complexity.PRODUCTION,
        ),
    )

    skill_dir = r"d:\Prompt\.codebuddy\skills\prompt-engineering"

    with PromptEngineeringSkillContext(skill_dir) as skill:
        response = skill.execute(input_data)

        if response.success and response.result:
            # 验证 Agent 关键要素
            r = response.result
            assert "ReAct" in r.technique, "Agent 任务应推荐 ReAct"
            assert "escape_hatch" in r.structure, "必须有退出选项"
            assert r.anti_pattern_check.failures == [], "不应有致命反模式"
            print("✓ Agent 提示词设计通过验证")


if __name__ == "__main__":
    example_design_classification()
    example_design_agent()
```

### 4.2 Optimize — 优化现有提示词

```python
# example_optimize.py

from models import (
    PromptSkillInput, PromptContext, Action, ModelFamily,
    FailureMode, OutputFormat
)
from skill_engine import PromptEngineeringSkill, PromptEngineeringSkillContext


def example_optimize_verbosity():
    """示例：优化啰嗦的提示词"""

    input_data = PromptSkillInput(
        action=Action.OPTIMIZE,
        task_description="这个提示词输出太啰嗦，需要更简洁",
        current_prompt="""你是一个帮助别人的助手，用户会问你问题，你需要回答他们的问题。
回答的时候请尽量详细一些，把你想到的都写出来，不要遗漏任何细节。
如果你觉得答案很长，那也没关系，继续写就好了。
你的回答应该尽可能的长和全面。""",
        context=PromptContext(
            model=ModelFamily.GENERIC,
            failure_mode=FailureMode.VERBOSITY,
        ),
    )

    skill_dir = r"d:\Prompt\.codebuddy\skills\prompt-engineering"

    with PromptEngineeringSkillContext(skill_dir) as skill:
        response = skill.execute(input_data)

        if response.success and response.result:
            r = response.result
            # 关键验证点
            has_length_limit = any(
                kw in r.prompt
                for kw in ["不超过", "3段以内", "300字", "concise", "brief"]
            )
            has_no_preamble = any(
                kw in r.prompt
                for kw in ["Skip the preamble", "去掉开场白", "直接回答"]
            )

            print(f"优化后技术: {r.technique}")
            print(f"长度限制: {'✓' if has_length_limit else '✗'}")
            print(f"去前言: {'✓' if has_no_preamble else '✗'}")
            print(f"反模式修复: {r.anti_pattern_check.passed}")
        else:
            print(f"失败: {response.error}")


def example_optimize_hallucination():
    """示例：优化有幻觉问题的提示词"""

    input_data = PromptSkillInput(
        action=Action.OPTIMIZE,
        task_description="模型在回答文档问题时经常编造信息",
        current_prompt="根据以下内容回答用户的问题：\n{document}\n问题：{question}",
        context=PromptContext(
            model=ModelFamily.CLAUDE_37_SONNET,
            failure_mode=FailureMode.HALLUCINATION,
        ),
    )

    skill_dir = r"d:\Prompt\.codebuddy\skills\prompt-engineering"

    with PromptEngineeringSkillContext(skill_dir) as skill:
        response = skill.execute(input_data)

        if response.success and response.result:
            r = response.result
            # 反幻觉关键验证
            has_escape = any(
                kw in r.prompt
                for kw in ["不确定", "I'm not sure", "不知道", "信息不足"]
            )
            has_citation = any(
                kw in r.prompt
                for kw in ["引用", "cite", "根据文档", "先找证据"]
            )
            has_xml = "<document>" in r.prompt or "<context>" in r.prompt

            print(f"退出选项: {'✓' if has_escape else '✗'}")
            print(f"引用验证: {'✓' if has_citation else '✗'}")
            print(f"XML分隔: {'✓' if has_xml else '✗'}")
            print(f"后续建议: {r.next_steps}")


if __name__ == "__main__":
    example_optimize_verbosity()
    example_optimize_hallucination()
```

### 4.3 Debug — 诊断修复提示词

```python
# example_debug.py

from models import (
    PromptSkillInput, PromptContext, Action, ModelFamily,
    FailureMode, OutputFormat
)
from skill_engine import PromptEngineeringSkill, PromptEngineeringSkillContext


def example_debug_hallucination():
    """示例：诊断 Claude 幻觉问题"""

    input_data = PromptSkillInput(
        action=Action.DEBUG,
        task_description="Claude总是在编造信息",
        current_prompt="你是一位知识渊博的助手。请回答用户的问题。\n问题：{question}",
        context=PromptContext(
            model=ModelFamily.CLAUDE_37_SONNET,
            failure_mode=FailureMode.HALLUCINATION,
        ),
    )

    skill_dir = r"d:\Prompt\.codebuddy\skills\prompt-engineering"

    with PromptEngineeringSkillContext(skill_dir) as skill:
        response = skill.execute(input_data)

        if response.success and response.result:
            r = response.result
            # 致命反模式应被检出
            print(f"诊断结果 - 致命反模式: {r.anti_pattern_check.failures}")
            print(f"诊断结果 - 警告: {r.anti_pattern_check.warnings}")
            print(f"根因分析: {r.reasoning}")
            print(f"\n修复建议 (按优先级):")
            for i, step in enumerate(r.next_steps, 1):
                print(f"  {i}. {step}")


def example_debug_format_mismatch():
    """示例：诊断 JSON 格式输出问题"""

    input_data = PromptSkillInput(
        action=Action.DEBUG,
        task_description="模型输出格式不对，需要JSON但返回了Markdown",
        current_prompt="请分析这封邮件的意图和紧急程度。\n邮件内容：{email}",
        context=PromptContext(
            model=ModelFamily.GPT4O,
            failure_mode=FailureMode.FORMAT_MISMATCH,
            output_format=OutputFormat.JSON,
        ),
    )

    skill_dir = r"d:\Prompt\.codebuddy\skills\prompt-engineering"

    with PromptEngineeringSkillContext(skill_dir) as skill:
        response = skill.execute(input_data)

        if response.success and response.result:
            r = response.result
            # 验证格式修复策略
            assert "#7" in str(r.anti_pattern_check.failures), "应检出 #7 格式缺失"
            assert "JSON" in r.prompt or "json" in r.prompt.lower(), "修复后应包含JSON定义"
            print(f"✓ 格式问题诊断通过")
            print(f"模型适配: {r.model_adaptation}")


if __name__ == "__main__":
    example_debug_hallucination()
    example_debug_format_mismatch()
```

### 4.4 Select — 技术推荐

```python
# example_select.py

from models import (
    PromptSkillInput, PromptContext, Action, TaskType, FailureMode
)
from skill_engine import PromptEngineeringSkill, PromptEngineeringSkillContext


def example_select_reasoning():
    """示例：推荐数学推理技术"""

    input_data = PromptSkillInput(
        action=Action.SELECT,
        task_description="什么提示技术适合数学推理？",
        context=PromptContext(task_type=TaskType.REASONING),
    )

    skill_dir = r"d:\Prompt\.codebuddy\skills\prompt-engineering"

    with PromptEngineeringSkillContext(skill_dir) as skill:
        response = skill.execute(input_data)

        if response.success and response.result:
            r = response.result
            print(f"推荐技术路径: {r.technique}")
            print(f"选型理由: {r.reasoning}")
            print(f"进阶建议:")
            for step in r.next_steps:
                print(f"  → {step}")


def example_select_qa():
    """示例：推荐知识QA技术（带反幻觉需求）"""

    input_data = PromptSkillInput(
        action=Action.SELECT,
        task_description="需要让模型基于文档回答问题，不能编造信息",
        context=PromptContext(
            task_type=TaskType.QA,
            failure_mode=FailureMode.HALLUCINATION,
        ),
    )

    skill_dir = r"d:\Prompt\.codebuddy\skills\prompt-engineering"

    with PromptEngineeringSkillContext(skill_dir) as skill:
        response = skill.execute(input_data)

        if response.success and response.result:
            r = response.result
            # 验证推荐包含反幻觉技术
            has_grounding = "grounding" in r.technique.lower() or "引用" in r.reasoning
            print(f"推荐路径: {r.technique}")
            print(f"反幻觉覆盖: {'✓' if has_grounding else '✗'}")


if __name__ == "__main__":
    example_select_reasoning()
    example_select_qa()
```

### 4.5 Template — 模板查找与定制

```python
# example_template.py

from models import (
    PromptSkillInput, PromptContext, Action, TaskType, Complexity, Language
)
from skill_engine import PromptEngineeringSkill, PromptEngineeringSkillContext


def example_template_code_review():
    """示例：查找并定制代码审查模板"""

    input_data = PromptSkillInput(
        action=Action.TEMPLATE,
        task_description="有没有代码审查的提示词模板？需要安全审计功能",
        context=PromptContext(
            task_type=TaskType.CODE,
            language=Language.ZH,
        ),
    )

    skill_dir = r"d:\Prompt\.codebuddy\skills\prompt-engineering"

    with PromptEngineeringSkillContext(skill_dir) as skill:
        response = skill.execute(input_data)

        if response.success and response.result:
            r = response.result
            print(f"匹配模板: {r.reasoning}")

            # 模板参数化定制
            customized = r.prompt
            customized = customized.replace("{{LANGUAGE}}", "Python")
            customized = customized.replace("{{FRAMEWORK}}", "Django")
            customized = customized.replace("{{SECURITY_FOCUS}}", "OWASP Top 10")

            print(f"\n定制后模板:\n{customized}")
            print(f"\n后续: {r.next_steps}")


def example_template_sql():
    """示例：查找SQL助手模板并参数化"""

    input_data = PromptSkillInput(
        action=Action.TEMPLATE,
        task_description="找一个SQL助手的prompt",
        context=PromptContext(
            task_type=TaskType.CODE,
            output_format=OutputFormat.CODE,
        ),
    )

    skill_dir = r"d:\Prompt\.codebuddy\skills\prompt-engineering"

    with PromptEngineeringSkillContext(skill_dir) as skill:
        response = skill.execute(input_data)

        if response.success and response.result:
            r = response.result

            # 参数化替换
            customized = r.prompt
            customized = customized.replace("{{DIALECT}}", "PostgreSQL")
            customized = customized.replace("{{SCHEMA}}", "users(id, name, email, created_at); orders(id, user_id, amount, status)")
            customized = customized.replace("{{QUESTION}}", "查找2024年消费最多的前10名用户")

            print(f"定制后SQL助手:\n{customized}")


if __name__ == "__main__":
    example_template_code_review()
    example_template_sql()
```

### 4.6 Review — 反模式审查

```python
# example_review.py

from models import (
    PromptSkillInput, PromptContext, Action, ModelFamily, FailureMode
)
from skill_engine import PromptEngineeringSkill, PromptEngineeringSkillContext


def example_review_critical():
    """示例：审查含致命反模式的提示词"""

    input_data = PromptSkillInput(
        action=Action.REVIEW,
        task_description="帮我检查这个提示词有什么问题",
        current_prompt="""帮我分析一下这个数据。

这是数据：{{data}}

请分析数据中的趋势。""",
        context=PromptContext(model=ModelFamily.CLAUDE_37_SONNET),
    )

    skill_dir = r"d:\Prompt\.codebuddy\skills\prompt-engineering"

    with PromptEngineeringSkillContext(skill_dir) as skill:
        response = skill.execute(input_data)

        if response.success and response.result:
            r = response.result
            print("=== 反模式审查报告 ===")
            print(f"\n✅ 通过 ({len(r.anti_pattern_check.passed)}项):")
            for item in r.anti_pattern_check.passed:
                print(f"   {item}")

            print(f"\n⚠️ 警告 ({len(r.anti_pattern_check.warnings)}项):")
            for item in r.anti_pattern_check.warnings:
                print(f"   {item}")

            print(f"\n❌ 致命 ({len(r.anti_pattern_check.failures)}项):")
            for item in r.anti_pattern_check.failures:
                print(f"   {item}")

            print(f"\n📋 分析: {r.reasoning}")
            print(f"\n🔧 修复建议 (按优先级):")
            for i, step in enumerate(r.next_steps, 1):
                print(f"   {i}. {step}")


def example_review_thinking_hidden():
    """示例：审查"思考但不输出"反模式"""

    input_data = PromptSkillInput(
        action=Action.REVIEW,
        task_description="这个prompt有什么反模式？",
        current_prompt="你是一位助手。用户会问你数学问题，请你思考后回答。\n但是思考过程不要显示出来，直接给答案就好。\n问题：{question}",
        context=PromptContext(model=ModelFamily.GENERIC),
    )

    skill_dir = r"d:\Prompt\.codebuddy\skills\prompt-engineering"

    with PromptEngineeringSkillContext(skill_dir) as skill:
        response = skill.execute(input_data)

        if response.success and response.result:
            r = response.result
            # "思考但不输出" 应被标记为反模式
            thinking_issue = any(
                "#9" in item
                for item in r.anti_pattern_check.warnings + r.anti_pattern_check.failures
            )
            print(f"思考可见性问题检出: {'✓' if thinking_issue else '✗'}")
            print(f"分析: {r.reasoning}")


if __name__ == "__main__":
    example_review_critical()
    example_review_thinking_hidden()
```

---

## 5. 批量处理与管道

### 5.1 批量 Action 执行

```python
# example_batch.py

from models import PromptSkillInput, PromptContext, Action, TaskType, ModelFamily
from skill_engine import PromptEngineeringSkill, PromptEngineeringSkillContext
from concurrent.futures import ThreadPoolExecutor, as_completed
from typing import Any


def batch_execute(
    skill_dir: str,
    inputs: list[PromptSkillInput],
    max_workers: int = 4,
) -> list[dict[str, Any]]:
    """
    批量执行多个 Skill 调用。

    Args:
        skill_dir: Skill 目录路径
        inputs: 输入列表
        max_workers: 并发线程数

    Returns:
        结果列表，每项包含 input_index, success, result/error
    """
    results = []

    def _execute_single(index: int, inp: PromptSkillInput) -> dict:
        # 每个线程创建独立的 Skill 实例，避免缓存冲突
        with PromptEngineeringSkillContext(skill_dir) as skill:
            response = skill.execute(inp)
            return {
                "index": index,
                "action": inp.action.value,
                "success": response.success,
                "result": response.result.model_dump() if response.result else None,
                "error": response.error,
            }

    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        futures = {
            executor.submit(_execute_single, i, inp): i
            for i, inp in enumerate(inputs)
        }
        for future in as_completed(futures):
            results.append(future.result())

    # 按原始顺序排序
    results.sort(key=lambda x: x["index"])
    return results


# ─── 使用示例 ───

def example_batch():
    skill_dir = r"d:\Prompt\.codebuddy\skills\prompt-engineering"

    inputs = [
        PromptSkillInput(
            action=Action.DESIGN,
            task_description="设计一个情感分析提示词",
            context=PromptContext(task_type=TaskType.CLASSIFICATION),
        ),
        PromptSkillInput(
            action=Action.DESIGN,
            task_description="设计一个代码生成提示词",
            context=PromptContext(task_type=TaskType.CODE),
        ),
        PromptSkillInput(
            action=Action.SELECT,
            task_description="推理任务用什么技术？",
            context=PromptContext(task_type=TaskType.REASONING),
        ),
    ]

    results = batch_execute(skill_dir, inputs, max_workers=2)

    for r in results:
        status = "✓" if r["success"] else "✗"
        print(f"[{status}] {r['action']}: {r.get('error') or '成功'}")


if __name__ == "__main__":
    example_batch()
```

### 5.2 设计→审查管道

```python
# example_pipeline.py

from models import (
    PromptSkillInput, PromptContext, Action, ModelFamily,
    TaskType, OutputFormat, Complexity, Language
)
from skill_engine import PromptEngineeringSkill, PromptEngineeringSkillContext


def design_then_review_pipeline(
    skill: PromptEngineeringSkill,
    task_description: str,
    context: PromptContext,
) -> dict:
    """
    设计→审查管道：先设计提示词，再自动审查反模式。

    这是生产环境的推荐流程，确保交付的提示词通过所有致命级检查。

    Args:
        skill: Skill 实例
        task_description: 任务描述
        context: 上下文

    Returns:
        包含设计结果和审查结果的字典
    """
    # Step 1: 设计
    design_input = PromptSkillInput(
        action=Action.DESIGN,
        task_description=task_description,
        context=context,
    )
    design_response = skill.execute(design_input)

    if not design_response.success:
        return {"success": False, "error": f"设计失败: {design_response.error}"}

    design_result = design_response.result

    # Step 2: 审查设计结果
    review_input = PromptSkillInput(
        action=Action.REVIEW,
        task_description="审查新生成的提示词质量",
        current_prompt=design_result.prompt,
        context=context,
    )
    review_response = skill.execute(review_input)

    if not review_response.success:
        return {
            "success": True,
            "design": design_result,
            "review_error": review_response.error,
        }

    review_result = review_response.result

    # Step 3: 汇总
    has_critical = len(review_result.anti_pattern_check.failures) > 0

    return {
        "success": True,
        "design": design_result,
        "review": review_result,
        "needs_fix": has_critical,
        "summary": (
            f"设计完成，审查发现 {len(review_result.anti_pattern_check.failures)} 项致命反模式"
            if has_critical
            else "设计完成，通过所有致命级检查"
        ),
    }


# ─── 使用示例 ───

def example_pipeline():
    skill_dir = r"d:\Prompt\.codebuddy\skills\prompt-engineering"

    with PromptEngineeringSkillContext(skill_dir) as skill:
        result = design_then_review_pipeline(
            skill=skill,
            task_description="构建一个客服工单分类系统",
            context=PromptContext(
                model=ModelFamily.CLAUDE_37_SONNET,
                task_type=TaskType.CLASSIFICATION,
                output_format=OutputFormat.JSON,
                language=Language.ZH,
                complexity=Complexity.PRODUCTION,
            ),
        )

        if result["success"]:
            print(result["summary"])
            if result["needs_fix"]:
                print(f"\n需修复的致命项: {result['review'].anti_pattern_check.failures}")
                print(f"修复建议: {result['review'].next_steps}")
            else:
                print(f"技术: {result['design'].technique}")
                print(f"结构: {result['design'].structure}")


if __name__ == "__main__":
    example_pipeline()
```

---

## 6. 错误处理最佳实践

### 6.1 分层错误处理

```python
# error_handling.py

from models import PromptSkillInput, PromptContext, Action
from skill_engine import PromptEngineeringSkill, PromptEngineeringSkillContext


def robust_execute(
    skill_dir: str,
    input_data: PromptSkillInput,
    max_retries: int = 2,
) -> dict:
    """
    带重试和降级的健壮执行。

    降级策略：
    1. 首次执行失败 → 重试（可能是临时文件读取问题）
    2. 重试仍失败 → 降级到 GENERIC 模型策略
    3. 降级仍失败 → 返回结构化错误信息
    """

    # ─── 第一层：正常执行 + 重试 ───
    for attempt in range(max_retries + 1):
        with PromptEngineeringSkillContext(skill_dir) as skill:
            response = skill.execute(input_data)

            if response.success:
                return {
                    "success": True,
                    "result": response.result.model_dump(),
                    "attempts": attempt + 1,
                    "degraded": False,
                }

            # 输入验证错误不重试
            if "输入验证失败" in (response.error or ""):
                return {
                    "success": False,
                    "error": response.error,
                    "attempts": 1,
                    "degraded": False,
                }

    # ─── 第二层：模型降级 ───
    if input_data.context.model != ModelFamily.GENERIC:
        degraded_input = input_data.model_copy(deep=True)
        degraded_input.context.model = ModelFamily.GENERIC

        with PromptEngineeringSkillContext(skill_dir) as skill:
            # 清除可能损坏的缓存
            skill.clear_cache()
            response = skill.execute(degraded_input)

            if response.success:
                return {
                    "success": True,
                    "result": response.result.model_dump(),
                    "attempts": max_retries + 2,
                    "degraded": True,
                    "degradation_note": "降级到 GENERIC 模型策略",
                }

    # ─── 第三层：返回结构化错误 ───
    return {
        "success": False,
        "error": response.error if response else "未知错误",
        "attempts": max_retries + 2,
        "degraded": True,
        "suggestion": "检查 Skill 目录结构和参考文件完整性",
    }
```

### 6.2 输入验证守卫

```python
# validation.py

from models import (
    PromptSkillInput, PromptContext, Action, ModelFamily,
    TaskType, FailureMode, OutputFormat
)


def validate_input_before_execute(input_data: PromptSkillInput) -> list[str]:
    """
    执行前输入验证，返回警告列表。空列表表示全部通过。

    检查项：
    - action 与必填字段的匹配
    - context 与 action 的逻辑一致性
    - 常见配置冲突
    """
    warnings = []

    # 1. action 必填字段检查
    if input_data.action in (Action.OPTIMIZE, Action.DEBUG, Action.REVIEW):
        if not input_data.current_prompt:
            warnings.append(
                f"[致命] action={input_data.action.value} 必须提供 current_prompt"
            )

    # 2. failure_mode 与 action 的逻辑一致性
    if input_data.action == Action.DESIGN and input_data.context.failure_mode:
        warnings.append(
            "[建议] design action 不需要 failure_mode，该参数将被忽略"
        )

    if input_data.action in (Action.OPTIMIZE, Action.DEBUG) and not input_data.context.failure_mode:
        warnings.append(
            "[建议] optimize/debug action 建议提供 failure_mode 以获得更精准的结果"
        )

    # 3. 模型与输出格式兼容性
    if (input_data.context.output_format == OutputFormat.JSON
            and input_data.context.model in (ModelFamily.LLAMA, ModelFamily.QWEN)):
        warnings.append(
            "[建议] 开源模型 JSON 输出不稳定，建议加 Few-Shot 示例锚定格式"
        )

    # 4. 生产级复杂度应提供更多上下文
    if (input_data.context.compatibility == "production"  # 假设有此字段
            and not input_data.context.task_type):
        warnings.append(
            "[建议] 生产级提示词建议明确指定 task_type 而非自动推断"
        )

    # 5. task_description 过短
    if len(input_data.task_description) < 10:
        warnings.append(
            "[建议] task_description 过短，提供更多细节可获得更精准的结果"
        )

    return warnings


# ─── 使用方式 ───

def safe_execute_with_validation(skill_dir: str, input_data: PromptSkillInput):
    """带前置验证的安全执行"""
    # 前置验证
    validation_warnings = validate_input_before_execute(input_data)

    # 致命级警告阻断执行
    fatal_warnings = [w for w in validation_warnings if "[致命]" in w]
    if fatal_warnings:
        for w in fatal_warnings:
            print(f"❌ {w}")
        return None

    # 建议级警告提示但不阻断
    for w in validation_warnings:
        if "[建议]" in w:
            print(f"⚠️ {w}")

    # 执行
    with PromptEngineeringSkillContext(skill_dir) as skill:
        return skill.execute(input_data)
```

---

## 7. 与 LLM API 集成

### 7.1 Skill 输出直接用于 Anthropic API

```python
# llm_integration.py

import anthropic
from models import PromptSkillInput, PromptContext, Action, ModelFamily, OutputFormat
from skill_engine import PromptEngineeringSkill, PromptEngineeringSkillContext


def skill_to_anthropic(
    skill_dir: str,
    task_description: str,
    api_key: str,
    user_query: str,
    model: str = "claude-3-7-sonnet-20250219",
):
    """
    Skill 输出 → Anthropic API 调用的完整流程。

    展示如何将 Skill 设计的提示词直接用于 Claude API。
    """
    # Step 1: 使用 Skill 设计提示词
    input_data = PromptSkillInput(
        action=Action.DESIGN,
        task_description=task_description,
        context=PromptContext(
            model=ModelFamily.CLAUDE_37_SONNET,
            output_format=OutputFormat.JSON,
        ),
    )

    with PromptEngineeringSkillContext(skill_dir) as skill:
        response = skill.execute(input_data)

    if not response.success:
        raise RuntimeError(f"Skill 执行失败: {response.error}")

    prompt_result = response.result

    # Step 2: 将 Skill 输出转化为 Anthropic API 格式
    client = anthropic.Anthropic(api_key=api_key)

    # Skill 建议的 prefill（如果有的话）
    prefill_text = ""
    if prompt_result.model_adaptation and "prefill" in prompt_result.model_adaptation.lower():
        # 从 model_adaptation 中提取 prefill 建议
        if "JSON" in prompt_result.model_adaptation or "json" in prompt_result.technique.lower():
            prefill_text = "{"

    api_response = client.messages.create(
        model=model,
        max_tokens=4096,
        system=prompt_result.prompt,  # Skill 生成的完整提示词作为 system
        messages=[
            {"role": "user", "content": user_query},
        ] + (
            [{"role": "assistant", "content": prefill_text}] if prefill_text else []
        ),
        temperature=0.0 if "hallucination" in prompt_result.reasoning.lower() else 0.7,
    )

    return {
        "skill_technique": prompt_result.technique,
        "skill_structure": prompt_result.structure,
        "anti_pattern_failures": prompt_result.anti_pattern_check.failures,
        "llm_response": api_response.content[0].text,
    }
```

### 7.2 Skill 输出直接用于 OpenAI API

```python
# llm_integration.py（续）

from openai import OpenAI


def skill_to_openai(
    skill_dir: str,
    task_description: str,
    api_key: str,
    user_query: str,
    model: str = "gpt-4o",
):
    """
    Skill 输出 → OpenAI API 调用的完整流程。
    """
    # Step 1: 使用 Skill 设计提示词
    input_data = PromptSkillInput(
        action=Action.DESIGN,
        task_description=task_description,
        context=PromptContext(
            model=ModelFamily.GPT4O,
            output_format=OutputFormat.JSON,
        ),
    )

    with PromptEngineeringSkillContext(skill_dir) as skill:
        response = skill.execute(input_data)

    if not response.success:
        raise RuntimeError(f"Skill 执行失败: {response.error}")

    prompt_result = response.result

    # Step 2: 转化为 OpenAI API 格式
    client = OpenAI(api_key=api_key)

    # 判断是否启用 JSON mode
    use_json_mode = (
        prompt_result.technique and
        any(kw in prompt_result.technique for kw in ["JSON", "Prefill", "Few-Shot"])
        and "json" in task_description.lower()
    )

    kwargs = {
        "model": model,
        "messages": [
            {"role": "system", "content": prompt_result.prompt},
            {"role": "user", "content": user_query},
        ],
        "temperature": 0.0,
    }

    if use_json_mode:
        kwargs["response_format"] = {"type": "json_object"}

    api_response = client.chat.completions.create(**kwargs)

    return {
        "skill_technique": prompt_result.technique,
        "json_mode_enabled": use_json_mode,
        "llm_response": api_response.choices[0].message.content,
    }
```

---

## 8. 完整主程序示例

```python
# main.py — 完整的命令行工具，演示 Skill 的所有功能

#!/usr/bin/env python3
"""
Prompt Engineering Skill CLI v2.2.0

用法:
    python main.py design "构建邮件分类系统"
    python main.py optimize --prompt "你的提示词" "让输出更简洁"
    python main.py debug --prompt "你的提示词" --failure hallucination "诊断幻觉问题"
    python main.py select "什么技术适合推理？"
    python main.py template "代码审查模板"
    python main.py review --prompt "你的提示词" "审查反模式"
"""

import argparse
import json
import sys
from pathlib import Path

from models import (
    PromptSkillInput, PromptContext, Action, ModelFamily,
    TaskType, OutputFormat, FailureMode, Language, Complexity
)
from skill_engine import PromptEngineeringSkill, PromptEngineeringSkillContext


SKILL_DIR = Path(__file__).parent / ".codebuddy" / "skills" / "prompt-engineering"


def parse_args():
    parser = argparse.ArgumentParser(description="Prompt Engineering Skill CLI")
    subparsers = parser.add_subparsers(dest="action", required=True)

    # ─── design ───
    p_design = subparsers.add_parser("design", help="从零设计提示词")
    p_design.add_argument("task", help="任务描述")
    p_design.add_argument("--model", default="generic", choices=[m.value for m in ModelFamily])
    p_design.add_argument("--task-type", choices=[t.value for t in TaskType])
    p_design.add_argument("--format", choices=[f.value for f in OutputFormat])
    p_design.add_argument("--complexity", default="moderate", choices=[c.value for c in Complexity])
    p_design.add_argument("--language", default="zh", choices=[l.value for l in Language])

    # ─── optimize / debug / review (需要 current_prompt) ───
    for action_name in ["optimize", "debug", "review"]:
        p = subparsers.add_parser(action_name, help=f"{action_name} 操作")
        p.add_argument("task", help="任务描述")
        p.add_argument("--prompt", required=True, help="现有提示词")
        p.add_argument("--model", default="generic", choices=[m.value for m in ModelFamily])
        p.add_argument("--failure", choices=[f.value for f in FailureMode])
        p.add_argument("--format", choices=[f.value for f in OutputFormat])

    # ─── select / template ───
    for action_name in ["select", "template"]:
        p = subparsers.add_parser(action_name, help=f"{action_name} 操作")
        p.add_argument("task", help="任务描述")
        p.add_argument("--task-type", choices=[t.value for t in TaskType])
        p.add_argument("--format", choices=[f.value for f in OutputFormat])

    return parser.parse_args()


def build_input(args) -> PromptSkillInput:
    """从命令行参数构建 Skill 输入"""
    context_kwargs = {}

    if hasattr(args, "model") and args.model:
        context_kwargs["model"] = ModelFamily(args.model)
    if hasattr(args, "task_type") and args.task_type:
        context_kwargs["task_type"] = TaskType(args.task_type)
    if hasattr(args, "format") and args.format:
        context_kwargs["output_format"] = OutputFormat(args.format)
    if hasattr(args, "failure") and args.failure:
        context_kwargs["failure_mode"] = FailureMode(args.failure)
    if hasattr(args, "complexity") and args.complexity:
        context_kwargs["complexity"] = Complexity(args.complexity)
    if hasattr(args, "language") and args.language:
        context_kwargs["language"] = Language(args.language)

    return PromptSkillInput(
        action=Action(args.action),
        task_description=args.task,
        current_prompt=getattr(args, "prompt", None),
        context=PromptContext(**context_kwargs),
    )


def format_output(response) -> str:
    """格式化输出"""
    if not response.success:
        return f"❌ 错误: {response.error}"

    r = response.result
    output_parts = []

    # 基本结果
    output_parts.append(f"📌 技术: {r.technique}")
    if r.structure:
        output_parts.append(f"🏗️ 结构: {' → '.join(r.structure)}")
    output_parts.append(f"💡 理由: {r.reasoning}")

    # 反模式检查
    apc = r.anti_pattern_check
    if apc.failures:
        output_parts.append(f"\n❌ 致命反模式 ({len(apc.failures)}):")
        for f in apc.failures:
            output_parts.append(f"   • {f}")
    if apc.warnings:
        output_parts.append(f"\n⚠️ 警告 ({len(apc.warnings)}):")
        for w in apc.warnings:
            output_parts.append(f"   • {w}")

    # 模型适配
    if r.model_adaptation:
        output_parts.append(f"\n🔧 模型适配: {r.model_adaptation}")

    # 提示词（如果是生成类 action）
    if r.prompt:
        output_parts.append(f"\n📝 生成提示词:\n{'─' * 40}\n{r.prompt}\n{'─' * 40}")

    # 后续建议
    if r.next_steps:
        output_parts.append(f"\n➡️ 后续建议:")
        for i, step in enumerate(r.next_steps, 1):
            output_parts.append(f"   {i}. {step}")

    return "\n".join(output_parts)


def main():
    args = parse_args()
    input_data = build_input(args)

    with PromptEngineeringSkillContext(SKILL_DIR) as skill:
        response = skill.execute(input_data)

    print(format_output(response))

    # 非成功退出码
    sys.exit(0 if response.success else 1)


if __name__ == "__main__":
    main()
```

---

## 9. 快速参考卡

### 最小可用示例

```python
from models import PromptSkillInput, PromptContext, Action
from skill_engine import PromptEngineeringSkill

skill = PromptEngineeringSkill(r"d:\Prompt\.codebuddy\skills\prompt-engineering")
response = skill.execute(PromptSkillInput(
    action=Action.DESIGN,
    task_description="构建一个客服分类提示词",
))
print(response.result.prompt if response.success else response.error)
skill.clear_cache()  # 释放缓存
```

### 调用决策树

```
用户请求
  ├── 需要新建提示词？ → action=design
  ├── 现有提示词效果不好？ → action=optimize + current_prompt
  ├── 提示词有具体问题？ → action=debug + current_prompt + failure_mode
  ├── 不知用什么技术？ → action=select
  ├── 需要现成模板？ → action=template
  └── 想检查提示词质量？ → action=review + current_prompt
```

### 资源释放三原则

1. **使用 `with` 上下文管理器** — 确保缓存自动清除
2. **批量场景用独立实例** — 每线程一个 Skill，避免缓存竞争
3. **长期运行时定期清理** — 调用 `skill.clear_cache()` 释放内存
