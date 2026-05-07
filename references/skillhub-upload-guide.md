# Prompt Engineering Skill 上传 SkillHub 平台指南

> 版本：2.2.0 | 适用平台：SkillHub（腾讯版）/ ClawHub / SkillHub（私有化版）
> 最后更新：2026-05-07

---

## 平台选择

当前有3个可上传的平台，根据需求选择：

| 平台 | 地址 | 适合场景 | 审核方式 |
|------|------|---------|---------|
| **SkillHub（腾讯版）** | https://skillhub.cn | 面向国内用户公开分享 | 异步安全扫描 + 人工审核（敏感场景） |
| **ClawHub** | https://clawhub.ai | 面向全球 OpenClaw 生态 | 实时验证，即发即上架 |
| **SkillHub（私有化版）** | 自部署 | 团队内部使用 | 管理员审核（可配置） |

**推荐**：先上传到 **SkillHub（腾讯版）**（国内节点快、中文搜索好），再同步到 **ClawHub**（全球覆盖）。

---

## 一、上传前准备

### 1.1 检查文件结构

当前 Skill 目录结构：

```
prompt-engineering/
├── SKILL.md                          ← 主配置（必须）
└── references/
    ├── techniques-overview.md
    ├── anthropic-tutorial.md
    ├── openai-guide.md
    ├── dspy-framework.md
    ├── tot-method.md
    ├── templates-library.md
    ├── anti-patterns.md
    ├── resources-index.md
    ├── chinese-ecosystem.md
    ├── source-paths.md
    ├── performance-benchmarks.md
    ├── test-suite.md
    ├── integration-examples.md
    ├── invocation-specification.md
    └── installation-guide.md
```

### 1.2 必须删除的文件

上传前**务必清理**以下文件，否则审核不通过：

```
❌ .git/              — Git 目录
❌ .DS_Store          — macOS 系统文件
❌ LICENSE            — 许可证文件（平台会自动添加 MIT-0）
❌ __pycache__/       — Python 缓存
❌ *.pyc              — 编译文件
❌ node_modules/      — 依赖目录
❌ .env               — 环境变量（含敏感信息）
❌ *.log              — 日志文件
```

### 1.3 检查 SKILL.md 格式合规性

SkillHub/ClawHub 要求 `SKILL.md` 的 front matter 必须包含 `name` 和 `description` 字段。

当前 SKILL.md 的 front matter：

```yaml
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
```

**需要调整**：`description` 字段建议精简到1-2句话（平台显示空间有限），触发词建议移到正文。调整为：

```yaml
---
name: prompt-engineering
version: "2.2.0"
description: 提示词工程专家技能 — 覆盖18种技术选型、OpenAI/Anthropic官方最佳实践、DSPy优化、100+参数化模板和15项反模式避坑清单
---
```

### 1.4 添加 README.md（强烈推荐）

平台详情页会优先显示 README.md，创建一份：

```markdown
# Prompt Engineering Expert Skill

> 提示词工程专家 — 让提示词编写从"凭感觉"变为"有纪律的工程过程"

## 一句话介绍

覆盖 18 种提示技术选型、6 种标准化工作流（设计/优化/调试/选型/模板/审查）、100+ 参数化模板和 15 项反模式避坑清单。

## 功能概览

| Action | 用途 | 示例 |
|--------|------|------|
| `design` | 从零设计提示词 | "帮我写一个客服分类的提示词" |
| `optimize` | 优化现有提示词 | "这个prompt输出太啰嗦了" |
| `debug` | 诊断修复问题 | "模型总是在编造信息" |
| `select` | 推荐提示技术 | "数学推理用什么技术？" |
| `template` | 查找定制模板 | "有没有代码审查的模板？" |
| `review` | 审查反模式 | "检查这个提示词有什么问题" |

## 快速使用

在对话中直接输入触发词即可：
- "帮我设计一个xxx的提示词"
- "优化这个prompt：[粘贴你的提示词]"
- "检查提示词有什么反模式"

## 知识库内容

- 18 种提示技术决策树（Zero-Shot → CoT → ToT → DSPy）
- Anthropic 9 章交互式教程精要
- OpenAI 6 策略 18 战术完整指南
- 100+ 参数化模板（支持 `{{变量}}` 插值）
- 15 项反模式分级检查清单
- 性能基准数据（模型×技术效果对比）

## 兼容性

支持所有主流 LLM：Claude、GPT-4、DeepSeek-R1、Llama、Qwen 等，自动适配格式策略。

## 版本

v2.2.0 | MIT-0 License
```

---

## 二、上传到 SkillHub（腾讯版）

### 步骤 1：访问平台并登录

1. 打开 https://skillhub.cn
2. 点击右上角 **登录**
3. 选择登录方式：
   - 微信扫码（推荐）
   - QQ 登录
   - 腾讯云账号登录
4. 首次登录需补充开发者信息：
   - **昵称**：显示名称
   - **邮箱**：用于接收审核通知

### 步骤 2：进入发布页面

- 点击导航栏 **「发布技能」** 或 **「导入技能」**
- 页面地址格式：`https://skillhub.cn/import` 或 `https://skillhub.cn/dashboard/publish`

### 步骤 3：填写元数据表单

| 字段 | 填写内容 | 注意事项 |
|------|---------|---------|
| **Slug** | `prompt-engineering` | ⚠️ **只能小写字母+短横线**，不能用大写或下划线 |
| **Display name** | `提示词工程专家` | 支持中文，用户搜索时看到的名字 |
| **Version** | `2.2.0` | 语义化版本号，首次建议 `1.0.0` |
| **Tags** | `latest` | 保留默认，可加 `beta`/`stable` |
| **Category** | `开发工具` / `AI辅助` | 选择最匹配的分类 |
| **Namespace** | 你的用户名或团队名 | 没有则自动创建 |
| **Visibility** | `PUBLIC`（公开分享）或 `PRIVATE`（私有） | 公开才能被搜索到 |

### 步骤 4：上传技能文件夹

1. 点击 **「选择文件夹」** 或 **「拖拽上传」**
2. 选择 `prompt-engineering/` 整个目录
3. 系统自动验证：
   - ✅ 检测到 `SKILL.md`
   - ✅ 自动读取 front matter 中的 `name`、`description`、`version`
   - ✅ 显示文件列表（14个参考文件 + SKILL.md + README.md）

### 步骤 5：确认并发布

1. 勾选 **MIT-0 许可证条款**（必须，否则无法发布）
2. 填写 **Changelog**：

```
v2.2.0 初始发布：
- 6种标准化Action工作流（design/optimize/debug/select/template/review）
- 18种提示技术决策树
- 100+参数化模板（支持{{变量}}插值）
- 15项反模式分级检查清单
- Anthropic/OpenAI官方最佳实践
- DSPy编程式优化 + ToT思维树
- 性能基准数据 + 自动化测试套件
- 完整集成示例（Python/API/CLI）
- 多平台安装指南（Cursor/Claude Desktop/ChatGPT）
```

3. 确认所有验证通过（无红色报错）
4. 点击 **「Publish skill」**

### 步骤 6：等待审核

发布后系统执行异步安全扫描：

| 阶段 | 耗时 | 说明 |
|------|------|------|
| 安全扫描 | 1-5分钟 | 恶意代码检测、敏感信息泄露、依赖漏洞 |
| 代码质量评估 | 自动 | 质量评分 |
| 人工审核（如有） | 1-3个工作日 | 仅涉及敏感数据的技能需要 |

**状态流转**：

```
上传文件 → 自动解析SKILL.md → 安全扫描中 → 审核中 → 已发布
                                                    ↓
                                                 审核拒绝 → 修改后重新提交
```

### 步骤 7：确认发布成功

- ✅ 页面顶部显示**绿色成功提示**
- ✅ 自动跳转到技能详情页：`https://skillhub.cn/你的用户名/prompt-engineering`
- ✅ 可在搜索框搜索"提示词工程"验证

---

## 三、上传到 ClawHub（全球版）

### 步骤 1：访问发布页面

打开 https://clawhub.ai/import

### 步骤 2：选择上传方式

#### 方式 A：直接上传文件夹（推荐新手）

1. 选择 **「Upload a folder」** 标签
2. 填写元数据（与 SkillHub 相同）：
   - Slug: `prompt-engineering`
   - Display name: `Prompt Engineering Expert`
   - Version: `1.0.0`
   - Tags: `latest`
3. 拖拽上传 `prompt-engineering/` 文件夹
4. 勾选 MIT-0 许可证
5. 填写 Changelog
6. 点击 **「Publish skill」**

#### 方式 B：从 GitHub 导入（推荐有仓库的用户）

1. 将 Skill 推送到 GitHub 公开仓库：

```bash
cd d:\Prompt
git init prompt-engineering-skill
cd prompt-engineering-skill

# 复制 Skill 文件（排除不需要的）
cp -r ../.codebuddy/skills/prompt-engineering/* .

# 创建 .gitignore
echo ".git" > .gitignore
echo "__pycache__/" >> .gitignore
echo "*.pyc" >> .gitignore
echo ".DS_Store" >> .gitignore
echo ".env" >> .gitignore

git add .
git commit -m "feat: prompt-engineering skill v2.2.0"
git tag v2.2.0
git remote add origin https://github.com/你的用户名/prompt-engineering-skill.git
git push -u origin main --tags
```

2. 在 ClawHub 选择 **「Import from GitHub」** 标签
3. 输入仓库地址：`https://github.com/你的用户名/prompt-engineering-skill`
4. 点击 **「Detect」**，系统自动识别并填充表单
5. 确认后点击 **「Publish skill」**

> 💡 GitHub 导入的优势：自动过滤 `.git` 等无关文件，更稳定。

#### 方式 C：使用 CLI 发布（推荐开发者）

```bash
# 安装 CLI（需 Node.js）
npm install -g clawhub

# 配置注册中心
export CLAWHUB_REGISTRY=https://clawhub.ai

# 登录
npx clawhub login

# 发布
npx clawhub publish ./prompt-engineering

# 发布到指定命名空间
npx clawhub publish ./prompt-engineering --namespace your-username
```

### 步骤 3：验证发布

- 访问 `https://clawhub.ai/你的用户名/prompt-engineering`
- 使用命令验证：`npx clawhub search prompt-engineering`

---

## 四、上传到 SkillHub 私有化版（团队内部）

### 步骤 1：部署 SkillHub（管理员操作）

```bash
# 一键部署（国内推荐阿里云镜像）
curl -fsSL https://imageless.oss-cn-beijing.aliyuncs.com/runtime.sh | sh -s -- up --aliyun

# 访问地址
# Web UI:    http://localhost:3000
# API:       http://localhost:8080
# API 文档:  http://localhost:8080/swagger-ui.html
```

### 步骤 2：登录

- 默认管理员：用户名 `admin`，密码 `ChangeMe!2026`
- ⚠️ 首次登录后立即修改密码

### 步骤 3：创建命名空间

1. 访问 http://localhost:3000/dashboard/namespaces
2. 点击 **「新建命名空间」**
3. 填写：
   - Slug: `team-skills`
   - Display name: `团队技能库`
   - 可见性: `INTERNAL`
4. 开启审核：设置 → 审核策略 → **开启**

### 步骤 4：发布技能

#### Web UI 方式

1. 访问 http://localhost:3000/dashboard/publish
2. 选择命名空间 `team-skills`
3. 上传 `prompt-engineering/` zip 文件
4. 选择可见性 `INTERNAL`
5. 点击 **「发布」**

#### CLI 方式

```bash
export CLAWHUB_REGISTRY=http://localhost:8080

# 登录
npx clawhub login

# 发布
npx clawhub publish ./prompt-engineering --namespace team-skills
```

#### REST API 方式

```bash
curl -X POST http://localhost:8080/api/v1/skills/team-skills/publish \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -F "file=@prompt-engineering.zip" \
  -F "visibility=INTERNAL"
```

### 步骤 5：审核发布

1. 管理员进入审核队列
2. 查看安全扫描报告
3. 点击 **「批准发布」**
4. 状态从"审核中"变为"已发布"

---

## 五、版本更新流程

### 修改 SKILL.md 版本号

```yaml
---
name: prompt-engineering
version: "2.3.0"    # ← 递增版本号
description: ...
---
```

### 版本号规则

| 变更类型 | 版本递增 | 示例 |
|----------|---------|------|
| 修复 bug、小调整 | patch +1 | 2.2.0 → 2.2.1 |
| 新增功能、新增参考文件 | minor +1 | 2.2.0 → 2.3.0 |
| 破坏性变更、不兼容修改 | major +1 | 2.2.0 → 3.0.0 |

### 发布新版本

```bash
# SkillHub（腾讯版）
# 重新上传文件夹，填写新版本号和 Changelog

# ClawHub CLI
npx clawhub publish ./prompt-engineering

# 私有化版
npx clawhub publish ./prompt-engineering --namespace team-skills
```

> ⚠️ **已发布的版本不可修改**，只能发布新版本。如发现严重问题，将 `latest` 标签指向上一个稳定版本。

---

## 六、常见上传错误与解决方案

| 错误信息 | 原因 | 解决方案 |
|----------|------|---------|
| **Slug must be lowercase** | Slug 含大写字母 | 改为 `prompt-engineering`（全小写+短横线） |
| **Remove non-text files** | 含 `.git`/`.DS_Store`/`LICENSE` 等 | 删除这些文件后重新上传 |
| **SKILL.md not found** | 缺少核心配置文件 | 确保根目录有 `SKILL.md` |
| **Missing required field: description** | front matter 缺 description | 在 `---` 之间添加 `description` 字段 |
| **GitHub API rate limit exceeded** | GitHub 导入请求太频繁 | 等10分钟重试，或改用文件夹上传 |
| **Connection lost while action was in flight** | 网络波动或文件过大 | 刷新重试，或精简 references/ 目录 |
| **Version already exists** | 版本号重复 | 递增版本号 |
| **File size exceeds limit (100MB)** | 技能包超过100MB | 压缩或移除大文件 |
| **审核被拒：含敏感信息** | SKILL.md 或参考文件含密钥/Token | 搜索并删除 `sk-`、`api_key`、`password` 等模式 |
| **审核被拒：恶意代码检测** | scripts/ 中的脚本有可疑操作 | 检查脚本逻辑，避免 `rm -rf`、`curl|sh` 等危险命令 |
| **安全扫描失败** | 依赖漏洞或行为异常 | 查看扫描报告，修复后重新提交 |

---

## 七、上传前最终检查清单

```
□ SKILL.md 存在且 front matter 包含 name + description
□ name 使用 kebab-case 格式（prompt-engineering）
□ description 精简到1-2句话
□ 无 .git/、.DS_Store、LICENSE 等非文本文件
□ 无 .env、密钥、Token 等敏感信息
□ 所有 .md 文件编码为 UTF-8
□ 文件总大小 < 100MB
□ README.md 已创建（用户看到的第一印象）
□ Changelog 已准备（描述本次版本变更）
□ 准备好勾选 MIT-0 许可证
□ 版本号已确认（首次建议 1.0.0）
```

---

## 八、一键打包脚本

```powershell
# pack-for-skillhub.ps1
# 将 Skill 打包为适合上传的干净目录

$src = "d:\Prompt\.codebuddy\skills\prompt-engineering"
$out = "d:\Prompt\dist\prompt-engineering"

# 清理输出目录
if (Test-Path $out) { Remove-Item $out -Recurse -Force }
New-Item -ItemType Directory -Path $out -Force | Out-Null
New-Item -ItemType Directory -Path "$out\references" -Force | Out-Null

# 复制核心文件
Copy-Item "$src\SKILL.md" $out

# 如果还没有 README.md，创建一份
if (-not (Test-Path "$src\README.md")) {
    $readme = @"
# Prompt Engineering Expert Skill

> 提示词工程专家 — 让提示词编写从"凭感觉"变为"有纪律的工程过程"

## 一句话介绍

覆盖 18 种提示技术选型、6 种标准化工作流（设计/优化/调试/选型/模板/审查）、100+ 参数化模板和 15 项反模式避坑清单。

## 快速使用

在对话中直接输入触发词即可：
- ``帮我设计一个xxx的提示词``
- ``优化这个prompt：[粘贴你的提示词]``
- ``检查提示词有什么反模式``

## 兼容性

支持所有主流 LLM：Claude、GPT-4、DeepSeek-R1、Llama、Qwen 等。

## 版本

v2.2.0 | MIT-0 License
"@
    Set-Content -Path "$out\README.md" -Value $readme -Encoding UTF8
} else {
    Copy-Item "$src\README.md" $out
}

# 复制参考文件（只复制 .md 文件）
Get-ChildItem "$src\references\*.md" | ForEach-Object {
    Copy-Item $_.FullName "$out\references\"
}

# 检查敏感信息
Write-Host "`n=== 安全检查 ===" -ForegroundColor Yellow
$allMd = Get-ChildItem "$out" -Recurse -Filter "*.md"
$patterns = @("sk-[a-zA-Z0-9]{20,}", "api_key\s*=\s*[""][^""]+", "password\s*=\s*[""][^""]+", "secret\s*=\s*[""][^""]+")
$found = $false

foreach ($file in $allMd) {
    $content = Get-Content $file.FullName -Raw
    foreach ($pattern in $patterns) {
        if ($content -match $pattern) {
            Write-Host "⚠️ 发现敏感信息: $($file.Name) 匹配 $pattern" -ForegroundColor Red
            $found = $true
        }
    }
}

if (-not $found) {
    Write-Host "✓ 未发现敏感信息" -ForegroundColor Green
}

# 统计
$fileCount = (Get-ChildItem $out -Recurse -File).Count
$totalSize = [Math]::Round(((Get-ChildItem $out -Recurse | Measure-Object -Property Length -Sum).Sum / 1KB), 1)

Write-Host "`n=== 打包完成 ===" -ForegroundColor Green
Write-Host "输出目录: $out"
Write-Host "文件数量: $fileCount"
Write-Host "总大小: ${totalSize} KB"
Write-Host "`n下一步: 打开 https://skillhub.cn/import 上传 $out 目录"
```

使用方法：

```powershell
powershell -ExecutionPolicy Bypass -File pack-for-skillhub.ps1
```
