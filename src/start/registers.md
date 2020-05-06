# 内存映射寄存器

就目前我们所知,嵌入式系统只能执行常规的Rust代码,操作内存中的数据(todo: 什么样的系统不是这样?)。如果我们想获取或者修改系统的任何信息(例如，闪烁LED，检测到按钮的按下或与某种总线上的外设进行通信)，我们将不得不深入了解外设及其“内存映射寄存器”。

现在已经存在不少问外设的crate,他们可以大致进行如下分类:

* 处理器架构相关Crate (Micro-architecture Crate) - 这种crate比较通用, 可处理CPU相关的通用例程以及一些通用外设。例如，[cortex-m]crate为您提供了启用和禁用中断的功能，这些功能对于所有基于Cortex-M的CPU都是相同的。它还使您可以访问所有基于Cortex-M的微控制器附带的时钟外设(SysTick)。

* 外设相关Crate(PAC)-这种crate实际上是对特定CPU型号的内存映射寄存器的一个简单封装。例如，[tm4c123x]这个crate是对德州仪器(TI)Tiva-C TM4C123系列CPU的封装，[stm32f30x]这个crate是对ST-Micro STM32F30x系列CPU的封装。借助他们，您可以按照CPU参考手册中给出的每个外设的操作说明直接与寄存器进行交互。

* HAL crate - 这些crate通过实现[embedded-hal]中定义的一些常见Trait，来提供更友好的处理器相关API。例如，此crate可能提供一个`Serial`结构体，该结构体提供一个构造函数来配置一组GPIO引脚和波特率，并提供某种`write_byte`函数来发送数据。有关[embedded-hal]的更多信息，请参见[可移植性]一章。

*开发板相关crate - 通过预先配置各种外设和GPIO引脚以适合特定的开发板，例如针对TM32F3DISCOVERY开发板的[F3]crate，这些crate相比HAL类crate,更易用。

[cortex-m]:https：//crates.io/crates/cortex-m
[tm4c123x]:https://crates.io/crates/tm4c123x
[stm32f30x]:https://crates.io/crates/stm32f30x
[embedded-hal]:https://crates.io/crates/embedded-hal
[可移植性]:../portability/index.md
[F3]:https://crates.io/crates/f3


## 从底层开始

让我们看一下所有基于Cortex-M的微控制器共有的SysTick外设。我们可以在[cortex-m]crate中找到一个相当低级的API，我们可以像这样使用它：

```rust , ignore
use cortex_m::peripheral::{syst, Peripherals};
use cortex_m_rt::entry;

#[entry]
fn main() -> ! {
    let mut peripherals = Peripherals::take().unwrap();
    let mut systick = peripherals.SYST;
    systick.set_clock_source(syst::SystClkSource::Core);
    systick.set_reload(1_000);
    systick.clear_current();
    systick.enable_counter();
    while !systick.has_wrapped() {
        // Loop
    }

    loop {}
}
```

SYST结构上的函数接口与ARM技术参考手册为此外围设备定义的功能非常接近。这个API中没有“延迟X毫秒”这样的函数接口,因此我们必须使用`while`循环来实现它。注意，在调用 `Peripherals::take()`之前，我们无法访问`SYST`结构体-这可确保整个程序中只有一个`SYST`实例。有关更多信息，请参见[外围设备]部分。

[外围设备]:../peripherals/index.md

## 使用外设crate(PAC)

如果我们将自己限制在每个Cortex-M附带的基本外围设备上，那么我们在嵌入式软件开发方面就不会走得太远。总有一天，我们需要编写一些特定于我们正在使用的特定微控制器的代码。在此示例中，假设我们使用德州仪器(TI)TM4C123这款款处理器(具有256 KiB Flash,80MHz的Cortex-M4)。我们需要[tm4c123x]这个crate以使用此芯片。

```rust , ignore
#![no_std]
#![no_main]

extern crate panic_halt; // panic handler

use cortex_m_rt::entry;
use tm4c123x;

#[entry]
pub fn init() -> (Delay, Leds) {
    let cp = cortex_m::Peripherals::take().unwrap();
    let p = tm4c123x::Peripherals::take().unwrap();

    let pwm = p.PWM0;
    pwm.ctl.write(|w| w.globalsync0().clear_bit());
    // Mode = 1 => Count up/down mode
    pwm._2_ctl.write(|w| w.enable().set_bit().mode().set_bit());
    pwm._2_gena.write(|w| w.actcmpau().zero().actcmpad().one());
    // 528 cycles (264 up and down) = 4 loops per video line (2112 cycles)
    pwm._2_load.write(|w| unsafe { w.load().bits(263) });
    pwm._2_cmpa.write(|w| unsafe { w.compa().bits(64) });
    pwm.enable.write(|w| w.pwm4en().set_bit());
}

```

除了调用`tm4c123x::Peripherals::take()`之外，我们访问PWM0外设的方式与之前访问SYST外设的方式完全相同。由于此crate是使用[svd2rust]自动生成的，因此我们寄存器的访问函数采用闭包而不是数字参数。尽管这看起来有很多代码，但是Rust编译器可以为我们执行一堆检查以及优化，然后生成与手写汇编代码非常接近的机器代码！自动生成的代码如果无法确定特定访问器函数的参数的所有可能值均有效(例如，SVD将寄存器定义为32位整数，但实际上只有其中的某些值才有特殊含义,才有意义)，则该函数被标记为“不安全”。我们在上面的示例中使用`bits()` 函数设置 `load` 和`compa` 子字段时可以看到这一点。

### 读访问

 `read()` 函数返回一个对象R，该对象只有对该寄存器中各个子字段的只读访问权限，这些权限由制造商的该芯片的SVD文件定义。R上面定义的所有函数功能,您可以在[tm4c123x文档] [tm4c123x文档R]中找到针对此款处理器,此种外设的具体寄存器的定义。


```rust , ignore
if pwm.ctl.read().globalsync0().is_set() {
    // Do a thing
}
```

### 写访问

 `write()`函数采用一个带有单个参数的闭包。通常我们将其称为 `w`。根据制造商关于此芯片的SVD文件，此参数可对该寄存器内的各个子字段进行读写访问。同样，`w`上面定义 所有函数功能,您可以在[tm4c123x文档] [tm4c123x文档W]中找到针对此处理器,此外设的具体寄存器的定义。请注意，我们未设置的所有子字段都将被设置默认值-寄存器中的任何现有内容都将丢失。

```rust , ignore
pwm.ctl.write(|w| w.globalsync0().clear_bit());
```

### 修改

如果我们只想更改该寄存器中的一个特定子字段，而使其他子字段保持不变，则可以使用`modify`函数。此函数采用带有两个参数的闭包-一个用于读取，一个用于写入。通常，我们分别将它们称为`r`和`w`。 r参数可用于读取寄存器的当前内容，w参数可用于修改寄存器的内容。

```rust , ignore
pwm.ctl.modify(|r, w| w.globalsync0().clear_bit());
```

`modify`函数在这里显示了闭包的强大。在C语言中，我们必须读入到一些临时值，修改特定位上的值，然后将其写回。这意味着不小的出错几率：

```C
uint32_t temp = pwm0.ctl.read();
temp |= PWM0_CTL_GLOBALSYNC0;
pwm0.ctl.write(temp);
uint32_t temp2 = pwm0.enable.read();
temp2 |= PWM0_ENABLE_PWM4EN;
pwm0.enable.write(temp); // Uh oh! Wrong variable!
```

[svd2rust]:https://crates.io/crates/svd2rust
[tm4c123x文档R]:https://docs.rs/tm4c123x/0.7.0/tm4c123x/pwm0/ctl/struct.R.html
[tm4c123x文档W]:https://docs.rs/tm4c123x/0.7.0/tm4c123x/pwm0/ctl/struct.W.html

## 使用HAL crate

具体芯片的HAL crate一般是通过为PAC crate导出的结构体实现自定义Trait来工作。通常，这个自定义crate为单体外设定义一个名为 `constrain()` 的函数，为具有多个引脚的GPIO端口之类的外设定义 `split()` 函数。该函数将消耗底层的原始外围设备结构，并返回具有更高级别API的新对象。这个API可能还会做一些事情，例如让串口`new`函数需要`Clock`结构体的借用，这个Clock结构体只能通过调用特定函数来生成,而这个函数会配置PLL并设置时钟频率。这样在没有先配置时钟频率的情况下，就不可能创建串口对象, 否则串口对象有可能将波特率误转换为错误的时钟滴答。一些crate甚至为每个GPIO引脚可以处于的状态定义了特殊的Trait，要求用户在将引脚传递到外设之前将其置于正确的状态(例如，通过选择适当的可选功能模式)。更重要的是,这些都是零成本抽象！

让我们来看一个例子：

```rust , ignore
#![no_std]
#![no_main]

extern crate panic_halt; // panic handler

use cortex_m_rt::entry;
use tm4c123x_hal as hal;
use tm4c123x_hal::prelude::*;
use tm4c123x_hal::serial::{NewlineMode, Serial};
use tm4c123x_hal::sysctl;

#[entry]
fn main() -> ! {
    let p = hal::Peripherals::take().unwrap();
    let cp = hal::CorePeripherals::take().unwrap();

    // Wrap up the SYSCTL struct into an object with a higher-layer API
    let mut sc = p.SYSCTL.constrain();
    // Pick our oscillation settings
    sc.clock_setup.oscillator = sysctl::Oscillator::Main(
        sysctl::CrystalFrequency::_16mhz,
        sysctl::SystemClock::UsePll(sysctl::PllOutputFrequency::_80_00mhz),
    );
    // Configure the PLL with those settings
    let clocks = sc.clock_setup.freeze();

    // Wrap up the GPIO_PORTA struct into an object with a higher-layer API.
    // Note it needs to borrow `sc.power_control` so it can power up the GPIO
    // peripheral automatically.
    let mut porta = p.GPIO_PORTA.split(&sc.power_control);

    // Activate the UART.
    let uart = Serial::uart0(
        p.UART0,
        // The transmit pin
        porta
            .pa1
            .into_af_push_pull::<hal::gpio::AF1>(&mut porta.control),
        // The receive pin
        porta
            .pa0
            .into_af_push_pull::<hal::gpio::AF1>(&mut porta.control),
        // No RTS or CTS required
        (),
        (),
        // The baud rate
        115200_u32.bps(),
        // Output handling
        NewlineMode::SwapLFtoCRLF,
        // We need the clock rates to calculate the baud rate divisors
        &clocks,
        // We need this to power up the UART peripheral
        &sc.power_control,
    );

    loop {
        writeln!(uart, "Hello, World!\r\n").unwrap();
    }
}
```
