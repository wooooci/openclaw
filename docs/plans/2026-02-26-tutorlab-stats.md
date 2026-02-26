# TutorLab（统计学课程编译器 + 自适应迁移训练）Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** 在 OpenClaw 里提供一个插件工具 `tutorlab`（对话入口建议用 skill `/tutor ...`），把教材“编译”为 12 周（3 个月、每天约 3 小时）的统计学课程，并以“迁移能力（A/B/C/D 场景都能重新建模）”为掌握标准，按学习进度与理解程度动态调整讲解与练习；同时产出 Notebook（探索/练习）+ Quarto/RMarkdown（可复现终稿）课程包。

**Architecture (high-level):**

- **Plugin tool**：`tutorlab` 维护一个可版本化的课程与学习者状态（JSON），对外暴露 `/tutor init|import|compile|lesson|answer|status|export|reset` 等子命令。
- **Course Compiler（重 token / 分层 + 可断点续传）**：导入教材 → **模块/章节级**粗粒度 DAG → **模块内**概念级微观 DAG → 12 周大纲 → 题库与 rubric（按概念×场景 A/B/C/D）→ 生成每日/每课模板。每个阶段落盘，失败可续跑，不需要重来。
- **Tutor Workbench（重反馈 / 高频迭代）**：每次 `lesson/answer` 都执行“讲解 → 迁移题 → 口试追问 → 对抗测试 → 更新掌握度 → 重新排课”闭环。
- **Transfer-first Learner Model**：每个概念维护 `masteryOverall` + `masteryByDomain(A/B/C/D)` + `misconceptions[]` + `evidence[]`，用 OOD（分布外）压力测试判定是否“真会”。
- **Graph RAG（图谱检索注入）**：`lesson next` 时**不注入全量概念图/大纲**；只注入当前节点 `V_i`、其直接前驱集合 `{V_pre}`、直接后继集合 `{V_post}`、以及所属模块摘要（必要时加 2-hop）。把上下文控制在“局部邻域”。
- **LLM orchestration**：使用 `runEmbeddedPiAgent()` 做 JSON-only 子调用（可 best-of-k + 多评委），所有 LLM 输出必须 Ajv 校验；任何无效 JSON 直接报错并提供重试指令。评分/诊断 JSON 内允许包含结构化推理字段（见 Task 3/6），但不允许 JSON 外的额外文本。
- **Export pipeline**：把课程资产写到 `workspaceDir/.tutorlab/course-packs/<slug>/`：`graphs/`（含 `concept-index.json`）、`syllabus/`、`problems/`、`diagnostics/`、`notebooks/`、`reports/`、`datasets/`。

**Tech Stack:**

- TypeScript (ESM) + TypeBox schema
- Ajv（LLM JSON 校验）
- Vitest（扩展测试：`vitest.extensions.config.ts`）
- 可选：Quarto（仅用于用户本机渲染；插件只生成 `.qmd` 模板与数据）

---

## 设计约束（为了“只要效果好”）

1. **掌握=迁移**：任何概念都必须在至少 2 个新语境中通过（A/B/C/D），且能回答“为什么适用/不适用、假设是什么、假设破坏怎么办”的口试追问。
2. **重诊断不重陪伴**：每天 3 小时必须高密度反馈：短讲→练→口试→纠错→再练；不依赖“长期坚持的温柔陪跑”。
3. **不省 token**：默认开启 best-of-k（生成更高质量题目/讲解）+ 多评委（更稳的评分与误区归因）+ 对抗出题（主动找弱点）。
4. **防死锁**：连续失败必须触发“回溯（Backtracking）”检查先修概念，而不是在同一概念同一场景无限对抗重试。
5. **间隔复习是硬需求**：成功不等于永久掌握；成功后必须按间隔重复（spaced repetition）在未来几天做跨域口试/迁移复测。
6. **数据—题目—Rubric 一致性**：凡是依赖合成数据的题目，必须先由本地 `data-gen.ts` 生成数据实例与 summary stats，再把 summary stats 注入 LLM 生成该题的 `question/expectedKeyPoints/rubric`；禁止“LLM 先写 Rubric/答案期待 → 事后再生成数据”。
7. **认知负荷梯度（Cleanroom → Trap Ramp）**：新概念的**第一道本域练习**默认必须是“干净题”（`dataTrapTags` 为空/缺省；对应 clean 数据或无数据），先建立标准模型与直觉；只有在探针题/基础题通过（或 `mastery` 达到阈值）后，才逐步引入数据陷阱与对抗题。

---

## 命令与用户体验（第一版必须可用）

**推荐对话用 skill（用户侧）**：`/tutor ...`（实际 dispatch 到 tool `tutorlab`）

**Tool：tutorlab 子命令（MVP 必须有）**

- `help`：用法与示例
- `init`：初始化课程（目标、每天学习时长、总周数、偏好场景 A/B/C/D）
- `import file <path>`：导入教材/讲义（pdf/txt/md；先做“可提取文本”的 PDF）
- `compile`：编译课程资产（**分阶段落盘，支持续跑**）。建议子模式：`compile graph`（模块 DAG + 模块内概念 DAG），`compile syllabus`，`compile bank`，`compile rubrics`，`compile diagnostics`，`compile all`。
- `lesson next`：生成下一节课（讲解+练习+迁移题+口试追问）
- `answer <freeform>`：提交当前练习/口试回答（自动评分+误区归因+更新掌握度+给下一步）
- `status`：展示进度（剩余周数、薄弱概念、各场景迁移雷达、下一步建议）
- `export [dir]`：导出课程包（notebooks + qmd + 数据 + 题库）
- `reset`：清空当前 session

---

## 数据模型（JSON，必须可版本化）

版本化根对象（示意）：

```ts
type TutorlabStateV1 = {
  version: 1;
  updatedAt: number;
  enabled: boolean;
  course: {
    id: string; // slug
    goal: string;
    weeks: number; // default 12
    dailyMinutes: number; // default 180
    domains: Array<"A" | "B" | "C" | "D">;
    textbookIndex?: { files: Array<{ path: string; sha256: string; kind: string }> };
  };
  assets?: {
    stagePath?: string;
    moduleGraphPath?: string;
    conceptIndexPath?: string;
    moduleConceptGraphsDir?: string;
    syllabusPath?: string;
    problemBankPath?: string;
    rubricsPath?: string;
    diagnosticsPath?: string;
  };
  learner: {
    fatigueIndex?: number; // 0..1 (higher = more fatigued)
    concepts: Record<
      string,
      {
        masteryOverall: number; // 0..1
        masteryByDomain: { A: number; B: number; C: number; D: number };
        streakFail?: number;
        streakSuccess?: number;
        lastAttemptAt?: number;
        lastSuccessAt?: number;
        nextDueAt?: number; // spaced repetition next review time (ms)
        misconceptions: Array<{ tag: string; confidence: number; note?: string }>;
        evidence: Array<{
          at: number;
          concept: string;
          domain: "A" | "B" | "C" | "D";
          score: number;
          note?: string;
        }>; // ring buffer (keep last 5~10); full log → telemetry.jsonl
      }
    >;
    queue: Array<{
      kind: "learn" | "review" | "transfer" | "oral" | "probe" | "backtrack";
      concept: string;
      domain?: "A" | "B" | "C" | "D";
      dueAt?: number;
      reason?: string;
      backtrackFrom?: string;
    }>;
    current?: {
      lessonId: string;
      items: Array<{ id: string; kind: string; concept: string; domain?: string }>;
    };
  };
  historyTail: Array<{ at: number; role: "user" | "assistant"; text: string }>;
};
```

状态存储位置（与其他插件一致）：

- `baseDir = <agentDir>/tutorlab/`（agentDir 不存在则退化到 `~/.openclaw/tutorlab/`）
- `sessions/<sha256(sessionKey)>.json`
- `course-packs/<courseId>/...`（导出的课程包）

---

## Task 1：新增插件骨架（tutorlab）+ 最小可用 help

**Files:**

- Create: `extensions/tutorlab/index.ts`
- Create: `extensions/tutorlab/package.json`
- Create: `extensions/tutorlab/openclaw.plugin.json`
- Create: `extensions/tutorlab/src/tutorlab-tool.ts`
- Test: `extensions/tutorlab/src/tutorlab-tool.test.ts`

**Step 1: 写 failing test：tool help 输出**

```ts
import { describe, expect, it } from "vitest";
import { createTutorlabTool } from "./tutorlab-tool.js";

describe("tutorlab tool", () => {
  it("prints help", async () => {
    const tool = createTutorlabTool(
      {
        id: "tutorlab",
        name: "TutorLab",
        source: "test",
        config: { agents: { defaults: { workspace: "/tmp" } } },
      } as any,
      {
        workspaceDir: "/tmp",
        agentDir: "/tmp",
        agentId: "main",
        sessionKey: "s1",
        sandboxed: false,
      } as any,
    );
    const res = await tool.execute("call1", { command: "help" });
    expect((res.content?.[0] as any).text).toContain("/tutor 用法");
  });
});
```

**Step 2: 运行测试确保失败**

Run: `vitest run --config vitest.extensions.config.ts extensions/tutorlab/src/tutorlab-tool.test.ts`
Expected: FAIL（`createTutorlabTool` 不存在）

**Step 3: 最小实现：注册 tool + help**

`extensions/tutorlab/index.ts`（最小）：

```ts
import type { OpenClawPluginApi } from "openclaw/plugin-sdk";
import { createTutorlabTool } from "./src/tutorlab-tool.js";

export default function register(api: OpenClawPluginApi) {
  api.registerTool((ctx) => createTutorlabTool(api, ctx));
}
```

`extensions/tutorlab/src/tutorlab-tool.ts`（最小）：

```ts
import { Type } from "@sinclair/typebox";
import type { OpenClawPluginApi } from "openclaw/plugin-sdk";
import type { OpenClawPluginToolContext } from "../../../src/plugins/types.js";

export function createTutorlabTool(_api: OpenClawPluginApi, _ctx: OpenClawPluginToolContext) {
  return {
    name: "tutorlab",
    description: "Adaptive statistics tutoring + course compiler (transfer-first).",
    parameters: Type.Object({
      command: Type.Optional(
        Type.String({ description: "Raw subcommand string (e.g. 'help', 'init', 'lesson next')." }),
      ),
    }),
    async execute(_id: string, params: Record<string, unknown>) {
      const command = String(params.command ?? "help").trim() || "help";
      if (command === "help") {
        return {
          content: [
            {
              type: "text",
              text: "/tutor 用法：/tutor init | /tutor import file <path> | /tutor compile | /tutor lesson next | /tutor answer <文本> | /tutor status | /tutor export [dir] | /tutor reset",
            },
          ],
        };
      }
      return { content: [{ type: "text", text: "未实现：先用 /tutor help" }] };
    },
  };
}
```

**Step 4: 再跑测试**

Run: `vitest run --config vitest.extensions.config.ts extensions/tutorlab/src/tutorlab-tool.test.ts`
Expected: PASS

**Step 5: 写 manifest + package.json**

- `extensions/tutorlab/package.json`：参考 `extensions/voice-call/package.json`，依赖至少包含 `ajv`、`@sinclair/typebox`，并把 `openclaw` 放在 `devDependencies`（`workspace:*`）。
- `extensions/tutorlab/openclaw.plugin.json`：写最小 `id/name/description/skills/configSchema`

**Step 6: Commit**

Run:

```bash
bash scripts/committer "ext(tutorlab): add skeleton tool" \
  extensions/tutorlab/index.ts \
  extensions/tutorlab/package.json \
  extensions/tutorlab/openclaw.plugin.json \
  extensions/tutorlab/src/tutorlab-tool.ts \
  extensions/tutorlab/src/tutorlab-tool.test.ts \
  docs/plans/2026-02-26-tutorlab-stats.md
```

---

## Task 2：状态存储（session state）+ init/reset/status（不接 LLM）

**Files:**

- Create: `extensions/tutorlab/src/state.ts`
- Create: `extensions/tutorlab/src/telemetry.ts`
- Modify: `extensions/tutorlab/src/tutorlab-tool.ts`
- Test: `extensions/tutorlab/src/state.test.ts`
- Test: `extensions/tutorlab/src/telemetry.test.ts`

**Step 1: failing test：reset 后状态文件被清空**

```ts
import fs from "node:fs/promises";
import os from "node:os";
import path from "node:path";
import { describe, expect, it } from "vitest";
import { loadState, saveState, defaultState, resolveStatePath } from "./state.js";

describe("tutorlab state", () => {
  it("persists under agentDir", async () => {
    const root = await fs.mkdtemp(path.join(os.tmpdir(), "tutorlab-state-"));
    const p = resolveStatePath({ agentDir: root, sessionKey: "s1" });
    await saveState(p, { ...defaultState(), enabled: true });
    const s = await loadState(p);
    expect(s.enabled).toBe(true);
  });
});
```

**Step 2: 实现 `state.ts`（版本化 + 原子写入）**

- `defaultState()`：默认 `weeks=12`、`dailyMinutes=180`、`domains=["A","B","C","D"]`
- `resolveBaseDir()`：优先 `agentDir/tutorlab`，否则 `~/.openclaw/tutorlab`
- `resolveStatePath({agentDir,sessionKey})`：`sessions/<sha256(sessionKey)>.json`
- `loadState()`：不存在则返回 `defaultState()`
- `saveState()`：`writeFile(tmp)+rename` 原子替换

**Step 2b: 实现 `telemetry.ts`（追加写日志，不膨胀 state）**

- `resolveTelemetryPath({ baseDir, courseId, sessionKey })`：`<baseDir>/telemetry/<courseId>/<sha256(sessionKey)>.jsonl`
- `appendTelemetryEvent(path, event)`：确保目录存在，按行追加 JSON（append-only）
- 约束：任何“每题一次”的细粒度记录都写 telemetry，不写进 state（state 只存摘要与少量 ring buffer）

**Step 3: tool 子命令**

- `init <goal?>`：设置 goal/weeks/dailyMinutes/domains（后续可扩展参数解析，但先支持最小）
- `status`：打印概览（是否 enabled、course goal、queue 长度）
- `reset`：写回 `defaultState()` 并删除当前 course pack（如果存在）

**Step 4: 测试与命令验证**

Run: `vitest run --config vitest.extensions.config.ts extensions/tutorlab/src/state.test.ts`
Expected: PASS

**Step 5: Commit**

Run:

```bash
bash scripts/committer "ext(tutorlab): add session state + init/reset/status" \
  extensions/tutorlab/src/state.ts \
  extensions/tutorlab/src/telemetry.ts \
  extensions/tutorlab/src/state.test.ts \
  extensions/tutorlab/src/telemetry.test.ts \
  extensions/tutorlab/src/tutorlab-tool.ts
```

---

## Task 3：LLM JSON-only 运行器（schema 校验 + best-of-k + 多评委）

**Files:**

- Create: `extensions/tutorlab/src/llm-json.ts`
- Modify: `extensions/tutorlab/src/tutorlab-tool.ts`
- Test: `extensions/tutorlab/src/llm-json.test.ts`

**Step 1: failing test：无效 JSON 必须报错**

```ts
import { describe, expect, it, vi } from "vitest";
import { runJsonOnly } from "./llm-json.js";

vi.mock("../../../src/agents/pi-embedded-runner.js", () => ({
  runEmbeddedPiAgent: vi.fn(async () => ({ payloads: [{ text: "NOT JSON" }] })),
}));

describe("llm-json", () => {
  it("rejects invalid JSON", async () => {
    await expect(
      runJsonOnly({ prompt: "x", input: { a: 1 }, schema: { type: "object" } } as any),
    ).rejects.toThrow(/invalid JSON/i);
  });
});
```

**Step 2: 实现 `runJsonOnly()`**

- 动态导入 `runEmbeddedPiAgent()`（src-first / dist-fallback，参考 `extensions/llm-task/src/llm-task-tool.ts`）
- 强制 system：只输出 JSON、禁止 markdown fence、禁止 JSON 外 commentary（JSON 内允许包含结构化推理/解释字段）
- 解析输出：`stripCodeFences` + `JSON.parse`
- Ajv 校验：不通过要输出可读错误（`<root> ...`）
- 支持：
  - `bestOfK`：生成 K 份候选 JSON，再用 judge prompt 选最优（默认开启，K=3）
  - `multiJudge`：评分/归因时用 3 个评委输出 `grade+tags`，取中位数并合并标签（默认开启）
- 所有 best-of-k/judge 仍必须 JSON-only + schema 校验

**Step 2b: 评分结果 schema（防“盲目打分”）**

- 对 “评分/诊断类” JSON 统一使用 `GradeResult` schema（在 `schemas.ts` 里定义）：
  - `stepByStepAnalysis: string[]`（至少 6 条；每条短句）
  - `identifiedFlaws: Array<{ tag: string; evidence: string }>`（至少 1 条）
  - `score: number`（0..5）
  - `nextAction: { kind: string; prompt: string }`（下一步补救/回溯指令）
- 说明：JSON 对象 key 顺序无法强制校验，“score 必须最后”只能作为 prompt 约束，不能作为 Ajv 约束；但可以通过 `minItems/minLength` 强制输出足够多的推理文本来降低误判。

**Step 3: 把 tool 内部调用统一走 `llm-json.ts`（后续 compile/lesson/grade 都复用）**

**Step 4: Commit**

Run:

```bash
bash scripts/committer "ext(tutorlab): add schema-validated json-only LLM runner" \
  extensions/tutorlab/src/llm-json.ts \
  extensions/tutorlab/src/llm-json.test.ts \
  extensions/tutorlab/src/tutorlab-tool.ts
```

---

## Task 4：教材导入与索引（import file）+ chunk 化（为编译做准备）

**Files:**

- Create: `extensions/tutorlab/src/textbook.ts`
- Modify: `extensions/tutorlab/src/state.ts`
- Modify: `extensions/tutorlab/src/tutorlab-tool.ts`
- Test: `extensions/tutorlab/src/textbook.test.ts`

**Step 1: 先支持最实用的输入**

- `import file <path>`：从 `workspaceDir` 解析相对路径（用 `api.resolvePath`），读取文件
- 支持扩展名：
  - `.md/.txt`：按 utf8 读取
  - `.pdf`：用 `pdfjs-dist/legacy/build/pdf.mjs` 提取文本（参考 `src/media/input-files.ts` 的实现；先做“文本型 PDF”）
- 保存：
  - `baseDir/textbooks/<sha256(file)>.json`（含 `path/sha256/kind/extractedText/chunks[]`）
  - state 里记录 `textbookIndex.files[]`

**Step 2: chunk 策略（为了后续概念提取）**

- 以段落/标题为优先边界；fallback：按字符数切分
- 每 chunk 记录：`id, headingPath?, startChar, endChar, text, sourceRef{file,page?}`（PDF 先只记录页范围近似）

**Step 3: failing test：导入 txt 后生成 chunks 并写入 index**

**Step 4: Commit**

Run:

```bash
bash scripts/committer "ext(tutorlab): add textbook import + chunk index" \
  extensions/tutorlab/src/textbook.ts \
  extensions/tutorlab/src/textbook.test.ts \
  extensions/tutorlab/src/state.ts \
  extensions/tutorlab/src/tutorlab-tool.ts
```

---

## Task 5a：Course Compiler（compile graph）：分层 DAG + 断点续传（先把图跑通）

**Files:**

- Create: `extensions/tutorlab/src/compiler.ts`
- Create: `extensions/tutorlab/src/schemas.ts`
- Modify: `extensions/tutorlab/src/tutorlab-tool.ts`
- Test: `extensions/tutorlab/src/compiler.test.ts`

**Step 1: 定义编译产物（文件落地，便于 diff/回滚；支持断点续传）**

- `course-packs/<courseId>/compiler/stage.json`（记录已完成阶段与输入 hashes；失败可续跑）
- `course-packs/<courseId>/graphs/module-graph.json`（章节/模块级 DAG）
- `course-packs/<courseId>/graphs/modules/<moduleId>.concept-graph.json`（模块内概念 DAG）
- `course-packs/<courseId>/graphs/concept-index.json`（概念索引：conceptId→moduleId、prereqs、nexts、sourceRefs；用于 Graph RAG）
- `course-packs/<courseId>/syllabus/syllabus.md`
- `course-packs/<courseId>/problems/problem-bank.json`
- `course-packs/<courseId>/problems/rubrics.json`
- `course-packs/<courseId>/diagnostics/diagnostics.json`

**Step 2: LLM schema（必须严格）**
在 `schemas.ts` 定义（TypeBox 或 JSON Schema）：

- `ConceptNode[]`：`id,label,prereqs[],summary,intuition,formal,commonMistakes[]`
- `SyllabusWeek[]`：每周 `goals, concepts[], deliverables, domainMix`
- `ProblemItem[]`：`id,concept,domain(A/B/C/D),difficulty,question,expectedKeyPoints,trapTags[],rubricRef`
- `Rubric[]`：`id,criteria[], commonFailureModes[]`
- `DiagnosticTest`：覆盖先修概念的短测（含口试题）
  > 额外：为图阶段新增 schema：`Module[]`、`ModuleEdge[]`、`ConceptIndex`。

**Step 3: 编译流程（token heavy，但必须可扩展且不触发上下文灾难）**

> 关键原则：避免把全书 N 个概念一次性丢给 LLM 推边（O(N^2) + Lost-in-the-Middle）。改为 **分层编译（Hierarchical Compilation）** + **局部推边**。

3a) `extract_modules`：先从 chunks 生成 “模块/章节” 列表（moduleId、标题路径、覆盖的 chunkIds、摘要）

3b) `build_module_dag`：只在模块层生成粗粒度 DAG（边数远小于概念层），并要求输出每条边的 `confidence` + `why`（短句）

3c) `extract_concepts_by_module`：对每个模块单独提取概念候选（每次处理的概念数可控）

3d) `merge_dedupe`：

- 模块内去重（同义词/别名合并）
- 全局去重（同一概念在多模块出现 → 统一 conceptId，记录出现位置）

3e) `build_concept_dag_by_module`：对每个模块生成概念级微观 DAG：

- 每个概念只要求给出 “直接先修” 列表（限制候选集：同模块概念 + 模块 DAG 上游模块的少量候选；候选集硬上限建议 25）
- 要求输出每条边的 `confidence`，并在本地做 cycle check；如出现环：**不做本地盲删**，而是“发现环即熔断”：
  - 提取环上的节点/边（含 `confidence/why`）
  - 发送单一 Prompt 给 LLM（附带 Pedagogical Axioms，例如“计算优先于解释”“生成模型优先于推断”），要求输出“要移除/替换的边集”并解释（短句）
  - 本地应用并再次 cycle check；若仍有环，最多重试 2 次；再失败才 fallback 移除最低置信度边并记录诊断

3f) `emit_concept_index`：从模块内图聚合出 `concept-index.json`（双向邻接表），为后续 Graph RAG/选题提供 O(1) 邻域查询

**Step 4: failing test：`compile graph` 会写出 graphs 产物并可续跑**

- mock `runEmbeddedPiAgent`：
  - 第一次只返回 modules/module-dag/concepts/graphs
  - 第二次模拟中途失败（throw）
  - 第三次从 stage.json 续跑成功
- 验证：stage.json 记录阶段、输出文件存在、concept-index.json 可用于邻域查询

**Step 5: Commit**

Run:

```bash
bash scripts/committer "ext(tutorlab): compiler graph (hierarchical DAG + resume)" \
  extensions/tutorlab/src/compiler.ts \
  extensions/tutorlab/src/schemas.ts \
  extensions/tutorlab/src/compiler.test.ts \
  extensions/tutorlab/src/tutorlab-tool.ts
```

---

## Task 5b：Course Compiler（compile syllabus）：12 周大纲（独立阶段）

**Files:**

- Modify: `extensions/tutorlab/src/compiler.ts`
- Test: `extensions/tutorlab/src/compiler.test.ts`

**Step 1: failing test：`compile syllabus` 依赖 graphs 且只写 syllabus/**

- 先写入一个最小 graphs 夹具（或复用 Task 5a 的输出）
- 运行 `compile syllabus` 后应生成 `syllabus/syllabus.md`，并更新 stage.json（不重写 graphs）

**Step 2: 实现 syllabus 生成**

- 输入：module DAG + 每个 module 概念列表 + 学习参数（12 周、每天 180min、domains=A/B/C/D）
- 输出：每周 goals + 概念清单 + deliverables + domainMix
- 约束：周计划必须显式列出“迁移训练场 A/B/C/D”覆盖比例（避免某域长期缺失）

**Step 3: Commit**

Run:

```bash
bash scripts/committer "ext(tutorlab): compiler syllabus stage" \
  extensions/tutorlab/src/compiler.ts \
  extensions/tutorlab/src/compiler.test.ts
```

---

## Task 5c：Course Compiler（compile bank+rubrics）：题库 + Rubric（含对抗与污染元数据）

**Files:**

- Create: `extensions/tutorlab/src/data-gen.ts`
- Modify: `extensions/tutorlab/src/compiler.ts`
- Modify: `extensions/tutorlab/src/schemas.ts`
- Test: `extensions/tutorlab/src/compiler.test.ts`
- Test: `extensions/tutorlab/src/data-gen.test.ts`

**Step 1: schema 补强（为了迁移与鲁棒性）**

- `ProblemItem` 增加：
  - `sourceRef`（chunk 引用）
  - `datasetRef?: { id: string; summaryPath: string; csvPath: string }`（若题目依赖数据；用于保证 Rubric 与数据一致）
  - `dataTrapTags?: string[]`（例如 simpson, mnar, heteroskedastic, outliers, autocorr, confounding；**可为空/缺省表示 cleanroom**）
  - `assumptions[]`（题目默认假设）
  - `robustnessChecks[]`（必须做的敏感性分析提示）
- `Rubric` 必须与 `GradeResult` 对齐（评分会检查“假设/适用性/反例/敏感性分析”是否出现）

**Step 1b: 本地数据生成器（data-gen）先落地**

- `data-gen.ts`：输入 `seed + domain + contaminationFlags`，输出 `{ csv, summary, metadata }`
- 测试用统计性质断言污染存在（Simpson/MNAR/异方差等）

**Step 2: failing test：`compile bank` / `compile rubrics` 各自可独立续跑**

- bank/rubrics 任意一个阶段失败后重跑，只重跑该阶段

**Step 3: 题库生成策略（避免“干净世界”）**

- 每个核心概念至少：
  - 1 **in-domain cleanroom**（`dataTrapTags` 为空/缺省；用于建立标准模型）
  - 1 cross-domain（迁移题；通常应带 `dataTrapTags`）
  - 1 adversarial（误区诱导；必须带 `dataTrapTags`）
  - 1 robustness（敏感性分析/假设破坏；必须带 `dataTrapTags`）
- 约束从“每题必带陷阱”改为“每个概念的题集必须覆盖 cleanroom → 陷阱递进”，以避免初学者在第一题被 Simpson/MNAR/异方差等压垮导致认知崩溃。

**Step 3b: “先数据、后题目/评分”联动生成（消灭 Rubric-数据脱节）**

- 对每个“依赖数据”的题目：
  1. 先用 `data-gen.ts` 生成数据实例（固定 seed），并写入：
     - `course-packs/<courseId>/datasets/problems/<datasetId>.csv`
     - `course-packs/<courseId>/datasets/problems/<datasetId>.summary.json`
  2. 再把 `summary.json` 注入 LLM，生成该题的 `question/expectedKeyPoints/rubric`（要求引用 datasetId）
- 评分时（Task 6）把 summary 作为 judge 的事实基准，避免“学生按 CSV 算对却被判错”
  > 细节：`dataTrapTags` 非空 → `contaminationFlags=dataTrapTags`；`dataTrapTags` 为空/缺省 → `contaminationFlags=[]`（cleanroom 数据），确保“第一题干净”的可实现性。

**Step 4: Commit**

Run:

```bash
bash scripts/committer "ext(tutorlab): compiler problem bank + rubrics stages" \
  extensions/tutorlab/src/data-gen.ts \
  extensions/tutorlab/src/compiler.ts \
  extensions/tutorlab/src/schemas.ts \
  extensions/tutorlab/src/compiler.test.ts \
  extensions/tutorlab/src/data-gen.test.ts
```

---

## Task 5d：Course Compiler（compile diagnostics）：诊断测验（先修 + 迁移口试）

**Files:**

- Modify: `extensions/tutorlab/src/compiler.ts`
- Test: `extensions/tutorlab/src/compiler.test.ts`

**Step 1: failing test：`compile diagnostics` 只生成 diagnostics/**

**Step 2: 诊断内容要求**

- 覆盖：关键先修概念 + 典型误区（p 值、相关≠因果、多重比较、抽样偏差、回归到均值）
- 必须包含：
  - 计算题（少量）
  - 解释题（为什么适用/不适用）
  - 迁移题（换语境）
  - 口试题（假设、反例、局限、敏感性分析）

**Step 3: Commit**

Run:

```bash
bash scripts/committer "ext(tutorlab): compiler diagnostics stage" \
  extensions/tutorlab/src/compiler.ts \
  extensions/tutorlab/src/compiler.test.ts
```

---

## Task 6：Tutor Workbench（lesson/answer）：迁移驱动的教学闭环

**Files:**

- Create: `extensions/tutorlab/src/lesson.ts`
- Create: `extensions/tutorlab/src/grading.ts`
- Modify: `extensions/tutorlab/src/state.ts`
- Modify: `extensions/tutorlab/src/tutorlab-tool.ts`
- Test: `extensions/tutorlab/src/lesson.test.ts`

**Step 1: lesson next：选题策略（不靠固定课表）**

- 输入：state（含 masteryByDomain、queue、剩余周数/天数）
- 目标函数（先实现可解释的 rule-based）：
  - 优先补齐 `min(masteryByDomain)` 最低的 domain（但不能导致死锁）
  - 优先补齐“关键概念”（高出现在大纲/多概念依赖）
  - 保证每天有 1 个迁移题 + 1 个口试题
  - **Backtracking**：若某概念在某 domain 连续失败达到阈值（默认 3），下一轮必须先做其先修概念的诊断/补救（从概念 DAG 回溯 1~2 层）
  - **Spaced repetition**：优先处理 `queue.dueAt <= now` 的复习/口试任务；成功后安排未来几天的跨域复测
  - **Fatigue-aware**：`fatigueIndex` 高时降低对抗强度，切换到复习/直觉重建/更短的口试题，避免原地消耗
  - **Pre-test Bypass（冷启动旁路）**：对于从未测过/证据不足的概念，先从 `diagnostics.json` 抽 1 道探针题（`kind:"probe"`），通过则直接把 `mastery` 提升到阈值（如 0.85）并进入 spaced repetition；失败才进入 miniLecture
  - **Trap Ramp（认知负荷递进）**：当概念处于“新/不稳”阶段（例如 `streakSuccess=0` 或 `masteryOverall < 0.6`），只派发 **cleanroom in-domain**（`dataTrapTags` 为空/缺省）；当基础通过后，才逐步派发 cross-domain/adversarial/robustness（带 `dataTrapTags`）进行迁移与鲁棒性训练
- 输出：`current.lessonId` + `items[]`（练习条目清单）

**Step 2: lesson 生成（LLM JSON-only）**
让 LLM 输出结构化 lesson：

- `miniLecture`（直觉→图形/模拟→形式化；面向文科生）
- `practice`（2~3 题，含 1 个跨域题）
- `oralExam`（2 个口试追问，必须问“假设/反例/局限”）
- `adversarial`（1 个对抗题：诱导常见误区）
  > Graph RAG 注入：lesson prompt 里只放当前 concept 的局部子图（`V_i, V_pre, V_post` + module summary），不放全量图/全量大纲。

**Step 3: answer：多评委评分 + 误区归因 + 更新掌握度（含回溯与间隔复习）**

- `grading.ts`：实现 `gradeAnswer()`：
  - 3 个评委分别输出 `GradeResult`（含 `stepByStepAnalysis/identifiedFlaws/score/nextAction`）
  - 聚合：取中位数 score；合并 identifiedFlaws（同 tag 合并证据，计算一致性置信度）
  - 生成“纠错解释 + 下一步练习/回溯指令”（仍可 best-of-k）
- 更新 learner：
  - `masteryOverall` 用简单 EMA 更新（先不做复杂 BKT）
  - `masteryByDomain[domain]` 单独更新（迁移失败就不涨总体）
  - 把新 evidence 追加进 `evidence[]`（ring buffer：只保留最近 5~10 条）
  - 把完整事件写入 `telemetry.jsonl`（append-only）
  - 维护 streak（success/fail）并更新 `fatigueIndex`
  - 若连续失败达到阈值：写入 `queue.push({kind:"backtrack", backtrackFrom:<concept>, concept:<prereq>...})`
  - 若成功：计算 `nextDueAt`（间隔重复）并写入 review/oral 任务（跨域优先）

**Step 4: failing test：answer 会把 evidence 追加，并且当分数低时 queue 增加补救项**

**Step 5: Commit**

Run:

```bash
bash scripts/committer "ext(tutorlab): add transfer-first lesson loop (lesson/answer)" \
  extensions/tutorlab/src/lesson.ts \
  extensions/tutorlab/src/grading.ts \
  extensions/tutorlab/src/lesson.test.ts \
  extensions/tutorlab/src/state.ts \
  extensions/tutorlab/src/tutorlab-tool.ts
```

---

## Task 7：导出课程包（Notebook + Quarto 模板）+ 四域项目脚手架

**Files:**

- Create: `extensions/tutorlab/src/export.ts`
- Create: `extensions/tutorlab/src/templates/quarto-report.qmd`
- Modify: `extensions/tutorlab/src/tutorlab-tool.ts`
- Test: `extensions/tutorlab/src/export.test.ts`

**Step 1: 课程包目录结构（导出到 workspace）**
默认：`workspaceDir/.tutorlab/course-packs/<courseId>/`

- `notebooks/`：
  - `W01.ipynb` ... `W12.ipynb`（最小：Markdown 说明 + 代码块占位）
- `reports/`：
  - `report.qmd`（模板：方法/结果/局限/附录；留引用与图表位置）
- `datasets/`：
  - `domain_A_survey.csv`、`domain_B_text_counts.csv`、`domain_C_experiment.csv`、`domain_D_ab.csv`（第一版就要“脏数据”而不是完美数据）
  - `datasets/README.md`（每个数据集包含哪些“污染/陷阱”以及为什么）
- `README.md`：如何用（含 Quarto 渲染命令）

**Step 2: failing test：export 会创建目录与关键文件**

**Step 3: Quarto 模板要求**

- 强制结构：
  - 背景与问题（1 段）
  - 数据与抽样（含偏差/缺失讨论）
  - 方法（假设/检验/模型）
  - 结果（效应量 + 不确定性）
  - 稳健性/敏感性分析（必须）
  - 局限与可重复性

**Step 3b: 数据污染注入（Data Contamination）要求**
导出的合成数据必须显式包含至少 3 类真实世界陷阱（可参数化开关，便于出题复用）：

- 辛普森悖论结构（隐藏分组导致总体趋势反转）
- 非随机缺失（MNAR）或分组缺失（Missingness depends on outcome）
- 异方差/厚尾离群点（破坏正态与同方差）
- （可选）自相关误差（时间序列/面板）
- （可选）隐性混杂（影响自变量与因变量）
  实现建议：直接复用 Task 5c 的本地 `data-gen.ts` 生成 CSV + summary + metadata（固定 seed，保证可复现），并把 metadata 写入 `datasets/README.md`。

**Step 4: Commit**

Run:

```bash
bash scripts/committer "ext(tutorlab): export course pack + dirty datasets" \
  extensions/tutorlab/src/export.ts \
  extensions/tutorlab/src/export.test.ts \
  extensions/tutorlab/src/templates/quarto-report.qmd \
  extensions/tutorlab/src/tutorlab-tool.ts
```

---

## Task 8：Skill（/tutor）+ 插件文档（最小可查）

**Files:**

- Create: `extensions/tutorlab/skills/tutor/SKILL.md`
- Modify: `extensions/tutorlab/openclaw.plugin.json`
- (Optional) Create: `docs/plugins/tutorlab.md`

**Step 1: Skill frontmatter（command-dispatch: tool）**

- `name: tutor`
- `command-tool: tutorlab`
- `command-arg-mode: raw`

**Step 2: 文档最小内容**

- 10 行以内：安装（plugins install）、启用（plugins enable）、常用命令示例

**Step 3: Commit**

Run:

```bash
bash scripts/committer "ext(tutorlab): add /tutor skill + minimal docs" \
  extensions/tutorlab/skills/tutor/SKILL.md \
  extensions/tutorlab/openclaw.plugin.json
```

---

## Task 9：验证（必须跑）

**Step 1: 扩展测试**

- Run: `vitest run --config vitest.extensions.config.ts`
- Expected: PASS

**Step 2: 代码质量门禁（可选但建议）**

- Run: `pnpm check && pnpm build`
- Expected: PASS（无格式化 churn）

---

## 执行交接（给 Claude）

计划写完并保存为：`docs/plans/2026-02-26-tutorlab-stats.md`。

两种执行方式（二选一）：

1. **Parallel Session（推荐）**：开新 session，使用 `superpowers:executing-plans` 严格按 Task/Step 执行与验证。
2. **Subagent-Driven（更快）**：同一 session 内按 Task 拆分，使用 `superpowers:subagent-driven-development` 每个 Task 一个子 agent 实现，主 session 做 review 与合并。
