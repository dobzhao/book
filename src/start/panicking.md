# 恐慌(Panicking)

恐慌是Rust语言的核心部分。诸如索引之类的内置操作会在运行时检查内存安全性。当尝试超出索引范围时，将导致恐慌。

在标准库中，恐慌具有确定的行为：恐慌会进行线程栈展开，除非用户选择在恐慌中中止程序。

但是，在没有标准库的程序中，恐慌行为未定义。可以通过声明一个 `#[panic_handler]` 函数来选择一种行为。该函数必须在程序的依赖关系中恰好出现一次，并且必须具有以下签名：`fn(＆PanicInfo)->！`，其中[`PanicInfo`]包含有恐慌相关的位置信息 。

[`PanicInfo`]:https://doc.rust-lang.org/core/panic/struct.PanicInfo.html

鉴于嵌入式系统的范围广泛,从消费类电子到对安全至关重要的系统(不能崩溃)，因此没有一种适合所有场景的恐慌处理行为，但是有许多常用行为。这些常见的行为已被打包到定义 `#[panic_handler]` 功能的crate中,常见的包括：

- [`panic-abort`] 恐慌时会执行abort指令。
- [`panic-halt`] 恐慌时会导致程序或者其所在线程通过进入死循环的方式停止。
- [`panic-itm`] 恐慌消息使用ITM(ARM Cortex-M特定的外围设备)记录。
- [`panic-semihosting`] 恐慌消息使用半主机技术记录到主机。

[`panic-abort`]:https://crates.io/crates/panic-abort
[`panic-halt`]:https://crates.io/crates/panic-halt
[`panic-itm`]:https://crates.io/crates/panic-itm
[`panic-semihosting`]:https://crates.io/crates/panic-semihosting

在crates.io上搜索[`panic-handler`]，您也许可以找到更多的crate。

[`panic-handler`]:https://crates.io/keywords/panic-handler

程序可以简单地通过链接到相应的crate来选择其中一种行为,还可以根据编译配置文件更改恐慌行为。例如：


``` rust , ignore
#![no_main]
#![no_std]

// dev profile: easier to debug panics; can put a breakpoint on `rust_begin_unwind`
#[cfg(debug_assertions)]
extern crate panic_halt;

// release profile: minimize the binary size of the application
#[cfg(not(debug_assertions))]
extern crate panic_abort;

// ..
```

在此示例中，使用开发人员配置文件(`cargo build`)构建时，crate链接到`panic-halt`，而当使用发布配置文件构建时，则链接到`panic-abort`crate(`cargo build --release `)。

##  一个例子

这是一个尝试索引越界的示例。该操作导致恐慌。

```rust , ignore
#![no_main]
#![no_std]

extern crate panic_semihosting;

use cortex_m_rt::entry;

#[entry]
fn main() -> ! {
    let xs = [0, 1, 2];
    let i = xs.len() + 1;
    let _y = xs[i]; // out of bounds access

    loop {}
}
```

本示例选择了`panic-semihosting`恐慌处理方式，该方式将恐慌消息打印到主机控制台。

``` console
$ cargo run
     Running `qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb (..)
panicked at 'index out of bounds: the len is 3 but the index is 4', src/main.rs:12:13
```

您可以尝试将行为更改为`panic-halt` ，并确认在这种情况下不打印任何消息。