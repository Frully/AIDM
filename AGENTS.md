# AGENTS.md

This is the Agent constitution for this repository — the single source of truth any coding Agent reads each session. `CLAUDE.md` is a symlink to this file, so Claude Code and other Agents read the same constitution with zero duplicate upkeep. Edit `AGENTS.md`, never the symlink.

## What this project is

**AI-Driven Development Methodology (AIDM)** — a tutorial/documentation project that teaches how to drive a *complete* real-world project from idea to production using AI coding Agents. Audience: people who already have some product and engineering grounding and want to level up their AI-assisted development. The deliverable is **prose and diagrams, not application code** — there is no app to build here, only a body of teaching material.

The working outline lives in [OUTLINE.md](OUTLINE.md) — that is the authoritative structure (序章 + 6 部分 18 章 + 附录). Read it before proposing structural changes.

## 语言：一律简体中文（硬性约束）

本仓库的**所有产出都用简体中文书写** —— 教程正文、章节标题、图表文字、示例说明，以及在本仓库工作时面向用户的对话回复，全部使用简体中文。技术专有名词（如 `AGENTS.md`、TypeScript、CI）可保留英文原文，但叙述、解释、过渡语句不得用英文。不要写中英混排的整句英文段落。术语保持与下文既有方法论词汇一致。

## 示例规范：统一用 Relay

所有具体示例一律用 **Relay** — 虚构的 AI 客户支持自动化 SaaS（多渠道对话接入 → LLM 分类/路由 + 起草回复 + 调工具如查 KB/订单/发退款 → 低风险自动回、高风险转人工审批 → 按已解决对话计费）。纯教学虚构，与任何真实业务无关。不要在没有充分理由的情况下发明新的主线示例。

## The methodology this project teaches

Central thesis: **Harness = 文档 + 检查 + 反馈.** Reliability comes from the scaffold around the model, not a cleverer prompt. The lifecycle is a per-task inner loop **Plan → Implement → Verify** plus a cross-iteration **Reflect** outer loop that writes lessons back into the harness. (Adapted from gstack's Think→Plan→Build→Review→Test→Ship→Reflect: Think folds into Plan, Build = Implement, Review+Test+Ship = Verify, Reflect is pulled out as the outer loop; Ship is the edge from Verify into production, not its own node.) The cross-cutting pillars below span multiple phases and are **not** lifecycle stages. Reserve **检查** for the automated-gate pillar and **验证** for the lifecycle phase — never conflate the two words. Describe each in the abstract first (portable principle), then optionally show the gstack reference implementation.

1. **`AGENTS.md` as the Agent constitution.** Hard, machine-enforceable constraints in one file the Agent reads every session: invariants, layer rules, red lines. Constraints beat reminders.
2. **A design system as single source of truth** (`DESIGN.md`). All visual/UX decisions in one doc; read before any UI work, flag drift in QA.
3. **Layered architecture with *enforced* dependency rules.** Define layers, enforce direction with a tool in CI/hooks. An Agent can't cross a layer if the build blocks it.
4. **Observability written *with* the code.** Structured logging over `console.log`, correlation IDs, a log-hygiene red line (never log PII), alert-event field names as a contract, diagnostic scripts as triage entry. Feedback must be reachable by the Agent (e.g. pull prod logs via CLI).
5. **Spec → Design → Implementation as a document flow**, plus task decomposition + artifact traceability so Agents pick up scoped work cold.
6. **Phase-appropriate routing + human-in-the-loop.** Different phases call different "hats"; calibrate how much autonomy by risk; reversibility is the precondition for letting an Agent drive ops.
7. **Author and review are separate passes.** Never self-approve in the same context.
8. **Worktree isolation for parallel Agents** — static code via worktree, dynamic env (ports/db/services) via scripts + hooks.
9. **`TODOS.md` as cold-pickup memory** + documentation as compound interest that flows back into the harness.
10. **Golden-number + idempotency tests for critical paths.** Money paths get exact-value fixtures and idempotency contracts; Agent self-verification (e2e, security/injection adversarial tests) + business acceptance close the loop. (Out of scope: testing the product's own LLM features — Eval suite design, judge calibration, accuracy tuning. That is AI-product engineering, not the development methodology this book teaches.)

## Repository status & conventions

This repo is **greenfield** — no toolchain, build, or test setup exists yet. Do not invent build/lint/test commands; there are none until a static-site generator or doc tool is chosen. If a toolchain is added later, record its real commands here.

- **Before writing or revising any chapter prose, read [STYLE.md](STYLE.md)** — it is the binding writing Spec (Diátaxis genre = explanation, the fixed per-chapter template, voice rules, terminology, banned phrases). Chapters that don't follow the template get fixed, not merged.
- Content is Markdown. Keep each chapter/section self-contained and skimmable; favor the same dense, example-driven, constraint-first voice the methodology itself preaches.
- Every chapter is two layers: **portable principle first, then gstack reference implementation** — keep the tool coupling explicit.
- Before adding a new concrete example, ask: does it belong to Relay? If not, fold it into Relay or drop it.
