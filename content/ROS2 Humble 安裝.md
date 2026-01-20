> Humble 官方支援 ubuntu 22.04 系統版本, 依照 [官方下載文檔](https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debs.html) 流程下載，內容會更新變動，此說明書安裝指令僅供參考 

#### YT 安裝教學連結 [🔗](https://youtu.be/0aPbWsyENA8?si=wNByG4C1D9lLXxQ8)

---

1. 確認有 UTF-8 編碼
```bash
locale
```

2. 開啟 Universe 倉庫
```bash
sudo apt install software-properties-common
sudo add-apt-repository universe
```


3. 安裝
```
# 更新系統套件清單，並安裝 curl 下載工具 (如果還沒裝的話)
sudo apt update && sudo apt install curl -y

# 從 GitHub API 自動抓取最新的版本號 (tag_name)，並存入變數中
export ROS_APT_SOURCE_VERSION=$(curl -s https://api.github.com/repos/ros-infrastructure/ros-apt-source/releases/latest | grep -F "tag_name" | awk -F\" '{print $4}')

# 根據抓到的版本號以及您的 Ubuntu 代號 (如 jammy)，下載對應的 .deb 安裝檔到 /tmp 資料夾
curl -L -o /tmp/ros2-apt-source.deb "https://github.com/ros-infrastructure/ros-apt-source/releases/download/${ROS_APT_SOURCE_VERSION}/ros2-apt-source_${ROS_APT_SOURCE_VERSION}.$(. /etc/os-release && echo ${UBUNTU_CODENAME:-${VERSION_CODENAME}})_all.deb"

# 安裝剛剛下載的 .deb 檔 (這會自動將 ROS 的官方軟體源加入您的系統)
sudo dpkg -i /tmp/ros2-apt-source.deb
```

## ros2 核心安裝

1. 系統更新
```bash
sudo apt update
sudo apt upgrade
```

2. Desktop Install (推薦): 包含 ROS 核心、RViz (視覺化工具)、Demo、教學範例。
```bash
sudo apt install ros-humble-desktop
```

3. 安裝開發工具
>這是為了以後您自己寫程式（建立 Workspace）時需要的編譯工具。
```bash
sudo apt install ros-dev-tools
```


4. 設定環境變數（讓系統每次都找得到 ROS）
>.bashrc 加入以下兩行
```bash
# 設定環境變數（讓系統每次都找得到 ROS）
source /opt/ros/humble/setup.bash
```

5. 顯示當前系統所有的環境變數中，跟 ROS 有關的設定
> 確認 ROS 是否已加入環境 
```bash
printenv | grep ROS
```



