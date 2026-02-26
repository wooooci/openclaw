# TutorLab 开发交接（OpenClaw Worktree）

**Worktree 路径**：本文件所在仓库根目录（避免在仓库 docs 里写绝对路径）。

**本交接的目标**：在 OpenClaw 代码库中实现 `extensions/tutorlab`（工具 `tutorlab`，可选 `/tutor` 聊天命令），把教材编译为 12 周统计学课程包，并在运行时做自适应迁移训练（A/B/C/D）。

---

## 必读 3 份文件（按顺序）

1. 纲领/高层设计：`docs/plans/2026-02-26-tutorlab-stats.md`
2. TDD 执行细则：`docs/plans/2026-02-26-tutorlab-stats-tdd-execution.md`
3. 本交接：`docs/plans/2026-02-26-tutorlab-handoff.md`

---

## 关键约束（不要“优化掉”）

TDD 细则里已经写成强约束（含硬兜底），这里仅摘要：

- **掌握=迁移**：必须跨域（A/B/C/D）验真会。
- **Cleanroom → Trap Ramp**：新概念第一题必须干净题；通过后再逐步引入陷阱/对抗/鲁棒性。
- **Graph RAG**：运行时只注入局部邻域，禁止全图/全书塞上下文。
- **防死锁**：连续失败触发 backtracking 回溯先修。
- **间隔复习**：成功后排 spaced repetition。
- **数据—题目—Rubric 一致性**：必须 data-first（先 data-gen + summary，再出题/评分）。
- **稳定性硬兜底**：
  - 图谱断环：LLM 失败两次后必须 SCC/删边兜底，确保编译不挂死。
  - 并发雪崩：并发限流 + 指数退避重试 + time budget + 自动降级。
  - PDF 抽取：必须标记 lossy 并强提示用户提供 Markdown/文本或外部转换器。

---

## 开工方式（建议）

> **For Claude:** 执行实现时请使用 `superpowers:executing-plans`（或 `superpowers:subagent-driven-development`）。

建议按批次推进（TDD 文档末尾也写了）：

- Batch 1：Task 1–4（骨架 + state + telemetry + llm-json runner）
- Batch 2：Task 5–7（init/import/stage）
- Batch 3：Task 8–13（graph/syllabus/data/bank/diagnostics）
- Batch 4：Task 14–16（lesson/grading/export）
- Batch 5：Task 17–18（聊天入口 + e2e）

---

## 本地验证命令（最小）

在 worktree 根目录：

- 安装依赖：`pnpm install`
- 扩展测试：`vitest run --config vitest.extensions.config.ts --reporter=dot`
- 类型构建：`pnpm build`

---

## LLM 供应商一致性（你已说明）

你已约束“LLM 使用 Codex 供应商一致”，实现时建议：

- 统一从 `api.config.agents.defaults.model.primary` 解析 provider/model（或由插件配置覆盖）
- `bestOfK/multiJudge` 的并发与重试要在 runner 层集中实现，避免各处重复造轮子
