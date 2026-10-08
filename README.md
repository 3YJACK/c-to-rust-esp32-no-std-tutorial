# c-to-rust-esp32-no-std-tutorial

C语言嵌入式开发者的个人Rust no_std嵌入式开发的学习记录，同时也是一份面向有着相似背景开发者的Rust no_std嵌入式开发入门教程。

以 ESP32S3 为硬件平台，从语法基础、环境搭建、外设驱动，一路讲到 embassy 异步并发，每章笔记都配有可运行源码。

## 目录结构

```
.
├── Notes/                          # 学习笔记（教程主体）
│   ├── 00语法基础.md                # Rust 语法最小必要知识
│   ├── 01环境搭建.md                # Rust + ESP32 工具链安装
│   ├── 02工程创建.md                # 使用 esp-generate 创建工程
│   ├── 03template.md               # 工程模板结构解析
│   ├── 04GPIO点灯.md                # GPIO 输出，点亮 LED
│   ├── 05uart串口.md                # UART 串口通信
│   ├── 06interrupt引脚中断.md        # 外部中断
│   ├── 07timer定时器.md             # 硬件定时器
│   ├── 08i2c-oled.md               # I2C 驱动 OLED 显示
│   ├── 09async异步.md               # 异步编程概念
│   ├── 10async-blink.md            # embassy 异步点灯
│   ├── 11async-priority.md         # 异步任务优先级
│   ├── 12async-signal&watch.md     # Signal 与 Watch 通信
│   ├── 13async-channel.md          # Channel 通信
│   └── 14async-mutex.md            # Mutex 共享数据
│
├── Code/                           # 对应笔记的可运行工程
│   ├── 03template/                 # 工程模板
│   ├── 04GPIO-blink/               # GPIO 点灯
│   ├── 05uart/                     # UART 串口
│   ├── 06interrupt/                # 引脚中断
│   ├── 07timer/                    # 定时器
│   ├── 08i2c-oled/                 # I2C OLED
│   ├── 10async-blink/             # 异步点灯
│   ├── 11async-priority/          # 异步优先级
│   ├── 12async-signal&watch/      # Signal & Watch
│   ├── 13async-channel/           # Channel
│   └── 14async-mutex/             # Mutex
│
├── Cargo.toml                      
└── README.md
```

## 学习路径

| 章节  | 笔记                        | 工程                      | 知识点                                         |
| --- | ------------------------- | ----------------------- | ------------------------------------------- |
| 00  | `00语法基础.md`               | —                       | Rust 语法最小必要知识：变量、函数、所有权、模块、泛型、枚举            |
| 01  | `01环境搭建.md`               | —                       | rustup / espup / espflash / esp-generate 安装 |
| 02  | `02工程创建.md`               | —                       | 使用 esp-generate 创建 no_std 工程                |
| 03  | `03template.md`           | `03template/`           | 工程模板结构、Cargo.toml、build.rs                  |
| 04  | `04GPIO点灯.md`             | `04GPIO-blink/`         | GPIO 输出、点亮 LED                              |
| 05  | `05uart串口.md`             | `05uart/`               | UART 串口通信、日志输出                              |
| 06  | `06interrupt引脚中断.md`      | `06interrupt/`          | 外部中断、中断处理                                   |
| 07  | `07timer定时器.md`           | `07timer/`              | 硬件定时器                                       |
| 08  | `08i2c-oled.md`           | `08i2c-oled/`           | I2C 总线、SSD1306 OLED 显示                      |
| 09  | `09async异步.md`            | —                       | 异步编程概念：Future / Waker / Executor            |
| 10  | `10async-blink.md`        | `10async-blink/`        | embassy 框架、异步点灯                             |
| 11  | `11async-priority.md`     | `11async-priority/`     | 异步任务优先级                                     |
| 12  | `12async-signal&watch.md` | `12async-signal&watch/` | Signal 信号、Watch 监视                          |
| 13  | `13async-channel.md`      | `13async-channel/`      | Channel 通道通信                                |
| 14  | `14async-mutex.md`        | `14async-mutex/`        | Mutex 异步互斥、共享数据安全                           |

> **建议从第 00 章开始顺序学习。** 如果已有 Rust 基础，可跳过 00 章直接从 01 章开始。

## 前置要求

### 硬件

-  ESP32S3 开发板（本项目使用 ESP32S3-WROOM-1 模组）

### 软件

- **Rust 工具链**：通过 [rustup](https://rustup.rs) 安装，项目通过 `rust-toolchain.toml` 自动管理版本
- **ESP32 工具链**：`espup`（Xtensa Rust 编译器）、`espflash`（烧录工具）、`esp-generate`（工程生成）
- **VS Code**（推荐）+ rust-analyzer 扩展

详细安装步骤见 [01环境搭建.md](Notes/01环境搭建.md)。

### 知识背景

- C 语言基础（必要）
- 嵌入式开发经验，了解 GPIO / UART / I2C / 中断 / 定时器等基本概念（必要）
- Rust 基础（非必要——第 00 章提供最小必要知识，但建议配合 [The Rust Programming Language](https://doc.rust-lang.org/stable/book/) 学习）

## 关于笔记

笔记是本教程的主体。每篇笔记的结构通常为：

1. **学习目标**——本章要做什么、要学什么
2. **概念讲解**——"是什么"和"与 C 的对比"
3. **完整源码**——配合 `./Code/` 中对应工程，可编译运行
4. **要点总结**——关键 API 和踩坑提示

笔记中的代码示例可直接在对应的 `./Code/` 工程中找到完整版本。建议边读笔记边对照源码，遇到不理解的地方直接修改源码跑一跑。

# 
