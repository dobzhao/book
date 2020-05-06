# 单例

>在软件工程中，单例模式是一种软件设计模式，它限制类只有一个实例。
>
> *维基百科：[单例模式] *

[单例模式]:https：//en.wikipedia.org/wiki/Singleton_pattern


## 为什么我们不能直接使用全局变量？

我们可以像这样将所有内容设为公共静态


```rust , ignore
static mut THE_SERIAL_PORT: SerialPort = SerialPort;

fn main() {
    let _ = unsafe {
        THE_SERIAL_PORT.read_speed();
    };
}
```


但这有一些问题。它是一个可变的全局变量，在Rust中，与它们进行交互总是不安全的。这些变量在整个程序中也是可见的，这意味着借用检查器无法帮助您跟踪这些变量的引用和所有权。

## 我们如何在Rust中做到这一点？

我们不是简单地将外设设为全局变量，而是创建一个全局变量，姑且称为“PERIPHERALS”，其中每个外围设备都包含一个“Option <T>”。

```rust , ignore
struct Peripherals {
    serial: Option<SerialPort>,
}
impl Peripherals {
    fn take_serial(&mut self) -> SerialPort {
        let p = replace(&mut self.serial, None);
        p.unwrap()
    }
}
static mut PERIPHERALS: Peripherals = Peripherals {
    serial: Some(SerialPort),
};
```

这种结构使我们可以获得外围设备的单个实例。如果我们尝试多次调用`take_serial()`，代码将会崩溃！

```rust , ignore
fn main() {
    let serial_1 = unsafe { PERIPHERALS.take_serial() };
    // This panics!
    // let serial_2 = unsafe { PERIPHERALS.take_serial() };
}
```

尽管与此结构进行交互是`unsafe`，但一旦取得了它内部的“SerialPort”，我们将不再需要使用`unsafe`或`PERIPHERALS`结构体。

这具有很小的运行时开销，因为我们必须将`SerialPort`结构包装在一个Option中，并且需要调用一次`take_serial()`，但是，这笔小小的前期成本使我们能够在其余所有过程中利用借用检查器检查我们的程序。

## 现有库支持

尽管我们在上面创建了自己的`Peripherals`结构体，但实际上你的代码中无需这么操作。 `cortex_m`crate包含一个名为`singleton!()`的宏，它将为您执行此操作。


```rust , ignore
#[macro_use(singleton)]
extern crate cortex_m;

fn main() {
    // OK if `main` is executed only once
    let x: &'static mut bool =
        singleton!(: bool = false).unwrap();
}
```

[cortex_m docs](https://docs.rs/cortex-m/latest/cortex_m/macro.singleton.html)

此外，如果您使用`cortex-m-rtfm`， 定义和获取这些外围设备的整个过程已经帮您封装好了，您将获得一个`Peripherals`结构，该结构包含非`Option <T>`版本的您定义的所有项目。

```rust , ignore
// cortex-m-rtfm v0.3.x
app! {
    resources: {
        static RX: Rx<USART1>;
        static TX: Tx<USART1>;
    }
}
fn init(p: init::Peripherals) -> init::LateResources {
    // Note that this is now an owned value, not a reference
    let usart1: USART1 = p.device.USART1;
}
```

[japaric.io rtfm v3](https://blog.japaric.io/rtfm-v3/)

##  但为什么？

但是这些单例化能产生什么显著不同?

```rust , ignore
impl SerialPort {
    const SER_PORT_SPEED_REG: *mut u32 = 0x4000_1000 as _;

    fn read_speed(
        &self // <------ This is really, really important
    ) -> u32 {
        unsafe {
            ptr::read_volatile(Self::SER_PORT_SPEED_REG)
        }
    }
}
```

这里有两个重要因素：

* 因为我们使用的是单例，所以只有一种方法可以获得`SerialPort`实例
* 要调用`read_speed()`方法，我们必须对`SerialPort`实例拥有借用或者所有权

这两个因素放在一起，再加上只有满足借用检查器的情况下，才可以访问硬件，这意味着我们绝对不会对同一硬件有多个可变引用！


```rust , ignore
fn main() {
    // missing reference to `self`! Won't work.
    // SerialPort::read_speed();

    let serial_1 = unsafe { PERIPHERALS.take_serial() };

    // you can only read what you have access to
    let _ = serial_1.read_speed();
}
```

## 将您的硬件视为数据

此外，由于某些引用是可变的，而有些则是不可变的，因此通过函数签名就可以判断是否可能潜在地修改硬件的状态。例如，

下面这个函数允许更改硬件设置：

```rust , ignore
fn setup_spi_port(
    spi: &mut SpiPort,
    cs_pin: &mut GpioPin
) -> Result<()> {
    // ...
}
```

下面这个则不可以：

```rust , ignore
fn read_button(gpio: &GpioPin) -> bool {
    // ...
}
```

这使我们能够在编译时(而不是在运行时)限制代码是否应该更改硬件。需要注意的是，这通常仅适用于单个应用程序，但是对于裸机系统，我们的软件将被编译到单个应用程序中，因此不是问题。(这里说的是如果存在多个进程,它们可以分别构建单例,但是实际上外设只有一个,还是不安全)