# Windows

## `arm-none-eabi-gdb`

ARM为Windows提供了exe安装程序。从这里下载[gcc]，然后按照说明进行操作。在安装过程即将完成之前，勾选“Add path to environment variable”选项。然后验证工具是否在您的“％PATH％”中：

``` console
$ arm-none-eabi-gdb -v
GNU gdb (GNU Tools for Arm Embedded Processors 7-2018-q2-update) 8.1.0.20180315-git
(..)
```

[gcc]:https://developer.arm.com/open-source/gnu-toolchain/gnu-rm/downloads

## OpenOCD

没有适用于Windows的OpenOCD官方二进制发行版，但是[这里](https://github.com/gnu-mcu-eclipse/openocd/releases)有非官方发行版。下载0.10.x zip文件，并将其解压缩到驱动器上的某个位置(我建议使用C:\OpenOCD)，然后更新`％PATH％`环境变量，使其包含以下路径：`C:\OpenOCD\bin`(或之前选择的路径)。
 

使用以下命令验证OpenOCD是否在您的“％PATH％”中：


``` console
$ openocd -v
Open On-Chip Debugger 0.10.0
(..)
```


## QEMU

从[官方网站](https://www.qemu.org/download/#windows)下载QEMU。
 
## ST-LINK USB驱动程序

您还需要安装[此USB驱动程序],否则OpenOCD无法正常工作。按照安装程序的说明进行操作，并确保您安装了正确版本的驱动程序(32位或64位)。

[此USB驱动程序]:http://www.st.com/en/embedded-software/stsw-link009.html

就这样！转到[下一部分]。

[下一部分]:verify.md