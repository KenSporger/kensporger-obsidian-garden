---
{"dg-publish":true,"permalink":"/学术研究/论文笔记/论文11-Semi-Automatic Flute Robot and Its Acoustic Sensing/","tags":["音乐","机器人"]}
---

# 概要

![[学术研究/论文笔记/assets/论文11-Semi-Automatic Flute Robot and Its Acoustic Sensing/d9ad055c397e7791a01c58e4bde394e8_MD5.png\|学术研究/论文笔记/assets/论文11-Semi-Automatic Flute Robot and Its Acoustic Sensing/d9ad055c397e7791a01c58e4bde394e8_MD5.png]]

论文设计了一种半自动长笛机械系统，并通过多组实验测试了系统的演奏效果。该系统包括两个部分：自动指法控制机构、射流偏移辅助机构。

自动指法控制机构由14个伺服舵机以线驱方式控制长笛按键开闭，并设计了按键帽以避免对乐器的损伤，通过MIDI将乐谱转为舵机指令实现演奏。射流偏移辅助机构使用舵机+齿轮来使笛头发生旋转。

在实验结果上，针对自动指法控制机构进行了全音域音高准确度测试、乐曲演奏音高准确度测试、按键动作速度测试；针对射流偏移辅助机构进行了射流偏移效果验证、笛头旋转速度测试。系统能够在±50音分的标准范围内输出预期音高，琴键的杠杆移动时间均控制在77.50毫秒以内；通过旋转笛头使得低音区三次谐波-二次谐波差值增加，改善了其音色，笛头旋转在40.00毫秒内完成。

# 内容

## 驱动原理

![[32e62a3f67f07258ca02eac44206b439_MD5.png\|501]]

论文使用的是闭孔长笛。对于标准的按键（水平向上、圆形）将伺服舵机（SG92R）通过线连接到长笛按键帽上，长笛按键帽采用3D打印，有三个锁片负责固定。

![[学术研究/论文笔记/assets/论文11-Semi-Automatic Flute Robot and Its Acoustic Sensing/4bd461e4ec623832e6c9b6e7b6e11c64_MD5.png\|学术研究/论文笔记/assets/论文11-Semi-Automatic Flute Robot and Its Acoustic Sensing/4bd461e4ec623832e6c9b6e7b6e11c64_MD5.png]]

对于非标准按键，如G#按键，不使用按键盖，直接将按键与线进行连接。对于侧边按键，采用[[机器人世界/硬件学习/01. 人形关节#^2cdc24\|丝杠]]的方式将旋转变为平移运动。

![[学术研究/论文笔记/assets/论文11-Semi-Automatic Flute Robot and Its Acoustic Sensing/3876f2243898769f4eaebb40d83d2e40_MD5.png\|学术研究/论文笔记/assets/论文11-Semi-Automatic Flute Robot and Its Acoustic Sensing/3876f2243898769f4eaebb40d83d2e40_MD5.png]]

最终成品如下。

![[学术研究/论文笔记/assets/论文11-Semi-Automatic Flute Robot and Its Acoustic Sensing/6aba1c7c3fecd01359b8b0bfcaee9026_MD5.png\|学术研究/论文笔记/assets/论文11-Semi-Automatic Flute Robot and Its Acoustic Sensing/6aba1c7c3fecd01359b8b0bfcaee9026_MD5.png]]
![[学术研究/论文笔记/assets/论文11-Semi-Automatic Flute Robot and Its Acoustic Sensing/932385e17e94c032f4ca9ff52b507e31_MD5.png\|学术研究/论文笔记/assets/论文11-Semi-Automatic Flute Robot and Its Acoustic Sensing/932385e17e94c032f4ca9ff52b507e31_MD5.png]]

## 实验设计

待补充。

# 评价

+ 这套系统优点在于无需改造乐器，按键帽的设计非常创新，但是对于非标准按键线驱仍然会对乐器产生损伤；
+ 该论文的研究目的是辅助初学者专注于气息，但是这样的半自动装置对于学习并无益处，一是没法以正常手型握持；二是指法与口风本身是联动的，要么都是人控，要么都是机控；
+ 采用舵机和齿轮组导致演奏噪音过大，线驱方案线容易产生形变，不耐用；
+ 射流偏移辅助机构不科学，因为旋转会影响吹奏者的口型，而且专业的演奏者并不会大幅度去调整笛子角度，二是去控制自己的口风方向；当然如果是全自动方案还是有一些参考意义；
+ 论文采用的基频、谐波分析、频谱分析、振幅检测这些测试方式值得后面借鉴；

# 灵感

+ 这种半自动演奏辅助装置一个卖点就是辅助演奏障碍人士，走社会创新；
+ 做一个声学量化监测、可视化吹奏姿势、物理引导指法的系统，去评测演奏的好坏；

# 下一步

+ **思考低噪音的驱动方案，或许可以结合灵巧手里的空心杯无刷电机；**
+ **了解MIDI原理和使用方式；**
+ 阅读其他文献，包括：
	+ [**模拟人工嘴唇与肺部用气的人形长笛机器人](https://doi.org/10.1016/j.robot.2016.08.024)：解决口风自动化、吐舌与手指配合的问题；**
	+ [基于听觉反馈的长笛演奏机器人WF-4RIV](https://doi.org/10.1109/ROBOT.2008.4543771)：能够自己去听自己演奏的音乐或者协奏者的音乐，达到反馈控制；
	+ [人形萨克斯演奏机器人的音乐动态表达性能](https://ieeexplore.ieee.org/document/8633975/)：让机器演奏有强弱变化，音乐的流动性（根据接下来的音高走向来控制？）；
	+ [AI在机器人演奏中的应用](https://doi.org/10.3390/engproc2025113053)
	+ [拟人化表演](https://doi.org/10.20965/jaciii.2019.p0838)
	+ [自动化四重奏](https://doi.org/10.1007/978-3-032-02555-5_52)

# 参考资料

+ [Semi-Automatic Flute Robot and Its Acoustic Sensing](https://arxiv.org/pdf/2603.14180)