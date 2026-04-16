# Eathou Harness

> 多 Agent 协作系统。需求入口 → 编码 → 测试 → 编译 → 审查 → 部署测试 → 归档交付。

## 架构更新 (2026-04-14)

本次更新重构了各 Agent 的职责边界，优化了工作流，增强了错误处理机制和上下文管理能力。

### 核心变更

1. **ManagerAgent**: 强化编排逻辑，增加全局上下文管理、用户交互机制和脚本管理机制
2. **CodingAgent**: 唯一代码实现入口，统一处理 TASK-C-*（初始开发）和 TASK-FIX-*（修复任务）
3. **CompileAgent**: 完善编译任务处理，支持详细日志记录、失败诊断和 scripts 目录治理
4. **ReviewingAgent**: 优化代码审查流程，通过 coding_task.md 管理所有修复任务
5. **整体**: 优化各 Agent 职责边界，增强工作流的健壮性

---

## 一目了然

```
用户需求 → Manager → Initial → Analyze → CODING ║ TEST → Compile → Review → DT → Gardening → 交付
```

- **Manager**：编排中枢，调度所有 Agent，管理全局上下文
- **Initial**：初始化，扫描生成全局基础文件（API列表、数据模型、架构文档）
- **Analyze**：需求分析，拆解为 feature → 精确task
- **Coding**：唯一代码实现入口，处理 TASK-C-*（初始开发）和 TASK-FIX-*（修复任务）
- **Test**：按 task 编写测试用例
- **Compile**：编译 + 测试，记录详细日志，治理 scripts 目录
- **Review**：代码审查（严重问题任务通过 coding_task.md 管理）
- **DT**：启动应用，验证 API 功能
- **Gardening**：归档整理，更新全局文件

---

## 核心流程

### 流程图（Mermaid）

```mermaid
flowchart TD
    START([IDLE]) --> CHECK{CHECKING}
    CHECK -->|global 完备| ANALYZING
    CHECK -->|global 缺失| INITIAL[InitialAgent]
    INITIAL --> ANALYZING
    
    ANALYZING[AnalyzeAgent] --> PAR{CODING ║ TEST}
    PAR -->|Coding| CODING[CodingAgent]
    PAR -->|Test| TEST[TestAgent]
    CODING --> COMPILING
    TEST --> COMPILING
    
    COMPILING[CompileAgent] --> COMPILE_RESULT
    COMPILE_RESULT -->|pass| REVIEWING[ReviewingAgent]
    COMPILE_RESULT -->|fail| FIXING
    
    REVIEWING --> REVIEW_RESULT
    REVIEW_RESULT -->|pass| DT[DTAgent]
    REVIEW_RESULT -->|fail| FIXING
    
    DT --> DT_RESULT
    DT_RESULT -->|pass| GARDENING[GardeningAgent]
    DT_RESULT -->|fail| FIXING
    
    GARDENING --> END([COMPLETED])
    
    FIXING -.-> CODING
    FIXING -.-> REVIEWING
```

### TDD 变体

Coding 和 Test **并行**执行，基于 Analyze 输出的精确 task：

- Coding 按 coding_task.md 实现（精确到类路径、方法名、出入参）
- Test 按 test_task.md 编写测试（精确到断言条件）

---

## 文件结构

```
artifacts/
├── global/                    # 全局文件（跨需求）
│   ├── api_list.yaml
│   ├── data_model.yaml
│   └── architecture.md
└── artifact-{demand}-{YYYY-mm-dd}/   # 需求目录
    ├── 02_analyze/
    │   ├── feature_list.json
    │   ├── coding_task.md
    │   └── test_task.md
    ├── 03_coding/            # 源码
    ├── 04_test/             # 测试
    ├── 05_compile/         # 编译结果
    ├── 06_reviewing/       # 审查报告
    ├── 07_dt/              # API测试
    └── 08_gardening/       # 归档
```

---

## 信号机制

- `.complete`：各阶段完成信号（空文件）
- 验证通过后 Manager 写入 `approve: timestamp`
- 验证不通过：删除信号，责令 Agent 继续

---

## 完成标准

| 阶段 | 标准 |
|------|------|
| Initial | api_list.yaml + data_model.yaml + architecture.md 存在 |
| Analyze | feature_list.json + coding_task.md + test_task.md |
| Coding | coding_task.md 中所有 TASK-C-* 和 TASK-FIX-* 完成（`[ ]` → `[x]`）|
| Test | test_task.md 中所有 `[ ]` → `[x]` |
| Compile | status = pass |
| Review | coding_task.md 中所有任务完成且无新增修复任务 |
| DT | pass_rate = 100% |
| Gardening | final_report.md + changelog.json |

---

## Review 审查标准

- **严重**：影响运行逻辑/健壮性 → 必须清零
- **一般**：代码规范问题
- **提示**：风格建议

---

## Harness 思想

### 1. 强制执行验证

- 关键阶段**必须执行命令**并获取控制台输出
- Compile 记录编译命令 + 输出（无错误的关键部分）
- DT 调用 API，记录实际响应
- 所有输出可追溯、可验证

### 2. 任务闭环

- task.md 中所有 `[ ]` 必须打勾变成 `[x]`
- Manager 验证：统计所有 TASK 标记，未完成则继续

### 3. 上下文完备

- 项目上下文必须完备才能开工
- Initial 扫描生成 global 文件（API/模型/架构）
- Analyze 解析 feature + 精确 task
- 上下文缺失 → 返回上一步补充

### 4. 汇报机制

- 所有 Subagent 由 Manager 创建
- 完成后向 Manager 汇报（生成 .complete 信号）
- Manager 验证通过才进入下一阶段

---

## DT 测试用例设计

基于 feature 设计，每个 feature 包含：

- 正向用例（期望成功）
- 负向用例（期望失败：参数错误、权限等）

---

## 状态文件

`artifacts/.state` 记录当前状态：

- current_state: 阶段名
- agents_status: 各 Agent 完成状态
- retry_counters: 重试计数

---

## 各 Agent 定义

见同名 .md 文件：

- `ManagerAgent.md` - 编排定义（已更新：强化上下文管理、错误处理、用户交互和脚本管理）
- `InitialAgent.md` - 初始化定义
- `AnalyzeAgent.md` - 需求分析定义
- `CodingAgent.md` - 编码定义（已更新：唯一代码实现入口，处理TASK-C-*和TASK-FIX-*）
- `TestAgent.md` - 测试定义
- `CompileAgent.md` - 编译定义（已更新：完善编译流程、日志记录、失败诊断和scripts目录治理）
- `ReviewingAgent.md` - 审查定义（已更新：优化审查流程，通过coding_task.md管理修复任务）
- `DTAgent.md` - 部署测试定义
- `GardeningAgent.md` - 归档定义

---

## 更新日志

### 2026-04-15

- **ManagerAgent**: 完善用户交互机制（问答支持和打断处理）和脚本管理机制（scripts 目录治理）
- **CodingAgent**: 明确为唯一代码实现入口，统一处理 TASK-C-* 和 TASK-FIX-* 任务
- **CompileAgent**: 增强日志记录、失败诊断功能，完善 scripts 目录治理
- **ReviewingAgent**: 优化任务管理方式，所有审查问题通过 coding_task.md 统一管理
- **整体**: 更新完成标准，优化各 Agent 职责边界，消除重复内容

### 2026-04-14

- **ManagerAgent**: 增强编排逻辑，优化全局上下文管理，完善错误处理机制
- **CompileAgent**: 完善编译任务处理流程，增强编译报告生成
- **DTAgent**: 强化部署测试能力，增加错误诊断和日志分析
- **ReviewingAgent**: 优化代码审查流程，完善问题分类（严重/一般/提示）
- **TestAgent**: 改进测试生成逻辑，增强上下文感知能力
- **整体**: 优化各 Agent 职责边界，增强工作流的健壮性

---

*最后更新: 2026-04-15*