# MARK ROBOT / TWRB 交接手冊

這份文件整理 MARK ROBOT / TWRB 專題的安裝、環境設定、執行流程與控制程式說明。目標是讓接手的人可以從零開始完成系統建置，並知道遇到問題時要回到哪一份文件查。

> twrb 是上一位學長命名的自走車名稱，本交接文件沿用此名稱。

## 系統組成

| 模組          | 主要用途                                     |
| ------------- | -------------------------------------------- |
| PC            | 開發、執行 ROS2、ROS1、ros1_bridge、SSH 連線 |
| TWRB / 樹莓派 | 執行 ROS1 節點，連接 Arduino 與車體          |
| Arduino MEGA  | 接收 PWM topic，控制馬達                     |
| 路由器        | 固定 PC 與 TWRB 的 IP，讓 ROS / SSH 通訊穩定 |

## 建議閱讀順序

### 1. 系統安裝

第一次接手或重灌時，先完成硬體與作業系統安裝。

- [PC 端系統安裝](01_系統安裝/PC端系統安裝.md)
- [TWRB 樹莓派 OS 燒錄](01_系統安裝/twrb%20樹梅派%20OS%20燒錄.md)
- [Arduino MEGA 程式碼燒錄](01_系統安裝/Arduino%20MEGA%20程式碼燒錄.md)

### 2. 環境配置

PC 端需要同時準備 ROS1 Noetic、ROS2 Humble，並透過 ros1_bridge 讓兩邊 topic 可以互通。

- [ROS2 Humble 安裝](02_環境配置/ROS2%20Humble%20安裝.md)
- [ROS1 Noetic 安裝](02_環境配置/ROS1%20Neotic%20安裝.md)
- [ROS bridge 安裝](02_環境配置/ROS%20bridge%20安裝.md)
- [bashrc 設定](02_環境配置/bashrc%20設定.md)

### 3. 執行與程式說明

完成安裝後，照執行流程啟動各端節點；需要理解控制邏輯時，再看 controller.py 的說明。

- [1_Examples 執行流程](03_執行流程/1_Examples%20執行流程.md)
- [controller.py 詳解](04_程式碼說明/controller.py%20詳解.md)

### 4. 附錄

非必要，但在需要遠端桌面或排除樹莓派畫面問題時可以參考。

- [Ubuntu 20.04 安裝 VNC Server 教程](99_附錄/Ubuntu%2020.04%20安裝%20VNC%20Server%20教程.md)

## 網路設定參考

固定 IP 會讓 SSH 與 ROS 通訊穩定很多。下表是目前文件中的範例設定，可依實際路由器環境調整。

| 裝置名稱 | IP 位址       |
| -------- | ------------- |
| 路由器   | 192.168.0.1   |
| PC       | 192.168.0.102 |
| TWRB 1   | 192.168.0.105 |
| TWRB 2   | 192.168.0.106 |

## 外部資源

- [範例程式碼 Google Drive](https://drive.google.com/drive/folders/1THvN7rc72iKJG5PodxgnEvfE37KvztTp?usp=drive_link)
- [範例影片](https://youtu.be/L9N0wVhpF3s)

## 完成檢查

- PC 可以正常進入 Ubuntu 22.04。
- ROS2 Humble 可以執行基本指令。
- ROS1 Noetic 可以啟動 `roscore`。
- ros1_bridge 可以轉發 topic。
- PC 可以透過 SSH 連進 TWRB。
- Arduino MEGA 已燒錄控制程式。
- TWRB 端可以啟動 `serial_node.py`。

最後更新: 2026/1/20
