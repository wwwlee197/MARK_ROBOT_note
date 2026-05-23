# controller.py 詳解

## 目的

說明 `controller.py` 中 `Controller` class 的控制邏輯。這支程式的核心工作是根據機器人當前位置、朝向與目標位置，計算左右輪 PWM，並輸出到 ROS topic。

## 前置知識

- ROS Node 與 topic。
- `std_msgs.msg.Int16MultiArray`。
- PID 控制概念。
- 自走車左右輪差速控制。

## Controller 初始化

```python
from std_msgs.msg import Int16MultiArray
import numpy as np
import math

class Controller:
    def __init__(self, node, topic):
        self.node = node
        # 建立 publish_node
        self.pwm_pub = node.create_publisher(Int16MultiArray, topic, 5)
```

`Controller` class 接收兩個參數：

| 參數    | 說明                                |
| ------- | ----------------------------------- |
| `node`  | 建立 `Controller` 時依附的 ROS Node |
| `topic` | 對外輸出控制命令的 topic 名稱       |

## PID 參數

```python
# PID 演算法 Kp-比例增益 Ki-積分增益 Kd-微分增益
# 控制機器人進退的「距離」誤差
self.Kp_linear = 20.0
self.Ki_linear = 0.1
self.Kd_linear = 20.0

# 控制機器人旋轉的「角度」誤差
self.Kp_angular = 10.0
self.Ki_angular = 0.01
self.Kd_angular = 30.0
```

PID 可用下面方式理解：

| 項目   | 直覺說明                                           |
| ------ | -------------------------------------------------- |
| P 比例 | 根據目前誤差直接給力，距離越遠或角度越歪，輸出越大 |
| I 積分 | 如果一直推不動或一直偏掉，逐漸補更多力             |
| D 微分 | 根據誤差變化速度煞車，避免衝太快或轉太過           |

距離控制：

- `Kp_linear`：離目標遠就跑快。
- `Ki_linear`：推不動時逐漸補力。
- `Kd_linear`：靠近目標時降低衝過頭的機率。

角度控制：

- `Kp_angular`：角度偏很多就轉多。
- `Ki_angular`：長時間偏差時補轉。
- `Kd_angular`：轉太快時抑制過衝。

## PID 記憶體與限制

```python
self.prev_err_dis = 0.0
self.prev_err_theta = 0.0
self.integral_dis = 0.0
self.integral_theta = 0.0

self.max_pwm = 50
self.deadzone = 30
```

| 變數             | 用途                              |
| ---------------- | --------------------------------- |
| `prev_err_dis`   | 上一次距離誤差，用於計算距離 D 項 |
| `prev_err_theta` | 上一次角度誤差，用於計算角度 D 項 |
| `integral_dis`   | 距離誤差累積，用於距離 I 項       |
| `integral_theta` | 角度誤差累積，用於角度 I 項       |
| `max_pwm`        | PWM 輸出上限                      |
| `deadzone`       | 馬達最低有效輸出補償              |

主迴圈每次呼叫 `compute_pwm()` 時，大致流程如下：

1. 計算目前距離誤差 `err_dis`。
2. 用目前誤差與上一次誤差計算 D 項。
3. 將目前誤差累加到 integral，計算 I 項。
4. 算出 linear 與 angular 輸出。
5. 更新 `prev_err_dis` 與 `prev_err_theta`。
6. 轉換成左右輪 PWM。

## compute_pwm 輸入

```python
def compute_pwm(self, robot_pos, robot_orient, target_pos):
    # robot_pos     機器人當前位置
    # robot_orient  機器人當前朝向
    # target_pos    目標點位置
    if robot_pos is None or robot_orient is None or target_pos is None:
        return [0, 0]

    dx = target_pos[0] - robot_pos[0]
    dy = target_pos[1] - robot_pos[1]
    err_dis = math.sqrt(dx ** 2 + dy ** 2)
```

如果位置、角度或目標點資料缺失，直接回傳 `[0, 0]` 停車。資料完整時，先算出目標點與機器人之間的距離誤差。

## 情況 A：接近目標時只做角度對齊

```python
if self.target_orientation is not None:
    err_theta = (self.target_orientation - robot_orient + math.pi) % (2 * math.pi) - math.pi
    self.integral_theta += err_theta
    derivative_theta = err_theta - self.prev_err_theta
    self.prev_err_theta = err_theta

    angular_output = (self.Kp_angular * err_theta +
                      self.Ki_angular * self.integral_theta +
                      self.Kd_angular * derivative_theta)
    linear_output = 0.0

    if abs(err_theta) < 0.75:
        if not self.stop_sent:
            print("ID2 對齊完成，傳送一次停止PWM")
            self.stop_sent = True
            return [0, 0]
        else:
            return [0, 0]
else:
    return [0, 0]
```

這段用於接近目標後的對齊階段。此時不再前進，只根據 `target_orientation` 調整角度。

重點：

- `linear_output = 0.0`，代表停止前進。
- `err_theta` 會被正規化到 `-pi` 到 `pi` 的範圍。
- 當角度誤差小於門檻時，只送一次停止 PWM，避免重複送停止訊號。

## 情況 B：距離較遠時先轉向再前進

```python
self.stop_sent = False

target_angle = math.atan2(dx, dy)
err_theta = (target_angle - robot_orient + math.pi) % (2 * math.pi) - math.pi

self.integral_dis += err_dis
self.integral_theta += err_theta

derivative_dis = err_dis - self.prev_err_dis
derivative_theta = err_theta - self.prev_err_theta
self.prev_err_dis = err_dis
self.prev_err_theta = err_theta

linear_output = self.Kp_linear * err_dis + self.Ki_linear * self.integral_dis + self.Kd_linear * derivative_dis
angular_output = self.Kp_angular * err_theta + self.Ki_angular * self.integral_theta + self.Kd_angular * derivative_theta

if abs(err_theta) > 0.3:
    linear_output = 0.0
```

這段是導航模式。機器人距離目標還遠時，會先計算目標方向與目前朝向的角度差。

重點：

- `target_angle` 是機器人指向目標點的方向。
- `err_theta` 決定轉向輸出。
- `linear_output` 決定前進輸出。
- 如果角度誤差大於 `0.3`，先禁止前進，避免邊歪邊走。

## 左右輪 PWM 轉換

```python
left_pwm = linear_output - angular_output
right_pwm = linear_output + angular_output

left_pwm = int(max(min(left_pwm, self.max_pwm), -self.max_pwm))
right_pwm = int(max(min(right_pwm, self.max_pwm), -self.max_pwm))

if 0 < abs(left_pwm) < self.deadzone:
    left_pwm = int(math.copysign(self.deadzone, left_pwm))
if 0 < abs(right_pwm) < self.deadzone:
    right_pwm = int(math.copysign(self.deadzone, right_pwm))

return [left_pwm, right_pwm]
```

差速控制公式：

```text
左輪 = 前進 - 轉向
右輪 = 前進 + 轉向
```

最後會做兩件事：

- 限制 PWM 不超過 `max_pwm`。
- 若 PWM 有方向但小於馬達有效輸出，補到 `deadzone`，保留正負號。

## 驗證方式

測試時可觀察：

- 沒有位置或目標資料時，是否回傳 `[0, 0]`。
- 角度偏差大時，是否先原地轉向。
- 接近目標時，是否停止前進並只做角度對齊。
- 輸出的左右輪 PWM 是否落在 `-max_pwm` 到 `max_pwm`。

## 調參方向

- 車子反應太慢：可小幅增加 `Kp_linear` 或 `Kp_angular`。
- 車子容易衝過頭：可增加 `Kd_linear` 或降低 `Kp_linear`。
- 轉向震盪：可降低 `Kp_angular` 或增加 `Kd_angular`。
- 馬達不動但 PWM 有值：檢查 `deadzone` 是否太低。
