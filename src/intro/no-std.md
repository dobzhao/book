#  `no_std` 环境

术语嵌入式编程用于各种不同类型的处理器。从仅有几KB的RAM和ROM的8位MCU(例如[ST72325xx](https://www.st.com/resource/zh/datasheet/st72325j6.pdf))，到拥有Cortex-A53处理器(该处理器有四个核心,主频1.4G赫兹)和1GB RAM的树莓派等系统([Model B 3+](https://en.wikipedia.org/wiki/Raspberry_Pi#Specifications))。编写代码时的各种不同限制完全取决于您的目标系统环境。

有两种常规的嵌入式编程分类：

## 托管环境
这些环境接近普通的PC环境。这意味着有操作系统支持[比如 POSIX](https://en.wikipedia.org/wiki/POSIX), 包括与各种系统资源进行交互的原语，例如文件系统，网络，内存管理，线程等。反过来，标准库通常依靠这些原语来实现其功能。您可能还具有某种sysroot和对RAM/ROM使用的限制，也许还有一些特殊的硬件或I/O外设。总体而言，感觉就像在专用PC环境中进行编码。

## 裸机环境

在裸机环境中，系统在运行你的代码之前,没有未加载任何代码。因为没有操作系统的支持，我们将无法使用标准库。
相反，程序及其使用的crate只能直接使用硬件(裸机)来运行。为了防止Rust加载标准库，必须使用`no_std`。可通过[核心库](https://doc.rust-lang.org/core/)获得标准库中与平台无关的部分。核心库还排除了嵌入式环境中并不总是需要的东西。其中之一是用于动态内存分配的内存分配器。如果您需要此功能或任何其他功能，通常会有第三方crate实现。


### 标准库运行时
如前所述，使用[标准库]需要某种类型的系统集成，但这不仅是因为[标准库] 提供了一种访问操作系统抽象的通用方法，它还提供了一个运行时。该运行时还负责设置堆栈溢出保护，处理命令行参数，并在调用程序的main函数之前生成主线程。这些功能在`no_std`环境中都无法提供。

[标准库]:(https://doc.rust-lang.org/std/)

## 总结
`#![no_std]` 是一个crate级属性，指示该crate将链接到核心库，而不是标准库。[核心库]是标准库的与平台无关的子集,它不对程序运行的系统做任何假设。它只提供了语言相关(例如浮点数，字符串和切片)的API，以及处理器功能(例如原子操作和SIMD指令)的API。但是，它缺少涉及平台集成的任何东西的API。 由于这些属性，`no_std`和[核心库]代码可用于任何类型的引导程序(阶段0)代码，例如bootloader，固件或内核。

[核心库]:(https://doc.rust-lang.org/core/)

### 总结

|功能 | no\_std |标准|
| ----------------------------------------------------------- | -------- | ----- |
|堆(动态内存)| * | ✓|
|集合(Vec，HashMap等)| ** | ✓|
|堆栈溢出保护| ✘| ✓|
|在main之前运行初始化代码| ✘| ✓|
| libstd可用| ✘| ✓|
| libcore可用| ✓| ✓|
|编写固件，内核或引导程序代码| ✓| ✘|

\* 仅当您使用`alloc` crate并使用合适的分配器(如[alloc-cortex-m])时。

\** 仅当您使用`collections`crate并配置全局默认分配器时。

[alloc-cortex-m]: https://github.com/rust-embedded/alloc-cortex-m

## 其他资料
* [RFC-1184](https://github.com/rust-lang/rfcs/blob/master/text/1184-stabilize-no_std.md)