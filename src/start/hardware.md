# 硬件

现在，您应该对工具和开发过程有所了解。在本节中，我们将切换到实际硬件,该过程将基本保持不变,让我们开始吧。

## 了解您的硬件

在我们开始之前，您需要确定目标设备的一些特征，因为这些特征将用于配置项目：

- ARM内核。例如Cortex-M3。

- ARM内核是否包括FPU？ Cortex-M4**F**和Cortex-M7**F**内核都有FPU。

-目标设备有多少闪存和RAM？例如256 KiB的闪存和32 KiB的RAM。

-闪存和RAM映射的地址空间在哪里？例如RAM是通常位于地址“0x2000_0000”。

通常您可以在数据手册或设备的参考手册中找到这些信息。

在本节中，我们将使用我们的参考硬件STM32F3DISCOVERY。该开发板包含STM32F303VCT6微控制器。该微控制器具有：

- 一个Cortex-M4F内核，其中包括一个单精度FPU

- 闪存的256 KiB位于地址0x0800_0000。

- 位于地址0x2000_0000的40KiBRAM。 (还有另一个RAM区域，为简单起见，我们将其忽略)。

## 配置

我们将从一个新的模板实例开始。如果没有`cargo-generate`工具,请参阅[上一小节的QEMU]。

[上一小节的QEMU]:qemu.md

``` console
$ cargo generate --git https://github.com/rust-embedded/cortex-m-quickstart
 Project Name: app
 Creating project called `app`...
 Done! New project created /tmp/app

 $ cd app
```

第一个步是在.cargo/config中设置默认的编译目标。

``` console
$ tail -n5 .cargo/config
```

``` toml
# Pick ONE of these compilation targets
# target = "thumbv6m-none-eabi"    # Cortex-M0 and Cortex-M0+
# target = "thumbv7m-none-eabi"    # Cortex-M3
# target = "thumbv7em-none-eabi"   # Cortex-M4 and Cortex-M7 (no FPU)
target = "thumbv7em-none-eabihf" # Cortex-M4F and Cortex-M7F (with FPU)
```


我们将使用`thumbv7em-none-eabihf`，因为它适合Cortex-M4F内核。

第二步是将存储区域信息输入到“memory.x”文件中。

``` console
$ cat memory.x
/* Linker script for the STM32F303VCT6 */
MEMORY
{
  /* NOTE 1 K = 1 KiBi = 1024 bytes */
  FLASH : ORIGIN = 0x08000000, LENGTH = 256K
  RAM : ORIGIN = 0x20000000, LENGTH = 40K
}
```

确保`debug::exit()`调用已被注释掉或删除，因为他仅用于在QEMU中运行。

```rust , ignore
#[entry]
fn main() -> ! {
    hprintln!("Hello, world!").unwrap();

    // exit QEMU
    // NOTE do not run this on hardware; it can corrupt OpenOCD state
    // debug::exit(debug::EXIT_SUCCESS);

    loop {}
}
```

现在，您可以像以前一样使用`cargo build`交叉编译程序，并使用`cargo-binutils`检查二进制文件。 `cortex-m-rt` crate可处理使您的芯片运行所需的所有魔术,几乎所有Cortex-M CPU都以相同的方式引导。

``` console
$ cargo build --example hello
```

## 调试

调试看起来会有所不同。实际上，根据目标设备的不同，第一步看起来可能会有所不同。在本节中，我们将介绍在STM32F3DISCOVERY上调试程序所需的步骤。有关设备的特定信息，请查看[Debugonomicon](https://github.com/rust-embedded/debugonomicon)。

和以前一样，我们将进行远程调试，客户端是GDB进程,服务器将是OpenOCD。

$ cat openocd.cfg
按照[验证]部分的操作，将开发板连接到笔记本电脑或者PC，并检查是否填充了ST-LINK接头连接器(todo ... check that the ST-LINK header is populated)。

[验证]: ../intro/install/verify.md

在终端上，运行“openocd”以连接到开发板上的ST-LINK。从模板的根目录运行此命令；`openocd`会根据`openocd.cfg`文件，找到要使用的接口文件和目标文件。

``` console
$ cat openocd.cfg
```

``` text
# Sample OpenOCD configuration for the STM32F3DISCOVERY development board

# Depending on the hardware revision you got you'll have to pick ONE of these
# interfaces. At any time only one interface should be commented out.

# Revision C (newer revision)
source [find interface/stlink-v2-1.cfg]

# Revision A and B (older revisions)
# source [find interface/stlink-v2.cfg]

source [find target/stm32f3x.cfg]
```

> **注意**如果您在[验证]部分发现开发板的版本较旧，则此时应修改`openocd.cfg`文件以使用`interface/stlink-v2.cfg`。

``` console
$ openocd
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
Info : Target voltage: 2.913879
Info : stm32f3x.cpu: hardware has 6 breakpoints, 4 watchpoints
```

在另一个终端上，也从模板的根目录运行GDB。

``` console
$ <gdb> -q target/thumbv7em-none-eabihf/debug/examples/hello
```

接下来，将GDB连接到OpenOCD，OpenOCD正在监听端口3333,等待新的TCP连接。

``` console
(gdb) target remote :3333
Remote debugging using :3333
0x00000000 in ?? ()
```

现在，使用`load`命令将程序加载到微控制器上。

``` console
(gdb) load
Loading section .vector_table, size 0x400 lma 0x8000000
Loading section .text, size 0x1e70 lma 0x8000400
Loading section .rodata, size 0x61c lma 0x8002270
Start address 0x800144e, load size 10380
Transfer rate: 17 KB/sec, 3460 bytes/write.
```

现在程序已加载。该程序使用半主机，因此在进行任何半主机调用之前，我们必须告诉OpenOCD启用半主机。您可以使用“ monitor”将命令发送到OpenOCD。

``` console
(gdb) monitor arm semihosting enable
semihosting is enabled
```

>您可以通过调用`monitor help`命令来查看所有OpenOCD命令。

像之前一样，我们可以使用断点和`continue`跳过所有跳转到`main`函数。

``` console
(gdb) break main
Breakpoint 1 at 0x8000d18: file examples/hello.rs, line 15.

(gdb) continue
Continuing.
Note: automatically using hardware breakpoints for read-only addresses.

Breakpoint 1, main () at examples/hello.rs:15
15          let mut stdout = hio::hstdout().unwrap();
```

> **注意**如果执行`continue`命令后GDB阻塞了终端而不是停在了断点上，则可能需要仔细检查`memory.x`文件中的内存区域信息是否配置正确(起始地址和长度)。

用`next`命令替代刚刚的`continue`,应该也会产生相同的结果。


``` console
(gdb) next
16          writeln!(stdout, "Hello, world!").unwrap();

(gdb) next
19          debug::exit(debug::EXIT_SUCCESS);
```

此时，您应该看到"Hello, world!" 打印在OpenOCD控制台上，等等。


``` console
$ openocd
(..)
Info : halted: PC: 0x08000e6c
Hello, world!
Info : halted: PC: 0x08000d62
Info : halted: PC: 0x08000d64
Info : halted: PC: 0x08000d66
Info : halted: PC: 0x08000d6a
Info : halted: PC: 0x08000a0c
Info : halted: PC: 0x08000d70
Info : halted: PC: 0x08000d72
```

发出另一个`next`将使处理器执行`debug::exit`。这充当断点并中止该过程：

``` console
(gdb) next

Program received signal SIGTRAP, Trace/breakpoint trap.
0x0800141a in __syscall ()
```

OpenOCD控制台将会打印如下内容：

``` console
$ openocd
(..)
Info : halted: PC: 0x08001188
semihosting: *** application exited ***
Warn : target not halted
Warn : target not halted
target halted due to breakpoint, current mode: Thread
xPSR: 0x21000000 pc: 0x08000d76 msp: 0x20009fc0, semihosting
```

但是，在微控制器上运行的进程尚未终止，您可以使用`continue`或类似命令将其恢复。

现在，您可以使用“ quit”命令退出GDB。

``` console
(gdb) quit
```

现在调试需要更多步骤，因此我们将所有这些步骤打包到一个名为`openocd.gdb`的GDB脚本中。


``` console
$ cat openocd.gdb
```

``` text
target remote :3333

# print demangled symbols
set print asm-demangle on

# detect unhandled exceptions, hard faults and panics
break DefaultHandler
break HardFault
break rust_begin_unwind

monitor arm semihosting enable

load

# start the process but immediately halt the processor
stepi
```

现在运行 `<gdb> -x openocd.gdb $program`将立即将GDB连接到OpenOCD，启用半主机，加载程序并启动该过程。

您也可以将`<gdb> -x openocd.gdb`转换为自定义运行器，这样`cargo run`会自动构建程序并开始GDB会话。该运行器已包含在`.cargo/config`中，只不过现在是被注释掉的状态。

``` console
$ head -n10 .cargo/config
```


``` toml
[target.thumbv7m-none-eabi]
# uncomment this to make `cargo run` execute programs on QEMU
# runner = "qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel"

[target.'cfg(all(target_arch = "arm", target_os = "none"))']
# uncomment ONE of these three option to make `cargo run` start a GDB session
# which option to pick depends on your system
runner = "arm-none-eabi-gdb -x openocd.gdb"
# runner = "gdb-multiarch -x openocd.gdb"
# runner = "gdb -x openocd.gdb"
```

``` console
$ cargo run --example hello
(..)
Loading section .vector_table, size 0x400 lma 0x8000000
Loading section .text, size 0x1e70 lma 0x8000400
Loading section .rodata, size 0x61c lma 0x8002270
Start address 0x800144e, load size 10380
Transfer rate: 17 KB/sec, 3460 bytes/write.
(gdb)
```