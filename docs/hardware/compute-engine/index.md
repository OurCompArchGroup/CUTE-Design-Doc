# 计算引擎

CUTE 的计算引擎由 MatrixTE（矩阵张量引擎）及其内部的 FPE（浮点运算引擎）组成，采用外积数据流与浮点/整数双路径，支持 opcode `0~6` 的 7 种数据类型。

## 模块总览

![compute 整体架构](compute.png)

## 导航

- [Matrix Tensor Engine (MTE)](mte.md) — PE 阵列结构、外积数据流、矩阵接口、FPE 四阶段总览
- [FPE 浮点运算引擎](reduce-pe.md) — Pipe0~Pipe4 流水、浮点/整数路径、CmpTree 指数比较、27-bit 对齐、归约与后处理
