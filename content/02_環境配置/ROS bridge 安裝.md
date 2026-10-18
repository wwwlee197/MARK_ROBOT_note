# ROS bridge 安裝

## 目的

使用 `ros1_bridge` 的 Dynamic Bridge 模式，讓 ROS1 與 ROS2 之間可以自動偵測並轉發同名 topic。

## 前置條件

- 已完成 [ROS1 Noetic 安裝](ROS1%20Noetic%20安裝.md)。
- 已完成 [ROS2 Humble 安裝](ROS2%20Humble%20安裝.md)。
- 已安裝 colcon 相關開發工具。

## 操作步驟

### 1. 建立 bridge workspace

```bash
mkdir -p ~/bridge_ws/src
cd ~/bridge_ws/src
git clone https://github.com/ros2/ros1_bridge.git -b master
```

### 2. 編譯 ros1_bridge

編譯前要依序載入 ROS1 與 ROS2 環境，讓 CMake 能同時找到兩邊。

```bash
cd ~/bridge_ws
source /opt/ros/noetic/setup.bash
source /opt/ros/humble/setup.bash
colcon build --symlink-install --packages-select ros1_bridge --cmake-force-configure
```

## 測試方式

測試時需要開啟四個 terminal。

### Terminal 1：啟動 ROS1 master

```bash
source /opt/ros/noetic/setup.bash
roscore
```

### Terminal 2：啟動 Dynamic Bridge

```bash
source /opt/ros/noetic/setup.bash
source /opt/ros/humble/setup.bash
source ~/bridge_ws/install/local_setup.bash
ros2 run ros1_bridge dynamic_bridge
```

### Terminal 3：啟動 ROS1 talker

```bash
source /opt/ros/noetic/setup.bash
rosrun rospy_tutorials talker
```

### Terminal 4：啟動 ROS2 listener

```bash
source /opt/ros/humble/setup.bash
ros2 run demo_nodes_cpp listener
```

## 驗證方式

ROS2 listener 若能收到 ROS1 talker 發出的 `/chatter` 訊息，代表 bridge 可以正常轉發 topic。

## 專題執行指令

實際跑 TWRB 時，使用雙向轉發參數，並先確認部署版支援的訊息型別配對：

```bash
ros2 run ros1_bridge dynamic_bridge --print-pairs
ros2 run ros1_bridge dynamic_bridge --bridge-all-2to1-topics --bridge-all-1to2-topics
```

輪速命令與回授分別需要 `std_msgs/Int16MultiArray`、`std_msgs/Int32MultiArray` 的 ROS1／ROS2 配對。舊指令的 `--qos-reliability`／`--qos-durability` 沒有列在[官方 dynamic_bridge 參數解析](https://github.com/ros2/ros1_bridge/blob/master/src/dynamic_bridge.cpp)中，不應假設是通用的全域 QoS 設定；若現場使用客製版本，先以 `ros2 run ros1_bridge dynamic_bridge --help` 核對。

本次保留既有 ROS1／ROS2 安裝與編譯方式；未在目前 Windows 文件環境重建 ROS。實際部署仍須確認 bridge 版本、型別配對與資料往返，詳見 [ROS Topic 與通訊協定](../04_程式碼說明/ROS%20Topic%20與通訊協定.md)。

完整啟動順序請看 [1_Examples 執行流程](../03_執行流程/1_Examples%20執行流程.md)。
