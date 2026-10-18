# Arduino 輪速閉迴路控制

## 目的與前置條件

說明 PC 的左右輪目標 RPS 如何在 Arduino 端轉成 PWM，並協助接手者辨識歷史測試程式。**目前無法從 Repository 確認正式燒錄版本；本頁不是車上韌體的認證規格。** 來源版本與待確認事項見[查核紀錄](../99_附錄/程式版本更新查核紀錄.md)。

閱讀前先理解 [PWM 與 RPS 通訊單位](ROS%20Topic%20與通訊協定.md)。燒錄請回到 [Arduino MEGA 程式碼燒錄](../01_系統安裝/Arduino%20MEGA%20程式碼燒錄.md)，先取得車輛維護者確認的原始檔與接線資料。

## 韌體來源盤點

本次查核未找到可作正式入口的 `.ino` 或逐車燒錄版本清單；找到的是 `底盤測試集/` 下的 Arduino 程式文字檔。原手冊提到的 Drive `MoveControlDemoROS_v3.ino` 是舊 PWM 參考來源，不能據此認定支援新輪速協定。

下表路徑都相對於 `twrb_project/底盤測試集/`，數值來自各檔實際常數，**僅描述歷史檔案**：

| 程式路徑                                                       | CPR／計數                 | 控制週期門檻                               | 左／右 `(Kp, Ki, Kd)`    | PWM slew     |
| -------------------------------------------------------------- | ------------------------- | ------------------------------------------ | ------------------------ | ------------ |
| `底盤測試_1/PID_test_5___Best/arduino_code.txt`                | 330，A 相 RISING          | 0.05 s，名目 20 Hz                         | `(55,22,0)`／`(55,26,0)` | 12／控制週期 |
| `底盤測試_2/PID_test_2_8_全功能測試/arduino_code.txt`          | 390，A 相 RISING          | 0.05 s，名目 20 Hz                         | `(90,24,0)`／`(90,32,0)` | 35／控制週期 |
| `底盤測試_3/PID_test_3_1_Ki=0/arduino_code.txt`                | 390，A 相 RISING          | 0.05 s，名目 20 Hz                         | `(90,0,0)`／`(90,0,0)`   | 35／控制週期 |
| `底盤測試_3/PID_test_3_2_控制迴圈0.05改0.025/arduino_code.txt` | 390，A 相 RISING          | 0.025 s，名目 40 Hz                        | `(90,24,0)`／`(90,32,0)` | 35／控制週期 |
| `底盤測試_4倍頻_120hz失敗案例/arduino.txt`                     | 1560，A/B CHANGE 與狀態表 | 8333 µs，約 120 Hz；狀態另以 50000 µs 排程 | `(90,24,0)`／`(90,32,0)` | 6／控制週期  |

`底盤測試_4倍頻/` 可見量測 log／SVG，但不能只靠這些檔案證明它們對應哪份正式韌體。`Best`、全功能、失敗案例等名稱亦不代表發布／驗收狀態。其餘測試 1／2 的中間版本有不同 CPR、增益與 slew，調查舊數據時必須配對同一目錄來源。

## 具名歷史範例的腳位與處理流程

以下只以表中的 **`PID_test_3_2_控制迴圈0.05改0.025/arduino_code.txt`** 作解說範例，不代表它是正式燒錄版本。

### MEGA／TB6612 與 Encoder

| 該檔定義                  | MEGA pin | 程式用途                                |
| ------------------------- | -------- | --------------------------------------- |
| `PWMA / AIN1 / AIN2`      | 4／6／5  | M1 左輪 PWM／方向                       |
| `PWMB / BIN1 / BIN2`      | 8／10／9 | M2 右輪 PWM／方向                       |
| `STBY`                    | 7        | 初始化時設 HIGH                         |
| `Pin_Encoder_Left_A / B`  | 20／21   | 左 encoder；A 上升緣中斷、讀 B 決定增減 |
| `Pin_Encoder_Right_A / B` | 2／3     | 右 encoder；A 上升緣中斷、讀 B 決定增減 |

`setWheelPWM(left, right)` 會呼叫 `SetPWM(1, left)` 與 `SetPWM(2, -right)`。通訊層仍使用「左右輪前進為正」的邏輯，右側底層符號反轉不可再任意搬到 PC 命令上。**實際線路、正反方向與 CPR 均需離地驗證，不能以此表取代現場接線核對。**

### Encoder 輪速換算

`initEncoder()` 設定 A 相上升緣中斷。`loop()` 在暫停中斷期間讀取累積 counts，與前次相減後計算：

$$
RPS_{actual}=sign\times\frac{count_{now}-count_{previous}}{CPR\times dt}
$$

該檔 `CPR=390.0`、左右 `ENCODER_SIGN=1`；`dt` 是 `micros()` 量到的實際經過秒數，不固定代入 0.025。`dtheta` 在這裡是 **encoder counts**，不是角度 rad。更換為四倍頻後必須使用相符的 CPR，否則輪速估計會成比例偏差。

### 目標輪速與 PI 控制

```text
cmdWheelCallback：Int16 左右值 ÷ 1000 → target RPS
loop：encoder 差分／CPR／dt → actual RPS
computePIDPWM：目標－實測 → 基礎 PWM + PID 修正
              → PWM 上限 → PWM slew → TB6612
publishStatus：目標／實測／PWM／counts／時間 → ROS1
```

`computePIDPWM` 的基本計算為：

$$
e=RPS_{target}-RPS_{actual},\quad
I\leftarrow clip(I+e\,dt,-40,40),\quad
D=\frac{e-e_{previous}}{dt}
$$

$$
PWM_{target}=PWM_{base}+\operatorname{int}(K_p e+K_i I+K_d D)
$$

函式名稱是 PID，但此檔左右 `Kd=0`，實際使用 PI 加基礎 PWM。非零且目標絕對值低於 0.80 RPS 時基礎 PWM 為 25，其他非零目標為 40，符號跟隨方向。計算結果先限制在 ±200，再以每控制週期最多 35 PWM 漸變。底層 `SetPWM` 另有 ±255 防線，不能把它當成閉迴路設定的輸出上限。

`cmdWheelCallback` 少於兩個元素便忽略；有效命令以左、右順序除以 1000。單輪停止或方向反轉會清該輪 PWM 與 I／D 狀態；目標改變超過 0.30 RPS 時清 I／D。此範例沒有額外的目標 RPS 上限檢查，PC 正式控制器的 2.0 RPS 限制也不會保護外部手動 publisher。

### 通訊與安全保護

- `setup()` 設 `115200` baud，使用 `ros.h`、`std_msgs/Int16MultiArray.h`、`std_msgs/Int32MultiArray.h`；不是 micro-ROS。
- 訂閱相對名稱 `cmd_wheel_rps_x1000`，發布 `wheel_status_x1000`；未加 namespace／remap 時即為本手冊使用的根 Topic。
- `CONTROL_INTERVAL=0.025`，控制後發布 11 欄狀態，名目約 40 Hz；檔中 50 ms 註解落後於常數。實際頻率受通訊與迴圈負載影響。
- 超過 `CMD_TIMEOUT_MS=3000` 未收到有效命令，將目標歸零、reset PID、`stopAll()`；不是安全監督器確認了車已停。
- 零目標在下一次控制處理時直接令 PWM 為零；逾時停止亦不經 slew。零 PWM 的實際煞停距離仍需量測。
- 沒有看到 encoder 故障／馬達堵轉的專用 fault gate，也沒有命令回覆序號或正式版本識別欄位。不能將此範例描述為所有硬體故障都會停車。

完整 11 欄排列見 [ROS Topic 與通訊協定](ROS%20Topic%20與通訊協定.md)。其他歷史版本可能使用 10 欄或不同發布週期，不可用此範例覆蓋全部數據。

## 實際驗證與資料分析

1. 先記錄車號、已確認韌體原始檔與雜湊、板型、libraries、接線、baud、CPR、控制與狀態週期。未能確認時只做離線查核，不直接燒錄測試檔。
2. 確認實體停止方式，車輪離地；先依[執行流程](../03_執行流程/1_Examples%20執行流程.md)測零命令與回授，再以現場核准的小命令逐輪檢查方向、encoder 正負號與 counts。
3. 比較 target／actual RPS、PWM 是否飽和、時間戳間距與 encoder 差分；失聯測試需有可立即切斷馬達動力的條件。
4. 保存 log 後再畫圖。於程式專案根目錄執行以下離線命令，將輸入改為已保存的實驗 log：

```bash
python3 '#_tool_py/generate_wheel_status_charts.py' \
  z_function/wheel_logs/wheel_status_latest.txt \
  --keep-zero-edges --output-dir /tmp/twrb-wheel-charts
```

工具使用 `data[10]` 的 Arduino 毫秒建立時間軸；舊 10 欄紀錄會退回圖表模組的假設取樣時間，不能當作已量到的週期。預設會裁掉前後零目標區段，分析停車時應加 `--keep-zero-edges`。純零紀錄不代表馬達停車時間已測得。

`#_tool_py/test_PID/test_PID.py` 是會直接發布輪速的實驗工具，不經正式 NavigationRuntime／SafetySupervisor。操作前必須讀取其 CLI 參數、停止流程及確認輸出路徑，不能與 `main.py` 同時控制同一台車。

## 常見問題與注意事項

- **發 PWM 沒反應**：本頁輪速範例沒有 `/set_pwm` subscriber；模式 2 必須另外確認相容韌體。
- **輪速約差四倍**：查 encoder 邊緣計數與 CPR 是否來自同一版本。
- **左右方向相反**：核對接線、encoder sign 與 `setWheelPWM`，不要只在 Topic 上交換數字掩蓋問題。
- **高頻圖表更抖**：先查每次 counts、時間戳與狀態頻率；120 Hz 目錄明確是失敗案例，不能直接推薦燒錄。
- **沒有新 log**：先查 `/wheel_status_x1000` 是否跨 bridge 收到，再查 Node 建立的實際輸出路徑。歷史 SVG 或舊 `latest` 檔不是本次通訊通過的證明。

本次未連接實體硬體，接線、編譯燒錄、馬達方向、閉迴路效果、逾時停車與煞停距離皆尚未實機驗證。
