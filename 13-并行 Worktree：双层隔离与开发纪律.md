> AI 驱动开发方法论 ｜ 第五部分 · 并行 Worktree 开发（Build 阶段 · 并行开发）
> 目录见 [README](README.md)

# 并行 Worktree：双层隔离与开发纪律

Relay 的工程师第一次跑并行开发时，给三个 Agent 会话各分配了一个任务：会话 A 开发退款判定，会话 B 重构分类路由，会话 C 修坐席审批台的前端 Bug。三个会话指向同一个工作目录，没有做任何隔离。两分钟后，A 和 B 同时修改了 `src/router/classifier.ts`，本地工作区的文件被轮流覆盖；C 起测试服务时发现 3000 端口已被 A 占用，直接报 `EADDRINUSE`；B 写入数据库的测试记录污染了 C 的端到端验证，断言全红。三条线互相踩了半小时，最后两个 Agent 的输出被丢弃，退回单线。

这次翻车的根源不是三个任务并行本身有问题，而是缺少两样东西：隔离和纪律。

**并行开发能跑起来，依赖的是两层物理隔离——git worktree 隔静态代码、脚本隔动态环境——以及配套的运行纪律，让隔离真正不被绕过。**

## 并行能力从多开会话来

Claude Code 之类的桌面端 CLI 可以同时开多个独立会话。每个会话有自己的上下文窗口，互不共享状态，可以视作一个独立运行的虚拟工程师。开三个会话，就是三条开发线同时走。

问题是，会话之间的隔离是逻辑上的，底下共享的文件系统和本地运行栈并没有隔离。多条线同时改文件，最后写入的把之前的覆盖掉；多条线同时起服务，端口只有一个能抢到；一条线写入数据库，另一条线的查询就会拿到脏数据。这三类冲突，只要有一个没解决，并行就会产生比串行更难排查的问题。

## 第一层：git worktree 隔代码

`git worktree` 允许同一个 Git 仓库同时把多个分支检出到不同的物理目录。每个目录有自己的工作区，修改互不可见，但共享同一套底层的提交历史和对象库。

Relay 给退款分支和路由分支各开了一个 worktree：

```bash
$ git worktree add ../wt-refund-01 feat/refund-decision
$ git worktree add ../wt-routing-02 feat/classification

$ git worktree list
relay-main      [main]
wt-refund-01    [feat/refund-decision]
wt-routing-02   [feat/classification]
```

现在会话 A 的工作目录指向 `wt-refund-01`，会话 B 指向 `wt-routing-02`。两个 Agent 同时修改文件，物理上落在两个不同的目录，不会相互覆盖。`git worktree` 在注册表层面也做了互锁：同一个分支不能同时检出到两个 worktree，防止意外的并发写入。

这一层解决了"文件互踩"的问题，但仅此而已。

## 第二层：脚本隔动态环境

代码文件隔开之后，运行时的冲突还在。两个 Agent 的本地服务默认都监听 3000，数据库默认都连同一个实例。第一层隔离管不到这些。

解决方案是会话启动时自动装配环境变量：给每个 worktree 分配一个专属端口和专属数据库，写进该目录根下的 `.env` 文件。Relay 的 worktree 管理脚本在创建目录时就做了这件事：

```ini
# wt-refund-01/.env
PORT=3001
DATABASE_URL="postgresql://localhost/relay_sandbox_wt_refund_01"
REDIS_URL="redis://localhost:6379/1"

# wt-routing-02/.env
PORT=3002
DATABASE_URL="postgresql://localhost/relay_sandbox_wt_routing_02"
REDIS_URL="redis://localhost:6379/2"
```

会话 A 起服务在 3001，写数据库 `relay_sandbox_wt_refund_01`；会话 B 在 3002，写 `relay_sandbox_wt_routing_02`。两条线的运行时完全不相交。

只有两层都到位，Agent 才能真正互不干扰地在本地跑完"开发→测试→自验证"这个完整循环。漏掉任何一层，剩下的一层都只是部分缓解。

## 隔离之后，纪律才是真正的保障

Relay 第一次配好双层隔离之后，又翻了一次车，这次原因不同。

会话 A 被交代去做退款判定，做到一半发现路由分类的一个辅助函数写得不对，于是顺手改了。这个函数同时被会话 B 的分支依赖，两条线在各自分支上对同一个函数做了不同的修改。代码物理上没有冲突，但逻辑上冲突了，等到 PR 阶段才发现，合并比想象的复杂得多。

同一周，另一个会话因为某个 TypeScript 类型错误一直编译不过，没有设置迭代上限，Agent 自己反复尝试了三十多轮，既没修好，也没停下来告诉任何人，直到有人注意到账单飙升才去看日志，发现会话已经跑了将近五十分钟。

这两个问题，隔离解决不了。它们需要纪律。

### 单一任务契约

每个会话只负责一件原子任务，边界在任务分发时就要划清楚。"退款判定"是一个任务，"修路由分类的辅助函数"是另一个任务，不能混在同一个会话里，哪怕它们看起来只相差几行代码。原因很直接：Agent 的上下文窗口有限，同时装着两件事会让它在两个上下文之间漂移，最终产出无法独立验收的大片脏提交。更重要的是，原子任务对应原子 PR，评审粒度小，合并时的逻辑冲突才能控制在可处理的范围内。

### 迭代护栏

Agent 卡住的时候不会主动停，除非你告诉它"最多试多少次"。Relay 在 `.agents/config.yaml` 里加了两个硬限制：

```yaml
# .agents/config.yaml
max_iterations: 12        # 单次会话最多执行 12 轮 Code-Action 循环
session_timeout_ms: 300000 # 5 分钟超时强杀
```

`max_iterations: 12` 是轮次上限，超过就挂起，等人介入。`session_timeout_ms: 300000` 是时钟上限，不管轮次，五分钟内没完成就强杀。两个护栏都要设，因为有些循环单轮耗时极短，轮次很快就吃满；有些循环单轮很慢，轮次没吃满但墙钟时间已经失控。

加这两行之前，Relay 有一个会话在一次 lint 修复里死循环了 47 轮，账单多出来的那部分 Token 消耗单独列了一行。

### 合并后立刻清理

分支合并进主干之后，对应的 worktree 和沙箱数据库不再有用，但它们还占着磁盘、数据库连接和端口号。如果不清，下次创建新 worktree 时端口分配脚本可能碰到已被占用的号段，数据库实例也会越来越多，本地环境的"本底噪声"越来越高。

Relay 的做法是把清理固化为合并流程的最后一步。`RELAY_REFUND_01` 的 PR 合并后，立刻跑清理命令：

```bash
$ npx wt-manage clean RELAY_REFUND_01
[INFO] Initiating resource cleanup for session: RELAY_REFUND_01
[INFO] Deleting git worktree path: wt-refund-01
[INFO] Dropping local database: relay_sandbox_wt_refund_01
[INFO] Freeing socket port: 3001
[INFO] Success. Worktree and dynamic resources removed.
```

三行 `[INFO]`，对应三种资源：文件目录、数据库实例、端口号。清完之后，下一个任务可以干净地从 3001 重新开始，不会碰到"端口已被上次的 wt 占着"这类随机失败。

## 反模式

**无护栏自动重试。** 这是最烧钱的一类故障。Agent 在编译阶段卡住，没有迭代上限，就进入"改一行→编译失败→再改"的死循环。整个过程在日志里看起来"它还在努力"，但实际上已经失控，没有任何有效进展。人介入时往往已经跑了几十轮。`max_iterations` 和 `session_timeout_ms` 必须都设，且要在第一个会话开起来之前就写进配置，不是出了事再补。

**陈旧 Worktree 堆积。** 合并之后只管开下一条线，不清理上一条线留下的目录和数据库。过一两周，本地会同时存在七八个 worktree，其中多数已经合并或废弃，但端口和数据库实例还在占着。某天新建一个 worktree，脚本分配端口时发现 3001 到 3006 全被旧的占着，报错后才发现需要手动逐一排查清理，这时候人工成本已经比一开始就清理高出一个数量级。清理动作应当被编排工具触发，做到"合并即清理"，不依赖工程师记住。

## 本章要点

- 并行能力来自多开会话，但会话的逻辑隔离不等于物理隔离，需要额外的两层保障。
- 第一层：`git worktree` 让每条开发线有自己的物理目录和分支，文件修改互不覆盖。
- 第二层：启动脚本给每个 worktree 分配独立端口和数据库实例，写进各自的 `.env`，运行时完全不相交。
- 迭代护栏（`max_iterations`、`session_timeout_ms`）防止 Agent 卡死后继续烧资源。
- 合并即清理：worktree 目录、沙箱数据库、端口号，三者同时回收，让下一条线从干净状态开始。

下一章讲怎么把 worktree 创建、端口分配和数据库 provision 封装成一键式脚本，让并行开发环境的搭建变成一条命令的事。
