基于STM32F4与AD9959的信号分离装置。

基于 STM32 arm_math 库的 FFT 频谱分析与信号生成系统程序，通过 ADC 采集输入信号并进行高精度频谱分析，根据分析结果控制 AD9959 输出对应频率信号。配置 FFT 参数（4096 点 FFT 长度、200kHz 采样频率，对应频率分辨率约 48.83Hz）
通过汉宁窗减少频谱泄漏。主循环中，按设定采样频率采集 ADC1_IN4 通道模拟信号，将采样数据转换为复数形式输入 FFT 运算（调用 arm_cfft_radix4_f32 函数），计算单边幅度谱后，通过抛物线插值优化的算法查找有效频率范围（50-100000Hz）内最大和第二大的两个频率分量；系统定期（每 500 次循环）将这两个频率分别配置到 AD9959 的 CH0 和 CH1 通道并更新输出，同时通过串口打印采样频率、频率分辨率、两大频率分量的频率与幅度等频谱信息，每 200 次循环闪烁 LED1 以指示系统正常运行。频谱分析情况可通过串口打印。



配置FFT参数：

    #define FFT_LEN 4096                 // 增加FFT长度提高频率分辨率  
    #define SAMPLING_FREQ 200000         // 提高采样频率到100kHz，覆盖0-100kHz范围
    #define MIN_FREQ_THRESHOLD 50        // 最小频率阈值，过滤低频噪声
    #define MAX_FREQ_THRESHOLD 100000    // 最大频率阈值，对应采样频率的一半

打印频谱分析结果：

      // 打印频谱信息
      void print_spectrum_info(FrequencyComponent first, FrequencyComponent second) {
          float frequency_resolution = (float)SAMPLING_FREQ / FFT_LEN;
          
          // 输出频谱信息
          printf("=== 频谱分析结果 ===\r\n");
          printf("采样频率: %d Hz\r\n", SAMPLING_FREQ);
          printf("频率分辨率: %.2f Hz\r\n", frequency_resolution);
          printf("最大频率分量: %.2f Hz, 幅度: %.4f\r\n", first.frequency, first.magnitude);
          printf("第二大频率分量: %.2f Hz, 幅度: %.4f\r\n", second.frequency, second.magnitude);
          printf("====================\r\n\r\n");
      }


主循环部分：

       while(1)
        {
    				i++;
            // 从ADC读取采样数据
            for (int i = 0; i < FFT_LEN; i++) {
                // 仅使用ADC1_IN4通道数据进行FFT分析
                adc_samples[i] = (float)ADC_ConvertedValue[0] * (3.3f / 4096.0f);
            
            // 应用汉宁窗函数，减少频谱泄漏
            adc_samples[i] *= hann_window[i];
            
            // 等待下一个采样点，确保采样率正确
            delay_us(1000000 / SAMPLING_FREQ);  // 根据采样率延时
        }				    
        
        // 准备FFT输入数据（转换为复数形式）
        for (int i = 0; i < FFT_LEN; i++) {
            fft_input[i * 2] = adc_samples[i];      // 实部
            fft_input[i * 2 + 1] = 0.0f;            // 虚部（初始化为0）
        }
			
			    // 当采样满FFT_LEN（4096点）时，执行FFT分析
					if (sample_count >= FFT_LEN)
					{
            // 1. 重置采样计数器，准备下一帧
            sample_count = 0;
            
            // 2. 对采集的adc_samples应用汉宁窗
            for (int i = 0; i < FFT_LEN; i++) {
                adc_samples[i] *= hann_window[i]; // 仅需加窗，无需再转换AD值
            }
            
            // 3. 后续FFT处理（与原有代码相同）
            for (int i = 0; i < FFT_LEN; i++) {
                fft_input[i * 2] = adc_samples[i];      // 实部
                fft_input[i * 2 + 1] = 0.0f;            // 虚部
            }
     
        // 执行FFT计算
        arm_cfft_radix4_f32(&scfft, fft_input);
        
        // 计算幅度谱
        arm_cmplx_mag_f32(fft_input, fft_output, FFT_LEN);
        
        // 查找最大的两个频率分量
        find_top_two_frequencies_improved(&first, &second);
        
			}
					
			   // 定期输出信息和更新LED状态					
			     if (i % 500 == 0) {		   
						 
            // 更新AD9959输出频率
            if(first.frequency > MIN_FREQ_THRESHOLD && first.frequency < MAX_FREQ_THRESHOLD) {
                AD9959_Set_Fre(CH0, (u32)first.frequency);
                AD9959_Set_Amp(CH0, 1023);
                AD9959_Set_Phase(CH0, 0);
                printf("CH0设置频率为%.2fHz\r\n", first.frequency);
            }          
            if(second.frequency > MIN_FREQ_THRESHOLD && second.frequency < MAX_FREQ_THRESHOLD) {
                AD9959_Set_Fre(CH1, (u32)second.frequency);
                AD9959_Set_Amp(CH1, 1023);
                AD9959_Set_Phase(CH1, 0);
                printf("CH1设置频率为%.2fHz\r\n", second.frequency);
            }          
            IO_Update();  // 更新AD9959输出           
            // 打印频谱分析结果
            print_spectrum_info(first, second);
						delay_ms(2000);
        }
			
        if (i % 200 == 0) {
            LED1 = !LED1;  // 闪烁LED指示系统运行
        }
    }
