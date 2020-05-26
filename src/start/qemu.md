# QEMU

我们现在开始为Cortex-M3微控制器[LM3S6965]编写程序。我们选择这个作为我们的最初目标是因为它可以使用QEMU [模拟](https://wiki.qemu.org/Documentation/Platforms/ARM#Supported_in_qemu-system-arm)，因此在本节中您无需关心硬件，只需专注于工具和开发过程。

[LM3S6965]:http://www.ti.com/product/LM3S6965

**重要**
在本教程中，我们将名称“app”用作项目名称。每当您看到“app”一词时，都应将其替换为自己的项目名称。或者您也可以直接将项目命名为“app”以避免替换。

## 创建一个非标准的Rust程序

我们将使用[`cortex-m-quickstart`]项目模板生成一个新项目。

[`cortex-m-quickstart`]:https://github.com/rust-embedded/cortex-m-quickstart

### 使用`cargo-generate`
首先安装cargo-generate


```console
cargo install cargo-generate
```

然后生成一个新项目

```console
cargo generate --git https://github.com/rust-embedded/cortex-m-quickstart
```


```text
 Project Name: app
 Creating project called `app`...
 Done! New project created /tmp/app
```
```console
cd app
```

### 使用git

克隆存储库

```console
git clone https://github.com/rust-embedded/cortex-m-quickstart app
cd app
```

然后将 `Cargo.toml` 中的`{{authors}}`,`{{project-name}}`替换为你自己的. 

```toml
[package]
authors = ["{{authors}}"] # "{{authors}}" -> "John Smith"
edition = "2018"
name = "{{project-name}}" # "{{project-name}}" -> "awesome-app"
version = "0.1.0"

# ..

[[bin]]
name = "{{project-name}}" # "{{project-name}}" -> "awesome-app"
test = false
bench = false
```

### 手工下载

获取`cortex-m-quickstart`模板的最新快照并解压缩它。

```console
curl -LO https://github.com/rust-embedded/cortex-m-quickstart/archive/master.zip
unzip master.zip
mv cortex-m-quickstart-master app
cd app
```

或者，您可以使用浏览器访问[`cortex-m-quickstart`]，单击绿色的“clone or download”按钮，然后单击“Download ZIP”。

然后按照[`使用git`](#使用git)一节中的第二部分中的操作，在Cargo.toml文件中填写自定义内容。

## 程序概述

为了方便起见，这是`src/main.rs`中源代码的最重要部分：

```rust , ignore
#![no_std]
#![no_main]

extern crate panic_halt;

use cortex_m_rt::entry;

#[entry]
fn main() -> ! {
    loop {
        // your code goes here
    }
}
```

该程序与标准Rust程序有点不同，因此让我们仔细看一下。

`#![no_std]`表示此程序*不会*链接到标准库,而是链接到其子集--核心库。

`#![no_main]`表示该程序将不使用大多数Rust程序使用的标准`main`接口。使用no_main的主要原因是在no_std上下文中使用main函数需要Rust的nightly版本。

`extern crate panic_halt;`。这个crate提供了一个 `panic_handler`，它定义了程序的恐慌行为。我们将在本书的[Panicking](panicking.md)一章中对此进行详细介绍。

[`#[entry]`][entry]是[`cortex-m-rt`]crate提供的属性，用于标记程序的入口。由于我们没有使用标准的“ main”接口，因此需要另一种方式来指示程序的入口，即 `#[entry]`。

[entry]:https://docs.rs/cortex-m-rt-macros/latest/cortex_m_rt_macros/attr.entry.html
[`cortex-m-rt`]:https://crates.io/crates/cortex-m-rt

注意main函数的签名是`fn main() -> !` ,因为我们的程序是目标硬件上唯一的程序，所以我们不希望它结束​​！我们使用[发散函数](https://doc.rust-lang.org/rust-by-example/fn/diverging.html)(函数签名中的`->！`表示没有返回值)来在编译时确保main不会结束。

## 交叉编译

下一步是针对Cortex-M3架构进行交叉编译。如果您知道编译目标($TRIPLE)应该是什么，那就直接运行`cargo build --target $TRIPLE`。不知道也没关系,模板项目中的.cargo/config里有答案：

```console
tail -n6 .cargo/config
```

```toml
[build]
# Pick ONE of these compilation targets
# target = "thumbv6m-none-eabi"    # Cortex-M0 and Cortex-M0+
target = "thumbv7m-none-eabi"    # Cortex-M3
# target = "thumbv7em-none-eabi"   # Cortex-M4 and Cortex-M7 (no FPU)
# target = "thumbv7em-none-eabihf" # Cortex-M4F and Cortex-M7F (with FPU)
```

为了针对Cortex-M3架构进行交叉编译，我们必须使用`thumbv7m-none-eabi`。该编译目标已设置为默认目标，因此以下两个命令具有相同的功能：

```console
cargo build --target thumbv7m-none-eabi
cargo build
```

## 检查

现在我们在`target/thumbv7m-none-eabi/debug/app`中有一个非本地的ELF二进制文件。我们可以使用`cargo-binutils`检查它。

使用`cargo-readobj`，我们可以打印ELF头以确认这是一个ARM二进制文件。

``` console
cargo readobj --bin app -- -file-headers
```

注意：
*`--bin app`是用于检查`target/$TRIPLE/debug/app`这个二进制文件
*`--bin app`还会在必要时(重新)编译二进制文件


``` text
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0x0
  Type:                              EXEC (Executable file)
  Machine:                           ARM
  Version:                           0x1
  Entry point address:               0x405
  Start of program headers:          52 (bytes into file)
  Start of section headers:          153204 (bytes into file)
  Flags:                             0x5000200
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         2
  Size of section headers:           40 (bytes)
  Number of section headers:         19
  Section header string table index: 18
```

`cargo-size`可以打印二进制文件的链接器部分的大小。

> **注意**此输出假定已经合并了[rust-embedded/cortex-m-rt#111](https://github.com/rust-embedded/cortex-m-rt/pull/111)这个PR
 

```console
cargo size --bin app --release -- -A
```

我们使用`--release`检查优化过的版本


``` text
app  :
section             size        addr
.vector_table       1024         0x0
.text                 92       0x400
.rodata                0       0x45c
.data                  0  0x20000000
.bss                   0  0x20000000
.debug_str          2958         0x0
.debug_loc            19         0x0
.debug_abbrev        567         0x0
.debug_info         4929         0x0
.debug_ranges         40         0x0
.debug_macinfo         1         0x0
.debug_pubnames     2035         0x0
.debug_pubtypes     1892         0x0
.ARM.attributes       46         0x0
.debug_frame         100         0x0
.debug_line          867         0x0
Total              14570
```

>关于ELF链接器部分的复习
>
>- `.text`包含程序代码
>- `.rodata`包含常量值，例如字符串
>- `.data`包含静态分配的变量，其初始值为非零
>- `.bss`包含静态分配的变量，其初始值为零
>- `.vector_table`是非标准部分,用于存储中断向量表
>- `.ARM.attributes`和`.debug_ *`部分包含元数据，这部分数据不会写入目标开发板的flash上。

**重要**：ELF文件包含诸如调试信息之类的元数据，因此它们在磁盘上的大小不会准确地反映程序在设备上真实占用的空间,因此应该总是使用`cargo-size`来检查二进制文件的真正大小。

`cargo-objdump` 可用于反汇编二进制文件。

```console
cargo objdump --bin app --release -- -disassemble -no-show-raw-insn -print-imm-hex
```

> **注意**此输出在您的系统上可能会有所不同。 不同版本的rustc，LLVM和库都会生成不同的指令。另外,由于空间问题,我们也对内容做了删减。

```text
app:  file format ELF32-arm-little

Disassembly of section .text:
main:
     400: bl  #0x256
     404: b #-0x4 <main+0x4>

Reset:
     406: bl  #0x24e
     40a: movw  r0, #0x0
     < .. truncated any more instructions .. >

DefaultHandler_:
     656: b #-0x4 <DefaultHandler_>

UsageFault:
     657: strb  r7, [r4, #0x3]

DefaultPreInit:
     658: bx  lr

__pre_init:
     659: strb  r7, [r0, #0x1]

__nop:
     65a: bx  lr

HardFaultTrampoline:
     65c: mrs r0, msp
     660: b #-0x2 <HardFault_>

HardFault_:
     662: b #-0x4 <HardFault_>

HardFault:
     663: <unknown>
```

## 运行

接下来，让我们看看如何在QEMU上运行嵌入式程序！这次，我们将使用`hello`示例。

为了方便起见，这是`examples/hello.rs`的源代码：

```rust , ignore
//! Prints "Hello, world!" on the host console using semihosting

#![no_main]
#![no_std]

extern crate panic_halt;

use cortex_m_rt::entry;
use cortex_m_semihosting::{debug, hprintln};

#[entry]
fn main() -> ! {
    hprintln!("Hello, world!").unwrap();

    // exit QEMU
    // NOTE do not run this on hardware; it can corrupt OpenOCD state
    debug::exit(debug::EXIT_SUCCESS);

    loop {}
}
```

该程序使用一种称为半主机(semihosting)的方式将文本打印到*主机*控制台。在使用实际硬件时，这需要调试会话支持，但是在使用QEMU时，直接使用就行了。

让我们从编译示例开始：

```console
cargo build --example hello
```

输出二进制文件将位于`target/thumbv7m-none-eabi/debug/examples/hello`。

要在QEMU上运行此二进制文件，请运行以下命令：

```console
qemu-system-arm \
  -cpu cortex-m3 \
  -machine lm3s6965evb \
  -nographic \
  -semihosting-config enable=on,target=native \
  -kernel target/thumbv7m-none-eabi/debug/examples/hello
```

```text
Hello, world!
```

打印文本后，该命令应成功退出退出代码为0。在*nix上，您可以使用以下命令进行检查：

```console
echo $?
```

```text
0
```

让我们分解一下QEMU命令：

-`qemu-system-arm` 这是QEMU仿真器。QEMU支持很多不同的架构。从名字可以看出,这是ARM处理器的完整仿真。

-`-cpu cortex-m3`。这告诉QEMU模拟Cortex-M3 CPU。指定CPU型号可以让我们捕获一些编译参数不当错误：例如，运行针对具有硬件FPU的Cortex-M4F编译的程序，QEMU将在其运行期间产生错误。

-`-machine lm3s6965evb`。这告诉QEMU模拟LM3S6965EVB，这是一个包含LM3S6965微控制器的开发板。

-`-nographic`。这告诉QEMU不要启动其GUI。

-`-semihosting-config(..)`。这告诉QEMU启用半主机。半主机使仿真设备可以使用主机stdout，stderr和stdin并在主机上创建文件。

-`-kernel $file`。这告诉QEMU在模拟机上加载并运行哪个二进制文件。

输入这么长的QEMU命令太麻烦了！我们可以设置一个自定义运行器以简化过程。`.cargo/config`有一行启动 QEMU的运行器被注释掉了,让我们去掉这行注释：

```console
head -n3 .cargo/config
```

```toml
[target.thumbv7m-none-eabi]
# uncomment this to make `cargo run` execute programs on QEMU
runner = "qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel"
```

该运行器仅适用于`thumbv7m-none-eabi`目标，这是我们的默认编译目标。现在直接运行`cargo run`就会编译程序并在QEMU上运行：

```console
cargo run --example hello --release
```

```text
   Compiling app v0.1.0 (file:///tmp/app)
    Finished release [optimized + debuginfo] target(s) in 0.26s
     Running `qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel target/thumbv7m-none-eabi/release/examples/hello`
Hello, world!
```

## 调试

调试对于嵌入式开发至关重要。让我们看看它是如何完成的。

调试嵌入式设备涉及远程调试，因为要调试的程序不会在运行调试器程序(GDB或LLDB)的计算机上运行。

远程调试涉及客户端和服务器。针对QEMU，客户端将是GDB(或LLDB)进程，而服务器将是运行嵌入式程序的QEMU进程。

在本节中，我们将使用已经编译的“hello”示例。

调试的第一步是在调试模式下启动QEMU：

```console
qemu-system-arm \
  -cpu cortex-m3 \
  -machine lm3s6965evb \
  -nographic \
  -semihosting-config enable=on,target=native \
  -gdb tcp::3333 \
  -S \
  -kernel target/thumbv7m-none-eabi/debug/examples/hello
```

此命令不会在控制台上显示任何内容，并且会阻塞终端。这次我们额外传递了两个参数：

- `-gdb tcp::3333`。这告诉QEMU监听TCP端口3333,等待GDB的连接。

- `-S` 这告诉QEMU在启动时冻结计算机。没有这个，可能我们还没有来得及启动调试器,程序就已经结束了！

接下来，我们在另一个终端中启动GDB，并告诉它加载示例的调试符号：


```console
gdb-multiarch -q target/thumbv7m-none-eabi/debug/examples/hello
```

**注意**：您可能需要其他版本的gdb而不是`gdb-multiarch`，具体取决于在安装一章中你安装的版本。也可能是`arm-none-eabi-gdb`或直接是`gdb`。

然后在GDB Shell中，我们连接到QEMU，它正在TCP端口3333上等待连接。

```console
target remote :3333
```

```text
Remote debugging using :3333
Reset () at $REGISTRY/cortex-m-rt-0.6.1/src/lib.rs:473
473     pub unsafe extern "C" fn Reset() -> ! {
```

您会看到该进程已停止，并且程序计数器指向了一个名为“Reset”的函数。那就是重启入口：即Cortex-M启动时执行程序的入口。

该函数最终将调用我们的main函数。让我们使用断点和`continue`命令一路跳过：

```console
break main
```

```text
Breakpoint 1 at 0x400: file examples/panic.rs, line 29.
```

```console
continue
```

```text
Continuing.

Breakpoint 1, main () at examples/hello.rs:17
17          let mut stdout = hio::hstdout().unwrap();
```

我们现在接近打印“ Hello，world！”的代码。让我们继续使用“ next”命令。

``` console
next
```

```text
18          writeln!(stdout, "Hello, world!").unwrap();
```

```console
next
```

```text
20          debug::exit(debug::EXIT_SUCCESS);
```

此时，您应该在运行`qemu-system-arm`的终端上看到"Hello, world!"。

```text
$ qemu-system-arm (..)
Hello, world!
```

再次调用`next`将终止QEMU过程。


```console
next
```

```text
[Inferior 1 (Remote target) exited normally]
```

现在，您可以退出GDB会话。

``` console
quit
```
