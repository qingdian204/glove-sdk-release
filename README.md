======================================================================
              IMU 手套骨骼可视化 SDK — 使用说明
======================================================================

一、文件说明
----------------------------------------------------------------------
本文件夹包含一个可直接运行的程序 glove-sdk，无需安装 Python。
手部模型文件已内置，无需额外下载。


二、校准（首次使用必须）
----------------------------------------------------------------------
每只手第一次使用前需要做一次三步校准（约 30 秒）。

  校准左手：
    ./glove-sdk --calibrate --port /dev/ttyACM0 --side left --text

  校准右手：
    ./glove-sdk --calibrate --port /dev/ttyACM0 --side right --text

  校准步骤：
    步骤 1：手掌朝下水平，手指伸直自然张开 → 按 Enter
    步骤 2：手腕向上转 90°（指尖朝上）      → 按 Enter
    步骤 3：手腕向下转 90°（指尖朝下）      → 按 Enter

  校准完成后会在程序目录生成 imu_calibration_left.npy 或
  imu_calibration_right.npy 文件，请勿删除。


三、磁力计校准（推荐）
----------------------------------------------------------------------
磁力计校准需要先在 ESP32 上烧录专用固件，再运行校准命令。

  步骤 1：烧录 MagCalibrate 固件
    arduino-cli upload --fqbn esp32:esp32:XIAO_ESP32S3 \
      --port /dev/ttyACM0 \
      esp32/MagCalibrate

    上电后串口会扫描 I2C 总线，列出在线 IMU 数量（期望 11 个）。

  步骤 2：运行磁力计校准
    ./glove-sdk --calibrate-mag --port /dev/ttyACM0

    按 Enter 开始 → 旋转约 30 秒（8字形）→ 再按 Enter 停止并保存。

  步骤 3：恢复工作固件
    校准完成后，重新烧录 IMU_Serial_Direct 固件：
    arduino-cli upload --fqbn esp32:esp32:XIAO_ESP32S3 \
      --port /dev/ttyACM0 \
      esp32/IMU_Serial_Direct


四、启动可视化
----------------------------------------------------------------------

  左手：
    ./glove-sdk --port /dev/ttyACM0 --side left --text

  右手：
    ./glove-sdk --port /dev/ttyACM0 --side right --text

  指定校准文件：
    ./glove-sdk --port /dev/ttyACM0 --side right --text \
                --calib imu_calibration_right.npy


五、命令行参数一览
----------------------------------------------------------------------

  --port 串口设备      默认 /dev/ttyACM0
  --side left/right    左手或右手
  --text               USB 直连文本模式
  --calib 文件路径     校准文件路径
  --baud 波特率        text 模式默认 921600
  --calibrate          进入校准模式
  --calibrate-mag      进入磁力计校准模式
  --out 输出路径       校准文件输出路径


六、程序界面说明
----------------------------------------------------------------------
启动后窗口分为两部分：

  左侧：3D 手部骨骼实时渲染（鼠标可旋转/缩放/平移）
  右侧：参数控制面板
    - Flip Direction    翻转手指弯曲方向和坐标轴
    - Coordinate Preset 坐标系旋转预设
    - DIP Coupling      DIP 关节跟随 PIP 的比例
    - Finger Splay      手指张开角度微调
    - Wrist Yaw Fix     手腕偏航修正
    - Reset Parameters  重置所有参数
    - FPS               实时帧率


七、常见问题
----------------------------------------------------------------------

Q: 串口打不开 / Permission denied
A: 执行 sudo usermod -aG dialout $USER，然后重新登录

Q: 提示校准文件找不到
A: 先执行校准命令（见第二步），或用 --calib 指定路径

Q: 没有 GPU 能用吗？
A: 本程序需要 NVIDIA GPU + CUDA 驱动，不支持纯 CPU 运行

======================================================================
