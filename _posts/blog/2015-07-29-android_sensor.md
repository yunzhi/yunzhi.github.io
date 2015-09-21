---
layout: post
title: Android Sensor 的分析
category: blog
description:  本文通过分析了Android sensor的驱动，hal层，framework层，和App中相应调用关系和流程。
---

＃手机传感器概要介绍

目前，智能手机上通常会安装如下传感器：

+ 1.光线传感器（用来感知外界环境光德强度以自动调节屏幕亮度）
+ 2.距离传感器（通过探测遮挡物距传感器的距离来调整通话中屏幕和触屏的工作状态，以节省电源，防止误按等）
+ 3.重力传感器（通过）
+ 4.指南针（磁力传感器）
+ 5.激光距离传感器

通过Android的code定义，我们在来看一下这些传感器。

	yunzhi-mac:hardware yunzhi$ grep -rn "#define SENSOR_TYPE" hardware/libhardware/include/hardware/sensors.h 
	sensors.h:156:#define SENSOR_TYPE_DEVICE_PRIVATE_BASE     0x10000
	sensors.h:230:#define SENSOR_TYPE_META_DATA                           (0)
	sensors.h:265:#define SENSOR_TYPE_ACCELEROMETER                    (1)
	sensors.h:282:#define SENSOR_TYPE_GEOMAGNETIC_FIELD                (2)
	sensors.h:283:#define SENSOR_TYPE_MAGNETIC_FIELD  SENSOR_TYPE_GEOMAGNETIC_FIELD
	sensors.h:323:#define SENSOR_TYPE_ORIENTATION                      (3)
	sensors.h:343:#define SENSOR_TYPE_GYROSCOPE                        (4)
	sensors.h:352:#define SENSOR_TYPE_LIGHT                            (5)
	sensors.h:361:#define SENSOR_TYPE_PRESSURE                         (6)
	sensors.h:364:#define SENSOR_TYPE_TEMPERATURE                      (7)
	sensors.h:376:#define SENSOR_TYPE_PROXIMITY                        (8)
	sensors.h:389:#define SENSOR_TYPE_GRAVITY                          (9)
	sensors.h:406:#define SENSOR_TYPE_LINEAR_ACCELERATION             (10)
	sensors.h:456:#define SENSOR_TYPE_ROTATION_VECTOR                 (11)
	sensors.h:466:#define SENSOR_TYPE_RELATIVE_HUMIDITY               (12)
	sensors.h:475:#define SENSOR_TYPE_AMBIENT_TEMPERATURE             (13)
	sensors.h:514:#define SENSOR_TYPE_MAGNETIC_FIELD_UNCALIBRATED     (14)
	sensors.h:538:#define SENSOR_TYPE_GAME_ROTATION_VECTOR            (15)
	sensors.h:581:#define SENSOR_TYPE_GYROSCOPE_UNCALIBRATED          (16)
	sensors.h:633:#define SENSOR_TYPE_SIGNIFICANT_MOTION              (17)
	sensors.h:659:#define SENSOR_TYPE_STEP_DETECTOR                   (18)
	sensors.h:712:#define SENSOR_TYPE_STEP_COUNTER                    (19)
	sensors.h:734:#define SENSOR_TYPE_GEOMAGNETIC_ROTATION_VECTOR            (20)
	
	
我们依次看看这十一种传感器

 

1 加速度传感器

加速度传感器又叫G-sensor，返回x、y、z三轴的加速度数值。

该数值包含地心引力的影响，单位是m/s^2。

将手机平放在桌面上，x轴默认为0，y轴默认0，z轴默认9.81。

将手机朝下放在桌面上，z轴为-9.81。

将手机向左倾斜，x轴为正值。

将手机向右倾斜，x轴为负值。

将手机向上倾斜，y轴为负值。

将手机向下倾斜，y轴为正值。

 

加速度传感器可能是最为成熟的一种mems产品，市场上的加速度传感器种类很多。

手机中常用的加速度传感器有BOSCH（博世）的BMA系列，AMK的897X系列，ST的LIS3X系列等。

这些传感器一般提供±2G至±16G的加速度测量范围，采用I2C或SPI接口和MCU相连，数据精度小于16bit。

 

2 磁力传感器

磁力传感器简称为M-sensor，返回x、y、z三轴的环境磁场数据。

该数值的单位是微特斯拉（micro-Tesla），用uT表示。

单位也可以是高斯（Gauss），1Tesla=10000Gauss。

硬件上一般没有独立的磁力传感器，磁力数据由电子罗盘传感器提供（E-compass）。

电子罗盘传感器同时提供下文的方向传感器数据。

 

3 方向传感器

方向传感器简称为O-sensor，返回三轴的角度数据，方向数据的单位是角度。

为了得到精确的角度数据，E-compass需要获取G-sensor的数据，

经过计算生产O-sensor数据，否则只能获取水平方向的角度。

方向传感器提供三个数据，分别为azimuth、pitch和roll。

azimuth：方位，返回水平时磁北极和Y轴的夹角，范围为0°至360°。

0°=北，90°=东，180°=南，270°=西。

pitch：x轴和水平面的夹角，范围为-180°至180°。

当z轴向y轴转动时，角度为正值。

roll：y轴和水平面的夹角，由于历史原因，范围为-90°至90°。

当x轴向z轴移动时，角度为正值。

 

电子罗盘在获取正确的数据前需要进行校准，通常可用8字校准法。

8字校准法要求用户使用需要校准的设备在空中做8字晃动，

原则上尽量多的让设备法线方向指向空间的所有8个象限。

 

手机中使用的电子罗盘芯片有AKM公司的897X系列，ST公司的LSM系列以及雅马哈公司等等。

由于需要读取G-sensor数据并计算出M-sensor和O-sensor数据，

因此厂商一般会提供一个后台daemon来完成工作，电子罗盘算法一般是公司私有产权。

 

4 陀螺仪传感器

陀螺仪传感器叫做Gyro-sensor，返回x、y、z三轴的角加速度数据。

角加速度的单位是radians/second。

根据Nexus S手机实测：

水平逆时针旋转，Z轴为正。

水平逆时针旋转，z轴为负。

向左旋转，y轴为负。

向右旋转，y轴为正。

向上旋转，x轴为负。

向下旋转，x轴为正。

 

ST的L3G系列的陀螺仪传感器比较流行，iphone4和google的nexus s中使用该种传感器。

 

5 光线感应传感器

光线感应传感器检测实时的光线强度，光强单位是lux，其物理意义是照射到单位面积上的光通量。

光线感应传感器主要用于Android系统的LCD自动亮度功能。

可以根据采样到的光强数值实时调整LCD的亮度。

 

6 压力传感器

压力传感器返回当前的压强，单位是百帕斯卡hectopascal（hPa）。

 

7 温度传感器

温度传感器返回当前的温度。

 

8 接近传感器

接近传感器检测物体与手机的距离，单位是厘米。

一些接近传感器只能返回远和近两个状态，

因此，接近传感器将最大距离返回远状态，小于最大距离返回近状态。

接近传感器可用于接听电话时自动关闭LCD屏幕以节省电量。

一些芯片集成了接近传感器和光线传感器两者功能。

 

 

下面三个传感器是Android2新提出的传感器类型，目前还不太清楚有哪些应用程序使用。

9 重力传感器

重力传感器简称GV-sensor，输出重力数据。

在地球上，重力数值为9.8，单位是m/s^2。

坐标系统与加速度传感器相同。

当设备复位时，重力传感器的输出与加速度传感器相同。

 

10 线性加速度传感器

线性加速度传感器简称LA-sensor。

线性加速度传感器是加速度传感器减去重力影响获取的数据。

单位是m/s^2，坐标系统与加速度传感器相同。

加速度传感器、重力传感器和线性加速度传感器的计算公式如下：

加速度 = 重力 + 线性加速度

 

11 旋转矢量传感器

旋转矢量传感器简称RV-sensor。

旋转矢量代表设备的方向，是一个将坐标轴和角度混合计算得到的数据。

RV-sensor输出三个数据：

x*sin(theta/2)

y*sin(theta/2)

z*sin(theta/2)

sin(theta/2)是RV的数量级。

RV的方向与轴旋转的方向相同。

RV的三个数值，与cos(theta/2)组成一个四元组。

RV的数据没有单位，使用的坐标系与加速度相同。	
	
	
	