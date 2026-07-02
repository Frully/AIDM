# AIDM · Harness 搭建清单（给 AI 的可执行指令）

> 这份文件只做一件事：**指导一个 AI Agent 在目标项目里把 harness 脚手架搭起来**。
> Harness = **文档（约束输入）+ 检查（拦截输出）+ 反馈（闭环自愈）**。下面每一节告诉你：建哪个文件、放在哪、里面写什么、配置长什么样。
> 不做的事：不教怎么规划需求、评审、业务验收——那是研发流程，不是 harness。这里只搭脚手架。
>
> **阅读约定**：正文是**通用结构，照着建**。凡带 `【示例·换成你的】` 标记的规则、字段名、代码片段，都是拿虚构项目 Relay（AI 客服 SaaS）举例说明"这类东西长什么样"，**不是要你照抄的规则**——换成你项目自己的不变量、字段名、关键路径。harness 的结构是通用的，填进去的内容是项目专属的。

---

## 0. 目标产物：一次把这些文件建出来

按这个树建，缺一个都会漏掉 harness 的一块。后续每节详述内容。

```
<project>/
├── AGENTS.md                      # 项目宪法（Agent 每次会话必读）
├── CLAUDE.md -> AGENTS.md         # 软链接，让各家 Agent 读同一份
├── DESIGN.md                      # 设计系统单一事实源（有 UI 才需要）
├── TODOS.md                       # 冷启动交接记忆
├── docs/
│   ├── design/                    # 设计文档：讲「为什么」这么做
│   ├── decisions/                 # 决策日志(ADR)：记录被否决的方案
│   ├── runbooks/                  # 机器可执行 runbook（if→then 规则）
│   └── qa/                        # QA 报告 + 截图/迹线物证
├── .github/workflows/ci.yml       # 远程门禁（不可绕过）
├── lefthook.yml                   # 本地门禁（pre-commit / pre-push）
├── .dependency-cruiser.js         # 分层依赖守卫
├── eslint.config.js               # lint 红线（no-console + 项目专属规则）
├── packages/logger/               # 结构化日志封装（禁 console.log）
├── scripts/
│   ├── wt-setup.sh                # worktree 动态环境装配
│   └── wt-clean.sh                # worktree 回收
├── tests/
│   ├── golden/                    # 关键正确性路径定额测试
│   ├── contracts/                 # 日志/接口字段契约(ajv)
│   └── adversarial/               # 注入对抗样本
└── .claude/
    └── settings.json              # SessionStart hook：自动喂 AGENTS.md
```

**建的顺序**：先 §1 文档 → 再 §2 检查 → 再 §3 反馈 → 最后 §4/§5 并行与自动装配。每建一个强制约束，配一个**负向测试证明它真的会拦**。原则是 **约束 > 提醒**：能落成机器门禁的，绝不写成一句提醒。

---

## 1. 文档层（约束输入）

### 1.1 `AGENTS.md` — 放仓库根，七节骨架

这是 Agent 每次冷启动的刚性输入。用通用文件名，再建软链接让其它 Agent 读同一份：

```bash
ln -s AGENTS.md CLAUDE.md
```

七节骨架照建；**每节填什么是你项目专属的**：

```markdown
# AGENTS.md — <项目名> Agent 宪法
## 1. 项目是什么          # 一句话定位 + 谁读这份、读它干什么
## 2. 语言 / 交流约束     # 产出语言、术语一致性（维护一份术语表）
## 3. 主线示例 / 领域词汇 # 统一命名，避免各处发明新名词
## 4. 硬约束（红线）      # 见下：把你项目的不变量列成红线
## 5. 架构层与依赖方向    # 见 §2.3，把「谁不能依赖谁」列死
## 6. 保留惯例            # 既有目录/命名/错误码/字段名不许乱改
## 7. 工作约定            # 作者与评审分离、TODOS.md 五要素、任务粒度
```

**§4 硬约束怎么写（这是 harness 最硬的一节）**：把你项目里"违反了就是 bug"的不变量逐条列出，**每条必须能落成一个机器门禁**——写不出执行工具的删掉（那是提醒，不是约束）。格式固定：`<约束> → <执行工具>`。

```
【示例·换成你的】Relay 的 §4 长这样，你替换成你项目的不变量：
- 金额一律整数分，禁浮点          → eslint 自定义规则
- 禁 console.log，走结构化日志封装 → eslint no-console
- 日志必带关联 ID(conversation_id) → logger 封装强制 + 契约测试
- 禁记录 PII（消息原文/手机/邮箱）→ eslint 自定义规则
- 依赖方向不许越层                 → dependency-cruiser
- 禁直接 DROP COLUMN，用两步迁移   → migration-linter
- 发钱路径必过定额+幂等测试        → CI 门禁
```

### 1.2 `DESIGN.md` — 放仓库根，动 UI 前必读

视觉/UX 决策集中一份：色板、字号阶梯、间距标尺、组件清单、动效时长。QA 时按它查漂移。没有 UI 的项目可省。

### 1.3 `docs/` — 按用途分四个子目录（别混在一起）

| 目录 | 放什么 | 判断标准 |
|---|---|---|
| `docs/design/` | 设计文档，讲**为什么**这么设计 | 一个冷启动 Agent 据此能直接动手 |
| `docs/decisions/` | 决策日志(ADR)，记录**被否决**的方案和理由 | 评审时能追到「为什么不那样做」 |
| `docs/runbooks/` | 机器可执行 runbook，写成 `if 条件 → 执行动作` | Agent 能照着自动排障，不是给人读的步骤 |
| `docs/qa/` | QA 报告 + 截图 + 终端迹线 | 自验证的物证 |

runbook 是声明式规则，不是步骤清单：

```yaml
# 【示例·换成你的】docs/runbooks/*.yml
- when: <某个可判定的线上条件>
  then: <Agent 可执行的动作>
```

### 1.4 `TODOS.md` — 冷启动交接，每条五要素

延迟工作只写一句"重构退款逻辑"= 无效。每条必须带全五要素，新 Agent 才能接手：**做什么 / 为什么 / 技术边界 / 前置依赖 / 推迟理由**。

```markdown
## <任务标题>
- 做什么：<一句话范围>
- 为什么：<触发原因>
- 技术边界：<只动哪层，不碰什么>
- 前置依赖：<需先合并什么>
- 推迟理由：<为什么现在不做>
```

---

## 2. 检查层（拦截输出）

两层阶梯：本地 hook 快拦低成本错误，远程 CI **不可绕过**拦高成本错误。配置由你按 `AGENTS.md` 生成、人 review 后提交。

### 2.1 本地门禁 `lefthook.yml`

```yaml
pre-commit:
  parallel: true
  commands:
    lint:      { run: pnpm biome check --staged }
    typecheck: { run: pnpm tsc --noEmit }
    arch:      { run: pnpm depcruise --config .dependency-cruiser.js src }
pre-push:
  commands:
    unit:     { run: pnpm test --filter=...[HEAD^1] }
    contract: { run: pnpm test tests/contracts }
```

### 2.2 远程门禁 `.github/workflows/ci.yml`

CI 必须包含这些关卡；关键正确性路径的定额/幂等测试**强制通过**才放行：

```yaml
name: ci
on: [pull_request]
jobs:
  gate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 2 }        # 需要 HEAD^1 做受影响包检测
      - run: pnpm install --frozen-lockfile
      - run: pnpm biome check .           # lint + 格式
      - run: pnpm tsc --noEmit            # 编译/类型
      - run: pnpm depcruise --config .dependency-cruiser.js src  # 分层守卫
      - run: pnpm test --filter=...[HEAD^1]   # 只跑受影响包，避免全量线性增长
      - run: pnpm test tests/golden        # 关键路径定额
      - run: pnpm test tests/contracts     # 字段契约
      - run: pnpm test tests/adversarial   # 注入对抗样本
      - run: pnpm migration-lint           # expand-and-contract 检查
```

### 2.3 分层依赖守卫 `.dependency-cruiser.js`

把 `AGENTS.md` §5 的「谁不能依赖谁」落成规则。层怎么划是你项目的事（常按变更频率和对基础设施的依赖程度划）：

```js
module.exports = {
  forbidden: [
    // 【示例·换成你的层】高层禁止依赖低层实现细节
    { name: "no-cross-layer",
      from: { path: "^src/<高层>" },
      to:   { path: "^src/<基础设施层>" } },
  ],
};
```

多语言等效工具：`import-linter`(Python) / `ArchUnit`(Java) / `go-arch-lint`(Go)。
**守卫必配负向测试**：故意写一行越层 import，断言 `depcruise` 退出码非 0，证明它真的会拦。

### 2.4 lint 红线 `eslint.config.js`

`no-console` 几乎所有项目通用（配合 §3.1 的日志封装）。**其余红线是把你项目 §4 的不变量逐条翻成 lint 规则**：

```js
export default [{
  rules: {
    "no-console": "error",                 // 通用：禁 console.log，走 logger
    // 【示例·换成你的】把项目数值/格式不变量写成规则。
    // 如 Relay「金额禁浮点」可用 no-restricted-syntax 粗拦浮点字面量：
    // "no-restricted-syntax": ["error", { selector: "...", message: "..." }],
  },
}];
// 复杂红线（如「禁在日志里出现 PII 键名」）写成自定义 eslint plugin，
// 规则实现可由 AI 生成，人 review 后挂上。
```

### 2.5 关键正确性路径：定额 + 幂等 `tests/golden/`

如果你项目有"错一分钱都不行"的路径（发钱、扣库存、配额结算……），它是机器能自动判定的最高优先级门禁：喂固定输入验精确结果，并验重放幂等。没有这类路径的项目可省。

```ts
// 【示例·换成你的关键路径】
test("定额：固定输入产出精确结果，无精度丢失", () => {
  expect(compute(fixedInput)).toEqual(exactExpected);   // 期望写死
});
test("幂等：同一 idempotency_key 重放多次只生效一次", async () => {
  const k = "req-abc";
  await Promise.all([run(k), run(k), run(k)]);           // 模拟网络抖动重放
  expect(await countEffects(k)).toBe(1);
});
```

### 2.6 字段契约 `tests/contracts/`

日志/接口字段名是服务间契约，重构不许改名。用 ajv 锁死 schema：

```ts
// 【示例·换成你的字段】required 里放你项目的契约字段名
const schema = { type: "object",
  required: ["event", "<你的关联ID>", "reason"],   // 缺字段即失败
  properties: { /* ... */ } };
test("日志符合契约", () => {
  expect(ajv.validate(schema, sampleLog)).toBe(true);
});
```

### 2.7 迁移守卫

若约束了"禁直接 DROP COLUMN，必须 expand-and-contract 两步"→ CI `migration-lint` 扫迁移 SQL，命中不可逆操作直接失败，两步之间留观察期。按你项目 DB 的红线配。

---

## 3. 反馈层（闭环自愈）

反馈必须**让 Agent 直接够得到**。本地是检查/测试输出，线上是可观测性。

### 3.1 结构化日志封装 `packages/logger/`

禁 `console.log`，统一走封装；字段名固定是契约，失败路径必带 `reason` 枚举：

```ts
// packages/logger/index.ts
import pino from "pino";
const base = pino();
export const logger = {
  // 【示例·换成你的字段】correlationId 在你项目里可能叫 request_id / trace_id
  event(name: string, ctx: { correlationId: string; reason?: string }) {
    base.info({ event: name, ...ctx });   // event / 关联ID / reason 是固定契约字段
  },
};
```

### 3.2 线上访问：只读优先，渐进放开

给 Agent 受控的线上访问才能自助排障。按信任里程碑逐层放：
**只读日志 → 只读 DB 查询 → 无副作用诊断脚本 → 有副作用操作**。

- 用云厂商只读 CLI（如配了只读 IAM 的 AWS CLI）拉日志，最小权限。
- 危险操作要参数级守卫 + 访问留痕。
- 诊断脚本放 `scripts/diagnose/`，Agent 按关联 ID 串全链路。

---

## 4. 并行 worktree 双层隔离

单条对话 = 一条并行线；多线指同一目录会互相覆盖。两层都要隔离，缺一层就打架。

- **第一层（静态代码）**：`git worktree add ../wt-<task> -b <task>`，独立目录 + 独立分支，共享 git 历史。
- **第二层（动态环境）**：端口/DB/服务进程也要隔离。装配脚本由 AI 生成、人 review 后提交，之后每个 worktree 用同一套：

```bash
# scripts/wt-setup.sh —— 并发安全的 slot 分配 + 幂等重入（DB/端口名换成你项目的）
set -euo pipefail
LOCK=.git/wt-slots.lock; REG=.git/wt-slots.reg
exec 9>"$LOCK"; flock 9                       # 目录锁防并发抢同一 slot
slot=$(comm -23 <(seq 1 20) <(sort "$REG" 2>/dev/null) | head -1)
echo "$slot" >> "$REG"
export PORT=$((3000 + slot))
export DB_NAME="app_wt_${slot}"
createdb "$DB_NAME" 2>/dev/null || true       # 幂等：已存在不报错
echo "slot=$slot PORT=$PORT DB=$DB_NAME"
```

```bash
# scripts/wt-clean.sh —— 合并后立即回收，别让陈旧 worktree 堆积
set -euo pipefail
slot="$1"
dropdb "app_wt_${slot}" 2>/dev/null || true
sed -i "/^${slot}$/d" .git/wt-slots.reg
git worktree prune
```

**纪律**：单一职责任务契约；上下文护栏（`max_iterations`、`session_timeout`）；**用完立即 `wt-clean.sh`**——陈旧 worktree 堆积是并行最大劝退点。

---

## 5. 会话自动装配 `.claude/settings.json`

让 `AGENTS.md` 每次会话开始自动喂给 Agent，不靠人工粘贴；worktree 会话顺带 provision 动态环境：

```json
{
  "hooks": {
    "SessionStart": [
      { "hooks": [{ "type": "command", "command": "cat AGENTS.md" }] },
      { "hooks": [{ "type": "command", "command": "test -f .git/wt-slots.reg && bash scripts/wt-setup.sh || true" }] }
    ]
  }
}
```

---

## 6. 搭完自检

逐条确认 harness 三件套都落地且**可强制**：

- [ ] `AGENTS.md` 七节齐全，§4 每条红线都标了执行工具，`CLAUDE.md` 软链接已建
- [ ] `docs/` 四目录建好，runbook 是 `if→then` 声明式，`TODOS.md` 用五要素模板
- [ ] `lefthook.yml` + `ci.yml` 两层门禁跑通，CI 不可绕过
- [ ] 分层守卫有配置**且有负向测试**证明会拦
- [ ] `no-console` + 项目专属 lint 红线生效
- [ ] 关键路径有定额 + 幂等测试；字段有 ajv 契约测试；迁移有 migration-lint（按项目是否需要）
- [ ] 结构化日志封装就位，`console.log` 被 lint 拦住
- [ ] 线上只读访问链路可用（Agent 能自己拉日志）
- [ ] `wt-setup.sh`/`wt-clean.sh` 双层隔离可用，SessionStart hook 自动喂 `AGENTS.md`

> 工具是可平替的实现层（gstack 参考实现：`pnpm` / `biome` / `lefthook` / `dependency-cruiser` / `pino` / `ajv`）。换栈时替换等效工具，但文档四目录、两层门禁、字段契约、双层 worktree 隔离这些**结构不变**。
