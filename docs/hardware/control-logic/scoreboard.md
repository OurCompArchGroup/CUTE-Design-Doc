# Scoreboard 依赖检查（Scoreboard）
## 1. 术语说明

| 术语 | 说明 |
|------|------|
| QueryReq | TC 发起的准入查询请求，描述本条候选 uop 的 `fuType/src/dest` |
| Update 事务 | TC 发起的状态更新事件，包含 `allocate/issue/finish` 全链路 |
| Reg 状态表 | `abRegStatus` / `cRegStatus`，记录寄存器是否被占用及生产者 FU |
| FU 状态表 | `fuStatuses`，记录 AML/BML/CML/Compute 的 busy、src、dest 状态 |
| reserve/release | 生产者占用/释放寄存器，形成并解除写依赖 |
| wakeup | 生产者完成后唤醒等待该结果的消费者源操作数 |

## 2. 设计规格

| 参数 | 说明 |
|------|------|
| AB 寄存器槽位数 | 4（`NumAbRegs`） |
| C 寄存器槽位数 | 2（`NumCRegs`） |
| FU 类型 | `AML/BML/CML/Compute`（每类单实例） |
| 查询接口 | `Decoupled(QueryReq)` |
| 更新接口 | `ScoreboardUpdateIO`（`load/compute/store` 的 issue 与 finish） |

## 3. 功能描述

Scoreboard 负责“是否允许发射”与“依赖状态闭环”两件事：
1. **准入判定**：根据目标写安全、源就绪、FU 资源可用性判断 `canIssueReq`。
2. **状态维护**：在 TC 发射与完成事件驱动下，更新 FU/寄存器状态并执行依赖唤醒。

职责边界：
- **TC**：构造查询、驱动更新事务。
- **Scoreboard**：仅负责相关性与资源状态，不参与数据搬运。

## 4. 微架构设计
### 4.1 状态结构

Scoreboard 内部维护三类核心状态：

1. **寄存器结果状态**
- `abRegStatus`：AB 寄存器 busy + fuId
- `cRegStatus`：C 寄存器 busy + fuId

2. **功能单元状态**
- `fuStatuses`：按 FU 类型索引
- 每项包含 `busy/fifoIdx/destValid/destReg/srcs(3)`

3. **源操作数状态（每个 FU 的 src 槽）**
- `valid`：源是否存在
- `reg`：源寄存器标识
- `ready`：是否就绪
- `waitFu`：等待的生产者 FU
- `readPending`：读是否尚未完成

### 4.2 准入判定：`canIssueReq`

`canIssueReq` 的核心判定项如下：

1. **请求合法性**
- `recognized = req.fuType =/= None`

2. **写目标安全**
- `destBusy`：目标寄存器是否被占用写
- `destHasConsumers`：目标寄存器是否仍有 pending reader
- `writesOk = !dest.valid || (!destBusy && !destHasConsumers)`

3. **源就绪**
- `srcsReady = regReady(src1) && regReady(src2) && regReady(src3)`

4. **FU 资源可用**
- `loadChecks`：AML/BML/CML(load) 对应 FU 空闲
- `computeChecks`：Compute FU 空闲
- `storeChecks`：CML(store) 空闲，且 `storeConsumersOk`

最终：
```scala
issueOk = (loadChecks || computeChecks || storeChecks) && srcsReady && writesOk
canIssueReq = recognized && issueOk
```

### 4.3 发射更新：`allocate/issue`

1. **`load_allocate`**
- 按 `has_a/has_b/has_c` 分配 AML/BML/CML(load)。
- 设置 FU busy/dest/fifoIdx，清空 src 槽。
- 对应寄存器执行 `reserveAbRegister` 或 `reserveCRegister`。

2. **`compute_issue`**
- 占用 Compute FU，先 `reserveCRegister(destC)`。
- 构建 `srcA/srcB/srcC`：
- 若源寄存器 busy：`ready=false, waitFu=producerFu`
- 否则：`ready=true`
- 对 A/B 源设置 `readPending=true`；
- C 源在与本次写目标冲突时进入等待。

3. **`store_issue`**
- 以 CML(store) 模式占用 FU（`destValid=false`）。
- 配置 `srcC` 为待读 C。
- 若 C 当前被写占用，则等待对应生产者 FU。

### 4.4 完成更新：`*_finish`、release 与 wakeup

1. **Load 写回完成**
- `load_finish_a/b/c`：释放对应 FU 与目标寄存器。
- 调用 `releaseAbRegister` 或 `releaseCRegister`。

2. **Compute 读完成**
- `compute_read_finish_a/b`：将对应源槽 `readPending := false`。

3. **Compute 写回完成**
- `compute_write_finish_c`：释放 Compute FU 与目标 C 寄存器。
- 触发对依赖消费者的唤醒。

4. **Store 完成**
- `store_finish`：释放 CML(store) busy，并清理 `srcC` 槽。

`release*` 内部会调用 `wakeup*Consumers`：
- 当消费者 `waitFu == producerFu` 且寄存器匹配时，设置 `ready := true`、`waitFu := None`。

## 5. 接口信号

| 信号组 | 方向 | 说明 |
|--------|------|------|
| `query.req.valid/bits/ready` | TC -> SB / SB -> TC | 发射前准入查询 |
| `update.load_allocate` | TC -> SB | Load 资源分配与寄存器 reserve |
| `update.compute_issue` | TC -> SB | Compute 发射与 src/dest 依赖建立 |
| `update.store_issue` | TC -> SB | Store 发射与 C 源依赖建立 |
| `update.load_finish_*` | TC -> SB | Load 写回完成 |
| `update.compute_read_finish_*` | TC -> SB | Compute 读 A/B 完成 |
| `update.compute_write_finish_c` | TC -> SB | Compute 写 C 完成 |
| `update.store_finish` | TC -> SB | Store 读 C 完成 |

## 6. 与其他模块的交互

```
TaskController
   ├── query.req  ─────────────► Scoreboard (准入判定)
   ├── update.issue/allocate ──► Scoreboard (建立依赖)
   └── update.finish ──────────► Scoreboard (释放/唤醒)
```

对 Top-Down 归因的对应关系：
- L2：`scoreboard_not_ready`
- L3：`fu_busy` / `src_not_ready` / `dest_busy_or_consumer` / `store_consumer_pending`

## 7. 参考
- 源码：`src/main/scala/Scoreboard.scala`
- 调度侧调用：`src/main/scala/TaskController.scala`
- 参数与类型：`src/main/scala/CUTEParameters.scala`
