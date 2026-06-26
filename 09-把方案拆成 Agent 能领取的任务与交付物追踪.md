> AI 驱动开发方法论 ｜ 第三部分 · 立项与规划（Think + Plan 阶段）
> 目录见 [README](README.md)

# 把方案拆成 Agent 能领取的任务与交付物追踪

在方案锁定与设计文档就绪后，规划阶段的最后一环是将系统设计切分为逻辑独立、职责单一的任务模块。这道切割工件直接决定了下一步并行 Worktree 开发的吞吐上限。如果划分边界模糊、多项任务存在重叠，并行的 Agent 会因为在同一物理代码块上修改冲突而造成灾难性的覆盖与混乱。

**必须将锁定的方案拆解为物理边界清晰、输入输出明确且单一职责的原子任务，建立起从 Spec、设计文档、任务单到代码提交的轻量级追溯链条。**

只有当任务的物理边界被清晰定义，且具备明确的输入与输出时，我们才能放心地将任务授予 Agent 自主完成，实现真正意义上的"监督式自治"。

## 原子任务拆解：规避越界冲突

AI 驱动开发的高吞吐红利建立在"并行 Worktree"的机制上。但若没有在前置阶段将方案合理地进行物理切分，并行就会成为代码冲突的制造器。

如果我们将"开发自动退款功能"作为一个大包袱直接塞给单个 Agent，它不仅会因为任务过重而在超长上下文会话中迷失，也阻断了多线并行的可能性。正确的做法是将其解耦拆分为多个无物理交叠的子任务：

* **退款风控判定逻辑**：纯算法计算，输入退款上下文，输出分类决策（自动/审批/拦截），其修改范围限制在独立的领域决策层。
* **审批申请状态机**：维护 `ApprovalRequest` 实体，状态变更为 `Pending` 挂起、`Approved` 审批通过以及 `Rejected` 驳回。
* **外部出款调用与 Saga 回滚**：调用第三方支付网关接口，处理网络重试与金额冲正。

对于每一个子任务，必须使用硬性不变量约束其修改范围（如"本任务仅允许改动 `src/domain/refund/decision.ts`"）。一旦任务具备了单一职责和可独立验收的特质，Agent 就不会在不相关的模块中越界改动，从而确保并行线之间互不串台。

## 任务粒度的三个判断标准

拆任务的环节最容易犯两种错：拆得太细，或者没有拆。判断粒度是否合适，看三个指标就够了。

第一，**能独立验证**。任务完成后，有没有一个具体的交付物可以被单独验证？一个 API endpoint 可以被测试调用，一个 Service 方法可以写单元测试，一个 UI 组件可以在 Storybook 里看到——这些都算。如果"完成"了但什么都验证不了，说明任务太小，只是实现碎片，放进 Agent 会话时 Agent 拿不到足够上下文来做判断。

第二，**不跨越多个架构层**。一个任务里不应该同时修改数据库 schema、业务逻辑和 UI 展示。层面越多，Agent 做出错误决策的风险越高，review 时改动量也越大。每个 Agent 会话只动一层，出问题能精确定位。

第三，**Agent 不需要反复猜测意图**。任务描述里的上下文，应该足够 Agent 在不暂停询问的情况下做完。如果 Agent 多次暂停问"这里调哪个接口"或"这个字段要不要落库"，说明任务描述不够清晰，或者某些前置决策还没做完就下发了任务。

用 Relay 的退款场景做对比，三种粒度的差距很明显：

**太小的任务**："给 `RefundService` 类加一个 `validateAmount` 方法。" Agent 会在会话开始时就陷入困境：这个方法由谁调用？校验规则是什么？超限时抛异常还是返回 error code？没有可独立验证的交付物，任务只是一个代码碎片，没有办法写出有意义的测试。

**太大的任务**："实现 Relay 的退款功能。" 这跨越了数据库 schema（退款记录表）、Service 逻辑（`RefundService.processRefund`）、外部 API 调用（支付网关）、前端确认弹窗、通知发送五个层面。Agent 会话里没有检查点，review 时改动量太大，一旦出问题（比如支付网关调用方式搞错了），定位成本极高。

**合适的任务**：

```
实现 RefundService.processRefund 方法。

入参：RefundRequest（含 conversation_id: string、amount_cents: number、reason: string）
出参：RefundResult（含 refund_id: string、status: 'pending' | 'approved' | 'rejected'）

金额超过 50000 分时抛出 RefundLimitExceeded 异常。
接口类型定义已在 src/types/refund.ts 里锁定，直接 import，不要改动类型文件。
本任务修改范围：src/services/RefundService.ts 及对应的单元测试文件。
```

有明确的接口契约，有独立可测的交付物（单元测试能覆盖正常路径和异常路径），不跨层，Agent 读完任务描述就能开工。

## 跨任务依赖：接口契约先行

当 Task A 的输出是 Task B 的输入时，依赖关系处理不当会让并行任务互相等待，或者产生更危险的情况——两个 Agent 在各自的 Worktree 里对同一接口做出了不同的假设，集成时才发现对不上。

Relay 的退款流程里，`RefundService`（Task A）完成后需要调用 `NotificationService`（Task B）发送退款成功通知。如果同时启动这两个任务，Task A 的 Agent 会自行假设 `NotificationService` 的方法签名，Task B 的 Agent 也在写自己认为合理的接口。两个 Worktree 合并时，接口不匹配，集成失败。

处理原则：**把两个任务共用的接口类型定义作为独立的先行交付物锁定**，再启动依赖它的任务。

具体操作分三步。第一步，单独跑一个任务，只做一件事：把 `src/types/notification.ts` 里的接口声明写完并 review 通过。这个任务很小，但它是所有下游任务的地基。

```typescript
// src/types/notification.ts（先行任务的交付物）
export interface NotificationService {
  sendRefundConfirmation(params: {
    conversation_id: string;
    amount_cents: number;
    recipient_email: string;
  }): Promise<{ notification_id: string }>;
}
```

第二步，Task A 和 Task B 都 import 这个类型文件。TypeScript 编译器在类型检查阶段就能保证调用方和实现方的契约一致，不需要人工核对。

第三步，在 `TODOS.md` 里把依赖关系显式标出来：

```markdown
## RELAY_REFUND_02 - 实现 RefundService.processRefund

状态: 待开始
前置依赖:
  - task_notify_interface（通知 Service 接口定义）已完成 ✓

修改范围: src/services/RefundService.ts
接口约定: src/types/notification.ts（已锁定，不可修改）
```

这个模式的关键在于：两个任务之间的依赖是一份写进文件的接口定义，而不是另一个任务的实现状态。Task A 的 Agent 不需要等 Task B 实现完，只需要接口文件存在就能开工。

## 中途发现边界划错了怎么办

Agent 开始实现后，有时会发现原来的任务边界有问题。Relay 就遇到过这种情况：Task A 负责实现 `RefundService.processRefund`，Agent 开工后发现调用支付网关的 `PaymentGateway` 类接口设计有问题——`charge` 方法直接接受浮点数金额，而 `AGENTS.md` 的红线要求"金额使用整数分"。要完成 Task A，必须先修改 `PaymentGateway` 的接口，但 `PaymentGateway` 不在 Task A 的修改范围里。

错误的处理方式是让 Agent 在当前会话里"顺手改一下"。这会产生两个问题：这次 PR 的 diff 同时包含 Service 实现和网关接口重构两件事，review 时审查者很难判断两块改动的关联是否合理；而且 Agent 在扩大后的范围里做出的决策（比如如何重构网关接口），可能和原始架构设计里对 `PaymentGateway` 的定位不一致。

正确处理分三步：

**第一步，不扩大当前会话范围**。暂停当前 Agent，记录到 `TODOS.md`：

```markdown
## RELAY_PAYMENT_GATEWAY_REFACTOR - 重构 PaymentGateway 接口（新增）

状态: 待开始
优先级: 高（RELAY_REFUND_02 的前置依赖）
原因: processRefund 实现中发现 PaymentGateway.charge 接受浮点数金额，
     违反 AGENTS.md 红线「金额使用整数分」

修改范围: src/gateways/PaymentGateway.ts
目标接口:
  charge(params: { amount_cents: number; currency: 'USD' }): Promise<ChargeResult>
```

**第二步，先完成 `RELAY_PAYMENT_GATEWAY_REFACTOR`**，review 通过后接口锁定。

**第三步，重启 Task A 的 Agent 会话，更新任务描述**："假设 `PaymentGateway` 接口已重构为 v2，`charge` 方法接受 `amount_cents: number`，直接 import `src/gateways/PaymentGateway.ts` 使用。"

多一个任务、多一次 review，但每次 review 的 diff 边界都是清晰的。发现问题时也能精确定位是网关接口改错了还是 Service 逻辑错了。

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
* **孤立的代码提交**：提交记录只写着"fix: refund"，无 Task ID 也无 Spec 引用，导致代码的业务意图在合并后彻底丢失。
* **中途在当前会话扩大范围**：Agent 发现需要改动范围外的模块时，在当前会话里"顺手改掉"。这让单次 PR 的 diff 横跨两个不同职责的改动，review 时审查者无法判断两块改动的关联是否合理，出了问题也难以定位。

## 本章要点

* 边界清楚与单一职责是任务拆解的前提，任务的粒度直接决定了并行的效率与规模。
* 判断粒度是否合适，看三个指标：能独立验证、不跨越多个架构层、Agent 不需要反复猜测意图。
* 利用相对路径下的配置文档显性约束任务的输入输出与修改文件范围，防止 Agent 跨模块乱改。
* 跨任务依赖的解法是把接口类型定义作为独立先行任务锁定，让所有下游任务都 import 同一份契约文件。
* 中途发现边界划错时，停在当前会话，把新发现的问题记入 `TODOS.md` 作为前置任务，重启时给 Agent 更新的上下文——不要在一次提交里混入两个职责的改动。
* 通过 Spec 编号一路透传至 Commit Message，建立轻量级、零工具负担的工件可追溯链。

> 随着立项规划的完成，方案与原子任务均已锁定。接下来我们将进入 Build 阶段的工程基座设计。下一章探讨如何为 Agent 提供一个没有物理隔离代码墙的全局工程环境。
