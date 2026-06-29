> AI 驱动开发方法论 ｜ 第四部分 · Implement 实现
> 目录见 [README](README.md)

# Monorepo：给 AI 一个没有上下文墙的代码库

进入实现（Implement）阶段后，代码仓库的物理结构会从底层直接制约 Agent 的交付质量。如果采取传统的"多仓库（Multi-Repo）"拆分模式，每一个物理仓库边界都会在无形中成为一堵信息屏蔽墙，割断了 Agent 探索整体项目关联上下文的视野。

**高频协同修改和共享数据契约的模块必须组织在同一个 Monorepo 中，以消除阻碍 Agent 跨包类型检索与依赖调用的上下文墙。**

Agent 产出代码的质量与缺陷率，高度取决于它在当前执行上下文里能直接"看见"多少与之关联的代码迹线。当仓库边界把这些关联生硬切断时，Agent 只能在盲区中依靠猜测进行盲目编码。

## 上下文墙：多仓拆分带来的破绽

假设我们将 Relay 示例项目按照常规思路拆分为多个物理仓库：`relay-frontend`（坐席台前端）、`relay-worker`（异步退款执行器）以及 `relay-contracts`（共享数据契约仓）。

当我们需要让 Agent 去修改自动退款的额度校验接口时，这一改动是跨越仓库物理边界的：它必须首先去 `relay-contracts` 修改类型定义，发布新版本；再去 `relay-worker` 升级依赖、修改实现逻辑；最后去 `relay-frontend` 引入新字段。

对于 Agent 而言，当它在 `relay-worker` 仓库工作时，它在本地无法直接看到前端是如何解析出款状态的，也看不见共享契约的完整调用地图。每一次跨仓库的操作，都迫使人类工程师充当"人肉数据搬运工"来回同步上下文。

在 Monorepo 的结构下，物理边界被彻底打破。Agent 在同一个上下文进程中，可以瞬间顺着调用链扫描到类型定义、所有关联的后端实现、前端消费组件以及对应的端到端集成测试，一次性完成全局重构。AI 最擅长执行这种跨切面的全局同步修改，多仓模式只会把这一长处彻底削弱。

## 物理边界与噪声过滤

Monorepo 并不等同于将公司内所有不相关的代码无脑堆进一个文件夹里。大仓的物理体积过大同样会带来负面代价：
* 导致本地静态扫描与 CI 校验速度呈指数级变慢，拉长反馈链路。
* 引入大量无关的背景代码噪声，干扰 Agent 的局部逻辑分析能力。

判断一个模块是否应当并入同一个 Monorepo，核心指标在于评估它们在演进时的**"协同修改频率"**与**"契约共享强度"**：
* 频繁一起修改、彼此共享核心数据协议的模块（如前后台服务、核心领域层、共享契约层）必须归入同一个 Monorepo。
* 独立演进、几乎无契约依赖且无需协同发布的子系统（如独立的内部基建脚本、离线数据抽取任务）则应当物理剥离，避免造成干扰。

## 工具选型：从哪里开始

Monorepo 的配置文件本身由 Agent 生成，但选哪个工具体系是你的判断，不能让 Agent 替你决定。下面是三个主流方案的对照：

| 工具 | 适合场景 | 核心特性 | 上手成本 |
|------|----------|----------|----------|
| pnpm workspaces | 纯依赖共享，不需要构建编排 | `workspace:` 协议、`pnpm -r` 批量运行 | 低 |
| Turborepo | 需要构建缓存加速（尤其 CI） | 任务依赖图、本地 + 远程构建缓存 | 中 |
| Nx | 需要代码生成器、微前端、插件生态 | 受影响模块检测、插件体系、代码生成 | 高 |

选择建议：新项目从 pnpm workspaces 开始，够用且复杂度低。如果 CI 全量打包时间超过 10 分钟，加入 Turborepo 的构建缓存层。只有当项目需要微前端架构或多平台代码生成器时，才考虑引入 Nx。引入过早的工具复杂度，会让 Agent 在配置文件层面的 debug 时间远超收益。

## 主线演练：Relay 单仓工作区配置与跨切面重构

Relay 示例项目在本地采用 pnpm workspaces 组织其 Monorepo 工程结构：

```yaml
# pnpm-workspace.yaml
packages:
  - "packages/contracts"  # 共享类型与接口契约
  - "packages/workers"    # 自动退款与核心编排 Worker
  - "packages/frontend"   # 坐席审批台前端应用
  - "packages/gateway"    # 多渠道消息接入网关
```

当负责 `RELAY_REFUND_01` 任务的 Agent 尝试为自动退款接口 `StripeRefund` 增加 `refund_reason` 字段时，得益于 Monorepo 的零屏障特征，Agent 能够直接在本地会话中检索并修改所有关联子包的代码：

1. **契约修改**：更新 `packages/contracts/src/refund.ts` 中的类型定义。
2. **逻辑消费**：在 `packages/workers/src/handlers.ts` 的退款消费者中填充对应的退款原因字段。
3. **前端消费**：同步修改 `packages/frontend/src/components/RefundModal.tsx`，将审批弹窗内的文本域表单与新加入的 `refund_reason` 字段进行绑定。

如果是多仓结构，在 worker 仓里工作的 Agent 在修改契约类型时，前端仓极有可能因未同步感知而发生静默失败。单仓结构让 Agent 一次看全、一次改对，并最终产出一个物理原子提交。

然而 Relay 最初并非这个结构。早期版本把 `contracts` 作为独立 npm 包单独维护，每次改动类型定义都要走 `npm publish` 流程，然后人工在各消费仓升级版本号。当 Agent 被分配到 `RELAY_REFUND_03` 任务时，它在 `packages/workers` 里找到了 `IRefundRequest` 接口，试图直接修改，但拿到的是旧版 `1.2.0` 的类型定义，而前端已悄悄升到 `1.3.0`。Agent 的修改通过了 worker 侧的本地类型检查，却在前端的集成测试里炸了：

```
TypeError: Cannot read properties of undefined (reading 'refund_reason')
    at RefundModal.tsx:47
```

排查花了将近两个小时，因为报错位置在前端，原因在 contracts 版本错位，现象在 worker 的消费逻辑。这不是 Agent 能力不足，而是物理结构把它逼进了盲区。把 `contracts` 并入 Monorepo 后，`packages/contracts` 通过 `workspace:*` 协议在各包间实时共享，版本号错位问题从根上消失。

## CI 中的受影响包检测

Monorepo 结构带来了一个新问题：包越多，如果每次 CI 都跑全量测试，时间会随包数线性增长。Relay 发展到六个包时，全量 CI 已经需要 23 分钟，大量时间花在和本次 commit 毫无关联的包上。

做法是让 CI 只跑"受当前 commit 影响的包"。Turborepo 内置了基于构建图的受影响检测：

```bash
# 只运行受当前 commit 影响的包的测试
pnpm turbo run test --filter=...[HEAD^1]
```

`--filter=...[HEAD^1]` 的含义是：找出从 `HEAD^1` 到 `HEAD` 这一个 commit 里有变更的包，再沿构建依赖图向上传播，找出所有依赖这些包的上游包，一并纳入测试范围。这是构建图上的传播，不只是直接修改过文件的包。如果你只改了 `packages/contracts`，而 `packages/workers` 和 `packages/frontend` 都依赖它，三个包都会跑测试。

pnpm 原生也有类似过滤能力，不需要 Turborepo：

```bash
# 找出相对于 main 分支有修改的 workspace，及其下游依赖包
pnpm --filter "...[origin/main]" test
```

在 GitHub Actions 里的完整用法：

```yaml
# .github/workflows/ci.yml（片段，由 Agent 根据 turbo.json 配置生成）
- name: 运行受影响包的测试
  run: pnpm turbo run test --filter=...[origin/main]
  env:
    TURBO_TOKEN: ${{ secrets.TURBO_TOKEN }}   # 可选：开启远程构建缓存
    TURBO_TEAM: ${{ secrets.TURBO_TEAM }}
```

Relay 切换到受影响检测后，日常 feature 分支的 CI 时间从 23 分钟降到 4-6 分钟，原因是绝大多数 commit 只触碰一两个包。只有修改 `packages/contracts` 这种被所有包依赖的基础层时，才会触发接近全量的测试。这个反馈速度的差异，决定了 Agent 能不能在一个工作会话内完成"改代码 → 看 CI 结果 → 修正"的闭环。23 分钟的等待基本宣告闭环断裂。

`turbo.json` 本身让 Agent 生成初稿：把 `pnpm-workspace.yaml` 的包结构和 `package.json` 里的脚本名称一并给 Agent，告诉它"生成任务依赖图，`build` 在 `test` 之前，`contracts` 是其他包的上游"，Agent 能生成可用的配置。你的工作是 review 依赖关系是否和实际构建顺序一致，而不是手写 JSON。

## 大型 Monorepo 的上下文窗口管理

当 Monorepo 有 30 个以上的包时，让 Agent 读整个仓库变得不现实。上下文窗口有限，而且读了大量无关包的代码会降低 Agent 对目标模块的聚焦度。

几个实践在 Relay 里验证有效：

**在 `AGENTS.md` 里标注当前任务相关包。** 每次为 Agent 开启新任务时，在任务描述或 `TODOS.md` 的任务条目里写明"本次任务仅涉及 `packages/payment` 和 `packages/refund`"。Agent 冷启动后读到这一行，就知道把文件系统视图限定在这两个目录，不去扫描其他包。没有这一行时，Agent 的惯性是从仓库根目录开始探索，大量读取无关文件，很快就把上下文窗口的主要位置占满。

**每个包自带 README 说明职责边界。** `packages/` 下每个包的 README 写两件事：这个包做什么，以及它对外暴露的核心接口（函数签名或 HTTP 端点列表）。Agent 做跨包任务时，先读目标包的 README 定位接口，再按需深入实现文件，而不是从 `index.ts` 开始盲扫。Relay 的 `packages/gateway/README.md` 里有一张渠道接入接口表，Agent 处理消息路由任务时平均少读 40% 的源文件。

**`TODOS.md` 任务条目标注"涉及包"字段。** 任务格式如下：

```markdown
## RELAY_REFUND_05：为退款审批增加操作人备注字段

涉及包：packages/contracts, packages/workers, packages/frontend
状态：待开始
背景：产品要求坐席在手动审批退款时必须填写备注原因，供合规审计。
```

Agent 拿到这个任务条目，就知道从哪三个包开始，不需要自己去仓库里推断。这个字段同时也是人工 review 的检查点：如果 Agent 的改动范围超出了"涉及包"列表，通常意味着设计出了问题。

## gstack 参考实现

在 gstack 体系中，Monorepo 工作区通常采用 `pnpm workspaces` 进行依赖隔离，并使用 `Turborepo` 实现多包的构建流水线加速。

工作区通过根目录下的 `pnpm-workspace.yaml` 声明物理子包边界，子包之间通过 `workspace:*` 依赖协议引用，从而保证在本地运行时实时感知代码签名变化：

```json
// packages/workers/package.json
{
  "dependencies": {
    "@relay/contracts": "workspace:*"
  }
}
```

在 CI 和本地提交流水线中，通过 `turbo.json` 配置的任务依赖关系（如 `test` 任务依赖 `build` 任务的产出），gstack 利用 Turborepo 的 filter 机制自动限制执行范围：

```bash
# 由 CI 运行的受影响测试过滤指令
$ pnpm turbo run test --filter=...[origin/main]
```

这套大仓的编排和过滤配置构成了物理交付的实现层。通过把复杂的构建依赖关系托付给工具链自动编排，Agent 在冷启动时仅需感知它当前任务涉及的特定子包范围，从而有效管理上下文窗口的有效容量。

## 典型反模式

* **强相关业务拆为物理多仓**：让 Agent 频繁面临"隔墙改代码"的局面，导致开发和调试链路极其冗长。
* **毫不相关的系统硬塞进单仓**：导致构建耗时失控，且过大的仓库体积会引入无关的静态类型噪声，污染 Agent 的上下文分析。
* **单仓内部无依赖规则约束**：单仓消除了上下文墙，但也极易导致内部包之间随意越层循环引用，演变为一团乱麻。
* **CI 跑全量测试不做过滤**：包数超过五六个后，全量 CI 时间会让 Agent 的修改-验证循环拉长到一个工作会话无法完成，实际上是在用构建时间惩罚 AI 速度优势。
* **任务条目不标注涉及包**：Agent 冷启动时靠自己推断任务范围，会把上下文窗口的大量位置浪费在无关包的探索上，降低输出质量。

## 本章要点

* Monorepo 能够为 Agent 提供无上下文屏蔽墙的全局研发视野，最适合放大 AI 跨切面重构的效能优势。
* 工具选型从 pnpm workspaces 开始，CI 打包超 10 分钟再加 Turborepo 缓存，需要微前端或代码生成器才考虑 Nx。
* `--filter=...[HEAD^1]` 让 CI 只跑受当前 commit 影响的包，是保住 AI 修改-验证快节奏闭环的基础手段。
* 原子提交机制保障了前端、后端与契约层的协同发布一致性，消除了多仓合并时间差带来的破绽。
* 是否归入单仓应以"协同修改频率"和"契约共享强度"为边界约束；单仓结构必须配合刚性分层依赖规则，防止内部架构腐化。
* 30 个包以上的 Monorepo 要在 `AGENTS.md` 和 `TODOS.md` 里主动标注任务涉及的包范围，避免 Agent 冷启动时盲扫整个仓库耗尽上下文窗口。

> Monorepo 抹去了上下文边界，但也为"包之间循环依赖"敞开了大门。我们必须在单仓内建立起清晰的单向依赖规矩。下一章探讨如何依靠编译工具和 CI 门禁强行锁死依赖方向。
