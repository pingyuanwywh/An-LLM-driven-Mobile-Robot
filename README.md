# Autonomous Mobile Manipulation for Ground-Level Grasping

> An LLM-driven mobile manipulator that takes a natural-language instruction, navigates to a target, and grasps it — closing the gap between meter-level position priors and centimeter-level grasping.

**[中文版见下方 / Chinese version below](#中文说明)**

<p align="center">
  <!-- Replace with your 15-second demo GIF -->
  <img src="docs/demo.gif" width="600" alt="End-to-end demo">
</p>

📺 **Full demo:** [[YouTube link] · [Bilibili link]](https://b23.tv/y1Y0TMn)

---

## What it does

A single natural-language command drives a mobile robot to autonomously navigate an unmapped-on-the-fly environment, locate colored cans, grasp them, and deposit them at a collection point.

**End-to-end result:** 5 cans collected in a single continuous run · ~6 minutes · fully autonomous.

---

## Why it's hard

The target scenario is air–ground coordination: an aerial platform provides a coarse (meter-level) position, and the ground robot must execute at centimeter-level precision.

| Stage | Precision |
|---|---|
| Position prior | meter-level |
| Navigation delivery | ±10 cm |
| Grasp capture window | **23 cm × 8 cm** |

Navigation gets the robot *near* the target, but not *into* the grasp window. 

---

## System architecture

```
Natural language   Translator → structured commands   (Qwen-2.5 · structured outputs)
    │
    ▼
goto can_1    Skill: Navigation      (move_base + TEB, ROS Noetic)  ★
    │
    ▼
Near can_1  Grounding, then grasp / place                (closed-loop visual approach)
    │
    ▼
Verifier     Execution outcome from a physical signal  (gripper servo readback)       ★
    │
    ▼
Memory       Task state (visited cans)
```


---

## Technical highlights

### 1. Closed-loop approach — bridging the precision gap

Navigation delivers ±10 cm; grasping needs a 6 cm window. Instead of trying to make navigation *more accurate*, I asked a different question: **does the robot's heading even need to be correct?**

It doesn't — the grasp only cares about the can's position in the arm base frame. So the robot doesn't rotate. Using the mecanum base's lateral motion (`linear.y`), a closed-loop controller translates the body until the can sits at (0.35, 0) in the base frame, with `angular.z = 0` throughout.

**Result:** approach loop converges 10/11 · no rotation, no drift · navigation stack untouched.

### 2. Physical-signal verification — not asking the model

The original grasp module always returned `True` — even on empty grabs. I replaced self-reported success with a **physical criterion**: the gripper servo's position readback.

| Readback | Meaning |
|---|---|
| ~63 | fully open |
| ~250 | **holding an object** |
| ~418 | closed on nothing (empty grab) |

This signal passes through neither the vision pipeline nor the language model, so it can't be fooled by a prompt or a hallucination. In embodied systems, letting a model judge its own success is a known failure mode — verification and execution share the same perception, so their errors correlate. An orthogonal physical signal breaks that correlation.

### 3. Real-world debugging (things simulation never shows)

- **Rotation-induced localization drift → root cause found.** Originally worked around by avoiding in-place rotation. Later traced to the real cause: **the LiDAR was mounted off-center**, so rotating in place swept the sensor through an arc — scan-matching saw motion the robot didn't make. Fixing the mounting offset resolved the drift at its source.
- **Robot grabbing itself.** The green chassis filled the bottom of the eye-in-hand camera; "largest contour" selected the robot's own body. Fixed by masking the bottom 30% of the frame and re-calibrating the LAB threshold (the old `max a = 110` was clipping half the can).
- **Mirror-finish cans have no depth at center.** Specular reflection returns invalid depth. Fixed by progressively widening the sampling window until ≥20 valid points, then taking the median.
- **Hard-coded 2 cm offset.** Four consecutive empty grabs with correct coordinates and valid IK. Traced — via the naked-eye observation "it's off to the right" — to a hard-coded constant. Removed; grasping went from 0/4 to consistent success.

---

## Hardware & low-level work

Hands-on with the full stack, not just the algorithms:

- **Servo bus control** via `board.py` — direct servo commands, ID mapping (base / shoulder / elbow / wrist / rotation / gripper)
- **Board communication constraints** — identified four defects in the vendor driver; cold-start allows only the first process to communicate
- **Sensor & power** — RPLidar A1 requires battery power for stable scans; serial port needs `chmod 666` after reboot
- **Platform** — Jetson · Ubuntu 20.04 · ROS Noetic · mecanum base · 5-DOF servo arm · **no wheel odometry**

---

## Known boundaries & next steps

| Aspect | Current state |
|---|---|
| Targets | red / green only, pre-calibrated LAB thresholds |
| Grasp range | 0.33 – 0.38 m; beyond → IK fails or knocks the can over |
| Transparent objects | can't grasp — no color blob, no valid depth |
| Semantic closed loop | **upstream link not yet connected** — failure info currently aborts rather than re-plans |

**Next:** connect the upstream path so the Verifier's four failure modes feed back into re-planning instead of aborting.

---

## Tech stack

ROS Noetic · move_base / TEB · hector SLAM · OpenCV (LAB color space) · depth camera · vendor IK (in-process) · Qwen-2.5 / Ollama structured outputs · Python

---

> Full source is being organized and can be provided on request or walked through in an interview.

---
---

# 中文说明

# JetRover — 面向地面抓取的自主移动操作系统

> 一个 LLM 驱动的移动机械臂:接收一句自然语言指令,自主导航到目标并抓取——填平米级位置先验与厘米级抓取之间的落差。

<p align="center">
  <img src="docs/demo.gif" width="600" alt="端到端演示">
</p>

📺 **完整演示:** [[YouTube link] · [Bilibili link]](https://b23.tv/y1Y0TMn)

---

## 做了什么

一句自然语言指令,驱动移动机器人在边走边建图的环境中自主导航、定位彩色易拉罐、抓取,并送到收集点。

**端到端成果:** 单次连续运行收集 5 个罐子 · 约 6 分钟 · 全程自主。

---

## 难在哪

目标场景是空地协同:空中平台提供粗略(米级)位置,地面机器人需以厘米级精度执行。

| 环节 | 精度 |
|---|---|
| 位置先验 | 米级 |
| 导航交付 | ±10 cm |
| 抓取捕获窗口 | **23 cm × 8 cm** |

导航能把车送到目标附近,但送不进抓取窗口。

---

## 系统架构

```
Translator   自然语言 → 结构化命令        (Qwen-2.5 · structured outputs)
    │
    ▼
goto can_1    skill：导航         (move_base + TEB, ROS Noetic)
    │
    ▼
车到罐子附近  逼近环 然后执行skill：夹取          
    │
    ▼
Verifier     从物理信号判定执行结果       (夹爪舵机回读)  
    │
    ▼
Memory       任务状态(已访问的罐子)
```



## 技术亮点

### 1. 闭环逼近 —— 填平精度落差

导航交付 ±10 cm,抓取需要 6 cm 窗口。我没有去把导航做得更准,而是换了个问题:**车的朝向到底需不需要对?**

不需要——抓取只关心罐子在臂基座坐标系下的位置。所以车不旋转。利用麦轮底盘的横向运动(`linear.y`),闭环控制车体平移,直到罐子落在基座系 (0.35, 0),全程 `angular.z = 0`。

**成果:** 逼近环收敛 10/11 · 不旋转、不漂移 · 导航栈零改动。

### 2. 物理判据验证 —— 不问模型

原始抓取模块永远返回 `True`,空夹也是。我用一个**物理判据**取代模型自评:夹爪舵机的位置回读。

| 读数 | 含义 |
|---|---|
| ~63 | 完全张开 |
| ~250 | **夹住物体** |
| ~418 | 空夹(合到底) |

这个信号不经过视觉链路,也不经过语言模型,因此提示词和幻觉都骗不了它。具身系统里让模型自评是否成功是公认的坑——验证和执行共享同一套感知,错误相关;正交的物理信号才能打破这种相关性。

### 3. 真实世界排障(仿真里遇不到的问题)

- **旋转导致定位漂移 → 找到真因。** 最初靠避免原地旋转绕开。后来定位到真正原因:**雷达偏心安装**,原地旋转时雷达其实在扫一个弧,scan-matching 看到了车没有做的运动。修正安装偏移后,漂移从根源解决。
- **机器人抓自己。** 绿色机身占满眼在手相机底部,「最大轮廓」选中了机身。修法:屏蔽画面底部 30%,并重标 LAB 阈值(旧的 `max a = 110` 把罐身切掉了一半)。
- **金属罐中心无深度。** 镜面反射返回无效深度。修法:逐步放大采样窗口直到 ≥20 个有效点,取中位数。
- **硬编码 2 cm 偏移。** 坐标正确、IK 有解却连续四次空夹。靠肉眼观察「偏右」定位到一个硬编码常数,删除后从 0/4 变为稳定成功。

---

## 硬件与底层

不只是算法,整条链路都上过手:

- **舵机总线控制** —— 通过 `board.py` 直接发舵机命令,ID 映射(底座/肩/肘/腕/旋转/夹爪)
- **板子通信约束** —— 识别出厂驱动的四个缺陷;冷启动时只有第一个进程能通信
- **传感与供电** —— RPLidar A1 需电池供电才稳;串口重启后需 `chmod 666`
- **平台** —— Jetson · Ubuntu 20.04 · ROS Noetic · 麦轮底盘 · 5-DOF 舵机臂 · **无轮式里程计**

---

## 已知边界与下一步

| 维度 | 现状 |
|---|---|
| 目标 | 仅 red / green,预标定 LAB 阈值 |
| 抓取距离 | 0.33 – 0.38 m,超出则 IK 无解或撞翻 |
| 透明物体 | 抓不了——无色块、无有效深度 |
| 语义闭环 | **上行通路未接**——失败信息目前用于中止而非重规划 |

**下一步:** 接上上行通路,让 Verifier 的四种失败原因回流至重规划,而非直接中止。

---

## 技术栈

ROS Noetic · move_base / TEB · hector SLAM · OpenCV(LAB 色彩空间)· 深度相机 · 厂商 IK(进程内)· Qwen-2.5 / Ollama structured outputs · Python

---

> 完整源码整理中,可应需提供。
