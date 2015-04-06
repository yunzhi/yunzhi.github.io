---
layout: post
title: 从Android振子谈起 
description: Android从软件架构上从底向上可以分为 linux kernel层，hardware(hal)、library、Anroid Runtime层，framework层和Application层四层结构。本文通过分析振子的驱动来解析这四层间是如何调用的。
category: blog
---

Android手机上最通用的功能之一——振动。本文通过从驱动开始分析，进而深入hal层，framework层，最后以振动的一个应用结束。


## 硬件原理
Android的手机振子硬件原理很简单，一个偏心轮振子，上电时旋转产生振动，我们感受到得振动强度其实就是振动时间长短决定的。驱动是实现上于是也就简单，提供一个接口，应用层传递一个振动时间，振动器于是加该时间长度的电。

## 软件实现

软件上根据高通8926平台进行分析，这些部分的代码都是开源的，可以在[Code Aurora](codeaurora.org)上查看。

### 驱动部分
kernel/arch/arm/configs/msm8226_defconfig-----kernel配置文件

	CONFIG_QPNP_VIBRATOR=y
kernel/drivers/platform/msm/Kconfig

	 76 config QPNP_VIBRATOR 
	 77   tristate "Vibrator support for QPNP PMIC" 
	 78   depends on OF_SPMI 
	 79   help 
	 80     This option enables device driver support for the vibrator
	 81     on the Qualcomm's QPNP PMICs. The vibrator is connected on the
	 82     VIB_DRV_N line and can be controlled manually or by the DTEST lines. 
	 83     It uses the android timed-output framework. 
kernel/drivers/platform/msm/Makefile

	14 obj-$(CONFIG_QPNP_VIBRATOR) += qpnp-vibrator.o 

驱动文件kernel/drivers/platform/msm/qpnp-vibrator.c，由于该驱动不长，我就整体拷贝过来，然后分析。

    /* Copyright (c) 2013, The Linux Foundation. All rights reserved.
     *
     * This program is free software; you can redistribute it and/or modify
     * it under the terms of the GNU General Public License version 2 and
     * only version 2 as published by the Free Software Foundation.
     *
     * This program is distributed in the hope that it will be useful,
     * but WITHOUT ANY WARRANTY; without even the implied warranty of
     * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
     * GNU General Public License for more details.
     */
    
    #include <linux/module.h>
    #include <linux/init.h>
    #include <linux/kernel.h>
    #include <linux/errno.h>
    #include <linux/slab.h>
    #include <linux/hrtimer.h>
    #include <linux/of_device.h>
    #include <linux/spmi.h>
    
    #include <linux/qpnp/vibrator.h>
    #include "../../staging/android/timed_output.h"
    
    #define QPNP_VIB_VTG_CTL(base)		(base + 0x41)
    #define QPNP_VIB_EN_CTL(base)		(base + 0x46)
    
    #define QPNP_VIB_MAX_LEVEL		31
    #define QPNP_VIB_MIN_LEVEL		12
    
    #define QPNP_VIB_DEFAULT_TIMEOUT	15000
    #define QPNP_VIB_DEFAULT_VTG_LVL	3100
    
    #define QPNP_VIB_EN			BIT(7)
    #define QPNP_VIB_VTG_SET_MASK		0x1F
    #define QPNP_VIB_LOGIC_SHIFT		4
    
    struct qpnp_vib {
    	struct spmi_device *spmi;
    	struct hrtimer vib_timer;
    	struct timed_output_dev timed_dev;
    	struct work_struct work;
    
    	u8  reg_vtg_ctl;
    	u8  reg_en_ctl;
    	u16 base;
    	int state;
    	int vtg_level;
    	int timeout;
    	struct mutex lock;
    };
    
    static struct qpnp_vib *vib_dev;
    
    static int qpnp_vib_read_u8(struct qpnp_vib *vib, u8 *data, u16 reg)
    {
    	int rc;
    
    	rc = spmi_ext_register_readl(vib->spmi->ctrl, vib->spmi->sid,
    							reg, data, 1);
    	if (rc < 0)
    		dev_err(&vib->spmi->dev,
    			"Error reading address: %X - ret %X\n", reg, rc);
    
    	return rc;
    }
    
    static int qpnp_vib_write_u8(struct qpnp_vib *vib, u8 *data, u16 reg)
    {
    	int rc;
    
    	rc = spmi_ext_register_writel(vib->spmi->ctrl, vib->spmi->sid,
    							reg, data, 1);
    	if (rc < 0)
    		dev_err(&vib->spmi->dev,
    			"Error writing address: %X - ret %X\n", reg, rc);
    
    	return rc;
    }
    
    int qpnp_vibrator_config(struct qpnp_vib_config *vib_cfg)
    {
    	u8 reg = 0;
    	int rc = -EINVAL, level;
    
    	if (vib_dev == NULL) {
    		pr_err("%s: vib_dev is NULL\n", __func__);
    		return -ENODEV;
    	}
    
    	level = vib_cfg->drive_mV / 100;
    	if (level) {
    		if ((level < QPNP_VIB_MIN_LEVEL) ||
    				(level > QPNP_VIB_MAX_LEVEL)) {
    			dev_err(&vib_dev->spmi->dev, "Invalid voltage level\n");
    			return -EINVAL;
    		}
    	} else {
    		dev_err(&vib_dev->spmi->dev, "Voltage level not specified\n");
    		return -EINVAL;
    	}
    
    	/* Configure the VTG CTL regiser */
    	reg = vib_dev->reg_vtg_ctl;
    	reg &= ~QPNP_VIB_VTG_SET_MASK;
    	reg |= (level & QPNP_VIB_VTG_SET_MASK);
    	rc = qpnp_vib_write_u8(vib_dev, &reg, QPNP_VIB_VTG_CTL(vib_dev->base));
    	if (rc)
    		return rc;
    	vib_dev->reg_vtg_ctl = reg;
    
    	/* Configure the VIB ENABLE regiser */
    	reg = vib_dev->reg_en_ctl;
    	reg |= (!!vib_cfg->active_low) << QPNP_VIB_LOGIC_SHIFT;
    	if (vib_cfg->enable_mode == QPNP_VIB_MANUAL)
    		reg |= QPNP_VIB_EN;
    	else
    		reg |= BIT(vib_cfg->enable_mode - 1);
    	rc = qpnp_vib_write_u8(vib_dev, &reg, QPNP_VIB_EN_CTL(vib_dev->base));
    	if (rc < 0)
    		return rc;
    	vib_dev->reg_en_ctl = reg;
    
    	return rc;
    }
    EXPORT_SYMBOL(qpnp_vibrator_config);
    
    static int qpnp_vib_set(struct qpnp_vib *vib, int on)
    {
    	int rc;
    	u8 val;
    
    	if (on) {
    		val = vib->reg_vtg_ctl;
    		val &= ~QPNP_VIB_VTG_SET_MASK;
    		val |= (vib->vtg_level & QPNP_VIB_VTG_SET_MASK);
    		rc = qpnp_vib_write_u8(vib, &val, QPNP_VIB_VTG_CTL(vib->base));
    		if (rc < 0)
    			return rc;
    		vib->reg_vtg_ctl = val;
    		val = vib->reg_en_ctl;
    		val |= QPNP_VIB_EN;
    		rc = qpnp_vib_write_u8(vib, &val, QPNP_VIB_EN_CTL(vib->base));
    		if (rc < 0)
    			return rc;
    		vib->reg_en_ctl = val;
    	} else {
    		val = vib->reg_en_ctl;
    		val &= ~QPNP_VIB_EN;
    		rc = qpnp_vib_write_u8(vib, &val, QPNP_VIB_EN_CTL(vib->base));
    		if (rc < 0)
    			return rc;
    		vib->reg_en_ctl = val;
    	}
    
    	return rc;
    }
    
    static void qpnp_vib_enable(struct timed_output_dev *dev, int value)
    {
    	struct qpnp_vib *vib = container_of(dev, struct qpnp_vib,
    					 timed_dev);
    
    	mutex_lock(&vib->lock);
    	hrtimer_cancel(&vib->vib_timer);
    
    	if (value == 0)
    		vib->state = 0;
    	else {
    		value = (value > vib->timeout ?
    				 vib->timeout : value);
    		vib->state = 1;
    		hrtimer_start(&vib->vib_timer,
    			      ktime_set(value / 1000, (value % 1000) * 1000000),
    			      HRTIMER_MODE_REL);
    	}
    	mutex_unlock(&vib->lock);
    	schedule_work(&vib->work);
    }
    
    static void qpnp_vib_update(struct work_struct *work)
    {
    	struct qpnp_vib *vib = container_of(work, struct qpnp_vib,
    					 work);
    	qpnp_vib_set(vib, vib->state);
    }
    
    static int qpnp_vib_get_time(struct timed_output_dev *dev)
    {
    	struct qpnp_vib *vib = container_of(dev, struct qpnp_vib,
    							 timed_dev);
    
    	if (hrtimer_active(&vib->vib_timer)) {
    		ktime_t r = hrtimer_get_remaining(&vib->vib_timer);
    		return (int)ktime_to_us(r);
    	} else
    		return 0;
    }
    
    static enum hrtimer_restart qpnp_vib_timer_func(struct hrtimer *timer)
    {
    	struct qpnp_vib *vib = container_of(timer, struct qpnp_vib,
    							 vib_timer);
    
    	vib->state = 0;
    	schedule_work(&vib->work);
    
    	return HRTIMER_NORESTART;
    }
    
    #ifdef CONFIG_PM
    static int qpnp_vibrator_suspend(struct device *dev)
    {
    	struct qpnp_vib *vib = dev_get_drvdata(dev);
    
    	hrtimer_cancel(&vib->vib_timer);
    	cancel_work_sync(&vib->work);
    	/* turn-off vibrator */
    	qpnp_vib_set(vib, 0);
    
    	return 0;
    }
    #endif
    
    static SIMPLE_DEV_PM_OPS(qpnp_vibrator_pm_ops, qpnp_vibrator_suspend, NULL);
    
    static int __devinit qpnp_vibrator_probe(struct spmi_device *spmi)
    {
    	struct qpnp_vib *vib;
    	struct resource *vib_resource;
    	int rc;
    	u8 val;
    	u32 temp_val;
    
    	vib = devm_kzalloc(&spmi->dev, sizeof(*vib), GFP_KERNEL);
    	if (!vib)
    		return -ENOMEM;
    
    	vib->spmi = spmi;
    
    	vib->timeout = QPNP_VIB_DEFAULT_TIMEOUT;
    	rc = of_property_read_u32(spmi->dev.of_node,
    			"qcom,vib-timeout-ms", &temp_val);
    	if (!rc) {
    		vib->timeout = temp_val;
    	} else if (rc != EINVAL) {
    		dev_err(&spmi->dev, "Unable to read vib timeout\n");
    		return rc;
    	}
    
    	vib->vtg_level = QPNP_VIB_DEFAULT_VTG_LVL;
    	rc = of_property_read_u32(spmi->dev.of_node,
    			"qcom,vib-vtg-level-mV", &temp_val);
    	if (!rc) {
    		vib->vtg_level = temp_val;
    	} else if (rc != -EINVAL) {
    		dev_err(&spmi->dev, "Unable to read vtg level\n");
    		return rc;
    	}
    
    	vib->vtg_level /= 100;
    
    	vib_resource = spmi_get_resource(spmi, 0, IORESOURCE_MEM, 0);
    	if (!vib_resource) {
    		dev_err(&spmi->dev, "Unable to get vibrator base address\n");
    		return -EINVAL;
    	}
    	vib->base = vib_resource->start;
    
    	/* save the control registers values */
    	rc = qpnp_vib_read_u8(vib, &val, QPNP_VIB_VTG_CTL(vib->base));
    	if (rc < 0)
    		return rc;
    	vib->reg_vtg_ctl = val;
    
    	rc = qpnp_vib_read_u8(vib, &val, QPNP_VIB_EN_CTL(vib->base));
    	if (rc < 0)
    		return rc;
    	vib->reg_en_ctl = val;
    
    	mutex_init(&vib->lock);
    	INIT_WORK(&vib->work, qpnp_vib_update);
    
    	hrtimer_init(&vib->vib_timer, CLOCK_MONOTONIC, HRTIMER_MODE_REL);
    	vib->vib_timer.function = qpnp_vib_timer_func;
    
    	vib->timed_dev.name = "vibrator";
    	vib->timed_dev.get_time = qpnp_vib_get_time;
    	vib->timed_dev.enable = qpnp_vib_enable;
    
    	dev_set_drvdata(&spmi->dev, vib);
    
    	rc = timed_output_dev_register(&vib->timed_dev);
    	if (rc < 0)
    		return rc;
    
    	vib_dev = vib;
    
    	return rc;
    }
    
    static int  __devexit qpnp_vibrator_remove(struct spmi_device *spmi)
    {
    	struct qpnp_vib *vib = dev_get_drvdata(&spmi->dev);
    
    	cancel_work_sync(&vib->work);
    	hrtimer_cancel(&vib->vib_timer);
    	timed_output_dev_unregister(&vib->timed_dev);
    	mutex_destroy(&vib->lock);
    
    	return 0;
    }
    
    static struct of_device_id spmi_match_table[] = {
    	{	.compatible = "qcom,qpnp-vibrator",
    	},
    	{}
    };
    
    static struct spmi_driver qpnp_vibrator_driver = {
    	.driver		= {
    		.name	= "qcom,qpnp-vibrator",
    		.of_match_table = spmi_match_table,
    		.pm	= &qpnp_vibrator_pm_ops,
    	},
    	.probe		= qpnp_vibrator_probe,
    	.remove		= __devexit_p(qpnp_vibrator_remove),
    };
    
    static int __init qpnp_vibrator_init(void)
    {
    	return spmi_driver_register(&qpnp_vibrator_driver);
    }
    module_init(qpnp_vibrator_init);
    
    static void __exit qpnp_vibrator_exit(void)
    {
    	return spmi_driver_unregister(&qpnp_vibrator_driver);
    }
    module_exit(qpnp_vibrator_exit);
    
    MODULE_DESCRIPTION("qpnp vibrator driver");
    MODULE_LICENSE("GPL v2");


头文件 kernel/include/linux/qpnp/vibrator.h
    
    #ifndef __QPNP_VIBRATOR_H__
    #define __QPNP_VIBRATOR_H__
    
    enum qpnp_vib_en_mode {
    	QPNP_VIB_MANUAL,
    	QPNP_VIB_DTEST1,
    	QPNP_VIB_DTEST2,
    	QPNP_VIB_DTEST3,
    };
    
    struct qpnp_vib_config {
    	u16			drive_mV;
    	u8			active_low;
    	enum qpnp_vib_en_mode	enable_mode;
    };
    #if defined(CONFIG_QPNP_VIBRATOR)
    
    int qpnp_vibrator_config(struct qpnp_vib_config *vib_config);
    #else
    
    static inline int qpnp_vibrator_config(struct qpnp_vib_config *vib_config)
    {
    	return -ENODEV;
    }
    #endif
    
    #endif /* __QPNP_VIBRATOR_H__ */

驱动分析：

+ 1.高通从8974之后的平台引入了device tree（文件后缀名位.dts和.dtsi）作为硬件描述的主要手段。和之前ARM驱动的主要区别是，对于设备而言，不再注册设备了，在boot.img解压后，驱动加载前，会遍历整个device tree(编译好的文件是二级制，后缀名位.dtb的格式打包在boot.img的最后)，驱动的match函数不在根据name去做总线判断是否成功的标志，而是以 compatible作为判断的依据。这个驱动中，在振子的device tree文件中做了兼容性设定为“qcom,qpnp-vibrator”，相应的驱动也设置了这个标志。于是probe函数被调用。
+ 2.振子在内核中得数据结构主要是一个 timed_output_dev 的结构，因此probe函数主要的作用就是初始化该结构的一些成员，然后将这个结构注册到系统中，timed_output_dev_register这个函数会创建/sys/class/timed_output/vibrator这样一个目录，该目录下有一个enable的文件, `cat enable` 会返回振动剩余的时间值，`echo 1000 > enable ` （需要root权限）会使手机振动1s。显然的，cat对应了timed_output_dev中得get_time的方法，写对应了enable函数。
+ 3.为了使振动时间更为精确，驱动中使用了hrtimer这个高精度内核定时器。（看到一些资料说这个定时器的使用红黑树实现的，改天有空了可以来看看，学习下这种没有用过的结构）。其功能和一般定时器一样，只是实现了us级的定时。定时器时间到后调用回调函数。
+ 4.enable函数中，考虑了可从入的问题，例如应用让振子振动2s，在振子还没有到时间的时候，又要求振动1s，于是乎，在第二次调用enable函数的时候，先取消掉之前的hrtimer，重新设定新的振动时间，然后将新的振动work推入队列，让系统进行调度。系统执行work的回调函数后，开始振动。如果时间设为零，则停止振动。开始和停止振动的实现方式就是写PMU的寄存器，让pmic的管教输出和停止电平来实现的。
+ 5.一般的，应用写下来的振动时间传给了hrtimer，于是在定时器时间到了之后，hrtimer的回调函数启动，用来停止振动，这个函数的返回值是HTTIMER_NORESTART，即执行完了这个定时器线程不再重启。
+ 6.最后还看到振子的驱动中也存在电源管理的函数，qpnp_vibrator_suspend，如果定义了CONFIG_PM，则系统suspend的时候振子驱动也将hrtimer取消掉，同时设置pmic相应的管脚输出低电平。从而保证在睡眠后振子不会继续振动。

### HAL层实现

代码路径位于：
hardware/libhardware_legacy/include/hardware_legacy/vibrator.h
hardware/libhardware_legacy/vibrator/vibrator.c
hardware/libhardware_legacy/vibrator/Android.mk

Android.mk

	# Copyright 2006 The Android Open Source Project

	LOCAL_SRC_FILES += vibrator/vibrator.c

vibrator.h

    /*
     * Copyright (C) 2008 The Android Open Source Project
     *
     * Licensed under the Apache License, Version 2.0 (the "License");
     * you may not use this file except in compliance with the License.
     * You may obtain a copy of the License at
     *
     *      http://www.apache.org/licenses/LICENSE-2.0
     *
     * Unless required by applicable law or agreed to in writing, software
     * distributed under the License is distributed on an "AS IS" BASIS,
     * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
     * See the License for the specific language governing permissions and
     * limitations under the License.
     */
    
    #ifndef _HARDWARE_VIBRATOR_H
    #define _HARDWARE_VIBRATOR_H
    
    #if __cplusplus
    extern "C" {
    #endif
    
    /**
     * Return whether the device has a vibrator.
     *
     * @return 1 if a vibrator exists, 0 if it doesn't.
     */
    int vibrator_exists();
    
    /**
     * Turn on vibrator
     *
     * @param timeout_ms number of milliseconds to vibrate
     *
     * @return 0 if successful, -1 if error
     */
    int vibrator_on(int timeout_ms);
    
    /**
     * Turn off vibrator
     *
     * @return 0 if successful, -1 if error
     */
    int vibrator_off();
    
    #if __cplusplus
    }  // extern "C"
    #endif
    
    #endif  // _HARDWARE_VIBRATOR_H


vibrator.c

    #include <hardware_legacy/vibrator.h>
    #include "qemu.h"
    
    #include <stdio.h>
    #include <unistd.h>
    #include <fcntl.h>
    #include <errno.h>
    
    #define THE_DEVICE "/sys/class/timed_output/vibrator/enable"
    
    int vibrator_exists()
    {
        int fd;
    
    #ifdef QEMU_HARDWARE
        if (qemu_check()) {
            return 1;
        }
    #endif
    
        fd = open(THE_DEVICE, O_RDWR);
        if(fd < 0)
            return 0;
        close(fd);
        return 1;
    }
    
    static int sendit(int timeout_ms)
    {
        int nwr, ret, fd;
        char value[20];
    
    #ifdef QEMU_HARDWARE
        if (qemu_check()) {
            return qemu_control_command( "vibrator:%d", timeout_ms );
        }
    #endif
    
        fd = open(THE_DEVICE, O_RDWR);
        if(fd < 0)
            return errno;
    
        nwr = sprintf(value, "%d\n", timeout_ms);
        ret = write(fd, value, nwr);
    
        close(fd);
    
        return (ret == nwr) ? 0 : -1;
    }
    
    int vibrator_on(int timeout_ms)
    {
        /* constant on, up to maximum allowed time */
        return sendit(timeout_ms);
    }
    
    int vibrator_off()
    {
        return sendit(0);
    }

#### hal层代码分析

+ 1.hal层中，对应驱动的接口函数是

> int vibrator_exists();用来查看是否存在振子这个设备，使用了open/close这组系统调用
> static int sendit(int timeout_ms);用来驱动振子振动多少时间，也使用了open/close这组系统调用

+ 2.hal层中，对应framework层的接口函数是

> int vibrator_exists();用来查看是否存在振子这个设备，使用了open/close这组系统调用
> int vibrator_on(int timeout_ms); 
> int vibrator_off()；

+ 3. enable这个文件节点的属性默认是0644，owner是root，系统想要使用必须改成system。
> -rw-r--r-- system   system       4096 2009-01-02 21:00 enable
> 这个属性值的改变在device/qcom/common/rootdir/etc/init.qcom.factory.sh中实现

### framework实现

这部分实现有两个部分，jni代码和java部分的代码。

#### JNI代码

代码位于 frameworks/base/services/jni/com_android_server_VibratorService.cpp

    #define LOG_TAG "VibratorService"
    
    #include "jni.h"
    #include "JNIHelp.h"
    #include "android_runtime/AndroidRuntime.h"
    
    #include <utils/misc.h>
    #include <utils/Log.h>
    #include <hardware_legacy/vibrator.h>
    
    #include <stdio.h>
    
    namespace android
    {
    
    static jboolean vibratorExists(JNIEnv *env, jobject clazz)
    {
        return vibrator_exists() > 0 ? JNI_TRUE : JNI_FALSE;
    }
    
    static void vibratorOn(JNIEnv *env, jobject clazz, jlong timeout_ms)
    {
        // ALOGI("vibratorOn\n");
        vibrator_on(timeout_ms);
    }
    
    static void vibratorOff(JNIEnv *env, jobject clazz)
    {
        // ALOGI("vibratorOff\n");
        vibrator_off();
    }
    
    static JNINativeMethod method_table[] = {
        { "vibratorExists", "()Z", (void*)vibratorExists },
        { "vibratorOn", "(J)V", (void*)vibratorOn },
        { "vibratorOff", "()V", (void*)vibratorOff }
    };
    
    int register_android_server_VibratorService(JNIEnv *env)
    {
        return jniRegisterNativeMethods(env, "com/android/server/VibratorService",
                method_table, NELEM(method_table));
    }
    
    };

JNI代码分析

1. 包含头文件 #include "jni.h"
2. 使用`int register_android_server_VibratorService(JNIEnv *env)`注册接口。
3. static JNINativeMethod method_table[]实现java函数和cpp函数的互相调用及传参定义
4. 需要在frameworks/base/services/jni/onload.cpp中注册该服务,同时将该文件添加到frameworks/base/services/jni/Android.mk文件中LOCAL_SRC_FILES参与编译

#### Java代码

framework中，振动的事件被作为一个服务来使用，代码路径位于
`frameworks/base/services/java/com/android/server/VibratorService.java`

    /*
     * Copyright (C) 2008 The Android Open Source Project
     *
     * Licensed under the Apache License, Version 2.0 (the "License");
     * you may not use this file except in compliance with the License.
     * You may obtain a copy of the License at
     *
     *      http://www.apache.org/licenses/LICENSE-2.0
     *
     * Unless required by applicable law or agreed to in writing, software
     * distributed under the License is distributed on an "AS IS" BASIS,
     * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
     * See the License for the specific language governing permissions and
     * limitations under the License.
     */
    
    package com.android.server;
    
    import android.app.AppOpsManager;
    import android.content.BroadcastReceiver;
    import android.content.Context;
    import android.content.Intent;
    import android.content.IntentFilter;
    import android.content.pm.PackageManager;
    import android.database.ContentObserver;
    import android.hardware.input.InputManager;
    import android.os.BatteryStats;
    import android.os.Handler;
    import android.os.IVibratorService;
    import android.os.PowerManager;
    import android.os.Process;
    import android.os.RemoteException;
    import android.os.IBinder;
    import android.os.Binder;
    import android.os.ServiceManager;
    import android.os.SystemClock;
    import android.os.UserHandle;
    import android.os.Vibrator;
    import android.os.WorkSource;
    import android.provider.Settings;
    import android.provider.Settings.SettingNotFoundException;
    import android.util.Slog;
    import android.view.InputDevice;
    
    import com.android.internal.app.IAppOpsService;
    import com.android.internal.app.IBatteryStats;
    
    import java.util.ArrayList;
    import java.util.LinkedList;
    import java.util.ListIterator;
    
    public class VibratorService extends IVibratorService.Stub
            implements InputManager.InputDeviceListener {
        private static final String TAG = "VibratorService";
    
        private final LinkedList<Vibration> mVibrations;
        private Vibration mCurrentVibration;
        private final WorkSource mTmpWorkSource = new WorkSource();
        private final Handler mH = new Handler();
    
        private final Context mContext;
        private final PowerManager.WakeLock mWakeLock;
        private final IAppOpsService mAppOpsService;
        private final IBatteryStats mBatteryStatsService;
        private InputManager mIm;
    
        volatile VibrateThread mThread;
    
        // mInputDeviceVibrators lock should be acquired after mVibrations lock, if both are
        // to be acquired
        private final ArrayList<Vibrator> mInputDeviceVibrators = new ArrayList<Vibrator>();
        private boolean mVibrateInputDevicesSetting; // guarded by mInputDeviceVibrators
        private boolean mInputDeviceListenerRegistered; // guarded by mInputDeviceVibrators
    
        private int mCurVibUid = -1;
    
        native static boolean vibratorExists();
        native static void vibratorOn(long milliseconds);
        native static void vibratorOff();
    
        private class Vibration implements IBinder.DeathRecipient {
            private final IBinder mToken;
            private final long    mTimeout;
            private final long    mStartTime;
            private final long[]  mPattern;
            private final int     mRepeat;
            private final int     mUid;
            private final String  mPackageName;
    
            Vibration(IBinder token, long millis, int uid, String packageName) {
                this(token, millis, null, 0, uid, packageName);
            }
    
            Vibration(IBinder token, long[] pattern, int repeat, int uid, String packageName) {
                this(token, 0, pattern, repeat, uid, packageName);
            }
    
            private Vibration(IBinder token, long millis, long[] pattern,
                    int repeat, int uid, String packageName) {
                mToken = token;
                mTimeout = millis;
                mStartTime = SystemClock.uptimeMillis();
                mPattern = pattern;
                mRepeat = repeat;
                mUid = uid;
                mPackageName = packageName;
            }
    
            public void binderDied() {
                synchronized (mVibrations) {
                    mVibrations.remove(this);
                    if (this == mCurrentVibration) {
                        doCancelVibrateLocked();
                        startNextVibrationLocked();
                    }
                }
            }
    
            public boolean hasLongerTimeout(long millis) {
                if (mTimeout == 0) {
                    // This is a pattern, return false to play the simple
                    // vibration.
                    return false;
                }
                if ((mStartTime + mTimeout)
                        < (SystemClock.uptimeMillis() + millis)) {
                    // If this vibration will end before the time passed in, let
                    // the new vibration play.
                    return false;
                }
                return true;
            }
        }
    
        VibratorService(Context context) {
            // Reset the hardware to a default state, in case this is a runtime
            // restart instead of a fresh boot.
            vibratorOff();
    
            mContext = context;
            PowerManager pm = (PowerManager)context.getSystemService(
                    Context.POWER_SERVICE);
            mWakeLock = pm.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK, "*vibrator*");
            mWakeLock.setReferenceCounted(true);
    
            mAppOpsService = IAppOpsService.Stub.asInterface(ServiceManager.getService(Context.APP_OPS_SERVICE));
            mBatteryStatsService = IBatteryStats.Stub.asInterface(ServiceManager.getService(
                    BatteryStats.SERVICE_NAME));
    
            mVibrations = new LinkedList<Vibration>();
    
            IntentFilter filter = new IntentFilter();
            filter.addAction(Intent.ACTION_SCREEN_OFF);
            context.registerReceiver(mIntentReceiver, filter);
        }
    
        public void systemReady() {
            mIm = (InputManager)mContext.getSystemService(Context.INPUT_SERVICE);
    
            mContext.getContentResolver().registerContentObserver(
                    Settings.System.getUriFor(Settings.System.VIBRATE_INPUT_DEVICES), true,
                    new ContentObserver(mH) {
                        @Override
                        public void onChange(boolean selfChange) {
                            updateInputDeviceVibrators();
                        }
                    }, UserHandle.USER_ALL);
    
            mContext.registerReceiver(new BroadcastReceiver() {
                @Override
                public void onReceive(Context context, Intent intent) {
                    updateInputDeviceVibrators();
                }
            }, new IntentFilter(Intent.ACTION_USER_SWITCHED), null, mH);
    
            updateInputDeviceVibrators();
        }
    
        public boolean hasVibrator() {
            return doVibratorExists();
        }
    
        private void verifyIncomingUid(int uid) {
            if (uid == Binder.getCallingUid()) {
                return;
            }
            if (Binder.getCallingPid() == Process.myPid()) {
                return;
            }
            mContext.enforcePermission(android.Manifest.permission.UPDATE_APP_OPS_STATS,
                    Binder.getCallingPid(), Binder.getCallingUid(), null);
        }
    
        public void vibrate(int uid, String packageName, long milliseconds, IBinder token) {
            if (mContext.checkCallingOrSelfPermission(android.Manifest.permission.VIBRATE)
                    != PackageManager.PERMISSION_GRANTED) {
                throw new SecurityException("Requires VIBRATE permission");
            }
            verifyIncomingUid(uid);
            // We're running in the system server so we cannot crash. Check for a
            // timeout of 0 or negative. This will ensure that a vibration has
            // either a timeout of > 0 or a non-null pattern.
            if (milliseconds <= 0 || (mCurrentVibration != null
                    && mCurrentVibration.hasLongerTimeout(milliseconds))) {
                // Ignore this vibration since the current vibration will play for
                // longer than milliseconds.
                return;
            }
    
            Vibration vib = new Vibration(token, milliseconds, uid, packageName);
    
            final long ident = Binder.clearCallingIdentity();
            try {
                synchronized (mVibrations) {
                    removeVibrationLocked(token);
                    doCancelVibrateLocked();
                    mCurrentVibration = vib;
                    startVibrationLocked(vib);
                }
            } finally {
                Binder.restoreCallingIdentity(ident);
            }
        }
    
        private boolean isAll0(long[] pattern) {
            int N = pattern.length;
            for (int i = 0; i < N; i++) {
                if (pattern[i] != 0) {
                    return false;
                }
            }
            return true;
        }
    
        public void vibratePattern(int uid, String packageName, long[] pattern, int repeat,
                IBinder token) {
            if (mContext.checkCallingOrSelfPermission(android.Manifest.permission.VIBRATE)
                    != PackageManager.PERMISSION_GRANTED) {
                throw new SecurityException("Requires VIBRATE permission");
            }
            verifyIncomingUid(uid);
            // so wakelock calls will succeed
            long identity = Binder.clearCallingIdentity();
            try {
                if (false) {
                    String s = "";
                    int N = pattern.length;
                    for (int i=0; i<N; i++) {
                        s += " " + pattern[i];
                    }
                    Slog.i(TAG, "vibrating with pattern: " + s);
                }
    
                // we're running in the server so we can't fail
                if (pattern == null || pattern.length == 0
                        || isAll0(pattern)
                        || repeat >= pattern.length || token == null) {
                    return;
                }
    
                Vibration vib = new Vibration(token, pattern, repeat, uid, packageName);
                try {
                    token.linkToDeath(vib, 0);
                } catch (RemoteException e) {
                    return;
                }
    
                synchronized (mVibrations) {
                    removeVibrationLocked(token);
                    doCancelVibrateLocked();
                    if (repeat >= 0) {
                        mVibrations.addFirst(vib);
                        startNextVibrationLocked();
                    } else {
                        // A negative repeat means that this pattern is not meant
                        // to repeat. Treat it like a simple vibration.
                        mCurrentVibration = vib;
                        startVibrationLocked(vib);
                    }
                }
            }
            finally {
                Binder.restoreCallingIdentity(identity);
            }
        }
    
        public void cancelVibrate(IBinder token) {
            mContext.enforceCallingOrSelfPermission(
                    android.Manifest.permission.VIBRATE,
                    "cancelVibrate");
    
            // so wakelock calls will succeed
            long identity = Binder.clearCallingIdentity();
            try {
                synchronized (mVibrations) {
                    final Vibration vib = removeVibrationLocked(token);
                    if (vib == mCurrentVibration) {
                        doCancelVibrateLocked();
                        startNextVibrationLocked();
                    }
                }
            }
            finally {
                Binder.restoreCallingIdentity(identity);
            }
        }
    
        private final Runnable mVibrationRunnable = new Runnable() {
            public void run() {
                synchronized (mVibrations) {
                    doCancelVibrateLocked();
                    startNextVibrationLocked();
                }
            }
        };
    
        // Lock held on mVibrations
        private void doCancelVibrateLocked() {
            if (mThread != null) {
                synchronized (mThread) {
                    mThread.mDone = true;
                    mThread.notify();
                }
                mThread = null;
            }
            doVibratorOff();
            mH.removeCallbacks(mVibrationRunnable);
            reportFinishVibrationLocked();
        }
    
        // Lock held on mVibrations
        private void startNextVibrationLocked() {
            if (mVibrations.size() <= 0) {
                reportFinishVibrationLocked();
                mCurrentVibration = null;
                return;
            }
            mCurrentVibration = mVibrations.getFirst();
            startVibrationLocked(mCurrentVibration);
        }
    
        // Lock held on mVibrations
        private void startVibrationLocked(final Vibration vib) {
            try {
                int mode = mAppOpsService.startOperation(AppOpsManager.getToken(mAppOpsService),
                        AppOpsManager.OP_VIBRATE, vib.mUid, vib.mPackageName);
                if (mode != AppOpsManager.MODE_ALLOWED) {
                    if (mode == AppOpsManager.MODE_ERRORED) {
                        Slog.w(TAG, "Would be an error: vibrate from uid " + vib.mUid);
                    }
                    mH.post(mVibrationRunnable);
                    return;
                }
            } catch (RemoteException e) {
            }
            if (vib.mTimeout != 0) {
                doVibratorOn(vib.mTimeout, vib.mUid);
                mH.postDelayed(mVibrationRunnable, vib.mTimeout);
            } else {
                // mThread better be null here. doCancelVibrate should always be
                // called before startNextVibrationLocked or startVibrationLocked.
                mThread = new VibrateThread(vib);
                mThread.start();
            }
        }
    
        private void reportFinishVibrationLocked() {
            if (mCurrentVibration != null) {
                try {
                    mAppOpsService.finishOperation(AppOpsManager.getToken(mAppOpsService),
                            AppOpsManager.OP_VIBRATE, mCurrentVibration.mUid,
                            mCurrentVibration.mPackageName);
                } catch (RemoteException e) {
                }
                mCurrentVibration = null;
            }
        }
    
        // Lock held on mVibrations
        private Vibration removeVibrationLocked(IBinder token) {
            ListIterator<Vibration> iter = mVibrations.listIterator(0);
            while (iter.hasNext()) {
                Vibration vib = iter.next();
                if (vib.mToken == token) {
                    iter.remove();
                    unlinkVibration(vib);
                    return vib;
                }
            }
            // We might be looking for a simple vibration which is only stored in
            // mCurrentVibration.
            if (mCurrentVibration != null && mCurrentVibration.mToken == token) {
                unlinkVibration(mCurrentVibration);
                return mCurrentVibration;
            }
            return null;
        }
    
        private void unlinkVibration(Vibration vib) {
            if (vib.mPattern != null) {
                // If Vibration object has a pattern,
                // the Vibration object has also been linkedToDeath.
                vib.mToken.unlinkToDeath(vib, 0);
            }
        }
    
        private void updateInputDeviceVibrators() {
            synchronized (mVibrations) {
                doCancelVibrateLocked();
    
                synchronized (mInputDeviceVibrators) {
                    mVibrateInputDevicesSetting = false;
                    try {
                        mVibrateInputDevicesSetting = Settings.System.getIntForUser(
                                mContext.getContentResolver(),
                                Settings.System.VIBRATE_INPUT_DEVICES, UserHandle.USER_CURRENT) > 0;
                    } catch (SettingNotFoundException snfe) {
                    }
    
                    if (mVibrateInputDevicesSetting) {
                        if (!mInputDeviceListenerRegistered) {
                            mInputDeviceListenerRegistered = true;
                            mIm.registerInputDeviceListener(this, mH);
                        }
                    } else {
                        if (mInputDeviceListenerRegistered) {
                            mInputDeviceListenerRegistered = false;
                            mIm.unregisterInputDeviceListener(this);
                        }
                    }
    
                    mInputDeviceVibrators.clear();
                    if (mVibrateInputDevicesSetting) {
                        int[] ids = mIm.getInputDeviceIds();
                        for (int i = 0; i < ids.length; i++) {
                            InputDevice device = mIm.getInputDevice(ids[i]);
                            Vibrator vibrator = device.getVibrator();
                            if (vibrator.hasVibrator()) {
                                mInputDeviceVibrators.add(vibrator);
                            }
                        }
                    }
                }
    
                startNextVibrationLocked();
            }
        }
    
        @Override
        public void onInputDeviceAdded(int deviceId) {
            updateInputDeviceVibrators();
        }
    
        @Override
        public void onInputDeviceChanged(int deviceId) {
            updateInputDeviceVibrators();
        }
    
        @Override
        public void onInputDeviceRemoved(int deviceId) {
            updateInputDeviceVibrators();
        }
    
        private boolean doVibratorExists() {
            // For now, we choose to ignore the presence of input devices that have vibrators
            // when reporting whether the device has a vibrator.  Applications often use this
            // information to decide whether to enable certain features so they expect the
            // result of hasVibrator() to be constant.  For now, just report whether
            // the device has a built-in vibrator.
            //synchronized (mInputDeviceVibrators) {
            //    return !mInputDeviceVibrators.isEmpty() || vibratorExists();
            //}
            return vibratorExists();
        }
    
        private void doVibratorOn(long millis, int uid) {
            synchronized (mInputDeviceVibrators) {
                try {
                    mBatteryStatsService.noteVibratorOn(uid, millis);
                    mCurVibUid = uid;
                } catch (RemoteException e) {
                }
                final int vibratorCount = mInputDeviceVibrators.size();
                if (vibratorCount != 0) {
                    for (int i = 0; i < vibratorCount; i++) {
                        mInputDeviceVibrators.get(i).vibrate(millis);
                    }
                } else {
                    vibratorOn(millis);
                }
            }
        }
    
        private void doVibratorOff() {
            synchronized (mInputDeviceVibrators) {
                if (mCurVibUid >= 0) {
                    try {
                        mBatteryStatsService.noteVibratorOff(mCurVibUid);
                    } catch (RemoteException e) {
                    }
                    mCurVibUid = -1;
                }
                final int vibratorCount = mInputDeviceVibrators.size();
                if (vibratorCount != 0) {
                    for (int i = 0; i < vibratorCount; i++) {
                        mInputDeviceVibrators.get(i).cancel();
                    }
                } else {
                    vibratorOff();
                }
            }
        }
    
        private class VibrateThread extends Thread {
            final Vibration mVibration;
            boolean mDone;
    
            VibrateThread(Vibration vib) {
                mVibration = vib;
                mTmpWorkSource.set(vib.mUid);
                mWakeLock.setWorkSource(mTmpWorkSource);
                mWakeLock.acquire();
            }
    
            private void delay(long duration) {
                if (duration > 0) {
                    long bedtime = duration + SystemClock.uptimeMillis();
                    do {
                        try {
                            this.wait(duration);
                        }
                        catch (InterruptedException e) {
                        }
                        if (mDone) {
                            break;
                        }
                        duration = bedtime - SystemClock.uptimeMillis();
                    } while (duration > 0);
                }
            }
    
            public void run() {
                Process.setThreadPriority(Process.THREAD_PRIORITY_URGENT_DISPLAY);
                synchronized (this) {
                    final long[] pattern = mVibration.mPattern;
                    final int len = pattern.length;
                    final int repeat = mVibration.mRepeat;
                    final int uid = mVibration.mUid;
                    int index = 0;
                    long duration = 0;
    
                    while (!mDone) {
                        // add off-time duration to any accumulated on-time duration
                        if (index < len) {
                            duration += pattern[index++];
                        }
    
                        // sleep until it is time to start the vibrator
                        delay(duration);
                        if (mDone) {
                            break;
                        }
    
                        if (index < len) {
                            // read on-time duration and start the vibrator
                            // duration is saved for delay() at top of loop
                            duration = pattern[index++];
                            if (duration > 0) {
                                VibratorService.this.doVibratorOn(duration, uid);
                            }
                        } else {
                            if (repeat < 0) {
                                break;
                            } else {
                                index = repeat;
                                duration = 0;
                            }
                        }
                    }
                    mWakeLock.release();
                }
                synchronized (mVibrations) {
                    if (mThread == this) {
                        mThread = null;
                    }
                    if (!mDone) {
                        // If this vibration finished naturally, start the next
                        // vibration.
                        mVibrations.remove(mVibration);
                        unlinkVibration(mVibration);
                        startNextVibrationLocked();
                    }
                }
            }
        };
    
        BroadcastReceiver mIntentReceiver = new BroadcastReceiver() {
            public void onReceive(Context context, Intent intent) {
                if (intent.getAction().equals(Intent.ACTION_SCREEN_OFF)) {
                    synchronized (mVibrations) {
                        doCancelVibrateLocked();
    
                        int size = mVibrations.size();
                        for(int i = 0; i < size; i++) {
                            unlinkVibration(mVibrations.get(i));
                        }
    
                        mVibrations.clear();
                    }
                }
            }
        };
    }


可以看到振动服务类继承与IVibratorService.Stub，并且实现了InputManager.InputDeviceListener这个接口。同时涉及了Android四大组件之一的服务的一些用法，这方面还需要了解下，就暂时不做深入分析了。


frameworks/base/core/java/android/os/IVibratorService.aidl

	package android.os;                                                                    
                                                                                       
	/** {@hide} */                                                                         
	interface IVibratorService                                                             
	{                                                                                      
    		boolean hasVibrator();                                                             
    		void vibrate(int uid, String packageName, long milliseconds, IBinder token);       
    		void vibratePattern(int uid, String packageName, in long[] pattern, int repeat, IBinder token);
    		void cancelVibrate(IBinder token);                                                 
	}    

在frameworks/base/Android.mk中LOCAL_SRC_FILES添加编译支持


### 应用层支持

应用中通过 `import android.os.IVibratorService;`来导入IVibratorService接口，从而在接口中调用

	private IVibratorService mVibratorService = null;
	
	mVibratorService.vibrate(Process.myUid(), null, 2000, new Binder());
	mVibratorService.cancelVibrate(new Binder());


## 小结
上面的文章从振子的驱动分析，采用了自底向上的方法介绍了android软件的调用流程。虽然这是一个极小的驱动，但是可以看出Android这个软件栈的一些基本的软硬件交互的架构。



## Android日志分析

日常工作中会用到的这一部分的分析，先放到这儿，之后在进行整理。

1.android log
http://blog.csdn.net/luoshengyang/article/details/6581828
kernel log —printk
android log

        用户空间程序开发时LOG的使用。Android系统在用户空间中提供了轻量级的logger日志系统，它是在内核中实现的一种设备驱动，与用户空间的logcat工具配合使用能够方便地跟踪调试程序。在Android系统中，分别为C/C++ 和Java语言提供两种不同的logger访问接口。C/C++日志接口一般是在编写硬件抽象层模块或者编写JNI方法时使用，而Java接口一般是在应用层编写APP时使用。

如果要使用C/C++日志接口，只要定义自己的LOG_TAG宏和包含头文件system/core/include/cutils/log.h就可以了：
         #define LOG_TAG "MY LOG TAG"
         #include <cutils/log.h>
就可以了，例如使用LOGV：
         LOGV("This is the log printed by LOGV in android user space.");
         
如果要使用Java日志接口，只要在类中定义的LOG_TAG常量和引用android.util.Log就可以了：

        private static final String LOG_TAG = "MY_LOG_TAG";
        Log.i(LOG_TAG, "This is the log printed by Log.i in android user space.");

 要查看这些LOG的输出，可以配合logcat工具。如果是在Eclipse环境下运行模拟器，并且安装了Android插件，那么，很简单，直接在Eclipse就可以查看了
 
在Android系统中，提供了一个轻量级的日志系统，这个日志系统是以驱动程序的形式实现在内核空间的，而在用户空间分别提供了Java接口和C/C++接口来使用这个日志系统，取决于你编写的是Android应用程序还是系统组件。

Logger驱动程序主要由两个文件构成，分别是：

       kernel/common/drivers/staging/android/logger.h
       kernel/common/drivers/staging/android/logger.c 
       
每条日志记录的有效负载长度加上结构体logger_entry的长度不能超过4K个字节。
日志系统的读写问题，其实是一个生产者-消费者的问题，因此，需要互斥量来保护log的并发访问。
对于模块里面定义的对象，也没有用对引用计数技术。


## Android的1号进程和init.rc 

init.rc 由 Action 和Service组成。

每一个action的命令将被顺序执行，action 的格式如下：
	on <tringger>
	<command>
	<command>
	
trigger 是action 的触发条件，一共有这几种：
boot----init 启动时， /init.conf 加载后
<name> = <vaule> 当name的属性被设置为value时
device-added-<path>/device-removed-<path> 当一个device node被添加删除时
service-exited-<name> 当某个服务退出时

command 有如下几种：
exec <path> [argument] : fork 并execute一个路径path下面的程序，执行完毕后,init继续执行。尽量避免使用这个command ，可能导致Init 阻塞

export <name> <vaule> 把全局变量name的值设置为value。这个command执行完成后，所有进程都会继承这个变量

ifup <interface> 打开某个网卡

class_start  <serviceclass> 如果某一类service没有运行，启动他们
class_staop <serviceclass> 如果某一类service正在运行，停止他们
insmod <path> 安装path指定的模块
mount
setkey
setprop
setrlimit
start
stop
symlink
write

Service 由init启动和重新启动的程序组成
service <name> <pathname> <argument>*
 <option>
 <option>
 
option 是service得修饰符，用来告诉init怎样及何时启动service
disable
socket <type> <name> <perm> <user> <group>
user <username> 默认是root
group
capability
oneshot 服务运行一次，退出后不再重启
class 

默认情况下，通过init启动的程序都会把stdout和stderr定向到/dev/null. 调试时，可以通过logwrapper程序启动摸个程序、被启动的程序的stdout 和stderr就被定向到android的log系统中了，通过logact查看。
service skmd /system/bin/logwraper /sbin/skmd





[Yunzhi]:    http://yunzhi.github.io  "Yunzhi"
