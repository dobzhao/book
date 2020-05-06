# 其他工具

进行嵌入式开发和在pc上进行开发不太一样,一般来说,你必须在远程设备上进行运行和调试,所以需要一些专门的控制器相关的工具的支持.

 
下面是要用的所有工具,给出的都是经过测试的版本,当然一般来说更高的版本也是可以的。

- Rust 1.31、1.31-beta或更新的工具链以及ARM Cortex-M编译支持。
- [`cargo-binutils`](https://github.com/rust-embedded/cargo-binutils)〜0.1.4
- [`qemu-system-arm`](https://www.qemu.org/)经过测试的版本： 3.0.0
- OpenOCD> = 0.8。经过测试的版本：v0.9.0和v0.10.0
- 具有ARM支持的GDB。强烈建议使用7.12版或更高版本。经过测试版本：7.10、7.11、7.12和8.1
- [`cargo-generate`](https://github.com/ashleygwilliams/cargo-generate)或`git`。这些工具是可选的，它能够让我们更容易使用书上的例子. 


## `cargo-generate`或`git`

裸机程序是非标准(`no_std`)Rust程序，一般需要介入链接过程以修正程序的内存布局。这需要一些其他文件(例如链接器脚本)和设置(链接参数)。我们已经为您打包了这些模板,这样您只需要填写缺少的信息(例如项目名称和目标硬件的特性)。

我们的模板兼容`cargo-generate`(这是一个cargo的子命令)。您也可以使用`git`，`curl`，`wget`或浏览器来下载模板。

## `cargo-binutils`

`cargo-binutils`是一系列Cargo子命令的集合，通过它们可以避免直接与Rust工具链附带的LLVM工具打交道,它包括objdump，nm和size等用于检查二进制文件的工具.

与GNU binutils相比，使用这些工具的优势在于:
- 安装简单,无论什么系统,一条命令(`rustup component add llvm-tools-preview`)与LLVM工具一同安装
- 跨平台,像`objdump`的这样的工具与rustc一样支持所有的架构(从ARM到x86_64),因为它们都共享相同的LLVM后端。

## `qemu-system-arm`

QEMU是一个通用模拟器,使用它可以完全模拟ARM处理器,这样可以在主机上运行嵌入式程序。幸亏有了 QEMU,这样就算是你没有任何硬件,也可以运行本书的大部分示例！

## GDB
调试器对于嵌入式开发非常重要,因为可能你都无法保证一定有条件可以向控制台打印日志. 比如有时你的硬件平台都没有提供闪烁的LED灯.
 

通常在调试方面，LLDB和GDB一样好，但是我们还没有找到了与GDB的“ load”命令相对应的LLDB命令，该命令可以将程序上传到目标硬件，因此当前我们建议您使用GDB。

## OpenOCD
 


GDB无法直接与STM32F3DISCOVERY开发板上的ST-Link调试硬件进行通信,OpenOCD起到了翻译器的作用。 OpenOCD运行在PC上，可在基于TCP/IP的GDB远程调试协议和基于USB的ST-Link协议之间进行转换。

OpenOCD还执行其他重要工作：
* 它知道如何与用于ARM CoreSight调试外围设备使用的内存映射寄存器进行交互。这些CoreSight寄存器允许：
  + 断点/观察点操作
  + 读取和写入CPU寄存器
  + 检测CPU何时因调试事件而暂停
  + 遇到调试事件后继续执行CPU
  + 其他功能
* 它也知道如何擦除和写入微控制器的FLASH