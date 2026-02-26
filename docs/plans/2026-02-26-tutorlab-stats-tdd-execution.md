# TutorLab（统计学自适应教学插件）TDD Execution Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal（一句话）**：在 `openclaw` 仓库里实现一个扩展 `extensions/tutorlab`，提供工具 `tutorlab`（可选注册 `/tutor` 命令），把教材“编译”为 12 周统计学课程包，并在运行时基于学习者理解程度动态调整“讲解/练习/迁移/口试/回溯/间隔复习”，以“迁移能力（A/B/C/D 场景）”作为掌握标准。

**Design Reference（先读这份高层设计再动手）**：

- `docs/plans/2026-02-26-tutorlab-stats.md`

**Tech Stack**：

- TypeScript (ESM)
- TypeBox + Ajv（所有 LLM 输出 JSON-only + schema 校验）
- Vitest（`vitest.extensions.config.ts`）

---

## 0) 施工前约束（必须遵守，否则“效果好”会崩）

### 0.1 课程与教学约束（产品级）

1. **掌握=迁移**：任何概念的“掌握”必须在至少 2 个新语境（A/B/C/D）通过，并能口试解释：适用边界、关键假设、假设破坏怎么办。
2. **认知负荷梯度（Cleanroom → Trap Ramp）**：
   - 新概念第一道 **in-domain** 练习必须是 **cleanroom**（`dataTrapTags` 为空/缺省；数据也应 clean 或无需数据）。
   - 只有当 probe/基础题通过（或 `masteryOverall` 达阈值）才逐步注入陷阱与对抗（Simpson/MNAR/异方差/离群/自相关/混杂等）。
   - 题库约束从“每题必带陷阱”改为“**每个概念的题集覆盖 cleanroom→trap ramp**”，避免新手第一题被陷阱压垮。
3. **防死锁**：连续失败 ≥3 必须触发 **Backtracking** 回溯先修（沿概念 DAG 向前），而不是原地无限对抗。
4. **间隔复习硬需求**：成功不等于永久掌握；成功后必须排入 spaced repetition，未来几天跨域复测/口试。
5. **Graph RAG**：运行时永远只注入当前概念的局部邻域（`V_i` + `V_pre` + `V_post` + module summary；必要时 2-hop），禁止把整本书/整张图塞进上下文。
6. **数据—题目—Rubric 一致性**：凡依赖合成数据的题，必须：
   - 先 `data-gen` 生成 CSV + summary stats（ground truth）
   - 再把 summary 注入 LLM 生成 `question/expectedKeyPoints/rubric`
   - 评分时同样注入 summary 作为事实锚点
7. **状态不膨胀**：
   - `evidence[]` 只能做 ring buffer（保留最近 5~10 条）
   - 全量交互写入 append-only `telemetry.jsonl`（不进核心 state JSON）

### 0.2 工程约束（仓库级）

- 插件只新增其 runtime deps 到 `extensions/tutorlab/package.json` 的 `dependencies`；不要把插件 deps加到根 `package.json`（除非 core 需要）。
- 工具 schema 避免 `Type.Union` / `anyOf` / `oneOf` / `allOf`（遵循仓库 guardrails）。
- 运行时需要兼容“源码目录（src）”与“构建目录（dist）”两种内部 import：采用 **src-first, dist-fallback** 动态导入（参考 `extensions/llm-task/src/llm-task-tool.ts`）。
- 测试：优先写最小单测/小集成测，LLM 一律 mock（`vi.mock("../../../src/agents/pi-embedded-runner.js", ...)`）。

### 0.3 Definition of Done（验收口径）

1. `tutorlab help/init/import/compile/lesson next/answer/status/export/reset` 全部可用。
2. `compile all` 可断点续跑（stage 落盘），中断后继续不重复重活。
3. `lesson next + answer` 能在无真实 LLM key 的情况下通过 mocked tests。
4. `export` 生成可复现的 course-pack（graphs/syllabus/problems/diagnostics/datasets/notebooks/reports）。
5. `vitest run --config vitest.extensions.config.ts` 通过；`pnpm build` 通过（至少本扩展不破坏 build）。

### 0.4 硬兜底（为了避免“编译/评分卡死”）

> 下面这些不是“可选优化”，而是让系统在真实世界稳定运行的**最低兜底**。实现时必须写死并加测试。

1. **DAG 断环兜底（Task 9）**
   - Primary：用 LLM + 教学公理断环（你已有）。
   - Fallback A（推荐）：若 LLM 连续 2 次仍无法产出无环边集 → **把强连通分量 SCC 压缩成“共修概念簇（corequisite cluster）”** 作为单节点继续编译（运行时对簇内概念做交错教学）。把 `clusters.json` 写入 `graphs/`，并在 `reports/` 记录“为何成簇”。
   - Fallback B（最后手段）：若必须得到纯 DAG（某些算法强依赖）→ **强制删边兜底**：
     - 规则：在环内删除 `confidence` 最低的边（若无 `confidence`，先补默认值并记录；或先让 LLM 仅输出每条边的 `confidence` 再删最低）。
     - 目的：宁可“教学次优”，也不要让编译卡死。
     - 记录：把删除的边与环路证据写入 `reports/graph-cycle-break.json`，便于人工复核与后续优化断环 prompt。

2. **教材输入兜底：PDF 对公式/表格强烈不友好（Task 6）**
   - MVP：`import file` 必须把 PDF 标为 `extraction.lossy=true` 并提示用户“强烈建议提供 Markdown/纯文本版本”。
   - 可选增强（效果优先）：允许配置外部转换器（如 `marker`/MathPix/Pandoc pipeline），如果未安装则降级并给出明确指引；不要假装 `pdfjs` 抽到了完整公式。

3. **多评委/Best-of-K 的延迟与限流兜底（Task 4 / Task 15）**
   - 必须实现并发限流（简单 semaphore 即可），默认 `maxConcurrentLlmCalls`（例如 2~4）并可配置。
   - 必须实现**指数退避重试**（带 jitter）用于处理 429/超时/网络抖动：`baseDelayMs * 2^attempt + rand(0..jitterMs)`，并且有最大尝试次数与最大等待上限。
   - 必须实现降级：当 `multiJudge` 超时/429/网络错误 → 自动退化到更少评委（例如 3→1）；当 `bestOfK` 失败 → 退化到 `k=1`；所有降级写 telemetry。
   - 必须实现 time budget：每次 `answer` 设一个总预算（例如 30s），超过预算直接用已有 judge 结果聚合，避免“无限等”。

4. **数据生成兜底：不依赖 numpy 也要能做（Task 11）**
   - `data-gen` 以“可控的 deterministic RNG + 经验统计摘要”为主（Box–Muller 正态、分组噪声、相关项、缺失机制等都可手写）。
   - **优先“构造式陷阱”而不是“靠复杂分布采样碰运气”**：例如 Simpson/MNAR/异方差可以用确定性公式 + 小噪声就做出来，并用测试验证特征存在。
   - 不强制引入重依赖；如确需随机分布轮子，优先选轻量可 seed 的库并锁定版本（例如只用于 CDF/采样的单一库）。
   - **可选高级后端（效果优先）**：允许配置 `dataGen.backend=python`（若检测到 `python3` 且脚本可运行），用外部脚本生成 CSV+summary（例如 numpy/scipy）。默认仍走 TS 后端，测试也只要求 TS 后端必过。

---

## 1) 建议工作区/分支策略（执行者照做）

> 本 plan 假设在隔离工作区执行（推荐 git worktree）。不要在 main 上直接写。

**Step 1: 创建 worktree（示例）**

- Run（示例，不强制）：`git worktree add ../openclaw-tutorlab -b codex/tutorlab`

**Step 2: 安装依赖**

- Run：`pnpm install`
- Expected：安装成功，无 lockfile 冲突

**Step 3: 跑一次基线测试（避免背锅）**

- Run：`vitest run --config vitest.extensions.config.ts --reporter=dot`
- Expected：PASS（如果 FAIL，先停止并报告）

---

## 2) 总体任务清单（TodoList）

> 下面每个 Task 都按 TDD：先写 failing test → 跑 FAIL → 最小实现 → 跑 PASS → 小步提交。

- [ ] Task 1：创建 `extensions/tutorlab` 包与插件注册（tool 最小 help）
- [ ] Task 2：实现 state 持久化（sessions.json + enabled 开关 + reset）
- [ ] Task 3：实现 telemetry.jsonl + evidence ring buffer（防状态膨胀）
- [ ] Task 4：实现 LLM JSON runner（best-of-k + 多评委聚合 + Ajv 校验 + 重试策略）
- [ ] Task 5：实现 `tutorlab init/status`（课程元信息 + learner model 初始化）
- [ ] Task 6：实现 `import file`（教材导入、文本抽取、chunk、索引落盘）
- [ ] Task 7：实现 compiler stage 框架（stage.json，可续跑）
- [ ] Task 8：实现分层图谱编译（module DAG + module 内 concept DAG）
- [ ] Task 9：实现 cycle 熔断与 LLM 教学公理断环
- [ ] Task 10：实现 syllabus 编译（12 周）
- [ ] Task 11：实现 `data-gen`（clean & contamination flags + summary stats）
- [ ] Task 12：实现 problem-bank 编译（Cleanroom→Trap Ramp 题集约束）
- [ ] Task 13：实现 diagnostics 编译（probe 题）
- [ ] Task 14：实现 lesson planner（Graph RAG + probe bypass + trap ramp + spaced repetition）
- [ ] Task 15：实现 grading + backtracking（连续失败回溯先修）
- [ ] Task 16：实现 export（course-pack 目录结构与最小 notebook/qmd 模板）
- [ ] Task 17（可选）：注册 `/tutor` 命令实现“纯聊天入口”（无需手打子命令）
- [ ] Task 18：补齐 e2e 小集成测（import→compile→lesson→answer→export）

---

## 3) 任务拆解（TDD 级别步骤）

> 注意：每个 Task 的 “Run test” 命令都写最窄范围，避免全仓测试太慢。

### Task 1：创建扩展包骨架 + tool 最小 help

**Files**

- Create: `extensions/tutorlab/package.json`
- Create: `extensions/tutorlab/openclaw.plugin.json`
- Create: `extensions/tutorlab/index.ts`
- Create: `extensions/tutorlab/src/tutorlab-tool.ts`
- Create: `extensions/tutorlab/src/tutorlab-tool.test.ts`

**Step 1: 写 failing test（help 能输出用法）**

`extensions/tutorlab/src/tutorlab-tool.test.ts`（最小示例）

```ts
import { describe, expect, it } from "vitest";
import type { OpenClawPluginApi, OpenClawPluginToolContext } from "../../../src/plugins/types.js";
import { createTutorlabTool } from "./tutorlab-tool.js";

function fakeApi(overrides: Partial<OpenClawPluginApi> = {}): OpenClawPluginApi {
  return {
    id: "tutorlab",
    name: "TutorLab",
    source: "test",
    config: {
      agents: { defaults: { workspace: "/tmp", model: { primary: "openai-codex/gpt-5.2" } } },
    } as any,
    pluginConfig: {},
    runtime: { version: "test" } as any,
    logger: { info() {}, warn() {}, error() {}, debug() {} },
    registerTool() {},
    registerHook() {},
    registerHttpHandler() {},
    registerChannel() {},
    registerGatewayMethod() {},
    registerCli() {},
    registerService() {},
    registerProvider() {},
    registerCommand() {},
    resolvePath: (p) => p,
    on() {},
    ...overrides,
  };
}

function fakeCtx(overrides: Partial<OpenClawPluginToolContext> = {}): OpenClawPluginToolContext {
  return {
    config: {} as any,
    workspaceDir: "/tmp",
    agentDir: "/tmp",
    agentId: "main",
    sessionKey: "main",
    sandboxed: false,
    ...overrides,
  };
}

describe("tutorlab tool", () => {
  it("prints help", async () => {
    const tool = createTutorlabTool(fakeApi(), fakeCtx());
    const res = await tool.execute("call1", { command: "help" });
    expect(res.content?.[0]?.type).toBe("text");
    expect((res.content?.[0] as any).text).toContain("/tutor 用法");
  });
});
```

**Step 2: 跑测试，确认 FAIL**

- Run：`vitest run --config vitest.extensions.config.ts extensions/tutorlab/src/tutorlab-tool.test.ts -t "prints help"`
- Expected：FAIL（找不到 `createTutorlabTool` 或输出不包含用法）

**Step 3: 最小实现 tool（只支持 command: string + help）**

`extensions/tutorlab/src/tutorlab-tool.ts`（最小可过测）

```ts
import { Type } from "@sinclair/typebox";
import type { OpenClawPluginApi, OpenClawPluginToolContext } from "../../../src/plugins/types.js";

export function createTutorlabTool(_api: OpenClawPluginApi, _ctx: OpenClawPluginToolContext) {
  return {
    name: "tutorlab",
    description:
      "TutorLab: compile a textbook into an adaptive stats course and tutor via transfer-first practice.",
    parameters: Type.Object({
      command: Type.String({
        description:
          "Subcommand string, e.g. help|init|import file ...|compile ...|lesson next|answer ...",
      }),
    }),
    async execute(_id: string, params: Record<string, unknown>) {
      const command = String(params.command ?? "").trim();
      if (!command || command === "help") {
        const text = [
          "/tutor 用法（TutorLab）",
          "  /tutor init",
          "  /tutor import file <path>",
          "  /tutor compile all",
          "  /tutor lesson next",
          "  /tutor answer <text>",
          "  /tutor status",
          "  /tutor export [dir]",
          "  /tutor reset",
        ].join("\n");
        return { content: [{ type: "text", text }] };
      }
      return { content: [{ type: "text", text: `未实现命令：${command}` }] };
    },
  };
}
```

**Step 4: 跑测试，确认 PASS**

- Run：`vitest run --config vitest.extensions.config.ts extensions/tutorlab/src/tutorlab-tool.test.ts -t "prints help"`
- Expected：PASS

**Step 5: 写插件注册与元数据**

- `extensions/tutorlab/index.ts` 参考 `extensions/llm-task/index.ts`：registerTool(createTutorlabTool(api, ctx))（注意：ToolFactory 需要 ctx，所以注册方式要与其他扩展一致，若仓库要求 ToolFactory 形式则遵循）。
- `extensions/tutorlab/openclaw.plugin.json`：声明 id/name/version/main 等（参考其他扩展）。
- `extensions/tutorlab/package.json`：设置 `"type":"module"`、依赖（TypeBox/Ajv 若用到）。

**Step 6: 提交**

- Run：`bash scripts/committer "feat(tutorlab): scaffold extension with help command" extensions/tutorlab`

---

### Task 2：state 持久化（sessions/<sha(sessionKey)>.json）+ enabled 开关 + reset

**Files**

- Modify: `extensions/tutorlab/src/tutorlab-tool.ts`
- Create: `extensions/tutorlab/src/state.ts`
- Create: `extensions/tutorlab/src/state.test.ts`

**Step 1: failing test（/tutor init 写入 state；/tutor reset 清空）**

`extensions/tutorlab/src/state.test.ts`（示例）

```ts
import crypto from "node:crypto";
import fs from "node:fs/promises";
import os from "node:os";
import path from "node:path";
import { describe, expect, it } from "vitest";
import { loadTutorlabState, saveTutorlabState, tutorlabStatePath } from "./state.js";

function sha256Hex(input: string): string {
  return crypto.createHash("sha256").update(input).digest("hex");
}

describe("tutorlab state", () => {
  it("persists per sessionKey under agentDir", async () => {
    const root = await fs.mkdtemp(path.join(os.tmpdir(), "openclaw-tutorlab-state-"));
    const sessionKey = "s1";
    const p = tutorlabStatePath(root, sessionKey);
    expect(p).toContain(path.join(root, "tutorlab", "sessions", `${sha256Hex(sessionKey)}.json`));

    const s0 = await loadTutorlabState(root, sessionKey);
    expect(s0.enabled).toBe(false);

    s0.enabled = true;
    await saveTutorlabState(root, sessionKey, s0);
    const raw = await fs.readFile(p, "utf8");
    const parsed = JSON.parse(raw) as any;
    expect(parsed.enabled).toBe(true);
  });
});
```

**Step 2: 跑测试确认 FAIL**

- Run：`vitest run --config vitest.extensions.config.ts extensions/tutorlab/src/state.test.ts`
- Expected：FAIL（缺少 state 模块）

**Step 3: 最小实现 state.ts（版本化结构 + 路径规则）**

- `TutorlabStateV1`（只先放 `version/updatedAt/enabled/course/learner` 的最小字段，后续 Task 逐步扩展）
- `tutorlabBaseDir(agentDir)` → `<agentDir>/tutorlab`
- `tutorlabStatePath(agentDir, sessionKey)` → `<baseDir>/sessions/<sha>.json`
- `loadTutorlabState`：不存在则返回默认 state
- `saveTutorlabState`：mkdirp + 原子写（先写 tmp 再 rename）

**Step 4: 跑测试 PASS**

- Run：同上
- Expected：PASS

**Step 5: 把 state 接到 tool 命令**

- 在 `tutorlab-tool.ts` 里：
  - 解析 ctx：`agentDir/sessionKey`，缺失时报错（或退化到 `os.homedir()` 下默认目录，按设计文档）
  - 实现 `init/status/reset/on/off` 子命令最小版

**Step 6: 增加 tool 测试（init/status/reset）**

- Modify: `extensions/tutorlab/src/tutorlab-tool.test.ts`：新增一条测试：
  - `init` 后 `status` 输出包含 weeks/dailyMinutes
  - `reset` 后回到 disabled/default

**Step 7: 提交**

- Run：`bash scripts/committer "feat(tutorlab): persist state per session and add init/status/reset" extensions/tutorlab`

---

### Task 3：telemetry.jsonl + evidence ring buffer（防状态膨胀）

**Files**

- Modify: `extensions/tutorlab/src/state.ts`
- Create: `extensions/tutorlab/src/telemetry.ts`
- Create: `extensions/tutorlab/src/telemetry.test.ts`

**Step 1: failing test（写入 jsonl；evidence 只保留 N 条）**

`extensions/tutorlab/src/telemetry.test.ts`

```ts
import fs from "node:fs/promises";
import os from "node:os";
import path from "node:path";
import { describe, expect, it } from "vitest";
import { appendTelemetryEvent, telemetryPath } from "./telemetry.js";

describe("tutorlab telemetry", () => {
  it("appends JSONL events", async () => {
    const root = await fs.mkdtemp(path.join(os.tmpdir(), "openclaw-tutorlab-telemetry-"));
    const p = telemetryPath(root, "course1");
    await appendTelemetryEvent(root, "course1", { at: 1, kind: "test", payload: { ok: true } });
    await appendTelemetryEvent(root, "course1", { at: 2, kind: "test", payload: { ok: false } });
    const raw = await fs.readFile(p, "utf8");
    const lines = raw.trim().split("\n");
    expect(lines).toHaveLength(2);
    expect(JSON.parse(lines[0] as string)).toMatchObject({ at: 1, kind: "test" });
  });
});
```

**Step 2: 最小实现 telemetry.ts**

- `telemetryPath(agentDir, courseId)` → `<agentDir>/tutorlab/course-packs/<courseId>/telemetry.jsonl`
- `appendTelemetryEvent`：mkdirp + appendFile（每行 JSON.stringify + "\n"）

**Step 3: evidence ring buffer**

- 在 `state.ts` 提供 helper：`pushRing<T>(arr, item, limit)`；默认 limit=10
- 修改任何写 evidence 的地方必须用 ring buffer

**Step 4: 提交**

- Run：`bash scripts/committer "feat(tutorlab): add telemetry JSONL and ring-buffer evidence" extensions/tutorlab`

---

### Task 4：LLM JSON runner（best-of-k + 多评委 + Ajv）

> 这一块是“效果好”的核心。所有后续编译/出题/评分都依赖它。

**Files**

- Create: `extensions/tutorlab/src/llm-json.ts`
- Create: `extensions/tutorlab/src/llm-json.test.ts`

**Step 1: failing test（会调用 runEmbeddedPiAgent；能 strip fences；能 Ajv 校验失败报错）**

`extensions/tutorlab/src/llm-json.test.ts`

```ts
import { describe, expect, it, vi } from "vitest";
import Ajv from "ajv";
import { runJsonTask } from "./llm-json.js";

vi.mock("../../../src/agents/pi-embedded-runner.js", () => {
  return {
    runEmbeddedPiAgent: vi.fn(async () => ({
      payloads: [{ text: '{"ok":true,"n":1}' }],
    })),
  };
});

describe("tutorlab llm-json", () => {
  it("returns parsed JSON and validates schema", async () => {
    const schema = {
      type: "object",
      properties: { ok: { type: "boolean" }, n: { type: "number" } },
      required: ["ok", "n"],
      additionalProperties: false,
    };
    const res = await runJsonTask({
      prompt: "Return {ok:true,n:1}",
      input: { x: 1 },
      provider: "openai-codex",
      model: "gpt-5.2",
      timeoutMs: 1000,
      schema,
    });
    expect(res).toMatchObject({ ok: true, n: 1 });
  });
});
```

**Step 2: 最小实现 llm-json.ts**

- 参考 `extensions/llm-task/src/llm-task-tool.ts`：
  - src-first/dist-fallback 加载 `runEmbeddedPiAgent`
  - `stripCodeFences`
  - `collectText`
  - system prompt：JSON-only、no commentary、no tools
  - Ajv 校验：`new Ajv({ allErrors:true, strict:false })`
  - 错误信息要可读（instancePath + message）

**Step 3: best-of-k 与多评委**

- `runJsonTaskBestOfK({ k, ... })`：并行跑 k 次，选“schema 最干净且字段完整”的那个；如果都通过 schema，再选 `score` 最大（若 schema 有 score）。
- `runJsonMultiJudge({ judges: m, aggregate: "median"|"mean"|"trimmedMean" })`：
  - 每个 judge 输出 `GradeResult`（后续 Task 15 会定义）并 Ajv 校验
  - aggregate 只聚合分数与 nextAction，保留每个 judge 的 analysis 供 debug（写 telemetry，不写 state）

**Step 3b: 并发限流 + 降级兜底（对应 0.4）**

- 增加简单 semaphore（或 p-limit 风格实现），限制同时进行的 LLM 调用数（可配置）。
- 对 `bestOfK/multiJudge` 增加 timeout + allSettled 聚合：
  - 有足够结果就先聚合返回（遵守总 time budget）
  - 失败/超时/429 自动降级为更少的评委/更小的 k
- 所有降级与异常必须写入 telemetry（便于后续调参与定位）。
- 增加 `withRetry()`：对可重试错误（429/ETIMEDOUT/ECONNRESET 等）做指数退避重试（带 jitter），并受 `timeoutMs` 与“总 time budget”双重约束。

**Step 4: 提交**

- Run：`bash scripts/committer "feat(tutorlab): add schema-validated llm JSON runner" extensions/tutorlab`

---

### Task 5：实现 init/status（课程元信息 + learner model 初始化）

**Files**

- Modify: `extensions/tutorlab/src/tutorlab-tool.ts`
- Modify: `extensions/tutorlab/src/state.ts`
- Modify: `extensions/tutorlab/src/tutorlab-tool.test.ts`

**约束**

- 默认 `weeks=12`，`dailyMinutes=180`，domains 默认 `["A","B","C","D"]`
- `learner.concepts` 为空时允许运行（cold start）

**TDD**

- 新增 tool test：`init` 后 `status` 输出必须包含：
  - 周数、每日分钟
  - domains 列表
  - 当前 queue 长度

**提交**

- Run：`bash scripts/committer "feat(tutorlab): init/status with learner defaults" extensions/tutorlab`

---

### Task 6：import file（教材导入→文本抽取→chunk→索引落盘）

**Files**

- Create: `extensions/tutorlab/src/importer.ts`
- Create: `extensions/tutorlab/src/importer.test.ts`
- Modify: `extensions/tutorlab/src/tutorlab-tool.ts`

**约束**

- MVP 先支持：`.txt` `.md` `.pdf(可提取文本)`；不可提取的 PDF 要给出明确错误与建议（例如先 OCR）。
- **新增（对应 0.4）：PDF 必须标记为 lossy**。对公式/表格密集教材，`pdfjs` 抽取往往会丢失上下标/符号；`import` 必须提示用户“优先提供 Markdown/文本版”，并在 `textbookIndex` 里记录 `extraction.lossy=true`（后续编译可选择更保守的 chunk/概念抽取策略）。
- 抽取后落盘：
  - `<baseDir>/textbooks/<sha256(file)>/raw.txt`
  - `<baseDir>/textbooks/index.json`（记录原始路径的 hash、sha256、kind）
  - `<baseDir>/textbooks/<sha>/chunks.jsonl`（每行一个 chunk，含 offset/page/heading）

**TDD（先做 txt/md）**

- failing test：写一个临时 txt，import 后 index.json 有条目，chunks.jsonl 至少 3 行
- pdf 抽取可以先 `skip`（后续再补）

**提交**

- Run：`bash scripts/committer "feat(tutorlab): import text files and chunk into textbook index" extensions/tutorlab`

---

### Task 7：compiler stage 框架（stage.json，可断点续跑）

**Files**

- Create: `extensions/tutorlab/src/compiler/stage.ts`
- Create: `extensions/tutorlab/src/compiler/stage.test.ts`
- Modify: `extensions/tutorlab/src/tutorlab-tool.ts`

**约束**

- `compile` 必须支持子阶段：`graph|syllabus|data|bank|diagnostics|all`
- 每阶段写 `<course-pack>/stage.json`：记录每阶段状态 `pending|done|error` + 输出路径
- 续跑时跳过 `done` 阶段，失败阶段允许 `--force` 重跑（MVP：先不做 flags，做 `compile <stage>` 即可）

**TDD**

- test：写一个 stage 状态，重载后仍然一致；标记 done 后再次 run 不重复（用 mock 计数）

**提交**

- Run：`bash scripts/committer "feat(tutorlab): add resumable compiler stage runner" extensions/tutorlab`

---

### Task 8：分层图谱编译（module DAG + module 内 concept DAG）

**Files**

- Create: `extensions/tutorlab/src/compiler/graph.ts`
- Create: `extensions/tutorlab/src/compiler/graph.test.ts`

**约束**

- 先 module-level DAG：节点=章节/模块；边=先修（粗粒度）
- 再 module 内 concept DAG：节点=概念；边=先修（微观）
- **边的结构必须包含 `confidence: number (0..1)`**（供 Task 9 的兜底删边与诊断报告使用）。可选包含 `why: string`（一行理由，便于 debug）。
- 图谱落盘：
  - `graphs/module-graph.json`
  - `graphs/concept-index.json`
  - `graphs/modules/<moduleId>/concept-graph.json`

**TDD（LLM 全 mock）**

- mock `runJsonTask` 返回固定 module 列表与边
- test：输出文件存在；边含 `confidence`；DAG 无环（用 topological sort 检查）

**提交**

- Run：`bash scripts/committer "feat(tutorlab): compile hierarchical concept graphs" extensions/tutorlab`

---

### Task 9：cycle 熔断与 LLM 教学公理断环

**Files**

- Modify: `extensions/tutorlab/src/compiler/graph.ts`
- Create: `extensions/tutorlab/src/compiler/cycle-break.test.ts`

**约束**

- 检测到 cycle：提取环上节点集合，构造一个“教学公理” prompt 让 LLM 输出新的无环边集
- 不把“删最低置信度边”作为常规策略（教学上危险）；**仅允许作为 0.4 的最后兜底**，确保编译不会卡死。
- **新增（对应 0.4）：必须实现兜底**：
  - LLM 断环重试上限（例如 2 次）
  - 若仍有环：优先 SCC 压缩为“共修概念簇”继续编译（推荐）
  - 仅当强依赖纯 DAG 时才启用“最小破坏删边”兜底，并写出诊断报告文件

**TDD**

- 构造一个有环的边集，mock LLM 返回断环后的边集
- 断言：新图无环；节点集合不丢失

**提交**

- Run：`bash scripts/committer "feat(tutorlab): break DAG cycles via LLM pedagogical axioms" extensions/tutorlab`

---

### Task 10：syllabus 编译（12 周）

**Files**

- Create: `extensions/tutorlab/src/compiler/syllabus.ts`
- Create: `extensions/tutorlab/src/compiler/syllabus.test.ts`

**约束**

- 输出 `syllabus/syllabus.json`：周→天→lesson slots（概念/目标/建议题型）
- 只需要“可被 runtime 调度”的结构，不追求漂亮排版

**TDD**

- mock 输入 concept-index + module DAG
- 断言：输出含 12 周；每周 7 天（或 5 天，按设计文档默认）；每天至少 1 lesson slot

**提交**

- Run：`bash scripts/committer "feat(tutorlab): compile 12-week syllabus" extensions/tutorlab`

---

### Task 11：data-gen（clean & contamination flags + summary stats）

**Files**

- Create: `extensions/tutorlab/src/data/data-gen.ts`
- Create: `extensions/tutorlab/src/data/data-gen.test.ts`

**约束**

- 输入：`seed` + `contaminationFlags: string[]`
- 输出：`{ csvPath, summary: { mean/var/quantiles/corr/... }, trapsApplied: string[] }`
- `contaminationFlags=[]` 必须生成 cleanroom 数据（近似同方差、无刻意混杂）
- deterministic：同 seed+flags → 输出统计摘要稳定（允许极小浮动，但尽量固定）
- **新增（对应 0.4）：优先经验统计摘要**。无需依赖 numpy；用 deterministic RNG 生成样本，再计算均值/方差/分位数/相关即可。
- **新增（对应审查意见）：尽量“构造式生成”**：
  - Simpson：用 2~3 个分组的不同基线/斜率 + 不同样本量权重，确保聚合与分组趋势相反（测试验证）。
  - 异方差：噪声方差随 `x` 或分组变化（例如 `noise ~ N(0, (a+bx)^2)`），测试验证残差方差随 `x` 增长。
  - MNAR：缺失概率依赖 `y` 或潜变量（测试验证缺失与目标相关）。
- **新增：可选 python 后端**（仅当你愿意牺牲“零依赖”换更强的统计生成能力）：
  - 如果配置 `dataGen.backend=python` 且检测到 `python3`：调用 `python3 extensions/tutorlab/scripts/data_gen.py ...` 生成 CSV+summary
  - 否则自动回退到 TS 后端，并记录 telemetry

**TDD**

- test：同 seed 两次生成 summary 完全一致
- test：flags=["heteroskedastic"] 时 summary 表现出异方差信号（例如分组方差差异超过阈值）

**提交**

- Run：`bash scripts/committer "feat(tutorlab): deterministic data generator with summary stats" extensions/tutorlab`

---

### Task 12：problem-bank 编译（Cleanroom→Trap Ramp 题集约束）

**Files**

- Create: `extensions/tutorlab/src/compiler/problem-bank.ts`
- Create: `extensions/tutorlab/src/compiler/problem-bank.test.ts`

**约束**

- 每概念至少生成 4 题：
  - 1 in-domain cleanroom（`dataTrapTags` 为空/缺省）
  - 1 cross-domain（通常带陷阱）
  - 1 adversarial（必须带陷阱）
  - 1 robustness（必须带陷阱）
- 对于需要数据的题：必须先走 Task 11 生成 summary，再注入 LLM 生成题目与 rubric

**TDD**

- mock `data-gen` 与 `runJsonTask`：
  - data-gen 返回固定 summary
  - LLM 返回固定题目 JSON
- 断言：每概念题集满足 cleanroom→trap ramp 约束；落盘 JSON 可被 Ajv 校验

**提交**

- Run：`bash scripts/committer "feat(tutorlab): compile problem bank with trap-ramp constraints" extensions/tutorlab`

---

### Task 13：diagnostics（probe 题）

**Files**

- Create: `extensions/tutorlab/src/compiler/diagnostics.ts`
- Create: `extensions/tutorlab/src/compiler/diagnostics.test.ts`

**约束**

- 每概念至少 1 道 probe（短、快、判别力强）
- runtime：当 mastery 未初始化时，先 probe 再 lecture

**TDD**

- 断言：diagnostics.json 覆盖所有概念

**提交**

- Run：`bash scripts/committer "feat(tutorlab): compile probe diagnostics per concept" extensions/tutorlab`

---

### Task 14：lesson planner（Graph RAG + probe bypass + trap ramp + spaced repetition）

**Files**

- Create: `extensions/tutorlab/src/runtime/lesson.ts`
- Create: `extensions/tutorlab/src/runtime/lesson.test.ts`
- Modify: `extensions/tutorlab/src/tutorlab-tool.ts`

**约束**

- `lesson next` 选择逻辑顺序（推荐）：
  1. due 的 spaced repetition（review）
  2. backtrack 队列（若存在）
  3. 新概念：先 probe
  4. 其他：按薄弱 domain + syllabus slot 选择
- trap ramp：
  - `masteryOverall < 0.6` 或 `streakSuccess=0`：只出 cleanroom in-domain
  - 达阈值后：逐步 cross-domain/adversarial/robustness
- Graph RAG 注入：只携带局部邻域（需要时最多 2-hop）

**TDD**

- 构造一个 learner state：
  - 先无 mastery：`lesson next` 返回 probe
  - probe 通过后：下一次返回 cleanroom
  - mastery 提升后：返回 cross-domain（带 dataTrapTags）

**提交**

- Run：`bash scripts/committer "feat(tutorlab): lesson planner with probe + trap ramp + spaced repetition" extensions/tutorlab`

---

### Task 15：grading + backtracking（连续失败回溯先修）

**Files**

- Create: `extensions/tutorlab/src/runtime/grading.ts`
- Create: `extensions/tutorlab/src/runtime/grading.test.ts`
- Modify: `extensions/tutorlab/src/state.ts`

**约束**

- `GradeResult`（JSON-only）必须包含：
  - `stepByStepAnalysis: string[]`（允许结构化推理字段，但仍在 JSON 内）
  - `identifiedFlaws: Array<{ tag: string; note?: string }>`
  - `score: 0|1|2|3|4`
  - `nextAction: "advance"|"retry"|"backtrack"|"review"`
- 连续失败 ≥3：
  - 追加 `queue` 项：`kind="backtrack"`，concept=prereq（从图谱取 `V_pre`）
  - 当前概念暂停对抗题，先补先修

**TDD**

- mock 多评委评分：三次返回低分 → 断言队列出现 backtrack
- 一次高分 → 断言 masteryOverall 上升且排入 spaced repetition

**提交**

- Run：`bash scripts/committer "feat(tutorlab): grading pipeline with backtracking triggers" extensions/tutorlab`

---

### Task 16：export（course-pack 目录 + notebook/qmd 模板）

**Files**

- Create: `extensions/tutorlab/src/export/export.ts`
- Create: `extensions/tutorlab/src/export/export.test.ts`
- Modify: `extensions/tutorlab/src/tutorlab-tool.ts`

**约束**

- `export` 默认输出到 `<workspaceDir>/.tutorlab/course-packs/<courseId>/`
- 至少生成：
  - `graphs/` `syllabus/` `problems/` `diagnostics/` `datasets/` `reports/`
  - `notebooks/README.md`（指引如何打开/运行）
  - 一个最小 `.qmd` 模板（不需要渲染，只要可复现结构）

**TDD**

- 在 tmp workspaceDir 调用 export → 断言目录/文件存在

**提交**

- Run：`bash scripts/committer "feat(tutorlab): export course-pack assets" extensions/tutorlab`

---

### Task 17（可选）：注册 `/tutor` 命令（纯聊天入口）

> 目的：在聊天渠道里只输入 `/tutor 继续` 也能走 lesson/answer，而不要求用户掌握子命令结构。

**Files**

- Modify: `extensions/tutorlab/index.ts`（或新增 `src/command.ts`）

**约束**

- 命令 handler 把 `ctx.args` 转换成 tool command（例如空 args→`lesson next`；有文本→`answer <text>`）
- 注意授权/allowlist 策略（默认 requireAuth=true）

**测试**

- 写一个轻量单测（不需要完整 channel），只断言 handler 文本路由

**提交**

- Run：`bash scripts/committer "feat(tutorlab): add /tutor chat command wrapper" extensions/tutorlab`

---

### Task 18：e2e 小集成测（import→compile→lesson→answer→export）

**Files**

- Create: `extensions/tutorlab/src/tutorlab.e2e.test.ts`

**约束**

- 全流程必须在 mock LLM 下可跑
- 只用 txt 作为教材输入（避免 pdf 依赖）

**Run**

- `vitest run --config vitest.extensions.config.ts extensions/tutorlab/src/tutorlab.e2e.test.ts`

**提交**

- Run：`bash scripts/committer "test(tutorlab): add mocked end-to-end flow test" extensions/tutorlab`

---

## 4) 执行建议（给 Claude 的执行方式选择）

> 这不是实现步骤，只是建议：如果任务太多，优先 Task 1~4 跑通“最小闭环”，再逐步加编译与运行时。

**推荐批次**

- Batch 1：Task 1~4（骨架 + state + telemetry + llm-json）
- Batch 2：Task 5~7（init/import/stage）
- Batch 3：Task 8~13（graph/syllabus/data/bank/diagnostics）
- Batch 4：Task 14~16（lesson/grading/export）
- Batch 5：Task 17~18（聊天入口 + e2e）
