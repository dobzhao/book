# 互操作性

Rust与C代码之间的互操作性始终取决于两种语言之间的数据转换。为此，在`stdlib`中有两个专用模块称为[`std::ffi`](https://doc.rust-lang.org/std/ffi/index.html)和[`std::os::raw`](https://doc.rust-lang.org/std/os/raw/index.html)。

`std::os::raw`处理可以由编译器隐式转换的低级基本类型，因为Rust和C之间的内存布局足够相似或相同。

`std::ffi` 提供了一些实用程序，用于转换更复杂的类型(例如字符串)，将`&str`和`String`都映射到更易于处理和更安全的C类型。

这两个模块都不在`core`中，但您可以在[`cstr_core`]crate中找到支持`#![no_std]`的`std::ffi::{CStr,CString}`，`std::os::raw`中的大多数类型可以在[`cty`] crate中找到。

[`cstr_core`]:https://crates.io/crates/cstr_core
[`cty`]:https://crates.io/crates/cty

|Rust类型| 中间类型|C类型 |
| ------------ | -------------- | -------------- |
| String     | CString      | *char        |
| &str       | CStr         | *const char  |
| ()         | c_void       | void         |
| u32 or u64 | c_uint       | unsigned int |
| etc        | ...          | ...          |


如上所述，基本类型可以由编译器隐式转换。

```rust , ignore
unsafe fn foo(num: u32) {
    let c_num: c_uint = num;
    let r_num: u32 = c_num;
}
```

## 与其他构建系统的互操作性

嵌入式项目中经常会碰到需要Cargo与现有的构建系统(例如make或cmake)结合的情况。

 [问题# 61]上有我们收集的示例。

[问题# 61]:https://github.com/rust-embedded/book/issues/61


## 与RTOS的互操作性

将Rust与FreeRTOS或ChibiOS等RTOS集成仍在进行中。特别是从Rust调用RTOS函数可能很棘手。

[问题# 62]上有我们收集的示例。

[问题# 62]:https://github.com/rust-embedded/book/issues/62