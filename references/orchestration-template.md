# 编排模板（Orchestration Template）

把 [SKILL.md](../SKILL.md) 的方法论落成一次可运行的编排。以下为 TypeScript 风格伪代码，
取自一次 11 功能域 RPC 面全量盘点+修复的实战脚本（已脱敏）；适配到任何支持「并行子代理 +
结构化返回 + 阶段日志 + 看板工件」的运行时（子代理工具、多 Agent 会话、CI 流水线均可）。
角色 system 提示词见 SKILL.md「角色」一节（含两个辅助角色的完整提示词），此处引用其简称：
AUDITOR_SYSTEM=审计员、VERIFIER_SYSTEM=复核员、REPAIR_SYSTEM=修复员、GATE_SYSTEM=门禁员、
DEDUP_SYSTEM=去重合并员、COMMITTER_SYSTEM=专职提交员。`agent()`/`board` 为宿主运行时原语
（无编排运行时的串行替代见文末）。

## 常量与数据结构

```ts
const WT = "/path/to/.worktrees/audit-YYYYMMDD";   // 独立工作树（基线已钉死）
const EVID = "/path/to/evidence-dir";              // 仓库外证据目录
const BASE = "<origin/main SHA，fetch 后记录>";     // 基线，全程引用
const BRANCH = "audit-fixes-YYYYMMDD";

interface Unit { id: string; title: string; files: string[]; focus: string; readme: string }

// 示例单元——照此形状换成你的功能域切分（files=仓库相对路径；focus=一句审计焦点线）
const units: Unit[] = [
  { id: "U01", title: "对外 API 入口层",
    files: ["src/api/router.ts", "src/api/middleware.ts", "src/api/errors.ts"],
    focus: "参数校验、错误映射、状态码语义与鉴权边界", readme: "docs/api.md" },
  { id: "U02", title: "持久化层",
    files: ["src/store/db.ts", "src/store/migrations/"],
    focus: "SQL 语义、事务与锁序、双写一致性、迁移安全性", readme: "README.md" },
];

// Finding / Verdict / VerifiedFinding 的完整字段见 SKILL.md「发现与裁定数据结构」
interface Finding { where: string; what: string; trigger: string; codeEvidence: string;
  semanticEvidence: string; impact: string; disproofAttempt: string; fixDirection: string;
  severity: "high" | "medium" | "low" }
interface AuditResult { findings: Finding[]; covered: string[]; notCovered: string[] }
interface Verdict { ruling: "maintain" | "overturn" | "revise"; // 维持 / 推翻 / 修正
  confirmNote: string;
  severity: "high" | "medium" | "low"; finalWhat?: string; finalFixDirection?: string }

// 发现 + 裁定 → 一条带生命周期的记录（fixStatus 起始 pending）
function toVerifiedFinding(f: Finding, v: Verdict, id: string, unitId: string): VerifiedFinding {
  return { ...f, id, unitId, ruling: v.ruling, confirmNote: v.confirmNote,
    fixDirection: v.finalFixDirection ?? f.fixDirection, finalSeverity: v.severity,
    finalStatus: v.ruling === "overturn" ? "overturned" : "confirmed",
    fixStatus: "pending", fixNote: "" };
}
interface VerifiedFinding extends Finding {
  id: string; unitId: string; ruling: string; confirmNote: string; finalSeverity: string;
  finalStatus: "confirmed" | "overturned"; fixStatus: "pending" | "fixed" | "merged"; fixNote: string
}
interface RepairReceipt { filesChanged: string[]; testsRun: { cmd: string; exitCode: number; tail: string }[];
  fixNotes: { id: string; note: string }[]; concerns: string }
interface GateReceipt { overall: "pass" | "fail";
  commands: { cmd: string; exitCode: number; tail: string }[]; failures: string[]; skipped: string[] }
```

## 流程骨架（六阶段）

```ts
// ── 阶段 1：并行审计（单元 = 功能域，Promise.all）──────────────────────────
const unitResults = await Promise.all(units.map(async (unit) => {
  const audit = await agent(`审计员-${unit.id}`, AUDITOR_SYSTEM).ask<AuditResult>(
    `目标仓库（唯一读这里）：${WT}（基线 ${BASE}）。
     功能域文件：${unit.files.join("\n- ")}；审计焦点：${unit.focus}。
     阅读顺序：仓库范围/非目标声明 → ${unit.readme} → 文件逐个精读（完整函数体+调用方交叉跟读）
     → 对应测试（测试通过不证明生产代码安全）→ 契约源码参考。
     方法论：双侧证据 / 可复算 / 先试图推翻 / null≠0 / severity 独立给理由。
     允许只读小探针，脚本与输出存 ${EVID}/${unit.id}-<编号>/；禁止装依赖/跑套件/起容器。
     文本匹配只是线索；文档与实现矛盾算 low。最多 8 条，宁缺毋滥；零发现就明说。
     返回 JSON：findings[] / covered[] / notCovered[]。`
  );

  // ── 阶段 2：每条发现并行推翻式复核 ────────────────────────────────────
  const verified = await Promise.all(audit.findings.map(async (f, i) => {
    const id = `${unit.id}-F${i + 1}`;
    const v = await agent(`复核员-${id}`, VERIFIER_SYSTEM).ask<Verdict>(
      `复核下面这条发现（仓库只读 ${WT}，基线 ${BASE}；先读仓库边界声明确认既定边界）：
       ${JSON.stringify(f)}
       1. 自己重读完整函数+调用方+测试+文档，独立复算，不信原结论。
       2. 可跑只读探针，产物存 ${EVID}/V-${id}/（不复用审计员脚本），confirmNote 引用探针编号。
       3. 裁定三选一，输出英文枚举：maintain=维持 / overturn=推翻（含已文档化边界、证据不足）/ revise=修正（写明修正后内容）。
       4. confirmNote 必须可复核：行号、测试名或探针编号 + 独立理由。`
    );
    board.report({ id, stage: v.ruling === "overturn" ? "已推翻" : "已确认", ... });
    return toVerifiedFinding(f, v, id, unit.id);
  }));
  return { unit, audit, verified };
}));

// ── 阶段 3：跨域去重（不同域可能从接口两侧看到同一根因）──────────────────
// allFindings = 阶段 1 各单元 verified 的汇总；confirmed = 其中 finalStatus === "confirmed" 的子集
const compact = confirmed.map((f) => ({ id: f.id, unit: f.unitId, where: f.where,
  what: f.what, severity: f.finalSeverity }));
const deduper = await agent("去重合并员", DEDUP_SYSTEM)
  .ask(`已确认清单 ${JSON.stringify(compact)}。找出同一代码位置+同一根因的重复项；
       仅措辞不同不算。返回 duplicates[]{keepId, mergeId, note}。`);
// mergeId 项标 fixStatus="merged" 并入 keepId 所在域，看板移入「已并入」列

// ── 阶段 4：按域串行修复（共享同一工作树，避免并发写冲突）─────────────────
for (const unit of units) {
  const todo = confirmed.filter((f) => f.unitId === unit.id && f.fixStatus !== "merged");
  if (!todo.length) continue;
  const receipt = await agent(`修复员-${unit.id}`, REPAIR_SYSTEM).ask<RepairReceipt>(
    `工作树（唯一可写）：${WT}，分支 ${BRANCH}，基线 ${BASE}。先读仓库协作规范。
     待修复问题（已独立复核；confirmNote 优先于原始描述，按裁定执行）：${JSON.stringify(todo)}
     硬规则：文件改动一律用编辑工具；禁止 git commit/push/add（主会话统一提交）；
     修法不得破坏仓库既定边界；同一根因全实例一次清；文档/测试同步；每条尽量带回归测试；
     环境装不上就在 concerns 说明。返回 JSON：filesChanged / testsRun（逐字 tail）/ fixNotes / concerns。`
  );
}

// ── 阶段 5：全量测试门禁（≤3 轮 门禁→善后修复 循环）────────────────────────
let prev: GateReceipt | undefined;   // 上一轮门禁结果，串给下一轮善后修复员
for (let round = 1; round <= 3; round++) {
  const gate = await agent("门禁员", GATE_SYSTEM).ask<GateReceipt>(
    `仓库：${WT}。跑仓库自己的确定性检查全集（单测/编译/协议/契约检查），逐字贴摘要行；
     环境不具备就记 skipped 及原因；需要容器的测试起临时容器、结束必须清理。
     ${prev && prev.overall === "fail" ? `上一轮失败项：${JSON.stringify(prev.failures)}` : ""}`
  );
  prev = gate;
  if (gate.overall === "pass") break;
  await agent(`善后修复员-r${round}`, REPAIR_SYSTEM).ask(`门禁失败项：…；定位根因修复，同一根因全实例一次清掉。`);
}

// ── 阶段 6：报告（三段：确认并修复 / 已推翻 / notCovered）──────────────────
// 摘要 + 覆盖率账本表（域/文件数/发现/确认）+ 每条确认项（位置/影响/复核裁定/修复说明）
// + 已推翻理由摘要 + 变更文件清单 + 门禁命令逐条 exit code + skipped 清单。
// 结论同时给出：「改动在工作树未提交，由主会话自查后经专职提交员（显式路径 add → commit →
// push → 开草稿 PR → 合入用 head-commit 精确匹配）落草稿 PR，等待合入决策。」
```

## 落地要点（实战换来的）

- **单元定义是脚本里最值钱的部分**：文件清单+焦点一句线写死在脚本里，不要让审计员自己找范围。
- 看板（已确认/已推翻/已修复/已并入）不是装饰——长跑编排里它是主会话恢复上下文的锚。
- 修复员共享一个 worktree ⇒ 按域**串行**；要并行就多 worktree + 文件集不相交分组，且
  **重编译门禁串行**（见 SKILL.md 门禁一节）。
- 多 PR 拆分手法：按包边界分组 commit 后 `git checkout -b <new> <base>` 带着其余脏改动切换
  （脏文件在两 commit 间无差即可），不需要 cherry-pick。
- 编排层只留静态字符串参数；一切含子代理产出文本或运行期派生值（SHA、PR 号、commit 正文）
  的 git/平台 CLI 写操作交给**专职提交子代理**（安全钩子会拦动态拼接，且这是注入面的正解）。
- 配额断点：用「仅改设置的续跑/换模型」原样重启，已完成阶段全部命中缓存；勿改脚本。
- 报告里的每个 exit code / 测试数都要来自门禁员逐字返回，主会话抽查复算再合入。

## 无编排运行时时的降级用法

单会话 Agent 也能按同构流程串行执行：把自己依次切换进四个角色（审→逐条自我推翻→修→跑
门禁），**关键是角色切换时丢弃上一角色的结论**（复核阶段只看发现 JSON 与代码，不看自己
写过的推理），并保留同样的证据目录与三段式报告。对抗性来自角色隔离，不来自并行。
