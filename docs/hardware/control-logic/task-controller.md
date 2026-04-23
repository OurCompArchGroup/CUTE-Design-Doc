# 指令译码与调度（TaskController）
## 1. 术语说明

| 术语 | 说明 |
|------|------|
| AMU uop | 上游通过 `ygjkctrl.amuCtrl` 下发的单条控制指令 |
| DecodedAmuCtrlEntry | TC 入队后的解码条目，包含原始 `ctrl` 及读写寄存器摘要 |
| decodedFifo | 解码后队列，队首作为唯一候选发射项 |
| Scoreboard Query | 发射前准入查询，判断依赖与资源是否允许发射 |
| Scoreboard Update | 发射/完成事件更新，维护寄存器与 FU 状态 |
| pending* | TC 内部在飞任务账本，用于完成回传与状态闭环 |

## 2. 设计规格

| 参数 | 说明 |
|------|------|
| Decoded AMU FIFO 深度 | 8（`DecodedAmuCtrlFIFODepth`） |
| 发射宽度 | 1 uop/周期（`issueFire`） |
| 调度模式 | 严格队首发射（Head-of-Queue） |
| 支持 uop 类型 | `load`、`store`、`mma`、`arith(mzeroacc/mzerotr)`、`release` |
| 依赖检查方式 | `Scoreboard` 查询 + 下游 `MicroTaskReady` 联合门控 |

## 3. 功能描述

TaskController 是 XSAI 控制路径的调度核心，主要职责如下：
1. **接入与解码**：接收 `ygjkctrl.amuCtrl`，将 AMU 指令重解释为 `AmuLsuIO/AmuMmaIO/AmuArithIO/AmuReleaseIO`，并写入 `decodedFifo`。
2. **队首准入**：仅检查队首指令；通过 `scoreboard query` 与各下游 `MicroTaskReady` 共同决定 `headReady`。
3. **发射与配置**：在 `issueFire` 时，向 AML/BML/CML、ADC/BDC/CDC、MTE 或 `mrelease` 下发配置，并同步写 `scoreboard update`。
4. **完成回传**：通过各模块 `MicroTaskEndValid` 回传完成事件，更新 `scoreboard finish` 并清除 `pending*`。

## 4. 微架构设计
### 4.1 指令接入与 `DecodedAmuCtrlEntry` 摘要

TC 以 `valid-ready` 方式接收 `ygjkctrl.amuCtrl`：
- `decodedFifo.io.enq.valid := io.ygjkctrl.amuCtrl.valid`
- `io.ygjkctrl.amuCtrl.ready := decodedFifo.io.enq.ready`

每条入队项都保存：
- 原始控制字段 `ctrl: AmuCtrlIO`
- `readRegs/readValid`（最多 3 个源）
- `writeRegs/writeValid`（最多 1 个目的）

摘要规则：
- `mma`：`ms1/ms2/md` 作为源，`md` 作为目的
- `load`：`ms` 作为目的
- `store`：`ms` 作为源
- `arith`：`md` 同时作为源和目的

### 4.2 Scoreboard 准入请求构造与判定

TC 对队首 `headEntry` 生成 `scoreboardReq`（按类型填 `fuType/src/dest`）：
- `load`：`fuType` 映射到 AML/BML/CML(load)
- `mma`：`fuType=Compute`，并填 `src1/src2/src3` 与 `dest`
- `arith`：`fuType` 映射到 AML 或 CML(load)
- `store`：`fuType=CML(store)`，仅填源

队首是否可发由两类条件共同决定：
1. **相关性条件**：`scoreboardReqReady = !scoreboardReqValid || scoreboard.io.query.req.ready`
2. **下游就绪条件**：`loadUnitsReady / storeUnitsReady / mmaUnitsReady / zero*UnitsReady / releaseReady`

最终门控：
```scala
headReady = MuxCase(... per-op ...)
decodedFifo.io.deq.ready := headValid && headReady
issueFire := decodedFifo.io.deq.fire
```

### 4.3 六类 `issue*` 分支

`issueFire` 后按指令类型进入分支：

1. **issueLoad**
- 下发 AML/BML/CML load 配置（按 `needA/needB/needC`）。
- 置位 `pendingLoad*`，并触发 `scoreboard.io.update.load_allocate`。

2. **issueZeroAcc（arith -> C 路 zero-load）**
- 通过 CML 下发 zero-load 配置。
- 置位 `pendingLoadC`，并触发 `load_allocate(has_c=true)`。

3. **issueZeroTr（arith -> A 路 zero-load）**
- 通过 AML 下发 zero-load 配置。
- 置位 `pendingLoadA`，并触发 `load_allocate(has_a=true)`。

4. **issueMma**
- 同步下发 ADC/BDC/CDC 配置。
- 计算并更新 `mmaDataType`，写入 `MTE_MicroTask_Config.dataType`。
- 置位 `pendingComputeA/B/C`，触发 `scoreboard.io.update.compute_issue`。

5. **issueStore**
- 通过 CML 下发 store 配置（`IsStoreMicroTask=true`）。
- 置位 `pendingStore`，触发 `scoreboard.io.update.store_issue`。

6. **issueRelease**
- 驱动 `io.ygjkctrl.mrelease`。
- 当前不写 scoreboard，受 `releaseReady = !pendingStore` 门控。

### 4.4 `MicroTaskEndValid/Ready` 回传与清账

TC 通过 `pending*` 决定各路 `MicroTaskEndReady`，在完成时执行两件事：
1. **更新 Scoreboard**：写入 `load_finish* / compute_read_finish* / compute_write_finish_c / store_finish`
2. **清除 pending 账本**：对应 `pendingX := false.B`

关键细节：
- CML 的完成口复用：优先回收 `pendingLoadC`，否则回收 `pendingStore`。
- Compute 完成分三段：`ADC(read A)`、`BDC(read B)`、`CDC(write C)`，分别对应不同 finish 事件。

## 5. 接口信号

| 信号名 | 方向 | 说明 |
|--------|------|------|
| `ygjkctrl` | Input (Flipped) | 上游控制通道，含 `amuCtrl` 输入与 `mrelease` 输出 |
| `ADC/BDC/CDC_MicroTask_Config` | Output | Compute 路微任务配置与完成握手 |
| `AML/BML/CML_MicroTask_Config` | Output | Load/Store 路微任务配置与完成握手 |
| `MTE_MicroTask_Config` | Output | MTE 数据类型配置（`dataType`） |
| `DebugTimeStampe` | Input | 调试时间戳输入 |

## 6. 与其他模块的交互

```
YGJKControl(amuCtrl)
        │
        ▼
TaskController ──query/update── Scoreboard
     │       ├── AML/BML/CML (Load/Store)
     │       ├── ADC/BDC/CDC (Compute)
     │       └── MTE(dataType)
     └── mrelease ──> YGJKControl
```

交互特征：
- 调度边界位于 TC 队首发射点（`issueFire`）。
- Scoreboard 仅负责准入与状态跟踪，不承担数据通路搬运。
- 完成事件由各子模块返回 TC，再由 TC 统一回写 Scoreboard。

## 7. 参考
- 源码：`src/main/scala/TaskController.scala`
- 依赖检查：`src/main/scala/Scoreboard.scala`
- 控制接口：`src/main/scala/config_ygjk.scala`
- 参数定义：`src/main/scala/CUTEParameters.scala`
