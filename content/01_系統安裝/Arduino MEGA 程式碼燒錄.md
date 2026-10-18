# Arduino MEGA 程式碼燒錄

## 目的

準備與所選 PC 模式相容的 Arduino 程式，確認版本與接線後再燒錄。模式 2 使用 PWM；模式 1、3、4、5、6、7、8 使用左右輪 RPS × 1000，不能只確認 Arduino IDE 顯示上傳成功。

**正式韌體來源待確認。** 本次依 `twrb_project` commit `98dba0d6` 查核，只找到 `底盤測試集/` 內多份 Arduino 測試文字檔，沒有足以確認目前車上燒錄版本的發布紀錄。不得直接選擇名稱含 `Best`、全功能或較高頻率的檔案。

## 前置條件

- 已安裝 Arduino IDE，且可選擇 Arduino Mega／Mega 2560 對應板型。
- 已向車輛維護者取得正式原始檔、版本／雜湊、車號及燒錄紀錄。
- 已核對實際 MEGA／TB6612／encoder 接線、電源、串列介面及緊急停止方式。
- 已確認韌體是接收 PWM 或 RPS，並取得相符的 ROS1 `ros_lib`。

## 操作步驟

### 1. 確認程式碼來源

原手冊的 [Google Drive](https://drive.google.com/drive/folders/1THvN7rc72iKJG5PodxgnEvfE37KvztTp?usp=drive_link) 中 `MoveControlDemoROS_v3.ino` 保留為**舊 PWM 參考**，本次沒有確認其內容或車上部署狀態。

輪速程式盤點、腳位與參數比較見 [Arduino 輪速閉迴路控制](../04_程式碼說明/Arduino%20輪速閉迴路控制.md)。若維護者確認某份 `.txt` 是指定來源，才將副本另存到 Arduino sketch 目錄中的同名 `.ino`；不要修改原始專案的歷史檔。保存完整來源與雜湊，避免只留下另一份無版本的新檔案。

正式版本尚未確認時，先完成 libraries 與通訊環境準備，保留燒錄為待確認步驟。

### 2. 準備 ros_lib

查核的輪速範例依賴 `ros.h`、`std_msgs/Int16MultiArray.h`、`std_msgs/Int32MultiArray.h` 與標準 `math.h`，沒有使用外加 Encoder／PID library。這只適用已讀取的範例，正式檔仍需逐一核對 `#include`。

在已安裝 ROS1 與 `rosserial_arduino` 的 Ubuntu 環境中，指定一個**新的輸出目錄**：

```bash
source /opt/ros/noetic/setup.bash
mkdir -p ~/twrb_ros_lib_export
rosrun rosserial_arduino make_libraries.py ~/twrb_ros_lib_export
```

輸出會包含 `ros_lib`，將整個資料夾放到 Arduino IDE 偏好設定所指定 sketchbook 的 `libraries/` 下。Windows 舊環境範例為：

```text
C:\Users\user\OneDrive\Documents\Arduino\libraries\ros_lib
```

不要同時保留會被 IDE 混用的不同版本 `ros_lib`。舊 Drive 副本僅可在已確認與 rosserial 版本及訊息定義相符時使用。產生器的路徑參數與既有輸出覆寫行為可核對 [rosserial 官方程式](https://github.com/ros-drivers/rosserial/blob/noetic-devel/rosserial_arduino/src/rosserial_arduino/make_libraries.py)。

### 3. 編譯與燒錄

1. 保存韌體來源、雜湊與 libraries 版本；馬達動力先隔離，避免啟動造成移動。
2. 關閉會占用同一 serial port 的 `serial_node.py` 或序列監控工具。
3. Arduino IDE 開啟已確認的 sketch，選擇實際 MEGA 板型、處理器及 port，先 Verify／Compile。
4. 確認無編譯錯誤後 Upload，記錄結果。以下圖片為既有 IDE 操作示意，不是目前韌體版本證明。

![](../assets/Arduino%20MEGA%20程式碼燒錄/file-20260114171016387.png)

## 驗證方式

1. **編譯／上傳**：IDE 無 compile／upload error；這只證明建置及傳輸成功。
2. **串列協商**：依[執行流程](../03_執行流程/1_Examples%20執行流程.md)開 rosserial，port 與 baud 必須符合正式韌體。歷史輪速範例是 115200，不能直接認定所有車都相同。
3. **Topic 契約**：以 [ROS 通訊協定](../04_程式碼說明/ROS%20Topic%20與通訊協定.md)核對 subscriber／publisher、左右順序、型別及單位；RPS 範例不訂閱 `/set_pwm`。
4. **停止優先**：車輪離地且實體急停已確認後，先測 `[0, 0]`，觀察回授 target／PWM／actual，再依現場核准方式驗證方向及斷線停車。

## 常見問題與注意事項

- 找不到 `ros.h`：核對 IDE 的實際 sketchbook 路徑與 `ros_lib` 是否多包了一層。
- 無法上傳：核對 board／port、占用程序與連接介面，不要先改動馬達參數。
- ROS 有 Topic 但車不動：核對韌體接收的命令單位，不能只把 `/set_pwm` 改名成 `/cmd_wheel_rps_x1000`。
- 實際接線、CPR、PI／PID、輪速／PWM 限制與 timeout 尚未確認時，保留「待確認」，不可據測試檔宣稱現行值。

本次文件更新沒有執行 Arduino 編譯、燒錄或馬達測試，以上實機驗證均待現場完成。
