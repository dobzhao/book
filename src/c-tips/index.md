# 嵌入式C开发人员的技巧

本章收集了各种技巧，这些技巧对于希望开始编写Rust的经验丰富的嵌入式C开发人员可能有用。它特别强调了您可能已经在C语言中习惯的事情在Rust中的不同之处。

## 预处理器

在C语言中，预处理器有多种用途，例如：

* #ifdef在编译时选择代码块
* 编译时数组大小和计算
* 宏可简化常见模式(避免函数调用开销)

Rust没有预处理器，因此许多用例的处理方式有所不同。在本节的其余部分，我们将介绍预处理器的各种替代方法。

### 编译时代码选择

在Rust中，与#ifdef ... #endif最接近的匹配项是[Cargo features]。这比C预处理器更加正式：每个crate都明确列出了所有可能的特性(features)，并且只能打开或关闭。当您将一个crate作为依赖项列出时,特性已经被打开：如果您的依赖关系树中的任何crate为另一个板条箱启用了某个特性，则该crate的这个特性在所有的crate中都会启用。

[Cargo features]:https://doc.rust-lang.org/cargo/reference/manifest.html#the-features-section

例如，您要实现一个提供信号处理原语的crate,你想避免每个人都编译或者声明一个巨大的常量表。您可以在`Cargo.toml`中为每个组件声明一个Cargo特性：

```toml
[features]
FIR = []
IIR = []
```

然后，在您的代码中使用`#[cfg(feature="FIR")]`来控制包含的内容。


```rust
/// In your top-level lib.rs

#[cfg(feature="FIR")]
pub mod fir;

#[cfg(feature="IIR")]
pub mod iir;
```


同样包含某个代码块的条件可以是只有某个特性未启用,或者某些特性组合启用或者未启用。

另外，Rust提供了许多可以自动使用的条件，例如`target_arch`可以根据架构选择不同的代码。有关条件编译支持的完整详细信息，请参见Rust手册的[条件编译]一章。

[条件编译]:https://doc.rust-lang.org/reference/conditional-compilation.html

条件编译仅适用于下一条语句或块。如果是多条语句或者多个代码块，那么`cfg`属性需要多次使用。值得注意的是，在大多数情况下，包含所有代码并允许编译器在优化时删除无效代码会更好：对于您和您的用户来说更简单，并且通常来说，编译器会很好地删除未使用的代码。

### 编译时大小和计算
Rust支持`const fn`，这些函数保证在编译时可以求值，因此可以在需要常量的地方使用，例如数组大小。可以与上述功能一起使用，例如：

```rust
const fn array_size() -> usize {
    #[cfg(feature="use_more_ram")]
    { 1024 }
    #[cfg(not(feature="use_more_ram"))]
    { 128 }
}

static BUF: [u32; array_size()] = [0u32; array_size()];
```

这些新特性刚刚在Rust 1.31版本稳定下来，因此文档仍然很少。在编写本文时，`const fn`可用的功能非常有限。在将来的Rust版本中，有望扩展`const fn`允许的范围。

### 宏

Rust提供了非常强大的[宏系统]。相比C预处理器几乎直接在源代码的文本上运行，Rust宏系统在更高级别上运行。 Rust宏有两种类型：声明宏和过程宏。前者更简单也最常见；宏看起来像函数调用，并且可以扩展为完整的表达式，语句，项目或模式。过程宏更加复杂，但是功能也更强大,它可以将任意Rust语法转换为新的Rust语法。

[宏系统]:https://doc.rust-lang.org/book/ch19-06-macros.html

通常，在使用C预处理器宏的地方，您可以试试用声明宏来替代。它们可以在您的crate中定义，既可以自己使用,也可以导出让其他crate使用。请注意，由于它们必须扩展为完整的表达式，语句，项目或模式，因此某些C预处理器宏的用例将无法替代，例如，宏展开后是变量名的一部分或list的部分子集。

与Cargo特性一样，是否需要宏也值得考虑。在许多情况下，常规函数更易于理解，并且内联可以起到与宏相同的效果。 `#[inline]`和`#[inline(always)]` [属性]可以更准确的控制是否内联，此处也应格外小心--编译器会在适当的情况下自动内联同一crate中的函数，因此，强迫它执行不当操作可能会导致性能下降。

[属性]:https://doc.rust-lang.org/reference/attributes.html#inline-attribute

解释整个Rust宏系统超出了本书的范围，因此，建议您查阅Rust文档以获取全部详细信息。

## 构建系统

大多数Rust crate都是使用Cargo构建的(尽管不是必需的)。这可以解决传统构建系统中的许多难题。但是，您可能希望自定义构建过程。 Cargo为此提供了[build.rs脚本]。它们是Rust脚本，可以根据需要与Cargo构建系统进行交互。

[build.rs脚本]:https://doc.rust-lang.org/cargo/reference/build-scripts.html

构建脚本的常见用例包括：

* 提供构建时信息，例如将构建日期或Git commit哈希静态嵌入到可执行文件中
* 在构建时根据所选功能或其他逻辑生成链接脚本
* 更改Cargo构建配置
* 添加额外的静态链接库

当前，不支持构建后脚本，传统上您可能会使用这些脚本来完成诸如从构建对象自动生成二进制文件或打印构建信息之类的任务。

### 交叉编译

将Cargo用于您的构建系统还可以简化交叉编译。在大多数情况下，只需告诉Cargo `--target thumbv6m-none-eabi`，就可以在`target/thumbv6m-none-eabi/debug/myapp`中找到生成的可执行文件。

对于Rust不直接支持的平台，您将需要自己构建`libcore`。在这样的平台上，[Xargo]可用作Cargo的替代品，它会自动为您构建`libcore`。

[Xargo]: https://github.com/japaric/xargo

## 迭代器与数组

在C语言中，您可能习惯于通过数组的索引直接访问数组：

```c
int16_t arr[16];
int i;
for(i=0; i<sizeof(arr)/sizeof(arr[0]); i++) {
    process(arr[i]);
}
```

在Rust中，这是一种反模式：索引访问可能较慢(因为需要对边界进行检查)，并且可能阻止各种编译器优化。这是一个重要的区别，值得重复：Rust将检查数组的越界访问，以确保内存安全，而C将愉快地接受越界访问。

所以，请使用迭代器：

```rust , ignore
let arr = [0u16; 16];
for element in arr.iter() {
    process(*element);
}
```

迭代器提供了一系列强大的在C中必须手动实现的功能，例如链式调用，zip，枚举，查找最小值或最大值，求和等等。迭代器方法可以链式调用以提高代码的可读性。

有关更多详细信息，请参见[Rust book中的迭代器]和[迭代器文档]。

[Rust book中的迭代器]:https://doc.rust-lang.org/book/ch13-02-iterators.html
[迭代器文档]:https://doc.rust-lang.org/core/iter/trait.Iterator.html

## 引用与指针

在Rust中，指针(称为[裸指针])仅在特定情况下使用，因为对它们的解引用始终被认为是“不安全的”(`unsafe`)-Rust无法为其指向的内容提供通常的保证。

[裸指针]:https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html#dereferencing-a-raw-pointer

在大多数情况下，我们改为使用由`&`符号表示的引用或由`&mut`符号表示的可变引用。引用的行为与指针相似，因为它们可以被解引用以访问指向的值，但是它们是Rust所有权系统的关键部分：Rust严格要求您任何时候只能拥有一个可变引用或多个非可变引用。

在实践中，这意味着您必须更加小心是否需要对数据进行可变访问：在C中，默认值是可变的，而对于`const`则必须明确，在Rust中则相反。

有一种情况,您可能只能使用裸指针,那就是与硬件直接打交道(例如，将指向缓冲区的指针写入DMA外设寄存器). 并且所有外设访问crate底层用得也是裸指针，从而可以读写内存映射寄存器。

## 易失性(volatile)访问

在C语言中，各个变量可以标记为 `volatile`，告诉编译器变量中的值可能在两次访问之间改变。`volatile`变量通常在嵌入式系统中用于内存映射寄存器的访问。

在Rust中，我们不是使用volatile来标记变量，而是使用特定的方法来实现volatile访问：[`core::ptr::read_volatile`]和[`core::ptr::write_volatile`]。这些方法使用`*const T` 或`*mut T`(如上所述的裸指针)作为参数来执行易失性读取或写入。

[`core::ptr::read_volatile`]:https://doc.rust-lang.org/core/ptr/fn.read_volatile.html
[`core::ptr::write_volatile`]:https://doc.rust-lang.org/core/ptr/fn.write_volatile.html

例如，在C中，您这样写：

```c
volatile bool signalled = false;

void ISR() {
    // Signal that the interrupt has occurred
    signalled = true;
}

void driver() {
    while(true) {
        // Sleep until signalled
        while(!signalled) { WFI(); }
        // Reset signalled indicator
        signalled = false;
        // Perform some task that was waiting for the interrupt
        run_task();
    }
}
```

Rust中的等效代码是在每次访问时使用volatile方法：

```rust , ignore
static mut SIGNALLED: bool = false;

#[interrupt]
fn ISR() {
    // Signal that the interrupt has occurred
    // (In real code, you should consider a higher level primitive,
    //  such as an atomic type).
    unsafe { core::ptr::write_volatile(&mut SIGNALLED, true) };
}

fn driver() {
    loop {
        // Sleep until signalled
        while unsafe { !core::ptr::read_volatile(&SIGNALLED) } {}
        // Reset signalled indicator
        unsafe { core::ptr::write_volatile(&mut SIGNALLED, false) };
        // Perform some task that was waiting for the interrupt
        run_task();
    }
}
```

示例代码中需要注意以下几点：
  * 我们可以将`&mut SIGNALLED`传递到需要`*mut T`的write_volatile中，因为`&mut T`会自动转换为`*mut T`(`&T`自动转换为`*const T`)。
  * 对于`read_volatile`/`write_volatile`方法，我们需要使用unsafe块，因为它们是unsafe函数。确保安全地使用这两个函数是程序员的责任：更多详细信息，请参见这两个方法的文档。

很少直接在您的代码中需要这些功能，因为更高级别的crate通常会为您处理这些功能。对于内存映射的外围设备，外围设备访问crate将自动实现易失性访问，而对于并发原语，则可以使用更好的抽象(请参见[并发章节])。

[并发章节]:../concurrency/index.md

## 内存对齐

在嵌入式C语言中，通常告诉编译器变量必须具有一定的对齐方式，或者必须打包而不是对齐结构体，这通常是为了满足特定的硬件或协议要求。

在Rust中，这是由结构体或联合的`repr`属性控制。默认表示形式不保证布局，因此不在与硬件或C互操作的代码中使用。编译器可能会重新排序结构体成员或插入填充，而编译器的这些行为可能会在Rust以后的版本中发生改变。

```rust
struct Foo {
    x: u16,
    y: u8,
    z: u16,
}

fn main() {
    let v = Foo { x: 0, y: 0, z: 0 };
    println!("{:p} {:p} {:p}", &v.x, &v.y, &v.z);
}

// 0x7ffecb3511d0 0x7ffecb3511d4 0x7ffecb3511d2
// Note ordering has been changed to x, z, y to improve packing.
```

为了确保布局可以和C互操作，请使用 `repr(C)`：

```rust
#[repr(C)]
struct Foo {
    x: u16,
    y: u8,
    z: u16,
}

fn main() {
    let v = Foo { x: 0, y: 0, z: 0 };
    println!("{:p} {:p} {:p}", &v.x, &v.y, &v.z);
}

// 0x7fffd0d84c60 0x7fffd0d84c62 0x7fffd0d84c64
// Ordering is preserved and the layout will not change over time.
// `z` is two-byte aligned so a byte of padding exists between `y` and `z`.
```

为了确保紧凑内存布局(一字节对齐)，请使用 `repr(packed)`：

```rust
#[repr(packed)]
struct Foo {
    x: u16,
    y: u8,
    z: u16,
}

fn main() {
    let v = Foo { x: 0, y: 0, z: 0 };
    // Unsafe is required to borrow a field of a packed struct.
    unsafe { println!("{:p} {:p} {:p}", &v.x, &v.y, &v.z) };
}

// 0x7ffd33598490 0x7ffd33598492 0x7ffd33598493
// No padding has been inserted between `y` and `z`, so now `z` is unaligned.
```

注意，使用`repr(packed)`还将类型的对齐方式设置为一字节。

最后，要指定特定的对齐方式，请使用`repr(align(n))`，其中`n`是要对齐的字节数(必须为2的幂)：

```rust
#[repr(C)]
#[repr(align(4096))]
struct Foo {
    x: u16,
    y: u8,
    z: u16,
}

fn main() {
    let v = Foo { x: 0, y: 0, z: 0 };
    let u = Foo { x: 0, y: 0, z: 0 };
    println!("{:p} {:p} {:p}", &v.x, &v.y, &v.z);
    println!("{:p} {:p} {:p}", &u.x, &u.y, &u.z);
}

// 0x7ffec909a000 0x7ffec909a002 0x7ffec909a004
// 0x7ffec909b000 0x7ffec909b002 0x7ffec909b004
// The two instances `u` and `v` have been placed on 4096-byte alignments,
// evidenced by the `000` at the end of their addresses.
```


注意，我们可以将 `repr(C)`与`repr(align(n))`结合使用以获得对齐并兼容C的布局。不允许将`repr(align(n))`与`repr(packed)`结合使用，因为`repr(packed)`将对齐方式设置为1。

有关类型布局的更多详细信息，请参见Rust参考中的[类型布局]一章。

[类型布局]:https://doc.rust-lang.org/reference/type-layout.html

## 其他资源

*本书中的：
  * [带有Rust的C](../interoperability/rust-with-c.md)
  * [C语言有点锈](../interoperability/rust-with-c.md)
* [嵌入式Rust常见问题解答](https://docs.rust-embedded.org/faq.html)
* [C程序员的Rust指针](http://blahg.josefsipek.net/?p=580)
* [I used to use pointers - now what?](https://github.com/diwic/reffers-rs/blob/master/docs/Pointers.md)