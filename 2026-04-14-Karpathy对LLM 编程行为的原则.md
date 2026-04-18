# 一个 CLAUDE.md 文件竟然拿下 1.7 万 star，它到底写了什么
 
原文:https://mp.weixin.qq.com/s/nwpiH-a5kE5ZGFtKlXguzw

github: https://github.com/forrestchang/andrej-karpathy-skills?tab=readme-ov-file

为什么是 Karpathy？

Karpathy 的特殊之处在于，他不是一个旁观者。他是 OpenAI 的联合创始人之一，是 GPT 时代最重要的推动者之一，后来又在特斯拉主导了 FSD 的 AI 架构。换句话说，他是亲手把这些模型做出来的人。当他说 LLM 在写代码时有系统性缺陷，这个判断的分量完全不同于一般用户的抱怨。

他在 X 上的原话是：

"The models make wrong assumptions on your behalf and just run along with them without checking. They don't manage their confusion, don't seek clarifications, don't surface inconsistencies, don't present tradeoffs, don't push back when they should."

"They really like to overcomplicate code and APIs, bloat abstractions, don't clean up dead code... implement a bloated construction over 1000 lines when 100 would do."

"They still sometimes change/remove comments and code they don't sufficiently understand as side effects, even if orthogonal to the task."

用过 AI 编程工具的人，大概都在这三段话里找到过自己的经历。

一个文件，四条规则

Forrest Chang 做的事情很克制。他没有开发插件，没有搞界面，没有写 SDK。就是把 Karpathy 的观察翻译成了四条可以直接塞进 CLAUDE.md 的行为准则，然后开源出来。

来看实际内容：

## 1. 先想后写（Think Before Coding）

**不要假设，不要藏着疑惑，把取舍摆出来。**

动手之前：
- 明确说出你的假设，如果不确定，就问。
- 如果存在多种理解，把它们列出来，不要自己悄悄选一个。
- 如果有更简单的方案，说出来，推一下。
- 如果有什么搞不清楚，停下来，说清楚哪里不明白，再问。

## 2. 能简则简（Simplicity First）

**用最少的代码解决问题，不做多余的事。**

- 不加用户没要求的功能。
- 不为一次性逻辑搭抽象层。
- 不预加"灵活性"或"可配置性"。
- 不为不可能发生的场景写错误处理。
- 如果写了 200 行但 50 行够用，重写。

自检标准：一个高级工程师看了会觉得过度设计吗？如果会，简化。

## 3. 精准修改（Surgical Changes）

**只动你该动的地方，只清理你自己制造的烂摊子。**

修改已有代码时：
- 不要"顺手优化"旁边的代码、注释或格式。
- 不要重构没有问题的逻辑。
- 保持原有风格，即使你觉得可以写得更好。
- 发现不相关的死代码，提一句，不要擅自删。

当你的改动制造了孤儿代码时：
- 清掉你的改动造成的多余 import、变量、函数。
- 原来就有的死代码，不要动，除非被要求。

检验标准：每一处改动都能直接追溯到用户的请求。

## 4. 目标驱动执行（Goal-Driven Execution）

**定义成功标准，循环直到验证通过。**

把指令转化成可验证的目标：
- "加一个校验" → "先写覆盖非法输入的测试，再让测试通过"
- "修这个 bug" → "先写能复现 bug 的测试，再修"
- "重构 X" → "确保重构前后测试都通过"

多步骤任务，列出简要计划：
1. [步骤] → 验证：[检查项]
2. [步骤] → 验证：[检查项]
3. [步骤] → 验证：[检查项]
每条规则都附了自检标准，不是空话，可以直接用于判断。

第四条规则背后的核心洞见

第四条单独拿出来说值得多花一点时间。Karpathy 在观察里写道：

"LLMs are exceptionally good at looping until they meet specific goals... Don't tell it what to do, give it success criteria and watch it go."

这句话点出了一个很多人用错了的地方。通常我们给 AI 的指令是"做什么"，而 Karpathy 建议换成"什么叫做好了"。两者之间的差距在复杂任务里非常明显：前者需要你持续介入判断是否做到位，后者让模型自己循环直到标准满足。

多步骤任务的格式建议是：

1. [步骤] → 验证：[检查项]
2. [步骤] → 验证：[检查项]
3. [步骤] → 验证：[检查项]
每一步都有明确的完成标准，模型可以自己核对，而不是做完了等你来判断。

怎么判断它在起效

项目 README 里有一段很实用的验收标准，说明这四条规则在起作用时你会看到什么：

• diff 里的改动更干净，只有被要求改的地方出现了变化
• 第一次生成的代码更简洁，不用再来回要求它简化
• 澄清性问题出现在动手之前，而不是做错之后
• PR 更整洁，没有"顺带优化"的冗余提交
这四个信号可以当作自己日常使用时的观察点，感受一下加了规则前后的差异。

如何使用

方式一：单项目，直接把 CLAUDE.md 下载到项目根目录：

curl -o CLAUDE.md https://raw.githubusercontent.com/forrestchang/andrej-karpathy-skills/main/CLAUDE.md
如果项目里已经有 CLAUDE.md 了，追加进去：

echo "" >> CLAUDE.md
curl https://raw.githubusercontent.com/forrestchang/andrej-karpathy-skills/main/CLAUDE.md >> CLAUDE.md


方式二：通过 Claude Code 插件全局安装，所有项目都生效：

/plugin marketplace add forrestchang/andrej-karpathy-skills
/plugin install andrej-karpathy-skills@karpathy-skills
作者建议把这个文件当作基础层，再在下面叠加自己项目的特定规则，比如"所有 API 必须有测试"之类的约定。

这个文件为什么有人用

把这四条规则真正对照一下使用体验，会发现它们针对的问题确实普遍存在，而且在长对话、复杂任务里尤其明显。

Claude Code 在处理简单的一次性任务时通常表现不错，但当项目变大、上下文变长，模型的"好心"开始变成负担，它会在你不知情的地方悄悄做一些"顺手"的修改，会把一个直接的需求包裹进一层不必要的抽象，会在自己迷糊的时候依然往前冲而不是停下来问你。

这些问题不是 bug，是模型在训练上的系统性倾向。单靠口头提醒效果有限，因为每次对话开始时模型都是从头来的，上次你说"别多改"这次不一定还记得。CLAUDE.md 的逻辑是把约束写死在项目层面，每次启动都读进去，不用重复叮嘱。

这也是 1.7 万 star 背后的真正原因，不是技术门槛有多高，而是它准确击中了一个每天都在发生的真实痛点，同时给出了足够简单的解法。

值得思考的一件事

这个项目走红的时机很有意思。最近一两年，AI 编程工具的数量在爆增，框架、插件、Agent 平台一个接一个。但这个排行榜前列的项目，是一个文本文件。

不是在说复杂工具没价值，而是它揭示了一个此前被低估的角度：模型的行为是可以被项目级别的指令系统性地塑造的，这件事本身的价值，可能不比工具链更低。

 