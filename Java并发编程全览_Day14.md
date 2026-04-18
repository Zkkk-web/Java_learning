# Java 并发编程全览 | Day 14

> 多线程基础、锁机制、并发容器、线程池、Unsafe、通信工具类

---

## 一、Java 多线程入门

### 创建线程的三种方式

```java
// 方式1：继承 Thread
class MyThread extends Thread {
    public void run() { System.out.println("线程运行"); }
}
new MyThread().start();

// 方式2：实现 Runnable（推荐）
new Thread(() -> System.out.println("线程运行")).start();

// 方式3：实现 Callable（有返回值）
FutureTask<String> task = new FutureTask<>(() -> "结果");
new Thread(task).start();
String result = task.get();  // 获取结果
```

### 获取线程执行结果

```java
// Future
Future<String> future = executor.submit(() -> "结果");
String result = future.get();         // 阻塞等待
String result = future.get(3, TimeUnit.SECONDS);  // 超时等待

// CompletableFuture（更强大）
CompletableFuture<String> cf = CompletableFuture.supplyAsync(() -> "结果");
cf.thenAccept(result -> System.out.println(result));  // 结果回调
cf.join();  // 等待完成
```

---

## 二、线程的 6 种状态

```
NEW          → 新建，还没 start()
RUNNABLE     → 运行中（包含就绪和运行）
BLOCKED      → 等待 synchronized 锁
WAITING      → 等待（wait()、join()、LockSupport.park()）
TIMED_WAITING → 超时等待（sleep()、wait(timeout)）
TERMINATED   → 执行完毕

状态转换：
NEW → RUNNABLE（start()）
RUNNABLE → BLOCKED（等锁）
RUNNABLE → WAITING（wait()）
RUNNABLE → TIMED_WAITING（sleep()）
RUNNABLE → TERMINATED（执行完）
```

---

## 三、线程组和线程优先级

```java
// 线程优先级（1-10，默认5）
Thread t = new Thread(() -> {});
t.setPriority(Thread.MAX_PRIORITY);  // 10，最高
t.setPriority(Thread.MIN_PRIORITY);  // 1，最低
t.setPriority(Thread.NORM_PRIORITY); // 5，默认

// 线程组
ThreadGroup group = new ThreadGroup("myGroup");
Thread t = new Thread(group, () -> {}, "myThread");

// 注意：优先级只是建议，不保证
// 实际调度由操作系统决定
```

---

## 四、进程与线程的区别

```
进程：资源分配的基本单位
  独立的内存空间
  进程间通信复杂（IPC）
  创建销毁开销大

线程：CPU调度的基本单位
  共享进程的内存空间
  线程间通信简单（共享内存）
  创建销毁开销小

一个进程可以有多个线程
线程是进程内的执行单元
```

---

## 五、线程安全性问题

```
① 原子性：操作不可分割
  i++ 不是原子操作
  解决：synchronized、AtomicInteger

② 可见性：一个线程的修改，其他线程能看到
  工作内存的缓存导致不可见
  解决：volatile、synchronized

③ 有序性：代码按顺序执行
  指令重排破坏有序性
  解决：volatile、happens-before 规则
```

---

## 六、synchronized 的四种锁状态

```
无锁
  ↓（第一个线程来了）
偏向锁
  对象头记录线程ID，同一线程再来不加锁
  ↓（第二个线程来竞争）
轻量级锁
  CAS 自旋竞争，不挂起线程
  ↓（竞争激烈，自旋多次失败）
重量级锁
  线程挂起，OS 调度，开销大

锁只能升级，不能降级
```

### 偏向锁深入

```
偏向锁适合：同一线程反复获取同一把锁
  → 记录线程ID，下次直接进，几乎无开销

偏向锁撤销：
  其他线程来竞争时，才撤销偏向锁
  升级为轻量级锁

JDK15 之后偏向锁被废弃
```

---

## 七、ReentrantReadWriteLock

### 是什么？

```
读写锁：读读不互斥，读写互斥，写写互斥

适合：读多写少的场景
  多个线程可以同时读
  写的时候独占，不能读
```

### 代码示例

```java
ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
ReadLock readLock = rwLock.readLock();
WriteLock writeLock = rwLock.writeLock();

// 读操作
readLock.lock();
try {
    // 可以多个线程同时读
    readData();
} finally {
    readLock.unlock();
}

// 写操作
writeLock.lock();
try {
    // 只能一个线程写
    writeData();
} finally {
    writeLock.unlock();
}
```

### 特点

```
锁降级：持有写锁时可以获取读锁，然后释放写锁
        写锁 → 读锁（可以）

锁升级：持有读锁时不能获取写锁
        读锁 → 写锁（不可以，会死锁）
```

---

## 八、等待通知条件 Condition

### 是什么？

```
比 wait/notify 更灵活的等待通知机制
一个锁可以有多个 Condition
可以精确通知某一组线程
```

### 代码示例（生产者消费者）

```java
ReentrantLock lock = new ReentrantLock();
Condition notFull = lock.newCondition();   // 队列未满的条件
Condition notEmpty = lock.newCondition();  // 队列非空的条件

// 生产者
lock.lock();
try {
    while (队列满了) notFull.await();   // 等待不满
    放入数据();
    notEmpty.signal();  // 通知消费者
} finally {
    lock.unlock();
}

// 消费者
lock.lock();
try {
    while (队列空了) notEmpty.await();  // 等待非空
    取出数据();
    notFull.signal();   // 通知生产者
} finally {
    lock.unlock();
}
```

### wait/notify vs Condition

| | wait/notify | Condition |
|---|---|---|
| 所属 | Object | ReentrantLock |
| 条件队列 | 一个 | 多个 |
| 精确通知 | ❌ | ✅ |
| 超时等待 | ✅ | ✅ |

---

## 九、线程阻塞唤醒类 LockSupport

### 是什么？

```
更底层的线程阻塞/唤醒工具
不需要持有锁
可以先 unpark 再 park（不会死锁）
```

### 代码示例

```java
Thread t = new Thread(() -> {
    System.out.println("线程阻塞前");
    LockSupport.park();  // 阻塞当前线程
    System.out.println("线程被唤醒");
});
t.start();

Thread.sleep(1000);
LockSupport.unpark(t);  // 唤醒指定线程
```

### 对比

```
wait/notify：需要持有锁，notify 必须在 wait 之后
LockSupport：不需要锁，unpark 可以在 park 之前
```

---

## 十、ConcurrentLinkedQueue

```
线程安全的无界非阻塞队列
基于 CAS 实现，不加锁

特点：
  不阻塞（队列空了 poll 返回 null，不等待）
  无界（不指定容量上限）
  适合：高并发、不需要阻塞的场景

vs LinkedBlockingQueue：
  LinkedBlockingQueue：有阻塞，可有界
  ConcurrentLinkedQueue：无阻塞，无界
```

---

## 十一、ScheduledThreadPoolExecutor

### 是什么？

```
支持定时和周期性任务的线程池
底层用 DelayQueue 存储任务
```

### 代码示例

```java
ScheduledExecutorService pool = Executors.newScheduledThreadPool(5);

// 延迟执行（3秒后执行一次）
pool.schedule(() -> doTask(), 3, TimeUnit.SECONDS);

// 固定频率执行（每隔1秒执行一次，从开始时间算）
pool.scheduleAtFixedRate(() -> doTask(), 0, 1, TimeUnit.SECONDS);

// 固定延迟执行（上次执行完后，等1秒再执行）
pool.scheduleWithFixedDelay(() -> doTask(), 0, 1, TimeUnit.SECONDS);
```

### scheduleAtFixedRate vs scheduleWithFixedDelay

```
scheduleAtFixedRate：
  每隔固定时间执行，不管上次有没有执行完
  如果任务执行时间 > 间隔，会连续执行

scheduleWithFixedDelay：
  上次执行完了，等固定时间再执行
  两次执行之间的间隔是固定的
```

---

## 十二、魔法类 Unsafe

### 是什么？

```
JDK 提供的底层操作类
可以直接操作内存、CAS操作、线程操作
绕过 JVM 安全检查

正常代码不能直接用，只有 JDK 内部使用
```

### 主要功能

```
① 内存操作
  allocateMemory()   直接分配堆外内存
  freeMemory()       释放堆外内存
  putXxx/getXxx      直接读写内存

② CAS 操作
  compareAndSwapInt()
  compareAndSwapObject()
  AtomicInteger 底层就是用这个

③ 线程操作
  park()   挂起线程
  unpark() 唤醒线程
  LockSupport 底层就是用这个

④ 对象操作
  allocateInstance()  不调用构造方法创建对象
  objectFieldOffset() 获取字段偏移量
```

---

## 十三、通信工具类汇总

| 工具类 | 用途 | 是否可重用 |
|---|---|---|
| CountDownLatch | 等待N个线程完成 | ❌ |
| CyclicBarrier | N个线程互相等待 | ✅ |
| Semaphore | 限制并发数量 | ✅ |
| Exchanger | 两线程交换数据 | ✅ |
| Phaser | 更灵活的CyclicBarrier | ✅ |

### Phaser（了解）

```java
// 比 CyclicBarrier 更灵活
// 可以动态增减参与者数量
Phaser phaser = new Phaser(3);  // 3个参与者

new Thread(() -> {
    doPhase1();
    phaser.arriveAndAwaitAdvance();  // 等所有人完成第一阶段
    doPhase2();
    phaser.arriveAndAwaitAdvance();  // 等所有人完成第二阶段
}).start();
```

---

## 十四、生产者-消费者模式

### 是什么？

```
生产者：生产数据，放入队列
消费者：从队列取数据，处理

队列起到缓冲作用
生产者和消费者解耦，速度不一致也没关系
```

### 三种实现方式

**方式1：BlockingQueue（最简单）**

```java
BlockingQueue<String> queue = new ArrayBlockingQueue<>(10);

// 生产者
new Thread(() -> {
    while (true) {
        queue.put("数据");  // 满了自动等待
    }
}).start();

// 消费者
new Thread(() -> {
    while (true) {
        String data = queue.take();  // 空了自动等待
        process(data);
    }
}).start();
```

**方式2：wait/notify**

```java
Queue<String> queue = new LinkedList<>();
int MAX = 10;

// 生产者
synchronized (queue) {
    while (queue.size() == MAX) queue.wait();
    queue.add("数据");
    queue.notifyAll();
}

// 消费者
synchronized (queue) {
    while (queue.isEmpty()) queue.wait();
    String data = queue.poll();
    queue.notifyAll();
}
```

**方式3：Condition（精确通知）**

```java
// 用 notFull.signal() 只唤醒生产者
// 用 notEmpty.signal() 只唤醒消费者
// 比 notifyAll 效率更高
```

---

## 十五、锁分类汇总（JUC）

```
按使用方式：
  隐式锁：synchronized（自动加锁解锁）
  显式锁：ReentrantLock（手动加锁解锁）

按公平性：
  公平锁：按队列顺序，先来先得
  非公平锁：可以插队（默认）

按读写：
  排他锁：只有一个线程能访问
  读写锁：读读共享，读写/写写互斥

按重入：
  可重入锁：同一线程可以多次获取（ReentrantLock）
  不可重入锁：同一线程再次获取会死锁

按乐观/悲观：
  乐观锁：CAS，先操作后验证
  悲观锁：synchronized/Lock，先锁再操作
```

---

## 总结

```
线程基础：
  6种状态、进程vs线程、创建方式

内存模型：
  JMM、volatile、happens-before

锁：
  synchronized四种状态、偏向锁
  ReentrantLock、读写锁
  Condition、LockSupport

并发容器：
  ConcurrentHashMap、CopyOnWriteArrayList
  ConcurrentLinkedQueue、BlockingQueue

线程池：
  ThreadPoolExecutor、ScheduledThreadPoolExecutor
  7大参数、4种拒绝策略

工具类：
  CountDownLatch、CyclicBarrier、Semaphore
  Exchanger、Phaser

底层：
  CAS、AQS、Unsafe、AtomicXxx

模式：
  生产者-消费者
```
