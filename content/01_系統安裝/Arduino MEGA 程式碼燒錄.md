# Arduino MEGA 程式碼燒錄

## 目的

將自走車控制程式燒錄到 Arduino MEGA，讓 Arduino 可以透過 ROS topic 接收 PWM 指令並控制馬達。

## 前置條件

- 已安裝 Arduino IDE。
- 已取得範例程式碼 `MoveControlDemoROS_v3.ino`。
- 已準備或產生 `ros_lib`。

## 操作步驟

### 1. 下載程式碼

範例程式碼放在 [Google Drive](https://drive.google.com/drive/folders/1THvN7rc72iKJG5PodxgnEvfE37KvztTp?usp=drive_link)，檔名為 `MoveControlDemoROS_v3.ino`。

### 2. 準備 ros_lib

方法一：在已安裝 ROS 的 Ubuntu PC 上產生。

```bash
rosrun rosserial_arduino make_libraries.py
```

方法二：從 [Google Drive](https://drive.google.com/drive/folders/1THvN7rc72iKJG5PodxgnEvfE37KvztTp?usp=drive_link) 下載現成的 `ros_lib`。

取得 `ros_lib` 後，將整個資料夾放到 Arduino libraries 路徑。Windows 參考路徑如下：

```text
C:\Users\user\OneDrive\Documents\Arduino\libraries
```

### 3. 燒錄程式碼

開啟 Arduino IDE，選擇 Arduino MEGA 對應的 board 與 port，接著燒錄 `MoveControlDemoROS_v3.ino`。

![](../assets/Arduino%20MEGA%20程式碼燒錄/file-20260114171016387.png)

## 驗證方式

燒錄成功後，Arduino IDE 不應顯示 compile 或 upload error。後續啟動 TWRB 端 `serial_node.py` 時，Arduino 應能透過 rosserial 與 ROS topic 溝通。

## 注意事項

程式中這行會訂閱 PWM 控制 topic：

```cpp
ros::Subscriber<std_msgs::Int16MultiArray> pwm_sub("set_pwm", pwmCallback);
```

`set_pwm` 是目前使用的 topic 名稱，如果專案端 topic 有調整，這裡也要同步修改。
