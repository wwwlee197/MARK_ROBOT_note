# controller.py 詳解

## 目的與版本定位

本頁保留舊版距離／角度 PID 直接輸出 PWM 的概念，並對照仍使用 PWM 的正式模式 2。**舊 `tmp_node_core/controller.py::Controller` 已是 rollback 參考，並非 `main.py` 模式 1～8 的正式控制器入口。** 現行整體資料流請讀 [PC 端控制架構](PC%20端控制架構.md)。

本次來源為 `twrb_project` commit `98dba0d6`，依 `MODE_ALIASES`、8 份 Profile 與 `node_factory` 查核。舊頁列出的 `Kp_linear=20`、`Kp_angular=10`、`max_pwm=50` 也不符合目前 rollback 檔，不能繼續當作現行調參值。

## 舊版控制概念

舊 `Controller` 自行建立 ROS publisher，以目前位置與目標位置算距離誤差，再以方向誤差算旋轉量：

```text
距離 PID → linear_output
角度 PID → angular_output
左輪 PWM = linear_output - angular_output
右輪 PWM = linear_output + angular_output
最後做 PWM 上限與 deadzone 補償
```

P 項對當前誤差反應，I 項累積誤差，D 項反應誤差變化；距離很近時可只做最終角度對齊。這不是 encoder 輪速閉迴路，也不是 Pure Pursuit 的曲率追蹤。舊影像座標與 yaw 慣例不可直接套用到現在的 Homography 世界座標。

## 正式模式 2 目前如何實作

前置條件是已確認底盤支援 PWM、相機校正有效且完成 ROS 部署。`config/profiles/single_quintic_rviz.yaml` 指向 `SingleQuinticRvizNavigationNode`，使用 `src/twrb_core/control/pwm.py::PwmController`：

```text
Homography 世界位置／yaw + 追蹤目標 PathPoint
  → PwmController.compute
  → PwmCommand（左／右整數 PWM）
  → SafePwmCommandService → SafetySupervisor
  → PwmCommandPublisher → /set_pwm
```

| 參數                        | 目前 `PwmController` 預設值   |
| --------------------------- | ----------------------------- |
| 距離 `(Kp, Ki, Kd)`         | `(420.0, 0.0, 20.0)`          |
| 角度 `(Kp, Ki, Kd)`         | `(80.0, 0.0, 45.0)`           |
| `max_pwm / deadzone`        | `120 / 40`                    |
| `turn_threshold_rad`        | `1.0`，超過時停止前進、先轉向 |
| `final_distance_m`          | `0.1`                         |
| `final_angle_tolerance_rad` | `0.75`                        |

模式 2 Node 直接建立 `PwmController()`，所以採用以上值。`compute` 使用 `atan2(dy, dx)` 與 ROS yaw 慣例；積分是每次呼叫累加誤差，微分是相鄰誤差相減，**沒有乘除實際 dt**。因此這些離散增益不能直接複製成 Arduino 輪速 PI 的參數。目前 `Ki=0`，並未啟用積分項貢獻。

控制器負責 PWM 數學，停止與發布由安全服務處理；不再讓控制器自行發 Topic。RViz 的 `/robot_pose` 與 TF 是另一個 adapter。模式 2 仍有「1280×720 校正檔對上 1024×576 擷取設定」的待確認問題，詳見[查核紀錄](../99_附錄/程式版本更新查核紀錄.md)。

## 操作與驗證

1. 依[執行流程](../03_執行流程/1_Examples%20執行流程.md)選擇模式 2，確認使用 PWM 相容韌體；不要使用本頁操作 RPS-only 韌體。
2. 監看 `/set_pwm`，確認左右整數命令及 ±120 上限；不是 RPS × 1000。
3. 確認定位缺失、退出或例外時走零命令出口，並在現場驗證車輛確實停止。
4. 若要了解其他正式模式，閱讀 [Pure Pursuit 與安全控制](PC%20端控制架構.md)，不要在 rollback controller 調參後期待正式模式行為改變。

## 常見問題與注意事項

- PWM 數值不能直接解讀為車速；有 PWM 但不動時先查接線、電源、Topic 與韌體，不直接提高上限或 deadzone。
- PC 距離／角度 PID 與 [Arduino 輪速 PI](Arduino%20輪速閉迴路控制.md) 控制的是不同誤差，參數與更新週期不可混用。
- 本次只更新說明，沒有調整任何控制參數，亦未完成模式 2 實機驗收。
