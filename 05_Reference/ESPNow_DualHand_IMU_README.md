# ESPNow_DualHand_IMU 技术说明

本文档说明 `ESPNow_DualHand_IMU` 工程的两种数据获取方式：

1. USB Text 直插模式：手套 ESP32-S3 通过 USB 直接连电脑，以文本行输出 IMU 数据，适合调试。
2. ESP-NOW Binary 无线模式：左右手通过 ESP-NOW 发给接收器，接收器通过 USB 输出固定 168 字节二进制帧，适合正式使用。

## 1. 工程结构

```text
D:\Users\Bingxu_Ma\Desktop\Glovity\ESPNow_DualHand_IMU
  LeftHand_TX\LeftHand_TX.ino              # 左手无线发射，hand_id=0
  RightHand_TX\RightHand_TX.ino            # 右手无线发射，hand_id=1
  LeftHand_TX_USB30\LeftHand_TX_USB30.ino  # 左手无线发射 + USB Text 30 Hz 副本
  RightHand_TX_USB30\RightHand_TX_USB30.ino# 右手无线发射 + USB Text 30 Hz 副本
  Receiver_RX\Receiver_RX.ino              # ESP-NOW 接收器，COM34 输出 168 字节二进制帧
  LeftHand_USB_Text\LeftHand_USB_Text.ino  # 左手 USB 直插文本输出
  RightHand_USB_Text\RightHand_USB_Text.ino# 右手 USB 直插文本输出
  README.md
```

当前使用 Arduino-ESP32 编译，FQBN：

```text
esp32:esp32:XIAO_ESP32S3
```

注意：硬件是自研 ESP32-S3 板，这里主要借用 XIAO ESP32S3 的 Arduino core 配置。

## 当前烧录状态

截至 2026-06-30，当前三块板的状态如下：

| 设备 | 端口 | MAC | 当前固件 | 状态 |
|---|---:|---|---|---|
| 左手 | COM29 | `8c:fd:49:88:eb:7c` | `LeftHand_TX_USB30` | 已烧录并短测通过 |
| 右手 | COM26 | `8c:fd:49:88:eb:40` | `RightHand_TX_USB30` | 已烧录并短测通过 |
| 接收器 | COM34 | `14:c1:9f:2e:79:c0` | `Receiver_RX` | 输出 168 字节二进制帧 |

当前手套 TX 固件使用 ESP-NOW 单播发送到接收器 MAC `14:c1:9f:2e:79:c0`，发送回调成功即视为无线链路有响应。

USB30 固件短测结果：

```text
wireless_left   ~30.43 Hz  loss=0.00%  bat=92%
wireless_right  ~30.43 Hz  loss=0.00%  bat=100%
usb_left        ~30.09 Hz  fail_mask=0x000
usb_right       ~29.95 Hz  fail_mask=0x000
```

## 2. 两种模式对比

| 模式 | 手套端程序 | 接收器程序 | 数据路径 | 电脑端格式 | 波特率 | 用途 |
|---|---|---|---|---|---:|---|
| USB Text 直插 | `LeftHand_USB_Text` / `RightHand_USB_Text` | 不需要 | 手套 USB 直连电脑 | 文本行 | 921600 | 纯直插调试、肉眼查看 |
| ESP-NOW Binary 无线 | `LeftHand_TX` / `RightHand_TX` | `Receiver_RX` | 手套 ESP-NOW -> 接收器 USB | 168 字节二进制帧 | 2000000 | 正式采集 |
| 整合模式 | `LeftHand_TX` / `RightHand_TX` | `Receiver_RX` | ESP-NOW 无线 + 手套 USB 文本 | COM34 二进制，手套 USB 文本 | COM34:2000000，手套:921600 | 当前推荐 |
| 整合 USB30 副本 | `LeftHand_TX_USB30` / `RightHand_TX_USB30` | `Receiver_RX` | ESP-NOW 无线 + 手套 USB 文本 | COM34 二进制，手套 USB 文本 | COM34:2000000，手套:921600 | 需要手套 USB 也输出 30 Hz 时使用 |

当前 `LeftHand_TX` 和 `RightHand_TX` 已经是整合版：它们会持续以 30 Hz 发送 ESP-NOW，同时在手套自己的 USB 串口以 5 Hz 输出 Text 调试帧。接收器 `COM34` 仍然只输出 168 字节二进制帧，不会混入文本。

`LeftHand_TX_USB30` 和 `RightHand_TX_USB30` 是整合版副本：ESP-NOW 仍为 30 Hz，USB Text 也改为 30 Hz。它适合直接观察手套 USB 满帧率输出，但文本量更大，可能增加 USB 串口和上位机显示压力。

独立的 `LeftHand_USB_Text` / `RightHand_USB_Text` 仍然保留，用于完全不启用 ESP-NOW 的纯直插调试。若把手套烧成独立 USB Text 程序，接收器将收不到该手套的 ESP-NOW 数据。

## 3. 共同硬件定义

左右手手套板使用相同引脚：

| 信号 | GPIO | 说明 |
|---|---:|---|
| I2C SDA | GPIO8 | 连接 IMU 总线 |
| I2C SCL | GPIO9 | 连接 IMU 总线 |
| I2C 频率 | 400 kHz | `Wire.begin(8, 9, 400000)` |
| IMU 电源控制 | GPIO1 | 高边 P-MOS，`LOW = ON` |
| IR 电源控制 | GPIO42 | 高边 P-MOS，`LOW = ON` |
| 自动关机控制 | GPIO41 | 低电压保护触发时输出 `HIGH` 约 3 s，等效长按 UOV-K20 电源键 |
| 电池电压 ADC | GPIO2 | VBAT 分压输入 |

IMU 地址固定为 11 个：

```text
0x50, 0x51, 0x52, 0x53, 0x54, 0x55, 0x56, 0x57, 0x58, 0x59, 0x60
```

电池分压：

```text
VBAT - 10k - GPIO2 - 22k - GND
```

代码中的比例：

```cpp
#define VBAT_DIVIDER_RATIO 1.454545f
```

电量百分比为线性估算：

```text
bat_percent = constrain((VBAT - 3.30) / (4.20 - 3.30) * 100, 0, 100)
```

该百分比不是精确锂电 SOC，仅用于状态显示和低电量提示。

低电压自动关机直接使用 VBAT 电压判断，不依赖百分比：

```text
threshold = 3.50 V
check interval = 1 s
sample average = 8 samples
hold time = 8 s continuous below threshold
shutdown pulse = GPIO41 HIGH for 3 s
```

触发后固件会打印 `battery low, auto power off`，随后停止 ESP-NOW/Wi-Fi，关闭 IMU 和 IR 负载，再将 `POWER_CTRL = GPIO41` 拉高保持约 3 秒，由板上 AO3400A 拉低 UOV-K20 `KEY`，等效长按电源键完成关机。

无接收器响应自动关机：

```text
wireless alive = 最近 30 s 内 ESP-NOW 单播发送回调成功
battery decreasing = 每 60 s 对比一次 VBAT，下降超过 0.01 V
power off = 电池正在下降 且 wireless alive=false 连续 15 min
```

如果 VBAT 没有下降，固件认为手套大概率处于 USB 供电/充电状态，不执行“无响应自动关机”。但低电压保护不受这个条件影响：只要 VBAT 连续低于 3.50 V，仍会自动关机。

## 4. 模式一：USB Text 直插模式

### 4.1 使用方式

把对应手套直接通过 USB 接到电脑，并烧录：

| 手套 | 程序 | hand_id |
|---|---|---:|
| 左手 | `LeftHand_USB_Text` | 0 |
| 右手 | `RightHand_USB_Text` | 1 |

串口参数：

```text
Baud: 921600
Data bits: 8
Parity: None
Stop bits: 1
Flow control: None
```

Arduino Serial Monitor 或其他串口工具选择手套所在 COM 口，波特率设为 `921600`。

### 4.2 Text 模式输出频率

目标输出频率约 30 Hz：

```cpp
#define PRINT_HZ 30
#define PRINT_INTERVAL_US (1000000UL / PRINT_HZ)
```

由于文本输出量较大，实际频率会受串口工具、USB 缓冲和系统调度影响。

### 4.3 Text 模式每帧格式

每帧以固定帧头开始：

```text
==== IMU Direct Reader
Frame: 123  Bat: 85%
[0x50] Roll:   -29.85 Pitch:   -27.34 Yaw:   133.64 Q:    0.4255    0.1114   -0.3199    0.8391
[0x51] Roll:   153.05 Pitch:     3.99 Yaw:  -111.98 Q:   -0.1021   -0.5503    0.8010    0.2122
...
[0x60] Roll:    10.50 Pitch:    -2.20 Yaw:    90.10 Q:    0.9980    0.0010    0.0300    0.0500
```

帧头字段：

| 文本 | 说明 |
|---|---|
| `==== IMU Direct Reader` | 一帧开始 |
| `Frame: xxx` | 手套本机帧号，从 0 递增 |
| `Bat: xx%` | 电池百分比 |

每个 IMU 行：

```text
[0xADDR] Roll: <deg> Pitch: <deg> Yaw: <deg> Q: <w> <x> <y> <z>
```

若读取失败：

```text
[0x56] Read failed
```

### 4.4 Text 模式数据来源

欧拉角读取 WT901/JY901S 寄存器：

```cpp
#define REG_ROLL 0x3D
```

连续读取：

```text
Roll, Pitch, Yaw
```

四元数读取寄存器：

```cpp
#define REG_Q0 0x51
```

连续读取：

```text
Q0, Q1, Q2, Q3
```

输出顺序：

```text
Q: w x y z
```

缩放：

```text
euler_deg = raw / 32768.0 * 180.0
quat_float = raw / 32768.0
```

### 4.5 Text 模式烧录命令

左手：

```powershell
& 'D:\Users\Bingxu_Ma\AppData\Local\Programs\Arduino IDE\resources\app\lib\backend\resources\arduino-cli.exe' upload --fqbn esp32:esp32:XIAO_ESP32S3 --port COM29 'D:\Users\Bingxu_Ma\Desktop\Glovity\ESPNow_DualHand_IMU\LeftHand_USB_Text'
```

右手：

```powershell
& 'D:\Users\Bingxu_Ma\AppData\Local\Programs\Arduino IDE\resources\app\lib\backend\resources\arduino-cli.exe' upload --fqbn esp32:esp32:XIAO_ESP32S3 --port COM26 'D:\Users\Bingxu_Ma\Desktop\Glovity\ESPNow_DualHand_IMU\RightHand_USB_Text'
```

## 5. 模式二：ESP-NOW Binary 无线/整合模式

### 5.1 角色

| 角色 | 当前端口 | 程序 | hand_id | 功能 |
|---|---:|---|---:|---|
| 左手 | COM29 | `LeftHand_TX` | 0 | 读取 11 个 IMU，30 Hz ESP-NOW 发送，同时 USB 5 Hz 文本输出 |
| 右手 | COM26 | `RightHand_TX` | 1 | 读取 11 个 IMU，30 Hz ESP-NOW 发送，同时 USB 5 Hz 文本输出 |
| 左手 USB30 副本 | COM29 | `LeftHand_TX_USB30` | 0 | 读取 11 个 IMU，30 Hz ESP-NOW 发送，同时 USB 30 Hz 文本输出 |
| 右手 USB30 副本 | COM26 | `RightHand_TX_USB30` | 1 | 读取 11 个 IMU，30 Hz ESP-NOW 发送，同时 USB 30 Hz 文本输出 |
| 接收器 | COM34 | `Receiver_RX` | N/A | 接收 ESP-NOW，USB 输出 168 字节二进制帧 |

手套端整合版宏：

```cpp
#define USB_TEXT_ENABLED 1
#define USB_TEXT_BAUD 921600
#define USB_TEXT_HZ 5
```

USB30 副本只修改：

```cpp
#define USB_TEXT_HZ 30
```

如果正式采集时不想让手套 USB 输出文本，可把 `USB_TEXT_ENABLED` 改为 0 后重新编译烧录。这样不会改变 COM34 的二进制协议。

### 5.1.1 手套端磁力计校准命令

当前 `LeftHand_TX` / `RightHand_TX` / `LeftHand_TX_USB30` / `RightHand_TX_USB30` 已集成磁力计校准维护模式。手套默认上电进入正常模式，继续 30 Hz ESP-NOW 回传；电脑端可通过手套自己的 USB 串口发送文本命令临时切换到校准。

串口参数：

```text
Port: 左手 COM29 / 右手 COM26，按实际枚举为准
Baud: 921600
Line ending: \n 或 \r\n
```

命令：

```text
$GLOVITY,CAL,START   进入磁力计校准模式，暂停正常 ESP-NOW/USB Text 回传
$GLOVITY,CAL,STOP    停止校准、保存 0x50~0x60 的校准结果，然后回到正常模式
$GLOVITY,CAL,STATUS  查询当前是否正在校准
$GLOVITY,NORMAL      若正在校准则停止并保存，否则确认正常模式
```

短命令也可用：

```text
CAL,START
CAL,STOP
CAL,STATUS
NORMAL
```

典型流程：

```text
1. 打开手套 USB 串口。
2. 发送 $GLOVITY,CAL,START。
3. 看到 $GLOVITY,CAL,STARTED 后，缓慢旋转手套 30~35 秒，覆盖各方向。
4. 发送 $GLOVITY,CAL,STOP。
5. 看到 0x50~0x60 全部 SAVE OK 和 $GLOVITY,CAL,COMPLETE。
6. 固件自动恢复正常 30 Hz 回传。
```

成功输出示例：

```text
$GLOVITY,CAL,SCAN_DONE,<active>,11
$GLOVITY,CAL,CALSW,0x50,0x07,OK
...
$GLOVITY,CAL,STARTED
$GLOVITY,CAL,MAG,12,0x50:...,0x51:...,...
$GLOVITY,CAL,CALSW,0x50,0x00,OK
...
$GLOVITY,CAL,SAVE,0x60,OK
$GLOVITY,CAL,COMPLETE
$GLOVITY,MODE,NORMAL
```

校准模式期间手套不会继续发送正常 ESP-NOW 帧，避免 Linux SDK 把校准过程中的姿态扰动当作正常采集数据。接收器 `COM34` 的 168 字节二进制协议未改动。

电脑端 `integrated_mag_calibrate.py` 会按 `SCAN_DONE` 中的 active IMU 数量检查 `CALSW=0x07`、`CALSW=0x00` 和 `SAVE` 是否全部成功；完整手套仍应看到 active 为 11。

### 5.1.2 USB30 副本烧录命令

左手 COM29：

```powershell
& 'D:\Users\Bingxu_Ma\AppData\Local\Programs\Arduino IDE\resources\app\lib\backend\resources\arduino-cli.exe' upload --fqbn esp32:esp32:XIAO_ESP32S3 --port COM29 'D:\Users\Bingxu_Ma\Desktop\Glovity\ESPNow_DualHand_IMU\LeftHand_TX_USB30'
```

右手 COM26：

```powershell
& 'D:\Users\Bingxu_Ma\AppData\Local\Programs\Arduino IDE\resources\app\lib\backend\resources\arduino-cli.exe' upload --fqbn esp32:esp32:XIAO_ESP32S3 --port COM26 'D:\Users\Bingxu_Ma\Desktop\Glovity\ESPNow_DualHand_IMU\RightHand_TX_USB30'
```

若要使用已编译好的缓存产物，也可以指定 `--input-dir`：

```powershell
& 'D:\Users\Bingxu_Ma\AppData\Local\Programs\Arduino IDE\resources\app\lib\backend\resources\arduino-cli.exe' upload --fqbn esp32:esp32:XIAO_ESP32S3 --port COM29 --input-dir 'D:\Users\Bingxu_Ma\AppData\Local\Temp\codex-arduino-build\DualHand_Left_TX_USB30' 'D:\Users\Bingxu_Ma\Desktop\Glovity\ESPNow_DualHand_IMU\LeftHand_TX_USB30'
& 'D:\Users\Bingxu_Ma\AppData\Local\Programs\Arduino IDE\resources\app\lib\backend\resources\arduino-cli.exe' upload --fqbn esp32:esp32:XIAO_ESP32S3 --port COM26 --input-dir 'D:\Users\Bingxu_Ma\AppData\Local\Temp\codex-arduino-build\DualHand_Right_TX_USB30' 'D:\Users\Bingxu_Ma\Desktop\Glovity\ESPNow_DualHand_IMU\RightHand_TX_USB30'
```

### 5.2 ESP-NOW 设置

三端固定使用：

```cpp
#define ESPNOW_CHANNEL 1
```

初始化：

```cpp
WiFi.mode(WIFI_STA);
WiFi.disconnect();
esp_wifi_set_channel(ESPNOW_CHANNEL, WIFI_SECOND_CHAN_NONE);
esp_now_init();
```

左右手使用广播地址：

```text
FF:FF:FF:FF:FF:FF
```

因此接收器不需要提前配置左右手 MAC。

### 5.3 无线发送频率

左右手各自 30 Hz：

```cpp
#define SEND_HZ 30
#define SEND_INTERVAL_US (1000000UL / SEND_HZ)
```

实测正常时：

```text
hand_id=0: 29.98 Hz
hand_id=1: 29.98 Hz
```

### 5.4 ESP-NOW 内部包格式

ESP-NOW 内部包为 `RadioPacket`：

```cpp
struct __attribute__((packed)) RadioPacket {
  uint32_t magic;
  uint16_t fail_mask;
  SerialFrame frame;
};
```

字段：

| 字段 | 长度 | 说明 |
|---|---:|---|
| `magic` | 4 | 固定 `0x48444D49` |
| `fail_mask` | 2 | IMU 失败位图，仅接收器内部使用 |
| `frame` | 168 | 最终输出给电脑的串口帧 |

总长度：

```text
4 + 2 + 168 = 174 bytes
```

`fail_mask` 位定义：

| bit | IMU 地址 |
|---:|---:|
| bit0 | 0x50 |
| bit1 | 0x51 |
| bit2 | 0x52 |
| bit3 | 0x53 |
| bit4 | 0x54 |
| bit5 | 0x55 |
| bit6 | 0x56 |
| bit7 | 0x57 |
| bit8 | 0x58 |
| bit9 | 0x59 |
| bit10 | 0x60 |

示例：

```text
0x000: 11 个 IMU 都成功
0x040: 0x56 失败
0x401: 0x50 和 0x60 失败
```

注意：`fail_mask` 不输出给电脑，接收器只用它控制状态灯。

## 6. COM34 Binary 串口帧格式

接收器收到合法 ESP-NOW 包后执行：

```cpp
Serial.write((const uint8_t *)&frame, sizeof(frame));
```

因此 COM34 输出纯二进制流，不包含文本、换行或时间戳。

串口参数：

```text
Port: COM34
Baud: 2000000
Data bits: 8
Parity: None
Stop bits: 1
Flow control: None
```

### 6.1 总帧长

每帧固定：

```text
168 bytes
```

结构体：

```cpp
struct __attribute__((packed)) SerialFrame {
  uint8_t magic[2];
  uint8_t hand_id;
  uint8_t imu_count;
  uint8_t bat_percent;
  uint16_t frame_id;
  int16_t imu[11][7];   // euler[3], quat[4]
  int16_t wrist_gyro[3];
  uint8_t checksum;
};
```

`packed` 表示无 padding。所有多字节数均为 little-endian。

### 6.2 字节偏移表

| 字节偏移 | 长度 | 类型 | 内容 | 说明 |
|---:|---:|---|---|---|
| 0 | 1 | uint8 | 0xAA | magic[0] |
| 1 | 1 | uint8 | 0x55 | magic[1] |
| 2 | 1 | uint8 | hand_id | 0=左手，1=右手 |
| 3 | 1 | uint8 | imu_count | 固定 11 |
| 4 | 1 | uint8 | bat_percent | 电池百分比 |
| 5 | 2 | uint16 LE | frame_id | 小端，0-65535 循环 |
| 7 | 154 | int16 LE[77] | IMU 数据 | 11 路，每路 7 个 int16 |
| 161 | 6 | int16 LE[3] | wrist_gyro | 当前来自 0x50 |
| 167 | 1 | uint8 | checksum | 前 167 字节 XOR |

总计：

```text
2 + 1 + 1 + 1 + 2 + 154 + 6 + 1 = 168
```

### 6.3 IMU 数据区

IMU 数据区：

```text
offset 7 - 160
```

每个 IMU 占 14 bytes：

```text
3 * int16 euler + 4 * int16 quaternion = 7 * 2 = 14 bytes
```

IMU 地址顺序：

| index | 地址 | 起始偏移 |
|---:|---:|---:|
| 0 | 0x50 | 7 |
| 1 | 0x51 | 21 |
| 2 | 0x52 | 35 |
| 3 | 0x53 | 49 |
| 4 | 0x54 | 63 |
| 5 | 0x55 | 77 |
| 6 | 0x56 | 91 |
| 7 | 0x57 | 105 |
| 8 | 0x58 | 119 |
| 9 | 0x59 | 133 |
| 10 | 0x60 | 147 |

公式：

```text
imu_base = 7 + index * 14
```

每路内部偏移：

| 相对偏移 | 类型 | 内容 |
|---:|---|---|
| +0 | int16 LE | roll_raw |
| +2 | int16 LE | pitch_raw |
| +4 | int16 LE | yaw_raw |
| +6 | int16 LE | quat_w_raw |
| +8 | int16 LE | quat_x_raw |
| +10 | int16 LE | quat_y_raw |
| +12 | int16 LE | quat_z_raw |

### 6.4 wrist_gyro

`wrist_gyro` 位于 `offset 161 - 166`：

| 偏移 | 类型 | 内容 |
|---:|---|---|
| 161 | int16 LE | gyro_x_raw |
| 163 | int16 LE | gyro_y_raw |
| 165 | int16 LE | gyro_z_raw |

当前实现中该字段来自 IMU `0x50`，读取 `GX/GY/GZ`。

### 6.5 checksum

第 167 字节为 XOR 校验：

```cpp
uint8_t checksum = 0;
for (int i = 0; i < 167; i++) {
  checksum ^= frame[i];
}
```

校验条件：

```text
checksum == frame[167]
```

### 6.6 原始值转换

欧拉角：

```text
angle_deg = raw / 32768.0 * 180.0
```

四元数：

```text
quat_float = raw / 32768.0
```

腕部角速度：

```text
gyro_dps = raw / 32768.0 * 2000.0
```

## 7. Python Binary 解析示例

```python
import serial
import struct

FRAME_LEN = 168

def xor_checksum(data):
    x = 0
    for b in data:
        x ^= b
    return x

def parse_frame(frame):
    if len(frame) != FRAME_LEN:
        raise ValueError("bad length")
    if frame[0] != 0xAA or frame[1] != 0x55:
        raise ValueError("bad magic")
    if xor_checksum(frame[:167]) != frame[167]:
        raise ValueError("bad checksum")

    hand_id = frame[2]
    imu_count = frame[3]
    bat_percent = frame[4]
    frame_id = struct.unpack_from("<H", frame, 5)[0]

    imus = []
    for i in range(11):
        base = 7 + i * 14
        raw = struct.unpack_from("<7h", frame, base)
        euler_deg = tuple(v / 32768.0 * 180.0 for v in raw[:3])
        quat = tuple(v / 32768.0 for v in raw[3:])
        imus.append({
            "addr": 0x50 + i,
            "euler_deg": euler_deg,
            "quat_wxyz": quat,
        })

    gyro_raw = struct.unpack_from("<3h", frame, 161)
    wrist_gyro_dps = tuple(v / 32768.0 * 2000.0 for v in gyro_raw)

    return {
        "hand_id": hand_id,
        "imu_count": imu_count,
        "bat_percent": bat_percent,
        "frame_id": frame_id,
        "imus": imus,
        "wrist_gyro_dps": wrist_gyro_dps,
    }

def read_loop(port="COM34"):
    ser = serial.Serial(port, 2000000, timeout=0.2)
    buf = bytearray()
    while True:
        buf += ser.read(4096)
        while len(buf) >= FRAME_LEN:
            idx = buf.find(b"\xAA\x55")
            if idx < 0:
                buf.clear()
                break
            if idx > 0:
                del buf[:idx]
            if len(buf) < FRAME_LEN:
                break
            frame = bytes(buf[:FRAME_LEN])
            del buf[:FRAME_LEN]
            try:
                obj = parse_frame(frame)
                print(obj["hand_id"], obj["frame_id"], obj["bat_percent"])
            except ValueError:
                del buf[:1]
```

## 8. 接收器状态灯

接收器状态灯默认：

```cpp
#define STATUS_LED_PIN 36
#define LED_ACTIVE_HIGH 1
```

如果硬件 LED 引脚不同，改 `STATUS_LED_PIN`。如果低电平点亮，改 `LED_ACTIVE_HIGH` 为 0。

状态逻辑：

| 状态 | 条件 | LED |
|---|---|---|
| 无信号 | 左右手都没有新鲜 ESP-NOW 数据 | 快闪，约 120 ms 翻转 |
| 有问题 | 任一手断流、任一 `fail_mask != 0`、任一电量 < 15% | 慢闪，约 600 ms 翻转 |
| 正常 | 左右手 1 秒内都有数据，`fail_mask=0`，电量 >= 15% | 常亮 |

新鲜数据定义：

```text
millis() - g_lastRxMs[hand] <= 1000
```

## 9. 烧录命令汇总

Arduino CLI：

```text
D:\Users\Bingxu_Ma\AppData\Local\Programs\Arduino IDE\resources\app\lib\backend\resources\arduino-cli.exe
```

FQBN：

```text
esp32:esp32:XIAO_ESP32S3
```

### 9.1 USB Text 模式

```powershell
# 左手直插文本
& 'D:\Users\Bingxu_Ma\AppData\Local\Programs\Arduino IDE\resources\app\lib\backend\resources\arduino-cli.exe' upload --fqbn esp32:esp32:XIAO_ESP32S3 --port COM29 'D:\Users\Bingxu_Ma\Desktop\Glovity\ESPNow_DualHand_IMU\LeftHand_USB_Text'

# 右手直插文本
& 'D:\Users\Bingxu_Ma\AppData\Local\Programs\Arduino IDE\resources\app\lib\backend\resources\arduino-cli.exe' upload --fqbn esp32:esp32:XIAO_ESP32S3 --port COM26 'D:\Users\Bingxu_Ma\Desktop\Glovity\ESPNow_DualHand_IMU\RightHand_USB_Text'
```

### 9.2 ESP-NOW Binary 模式

```powershell
# 左手无线
& 'D:\Users\Bingxu_Ma\AppData\Local\Programs\Arduino IDE\resources\app\lib\backend\resources\arduino-cli.exe' upload --fqbn esp32:esp32:XIAO_ESP32S3 --port COM29 'D:\Users\Bingxu_Ma\Desktop\Glovity\ESPNow_DualHand_IMU\LeftHand_TX'

# 右手无线
& 'D:\Users\Bingxu_Ma\AppData\Local\Programs\Arduino IDE\resources\app\lib\backend\resources\arduino-cli.exe' upload --fqbn esp32:esp32:XIAO_ESP32S3 --port COM26 'D:\Users\Bingxu_Ma\Desktop\Glovity\ESPNow_DualHand_IMU\RightHand_TX'

# 接收器
& 'D:\Users\Bingxu_Ma\AppData\Local\Programs\Arduino IDE\resources\app\lib\backend\resources\arduino-cli.exe' upload --fqbn esp32:esp32:XIAO_ESP32S3 --port COM34 'D:\Users\Bingxu_Ma\Desktop\Glovity\ESPNow_DualHand_IMU\Receiver_RX'
```

## 10. 常见故障

### 10.1 接收器找不到某只手

优先检查该手是否被烧成了独立 `*_USB_Text`。独立 USB Text 模式不会发送 ESP-NOW。

恢复无线模式：

```text
左手烧回 LeftHand_TX
右手烧回 RightHand_TX
```

### 10.2 Arduino 串口监视器看到乱码

如果看的是 COM34，这是正常的，因为 COM34 在 Binary 模式下输出 168 字节二进制帧。

如果需要人眼可读信息，请直连手套并烧录 `LeftHand_USB_Text` 或 `RightHand_USB_Text`。

### 10.3 Text 模式无输出

检查：

1. 波特率是否为 921600。
2. 是否选对手套 COM 口。
3. ESP32-S3 是否退出 boot/download 模式。
4. USB CDC On Boot 是否可用。
5. DTR/RTS 是否导致板子复位。

### 10.4 Binary 模式丢帧

上位机应按 `hand_id` 分开检查 `frame_id` 连续性。若丢帧明显：

1. 检查手套供电。
2. 检查 ESP-NOW 信道一致。
3. 检查接收器串口读取是否及时。
4. 确认接收器串口和上位机都使用 `2000000`；如果为临时调试降速，必须同步修改接收器固件和上位机。
