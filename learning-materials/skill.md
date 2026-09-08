# Skill（技能）学习资料

## 一、概念个人解释

我的理解：**Skill 就是把"怎么做一件事"打包成一个文件夹，让 Agent 需要的时候自己翻出来照着做。**

平时我们想让 AI 稳定地干一件事（比如每次都按固定格式写学习笔记），就得每次都把那一大段要求重新粘贴一遍。
Skill 相当于把这段要求存成一个文件放在项目里，AI 自己判断"这个任务好像该用那个 Skill"，然后自己去读。

形式上特别朴素：

```
skill-name/
└── SKILL.md     # 就这么一个文件就够了
```

SKILL.md 分两部分：

- **开头 YAML 元数据**：只有 `name` 和 `description`，告诉 AI "我是谁、什么时候该用我"。
- **下面正文**：具体步骤、输出格式、检查要求。

关键在于 `description`——AI 全靠这一句话决定要不要加载这个 Skill。写得含糊，它就永远不会被触发。

一句话版本：**Skill 是给 Agent 写的岗位操作手册，不是给它的一次性命令。**

## 二、核心机制与组成

**1. 渐进式披露（Progressive Disclosure）——最核心的设计**

AI 不会一上来把全部 Skill 都读进上下文，分三层加载：

- **第一层（启动时）**：只读所有 Skill 的 name + description，每个大概几十个 token。装 50 个 Skill 也占不了多少地方。
- **第二层（判断相关后）**：AI 觉得这条任务用得上，才把那个 SKILL.md 的正文完整读进来。
- **第三层（执行中需要时）**：SKILL.md 里提到"详见 reference.md"或者"运行 scripts/xx.py"时，才去读那个文件 / 跑那个脚本。

好处是：**一个 Skill 能带的东西几乎没有上限**，因为不用的部分永远不占上下文。这也是为什么 Skill 比"把一堆提示词塞进系统提示"高级。

**2. 一个 Skill 目录里能放什么**

```
my-skill/
├── SKILL.md          # 主入口，AI 自动读
├── reference.md      # 详细参考资料，按需读
├── scripts/          # 可执行脚本，AI 直接跑（脚本内容不进上下文）
└── assets/           # 模板、示例文件
```

注意脚本那条：让 AI 跑 `sort.py` 排序，比让它一个一个 token 生成排序结果又快又准，而且脚本本身不占上下文。这是 Skill 很实用的一点。

**3. 两种存放位置**

- **项目级**：放在仓库里的 `.workbuddy/skills/<skill名>/`（本项目用的就是这种）。跟着 git 走，克隆仓库的人自动获得同一批 Skill。
- **个人级**：放在用户目录 `~/.workbuddy/skills/` 下，所有项目通用。

本次作业要求的是项目级。

**4. 触发方式**

Skill 是**模型自己判断**要不要用的，不是用户手动点命令。所以 description 里要把"什么时候用"写清楚，甚至写几个触发关键词。

## 三、实际应用场景

**场景：本次作业里的概念学习 Skill。**

我建了一个 `concept-learn-skill`，放在仓库的 `.workbuddy/skills/concept-learn-skill/SKILL.md`。

它的 description 写的是：接收任意 AI 概念名词，产出结构化学习笔记。
正文里规定了必须输出的五个板块（个人解释 / 核心机制 / 应用场景 / 边界辨析 / 参考来源），还要求参考链接必须真实可访问。

实际效果：

- 输入 `Agent` → 输出一份 agent.md；
- 输入 `大模型的上下文` → 输出一份 llm-context.md；
- 输入 `Skill` → 输出一份 skill.md。

三次调用，格式完全一致，我不用再重复解释"要分五个板块、链接不能瞎编"。
而且它没写死只能处理这三个词——换成 `RAG`、`微调`、`多模态`，同样能出一份。这就是 Skill 相对"一次性提示词"的价值：**流程固化下来，可复用、可版本管理、可分享给同学。**

## 四、易混淆问题 & 使用边界

**1. Skill ≠ 提示词**

提示词是一次性的、口头的。Skill 是存在文件系统里的、有结构的、能带脚本和参考资料的。
一句话提示词偶尔用用没问题，要反复用、要团队共用，就该做成 Skill。

**2. Skill ≠ MCP / 工具**

Skill 是"教它怎么做"（知识、流程）；MCP 和工具是"给它能力"（真正去连数据库、调接口）。
Skill 里可以规定"第 3 步调用 xx 工具"，但它本身不提供工具。

**3. Skill ≠ 记忆 / 上下文**

Skill 文件躺在磁盘上，平时不占上下文；被触发才读进来。它不会随着对话自动"记住"你的偏好。
需要长期记住的东西，得写进记忆文件或项目说明里。

**4. Skill ≠ 子 Agent**

Skill 是给当前 Agent 加载的一段操作指引，不会新开一个独立的执行体。

**什么时候不适合做成 Skill：**

- 只用一次的需求，直接说就行，没必要建文件。
- 流程根本不固定的探索性任务，写死了反而限制它。
- 需要严格确定性执行的（比如金融计算），应该写成脚本让 AI 跑，而不是写一段自然语言让它照着算。

**写 Skill 最常见的坑：**

- description 写得太虚（"帮助处理文档"），AI 判断不出来什么时候该用 → 永远不触发。
- SKILL.md 写成几千行一个大文件 → 一触发就把上下文吃掉一半。应该拆到 reference 文件里。
- 文件名写成小写的 `skill.md` 或者带 `.txt` 后缀 → 软件识别不到（Windows 默认隐藏后缀，最容易中招）。

## 五、参考资料来源

1. Anthropic 官方工程博客《Equipping agents for the real world with Agent Skills》（Skill 的设计原理、渐进式披露）：
   https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills
2. Anthropic 官方课程《Workflows vs agents》（理解 Skill 在 Agent 体系里的位置）：
   https://academy.claude.com/courses/building-with-the-claude-api/workflows-vs-agents
3. Model Context Protocol 官方站（与 Skill 配合的工具协议）：
   https://modelcontextprotocol.io/
4. WorkBuddy（腾讯）官方文档 - 产品简介（本仓库项目级 Skill `.workbuddy/skills/` 所依托的产品）：
   https://www.workbuddy.cn/docs/workbuddy/Overview
