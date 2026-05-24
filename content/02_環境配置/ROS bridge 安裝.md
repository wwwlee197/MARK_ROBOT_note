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

實際跑 TWRB 時，bridge mode 會使用較完整的參數：

```bash
ros2 run ros1_bridge dynamic_bridge --bridge-all-2to1-topics --bridge-all-1to2-topics --qos-reliability best_effort --qos-durability volatile
```

完整啟動順序請看 [1_Examples 執行流程](../03_執行流程/1_Examples%20執行流程.md)。
