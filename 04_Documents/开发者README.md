# Glovity 开发者交接 README

版本日期：2026-07-01

## 1. 交付范围

本交付包覆盖 Glovity 数据手套从硬件采集、ESP32 固件、无线/有线数据链路、测试调试工具到 MANO 可视化 SDK 的完整交接内容。软件部门提供的 MANO/SDK 主程序已原样放入 `01_MANO_Visualization_SDK/glove-sdk-release`，本次整理未修改其中任何文件。

交付包顶层结构如下：

| 路径 | 说明 |
|---|---|
| `01_MANO_Visualization_SDK/glove-sdk-release` | 可视化 MANO/SDK 主程序，可执行文件为 `glove-sdk` |
| `02_Firmware/30Hz_Product` | 正式推荐固件包，左右手 USB Text 均为 30Hz |
| `02_Firmware/5Hz_LowUSB_Debug` | 备用固件包，ESP-NOW 仍为 30Hz，手套 USB Text 降为 5Hz |
| `02_Firmware/Pure_USB_Text` | 纯 USB 文本调试固件，不启用 ESP-NOW |
| `03_Test_Debug_Tools/Local_TripleSerial_LinkTester` | 三串口链路测试工具 |
| `03_Test_Debug_Tools/Integrated_USB_MagCalibration` | 集成磁力计校准触发脚本 |
| `03_Test_Debug_Tools/Legacy_Standalone_MagCalibration` | 旧版独立磁校准固件和脚本备份 |
| `04_Documents` | 本文档及使用者文档的 Markdown/Word 版本 |
| `05_Reference` | 协议、原始 SDK 使用说明和历史参考文档 |

## 2. 硬件概况

系统由两只手套端 ESP32-S3 和一块接收器 ESP32-S3 组成。每只手套连接 11 颗 WT901/JY901S IMU，接收器负责接收左右手 ESP-NOW 数据并通过 USB 串口输出给电脑端。

当前已知设备信息：

| 设备 | 常用端口 | MAC | 作用 |
|---|---|---|---|
| 左手手套 | COM29 | `8c:fd:49:88:eb:7c` | 左手 11 路 IMU 采集和发送 |
| 右手手套 | COM26 | `8c:fd:49:88:eb:40` | 右手 11 路 IMU 采集和发送 |
| 接收器 | COM34 | `14:c1:9f:2e:79:c0` | ESP-NOW 接收，USB 输出 168 字节二进制帧 |

手套硬件引脚约定：

| 信号 | GPIO | 说明 |
|---|---:|---|
| I2C SDA | GPIO8 | IMU 总线数据 |
| I2C SCL | GPIO9 | IMU 总线时钟 |
| I2C 频率 | 400 kHz | `Wire.begin(8, 9, 400000)` |
| IMU 电源控制 | GPIO1 | 高边 P-MOS，`LOW = ON` |
| IR/辅助电源控制 | GPIO42 | 高边 P-MOS，`LOW = ON` |
| 电池 ADC | GPIO2 | VBAT 分压输入 |
| 自动关机控制 | GPIO41 | 触发时输出 HIGH 约 3s，等效长按 UOV-K20 电源键 |

IMU 地址固定为 `0x50` 到 `0x60`，共 11 路。当前 `0x50` 被视为 wrist / hand-back reference，接收器二进制帧中的 `wrist_gyro` 也来自 `0x50`。

## 3. 固件分类

### 3.1 正式推荐：30Hz Product

路径：`02_Firmware/30Hz_Product`

| 设备 | 固件目录 | 说明 |
|---|---|---|
| 左手 | `LeftHand_TX_USB30` | ESP-NOW 30Hz + USB Text 30Hz + 磁校准命令监听 |
| 右手 | `RightHand_TX_USB30` | ESP-NOW 30Hz + USB Text 30Hz + 磁校准命令监听 |
| 接收器 | `Receiver_RX` | 接收左右手 ESP-NOW，COM34 输出 168B binary @ 2000000 |

推荐烧录命令：

```powershell
$ARDUINO = "C:\Users\Bingxu_Ma\Documents\Glovity\.tools\arduino-cli\arduino-cli.exe"
$ROOT = "D:\Users\Bingxu_Ma\Desktop\Glovity\Glovity_Complete_Delivery_Package_20260701"

& $ARDUINO upload --fqbn esp32:esp32:XIAO_ESP32S3 --port COM29 "$ROOT\02_Firmware\30Hz_Product\LeftHand_TX_USB30"
& $ARDUINO upload --fqbn esp32:esp32:XIAO_ESP32S3 --port COM26 "$ROOT\02_Firmware\30Hz_Product\RightHand_TX_USB30"
& $ARDUINO upload --fqbn esp32:esp32:XIAO_ESP32S3 --port COM34 "$ROOT\02_Firmware\30Hz_Product\Receiver_RX"
```

### 3.2 备用：5Hz Low USB Debug

路径：`02_Firmware/5Hz_LowUSB_Debug`

该包内左右手 ESP-NOW 仍为 30Hz，但手套自己的 USB Text 输出降为 5Hz，可降低手套 USB 文本输出压力。接收器仍为同一个 `Receiver_RX`。

### 3.3 纯 USB Text

路径：`02_Firmware/Pure_USB_Text`

该模式只通过手套 USB 口输出文本帧，不启用 ESP-NOW，不需要接收器。适合单独排查 I2C、IMU、供电、电池读数等底层问题，不适合作为正式无线链路。

## 4. 数据链路和协议

### 4.1 正式链路

```text
LeftHand_TX_USB30 / RightHand_TX_USB30
  -> ESP-NOW unicast
  -> Receiver_RX
  -> USB COM34
  -> Linux SDK / MANO 可视化程序
```

ESP-NOW 已从广播改为单播，目标为接收器 MAC：

```text
14:c1:9f:2e:79:c0
```

单播发送成功回调用于判断无线链路是否有响应。

### 4.2 COM34 二进制帧

接收器串口参数：

```text
baud = 2000000
frame size = 168 bytes
data bits = 8
parity = none
stop bits = 1
```

168 字节帧结构：

| 偏移 | 长度 | 内容 |
|---:|---:|---|
| 0 | 2 | magic，固定 `0xAA 0x55` |
| 2 | 1 | `hand_id`，0=left，1=right |
| 3 | 1 | `imu_count`，固定 11 |
| 4 | 1 | 电池百分比 |
| 5 | 2 | `frame_id`，uint16 little-endian |
| 7 | 154 | 11 路 IMU 数据，每路 7 个 int16 |
| 161 | 6 | wrist gyro，3 个 int16 |
| 167 | 1 | 前 167 字节 XOR checksum |

每路 IMU 数据为：

```text
roll_raw, pitch_raw, yaw_raw, quat_w_raw, quat_x_raw, quat_y_raw, quat_z_raw
```

换算：

```text
angle_deg = raw / 32768.0 * 180.0
quat = raw / 32768.0
gyro_dps = raw / 32768.0 * 2000.0
```

### 4.3 手套 USB Text / 校准口

手套 USB 口串口参数：

```text
COM29 / COM26
baud = 921600
```

正式 30Hz 固件中该口输出文本调试帧，并监听磁力计校准命令：

```text
$GLOVITY,CAL,START
$GLOVITY,CAL,STOP
$GLOVITY,CAL,STATUS
$GLOVITY,NORMAL
```

## 5. 自维护和自动关机

### 5.1 低电压保护

固件每 1 秒检查一次 VBAT，每次做 8 次 ADC 采样平均。若 VBAT 连续约 8 秒低于 3.50V，则触发自动关机：

```text
打印 battery low, auto power off
停止 ESP-NOW / Wi-Fi
关闭 IR 和 IMU 负载
GPIO41 HIGH 保持约 3 秒
```

### 5.2 无接收器响应关机

该策略不依赖 USB DTR。当前判据为：

```text
最近 30 秒内没有 ESP-NOW 单播发送成功
且 VBAT 每 60 秒对比确认正在下降，下降阈值 0.01V
且上述状态连续 15 分钟
-> 执行 GPIO41 自动关机
```

如果 VBAT 没有下降，固件认为设备可能正在 USB 供电或充电，不执行无响应自动关机。但低电压保护始终有效。

## 6. 磁力计校准

推荐使用集成校准脚本，无需切换手套固件：

路径：`03_Test_Debug_Tools/Integrated_USB_MagCalibration/Host_Tester`

左手：

```powershell
python .\integrated_mag_calibrate.py --port COM29 --duration 35
```

右手：

```powershell
python .\integrated_mag_calibrate.py --port COM26 --duration 35
```

流程：

```text
按 Enter 开始
旋转手套约 35 秒
脚本自动发送 STOP
固件保存校准并回到 NORMAL
```

旧版独立 MagCalibrate 流程保留在 `03_Test_Debug_Tools/Legacy_Standalone_MagCalibration`，只作为备份和兼容用途。

## 7. 本地链路测试

路径：`03_Test_Debug_Tools/Local_TripleSerial_LinkTester/Host_Link_Tester`

推荐运行：

```powershell
python .\dualhand_link_tester.py --left COM29 --right COM26 --receiver COM34
```

该工具同时读取：

| 来源 | 端口 | 作用 |
|---|---|---|
| 左手 USB Text | COM29 @ 921600 | 源端帧号、电量、fail_mask |
| 右手 USB Text | COM26 @ 921600 | 源端帧号、电量、fail_mask |
| 接收器 binary | COM34 @ 2000000 | 无线帧率、丢包、checksum |

## 8. MANO / SDK 主程序

路径：`01_MANO_Visualization_SDK/glove-sdk-release`

该目录由软件部门提供，已原样复制。根据其原始 `使用说明.txt`，主程序为可直接运行的 Linux 可执行文件 `glove-sdk`，无需安装 Python；手部模型文件已内置，无需额外下载。运行环境需要 NVIDIA GPU + CUDA 驱动，不支持纯 CPU 运行。

### 8.1 SDK 姿态校准入口

每只手第一次使用前需要做一次三步姿态校准，约 30 秒。原始说明中的命令为：

```bash
./glove-sdk --calibrate --port /dev/ttyACM0 --side left --text
./glove-sdk --calibrate --port /dev/ttyACM0 --side right --text
```

三步动作为：

```text
1. 手掌朝下水平，手指伸直自然张开 -> Enter
2. 手腕向上转 90 度，指尖朝上 -> Enter
3. 手腕向下转 90 度，指尖朝下 -> Enter
```

校准完成后，程序目录会生成：

```text
imu_calibration_left.npy
imu_calibration_right.npy
```

### 8.2 SDK 可视化入口

原始说明中的 Text 模式启动命令为：

```bash
./glove-sdk --port /dev/ttyACM0 --side left --text
./glove-sdk --port /dev/ttyACM0 --side right --text
```

指定校准文件：

```bash
./glove-sdk --port /dev/ttyACM0 --side right --text --calib imu_calibration_right.npy
```

主要命令行参数：

| 参数 | 说明 |
|---|---|
| `--port` | 串口设备，默认 `/dev/ttyACM0` |
| `--side left/right` | 左手或右手 |
| `--text` | USB 直连文本模式 |
| `--calib` | 指定校准文件 |
| `--baud` | Text 模式默认 921600 |
| `--calibrate` | 进入 SDK 姿态校准 |
| `--out` | 校准文件输出路径 |

程序界面左侧为 3D 手部骨骼实时渲染，支持鼠标旋转、缩放、平移；右侧为参数控制面板，包括 `Flip Direction`、`Coordinate Preset`、`DIP Coupling`、`Finger Splay`、`Wrist Yaw Fix`、`Reset Parameters` 和 `FPS`。

### 8.3 SDK 原始磁力计入口的处理

原始 `使用说明.txt` 中的 `./glove-sdk --calibrate-mag --port /dev/ttyACM0` 是旧版流程，要求先烧录 SDK 目录内的 `esp32/MagCalibrate`，校准后再恢复 `esp32/IMU_Serial_Direct`。这与当前产品固件设计不一致。

新版产品固件已在左右手主程序中集成 USB 触发磁力计校准，不需要用户手动切换固件。因此：

| 场景 | 推荐做法 |
|---|---|
| 产品/生产/演示校准 | 使用 `03_Test_Debug_Tools/Integrated_USB_MagCalibration` |
| 兼容旧 SDK 调试 | 可参考 `03_Test_Debug_Tools/Legacy_Standalone_MagCalibration` |
| SDK 后续统一入口 | 建议把 `--calibrate-mag` 改为发送 `$GLOVITY,CAL,START/STOP`，而不是要求切换固件 |

## 9. 交接注意事项

- 当前正式固件以 `02_Firmware/30Hz_Product` 为准。
- 不建议混用 SDK 原始 `esp32` 目录中的旧固件和当前产品 TX/RX 固件。
- COM34 只输出二进制，不输出文本；串口工具看到乱码是正常现象。
- 手套 USB Text 口可读文本，也可发送磁校准命令。
- 若更换接收器硬件，必须同步更新 TX 固件中的接收器 MAC。
- 若更改电池分压电阻，必须同步修改 `VBAT_DIVIDER_RATIO` 和自动关机阈值验证。
