# 学习目标

使用`esp-generate`创建工程并参考`esp-rs/esp-hal`仓库的`./example/interrupt/uart`示例，编写代码并实现简单UART串口通信功能。 

**前置知识：**

本篇内容建议在掌握了[00语法基础](./00语法基础.md)中 **<u>枚举类型</u>** 的 **<u>Result</u>** 之后再进行学习。

# 完整源码

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
    main,
    time::{Duration, Instant},
    uart::*, // includes Uart Module 
};

use log::info;

use esp_backtrace as _;

extern crate alloc;

// This creates a default app-descriptor required by the esp-idf bootloader.
// For more information see: <https://docs.espressif.com/projects/esp-idf/en/stable/esp32/api-reference/system/app_image_format.html#application-description>
esp_bootloader_esp_idf::esp_app_desc!();

#[allow(
    clippy::large_stack_frames,
    reason = "it's not unusual to allocate larger buffers etc. in main"
)]
#[main]
fn main() -> ! {
    // generator version: 1.3.0
    // generator parameters: --chip esp32s3 -o esp32s3-wroom-1-octal-psram -o unstable-hal -o alloc -o stack-smashing-protection -o log -o esp-backtrace -o vscode

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

    // uart initialization
    let uart_config = Config::default()
        .with_baudrate(115200)
        .with_data_bits(DataBits::_8)
        .with_parity(Parity::None)
        .with_stop_bits(StopBits::_1);

    let mut uart = Uart::new(peripherals.UART1, uart_config)
        .expect("Failed to initialize UART")    
        .with_tx(peripherals.GPIO43)
        .with_rx(peripherals.GPIO44);   

    let message = b"Hello, UART!\n";

    // delay initialization
    let delay = esp_hal::delay::Delay::new();

    loop {
        uart.write(message);
        delay.delay_millis(1000);
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

成功烧录并运行程序后，日志输出信息应每秒打印一次 *“Hello, UART!”*。

# 代码讲解

## UART

UART的初始化流程与上一篇的GPIO如出一辙：

**创建默认配置→修改配置结构体字段→创建串口对象，绑定引脚**

```rust
    // uart initialization
    let uart_config = Config::default()
        .with_baudrate(115200)
        .with_data_bits(DataBits::_8)
        .with_parity(Parity::None)
        .with_stop_bits(StopBits::_1);

    let mut uart = Uart::new(peripherals.UART1, uart_config)
        .expect("Failed to initialize UART")    
        .with_tx(peripherals.GPIO43)
        .with_rx(peripherals.GPIO44);   
```

完成初始化即可操作UART收发数据，相关的函数方法，数据类型等待都可在官方文档[esp_hal::uart - Rust](https://docs.espressif.com/projects/rust/esp-hal/1.1.0/esp32s3/esp_hal/uart/index.html)中查阅。

## Result

在上示的UART初始化代码片段中，`Uart::new()`返回的是一个`Result`类型，其中包含着UART的初始化结果。如果初始化成功，则可以从`Result`中取出相应的UART实例，否之取出的是UART的错误码，关于错误码具体可见官方文档中的`uart`模块中的`ConfigError`。

```rust
    let mut uart = Uart::new(peripherals.UART1, uart_config)
        .expect("Failed to initialize UART")    
```

示例中使用了`.expect()`方法对结果进行展开，如果结果为错误的话还会打印消息*"Failed to initialize UART"*。

## 字节切片

```rust
let message = b"Hello, UART!\n";
```

这里在`Hello UART!`前面加个`b`是将字符串切片`&str`转换成字节切片`&[u8]`，等效于下示代码：

```rust
let msg:&str = "Hello";
let bytes: &[u8] = msg.as_bytes(); // 得到 [72, 101, 108, 108, 111]
```

两者的区别在于：

- **`&str`（字符串切片）**：**必须**是有效的 **UTF-8** 编码。它只能指向合法的、符合 UTF-8 标准的字符序列。

- **`&[u8]`（字节切片）**：**可以是任何数据**。它只是一段内存的字节集合，不关心这些字节代表什么。它可以是 UTF-8 文本、图片的二进制数据、传感器原始读数，或者是乱码。

串口通信时要求使用的是字节切片`&[u8]`，而其本身传输的数据也是字节流。
