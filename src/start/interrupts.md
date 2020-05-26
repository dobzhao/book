# 中断

中断在很多方面与异常不同，但是它们的操作和使用在很大程度上相似，并且它们也由同一中断控制器处理。异常是由Cortex-M架构统一定义的，中断则在命名和功能上随供应商(甚至是芯片)不同而不同。

中断确实具有很大的灵活性，在尝试以高级方式使用它们时需要考虑这些灵活性。我们不会在本书中介绍这些用法，但是请牢记以下几点：

* 中断具有可编程的优先级，该优先级确定其处理程序的执行顺序
* 中断可以嵌套和抢占，即中断处理程序的执行可能会被另一个更高优先级的中断抢占
* 通常需要清除导致中断触发的事件，以防止无限次重新进入中断处理程序

中断的常规初始化步骤始终相同：
* 配置外设,在需要的情况下生成中断请求
* 在中断控制器中设置所需的中断处理程序优先级
* 在中断控制器中启用中断处理程序

与异常类似，`cortex-m-rt` crate 提供了一个[`interrupt`]属性来声明中断处理程序。可用的中断(及其在中断处理程序表中的位置)通常是使用`svd2rust`基于SVD描述文件自动生成的。

[`interrupt`]:https://docs.rs/cortex-m-rt-macros/0.1.5/cortex_m_rt_macros/attr.interrupt.html

``` rust , ignore
// Interrupt handler for the Timer2 interrupt
#[interrupt]
fn TIM2() {
    // ..
    // Clear reason for the generated interrupt request
}
```

中断处理程序类似于异常处理程序，它们不能被固件的其他部分直接调用。但是可以在软件中生成中断请求，以触发对中断处理程序。

与异常处理程序类似，在中断处理程序中声明`static mut`变量也是安全的。

``` rust , ignore
#[interrupt]
fn TIM2() {
    static mut COUNT: u32 = 0;

    // `COUNT` has type `&mut u32` and it's safe to use
    *COUNT += 1;
}
```

有关此处演示的机制的更详细说明，请参考[异常]。

[异常]:./exceptions.md