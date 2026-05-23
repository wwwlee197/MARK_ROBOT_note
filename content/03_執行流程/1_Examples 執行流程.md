# 1_Examples 執行流程

## 目的

整理 PC 端與 TWRB 端啟動順序，讓接手者可以依序開啟 ROS master、ros1_bridge 與 rosserial 節點。

## 前置條件

- PC、TWRB 與路由器在同一網段。
- PC 端已完成 ROS1、ROS2、ros1_bridge 與 `.bashrc` 設定。
- Arduino MEGA 已燒錄控制程式。
- TWRB 可透過 SSH 連線。

## 網路資訊

目前文件使用的範例 IP：

| 裝置   | IP            |
| ------ | ------------- |
| PC     | 192.168.0.102 |
| TWRB 1 | 192.168.0.105 |
| TWRB 2 | 192.168.0.106 |

## PC 端流程

### 1. SSH 連線進 TWRB

連線 TWRB 1：

```bash
ssh pi@192.168.0.105
```

連線 TWRB 2：

```bash
ssh pi@192.168.0.106
```

### 2. 啟動 ROS1 master

在 PC 端開新 terminal，選擇 `.bashrc` 的 ROS1 mode。

```bash
roscore
```

### 3. 啟動 ros1_bridge

在 PC 端另開 terminal，選擇 `.bashrc` 的 Bridge Mode。

```bash
ros2 run ros1_bridge dynamic_bridge --bridge-all-2to1-topics --bridge-all-1to2-topics --qos-reliability best_effort --qos-durability volatile
```

## TWRB 端流程

### TWRB No.0

在 TWRB 端 ROS1 環境中啟動：

```bash
rosrun rosserial_python serial_node.py _port:=/dev/ttyUSB1 _baud:=115200 __name:=serial_node_00
```

### TWRB No.1

在 TWRB 端 ROS1 環境中啟動：

```bash
rosrun rosserial_python serial_node.py _port:=/dev/ttyUSB1 _baud:=115200 __name:=serial_node_01
```

## 驗證方式

PC 端可使用 topic 指令確認節點與 topic 是否出現：

```bash
rostopic list
ros2 topic list
```

若 bridge 正常，ROS1 / ROS2 同名 topic 應可以互相轉發。

## 注意事項

- `/dev/ttyUSB1` 可能因插拔順序改變，若啟動失敗要檢查實際 port。
- `serial_node_00` 與 `serial_node_01` 用來避免兩台車節點名稱衝突。
- 若 SSH 連不上，先確認 PC 與 TWRB 是否在同一網段，並檢查固定 IP 設定。
