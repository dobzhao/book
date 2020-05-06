# 类型状态机(TypesState)编程

[typestates]就是通过对象的类型来表示对象的状态信息。尽管这听起来有些不可思议，但是如果您在Rust中使用了[建造者模式]，那么您已经开始使用类型状态机了. 

[typestates]:https://en.wikipedia.org/wiki/Typestate_analysis
[建造者模式]:https://doc.rust-lang.org/1.0.0/style/ownership/builders.html

```rust
pub mod foo_module {
    #[derive(Debug)]
    pub struct Foo {
        inner: u32,
    }

    pub struct FooBuilder {
        a: u32,
        b: u32,
    }

    impl FooBuilder {
        pub fn new(starter: u32) -> Self {
            Self {
                a: starter,
                b: starter,
            }
        }

        pub fn double_a(self) -> Self {
            Self {
                a: self.a * 2,
                b: self.b,
            }
        }

        pub fn into_foo(self) -> Foo {
            Foo {
                inner: self.a + self.b,
            }
        }
    }
}

fn main() {
    let x = foo_module::FooBuilder::new(10)
        .double_a()
        .into_foo();

    println!("{:#?}", x);
}
```

在这个例子中，没有直接的方法来创建一个`Foo`对象。我们必须创建一个`FooBuilder`并正确地对其进行初始化，然后才能获得所需的`Foo`对象。

这个最小的示例对两种状态进行编码：

* `FooBuilder`，代表“未配置”或“正在配置”状态
* `Foo`，表示“已配置”或“准备使用”状态。

## 强类型

由于Rust具有[强类型系统]，因此没有简单的方法直接创建`Foo`实例，或将`FooBuilder`转换为`Foo`而无需调用`into_foo()`方法。另外，调用`into_foo()`方法会消耗原始的`FooBuilder`对象，这意味着如果不创建新实例就无法重用它。

[强类型系统]: https://en.wikipedia.org/wiki/Strong_and_weak_typing

这使我们可以将系统的状态表示为类型，并将状态转换所必需的动作包括在将一种类型转换换为另一种类型的方法中。通过创建一个 `FooBuilder`，并将其转换为一个`Foo`对象，我们实现了最基本的状态机。