# Circuit Breaker 熔断器设计

> 解决 Eathou Harness 中 Agent 无限回退、卡死的问题

## 问题背景

当前系统存在以下问题：
1. **无限回退循环**：Compile/Review/DT 失败时回退到 Coding，修复后重新走流程，反复失败形成死循环
2. **无超时机制**：Agent 可能卡在某个阶段无限执行
3. **无强制转移**：重试次数达到上限后没有明确的处理逻辑

## 熔断器核心概念

熔断器（Circuit Breaker）模式借鉴自分布式系统故障处理，用于防止级联故障和快速失败。

### 三种状态

```
┌─────────────┐    失败次数 < 阈值     ┌─────────────┐
│   CLOSED    │ ◄──────────────────── │   OPEN      │
│  (正常通行)  │                       │  (熔断状态)  │
└──────┬──────┘                       └──────┬──────┘
       │                                      │
       │ 失败次数 >= 阈值                       │ 超时后
       ▼                                      ▼
┌─────────────┐    半开状态下成功    ┌─────────────┐
│ HALF_OPEN   │ ──────────────────► │   CLOSED    │
│  (半开试探)  │                       │  (正常通行)  │
└─────────────┘                       └─────────────┘
```

| 状态 | 含义 | 行为 |
|------|------|------|
| **CLOSED** | 关闭状态（正常） | 正常执行流程，记录失败次数 |
| **OPEN** | 打开状态（熔断） | 跳过执行，直接返回失败或强制推进 |
| **HALF_OPEN** | 半开状态（试探） | 允许有限次尝试，成功后恢复CLOSED |

## 熔断器配置

```yaml
# 熔断器全局配置
circuit_breaker:
  enabled: true
  
  # 各阶段熔断阈值
  thresholds:
    coding: 5        # Coding阶段最多重试5次
    test: 5          # Test阶段最多重试5次
    compile: 8       # Compile阶段最多重试8次
    reviewing: 8     # Review阶段最多重试8次
    dt: 10           # DT阶段最多重试10次
  
  # 熔断后行为
  on_open:
    strategy: "escalate"  # 策略: escalate(升级), force_skip(强制跳过), abort(终止)
    escalate_to: "manual_review"  # escalate策略下的升级目标
  
  # 半开状态配置
  half_open:
    max_attempts: 2       # 半开状态最多尝试2次
    timeout: "5m"         # 半开状态超时时间
  
  # 熔断恢复时间
  recovery_timeout: "10m"  # OPEN状态持续10分钟后尝试进入HALF_OPEN
```

## 熔断器状态文件

熔断器状态记录在 `.state` 文件中：

```json
{
  "circuit_breakers": {
    "compile": {
      "state": "CLOSED",
      "failure_count": 3,
      "last_failure_at": "2024-01-15T10:30:00Z",
      "opened_at": null,
      "half_open_attempts": 0
    },
    "reviewing": {
      "state": "OPEN",
      "failure_count": 8,
      "last_failure_at": "2024-01-15T10:45:00Z",
      "opened_at": "2024-01-15T10:45:00Z",
      "half_open_attempts": 0
    }
  }
}
```

## 熔断器工作流程

### 1. 正常流程（CLOSED状态）

```
ManagerAgent调用Agent
        ↓
Agent执行并返回结果
        ↓
成功? ──Yes──> 重置failure_count=0，继续下一流程
        No
        ↓
failure_count++
        ↓
failure_count >= threshold?
        ↓
       Yes ──> 进入OPEN状态，触发熔断处理
        No
        ↓
回退到修复阶段（原逻辑）
```

### 2. 熔断处理（OPEN状态）

当某阶段进入OPEN状态，ManagerAgent执行熔断处理：

```python
def handle_circuit_open(phase: str, demand_dir: str):
    """处理熔断状态"""
    
    strategy = config.circuit_breaker.on_open.strategy
    
    if strategy == "escalate":
        # 升级策略：记录问题，降低验收标准，继续推进
        record_degradation(phase, demand_dir)
        return "force_continue_with_degradation"
    
    elif strategy == "force_skip":
        # 强制跳过策略：跳过当前阶段，进入下一阶段
        record_skipped_phase(phase, demand_dir)
        return "force_skip_to_next"
    
    elif strategy == "abort":
        # 终止策略：停止流程，等待人工介入
        record_abort_reason(phase, demand_dir)
        update_state_status("FAILED")
        return "abort_wait_manual"
```

### 3. 渐进式降级规则

当阶段反复失败，逐步降低验收标准：

| 重试次数 | Compile标准 | Review标准 | DT标准 |
|----------|-------------|------------|--------|
| 1-3次 | 0 error, 0 warning | 严重问题=0 | pass_rate=100% |
| 4-6次 | 0 error, 允许warning | 严重+一般问题=0 | pass_rate>=95% |
| 7-8次 | 可运行即可 | 严重问题=0 | pass_rate>=80% |
| >8次(熔断) | 人工确认 | 人工确认 | 人工确认 |

### 4. 半开恢复流程

OPEN状态持续`recovery_timeout`后，尝试进入HALF_OPEN：

```
OPEN状态超时
    ↓
进入HALF_OPEN状态
    ↓
允许有限次尝试（half_open.max_attempts）
    ↓
成功? ──Yes──> 恢复CLOSED状态，重置计数器
    No
    ↓
回到OPEN状态，延长recovery_timeout
```

## 与 ManagerAgent 集成

### 修改点1：调用Agent前检查熔断器

```python
def call_agent_with_circuit_breaker(phase: str, demand_dir: str):
    """带熔断器保护的Agent调用"""
    
    # 1. 检查熔断器状态
    cb_state = get_circuit_breaker_state(phase)
    
    if cb_state == "OPEN":
        # 检查是否可以进入HALF_OPEN
        if can_attempt_half_open(phase):
            update_circuit_state(phase, "HALF_OPEN")
            log.info(f"{phase} 熔断器进入半开状态，允许试探性执行")
        else:
            # 执行熔断处理
            return handle_circuit_open(phase, demand_dir)
    
    # 2. 正常调用Agent
    result = call_subagent(phase, demand_dir)
    
    # 3. 根据结果更新熔断器
    if result.success:
        if cb_state == "HALF_OPEN":
            # 半开状态成功，恢复CLOSED
            reset_circuit_breaker(phase)
            log.info(f"{phase} 熔断器恢复关闭状态")
        else:
            # 正常成功，重置失败计数
            reset_failure_count(phase)
    else:
        # 失败，增加计数
        increment_failure_count(phase)
        
        # 检查是否达到阈值
        if get_failure_count(phase) >= get_threshold(phase):
            open_circuit_breaker(phase)
            log.warning(f"{phase} 熔断器打开，失败次数达到阈值")
    
    return result
```

### 修改点2：回退逻辑增加熔断检查

```python
def handle_phase_failure(phase: str, demand_dir: str):
    """处理阶段失败"""
    
    # 1. 增加重试计数
    retry_count = increment_retry_counter(phase)
    max_retries = get_max_retries(phase)
    
    # 2. 检查熔断器
    cb_state = get_circuit_breaker_state(phase)
    
    if cb_state == "OPEN":
        # 熔断状态下，不再回退，执行强制推进
        log.warning(f"{phase} 处于熔断状态，执行强制推进策略")
        return execute_force_transfer(phase, demand_dir)
    
    # 3. 检查是否达到最大重试次数
    if retry_count >= max_retries:
        log.warning(f"{phase} 达到最大重试次数({max_retries})，触发熔断")
        open_circuit_breaker(phase)
        return execute_force_transfer(phase, demand_dir)
    
    # 4. 原逻辑：回退到修复阶段
    log.info(f"{phase} 失败，回退到修复阶段（重试 {retry_count}/{max_retries}）")
    return rollback_to_fixing(phase, demand_dir)
```

## 状态强制转移策略

### 策略1：降级推进（Degrade & Continue）

当阶段熔断时，降低验收标准后继续推进：

```python
def execute_degrade_and_continue(phase: str, demand_dir: str):
    """降级并继续"""
    
    # 1. 记录降级信息
    degradation_record = {
        "phase": phase,
        "original_standard": get_standard(phase),
        "degraded_standard": get_degraded_standard(phase),
        "reason": "circuit_breaker_open",
        "timestamp": now()
    }
    save_degradation_record(degradation_record, demand_dir)
    
    # 2. 生成降级报告
    generate_degradation_report(phase, demand_dir)
    
    # 3. 强制标记阶段完成（带降级标记）
    force_mark_complete(phase, demand_dir, degraded=True)
    
    # 4. 更新状态并推进
    update_state_and_proceed(phase, demand_dir)
    
    log.info(f"{phase} 执行降级推进，原标准：{degradation_record['original_standard']}，降级后：{degradation_record['degraded_standard']}")
```

### 策略2：强制跳过（Force Skip）

当阶段无法完成时，跳过该阶段：

```python
def execute_force_skip(phase: str, demand_dir: str):
    """强制跳过当前阶段"""
    
    # 1. 记录跳过信息
    skip_record = {
        "phase": phase,
        "reason": "circuit_breaker_open_after_max_retries",
        "timestamp": now(),
        "warning": f"{phase}阶段被强制跳过，可能导致后续阶段失败"
    }
    save_skip_record(skip_record, demand_dir)
    
    # 2. 生成跳过报告
    generate_skip_report(phase, demand_dir)
    
    # 3. 更新状态为跳过
    update_state_status(phase, "skipped")
    
    # 4. 直接进入下一阶段
    next_phase = get_next_phase(phase)
    update_current_state(next_phase)
    
    log.warning(f"{phase} 被强制跳过，进入 {next_phase}")
```

### 策略3：人工介入（Manual Review）

当阶段无法自动完成时，暂停等待人工处理：

```python
def execute_manual_review(phase: str, demand_dir: str):
    """暂停等待人工介入"""
    
    # 1. 记录等待信息
    wait_record = {
        "phase": phase,
        "status": "waiting_manual_review",
        "reason": "circuit_breaker_open",
        "failure_history": get_failure_history(phase),
        "timestamp": now()
    }
    save_manual_review_record(wait_record, demand_dir)
    
    # 2. 生成等待人工处理报告
    generate_manual_review_report(phase, demand_dir)
    
    # 3. 更新状态为等待人工
    update_state_status("WAITING_MANUAL_REVIEW")
    
    # 4. 通知用户
    notify_user_for_manual_review(phase, demand_dir)
    
    log.info(f"{phase} 等待人工介入，请检查 {demand_dir}/manual_review_report.md")
    
    # 5. 暂停流程（不自动继续）
    return "paused_waiting_manual"
```

## 人工介入后的恢复

当人工介入后，用户可以通过指令恢复流程：

```
用户指令：
- "继续并强制通过" -> 标记当前阶段完成（degraded=true），继续下一流程
- "继续并重试" -> 重置熔断器，重新执行当前阶段
- "终止流程" -> 标记FAILED，结束流程
- "回退到XX阶段" -> 回退到指定阶段重新执行
```

## 熔断器可视化

ManagerAgent 输出中应包含熔断器状态：

```
【熔断器状态】
┌──────────┬─────────┬──────────────┬────────────────┐
│ 阶段      │ 状态     │ 失败次数      │ 最后失败时间     │
├──────────┼─────────┼──────────────┼────────────────┤
│ Coding   │ CLOSED  │ 0/5          │ -              │
│ Test     │ CLOSED  │ 1/5          │ 10:15          │
│ Compile  │ OPEN    │ 8/8 ⚠️       │ 10:45 🔥       │
│ Review   │ CLOSED  │ 2/8          │ 10:30          │
│ DT       │ CLOSED  │ 0/10         │ -              │
└──────────┴─────────┴──────────────┴────────────────┘

⚠️ Compile阶段已熔断，执行降级推进策略
原标准：0 error, 0 warning
降级后：可运行即可
```

## 实施建议

1. **渐进式启用**：先在高频失败阶段（Compile、DT）启用熔断器
2. **监控告警**：熔断器打开时立即通知用户
3. **阈值调优**：根据实际运行情况调整各阶段阈值
4. **日志记录**：详细记录熔断器状态转换历史，便于事后分析

## 与超时机制配合

熔断器与超时机制配合，形成双重保护：

```
超时检测 ──Yes──> 强制中断 ──> 记录超时失败 ──> 检查熔断器
                                              ↓
正常完成 ◄── 继续流程 ◄── 未达阈值 ◄──Yes── 熔断器OPEN?
                                              No
                                              ↓
                                        增加失败计数
                                              ↓
                                        达到阈值?
                                              ↓
                                        Yes──> 打开熔断器
```
