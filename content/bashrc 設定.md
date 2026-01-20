> PC 端設定每個終端機開啟前選擇自定義的環境

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
    # 1. 載入系統級 ROS1
    source /opt/ros/noetic/setup.bash
    
    # 2. 網路設定
    export ROS_MASTER_URI=http://$MY_IP:11311
    export ROS_IP=$MY_IP

elif [[ $choice == "3" ]]; then
    echo ">>> Loading ROS1 & ROS2 Bridge Mode..."
    # 1. 載入 ROS1
    source /opt/ros/noetic/setup.bash
    
    # 2. 載入 ROS2
    source /opt/ros/humble/setup.bash
    
    # 3. 載入 Bridge 工作空間 
    source ~/bridge_ws/install/setup.bash

    # 4. 網路與通訊設定
    export ROS_MASTER_URI=http://$MY_IP:11311
    export ROS_IP=$MY_IP
    export ROS_DOMAIN_ID=30
    
    # 5. [關鍵] 修正環境變數，讓 Colcon 編譯器也能同時看到兩者 (未來編譯 bridge 用)
    export CMAKE_PREFIX_PATH=/opt/ros/humble:/opt/ros/noetic

else
    echo ">>> Loading ROS2 Humble..."
    # 1. 載入系統級 ROS2
    source /opt/ros/humble/setup.bash
    source /usr/share/colcon_argcomplete/hook/colcon-argcomplete.bash
    
    # 2. 載入使用者級 ROS2 工作空間 (假設您有)
    # source ~/ros2_ws/install/setup.bash
    
    # 3. 網路設定
    export ROS_DOMAIN_ID=30
    export ROS_IP=$MY_IP
fi
```