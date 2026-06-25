> AI 驱动开发方法论 ｜ 第八部分 · 评审、验证与交付（Review + Test + Ship）
> 目录见 [README](README.md)

# Agent 自验证与业务验收

代码在静态层面通过了严格的 Review 与安全审计，这保障了其结构与质量的纯净。然而，静态检查的绿灯并不能等同于用户视角的“行为正确性”。尤其在复杂的 Web 系统中，API 接口类型的匹配无法拦截前端表单渲染错误、交互死锁等只能由“人眼观察”才能捕获的界面逻辑缺陷。

**未经过 Agent 亲自模拟真实用户行为进行浏览器端到端自验证（Self-Verification）的产出，决不允许声明为完成（Definition of Done）。同时，AI 产品的最终交付必须通过真实业务 KPI（如解决率、投诉回流率）的周期性验收，技术指标的全绿不等于业务有效。**

## 完成定义（DoD）中的自验证铁律

为防止 Agent 投机取巧，项目的“完成定义”必须强制嵌入端到端自验证流程：

```
+--------------------------------------------------------+
|                 DoD (完成定义) 检查清单                  |
| 1. [x] 静态编译与 Linter 规则无警告                     |
| 2. [x] 自动化单元测试与集成测试 100% 通过                |
| 3. [x] 核心 LLM 评测套件无退化                          |
| 4. [x] 独立上下文代码评审与安全 CSO 审计通过             |
| 5. [x] Agent 运行 Playwright 模拟用户进行端到端跑通并留存截图 |
+--------------------------------------------------------+
```

Agent 不能只给出“我修改了文件”的口头声明，必须呈递物理级的证据——包含浏览器操作轨迹的终端迹线与运行截图。

## 端到端自动化运行机制

在自验证阶段，Agent 需要调用自动化测试框架（如 `Playwright` 或 `Puppeteer`）真实拉起无头（Headless）或有头浏览器，在本地沙箱数据库装配就绪后（参见第六部分），模拟客户进行真实点击与文本输入，以此检验整个链路的顺畅度。

## 主线演练：Playwright 端到端自验证迹线

在 Relay 中，为了自验证退款工单详情页的正常显示，Agent 编写并执行了端到端测试。以下是其自验证产生的终端执行及截图保存迹线：

```bash
$ npx playwright test src/e2e/refund-workflow.spec.ts

[INFO] Starting Playwright browser instance... (Chromium)
[INFO] Navigating to: http://localhost:3001/agent/refund-queue
[INFO] Waiting for element "#queue-table" to render...

[ACTION] Selector "#queue-table tr[data-id='ref_98274a']" clicked.
[INFO] Navigating to detail page for refund: ref_98274a
[INFO] Waiting for element ".refund-amount" ...

[VERIFICATION] Expected element ".refund-amount" to display "$1,200.00"
               Actual element value: "$1,200.00"
               Status: PASSED

[VERIFICATION] Expected element ".decision-badge" to contain text "Under Review"
               Actual element value: "Under Review"
               Status: PASSED

[INFO] Capture page state screenshot. 
       Saving artifact: /workspace/relay/artifacts/screenshots/e2e_pass_ref_98274a.png

[SUMMARY] 1 spec passed. End-to-end self-verification succeeded.
```

## 调试挫败细节：消失的前端“元”与“分”之差

在重构 Relay 客服后台审批表单时，Agent 修改了后端的退款 API 结构，将原本返回的浮点数金额字段统一规范化为了数据库常用的“整数分（Cents）”。这一变动不仅让所有的后端 API 单元测试跑通，而且类型系统也顺畅对齐。

然而，当 Agent 在本地拉起 Playwright 进行端到端自验证时，截图对比工具报出了红色错误。

原来，由于前端模板中漏掉了针对新分字段的 `/100` 格式化过滤器，导致原本一笔 1200 元的退款请求，在坐席审核大屏上被赫然渲染显示为了 `120000.00 元`。

任何普通的后端逻辑检查都绝对抓不住这个显示层面的漏洞。如果直接交付上线，将会引发客服人员的巨大恐慌，甚至导致资金的误批。正是因为 DoD 中强加了“Agent 必须拉起真实浏览器并核对截图”这一物理红线，该缺陷在本地交付前最后一刻被强行按下，并由 Agent 自动追加了 `/100` 格式化器予以修复。

## 典型反模式

* **测试通过即完成**：CI 跑通后直接宣布“完工”，不屑于进行任何真实的端到端行为演练，导致界面白屏、CSS 漂移等显性 bug 一路绿灯滑向生产。
* **自验证缺少物证**：放任 Agent 给出“我已在本地用浏览器手动点过了”的模糊描述，缺少运行截图、网络调用 HAR 归档等不可篡改的交付物证。
* **只重代码不重业务**：上线后只关注服务“活着”，而对于“自动退款成功率从 80% 跌到 10%”的业务指标塌陷视而不见，使技术交付沦为无效交付。

## 本章要点

* 检查通过不等于行为正确。必须在完成定义（DoD）中强制约束 Agent 执行基于真实浏览器的端到端自验证。
* 自验证过程必须输出不可篡改的终端迹线与渲染截图，作为放手让 AI 独立工作的可信凭证。
* 交付的最终节点是业务 KPI（如自动解决率）的达标，以此将线上反馈重新喂回在线评测与 Eval 数据集。

> 自验证和业务指标对齐闭环了交付的最后一米。接下来，当系统真正要从本地 Worktree 部署推向生产环境那一刻，我们该如何安全地防卫数据库迁移与版本灰度过程？我们将在第 26 章 `上线与交付` 中探讨可逆性在发布阶段的设计。