# TurtleBots & TWRB 連線與執行流程

## 目的

整理日常操作 TurtleBots / TWRB 時會用到的連線、rosserial、ros1_bridge 與測試 topic 指令。照這份流程開 terminal，可以完成從 PC 控制 TWRB 馬達的基本流程。

## 前置條件

- PC、TWRB 與路由器在同一網段。
- PC 端已完成 ROS1、ROS2、ros1_bridge 與 `.bashrc` 設定。
- Arduino MEGA 已燒錄控制程式。
- TWRB 可透過 SSH 連線。
- TWRB 端已接好 Arduino 或對應的 serial 裝置。

## 網路資訊

目前文件使用的範例 IP：

| 裝置   | IP            |
| ------ | ------------- |
| PC     | 192.168.0.102 |
| TWRB 1 | 192.168.0.105 |
| TWRB 2 | 192.168.0.106 |

## Terminal 啟動順序

建議依照下面順序開 terminal，比較容易定位問題：

| 順序 | 位置 | 環境        | 用途                            |
| ---- | ---- | ----------- | ------------------------------- |
| 1    | PC   | ROS1        | 啟動 `roscore`                  |
| 2    | PC   | Bridge Mode | 啟動 `ros1_bridge`              |
| 3    | PC   | SSH         | 連線到 TWRB                     |
| 4    | TWRB | ROS1        | 啟動 `serial_node.py`           |
| 5    | PC   | ROS2 / ROS1 | 發送 `/set_pwm` 或觀察 feedback |

## 實際操作參考

第一次照流程跑時，可以搭配 [實際執行影片](https://www.youtube.com/watch?v=L9N0wVhpF3s) 對照 terminal 啟動順序與車體反應。

## TWRB 端指令

TWRB 端指令通常是從 PC 使用 SSH 連進樹莓派後執行。

### 1. 從 PC 連線到 TWRB

連線到 TWRB 1：

```bash
ssh pi@192.168.0.105
```

連線到 TWRB 2：

```bash
ssh pi@192.168.0.106
```

### 2. 啟動 rosserial：樹莓派 UART

在 TWRB 端 ROS1 環境中執行。這個指令會把樹莓派收到的馬達控制資訊，透過 `/dev/serial0` 傳給 Arduino。

```bash
rosrun rosserial_python serial_node.py _port:=/dev/serial0 _baud:=115200
```

### 3. 啟動 rosserial：USB serial

在 TWRB 端 ROS1 環境中執行。若 Arduino 或 serial 裝置是透過 USB 連接，常見 port 會是 `/dev/ttyUSB1`。

```bash
rosrun rosserial_python serial_node.py _port:=/dev/ttyUSB1 _baud:=115200 __name:=serial_node_01
```

## PC 端指令

### 1. 啟動 ROS1 master

在 PC 端開新 terminal，選擇 `.bashrc` 的 ROS1 mode，啟動 ROS1 master。

```bash
roscore
```

### 2. 啟動 ros1_bridge

在 PC 端另開 terminal，選擇 `.bashrc` 的 Bridge Mode。這個 bridge 會讓 ROS1 與 ROS2 的同名 topic 可以互相轉發。

```bash
ros2 run ros1_bridge dynamic_bridge --bridge-all-2to1-topics --bridge-all-1to2-topics --qos-reliability best_effort --qos-durability volatile
```

## 日常測試指令

### 1. ROS2 發送 `/set_pwm`

在 PC 端 ROS2 環境中執行。這個 topic 用來傳送馬達速率資訊，控制對應的 TWRB。

```bash
ros2 topic pub /set_pwm std_msgs/msg/Int16MultiArray "{layout: {dim: [], data_offset: 0}, data: [0, 0]}"
```

### 2. ROS2 發送 `/set_pwm_01`

在 PC 端 ROS2 環境中執行。若有第二台車或第二組控制 topic，可使用 `/set_pwm_01`。

```bash
ros2 topic pub /set_pwm_01 std_msgs/msg/Int16MultiArray "{layout: {dim: [], data_offset: 0}, data: [0, 0]}"
```

## 補充：ROS1 測試與回饋確認

### 1. ROS1 發送 `/set_pwm`

在 PC 端 ROS1 環境中執行。這是 ROS1 版本的馬達控制測試指令。

```bash
rostopic pub /set_pwm std_msgs/Int16MultiArray "layout: {dim: [], data_offset: 0}, data: [0, 0]"
```

### 2. 確認 TWRB 收到的回饋

在 PC 端 ROS1 環境中執行，用來即時確認樹莓派收到的數值。

```bash
rostopic echo /bridge_feedback
```

## 驗證方式

PC 端可使用 topic 指令確認節點與 topic 是否出現：

```bash
rostopic list
ros2 topic list
```

若 bridge 正常，ROS1 / ROS2 同名 topic 應可以互相轉發。發送 `/set_pwm` 或 `/set_pwm_01` 時，TWRB 端 rosserial terminal 不應出現連線錯誤。

## 注意事項

- `/dev/serial0` 常見於樹莓派 UART；`/dev/ttyUSB1` 常見於 USB serial，實際 port 可能因接線方式或插拔順序改變。
- 若 `serial_node.py` 啟動失敗，先確認 Arduino 是否接上、baud rate 是否為 `115200`，以及 port 是否正確。
- `serial_node_01` 用來避免多台車或多個 rosserial 節點名稱衝突。
- 若 SSH 連不上，先確認 PC 與 TWRB 是否在同一網段，並檢查固定 IP 設定。
- 發送 `[0, 0]` 代表左右輪 PWM 都為 0，通常用於停止或安全測試。
