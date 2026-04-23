# YGJK / RoCC 接口

## 1. 术语说明

| 术语 | 说明 |
|------|------|
| YGJKControl | CUTE 顶层控制通道，包含 `amuCtrl` 与 `mrelease` |
| Cute2TL | TileLink 客户端封装，负责 MMU 请求到 TL 请求的适配 |
| CUTE2TLImp | `Cute2TL` 的具体实现模块，维护 SourceID busy 表与收发仲裁 |
| MMU2TLIO | CUTE 内部 MMU 与 TL 适配层之间的请求/响应接口 |
| Source ID | TileLink 并发事务标识，用于请求-响应匹配和 in-flight 跟踪 |

## 2. 设计规格

| 参数 | 说明 |
|------|------|
| 控制接口 | `YGJKControl`（`amuCtrl` 输入、`mrelease` 输出） |
| 访存适配 | `Cute2TL/CUTE2TLImp`（TL Client） |
| Source ID 空间 | `0 ~ LLCSourceMaxNum-1` |
| 请求类型 | `Get/Put`（由 `RequestType_isWrite` 选择） |
| TL 用户域 | `MatrixKey`、`AmeIndex`（按请求类型与矩阵类型赋值） |

## 3. 功能描述

CUTE 控制路径由两部分组成：
1. **控制通道**：`YGJKControl` 将 AMU 控制指令送入 TaskController，并接收 release 反馈。
2. **访存适配通道**：`Cute2TL` 将 `MMU2TLIO` 请求映射到 TileLink `A` 通道，并将 `D` 通道响应回填。

### 3.1 控制通道（`YGJKControl`）

当前实现中，控制接口字段为：
- `amuCtrl: Decoupled[AmuCtrlIO]`（输入到 TC）
- `mrelease: Valid[MreleaseIO]`（TC 回传）

在顶层连接中：
```scala
io.ctrl2top <> TaskCtrl.io.ygjkctrl
```

该路径直接驱动 TaskController 的解码与发射，不再使用原版宏指令配置寄存器流程作为现行主路径。

### 3.2 MMU 请求分发（`MMU2TLIO -> TL A`）

`CUTE2TLImp` 根据 `io.mmu.Request` 生成 TL 请求：
- `RequestType_isWrite==0`：发 `Get`
- `RequestType_isWrite==1`：发 `Put`

同时维护 TL user 字段：
- `MatrixKey`：由 `MatrixIsAcc` 与读写方向决定
- `AmeIndex`：写入当前分配的 SourceID

请求可发条件：
- `!is_full`（SourceID 未耗尽）
- 下游 `tl_out.a.ready`

### 3.3 响应回传与仲裁（`TL D` + `matrix_data_in`）

XSAI 版本支持两路响应合流：
- `tl_out.d`（TL 返回）
- `matrix_data_in`（外部矩阵数据返回通道）

仲裁策略：
- `matrix_data_in` 优先级高于 `tl_out.d`。
- 同拍同时有效时，保证 `matrix_data_in` 优先 fire，`tl_out.d` 延后。

任一路响应 fire 后：
- 根据 `source` 清除对应 `busy(source)`。
- 响应数据回填 `io.mmu.Response`。

## 4. 微架构设计

### 4.1 关键组件

| 组件 | 功能 |
|------|------|
| `YGJKControl` | 控制指令输入（`amuCtrl`）与 release 输出（`mrelease`） |
| `TaskController` | AMU uop 解码、准入、发射、完成回传 |
| `Cute2TL` | TL Client 节点封装 |
| `CUTE2TLImp` | SourceID 分配、A/D 通道收发、响应回传 |

### 4.2 Source ID 管理

`CUTE2TLImp` 使用 `busy: Vec[Bool]` 跟踪 in-flight 请求：
1. 扫描 `busy` 找空闲 ID，输出 `ConherentRequsetSourceID.bits`。
2. `tl_out.a.fire` 后置位 `busy(id)=true`。
3. `matrix_data_in.fire || tl_out.d.fire` 后清除 `busy(source)=false`。
4. `is_full = busy.reduce(_ & _)` 时不再分配新 ID。

该机制保证并发请求有界，并支持响应按 SourceID 精确回收。

## 5. 接口寄存器

当前 XSAI 实装不采用原版“配置寄存器组”作为主控制接口，主要接口如下：

| 接口/字段 | 类型 | 说明 |
|------|------|------|
| `YGJKControl.amuCtrl` | Decoupled | 上游下发 AMU 控制指令 |
| `YGJKControl.mrelease` | Valid | TC 回传 release token |
| `MMU2TLIO.Request` | Decoupled | MMU 发起读写请求 |
| `MMU2TLIO.Response` | Decoupled | MMU 接收读写响应 |
| `ConherentRequsetSourceID` | Valid | 当前可分配 coherent SourceID |

## 6. 编程模型

当前控制流程按“uop 流 + 异步完成回传”工作：
1. 上游向 `amuCtrl` 推送 AMU uop。
2. TaskController 逐条解码并发射到 load/compute/store 路径。
3. 各子模块通过 `MicroTaskEndValid` 回传完成事件，TC 更新依赖状态。
4. release 指令由 TC 通过 `mrelease` 返回上游。

访存路径由 MMU 发起，Cute2TL 负责 TL 事务适配和并发 SourceID 管理。

## 7. 与其他模块的交互

```
上游控制
   │ (YGJKControl.amuCtrl)
   ▼
TaskController ───────────► 各执行/访存子模块
   │
   └── mrelease ─────────► 上游控制

LocalMMU/MMU2TLIO ──► Cute2TL(CUTE2TLImp) ──► TileLink
                                ▲
                                └── matrix_data_in / tl_out.d 响应合流
```

## 8. 参考
- 源码：`src/main/scala/CUTE2YGJK.scala`
- 控制接口：`src/main/scala/config_ygjk.scala`
- 顶层连接：`src/main/scala/CUTETOP.scala`
