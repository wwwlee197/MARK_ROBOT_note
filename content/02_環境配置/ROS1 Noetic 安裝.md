---
aliases:
  - 02_環境配置/ROS1 Neotic 安裝
---

# ROS1 Noetic 安裝

## 目的

在 Ubuntu 22.04 上準備 ROS1 Noetic，讓 PC 端可以與 TWRB / Arduino 端既有 ROS1 流程相容。

## 前置條件

- Ubuntu 22.04 已安裝完成。
- PC 可連上網路。
- 已具備 sudo 權限。

## 背景說明

Ubuntu 22.04 主要支援 ROS2 發行版。若要在 Ubuntu 22.04 使用 ROS Noetic，通常需要自行編譯 ROS Noetic 原始碼，流程較繁瑣。

本文件使用第三方軟體源提供的 `ros-noetic-autolabor` 套件，透過 apt 安裝 ROS1 Noetic。

## 操作步驟

### 1. 加入第三方軟體源

```bash
echo "deb [trusted=yes arch=amd64] http://deb.repo.autolabor.com.cn jammy main" | sudo tee /etc/apt/sources.list.d/autolabor.list
```

### 2. 更新軟體倉庫

```bash
sudo apt update
```

### 3. 安裝 ROS1 Noetic

```bash
sudo apt install ros-noetic-autolabor
```

## 驗證方式

安裝後可開啟 ROS1 環境並啟動 master：

```bash
source /opt/ros/noetic/setup.bash
roscore
```

若 `roscore` 可以正常啟動，代表 ROS1 基本環境可用。

## 注意事項

本文件採用第三方套件源，後續若套件源失效或版本異動，需要重新評估安裝方式。
