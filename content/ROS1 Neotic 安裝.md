>Ubuntu 22.04 後續只支援的 ROS2 發行版。如果想在22.04使用ROS Noetic就只能自己編譯ros noetic原始碼了 (超級麻煩)


>有人編譯好並通過一些壓縮方式發行到 ubuntu 支援的第三方軟體源上，任何用戶可通過軟體源添加然後使用apt install命令安裝該軟體了(此教學方法)


```bash
# 1、添加軟件源
echo "deb [trusted=yes arch=amd64] http://deb.repo.autolabor.com.cn jammy main" | sudo tee /etc/apt/sources.list.d/autolabor.list
# 2、更新軟件倉庫
sudo apt update
# 3、安裝
sudo apt install ros-noetic-autolabor
```

