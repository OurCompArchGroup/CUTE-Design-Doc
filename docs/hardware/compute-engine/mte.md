# Matrix Tensor Engine (MTE)

## 1. 术语说明

| 术语 | 说明 |
|------|------|
| MTE | Matrix Tensor Engine，矩阵张量引擎，CUTE 的核心计算模块 |
| FPE | Floating-Point Engine，浮点运算引擎，每个 PE 的内部计算逻辑 |
| PE | Processing Element，处理单元，MTE 阵列中的基本计算节点 |
| MAC | Multiply-Accumulate，乘加运算 |
| Outer Product | 外积数据流，A 广播行方向、B 广播列方向 |
| Output-Stationary | 输出驻留数据流，累加结果驻留在 C MatrixReg |

## 2. 设计规格

| 参数 | 含义 | 默认值 |
|------|------|--------|
| `Matrix_MN` | PE 阵列行数/列数 | 4 |
| `ReduceWidth` | 每个 PE 的归约位宽 (bit) | `ReduceWidthByte × 8`（默认 256 或 512） |
| `ResultWidth` | 输出结果位宽 (bit) | 32 |

**吞吐量计算公式（n-bit 数据格式）：**

```
Throughput(n-bit) = Freq × Matrix_MN × Matrix_MN × (ReduceWidth / n) × 2
```

以默认配置为例（4×4 阵列、256-bit 归约宽度、2GHz）：

| 数据类型 | 元素位宽 | 吞吐量 |
|---------|---------|--------|
| INT8 | 8 bit | `2G × 4 × 4 × 32 × 2 = 2 TOPS` |
| FP16/BF16 | 16 bit | `2G × 4 × 4 × 16 × 2 = 1 TOPS` |
| TF32 | 19 bit | `2G × 4 × 4 × 8 × 2 = 512 GOPS` |

## 3. 功能描述

MatrixTE 是 CUTE 的核心计算模块，执行矩阵乘法的乘加运算。它接收来自 DataController 的 A/B 矩阵数据以及 C 矩阵累加值，计算 `D = A × B + C`。

**核心特征：**

- **外积数据流**：A 向量按行广播到所有 PE 行，B 向量按列广播到所有 PE 列。每个 PE 计算一个输出元素的外积累加
- **Output-Stationary**：累加结果驻留在 C MatrixReg 中，整个 K 维度遍历完毕后才写回
- **锁步执行**：所有 `Matrix_MN × Matrix_MN` 个 PE 同步运行，由 `ComputeGo` 信号驱动

**支持的操作：**
- 矩阵乘法：`D[M×N] = A[M×K] × B[K×N] + C[M×N]`

## 4. 微架构设计

### 4.1 总体结构

**PE 阵列结构：**
- `Matrix_MN × Matrix_MN` 个 FPE 实例组成二维 PE 阵列
- 每个PE 内部包含浮点路径和整数路径
- 同一行的 PE 共享相同的 A 向量，同一列的 PE 共享相同的 B 向量

**广播机制：**

| 广播维度 | 广播对象 | 广播方向 | 共享范围 |
|---------|---------|---------|---------|
| 行广播 | VectorA | 水平方向 | 同一行内所有 `Matrix_MN` 个 PE |
| 列广播 | VectorB | 垂直方向 | 同一列内所有 `Matrix_MN` 个 PE |

### 4.2 FPE 四阶段逻辑架构

| 阶段 | 功能 | 关键操作 |
|------|------|---------|
| **阶段 A（预处理与解码）** | 多格式解码；尾数乘积计算；指数预算 | FVecDecoder、12×12 乘法器 |
| **阶段 B（全局对阶与对齐）** | CmpTree 查找 MaxExp；动态右移对齐；33 路 27-bit 对齐寄存器 | CmpTree 位并行比较、27-bit 移位阵列 |
| **阶段 C（归约与累加）** | 符号注入（补码转换）；多级加法树归约；C 值合并 | 符号翻转、分组归约、全局归约、溢出位宽扩展 |
| **阶段 D（后处理与规格化）** | CLZ 归一化；移位/截位处理；异常处理；FP32/INT32 输出 | CLZ、指数调整、移位/截位、溢出/下溢判定、结果打包 |

### 4.3 数据类型支持

每个 PE 通过 3-bit opcode 选择数据类型。浮点类型进入浮点路径，整数类型进入整数路径：

| opcode | 数据类型 | 累加精度 | 输出 | 需要 Scale |
|--------|---------|---------|------|-----------|
| 0 | I8 × I8 → I32 | INT32 | INT32 | 否 |
| 1 | FP16 × FP16 → F32 | FP32 | FP32 | 否 |
| 2 | BF16 × BF16 → F32 | FP32 | FP32 | 否 |
| 3 | TF32 × TF32 → F32 | FP32 | FP32 | 否 |
| 4~6 | I8 × U8 / U8 × I8 / U8 × U8 → I32 | INT32 | INT32 | 否 |

### 4.4 块缩放数据路径

XSAI 版本不包含块缩放数据路径，MatrixTE 不再接收 ScaleA/ScaleB 输入，也不支持 MXFP/NVFP 相关计算。

## 5. 接口信号

| 信号名 | 方向 | 位宽 | 说明 |
|--------|------|------|------|
| `VectorA` | Input | `ReduceWidth × Matrix_MN` | A 矩阵输入向量（行广播） |
| `VectorB` | Input | `ReduceWidth × Matrix_MN` | B 矩阵输入向量（列广播） |
| `MatrixC` | Input | `ResultWidth × Matrix_MN × Matrix_MN` | C 累加值输入 |
| `MatrixD` | Output | `ResultWidth × Matrix_MN × Matrix_MN` | D 结果输出 |
| `dataType` | Input | 3 | 数据类型选择（`DataTypeBitWidth = 3`） |
| `ComputeGo` | Output | 1 | 计算完成握手信号 |

## 6. 与其他模块的交互

| 交互模块 | 方向 | 说明 |
|---------|------|------|
| ADataController | → VectorA | 提供每行的 A 矩阵数据 |
| BDataController | → VectorB | 提供每列的 B 矩阵数据 |
| CDataController | → MatrixC | 提供累加初值 |
| CDataController | ← MatrixD | 输出计算结果 |
| TaskController | → dataType | 配置数据类型 |


## 7. 参考

- 源码：`src/main/scala/MatrixTE.scala`
- PE 源码：`cute-fpe/fpe/src/main/scala/top/FReducePE.scala`（FPE 实现）
- 参数定义：`src/main/scala/CUTEParameters.scala`（`MTEMicroTaskConfigIO`、`ElementDataType`）
- FPE 详细设计：[reduce-pe.md](reduce-pe.md)
