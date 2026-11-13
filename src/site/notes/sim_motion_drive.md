---
{"dg-publish":true,"permalink":"/sim-motion-drive/","tags":["gardenEntry"],"dg-note-properties":{}}
---





personal token: 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
[Digital Garden](https://simmotiondrive.vercel.app/)



### 一、硬件与被控对象
- 电机类型：PMSM / BLDC / 感应电机
- [Init_plant](D:\project\sim_motion_drive\plant\Init_plant.m)

- 控制目标：位置 / 速度 / 转矩
- 传感器：电流、编码器、母线电压
- 控制器：TI F28379D、PWM 频率、ADC 采样方式
- 功率级：三相逆变桥、死区、保护电路

### 二、控制架构
- 位置环 → 速度环 → 电流环

- 内环周期、外环周期分别是多少
- Id、Iq 分别怎么给定
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
- 库依赖：`Motion Drive Library` 提供哪些基础模块
- 模型引用关系：谁引用谁，谁是 Top Model

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

