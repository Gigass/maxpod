# iPod 4th Gen Click Wheel 开发总参考

## 1. 目的

本文档汇总当前与 `iPod 4th gen / iPod Photo` click wheel 外接开发直接相关的核心信息，作为后续硬件设计、固件实现、BLE 适配和联调验证的统一参考。

本文档的目标不是复述所有历史资料，而是把后续开发真正需要的内容压缩成一份可直接执行的工程文档。

## 2. 核心结论

### 2.1 适用范围

本文档只针对以下对象：

- `iPod (4th generation)` click wheel
- `iPod Photo / iPod with color display` click wheel

不直接适用于：

- `iPod Video / 5th gen`
- `iPod Classic / 6th-7th gen`

### 2.2 结论摘要

基于现有公开资料，4 代 click wheel 已经具备足够的公开信息支持外接开发：

- pinout 已有可用结论
- 物理层行为已有较清晰描述
- 事件 packet 格式已有可用结论
- AVR 示例固件已验证按键和滚轮事件可读
- 后续迁移到 `ESP32-C3` 的主要工作在于：
  - GPIO / 中断重写
  - packet parser 重写
  - BLE 事件映射

## 3. 统一参考源

后续开发如需追溯原始资料，以以下三份为准：

1. Jason Garr 的早期逆向与 AVR 驱动：
   - [https://jasongarr.wordpress.com/project-pages/ipod-clickwheel-hack/](https://jasongarr.wordpress.com/project-pages/ipod-clickwheel-hack/)
2. Gigahawk 的 AVR 示例固件：
   - [https://github.com/Gigahawk/clickwheel_sample_firmware](https://github.com/Gigahawk/clickwheel_sample_firmware)
3. Gigahawk 的协议文档与逻辑分析：
   - [https://github.com/Gigahawk/clickwheel_reverse_eng](https://github.com/Gigahawk/clickwheel_reverse_eng)

## 4. 三份资料各自提供了什么

### 4.1 Jason Garr

价值：

- 最早公开了可工作的 pinout 和 AVR 驱动思路
- 证明 click wheel 可以被第三方 MCU 读取
- 给出了按键和滚轮事件的可运行代码

局限：

- 描述偏早期，部分引脚定义更接近“观察结果”，不如后续协议文档精确
- 更偏“能跑”，不是完整协议参考

### 4.2 `clickwheel_sample_firmware`

价值：

- 给出一个最小可运行的事件读取实现
- 已确认在 `4th gen / Photo` 上工作
- 说明基于事件层可以稳定识别：
  - 顺时针
  - 逆时针
  - Menu
  - Play/Pause
  - Back
  - Forward
  - Center

局限：

- 强绑定 `Arduino Mega 2560 / AVR`
- 只利用了简化的读取方式，不等于完整协议实现
- 不能直接照搬到 `ESP32-C3`

### 4.3 `clickwheel_reverse_eng`

价值：

- 给出更完整的 pinout
- 给出物理层描述
- 给出 packet 格式
- 给出 `CFG1` / `MISO` 相关行为
- 给出 sleep / wake / init 相关信息

局限：

- 明确说明研究对象是 `4th gen Photo`
- 对 5 代及以上只说“可能部分适用”，不应外推成已验证

## 5. 最终采用的 4 代 pinout 结论

### 5.1 推荐采用的 pin 定义

以 `clickwheel_reverse_eng` 为主、Jason Garr 为辅，当前工程上应采用如下定义：

| FPC Pin | 名称 | 方向 | 当前结论 |
|---|---|---|---|
| 1 | `VBAT` | 输入到 click wheel | 主供电脚，可用 3.3V 原型验证 |
| 2 | `SCK` | click wheel 输出 | 串行时钟，空闲高阻，发送时由 click wheel 驱动 |
| 3 | `CFG1` | 输入到 click wheel | 配置脚，影响发包行为 |
| 4 | `BTN1` | click wheel 输出 | Center/Menu 相关脉冲输出 |
| 5 | `UNKNOWN` | click wheel 输出？ | 目前未确认用途 |
| 6 | `MOSI` | click wheel 输出 | click wheel -> MCU 数据线，开漏 |
| 7 | `MISO` | 输入到 click wheel | MCU -> click wheel 数据线 |
| 8 | `GND` | 电源地 | 地 |

### 5.2 Jason Garr 早期描述与当前解释的关系

Jason Garr 对 8-pin 的早期描述是：

- `1` = `Vbatt`
- `2` = `clock`
- `3` = `hw enable?`
- `4/5` = `N/C`
- `6` = `serial data`
- `7` = `hw enable?`
- `8` = `GND`

结合后续协议文档，当前更合理的解释是：

- Jason 所说的 `clock` = `SCK`
- Jason 所说的 `serial data` = `MOSI`
- Jason 所说的 `hw enable?` 更可能是 `CFG1` / `MISO` 这类与配置/控制相关的脚

因此，后续硬件设计不应再把 `3` 和 `7` 当作“随便绑 3.3V 的 enable 脚”处理。

## 6. 供电与电气层结论

### 6.1 供电

已知结论：

- `VBAT` 是 click wheel 主供电脚
- 资料显示它在原机里直接连到 Li-po 电池
- 公开验证中已用 `3.3V` 成功驱动原型

工程结论：

- 第一版原型可先用 `3.3V` 供电验证
- 供电纹波和上电时序仍需实测确认

### 6.2 信号电平

已知结论：

- `SCK` 发送时为 push-pull 驱动，空闲高阻
- `MOSI` 是开漏输出
- 公开资料默认 `SCK`、`MOSI` 需要上拉到 `3.3V`

工程结论：

- `SCK`、`MOSI` 两线都应设计上拉位
- 推荐板上预留可调上拉，不要把阻值焊死在单一假设上

### 6.3 pull-up 行为

`clickwheel_reverse_eng` 的物理层结论：

- 无上拉或下拉时，`SCK` 空闲低
- 加上拉后，`SCK` 空闲高
- 在这种条件下，click wheel 可按 `SPI-like` 设备来理解

工程结论：

- 原型设计优先采用“上拉到 3.3V”的工作方式
- 固件按 `idle high` 路径实现更容易统一处理

## 7. 物理层行为

### 7.1 基本工作方式

4 代 click wheel 不是 MCU 主动发起通信的普通 SPI 从设备，而是：

- **click wheel 自己是主设备**
- 它自己产生时钟
- 它自己把事件数据送出来

这是整个项目最重要的概念之一。

### 7.2 时序要点

`clickwheel_reverse_eng` 给出的关键物理层结论：

- 发包前有一个约 `2.9us` 的 start pulse
- 之后时钟约 `32` 个脉冲
- 时钟频率约 `55kHz`
- 最后一拍略长，约 `11us`
- 其余时钟脉冲大约 `9us`

### 7.3 边沿与位序

当前可采用的工程假设：

- `CPOL = 1`
- `CPHA = 0`
- `LSB first`

也就是：

- 时钟空闲高
- 下降沿领先
- 数据按 LSB first 解析

### 7.4 固件实现含义

对 `ESP32-C3` 这意味着：

- 不要把 click wheel 当普通 SPI 外设套现成高层驱动
- 第一版更适合：
  - GPIO 中断采样 `SCK`
  - 在中断中读取 `MOSI`
  - 软件重组 32bit packet

## 8. `CFG1` 配置脚结论

### 8.1 `CFG1 = 0`

当 `CFG1 = 0` 时：

- click wheel 会持续发包
- 空闲时数据包内容为 `0xFF`
- 发包频率约 `1.465kHz`
- 有触摸/按键事件时，最快有效事件率约 `67Hz`
- 在此模式下，MCU 可以通过 `MISO` 发命令给 click wheel

### 8.2 `CFG1 = 1`

当 `CFG1 = 1` 时：

- click wheel 只在触摸事件发生时发包
- 最快约 `67Hz`

### 8.3 工程建议

第一版原型必须把 `CFG1` 留给主控或至少留可改线位，原因如下：

- 调试阶段可能需要在“持续发包”和“事件发包”之间切换
- 后续如果要进入 sleep / wake / command 路径，`CFG1` 不是可选项

## 9. `BTN1` 引脚结论

### 9.1 已知行为

`BTN1` 当前已知特性：

- 空闲低
- 当 `Center` 或 `Menu` 按下时，产生约 `29.83ms` 的高脉冲
- 脉冲约在事件数据包开始后 `15ms` 发出
- 按钮释放时不会产生对应脉冲
- 资料推测它还参与冷启动唤醒

### 9.2 工程含义

- `BTN1` 不能当作无关脚忽略
- 至少应留测试点
- 更稳妥的做法是也接入 MCU GPIO，便于验证唤醒/边界行为

## 10. `MISO` 引脚结论

### 10.1 已知用途

`MISO` 是 MCU -> click wheel 的输入线，用于：

- 在 `CFG1 = 0` 模式下发送命令
- 参与 sleep / wake / init 相关交互

### 10.2 当前阶段的作用

对第一版 BLE 原型来说，`MISO` 不是“最小闭环必须使用”的脚，但它是：

- 后续实现低功耗/状态控制的入口
- 排查“为什么这块 wheel 行为和样例不一样”的重要调试口

### 10.3 工程建议

- 第一版板子应把 `MISO` 接到 MCU GPIO 或至少预留 0R 跳线位
- 不建议省掉

## 11. 第 5 脚 `UNKNOWN` 的结论

当前公开资料对第 5 脚没有可靠功能定义。

已知观察：

- 加 `10k` 上拉到 `3.3V` 时仍然始终低
- 未发现明确切换条件
- 掉电后并不等同于直接接地

工程建议：

- 第一版板子保留测试点
- 不接入关键功能路径
- 不要擅自绑高/绑低

## 12. 协议层：Event Packet

### 12.1 包长度

每个事件包总长度为：

- `4 byte`
- 即 `32 bit`

### 12.2 第一字节

协议文档结论：

- 第一字节固定 header
- 当前已知固定值为 `0x35`

### 12.3 第二字节：按钮状态

第二字节表示按钮按下状态，已知位定义如下：

| Bit 语义 | 当前结论 |
|---|---|
| Menu | 已知 |
| Play/Pause | 已知 |
| Prev | 已知 |
| Next | 已知 |
| Center | 已知 |
| 其余位 | 未完全确认 |

协议文档给出的位图可概括为：

- `Menu`
- `Play/Pause`
- `Prev`
- `Next`
- `Center`

按下时相应位为 `1`

### 12.4 第三字节：滚轮触摸位置

第三字节表示触摸位置：

- 从顶部 `Menu` 区域开始是 `0x00`
- 顺时针每次增量 `2`
- 最大到 `0xBE`
- 共 `96` 个离散位置

工程含义：

- 这不是简单“上/下”事件，而是有实际位置值
- 如果后续需要更细腻交互，可利用这个位置值而不只是 CW/CCW

### 12.5 第四字节：触摸有效标志

第四字节表示当前触摸是否有效：

- `0x00`：没有手指接触
- `0x80`：有手指接触，第三字节位置有效

工程结论：

- 当第四字节不是触摸有效状态时，第三字节位置变化应忽略
- 按键动作引起的随机位置变化不能当作滚轮事件处理

## 13. Jason Garr / Sample Firmware 的简化事件模型

### 13.1 简化思路

Jason Garr 和 `clickwheel_sample_firmware` 的代码没有实现完整协议抽象，而是：

- 在中断里把 32bit 数据逐步右移拼起来
- 用某个固定包尾条件判断“新包到达”
- 若按钮字节非零，则直接按按钮事件处理
- 若按钮字节为零，则比较 wheel byte 与上一次位置值，推导：
  - 顺时针
  - 逆时针

### 13.2 他们已验证可识别的事件

已跑通事件：

- `CW_CMD_CW`
- `CW_CMD_CCW`
- `CW_CMD_PP`
- `CW_CMD_MENU`
- `CW_CMD_BACK`
- `CW_CMD_FORWARD`
- `CW_CMD_CBTN`

### 13.3 它的局限

这个实现的价值是“证明可用”，但不能当作完整规范：

- 它没有覆盖 `CFG1` / `MISO` 的控制行为
- 它没有严格利用 `0x35 header + touch valid` 这一层协议信息
- 它带有 AVR 和样机环境假设

工程结论：

- 可参考其中断拼包和事件分发思路
- 不建议原封不动迁移

## 14. 命令与状态控制

### 14.1 Sleep 命令

已知命令：

- `95 02 00 C0`

效果：

- click wheel 进入 sleep
- 只发送按钮事件
- 不发送触摸事件

### 14.2 唤醒与初始化

已知交互：

- CPU 唤醒后会发 `1D 01 00 C0`
- click wheel 预期回复 `75 04 02 00`
- 之后 CPU 会再发 `95 82 00 C0`

### 14.3 当前阶段的工程意义

对 BLE 原型首版：

- 可先不实现这部分
- 但硬件必须保留 `CFG1`、`MISO`，否则后面无法补做

## 15. 低功耗与唤醒

### 15.1 已知低功耗状态

公开资料提到三种相关状态：

1. `Power off`
   - 主处理器深睡
   - click wheel 可能也在 sleep
   - 只能通过 `Center` 或 `Menu` 唤醒
2. `Sleep`
   - 主处理器和 click wheel 都休眠
   - 不发送触摸事件，只发送按钮事件
3. `Hold switch`
   - click wheel 电源被切断
   - 解锁后会发一串 `AA`，表示 ready

### 15.2 当前项目的建议

第一版不以低功耗为主目标，但应保留后续验证能力：

- `BTN1`
- `CFG1`
- `MISO`

## 16. Clicker 声音

`clickwheel_reverse_eng` 还给出了一条与主项目非关键但可扩展的信息：

- clicker（机械点击声）可复现
- 直接用 GPIO 驱动声音偏小
- 通过低 `Rds(on)` MOSFET 切换 `3.3V`，声音更接近原机

当前项目结论：

- 第一版可不做
- 如果后续想做“更像 iPod 的手感反馈”，这是可扩展项

## 17. 硬件设计结论

### 17.1 第一版必须引出的脚

后续开发所需的最小“稳妥版”接口，不是只接两三根线，而是以下全部：

- `VBAT`
- `GND`
- `SCK`
- `MOSI`
- `CFG1`
- `MISO`
- `BTN1`
- `UNKNOWN` 测试点

### 17.2 建议接法

| 网络 | 第一版建议 |
|---|---|
| `VBAT` | 接可控 3.3V 原型电源 |
| `GND` | 公共地 |
| `SCK` | 接 MCU GPIO 输入，预留上拉 |
| `MOSI` | 接 MCU GPIO 输入，预留上拉 |
| `CFG1` | 接 MCU GPIO，或跳线可切高/低 |
| `MISO` | 接 MCU GPIO 输出/双向配置 |
| `BTN1` | 接 MCU GPIO 输入并留测试点 |
| `UNKNOWN` | 只留测试点 |

### 17.3 不建议的做法

- 不建议只把 `SCK + MOSI` 画进正式板而忽略其他协议脚
- 不建议把 `CFG1`、`MISO` 硬绑成固定值
- 不建议省掉测试点

## 18. 固件迁移结论

### 18.1 可直接沿用的逻辑

可沿用：

- 中断驱动采样
- 32bit packet 重组
- packet -> 事件 抽象层
- deadband 思路

### 18.2 必须重写的部分

必须重写：

- AVR 端口读写
- AVR 中断配置
- `digitalRead(19)` / `attachInterrupt()` 路径
- 对 wheel byte 的粗糙判断逻辑

### 18.3 推荐的 `ESP32-C3` 固件结构

建议拆成：

1. `clickwheel_phy`
   - GPIO 初始化
   - `SCK` 边沿中断
   - 读取 `MOSI`
   - 拼 32bit packet

2. `clickwheel_proto`
   - 校验 header 是否 `0x35`
   - 解析按钮 byte
   - 解析位置 byte
   - 解析 touch valid byte

3. `clickwheel_events`
   - 生成：
     - button press
     - touch down
     - touch move
     - touch up
     - CW
     - CCW

4. `ble_transport`
   - BLE HID 或 BLE GATT

5. `debug_dump`
   - 串口输出原始 packet 与事件

### 18.4 解析优先级建议

推荐优先级：

1. 先稳定拿到原始 4-byte packet
2. 再验证 button byte
3. 再验证 touch valid
4. 再做 wheel 位置/方向推导
5. 最后接 BLE

## 19. BLE 侧映射建议

### 19.1 第一版最合理的事件层

不要直接把一切都塞进“键盘按键模拟”。更合理的事件层是：

- `CENTER_PRESS`
- `MENU_PRESS`
- `PLAY_PAUSE_PRESS`
- `NEXT_PRESS`
- `PREV_PRESS`
- `TOUCH_DOWN`
- `TOUCH_MOVE(position)`
- `TOUCH_UP`
- `WHEEL_CW_STEP`
- `WHEEL_CCW_STEP`

### 19.2 BLE 首版建议

首版建议：

- 先做 BLE GATT 原始事件传输或简化事件传输
- 若需要快速控制 iPhone，再加 BLE HID 映射

原因：

- click wheel 有位置数据，不只是离散按键
- 先保留事件完整性，后续更灵活

## 20. 目前已知的坑

### 20.1 不要把 4 代资料外推到 5 代 / classic

`clickwheel_reverse_eng` 自己明确写了：

- 5 代及以上按键开关位于主板上
- 触摸 flex 上有更多引脚

所以：

- 4 代方案不能被当作 5 代 / classic 的现成协议文档

### 20.2 不要把示例固件当完整协议

`clickwheel_sample_firmware` 的价值是：

- 证明能读到事件

它不是：

- 最终硬件接口规范
- 完整协议实现
- 最佳 ESP32 迁移方案

### 20.3 不要忽略 `touch valid`

按键时第三字节可能出现随机位置值。

因此：

- 若第四字节不表示“触摸有效”，第三字节变化必须忽略

### 20.4 不要省略 `CFG1` / `MISO`

如果把它们省掉：

- 后面 sleep / wake / init / 兼容性验证会很被动

## 21. 当前未知与未闭合点

截至目前，仍未完全闭合的部分包括：

- 第 5 脚 `UNKNOWN` 的真实用途
- 不同 4 代 wheel 批次是否存在细微电气差异
- 3.3V 是否覆盖所有 4 代 wheel 原型样机条件
- `CFG1` 在不同上电状态下的默认最优使用模式
- `BTN1` 除 Center/Menu 外是否还有更深的系统用途

这些未知不影响首版原型启动，但必须保留调试余地。

## 22. 后续开发的硬性要求

后续开发必须遵守以下约束：

1. 原理图必须保留 `CFG1`、`MISO`、`BTN1`
2. `SCK`、`MOSI` 必须预留上拉方案
3. 固件必须先做 raw packet dump，再做高层事件
4. 不允许一开始就只写“模拟按键”的最简 BLE 逻辑
5. 每次调试都应记录：
   - `CFG1` 状态
   - pull-up 配置
   - 电源电压
   - 原始 packet

## 23. 第一版验证清单

### 23.1 硬件

- `VBAT` 上电正常
- `SCK` 可见时钟输出
- `MOSI` 可见数据变化
- `CFG1` 切换后发包行为变化符合预期
- `BTN1` 在 Center/Menu 时产生脉冲

### 23.2 协议

- 能稳定读到 `4-byte packet`
- 第一字节稳定为 `0x35`
- 第二字节能正确映射按钮
- 第三字节能反映触摸位置
- 第四字节能正确指示触摸有效状态

### 23.3 事件

- 可稳定识别：
  - Menu
  - Play/Pause
  - Prev
  - Next
  - Center
  - CW
  - CCW

### 23.4 BLE

- 能稳定把事件送到手机侧
- 不丢包到无法使用

## 24. 当前工程的最终开发建议

对于本项目，基于这三份资料，后续路线应固定为：

1. 按本文档定义的完整 4 代引脚做硬件
2. 先实现原始采样与 packet dump
3. 再做协议层 parser
4. 再做事件抽象
5. 最后做 BLE 映射

不要反过来做。

## 25. 一句话结论

4 代 click wheel 已经具备足够公开资料支持外接开发，但正确做法不是“抄一个 AVR 示例就完”，而是：

- 以 `clickwheel_reverse_eng` 为协议主参考
- 以 Jason Garr 为早期验证补充
- 以 `clickwheel_sample_firmware` 为事件层实现参考

这三者合并起来，已经足够支撑本项目后续原理图、固件和 BLE 原型开发。
