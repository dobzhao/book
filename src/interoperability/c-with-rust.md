# Rust中使用C代码

在Rust项目中使用C或C++代码包含两个主要部分：

- 封装导出的的C API以供Rust调用
- 构建要与Rust代码集成的C或C++代码

由于C++没有稳定的ABI，因此将Rust与C或C++结合使用时，建议使用`C` ABI。

## 定义接口

在Rust中使用C或C++代码之前，有必要定义(用Rust编写)这些代码中存在哪些数据类型和函数。在C或C++中使用这些代码时，您需要包含定义相关的头文件(“.h”或“.hpp”)。在Rust中，需要将这些头文件手动转换为Rust代码，或使用工具生成。

首先，我们将介绍如何将这些代码从C/C++手动转换为Rust。

### 封装C函数和数据类型

通常，用C或C++编写的库将提供头文件，该头文件定义公共接口中使用的所有类型和函数。比如下面的例子：

```C
/* File: cool.h */
typedef struct CoolStruct {
    int x;
    int y;
} CoolStruct;

void cool_function(int i, char c, CoolStruct* cs);
```

转换为Rust后，代码如下所示：

```rust , ignore
/* File: cool_bindings.rs */
#[repr(C)]
pub struct CoolStruct {
    pub x: cty::c_int,
    pub y: cty::c_int,
}

pub extern "C" fn cool_function(
    i: cty::c_int,
    c: cty::c_char,
    cs: *mut CoolStruct
);
```

让我们一次查看一个定义，以解释每个部分。


```rust , ignore
#[repr(C)]
pub struct CoolStruct { ... }
```


默认情况下，Rust不保证`struct`中包含的数据的顺序，填充或大小。为了保证与C代码的兼容性，我们加入了`#[repr(C)]` 属性，该属性指示Rust编译器使用C规则来组织结构体中的数据。


```rust , ignore
pub x: cty::c_int,
pub y: cty::c_int,
```

由于C/C++中`int`和`char`类型的灵活性，建议使用`cty`中定义的原始数据类型，它将原始类型从C映射到Rust中的类型。

```rust , ignore
pub extern "C" fn cool_function( ... );
```

该语句定义使用C ABI的函数的签名，称为“ cool_function”。这里只定义了签名，需要在其他位置提供此函数的定义，或者将其链接到相关的动态或者库文件中。

```rust , ignore
    i: cty::c_int,
    c: cty::c_char,
    cs: *mut CoolStruct
```

与上面的数据类型类似，我们使用C兼容的定义来定义函数参数的数据类型。为了清楚起见，我们还保留相同的参数名称。

我们这里有一种新类型，即`*mut CoolStruct`。由于C没有Rust引用的概念：`＆mut CoolStruct`，因此我们有一个裸指针。由于解引用此指针是“不安全的”，并且实际上该指针可能是“空”指针，因此在与C或C++代码进行交互时，必须小心确保Rust的典型保证。

### 自动生成接口

相比手动生成这些接口(可能很乏味且容易出错)，可以使用一种名为[bindgen]的工具来自动执行这些转换。有关[bindgen]用法的说明，请参阅[bindgen用户手册]，但是典型过程包括以下内容：

1. 收集所有要在Rust中使用的接口或数据类型的C或C++头文件
2. 编写一个“bindings.h”文件，其中的“ #include“ ...”`是您在第一步中收集的每个文件。
3. 将此“bindings.h”文件以及用于编译的所有编译标志提供给`bindgen`。注意使用``Builder.ctypes_prefix("cty")` /
  `--ctypes-prefix=cty`和`Builder.use_core()` ,这样生成的代码才能和`#![no_std]` 兼容。
4. `bindgen`将生成的Rust代码生成输出到终端。该输出可以通过管道重定向到文件，例如“ bindings.rs”。您可以在Rust项目中使用此文件与作为外部库编译和链接的C/C ++代码进行交互。提示：如果生成的绑定中的类型以`cty`作为前缀，请不要忘记使用[`cty`](https://crates.io/crates/cty)crate。

[bindgen]: https://github.com/rust-lang/rust-bindgen
[bindgen用户手册]: https://rust-lang.github.io/rust-bindgen/

## 构建C / C ++代码

由于Rust编译器不知道如何编译C或C++代码(或来自任何其他语言的代码，只要提供C接口即可)，因此有必要提前编译非Rust代码。

对于嵌入式项目，这通常意味着将C/C ++代码编译为静态归档文件(例如“cool-library.a”)，然后可以在最后的链接步骤将其与Rust代码合并。

如果您要使用的库已经作为静态库分发，则无需重新构建代码。只需像上面提到的转换接口文件，并在编译/链接时包含静态库文件。

如果您依赖的代码以源代码形式提供，则必须先用现有的构建系统(例如“ make”，“ CMake”等)编译,或者移植编译过程使用`cc` crate进行编译。对于这两种情况，都需要使用一个build.rs脚本。

### Rust`build.rs`构建脚本

`build.rs`脚本是用Rust语法编写的文件，该文件在编译机上执行，在构建完项目本身的依赖项之后，但在构建项目本身之前。

完整的参考资料可以在[这里](https://doc.rust-lang.org/cargo/reference/build-scripts.html)中找到。 `build.rs`脚本对于生成代码(例如通过[bindgen])，调用外部构建系统(例如Make)或通过使用`cc` crate直接编译C/C ++非常有用。

### 调用外部构建系统

对于复杂的项目，最简单的方法是使用[`std::process::Command`]遍历相对路径，调用固定命令(例如`make library`，然后将生成的静态库复制到`target`目录中的正确位置。

虽然你自己的项目以`no_std`嵌入式平台为目标，但是`build.rs`仅在执行编译的计算机上执行。这意味着您可以在`build.rs`中使用编译主机上的任何Rust crate。

### 使用`cc` crate构建C/C ++代码

对于不太复杂或者依赖较少的项目，或者难以修改构建系统以生成静态库(而不是最终的二进制文件或可执行文件)的项目，使用[`cc` crate]可能会更容易，它为主机提供的编译器封装了惯用的Rust接口。

[`cc` crate]:https://github.com/alexcrichton/cc-rs


对于只有一个c文件的静态库的最简单情况,下面给出一个使用[`cc` crate]的示例:

```rust , ignore
extern crate cc;

fn main() {
    cc::Build::new()
        .file("foo.c")
        .compile("libfoo.a");
}
```