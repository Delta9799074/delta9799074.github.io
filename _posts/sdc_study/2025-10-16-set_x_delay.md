---
title: set_input_delay/set_output_delay/set_max_delay傻傻分不清楚
description: 《Static Timing Analysis for Nanometer Designs:A Practical Approach》的学习笔记
author: Delta
date: 2025-10-16 18:41:00 +0800
categories: [笔记,数字IC前端]
tags: [STA,SDC,读书笔记]
pin: true
math: true
mermaid: true
# image:
#   path: /commons/devices-mockup.png
#   lqip: data:image/webp;base64,UklGRpoAAABXRUJQVlA4WAoAAAAQAAAADwAABwAAQUxQSDIAAAARL0AmbZurmr57yyIiqE8oiG0bejIYEQTgqiDA9vqnsUSI6H+oAERp2HZ65qP/VIAWAFZQOCBCAAAA8AEAnQEqEAAIAAVAfCWkAALp8sF8rgRgAP7o9FDvMCkMde9PK7euH5M1m6VWoDXf2FkP3BqV0ZYbO6NA/VFIAAAA
#   alt: Responsive rendering of Chirpy theme on multiple devices.
---


# set_input_delay/set_output_delay/set_max_delay傻傻分不清楚

刚学SDC约束的时候，总是被set_input_delay/set_output_delay/set_max_delay三个很相似的命令搞晕，只知道约束的值会对电路产生的影响，但是不知其所以然。刚好最近在看STA时序分析很经典的一本书《Static Timing Analysis for Nanometer Designs:A Practical Approach》，对这三个命令有了更深入的理解。接下来就一起来看看这三个命令的原理吧！

## 定义

1. `set_input_delay`命令：用于约束外部寄存器经过组合逻辑输入到模块port的延时。
    
    ![image.png](../assets/posts/sdc_study/2025-10-16-set_x_delay/image.png)
    
2. `set_output_delay`命令：用于约束模块输出信号到达下级模块的port的延时。
3. `set_max_delay`命令：用于约束pin to pin之间的延时。

## 约束施加场景

上面的定义可能不是这么好理解，我们用下面这张图说明。

![image.png](../assets/posts/sdc_study/2025-10-16-set_x_delay/image%201.png)

`set_input_delay`约束了从外部Board Device A寄存器到模块输入port的延时，`set_output_delay`约束了模块输出port到外部Board Device B寄存器的延时。而对于对于PAD-PAD的纯组合逻辑，使用`set_max_delay`来约束。为什么这里需要set_max_delay来约束PAD-PAD的组合逻辑呢？

如果我们对所有的port，参考board clock约束了input/output delay，那么按照时序约束，组合逻辑的Data Path Delay此时为T - input_delay - output_delay。该时序约束是非常紧张的，并且不符合实际的情况。所以需要为组合逻辑施加`set_max_delay`约束，施加了`set_max_delay`约束后，现在的Data Path Delay则为：`set_max_delay` 所指定的延时值 - input_delay - output_delay。

实际中对于以上电路往往是这样约束的：

1. 首先声明一个virtual clock，用于约束input/output delay
2. 约束模块输入port的input delay
3. 约束模块输出port的output delay
4. `set_max_delay`约束组合逻辑的延时

此外，`set_max_delay` 约束还常用于异步FIFO中跨时钟域的格雷码指针约束，这是为了保证从一个时钟域跨到另一个时钟域时，延时过大造成格雷码失效，这里就不展开说明了。

### input_delay约束为负值的情况

假设两个模块之间，由上级DFF输出，经过了一段组合逻辑到达下级模块的输入port，此时下级模块的input_delay应该怎么设？

![image.png](../assets/posts/sdc_study/2025-10-16-set_x_delay/image%202.png)

假设Launch clock与Capture clock为同一个，此时希望满足的时序如下：

setup：$T_{clk2q} + T_{c1} + T_{c2} + T_{setup}  /leq T+T_{skew}$

hold：$T_{clk2q} + T_{c1} + T_{c2}  /geq T_{skew} + T_{hold}$

因为UFF0和C1组合逻辑在模块的外部，这部分的延时需要`set_input_delay`命令约束。

对于setup，$T_{clk2q} + T_{c1}$ 的最大值为 $T + T_{skew} - T_{setup} - T_{c2}$，因此这部分延时用`set_input_delay -max` 约束，约束的值越大，对setup的约束越严，要求内部C2组合逻辑的延时越短，给前级留有的裕量越大。

对于hold，$T_{clk2q} + T_{c1}$ 的最小值为$T_{skew} + T_{hold} - T_{c2}$，因此这部分延时用`set_input_delay -min`约束，约束的值越小，对hold的约束越严，约束为负值时，工具会尽可能增加C2组合逻辑的延时以满足UFF1的hold要求。换句话说，外部C1组合逻辑的延时无法满足UFF1的hold要求，需要内部的C2逻辑增加组合逻辑延时以满足hold要求。

### output_delay约束为负值的情况

假设两个模块之间，上级模块的输出port，在到达下级DFF的输入时，中间经过了一段组合逻辑。此时上级模块的output delay应该怎么设？

![image.png](../assets/posts/sdc_study/2025-10-16-set_x_delay/image%203.png)

假设Launch clock与Capture clock为同一个，此时希望满足的时序如下：

setup：$T_{clk2q} + T_{c1} + T_{c2} + T_{setup}  \leq T+T_{skew}$

hold：$T_{clk2q} + T_{c1} + T_{c2}  \geq T_{skew} + T_{hold}$

因为C2组合逻辑和UFF1在该模块的外部，这部分的延时就需要使用`set_output_delay`约束。

对于setup，$T_{c2}+T_{setup} - T_{skew}$的最大值为$T - T_{clk2q} - T_{c1}$，因此这部分用`set_output_delay -max`命令来约束，这部分值约束的越大，对setup的约束越严，也就是说要求C1组合逻辑的延时尽量短，给后级所留有的裕量越大。

对于hold，$T_{hold}+T_{skew}-T_{c2}$的最小值为$T_{clk2q} + T_{c1}$，因此这部分用`set_output_delay -max`命令来约束，这部分的值约束的越小，对hold的约束越严，约束为负值时，工具必须在UFF0 → UFF1之间的路径上插入buffer等以满足后级的要求。换个方式理解，也就是说C2组合逻辑的最小延时不能满足UFF1的hold要求，需要该模块内部的C1组合逻辑增加信号的延时以满足UFF1的hold要求。