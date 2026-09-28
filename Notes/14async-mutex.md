# 学习目标

使用`esp-generate`创建工程并参考`esp-rs/esp-hal`仓库的`./example/interrupt/gpio`示例，编写代码并实现

**前置知识：**

# 完整源码

**引脚连接参照表：**

| 外设  | 对应引脚 |
| --- | ---- |
|     |      |

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

# 代码讲解

## Mutex

用于**保护共享资源，避免资源竞争**的一种同步通信方式，定义如下：

```rust
pub struct Mutex<M, T>
```

其中`M`为锁的类型，`T`为被保护数据的类型。

`Mutex<T>`的本质是一个容器，其内部持有着数据 `T`，但它**本身不提供对数据的直接访问，而通过 `.lock()` 方法来“申请对数据的访问权”**，也就是`MutexGuard<'a, T>`——调用 `.lock()` 后返回的守卫，它是一个智能指针，通过它可以像操作 `&mut T` 一样操作互斥锁的内部保护数据。**守卫离开作用域时会被销毁，锁也会随之自动释放。**

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

critical_section与embassy_sync

freertos与embassy的区别
