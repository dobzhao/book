# C中使用Rust代码

在C或C++项目中使用Rust代码主要包括两部分。

- 在Rust中创建C友好的API
- 将Rust项目嵌入到外部构建系统中

除了`cargo`和`meson`外，大多数构建系统没有本地Rust支持。因此，最有可能最好是使用`cargo`来编译crate和所有依赖项。

## 建立一个项目

照常创建一个新的`cargo`项目。

通能参数可以告诉`cargo`生成一个系统库crate，而不是常规的Rust项目。您也可以为库设置不同的输出名称。

```toml
[lib]
name = "your_crate"
crate-type = ["cdylib"]      # Creates dynamic lib
# crate-type = ["staticlib"] # Creates static lib
```

## 构建`C` API

由于C++没有稳定的ABI，因此我们将`C`用于不同语言之间的任何互操作。在C和C++代码中使用Rust时也不例外。

### `# [no_mangle]`

Rust编译器处理符号名称的方式与c语言链接器期望的方式不同。因此，需要告知Rust编译器不要对要在Rust之外使用的任何函数进行改动。

### `extern“ C”`

默认情况下，您在Rust中编写的任何函数都将使用Rust ABI(它也是不稳定的)。而当构建FFI API时，我们需要告诉编译器使用系统ABI。

根据您的平台，您可能要特定​​的ABI版本，这些在[此处](https://doc.rust-lang.org/reference/items/external-blocks.html)中进行了说明。

---

将刚刚的内容总结在一起，您将获得一个大致如下所示的函数。

```rust , ignore
#[no_mangle]
pub extern "C" fn rust_function() {

}
```

就像在Rust项目中使用C代码一样，您现在需要将数据转换成其他应用程序可以理解的格式。

## 链接和更大的项目上下文。

因此，现在只是解决了问题的一半。您现在如何使用它？

**这在很大程度上取决于您的项目和/或构建系统**

`cargo`将根据您的平台和设置创建一个`my_lib.so`/`my_lib.dll` / `my_lib.a` 文件。该库可以直接由您的构建系统链接。

但是，从C调用Rust函数需要一个头文件来声明函数签名。

Rust-ffi API中的每个函数都需要具有相应的函数声明。

```rust , ignore
#[no_mangle]
pub extern "C" fn rust_function() {}
```
需要这样一个声明:

```C
void rust_function();
```
 
有一个工具可以自动执行此过程，称为[cbindgen]，它可以分析Rust代码，然后从中生成C和C++项目的头文件。

[cbindgen]:https://github.com/eqrion/cbindgen

至此，在C语言中调用Rust函数只需添加头文件并调用它们！


```C
#include "my-rust-project.h"
rust_function();
```
