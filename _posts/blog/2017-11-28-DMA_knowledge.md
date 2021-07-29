---
layout: post
title: DMA 知识
category: blog
description: 介绍一些DMA知识
---

# DMA 知识

在网上看到一个非常容易理解的DMA解释：

1.之前发送数据的方式
1) 数据放到串口数据寄存器里面
2) 等待一个字节发送完成
3) 重复第一二步。 
看到我们平时的方式我们就会有个想法，如果我们发送五百个字节，我们就需要让CPU在这里等待五百次。也就是在等待过程中我们是不能够去做其他事情的，只能够通过一个while循环去查看串口状态寄存器里面对应的发送完成标志是否为1。

2.现在我们大致描述一下DMA传输模式:
1) 数据写好放在内存  
2) 告诉DMA，我们发送多少数据
3) 干其他事情
4) 有想要发送数据了，先看看发送完了没有，没有发送完需要等待
5) 重复1)2)3)4)步骤

我们看到其实DMA也需要等，很多同学就说，这不是一样吗，我们都需要等待。
其实这个情况不一样，这就像一岁小孩吃饭和十岁小孩吃饭一个道理，一岁小孩吃饭需要我们大人一口一口的喂他，在这期间，我们不能够去干其他事情，只能够看到他，第一口吃完了才能够喂第二口。
但是十岁小孩子吃饭就不一样了，你把饭给弄好，然后给他，让他自己吃，这个期间我们就可以去洗衣服了，如果你还想让小孩子吃完这一碗饭再吃一碗，那么你等一会儿去看看就可以了。
言归正传，如果我们用普通方式去做的话，相当于软件去循环查询，但是DMA的话，相当于硬件来做，到底谁快谁慢，我们就不用说了。
其实说了那么多，就是想给大家普及一下DMA大致的工作方式，到底好不好，还是要看具体的项目，对时间的要求等等。下面我对我的代码进行一个讲解。

## 配置DMA

配置DMA需要想到哪些东西?所以我们应该有以下问题：
-1 我们使用的外设对应DMA 的StreamX channelX
-2 我们的外设地址是多少
-3 我们的内存地址是多少
-4 我们数据传输方向是什么，是内存->外设还是外设->内存
-5 内存的大小是多少
-6 外设地址是否需要增加，比如ADDR ADDR+1 ADDR+2
-7 内存地址是否需要增加，比如ADDR ADDR+1 ADDR+2
-8 内存和外设分别用的是byte,short,还是Int，也就是位宽是多少
-9 我们的DMA是一直运转还是只需要运转一次就停止
-10  当前DMA请求的等级高度
-11 是否使用FIFO
-12 如果使用FIFO,当FIFO数据存量为多少的时候将数据发送出去
-13 还有就是对内存和外设burst模式的配置


## Stm32 Uart DMA的使用

1. 配置 DMA

    /* DMA controller clock enable */
    __HAL_RCC_DMA1_CLK_ENABLE();

    /* DMA interrupt init */
    /* DMA1_Channel4_IRQn interrupt configuration */
    HAL_NVIC_SetPriority(DMA1_Channel4_IRQn, 0, 0);
    HAL_NVIC_EnableIRQ(DMA1_Channel4_IRQn);
    /* DMA1_Channel5_IRQn interrupt configuration */
    HAL_NVIC_SetPriority(DMA1_Channel5_IRQn, 0, 0);
    HAL_NVIC_EnableIRQ(DMA1_Channel5_IRQn);



2. 串口终端 IDLE 处理


    /* UART in mode IDLE -------------------------------------------------------*/
    tmp_flag = __HAL_UART_GET_FLAG(huart, UART_FLAG_IDLE);
    tmp_it_source = __HAL_UART_GET_IT_SOURCE(huart, UART_IT_IDLE);  

    if ((tmp_flag != RESET) && (tmp_it_source != RESET))
        HAL_UART_RecvIdleIRQHandler(huart);



3.  使能IDLE中端

      /* Enable the UART IDLE interrupt */
      __HAL_UART_ENABLE_IT(huart, UART_IT_IDLE);

4.  中端函数处理

  void Uart1RecvDataDmaNotify(UART_HandleTypeDef *huart)
  {	
      int16_t recvLen;

      recvLen = UART1_BUFF_LEN - __HAL_DMA_GET_COUNTER(huart->hdmarx);

      if (recvLen > 0)
      {
          memcpy(uart1TxBuf, uart1RxBuf, recvLen);
          HAL_UART_Transmit_DMA(huart, uart1TxBuf, recvLen);
      }
  }

  void HAL_UART_RecvIdleIRQHandler(UART_HandleTypeDef *huart)
  {
     if ( USART1 == huart->Instance ) {

          /* 关闭DMA */
          __HAL_DMA_DISABLE(huart->hdmarx);

          /* 清除IDLE中断标志位 */
          __HAL_UART_CLEAR_IDLEFLAG(huart);

          /* 处理接收数据 */		
          Uart1RecvDataDmaNotify(huart);

          /* 清除相关标志位 */
          __HAL_DMA_CLEAR_FLAG (huart->hdmarx, __HAL_DMA_GET_TC_FLAG_INDEX(huart->hdmarx));
          __HAL_DMA_CLEAR_FLAG (huart->hdmarx, __HAL_DMA_GET_HT_FLAG_INDEX(huart->hdmarx));
          __HAL_DMA_CLEAR_FLAG (huart->hdmarx, __HAL_DMA_GET_TE_FLAG_INDEX(huart->hdmarx));
          //__HAL_DMA_CLEAR_FLAG (huart->hdmarx, __HAL_DMA_GET_DME_FLAG_INDEX(huart->hdmarx));
          //__HAL_DMA_CLEAR_FLAG (huart->hdmarx, __HAL_DMA_GET_FE_FLAG_INDEX(huart->hdmarx));

          /* 重新设置数据长度 */
          huart->hdmarx->Instance->CNDTR = UART1_BUFF_LEN;

          /* 重新启动DMA接收 */
          __HAL_DMA_ENABLE(huart->hdmarx);

      }
      return;
  }


