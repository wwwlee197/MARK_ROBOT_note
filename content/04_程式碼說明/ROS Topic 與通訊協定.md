# ROS Topic 與通訊協定

## 目的與前置條件

集中列出正式 PC 程式的 Topic、命令單位、歷史輪速韌體的回授契約，以及逐段診斷方法。版本為 `twrb_project` commit `98dba0d6`，來源索引見[查核紀錄](../99_附錄/程式版本更新查核紀錄.md)。車上韌體仍待確認；下列回授排列須與實際來源核對。

先完成 ROS1／ROS2／bridge 環境、網路與 rosserial 設定，啟動順序見[執行流程](../03_執行流程/1_Examples%20執行流程.md)。同一台車一次只允許一個控制來源。

## 資料傳遞方向

```text
PC：main.py／ROS2 command publisher
  → PC：ros1_bridge（ROS2 → ROS1）
  → 樹莓派：ROS1 rosserial_python serial_node.py
  → UART 或 USB serial → Arduino：ros_lib subscriber

Arduino：encoder／控制狀態 publisher
  → serial → 樹莓派：rosserial
  → PC：ros1_bridge（ROS1 → ROS2）
  → WheelStatusAdapter／監看工具
```

ROS1 master 用於節點發現，不是每筆馬達資料都經 master 轉送。SSH 只用來遠端啟動樹莓派指令。rosserial 負責 ROS1 與 MCU 的串列封包；ROS1／ROS2 型別轉換由 ros1_bridge 負責，兩者不能互相取代。原有 ROS 安裝與網路教學仍適用此部署關係。

## Topic 契約

| Topic                  | ROS2 型別                       | 資料／方向                                           | 適用範圍                                                            |
| ---------------------- | ------------------------------- | ---------------------------------------------------- | ------------------------------------------------------------------- |
| `/cmd_wheel_rps_x1000` | `std_msgs/msg/Int16MultiArray`  | `[左目標RPS×1000, 右目標RPS×1000]`，PC → MCU         | 模式 1、3、4、5、6、7、8                                            |
| `/set_pwm`             | `std_msgs/msg/Int16MultiArray`  | `[左PWM, 右PWM]`，PC → 相容底盤韌體                  | 模式 2                                                              |
| `/set_pwm_01`          | `std_msgs/msg/Int16MultiArray`  | PWM；Gateway 支援此名稱                              | 現行 8 份 Profile 沒有選用，不代表第二台車已配置                    |
| `/wheel_status_x1000`  | `std_msgs/msg/Int32MultiArray`  | 目標／實測輪速、PWM、counts、時間，MCU → PC          | 正式 RPS Node 的 `WheelStatusAdapter` 訂閱；模式 2 未建立此 adapter |
| `/robot_pose`          | `geometry_msgs/msg/PoseStamped` | 世界位置 m、姿態 quaternion，PC → RViz               | 模式 2，`RvizTfAdapter`                                             |
| `/tf`、`/tf_static`    | `tf2_msgs/msg/TFMessage`        | `map → base_link` 動態、`base_link → laser` 靜態變換 | 模式 2 的 TF broadcaster                                            |

ROS1 對應型別移除 `/msg`，例如 `std_msgs/Int16MultiArray`。`/bridge_feedback` 在本次查核的正式 PC 路徑與輪速範例未找到對應契約，舊手冊的此項不再作為驗收條件。

### 單位與左右順序

`WheelCommand` 在 Python 內保存浮點 RPS；只有 `WheelCommandPublisher.publish` 轉成 `int(RPS × 1000)`。例如 `[500, -500]` 的含義是左 +0.5 RPS、右 −0.5 RPS，**不是建議直接執行的測試值**。PWM 沒有 m/s 或 RPS 的固定換算，禁止把同一陣列從一種 Topic 複製到另一種。

RPS 乘 `2π` 才是輪軸 rad/s；再乘輪半徑才是輪緣 m/s。Core 內的車體角速度 `omega_radps` 亦不等於左右輪 RPS。零陣列都表示零命令，但實際停止仍取決於韌體、通訊與機械慣性。

正式 RPS Gateway 的 queue size 為 5，整數超過 Int16 範圍時改送零；PWM Gateway 依 `max_abs_pwm` 檢查，模式 2 為 120，超限亦送零。這些是 PC 的命令保護，不代表手動 ROS publisher 也會經過同一防線。

## Wheel Status 的 11 欄排列

以下來自歷史 `底盤測試集/底盤測試_3/PID_test_3_2_控制迴圈0.05改0.025/arduino_code.txt::publishStatus`：

| data index | 意義                        | 單位／解讀                                            |
| ---------- | --------------------------- | ----------------------------------------------------- |
| 0、1       | target left、target right   | RPS × 1000，除以 1000 才是 RPS                        |
| 2、3       | actual left、actual right   | encoder 算出的 RPS × 1000                             |
| 4、5       | pwm left、pwm right         | 邏輯 PWM 命令；不是轉速                               |
| 6、7       | dtheta left、dtheta right   | 該控制取樣區間的 encoder counts，非 rad               |
| 8、9       | encoder left、encoder right | encoder 累積 counts                                   |
| 10         | timestamp                   | Arduino 開機後 `millis()`，非 PC 時間或 ROS timestamp |

`Int32MultiArray` 的欄位名稱與單位並未編碼在訊息本身，必須依韌體來源解讀。11 欄也沒有車號、韌體版號、命令序號或確認回覆；收到目標值不等於已到達該實測輪速。舊 10 欄資料沒有第 10 項時間戳，不能直接推算實際取樣時間。

## 實際操作與驗證

下列 ROS1 與 ROS2 指令需在各自已載入環境的 terminal 執行。

### 1. 先查型別與連線

```bash
# ROS1
rostopic list
rostopic type /cmd_wheel_rps_x1000
rostopic info /cmd_wheel_rps_x1000
rostopic info /wheel_status_x1000

# ROS2
ros2 topic list -t
ros2 topic info /cmd_wheel_rps_x1000 --verbose
ros2 topic info /wheel_status_x1000 --verbose
```

確認 Topic 名稱、型別、publisher／subscriber 及 ROS2 QoS。bridge 本身也會出現在端點列表，看到 Topic 不代表 Arduino 已連上；需同時看 rosserial 的協商狀態與回授。

### 2. 觀察回授與頻率

```bash
# ROS1：先證明串列側有資料
rostopic echo /wheel_status_x1000
rostopic hz /wheel_status_x1000

# ROS2：再證明資料跨過 bridge
ros2 topic echo /wheel_status_x1000 std_msgs/msg/Int32MultiArray
ros2 topic hz /wheel_status_x1000
ros2 topic echo /cmd_wheel_rps_x1000 std_msgs/msg/Int16MultiArray
```

若 ROS1 有資料、ROS2 沒有，檢查 bridge 的 `--print-pairs` 是否包含 Int16／Int32 MultiArray 配對，以及 domain／QoS。若 ROS1 也沒有資料，先查電源、serial port、baud、rosserial 協商及韌體，無須先調控制器。

若 echo 顯示 QoS 不相容，可依 `ros2 topic info --verbose` 的實際設定調整監看端，例如使用 `--qos-reliability best_effort`；這是 **ros2 topic echo** 的選項，不是通用的 dynamic_bridge 選項。

### 3. 零命令與非零測試邊界

確認導航與其他測試 publisher 已退出後，依韌體支援的單位擇一：

```bash
# ROS2：輪速模式
ros2 topic pub --once /cmd_wheel_rps_x1000 std_msgs/msg/Int16MultiArray '{data: [0, 0]}'

# ROS2：模式 2 的 PWM 相容韌體
ros2 topic pub --once /set_pwm std_msgs/msg/Int16MultiArray '{data: [0, 0]}'

# ROS1：可直接隔離 ROS2／bridge 問題，使用輪速相容韌體
rostopic pub -1 /cmd_wheel_rps_x1000 std_msgs/Int16MultiArray '{data: [0, 0]}'
```

輪速回授應顯示 target 為零、PWM 歸零，實測輪速隨機械停止降至零。若 `--once` 等待訂閱者或沒有回授，不能將終端成功退出當作安全停車證明。

非零測試只能在版本、接線、急停已確認且車輪離地後，以現場核准值進行。以下是 **Bash 範本**；必須先設定兩個整數變數才會送出：

```bash
# LEFT_X1000／RIGHT_X1000 由現場核准值設定，單位為 RPS × 1000
ros2 topic pub --rate 10 /cmd_wheel_rps_x1000 std_msgs/msg/Int16MultiArray \
  "{data: [${LEFT_X1000:?請先設定核准的左輪命令}, ${RIGHT_X1000:?請先設定核准的右輪命令}]}"
```

10 Hz 是此診斷範本的發送率，不是導航或 MCU 控制頻率。結束連續發布後須另送零；只按 Ctrl+C 關閉 `ros2 topic pub` 不保證會送零。模式 2 的非零測試則必須使用 `/set_pwm` 與核准的 PWM 值，不能套用上面輪速變數。

### 4. 正常離開與緊急停止

正常離開先按導航視窗 ESC 或主程式 Ctrl+C，觀察最終零命令及回授，再關 rosserial、bridge、roscore。手動 publisher 與其他控制程式必須一起停止，否則後續非零命令可能覆蓋零命令。

若車體失控或軟體停止無效，使用已在現場確認的實體急停／切斷馬達動力方式；此 Repository 沒有足夠資料可指定急停開關腳位。斷線後 PC 無法保證零命令送達，歷史範例的 3 秒 timeout 不是正式車輛的保證，也不是適合等待的緊急煞停時間。

## 紀錄與常見問題

`WheelStatusAdapter` 只把 `message.data` 傳給 `WheelStatusTxtLogger`，不核對回授長度、不作通訊 watchdog 或 encoder 故障裁決。正式輪速模式預設寫入 `z_function/wheel_logs/wheel_status_latest.txt`；建立 logger 會覆寫 latest，並刪除同目錄舊 `wheel_status_*.txt` 與 SVG，重跑前先備份。

tracking CSV 則在工作目錄 `wheel_logs/`，詳細差異見 [PC 控制架構](PC%20端控制架構.md)。有回授、圖表或 CSV 並不等於閉迴路或實車導航已驗收。

此部署採用官方 [ros1_bridge](https://github.com/ros2/ros1_bridge) 與 [rosserial](https://github.com/ros-drivers/rosserial) 的不同通訊邊界；兩邊 topic 型別相同、單位正確且資料確實往返，才算完成通訊層驗證。
