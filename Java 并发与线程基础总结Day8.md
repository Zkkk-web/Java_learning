# Java 并发与线程基础总结

## 1. 并行（Parallelism）与并发（Concurrency）的区别

- **并发（Concurrency）**：多个任务在同一时间段内交替执行（单核CPU通过时间片切换实现）
- **并行（Parallelism）**：多个任务在同一时刻真正同时执行（需要多核CPU）

👉 简单理解：
- 并发 = 宏观同时
- 并行 = 微观同时

---

## 2. 进程（Process）和线程（Thread）的区别

| 对比项 | 进程 | 线程 |
|--------|------|------|
| 定义 | 程序的一次执行 | 进程中的执行单元 |
| 资源 | 独立分配 | 共享进程资源 |
| 开销 | 大 | 小 |
| 通信 | 复杂（IPC） | 简单（共享内存） |
| 稳定性 | 高 | 低 |

---

## 3. 线程的创建方式

### 1️⃣ 继承 Thread 类
```java
class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Thread running");
    }
}
```

### 2️⃣ 实现 Runnable 接口（推荐）
```java
class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("Runnable running");
    }
}
```

### 3️⃣ 实现 Callable + Future
```java
import java.util.concurrent.Callable;
import java.util.concurrent.FutureTask;

Callable<Integer> task = () -> 1 + 1;
FutureTask<Integer> future = new FutureTask<>(task);
new Thread(future).start();
```

---

## 4. 调用 start() 方法会执行什么？

- `start()`：启动线程，JVM 创建新线程，并自动调用 `run()`
- ❌ 直接调用 `run()`：只是普通方法调用，不会创建新线程

👉 本质流程：
```
start() → 创建新线程 → 调用 run()
```

---

## 5. 线程常用调度方法

- `sleep(long millis)`：线程休眠（不会释放锁）
- `yield()`：让出 CPU（不一定生效）
- `join()`：等待线程执行完成
- `setPriority()`：设置线程优先级（不一定生效）

---

## 6. 线程的状态

Java 线程共有 6 种状态：

1. NEW（新建）
2. RUNNABLE（就绪/运行）
3. BLOCKED（阻塞，等待锁）
4. WAITING（无限等待）
5. TIMED_WAITING（超时等待）
6. TERMINATED（结束）

---

## 7. 什么是线程上下文切换？

👉 定义：
CPU 从一个线程切换到另一个线程时，需要保存当前线程状态并恢复另一个线程状态。

👉 保存内容：
- 程序计数器（PC）
- 寄存器状态
- 栈信息

👉 特点：
- 有性能开销
- 频繁切换会降低效率

---

## 8. 什么是守护线程（Daemon Thread）？

👉 定义：
为用户线程提供服务的后台线程

👉 特点：
- 所有用户线程结束后，守护线程自动结束
- 常见例子：垃圾回收线程（GC）

👉 设置方式：
```java
thread.setDaemon(true);
```

---

## 9. 线程间通信方式

### 1️⃣ 共享变量（需同步控制）
- volatile
- synchronized

### 2️⃣ wait / notify
```java
synchronized(obj) {
    obj.wait();
    obj.notify();
}
```

### 3️⃣ Lock + Condition
```java
import java.util.concurrent.locks.*;

Lock lock = new ReentrantLock();
Condition condition = lock.newCondition();

condition.await();
condition.signal();
```

### 4️⃣ 并发工具类
- BlockingQueue
- CountDownLatch
- CyclicBarrier

---

## 10. sleep 和 wait 的区别

| 对比项 | sleep | wait |
|--------|------|------|
| 所属类 | Thread | Object |
| 是否释放锁 | ❌ 不释放 | ✅ 释放 |
| 唤醒方式 | 时间结束自动恢复 | 需要 notify/notifyAll |
| 使用场景 | 控制执行时间 | 线程通信 |

👉 核心区别：
- sleep：让线程暂停
- wait：用于线程协作

---

# 📌 总结

- 并发和并行是两个不同概念
- 线程是进程的执行单元
- 推荐使用 Runnable 或 Callable 创建线程
- start() 才是真正启动线程的方法
- 线程通信是并发编程核心
