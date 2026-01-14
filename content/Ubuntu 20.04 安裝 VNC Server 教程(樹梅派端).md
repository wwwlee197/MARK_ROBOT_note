1. 安装Xfce桌面环境：
```
sudo apt install xfce4 xfce4-goodies
```
2. 安装TightVNC服务器：
```
sudo apt install tightvncserver
```

3. 啟動VNC伺服器並設定訪問密碼：
```
vncserver
```

4. 配置啟動腳本 (解決灰畫面關鍵 ⚠️)
>要強制 VNC 使用 XFCE 環境，避開 GNOME 的干擾。

**建立新檔**
```
nano ~/.vnc/xstartup
```

**貼上以下內容**
```Bash
#!/bin/sh

# 解除 GNOME 干擾
unset SESSION_MANAGER
unset DBUS_SESSION_BUS_ADDRESS

# 載入 X 資源
[ -x /etc/vnc/xstartup ] && exec /etc/vnc/xstartup
[ -r $HOME/.Xresources ] && xrdb $HOME/.Xresources

# 設定背景並啟動 XFCE4
xsetroot -solid grey
vncconfig -iconic &
startxfce4 &
```

**賦予執行權限 (重要)：**
```
chmod +x ~/.vnc/xstartup
```

- **查詢 VNC 狀態：** `sudo systemctl status vncserver@1.service`
    
- **重啟 VNC 服務：** `sudo systemctl restart vncserver@1.service`
    
- **停止 VNC 服務：** `sudo systemctl stop vncserver@1.service`
    
- **修改解析度：** 修改 `/etc/systemd/system/vncserver@.service` 裡面的 `-geometry` 數值，然後執行 `systemctl daemon-reload` 再重啟服務。