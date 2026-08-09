# 我为什么从 Superpowers 转向一个更克制的 Grilling

我最早系统使用 Agent Skills，是从 [Superpowers](https://github.com/obra/superpowers) 开始的。

它给 coding agent 配了一套完整的软件开发方法：先澄清需求，再写设计和实现计划，之后用 worktree、TDD、子 Agent 和代码审查完成开发。对刚开始使用 coding agent 的人，或者希望团队统一工作方式的人，这套流程很有价值。许多容易被模型跳过的工程动作，都被它写成了明确的门禁。

但模型能力越来越强以后，我在自己的工作流里遇到了另一个问题：流程开始显得太重。

Superpowers 的 [`using-superpowers`](https://github.com/obra/superpowers/blob/main/skills/using-superpowers/SKILL.md) 要求，只要有 1% 的可能适用，就必须先加载相关 skill；[`brainstorming`](https://github.com/obra/superpowers/blob/main/skills/brainstorming/SKILL.md) 则覆盖几乎所有创造性工作，还会继续进入设计文档和实现计划。这样的强约束有它的理由，但在我已经能判断任务边界、模型也能处理大量常规工程选择以后，过宽的触发范围会增加上下文和交互成本。

我并不是认为 Superpowers 不好。恰恰相反，它很完整。只是我不再需要每次都启动一整套方法论。我更想要一个小工具：只在我明确需要时介入，把真正没想清楚的决策问透，然后停下来。

于是我转向了 Matt Pocock 的 [`grilling`](https://github.com/mattpocock/skills/blob/main/skills/productivity/grilling/SKILL.md)。

## 一次只问一个问题，也会成为问题

早期的 `grilling` 很简单：沿着决策树逐个提问，每个问题给出推荐答案；能从代码库找到的事实，由 Agent 自己调查；其余问题一次只问一个。

这个设计比完整的 brainstorming 流程轻得多，我很喜欢。但使用一段时间以后，我发现“一次一个”也有明显成本。有些设计会产生十几个甚至几十个问题。问题本身可能都有价值，可每问一个就等待一次，整场讨论会被拉得很长，模型也要在很多轮对话中反复携带上下文。上游甚至出现过一位用户被问了 [200 个问题](https://github.com/mattpocock/skills/issues/44) 的记录。

我不想简单改成“一次把所有问题都扔出来”。那样虽然轮次少了，但后面的问题可能依赖前面尚未给出的答案。真正需要控制的不是问题总数，而是哪些问题可以安全地出现在同一轮。

2026 年 7 月 2 日，我在独立仓库发布了自己的第一个 [`grilling` 版本](https://github.com/wufei-png/grilling/commit/0479a014e496d172b7fa186ec2c618674fb6829d)。它仍然默认一次问一个问题，但允许用户给出“每轮最多几个问题”。这个数字只是上限，只有关系紧密、位于同一决策分支的问题才可以合并。

7 月 10 日，我又补上了一条后来一直保留的规则：如果约束条件已经让某个选项明显更好，Agent 应该直接选择，不要为了互动而制造问题；只有存在真实取舍的决策，才值得交给用户。

这两处改动解决的是同一件事：减少没有决策价值的来回对话。

## 后来，Matt 也开始批量提问

7 月 16 日，Matt 提交了实验性的 [`batch-grill-me`](https://github.com/mattpocock/skills/commit/c70cb091933617c61acf9bd6c3b01c1140329cf1)。它把待讨论内容建模成 decision tree，并把前置决策已经确定的问题称为 frontier。每一轮询问整个 frontier，得到答案后重新计算下一轮。

随后，这套逻辑被合并进 `grilling`。上游的 [v1.2 PR](https://github.com/mattpocock/skills/pull/593) 用一个很直观的例子解释收益：原本需要 13 轮的 13 个问题，大约可以压缩到 3 轮。需要调查的事实还可以交给后台子 Agent，不必阻塞同一轮里与它无关的问题。

从公开提交时间看，我的版本比 `batch-grill-me` 早了约 13 天 18 小时。看到这个时间线时，我确实很开心。它至少说明，我在真实使用中感受到的摩擦并不是偶然的。

不过这不是“谁发明了批量 grilling”的证明。我的初版是用户设定上限后的谨慎合并，Matt 的版本则默认询问整个 frontier。两者共享依赖安全的方向，具体策略并不相同。更合理的理解是，我们遇到了相似的问题，各自走到了相近的答案。

## 我吸收了什么，又删掉了什么

我重新读了最新版上游 `grilling`，其中有几处改进值得保留。

- 用 decision tree 和依赖顺序约束问题，而不是预先生成一张固定问卷。
- 只有前置条件已经确定的问题才能进入当前轮。
- 把事实和决策分开。事实由 Agent 调查，决策才交给用户。
- 新答案改变原有假设时，重新计算后续问题。
- 问题结束不等于可以开工，仍要等待用户确认。

我没有照搬“询问整个 frontier”。模型判断两个问题是否独立，本身就可能出错；一轮问题过多，也会把 Agent 的轮次成本转移成人的认知负担。上游的 [讨论 #663](https://github.com/mattpocock/skills/issues/663) 里，有用户明确提出批量模式削弱了原来一次一问的专注感。其他使用者则认为，自己已经比较清楚目标时，批量模式快得多。两边都没有错，它取决于任务和使用者。

我也没有保留固定的问答外观。最新版上游要求每个问题使用 `❓` 和 `➡️`，再配上固定的标题格式。我个人不喜欢这种表达。表情符号和模板没有改变决策行为，却占用了 skill 文本和回答样式。对一个底层访谈 skill 来说，我更希望规则约束行为，而不是规定语气。

最后形成的版本采用了一个中间方案：

| 设计点             | 当前做法                                     |
| ------------------ | -------------------------------------------- |
| 每轮问题数         | 默认一个；用户可以给出上限，但不是必须凑满   |
| 合并条件           | 问题关系紧密、前置条件已确定，并且彼此不依赖 |
| 明显更优的选择     | Agent 直接决定，不提问                       |
| 需要提问的内容     | 只保留存在真实取舍的决策                     |
| 可调查的事实       | 使用当前上下文和工具自行解决                 |
| 被新信息推翻的决定 | 重新打开相关分支                             |
| 结束条件           | 总结已达成的设计并请用户核实                 |
| 代码实现           | 另行请求明确授权，不能把讨论许可当成实现许可 |

目前整个 [`SKILL.md`](https://github.com/wufei-png/skills/blob/main/skills/productivity/grilling/SKILL.md) 连 frontmatter 也只有 17 行。我希望它保留 Matt Pocock 写 skill 时那种克制和高信息密度，同时允许使用者自己决定一轮能处理几个问题。

## 我为什么又做了三个 review skill

`grilling` 现在已经迁入我的 [skills 仓库](https://github.com/wufei-png/skills)。仓库里还有三个 engineering skill。它们延续了同一个原则：不要重新发明一套完整开发方法，只补上我在实际使用中反复需要的控制点。

### `delegated-change-review`

这是最小的一个审查门。它的核心不是再写一套更长的 review prompt，而是把审查上下文和实现上下文分开。新的 reviewer 使用 `fork_turns: "none"` 启动，默认不继承主会话历史，只拿到本次审查需要的目标和约束。它看到的是代码和变更本身，不是实现者一路走来的解释。

具体的审查能力直接复用 Codex 官方内置的 `$review-agent`。这是一个简洁、通用、只读、以缺陷为先的 review skill，会检查完整 diff、周边代码、测试和调用点，只返回具体且值得修复的问题。`delegated-change-review` 不再自己发明另一份审查标准，只负责隔离上下文、限定职责和处理审查结果。

reviewer 提交报告后就退出。当前任务的 owner 要独立核实每条发现，记录不接受的理由；如果原来的实现子 Agent 仍然负责这项改动，就把接受的意见交还给它修复，否则由主会话修复。相关测试和最终结论也由 owner 负责。reviewer 始终只是 reviewer，不会在发现问题后顺手变成实现者。

它适合放进其他流程内部，例如一个实现阶段完成以后做一次独立审查。它刻意只有一轮，不负责规划、提交或反复追踪。与 Superpowers 的 [`requesting-code-review`](https://github.com/obra/superpowers/blob/main/skills/requesting-code-review/SKILL.md) 相比，它更短，也不要求进入 Superpowers 的完整任务生命周期。

### `review-loop`

`review-loop` 可以理解成多轮版本的 `delegated-change-review`。有些改动经过一轮修复后，还可能引入新的问题，因此它把“隔离审查、核实、修复、测试”做成有上限的循环。用户可以自定义 `max_rounds`，没有指定时默认最多三轮。

它更强的地方不在于单个 reviewer 获得了更多权限，而在于每轮修复之后都会换一个全新的只读 reviewer。新的 reviewer 同样不继承主会话和上一轮审查的上下文，不会因为知道某个问题“已经修过”就倾向于接受当前结果。Minor 问题不会阻塞循环，Critical、Important 和确实值得处理的问题才会进入 owner 的裁决。

我在需求复杂、主会话已经很长时尤其喜欢用它。即使实现来自我当前使用的 GPT-5.6 Sol 这类旗舰 coding model，新的 reviewer 通常仍能找到值得修复的问题。这不是一项模型能力评测，也不意味着 reviewer 比实现模型更聪明。很多时候，只是因为它没有参与前面的推理，更容易看到主会话已经习惯甚至忽略的假设。

它适合边界复杂的 bug 修复、重构和合并前清理。如果只是想得到一份审查报告，Matt Pocock 的 [`code-review`](https://github.com/mattpocock/skills/blob/main/skills/engineering/code-review/SKILL.md) 会并行检查规范和需求两个维度；如果需要审查后继续修复，并用新 reviewer 检查结果，`review-loop` 更贴近这个目标。

### `review-gated-implementation`

这个 skill 用在需求、范围、仓库和实现权限都已经明确之后。它把改动拆成依赖有序的阶段，每个阶段都必须产生可独立验证的结果，经过检查、`delegated-change-review` 和最终验证后形成一个干净提交。所有阶段完成以后，再对完整变更运行一次集成审查。

它与 Superpowers 的 `subagent-driven-development` 最接近，但侧重点不同。Superpowers 提供 worktree、详细计划、TDD、逐任务 implementer 和多层 review 组成的完整方法；我的版本不规定谁来实现，也不接管前面的需求和计划阶段。它只关心已经授权的变更能否被切成可验证、可审查、可恢复的提交。

它也应该和 Matt Pocock 的 [`to-tickets`](https://github.com/mattpocock/skills/blob/main/skills/engineering/to-tickets/SKILL.md) 加 [`implement`](https://github.com/mattpocock/skills/blob/main/skills/engineering/implement/SKILL.md) 放在一起看。`to-tickets` 把较大的方案拆成带阻塞关系的 tracer-bullet tickets，发布到 issue tracker，并让每个 ticket 适合由一个全新会话独立完成。`implement` 按一次一个 ticket 的节奏执行 TDD、测试、code review 和提交。这套组合更关心如何把一项跨会话的工作分发出去。

`review-gated-implementation` 不创建 ticket，也不接管 issue tracker。它在同一个 working tree 中处理一项已经授权的变更，默认最多拆成十个依赖有序的阶段。每个阶段都要验证、检查 staged diff、接受隔离审查并形成干净提交，最后再对整个变更做一次集成审查。它更关心的不是“怎样把大任务发给多个会话”，而是“怎样让这一个变更以一串始终有效的提交安全落地”。

如果你需要一套从想法到合并的统一工程方法，Superpowers 更合适。如果工作需要跨多个会话、通过 tracker 分发，Matt 的 `to-tickets + implement` 更自然。如果你已经有自己的工作流，只想让一项已授权的改动经过分阶段验证和独立 review，`review-gated-implementation` 更直接。

## 应该在什么时候使用

我的建议很直接：

- 方案仍有重要分歧，错误决策会带来明显返工时，使用 `grilling`。
- 需求复杂、主会话很长，或者改动值得经过几轮独立审查和修复时，使用 `review-loop`。
- 只需要在现有流程中插入一次只读审查时，使用 `delegated-change-review`。
- 任务已经获准实施，而且适合拆成多个可验证提交时，使用 `review-gated-implementation`。

简单事实、低风险的小改动以及一次命令就能验证的机械工作，不必为了“流程完整”强行调用它们。

四个 skill 都只能手动触发。`disable-model-invocation: true` 和 Codex 的 `allow_implicit_invocation: false` 明确阻止模型自行决定何时加载。这个选择会牺牲一点自动化，但能换回更可预测的上下文成本和授权边界。

可以一次浏览并选择需要安装的 skill：

```bash
npx skills@latest add wufei-png/skills
```

如果单独安装 `review-gated-implementation`，记得同时选择它依赖的 `delegated-change-review`。

如果只想先试一个，我推荐从 `grilling` 开始。下一次准备让 Agent 实现一个还没完全想清楚的方案时，先让它把真正的取舍找出来。问题可以问得很严格，skill 本身不必很重。

这也是我现在设计这些 skill 的核心原则：简洁，高效，只在确实需要判断的地方增加流程。

欢迎试用，也欢迎把具体的失效案例告诉我。对 skill 来说，最有价值的反馈不是“感觉不错”，而是它在哪一轮问多了、漏问了，或者越过了本来应该停下来的授权边界。
