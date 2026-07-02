# IntegratedMagCal Test Package

用途：正常手套固件内集成磁力计校准命令监听，电脑端通过 USB 串口触发校准，不需要再手动切换到单独的 MagCalibrate 固件。

## 内容

```text
IntegratedMagCal_TestPackage_20260701/
  Firmware/
    LeftHand_TX_USB30/      左手正常固件，ESP-NOW 30Hz + USB Text 30Hz + 校准命令
    RightHand_TX_USB30/     右手正常固件，ESP-NOW 30Hz + USB Text 30Hz + 校准命令
    LeftHand_TX/            左手正常固件，ESP-NOW 30Hz + USB Text 5Hz + 校准命令
    RightHand_TX/           右手正常固件，ESP-NOW 30Hz + USB Text 5Hz + 校准命令
    Receiver_RX/            匹配上述手套 RadioPacket 的接收器，USB 以 2000000 baud 输出 168 字节二进制帧
  Host_Tester/
    integrated_mag_calibrate.py  电脑端校准触发脚本
```

## 当前推荐烧录版本

```text
左手 COM29 -> Firmware/LeftHand_TX_USB30
右手 COM26 -> Firmware/RightHand_TX_USB30
```

烧录命令示例：

```powershell
$ARDUINO = "C:\Users\Bingxu_Ma\Documents\Glovity\.tools\arduino-cli\arduino-cli.exe"
$ROOT = "D:\Users\Bingxu_Ma\Desktop\Glovity\IntegratedMagCal_TestPackage_20260701"

& $ARDUINO upload --fqbn esp32:esp32:XIAO_ESP32S3 --port COM29 "$ROOT\Firmware\LeftHand_TX_USB30"
& $ARDUINO upload --fqbn esp32:esp32:XIAO_ESP32S3 --port COM26 "$ROOT\Firmware\RightHand_TX_USB30"
& $ARDUINO upload --fqbn esp32:esp32:XIAO_ESP32S3 --port COM34 "$ROOT\Firmware\Receiver_RX"
```

注意：这里的手套 TX 固件发送 `RadioPacket = magic + fail_mask + 168B SerialFrame`。必须搭配本包里的 `Receiver_RX`，不要混用旧 `mano_minimal_v2/esp32/ESP_NOW_Receiver`。

正常采集链路参数：手套 USB Text 调试/校准口为 `921600`；接收器 `COM34` 的 168 字节二进制主数据口为 `2000000`，与 Linux SDK 的 Binary 模式保持一致。

低电压自动关机：手套 TX 固件每 1 秒对 VBAT 做 8 次 ADC 采样平均；若 VBAT 低于 `3.50V` 连续约 `8s`，会打印 `battery low, auto power off`，停止 ESP-NOW/Wi-Fi，关闭 IMU/IR 负载，并将 `POWER_CTRL = GPIO41` 拉高约 `3s`，等效长按 UOV-K20 电源键关机。

无接收器响应自动关机：手套 ESP-NOW 单播发送到接收器 `14:c1:9f:2e:79:c0`。若最近 `30s` 内没有单播发送成功，同时 VBAT 每 `60s` 对比确认正在下降，且该状态连续 `15min`，固件会执行同一套 GPIO41 自动关机流程。若 VBAT 没有下降，则认为可能正在 USB 供电/充电，不执行无响应关机。

## 电脑端启动校准

左手：

```powershell
cd "D:\Users\Bingxu_Ma\Desktop\Glovity\IntegratedMagCal_TestPackage_20260701\Host_Tester"
python .\integrated_mag_calibrate.py --port COM29 --duration 35
```

右手：

```powershell
cd "D:\Users\Bingxu_Ma\Desktop\Glovity\IntegratedMagCal_TestPackage_20260701\Host_Tester"
python .\integrated_mag_calibrate.py --port COM26 --duration 35
```

脚本会等待按 Enter 后发送：

```text
$GLOVITY,CAL,START
```

35 秒后自动发送：

```text
$GLOVITY,CAL,STOP
```

成功判据：

```text
$GLOVITY,CAL,SCAN_DONE,<active>,11
$GLOVITY,CAL,STARTED
$GLOVITY,CAL,SAVE,0x50,OK
...
$GLOVITY,CAL,SAVE,0x60,OK
$GLOVITY,CAL,COMPLETE
$GLOVITY,MODE,NORMAL
```

校准期间手套会暂停正常 ESP-NOW/USB Text 回传；校准保存完成后自动回到正常 30Hz 回传。

脚本会按 `SCAN_DONE` 中的 active IMU 数量判断成功，不再硬编码必须 `11/11`；但量产/完整手套仍建议确认 active 为 11。
