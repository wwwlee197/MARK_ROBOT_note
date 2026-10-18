# MARK ROBOT / TWRB 交接手冊

這份文件整理 MARK ROBOT / TWRB 專題的安裝、環境設定、執行流程與控制程式說明。目標是讓接手的人可以從零開始完成系統建置，並知道遇到問題時要回到哪一份文件查。

> twrb 是上一位學長命名的自走車名稱，本交接文件沿用此名稱。

## 系統組成

| 模組          | 主要用途                                          |
| ------------- | ------------------------------------------------- |
| PC            | 開發、執行 ROS2、ROS1、ros1_bridge、SSH 連線      |
| TWRB / 樹莓派 | 執行 ROS1 節點，連接 Arduino 與車體               |
| Arduino MEGA  | 依相容韌體接收 PWM 或輪速命令；正式燒錄版本待確認 |
| 路由器        | 固定 PC 與 TWRB 的 IP，讓 ROS / SSH 通訊穩定      |

## 建議閱讀順序

### 1. 系統安裝

第一次接手或重灌時，先完成硬體與作業系統安裝。

- [PC 端系統安裝](01_系統安裝/PC端系統安裝.md)
- [TWRB 樹莓派 OS 燒錄](01_系統安裝/twrb%20樹莓派%20OS%20燒錄.md)
- [Arduino MEGA 程式碼燒錄](01_系統安裝/Arduino%20MEGA%20程式碼燒錄.md)

### 2. 環境配置

PC 端需要同時準備 ROS1 Noetic、ROS2 Humble，並透過 ros1_bridge 讓兩邊 topic 可以互通。

- [ROS2 Humble 安裝](02_環境配置/ROS2%20Humble%20安裝.md)
- [ROS1 Noetic 安裝](02_環境配置/ROS1%20Noetic%20安裝.md)
- [ROS bridge 安裝](02_環境配置/ROS%20bridge%20安裝.md)
- [bashrc 設定](02_環境配置/bashrc%20設定.md)

### 3. 執行與程式說明

完成安裝後，先讀執行流程與通訊單位，再了解現行 PC 架構與底盤閉迴路。模式 2 使用 PWM，其餘正式模式使用 RPS × 1000；Arduino 韌體必須與模式相容。

- [1_Examples 執行流程](03_執行流程/1_Examples%20執行流程.md)
- [ROS Topic 與通訊協定](04_程式碼說明/ROS%20Topic%20與通訊協定.md)
- [PC 端控制架構與模式比較](04_程式碼說明/PC%20端控制架構.md)
- [Arduino 輪速閉迴路控制](04_程式碼說明/Arduino%20輪速閉迴路控制.md)
- [controller.py 歷史說明與現行模式 2](04_程式碼說明/controller.py%20詳解.md)

### 4. 附錄

版本查核紀錄列出技術依據、來源矛盾與待確認事項，實車操作前應先閱讀；VNC 則是非必要的遠端桌面補充。

- [程式版本更新查核紀錄](99_附錄/程式版本更新查核紀錄.md)
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
- Arduino MEGA 的正式韌體來源已確認，且與所選 PWM／RPS 模式相容。
- TWRB 端可以啟動 `serial_node.py`。
- 命令單位、左右輪方向與回授排列已核對；停止、逾時及煞停距離已有本車實測紀錄。

以上是接手者的驗收清單，並非本次文件更新已完成的實機測試。技術內容依 `twrb_project` 主分支 commit `98dba0d6` 查核，正式韌體與硬體驗收仍有待確認項目。

最後更新: 2026/10/18
