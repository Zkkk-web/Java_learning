# AQS、锁机制、CAS 与并发工具类 | Day 17

> AQS、ReentrantLock、CAS、原子类、死锁、悲观乐观锁、并发工具类

---

## 一、AQS 了解多少？⭐

### 是什么？

```
AQS = AbstractQueuedSynchronizer（抽象队列同步器）

Java 并发包的底层框架
ReentrantLock、Semaphore、CountDownLatch 等都基于它实现
```

### 核心结构

```
① state（同步状态）
  volatile int state
  0 = 未加锁
  1 = 已加锁
  >1 = 重入次数

② CLH 队列（等待队列）
  双向链表，存放等待获取锁的线程

  Head → [Node(线程A)] → [Node(线程B)] → [Node(线程C)] → null

  Node 的状态：
    CANCELLED：线程取消等待
    SIGNAL：后继节点需要被唤醒
    CONDITION：在条件队列中等待
    PROPAGATE：共享模式下传播唤醒
```

### 两种模式

```
独占模式：同一时间只有一个线程持有锁
  → ReentrantLock

共享模式：同一时间多个线程可以持有
  → CountDownLatch、Semaphore、ReadLock
```

### 加锁流程

```
线程尝试获取锁
  ↓
CAS 把 state 从 0 改为 1
  ↓
成功 → 设置当前线程为持有者，进入临界区
失败 → 加入 CLH 队列尾部，挂起等待
  ↓
前驱节点释放锁 → 唤醒队列头部线程 → 重新竞争
```

### 解锁流程

```
state - 1
state = 0 → 真正释放锁
唤醒 CLH 队列头部的线程
```

---

## 二、ReentrantLock 详解 ⭐

### 基本特性

```
可重入：同一线程多次加锁，state 递增
  每次 lock() → state + 1
  每次 unlock() → state - 1
  state = 0 才真正释放

基于 AQS 实现：
  Sync 继承 AQS
  FairSync（公平锁）
  NonfairSync（非公平锁）
```

### 公平锁 vs 非公平锁

```java
// 非公平锁（默认）
ReentrantLock lock = new ReentrantLock();

// 公平锁
ReentrantLock lock = new ReentrantLock(true);
```

```
公平锁：
  新来的线程必须排队，不能插队
  严格按 CLH 队列顺序
  优点：不会饿死
  缺点：吞吐量低，线程切换频繁

非公平锁：
  新来的线程可以直接 CAS 抢锁
  抢不到再排队
  优点：吞吐量高，减少切换
  缺点：可能有线程一直抢不到（饿死）
```

### 高级功能

```java
// 可中断加锁
lock.lockInterruptibly();

// 尝试加锁（立即返回）
boolean got = lock.tryLock();

// 超时加锁
boolean got = lock.tryLock(3, TimeUnit.SECONDS);

// 多条件变量
Condition c1 = lock.newCondition();
Condition c2 = lock.newCondition();
c1.await();    // 等待条件1
c2.signal();   // 唤醒条件2 的线程
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

---

## 三、ReentrantLock 怎么创建公平/非公平锁？

```java
// 底层源码
public ReentrantLock() {
    sync = new NonfairSync();  // 默认非公平
}

public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}

// 公平锁：tryAcquire 时先检查队列，有等待线程就排队
// 非公平锁：直接 CAS 抢，抢不到再排队
```

---

## 四、CAS 了解多少？⭐

### 是什么？

```
CAS = Compare And Swap（比较并交换）

原子操作，底层靠 CPU 的 CMPXCHG 指令
不需要加锁，性能比 synchronized 好
```

### 原理

```
CAS(内存地址V, 期望值A, 新值B)

① 读取 V 的当前值
② 如果当前值 == A → 把 V 改成 B，返回 true
   如果当前值 != A → 不改，返回 false（说明被别人改了）

整个过程由 CPU 保证原子性
```

### Java 中的 CAS

```java
// Unsafe 类提供底层 CAS
unsafe.compareAndSwapInt(obj, offset, expect, update);

// AtomicInteger 的 incrementAndGet 底层
public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}

// getAndAddInt 自旋
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        v = getIntVolatile(o, offset);  // 读当前值
    } while (!compareAndSwapInt(o, offset, v, v + delta));  // 失败就重试
    return v;
}
```

---

## 五、CAS 有什么问题？⭐

### 问题① ABA 问题

```
线程A读取 i = A
线程B把 i 改成 B
线程B又把 i 改回 A
线程A：CAS(i, A, newValue)，成功！

但 i 已经被改过了，线程A不知道
```

**解决：版本号**

```java
// AtomicStampedReference（值+版本号）
AtomicStampedReference<Integer> ref =
    new AtomicStampedReference<>(0, 0);

// 同时比较值和版本号
ref.compareAndSet(
    0,   // 期望值
    1,   // 新值
    0,   // 期望版本号
    1    // 新版本号
);
```

### 问题② 自旋消耗 CPU

```
CAS 失败会不断重试（自旋）
竞争激烈时，CPU 空转严重

解决：
  限制自旋次数
  或者用锁（竞争激烈时锁反而更好）
```

### 问题③ 只能操作一个变量

```
CAS 一次只能保证一个变量的原子操作
多个变量需要同时修改：
  用 AtomicReference 把多个变量封装成一个对象
  或者用锁
```

---

## 六、Java 有哪些保证原子性的方法？

```
① synchronized
  加锁，保证代码块原子执行

② ReentrantLock
  和 synchronized 效果类似，更灵活

③ 原子类（AtomicXxx）
  基于 CAS，无锁，性能好
  AtomicInteger、AtomicLong、AtomicReference

④ LongAdder（高并发下比 AtomicLong 更快）
  把一个变量拆成多个 Cell 分散竞争
  最终求和得到结果
```

---

## 七、原子操作类了解多少？

### 分类

```
基本类型：
  AtomicInteger、AtomicLong、AtomicBoolean

引用类型：
  AtomicReference
  AtomicStampedReference   解决 ABA（带版本号）
  AtomicMarkableReference  带标记位

数组类型：
  AtomicIntegerArray、AtomicLongArray

字段更新器：
  AtomicIntegerFieldUpdater  对普通 volatile int 字段原子更新

高性能累加器：
  LongAdder（高并发累加推荐）
  LongAccumulator
```

### 常用方法

```java
AtomicInteger i = new AtomicInteger(0);

i.get();               // 获取值
i.set(5);              // 设置值
i.getAndIncrement();   // i++，返回旧值
i.incrementAndGet();   // ++i，返回新值
i.getAndAdd(10);       // i+=10，返回旧值
i.addAndGet(10);       // i+=10，返回新值
i.compareAndSet(0, 1); // CAS，成功返回true
```

---

## 八、AtomicInteger 源码分析

```java
public class AtomicInteger extends Number {
    // Unsafe 实例
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    // value 字段在对象中的内存偏移量
    private static final long valueOffset;
    // volatile 保证可见性
    private volatile int value;

    static {
        // 初始化偏移量
        valueOffset = unsafe.objectFieldOffset(
            AtomicInteger.class.getDeclaredField("value"));
    }

    // 自旋 CAS 实现
    public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }
}
```

---

## 九、线程死锁了解吗？

### 定义

```
两个或多个线程互相等待对方释放锁
谁都不肯让步，永远卡住
```

### 死锁四个必要条件

```
① 互斥：一个锁同时只有一个线程持有
② 占有并等待：持有锁的同时，等待其他锁
③ 不可剥夺：锁不能被强制抢走，只能主动释放
④ 循环等待：A等B，B等A，形成环路

破坏任意一个 → 避免死锁
```

### 代码示例

```java
Object lock1 = new Object();
Object lock2 = new Object();

// 线程A：先锁1，再锁2
new Thread(() -> {
    synchronized (lock1) {
        sleep(100);
        synchronized (lock2) { }  // 等 lock2
    }
}).start();

// 线程B：先锁2，再锁1
new Thread(() -> {
    synchronized (lock2) {
        sleep(100);
        synchronized (lock1) { }  // 等 lock1
    }
}).start();
// 死锁！
```

---

## 十、死锁问题怎么排查？⭐

### 方法① jstack

```bash
# 1. 找进程ID
jps -l

# 2. 查看线程状态
jstack <pid>

# 输出里会有：
# Found one Java-level deadlock:
# 直接告诉你哪两个线程死锁了，持有什么锁，等待什么锁
```

### 方法② VisualVM / JConsole

```
JDK 自带图形化工具
Thread → 检测死锁 → 直接显示
```

### 预防死锁

```
① 固定加锁顺序
  所有线程都按 lock1 → lock2 的顺序加锁
  不会形成循环等待

② 超时放弃
  lock.tryLock(3, TimeUnit.SECONDS)
  超时就放弃，不死等

③ 减少锁的粒度
  避免持有锁时调用其他需要锁的方法

④ 使用无锁数据结构
  ConcurrentHashMap、AtomicInteger 等
```

---

## 十一、线程同步和互斥

```
互斥：同一时间只有一个线程访问资源
  → 保证数据安全
  → 用锁实现

同步：线程按照特定顺序执行
  → 协调线程之间的协作
  → 用 wait/notify 或 Condition 实现

互斥是同步的一种特殊情况
```

---

## 十二、悲观锁和乐观锁 ⭐

### 悲观锁

```
思想：总觉得有人来抢，先加锁再操作

代表：synchronized、ReentrantLock

流程：
  加锁 → 操作数据 → 释放锁

适合：写多读少，竞争激烈
缺点：加锁开销大，性能相对低
```

### 乐观锁

```
思想：觉得不会有人来抢，先操作，提交时再检查

代表：CAS、版本号机制

流程：
  读数据 → 操作数据 → 提交时验证有没有被改过
  没被改 → 提交成功
  被改了 → 重试或报错

适合：读多写少，竞争不激烈
缺点：竞争激烈时，频繁重试浪费 CPU
```

### 数据库中的乐观锁

```sql
-- 加版本号字段
UPDATE orders
SET status = 'paid', version = version + 1
WHERE id = 1 AND version = 3;  -- 检查版本号

-- 如果 version 不是3，更新0行，说明被别人改了，需要重试
```

---

## 十三、CountDownLatch 了解吗？

```
倒计时门闩
设置数字N，每完成一件事 N-1
N=0 时放行所有等待线程
```

```java
CountDownLatch latch = new CountDownLatch(3);

// 3个子线程各自执行完后 countDown
new Thread(() -> { doTask(); latch.countDown(); }).start();
new Thread(() -> { doTask(); latch.countDown(); }).start();
new Thread(() -> { doTask(); latch.countDown(); }).start();

// 主线程等待3个都完成
latch.await();
System.out.println("所有任务完成！");
```

**特点：一次性，N到0后不能重置**

---

## 十四、CyclicBarrier 了解吗？

```
循环屏障
N个线程都到达屏障后，一起放行
可以重置，循环使用
```

```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    System.out.println("所有线程到达，继续！");
});

// 每个线程到达后等待其他线程
new Thread(() -> {
    doPhase1();
    barrier.await();  // 等其他人
    doPhase2();
    barrier.await();  // 可以重复使用
}).start();
```

---

## 十五、CyclicBarrier 和 CountDownLatch 的区别

| | CountDownLatch | CyclicBarrier |
|---|---|---|
| 计数 | 倒计时（countDown） | 累加（await等N个） |
| 可重用 | ❌ 一次性 | ✅ 可重置 |
| 等待方式 | 一方等多方完成 | 多方互相等待 |
| 到达后 | 放行等待线程 | 所有线程一起继续 |
| 适合 | 等待多任务完成 | 多阶段同步 |

---

## 十六、Semaphore 了解吗？

```
信号量，控制同时访问的线程数量

类比停车场：只有N个车位
  acquire() → 占一个位（没有就等）
  release() → 释放一个位
```

```java
Semaphore semaphore = new Semaphore(5);  // 最多5个并发

new Thread(() -> {
    semaphore.acquire();  // 获取许可
    try {
        accessResource();
    } finally {
        semaphore.release();  // 释放许可
    }
}).start();
```

**适合：限流、资源池控制**

---

## 十七、Exchanger 了解吗？

```
两个线程之间交换数据
两个线程都到达交换点后，互换数据
```

```java
Exchanger<String> exchanger = new Exchanger<>();

new Thread(() -> {
    String result = exchanger.exchange("A的数据");
    System.out.println("A收到：" + result);  // B的数据
}).start();

new Thread(() -> {
    String result = exchanger.exchange("B的数据");
    System.out.println("B收到：" + result);  // A的数据
}).start();
```

---

## 十八、ConcurrentHashMap ⭐

### JDK7 实现

```
分段锁（Segment）
把数组分成16段，每段有独立的锁
A操作第1段，B操作第2段，互不影响
并发度 = 16
```

### JDK8 实现

```
CAS + synchronized（锁单个桶）
格子为空 → CAS 直接放（不加锁）
格子有元素 → synchronized 锁这个格子
数据结构：数组 + 链表 + 红黑树（链表≥8转红黑树）
并发度 = 数组长度，更高
```

### key 和 value 为什么不能为 null？

```
多线程下 map.get(key) 返回 null 有歧义：
  ① key 不存在
  ② key 存在但 value 是 null

单线程可以用 containsKey 再确认
多线程下 containsKey 和 get 之间可能被修改
这种歧义在多线程下不可接受

所以直接禁止 null
```

---

## 总结

| 概念 | 核心要点 |
|---|---|
| AQS | state + CLH队列，独占/共享两种模式 |
| ReentrantLock | 可重入，公平/非公平，比synchronized更灵活 |
| CAS | 无锁原子操作，ABA用版本号解决 |
| 原子类 | 基于CAS，AtomicInteger、LongAdder |
| 死锁 | 四个必要条件，jstack排查，固定顺序预防 |
| 悲观锁 | 先加锁，适合写多 |
| 乐观锁 | 先操作后验证，适合读多 |
| CountDownLatch | 等N个完成，一次性 |
| CyclicBarrier | N个互相等，可重用 |
| Semaphore | 限流，控制并发数 |
| ConcurrentHashMap | JDK8用CAS+synchronized锁单桶 |
