# bashrc 設定

## 目的

PC 端需要在 ROS1、ROS2、Bridge Mode 之間切換。本文件提供一段 `.bashrc` 設定，讓每次開啟 terminal 時可以選擇要載入哪一種環境。

## 前置條件

- 已完成 ROS1 Noetic 安裝。
- 已完成 ROS2 Humble 安裝。
- 若要使用 Bridge Mode，已完成 ros1_bridge 編譯。

## 操作步驟

### 1. 編輯 .bashrc

```bash
nano ~/.bashrc
```

### 2. 加入環境選擇器

依照實際網路環境修改 `MY_IP`。範例中的 PC IP 是 `192.168.0.102`。

```bash
# =======================================
# 自定義 ROS 環境選擇器
# =======================================

echo -n "ROS1 noetic(1) | ROS2 humble(2) | Bridge Mode(3) [2]: "
read choice

# 如果直接按 Enter，預設選 2
if [[ -z "$choice" ]]; then
    choice="2"
fi

# 本機內通訊： 127.0.0.1
# 跨電腦通訊： 改為實際 IP (例如 192.168.0.x)
export MY_IP=192.168.0.102

if [[ $choice == "1" ]]; then
    echo ">>> Loading ROS1 Noetic ..."
    source /opt/ros/noetic/setup.bash

    export ROS_MASTER_URI=http://$MY_IP:11311
    export ROS_IP=$MY_IP

elif [[ $choice == "3" ]]; then
    echo ">>> Loading ROS1 & ROS2 Bridge Mode..."
    source /opt/ros/noetic/setup.bash
    source /opt/ros/humble/setup.bash
    source ~/bridge_ws/install/setup.bash

    export ROS_MASTER_URI=http://$MY_IP:11311
    export ROS_IP=$MY_IP
    export ROS_DOMAIN_ID=30

    # 讓 Colcon 編譯器同時找到 ROS1 與 ROS2
    export CMAKE_PREFIX_PATH=/opt/ros/humble:/opt/ros/noetic

else
    echo ">>> Loading ROS2 Humble..."
    source /opt/ros/humble/setup.bash
    source /usr/share/colcon_argcomplete/hook/colcon-argcomplete.bash

    # 若有自己的 ROS2 workspace，可在這裡載入
    # source ~/ros2_ws/install/setup.bash

    export ROS_DOMAIN_ID=30
    export ROS_IP=$MY_IP
fi
```

### 3. 重新載入設定

```bash
source ~/.bashrc
```

## 驗證方式

開啟新 terminal 後，應出現以下選單：

```text
ROS1 noetic(1) | ROS2 humble(2) | Bridge Mode(3) [2]:
```

輸入 `1` 後可執行：

```bash
roscore
```

輸入 `2` 後可執行：

```bash
ros2 topic list
```

輸入 `3` 後可啟動 ros1_bridge。

## 注意事項

- `MY_IP` 必須與 PC 的固定 IP 一致。
- 跨電腦通訊時不要使用 `127.0.0.1`。
- 若 bridge workspace 路徑不是 `~/bridge_ws`，需要同步修改 `source ~/bridge_ws/install/setup.bash`。
