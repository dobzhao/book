# 零成本抽象

类型状态机也是零成本抽象的一个很好的例子--能够将某些需要运行时执行和检查的行为提前到编译时。这些类型状态不包含实际数据，而是用作标记。由于它们不包含任何数据，因此它们在运行时不占用额外的内存空间：

```rust , ignore
use core::mem::size_of;

let _ = size_of::<Enabled>();    // == 0
let _ = size_of::<Input>();      // == 0
let _ = size_of::<PulledHigh>(); // == 0
let _ = size_of::<GpioConfig<Enabled, Input, PulledHigh>>(); // == 0
```

## 零大小类型

```rust , ignore
struct Enabled;
```

像这样定义的结构称为零大小类型，因为它们不包含实际数据。尽管这些类型在编译时表现为“真实”-您可以复制，移动它们，引用它们等，但是编译器在优化后就会像不存在一样。

在这段代码中：


```rust , ignore
pub fn into_input_high_z(self) -> GpioConfig<Enabled, Input, HighZ> {
    self.periph.modify(|_r, w| w.input_mode().high_z());
    GpioConfig {
        periph: self.periph,
        enabled: Enabled,
        direction: Input,
        mode: HighZ,
    }
}
```

我们返回的GpioConfig在运行时永远不会存在。调用此函数实际上就是一条汇编指令-将一个常量写入到寄存器中。这意味着我们开发的类型状态机接口是一种零成本的抽象方法(zero cost abstraction)-它不需要使用CPU，RAM或代码空间来跟踪`GpioConfig`的状态，最终优化后与手写的直接写寄存器的代码相同。

## 嵌套

通常，这些抽象对象可以任意嵌套,只要使用的所有对象都是零大小的类型，整个结构体在运行时就不会存在。

对于复杂或深度嵌套的结构，定义状态的所有可能组合会很繁琐, 这时可以借助宏生成所有的状态。