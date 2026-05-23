# ROS2 Humble 安裝

## 目的

在 PC 端 Ubuntu 22.04 環境中安裝 ROS2 Humble，作為專題主要開發與通訊環境之一。

## 前置條件

- Ubuntu 22.04 已安裝完成。
- PC 可連上網路。
- 已具備 sudo 權限。

> Humble 官方支援 Ubuntu 22.04。官方文件可能更新，本文件指令作為交接參考；若遇到差異，以 [ROS2 Humble 官方安裝文件](https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debs.html) 為準。

## 操作步驟

### 1. 確認 UTF-8 編碼

```bash
locale
```

### 2. 開啟 Universe 倉庫

```bash
sudo apt install software-properties-common
sudo add-apt-repository universe
```

### 3. 加入 ROS2 apt source

```bash
sudo apt update && sudo apt install curl -y
export ROS_APT_SOURCE_VERSION=$(curl -s https://api.github.com/repos/ros-infrastructure/ros-apt-source/releases/latest | grep -F "tag_name" | awk -F\" '{print $4}')
curl -L -o /tmp/ros2-apt-source.deb "https://github.com/ros-infrastructure/ros-apt-source/releases/download/${ROS_APT_SOURCE_VERSION}/ros2-apt-source_${ROS_APT_SOURCE_VERSION}.$(. /etc/os-release && echo ${UBUNTU_CODENAME:-${VERSION_CODENAME}})_all.deb"
sudo dpkg -i /tmp/ros2-apt-source.deb
```

### 4. 安裝 ROS2 Humble Desktop

```bash
sudo apt update
sudo apt upgrade
sudo apt install ros-humble-desktop
```

### 5. 安裝開發工具

```bash
sudo apt install ros-dev-tools
```

### 6. 設定環境變數

如果只需要單純使用 ROS2，可在 `.bashrc` 加入：

```bash
source /opt/ros/humble/setup.bash
```

本專題需要切換 ROS1 / ROS2 / bridge mode，建議最後改用 [bashrc 設定](bashrc%20設定.md) 中的環境選擇器。

## 驗證方式

確認 ROS 相關環境變數：

```bash
printenv | grep ROS
```

也可以開啟兩個 terminal 測試 ROS2 talker / listener。

## 相關連結

- [ROS2 Humble 官方安裝文件](https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debs.html)
- [YouTube 安裝教學](https://youtu.be/0aPbWsyENA8?si=wNByG4C1D9lLXxQ8)
