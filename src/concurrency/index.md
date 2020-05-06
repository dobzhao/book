# 并发

只要程序的不同部分可能在不同时间执行或者乱序执行就存在并发。在嵌入式上下文中，这包括：

* 中断处理程序，每当相关中断发生时运行，
* 多种形式的多线程，您的微处理器定期在程序的各个部分之间进行交换，
* 在某些系统中是多核微处理器，其中每个核可以同时独立运行程序的不同部分。

由于许多嵌入式程序需要处理中断，因此并发通常迟早会出现，这也是可能会发生许多细微而困难的错误的地方。幸运的是，Rust提供了许多抽象和安全保证来帮助我们编写正确的代码。

## 没有并发

嵌入式程序最简单的并发就是没有并发：您的软件由一个主循环组成，根本没有中断。有时，这非常适合现实情况！通常循环读取一些输入，执行一些处理，然后进行输出。


```rust , ignore
#[entry]
fn main() {
    let peripherals = setup_peripherals();
    loop {
        let inputs = read_inputs(&peripherals);
        let outputs = process(inputs);
        write_outputs(&peripherals, outputs);
    }
}
```


由于没有并发性，因此无需担心在程序各部分之间共享数据或对外设的同步访问。如果您可以采用这种简单的方法，那将是一个很好的解决方案。

## 全局可变数据

与非嵌入式Rust不同，我们通常不会奢侈地使用堆分配内存并将对该数据的引用传递到新创建的线程中。相反，我们的中断处理程序可能随时被调用，并且必须知道如何访问我们正在使用的任何共享内存。这意味着我们在底层必须具有“静态分配”的可变内存，中断处理程序和主代码都可以引用该可变内存。

在Rust中，此类['static mut`]变量读写始终是不安全的，因为如果不特别注意，您可能会触发竞争条件，其中对变量的访问可能会随时被中断，而相应中断处理程序同样需要访问该变量。

[`static mut`]:https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html#accessing-or-modifying-a-mutable-static-variable

让我们来看一个例子,来看看此行为是如何导致代码出现细微的错误. 请考虑一个嵌入式程序，该程序统计一秒内(频率计数器)某些输入信号的上升沿出现的次数：

```rust , ignore
static mut COUNTER: u32 = 0;

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            // DANGER - Not actually safe! Could cause data races.
            unsafe { COUNTER += 1 };
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    unsafe { COUNTER = 0; }
}
```

定时器中断每秒都会将计数器重置为0。与此同时，主循环不断地测量信号，并在看到从低到高的变化时增加计数器。我们必须使用`unsafe`来访问`COUNTER`，因为它是`static mut`，使用`unsafe`意味着我们向编译器保证不会引起任何未定义的行为。你能发现其中的竞争问题吗？不能保证`COUNTER`上的增加是原子的-实际上，在大多数嵌入式平台上，它将被分为读入，增加，然后是写回。如果在读入之后但在写回之前发生了中断，则在中断返回后,重置为0的操作被忽略,我们将计数两倍的转换次数。

## 临界区

那么，我们该如何处理数据竞赛？一种简单的方法是使用“临界区”，在关键部分中中断是被禁用的。通过将`main`函数中对`COUNTER`的访问部分放在临界区中，我们可以确保在完成递增`COUNTER`之前不会被计时器中断：

```rust , ignore
static mut COUNTER: u32 = 0;

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            // New critical section ensures synchronised access to COUNTER
            cortex_m::interrupt::free(|_| {
                unsafe { COUNTER += 1 };
            });
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    unsafe { COUNTER = 0; }
}
```

在这个例子中，我们使用`cortex_m::interrupt::free`，其他平台也有类似的机制。这也与禁用中断，运行一些代码然后重新启用中断相同。

请注意，由于两个原因，我们不需要在计时器中断中放置临界区：
* 向`COUNTER`写入0不会受到竞态问题的影响，因为我们没有读它
* 它永远不会被`main`线程打断

如果`COUNTER`被多个可能相互抢占的中断处理程序共享，则每个中断处理程序也可能需要一个临界区。

这解决了我们的迫在眉睫的问题，但是我们仍然需要编写很多不安全的代码，这些代码我们需要仔细检查，并且可能会不必要地使用临界区。由于每个临界区都会暂时中止中断处理，因此会产生一些额外的代码，并产生更高的中断延迟和抖动(中断可能需要更长的时间才能被处理，并且处理之前的等待时间会更加不确定)。这是否有问题取决于您的系统，但总的来说，我们希望避免这种情况。

值得注意的是，尽管临界区保证不会触发任何中断，但它不能在多核系统上提供排他性保证！另一个内核可能很高兴访问与您的内核相同的内存，即使没有中断也是如此。如果使用多个内核，则将需要更强大的同步原语。

## 原子访问

在某些平台上，可以使用原子指令，这些指令保证了读-修改-写操作(CAS compare and set)是原子的。在Cortex-M架构中，`thumbv6`(Cortex-M0)不提供原子指令，而`thumbv7`(Cortex-M3及更高版本)则提供原子指令。这些指令可以避免禁用所有中断：我们可以尝试递增，它在大多数时间都会成功，但是如果被中断，它将自动重试整个递增操作。即使在多个内核之间，这些原子操作也是安全的。

```rust , ignore
use core::sync::atomic::{AtomicUsize, Ordering};

static COUNTER: AtomicUsize = AtomicUsize::new(0);

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            // Use `fetch_add` to atomically add 1 to COUNTER
            COUNTER.fetch_add(1, Ordering::Relaxed);
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    // Use `store` to write 0 directly to COUNTER
    COUNTER.store(0, Ordering::Relaxed)
}
```

这次，COUNTER是一个安全的static变量。由于使用了`AtomicUsize`类型，可以从中断处理程序和主线程安全地修改`COUNTER'，而无需禁用中断。如果可能，这是一个更好的解决方案-但您的平台可能不支持它。

关于[`Ordering]的注释：这会影响编译器和硬件如何对指令进行重新排序，并对缓存可见性产生影响。假设目标是单核心平台，那么 `Relaxed`就足够了，并且在这种情况下是最有效的选择。更严格的顺序将导致编译器在原子操作前后发出内存屏障。取决于您正在使用原子操作的种类，您可能需要也可能不需要更严格的顺序！原子模型的精确细节非常复杂，在其他地方有最好的描述。

有关原子操作和顺序的更多详细信息，请参见[nomicon]。

[`Ordering`]:https：//doc.rust-lang.org/core/sync/atomic/enum.Ordering.html
[nomicon]:https：//doc.rust-lang.org/nomicon/atomics.html


## 抽象，Send和Sync

上述解决方案都不是特别令人满意。他们要求使用 `unsafe`代码(todo atomic方案明明不需要啊?!)，这些代码必须非常仔细地检查并且不符合人体工程学。当然，我们可以在Rust中做得更好！

我们可以将计数器抽象为一个安全的接口，该接口可以在代码中的其他位置安全地使用。在此示例中，我们将使用临界区计数器，您仍然可以执行类似原子操作的操作。

```rust , ignore
use core::cell::UnsafeCell;
use cortex_m::interrupt;

// Our counter is just a wrapper around UnsafeCell<u32>, which is the heart
// of interior mutability in Rust. By using interior mutability, we can have
// COUNTER be `static` instead of `static mut`, but still able to mutate
// its counter value.
struct CSCounter(UnsafeCell<u32>);

const CS_COUNTER_INIT: CSCounter = CSCounter(UnsafeCell::new(0));

impl CSCounter {
    pub fn reset(&self, _cs: &interrupt::CriticalSection) {
        // By requiring a CriticalSection be passed in, we know we must
        // be operating inside a CriticalSection, and so can confidently
        // use this unsafe block (required to call UnsafeCell::get).
        unsafe { *self.0.get() = 0 };
    }

    pub fn increment(&self, _cs: &interrupt::CriticalSection) {
        unsafe { *self.0.get() += 1 };
    }
}

// Required to allow static CSCounter. See explanation below.
unsafe impl Sync for CSCounter {}

// COUNTER is no longer `mut` as it uses interior mutability;
// therefore it also no longer requires unsafe blocks to access.
static COUNTER: CSCounter = CS_COUNTER_INIT;

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            // No unsafe here!
            interrupt::free(|cs| COUNTER.increment(cs));
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    // We do need to enter a critical section here just to obtain a valid
    // cs token, even though we know no other interrupt could pre-empt
    // this one.
    interrupt::free(|cs| COUNTER.reset(cs));

    // We could use unsafe code to generate a fake CriticalSection if we
    // really wanted to, avoiding the overhead:
    // let cs = unsafe { interrupt::CriticalSection::new() };
}
```


我们已经将“不安全”代码移到了经过精心计划的抽象内部，现在，我们的应用程序代码不包含任何“不安全”代码。

这种设计要求应用程序在其中传递一个“ CriticalSection”令牌：这些令牌仅由`interrupt::free`安全地生成，通过传递一个令牌，我们确保我们在临界区内进行操作，而不必实际执行锁定。编译器静态地保证与`cs`相关操作没有任何运行时开销。如果我们有多个计数器，可以传递给它们相同的`cs`，而无需多个嵌套的临界区。

这也引出了Rust并发中的一个重要的话题：[`Send`和`Sync`] trait。总结一下，当一个类型可以安全地将其移动到另一个线程时，它满足Send；当一个类型可以在多个线程之间安全地只读地共享时，它满足Sync。在嵌入式上下文中，我们认为中断是在与应用程序代码不同的线程中执行的，因此，由中断和主程序代码访问的变量必须为Sync。

[`Send`和`Sync`]:https://doc.rust-lang.org/nomicon/send-and-sync.html

对于Rust中的大多数类型，这两个trait都是由编译器自动为您生成的。但是由于`CSCounter`包含[`UnsafeCell`]，因此它不是`Sync`的，因此我们不能声明`static CSCounter`：`static`变量必须是Sync的，因为它们可能被多个线程访问。

[`UnsafeCell`]:https://doc.rust-lang.org/core/cell/struct.UnsafeCell.html

为了告诉编译器我们的`CSCounter`实际上可以安全地在线程之间共享，我们明确实现了`Sync` trait。与以前使用临界区一样，这仅在单核平台上才是安全的：对于多核，您将需要做更多的工作才能确保安全。

## 互斥锁(Mutex)

我们已经针对计数器问题创建了一个有用的抽象，但是针对并发问题,有更多常见的抽象。

一种这样的“同步原语”是互斥锁(mutex: mutual exclusion)。互斥锁确保对变量(例如我们的计数器)的独占访问。线程可以尝试执行互斥锁的_lock_(或_acquire_)，结果可能是立即成功获取到锁，或者阻塞等待直到获取到锁，或者因为互斥锁无法锁定而返回错误。当该线程持有锁时，它被授予对受保护数据的访问权限。访问完成后，它将_unlocks_(或_releases_)互斥锁，从而允许另一个线程将其锁定。在Rust中，我们通常会使用[`Drop`]trait来实现解锁，以确保在互斥锁超出范围时始终将其释放。

[`Drop`]:https://doc.rust-lang.org/core/ops/trait.Drop.html

将互斥锁与中断处理程序一起使用可能会很棘手：中断处理程序通常无法接受阻塞，并且在中断中阻塞等待主线程释放锁尤其会造成灾难性的后果，因为这样我们就会死锁(线程永远不会释放锁，因为中断处理程序没有返回)。死锁并不被认为是不安全的：即使在安全的Rust中也有可能。

为了完全避免这种行为，我们可以实现一个互斥锁，该互斥锁需要一个临界区进行锁定，就像前面的计数器示例一样。只要临界区必须持续与锁定一样长的时间，我们就可以确保对包装变量的独占访问权，甚至无需跟踪互斥锁的锁定/解锁状态。

实际上， `cortex_m` crate已经帮我们做好了！我们可以使用它来编写计数器：

```rust , ignore
use core::cell::Cell;
use cortex_m::interrupt::Mutex;

static COUNTER: Mutex<Cell<u32>> = Mutex::new(Cell::new(0));

#[entry]
fn main() -> ! {
    set_timer_1hz();
    let mut last_state = false;
    loop {
        let state = read_signal_level();
        if state && !last_state {
            interrupt::free(|cs|
                COUNTER.borrow(cs).set(COUNTER.borrow(cs).get() + 1));
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    // We still need to enter a critical section here to satisfy the Mutex.
    interrupt::free(|cs| COUNTER.borrow(cs).set(0));
}
```

我们现在使用的是[`Cell`]，它与`RefCell`一样用于提供安全的内部可变性。我们已经看到过`UnsafeCell`，它是Rust中内部可变性的底层：它允许您获取对其包括的值的多个可变引用，但只能使用不安全的代码。一个`Cell`就像一个`UnsafeCell`一样，但是它提供了一个安全的接口：它只允许获取当前值的副本或替换当前值，而获取不到引用，并且由于它不满足Sync，因此不能在线程之间共享。这些限制意味着可以安全使用，但是我们不能直接在``static`变量中使用它，因为`static`必须为Sync。

[`Cell`]:https：//doc.rust-lang.org/core/cell/struct.Cell.html

那么，为什么上面的示例起作用？ `Mutex <T>`对要任何实现了`Send`的`T`(比如这里的`Cell`)都实现了`Sync`。它之所以安全，是因为它仅在临界区内允许访问其内容。因此，我们可以实现一个没有任何不安全代码的安全计数器！

这对于像`u32`这样的简单类型非常有用，但是对于没有实现`Copy`的更复杂类型呢？在嵌入式上下文中，一个非常常见的示例是外设结构体，他通常没有实现`Copy`。针对这种,我们可以使用`RefCell`。

## 共享外设

通过强制一次只能存在一个外设实例，使用`svd2rust`生成的`Device` crate和类似抽象提供了对外设的安全访问。这样虽然安全，但是很难同时从主线程和中断处理程序访问外围设备。

为了安全地共享外围设备访问权限，我们可以使用我们刚刚介绍的`Mutex`。我们还需要使用[`RefCell`]，`RefCell`通过运行时检查来确保一次仅给出一个对外设的可变引用。这比普通的`Cell`有更多的开销，由于我们给出的是引用而不是副本，因此我们必须确保一次仅存在一个可变引用。

[`RefCell`]:https://doc.rust-lang.org/core/cell/struct.RefCell.html

最后，在主代码中初始化外设后，我们还必须考虑将外设移入共享变量的方式。为此，我们可以使用`Option`类型，先将其初始化为`None` ，然后再将其设置为外设的实例。

```rust , ignore
use core::cell::RefCell;
use cortex_m::interrupt::{self, Mutex};
use stm32f4::stm32f405;

static MY_GPIO: Mutex<RefCell<Option<stm32f405::GPIOA>>> =
    Mutex::new(RefCell::new(None));

#[entry]
fn main() -> ! {
    // Obtain the peripheral singletons and configure it.
    // This example is from an svd2rust-generated crate, but
    // most embedded device crates will be similar.
    let dp = stm32f405::Peripherals::take().unwrap();
    let gpioa = &dp.GPIOA;

    // Some sort of configuration function.
    // Assume it sets PA0 to an input and PA1 to an output.
    configure_gpio(gpioa);

    // Store the GPIOA in the mutex, moving it.
    interrupt::free(|cs| MY_GPIO.borrow(cs).replace(Some(dp.GPIOA)));
    // We can no longer use `gpioa` or `dp.GPIOA`, and instead have to
    // access it via the mutex.

    // Be careful to enable the interrupt only after setting MY_GPIO:
    // otherwise the interrupt might fire while it still contains None,
    // and as-written (with `unwrap()`), it would panic.
    set_timer_1hz();
    let mut last_state = false;
    loop {
        // We'll now read state as a digital input, via the mutex
        let state = interrupt::free(|cs| {
            let gpioa = MY_GPIO.borrow(cs).borrow();
            gpioa.as_ref().unwrap().idr.read().idr0().bit_is_set()
        });

        if state && !last_state {
            // Set PA1 high if we've seen a rising edge on PA0.
            interrupt::free(|cs| {
                let gpioa = MY_GPIO.borrow(cs).borrow();
                gpioa.as_ref().unwrap().odr.modify(|_, w| w.odr1().set_bit());
            });
        }
        last_state = state;
    }
}

#[interrupt]
fn timer() {
    // This time in the interrupt we'll just clear PA0.
    interrupt::free(|cs| {
        // We can use `unwrap()` because we know the interrupt wasn't enabled
        // until after MY_GPIO was set; otherwise we should handle the potential
        // for a None value.
        let gpioa = MY_GPIO.borrow(cs).borrow();
        gpioa.as_ref().unwrap().odr.modify(|_, w| w.odr1().clear_bit());
    });
}
```

这段代码很复杂,让我们一行一行分析.

```rust , ignore
static MY_GPIO: Mutex<RefCell<Option<stm32f405::GPIOA>>> =
    Mutex::new(RefCell::new(None));
```

现在，我们的共享变量的类型是` Mutex<RefCell<Option<stm32f405::GPIOA>>>`。 `Mutex”`可确保我们仅在临界区内具有访问权限，因此就算是`RefCell`不支持`Sync`,变量`MY_GPIO`也能够支持Sync。 `RefCell`为我们提供了带有引用的内部可变性， `Option`使我们可以先将该变量初始化为空，稍后才将其实际内容移入。我们不能直接使用static的单例`GPIOA`,所有这一切都是必须的。


```rust , ignore
interrupt::free(|cs| MY_GPIO.borrow(cs).replace(Some(dp.GPIOA)));
```

在临界区内，我们可以在互斥锁上调用 `borrow()`，从而获得`RefCell`的引用。然后，我们调用 `replace()`将新值移入`RefCell`。

```rust , ignore
interrupt::free(|cs| {
    let gpioa = MY_GPIO.borrow(cs).borrow();
    gpioa.as_ref().unwrap().odr.modify(|_, w| w.odr1().set_bit());
});
```

终于我们可以安全并且支持并发的使用`MY_GPIO`。临界区阻止了中断的发生，并让我们借用到互斥锁。然后`RefCell`通过`as_ref()`给我们一个`&Option<&GPIOA>` ，并跟踪借用范围--一旦借用结束,`RefCell`会更新其内部的值。
Finally we use `MY_GPIO` in a safe and concurrent fashion. The critical section prevents the interrupt firing as usual, and lets us borrow the mutex.  The `RefCell` then gives us an `&Option<GPIOA>`, and tracks how long it remains borrowed - once that reference goes out of scope, the `RefCell` will be updated to indicate it is no longer borrowed.
todo 感觉这段话是错的,需要验证.
 

由于我们无法将`GPIOA`从`＆Option”`中移出，因此我们需要使用`as_ref()`将其转换为`&Option<&GPIOA>`，最后我们可以通过`unwrap()` 获得到`＆GPIOA`，从而可以修改外设状态。(todo 此处应该是可以访问外设)
todo &GPIOaA是只读借用啊,在怎么修改?

如果我们需要对共享资源的可变引用，则应该使用`borrow_mut` 和 `deref_mut`。以下代码显示了使用TIM2计时器的示例。

```rust , ignore
use core::cell::RefCell;
use core::ops::DerefMut;
use cortex_m::interrupt::{self, Mutex};
use cortex_m::asm::wfi;
use stm32f4::stm32f405;

static G_TIM: Mutex<RefCell<Option<Timer<stm32::TIM2>>>> =
	Mutex::new(RefCell::new(None));

#[entry]
fn main() -> ! {
    let mut cp = cm::Peripherals::take().unwrap();
    let dp = stm32f405::Peripherals::take().unwrap();

    // Some sort of timer configuration function.
    // Assume it configures the TIM2 timer, its NVIC interrupt,
    // and finally starts the timer.
    let tim = configure_timer_interrupt(&mut cp, dp);

    interrupt::free(|cs| {
        G_TIM.borrow(cs).replace(Some(tim));
    });

    loop {
        wfi();
    }
}

#[interrupt]
fn timer() {
    interrupt::free(|cs| {
        if let Some(ref mut tim)) =  G_TIM.borrow(cs).borrow_mut().deref_mut() {
            tim.start(1.hz());
        }
    });
}

```

> **注意**
>
>目前， `cortex-m` crate将某些函数的const版本(包括 `Mutex::new()`)隐藏在`const-fn`特性的后面。因此您需要在Cargo.toml中将`const-fn`特性添加到cortex-m的依赖项，以使上述示例起作用：
>
> ``` toml
> [dependencies.cortex-m]
> version="0.6.0"
> features=["const-fn"]
> ```
>同时，`const-fn`已经在稳定版Rust上工作了一段时间。因此预计这个特性很快会成为`cortex-m`的默认配置,这样以后就不必在Cargo.toml中配置磁特性了。
>

目前这样虽然安全，但还有点笨拙。我们还有什么可以做的吗？

## RTFM

一种替代方法是[RTFM框架],RTFM的全称是Real Time For the Masses。它强制执行静态优先级，并跟踪对`static mut` 变量(“资源”)的访问，以静态地确保始终安全地访问共享资源，而不需要临界区分和使用引用计数(如在“ RefCell”中)的开销。这具有许多优点，例如，确保没有死锁，并提供极低的时间和内存开销。

[RTFM框架]:https://github.com/rtfm-rs/cortex-m-rtfm

该框架还包括其他功能，例如消息传递，可以减少对显式共享状态的需求，还可以计划在给定时间运行的任务，可以用来执行定期任务。请查看[RTFM文档]以获取更多信息！

[RTFM文档]:https://japaric.github.io/cortex-m-rtfm/book/

## 实时操作系统

嵌入式并发的另一个常见模型是实时操作系统(RTOS)。尽管目前在Rust中的研究较少，但它们已广泛用于传统的嵌入式开发中。开源的RTOS有[FreeRTOS]和[ChibiOS]。这些RTOS支持运行多个应用程序线程，线程的调度触发机制包括线程主动出让控制权(称为协作多任务)和基于常规计时器或中断(称为抢占多任务)。 RTOS通常提供互斥锁和其他同步原语，并且通常与DMA引擎等硬件特性进行互操作。

[FreeRTOS]:https://freertos.org/
[ChibiOS]:http：//chibios.org/

在撰写本文时，没有太多的Rust相关的RTOS，但是这是一个有趣的领域，所以请留意这个领域！

## 多核

在嵌入式处理器中拥有两个或多个内核变得越来越普遍，这给并发增加了额外的复杂性。所有使用临界区的示例(包括`cortex_m::interrupt::Mutex`)都假定只有中断线程，但在多核系统上不再如此。因此我们需要为多核专门设计同步原语(对于对称多处理，也称为SMP)。

多核系统通常使用我们之前看到的原子指令，因为处理系统将确保在所有内核上保持原子性。

目前，详细讨论这些主题超出了本书的范围，但是一般模式与单核情况相同。