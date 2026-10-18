# PC 端控制架構

## 目的與前置條件

說明 `main.py` 正式模式 1～8 如何從影像定位、規劃路徑到輸出馬達命令，並提供除錯入口。程式以 `twrb_project` commit `98dba0d6` 為準；完整版本與來源差異見[查核紀錄](../99_附錄/程式版本更新查核紀錄.md)。本文中的參數是軟體設定，尚未代表實車量測值。

先完成[環境配置](../02_環境配置/bashrc%20設定.md)與[執行流程](../03_執行流程/1_Examples%20執行流程.md)，並確認所選模式的命令單位與車上韌體相容。

## 系統資料流程與模組職責

```text
main.py::create_node
  → ModeRegistry.profile_for → ProfileRepository.load
  → twrb_ros.node_factory.create_node_from_profile
  → Profile 指定的正式 ROS Node／factory

相機 → 去畸變／ArUco → Homography → RobotPose（m、rad）
     → waypoint／障礙輸入 → Planner → 路徑／追蹤目標
     → Controller → 有單位的命令
     → NavigationRuntime 或 SafeCommandService
     → SafetySupervisor → Command Publisher → ROS Topic

Arduino wheel status → WheelStatusAdapter → 文字紀錄／SVG
```

| 層級                    | 職責與主要入口                                                                                     |
| ----------------------- | -------------------------------------------------------------------------------------------------- |
| `src/twrb_core/`        | `RobotPose`、`PathPoint`、`WheelCommand`、`PwmCommand`，純控制數學與安全裁決；不建立 ROS publisher |
| `src/twrb_application/` | `ModeRegistry`、`ProfileRepository`、導航狀態與命令協調；不開相機、不建立 ROS Node                 |
| `src/twrb_ros/`         | factory、Node lifecycle、相機／UI／YOLO／RViz adapters，以及 ROS 訊息轉換                          |
| `z_function/`           | Homography、五次多項式、A\*、曲率安全檢查、雙鏡頭幾何與紀錄工具                                    |

`ModeRegistry` 只將數字映射到 Profile 名稱；真正的動態匯入在 ROS factory。Profile 檢查名稱、Topic、單位等欄位，factory 再核對 planner／controller／RViz 組合。設定錯誤會明確失敗，不會自動改用另一模式。

**不是每個模式都建立 `NavigationRuntime`。** 模式 1／8 使用它負責規劃與 tracking；模式 2 使用 `SafePwmCommandService`，模式 3／4／5／6／7 使用 `SafeWheelCommandService`。這些分支都經 `SafetySupervisor`，再由唯一對應單位的 Gateway 發送。`tmp_node_core/`、舊 `controller.py`、`mode1_standalone.py` 與模擬程式不在此正式入口路徑。

## 執行模式與控制策略比較

以下 Profile 名稱皆位於 `config/profiles/`，加上 `.yaml` 即為原始設定檔。

| 模式 | Profile                       | 規劃／用途                             | 控制與命令                                        |
| ---- | ----------------------------- | -------------------------------------- | ------------------------------------------------- |
| 1    | `single_quintic`              | 單鏡頭，多 waypoint 五次多項式         | 一般 Pure Pursuit，RPS × 1000                     |
| 2    | `single_quintic_rviz`         | 單鏡頭五次多項式，加 RViz／TF          | 距離與角度 PID，PWM                               |
| 3    | `dual_quintic`                | 雙鏡頭五次多項式，從目前車位起步       | 一般 Pure Pursuit，RPS × 1000                     |
| 4    | `single_curvature_astar`      | 手動障礙，曲率安全 A\*                 | 一般 Pure Pursuit，RPS × 1000                     |
| 5    | `single_yolo_astar`           | YOLO／背景／手動障礙佔用網格及安全 A\* | 一般 Pure Pursuit，RPS × 1000；感知與路徑侵入檢查 |
| 6    | `dual_edge_safe`              | 雙鏡頭五次曲線完整可見性、相機交接停等 | 一般 Pure Pursuit，RPS × 1000；模式 6 policy      |
| 7    | `single_curvature_astar_ffis` | 重用模式 4 Node 與規劃                 | FFIS + Pure Pursuit，RPS × 1000                   |
| 8    | `single_quintic_ffis`         | 重用模式 1 Node 與規劃                 | FFIS + Pure Pursuit，RPS × 1000                   |

模式 2 的 `/set_pwm` 不能直接替換成輪速 Topic。其目前 `PwmController` 仍是 PID，但已拆成純運算，透過安全服務輸出，詳見 [controller.py 詳解](controller.py%20詳解.md)。其餘模式的命令是 `/cmd_wheel_rps_x1000`。

### 相機定位與 Homography

單鏡頭 Profile 使用 Camera 4 的 `npz_data/b1_calibration_1280_720.npz`；雙鏡頭 3／6 使用 Camera 4／6，並以 `camera_configs` 分別指定 b1／b2 校正檔。相機 index 不是跨電腦固定的硬體識別碼，啟動前應核對。

相機內參負責去畸變，Homography 則將地板平面像素投影為公尺座標：

$$
s\begin{bmatrix}x\\y\\1\end{bmatrix}
=H\begin{bmatrix}u\\v\\1\end{bmatrix}
$$

程式在 `z_function/qp_h_common.py::pixel_to_world_coords` 做齊次座標除法；`estimate_robot_pose_from_h` 由車上 ArUco ID 0 的角點估計位置與 yaw，再加入模式指定的 `marker_forward_offset_m`。世界 yaw 採 `+X=0`、`+Y=π/2`、逆時針為正。

地面 ArUco ID 1～6 的實際位置必須對應各 Node 的 `WORLD_COORDS_M`，不能只排出六張標籤就當作完成校正。模式 1／8 offset 為 `-0.03 m`，模式 4／7 為 `-0.00 m`，模式 2 為 `0.14 m`，不能跨模式套用。

**模式 2 有待確認的解析度差異：** Profile 指向 1280×720 校正檔，`SingleQuinticRvizNavigationNode` 卻要求 1024×576 擷取。需確認攝影機實際輸出與內參是否匹配；本次沒有改動程式或宣稱定位通過。

### 五次多項式與安全 A\*

`z_function/qp_planning.py::plan_continuous_path` 以五次多項式描述 x/y，配合端點位置、速度與加速度條件，搜尋滿足加速度、jerk 約束的路徑時間。`QuinticPlannerAdapter` 將結果轉成 Core 的 `PathPoint`；模式 1／8 在按 Enter 時把目前車位接到使用者 waypoint 前方。

模式 4／7 經 `CurvatureSafePlannerAdapter` 呼叫 `plan_curvature_safe_path`：A\* → 安全捷徑化／簡化 → 五次式平滑 → 邊界、碰撞、曲率檢查 → 必要時回插 waypoint 或圓弧 Fillet。只有驗證通過才能追蹤；模擬中的分段 stop-turn-go 備援未由正式 adapter 啟用。模式 5 加入 `occupancy_grid.py` 的同一份世界佔用網格作為規劃與剩餘路徑檢查依據。

**規劃速度與行車速度要分開看。** 共用 `QUINTIC_PLANNING_CRUISE_SPEED_MPS=0.17` 用於路徑生成；模式 1／8 的 adapter 只傳出 x/y，不附時間速度。Runtime 因此採 Profile 的 `path_cruise_speed_mps=0.37` 作追蹤請求上限，不能把規劃的加速度／jerk 約束當成實際馬達已遵守的限制。模式 3／6 另外從路徑速度視窗產生追蹤速度請求。

## Pure Pursuit 與輪速映射

`PurePursuitController.compute` 將追蹤目標轉到車體座標。設目標在車體座標為 `(x_b,y_b)`、實際目標距離為 `L_d`：

$$
\kappa=\frac{2y_b}{L_d^2},\qquad \omega=v\kappa
$$

`NavigationRuntime._select_target` 先找局部最近路徑點，再向前找達到 lookahead 距離的點；公式中的 `L_d` 是車到選中點的實際距離，未必恰好等於設定值。

沒有注入 FFIS 時，基本速度為 `max_linear_mps / (1 + curvature_speed_gain × |κ|)`，再做上下限與請求速度限制。目標落在車後方且角度過大時，控制器可進入原地轉向，並使用不同進入／退出門檻避免反覆切換。

差動輪運動學使用車體線速度 `v`（m/s）、角速度 `ω`（rad/s）、輪距 `b`（m）、輪半徑 `r`（m）：

$$
RPS_L=\frac{v-\omega b/2}{2\pi r},\qquad
RPS_R=\frac{v+\omega b/2}{2\pi r}
$$

RPS 是每秒圈數，輪軸角速度則為 `2π × RPS` rad/s。兩者不可直接互換。Core 預設 `r=0.035 m`、`b=0.16 m`，這是程式假設，實際車體尺寸仍需量測。

| 限制       | 目前軟體行為                                                                                                   |
| ---------- | -------------------------------------------------------------------------------------------------------------- |
| 輪速上限   | `PurePursuitController.max_rps=2.0`，左右輪各自截限                                                            |
| 輪速 slew  | 一般預設每次 compute 最多改變 `0.20 RPS`；模式 3／6 明確改為 `0.10 RPS`，不是每秒變化量                        |
| 一般線速度 | 模式 1／4 的 Core 上限 0.40 m/s，追蹤請求上限 0.37；模式 3 Profile 0.32；模式 5 Core 0.40；模式 6 Profile 0.40 |
| 排程       | 單鏡頭 Node timer 為 0.033 s，雙鏡頭為 1/30 s；不保證實際每幀準時，也不是 Arduino 控制週期                     |
| 封包       | `WheelCommandPublisher.publish` 做 `int(RPS × 1000)`，順序左、右；超出 Int16 範圍送零                          |

輪速截限與 slew 後的命令可能不再精確等於原先的 v／ω；紀錄中的控制器速度不等於實測車速。

## 模式 7／8 的 FFIS 速度控制

兩個小型 factory 注入 `FractionalFuzzySpeedController`，規劃器與安全出口仍沿用各自 Node。`evaluate(κ, heading_error)` 使用曲率與目標方位誤差的絕對值，計算 small／medium／large 隸屬度，再以 `μ^alpha` 調整規則強度。現行 Sugeno singleton 的 recovery／slow／medium／fast 代表值為 0.08／0.14／0.22／0.37 m/s，經規則加權後再限制速度。

目前兩份 Profile 的 `alpha=0.8`、速度區間 0.08～0.37 m/s。`ffis_output_samples=151` 只是 API 相容設定，現行 `evaluate` 不做 Mamdani 輸出積分。

Recovery 在方向誤差超過 30° 時進入，低於 18° 並維持 0.25 s 才退出。API 也支援橫向誤差，但正式 `PurePursuitController.compute` 目前只傳曲率與角度，不能宣稱已把追蹤 CSV 的橫向誤差送入 FFIS。

模式 7／8 另啟用：

- 速度自適應 lookahead：`clip(0.075 + 0.12 × 前次速度, 0.085, 0.12)`，距離單位 m。
- 線速度上升／下降限制：0.20／0.40 m/s²，依控制器設定的 0.033 s 計算每次變化量，不是量測實際 dt。
- 後續仍做差動輪速換算、輪速截限與 slew。安全停止直接送零，不必經慢速斜坡降至零。

## 安全停止與模式差異

`SafetySupervisor.enforce` 在收到 fault 時，以同單位零命令覆蓋控制器輸出。它依 Node／Runtime 提供的健康狀態裁決，並不自行監控網路或 Arduino 回授；`WheelStatusAdapter` 目前只記錄回授，沒有自動以輪速追蹤誤差急停的閉迴路。

模式 1／8 的 ArUco 短暫丟失會保留前次 pose 最多 0.15 s，前 0.10 s 將輪速命令乘 0.50，之後乘 0.40；超時才送零。因此不能把「一幀失標」寫成所有模式都立即停車。模式 4／7 有相同 hold 參數；模式 5 還必須通過感知配對檢查，模式 6 則不允許只靠舊 EMA pose 放行。

模式 5 將 live pose 與非同步 YOLO snapshot 分開。frame、Homography revision、background revision、ROI、來源尺寸與 occupancy 必須配對且新鮮，剩餘路徑遭侵入也會中止導航。障礙移除後需再按 Enter 規劃，不會自動續行。

模式 6 使用 `DualCameraPolicySet.mode6()`：四周 50 px 排除、SUBPIX、世界安全區內縮 0.11 m，以 0.02 s 取樣驗證完整曲線落在兩相機多邊形的真實聯集內。交接要求 5 個有效 frame、0.5 s 切換間隔與 0.05 m hold margin；目前 Profile 速度 0.40 m/s 使最小連續 overlap 為 **0.25 m**。每次非零命令還要有本幀原始 Marker 中心被來源相機自己的安全區涵蓋。模式 3 不套用這些額外 gate。

ESC、Ctrl+C、導航完成、初始化或執行例外都應經對應停止出口。`main.py` 自行接管 Ctrl+C，在 ROS context 仍有效時依序 `clean_up()` → `destroy_node()` → `rclpy.shutdown()`。Runtime／SafeCommandService 先嘗試發布零，再 reset controller；各 Node 分別清理相機、視窗與紀錄資源。重複 cleanup 可能略過已成功停止的步驟，不能假設會持續重送零。

軟體成功 publish 不等於 Arduino 已收到；網路斷線仍依賴已確認的 MCU timeout 與實體急停，見[Arduino 輪速閉迴路控制](Arduino%20輪速閉迴路控制.md)。

## 實際操作、驗證與排錯

1. 依[執行流程](../03_執行流程/1_Examples%20執行流程.md)啟動各端，確認 Topic 型別及韌體，再從專案根目錄執行 `python3 main.py`。
2. 選模式、確認相機／校正／單位，鎖定 Homography 後再輸入路徑；先以車輪離地方式完成停止與方向檢查。
3. 用[通訊協定頁](ROS%20Topic%20與通訊協定.md)的指令觀察「最終命令」與「Arduino 回授」，不要以畫面有路徑當作馬達控制正常。
4. 模式 1／4／7／8 的追蹤 CSV 在工作目錄 `wheel_logs/modeN_tracking_latest.csv`；參考路徑為 `modeN_reference_path_latest.csv`，以 `mission_id` 對應。相同模式重開會覆寫，請先備份。
5. 實測輪速文字檔在 `z_function/wheel_logs/wheel_status_latest.txt`，不要與上項 CSV 混用。CSV 控制器欄位尚未套用 hold 降速或 safety 裁決，不是 encoder 實測值。

| 現象                   | 優先排查                                                           |
| ---------------------- | ------------------------------------------------------------------ |
| ProfileError           | 模式 alias、必要欄位、factory 匯入、Topic／單位／controller 組合   |
| 相機開啟失敗或座標歪斜 | Camera index、實際解析度、內參、地面標籤位置、各模式 offset        |
| 有路徑但沒有非零命令   | Node 導航狀態、目前 pose、規劃失敗理由、模式 5 感知或模式 6 gate   |
| 命令有值但車不動       | 訂閱者、bridge、rosserial、韌體單位、電源；不要直接提高 PID 或 PWM |
| 回授追不上命令         | 先檢查左右順序、encoder 符號／CPR、飽和與供電，再做受控實車調校    |

可核對 `tests/test_main_modes.py`、`tests/test_twrb_profiles.py`、`tests/test_twrb_architecture_boundaries.py` 的軟體契約；ROS、相機、導航與煞停行為仍須逐模式實機驗收。
