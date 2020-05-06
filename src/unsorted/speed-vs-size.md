# 优化：速度大小的权衡

每个人都希望他们的程序超快，超小，但通常不可能兼具这两个特性。本节讨论`rustc` 提供的不同优化级别，以及它们如何影响程序的执行时间和二进制大小。

## 没有优化

这是默认值。当您调用`cargo build`时，可以使用开发(也就是`dev`)配置文件。这个配置文件针对调试进行了优化，因此它启用调试信息并且*不*启用任何优化，即它使用`-C opt-level = 0`。

至少对于裸机开发而言，debuginfo的成本为零，因为它不会占用Flash/ROM中的空间，因此我们建议您在发行配置文件中启用debuginfo(默认情况下处于禁用状态)。这样您就可以在调试发行版时使用断点。

``` toml
[profile.release]
# symbols are nice and they don't increase the size on Flash
debug = true
```

禁用优化对调试很有效，因为单步执行代码就像逐行执行代码一样，而且可以在GDB中“打印”堆栈变量和函数参数。优化代码后，尝试打印变量将导致打印`$0 = <value optimized out>`。

dev配置文件的最大缺点是，生成的二进制文件会很大且很慢。大小通常是一个更大的问题，因为未优化的二进制文件可能会占用数十KiB的Flash，而目标设备可能没有这么大空间，这会导致：未优化的二进制文件不适合您的设备！

我们可以使用较小的，调试器友好的二进制文件吗？是的，有个窍门。

### 优化依赖

有一个名为[`profile-overrides`]的cargo特性，可让您覆盖依赖的crate的优化级别。您可以使用该功能优化所有依赖的crate的大小，同时保持项目自身不优化和对调试器的友好性。

[`profile-overrides`]: https://doc.rust-lang.org/cargo/reference/profiles.html#overrides

这是一个例子：


``` toml
# Cargo.toml
[package]
name = "app"
# ..

[profile.dev.package."*"] # +
opt-level = "z" # +
```


没有覆盖默认优化时：

``` console
$ cargo size --bin app -- -A
app  :
section               size        addr
.vector_table         1024   0x8000000
.text                 9060   0x8000400
.rodata               1708   0x8002780
.data                    0  0x20000000
.bss                     4  0x20000000
```


使用覆盖：

``` console
$ cargo size --bin app -- -A
app  :
section               size        addr
.vector_table         1024   0x8000000
.text                 3490   0x8000400
.rodata               1100   0x80011c0
.data                    0  0x20000000
.bss                     4  0x20000000
```
闪存使用量减少了6 KiB，而项目自身的可调试性没有任何损失。如果在调试时,您进入依赖项，您将再次看到那些`<value Optimized out>`消息，但是通常情况下，您要调试的是项目自身而不是依赖项。而且如果您需要调试依赖项，则可以使用`profile-overrides`特性来排除特定的依赖项，以使其不被优化。请参见下面的示例：

``` toml
# ..

# don't optimize the `cortex-m-rt` crate
[profile.dev.package.cortex-m-rt] # +
opt-level = 0 # +

# but do optimize all the other dependencies
[profile.dev.package."*"]
codegen-units = 1 # better optimizations
opt-level = "z"
```

现在，项目自身和`cortex-m-rt`都对调试器友好了！

## 优化速度

自2018年9月18日起，rustc支持三种“优化速度”级别`opt-level = 1,2,3`。当您运行`cargo build --release`时，您使用的是发布配置文件，默认为`opt-level = 3`。

`opt-level = 2` 和`3`都针对速度进行了优化，但以二进制大小为代价，3级比2级进行了更多的矢量化和内联。特别是，您将看到在opt-level等于或大于2的情况下，LLVM可能会展开循环。就Flash/ROM而言，循环展开具有相当高的成本(例如，对于一个将数组清零的循环，大小可能从26个字节到增大到194个字节)，但在合适的条件下(例如迭代次数足够大)，也可以将执行时间减半。

当前没有办法在 `opt-level = 2,3`时禁用循环展开，因此，如果您负担不起其成本，则应针对大小优化程序。

## 优化尺寸

从2018年9月18日开始，rustc支持两个“大小优化”级别：`opt-level = s,z`。这些名称是从clang/LLVM继承的，所以描述性不太强，“z”的含义是产生比“s”更小的二进制文件。

如果您希望优化发布二进制文件的大小，请按如下所示,在Cargo.toml中更改profile.release.opt-level设置。

``` toml
[profile.release]
# or "z"
opt-level = "s"
```

这两个优化级别大大降低了LLVM的内联阈值，该阈值用于确定是否内联函数。 Rust原则之一是零成本抽象。这些抽象倾向于使用大量的新类型和小的函数来保存不变式(例如，诸如“ deref”，“as_ref”之类的借用内部值的函数)，因此低的内联阈值会使LLVM错过优化机会(例如，消除无效分支，闭包的内联)。

在优化大小时，您可能想尝试增加内联阈值，以查看这是否对二进制大小有影响。推荐的更改内联阈值的方法是将`-C inline-threshold` 参数附加到`.cargo/config`中的rustflags。

``` toml
# .cargo/config
# this assumes that you are using the cortex-m-quickstart template
[target.'cfg(all(target_arch = "arm", target_os = "none"))']
rustflags = [
  # ..
  "-C", "inline-threshold=123", # +
]
```

内联阈值使用什么值合适？从1.29.0开始，下面是不同优化级别使用的[内联阈值]:

[在线阈值]:https://github.com/rust-lang/rust/blob/1.29.0/src/librustc_codegen_llvm/back/write.rs#L2105-L2122

- `opt-level = 3` 使用275
- `opt-level = 2` 使用225
- `opt-level =“ s”` 使用75
- `opt-level =“ z” `使用25

在优化大小时，应尝试使用较大的内联阈值比如“225”和“275”。