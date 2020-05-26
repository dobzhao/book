# 验证安装

在本节中，我们检查是否已正确安装和配置了必需的工具和驱动程序。

使用micro-USB电缆将开发板连接到笔记本电脑/PC。开发板有两个USB接口。请使用位于板边缘中央的标有“USB ST-LINK”的USB接口。

还要检查ST-LINK跳线是否连接。见下图； ST-LINK标头用红色圈出。

<p align="center">
<img title="Connected discovery board" src="../../assets/verify.jpeg">
</p>


现在运行以下命令:

``` console
$ openocd -f interface/stlink-v2-1.cfg -f target/stm32f3x.cfg
```


您应该获得以下输出，并且阻塞控制台:


``` text
Open On-Chip Debugger 0.10.0
Licensed under GNU GPL v2
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
Info : auto-selecting first available session transport "hla_swd". To override use 'transport select <transport>'.
adapter speed: 1000 kHz
adapter_nsrst_delay: 100
Info : The selected transport took over low-level target control. The results might differ compared to plain JTAG/SWD
none separate
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : clock speed 950 kHz
Info : STLINK v2 JTAG v27 API v2 SWIM v15 VID 0x0483 PID 0x374B
Info : using stlink api v2
Info : Target voltage: 2.919881
Info : stm32f3x.cpu: hardware has 6 breakpoints, 4 watchpoints
```

内容可能不完全匹配，但是您应该看到有关断点和观察点的最后一行。如果看到了，则终止OpenOCD进程并移至[下一部分]。

[下一部分]: ../../start/index.md

如果没有得到“断点”行，请尝试以下命令。

``` console
$ openocd -f interface/stlink-v2.cfg -f target/stm32f3x.cfg
```

如果该命令有效，则说明您的开发板的硬件版本较旧。这虽然不是一个问题，但是请记住你稍后需要对配置做一些修改。现在您可以转到[下一部分]。

如果这两个命令都不能作为普通用户使用，请尝试以root权限运行它们（例如`sudo openocd ..`）。如果这时可以正常工作，则请检查[udev规则]是否已正确设置。

[udev规则]:linux.md#udev-rules

如果您到了这一步，OpenOCD无法正常工作，请提交一个[问题]，我们将为您提供帮助！

[问题]:https://github.com/rust-embedded/book/issues