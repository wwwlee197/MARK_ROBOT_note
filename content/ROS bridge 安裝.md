>使用 `ros1_bridge` 的 **Dynamic Bridge (動態橋接)** 模式。這是最簡單的模式，它會自動偵測並轉發兩邊同名的 Topic。


### 1. 準備編譯環境
```bash
# 1. 建立工作空間目錄
mkdir -p ~/bridge_ws/src
cd ~/bridge_ws/src

# 2. 下載 ros1_bridge 原始碼 (使用與 Humble 相容的分支)
git clone https://github.com/ros2/ros1_bridge.git -b master
```


### 2. 編譯 ros1_bridge 
```bash
cd ~/bridge_ws

# 1. 先 source ROS 1 (Noetic)
source /opt/ros/noetic/setup.bash

# 2. 再 source ROS 2 (Humble)
source /opt/ros/humble/setup.bash

# 3. 開始編譯 (使用 colcon)
# --packages-select ros1_bridge: 只編譯 bridge 本身
# --cmake-force-configure: 強制重新配置 CMake 以確保它抓到兩個環境
colcon build --symlink-install --packages-select ros1_bridge --cmake-force-configure
```

---
# 測試 
>(開啟四個 Terminal 測試)

### 第一步：啟動 ROS 1 Master (Terminal 1)
>ROS 1 需要一個 Master 節點才能運作。
```bash
source /opt/ros/noetic/setup.bash
roscore
```
### 第二步：啟動 Bridge (Terminal 2)
>同時載入 ros bridge 、 ros1 noetic 、 ros2 humble 環境後
>啟動 ros bridge 

```bash
# 1. 載入 ROS 1
source /opt/ros/noetic/setup.bash

# 2. 載入 ROS 2
source /opt/ros/humble/setup.bash

# 3. 載入您編譯好的 Bridge Workspace (注意路徑是否正確)
source ~/bridge_ws/install/local_setup.bash

# 4. 啟動動態橋接器
ros2 run ros1_bridge dynamic_bridge
```

### 第三步：啟動 ROS 1 發話者 (Terminal 3)
>我們用 ROS 1 發送一個名為 /chatter 的訊息。
>狀態：您應該會看到它一直印出 [INFO]: hello world ...

```bash
source /opt/ros/noetic/setup.bash
rosrun rospy_tutorials talker
```

### 第四步：啟動 ROS 2 收聽者 (Terminal 4)
>用 ROS 2 來收聽同名的 /chatter 訊息。

```bash
source /opt/ros/humble/setup.bash
ros2 run demo_nodes_cpp listener
```