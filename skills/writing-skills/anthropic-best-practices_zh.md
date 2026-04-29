# 技能编写最佳实践

> 学习如何编写有效的技能，以便Claude能够成功发现和使用。

优秀的技能应当简洁、结构良好，并经过实际使用测试。本指南提供实用的编写决策，帮助您编写Claude能够有效发现和使用的技能。

关于技能工作原理的概念背景，请参阅[技能概述](/en/docs/agents-and-tools/agent-skills/overview)。

## 核心原则

### 简洁是关键

[上下文窗口](https://platform.claude.com/docs/en/build-with-claude/context-windows)是公共资源。您的技能与Claude需要知道的所有其他内容共享上下文窗口，包括：

* 系统提示
* 对话历史
* 其他技能的元数据
* 您的实际请求

并非技能中的每个标记都有即时成本。启动时，仅预加载所有技能的元数据（名称和描述）。Claude仅在技能变得相关时读取SKILL.md，并根据需要读取其他文件。然而，保持SKILL.md简洁仍然很重要：一旦Claude加载它，每个标记都会与对话历史和其他上下文竞争。

**默认假设**：Claude已经非常智能

只添加Claude尚未掌握的上下文。质疑每条信息：

* “Claude真的需要这个解释吗？”
* “我可以假设Claude知道这个吗？”
* “这段文字是否值得其标记成本？”

**良好示例：简洁**（约50个标记）：

````markdown  theme={null}
## 提取PDF文本

使用pdfplumber进行文本提取：

```python
import pdfplumber

with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```
````

**不良示例：过于冗长**（约150个标记）：

```markdown  theme={null}
## 提取PDF文本

PDF（便携式文档格式）文件是一种常见的文件格式，包含文本、图像和其他内容。要从PDF中提取文本，您需要使用一个库。有许多可用于PDF处理的库，但我们推荐pdfplumber，因为它易于使用且能处理大多数情况。首先，您需要使用pip安装它。然后您可以使用下面的代码...
```

简洁版本假设Claude知道PDF是什么以及库如何工作。

### 设置适当的自由度

将详细程度与任务的脆弱性和可变性相匹配。

**高自由度**（基于文本的指令）：

适用于：

* 多种方法都有效
* 决策取决于上下文
* 启发式方法指导操作

示例：

```markdown  theme={null}
## 代码审查流程

1. 分析代码结构和组织
2. 检查潜在错误或边缘情况
3. 提出可读性和可维护性改进建议
4. 验证是否符合项目约定
```

**中等自由度**（伪代码或带参数的脚本）：

适用于：

* 存在首选模式
* 允许一定变化
* 配置影响行为

示例：

````markdown  theme={null}
## 生成报告

使用此模板并根据需要自定义：

```python
def generate_report(data, format="markdown", include_charts=True):
    # 处理数据
    # 生成指定格式的输出
    # 可选包含可视化
```
````

**低自由度**（特定脚本，很少或没有参数）：

适用于：

* 操作脆弱且容易出错
* 一致性至关重要
* 必须遵循特定顺序

示例：

````markdown  theme={null}
## 数据库迁移

精确运行此脚本：

```bash
python scripts/migrate.py --verify --backup
```

不要修改命令或添加额外标志。
````

**类比**：将Claude视为探索路径的机器人：

* **两侧都是悬崖的狭窄桥梁**：只有一条安全的前进道路。提供具体的护栏和精确指令（低自由度）。示例：必须按精确顺序运行的数据库迁移。
* **没有危险的开放田野**：许多路径都能成功。给出一般方向并信任Claude找到最佳路线（高自由度）。示例：上下文决定最佳方法的代码审查。

### 在计划使用的所有模型上进行测试

技能作为模型的补充，因此有效性取决于底层模型。在计划使用的所有模型上测试您的技能。

**按模型考虑的测试因素**：

* **Claude Haiku**（快速、经济）：技能是否提供足够的指导？
* **Claude Sonnet**（平衡）：技能是否清晰高效？
* **Claude Opus**（强大推理）：技能是否避免过度解释？

对Opus完美有效的内容可能需要对Haiku提供更多细节。如果您计划在多个模型中使用技能，请确保指令在所有模型上都能良好工作。

## 技能结构

<Note>
  **YAML Frontmatter**：SKILL.md的frontmatter需要两个字段：

  * `name` - 技能的人类可读名称（最多64个字符）
  * `description` - 技能功能及使用时机的一行描述（最多1024个字符）

  有关完整技能结构详情，请参阅[技能概述](/en/docs/agents-and-tools/agent-skills/overview#skill-structure)。
</Note>

### 命名约定

使用一致的命名模式，使技能更易于引用和讨论。我们建议对技能名称使用**动名词形式**（动词+ing），因为这能清晰描述技能提供的活动或能力。

**良好命名示例（动名词形式）**：

* "处理PDF"
* "分析电子表格"
* "管理数据库"
* "测试代码"
* "编写文档"

**可接受的替代方案**：

* 名词短语："PDF处理"、"电子表格分析"
* 面向行动："处理PDF"、"分析电子表格"

**避免**：

* 模糊名称："助手"、"工具"、"实用程序"
* 过于通用："文档"、"数据"、"文件"
* 技能集合内的不一致模式

一致的命名使得：

* 在文档和对话中引用技能更容易
* 一目了然地理解技能功能
* 组织和搜索多个技能更容易
* 维护专业、连贯的技能库

### 编写有效描述

`description`字段支持技能发现，应包含技能功能和使用时机。

<Warning>
  **始终使用第三人称**。描述会注入到系统提示中，不一致的人称可能导致发现问题。

  * **良好**："处理Excel文件并生成报告"
  * **避免**："我可以帮助您处理Excel文件"
  * **避免**："您可以使用此功能处理Excel文件"
</Warning>

**具体并包含关键术语**。包括技能功能以及使用时的具体触发条件/上下文。

每个技能只有一个描述字段。描述对于技能选择至关重要：Claude使用它从可能100多个可用技能中选择正确的技能。您的描述必须提供足够的细节让Claude知道何时选择此技能，而SKILL.md的其余部分提供实现细节。

有效示例：

**PDF处理技能：**

```yaml  theme={null}
description: 从PDF文件中提取文本和表格，填写表单，合并文档。在处理PDF文件或用户提到PDF、表单或文档提取时使用。
```

**Excel分析技能：**

```yaml  theme={null}
description: 分析Excel电子表格，创建数据透视表，生成图表。在分析Excel文件、电子表格、表格数据或.xlsx文件时使用。
```

**Git提交助手技能：**

```yaml  theme={null}
description: 通过分析git差异生成描述性提交消息。在用户请求帮助编写提交消息或审查暂存更改时使用。
```

避免以下模糊描述：

```yaml  theme={null}
description: 帮助处理文档
```

```yaml  theme={null}
description: 处理数据
```

```yaml  theme={null}
description: 处理文件相关事务
```

### 渐进式披露模式

SKILL.md作为概述，根据需要指向详细材料，就像入职指南中的目录。有关渐进式披露工作原理的解释，请参阅概述中的[技能工作原理](/en/docs/agents-and-tools/agent-skills/overview#how-skills-work)。

**实用指导**：

* 保持SKILL.md主体在500行以内以获得最佳性能
* 接近此限制时将内容拆分为单独文件
* 使用以下模式有效组织指令、代码和资源

#### 视觉概述：从简单到复杂

基本技能仅包含一个SKILL.md文件，其中包含元数据和指令：

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=87782ff239b297d9a9e8e1b72ed72db9" alt="简单的SKILL.md文件显示YAML frontmatter和markdown主体" data-og-width="2048" width="2048" data-og-height="1153" height="1153" data-path="images/agent-skills-simple-file.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=c61cc33b6f5855809907f7fda94cd80e 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=90d2c0c1c76b36e8d485f49e0810dbfd 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=ad17d231ac7b0bea7e5b4d58fb4aeabb 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=f5d0a7a3c668435bb0aee9a3a8f8c329 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=0e927c1af9de5799cfe557d12249f6e6 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-simple-file.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=46bbb1a51dd4c8202a470ac8c80a893d 2500w" />

随着技能增长，您可以捆绑Claude仅在需要时加载的额外内容：

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=a5e0aa41e3d53985a7e3e43668a33ea3" alt="捆绑额外参考文件如reference.md和forms.md。" data-og-width="2048" width="2048" data-og-height="1327" height="1327" data-path="images/agent-skills-bundling-content.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=f8a0e73783e99b4a643d79eac86b70a2 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=dc510a2a9d3f14359416b706f067904a 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=82cd6286c966303f7dd914c28170e385 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=56f3be36c77e4fe4b523df209a6824c6 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=d22b5161b2075656417d56f41a74f3dd 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-bundling-content.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=3dd4bdd6850ffcc96c6c45fcb0acd6eb 2500w" />

完整的技能目录结构可能如下所示：

```
pdf/
├── SKILL.md              # 主要指令（触发时加载）
├── FORMS.md              # 表单填写指南（需要时加载）
├── reference.md          # API参考（需要时加载）
├── examples.md           # 使用示例（需要时加载）
└── scripts/
    ├── analyze_form.py   # 实用脚本（执行，不加载）
    ├── fill_form.py      # 表单填写脚本
    └── validate.py       # 验证脚本
```

#### 模式1：高级指南与参考

````markdown  theme={null}
---
name: PDF处理
description: 从PDF文件中提取文本和表格，填写表单，合并文档。在处理PDF文件或用户提到PDF、表单或文档提取时使用。
---

# PDF处理

## 快速开始

使用pdfplumber提取文本：
```python
import pdfplumber
with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```

## 高级功能

**表单填写**：完整指南请参阅[FORMS.md](FORMS.md)
**API参考**：所有方法请参阅[REFERENCE.md](REFERENCE.md)
**示例**：常见模式请参阅[EXAMPLES.md](EXAMPLES.md)
````

Claude仅在需要时加载FORMS.md、REFERENCE.md或EXAMPLES.md。

#### 模式2：领域特定组织

对于具有多个领域的技能，按领域组织内容以避免加载无关上下文。当用户询问销售指标时，Claude只需要读取销售相关模式，而不是财务或营销数据。这可以保持标记使用量低且上下文集中。

```
bigquery-skill/
├── SKILL.md (概述和导航)
└── reference/
    ├── finance.md (收入、账单指标)
    ├── sales.md (机会、管道)
    ├── product.md (API使用、功能)
    └── marketing.md (活动、归因)
```

````markdown SKILL.md theme={null}
# BigQuery数据分析

## 可用数据集

**财务**：收入、ARR、账单 → 参阅[reference/finance.md](reference/finance.md)
**销售**：机会、管道、账户 → 参阅[reference/sales.md](reference/sales.md)
**产品**：API使用、功能、采用 → 参阅[reference/product.md](reference/product.md)
**营销**：活动、归因、电子邮件 → 参阅[reference/marketing.md](reference/marketing.md)

## 快速搜索

使用grep查找特定指标：

```bash
grep -i "revenue" reference/finance.md
grep -i "pipeline" reference/sales.md
grep -i "api usage" reference/product.md
```
````

#### 模式3：条件性细节

显示基本内容，链接到高级内容：

```markdown  theme={null}
# DOCX处理

## 创建文档

使用docx-js创建新文档。参阅[DOCX-JS.md](DOCX-JS.md)。

## 编辑文档

对于简单编辑，直接修改XML。

**对于跟踪更改**：参阅[REDLINING.md](REDLINING.md)
**对于OOXML细节**：参阅[OOXML.md](OOXML.md)
```

Claude仅在用户需要这些功能时读取REDLINING.md或OOXML.md。

### 避免深层嵌套引用

当从其他引用文件引用时，Claude可能会部分读取文件。遇到嵌套引用时，Claude可能使用`head -100`等命令预览内容而不是读取整个文件，导致信息不完整。

**保持引用从SKILL.md开始仅一级深度**。所有引用文件都应直接从SKILL.md链接，以确保Claude在需要时读取完整文件。

**不良示例：太深**：

```markdown  theme={null}
# SKILL.md
参阅[advanced.md](advanced.md)...

# advanced.md
参阅[details.md](details.md)...

# details.md
这是实际信息...
```

**良好示例：一级深度**：

```markdown  theme={null}
# SKILL.md

**基本用法**：[SKILL.md中的指令]
**高级功能**：参阅[advanced.md](advanced.md)
**API参考**：参阅[reference.md](reference.md)
**示例**：参阅[examples.md](examples.md)
```

### 使用目录结构组织较长的参考文件

对于超过100行的参考文件，在顶部包含目录。这确保Claude即使在部分读取预览时也能看到可用信息的完整范围。

**示例**：

```markdown  theme={null}
# API参考

## 目录
- 身份验证和设置
- 核心方法（创建、读取、更新、删除）
- 高级功能（批量操作、Webhooks）
- 错误处理模式
- 代码示例

## 身份验证和设置
...

## 核心方法
...
```

然后Claude可以根据需要读取完整文件或跳转到特定部分。

有关这种基于文件系统的架构如何实现渐进式披露的详细信息，请参阅下面高级部分中的[运行时环境](#runtime-environment)部分。

## 工作流程和反馈循环

### 对复杂任务使用工作流程

将复杂操作分解为清晰、顺序的步骤。对于特别复杂的工作流程，提供一个检查清单，Claude可以复制到其响应中并在进展过程中勾选。

**示例1：研究综合工作流程**（适用于无代码技能）：

````markdown  theme={null}
## 研究综合工作流程

复制此检查清单并跟踪进度：

```
研究进度：
- [ ] 步骤1：阅读所有源文档
- [ ] 步骤2：识别关键主题
- [ ] 步骤3：交叉验证主张
- [ ] 步骤4：创建结构化摘要
- [ ] 步骤5：验证引用
```

**步骤1：阅读所有源文档**

审查`sources/`目录中的每个文档。记录主要论点和支持证据。

**步骤2：识别关键主题**

寻找跨来源的模式。哪些主题反复出现？来源在哪些方面一致或分歧？

**步骤3：交叉验证主张**

对于每个主要主张，验证其是否出现在源材料中。记录每个观点的支持来源。

**步骤4：创建结构化摘要**

按主题组织发现。包括：
- 主要主张
- 来源的支持证据
- 冲突观点（如果有）

**步骤5：验证引用**

检查每个主张是否引用了正确的源文档。如果引用不完整，返回步骤3。
````

此示例展示了工作流程如何应用于不需要代码的分析任务。检查清单模式适用于任何复杂、多步骤的过程。

**示例2：PDF表单填写工作流程**（适用于有代码技能）：

````markdown  theme={null}
## PDF表单填写工作流程

复制此检查清单并在完成项目时勾选：

```
任务进度：
- [ ] 步骤1：分析表单（运行analyze_form.py）
- [ ] 步骤2：创建字段映射（编辑fields.json）
- [ ] 步骤3：验证映射（运行validate_fields.py）
- [ ] 步骤4：填写表单（运行fill_form.py）
- [ ] 步骤5：验证输出（运行verify_output.py）
```

**步骤1：分析表单**

运行：`python scripts/analyze_form.py input.pdf`

这将提取表单字段及其位置，保存到`fields.json`。

**步骤2：创建字段映射**

编辑`fields.json`为每个字段添加值。

**步骤3：验证映射**

运行：`python scripts/validate_fields.py fields.json`

在继续之前修复任何验证错误。

**步骤4：填写表单**

运行：`python scripts/fill_form.py input.pdf fields.json output.pdf`

**步骤5：验证输出**

运行：`python scripts/verify_output.py output.pdf`

如果验证失败，返回步骤2。
````

清晰的步骤防止Claude跳过关键验证。检查清单帮助Claude和您跟踪多步骤工作流程的进度。

### 实现反馈循环

**常见模式**：运行验证器 → 修复错误 → 重复

此模式大大提高了输出质量。

**示例1：样式指南合规性**（适用于无代码技能）：

```markdown  theme={null}
## 内容审查流程

1. 按照STYLE_GUIDE.md中的指南起草内容
2. 对照检查清单审查：
   - 检查术语一致性
   - 验证示例是否遵循标准格式
   - 确认所有必需部分都存在
3. 如果发现问题：
   - 记录每个问题并注明具体部分引用
   - 修订内容
   - 再次审查检查清单
4. 仅在所有要求满足时继续
5. 最终确定并保存文档
```

这展示了使用参考文档而非脚本的验证循环模式。"验证器"是STYLE\_GUIDE.md，Claude通过读取和比较来执行检查。

**示例2：文档编辑流程**（适用于有代码技能）：

```markdown  theme={null}
## 文档编辑流程

1. 对`word/document.xml`进行编辑
2. **立即验证**：`python ooxml/scripts/validate.py unpacked_dir/`
3. 如果验证失败：
   - 仔细查看错误消息
   - 修复XML中的问题
   - 再次运行验证
4. **仅在验证通过时继续**
5. 重建：`python ooxml/scripts/pack.py unpacked_dir/ output.docx`
6. 测试输出文档
```

验证循环及早捕获错误。

## 内容指南

### 避免时间敏感信息

不要包含会过时的信息：

**不良示例：时间敏感**（会变得错误）：

```markdown  theme={null}
如果在2025年8月之前执行此操作，请使用旧API。
2025年8月之后，请使用新API。
```

**良好示例**（使用"旧模式"部分）：

```markdown  theme={null}
## 当前方法

使用v2 API端点：`api.example.com/v2/messages`

## 旧模式

<details>
<summary>遗留v1 API（2025年8月弃用）</summary>

v1 API使用：`api.example.com/v1/messages`

此端点不再受支持。
</details>
```

旧模式部分提供历史上下文，而不会使主要内容混乱。

### 使用一致的术语

选择一个术语并在整个技能中一致使用：

**良好 - 一致**：

* 始终使用"API端点"
* 始终使用"字段"
* 始终使用"提取"

**不良 - 不一致**：

* 混合使用"API端点"、"URL"、"API路由"、"路径"
* 混合使用"字段"、"框"、"元素"、"控件"
* 混合使用"提取"、"拉取"、"获取"、"检索"

一致性帮助Claude理解和遵循指令。

## 常见模式

### 模板模式

为输出格式提供模板。根据需求匹配严格程度。

**对于严格要求**（如API响应或数据格式）：

````markdown  theme={null}
## 报告结构

始终使用此精确模板结构：

```markdown
# [分析标题]

## 执行摘要
[关键发现的一段概述]

## 关键发现
- 发现1及支持数据
- 发现2及支持数据
- 发现3及支持数据

## 建议
1. 具体可操作建议
2. 具体可操作建议
```
````

**对于灵活指导**（当适应有用时）：

````markdown  theme={null}
## 报告结构

这是一个合理的默认格式，但根据分析使用最佳判断：

```markdown
# [分析标题]

## 执行摘要
[概述]

## 关键发现
[根据发现调整部分]

## 建议
[根据特定上下文定制]
```

根据分析类型调整部分。
````

### 示例模式

对于输出质量取决于看到示例的技能，提供输入/输出对，就像常规提示中一样：

````markdown  theme={null}
## 提交消息格式

按照以下示例生成提交消息：

**示例1：**
输入：添加了带有JWT令牌的用户身份验证
输出：
```
feat(auth): 实现基于JWT的身份验证

添加登录端点和令牌验证中间件
```

**示例2：**
输入：修复了报告中日期显示不正确的错误
输出：
```
fix(reports): 修正时区转换中的日期格式

在报告生成中一致使用UTC时间戳
```

**示例3：**
输入：更新依赖项并重构错误处理
输出：
```
chore: 更新依赖项并重构错误处理

- 将lodash升级到4.17.21
- 标准化端点间的错误响应格式
```

遵循此风格：类型(范围): 简要描述，然后详细解释。
````

示例帮助Claude比单独描述更清楚地理解所需风格和详细程度。

### 条件工作流程模式

指导Claude通过决策点：

```markdown  theme={null}
## 文档修改工作流程

1. 确定修改类型：

   **创建新内容？** → 遵循下面的"创建工作流程"
   **编辑现有内容？** → 遵循下面的"编辑工作流程"

2. 创建工作流程：
   - 使用docx-js库
   - 从头构建文档
   - 导出为.docx格式

3. 编辑工作流程：
   - 解包现有文档
   - 直接修改XML
   - 每次更改后验证
   - 完成后重新打包
```

<Tip>
  如果工作流程变得庞大或步骤复杂，考虑将它们推入单独文件，并告诉Claude根据任务读取适当的文件。
</Tip>

## 评估和迭代

### 先构建评估

**在编写大量文档之前创建评估**。这确保您的技能解决实际问题，而不是记录想象的问题。

**评估驱动开发**：

1. **识别差距**：在没有技能的情况下在代表性任务上运行Claude。记录具体失败或缺失的上下文
2. **创建评估**：构建三个测试这些差距的场景
3. **建立基线**：测量没有技能时Claude的性能
4. **编写最小指令**：创建足够的内容来弥补差距并通过评估
5. **迭代**：执行评估，与基线比较，并改进

这种方法确保您解决实际问题，而不是预测可能永远不会出现的需求。

**评估结构**：

```json  theme={null}
{
  "skills": ["pdf-processing"],
  "query": "从此PDF文件中提取所有文本并保存到output.txt",
  "files": ["test-files/document.pdf"],
  "expected_behavior": [
    "使用适当的PDF处理库或命令行工具成功读取PDF文件",
    "从文档的所有页面提取文本内容，不遗漏任何页面",
    "以清晰、可读的格式将提取的文本保存到名为output.txt的文件中"
  ]
}
```

<Note>
  此示例演示了具有简单测试标准的基于数据的评估。我们目前不提供运行这些评估的内置方法。用户可以创建自己的评估系统。评估是衡量技能有效性的真实来源。
</Note>

### 与Claude一起迭代开发技能

最有效的技能开发过程涉及Claude本身。与一个Claude实例（"Claude A"）合作创建将被其他实例（"Claude B"）使用的技能。Claude A帮助您设计和改进指令，而Claude B在实际任务中测试它们。这是有效的，因为Claude模型既理解如何编写有效的代理指令，也理解代理需要什么信息。

**创建新技能**：

1. **在没有技能的情况下完成任务**：使用正常提示与Claude A一起解决问题。在工作过程中，您自然会提供上下文、解释偏好并分享程序性知识。注意您反复提供的信息。

2. **识别可重用模式**：完成任务后，识别您提供的哪些上下文对类似未来任务有用。

   **示例**：如果您完成了BigQuery分析，您可能提供了表名、字段定义、过滤规则（如"始终排除测试账户"）和常见查询模式。

3. **要求Claude A创建技能**："创建一个捕获我们刚刚使用的BigQuery分析模式的技能。包括表模式、命名约定以及关于过滤测试账户的规则。"

   <Tip>
     Claude模型原生理解技能格式和结构。您不需要特殊的系统提示或"编写技能"技能来让Claude帮助创建技能。只需要求Claude创建技能，它就会生成适当结构的SKILL.md内容，包含适当的frontmatter和主体内容。
   </Tip>

4. **审查简洁性**：检查Claude A是否添加了不必要的解释。询问："删除关于胜率含义的解释 - Claude已经知道这一点。"

5. **改进信息架构**：要求Claude A更有效地组织内容。例如："重新组织此内容，使表模式放在单独的参考文件中。我们以后可能会添加更多表。"

6. **在类似任务上测试**：在相关用例上使用带有技能的Claude B（加载了技能的新实例）。观察Claude B是否找到正确的信息，正确应用规则，并成功处理任务。

7. **基于观察迭代**：如果Claude B遇到困难或遗漏某些内容，返回Claude A并提供具体信息："当Claude使用此技能时，它忘记为Q4按日期过滤。我们应该添加关于日期过滤模式的部分吗？"

**迭代现有技能**：

改进技能时，相同的分层模式继续。您在以下两者之间交替：

* **与Claude A合作**（帮助改进技能的专家）
* **使用Claude B测试**（使用技能执行实际工作的代理）
* **观察Claude B的行为**并将见解带回Claude A

1. **在实际工作流程中使用技能**：给加载了技能的Claude B实际任务，而不是测试场景

2. **观察Claude B的行为**：注意它在哪些方面遇到困难、成功或做出意外选择

   **示例观察**："当我要求Claude B提供区域销售报告时，它编写了查询但忘记过滤掉测试账户，即使技能提到了此规则。"

3. **返回Claude A进行改进**：分享当前的SKILL.md并描述您的观察。询问："我注意到当我要求区域报告时，Claude B忘记过滤测试账户。技能提到了过滤，但可能不够突出？"

4. **审查Claude A的建议**：Claude A可能建议重新组织以使规则更突出，使用更强的语言如"必须过滤"而不是"始终过滤"，或重构工作流程部分。

5. **应用和测试更改**：使用Claude A的改进更新技能，然后在类似请求上再次使用Claude B测试

6. **基于使用重复**：在遇到新场景时继续此观察-改进-测试循环。每次迭代都基于实际代理行为而非假设改进技能。

**收集团队反馈**：

1. 与团队成员分享技能并观察他们的使用情况
2. 询问：技能是否在预期时激活？指令是否清晰？缺少什么？
3. 纳入反馈以解决您自己使用模式中的盲点

**为什么这种方法有效**：Claude A理解代理需求，您提供领域专业知识，Claude B通过实际使用揭示差距，迭代改进基于观察到的行为而非假设改进技能。

### 观察Claude如何导航技能

在迭代技能时，注意Claude实际如何使用它们。观察：

* **意外的探索路径**：Claude是否以您未预期的顺序读取文件？这可能表明您的结构不如您想象的直观
* **错过的连接**：Claude是否未能遵循重要文件的引用？您的链接可能需要更明确或更突出
* **过度依赖某些部分**：如果Claude反复读取同一文件，考虑该内容是否应放在主SKILL.md中
* **忽略的内容**：如果Claude从未访问捆绑文件，它可能是不必要的或在主指令中信号不佳

基于这些观察而非假设进行迭代。技能元数据中的'name'和'description'特别关键。Claude在决定是否针对当前任务触发技能时使用这些。确保它们清晰描述技能功能以及何时应使用。

## 要避免的反模式

### 避免Windows风格路径

始终在文件路径中使用正斜杠，即使在Windows上：

* ✓ **良好**：`scripts/helper.py`、`reference/guide.md`
* ✗ **避免**：`scripts\helper.py`、`reference\guide.md`

Unix风格路径在所有平台上都有效，而Windows风格路径在Unix系统上会导致错误。

### 避免提供太多选项

除非必要，不要呈现多种方法：

````markdown  theme={null}
**不良示例：太多选择**（令人困惑）：

"你可以使用 pypdf，或者 pdfplumber，或者 PyMuPDF，或者 pdf2image，或者……"

**良好示例：提供默认方案**（并预留备用方案）：
"使用 pdfplumber 进行文本提取：
```python
import pdfplumber
```

对于需要 OCR 的扫描版 PDF，请改用 pdf2image 配合 pytesseract。"
````

## 进阶：包含可执行代码的技能

以下部分重点介绍包含可执行脚本的技能。如果你的技能仅使用 Markdown 说明，请跳转到[有效技能检查清单](#checklist-for-effective-skills)。

### 解决问题，而非推诿

为技能编写脚本时，应处理错误条件，而不是推给 Claude 处理。

**良好示例：显式处理错误**：

```python  theme={null}
def process_file(path):
    """处理文件，若不存在则创建。"""
    try:
        with open(path) as f:
            return f.read()
    except FileNotFoundError:
        # 文件不存在时创建默认文件，而非直接失败
        print(f"文件 {path} 未找到，创建默认文件")
        with open(path, 'w') as f:
            f.write('')
        return ''
    except PermissionError:
        # 提供替代方案，而非直接失败
        print(f"无法访问 {path}，使用默认值")
        return ''
```

**不良示例：推诿给 Claude**：

```python  theme={null}
def process_file(path):
    # 直接失败，让 Claude 去解决
    return open(path).read()
```

配置参数也应加以说明和记录，避免出现"魔法常量"（Ousterhout 定律）。如果你不知道正确的值，Claude 又如何确定呢？

**良好示例：自解释代码**：

```python  theme={null}
# HTTP 请求通常在 30 秒内完成
# 较长的超时时间考虑了慢速连接
REQUEST_TIMEOUT = 30

# 三次重试在可靠性和速度之间取得平衡
# 大多数间歇性故障在第二次重试时已解决
MAX_RETRIES = 3
```

**不良示例：魔法数字**：

```python  theme={null}
TIMEOUT = 47  # 为什么是 47？
RETRIES = 5   # 为什么是 5？
```

### 提供实用脚本

即使 Claude 可以编写脚本，预制的脚本也有其优势：

**实用脚本的好处**：

* 比生成的代码更可靠
* 节省 token（无需在上下文中包含代码）
* 节省时间（无需生成代码）
* 确保跨使用场景的一致性

<img src="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=4bbc45f2c2e0bee9f2f0d5da669bad00" alt="将可执行脚本与说明文件捆绑" data-og-width="2048" width="2048" data-og-height="1154" height="1154" data-path="images/agent-skills-executable-scripts.png" data-optimize="true" data-opv="3" srcset="https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=280&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=9a04e6535a8467bfeea492e517de389f 280w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=560&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=e49333ad90141af17c0d7651cca7216b 560w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=840&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=954265a5df52223d6572b6214168c428 840w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=1100&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=2ff7a2d8f2a83ee8af132b29f10150fd 1100w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=1650&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=48ab96245e04077f4d15e9170e081cfb 1650w, https://mintcdn.com/anthropic-claude-docs/4Bny2bjzuGBK7o00/images/agent-skills-executable-scripts.png?w=2500&fit=max&auto=format&n=4Bny2bjzuGBK7o00&q=85&s=0301a6c8b3ee879497cc5b5483177c90 2500w" />

上图展示了可执行脚本如何与说明文件协同工作。说明文件（forms.md）引用脚本，Claude 可以执行脚本而无需将其内容加载到上下文中。

**重要区别**：在你的说明中明确 Claude 应该：

* **执行脚本**（最常见）："运行 `analyze_form.py` 以提取字段"
* **将其作为参考阅读**（用于复杂逻辑）："查看 `analyze_form.py` 了解字段提取算法"

对于大多数实用脚本，首选执行方式，因为它更可靠和高效。有关脚本执行工作原理的详细信息，请参阅下面的[运行时环境](#runtime-environment)部分。

**示例**：

````markdown  theme={null}
## 实用脚本

**analyze_form.py**：从 PDF 提取所有表单字段

```bash
python scripts/analyze_form.py input.pdf > fields.json
```

输出格式：
```json
{
  "field_name": {"type": "text", "x": 100, "y": 200},
  "signature": {"type": "sig", "x": 150, "y": 500}
}
```

**validate_boxes.py**：检查重叠的边界框

```bash
python scripts/validate_boxes.py fields.json
# 返回："OK" 或列出冲突
```

**fill_form.py**：将字段值应用到 PDF

```bash
python scripts/fill_form.py input.pdf fields.json output.pdf
```
````

### 使用视觉分析

当输入可以渲染为图像时，让 Claude 对其进行分析：

````markdown  theme={null}
## 表单布局分析

1. 将 PDF 转换为图像：
   ```bash
   python scripts/pdf_to_images.py form.pdf
   ```

2. 分析每个页面图像以识别表单字段
3. Claude 可以直观地看到字段位置和类型
````

<Note>
  在此示例中，你需要编写 `pdf_to_images.py` 脚本。
</Note>

Claude 的视觉能力有助于理解布局和结构。

### 创建可验证的中间输出

当 Claude 执行复杂、开放式的任务时，可能会出错。"计划-验证-执行"模式通过让 Claude 首先以结构化格式创建计划，然后使用脚本验证该计划，最后再执行它，从而及早发现错误。

**示例**：假设要求 Claude 根据电子表格更新 PDF 中的 50 个表单字段。如果没有验证，Claude 可能会引用不存在的字段、创建冲突的值、遗漏必填字段或错误地应用更新。

**解决方案**：使用上面所示的工作流模式（PDF 表单填充），但添加一个中间 `changes.json` 文件，在应用更改之前进行验证。工作流变为：分析 → **创建计划文件** → **验证计划** → 执行 → 验证。

**此模式有效的原因：**

* **及早发现错误**：验证在应用更改之前发现问题
* **机器可验证**：脚本提供客观验证
* **可逆规划**：Claude 可以在不触及原始文件的情况下迭代计划
* **清晰调试**：错误消息指向具体问题

**何时使用**：批量操作、破坏性更改、复杂验证规则、高风险操作。

**实现技巧**：使验证脚本详细，并提供具体的错误消息，如"字段 'signature_date' 未找到。可用字段：customer_name, order_total, signature_date_signed"，以帮助 Claude 修复问题。

### 打包依赖项

技能在代码执行环境中运行，存在平台特定的限制：

* **claude.ai**：可以从 npm 和 PyPI 安装包，并从 GitHub 仓库拉取
* **Anthropic API**：无网络访问权限，无法在运行时安装包

在你的 SKILL.md 中列出所需的包，并验证它们在[代码执行工具文档](/en/docs/agents-and-tools/tool-use/code-execution-tool)中可用。

### 运行时环境

技能在具有文件系统访问权限、bash 命令和代码执行能力的代码执行环境中运行。有关此架构的概念性解释，请参阅概述中的[技能架构](/en/docs/agents-and-tools/agent-skills/overview#the-skills-architecture)。

**这对你的创作有何影响：**

**Claude 如何访问技能：**

1. **元数据预加载**：启动时，所有技能的 YAML frontmatter 中的名称和描述都会加载到系统提示中
2. **按需读取文件**：Claude 使用 bash Read 工具在需要时从文件系统访问 SKILL.md 和其他文件
3. **高效执行脚本**：实用脚本可以通过 bash 执行，而无需将其完整内容加载到上下文中。只有脚本的输出会消耗 token
4. **大文件无上下文惩罚**：参考文件、数据或文档在真正被读取之前不消耗上下文 token

* **文件路径很重要**：Claude 像文件系统一样导航你的技能目录。使用正斜杠（`reference/guide.md`），而非反斜杠
* **描述性命名文件**：使用能表明内容的名称：`form_validation_rules.md`，而非 `doc2.md`
* **为发现而组织**：按领域或功能组织目录结构
  * 良好：`reference/finance.md`、`reference/sales.md`
  * 不良：`docs/file1.md`、`docs/file2.md`
* **捆绑全面资源**：包含完整的 API 文档、大量示例、大型数据集；在访问前无上下文惩罚
* **确定性操作首选脚本**：编写 `validate_form.py`，而非要求 Claude 生成验证代码
* **明确执行意图**：
  * "运行 `analyze_form.py` 以提取字段"（执行）
  * "查看 `analyze_form.py` 了解提取算法"（作为参考阅读）
* **测试文件访问模式**：通过真实请求测试，验证 Claude 可以导航你的目录结构

**示例：**

```
bigquery-skill/
├── SKILL.md（概述，指向参考文件）
└── reference/
    ├── finance.md（收入指标）
    ├── sales.md（管道数据）
    └── product.md（使用分析）
```

当用户询问收入时，Claude 读取 SKILL.md，看到对 `reference/finance.md` 的引用，并调用 bash 仅读取该文件。sales.md 和 product.md 文件保留在文件系统上，在需要之前消耗零上下文 token。这种基于文件系统的模型使得渐进式披露成为可能。Claude 可以导航并有选择地加载每个任务所需的确切内容。

有关技术架构的完整详细信息，请参阅技能概述中的[技能工作原理](/en/docs/agents-and-tools/agent-skills/overview#how-skills-work)。

### MCP 工具引用

如果你的技能使用 MCP（模型上下文协议）工具，请始终使用完全限定的工具名称，以避免"工具未找到"错误。

**格式**：`ServerName:tool_name`

**示例**：

```markdown  theme={null}
使用 BigQuery:bigquery_schema 工具检索表模式。
使用 GitHub:create_issue 工具创建问题。
```

其中：

* `BigQuery` 和 `GitHub` 是 MCP 服务器名称
* `bigquery_schema` 和 `create_issue` 是这些服务器内的工具名称

没有服务器前缀，Claude 可能无法定位工具，尤其是在有多个 MCP 服务器可用时。

### 避免假设工具已安装

不要假设包已可用：

````markdown  theme={null}
**不良示例：假设已安装**：
"使用 pdf 库处理文件。"

**良好示例：明确依赖关系**：
"安装所需包：`pip install pypdf`

然后使用它：
```python
from pypdf import PdfReader
reader = PdfReader("file.pdf")
```"
````

## 技术说明

### YAML frontmatter 要求

SKILL.md 的 frontmatter 需要 `name`（最多 64 个字符）和 `description`（最多 1024 个字符）字段。有关完整结构详情，请参阅[技能概述](/en/docs/agents-and-tools/agent-skills/overview#skill-structure)。

### Token 预算

为获得最佳性能，请将 SKILL.md 正文保持在 500 行以内。如果你的内容超过此限制，请使用前面描述的渐进式披露模式将其拆分为单独的文件。有关架构细节，请参阅[技能概述](/en/docs/agents-and-tools/agent-skills/overview#how-skills-work)。

## 有效技能检查清单

分享技能前，请验证：

### 核心质量

* [ ] 描述具体且包含关键术语
* [ ] 描述包括技能的功能和使用时机
* [ ] SKILL.md 正文在 500 行以内
* [ ] 额外细节在单独文件中（如需要）
* [ ] 无时效性信息（或在"旧模式"部分）
* [ ] 术语使用一致
* [ ] 示例具体而非抽象
* [ ] 文件引用仅一层深度
* [ ] 适当使用渐进式披露
* [ ] 工作流步骤清晰

### 代码和脚本

* [ ] 脚本解决问题而非推诿给 Claude
* [ ] 错误处理明确且有用
* [ ] 无"魔法常量"（所有值均有说明）
* [ ] 所需包在说明中列出并验证可用
* [ ] 脚本有清晰的文档
* [ ] 无 Windows 风格路径（全部使用正斜杠）
* [ ] 关键操作有验证/确认步骤
* [ ] 质量关键任务包含反馈循环

### 测试

* [ ] 至少创建三个评估
* [ ] 使用 Haiku、Sonnet 和 Opus 测试
* [ ] 使用真实使用场景测试
* [ ] 纳入团队反馈（如适用）

## 后续步骤

<CardGroup cols={2}>
  <Card title="开始使用 Agent Skills" icon="rocket" href="/en/docs/agents-and-tools/agent-skills/quickstart">
    创建你的第一个技能
  </Card>

  <Card title="在 Claude Code 中使用技能" icon="terminal" href="/en/docs/claude-code/skills">
    在 Claude Code 中创建和管理技能
  </Card>

  <Card title="通过 API 使用技能" icon="code" href="/en/api/skills-guide">
    以编程方式上传和使用技能
  </Card>
</CardGroup>
