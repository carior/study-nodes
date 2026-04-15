🚀🚀 [core.mdc, AGENTS.md, CLAUDE.md]

我看了 `@.agents/skills` 下的 13 个 `SKILL.md`，可以按“用途”分成下面几类：

## 规范与基础

- `architecture-standards`：项目级开发总规范（组件选型、样式、请求成功判断、类型来源、命名/缩进等）；做需求设计/写代码前必须先加载。
- `component-guide`：页面组件选型与模板落地指南；强调“模板优先”、列表页 `useFormATable + QmSearchFormArea + q-antd-table`，并明确禁用 `q-antd-form`。
- `api-guide`：API 对接流程；从 `genesis-ts`/`nova-ts` 找类型与方法，更新依赖后按 `res.data` 判断成功，避免本地重复造类型。

## 文档与需求设计

- `designing-docs`：本地需求文档两阶段流程（Phase 1 澄清、Phase 2 设计稿）；输出到 `docs/design-docs.local/{app}/{需求名}/{模块名}.md`，并要求阶段间用户确认。
- `fast-design-docs`：飞书直读版设计文档流程（lark-cli 拉文档→澄清→产出设计稿→可选同步飞书）；适合“直接从飞书需求开工”。
- `doc-coauthoring`：通用文档共创方法论（收集上下文→逐节打磨→读者测试），适合写方案/RFC/决策文档。
- `spec-mode-guide`：`/spec` 工作流专用；先产出 `spec.md/tasks.md/checklist.md`，确认后再进入实现阶段。

## 代码质量与评审

- `code-reviewing`：结构化代码审查编排器；先风格审查再逻辑审查，支持按功能拆任务并汇总报告。
- `code-simplifier`：在不改行为前提下做代码简化与可维护性优化（降复杂度、提可读性、避免嵌套三元等）。

## 工程协作与交付

- `create-merge-request`：基于当前分支改动创建 MR（目标 `develop`），支持飞书需求型和普通需求型两套流程与模板。
- `managing-storybook`：检测 `packages/vue3-antd-ui` 组件改动，自动新建/增量更新对应 Storybook stories。

## 自动化与外部文档处理

- `playwright-skill`：通用浏览器自动化测试；自动探测本地 dev server，脚本写到 `/tmp`，执行页面验证/截图/流程测试。
- `feishu-download-pdf`：飞书文档下载为 PDF 并解析成 Markdown + 图片，产物落地到 `docs/feishu-docs/<文档标题>/`。

---

如果你愿意，我可以再给你一版“**什么时候该用哪个 skill**”的快捷决策图（比如“要审查改动→`code-reviewing`；要对接接口→`api-guide`；要从飞书做设计→`fast-design-docs`）。

按“从接需求到提 MR”的**一条可落地流水线**来梳理，这 13 个 skills 的依赖关系可以这样看：

## 先给结论：核心依赖主线

1. **`architecture-standards` 是全局前置依赖**  
   只要涉及需求设计、代码实现、代码设计，都必须先加载它。
2. 需求阶段有两条互斥入口（二选一）  
   - 本地文档入口：`designing-docs`  
   - 飞书直读入口：`fast-design-docs`
3. 设计完成后进入实现/对接阶段  
   - 组件实现依赖：`component-guide`（实现页面时强相关）  
   - 接口实现依赖：`api-guide`（有 API 对接时强相关）
4. 实现后质量关口  
   - 可读性收敛：`code-simplifier`（建议）  
   - 评审编排：`code-reviewing`（强建议）
5. 交付前后可选动作  
   - UI 组件库变更时：`managing-storybook`  
   - 页面行为验证时：`playwright-skill`  
   - 提 MR：`create-merge-request`（收口动作）

---

## 按完整工作流步骤看（你应该这样读）

### Step 0：确定“当前任务类型”
先问自己是：需求澄清、设计文档、实现代码、评审、提 MR，还是飞书文档处理。  
**只要不是纯查询，先加载 `architecture-standards`。**

---

### Step 1：需求输入与设计入口（二选一）
- 如果你已经有 `docs/feishu-docs/.../content.md` 这类本地材料 → 走 `designing-docs`
- 如果你只有飞书链接/token，希望直接从飞书拉正文 → 走 `fast-design-docs`
- 这两个 skill 目标相同（产出前端设计稿），只是输入源不同；一般不要同时并行跑。

**依赖关系**：  
`designing-docs` / `fast-design-docs` →（前置）`architecture-standards`

---

### Step 2：澄清与设计产出
- `designing-docs`：强调 Phase 1/2、澄清文件、模块化产出、阶段间停顿确认
- `fast-design-docs`：强调飞书直读、串行澄清、可选同步回飞书

两者产物本质一致：都落到 `docs/design-docs.local/{app}/{需求名}/{模块名}.md`（或等价设计稿）。

---

### Step 3：进入实现（编码）前的两大“执行依赖”
开始写页面/逻辑时通常会同时触发两个技能：
- `component-guide`：你要选组件、搭页面、列表/弹窗/抽屉/表单时
- `api-guide`：你要接 `genesis-ts` / `nova-ts` 接口与类型时

**依赖关系**：  
`component-guide`、`api-guide` 都是 `architecture-standards` 之下的执行规范技能。  
其中 `component-guide` 被 `architecture-standards` 明确要求“做组件时必须加载”。

---

### Step 4：实现后收敛（质量门）
- `code-simplifier`：先把你刚改的代码做“行为不变”的可读性收敛
- `code-reviewing`：再做正式审查（风格审查 → 逻辑审查 → 汇总）

**依赖关系**：  
`code-reviewing` 的风格标准本身引用 `architecture-standards`。  
所以是：实现产物 → `code-simplifier`（可选）→ `code-reviewing`（推荐必做）

---

### Step 5：测试与展示（按需插入）
- 改的是 `packages/vue3-antd-ui` 组件：加 `managing-storybook`
- 需要端到端流程/页面校验：加 `playwright-skill`
- 这两个是“验证增强”，不替代代码审查。

---

### Step 6：交付收口
- 用 `create-merge-request` 生成 MR（支持飞书需求型与普通需求型）
- 如果需求源在飞书，MR 描述可挂飞书链接，形成闭环。

---

## 关键依赖图（文字版）

`architecture-standards`  
→ (`designing-docs` **或** `fast-design-docs`)  
→ (`component-guide` + `api-guide`)  
→ (`code-simplifier`)  
→ `code-reviewing`  
→ (`managing-storybook` / `playwright-skill` 按需)  
→ `create-merge-request`

额外并行工具：  
- `doc-coauthoring`：写方案/RFC/说明文档时可插在需求或设计前后  
- `spec-mode-guide`：当你要走 `/spec` 规范驱动流程时，作为另一条“流程框架”

---

## 一版可直接执行的“最小流程”（最短可跑通）

下面这版是你日常最实用的最小闭环（有需求、有实现、有 MR）：

1. 加载 `architecture-standards`
2. 选入口：  
   - 有本地需求文档 → `designing-docs`  
   - 只有飞书链接 → `fast-design-docs`
3. 产出并确认设计稿（至少 1 篇模块文档）
4. 实现时加载：  
   - `component-guide`（页面/组件）  
   - `api-guide`（接口/类型）
5. 自检收敛：`code-simplifier`（可选但建议）
6. 审查：`code-reviewing`（至少跑一轮）
7. 若改了组件库，补 `managing-storybook`；若有页面流程风险，跑 `playwright-skill`
8. 收口：`create-merge-request`

---

如果你愿意，我可以再给你一版“**你们团队可贴在 README 的流程卡片**”（含触发关键词、何时切换 skill、失败回退策略）。

🚀🚀 [core.mdc, AGENTS.md, CLAUDE.md]

你的分阶段思路非常合理，且和这些 skill 的职责边界基本对齐。我建议把它固化成“**阶段门禁流程**”（每阶段有进入条件/产物/退出条件），这样就不会混阶段。

## 你这套流程的关系判断（结论）

- **主干正确**：`architecture-standards` 全程基线 + 阶段化触发专项 skill。
- **触发方式正确**：
  - `component-guide`：阶段1开发时自主调用（正确）
  - `playwright-skill`：阶段1验证（正确）
  - `api-guide`：阶段2主动触发（正确）
  - `code-reviewing`：阶段3主动触发（正确）
  - `code-simplifier`：阶段3自动执行（合理，建议在 review 前跑一轮）
  - `project-docs-sync`：阶段4主动触发（正确）
  - `create-merge-request`：阶段5主动触发（正确）
- **补充一点**：阶段1如果是“飞书 PDF/飞书地址”输入，建议入口明确：
  - 飞书地址：`fast-design-docs`
  - PDF 导出+解析：`feishu-download-pdf` → 再走 `designing-docs`（或继续 fast 流）

---

## 可直接执行的最小分阶段 SOP（建议你照这个跑）

### 阶段1：需求转设计 + 静态页面
- **输入**：飞书 URL 或 PDF
- **执行**：
  1. 先按输入选择 `fast-design-docs` 或 `feishu-download-pdf + designing-docs`
  2. 产出设计文档（模块级）
  3. 开发静态页面时调用 `component-guide`
  4. 用 `playwright-skill` 做页面交互/展示验证
- **产物**：
  - `docs/design-docs.local/{app}/{需求名}/{模块名}.md`
  - 可运行的静态页面（不依赖真实 API）
- **退出条件**：页面结构、交互、状态流转已可演示，设计稿已确认

### 阶段2：联调接入
- **进入条件**：服务端方案 + 接口文档已稳定
- **执行**：主动触发 `api-guide` 完成真实 API 对接、类型对齐、字段对齐
- **产物**：页面由 mock/静态切到真实接口
- **退出条件**：主流程联调通过，关键接口返回与页面展示一致

### 阶段3：质量收敛
- **执行顺序建议**：
  1. 自动跑 `code-simplifier`
  2. 主动触发 `code-reviewing`（风格→逻辑）
- **产物**：审查问题清单 + 修复结果
- **退出条件**：高优先级问题清零，代码可维护性达标

### 阶段4：文档追平实现
- **执行**：主动触发 `project-docs-sync`
- **产物**：设计文档更新为“与实现一致”
- **退出条件**：文档与实际页面/API 行为无偏差

### 阶段5：提交 MR
- **执行**：主动触发 `create-merge-request`
- **产物**：MR 标题/描述/测试建议完整
- **退出条件**：MR 可评审、可回归

---

## 一个关键操作原则（防止串阶段）

每次只做一件事：  
**“阶段内闭环后再进入下一阶段”**，不要在阶段1提前做阶段2的接口细节，也不要在阶段2同时改文档结构。  
这会让 `code-review`、`docs-sync`、`MR` 质量明显更高。

如果你愿意，我可以下一步给你一版“**阶段切换清单模板**”（每次只要勾选就知道是否能进入下一阶段）。