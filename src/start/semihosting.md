# 半主机

半主机是这样一种机制，它允许嵌入式设备在主机上执行I/O操作，主要用于将消息记录到主机控制台。半主机除了需要调试会话之外，几乎不需要其他任何操作(不需要额外的接线！)，因此使用起来超级方便。缺点是它非常慢：根据您使用的硬件调试器不同(例如ST-Link)，每个写入操作可能要花费几毫秒。

[`cortex-m-semihosting`]crate提供了一个API，可以在Cortex-M设备上进行半主机操作。下面的程序是“ Hello，world！”的半主机版本：

[`cortex-m-semihosting`]:https://crates.io/crates/cortex-m-semihosting

```rust , ignore
#![no_main]
#![no_std]

extern crate panic_halt;

use cortex_m_rt::entry;
use cortex_m_semihosting::hprintln;

#[entry]
fn main() -> ! {
    hprintln!("Hello, world!").unwrap();

    loop {}
}
```

如果您在硬件上运行此程序，则会在OpenOCD日志中看到"Hello, world!" 消息。

``` console
$ openocd
(..)
Hello, world!
(..)
```

您需要先从GDB中启用OpenOCD的半主机：

``` console
(gdb) monitor arm semihosting enable
semihosting is enabled
```

QEMU能够理解半主机操作，因此上述程序也可以与`qemu-system-arm`一起使用，而无需启动调试会话。注意，您需要将`-semihosting-config`参数传递给QEMU以启用半主机支持。这些参数已经包含在模板的`cargo/config`文件中。

``` console
$ # this program will block the terminal
$ cargo run
     Running `qemu-system-arm (..)
Hello, world!
```

还有一个`exit`半主机操作可用于终止QEMU进程。重要提示：不要在硬件上使用`debug::exit`；此功能可能会破坏您的OpenOCD会话，并且只有重新启动它才能调试更多程序。


```rust , ignore
#![no_main]
#![no_std]

extern crate panic_halt;

use cortex_m_rt::entry;
use cortex_m_semihosting::debug;

#[entry]
fn main() -> ! {
    let roses = "blue";

    if roses == "red" {
        debug::exit(debug::EXIT_SUCCESS);
    } else {
        debug::exit(debug::EXIT_FAILURE);
    }

    loop {}
}
```

``` console
$ cargo run
     Running `qemu-system-arm (..)

$ echo $?
1
```

最后一个提示：您可以将恐慌行为设置为`exit(EXIT_FAILURE)`。这将使您编写可以在QEMU上运行的`no_std`测试案例。

为方便起见，`panic-semihosting`crate具有“退出”功能，启用后，会将panic消息记录到主机stderr后调用`exit(EXIT_FAILURE)`。

```rust , ignore
#![no_main]
#![no_std]

extern crate panic_semihosting; // features = ["exit"]

use cortex_m_rt::entry;
use cortex_m_semihosting::debug;

#[entry]
fn main() -> ! {
    let roses = "blue";

    assert_eq!(roses, "red");

    loop {}
}
```

``` console
$ cargo run
     Running `qemu-system-arm (..)
panicked at 'assertion failed: `(left == right)`
  left: `"blue"`,
 right: `"red"`', examples/hello.rs:15:5

$ echo $?
1
```

**注意**：要在`panic-semihosting`上启用此功能，请在您的`Cargo.toml`依赖项部分中编辑`panic-semihosting`：

``` toml
panic-semihosting = { version = "VERSION", features = ["exit"] }
```

其中`VERSION` 是所需的版本。有关依赖项功能的更多信息，请参阅《Cargo手册》中的[`specifying dependencies`]部分。

[`specifying dependencies`]:https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html