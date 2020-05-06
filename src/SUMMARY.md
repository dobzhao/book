# Summary

<!--

Definition of the organization of this book is still a work in process.

Refer to https://github.com/rust-embedded/book/issues for
more information and coordination

-->

- [简介](./intro/index.md)
    - [硬件](./intro/hardware.md)
    - [`no_std`](./intro/no-std.md)
    - [工具](./intro/tooling.md)
    - [安装](./intro/install.md)
        - [Linux](./intro/install/linux.md)
        - [MacOS](./intro/install/macos.md)
        - [Windows](./intro/install/windows.md)
        - [验证安装](./intro/install/verify.md)
- [入门](./start/index.md)
  - [QEMU](./start/qemu.md)
  - [硬件](./start/hardware.md)
  - [内存映射寄存器](./start/registers.md)
  - [半主机](./start/semihosting.md)
  - [恐慌](./start/panicking.md)
  - [异常](./start/exceptions.md)
  - [中断](./start/interrupts.md)
  - [IO](./start/io.md)
- [外设](./peripherals/index.md)
    - [初试Rust](./peripherals/a-first-attempt.md)
    - [借用检查器](./peripherals/borrowck.md)
    - [单例](./peripherals/singletons.md)
- [静态保证](./static-guarantees/index.md)
    - [类型状态机编程](./static-guarantees/typestate-programming.md)
    - [外设作为状态机](./static-guarantees/state-machines.md)
    - [设计合约](./static-guarantees/design-contracts.md)
    - [零成本抽象](./static-guarantees/zero-cost-abstractions.md)
- [可移植性](./portability/index.md)
- [并发](./concurrency/index.md)
- [容器](./collections/index.md)
- [嵌入式C开发人员的技巧](./c-tips/index.md)
    <!-- TODO: Define Sections -->
- [互操作性](./interoperability/index.md)
    - [Rust中使用C代码](./interoperability/c-with-rust.md)
    - [C中使用Rust代码](./interoperability/rust-with-c.md)
- [其他主题](./unsorted/index.md)
  - [优化：速度大小的权衡](./unsorted/speed-vs-size.md)

---

[Appendix A: Glossary](./appendix/glossary.md)
