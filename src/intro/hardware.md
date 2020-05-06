# 认识您的硬件

让我们熟首先了解一下我们将要使用的硬件。

## STM32F3DISCOVERY(以下简称"F3")

<p align ="center">
<img title ="F3" src ="../assets/f3.jpg">
</p>

该板包含什么？

- [STM32F303VCT6](https://www.st.com/en/microcontrollers/stm32f303vc.html)微控制器。该微控制器具有
    - 支持单精度浮点数的单核ARM Cortex-M4F处理器，最大时钟频率为72 MHz。
    - 256KiB的Flash。 (1 KiB = 1024字节)
    - 48KiB的RAM。
    - 各种集成外设，例如定时器，I2C，SPI和USART。
    - 通用输入输出(GPIO)和其他类型的引脚，可通过板侧的两排引脚访问。
    - 一个USB接口 标有"USB USER"的USB端口。
- [加速度计](https://en.wikipedia.org/wiki/Accelerometer)(作为[LSM303DLHC](https://www.st.com/en/mems-and-sensors/lsm303dlhc.html)的一部分)。

- [磁力仪](https://en.wikipedia.org/wiki/Magnetometer)(作为[LSM303DLHC](https://www.st.com/en/mems-and-sensors/lsm303dlhc.html)的一部分)。

- [陀螺仪](https://en.wikipedia.org/wiki/Gyroscope) (作为[L3GD20](https://www.pololu.com/file/0J563/L3GD20.pdf)芯片的一部分)。

- 8个LED(以指南针样式排列)

- 第二个微处理器：[STM32F103](https://www.st.com/en/microcontrollers/stm32f103cb.html)。该微控制器实际上是板载编程器/调试器的一部分，连接到名为"USB ST-LINK"的USB端口。

有关该板子的功能和更多规格的详细列表，请访问[STMicroelectronics](https://www.st.com/en/evaluation-tools/stm32f3discovery.html)。

**特别注意**：如果要将外部信号施加到板上，请小心。微控制器STM32F303VCT6引脚的标称电压为3.3伏。有关更多信息，请参阅[手册中的6.2章节 绝对最大额定值部分](https://www.st.com/resource/zh/datasheet/stm32f303vc.pdf)