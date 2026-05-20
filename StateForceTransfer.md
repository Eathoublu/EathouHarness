# State Force Transfer 状态强制转移规范

> 定义 Eathou Harness 在极端情况下的状态强制转移规则

## 背景

在复杂项目开发中，某些阶段可能由于以下原因无法按预期完成：
- 技术债务过重，无法快速修复
- 环境问题难以解决
- 需求本身存在模糊或矛盾
- 外部依赖阻塞

传统的"失败-回退-重试"循环在这些场景下会形成死锁，需要**状态强制转移**机制来打破僵局。

## 强制转移触发条件

### 条件1：重试次数耗尽

```python
def should_force_transfer(phase: str, demand_dir: str) -> bool:
    """判断是否应该强制转移"""
    
    retry_count = get_retry_count(phase, demand_dir)
    max_retries = get_max_retries(phase)
    
    # 达到最大重试次数
    if retry_count >= max_retries:
        return True
    
    return False
```

### 条件2：熔断器打开

```python
def should_force_transfer(phase: str, demand_dir: str) -> bool:
    """判断是否应该强制转移"""
    
    # 熔断器处于OPEN状态
    if get_circuit_breaker_state(phase) == "OPEN":
        return True
    
    return False
```

### 条件3：阶段执行超时

```python
def should_force_transfer(phase: str, demand_dir: str) -> bool:
    """判断是否应该强制转移"""
    
    start_time = get_phase_start_time(phase, demand_dir)
    max_duration = get_max_duration(phase)
    
    if time.now() - start_time > max_duration:
        return True
    
    return False
```

### 条件4：循环依赖检测

```python
def should_force_transfer(phase: str, demand_dir: str) -> bool:
    """判断是否应该强制转移"""
    
    # 检测是否在同一组阶段间循环超过N次
    state_history = get_state_history(demand_dir)
    recent_states = state_history[-20:]  # 最近20次状态变更
    
    # 检测 CODEFIX->COMPILE->CODEFIX 循环
    codefix_cycles = count_codefix_cycles(recent_states)
    if codefix_cycles >= 3:
        return True
    
    # 检测 REVIEWFIX->REVIEW->REVIEWFIX 循环
    reviewfix_cycles = count_reviewfix_cycles(recent_states)
    if reviewfix_cycles >= 3:
        return True
    
    return False
```

## 阶段超时配置

```yaml
# 各阶段最大执行时间（超时后强制转移）
phase_timeouts:
  initial: "30m"      # 初始化阶段30分钟
  analyze: "45m"      # 分析阶段45分钟
  coding: "2h"        # 编码阶段2小时
  test: "30m"         # 测试生成阶段30分钟
  compile: "15m"      # 编译阶段15分钟
  reviewing: "30m"    # 审查阶段30分钟
  dt: "30m"           # 部署测试阶段30分钟
  gardening: "15m"    # 归档阶段15分钟

# 强制转移策略配置
force_transfer:
  # 超时后策略
  on_timeout:
    strategy: "degrade"  # degrade(降级), skip(跳过), abort(终止)
    
  # 熔断后策略
  on_circuit_open:
    strategy: "degrade"
    
  # 重试耗尽策略
  on_retry_exhausted:
    strategy: "degrade"
```

## 强制转移目标状态

根据当前阶段和失败原因，强制转移到不同目标：

```
当前阶段          失败原因              强制转移目标
─────────────────────────────────────────────────────────
CODING           编码无法完成    ───>  MANUAL_REVIEW
                 (逻辑无法实现)

TEST             测试无法生成    ───>  SKIP_TEST
                 (依赖不明确)

COMPILE          编译持续失败    ───>  DEGRADED_COMPILE
                 (环境问题)           或 SKIP_COMPILE

REVIEWING        审查无法通过    ───>  DEGRADED_REVIEW
                 (代码债务重)         或 FORCE_APPROVE

DT               测试持续失败    ───>  DEGRADED_DT
                 (API不稳定)          或 SKIP_DT
```

## 强制转移实现

### 核心函数

```python
def execute_force_transfer(
    current_phase: str,
    demand_dir: str,
    reason: str
) -> str:
    """
    执行状态强制转移
    
    Args:
        current_phase: 当前阶段
        demand_dir: 需求目录
        reason: 转移原因 (timeout/circuit_open/retry_exhausted/cycle_detected)
    
    Returns:
        next_phase: 下一阶段名称
    """
    
    # 1. 记录强制转移事件
    transfer_record = {
        "from_phase": current_phase,
        "reason": reason,
        "timestamp": now(),
        "state_snapshot": get_current_state(demand_dir)
    }
    save_force_transfer_record(transfer_record, demand_dir)
    
    # 2. 确定转移策略
    strategy = determine_transfer_strategy(current_phase, reason)
    
    # 3. 执行转移
    if strategy == "degrade":
        next_phase = execute_degraded_transfer(current_phase, demand_dir)
    elif strategy == "skip":
        next_phase = execute_skip_transfer(current_phase, demand_dir)
    elif strategy == "abort":
        next_phase = execute_abort_transfer(current_phase, demand_dir)
    elif strategy == "manual_review":
        next_phase = execute_manual_review_transfer(current_phase, demand_dir)
    else:
        raise ValueError(f"Unknown strategy: {strategy}")
    
    # 4. 更新状态
    update_state_after_force_transfer(current_phase, next_phase, demand_dir)
    
    # 5. 输出转移报告
    generate_force_transfer_report(transfer_record, strategy, demand_dir)
    
    log.warning(f"强制转移: {current_phase} -> {next_phase} (原因: {reason}, 策略: {strategy})")
    
    return next_phase
```

### 降级转移

```python
def execute_degraded_transfer(phase: str, demand_dir: str) -> str:
    """执行降级转移"""
    
    # 1. 确定降级标准
    degradation_level = calculate_degradation_level(phase, demand_dir)
    degraded_standard = get_degraded_standard(phase, degradation_level)
    
    # 2. 生成降级标记文件
    degradation_mark = {
        "phase": phase,
        "level": degradation_level,
        "original_standard": get_original_standard(phase),
        "degraded_standard": degraded_standard,
        "reason": "force_transfer",
        "timestamp": now()
    }
    save_degradation_mark(degradation_mark, demand_dir)
    
    # 3. 强制验证通过（按降级标准）
    force_validate_with_degradation(phase, degraded_standard, demand_dir)
    
    # 4. 进入下一阶段
    next_phase = get_next_phase(phase)
    
    return next_phase
```

### 跳过转移

```python
def execute_skip_transfer(phase: str, demand_dir: str) -> str:
    """执行跳过转移"""
    
    # 1. 生成跳过标记文件
    skip_mark = {
        "skipped_phase": phase,
        "reason": "force_transfer",
        "warning": f"{phase}阶段被强制跳过，可能影响后续阶段",
        "timestamp": now()
    }
    save_skip_mark(skip_mark, demand_dir)
    
    # 2. 创建空产出物（满足后续阶段依赖）
    create_stub_artifacts(phase, demand_dir)
    
    # 3. 直接进入下一阶段
    next_phase = get_next_phase(phase)
    
    return next_phase
```

### 终止转移

```python
def execute_abort_transfer(phase: str, demand_dir: str) -> str:
    """执行终止转移"""
    
    # 1. 生成失败报告
    abort_report = {
        "failed_phase": phase,
        "reason": "force_transfer_abort",
        "failure_history": get_failure_history(phase, demand_dir),
        "suggestions": generate_abort_suggestions(phase, demand_dir),
        "timestamp": now()
    }
    save_abort_report(abort_report, demand_dir)
    
    # 2. 更新状态为FAILED
    update_state_status("FAILED")
    
    # 3. 通知用户
    notify_user_abort(phase, demand_dir)
    
    return "FAILED"
```

### 人工审核转移

```python
def execute_manual_review_transfer(phase: str, demand_dir: str) -> str:
    """执行人工审核转移"""
    
    # 1. 生成人工审核报告
    manual_review_report = {
        "pending_phase": phase,
        "status": "waiting_manual_review",
        "context": get_phase_context(phase, demand_dir),
        "failure_summary": summarize_failures(phase, demand_dir),
        "suggested_actions": [
            "继续并强制通过：标记完成并继续",
            "继续并重试：重置状态后重试",
            "终止流程：标记失败并结束",
            "回退到XX阶段：回退到指定阶段"
        ],
        "timestamp": now()
    }
    save_manual_review_report(manual_review_report, demand_dir)
    
    # 2. 更新状态为WAITING_MANUAL_REVIEW
    update_state_status("WAITING_MANUAL_REVIEW")
    
    # 3. 暂停自动流程
    pause_auto_flow(demand_dir)
    
    return "WAITING_MANUAL_REVIEW"
```

## 各阶段降级标准

### Compile 阶段降级

| 降级级别 | 标准 | 说明 |
|----------|------|------|
| Level 0 (正常) | 0 error, 0 warning | 理想状态 |
| Level 1 | 0 error, 允许warning | 编译通过即可 |
| Level 2 | 可运行即可 | 忽略部分非致命错误 |
| Level 3 | 人工确认 | 由人工判断是否可接受 |

### Review 阶段降级

| 降级级别 | 标准 | 说明 |
|----------|------|------|
| Level 0 (正常) | 严重=0, 一般=0, 提示任意 | 完美代码 |
| Level 1 | 严重=0, 一般<=3 | 允许少量规范问题 |
| Level 2 | 严重=0 | 只要求无致命问题 |
| Level 3 | 人工确认 | 由人工判断是否可接受 |

### DT 阶段降级

| 降级级别 | 标准 | 说明 |
|----------|------|------|
| Level 0 (正常) | pass_rate=100% | 全部通过 |
| Level 1 | pass_rate>=95% | 允许少量失败 |
| Level 2 | pass_rate>=80% | 核心场景通过 |
| Level 3 | 人工确认 | 由人工判断是否可接受 |

## 状态文件更新

强制转移后，`.state` 文件更新：

```json
{
  "current_state": "COMPILING",
  "force_transfers": [
    {
      "from": "COMPILE",
      "to": "REVIEWING",
      "reason": "circuit_breaker_open",
      "strategy": "degrade",
      "degradation_level": 2,
      "timestamp": "2024-01-15T11:00:00Z"
    }
  ],
  "degradation_marks": {
    "compile": {
      "level": 2,
      "standard": "可运行即可",
      "applied_at": "2024-01-15T11:00:00Z"
    }
  },
  "agents_status": {
    "compile": {
      "status": "completed",
      "degraded": true,
      "force_transferred": true
    },
    "reviewing": {
      "status": "running"
    }
  }
}
```

## 强制转移报告

每次强制转移生成报告：

```markdown
# 状态强制转移报告

## 转移信息
- 转移时间: 2024-01-15 11:00:00
- 从阶段: COMPILE
- 到阶段: REVIEWING
- 转移原因: circuit_breaker_open (熔断器打开)
- 转移策略: degrade (降级推进)

## 背景
COMPILE阶段已连续失败8次，达到最大重试阈值。
失败原因主要为环境依赖问题，非代码逻辑问题。

## 降级详情
- 原验收标准: 0 error, 0 warning
- 降级后标准: 可运行即可 (Level 2)
- 降级原因: 环境问题短期内无法解决

## 影响评估
- 代码质量: 可能包含警告，但不影响运行
- 后续阶段: Review阶段可能发现更多问题
- 风险等级: 中等

## 建议
1. 项目完成后清理编译警告
2. 完善环境依赖文档
3. 考虑使用容器化构建避免环境问题

## 人工确认
如需回退或调整策略，请回复：
- "回退到COMPILE" - 重新尝试编译
- "接受降级" - 继续当前流程
- "终止流程" - 停止项目
```

## 与 ManagerAgent 集成

### 在Agent调用前检查

```python
def call_agent(phase: str, demand_dir: str):
    """调用Agent（集成强制转移检查）"""
    
    # 1. 检查是否需要强制转移
    if should_force_transfer(phase, demand_dir):
        reason = get_force_transfer_reason(phase, demand_dir)
        log.warning(f"{phase} 触发强制转移，原因: {reason}")
        return execute_force_transfer(phase, demand_dir, reason)
    
    # 2. 检查是否超时
    if is_phase_timeout(phase, demand_dir):
        log.warning(f"{phase} 执行超时，触发强制转移")
        return execute_force_transfer(phase, demand_dir, "timeout")
    
    # 3. 正常调用Agent
    return call_subagent(phase, demand_dir)
```

### 在失败处理中集成

```python
def handle_agent_failure(phase: str, demand_dir: str, error: str):
    """处理Agent失败"""
    
    # 1. 增加失败计数
    increment_failure_count(phase, demand_dir)
    
    # 2. 检查是否需要强制转移
    if should_force_transfer(phase, demand_dir):
        return execute_force_transfer(phase, demand_dir, "retry_exhausted")
    
    # 3. 原逻辑：回退到修复
    return rollback_to_fixing(phase, demand_dir, error)
```

## 最佳实践

1. **记录所有强制转移**：便于事后分析和流程改进
2. **区分可恢复和不可恢复失败**：环境问题是暂时的，架构问题可能需要重启
3. **人工介入是最后手段**：自动降级优先于人工介入
4. **监控强制转移频率**：频繁强制转移说明流程或工具存在问题
5. **逐步收紧策略**：初期宽松，根据运行情况逐步收紧标准
