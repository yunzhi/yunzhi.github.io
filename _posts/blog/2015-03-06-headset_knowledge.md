---
layout: post
title: Android 耳机驱动知识
description: 描述耳机识别的原理和android驱动
category: blog
---

工作以后接手的第一个驱动就是android平台下耳机的插拔检测和按键检测。这部分涉及的硬件知识比较简单，但是软件上对中断的处理，软件检测的鲁棒性，都有比较高的要求，涉及到驱动开发中经常使用的中断申请，工作队列，tasklet，竟态和同步，linux input子系统，android 键值映射等知识。

## 耳机接口知识介绍

1.耳机的通用接口为一个裸露的圆柱体，从头端到线侧的直径依次增大，并通过橡胶环进行绝缘而设计，这样方便无论从哪个角度都可以插入。在耳机座上，通过弹片和耳机头的金属环触而形成电路导通。

2.市面上流通的耳机从接口大小上分3.5mm和2.5mm两种，主要适配不同尺寸的插口，比较常见的是3.5mm。从接口电气特性上分三段式和四段式，四段式在三段耳机的基础上增加了mic端——耳机内部的一个声电转化装置。

+ 三段式耳机接口从头部到线一侧的定义式左声道，右声道，GND。
+ 四段式耳机分美标（CTIA）和欧标（国内要求为欧标，OMTP-Open Mobile Terminal Platform开放移动终端平台），主要区别在于耳机线侧最后两端的定义，美标为左声道（L），右声道（R），GND（G），MIC（M），括号内为缩写，下面为清晰主要采用缩写，国标为L，R，M，G。

![三段和四段耳机的简易电路示例图](/images/headset/simple_circuit.jpg)

3.从耳机识别的角度来讲，耳机上的电声转化装置（左声道听音器和右声道听音器）可以认为是一个16欧或者32欧的电阻，电阻值根据耳机厂商的设计而不同，一般的标准为16欧或者32欧，但有些比较好的耳机这个内阻值比较大；mic端可以认为是一个大电阻（通常为1k欧）和一个开关（多按键耳机可以认为好多个开关串上不同组值得电阻）。

### 4端3.5mm耳机的接口分类
#### 耳机标准-美标 (CTIA，通常称为美标)

从插入端到线分别是： 左声道，右声道，GND，MIC。耳机上德绝缘橡胶环一般是白色的
代表品牌：iphone，MOTO，小米，魅族，索尼

![ctia headset](/images/headset/ctia_headset_interface.jpg)

#### 耳机接口标准 (OMTP，通常称为欧标)

从插入端到线分别是： 左声道，右声道，MIC，GND。耳机上德绝缘橡胶环一般是黑色的
代表品牌：诺基亚，三星，HTC

![omtp headset](/images/headset/omtp_headset_interface.jpg)

## 耳机座接口介绍
相应的，耳机座也分为支持欧标设计的耳机座和支持美标设计的耳机座。另外，从耳机座左声道的检测方式来又可以分为 “Nomally-closed type”(常闭型) 和 “Normally-open type”（常开型） 两种。其简易设计如下图

![headset jack type](/images/headset/headset_jack_type.jpg)

图中所示的耳机座为美标的。

* 在常闭型中，不接耳机时，耳机座左声道和检测端HS-DET接触，插入耳机时，HS-DET与HPH-L不导通。

* 在常开型中，不接耳机时，耳机座左声道和检测端HS-DET不接触，插入耳机时，HS-DET与HPH-L导通。

## 耳机的硬件检测原理

下图是一个高通平台下耳机座设计的原理图

![headset detect priciple](/images/headset/headset_circuit_design.jpg)

可以看到，该耳机座为常开型，采用了左声道检测的机制——CDC_HS_DET为插入耳机触发硬件中断的的管脚。当没有插入耳机时，由于CDC_HS_DET悬空，而该网络对应的平台端的gpio（输入状态）口为低电平。当插入耳机后，由于耳机左声道内部相当于1个16欧的电阻和GND相接，于是有如下的模拟图：

![headset circuit](/images/headset/headset_principle.jpg)
正常情况下，CDC_HPH_L会有一点电压存在，通过电阻的分压，于是CDC_HS_DET接收到了高电平，引起了软件中断。软件上通过debounce后，检测到持续的高电平，于是认为有耳机插入。这时候需要判断，插入的是三段还是四段。平台上打开mic_bias，当插入的是三段耳机时，MIC_IN2_P端口被拉低(忽略原理图R3501处的NC，应该是个笔误)，于是判断为三段耳机。若为四段耳机，MIC_IN2_P的电平接近于MIC_BIAS，软件判断该处的直流电压之后设置识别了四段耳机。当按键按下时，MIC_IN2_P的电压发生变化，触发了系统中断，之后软件通过采样该处的电压值判断按键阻值而确定按下了哪一个按键。

一般的，一键耳机按下后电阻值在10欧以下，三键带音量加减的耳机上键的电阻范围在60欧到100欧之间，中键在10欧以下，下键在120欧~200欧之间。

## 软件检测实现

我接触过四个平台的耳机驱动，mtk、高通、Nividia和spreadtrum。除了高通将检测耳机插拔的事件也申请为input设备外，，其他平台都注册为switch/h2w设备。mtk平台的耳机驱动称为ACCDET+EINT的模式，高通的机制叫做MBHC，都是一套看起来特别麻烦的机制。而展讯的code将耳机驱动作为misc下得一个设备驱动来用，很体现linux “write code do one thing and do it well”的哲理。下面来看看展讯的耳机驱动。

### 展讯平台的耳机驱动

headset.h

    /*
     * Copyright (C) 2012 Spreadtrum Communications Inc.
     *
     * This software is licensed under the terms of the GNU General Public
     * License version 2, as published by the Free Software Foundation, and
     * may be copied, distributed, and modified under those terms.
     *
     * This program is distributed in the hope that it will be useful,
     * but WITHOUT ANY WARRANTY; without even the implied warranty of
     * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
     * GNU General Public License for more details.
     */
    
    #ifndef __HEADSET_H__
    #define __HEADSET_H__
    #include <linux/switch.h>
    #include <linux/input.h>
    #include <linux/platform_device.h>
    enum {
    	BIT_HEADSET_OUT = 0,
    	BIT_HEADSET_MIC = (1 << 0),
    	BIT_HEADSET_NO_MIC = (1 << 1),
    };
    
    enum {
    	HEADSET_BUTTON_DOWN_INVALID = -1,
    	HEADSET_BUTTON_DOWN_SHORT,
    	HEADSET_BUTTON_DOWN_LONG,
    };
    
    struct _headset_gpio {
    	int active_low;
    	int gpio;
    	int irq;
    	unsigned int irq_type_active;
    	unsigned int irq_type_inactive;
    	int debounce;
    	int debounce_sw;
    	int holded;
    	int active;
    	int irq_enabled;
    	const char *desc;
    	struct _headset *parent;
    	unsigned int timeout_ms;
    	struct hrtimer timer;
    	enum hrtimer_restart (*callback)(int active, struct _headset_gpio *hgp);
    };
    
    struct _headset_keycap {
    	unsigned int type;
    	unsigned int key;
    };
    
    struct _headset_button {
    	struct _headset_keycap cap[15];
    	unsigned int (*headset_get_button_code_board_method)(int v);
    	unsigned int (*headset_map_code2push_code_board_method)(unsigned int code, int push_type);
    };
    
    struct _headset {
    	struct switch_dev sdev;
    	struct input_dev *input;
    	struct _headset_gpio detect;
    	struct _headset_gpio button;
    	int headphone;
    	int type;
    	struct work_struct switch_work;
    	struct workqueue_struct * switch_workqueue;
    };
    
    #ifndef ARRY_SIZE
    #define ARRY_SIZE(A) (sizeof(A)/sizeof(A[0]))
    #endif
    
    #endif


headset.c

    /*
     * Copyright (C) 2012 Spreadtrum Communications Inc.
     *
     * This software is licensed under the terms of the GNU General Public
     * License version 2, as published by the Free Software Foundation, and
     * may be copied, distributed, and modified under those terms.
     *
     * This program is distributed in the hope that it will be useful,
     * but WITHOUT ANY WARRANTY; without even the implied warranty of
     * MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
     * GNU General Public License for more details.
     */
    
    #include <linux/interrupt.h>
    #include <linux/irq.h>
    #include <linux/delay.h>
    #include <mach/gpio.h>
    #include <linux/headset.h>
    #include <mach/board.h>
    
    #ifndef HEADSET_DETECT_GPIO
    #define HEADSET_DETECT_GPIO 165
    #endif
    #ifndef HEADSET_BUTTON_GPIO
    #define HEADSET_BUTTON_GPIO 164
    #endif
    
    #ifndef HEADSET_DETECT_GPIO_ACTIVE_LOW
    #define HEADSET_DETECT_GPIO_ACTIVE_LOW 1
    #endif
    #ifndef HEADSET_BUTTON_GPIO_ACTIVE_LOW
    #define HEADSET_BUTTON_GPIO_ACTIVE_LOW 0
    #endif
    
    #ifndef HEADSET_DETECT_GPIO_DEBOUNCE_SW
    #define HEADSET_DETECT_GPIO_DEBOUNCE_SW 1000
    #endif
    #ifndef HEADSET_BUTTON_GPIO_DEBOUNCE_SW
    #define HEADSET_BUTTON_GPIO_DEBOUNCE_SW 100
    #endif
    
    static enum hrtimer_restart report_headset_button_status(int active, struct _headset_gpio *hgp);
    static enum hrtimer_restart report_headset_detect_status(int active, struct _headset_gpio *hgp);
    static struct _headset headset = {
    	.sdev = {
    		.name = "h2w",
    	},
    	.detect = {
    		.desc = "headset detect",
    		.active_low = HEADSET_DETECT_GPIO_ACTIVE_LOW,
    		.gpio = HEADSET_DETECT_GPIO,
    		.debounce = 0,
    		.debounce_sw = HEADSET_DETECT_GPIO_DEBOUNCE_SW,
    		.irq_enabled = 1,
    		.callback = report_headset_detect_status,
    	},
    	.button = {
    		.desc = "headset button",
    		.active_low = HEADSET_BUTTON_GPIO_ACTIVE_LOW,
    		.gpio = HEADSET_BUTTON_GPIO,
    		.debounce = 0,
    		.debounce_sw = HEADSET_BUTTON_GPIO_DEBOUNCE_SW,
    		.irq_enabled = 1,
    		.callback = report_headset_button_status,
    		.timeout_ms = 800, /* 800ms for long button down */
    	},
    };
    
    
    #ifndef headset_gpio_init
    #define headset_gpio_init(gpio, desc) \
    	do { \
    		gpio_request(gpio, desc); \
    		gpio_direction_input(gpio); \
    	} while (0)
    #endif
    
    #ifndef headset_gpio_free
    #define headset_gpio_free(gpio) \
    	gpio_free(gpio)
    #endif
    
    #ifndef headset_gpio2irq_free
    #define headset_gpio2irq_free(irq, args) { }
    #endif
    
    #ifndef headset_gpio2irq
    #define headset_gpio2irq(gpio) \
    	gpio_to_irq(gpio)
    #endif
    
    #ifndef headset_gpio_set_irq_type
    #define headset_gpio_set_irq_type(irq, type) \
    	irq_set_irq_type(irq, type)
    #endif
    
    #ifndef headset_gpio_get_value
    #define headset_gpio_get_value(gpio) \
    	gpio_get_value(gpio)
    #endif
    
    #ifndef headset_gpio_debounce
    #define headset_gpio_debounce(gpio, ms) \
    	gpio_set_debounce(gpio, ms)
    #endif
    
    #ifndef headset_hook_detect
    #define headset_hook_detect(status) { }
    #endif
    
    #define HEADSET_DEBOUNCE_ROUND_UP(dw) \
    	dw = (((dw ? dw : 1) + HEADSET_GPIO_DEBOUNCE_SW_SAMPLE_PERIOD - 1) / \
    		HEADSET_GPIO_DEBOUNCE_SW_SAMPLE_PERIOD) * HEADSET_GPIO_DEBOUNCE_SW_SAMPLE_PERIOD;
    
    static struct _headset_keycap headset_key_capability[20] = {
    	{ EV_KEY, KEY_MEDIA },
    	{ EV_KEY, KEY_END },
    	{ EV_KEY, KEY_RESERVED },
    };
    
    static unsigned int (*headset_get_button_code_board_method)(int v);
    static unsigned int (*headset_map_code2push_code_board_method)(unsigned int code, int push_type);
    static __devinit int headset_button_probe(struct platform_device *pdev)
    {
    	struct _headset_button *headset_button = platform_get_drvdata(pdev);
    	headset_get_button_code_board_method = headset_button->headset_get_button_code_board_method;
    	headset_map_code2push_code_board_method = headset_button->headset_map_code2push_code_board_method;
    	memcpy(headset_key_capability, headset_button->cap, sizeof headset_button->cap);
    	return 0;
    }
    
    static struct platform_driver headset_button_driver = {
    	.driver = {
    		.name = "headset-button",
    		.owner = THIS_MODULE,
    	},
    	.probe = headset_button_probe,
    };
    
    static unsigned int headset_get_button_code(int v)
    {
    	unsigned int code;
    	if (headset_get_button_code_board_method)
    		code = headset_get_button_code_board_method(v);
    	else
    		code = KEY_MEDIA;
    	return code;
    }
    
    static unsigned int headset_map_code2key_type(unsigned int code)
    {
    	unsigned int key_type = EV_KEY;
    	int i;
    	for(i = 0; headset_key_capability[i].key != KEY_RESERVED &&
    		headset_key_capability[i].key != code && i < ARRY_SIZE(headset_key_capability); i++);
    	if (i < ARRY_SIZE(headset_key_capability) &&
    		headset_key_capability[i].key == code)
    		key_type = headset_key_capability[i].type;
    	else
    		pr_err("headset not find code [0x%x]'s maping type\n", code);
    	return key_type;
    }
    
    static unsigned int headset_map_code2push_code(unsigned int code, int push_type)
    {
    	if (headset_map_code2push_code_board_method)
    		return headset_map_code2push_code_board_method(code, push_type);
    
    	switch (push_type) {
    	case HEADSET_BUTTON_DOWN_SHORT:
    		code = KEY_MEDIA;
    		break;
    	case HEADSET_BUTTON_DOWN_LONG:
    		code = KEY_END;
    		break;
    	}
    
    	return code;
    }
    
    /*tangyao modified on 2013-01-25*/
    static void headset_gpio_irq_enable(int enable, struct _headset_gpio *hgp);
    #define HEADSET_GPIO_DEBOUNCE_SW_SAMPLE_PERIOD	50 /* 10 */
    static enum hrtimer_restart report_headset_button_status(int active, struct _headset_gpio *hgp)
    {
    	enum hrtimer_restart restart;
    	static int step = 0;
    
    	if (active < 0) {
    		step = 0;
    		return HRTIMER_NORESTART;
    	}
    	if (active) {
    		restart = HRTIMER_RESTART;
    		if (++step > 3)
    			step = 0;
    		switch (step) {
    		case 1:
    			/*short press report*/
    			input_event(hgp->parent->input,EV_KEY,KEY_MEDIA, 1);
    			input_sync(hgp->parent->input);
    			break;
    		case 2:
    			/*long press report,first report short press release,then long press start*/
    			input_event(hgp->parent->input,EV_KEY,KEY_MEDIA, 0);
    			input_sync(hgp->parent->input);
    			input_event(hgp->parent->input,EV_KEY,KEY_END, 1);
    			input_sync(hgp->parent->input);
    			break;
    		default:
    			pr_info("Are you press too long? step = %d\n",step);
    		}
    	} else {
    		restart = HRTIMER_NORESTART;
    		if (step == 1){
    			/*short press release report*/
    			input_event(hgp->parent->input,EV_KEY,KEY_MEDIA, 0);
    			input_sync(hgp->parent->input);
    		}else{
    			/*long press release report*/
    			input_event(hgp->parent->input,EV_KEY,KEY_END, 0);
    			input_sync(hgp->parent->input);
    		}
    		step = 0;
    	}
    	
    	return restart;
    }
    
    static enum hrtimer_restart report_headset_detect_status(int active, struct _headset_gpio *hgp)
    {
    	struct _headset * ht = hgp->parent;
    	if (active) {
    		headset_hook_detect(1);
    		ht->headphone = 0;
    		/*headphone support,tangyao modified on 2012-01-25*/
    		ht->headphone = ht->button.active_low ^ headset_gpio_get_value(ht->button.gpio); 
    		if (ht->headphone) {
    			ht->type = BIT_HEADSET_NO_MIC;
    			queue_work(ht->switch_workqueue, &ht->switch_work);
    			pr_info("headphone plug in\n");
    		} else {
    			ht->type = BIT_HEADSET_MIC;
    			queue_work(ht->switch_workqueue, &ht->switch_work);
    			pr_info("headset plug in\n");
    			headset_gpio_set_irq_type(ht->button.irq, ht->button.irq_type_active);
    			headset_gpio_irq_enable(1, &ht->button);
    		}
    	} else {
    		headset_gpio_irq_enable(0, &ht->button);
    		ht->button.callback(-1, &ht->button);
    		headset_hook_detect(0);
    		if (ht->headphone)
    			pr_info("headphone plug out\n");
    		else
    			pr_info("headset plug out\n");
    		ht->type = BIT_HEADSET_OUT;
    		queue_work(ht->switch_workqueue, &ht->switch_work);
    	}
    	/* use below code only when gpio irq misses state, because of the dithering */
    	headset_gpio_set_irq_type(hgp->irq, active ? hgp->irq_type_inactive : hgp->irq_type_active);
    	return HRTIMER_NORESTART;
    }
    
    static enum hrtimer_restart headset_gpio_timer_func(struct hrtimer *timer)
    {
    	enum hrtimer_restart restart = HRTIMER_RESTART;
    	struct _headset_gpio *hgp =
    		container_of(timer, struct _headset_gpio, timer);
    	int active = hgp->active_low ^ headset_gpio_get_value(hgp->gpio); /* hgp->active */
    	int green_ch = (!active && &hgp->parent->detect == hgp);
    	if (active != hgp->active) {
    		pr_info("The value %s mismatch [%d:%d] at %dms!\n",
    				hgp->desc, active, hgp->active, hgp->holded);
    		hgp->holded = 0;
    	}
    	pr_debug("%s : %s %s green_ch[%d], holed=%d, debounce_sw=%d\n", __func__,
    			hgp->desc, active ? "active" : "inactive", green_ch, hgp->holded, hgp->debounce_sw);
    	hgp->holded += HEADSET_GPIO_DEBOUNCE_SW_SAMPLE_PERIOD;
    	if (hgp->holded >= hgp->debounce_sw || green_ch) {
    		if (hgp->holded == hgp->debounce_sw || \
    			hgp->holded == hgp->timeout_ms || \
    			green_ch) {
    			pr_debug("call headset gpio handler\n");
    			restart = hgp->callback(active, hgp);
    		} else
    			pr_debug("gpio <%d> has kept active for %d ms\n", hgp->gpio, hgp->holded);
    	}
    	if (restart == HRTIMER_RESTART)
    		hrtimer_forward_now(timer,
    			ktime_set(HEADSET_GPIO_DEBOUNCE_SW_SAMPLE_PERIOD / 1000,
    					(HEADSET_GPIO_DEBOUNCE_SW_SAMPLE_PERIOD % 1000) * 1000000)); /* repeat timer */
    	return restart;
    }
    
    static irqreturn_t headset_gpio_irq_handler(int irq, void *dev)
    {
    	struct _headset_gpio *hgp = dev;
    	hrtimer_cancel(&hgp->timer);
    	hgp->active = hgp->active_low ^ headset_gpio_get_value(hgp->gpio);
    	headset_gpio_set_irq_type(hgp->irq, hgp->active ? hgp->irq_type_inactive : hgp->irq_type_active);
    	pr_debug("%s : %s %s\n", __func__, hgp->desc, hgp->active ? "active" : "inactive");
    	hgp->holded = 0;
    	hrtimer_start(&hgp->timer,
    			ktime_set(HEADSET_GPIO_DEBOUNCE_SW_SAMPLE_PERIOD / 1000,
    				(HEADSET_GPIO_DEBOUNCE_SW_SAMPLE_PERIOD % 1000) * 1000000),
    			HRTIMER_MODE_REL);
    	return IRQ_HANDLED;
    }
    
    static void headset_gpio_irq_enable(int enable, struct _headset_gpio *hgp)
    {
    	int action = 0;
    	if (enable) {
    		if (!hgp->irq_enabled) {
    			hrtimer_cancel(&hgp->timer);
    			hgp->irq_enabled = 1;
    			action = 1;
    			hgp->holded = 0;
    			enable_irq(hgp->irq);
    		}
    	} else {
    		if (hgp->irq_enabled) {
    			disable_irq(hgp->irq);
    			hrtimer_cancel(&hgp->timer);
    			hgp->irq_enabled = 0;
    			action = 1;
    			hgp->holded = 0;
    		}
    	}
    	pr_info("%s [ irq=%d ] --- %saction %s\n", __func__, hgp->irq_enabled, action ? "do " : "no ", hgp->desc);
    }
    
    static void headset_switch_state(struct work_struct *work)
    {
    	struct _headset *ht;
    	int type;
    
    	ht = container_of(work, struct _headset, switch_work);
    	type = ht->type;
    	switch_set_state(&headset.sdev, type);
    	pr_info("set headset state to %d\n", type);
    }
    
    static int __init headset_init(void)
    {
    	int ret, i;
    	struct _headset *ht = &headset;
    	ret = switch_dev_register(&ht->sdev);
    	if (ret < 0) {
    		pr_err("switch_dev_register failed!\n");
    		return ret;
    	}
    	platform_driver_register(&headset_button_driver);
    	ht->input = input_allocate_device();
    	if (ht->input == NULL) {
    		pr_err("switch_dev_register failed!\n");
    		goto _switch_dev_register;
    	}
    	ht->input->name = "headset-keyboard";
    	ht->input->id.bustype = BUS_HOST;
    	ht->input->id.vendor = 0x0001;
    	ht->input->id.product = 0x0001;
    	ht->input->id.version = 0x0100;
    
    	for(i = 0; headset_key_capability[i].key != KEY_RESERVED; i++) {
    		__set_bit(headset_key_capability[i].type, ht->input->evbit);
    		input_set_capability(ht->input, headset_key_capability[i].type, headset_key_capability[i].key);
    	}
    
    	if (input_register_device(ht->input))
    		goto _switch_dev_register;
    
    	headset_gpio_init(ht->detect.gpio, ht->detect.desc);
    	headset_gpio_init(ht->button.gpio, ht->button.desc);
    
    	headset_gpio_debounce(ht->detect.gpio, ht->detect.debounce * 1000);
    	headset_gpio_debounce(ht->button.gpio, ht->button.debounce * 1000);
    
    	hrtimer_init(&ht->button.timer, CLOCK_MONOTONIC, HRTIMER_MODE_REL);
    	ht->button.timer.function = headset_gpio_timer_func;
    	HEADSET_DEBOUNCE_ROUND_UP(ht->button.debounce_sw);
    	HEADSET_DEBOUNCE_ROUND_UP(ht->button.timeout_ms);
    	ht->button.parent = ht;
    	ht->button.irq = headset_gpio2irq(ht->button.gpio);
    	ht->button.irq_type_active = ht->button.active_low ? IRQF_TRIGGER_LOW : IRQF_TRIGGER_HIGH;
    	ht->button.irq_type_inactive = ht->button.active_low ? IRQF_TRIGGER_HIGH : IRQF_TRIGGER_LOW;
    	ret = request_irq(ht->button.irq, headset_gpio_irq_handler,
    					ht->button.irq_type_active, ht->button.desc, &ht->button);
    	if (ret) {
    		pr_err("request_irq gpio %d's irq failed!\n", ht->button.gpio);
    		goto _gpio_request;
    	}
    	headset_gpio_irq_enable(0, &ht->button);
    
    	hrtimer_init(&ht->detect.timer, CLOCK_MONOTONIC, HRTIMER_MODE_REL);
    	ht->detect.timer.function = headset_gpio_timer_func;
    	HEADSET_DEBOUNCE_ROUND_UP(ht->detect.debounce_sw);
    	ht->detect.parent = ht;
    	ht->detect.irq = headset_gpio2irq(ht->detect.gpio);
    	ht->detect.irq_type_active = ht->detect.active_low ? IRQF_TRIGGER_LOW : IRQF_TRIGGER_HIGH;
    	ht->detect.irq_type_inactive = ht->detect.active_low ? IRQF_TRIGGER_HIGH : IRQF_TRIGGER_LOW;
    	ret = request_irq(ht->detect.irq, headset_gpio_irq_handler,
    					ht->detect.irq_type_active, ht->detect.desc, &ht->detect);
    	if (ret) {
    		pr_err("request_irq gpio %d's irq failed!\n", ht->detect.gpio);
    		goto _headset_button_gpio_irq_handler;
    	}
    
    	INIT_WORK(&ht->switch_work, headset_switch_state);
    	ht->switch_workqueue = create_singlethread_workqueue("headset_switch");
    
    	if (ht->switch_workqueue == NULL) {
    		pr_err("can't create headset switch workqueue\n");
    		ret = -ENOMEM;
    		goto _headset_workqueue;
    	}
    
    	return 0;
    _headset_workqueue:
    	destroy_workqueue(ht->switch_workqueue);
    _headset_button_gpio_irq_handler:
    	free_irq(ht->button.irq, &ht->button);
    	headset_gpio2irq_free(ht->button.irq, &ht->button);
    _gpio_request:
    	headset_gpio_free(ht->detect.gpio);
    	headset_gpio_free(ht->button.gpio);
    	input_free_device(ht->input);
    _switch_dev_register:
    	platform_driver_unregister(&headset_button_driver);
    	switch_dev_unregister(&ht->sdev);
    	return ret;
    }
    module_init(headset_init);
    
    static void __exit headset_exit(void)
    {
    	struct _headset *ht = &headset;
    	destroy_workqueue(ht->switch_workqueue);
    	headset_gpio_irq_enable(0, &ht->button);
    	headset_gpio_irq_enable(0, &ht->detect);
    	free_irq(ht->detect.irq, &ht->detect);
    	headset_gpio2irq_free(ht->detect.irq, &ht->detect);
    	free_irq(ht->button.irq, &ht->button);
    	headset_gpio2irq_free(ht->button.irq, &ht->button);
    	headset_gpio_free(ht->detect.gpio);
    	headset_gpio_free(ht->button.gpio);
    	input_free_device(ht->input);
    	platform_driver_unregister(&headset_button_driver);
    	switch_dev_unregister(&ht->sdev);
    }
    module_exit(headset_exit);
    
    MODULE_DESCRIPTION("headset & button detect driver");
    MODULE_AUTHOR("Luther Ge <luther.ge@spreadtrum.com>");
    MODULE_LICENSE("GPL");

分析：

1. 360-373行，注册input设备。可以看到，一个input设备的注册方法，首先使用input_allocate_device为设备申请相关数据结构，然后初始化该结构的相关成员，如input->name，input->id.vendor, input->id.product, input->id.version(这四个字符串决定了键盘映射文件的名称)，然后调用__set_bit设置该input设备支持的事件类型，及调用input_set_capability设置支持的按键值，最后调用input_register_device将输出化完成的数据结构注册到input子系统中。一般的，我们不用去实现他的handle函数，evdev.c就可以完成该目的。

2. 374-378行，初始化耳机和按键检测时用到的gpio口
3. 380-394行，申请耳机按键检测的中断处理函数，初始化中断下半段的处理机制。可以看到这里使用了hr_timer这样一个内核中的高精度定时器来实现
4. 396-417行，申请耳机插拔检测的中断处理函数，初始化中断下半段的处理机制。可以看到这里使用了work_queue这样一个机制来实现。

可以看到，耳机在中断下半段处理时采用了内核定时器timer来实现。另外，耳机插拔的检测使用了h2w这个class，hook按键上报则采用了input子系统。


### headset插拔识别的框架代码分析和hook按键的处理

#### headset插拔识别的框架代码分析

涉及的相关文件如下
hardware/libhardware_legacy/uevent.c
frameworks/base/core/jni/android_os_UEventObserver.cpp
frameworks/base/core/java/android/os/UEventObserver.java
frameworks/services/java/com/android/server/SystemServer.java
frameworks/base/services/java/com/android/server/WiredAccessoryManager.java 

流程待分析

#### hook按键的处理

hook按键通过input子系统上报给Android，在Android手机/system/usr/keylayout/目录下保存着键值映射配置文件。

一般的，耳机按键对应的按键映射：
key 231 CALL 
key 122 ENDCALL WAKE 
key 166 MEDIA_STOP 
key 163 HEADSETHOOK 
key 164 MEDIA_PLAY_PAUSE 
key 165 MEDIA_PREVIOUS 
key 114 VOLUME_DOWN 
key 115 VOLUME_UP

 这个按键配置文件第三列的字符串在/frameworks/base/include/androidfw/KeycodeLabels.h (android 4.0), frameworks/native/include/input/KeycodeLabels.h(android 4.4), 被定义成：

	static const KeycodeLabel KEYCODES[] = {
	...
	{ "CALL", 5 },
	{ "ENDCALL", 6 },
	...
	{ "MEDIA_PLAY_PAUSE", 85 },
	{ "MEDIA_STOP", 86 },
	...
	}
最终/frameworks/base/core/java/android/view/KeyEvent.java会把这个数字定义成这样的常量：

	public class KeyEvent extends InputEvent implements Parcelable {
	...
	public static final int KEYCODE_CALL = 5;
	/** Key code constant: End Call key. */
	public static final int KEYCODE_ENDCALL = 6;
	...
	/** Key code constant: Play/Pause media key. */
	public static final int KEYCODE_MEDIA_PLAY_PAUSE= 85;
	/** Key code constant: Stop media key. */
	public static final int KEYCODE_MEDIA_STOP = 86;
	...
	}


## 总结
手机耳机是手机非常重要的功能之一，耳机的插拔检测和按键检测和相对比较麻烦，日常工作中也容易出现一些新的需求，如新的设备需要通过耳机接口被接入到手机中。因此，研究其驱动和应用层的实现还是很有必要的。





[Yunzhi]:    http://yunzhi.github.io  "Yunzhi"
