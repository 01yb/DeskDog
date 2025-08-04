<img width="3840" height="2160" alt="image" src="https://github.com/user-attachments/assets/2de4cf12-2714-4b85-9bf5-967f7c12884b" />

# STM32 桌面智能萌宠小狗（桌面桌宠项目笔记）

---

## 一、项目概述

| 项目 | 属性 |
|-----|------|
| 名称 | STM32 智能桌面宠物小狗 |
| 控制器 | STM32F103C8T6（Mini “蓝板”） |
| 控制方式 | **语音控制（SU‑03T1 模块）** + **蓝牙遥控（HC‑05/HC‑06 或 BLE）**共存:contentReference[oaicite:1]{index=1} |
| 功能动作 | 共 16 种 (立正、前进、后退、左 右 转、摇摆、摇尾巴、打招呼、伸懒腰、跳跃等):contentReference[oaicite:2]{index=2} |
| 表情反馈 | OLED 128×64 显示图像切换（睡觉／快乐／狂热／打招呼等）:contentReference[oaicite:3]{index=3} |
| 驱动舵机 | 共 5 伺服舵机（四肢 + 尾巴），采用 TIM2×4 通道 + TIM3 尾巴通道:contentReference[oaicite:4]{index=4} |
| 供电设计 | 3.7 V 锂电池 → TP4056 充放电模块 → 5 V → 板上稳压芯片 3.3 V → STM32／舵机／OLED／模块:contentReference[oaicite:5]{index=5} |

---

## 二、核心功能与操作方式

1. **交互控制**  
   - 串口中断接收命令，自动切换 `Action_Mode` 与 `Face_Mode`。  
   - 支持“按住蓝牙命令则持续动作”（Bluetooth 控制持续方式）。:contentReference[oaicite:7]{index=7}

2. **动作执行逻辑**  
   - 中断处理后进入主循环，根据 `Action_Mode` 通过 Servo 控制舵机动作（组合为连续步态动画）。  
   - 每个动作函数设置完善的角度组合、延迟控制与动作次数变量（防止电流冲击中断语音模块）。:contentReference[oaicite:8]{index=8}

3. **视觉反馈**  
   - OLED 用于显示不同表情图，调用 `Face_Config()` 绘制当前 `Face_Mode` 图像（预定义 Face_sleep、Face_happy 等资源）。:contentReference[oaicite:9]{index=9}

4. **LED 呼吸灯**  
   - TIM3 中断实现 LED 呼吸效果：PWM 周期 20 ms、亮度循环渐变，支持开/闭机制。:contentReference[oaicite:10]{index=10}

---

## 三、硬件 BOM（物料清单）

| 类型 | 型号 |
|------|------|
| MCU 板 | STM32F103C8T6 “蓝板” |
| 舵机（5 只） | SG90/SG92R |
| OLED 模块 | OLED 0.96″ 或 1.3″（SSD1306 驱动） |
| 蓝牙模块 | HC‑05/06 或 BLE 串口透传 |
| 语音模块 | SU‑03T1（机芯智能，可自设唤醒词） |
| 电源 | 锂电池（3.7 V）×1／TP4056 充放电模块 |
| 扩展图 | MCU 核心板库、3D 模型、PCB 文件打开链接可见:contentReference[oaicite:11]{index=11} |

---

## 四、软件架构示意图

```

main.c ─── Timer（PWM）初始化
├─ Servo 驱动舵机模块
├─ OLED/Face\_Config 图形显示模块
├─ BlueTooth / USART1 中断 (su‑03) ／USART3 中断 (BLE)
└─ PetAction 动作函数内置多种 gait 逻辑

delay.c ── 软件延时函数
PWM.c    ── 初始化 TIM2/TIM3，封装 PWM\_SetCompare 功能
Servo.c  ── 输入角度统一映射，驱动舵机运动
Face\_Config.c ── 根据 Face\_Mode 绘制 OLED 表情图
PetAction.c ── 各个动作函数：前进／转弯／摇尾／打招呼等
BlueTooth.c ── USART 接收字符改变 Action/Face 模式、控制 LED 呼吸灯、速度调节
OLED.c ── OLED 驱动实现 I²C／软件模拟，显示图形接口
Delay.c ── 精确延迟与滴答控制支持

```

所有模块均使用标准库（STD_PERI）的 HAL 封装，每个功能函数独立，符合嵌入式工程层次结构。

---

## 五、使用说明与编译方式

1. 使用 STM32CubeMX（或 CubeIDE）生成 HAL 工程核心配置（GPIO、TIM、USART）。  
2. 将 `lib／Drivers` 模块放入对应源文件（servo/Face_Config/OLED/BlueTooth 等）。  
3. 烧录 `main.bin` 后接入舵机、蓝牙、OLED、语音模块，插电后点击控制指令实验即可（录制动作自动 ○）。  
4. Android/iOS/PC 端测试时，可用串口助手发送指令（比如：  
   `0x33` = 前进、`0x43` = 打招呼、`0x40` = 摇尾巴等:contentReference[oaicite:13]{index=13}

---

## 六、项目亮点与扩展建议

- **跨模态交互初尝试**：动作（舵机）＋ 表情（OLED）＋ 声音（语音／语音反馈） = 小型交互人机界面系统。  
- **PWM 控制与姿态编码思路启发**：每个动作对应编码组合，具备“动作语义映射”的潜质，可尝试引入触觉振动马达模拟“触感反馈”。  
- **适合进一步融合嵌入式姿态/触觉交互研究**：可在头部嵌入 MPU6050，模拟“姿态→表征动作”；或在尾部舵机联动 Linear Resonance Actuator 模拟触觉反馈编码。  
- **可扩展模块**：添加触觉电机输出、摄像头识别（姿态/人脸）、语音合成（TTS）、触觉扩展（如 LRA）, 与导师方向形成研究级原型。  

---

许可证：**CC BY‑SA‑4.0，在保留署名与授权协议的前提下，可自由复制、修改与发布**。

---
