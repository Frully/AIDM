> AI 驱动开发方法论 ｜ 第二部分 · 搭建脚手架
> 目录见 [README](README.md)

# 项目宪法与 Agent 协同原语

构建 Harness 研发脚手架的起点，是为项目确立一部不变量宪法——一个集中定义所有底线规矩的刚性文件。该文件必须作为 Agent 每次启动会话时的冷启动必读输入。

**项目应当确立一个通用的 `AGENTS.md` 文件作为全局单一事实源，集中定义系统的业务不变量、架构分层规则与安全红线，并通过软链接兼容不同 Agent 工具的上下文加载机制。**

这部不变量宪法是 Harness 中“文档”机制的基石，它决定了 Agent 的产出被限制在怎样的边界内。

## 项目宪法的物理形态与载体设计

Agent 在每次新建会话时都处于冷启动的“无记忆”状态。即使你在上一次会话中反复强调“交易金额需用分表示”，新会话启动后它依然对此一无所知。如果这些规矩只口头交代在聊天历史里，你很快就会因为重复说教而疲惫，规则也会随之发生漂移。

将不变量沉淀为项目库中的物理文件，问题便迎刃而解：规则拥有了确定的物理位置，Agent 每次加载上下文时都会将其吞入。同一条规则不应存在第二个出处，否则多处冲突将直接导致 Agent 执行决策产生分歧。这即是“单一事实源”（SSOT）在研发约束层面的实践。

在命名上，建议使用通用的 `AGENTS.md` 放置在项目根目录下。由于不同的 AI Agent 工具在默认行为上可能倾向于读取不同的指令文件名（例如 Claude Code 倾向于加载根目录下的 `CLAUDE.md`，而 Cursor 则读取系统指定的 Prompt 文件），我们不应当为不同的工具各自维护一份内容相同的规则文件。

规范的工程实践是：以根目录下的 `AGENTS.md` 为真文件，而对于其他特定 Agent 工具所需的指令配置文件，通过软链接（Symbolic Link）将其指向 `AGENTS.md`：
```bash
$ ln -s AGENTS.md CLAUDE.md
```
这样既保证了所有工具在启动会话时拉取的是同一份约束内容，又避免了多份副本带来的维护噩梦。

宪法文件的内容组织应极度克制，主要包含以下三类核心指令：
* **业务与技术不变量**：在系统中永远成立的物理事实（如“交易状态仅包含初始化、处理中、成功、失败四种”）。
* **分层依赖规则**：系统设计上强化的架构依赖防线（如“应用层不许越权访问底层的持久化层模型”）。
* **安全红线**：绝不可触碰的物理雷区（如“日志脱敏规范下严禁明文打印用户手机号”）。

宪法应当保持短小精炼。它应当是一张用于快速匹配的拦截网，而非长篇大论的学习教程。信息密度过低会导致模型在长上下文检索中发生关键红线遗忘。

## 自动化 Harness 体系的工程原语

仅有静态的 `AGENTS.md` 宪法尚不足以构成一个自愈的反馈回路。必须借助开发环境提供的一组工程原语，将静态的文档同动态的编译、运行卡口有机连接：

* **Hooks（自动化钩子）**：在特定的生命周期时机（如提交前 `pre-commit`、推送前 `pre-push` 或会话启动 `session-start`）自动执行的脚本，是“强制性检查”自动运行的载体。
* **Settings（项目级配置）**：定义 Agent 在当前工作区内的系统行为限制与可用工具权限（如限制其仅能访问特定的子目录或禁用特定命令行指令）。
* **Subagents（子代代理）**：用于派生拥有独立干净上下文的子 Agent 去并发专注完成某项子特性开发，防止主会话上下文污染与 Token 拥堵。
* **Slash Commands / Skills（快捷指令/技能扩展）**：将复杂的多阶段工作流（如跑完单元测试后收集报告并一键发布灰度）固化为命令行命令，供 Agent 一键调用。

重点在于理清这些原语在整体 Harness 回路中所扮演的角色。配置 Git Hook 的动机并非追求工具链的完整度，而是为了在代码提交的物理关口处，强制触发 Lint 与安全扫描门禁。

## gstack 对工程原语的封装设计

在 gstack 工具链的参考实现中，上述工程原语被内置且开箱即用：

* **Skills 命令扩展**：gstack 将七阶段生命周期的各核心卡口固化为内置的扩展指令（如 `/plan-review` 调起多视角方案评审，`/ship` 自动校验流水线后进行灰度部署），Agent 可在命令行中无缝触发这些定义好的标准化工作流。
* **SessionStart Hook 挂载**：利用 Hook 机制，在 Agent 启动新的工作对话时，自动为当前分支挂接相对独立的 `worktree` 隔离环境，自动将 `AGENTS.md` 注入其系统上下文。
* **软链兼容方案**：项目规范默认在根目录下建立 `AGENTS.md`，并在构建时自动生成指向它的工具适配链（如 `CLAUDE.md`）。

即使在特定的私有开发环境中不采用 gstack，上述原则依然可迁移：通过本地 Shell 脚本手动挂接 `.git/hooks`，使用自定义的脚本执行 CI 门禁拦截，且依然将约束写入全局 `AGENTS.md` 中。

## 主线演练：Relay 宪法的条目设计与门禁对照

以 Relay 为例，我们来看一套硬核的 `./AGENTS.md` 条目是如何设计的，以及它们背后对应的检查卡口：

```markdown
# Relay 核心宪法 (.agents/rules.md)

## 1. 交易不变量
- 金额字段（Amount）在数据存储、内存计算与网络交互中一律使用整数分（Cents），严禁使用任何浮点数进行结算。
- 所有的退款操作（`RefundRecord`）在数据设计上必须支持冲正，数据表结构必须强制包含 `original_transaction_id` 唯一非空外键。

## 2. 架构依赖守卫
- 底层资金划扣核心（`src/services/billing/`）严禁被低权限的外围聊天通道层（`src/channels/`）直接 `import`，任何资金操作必须通过 `src/api/gateway/` 进行代理路由。

## 3. 安全与合规红线
- 敏感日志防护：应用生成的本地和生产日志中，严禁以任何方式输出用户的明文支付凭证（如 `CVV`、`BankCardToken`）及客户原始聊天记录。
- 支付重试幂等：调用扣款与退款的第三方服务接口时，必须在 Headers 中携带根据 `conversation_id` 与动作序列计算生成的 `Idempotency-Key`。
```

这些写入宪法的红线，其背后均有一套刚性检查工具在 CI 阶段执行强力兜底：
* 架构依赖守卫由静态依赖扫描工具（如 `dependency-cruiser`）强制执行。当开发 Agent 在尝试编写一个 SlackWebHook 快速卡片退款功能时，为了图快直接在 `src/channels/slack_webhook.ts` 中写了一行 `import { executeDirectRefund } from '../services/billing/refund'`。
* 在执行 `git commit` 时，挂接在 Pre-commit Hook 上的校验脚本被自动触发，终端弹出了如下报错迹线：
  ```bash
  $ npx depcruise src/
  ✖ dependency-violation: src/channels/slack_webhook.ts -> src/services/billing/refund.ts
    Rule ID: billing-layer-isolation
    Severity: error
    Description: Billing core modules must not be imported directly by presentation or channel layers.
  ```
  这一报错作为**反馈**返回给 Agent，使其在本地立即进行了逻辑重构，移除了越权依赖。
* 敏感日志防护则由敏感词与静态正则扫描插件在构建流水线中执行强力阻断。如果 Agent 试图在调试时插入一行 `console.log("refund data:", paymentDetail)`，流水线将直接报错挂起，杜绝风险代码上线。

## AGENTS.md 骨架模板

这个骨架可以直接复用：人填写、或者把项目描述扔给 AI 让它草拟、review 后提交进代码库。关键是写完之后每节都能回答同一个问题：「如果 Agent 冷启动只读这一节，它能拿到什么决策依据？」

```markdown
# AGENTS.md — [项目名称] Agent 宪法

## 项目是什么
[1-3 句话说明项目性质、主要交付物、目标用户]
[Agent 每次冷启动都会读这里，要能从这三句话理解当前要做的是什么]

## 语言与风格约束
[哪些语言、哪些格式是硬约束，不是建议]
- 所有代码注释和文档用简体中文
- API 错误消息用英文（与第三方集成保持一致）

## 主线示例
[约定贯穿全项目的示例领域，防止 Agent 随意发明新概念]
所有示例统一使用 [示例名] —— [一句话描述]

## 硬约束（Red Lines）
[写绕不过的工程红线，而非"尽量做到"的建议。能落进 lint 规则和 CI 的，同时标注检查工具名称]
- 金额统一使用整数分（cents），禁止使用浮点数；CI 的 `amount-linter` 会直接拦截
- 禁止在代码里使用 `console.log`；所有日志通过 `@relay/logger` 封装；ESLint `no-console: error` 强制执行
- 数据库迁移文件禁止包含 `DROP COLUMN`；`migration-linter` 在 CI 中检查

## 架构约束
[层依赖方向，AI 生成代码时必须遵守]
- 依赖方向：应用层 → 域层 → 基础设施层，禁止反向依赖
- 所有 LLM 调用只能在 `src/infra/llm/` 目录下，域层不能直接 import 模型 SDK
- Prompt 模板放在 `src/prompts/`，不能散落在业务逻辑里

## 保留惯例
[历史决策的记录，防止 Agent 推翻已经拍板的方案]
- 2024-03：退款流程选择异步队列而非同步，原因是支付网关 P99 超过 3 秒。同步方案已评估并否决。

## 工作约定
- 每次开始新任务前读 TODOS.md，按其中标记的优先级领取
- 提交信息格式：`[type]: 描述`（type: feat/fix/refactor/docs/test）
- 不要删除或修改 AGENTS.md 本身
```

有几点值得说明。

「硬约束」和「架构约束」两节是宪法密度最高的地方，也是最容易写废的地方。常见的废法是只写规则，不写执行工具：「所有金额用整数分」写完就结束。这样的规则 Agent 读了可能遵守，也可能在复杂上下文里遗忘。有效的写法是在每条规则后面注明对应的检查工具名——`amount-linter`、`no-console`、`depcruise`——让 Agent 知道「这不只是建议，CI 里有工具会拦截」，它对这条规则的遵守度就会明显提高。

「保留惯例」节容易被跳过，实际上很有必要。AI Agent 在生成代码时有一种自然倾向：看到异步队列就会问「为什么不做成同步的？」或者直接换成同步实现。把「已否决的方案及理由」写在宪法里，等于把历史决策固化为约束，防止 Agent 在每次新会话里重新开始争论已经拍板的设计。

宪法应当保持在 200 行以内。信息密度过低会导致模型在长上下文中漏掉关键红线，超长的宪法和没有宪法的效果差不多。

## SessionStart Hook：让约束在对话开始前就到位

即便 `AGENTS.md` 写得再严密，如果 Agent 不读，一切归零。CI 门禁能在代码提交时兜底，但它拦不住 Agent 在整个会话过程中产生的错误思路。

Claude Code 的 `.claude/settings.json` 支持在会话启动时自动执行 shell 命令，把需要 Agent 知道的信息直接打印到上下文里：

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "echo '=== Agent 会话启动 ===' && cat AGENTS.md && echo '=== 当前 TODOS ===' && head -50 TODOS.md"
          }
        ]
      }
    ]
  }
}
```

这个文件提交进代码库，团队里每个人开新会话时都会自动执行。效果直接：Agent 在对话开始时就看到了项目约束和当前待办，而不是等用户手动粘贴，或等到 CI 报错才意识到有规则。

对于内容稍复杂的项目，把启动逻辑拆进独立的 shell 脚本更清晰：

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/session-start.sh"
          }
        ]
      }
    ]
  }
}
```

`.claude/session-start.sh` 的内容可以让 AI 根据「会话开始时我希望 Agent 知道什么」来生成，不需要手写：

```bash
#!/bin/bash
echo "=== RELAY 项目 AGENT 会话启动 ==="
cat AGENTS.md
echo ""
echo "=== 当前开放的 Worktree ==="
git worktree list
echo ""
echo "=== 待处理的 TODOS ==="
grep -E "^- \[ \]" TODOS.md | head -20
```

生成后 review 一遍、提交进代码库，之后每次会话都会自动执行。需要调整输出内容时，直接改脚本就行。

### Relay 的实际教训

Relay 在引入 SessionStart Hook 之前，`AGENTS.md` 里的金额约束被触发过两次。第一次，Agent 在开发对话路由新功能时，计算了一个预估退款金额并存成了 `amount: 12.5`，浮点数。`amount-linter` 在 CI 里拦住了：

```
amount-linter: src/services/routing/escalation.ts:47
  Error: Float amount detected — use integer cents (1250) instead of 12.5
  Rule: no-float-amount
```

Agent 收到报错后修正了那一行，但没有意识到同一个函数里另外两处也用了浮点数——因为 CI 只报了第一个错，Agent 修完就提交了，第二次 CI 再次拦截。来回两次，合计浪费了将近半个小时。

加了 SessionStart Hook 之后，`AGENTS.md` 里「金额用整数分，`amount-linter` 强制执行」这条规则在会话开始时就出现在上下文里。同样类型的功能，Agent 在写第一行代码之前就用了 `1250`，CI 一次通过。前后的区别不在于规则本身变了，而在于规则在 Agent 决策时是否可见。

CI 门禁的作用是兜底，SessionStart Hook 的作用是预防。两者都要有，缺一个都会增加不必要的来回。

## 典型反模式

* **用冗长提示词取代宪法**：每次开启新会话时都通过手动黏贴大段的初始化 Prompt 来规定项目规范。这既浪费了昂贵的上下文 Token 空间，又会导致规则随着操作者的疏漏而产生边界漂移。
* **宪法沦为长篇大论的教程**：在 `AGENTS.md` 中塞入系统设计背景、开发流程指南或冗长的框架使用说明。如果文件字数膨胀至数千字，Agent 在检索时会产生注意力漂移，导致关键的安全红线被淹没在冗余信息中。
* **一规多源与规则冲突**：在 `AGENTS.md` 里写了一套数据库规范，而在 wiki 或设计文档中写了另一套过期的规范。当多份文档存在冲突时，Agent 将面临逻辑混乱，最终输出随机的代码决策。

## 本章要点

* **单一事实源**：项目宪法必须集中写在一个通用的 `AGENTS.md` 文件中，并采用软链接方式适配不同的 Agent 工具链。
* **规则组织准则**：宪法仅应写入业务不变量、架构分层规则和底线安全红线，内容必须极度精炼。
* **原语是连接零件**：利用 Hooks、Settings、Subagents 与 Skills 等原语，将静态的文档约束与动态的构建、运行拦截接通。
* **宪法与检查配对**：写入宪法的每一条“应该/不该”规范，其背后都必须设计有机器强制执行的“拦截卡口”进行兜底。

---

下一章将讨论另一类专门约束系统“视觉与体验一致性”的单一事实源：设计系统（Design System）。
