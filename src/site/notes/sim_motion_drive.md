---
{"dg-publish":true,"permalink":"/sim-motion-drive/","tags":["gardenEntry"],"dg-note-properties":{}}
---

### 一、硬件与被控对象
- 电机类型：PMSM / BLDC / 感应电机 见 [...\plant\Init_plant](D:\project\sim_motion_drive\plant\Init_plant.m)
![Pasted image 20261116095807.png](/img/user/images/Pasted%20image%2020261116095807.png)
- 控制目标：位置 / 速度 / 转矩
- 传感器：电流、编码器、母线电压
- 控制器：TI F28379D、PWM 频率、ADC 采样方式
- 功率级：三相逆变桥、死区、保护电路

### 二、控制架构
- 位置环 → 速度环 → 电流环
**`Pos_spd_control`模块**均由主cpu运算得到。
#### **位置环**
![Pasted image 20261118094833.png](/img/user/images/Pasted%20image%2020261118094833.png)
1. Position_measurement包含传动比转换。
2. Position_calculation:![Pasted image 20261116102123.png](/img/user/images/Pasted%20image%2020261116102123.png)
	其中SimulinkBlock_TrajGen_3ad找不到S-Function函数，系统运行时调用的是.mexw64文件，即编译后文件。
	ctrl_position_measurement给terminal了，所以目前是个开环。
	jerk是加速度微分。标橙的变量在modules里定义。
3. 变体源部分可选择debug模式，用阶跃作位置参考值，同时前馈的速度参考值为0
4. PI_arw_pos:
![Pasted image 20261116111759.png](/img/user/images/Pasted%20image%2020261116111759.png)
	输出速度修正量。
	Anti-Windup法抗积分饱和：
	![Pasted image 20261118095652.png](/img/user/images/Pasted%20image%2020261118095652.png)
	ctrl_kp_position=0.7，ctrl_ki_position=0
#### **速度环**
![Pasted image 20261123100114.png](/img/user/images/Pasted%20image%2020261123100114.png)
1. 速度环同样包含前馈控制+反馈控制，结构同位置环。`ctrl_kp_speed=0.22，ctrl_ki_speed=50.
2. 当前`ctrl_torque_loop_en=0`，执行速度环。`ctrl_iq_loop=0`，理论上是否这里也会有类似速度环结构的转矩调节器。
3. `exicitation control`中`ctrl_id_set`直接由`cla_speed_meas`赋值，单位Hz，这个值是由电机模型输出`eQEP_speed_meas`，机械角度转换为电角度，经过传动比和单位换算（k=1/2/pi,rad/s-->Hz）得来的，没有其他处理。（?）
4. 当前id_set=0
5. **加速度前馈** ==🟡TODO==
	![Pasted image 20261123134903.png](/img/user/images/Pasted%20image%2020261123134903.png)
	- `ctrl_iq_offset=k*ctrl_speed_set`
	- `ctrl_iq_pulse`首个传递条件`ctrl_speed_target~=0`. 当前 `ctrl_FFC_gain_a=0`，加速度前馈关闭。分 acceleration 和 deceleration，`ctrl_acceleration_set=10, ctrl_deceleration_set=20`（负载特性不对称?）。
	- `ctrl_iq_offset`和`ctrl_iq_pulse`在下一个模块将被命名为`cla_iq_offset`和`cla_iq_pulse`


`Tq_control_CLA`模块在CLA。需要把这个子系统里的代码编译并分配到 CLA 的专属内存（LSRAM）中去执行，在仿真中并没有作区分。[控制律加速器 (CLA)](https://dev.ti.com/tirex/explore/node?isTheia=false&node=A__Ab9r7Ln8zLUQPv5MUnzbmg__C28X-ACADEMY-CN__hMpsP7d__LATEST)
**电流运算**
![Pasted image 20261123141137.png](/img/user/images/Pasted%20image%2020261123141137.png)
- 内环周期、外环周期分别是多少
- Id、Iq 分别怎么给定
1. 
- 是否有弱磁、MTPA、前馈
- 是否使用无感观测器，还是编码器反馈

### 三、FOC 算法链路

- Clarke 变换：abc → αβ
- Park 变换：αβ → dq
- 电流环 PI：Id、Iq 控制
- 反 Park：dq → αβ
- SVPWM：生成三相占空比
- 转子角度来源：编码器 / 观测器

### 四、Simulink 工程结构
- 顶层模型：`Sim_environment.slx`
![Pasted image 20261113141137.png](/img/user/images/Pasted%20image%2020261113141137.png)

Sim_environment.slx![Pasted image 20261113133704.png](/img/user/images/Pasted%20image%2020261113133704.png)
这里Pos_spd_control和Tq_control_CLA均为模型引用，在modules里，和Motion_Drive_Library相关是库链接

- 子模型：`Pos_spd_control`、`Tq_control` 各负责什么
- CLA 模块：哪些算法跑在 CLA 上


### 五、参数与初始化
- 电机参数：Rs、Ld、Lq、极对数、磁链
- 控制参数：PI 系数、采样周期、PWM 频率
- `USER_MOTOR` 定义在哪里
- `Init_mc_cla_motor_ident` 初始化了哪些变量
- `Init_all` 的调用顺序

### 六、嵌入式映射
- ADC 对应哪些通道
- ePWM 如何输出
- 编码器接 eQEP 还是其他接口
- CLA 承担哪些任务
- CPU 主循环负责哪些任务
- 中断触发源是什么

### 七、运行流程
- 上电后先执行哪些初始化
- 进入控制循环前有哪些条件
- 每个控制周期依次做什么
- 外环多久更新一次
- 故障时如何进入保护状态

### 八、保护与限制
- 过流、过压、欠压、过温
- 电压限幅：`mc_cla_vol_lim`
- PWM 占空比限幅
- 电流限幅
- 堵转、失步、参数异常处理

### 九、调试验证
- 开环能否正常发波
- 电流环能否跟踪 Id/Iq
- 速度环能否稳定调速
- 位置环定位精度如何
- 观测器角度是否准确
- 参数辨识是否通过

