>下載 arduino IDE，即可開始此教學

### 1. 程式碼下載: [🔗](https://drive.google.com/drive/folders/1THvN7rc72iKJG5PodxgnEvfE37KvztTp?usp=drive_link) **MoveControlDemoROS_v3.ino**


### 2. 套件準備: ros_lib
- 方法 1. PC 安裝好 ubuntu 後，使用以下指令產生
	```
	rosrun rosserial_arduino make_libraries.py
	```

- 方法 2. 從我這裡下載 [🔗](https://drive.google.com/drive/folders/1THvN7rc72iKJG5PodxgnEvfE37KvztTp?usp=drive_link)，但自己產生的比較有成就感吧

取得 ros_lib 後，將整個資料夾放到以下路徑(參考)
`C:\Users\user\OneDrive\Documents\Arduino\libraries`


### 3. 燒錄程式碼
![](assets/Arduino%20MEGA%20程式碼燒錄/file-20260114171016387.png)

---

```
ros::Subscriber<std_msgs::Int16MultiArray> pwm_sub("set_pwm", pwmCallback);
```

"set_pwm" 是訂閱的 topic ，根據專案靈活調整