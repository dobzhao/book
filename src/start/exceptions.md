# 异常

异常和中断是一种硬件机制，处理器通过该机制处理异步事件和致命错误(例如执行无效指令)。异常意味着抢占，也包括异常处理程序，发生异常时,这些子程序会被立即执行,以响应触发异常的信号。

cortex-m-rt crate提供了一个[`exception`]属性来声明异常处理程序。

[`exception`]:https://docs.rs/cortex-m-rt-macros/latest/cortex_m_rt_macros/attr.exception.html

``` rust , ignore
// Exception handler for the SysTick (System Timer) exception
#[exception]
fn SysTick() {
    // ..
}
```

除了`exception` 属性之外，异常处理程序看起来像普通函数，但还有另外一个区别：`exception` 处理程序不能被软件调用。上面的示例中，语句`SysTick();`将导致编译错误。

这种行为是有意为之，`exception`属性还让在异常处理程序中声明`static mut`变量是安全的。

``` rust , ignore
#[exception]
fn SysTick() {
    static mut COUNT: u32 = 0;

    // `COUNT` has type `&mut u32` and it's safe to use
    *COUNT += 1;
}
```

如您所知，在函数中使用`static mut`变量使其成为[不可重入](https://en.wikipedia.org/wiki/Reentrancy_(computing))。 从多个异常/中断处理程序或`main`中直接或间接调用不可重入函数是不确定的行为。

Safe Rust绝不能导致不确定的行为，因此非可重入函数必须标记为 `unsafe`。但是我刚刚却说异常处理程序可以安全地使用`static mut`变量。这怎么可能？这是可能的，因为异常处理程序不能被函数调用，因此无法重入。

## 一个完整的例子

这是一个使用系统计时器每秒引发一次`SysTick` 异常的示例。 SysTick异常处理程序通过COUNT变量跟踪自己被调用了多少次，然后使用半主机将COUNT的值打印到主机控制台。

> **注意**：您可以在任何Cortex-M设备上运行此示例；您也可以在QEMU上运行它

```rust , ignore
#![deny(unsafe_code)]
#![no_main]
#![no_std]

extern crate panic_halt;

use core::fmt::Write;

use cortex_m::peripheral::syst::SystClkSource;
use cortex_m_rt::{entry, exception};
use cortex_m_semihosting::{
    debug,
    hio::{self, HStdout},
};

#[entry]
fn main() -> ! {
    let p = cortex_m::Peripherals::take().unwrap();
    let mut syst = p.SYST;

    // configures the system timer to trigger a SysTick exception every second
    syst.set_clock_source(SystClkSource::Core);
    // this is configured for the LM3S6965 which has a default CPU clock of 12 MHz
    syst.set_reload(12_000_000);
    syst.clear_current();
    syst.enable_counter();
    syst.enable_interrupt();

    loop {}
}

#[exception]
fn SysTick() {
    static mut COUNT: u32 = 0;
    static mut STDOUT: Option<HStdout> = None;

    *COUNT += 1;

    // Lazy initialization
    if STDOUT.is_none() {
        *STDOUT = hio::hstdout().ok();
    }

    if let Some(hstdout) = STDOUT.as_mut() {
        write!(hstdout, "{}", *COUNT).ok();
    }

    // IMPORTANT omit this `if` block if running on real hardware or your
    // debugger will end in an inconsistent state
    if *COUNT == 9 {
        // This will terminate the QEMU process
        debug::exit(debug::EXIT_SUCCESS);
    }
}
```


``` console
$ tail -n5 Cargo.toml
```

``` toml
[dependencies]
cortex-m = "0.5.7"
cortex-m-rt = "0.6.3"
panic-halt = "0.2.0"
cortex-m-semihosting = "0.3.1"
```

``` console
$ cargo run --release
     Running `qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb (..)
123456789
```

如果在开发板上运行此命令，则会在OpenOCD控制台上看到输出。但是当计数达到9时，程序将**不**停止。

## 默认异常处理程序

 `exception`属性的实际作用是**覆盖**特定异常的默认异常处理程序。如果您不重写特定异常的处理程序，它将由`DefaultHandler`函数处理，该函数默认为：

``` rust , ignore
fn DefaultHandler() {
    loop {}
}
```

此功能由`cortex-m-rt`crate提供，并标记为 `#[no_mangle]`，因此您可以在“ DefaultHandler”上放置断点并捕获**未处理**异常。

可以使用`exception`属性覆盖这个`DefaultHandler`：

``` rust , ignore
#[exception]
fn DefaultHandler(irqn: i16) {
    // custom default handler
}
```

irqn是正在处理的异常编号。负值表示Cortex-M异常；零或正值表示设备特定的异常，即AKA中断。

## 硬故障处理程序

`HardFault`异常有点特殊。当程序进入无效状态时，将引发此异常，因此它的处理程序不能返回，因为这可能导致未定义的行为。另外，在调用用户定义的`HardFault`前，运行时crate会做一些工作以提高程序的可调试性。

所以`HardFault`处理函数必须具有以下签名：`fn(＆ExceptionFrame)->！`。处理程序的参数是指向被异常压入堆栈的寄存器的指针。这些寄存器是异常触发时处理器状态的快照，可用于诊断故障。

这是一个执行非法操作的示例：读取不存在的内存位置。

> **注意**：该程序在QEMU上不起作用，即不会崩溃，因为`qemu-system-arm -machine lm3s6965evb`不会检查内存读取，并且在读取到无效内存时会很高兴地返回`0`。


```rust , ignore
#![no_main]
#![no_std]

extern crate panic_halt;

use core::fmt::Write;
use core::ptr;

use cortex_m_rt::{entry, exception, ExceptionFrame};
use cortex_m_semihosting::hio;

#[entry]
fn main() -> ! {
    // read a nonexistent memory location
    unsafe {
        ptr::read_volatile(0x3FFF_FFFE as *const u32);
    }

    loop {}
}

#[exception]
fn HardFault(ef: &ExceptionFrame) -> ! {
    if let Ok(mut hstdout) = hio::hstdout() {
        writeln!(hstdout, "{:#?}", ef).ok();
    }

    loop {}
}
```

`HardFault`处理程序将打印`ExceptionFrame`值。如果运行此程序，您将在OpenOCD控制台上看到类似的内容。

``` console
$ openocd
(..)
ExceptionFrame {
    r0: 0x3ffffffe,
    r1: 0x00f00000,
    r2: 0x20000000,
    r3: 0x00000000,
    r12: 0x00000000,
    lr: 0x080008f7,
    pc: 0x0800094a,
    xpsr: 0x61000000
}
```

`pc` 值是发生异常时程序计数器的值，它指向触发异常的指令。

如果您查看程序的反汇编：


``` console
$ cargo objdump --bin app --release -- -d -no-show-raw-insn -print-imm-hex
(..)
ResetTrampoline:
 8000942:       movw    r0, #0xfffe
 8000946:       movt    r0, #0x3fff
 800094a:       ldr     r0, [r0]
 800094c:       b       #-0x4 <ResetTrampoline+0xa>
```

您可以在反汇编中查找程序计数器`0x0800094a` 的值。您将看到加载操作(`ldr r0，[r0]`)引起了异常。 `ExceptionFrame`的`r0`字段将告诉您寄存器r0的值为当时的0x3fff_fffe。