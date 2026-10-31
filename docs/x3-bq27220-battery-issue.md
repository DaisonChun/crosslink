> 提交说明：CrossLink 的 GitHub Issues 已关闭，因此通过 PR 提交问题报告。
> 这不是代码改动，只是把复现、根因和建议修复整理成文档，方便作者直接查看。
> 如果更合适，也可以移到 Gitee issue 或直接关闭。

# CrossLink Pixel 4.2.1（X3）电量显示到 80% 直接关机

## 目录

1. [现象](#1-现象)
2. [环境](#2-环境)
3. [根因](#3-根因)
4. [为什么官方固件正常](#4-为什么官方固件正常)
5. [建议修复](#5-建议修复)
6. [参考实现](#6-参考实现)
7. [验证方法](#7-验证方法)
8. [附录：4.2.1 固件检查结果](#8-附录421-固件检查结果)

## 1. 现象

阅星瞳 X3 刷 CrossLink Pixel 4.2.1 后，电量从 100% 正常往下走，但到 **80% 左右会直接关机**，不是逐渐掉到 0%。

- 续航时间本身正常，说明电池容量没有明显衰减。
- 关机后充电能正常充进去，说明不是电池保护板锁死。
- 官方固件下不会这样，只有第三方固件出现。

## 2. 环境

| 项目 | 值 |
|---|---|
| 设备 | 阅星瞳 X3（XTEINK X3） |
| SoC | ESP32-C3 |
| 固件 | CrossLink Pixel 4.2.1，X3 beta 包 `firmware-x3-beta-v4.2.1.bin` |
| 电量计 | TI BQ27220，I2C 地址 `0x55` |
| I2C | SDA GPIO20，SCL GPIO0，400 kHz |
| 电芯 | 单节锂电，约 650 mAh（官方 CEDV profile 用 642 mAh） |

## 3. 根因

BQ27220 出厂数据内存里的 **Design Capacity 默认是 3000 mAh**。X3 实际电芯只有约 650 mAh。

电量计按 3000 mAh 的标尺计算 SOC：

```text
显示 SOC ≈ 剩余容量 / 3000 mAh × 100%
实际可用容量 ≈ 650 mAh
650 / 3000 ≈ 21.7%
```

所以当电芯真正放空、电压到达截止点时，电量计仍然认为只用了约 21.7%，屏幕还显示 **约 78–80%**。设备随后因为欠压保护直接关机，看起来就是“80% 没电”。

关键点：

- BQ27220 的 SOC 主要跟 **Full Charge Capacity（学习容量，数据地址 `0x929D`）** 走，不只是 Design Capacity。
- 只改 `0x929F`（Design Capacity）而不改 `0x929D`，电量计仍可能按旧的 3000 mAh 学习值计算，现象不变。
- BQ27220 的 data memory 是 **易失的 RAM**，每次断电/重启都会回到默认 3000 mAh，所以必须在每次启动时重新写入，不能只写一次。

## 4. 为什么官方固件正常

官方 X3 固件在启动时会调用 `battery_bq27220_configure`，通过解封、进入 CFG_UPDATE、写数据内存的方式，给 BQ27220 写入一套 642 mAh 的 CEDV 配置，包括：

- FullChargeCapacity：642 mAh
- DesignCapacity：642 mAh
- EDV0/EDV1/EDV2：3210 / 3150 / 3100 mV
- OCV–SOC 曲线、C0/R0/T0 等 CEDV 参数

因此官方固件的 SOC 和实际电量基本一致。

CrossLink 目前的 `HalPowerManager::getBatteryPercentage()` 只做了一件事：

```cpp
// 读 0x2C StateOfCharge
readReg16(0x55, 0x2C, soc);
```

它没有在启动时给 BQ27220 写容量配置，于是电量计一直用 3000 mAh 默认值。

## 5. 建议修复

在 `lib/hal/HalPowerManager.cpp` 的 I2C 初始化之后，增加一个 **BQ27220 容量初始化状态机**，在 `getBatteryPercentage()` 被轮询时推进。不要阻塞启动流程，因为解封密钥之间需要约 1.5 秒间隔。

推荐流程：

1. 读 `DesignCapacity()`（命令 `0x3C`）和 `OperationStatus()`（`0x3A`）。
2. 如果 DesignCapacity 已经是 650 mAh 且已上锁，直接跳过。
3. 如果处于 CFG_UPDATE（OperationStatus bit 10），说明上次写到一半，先退出并重新上锁。
4. 解封：
   - `Control(0x0414)`，等待 1.5 s
   - `Control(0x3672)`，等待 1.5 s
5. 进入 FULL ACCESS：
   - `Control(0xFFFF)`
   - `Control(0xFFFF)`
6. 进入 CFG_UPDATE：
   - `Control(0x0090)`
7. 写两个数据内存参数：
   - Learned FCC：`0x929D` = 650
   - Design Capacity：`0x929F` = 650
8. 退出 CFG_UPDATE 并重新初始化：
   - `Control(0x0091)`
9. 上锁：
   - `Control(0x0030)`
10. 读回 `0x3C` 和 `0x12`，确认都是 650 mAh。

写数据内存时的校验和规则：

```text
向 0x3E 写地址低字节，向 0x3F 写地址高字节
向 0x40..0x5F 写 32 字节数据
向 0x60 写校验和：
checksum = 0xFF - (addr_lsb + addr_msb + data[0] + ... + data[31])
```

注意：读取 `0x60` 会让 BQ27220 切到下一个数据块，所以写完校验和前要重新向 `0x3E/0x3F` 写一次目标地址。Tenor 的实现里专门处理了这个坑。

## 6. 参考实现

Tenor 的 freeink-sdk 已经实现了完整流程，可以直接参考或移植：

- 文件：`libs/hardware/BatteryMonitor/src/BatteryMonitor.cpp`
- 分支：`dev/v1.1.1` 或 `release/v1.1.1`
- 函数：`bq27220LoadStep()`、`bq27220WriteParam()`、`bq27220TooHigh()`
- 板级配置：`designCapacityMah = 650`（XTEINK_X3）

链接：

- <https://github.com/TenorGroup/freeink-sdk/blob/dev/v1.1.1/libs/hardware/BatteryMonitor/src/BatteryMonitor.cpp>
- <https://github.com/crosspoint-reader/crosspoint-reader/issues/3752>
- <https://gist.github.com/CrazyCoder/1c5f846adee18e21f91e264601a6ddce>

Tenor 的实现有几个值得直接复用的细节：

- 解封密钥间隔 1.5 秒，避免 X3 的电量计漏掉连续写入。
- 先写 Learned FCC，隔 0.5 秒再写 Design Capacity。
- 写入前先比较旧值，已经是 650 mAh 就跳过。
- 如果上次写到一半（CFG_UPDATE 仍为 1），先退出并重新上锁。
- 用状态机分步执行，不阻塞启动。

## 7. 验证方法

修复后建议这样验证：

1. 充满电，开机，确认 `0x3C` 和 `0x12` 读回都是 650 mAh。
2. 正常阅读到自动关机，观察电量是否能走到 5% 以下再关机。
3. 第一次修复后电量可能会跳变一次，这是电量计重新初始化导致的，之后应跟随实际电量。
4. 如果仍然在 80% 关机，检查是否只写了 `0x929F` 而漏了 `0x929D`，或者写入后没有执行 `0x0091` 重新初始化。

## 8. 附录：4.2.1 固件检查结果

对 `firmware-x3-beta-v4.2.1.bin` 的检查结果：

- UI 版本字符串为 `4.2.1`。
- 启动日志仍为 `Starting CrossPoint version 20260617-x3`，说明底层还是 2026-06-17 的构建。
- 4.2.1 更新日志只有：
  - 主页面新增双排书架主题
  - 支持查看书籍简介和书籍下架功能
  - 支持下载系统字体扩充包
- 在固件中未发现 BQ27220 容量写入所需的解封序列 `0x0414` / `0x3672`，也未发现 `0x929D` / `0x929F` 的数据内存写入路径。
- 因此判断 4.2.1 仍然只读取 `0x2C` 的 SOC，没有做容量初始化。

这不是电池硬件故障，也不是校准一次就能解决的问题；需要固件在每次启动时把 650 mAh 写入 BQ27220。
