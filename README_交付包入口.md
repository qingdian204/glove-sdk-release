# Glovity 完整交付包入口

更新时间：2026-07-01

本文件夹用于项目交接，包含 MANO 可视化 SDK、ESP32 固件、测试调试工具和交付文档。软件部门提供的主程序 `glove-sdk-release` 已原样放入 `01_MANO_Visualization_SDK`，未修改其内部内容。

## 文件夹结构

| 文件夹 | 内容 |
|---|---|
| `01_MANO_Visualization_SDK` | 软件部门提供的可视化 MANO/SDK 主程序，包含可执行文件 `glove-sdk`、依赖目录和原始说明 |
| `02_Firmware/30Hz_Product` | 推荐正式固件：左右手 ESP-NOW 30Hz + USB Text 30Hz + 接收器 |
| `02_Firmware/5Hz_LowUSB_Debug` | 备用调试固件：左右手 ESP-NOW 30Hz + USB Text 5Hz + 接收器 |
| `02_Firmware/Pure_USB_Text` | 纯 USB 文本调试固件：不走 ESP-NOW，不需要接收器 |
| `03_Test_Debug_Tools/Local_TripleSerial_LinkTester` | 本地三串口链路测试工具，同时观察 COM29/COM26/COM34 |
| `03_Test_Debug_Tools/Integrated_USB_MagCalibration` | 集成磁力计校准脚本，通过手套 USB 触发，不需要切换固件 |
| `03_Test_Debug_Tools/Legacy_Standalone_MagCalibration` | 旧版独立磁校准流程，保留作备份 |
| `04_Documents` | 开发者 README、使用者 README，以及对应 Word 文档 |
| `05_Reference` | 原始/参考文档副本，包括 SDK 使用说明和嵌入式协议文档 |

## 推荐正式烧录组合

```text
左手手套 COM29 -> 02_Firmware/30Hz_Product/LeftHand_TX_USB30
右手手套 COM26 -> 02_Firmware/30Hz_Product/RightHand_TX_USB30
接收器   COM34 -> 02_Firmware/30Hz_Product/Receiver_RX
```

接收器输出：

```text
COM34 @ 2000000
168 bytes/frame
```

手套 USB 调试/校准口：

```text
COM29 / COM26 @ 921600
```

## 重要说明

- `01_MANO_Visualization_SDK/glove-sdk-release` 是主程序交付物，已保持原样。
- SDK 目录内自带的 `esp32` 示例固件不是当前产品固件入口；当前产品固件以 `02_Firmware` 为准。
- 磁力计校准推荐使用 `03_Test_Debug_Tools/Integrated_USB_MagCalibration`，无需切换到独立 MagCalibrate 固件。
- 当前 I2C 为 `SDA=GPIO8`、`SCL=GPIO9`；电池 ADC 为 `GPIO2`；自动关机控制为 `GPIO41`。

