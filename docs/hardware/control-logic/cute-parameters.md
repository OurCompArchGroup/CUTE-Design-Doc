# 硬件参数配置（CUTEParameters）

## 1. 概述

CUTE 的所有硬件参数集中在 `CUTEParameters.scala` 中定义，通过 `case class CuteParams` 与 `CUTEImplParameters` 派生出控制逻辑、访存路径和计算路径所需参数。

## 2. 核心参数

### 2.1 基础维度参数

| 参数 | 含义 | 默认值 | 约束 |
|------|------|--------|------|
| `Tensor_MN` | MatrixReg 保存的 M/N 维 tile 大小 | 128 | 2 的幂 |
| `Tensor_K` | MatrixReg 保存的 K 维 tile 大小（按 8-bit 元素计） | 64 | 2 的幂 |
| `Matrix_MN` | MTE 计算阵列维度（M 与 N） | 4 | 2 的幂 |

### 2.2 位宽参数

| 参数 | 含义 | 默认值 |
|------|------|--------|
| `outsideDataWidth` | 外部总线数据宽度 (bit) | 512 |
| `MemoryDataWidth` | 内存数据宽度 (bit) | 64 |
| `VectorWidth` | 向量宽度 (bit) | 256 |
| `ReduceWidthByte` | 归约宽度（字节） | 32 |
| `ResultWidthByte` | 结果宽度（字节） | 4 |

### 2.3 派生参数（自动计算）

| 参数 | 计算公式 |
|------|---------|
| `outsideDataWidthByte` | `outsideDataWidth / 8` |
| `ReduceWidth` | `ReduceWidthByte × 8` |
| `ReduceGroupSize` | `Tensor_K / ReduceWidthByte` |
| `ABMatrixRegSize` | `Tensor_MN × ReduceGroupSize × ReduceWidthByte` |
| `CMatrixRegSize` | `Tensor_MN × Tensor_MN × ResultWidthByte` |
| `ABMatrixRegNBanks` | `Matrix_MN` |
| `CMatrixRegNBanks` | `Matrix_MN` |
| `ABMatrixRegBankNEntrys` | `ABMatrixRegBankSize / ABMatrixRegEntryByteSize` |
| `CMatrixRegBankNEntrys` | `CMatrixRegBankSize / CMatrixRegEntryByteSize` |
| `DecodedAmuCtrlFIFODepth` | 8（控制队列深度） |
| `ABMatrixRegCount` | 4（实现常量） |
| `CMatrixRegCount` | 4（实现常量） |

### 2.4 系统参数

| 参数 | 含义 | 默认值 |
|------|------|--------|
| `LLCSourceMaxNum` | LLC Source ID 最大数 | 64 |
| `MemorysourceMaxNum` | Memory Source ID 最大数 | 64 |
| `MMUAddrWidth` | MMU 地址宽度 | 64 |
| `MMUDataWidth` | MMU 数据位宽（等于 `outsideDataWidth`） | 512 |
| `MMUMaskWidth` | MMU 掩码位宽（字节） | `MMUDataWidth/8` |

## 3. 性能预设配置

| 配置名 | Matrix_MN | Tensor_MN | ReduceWidthByte | 备注 |
|--------|-----------|-----------|-----------------|------|
| `CUTE_32Tops_128SCP` | 16 | 256 | 32 | 高吞吐预设 |
| `CUTE_8Tops_128SCP` | 8 | 128 | 32 | 常用预设 |
| `CUTE_2Tops` | 4 | 64 | 32 | 小规模预设 |
| `CUTE_2Tops_debug` | 4 | 64 | 32 | 含调试开关 |

## 4. 数据类型编码

XSAI 当前采用 3-bit 数据类型编码（`DataTypeBitWidth = 3`），支持 7 种计算类型：

| 编码 | 源码名称 | A 类型 | B 类型 | 累加类型 |
|------|---------|--------|--------|---------|
| 0 | `DataTypeI8I8I32` | INT8 | INT8 | INT32 |
| 1 | `DataTypeF16F16F32` | FP16 | FP16 | FP32 |
| 2 | `DataTypeBF16BF16F32` | BF16 | BF16 | FP32 |
| 3 | `DataTypeTF32TF32F32` | TF32 | TF32 | FP32 |
| 4 | `DataTypeI8U8I32` | INT8 | UINT8 | INT32 |
| 5 | `DataTypeU8I8I32` | UINT8 | INT8 | INT32 |
| 6 | `DataTypeU8U8I32` | UINT8 | UINT8 | INT32 |

宽度类别常量：
- `DataTypeWidth32 = 4`
- `DataTypeWidth16 = 2`
- `DataTypeWidth8 = 1`
- `DataTypeWidth4 = 7`

## 5. LocalMMU 任务类型

| 编码 | 名称 | 说明 |
|------|------|------|
| 0 | `AFirst` | A MemoryLoader |
| 1 | `BFirst` | B MemoryLoader |
| 2 | `CFirst` | C MemoryLoader |

`LocalMMUTaskType` 当前定义：
- `TaskTypeBitWidth = 2`
- `TaskTypeMax = 3`

## 6. 算力-带宽约束模型

控制逻辑调度与参数选择仍遵循“算力-带宽平衡”原则。对 XSAI 当前参数，关键约束集中在：
- `Matrix_MN` 决定并行计算规模；
- `ReduceWidthByte` 决定单拍归约输入带宽；
- `outsideDataWidth` 与 SourceID 容量决定访存并发上限。

工程上可通过调参对齐以下目标：
1. Compute 侧持续有任务可发（降低 `issue` 空槽）；
2. Memory 侧请求和回包无长期背压；
3. `MatrixReg` 银行带宽满足 DataController/MemoryLoader 并行访问需求。

## 7. 参考
- 源码：`src/main/scala/CUTEParameters.scala`
- 调度使用：`src/main/scala/TaskController.scala`
- 类型定义：`src/main/scala/Bundles.scala`
