# Ubuntu 20.04 安裝 VNC Server 教程

## 目的

在樹莓派或 Ubuntu 20.04 環境中安裝 VNC Server，方便需要圖形化遠端桌面時使用。此功能非必要，SSH 已足夠完成大多數操作。

## 前置條件

- Ubuntu 20.04 環境。
- 可使用 terminal。
- 已具備 sudo 權限。

## 操作步驟

### 1. 安裝 Xfce 桌面環境

```bash
sudo apt install xfce4 xfce4-goodies
```

### 2. 安裝 TightVNC Server

```bash
sudo apt install tightvncserver
```

### 3. 啟動 VNC Server 並設定密碼

```bash
vncserver
```

### 4. 設定啟動腳本

建立或編輯 `~/.vnc/xstartup`：

```bash
nano ~/.vnc/xstartup
```

貼上以下內容，強制 VNC 使用 XFCE 環境，避免 GNOME 造成灰畫面：

```bash
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

### 5. 賦予執行權限

```bash
chmod +x ~/.vnc/xstartup
```

## 常用指令

查詢 VNC 狀態：

```bash
sudo systemctl status vncserver@1.service
```

重啟 VNC 服務：

```bash
sudo systemctl restart vncserver@1.service
```

停止 VNC 服務：

```bash
sudo systemctl stop vncserver@1.service
```

## 注意事項

若要修改解析度，調整 `/etc/systemd/system/vncserver@.service` 裡面的 `-geometry` 數值，接著執行：

```bash
sudo systemctl daemon-reload
sudo systemctl restart vncserver@1.service
```
