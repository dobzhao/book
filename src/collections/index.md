# 容器

最终，您将要在程序中使用动态数据结构(也就是容器)。 `std`提供了一组通用容器：[`Vec`]，[`String`]，[`HashMap`]等。在`std`中实现的所有容器都使用了全局动态内存分配器(也称为堆)。

[`Vec`]:https://doc.rust-lang.org/std/vec/struct.Vec.html
[`String`]:https://doc.rust-lang.org/std/string/struct.String.html
[`HashMap`]:https://doc.rust-lang.org/std/collections/struct.HashMap.html

`core`本身是没有动态内存分配的,但是编译器自带了一个**unstable**的`alloc` crate支持动态内存分配.

 
如果需要容器，基于堆的实现不是唯一的选择。您还可以使用“固定容量”容器；可以在[`heapless`]crate中找到一种这样的实现。

[`heapless`]:https://crates.io/crates/heapless

在本节中，我们将探索和比较这两种实现。

## 使用`alloc`

标准的Rust发行版中已经包含了`alloc`,您可以直接使用它，而无需在Cargo.toml文件中将其声明为依赖项。

``` rust , ignore
#![feature(alloc)]

extern crate alloc;

use alloc::vec::Vec;
```

要使用容器，您首先需要使用`global_allocator`属性来声明程序将使用的全局分配器。这个分配器要实现[`GlobalAlloc`]trait。

[`GlobalAlloc`]:https://doc.rust-lang.org/core/alloc/trait.GlobalAlloc.html

为了完整起见，并保持本节尽可能独立，我们将实现一个简单的凹凸指针分配器，并将其用作全局分配器。但是，我们强烈建议您在程序中使用crates.io上经过充分实战测试的分配器，而不要使用此分配器。

``` rust , ignore
// Bump pointer allocator implementation

extern crate cortex_m;

use core::alloc::GlobalAlloc;
use core::ptr;

use cortex_m::interrupt;

// Bump pointer allocator for *single* core systems
struct BumpPointerAlloc {
    head: UnsafeCell<usize>,
    end: usize,
}

unsafe impl Sync for BumpPointerAlloc {}

unsafe impl GlobalAlloc for BumpPointerAlloc {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        // `interrupt::free` is a critical section that makes our allocator safe
        // to use from within interrupts
        interrupt::free(|_| {
            let head = self.head.get();
            let size = layout.size();
            let align = layout.align();
            let align_mask = !(align - 1);

            // move start up to the next alignment boundary
            let start = (*head + align - 1) & align_mask;

            if start + size > self.end {
                // a null pointer signal an Out Of Memory condition
                ptr::null_mut()
            } else {
                *head = start + size;
                start as *mut u8
            }
        })
    }

    unsafe fn dealloc(&self, _: *mut u8, _: Layout) {
        // this allocator never deallocates memory
    }
}

// Declaration of the global memory allocator
// NOTE the user must ensure that the memory region `[0x2000_0100, 0x2000_0200]`
// is not used by other parts of the program
#[global_allocator]
static HEAP: BumpPointerAlloc = BumpPointerAlloc {
    head: UnsafeCell::new(0x2000_0100),
    end: 0x2000_0200,
};
```

除了选择全局分配器之外，用户还必须处理内存不足(OOM)错误,这个可以借助**unstable**的`alloc_error_handler`属性。

``` rust , ignore
#![feature(alloc_error_handler)]

use cortex_m::asm;

#[alloc_error_handler]
fn on_oom(_layout: Layout) -> ! {
    asm::bkpt();

    loop {}
}
```

一切就绪后，就以使用`alloc`中的容器了。


```rust , ignore
#[entry]
fn main() -> ! {
    let mut xs = Vec::new();

    xs.push(42);
    assert!(xs.pop(), Some(42));

    loop {
        // ..
    }
}
```

这些容器与标准库中的容器实现完全一样,你使用起来会觉得非常熟悉.

## 使用`heapless`

`heapless`不需要设置，因为其容器不依赖于全局内存分配器,所以开箱即用：

```rust , ignore
extern crate heapless; // v0.4.x

use heapless::Vec;
use heapless::consts::*;

#[entry]
fn main() -> ! {
    let mut xs: Vec<_, U8> = Vec::new();

    xs.push(42).unwrap();
    assert_eq!(xs.pop(), Some(42));
}
```
 
您会注意到这些容器与alloc中的两个区别。

首先，您必须预先声明容器的容量。 `heapless`容器从不重新分配内存并且具有固定容量；容量大小是容器类型签名的一部分。这里我们声明`xs`是容量有8个元素的Vector。这由类型签名中的`U8`(请参阅​​[`typenum`])指明。

[`typenum`]:https://crates.io/crates/typenum

其次，`push`方法和许多其他方法都返回`Result`。由于`heapless`容器具有固定的容量，因此将元素插入容器的所有操作都可能会失败。 API通过返回结果`Result`来表明是成功还是失败。相反，`alloc`容器将自己在堆上重新分配以增加其容量。

从v0.4.x版本开始，所有`heapless`容器都内联存储所有元素。这意味着像`let x = heapless::Vec::new();`这样的操作将在栈上分配容器，当然你也可以在`static`变量上甚至在堆上分配容器(`Box<Vec<_, _>>`)。

## 权衡取舍

在堆分配可重定位的容器和固定容量的容器之间进行选择时，请从以下角度考虑. 

### 内存不足(OOM)和错误处理

使用堆分配总是存在内存不足的可能性，并且可能发生在需要增长容器的任何地方：例如，所有`alloc::Vec.push`调用都可能会导致OOM。因此某些操作可能会悄无声息的失败。某些alloc容器公开了`try_reserve`方法，这些方法可让您在容器增长时检查潜在的OOM，但您需要主动使用它们。

如果您只使用`heapless`容器，并且不在任何其他地方使用内存分配器，那么肯定不会发生OOM。取而代之的是，您每次都要考虑容器的容量问题。也就是您必须处理所有`Vec.push`之类的方法返回的`Result`。

直接在`heapless::Vec.push`返回的`Result`上unwrap当然可能会触发OOM错误,但是还有其他更难调试的OOM错误,这是因为你观察到的错误位置可能不是引起问题的实际位置.例如，如果由于其他容器正在发生了内存泄漏(安全的Rust中可能发生内存泄漏)而导致几乎无内存可用，那么即使是`vec.reserve(1)`也会触发OOM。


###  内存使用情况

很难对堆分配的容器的内存使用进行准确判断，因为使用周期很长的容器的容量可以在运行时更改。有些操作可能会隐式地重定位容器，从而增加其内存使用量，而某些容器会提供`shrink_to_fit`之类的方法，这些方法可能会减少容器使用的内存，甚至可能由分配器决定是否实际缩小内存分配。此外，分配器可能必须处理内存碎片，这可能会增加表面上的内存占用。

另一方面，如果您使用固定容量容器，将它们中的大多数存储在静态变量中，并设置栈的最大大小，那么链接器会检测到您使用的内存是否超过实际可用的内存。

此外，分配在栈上的固定容量容器的大小可以通过[`-Z emit-stack-sizes`]参数来报告，分析栈使用情况的工具(例如[`stack-sizes`])会将此信息包含在分析结果中。

[`-Z emit-stack-sizes`]:https://doc.rust-lang.org/beta/unstable-book/compiler-flags/emit-stack-sizes.html
[`stack-sizes`]:https://crates.io/crates/stack-sizes

但是，固定容量的容器不能缩小，这可能导致其负载因子(容器实际大小与其容量之间的比率)低于堆分配的可重定位容器。

### 最坏情况执行时间(WCET)

如果要构建对时间敏感的应用程序或硬实时应用程序，那么您可能会担心程序的不同部分在最坏情况下的执行时间。

`alloc`容器可能会重新分配内存，因此容器增长操作的WCET也将包括重新分配容器所需的时间，而这个时间取决于容器的运行时容量,这就很难确定WCET是多少. 例如`alloc::Vec.push`操作所用时间既依赖于所用的分配器实现算法也依赖于容器当时的容量。

相比之下，固定容量容器永远不会重新分配内存，因此所有操作都具有可预测的执行时间。例如，`heapless::Vec.push`将在固定时间内执行。

### 使用方便性

`alloc`需要设置全局分配器，而 `heapless` 则不需要。但是 `heapless` 要求您在实例化时确定每个容器的容量。

每个Rust开发人员都熟悉 `alloc` API。  `heapless` API试图尽可能地模仿`alloc`，但由于其显式错误处理，他们永远不会完全相同-一些开发人员可能会觉得显式错误处理过于繁琐。