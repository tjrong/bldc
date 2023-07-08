1、通信协议代码分析

各通信接口调用初始化函数设置处理与回应函数指针，具体实现在实现在packet.c中

void packet_init(void (*s_func)(unsigned char *data, unsigned int len),
		void (*p_func)(unsigned char *data, unsigned int len), PACKET_STATE_t *state) {
	memset(state, 0, sizeof(PACKET_STATE_t));
	state->send_func = s_func;
	state->process_func = p_func;
}
![img_1.png](img_1.png)
void packet_process_byte(uint8_t rx_data, PACKET_STATE_t *state)-> state->process_func(process_packet)->commands_process_packet()->做相关处理->调用相应的回应用函数或如果为阻塞处理，应在blocking_thread（）任务中
![img.png](img.png)
2、Foc大致框架
mcpwm_foc_init（）函数初始化电机参数，PWM定时器（TIM1、8用于FOC矢量控制,TIM2用于ADC采样触发）、15个ADC通信为规则同时模式、用于ADC的DMA及最后创建(timer_thread、hfi_thread、pid_thread)任务。

a、控制流程，TIM1或TIM触发TIM2输出下降沿触发ADC采集16个通道；ADC采样结束后,DMA会触发传输完成中断进入mcpwm_foc_adc_int_handler（）
b、TIM1或TIM8为中心对齐PWM1工作模式，TIM_CNT < TIM_CRRX 输出为有效电平高，一个周期会产生两次更新中断V0、V7，V7定义为is_v7 = !(TIM1->CR1 & TIM_CR1_DIR)
0：计数器递增计数，所以pwm低端为V0；


3、Three-phase PWM, and zero-sequence voltage
Sine-triangle PWM will work for three sine waves that are 120° out of phase:

The maximum line-to-line voltage you can get in this case, without creating distorted waveforms, is √3/2Vdc, or 86.6% of the DC link voltage. But we can do better than this.


MTPA 最大转矩电流比控制
简单来说，现代电机控制主要有两大类控制策略，一个是FOC，另一个是DTC，MTPA通常是磁场定向在转子坐标下。在能够对PMSM调速了之后，往往会考虑效率问题，在额定转速以下，采用MTPA控制（当然未考虑到铁耗，低速度铁耗较小），可以保证定子电流最小，从而定子电阻铜耗较低。

![img_4.png](img_4.png)

构造拉格朗日辅助函数

![img_5.png](img_5.png)

解得：

![img_6.png](img_6.png)

目前常见的两种闭环控制策略：
（1）外环转速环的输出为iq给定，id由（1）式给定，该方法较简单。
![img_2.png](img_2.png)

(2)外环转速环的输出为转矩给定，需要联立转矩方程和（2） 式给定，关键是解这个高次方程
![img_3.png](img_3.png)

<1> 实际运用采用查表法或者曲线拟合方法，为了方便移值，常采用标幺值曲线拟合，将各个物理量转化为标幺值 然后制作成表，可以通过查表或者曲线拟合得到给定dq电流
![img_7.png](img_7.png)