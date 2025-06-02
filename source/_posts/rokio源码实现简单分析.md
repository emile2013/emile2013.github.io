---
title: rokio源码实现简单分析
date: 2025-06-02 16:41:48
tags:
    - rust
---

待完成


## 前言
![](https://github.com/emile2013/emile2013.github.io/blob/master/imgs/tokio_classes.png?raw=true) 
Tokio 是一个 Rust 的异步运行时，它提供了一个完整的异步 I/O 框架。其实现在AI Code分析工具，比如Cursor、windsurf等基本都能分析出tokio核心实现，本文并不做八股文总结，仅尝试从任务调度实现角度分析，给出我们可以借鉴的设计思想，其中包括：
- 工作窃取实现
- 任务调度优化
- 阻塞任务实现

先来看tokio简单示例：
- 创建运行时
```rust
let rt = Runtime::builder()
    .worker_threads(4)
    .build()
    .unwrap();
```

- 提交任务
```rust
rt.spawn(async {
    println!("Running on worker thread");
});
```

- 提交阻塞任务
```rust
rt.spawn_blocking(|| {
    // 执行阻塞操作
});
```
<!-- more -->
## 整体架构
作为异步 I/O 框架，其核心架构包含以下几个主要组件：
### 核心组件
```rust
pub struct Runtime {
    /// 调度器
    scheduler: Scheduler,
    /// 运行时句柄
    handle: Handle,
    /// 阻塞线程池
    blocking_pool: BlockingPool,
}
```

### 调度器类型
```rust
pub enum Scheduler {
    /// 单线程调度器
    CurrentThread(CurrentThread),
    /// 多线程调度器
    MultiThread(MultiThread),
}
```

### Worker
每一个Worker对应一个线程，其实也可以称Worker线程。
```rust
pub(super) struct Worker {
    /// 调度器句柄
    handle: Arc<Handle>,
    /// Worker 索引
    index: usize,
    /// 核心数据结构
    core: AtomicCell<Core>,
}

pub(super) struct Core {
    /// 本地任务队列
    run_queue: queue::Local,
    /// LIFO 槽位
    lifo_slot: Option<Notified>,
    /// 是否正在搜索任务
    is_searching: bool,
    /// 是否已关闭
    is_shutdown: bool,
}
```

### 任务队列
```rust
pub(super) struct Shared {
    /// 全局任务队列
    pub(super) inject: inject::Shared,
    /// 远程 Worker 列表
    pub(super) remotes: Box<[Remote]>,
    /// 空闲 Worker 管理
    pub(super) idle: Idle,
    /// 调度器配置
    pub(super) config: Config,
}
```

## 工作窃取算法

### 任务调度流程
```rust
impl Worker {
    fn get_next_task(&mut self) -> Option<Notified> {
        // 1. 检查本地队列
        if let Some(task) = self.core.run_queue.pop() {
            return Some(task);
        }

        // 2. 检查全局注入队列
        if let Some(task) = self.handle.shared.inject.pop() {
            return Some(task);
        }

        // 3. 尝试从其他 Worker 窃取
        self.steal_work()
    }
}
```

### 工作窃取实现
```rust
impl Worker {
    fn steal_work(&self) -> Option<Notified> {
        let remotes = &self.handle.shared.remotes;
        
        for remote in remotes {
            if let Some(task) = remote.steal.steal_into(&mut self.core.run_queue) {
                return Some(task);
            }
        }
        
        None
    }
}
```

## 任务调度优化

### LIFO 优化
```rust
impl Core {
    // 使用 LIFO 槽位优化最近提交的任务
    lifo_slot: Option<Notified>,
}
```

### 批量处理
```rust
impl Worker {
    fn steal_work(&self) -> Option<Notified> {
        let mut batch = Vec::new();
        if let Some(remote) = self.select_remote() {
            remote.steal.steal_into_batch(&mut batch);
        }
        batch.pop()
    }
}
```

## 阻塞任务处理

### 阻塞线程池
```rust
pub(crate) struct BlockingPool {
    spawner: Spawner,
    shutdown_rx: shutdown::Receiver,
}

#[derive(Clone)]
pub(crate) struct Spawner {
    inner: Arc<Inner>,
}
```

### 任务提交
```rust
impl Spawner {
    pub(crate) fn spawn_blocking<F, R>(&self, rt: &Handle, func: F) -> JoinHandle<R>
    where
        F: FnOnce() -> R + Send + 'static,
        R: Send + 'static,
    {
        let (join_handle, spawn_result) = self.spawn_blocking_inner(
            func,
            Mandatory::NonMandatory,
            SpawnMeta::new_unnamed(fn_size),
            rt,
        );
        // ...
    }
}
```

