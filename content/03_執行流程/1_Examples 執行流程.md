# TurtleBots & TWRB 連線與執行流程

## 目的

依現行 `twrb_project/main.py` 啟動 ROS 通訊與導航，確認命令確實到達相容的 Arduino 韌體。以下對應 commit `98dba0d6`；模式 2 使用 PWM，其餘模式使用 RPS × 1000。詳細契約見 [ROS Topic 與通訊協定](../04_程式碼說明/ROS%20Topic%20與通訊協定.md)。

## 前置條件

- PC、TWRB 與路由器在同一網段，PC 已完成 ROS1、ROS2、`ros1_bridge` 與 [bashrc 設定](../02_環境配置/bashrc%20設定.md)。
- TWRB 可透過 SSH 連線，已接妥 Arduino 串列介面。
- **已確認車上韌體與選定模式相容。** 正式燒錄來源仍待現場確認，不能直接使用舊 PWM 韌體執行 RPS 模式，見[燒錄說明](../01_系統安裝/Arduino%20MEGA%20程式碼燒錄.md)。
- 車輪先離地、場地清空且實體停止方式可用；同一台車只啟動一個控制程式。
- PC 使用可存取 ROS2 `rclpy` 的 Python 環境，並具備相機／圖形顯示、校正檔及所選模式的相依套件。

### PC 程式環境預檢

在 Ubuntu 的程式專案根目錄操作，路徑依實際位置調整：

```bash
cd ~/twrb_project
source /opt/ros/humble/setup.bash
python3 -m pip install -r requirements.txt
python3 -c 'import rclpy, yaml, cv2, numpy, matplotlib; from cv2 import aruco; from std_msgs.msg import Int16MultiArray, Int32MultiArray'
```

`requirements.txt` 包含 PyYAML、NumPy、pandas、opencv-contrib-python、matplotlib、ultralytics；ROS 的 `rclpy`／message packages 由 ROS 發行版提供，不能以 pip 安裝替代。安裝應使用專題既定 Python 環境，避免把 ROS Python 與不相容的 Conda interpreter 混用。

模式 5 另核對 `config/profiles/single_yolo_astar.yaml` 指定的 `yolo26m-seg.pt`。相機 index、內參與世界標籤位置依各模式設定；模式 2 的解析度差異尚待確認，詳見 [PC 控制架構](../04_程式碼說明/PC%20端控制架構.md)。

## 網路資訊

以下沿用舊文件的範例 IP，不是自動偵測結果：

| 裝置            | IP            |
| --------------- | ------------- |
| PC／ROS1 master | 192.168.0.102 |
| TWRB 1          | 192.168.0.105 |
| TWRB 2          | 192.168.0.106 |

PC 的 ROS1 與 Bridge terminal 應使用 `ROS_MASTER_URI=http://192.168.0.102:11311`、`ROS_IP=192.168.0.102`。樹莓派指向同一 master，但 `ROS_IP` 必須是樹莓派自己的 IP。所有 ROS2／bridge terminal 沿用 `ROS_DOMAIN_ID=30`。

## Terminal 啟動順序

| 順序 | 位置／環境                     | 用途                                         |
| ---- | ------------------------------ | -------------------------------------------- |
| 1    | PC／ROS1                       | 啟動 `roscore`                               |
| 2    | PC／Bridge Mode                | 啟動雙向 `ros1_bridge`                       |
| 3    | PC／SSH → TWRB／ROS1           | 啟動一個對應 serial port 的 rosserial        |
| 4    | PC／ROS1、ROS2 各一個 terminal | 查型別、監看回授、確認零命令                 |
| 5    | PC／ROS2                       | 停止手動 publisher 後，執行 `main.py` 選模式 |

### 1. PC 啟動 ROS1 master

開新 terminal，選 `.bashrc` 的 ROS1 mode：

```bash
roscore
```

### 2. PC 啟動 ros1_bridge

另開 terminal，選 Bridge Mode，確認已依序載入 ROS1、ROS2、bridge workspace：

```bash
ros2 run ros1_bridge dynamic_bridge --print-pairs
ros2 run ros1_bridge dynamic_bridge --bridge-all-2to1-topics --bridge-all-1to2-topics
```

`--print-pairs` 應列出需要的 Int16／Int32 MultiArray 轉換配對；第二行會持續執行。舊文件附帶的 `--qos-reliability best_effort --qos-durability volatile` 並非官方 `dynamic_bridge` 的通用參數；先以部署版 `--help` 查核，不要假設有生效。依據見 [bridge 安裝頁](../02_環境配置/ROS%20bridge%20安裝.md)。

### 3. SSH 與 TWRB 的 rosserial

以 TWRB 1 為例；第二台改成 `.106`，且需另外規劃命令隔離，不可讓兩台車無意訂閱同一根 Topic：

```bash
ssh pi@192.168.0.105
```

進入樹莓派後載入映像檔原有的 ROS1 環境，再設定：

```bash
export ROS_MASTER_URI=http://192.168.0.102:11311
export ROS_IP=192.168.0.105
printenv | grep ROS
ls -l /dev/serial0 /dev/serial/by-id/
```

樹莓派的 ROS1 發行版與安裝路徑依映像檔確認，不應直接假設與 PC 的 Noetic 路徑相同。以下兩種連接方式擇一，port、baud 必須與正式韌體一致：

```bash
# UART：沿用既有部署範例
rosrun rosserial_python serial_node.py _port:=/dev/serial0 _baud:=115200
```

```bash
# USB：/dev/ttyUSB1 只是舊環境範例，實際也可能是 ttyACM 裝置
rosrun rosserial_python serial_node.py _port:=/dev/ttyUSB1 _baud:=115200 __name:=serial_node_01
```

同一 serial port 只啟動一個 rosserial。`__name` 只改 ROS Node 名稱，不會把馬達 Topic 自動改為 `/set_pwm_01`，也不會自動隔離第二台車。

### 4. 先監看與測試零命令

PC 的 ROS1 terminal：

```bash
rostopic list
rostopic echo /wheel_status_x1000
```

PC 的 ROS2 terminal（RPS 相容韌體）：

```bash
ros2 topic list -t
ros2 topic info /cmd_wheel_rps_x1000 --verbose
ros2 topic echo /wheel_status_x1000 std_msgs/msg/Int32MultiArray
```

另開 ROS2 terminal 發送一次零命令；PWM 模式與 RPS 模式擇一：

```bash
# 模式 1、3、4、5、6、7、8
ros2 topic pub --once /cmd_wheel_rps_x1000 std_msgs/msg/Int16MultiArray '{data: [0, 0]}'
```

```bash
# 模式 2，需 PWM 相容韌體
ros2 topic pub --once /set_pwm std_msgs/msg/Int16MultiArray '{data: [0, 0]}'
```

輪速回授應能辨識 target／actual／PWM；PWM 韌體的回授能力需另行確認。不要只因 Topic 出現、或 rosserial 沒報錯就認定車已停止。非零診斷範本、ROS1 直接測試與 11 欄排列集中在 [ROS 通訊協定](../04_程式碼說明/ROS%20Topic%20與通訊協定.md)。

### 5. 啟動正式導航

確認沒有手動或實驗 publisher 持續發命令。在 PC 的 ROS2 terminal、專案根目錄執行：

```bash
cd ~/twrb_project
python3 main.py
```

`main.py` 自行加入 `src/` 匯入路徑，不需要把它當 ROS package 用 `ros2 run` 啟動。輸入 1～8 的模式代號；對應 Profile／控制策略見 [PC 控制架構](../04_程式碼說明/PC%20端控制架構.md)。

| 模式 | 基本操作順序                                                                                                           |
| ---- | ---------------------------------------------------------------------------------------------------------------------- |
| 1／8 | 地面標籤定位 → Space 鎖 H → 確認車上 ID 0 可見 → 點選至少一個目標 waypoint → Enter；以目前車位作起點，Z 可移除最後目標 |
| 2    | 確認校正尺寸與 PWM 相容 → Space 鎖 H → 點選至少兩個 waypoint → Enter；同時發布 RViz pose／TF                           |
| 3    | 兩台相機完成 Homography → Space → 至少一個目標 waypoint → Enter；目前車位作起點                                        |
| 4／7 | Space 鎖 H → G 後點選終點 → O 拖曳障礙框 → Enter；以目前車位作 A\* 起點，Z 復原、C 清障礙                              |
| 5    | Space 鎖 H → 清空場地按 B 擷取背景 → 必要時畫封鎖框 → G 選終點 → Enter；障礙侵入或感知失效後需人工重規劃               |
| 6    | 兩相機安全校正有效 → Space → 在安全區選至少兩點 → Enter；驗證完整路徑及交接重疊後才追蹤                                |

操作以相機視窗取得鍵盤焦點為前提。模式 1／8 可能顯示規劃動畫，關閉動畫後才進入追蹤；規劃失敗或安全條件未成立時應保持停止。不同模式的輸入與失標處理不完全相同，不要套用同一組操作假設。

## 正常結束與驗證方式

1. 正常離開用導航視窗 ESC 或主程式 Ctrl+C；主程式先清理並嘗試送零，再結束 ROS。
2. 確認最終命令為零且車輪停止，再關閉 rosserial、bridge、roscore。緊急狀況使用現場已確認的實體停止方式，不能只等軟體 timeout。
3. 檢查本次新產生的 `z_function/wheel_logs/wheel_status_latest.txt`。模式 1／4／7／8 的移動／參考路徑 CSV 則位於工作目錄 `wheel_logs/`；重跑前備份，詳見 [PC 控制架構](../04_程式碼說明/PC%20端控制架構.md)。
4. 實車驗收另記車號、韌體、Profile、相機、場地、命令／回授紀錄與停車結果。歷史影片或圖表不代表本版驗收完成。

## 常見問題與注意事項

- SSH 失敗：先查 IP、路由與網段。
- rosserial 失敗：查正確 port、權限、baud、連線占用、韌體與 `ros_lib` 版本。
- ROS1 有回授、ROS2 沒回授：查 bridge 型別配對、環境載入、ROS_DOMAIN_ID 與 QoS。
- Topic 有命令但沒有動作：先查訂閱者、韌體單位與動力，不直接提高 PWM／PI 參數。
- 不啟動多個 `main.py`，也不與 `test_PID.py` 同時控制同一台車；bridge 不是多來源命令仲裁器。

舊[實際執行影片](https://www.youtube.com/watch?v=L9N0wVhpF3s)保留作連線參考，影片中的 PWM 流程不代表新模式與韌體已驗證。本次文件更新未連接 ROS／相機／Arduino 實機。
