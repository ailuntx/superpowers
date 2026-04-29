# OpenCode 支持实现计划

> **对于智能体工作者：** 必需子技能：使用超能力：执行计划来逐步实施此计划任务。

**目标：** 为 OpenCode.ai 添加完整的超能力支持，通过一个原生 JavaScript 插件，与现有的 Codex 实现共享核心功能。

**架构：** 将通用的技能发现/解析逻辑提取到 `lib/skills-core.js` 中，重构 Codex 以使用它，然后使用 OpenCode 的原生插件 API 构建 OpenCode 插件，包含自定义工具和会话钩子。

**技术栈：** Node.js, JavaScript, OpenCode 插件 API, Git 工作树

---

## 第一阶段：创建共享核心模块

### 任务 1：提取 Frontmatter 解析

**文件：**
- 创建：`lib/skills-core.js`
- 参考：`.codex/superpowers-codex` (第 40-74 行)

**步骤 1：创建 lib/skills-core.js 并包含 extractFrontmatter 函数**

```javascript
#!/usr/bin/env node

const fs = require('fs');
const path = require('path');

/**
 * 从技能文件中提取 YAML frontmatter。
 * 当前格式：
 * ---
 * name: skill-name
 * description: Use when [condition] - [what it does]
 * ---
 *
 * @param {string} filePath - SKILL.md 文件的路径
 * @returns {{name: string, description: string}}
 */
function extractFrontmatter(filePath) {
    try {
        const content = fs.readFileSync(filePath, 'utf8');
        const lines = content.split('\n');

        let inFrontmatter = false;
        let name = '';
        let description = '';

        for (const line of lines) {
            if (line.trim() === '---') {
                if (inFrontmatter) break;
                inFrontmatter = true;
                continue;
            }

            if (inFrontmatter) {
                const match = line.match(/^(\w+):\s*(.*)$/);
                if (match) {
                    const [, key, value] = match;
                    switch (key) {
                        case 'name':
                            name = value.trim();
                            break;
                        case 'description':
                            description = value.trim();
                            break;
                    }
                }
            }
        }

        return { name, description };
    } catch (error) {
        return { name: '', description: '' };
    }
}

module.exports = {
    extractFrontmatter
};
```

**步骤 2：验证文件已创建**

运行：`ls -l lib/skills-core.js`
预期：文件存在

**步骤 3：提交**

```bash
git add lib/skills-core.js
git commit -m "feat: 创建共享技能核心模块，包含 frontmatter 解析器"
```

---

### 任务 2：提取技能发现逻辑

**文件：**
- 修改：`lib/skills-core.js`
- 参考：`.codex/superpowers-codex` (第 97-136 行)

**步骤 1：向 skills-core.js 添加 findSkillsInDir 函数**

在 `module.exports` 之前添加：

```javascript
/**
 * 递归查找目录中的所有 SKILL.md 文件。
 *
 * @param {string} dir - 要搜索的目录
 * @param {string} sourceType - 用于命名空间的 'personal' 或 'superpowers'
 * @param {number} maxDepth - 最大递归深度（默认：3）
 * @returns {Array<{path: string, name: string, description: string, sourceType: string}>}
 */
function findSkillsInDir(dir, sourceType, maxDepth = 3) {
    const skills = [];

    if (!fs.existsSync(dir)) return skills;

    function recurse(currentDir, depth) {
        if (depth > maxDepth) return;

        const entries = fs.readdirSync(currentDir, { withFileTypes: true });

        for (const entry of entries) {
            const fullPath = path.join(currentDir, entry.name);

            if (entry.isDirectory()) {
                // 检查此目录中是否有 SKILL.md
                const skillFile = path.join(fullPath, 'SKILL.md');
                if (fs.existsSync(skillFile)) {
                    const { name, description } = extractFrontmatter(skillFile);
                    skills.push({
                        path: fullPath,
                        skillFile: skillFile,
                        name: name || entry.name,
                        description: description || '',
                        sourceType: sourceType
                    });
                }

                // 递归进入子目录
                recurse(fullPath, depth + 1);
            }
        }
    }

    recurse(dir, 0);
    return skills;
}
```

**步骤 2：更新 module.exports**

将导出行替换为：

```javascript
module.exports = {
    extractFrontmatter,
    findSkillsInDir
};
```

**步骤 3：验证语法**

运行：`node -c lib/skills-core.js`
预期：无输出（成功）

**步骤 4：提交**

```bash
git add lib/skills-core.js
git commit -m "feat: 向核心模块添加技能发现函数"
```

---

### 任务 3：提取技能路径解析逻辑

**文件：**
- 修改：`lib/skills-core.js`
- 参考：`.codex/superpowers-codex` (第 212-280 行)

**步骤 1：添加 resolveSkillPath 函数**

在 `module.exports` 之前添加：

```javascript
/**
 * 将技能名称解析为其文件路径，处理覆盖（个人技能覆盖超能力技能）。
 *
 * @param {string} skillName - 类似 "superpowers:brainstorming" 或 "my-skill" 的名称
 * @param {string} superpowersDir - 超能力技能目录的路径
 * @param {string} personalDir - 个人技能目录的路径
 * @returns {{skillFile: string, sourceType: string, skillPath: string} | null}
 */
function resolveSkillPath(skillName, superpowersDir, personalDir) {
    // 如果存在 superpowers: 前缀则去除
    const forceSuperpowers = skillName.startsWith('superpowers:');
    const actualSkillName = forceSuperpowers ? skillName.replace(/^superpowers:/, '') : skillName;

    // 首先尝试个人技能（除非明确指定 superpowers:）
    if (!forceSuperpowers && personalDir) {
        const personalPath = path.join(personalDir, actualSkillName);
        const personalSkillFile = path.join(personalPath, 'SKILL.md');
        if (fs.existsSync(personalSkillFile)) {
            return {
                skillFile: personalSkillFile,
                sourceType: 'personal',
                skillPath: actualSkillName
            };
        }
    }

    // 尝试超能力技能
    if (superpowersDir) {
        const superpowersPath = path.join(superpowersDir, actualSkillName);
        const superpowersSkillFile = path.join(superpowersPath, 'SKILL.md');
        if (fs.existsSync(superpowersSkillFile)) {
            return {
                skillFile: superpowersSkillFile,
                sourceType: 'superpowers',
                skillPath: actualSkillName
            };
        }
    }

    return null;
}
```

**步骤 2：更新 module.exports**

```javascript
module.exports = {
    extractFrontmatter,
    findSkillsInDir,
    resolveSkillPath
};
```

**步骤 3：验证语法**

运行：`node -c lib/skills-core.js`
预期：无输出

**步骤 4：提交**

```bash
git add lib/skills-core.js
git commit -m "feat: 添加支持覆盖的技能路径解析"
```

---

### 任务 4：提取更新检查逻辑

**文件：**
- 修改：`lib/skills-core.js`
- 参考：`.codex/superpowers-codex` (第 16-38 行)

**步骤 1：添加 checkForUpdates 函数**

在 require 之后顶部添加：

```javascript
const { execSync } = require('child_process');
```

在 `module.exports` 之前添加：

```javascript
/**
 * 检查 git 仓库是否有可用更新。
 *
 * @param {string} repoDir - git 仓库的路径
 * @returns {boolean} - 如果有更新则为 true
 */
function checkForUpdates(repoDir) {
    try {
        // 快速检查，设置 3 秒超时，避免网络不通时延迟
        const output = execSync('git fetch origin && git status --porcelain=v1 --branch', {
            cwd: repoDir,
            timeout: 3000,
            encoding: 'utf8',
            stdio: 'pipe'
        });

        // 解析 git status 输出以查看是否落后
        const statusLines = output.split('\n');
        for (const line of statusLines) {
            if (line.startsWith('## ') && line.includes('[behind ')) {
                return true; // 我们落后于远程
            }
        }
        return false; // 已是最新
    } catch (error) {
        // 网络不通、git 错误、超时等 - 不阻塞引导
        return false;
    }
}
```

**步骤 2：更新 module.exports**

```javascript
module.exports = {
    extractFrontmatter,
    findSkillsInDir,
    resolveSkillPath,
    checkForUpdates
};
```

**步骤 3：验证语法**

运行：`node -c lib/skills-core.js`
预期：无输出

**步骤 4：提交**

```bash
git add lib/skills-core.js
git commit -m "feat: 向核心模块添加 git 更新检查"
```

---

## 第二阶段：重构 Codex 以使用共享核心

### 任务 5：更新 Codex 以导入共享核心

**文件：**
- 修改：`.codex/superpowers-codex` (在顶部添加导入)

**步骤 1：添加导入语句**

在文件顶部现有的 require 之后（大约第 6 行），添加：

```javascript
const skillsCore = require('../lib/skills-core');
```

**步骤 2：验证语法**

运行：`node -c .codex/superpowers-codex`
预期：无输出

**步骤 3：提交**

```bash
git add .codex/superpowers-codex
git commit -m "refactor: 在 codex 中导入共享技能核心"
```

---

### 任务 6：将 extractFrontmatter 替换为核心版本

**文件：**
- 修改：`.codex/superpowers-codex` (第 40-74 行)

**步骤 1：删除本地的 extractFrontmatter 函数**

删除第 40-74 行（整个 extractFrontmatter 函数定义）。

**步骤 2：更新所有 extractFrontmatter 调用**

查找并将所有 `extractFrontmatter(` 调用替换为 `skillsCore.extractFrontmatter(`

受影响的行大约为：90, 310

**步骤 3：验证脚本仍然有效**

运行：`.codex/superpowers-codex find-skills | head -20`
预期：显示技能列表

**步骤 4：提交**

```bash
git add .codex/superpowers-codex
git commit -m "refactor: 在 codex 中使用共享的 extractFrontmatter"
```

---

### 任务 7：将 findSkillsInDir 替换为核心版本

**文件：**
- 修改：`.codex/superpowers-codex` (大约第 97-136 行)

**步骤 1：删除本地的 findSkillsInDir 函数**

删除整个 `findSkillsInDir` 函数定义（大约第 97-136 行）。

**步骤 2：更新所有 findSkillsInDir 调用**

将 `findSkillsInDir(` 调用替换为 `skillsCore.findSkillsInDir(`

**步骤 3：验证脚本仍然有效**

运行：`.codex/superpowers-codex find-skills | head -20`
预期：显示技能列表

**步骤 4：提交**

```bash
git add .codex/superpowers-codex
git commit -m "refactor: 在 codex 中使用共享的 findSkillsInDir"
```

---

### 任务 8：将 checkForUpdates 替换为核心版本

**文件：**
- 修改：`.codex/superpowers-codex` (大约第 16-38 行)

**步骤 1：删除本地的 checkForUpdates 函数**

删除整个 `checkForUpdates` 函数定义。

**步骤 2：更新所有 checkForUpdates 调用**

将 `checkForUpdates(` 调用替换为 `skillsCore.checkForUpdates(`

**步骤 3：验证脚本仍然有效**

运行：`.codex/superpowers-codex bootstrap | head -50`
预期：显示引导内容

**步骤 4：提交**

```bash
git add .codex/superpowers-codex
git commit -m "refactor: 在 codex 中使用共享的 checkForUpdates"
```

---

## 第三阶段：构建 OpenCode 插件

### 任务 9：创建 OpenCode 插件目录结构

**文件：**
- 创建：`.opencode/plugin/superpowers.js`

**步骤 1：创建目录**

运行：`mkdir -p .opencode/plugin`

**步骤 2：创建基本插件文件**

```javascript
#!/usr/bin/env node

/**
 * OpenCode.ai 的超能力插件
 *
 * 提供用于加载和发现技能的自定义工具，
 * 并在会话开始时自动引导。
 */

const skillsCore = require('../../lib/skills-core');
const path = require('path');
const fs = require('fs');
const os = require('os');

const homeDir = os.homedir();
const superpowersSkillsDir = path.join(homeDir, '.config/opencode/superpowers/skills');
const personalSkillsDir = path.join(homeDir, '.config/opencode/skills');

/**
 * OpenCode 插件入口点
 */
export const SuperpowersPlugin = async ({ project, client, $, directory, worktree }) => {
  return {
    // 自定义工具和钩子将放在这里
  };
};
```

**步骤 3：验证文件已创建**

运行：`ls -l .opencode/plugin/superpowers.js`
预期：文件存在

**步骤 4：提交**

```bash
git add .opencode/plugin/superpowers.js
git commit -m "feat: 创建 opencode 插件脚手架"
```

---

### 任务 10：实现 use_skill 工具

**文件：**
- 修改：`.opencode/plugin/superpowers.js`

**步骤 1：添加 use_skill 工具实现**

将插件返回语句替换为：

```javascript
export const SuperpowersPlugin = async ({ project, client, $, directory, worktree }) => {
  // 导入 zod 用于模式验证
  const { z } = await import('zod');

  return {
    tools: [
      {
        name: 'use_skill',
        description: '加载并读取特定技能以指导你的工作。技能包含经过验证的工作流程、强制流程和专家技术。',
        schema: z.object({
          skill_name: z.string().describe('要加载的技能名称（例如 "superpowers:brainstorming" 或 "my-custom-skill"）')
        }),
        execute: async ({ skill_name }) => {
          // 解析技能路径（处理覆盖：个人 > 超能力）
          const resolved = skillsCore.resolveSkillPath(
            skill_name,
            superpowersSkillsDir,
            personalSkillsDir
          );

          if (!resolved) {
            return `错误：未找到技能 "${skill_name}"。\n\n运行 find_skills 查看可用技能。`;
          }

          // 读取技能内容
          const fullContent = fs.readFileSync(resolved.skillFile, 'utf8');
          const { name, description } = skillsCore.extractFrontmatter(resolved.skillFile);

          // 提取 frontmatter 之后的内容
          const lines = fullContent.split('\n');
          let inFrontmatter = false;
          let frontmatterEnded = false;
          const contentLines = [];

          for (const line of lines) {
            if (line.trim() === '---') {
              if (inFrontmatter) {
                frontmatterEnded = true;
                continue;
              }
              inFrontmatter = true;
              continue;
            }

            if (frontmatterEnded || !inFrontmatter) {
              contentLines.push(line);
            }
          }

          const content = contentLines.join('\n').trim();
          const skillDirectory = path.dirname(resolved.skillFile);

          // 格式化输出，类似于 Claude Code 的 Skill 工具
          return `# ${name || skill_name}
# ${description || ''}
# 支持的工具和文档位于 ${skillDirectory}
# ============================================

${content}`;
        }
      }
    ]
  };
};
```

**步骤 2：验证语法**

运行：`node -c .opencode/plugin/superpowers.js`
预期：无输出

**步骤 3：提交**

```bash
git add .opencode/plugin/superpowers.js
git commit -m "feat: 为 opencode 实现 use_skill 工具"
```

---

### 任务 11：实现 find_skills 工具

**文件：**
- 修改：`.opencode/plugin/superpowers.js`

**步骤 1：向工具数组添加 find_skills 工具**

在 use_skill 工具定义之后，工具数组关闭之前添加：

```javascript
      {
        name: 'find_skills',
        description: '列出超能力和个人技能库中的所有可用技能。',
        schema: z.object({}),
        execute: async () => {
          // 在两个目录中查找技能
          const superpowersSkills = skillsCore.findSkillsInDir(
            superpowersSkillsDir,
            'superpowers',
            3
          );
          const personalSkills = skillsCore.findSkillsInDir(
            personalSkillsDir,
            'personal',
            3
          );

          // 合并并格式化技能列表
          const allSkills = [...personalSkills, ...superpowersSkills];

          if (allSkills.length === 0) {
            return '未找到技能。请安装超能力技能到 ~/.config/opencode/superpowers/skills/';
          }

          let output = '可用技能：\n\n';

          for (const skill of allSkills) {
            const namespace = skill.sourceType === 'personal' ? '' : 'superpowers:';
            const skillName = skill.name || path.basename(skill.path);

            output += `${namespace}${skillName}\n`;
            if (skill.description) {
              output += `  ${skill.description}\n`;
            }
            output += `  目录：${skill.path}\n\n`;
          }

          return output;
        }
      }
```

**步骤 2：验证语法**

运行：`node -c .opencode/plugin/superpowers.js`
预期：无输出

**步骤 3：提交**

```bash
git add .opencode/plugin/superpowers.js
git commit -m "feat: 为 opencode 实现 find_skills 工具"
```

---

### 任务 12：实现会话启动钩子

**文件：**
- 修改：`.opencode/plugin/superpowers.js`

**步骤 1：添加 session.started 钩子**

在工具数组之后添加：

```javascript
    'session.started': async () => {
      // 读取 using-superpowers 技能内容
      const usingSuperpowersPath = skillsCore.resolveSkillPath(
        'using-superpowers',
        superpowersSkillsDir,
        personalSkillsDir
      );

      let usingSuperpowersContent = '';
      if (usingSuperpowersPath) {
        const fullContent = fs.readFileSync(usingSuperpowersPath.skillFile, 'utf8');
        // 去除 frontmatter
        const lines = fullContent.split('\n');
        let inFrontmatter = false;
        let frontmatterEnded = false;
        const contentLines = [];

        for (const line of lines) {
          if (line.trim() === '---') {
            if (inFrontmatter) {
              frontmatterEnded = true;
              continue;
            }
            inFrontmatter = true;
            continue;
          }

          if (frontmatterEnded || !inFrontmatter) {
            contentLines.push(line);
          }
        }

        usingSuperpowersContent = contentLines.join('\n').trim();
      }

      // 工具映射说明
      const toolMapping = `
**OpenCode 的工具映射：**
当技能引用你没有的工具时，请替换为 OpenCode 的等效工具：
- \`TodoWrite\` → \`update_plan\`（你的计划/任务跟踪工具）
- 带有子代理的 \`Task\` 工具 → 使用 OpenCode 的子代理系统（@mention 语法或自动分派）
- \`Skill\` 工具 → \`use_skill\` 自定义工具（已可用）
- \`Read\`, \`Write\`, \`Edit\`, \`Bash\` → 使用你的原生工具

**技能目录包含支持文件：**
- 你可以使用 bash 工具运行的脚本
- 你可以阅读的额外文档
- 特定于该技能的实用程序和助手

**技能命名：**
- 超能力技能：\`superpowers:skill-name\`（来自 ~/.config/opencode/superpowers/skills/）
- 个人技能：\`skill-name\`（来自 ~/.config/opencode/skills/）
- 当名称匹配时，个人技能会覆盖超能力技能
`;

      // 检查更新（非阻塞）
      const hasUpdates = skillsCore.checkForUpdates(
        path.join(homeDir, '.config/opencode/superpowers')
      );

      const updateNotice = hasUpdates ?
        '\n\n⚠️ **有可用更新！** 运行 `cd ~/.config/opencode/superpowers && git pull` 以更新超能力。' :
        '';

      // 返回要注入会话的上下文
      return {
        context: `<EXTREMELY_IMPORTANT>
你拥有超能力。

**以下是你的 'superpowers:using-superpowers' 技能的全部内容 - 这是你使用技能的介绍。对于所有其他技能，请使用 'use_skill' 工具：**

${usingSuperpowersContent}

${toolMapping}${updateNotice}
</EXTREMELY_IMPORTANT>`
      };
    }
```

**步骤 2：验证语法**

运行：`node -c .opencode/plugin/superpowers.js`
预期：无输出

**步骤 3：提交**

```bash
git add .opencode/plugin/superpowers.js
git commit -m "feat: 为 opencode 实现 session.started 钩子"
```

---

## 第四阶段：文档

### 任务 13：创建 OpenCode 安装指南

**文件：**
- 创建：`.opencode/INSTALL.md`

**步骤 1：创建安装指南**

```markdown
# 为 OpenCode 安装超能力

## 先决条件

- 已安装 [OpenCode.ai](https://opencode.ai)
- 已安装 Node.js
- 已安装 Git

## 安装步骤

### 1. 安装超能力技能

```bash
# 将超能力技能克隆到 OpenCode 配置目录
mkdir -p ~/.config/opencode/superpowers
git clone https://github.com/obra/superpowers.git ~/.config/opencode/superpowers
```

### 2. 安装插件

插件包含在你刚刚克隆的超能力仓库中。

OpenCode 将自动从以下位置发现它：
- `~/.config/opencode/superpowers/.opencode/plugin/superpowers.js`

或者你可以将其链接到项目本地的插件目录：

```bash
# 在你的 OpenCode 项目中
mkdir -p .opencode/plugin
ln -s ~/.config/opencode/superpowers/.opencode/plugin/superpowers.js .opencode/plugin/superpowers.js
```

### 3. 重启 OpenCode

重启 OpenCode 以加载插件。在下一个会话中，你应该会看到：

```
你拥有超能力。
```

## 使用方法

### 查找技能

使用 `find_skills` 工具列出所有可用技能：

```
使用 find_skills 工具
```

### 加载技能

使用 `use_skill` 工具加载特定技能：

```
使用 use_skill 工具，skill_name: "superpowers:brainstorming"
```

### 个人技能

在 `~/.config/opencode/skills/` 中创建你自己的技能：

```bash
mkdir -p ~/.config/opencode/skills/my-skill
```

创建 `~/.config/opencode/skills/my-skill/SKILL.md`：

```markdown
---
name: my-skill
description: Use when [condition] - [what it does]
---

# 我的技能

[你的技能内容在这里]
```

个人技能会覆盖同名的超能力技能。

## 更新

```bash
cd ~/.config/opencode/superpowers
git pull
```

## 故障排除

### 插件未加载

1. 检查插件文件是否存在：`ls ~/.config/opencode/superpowers/.opencode/plugin/superpowers.js`
2. 检查 OpenCode 日志中的错误
3. 验证 Node.js 是否已安装：`node --version`

### 未找到技能

1. 验证技能目录是否存在：`ls ~/.config/opencode/superpowers/skills`
2. 使用 `find_skills` 工具查看发现了什么
3. 检查文件结构：每个技能应有一个 `SKILL.md` 文件

### 工具映射问题

当技能引用你没有的 Claude Code 工具时：
- `TodoWrite` → 使用 `update_plan`
- 带有子代理的 `Task` → 使用 `@mention` 语法调用 OpenCode 子代理
- `Skill` → 使用 `use_skill` 工具
- 文件操作 → 使用你的原生工具

## 获取帮助

- 报告问题：https://github.com/obra/superpowers/issues
- 文档：https://github.com/obra/superpowers
```

**步骤 2：验证文件已创建**

运行：`ls -l .opencode/INSTALL.md`
预期：文件存在

**步骤 3：提交**

```bash
git add .opencode/INSTALL.md
git commit -m "docs: 添加 opencode 安装指南"
```

---

### 任务 14：更新主 README

**文件：**
- 修改：`README.md`

**步骤 1：添加 OpenCode 部分**

找到关于支持平台的部分（在文件中搜索 "Codex"），并在其后添加：

```markdown
### OpenCode

超能力通过原生 JavaScript 插件与 [OpenCode.ai](https://opencode.ai) 协同工作。

**安装：** 参见 [.opencode/INSTALL.md](.opencode/INSTALL.md)

**功能：**
- 自定义工具：`use_skill` 和 `find_skills`
- 自动会话引导
- 支持覆盖的个人技能
- 支持文件和脚本访问
```

**步骤 2：验证格式**

运行：`grep -A 10 "### OpenCode" README.md`
预期：显示你添加的部分

**步骤 3：提交**

```bash
git add README.md
git commit -m "docs: 在 readme 中添加 opencode 支持"
```

---

### 任务 15：更新发布说明

**文件：**
- 修改：`RELEASE-NOTES.md`

**步骤 1：为 OpenCode 支持添加条目**

在文件顶部（标题之后），添加：

```markdown
## [未发布]

### 新增

- **OpenCode 支持**：OpenCode.ai 的原生 JavaScript 插件
  - 自定义工具：`use_skill` 和 `find_skills`
  - 带有工具映射说明的自动会话引导
  - 用于代码重用的共享核心模块（`lib/skills-core.js`）
  - 安装指南位于 `.opencode/INSTALL.md`

### 变更

- **重构 Codex 实现**：现在使用共享的 `lib/skills-core.js` 模块
  - 消除了 Codex 和 OpenCode 之间的代码重复
  - 技能发现和解析的单一事实来源

---

```

**步骤 2：验证格式**

运行：`head -30 RELEASE-NOTES.md`
预期：显示你的新部分

**步骤 3：提交**

```bash
git add RELEASE-NOTES.md
git commit -m "docs: 在发布说明中添加 opencode 支持"
```

---

## 第五阶段：最终验证

### 任务 16：测试 Codex 是否仍然有效

**文件：**
- 测试：`.codex/superpowers-codex`

**步骤 1：测试 find-skills 命令**

运行：`.codex/superpowers-codex find-skills | head -20`
预期：显示带有名称和描述的技能列表

**步骤 2：测试 use-skill 命令**

运行：`.codex/superpowers-codex use-skill superpowers:brainstorming | head -20`
预期：显示头脑风暴技能内容

**步骤 3：测试 bootstrap 命令**

运行：`.codex/superpowers-codex bootstrap | head -30`
预期：显示带有说明的引导内容

**步骤 4：如果所有测试通过，记录成功**

无需提交 - 这只是验证。

---

### 任务 17：验证文件结构

**文件：**
- 检查：所有新文件是否存在

**步骤 1：验证所有文件已创建**

运行：
```bash
ls -l lib/skills-core.js
ls -l .opencode/plugin/superpowers.js
ls -l .opencode/INSTALL.md
```

预期：所有文件都存在

**步骤 2：验证目录结构**

运行：`tree -L 2 .opencode/`（如果 tree 不可用，则使用 `find .opencode -type f`）
预期：
```
.opencode/
├── INSTALL.md
└── plugin/
    └── superpowers.js
```

**步骤 3：如果结构正确，继续**

无需提交 - 这只是验证。

---

### 任务 18：最终提交和总结

**文件：**
- 检查：`git status`

**步骤 1：检查 git 状态**

运行：`git status`
预期：工作树干净，所有更改已提交

**步骤 2：查看提交日志**

运行：`git log --oneline -20`
预期：显示此实现的所有提交

**步骤 3：创建总结文档**

创建完成总结，显示：
- 总共进行的提交数
- 创建的文件：`lib/skills-core.js`, `.opencode/plugin/superpowers.js`, `.opencode/INSTALL.md`
- 修改的文件：`.codex/superpowers-codex`, `README.md`, `RELEASE-NOTES.md`
- 执行的测试：Codex 命令已验证
- 准备就绪：使用实际的 OpenCode 安装进行测试

**步骤 4：报告完成**

向用户呈现总结并提供以下选项：
1. 推送到远程
2. 创建拉取请求
3. 使用真实的 OpenCode 安装进行测试（需要安装 OpenCode）

---

## 测试指南（手动 - 需要 OpenCode）

这些步骤需要安装 OpenCode，不属于自动化实现的一部分：

1. **安装技能**：按照 `.opencode/INSTALL.md` 操作
2. **启动 OpenCode 会话**：验证引导是否出现
3. **测试 find_skills**：应列出所有可用技能
4. **测试 use_skill**：加载一个技能并验证内容是否出现
5. **测试支持文件**：验证技能目录路径是否可访问
6. **测试个人技能**：创建一个个人技能并验证它是否覆盖核心技能
7. **测试工具映射**：验证 TodoWrite → update_plan 映射是否有效

## 成功标准

- [ ] `lib/skills-core.js` 已创建，包含所有核心函数
- [ ] `.codex/superpowers-codex` 已重构为使用共享核心
- [ ] Codex 命令仍然有效（find-skills, use-skill, bootstrap）
- [ ] `.opencode/plugin/superpowers.js` 已创建，包含工具和钩子
- [ ] 安装指南已创建
- [ ] README 和 RELEASE-NOTES 已更新
- [ ] 所有更改已提交
- [ ] 工作树干净
