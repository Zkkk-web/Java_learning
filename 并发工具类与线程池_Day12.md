# Java 并发工具类与线程池 | Day 12

> CountDownLatch、CyclicBarrier、Semaphore、ConcurrentHashMap、BlockingQueue、线程池

---

## 一、CountDownLatch 了解吗？

### 是什么？

```
倒计时门闩

设置一个数字 N
每完成一件事，N - 1
N = 0 时，等待的线程全部放行
```

### 生活举例

```
运动会：裁判等所有运动员跑完才宣布成绩
  运动员1跑完 → N-1
  运动员2跑完 → N-1
  ...
  所有人跑完（N=0）→ 裁判宣布成绩
```

### 代码示例

```java
CountDownLatch latch = new CountDownLatch(3);  // N=3

// 三个线程各自执行
new Thread(() -> {
    doSomething();
    latch.countDown();  // 完成一个，N-1
}).start();

// 主线程等待所有线程完成
latch.await();  // 阻塞，直到 N=0
System.out.println("所有线程完成！");
```

### 特点

```
一次性的，N减到0就不能重置
适合：等待多个线程都完成后再继续
```

---

## 二、CyclicBarrier 了解吗？

### 是什么？

```
循环屏障

设置一个数字 N
N 个线程都到达屏障后，一起放行
然后屏障重置，可以再次使用（循环）
```

### 生活举例

```
团建活动：等所有人都到齐了才出发
  第1个人到 → 等
  第2个人到 → 等
  ...
  第N个人到 → 一起出发！
  然后重置，下一个地点继续等
```

### 代码示例

```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    System.out.println("所有线程到达，一起继续！");  // 可选，到齐后执行
});

new Thread(() -> {
    doPhase1();
    barrier.await();  // 等其他线程到达
    doPhase2();
    barrier.await();  // 可以重复使用！
}).start();
```

---

## 三、CyclicBarrier 和 CountDownLatch 的区别

| | CountDownLatch | CyclicBarrier |
|---|---|---|
| 计数方式 | 倒计时（countDown减1） | 累加（等N个到达） |
| 是否可重用 | ❌ 一次性 | ✅ 可重置循环使用 |
| 等待对象 | 一个线程等多个线程完成 | 多个线程互相等待 |
| 到达后 | 放行等待的线程 | 所有线程一起继续 |
| 适合场景 | 等待多任务完成 | 多线程分阶段同步 |

---

## 四、Semaphore 了解吗？

### 是什么？

```
信号量，控制同时访问的线程数量
```

### 生活举例

```
停车场只有3个车位
  来一辆车 → 占一个位（acquire）
  走一辆车 → 释放一个位（release）
  3个位都满了 → 后来的车等待
```

### 代码示例

```java
Semaphore semaphore = new Semaphore(3);  // 最多3个线程同时执行

new Thread(() -> {
    semaphore.acquire();  // 获取许可，没有就等
    try {
        doSomething();
    } finally {
        semaphore.release();  // 释放许可
    }
}).start();
```

### 适合场景

```
限流：控制并发数量
  比如：数据库连接池最多10个连接
        接口最多同时处理100个请求
```

---

## 五、Exchanger 了解吗？

### 是什么？

```
两个线程之间交换数据
两个线程都到达交换点后，互换数据
```

### 代码示例

```java
Exchanger<String> exchanger = new Exchanger<>();

// 线程A
new Thread(() -> {
    String result = exchanger.exchange("线程A的数据");
    System.out.println("A收到：" + result);  // 线程B的数据
}).start();

// 线程B
new Thread(() -> {
    String result = exchanger.exchange("线程B的数据");
    System.out.println("B收到：" + result);  // 线程A的数据
}).start();
```

---

## 六、ConcurrentHashMap 详解 ⭐

### 为什么用它？

```
HashMap 线程不安全
Hashtable 线程安全但全部加锁，性能差
ConcurrentHashMap 线程安全且性能好
```

### JDK7 实现：分段锁

```
把数组分成 16 个 Segment（段）
每个 Segment 是一个小 HashMap，有自己的锁

A操作第1段 → 只锁第1段
B操作第2段 → 只锁第2段
互不影响 ✅

并发度 = 16（最多16个线程同时写）
```

### JDK8 实现：CAS + synchronized

```
不分段了，直接锁单个桶（格子）

格子为空 → 用 CAS 放入（不加锁）
格子有元素 → 用 synchronized 锁这个格子

并发度更高，粒度更细
```

### JDK7 vs JDK8 对比

| | JDK7 | JDK8 |
|---|---|---|
| 结构 | 分段数组+链表 | 数组+链表+红黑树 |
| 锁 | 分段锁（Segment） | CAS+synchronized |
| 并发度 | 固定16 | 等于数组长度 |
| 性能 | 一般 | 更好 |

---

## 七、ConcurrentHashMap 怎么保证复合操作原子性？

### 问题

```java
// 这两行合起来不是原子操作！
if (!map.containsKey(key)) {
    map.put(key, value);
}
// 两行之间可能被其他线程插入
```

### 解决

```java
// 用 putIfAbsent（原子操作）
map.putIfAbsent(key, value);

// 或者 computeIfAbsent
map.computeIfAbsent(key, k -> new Value());
```

---

## 八、为什么 ConcurrentHashMap 的 key 和 value 不能为 null？

```
HashMap 允许 key 和 value 为 null
ConcurrentHashMap 不允许！

原因：
多线程环境下，map.get(key) 返回 null
你不知道是：
  ① key 不存在
  ② key 存在但 value 是 null

单线程可以用 containsKey 再检查
多线程下，containsKey 和 get 之间可能被修改
这种模糊性在多线程下会造成歧义

所以直接禁止 null，避免这个问题
```

---

## 九、CopyOnWriteArrayList 详解

### 原理

```
读：直接读，不加锁
写：复制一份新数组，在新数组上改，改完替换

保证读操作永远不被阻塞
写操作之间互相加锁
```

### 适合场景

```
读多写少
比如：配置信息、白名单、黑名单
```

### 缺点

```
① 内存占用：写时复制，同时存两份数组
② 数据一致性：读的可能是旧数据
   （刚写完还没替换，另一个线程读到的是旧的）
```

---

## 十、BlockingQueue 详解

### 是什么？

```
阻塞队列
队列满了 → put 操作阻塞，等有空位
队列空了 → take 操作阻塞，等有数据
```

### 常用方法

```java
// 阻塞方法（推荐）
queue.put(item);    // 放入，满了就等
queue.take();       // 取出，空了就等

// 非阻塞方法
queue.offer(item);  // 放入，满了返回false
queue.poll();       // 取出，空了返回null

// 查看队头（不取出）
queue.peek();
```

### 常见实现类

```
ArrayBlockingQueue   → 有界队列，数组实现，必须指定容量
LinkedBlockingQueue  → 可有界可无界，链表实现
PriorityBlockingQueue → 优先级队列，按优先级出队
SynchronousQueue    → 没有容量，put必须等take，一对一传递
DelayQueue          → 延迟队列，到期才能取出
```

### 经典应用：生产者消费者

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

---

## 十一、什么是线程池？⭐

### 为什么需要线程池？

```
每次 new Thread() 都要：
  创建线程（消耗资源）
  执行任务
  销毁线程（消耗资源）

线程池：
  提前创建好一批线程
  任务来了直接用，用完放回去
  不需要反复创建销毁
```

### 好处

```
① 降低资源消耗：复用线程，减少创建销毁开销
② 提高响应速度：任务来了直接有线程可用
③ 便于管理：统一管理线程数量，防止无限创建
```

---

## 十二、项目中怎么用线程池？

```java
// 创建线程池
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    5,                      // 核心线程数
    10,                     // 最大线程数
    60,                     // 空闲线程存活时间
    TimeUnit.SECONDS,       // 时间单位
    new ArrayBlockingQueue<>(100),  // 任务队列
    new ThreadPoolExecutor.CallerRunsPolicy()  // 拒绝策略
);

// 提交任务
executor.execute(() -> {
    doSomething();
});

// 关闭线程池
executor.shutdown();
```

---

## 十三、线程池的工作流程 ⭐

```
任务来了
  ↓
核心线程数没满？
  是 → 创建核心线程执行
  否 ↓
任务队列没满？
  是 → 放入队列等待
  否 ↓
最大线程数没满？
  是 → 创建非核心线程执行
  否 ↓
执行拒绝策略
```

### 图示

```
[任务] → 核心线程（5个）→ 满了 → 队列（100个）→ 满了 → 非核心线程（5个）→ 满了 → 拒绝
```

### 七大参数

```java
new ThreadPoolExecutor(
    int corePoolSize,        // 核心线程数，一直存活
    int maximumPoolSize,     // 最大线程数
    long keepAliveTime,      // 非核心线程空闲多久销毁
    TimeUnit unit,           // 时间单位
    BlockingQueue workQueue, // 任务队列
    ThreadFactory threadFactory,     // 线程工厂（可选）
    RejectedExecutionHandler handler // 拒绝策略
)
```

### 四种拒绝策略

```
AbortPolicy（默认）：直接抛出异常
CallerRunsPolicy：让调用者线程自己执行
DiscardPolicy：直接丢弃任务，不报错
DiscardOldestPolicy：丢弃队列最老的任务，把新任务放入
```

---

## 总结

| 工具 | 用途 |
|---|---|
| CountDownLatch | 等待多个线程完成（一次性） |
| CyclicBarrier | 多线程互相等待，可重用 |
| Semaphore | 限制并发数量 |
| Exchanger | 两个线程互换数据 |
| ConcurrentHashMap | 线程安全的HashMap |
| CopyOnWriteArrayList | 读多写少的线程安全List |
| BlockingQueue | 生产者消费者，线程池任务队列 |
| 线程池 | 复用线程，提高性能 |

### 线程池参数设置经验

```
CPU密集型任务（计算多）：
  核心线程数 = CPU核心数 + 1

IO密集型任务（网络、数据库）：
  核心线程数 = CPU核心数 × 2
```
