> AI 驱动开发方法论 ｜ 第三部分 · 立项与规划（Think + Plan 阶段）
> 目录见 [README](README.md)

# 把方案拆成 Agent 能领取的任务与交付物追踪

在方案锁定与设计文档就绪后，规划阶段的最后一环是将系统设计切分为逻辑独立、职责单一的任务模块。这道切割工件直接决定了下一步并行 Worktree 开发的吞吐上限。如果划分边界模糊、多项任务存在重叠，并行的 Agent 会因为在同一物理代码块上修改冲突而造成灾难性的覆盖与混乱。

**必须将锁定的方案拆解为物理边界清晰、输入输出明确且单一职责的原子任务，建立起从 Spec、设计文档、任务单到代码提交的轻量级追溯链条。**

只有当任务的物理边界被清晰定义，且具备明确的输入与输出时，我们才能放心地将任务授予 Agent 自主完成，实现真正意义上的“监督式自治”。

## 原子任务拆解：规避越界冲突

AI 驱动开发的高吞吐红利建立在“并行 Worktree”的机制上。但若没有在前置阶段将方案合理地进行物理切分，并行就会成为代码冲突的制造器。

如果我们将“开发自动退款功能”作为一个大包袱直接塞给单个 Agent，它不仅会因为任务过重而在超长上下文会话中迷失，也阻断了多线并行的可能性。正确的做法是将其解耦拆分为多个无物理交叠的子任务：

* **退款风控判定逻辑**：纯算法计算，输入退款上下文，输出分类决策（自动/审批/拦截），其修改范围限制在独立的领域决策层。
* **审批申请状态机**：维护 `ApprovalRequest` 实体，状态变更为 `Pending` 挂起、`Approved` 审批通过以及 `Rejected` 驳回。
* **外部出款调用与 Saga 回滚**：调用第三方支付网关接口，处理网络重试与金额冲正。

对于每一个子任务，必须使用硬性不变量约束其修改范围（如“本任务仅允许改动 `src/domain/refund/decision.ts`”）。一旦任务具备了单一职责和可独立验收的特质，Agent 就不会在不相关的模块中越界改动，从而确保并行线之间互不串台。

## 工件可追溯性：从代码追踪设计意图

在工程生命周期中，我们常在代码中遇到看似冗余或多余的校验分支（例如在扣减库存前多做一次无谓的二次乐观锁校验）。如果缺乏工件可追溯链，后续接手的 Agent 可能会因为看不懂其技术意图，而将其当作冗余的历史包袱误删，导致生产环境发生死锁或超卖。

所谓交付物追踪，即是建立一条从 Spec（需求编号）→ 设计决策（文档段落）→ 任务单（Task ID）→ 最终代码提交与测试用例的闭环链路。

这一链路不需要依赖重型的看板或复杂的商业管理系统，只需在开发约定中强制维护一个轻量级的关联契约。例如，Spec 中的条款被打上独一无二的编号（如 `REFUND_GATE_01`），后续所有的衍生设计、任务定义以及 Git Commit Message 都必须携带此编号作为前缀。即使在半年后，任何冷启动的 Agent 都能通过提交哈希一路反向追溯到最初的业务动机。

## 主线演练：Relay 退款任务拆解与提交迹线

在 Relay 示例项目的开发规划中，锁定后的方案被拆解为具体的任务配置文件，作为并发 Agent 领取的物理凭证：

```yaml
# .agents/tasks/task_refund_decision.yaml
task_id: "RELAY_REFUND_01"
spec_ref: "SPEC_REFUND_v1#section_3.1"
design_ref: "docs/designs/auto_refund_v1.md#section_2"
scope:
  files:
    - "src/domain/refund/decision.ts"
    - "src/domain/refund/__tests__/decision.test.ts"
  allowed_modifications:
    - "refundDecisionRules"
inputs:
  type: "RefundContext"
outputs:
  type: "DecisionResult"
```

当负责该任务的 Agent 完成开发并通过本地静态门禁后，它会生成一条严格携带任务追溯标识的 Commit Message：

```bash
$ git log -n 3 --oneline
commit a1b2c3d [RELAY_REFUND_01] feat: implement refund risk decision and transaction limit rules
commit e4f5g6h [RELAY_REFUND_02] feat: setup asynchronous refund queue and dead letter handlers
commit 9876543 [RELAY_REFUND_03] feat: construct striped idempotent client and Saga rollback logic
```

通过这一轻量级的物理链路，代码仓库中的每一行变更均具备了明确的来历。人类审查者在 Code Review 时能立刻提取出该 PR 背后对应的任务与需求边界，大大减轻了评审的认知负担。

## 典型反模式

* **将高度耦合的超级大任务单塞给 Agent**：导致会话迅速耗尽上下文窗口产生漂移，最终产出无法局部验收的半成品。
* **边界模糊的任务并发分发**：两个任务定义中同时包含对共享业务编排模块的随意修改权限，导致两个 Worktree 中的 Agent 互相覆盖对方的代码。
* **孤立的代码提交**：提交记录只写着“fix: refund”，无 Task ID 也无 Spec 引用，导致代码的业务意图在合并后彻底丢失。

## 本章要点

* 边界清楚与单一职责是任务拆解的前提，任务的粒度直接决定了并行的效率与规模。
* 利用相对路径下的配置文档显性约束任务的输入输出与修改文件范围，防止 Agent 跨模块乱改。
* 通过 Spec 编号一路透传至 Commit Message，建立轻量级、零工具负担的工件可追溯链。

> 随着立项规划的完成，方案与原子任务均已锁定。接下来我们将进入 Build 阶段的工程基座设计。我们将在第 11 章 `Monorepo` 中探讨如何为 Agent 提供一个没有物理隔离代码墙的全局工程环境。