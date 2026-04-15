# Java 并发编程：AQS、CAS 与锁 | Day 11

> AQS、ReentrantLock、CAS、原子类、死锁、悲观锁与乐观锁

---

## 一、AQS 了解多少？

### 是什么？

```
AQS = AbstractQueuedSynchronizer
抽象队列同步器

是 Java 并发包的底层框架
ReentrantLock、CountDownLatch、Semaphore 等都基于它实现
```

### 核心结构

```
AQS 里有两个核心：

① state（状态变量）
  用 volatile int 表示锁的状态
  0 = 未加锁
  1 = 已加锁
  >1 = 重入次数

② CLH 队列（等待队列）
  抢不到锁的线程排队等待
  是一个双向链表

  Head → [线程A] → [线程B] → [线程C] → null
```

### 加锁流程

```
线程来了
  ↓
CAS 尝试把 state 从 0 改成 1
  ↓
成功 → 拿到锁，继续执行
失败 → 加入 CLH 队列，挂起等待
  ↓
锁被释放 → 唤醒队列头部的线程 → 重新竞争
```

### AQS 两种模式

```
独占模式：同一时间只有一个线程能拿到锁
  → ReentrantLock

共享模式：同一时间多个线程可以拿到锁
  → CountDownLatch、Semaphore
```

---

## 二、ReentrantLock 详解

### 是什么？

```
可重入锁
同一个线程可以多次加锁，不会死锁
每加一次锁，state + 1
每释放一次锁，state - 1
state = 0 才真正释放锁
```

### 公平锁 vs 非公平锁

```java
// 非公平锁（默认）
ReentrantLock lock = new ReentrantLock();

// 公平锁
ReentrantLock lock = new ReentrantLock(true);
```

```
公平锁：严格按队列顺序，先来先得
  → 不会有线程饿死
  → 但吞吐量低（每次都要排队）

非公平锁：新来的线程可以插队
  → 可能有线程一直抢不到（饿死）
  → 但吞吐量高（减少线程切换）
```

### 使用模板

```java
ReentrantLock lock = new ReentrantLock();

lock.lock();
try {
    // 业务逻辑
} finally {
    lock.unlock();  // 必须在 finally 里！
}
```

### 高级功能

```java
// 可中断的加锁（可以响应中断）
lock.lockInterruptibly();

// 尝试加锁（加不上立刻返回，不阻塞）
boolean success = lock.tryLock();

// 超时加锁
boolean success = lock.tryLock(3, TimeUnit.SECONDS);

// 条件变量（比 wait/notify 更灵活）
Condition condition = lock.newCondition();
condition.await();   // 等待
condition.signal();  // 唤醒
```

---

## 三、CAS 了解多少？

### 是什么？

```
CAS = Compare And Swap（比较并交换）

原子操作，底层靠 CPU 指令保证
不需要加锁，性能更好
```

### 原理

```
CAS(内存地址, 期望值, 新值)

步骤：
1. 读取内存里的值
2. 比较是否等于期望值
3. 等于 → 把内存里的值改成新值，返回 true
   不等于 → 不改，返回 false（说明被别人改了）

整个过程是原子的，不可分割
```

### 举例

```
i = 0

线程A：CAS(i, 0, 1)
  读取 i = 0
  期望值 = 0，相等
  改成 1，成功 ✅

线程B：CAS(i, 0, 1)
  读取 i = 1（已经被A改了）
  期望值 = 0，不相等
  失败，重试 🔄
```

### 代码示例

```java
AtomicInteger i = new AtomicInteger(0);

// CAS 操作
boolean success = i.compareAndSet(0, 1);
// 期望值是0，改成1，成功返回true
```

---

## 四、CAS 有什么问题？

### 问题① ABA 问题（最重要）

```
线程A读取 i = A
线程B把 i 改成 B
线程B又把 i 改回 A
线程A：CAS(i, A, newValue)，成功了！

但其实 i 已经被改过了，只是又改回来了
线程A不知道中间发生了什么
```

**解决：加版本号**

```java
// AtomicStampedReference 带版本号的原子引用
AtomicStampedReference<Integer> ref =
    new AtomicStampedReference<>(0, 0);  // (值, 版本号)

// CAS 同时比较值和版本号
ref.compareAndSet(0, 1, 0, 1);
// 期望值=0，新值=1，期望版本号=0，新版本号=1
```

### 问题② 循环时间长，CPU 消耗大

```
CAS 失败后会一直重试（自旋）
如果竞争很激烈，一直失败
CPU 空转，浪费资源

解决：限制自旋次数，超过就挂起线程
```

### 问题③ 只能保证一个变量的原子操作

```
多个变量需要同时修改时，CAS 做不到
需要用锁或者把多个变量封装成一个对象
```

---

## 五、Java 有哪些保证原子性的方法？

```
① synchronized
  加锁，简单粗暴

② ReentrantLock
  比 synchronized 更灵活

③ 原子类（AtomicXxx）
  基于 CAS，不加锁，性能好

④ volatile
  只保证可见性，不保证原子性
  i++ 用 volatile 还是不安全！
```

---

## 六、原子操作类了解多少？

### 分类

```
基本类型：
  AtomicInteger    整数
  AtomicLong       长整数
  AtomicBoolean    布尔值

引用类型：
  AtomicReference          普通引用
  AtomicStampedReference   带版本号（解决ABA）
  AtomicMarkableReference  带标记位

数组类型：
  AtomicIntegerArray
  AtomicLongArray

字段更新器：
  AtomicIntegerFieldUpdater  对某个字段进行原子更新
```

### 常用方法

```java
AtomicInteger i = new AtomicInteger(0);

i.get();              // 获取值
i.set(5);             // 设置值
i.getAndIncrement();  // i++（返回旧值）
i.incrementAndGet();  // ++i（返回新值）
i.getAndAdd(10);      // i += 10（返回旧值）
i.compareAndSet(0, 1); // CAS
```

---

## 七、AtomicInteger 源码

### 核心代码

```java
public class AtomicInteger {
    // volatile 保证可见性
    private volatile int value;

    // 底层用 Unsafe 类实现 CAS
    public final int incrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
    }

    // getAndAddInt 是个自旋操作
    public final int getAndAddInt(Object o, long offset, int delta) {
        int v;
        do {
            v = getIntVolatile(o, offset);  // 读取当前值
        } while (!compareAndSwapInt(o, offset, v, v + delta));  // CAS失败就重试
        return v;
    }
}
```

---

## 八、线程死锁了解吗？

### 是什么？

```
两个线程互相等待对方释放锁
谁都不肯让步，永远卡住

线程A：持有锁1，等待锁2
线程B：持有锁2，等待锁1

→ 永远等下去，死锁！
```

### 死锁的四个必要条件

```
① 互斥：一个锁同时只能被一个线程持有
② 占有并等待：持有锁的同时，等待其他锁
③ 不可剥夺：锁不能被强制抢走，只能自己释放
④ 循环等待：A等B，B等A，形成环

四个条件缺一不可，破坏任意一个就能避免死锁
```

### 代码示例

```java
Object lock1 = new Object();
Object lock2 = new Object();

// 线程A
new Thread(() -> {
    synchronized (lock1) {
        Thread.sleep(100);
        synchronized (lock2) { }  // 等 lock2
    }
}).start();

// 线程B
new Thread(() -> {
    synchronized (lock2) {
        Thread.sleep(100);
        synchronized (lock1) { }  // 等 lock1
    }
}).start();

// 死锁！
```

---

## 九、死锁问题怎么排查？

### 方法① jstack 命令

```bash
# 找到 Java 进程 ID
jps -l

# 查看线程状态
jstack <进程ID>

# 输出里会有：
# Found one Java-level deadlock:
# 直接告诉你死锁了
```

### 方法② VisualVM 工具

```
JDK 自带的图形化工具
可以看到线程状态和死锁信息
```

### 方法③ 代码预防

```
① 固定加锁顺序
  所有线程都按 lock1 → lock2 的顺序加锁
  永远不会形成循环等待

② 使用 tryLock（超时）
  lock.tryLock(3, TimeUnit.SECONDS)
  加不上锁就放弃，不会永远等

③ 减少锁的粒度
  不要在持有锁的情况下调用其他需要锁的方法
```

---

## 十、线程同步和互斥

```
互斥：同一时间只有一个线程访问共享资源
  → 用锁实现

同步：线程之间按顺序协调执行
  → 用 wait/notify 或 Condition 实现
```

```java
// 互斥
synchronized (lock) {
    // 只有一个线程能进来
}

// 同步（生产者消费者）
// 生产者
synchronized (lock) {
    queue.add(item);
    lock.notify();  // 通知消费者
}

// 消费者
synchronized (lock) {
    while (queue.isEmpty()) {
        lock.wait();  // 没数据就等待
    }
    queue.poll();
}
```

---

## 十一、悲观锁和乐观锁 ⭐

### 悲观锁

```
悲观：总觉得会有人来抢，先加锁再操作

代表：synchronized、ReentrantLock

流程：
加锁 → 操作数据 → 释放锁

适合：写多读少，竞争激烈的场景
缺点：加锁解锁有开销，性能差一些
```

### 乐观锁

```
乐观：觉得不会有人来抢，先操作，出问题再处理

代表：CAS、版本号机制

流程：
操作数据 → 提交时检查有没有被改过
  没被改 → 提交成功
  被改了 → 重试或报错

适合：读多写少，竞争不激烈的场景
缺点：竞争激烈时，一直重试，CPU消耗大
```

### 数据库里的乐观锁

```sql
-- 加版本号字段
UPDATE table
SET value = newValue, version = version + 1
WHERE id = 1 AND version = oldVersion

-- 如果 version 被别人改了，更新0行，说明失败，重试
```

### 对比总结

| | 悲观锁 | 乐观锁 |
|---|---|---|
| 思想 | 先锁再操作 | 先操作再验证 |
| 实现 | synchronized、Lock | CAS、版本号 |
| 适合 | 写多竞争激烈 | 读多竞争少 |
| 性能 | 相对低 | 相对高 |
| 问题 | 死锁 | ABA问题 |

---

## 总结

| 概念 | 核心要点 |
|---|---|
| AQS | 并发框架基石，state + CLH队列 |
| ReentrantLock | 可重入，支持公平/非公平，功能比synchronized多 |
| CAS | 无锁原子操作，有ABA、自旋消耗CPU等问题 |
| 原子类 | 基于CAS，线程安全，性能好 |
| 死锁 | 四个必要条件，用jstack排查，固定加锁顺序预防 |
| 悲观锁 | 先加锁，写多场景用 |
| 乐观锁 | 先操作后验证，读多场景用 |
