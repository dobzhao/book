# 介绍

欢迎阅读《嵌入式Rust编程》: 一本介绍使用Rust在
“裸机”嵌入式系统(例如微控制器)上编程的入门书籍。

本书[github地址](https://github.com/nkbai/book) 欢迎提出问题. [英文原书地址](https://github.com/rust-embedded/book)
## 本书的潜在读者

本书适用于希望使用Rust提供的高级概念和安全性的嵌入式开发工程师。(另请参见[Rust的目标对象](https://doc.rust-lang.org/book/ch00-00-introduction.html))

## 范围

本书的目标是: 

* 使开发人员与嵌入式Rust开发同步。即如何建立开发环境。

* 分享 *当前* 关于使用Rust进行嵌入式开发的最佳实践。即
  如何最好地使用Rust语言特性来编写更正确的嵌入式系统。

* 也可以作为手册。例如如何在同一个项目中混合使用C和Rust？

本书试图尽可能地涵盖更多议题，但是为了既降低对读者也降低对作者的要求,本书所有的例子都针对Cortex-M架构的ARM处理器。 但是，本书并不假定读者对此处理架构非常熟悉，因此会在需要的地方解释该架构的特定细节。

## 这本书适合谁
本书面向的是具有嵌入式背景或熟悉Rust语言的人，但是我们相信每个对嵌入式Rust编程感兴趣的人都可以从本书中学到一些东西。对于那些没有任何先验知识的人，我们建议您阅读[假设和先决条件](#假设和先决条件)部分，并补上缺少的知识。您可以查看[其他资源](#其他资源)部分以找到有关主题的资源。
 

### 假设和先决条件

* 您很习惯使用Rust编程语言， 在桌面环境上编写,运行和调试过Rust应用程序。你也应该熟悉本书针对的[2018版](https://doc.rust-lang.org/edition-guide/)语法。
 
* 您可以轻松地在使用至少一种语言开发和调试嵌入式系统，例如C，C++或Ada，并且熟悉以下概念: 
  * 交叉编译
  * 内存映射外设
  * 中断
  * 通用接口，例如I2C，SPI，串行等。

### 其他资源
如果您不熟悉上述任何内容，或者想要了解有关本书中提到的特定主题的更多信息，下面的资源可能会有所帮助。

|主题|资源|描述
| -------------- | ---------- | ------------- |
|Rust| [Rust Book](https://doc.rust-lang.org/book/)|如果您对Rust尚不熟悉，我们强烈建议您阅读本书。 |
|Rust,嵌入式| [Discovery Book](https://docs.rust-embedded.org/discovery/)|如果您从未做过任何嵌入式编程，那么本书可能是一个更好的开始|
|Rust,嵌入式| [嵌入式Rust书架](https://docs.rust-embedded.org)|在这里，您可以找到Rust嵌入式工作组提供的其他一些资源。 |
|Rust,嵌入式| [Embedonomicon](https://docs.rust-embedded.org/embedonomicon/)|用Rust进行嵌入式编程，细节非常棒。 |
|Rust,嵌入式| [嵌入式常见问题解答](https://docs.rust-embedded.org/faq.html)|关于嵌入式Rust的常见问题。 |
|中断| [中断](https://en.wikipedia.org/wiki/Interrupt)| -|
|内存映射的IO外设| [内存映射的I/O](https://en.wikipedia.org/wiki/Memory-mapped_I/O)| -|
| SPI，UART，RS232，USB，I2C，TTL | [有关SPI，UART和其他接口的堆栈交换](https://electronics.stackexchange.com/questions/37814/usart-uart-rs232-usb-spi-i2c-ttl-etc-what-are-all-of-these-and-how-do-th)| -|

## 如何使用这本书

本书通常假定您会从头到尾阅读它。后面的章节会构建在前面各章的基础上，前面的章节可能会在一个主题上点到即止，而后面的章节则会重新深入讨论该主题。

本书大多数示例都基于[STM32F3DISCOVERY](http://taobao.com)开发板。这个板子基于ARM Cortex-M架构. 此架构的大多数CPU的最基本功能都是相同的，但是外设和实现细节则随供应商不同而不同，甚至同一供应商的不同系列微处理器家族之间也不尽相同。

因此，为了遵循本书中的示例，我们建议购买[STM32F3DISCOVERY](http://www.st.com/en/evaluation-tools/stm32f3discovery.html)开发板

 

## 改进本书

本书的工作在[此存储库](https://github.com/rust-embedded/book)中，主要是由[Rust资源团队](https://github.com/rust-embedded/wg#the-resources-team)开发。
 

如果您在阅读本书时遇到困难，或者发现一些本书的部分内容不够清晰或难以理解，可以在本书的[问题跟踪](https://github.com/rust-embedded/book/issues/)中进行报告。
 

欢迎针对本书提供任何但不限于有关拼写和新内容的PR.

## 重复使用此材料

本书遵循以下许可: 

* 本书中包含的示例代码和独立的Cargo项目均遵循[MIT许可]和[Apache许可v2.0]的条款。
* 本书中包含的书面散文，图片和图表均遵循[CC-BY-SA v4.0]许可条款。

[MIT许可证]: https://opensource.org/licenses/MIT
[Apache许可证v2.0]:http://www.apache.org/licenses/LICENSE-2.0
[CC-BY-SA v4.0]: https://creativecommons.org/licenses/by-sa/4.0/legalcode

TL; DR: 如果您想在工作中使用我们的文字或图片，则需要: 

* 给予适当的感谢(即在幻灯片上提及此书，并提供指向相关页面的链接)
* 提供[CC-BY-SA v4.0]许可证的链接
* 指出您是否以任何方式更改了材料，并根据相同的许可对我们的材料进行了任何更改

另外，如果您觉得这本书有用，请告诉我们！