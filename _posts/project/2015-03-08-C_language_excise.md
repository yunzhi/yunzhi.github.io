---
layout: post
title: 一些C语言小程序
category: project
description: 平时看书或者网上找到的一些c语言的练习程序，以备将来参考
---

## 高通平台下耳机驱动的初始化分析

从高通8x20平台以来，高通耳机驱动模块引入了MBHC这样一个耳机检测机制。这个机制中需要用户去设定一堆的值，然后通过初始化的调用写到相应的寄存器中。本来有一大堆的数据结构要去kmalloc/kzmalloc（对应应用程序中是malloc/zalloc），高通的软件工程师巧妙的借用了C语言的数据指针转化和内存定位来完成这个工作，当时读这一段代码时，非常受益。攒到下面，有机会也在自己的代码中借用下。

    /*
     * qcom headset data struct initialition
     */
    
    #include<stdio.h>
    #include<stdlib.h>
    
    typedef short s16;
    typedef unsigned short u16;
    typedef int s32;
    typedef unsigned int u32;
    typedef char s8;
    typedef unsigned char u8;
    typedef s16 INT16;
    typedef u16 UINT16;
    typedef s32 INT32;
    typedef u32 UINT32;
    typedef s8 CHAR8;
    typedef u8 UCHAR8;
    
    #define WCD9XXX_MBHC_DEF_BUTTONS 8
    #define WCD9XXX_MBHC_DEF_RLOADS  5
    
    enum wcd9xxx_micbias_num {
    	MBHC_MICBIAS_INVALID = -1,
    	MBHC_MICBIAS1,
    	MBHC_MICBIAS2,
    	MBHC_MICBIAS3,
    	MBHC_MICBIAS4,
    };
    
    enum wcd9xxx_mbhc_clk_freq {
    	TAIKO_MCLK_12P2MHZ = 0,
    	TAIKO_MCLK_9P6MHZ,
    	TAIKO_NUM_CLK_FREQS,
    };
    
    enum tapan_pid_current {
    	TAPAN_PID_MIC_2P5_UA,
    	TAPAN_PID_MIC_5_UA,
    	TAPAN_PID_MIC_10_UA,
    	TAPAN_PID_MIC_20_UA,
    };
      
    enum wcd9xxx_mbhc_btn_det_mem {
    	MBHC_BTN_DET_V_BTN_LOW,
    	MBHC_BTN_DET_V_BTN_HIGH,
    	MBHC_BTN_DET_N_READY,
    	MBHC_BTN_DET_N_CIC,
    	MBHC_BTN_DET_GAIN
    };
    
    struct wcd9xxx_mbhc_general_cfg {
    	u8 t_ldoh;
    	u8 t_bg_fast_settle;
    	u8 t_shutdown_plug_rem;
    	u8 mbhc_nsa;
    	u8 mbhc_navg;
    	u8 v_micbias_l;
    	u8 v_micbias;
    	u8 mbhc_reserved;
    	u16 settle_wait;
    	u16 t_micbias_rampup;
    	u16 t_micbias_rampdown;
    	u16 t_supply_bringup;
    } __attribute__((packed));
    
    struct wcd9xxx_mbhc_plug_detect_cfg {
    	u32 mic_current;
    	u32 hph_current;
    	u16 t_mic_pid;
    	u16 t_ins_complete;
    	u16 t_ins_retry;
    	u16 v_removal_delta;
    	u8 micbias_slow_ramp;
    	u8 reserved0;
    	u8 reserved1;
    	u8 reserved2;
    } __attribute__((packed));
    
    
    struct wcd9xxx_mbhc_plug_type_cfg {
    	u8 av_detect;
    	u8 mono_detect;
    	u8 num_ins_tries;
    	u8 reserved0;
    	s16 v_no_mic;
    	s16 v_av_min;
    	s16 v_av_max;
    	s16 v_hs_min;
    	s16 v_hs_max;
    	u16 reserved1;
    } __attribute__((packed));

    struct wcd9xxx_mbhc_btn_detect_cfg {
    	s8 c[8];
    	u8 nc;
    	u8 n_meas;
    	u8 mbhc_nsc;
    	u8 n_btn_meas;
    	u8 n_btn_con;
    	u8 num_btn;
    	u8 reserved0;
    	u8 reserved1;
    	u16 t_poll;
    	u16 t_bounce_wait;
    	u16 t_rel_timeout;
    	s16 v_btn_press_delta_sta;
    	s16 v_btn_press_delta_cic;
    	u16 t_btn0_timeout;
    	s16 _v_btn_low[0]; /* v_btn_low[num_btn] */
    	s16 _v_btn_high[0]; /* v_btn_high[num_btn] */
    	u8 _n_ready[TAIKO_NUM_CLK_FREQS];
    	u8 _n_cic[TAIKO_NUM_CLK_FREQS];
    	u8 _gain[TAIKO_NUM_CLK_FREQS];
    } __attribute__((packed));
    
    struct wcd9xxx_mbhc_imped_detect_cfg {
    	u8 _hs_imped_detect;
    	u8 _n_rload;
    	u8 _hph_keep_on;
    	u8 _repeat_rload_calc;
    	u16 _t_dac_ramp_time;
    	u16 _rhph_high;
    	u16 _rhph_low;
    	u16 _rload[0]; /* rload[n_rload] */
    	u16 _alpha[0]; /* alpha[n_rload] */
    	u16 _beta[3];
    } __attribute__((packed));
    
    #define WCD9XXX_MBHC_CAL_SIZE(buttons, rload) ( \
    	sizeof(enum wcd9xxx_micbias_num) + \
    	sizeof(struct wcd9xxx_mbhc_general_cfg) + \
    	sizeof(struct wcd9xxx_mbhc_plug_detect_cfg) + \
    	    ((sizeof(s16) + sizeof(s16)) * buttons) + \
    	sizeof(struct wcd9xxx_mbhc_plug_type_cfg) + \
    	sizeof(struct wcd9xxx_mbhc_btn_detect_cfg) + \
    	sizeof(struct wcd9xxx_mbhc_imped_detect_cfg) + \
    	    ((sizeof(u16) + sizeof(u16)) * rload) \
    	)
    
    #define WCD9XXX_MBHC_CAL_GENERAL_PTR(cali) ( \
    	    (struct wcd9xxx_mbhc_general_cfg *) cali)
    #define WCD9XXX_MBHC_CAL_PLUG_DET_PTR(cali) ( \
    	    (struct wcd9xxx_mbhc_plug_detect_cfg *) \
    	    &(WCD9XXX_MBHC_CAL_GENERAL_PTR(cali)[1]))
    #define WCD9XXX_MBHC_CAL_PLUG_TYPE_PTR(cali) ( \
    	    (struct wcd9xxx_mbhc_plug_type_cfg *) \
    	    &(WCD9XXX_MBHC_CAL_PLUG_DET_PTR(cali)[1]))
    #define WCD9XXX_MBHC_CAL_BTN_DET_PTR(cali) ( \
    	    (struct wcd9xxx_mbhc_btn_detect_cfg *) \
    	    &(WCD9XXX_MBHC_CAL_PLUG_TYPE_PTR(cali)[1]))
    #define WCD9XXX_MBHC_CAL_IMPED_DET_PTR(cali) ( \
    	    (struct wcd9xxx_mbhc_imped_detect_cfg *) \
    	    (((void *)&WCD9XXX_MBHC_CAL_BTN_DET_PTR(cali)[1]) + \
    	     (WCD9XXX_MBHC_CAL_BTN_DET_PTR(cali)->num_btn * \
    	      (sizeof(WCD9XXX_MBHC_CAL_BTN_DET_PTR(cali)->_v_btn_low[0]) + \
    	       sizeof(WCD9XXX_MBHC_CAL_BTN_DET_PTR(cali)->_v_btn_high[0])))) \
    	)

    void *wcd9xxx_mbhc_cal_btn_det_mp(
    			    const struct wcd9xxx_mbhc_btn_detect_cfg *btn_det,
    			    const enum wcd9xxx_mbhc_btn_det_mem mem)
    {
    	void *ret = &btn_det->_v_btn_low;
    
    	switch (mem) {
    	case MBHC_BTN_DET_GAIN:
    		ret += sizeof(btn_det->_n_cic);
    	case MBHC_BTN_DET_N_CIC:
    		ret += sizeof(btn_det->_n_ready);
    	case MBHC_BTN_DET_N_READY:
    		ret += sizeof(btn_det->_v_btn_high[0]) * btn_det->num_btn;
    	case MBHC_BTN_DET_V_BTN_HIGH:
    		ret += sizeof(btn_det->_v_btn_low[0]) * btn_det->num_btn;
    	case MBHC_BTN_DET_V_BTN_LOW:
    		/* do nothing */
    		break;
    	default:
    		ret = NULL;
    	}
    
    	return ret;
    }
    
    
    void* def_tapan_mbhc_cal(void)
    {
    	void *tapan_cal;
    	struct wcd9xxx_mbhc_btn_detect_cfg *btn_cfg;
    	u16 *btn_low, *btn_high;
    	u8 *n_ready, *n_cic, *gain;
    
    	tapan_cal = malloc(WCD9XXX_MBHC_CAL_SIZE(WCD9XXX_MBHC_DEF_BUTTONS,
    						WCD9XXX_MBHC_DEF_RLOADS));
    	if (!tapan_cal) {
    		printf("%s: out of memory\n", __func__);
    		return NULL;
    	}
    
    #define S(X, Y) ((WCD9XXX_MBHC_CAL_GENERAL_PTR(tapan_cal)->X) = (Y))
    #define P(X) (printf(#X " = %d\n",WCD9XXX_MBHC_CAL_GENERAL_PTR(tapan_cal)->X))
    	S(t_ldoh, 100);
    	S(t_bg_fast_settle, 100);
    	S(t_shutdown_plug_rem, 255);
    	S(mbhc_nsa, 2);
    	S(mbhc_navg, 128);
    
    	P(t_ldoh);
    	P(t_bg_fast_settle);
    	P(t_shutdown_plug_rem);
    	P(mbhc_nsa);
    	P(mbhc_navg);
    #undef P
    #undef S
    #define S(X, Y) ((WCD9XXX_MBHC_CAL_PLUG_DET_PTR(tapan_cal)->X) = (Y))
    #define P(X) (printf(#X " = %d\n",WCD9XXX_MBHC_CAL_PLUG_DET_PTR(tapan_cal)->X))
    	S(mic_current, TAPAN_PID_MIC_5_UA);
    	S(hph_current, TAPAN_PID_MIC_5_UA);
    	S(t_mic_pid, 100);
    	S(t_ins_complete, 250);
    	S(t_ins_retry, 200);
    
    	P(mic_current);
    	P(hph_current);
    	P(t_mic_pid);
    	P(t_ins_complete);
    	P(t_ins_retry);
    #undef P
    #undef S
    #define S(X, Y) ((WCD9XXX_MBHC_CAL_PLUG_TYPE_PTR(tapan_cal)->X) = (Y))
    #define P(X) (printf(#X " = %d\n",WCD9XXX_MBHC_CAL_PLUG_TYPE_PTR(tapan_cal)->X))
    	S(v_no_mic, 30);
    	S(v_hs_max, 2450);
    
    	P(v_no_mic);
      P(v_hs_max);
    #undef P
    #undef S
    #define S(X, Y) ((WCD9XXX_MBHC_CAL_BTN_DET_PTR(tapan_cal)->X) = (Y))
    #define P(X) (printf(#X " = %d\n", WCD9XXX_MBHC_CAL_BTN_DET_PTR(tapan_cal)->X))
    	S(c[0], 62);
    	S(c[1], 124);
    	S(nc, 1);
    	S(n_meas, 5);
    	S(mbhc_nsc, 10);
    	S(n_btn_meas, 1);
    	S(n_btn_con, 2);
    	S(num_btn, WCD9XXX_MBHC_DEF_BUTTONS);
    	S(v_btn_press_delta_sta, 100);
    	S(v_btn_press_delta_cic, 50);
    
    	P(c[0]);
    	P(c[1]);
    	P(nc);
    	P(n_meas);
    	P(mbhc_nsc);
    	P(n_btn_meas);
    	P(n_btn_con);
    	P(num_btn);
    	P(v_btn_press_delta_sta);
    	P(v_btn_press_delta_cic);
    #undef P
    #undef S
    	btn_cfg = WCD9XXX_MBHC_CAL_BTN_DET_PTR(tapan_cal);
    	btn_low = wcd9xxx_mbhc_cal_btn_det_mp(btn_cfg, MBHC_BTN_DET_V_BTN_LOW);
    	btn_high = wcd9xxx_mbhc_cal_btn_det_mp(btn_cfg,
    					       MBHC_BTN_DET_V_BTN_HIGH);
    	btn_low[0] = -50;
    	btn_high[0] = 20;
    	btn_low[1] = 21;
    	btn_high[1] = 61;
    	btn_low[2] = 62;
    	btn_high[2] = 104;
    	btn_low[3] = 105;
    	btn_high[3] = 148;
    	btn_low[4] = 149;
    	btn_high[4] = 189;
    	btn_low[5] = 190;
    	btn_high[5] = 228;
    	btn_low[6] = 229;
    	btn_high[6] = 269;
    	btn_low[7] = 270;
    	btn_high[7] = 500;
    	n_ready = wcd9xxx_mbhc_cal_btn_det_mp(btn_cfg, MBHC_BTN_DET_N_READY);
    	n_ready[0] = 80;
    	n_ready[1] = 12;
    	n_cic = wcd9xxx_mbhc_cal_btn_det_mp(btn_cfg, MBHC_BTN_DET_N_CIC);
    	n_cic[0] = 60;
    	n_cic[1] = 47;
    	gain = wcd9xxx_mbhc_cal_btn_det_mp(btn_cfg, MBHC_BTN_DET_GAIN);
    	gain[0] = 11;
    	gain[1] = 14;
    
    	return tapan_cal;
    }
    
    
    int main(int argc, char* argv[])
    {
    	def_tapan_mbhc_cal();
    	return 0;
    }

## 动态数组
1.这个程序介绍动态数据的用法。接受用户的输入数据，升序排列后打印。

    #include<stdio.h>
    #include<stdlib.h>
    
    int compare_intergers(void const *a,void const *b)
    {
      register int const *pa = a;
      register int const *pb = b;
    
      return *pa > *pb ? 1 : ( *pa < *pb ? -1 : 0 );
    }
    
    int main()
    {
      int *array;
      int n_values;
      int i;
    
      printf("How many values are there?\n");
      if( (scanf("%d",&n_values) != 1) || n_values < 0){
        printf("Iligeal number of values.\n");
        exit(EXIT_FAILURE);
      }
    
      array = malloc(sizeof(n_values) * sizeof(int));
      if(array == NULL){
        printf("Failed to allloc the memory!\n");
        exit(EXIT_FAILURE);
      }
    
      for(i=0;i< n_values;i++){
        printf("? ");
        if(scanf("%d",array + i) != 1){
          printf("Error reading values %d\n",i);
          free(array);
          exit(EXIT_FAILURE);
        }
      }
    
      qsort(array,n_values,sizeof(int),compare_intergers);
    
      for(i=0;i<n_values;i++){
        printf("%d\n",array[i]);
      }
    
      free(array);
      return EXIT_SUCCESS;
    }

