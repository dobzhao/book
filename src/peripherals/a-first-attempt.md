# 初试Rust

## 寄存器

让我们看一下`SysTick`外设(每个Cortex-M处理器内核随附的简单计时器)。通常，您会在芯片制造商的《技术参考手册》中查找这些信息，但是此示例对于所有ARM Cortex-M内核都是通用的，因此也可以在[ARM参考手册]中进行查到。我们看到有四个寄存器：

[ARM参考手册]:http://infocenter.arm.com/help/topic/com.arm.doc.dui0553a/Babieigh.html

|偏移|名称|描述|位宽|
| -------- | ------------- | -------------------------- --- | -------- |
| 0x00 | SYST_CSR |控制和状态寄存器| 32位|
| 0x04 | SYST_RVR |重新加载值寄存器| 32位|
| 0x08 | SYST_CVR |当前值寄存器| 32位|
| 0x0C | SYST_CALIB |校准值寄存器| 32位|

## C方法

在Rust中，我们可以用像C语言一样使用`struct`表示一系列寄存器。

```rust , ignore
#[repr(C)]
struct SysTick {
    pub csr: u32,
    pub rvr: u32,
    pub cvr: u32,
    pub calib: u32,
}
```

限定符`#[repr(C)]`告诉Rust编译器像C编译器那样布局此结构体。这非常重要，因为Rust允许对结构体字段进行重新排序，而C不允许。您可以想象如果编译器以静默方式重新排列了这些字段，我们调试起来会有多困难！有了此限定符后，我们就有四个32位字段，它们与上表相对应。但是，当然，这个 `struct` 本身是没有用的-我们需要一个变量。

```rust , ignore
let systick = 0xE000_E010 as *mut SysTick;
let time = unsafe { (*systick).cvr };
```

## 易失性访问

现在，上述方法存在以下问题:

1. 每次访问外设时，我们都必须使用unsafe关键字。
2. 我们无法指定哪些寄存器是只读或读写寄存器。
3. 程序中任何地方的任何代码段都可以通过这种结构访问硬件。
4. 最重要的是，它实际上不起作用...

现在的问题是编译器很聪明。如果您对同一块RAM紧挨着进行两次写入，则编译器会注意到这一点，并且会跳过第一次写入。在C语言中，我们可以将变量标记为`volatile`，以确保每次读取或写入均按预期进行。在Rust中，我们则是将**访问本身**标记为volatile，而不是变量。


```rust , ignore
let systick = unsafe { &mut *(0xE000_E010 as *mut SysTick) };
let time = unsafe { core::ptr::read_volatile(&mut systick.cvr) };
```

现在，我们已经解决了四个问题之一，但是现在我们有了更多的`unsafe`代码！幸运的是，第三方crate[ʻvolatile_register`]可以提供帮助。

[`volatile_register`]:https：//crates.io/crates/volatile_register

```rust , ignore
use volatile_register::{RW, RO};

#[repr(C)]
struct SysTick {
    pub csr: RW<u32>,
    pub rvr: RW<u32>,
    pub cvr: RW<u32>,
    pub calib: RO<u32>,
}

fn get_systick() -> &'static mut SysTick {
    unsafe { &mut *(0xE000_E010 as *mut SysTick) }
}

fn get_time() -> u32 {
    let systick = get_systick();
    systick.cvr.read()
}
```

现在，通过`read`和`write`方法会自动执行易失性(volatile)访问。但是执行写入仍然是`unsafe`，公平地说，硬件是一堆易变的状态，编译器无法知道这些写入是否实际上是安全的，因此这是一个很好的默认设置。

## Rust封装

我们需要将此`struct`封装到一个更高层API中，以使我们的用户可以安全地调用它。作为驱动程序开发者，我们手动验证不安全的代码是否正确，然后为我们的用户提供一个安全的API，以便他们不必担心它(只要他们相信我们是正确的!)。

一个示例可能是：

```rust , ignore
use volatile_register::{RW, RO};

pub struct SystemTimer {
    p: &'static mut RegisterBlock
}

#[repr(C)]
struct RegisterBlock {
    pub csr: RW<u32>,
    pub rvr: RW<u32>,
    pub cvr: RW<u32>,
    pub calib: RO<u32>,
}

impl SystemTimer {
    pub fn new() -> SystemTimer {
        SystemTimer {
            p: unsafe { &mut *(0xE000_E010 as *mut RegisterBlock) }
        }
    }

    pub fn get_time(&self) -> u32 {
        self.p.cvr.read()
    }

    pub fn set_reload(&mut self, reload_value: u32) {
        unsafe { self.p.rvr.write(reload_value) }
    }
}

pub fn example_usage() -> String {
    let mut st = SystemTimer::new();
    st.set_reload(0x00FF_FFFF);
    format!("Time is now 0x{:08x}", st.get_time())
}
```

但是这种方法的问题在于以下有问题的代码可以正常编译：

```rust , ignore
fn thread1() {
    let mut st = SystemTimer::new();
    st.set_reload(2000);
}

fn thread2() {
    let mut st = SystemTimer::new();
    st.set_reload(1000);
}
```

set_reload函数的`＆mut self`参数确保没有其他对这个特定的`SystemTimer`实例的引用，但是它不会阻止用户创建第二个`SystemTimer`实例，明显它们指向完全相同的外设！ 当然如果程序员很努力的避免创建多个实例,则以这种方式编写的代码也可以工作，但是一旦代码分散到不同模块，不同驱动程序，由多个程序员维护,则难免会出现各种错误.。