# TAP Migration Skill Design

## 什么是 Migration

Migration 是将测试资产（用例、数据）注册到 TAP，并将执行调度和结果收集的控制权移交给 TAP 的过程。

底层测试框架（RF、pytest 等）和测试逻辑**不变**。变的是：
- 测试用例和数据从本地/文件 → **由 TAP 管理**
- 执行触发从 CI/CD 直接调脚本 → **由 TAP 调度**
- 结果收集从本地日志 → **由 TAP 统一收集**

---

## Migration 的步骤

| 步骤 | 说明 | 谁来做 |
|---|---|---|
| 1. Assessment | 理解现状、理解 TAP 能力、找差距、产出需求文档 | **Agent**（用 skill） |
| 2. Gap Resolution | TAP 团队实现缺失的能力 | **TAP 团队**（agent 不参与） |
| 3. Data Prep + Import | 编写转换脚本、调用 TAP API 上传数据 | **开发**（工程实现，人来做） |
| 4. Validation | 跑测试验证结果正确回流 | **开发 / QA** |
| 5. Cutover | 更新 CI/CD，通知干系人，关闭旧系统 | **运维 / 开发** |

**Agent 只深度参与 Step 1。** Step 2–5 的执行主体是人，这也是为什么只需要 1 个 skill。

---

## 哪些步骤需要 Skill

**判断原则：没有这个 skill，agent 会系统性地犯错吗？**

| 步骤 | 需要 Skill？ | 原因 |
|---|---|---|
| Assessment | **是** | 没有 skill，agent 会做随意分析，漏掉关键模式，不知道对比 capability manifest，产出错误需求文档 |
| Gap Resolution | 否 | TAP 团队的事，agent 不参与 |
| Data Prep + Import | 否 | 是普通的实现工作（写代码转换数据、调 API），不是需要特殊技术的，写进计划就够 |
| Validation | 否 | "跑测试、检查结果"是显然的，不需要特殊技术 |
| Cutover | 否 | 改 CI/CD 配置是 checklist，不是技术 |

**结论：只需要 1 个 skill。**

---

## 最终结构

### Skill：`tap-migration-assessment`

位置：`skills/tap-migration-assessment/`

内容：
- Brainstorm 项目（用 `superpowers:brainstorming`）
- 派 4 个并行 agent 分析项目（按 TAP 的关切划分：Test Cases / Test Data / Execution / Results）
- 读取 TAP capability manifest（`tap-capabilities.json`）
- Gap 分析（需求 vs 现有能力）
- 产出 `tap-requirements.md`（只写差距，不写已有的）

配套资源：
- `tap-capabilities.template.json` — TAP 能力清单模板，供 TAP 团队填写

### 项目计划（不是 Skill）

当真正启动迁移时，用 `superpowers:writing-plans` 生成一次性的项目计划：

```
Phase 1: 运行 tap-migration-assessment skill
Phase 2: 等待 TAP 团队实现 gap（人工跟进）
Phase 3: 转换数据 + 调用 TAP API 导入（具体实现）
Phase 4: 验证（跑测试，检查结果回流）
Phase 5: Cutover checklist（更新 CI/CD，通知干系人，关闭旧系统）
```

---

## 设计决策记录

### 为什么不拆成多个 skill 目录

拆分的代价：文件重复（`analyze.py`、`tap-capabilities.template.json` 需要维护多份），skill 列表噪音增加，相关知识分散。收益：几乎没有，因为各 skill 之间高度耦合。

### 为什么不把 RF 逻辑单独成 skill

RF+Excel 是 assessment 的一个**发现**，不是一条独立的路径。流程相同，只是 agent 扫描时找的文件不同。Pattern B（单个 executor + Excel registry）是扫描结果，影响需求文档的内容，不影响流程分叉。

### 为什么 Assessment 是唯一的 Skill

Migration 是一个项目，不是一个技术。项目里只有 assessment 包含了真正可复用、不显然的技术（并行 agent 分析 + capability manifest 对比 + gap 分析）。其余步骤是实现工作，放进项目计划比封装成 skill 更合适。

### 关于 Skill 的命名

`tap-migration-assessment` 如实反映它做的事。`migrating-to-tap` 是项目概念，不是 skill 名称——它是用 `superpowers:writing-plans` 生成的计划的标题。
