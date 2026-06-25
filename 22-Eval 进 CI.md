> AI 驱动开发方法论 ｜ 第七部分 · 对 AI 项目做 Eval
> 目录见 [README](README.md)

# Eval 进 CI

如果把评测（Eval）仅仅当成开发人员想起来才在本地随手跑跑的脚本，那这套防退化网迟早会沦为形同虚设的摆设。特别是在项目进入冲刺阶段、当 Agent 或开发人员频繁紧急调整提示词时，极易因为赶进度而省去本地完整回归的步骤，导致严重的逻辑倒退悄然上线。

**凡触碰提示词（Prompts）、大模型版本选择、向量数据库检索参数的 PR，必须在 CI 管道中自动化触发 Eval 回归。对于发钱等关键风险等级的路径，核心样本的通过率必须为 100% 绝对拦截；非核心路径可采用卡分值（如平均分 > 85）并设定合理抖动容差。**

## 触发条件与资源裁剪配置

在 CI 环境中运行大模型评测面临着两个非常现实的技术瓶颈：**高昂的大模型调用 API 账单**与**缓慢的网络 I/O 耗时**。如果一个 PR 仅仅修改了前端的 CSS 样式或者辅助工具函数，却要硬跑一遍完整的 LLM 评测，不仅是对资金的浪费，更会拉长开发迭代的时间。

因此，CI 管道必须配置精确的文件级依赖感知触发器。

### 1. GitHub Actions 中的触发路径定义

```yaml
# .github/workflows/llm-eval.yml
name: LLM Prompt Evaluation Gate

on:
  pull_request:
    paths:
      # 仅当提示词文件、模型服务配置文件或底层检索服务代码变更时才激活
      - 'src/prompts/**'
      - 'src/common/llm-config.json'
      - 'src/vector-db/**'
      - 'src/payment/eval/**'

jobs:
  run-eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Environment
        uses: actions/setup-node@v4
        with:
          node-version: 20
          
      - name: Install Dependencies
        run: npm ci
        
      - name: Execute Eval Suite
        # 挂载受控的只读 LLM 评测 API 密钥，一旦失败阻断 PR 合并
        env:
          OPENAI_API_KEY: ${{ secrets.PROD_EVAL_OPENAI_API_KEY }}
        run: npm run eval:ci
```

### 2. 差异化拦截阈值策略

在评测运行后，针对不同的业务风险等级，我们制定了差异化的通过线：

* **发钱/安全路径（Stripe 结算决策、对抗注入防护）**：核心样本通过率必须为 `100%`，不允许任何波动。
* **起草/分类路径（回复润色、情感倾向归类）**：允许保留一定的容差（如 `Total Score > 90/100` 或整体通过率 `>= 95%`），以此来吸收因大模型非确定性产生的边际打分漂移，防止无意义的假警报阻塞正常合并。

## 主线演练：CI 拦截提示词退化输出

在一次针对 Relay 客服分类逻辑的优化 PR 中，开发人员修改了 `src/prompts/classifier.txt` 中的分类规则。以下是 CI 系统中 Eval 运行后抛出报错的 stdout 迹线：

```bash
$ npm run eval:ci

> relay-eval@1.0.0 eval:ci
> node src/payment/eval/run-ci-pipeline.js

[INFO] Diff analyzer detected change in: src/prompts/classifier.txt
[INFO] Triggering Selective Eval Runner...
[INFO] Executing [Intent Classifier Eval] with 120 golden fixtures...

[TEST_RUN] Case #45: "I paid twice for order 1928, return my money!"
           - Expected Category: REFUND_REQUEST
           - Actual Category: COMPLAINT
           - Status: FAILED (AssertionError: Expected REFUND_REQUEST but got COMPLAINT)

[TEST_RUN] Case #89: "My credit card was stolen, cancel transactions!"
           - Expected Category: SEC_FRAUD
           - Actual Category: COMPLAINT
           - Status: FAILED (AssertionError: Expected SEC_FRAUD but got COMPLAINT)

[SUMMARY] Intent Classifier Eval Finished.
          - Total Cases: 120
          - Passed: 114
          - Failed: 6
          - Pass Rate: 95.0%
          - Target Threshold: 98.0%

[ERROR] CI Gate Blocked: Pass rate (95.0%) is below the security threshold of 98.0%.
[ERROR] Please check detailed report: /workspace/relay/eval-report.json
Process finished with exit code 1
```

## 调试挫败细节：吞噬研发资金的 CI 账单黑洞

在将 Eval 接进大仓的第二周，由于没有配置 `paths` 触发过滤器，每一次普通的 Git 提交都会在 CI 机器上拉起全部 300 个 Eval 用例（并且每个用例为了保证稳定性会跑 3 次取平均分）。

不到三天时间，团队的 OpenAI 账户余额就因为高强度的并发 CI 测试被刷掉了近 2000 美元，甚至因为并发限制（Rate Limit）导致正常的本地联调也陷入了停摆。

该开销数据促使团队重新调整了 CI 评测流水线的触发机制：
1. **加上精准触发器**：利用上文展示的 `paths` 限制，非提示词修改决不触发大模型调用。
2. **测试数据子集剪枝（Pruning）**：通过脚本计算 PR 改动的影响范围，仅挑选和被改动 prompt 直接关联的子模块（Sub-Eval）运行，成功将单次 CI 评测成本从 6.5 美元压低到 0.15 美元，时间从 12 分钟缩短至 40 秒，彻底消除了“评测阻碍开发”的负面效应。

## 典型反模式

* **全量触发无过滤**：不论修改代码的哪个边角，CI 都会执着地去跑完全量大模型回归，不仅耗尽了月度研发费用，还导致 CI 队列频繁积压。
* **无物理密钥自动挂起**：在外部开发者提交 PR 时，CI 因为拉取不到仓库的 secrets 导致大模型接口调用直接抛出 Unhandled Exception 崩溃，从而阻塞了正常的开源协作贡献流程。
* **一刀切的 100% 卡分**：在主观润色和分类等高熵任务上，固执地将通过率阈值锁死在 100%，导致大模型由于微小的语义浮动频繁将构建任务判定为失败，迫使团队最后直接关闭了 CI 门禁。

## 本章要点

* 大模型评测进 CI 必须锁定提示词、模型配置等“会改变 AI 行为的输入”作为触发门禁的判定前置。
* 核心支付防线阈值必须为绝对 100% 拦截，一般性分类或回复质量可配置百分比容差以吸收概率波动。
* CI 中的 Eval 任务必须设计密钥降级兜底逻辑，且通过选择性路径过滤（Path Filter）避免研发账单失控。

> 研发期在 CI 门禁拦截了本地可见的逻辑退化。然而，大模型上线后，真实用户的多变话术、大模型供应商在远端悄悄重构的模型底座，依然会在运行时引入漂移。我们将在第 23 章 `在线评测与成本治理` 中讨论。