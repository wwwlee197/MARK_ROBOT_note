
自走車 twrb 是上一位學長所命名，此教學繼續沿用，是一切的起點


# 1. 系統安裝
#### [PC端系統安裝](PC端系統安裝.md)
>進入此章節前，先將筆電資料備份，準備好外接SSD(建議256GB🔼)，USB(16GB)
系統安裝完後 vscode，新酷音輸入法，你習慣的設定都去搞定吧~

#### [twrb 樹梅派 OS 燒錄](twrb%20樹梅派%20OS%20燒錄.md)

#### [Arduino MEGA 程式碼燒錄](Arduino%20MEGA%20程式碼燒錄.md)

---
# 2. 環境配置

## ---- PC 端 ----
 >此章皆需要安裝 ros1 noetic，ros2 humble
> 將ros1 & ros2 通訊搭起的橋樑 ros_bridge
#### [ROS2 Humble 安裝](ROS2%20Humble%20安裝.md)

#### [ROS1 Neotic 安裝](ROS1%20Neotic%20安裝.md)

#### [ROS bridge 安裝](ROS%20bridge%20安裝.md)

#### [bashrc 設定](bashrc%20設定.md)

## ---- 樹梅派 端 ----
>第一次燒錄好，連接與 PC 同一網段才可以用 SSH
>- 方法一: 有外接螢幕 + mini HDMI 線
>- 方法二: 將樹梅派連接上路由器，使用 [angry IP](https://angryip.org/) 查看樹梅派IP，SSH 連線進去設定NetworkManager 連接 wifi，再去設定固定 IP 即可

#### [Ubuntu 20.04 安裝 VNC Server 教程(非必要)](Ubuntu%2020.04%20安裝%20VNC%20Server%20教程(非必要).md)

## ---- 路由器端 ----
>設定固定 IP 會方便很多，說明範例設定如下表，僅供參考不必完全相同

| 裝置名稱   | IP 位址         |
| ------ | ------------- |
| 路由器    | 192.168.0.1   |
| PC     | 192.168.0.102 |
| TWRB 1 | 192.168.0.105 |
| TWRB 2 | 192.168.0.106 |



---

# 3. 程式碼

範例程式碼下載 [1_Examples](https://drive.google.com/drive/folders/1THvN7rc72iKJG5PodxgnEvfE37KvztTp?usp=drive_link)

範例影片 [🔗](https://youtu.be/L9N0wVhpF3s)



---


最後更新: 2026/1/20

