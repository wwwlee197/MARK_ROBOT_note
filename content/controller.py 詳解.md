
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
Controller class 類別接收兩個個參數
1. node: 實體化該類別 Controller class 時候所依附的 ROS Node 
2. topic: Controller 對外輸出控制命令 topic 名稱 


```python
        # PID 演算法 Kp-比例增益    Ki-積分增益    Kd-微分增益
        # 控制機器人進退的「距離」誤差
        self.Kp_linear = 20.0
        self.Ki_linear = 0.1
        self.Kd_linear = 20.0

        # 控制機器人旋轉的「角度」誤差
        self.Kp_angular = 10.0
        self.Ki_angular = 0.01
        self.Kd_angular = 30.0
```
#### PID 演算法
P(比例): 根據距離決定推力大小
I(積分): 若當前的推力太小卡住，逐漸加大推力
D(微分): 推力太大則踩煞車減速

##### 距離控制（前進後退）
- P：離目標遠就跑快(Kp_linear)
- I：推不動就補力(Ki_linear)
- D：快撞到了就減速(Kd_linear)
##### 角度控制（轉向）
- P：角度歪很多就轉多(Kp_angular)
- I：一直歪就補轉(Ki_angular)
- D：轉太快就煞住(Kd_angular)
```python
        self.prev_err_dis = 0.0     # 上一次的距離誤差
        self.prev_err_theta = 0.0   # 上一次的角度誤差
        self.integral_dis = 0.0     # 距離誤差的積分值
        self.integral_theta = 0.0   # 角度誤差的積分值

        self.max_pwm = 50       # 馬達最 大 值 
        self.deadzone = 30      # 馬達最 小 值
```
### PID 的記憶體（memory）
- prev_err_xxx : 計算上一秒的離目標的誤差
用於 D(微分) 的計算 `現在誤差 - 上一次誤差`
- integral_xxx : 你到底卡在這裡多久了(無法前進 or 無法轉向)
用於 I(微分) 的累積 `integral_dis = 0.15 + 0.15 + 0.15 + ...
`
設定馬達轉速上下限: I 隨著摩擦力or配重逐漸加大轉速的範圍

### 主迴圈每一次呼叫 `compute_pwm()`：
1. 算現在誤差 `err_dis`
2. 用 `prev_err_dis` 算變化量（D）
3. 把 `err_dis` 加進 `integral_dis`（I）
4. 算 PWM
5. 把現在誤差存成 `prev_err_dis`


```python
	    self.marker_length = 0.09
        self.aligning = False
        self.target_orientation = None
        self.stop_sent = False  # ✅ 加入對齊停止旗標
```

---
## compute_pwm

```python
    # 輸出控制左右輪馬達pwm
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
- 防呆：沒有資料就停
- 算「距離誤差」err_dis

```python
# 接近目標時候 專注於角度調整
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
			print("✅ ID2 對齊完成，傳送一次停止PWM")
			self.stop_sent = True
			return [0, 0]
		else:
			return [0, 0]
else:
	return [0, 0]	
```
情況 A：近距離（err_dis < 0.1）【只對齊、不前進】


---


```python
# 距離較遠時 先轉想 後前進

self.stop_sent = False  # ✅ 若有移動，重新允許送停止PWM

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
情況 B：遠距離（err_dis ≥ 0.1）【導航模式】

- target_angle: 自走車指向目標點的向量
- err_theta: 自走車的的朝向 與 自走車指向目標點的向量 的 {**誤差角度θ**}
→ 決定自走車的 轉向 推力

- integral_xxx += err_xxx  
→ 每跑一次 control loop，計算累積的誤差值

- derivative_xxx = err_xxx - self.prev_err_xxx
→ 跟上一瞬間相比，前進速度 & 轉速

- self.prev_err_xxx = err_xxx
→ 存起來當前的誤差給下一次 control loop 調用

linear_output  = (...)
angular_output = (...)    右轉為正，左轉為負
→ 決定前進推力 與 轉向推力

- if abs(err_theta) > 0.3: linear_output = 0.0
→ 先轉正對齊目標，不准邊歪邊走


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
>**左輪 = 前進 − 轉向**  
>**右輪 = 前進 + 轉向**

- if 0 < abs(right_pwm) < self.deadzone:
→ 馬達轉速補償: 馬達轉速未達到最低值，則直接補上
方向不變，只補力道(正負號保留)