# 学习目标

使用`esp-generate`创建工程(注意启用embassy异步框架)，并参考embassy同步通信模块的官方文档[embassy_sync - Rust](https://docs.rs/embassy-sync/latest/embassy_sync/)，编写代码并完成使用`Mutex`进行共享数据读写的同步通信示例。

# 完整源码

需先在终端中通过下面命令添加对应依赖，才能导入emabssy的同步通信模块：

```powershell
cargo add embassy-sync
```

```rust
#![no_std]
#![no_main]
#![deny(
    clippy::mem_forget,
    reason = "mem::forget is generally not safe to do with esp_hal types, especially those \
    holding buffers for the duration of a data transfer."
)]
#![deny(clippy::large_stack_frames)]

use esp_hal::{
    clock::CpuClock,
    timer::timg::TimerGroup,
    interrupt::software::SoftwareInterruptControl,
};

use embassy_executor::Spawner;
use embassy_time::{Duration, Timer};
use embassy_sync::{     // 导入emabssy同步通信模块
    blocking_mutex::raw::CriticalSectionRawMutex, 
    mutex::Mutex,
};

use log::info;

use esp_backtrace as _;

extern crate alloc;

// This creates a default app-descriptor required by the esp-idf bootloader.
// For more information see: <https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/app_image_format.html#application-description>
esp_bootloader_esp_idf::esp_app_desc!();

#[embassy_executor::task]
async fn write_task_a(
    shared_data: &'static Mutex<CriticalSectionRawMutex, u32>
) {
    loop {
        Timer::after(Duration::from_millis(1000)).await;
        {
            let mut guard = shared_data.lock().await;
            *guard =  *guard + 1;
            info!("data: {}, written by Task A", *guard);
        }
    }
}

#[embassy_executor::task]
async fn write_task_b(
    shared_data: &'static Mutex<CriticalSectionRawMutex, u32>
) {
    loop {
        Timer::after(Duration::from_millis(1500)).await;
        {
            let mut guard = shared_data.lock().await;
            *guard =  *guard + 1;
            info!("data: {}, written by Task B", *guard);
        }
    }
}

#[allow(
    clippy::large_stack_frames,
    reason = "it's not unusual to allocate larger buffers etc. in main"
)]
#[esp_rtos::main]
async fn main(spawner: Spawner) -> ! {
    // generator version: 1.3.0
    // generator parameters: --chip esp32s3 -o esp32s3-wroom-1-octal-psram -o unstable-hal -o alloc -o embassy -o stack-smashing-protection -o log -o esp-backtrace -o vscode

    esp_println::logger::init_logger_from_env();

    let config = esp_hal::Config::default().with_cpu_clock(CpuClock::max());
    let peripherals = esp_hal::init(config);

    // The following pins are used to bootstrap the chip. They are available
    // for use, but check the datasheet of the module for more information on them.
    // - GPIO0
    // - GPIO3
    // - GPIO45
    // - GPIO46
    // These GPIO pins are in use by some feature of the module and should not be used.
    let _ = peripherals.GPIO27;
    let _ = peripherals.GPIO28;
    let _ = peripherals.GPIO29;
    let _ = peripherals.GPIO30;
    let _ = peripherals.GPIO31;
    let _ = peripherals.GPIO32;
    let _ = peripherals.GPIO33;
    let _ = peripherals.GPIO34;
    let _ = peripherals.GPIO35;
    let _ = peripherals.GPIO36;
    let _ = peripherals.GPIO37;

    esp_alloc::heap_allocator!(#[esp_hal::ram(reclaimed)] size: 73744);

    let timg0 = TimerGroup::new(peripherals.TIMG0);
    let sw_interrupt = SoftwareInterruptControl::new(peripherals.SW_INTERRUPT);
    esp_rtos::start(timg0.timer0, sw_interrupt.software_interrupt0);

    info!("Embassy initialized!");

    static MUTEX: Mutex<CriticalSectionRawMutex, u32> = Mutex::new(0);

    spawner.spawn(write_task_a(&MUTEX).expect("Failed to spawn write_task_a"));
    spawner.spawn(write_task_b(&MUTEX).expect("Failed to spawn write_task_b"));

    loop {
        Timer::after(Duration::from_millis(5000)).await;
    }
    // for inspiration have a look at the examples at https://github.com/esp-rs/esp-hal/tree/esp-hal-v1.1.0/examples
}
```

# 烧录运行

使用下列命令进行编译：

```powershell
cargo build 
```

使用下列命令进行烧录运行：

```powershell
cargo espflash flash --monitor
```

**预期效果：**

通过互斥锁，task_a和task_b交替访问并修改共享数据，同时将数据的值打印出来。

# 代码讲解

## Mutex

用于**保护共享资源，避免资源竞争**的一种同步通信方式，定义如下：

```rust
pub struct Mutex<M, T>
```

其中`M`为锁的类型，`T`为被保护数据的类型。

`Mutex<T>`的本质是一个容器，其内部持有着数据 `T`，但它**本身不提供对数据的直接访问，而通过 `.lock()` 方法来“申请对数据的访问权”**，也就是`MutexGuard<'a, T>`——调用 `.lock()` 后返回的守卫，它是一个智能指针，通过它可以像操作 `&mut T` 一样操作互斥锁的内部保护数据。

守卫是Rust语言在`mutex`安全性设计上的核心机制，不同于C语言中将锁与数据分离，**守卫将"持有锁"和"数据访问权限"绑定到一起，从而保证了“先锁定数据再访问”的操作顺序，同时两者的生命周期也被绑定到一起，当守卫在离开作用域时会被销毁，锁也会随之自动释放，从而避免了C语言中“忘记释放锁”或“锁与数据两者其中之一已失效但另一方仍在使用”的经典并发问题**。

关于互斥锁的创建与使用的简单示例如下：

```rust
static MUTEX: Mutex<CriticalSectionRawMutex, u32> = Mutex::new();

// 异步获取锁，若已被占用则挂起直到锁释放
let mut guard = MUTEX.lock().await;
*guard += 1;
// guard 离开作用域时自动释放锁

// 同步非阻塞尝试获取锁，若已被占用则立即返回 None
if let Some(mut guard) = MUTEX.try_lock() {
    *guard += 1;
}
// guard 离开作用域时自动释放锁
```

此外Mutex还有两个方法，比较适用于不需要锁的场景：

```rust
static MUTEX: Mutex<CriticalSectionRawMutex, u32> = Mutex::new(42);

// 销毁锁并直接取出其内部值，适用于不再需要使用锁了
let value = MUTEX.into_inner();  // mutex被消耗，value = 42
// mutex在这里已经无效，无法再使用

// 直接获取可变引用而不加锁，适用于没有并发的阶段，避免锁的开销
// 如果在编译期就能确保没有并发安全问题，可以通过编译器检查，就不需要在运行时加锁
let value = MUTEX.get_mut();
*value += 1;  

// 需要并发访问时，再加锁
let guard = MUTEX.lock();
```

**`critical_section`与`embassy_sync`的`Mutex`区别：**

在之前的中断学习中，我们也接触过`Mutex`，但那个是`critical_section`的`Mutex`，现在我们所学习的是`embassy_sync`的`Mutex`，两者的区别可见下表：

|           | `critical_section::Mutex`         | `embassy_sync::mutex::Mutex`                          |
| --------- | --------------------------------- | ----------------------------------------------------- |
| **核心机制**  | 通过**禁用中断**（临界区）来保证原子性。            | 内部使用一个**`RawMutex`** 来保护其“锁定状态”标志。                    |
| **同步/异步** | **同步**，获取锁时会**忙等**，直到锁可用。         | **异步**。`lock().await` 在锁被占用时会**挂起当前任务**，让出 CPU，而不是忙等。 |
| **适用场景**  | **任务与中断之间**的同步，或**单核**系统中的共享数据保护。 | **异步任务之间**的同步，特别是在 Embassy 执行器中。                      |
| **中断中使用** | ✅ **可以**。它是为在中断上下文中使用而设计的。        | ❌ **不可以**。中断中不能使用 `.await`。                           |
| **性能开销**  | 锁的获取和释放会**短暂禁用全局中断**，可能影响系统实时性。   | `lock().await` 涉及任务调度和上下文切换，开销比临界区大，但不阻塞其他任务。         |
| **锁的类型**  | 本身就是锁。                            | 是一个**组合式**锁，其行为由传入的 `RawMutex` 类型参数决定。                |
