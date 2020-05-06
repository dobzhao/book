# 安装工具

 

### Rust工具链

按照[rustup](https://rustup.rs)上的说明安装rustup。

**注意**确保您使用的编译器版本不低于1.31。 `rustc -V`应该返回比下面示例中更新的一个日期。

```sh 
$ rustc -V
rustc 1.31.1(b6c32da9b 2018-12-18)
```


rustup的默认安装仅支持本机编译,因此需要添加对ARM Cortex-M的交叉编译支持。对于STM32F3DISCOVERY这个本书示例使用的开发板，请使用target "thumbv7em-none-eabihf"。

针对Cortex-M0，M0+和M1(ARMv6-M架构)：
```sh
$ rustup target add thumbv6m-none-eabi
```

针对Cortex-M3(ARMv7-M架构)：
```sh
$ rustup target add thumbv7m-none-eabi
```

针对没有硬浮点的Cortex-M4和M7(ARMv7E-M架构)：
```sh
$ rustup target add thumbv7em-none-eabi
```

针对具有硬浮点的Cortex-M4F和M7F(ARMv7E-M架构)：
```sh
$ rustup target add  thumbv7em-none-eabihf
```

### `cargo-binutils`

```sh 
$ cargo install cargo-binutils
$ rustup component add  llvm-tools-preview
```

### `cargo-generate`

稍后我们将使用它从模板生成项目。

```sh
$ cargo install cargo-generate
```

### 操作系统相关的安装说明

接下来是平台相关的安装过程：

- [Linux](install/linux.md)
- [Windows](install/windows.md)
- [macOS](install/macos.md)