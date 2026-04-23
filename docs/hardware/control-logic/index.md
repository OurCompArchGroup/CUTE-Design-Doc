# 控制逻辑

CUTE 的控制逻辑负责接收上游 AMU 指令、完成发射准入与依赖检查，并驱动 Load/Compute/Store 微任务闭环执行。

## 模块总览

![ctrl](ctrl.png)

## 导航

- [指令译码与调度](task-controller.md) — AMU uop 解码、队首发射、完成回传
- [Scoreboard 依赖检查](scoreboard.md) — 发射准入、寄存器/FU 依赖追踪与唤醒
- [硬件参数配置](cute-parameters.md) — 控制逻辑相关参数与类型定义
- [YGJK / RoCC 接口](cute2ygjk.md) — 控制通道与 TileLink 适配路径
