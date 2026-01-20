
# twrb 連線說明

### -------- PC 端 --------

#### 1. ssh 連線進樹梅派,選ros1
```
ssh pi@192.168.0.105
```

```
ssh pi@192.168.0.106
```

#### 2. 啟動 ros1 主要節點(mode_1)
```
roscore
```

#### 3. ros_bridge(mode_3)
```
ros2 run ros1_bridge dynamic_bridge --bridge-all-2to1-topics --bridge-all-1to2-topics --qos-reliability best_effort --qos-durability volatile
```


---


### -------- twrb 端 --------
#### (ros1) No.0
```
rosrun rosserial_python serial_node.py _port:=/dev/ttyUSB1 _baud:=115200 __name:=serial_node_00

```

#### (ros1) No.1
``` 
rosrun rosserial_python serial_node.py _port:=/dev/ttyUSB1 _baud:=115200 __name:=serial_node_01

```

