# 并发容器与线程池深入 | Day 18

> ConcurrentHashMap、CopyOnWriteArrayList、BlockingQueue、线程池全解、并发容器框架

---

## 一、为什么 ConcurrentHashMap 的 size 不准确？

```
size() 方法返回的是一个估算值

原因：
  多线程同时 put/remove
  统计期间数据在不断变化
  无法保证 size() 是精确的实时值

JDK8 的解决方案：
  用 CounterCell 数组分散计数
  每个线程更新自己的 CounterCell
  size() 时把所有 CounterCell 加起来
  减少竞争，但仍是估算值

如果需要精确计数：
  用 AtomicInteger 单独维护计数器
```

---

## 二、CopyOnWriteArrayList 详解

### 原理

```
读：直接读，不加锁，性能极好
写：
  ① 加锁（ReentrantLock）
  ② 复制一份新数组
  ③ 在新数组上修改
  ④ 将引用指向新数组
  ⑤ 解锁
```

### 代码示例

```java
CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();

// 写操作（加锁+复制）
list.add("A");
list.remove("A");

// 读操作（无锁）
list.get(0);
for (String s : list) { }  // 遍历快照，不会抛异常
```

### 优缺点

```
优点：
  读完全不加锁，读性能极高
  遍历时不会 ConcurrentModificationException
  因为遍历的是旧快照

缺点：
  写时复制，内存占用翻倍
  数据一致性弱：读的可能是旧数据
  写性能差，不适合写多场景
```

### 适合场景

```
读多写少：配置文件、白名单、黑名单
写很少，读很频繁
对数据实时性要求不高
```

---

## 三、BlockingQueue 详解 ⭐

### 是什么？

```
阻塞队列
  队列满了 → put() 阻塞，等有空位
  队列空了 → take() 阻塞，等有数据

天然适合生产者-消费者模式
```

### 四组方法

```
操作    抛异常       返回值（不阻塞）   阻塞        超时
插入    add(e)      offer(e)          put(e)      offer(e,t,unit)
删除    remove()    poll()            take()      poll(t,unit)
查看    element()   peek()            -           -
```

### 常见实现类

```
ArrayBlockingQueue
  有界，数组实现，必须指定容量
  公平/非公平可选（默认非公平）
  适合：固定容量的生产消费

LinkedBlockingQueue
  可有界可无界（默认 Integer.MAX_VALUE）
  链表实现，吞吐量比 Array 高
  ⚠️ 不指定容量可能 OOM

SynchronousQueue
  没有容量，不存储数据
  put 必须等 take，一对一传递
  CachedThreadPool 使用它

PriorityBlockingQueue
  无界，按优先级出队
  元素需实现 Comparable

DelayQueue
  无界，延迟队列，到期才能取出
  适合：定时任务、缓存过期
```

### 生产者消费者

```java
BlockingQueue<String> queue = new ArrayBlockingQueue<>(10);

// 生产者
new Thread(() -> {
    while (true) {
        queue.put("数据");      // 满了自动等待
        Thread.sleep(100);
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

## 四、什么是线程池？⭐

### 为什么需要？

```
没有线程池：
  new Thread() → 执行 → 销毁
  频繁创建销毁，开销大，响应慢

有了线程池：
  提前创建好一批线程
  任务来了直接用，用完放回池子
  复用线程，快速响应
```

### 三大好处

```
① 降低资源消耗：复用线程，减少创建销毁
② 提高响应速度：任务来了直接有线程可用
③ 便于管理：统一控制线程数量，防止无限创建
```

---

## 五、线程池的工作流程 ⭐

```
任务来了
  ↓
核心线程数未满？
  是 → 创建核心线程执行任务
  否 ↓
任务队列未满？
  是 → 放入队列等待
  否 ↓
最大线程数未满？
  是 → 创建非核心线程执行
  否 ↓
执行拒绝策略

图示：
[任务] → [核心线程(5)] → 满 → [队列(100)] → 满 → [非核心线程(5)] → 满 → [拒绝]
```

---

## 六、线程池的主要参数 ⭐

```java
new ThreadPoolExecutor(
    5,                              // ① corePoolSize 核心线程数
    10,                             // ② maximumPoolSize 最大线程数
    60,                             // ③ keepAliveTime 非核心线程存活时间
    TimeUnit.SECONDS,               // ④ unit 时间单位
    new ArrayBlockingQueue<>(100),  // ⑤ workQueue 任务队列
    Executors.defaultThreadFactory(), // ⑥ threadFactory 线程工厂
    new AbortPolicy()               // ⑦ handler 拒绝策略
);
```

| 参数 | 说明 |
|---|---|
| corePoolSize | 核心线程数，一直存活 |
| maximumPoolSize | 最大线程数，队列满了才创建 |
| keepAliveTime | 非核心线程空闲多久销毁 |
| workQueue | 任务等待队列 |
| threadFactory | 创建线程的工厂，可自定义线程名 |
| handler | 队列满、线程满时的处理策略 |

---

## 七、线程池的拒绝策略 ⭐

```
① AbortPolicy（默认）
   直接抛出 RejectedExecutionException
   调用方知道任务被拒绝

② CallerRunsPolicy
   让提交任务的线程自己执行
   不丢任务，起到限流效果
   适合：不允许丢任务的场景

③ DiscardPolicy
   直接丢弃，不报错不通知
   适合：允许丢失的场景

④ DiscardOldestPolicy
   丢弃队列里最老的任务
   把新任务放入队列

⑤ 自定义（推荐）
```

```java
// 自定义拒绝策略：记录日志+存库
executor.setRejectedExecutionHandler((r, executor) -> {
    log.warn("任务被拒绝，记录到数据库重试");
    db.save(r);
});
```

---

## 八、线程池有哪几种阻塞队列？

```
ArrayBlockingQueue  → 有界，推荐使用
LinkedBlockingQueue → 可有界可无界（注意OOM）
SynchronousQueue    → 无容量，CachedThreadPool用
PriorityBlockingQueue → 优先级队列，无界
DelayQueue          → 延迟队列，定时任务
```

---

## 九、execute 和 submit 的区别

```java
// execute：Runnable，无返回值
executor.execute(() -> doTask());

// submit：Runnable/Callable，有返回值
Future<String> future = executor.submit(() -> "结果");
String result = future.get();  // 阻塞获取
```

| | execute | submit |
|---|---|---|
| 参数 | Runnable | Runnable/Callable |
| 返回值 | 无 | Future |
| 异常 | 直接抛出 | 封装在 Future 里 |

---

## 十、线程池怎么关闭？

```java
// 温柔关闭（推荐）
executor.shutdown();
// 不接受新任务，等队列里的任务执行完

// 强制关闭
List<Runnable> unfinished = executor.shutdownNow();
// 不接受新任务，中断执行中的任务
// 返回未执行的任务列表

// 等待关闭完成
boolean done = executor.awaitTermination(60, TimeUnit.SECONDS);
```

---

## 十一、线程数怎么配置？⭐

```
CPU 密集型（大量计算，少IO）：
  核心线程数 = CPU核心数 + 1
  原因：CPU满负荷运行，多一个防止偶尔停顿

IO 密集型（数据库、网络、文件）：
  核心线程数 = CPU核心数 × 2
  原因：IO期间CPU空闲，可以让更多线程用CPU

混合型：
  核心线程数 = CPU核心数 × (1 + 等待时间/计算时间)

查看CPU核心数：
  int cores = Runtime.getRuntime().availableProcessors();
```

---

## 十二、四种常见线程池

```java
// ① FixedThreadPool（固定数量）
ExecutorService pool = Executors.newFixedThreadPool(10);
// 核心=最大=10，LinkedBlockingQueue（无界）⚠️OOM风险

// ② CachedThreadPool（可缓存）
ExecutorService pool = Executors.newCachedThreadPool();
// 核心=0，最大=Integer.MAX_VALUE ⚠️线程太多OOM
// SynchronousQueue，线程空闲60秒销毁

// ③ SingleThreadExecutor（单线程）
ExecutorService pool = Executors.newSingleThreadExecutor();
// 只有1个线程，顺序执行
// LinkedBlockingQueue（无界）⚠️OOM风险

// ④ ScheduledThreadPool（定时）
ScheduledExecutorService pool = Executors.newScheduledThreadPool(5);
pool.scheduleAtFixedRate(task, 0, 1, TimeUnit.SECONDS);
```

### 为什么不推荐 Executors？

```
Fixed/Single → 无界队列，任务堆积OOM
Cached → 线程无上限，创建太多OOM

推荐：手动 new ThreadPoolExecutor，明确每个参数
```

---

## 十三、线程池异常怎么处理？

```java
// 方式1：execute + try-catch（推荐）
executor.execute(() -> {
    try {
        doTask();
    } catch (Exception e) {
        log.error("任务异常", e);
    }
});

// 方式2：submit + future.get()
Future<?> future = executor.submit(() -> doTask());
try {
    future.get();
} catch (ExecutionException e) {
    log.error("任务异常", e.getCause());
}

// 方式3：自定义 ThreadFactory 设置异常处理器
ThreadFactory factory = r -> {
    Thread t = new Thread(r);
    t.setUncaughtExceptionHandler((thread, e) -> {
        log.error("未捕获异常", e);
    });
    return t;
};
```

---

## 十四、线程池有几种状态？

```
RUNNING     → 正常运行，接受新任务，处理队列任务
SHUTDOWN    → shutdown() 后，不接受新任务，处理完队列
STOP        → shutdownNow() 后，不接受，中断所有
TIDYING     → 所有任务结束，线程数=0，准备terminated
TERMINATED  → terminated() 执行完，彻底结束

转换：
RUNNING → SHUTDOWN（shutdown()）
RUNNING → STOP（shutdownNow()）
SHUTDOWN → TIDYING（队列空了，线程空了）
STOP → TIDYING（线程空了）
TIDYING → TERMINATED
```

---

## 十五、线程池如何实现参数动态修改？

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(...);

// 动态修改核心线程数
executor.setCorePoolSize(20);

// 动态修改最大线程数
executor.setMaximumPoolSize(50);

// 动态修改存活时间
executor.setKeepAliveTime(30, TimeUnit.SECONDS);

// 动态修改拒绝策略
executor.setRejectedExecutionHandler(new CallerRunsPolicy());
```

```
实际项目中：
  把参数存配置中心（Nacos、Apollo）
  监听配置变化，动态下发
  业务高峰扩容，低谷缩容
```

---

## 十六、线程池调优 ⭐

### 监控指标

```java
// 活跃线程数（正在执行任务的）
executor.getActiveCount();

// 队列积压数
executor.getQueue().size();

// 历史最大线程数
executor.getLargestPoolSize();

// 完成的任务总数
executor.getCompletedTaskCount();

// 总任务数（完成+队列+执行中）
executor.getTaskCount();
```

### 调优方向

```
队列积压严重 → 增加核心线程数 或 增大队列
频繁拒绝    → 增大最大线程数 或 换更大队列
CPU飙高     → 减少线程数，减少上下文切换
响应慢      → 检查是否有任务阻塞了线程

原则：
  先压测，再根据数据调整
  不要靠感觉设置参数
```

---

## 十七、设计一个线程池（手写思路）

```java
public class MyThreadPool {
    private final int coreSize;
    private final int maxSize;
    private final BlockingQueue<Runnable> queue;
    private final List<Thread> workers = new ArrayList<>();

    public MyThreadPool(int coreSize, int maxSize, int queueSize) {
        this.coreSize = coreSize;
        this.maxSize = maxSize;
        this.queue = new ArrayBlockingQueue<>(queueSize);

        // 初始化核心线程
        for (int i = 0; i < coreSize; i++) {
            Thread t = new Thread(() -> {
                while (true) {
                    try {
                        Runnable task = queue.take();  // 没任务就等
                        task.run();
                    } catch (InterruptedException e) {
                        break;
                    }
                }
            });
            t.start();
            workers.add(t);
        }
    }

    public void execute(Runnable task) {
        if (!queue.offer(task)) {
            // 队列满了，执行拒绝策略
            throw new RuntimeException("任务队列已满");
        }
    }
}
```

---

## 十八、线程池执行中断电了怎么办？

```
任务丢失！线程池内存数据不持久化

解决方案：

① 任务持久化到数据库
  提交任务 → 先存库（状态=待执行）
  执行完   → 更新状态（状态=已完成）
  重启后   → 捞出待执行任务重新提交

② 消息队列（推荐）
  任务发到 MQ（Kafka/RocketMQ）
  MQ 持久化任务
  线程池从 MQ 消费
  断电重启，未消费的任务还在 MQ

③ 幂等设计
  任务可以重复执行，结果一样
  不怕重启后重试
```

---

## 十九、并发容器有哪些？

```
List：
  CopyOnWriteArrayList         读多写少

Map：
  ConcurrentHashMap            线程安全HashMap（最常用）
  ConcurrentSkipListMap        有序，跳表实现，O(logN)

Set：
  CopyOnWriteArraySet          基于CopyOnWriteArrayList
  ConcurrentSkipListSet        有序Set

Queue：
  ArrayBlockingQueue           有界阻塞
  LinkedBlockingQueue          链表阻塞（可无界）
  ConcurrentLinkedQueue        无界非阻塞，CAS实现
  PriorityBlockingQueue        优先级阻塞
  DelayQueue                   延迟队列
  SynchronousQueue             无容量，一对一
```

---

## 二十、Java 的并发关键字

```
synchronized  → 加锁，保证原子性+可见性+有序性
volatile      → 可见性+禁止重排，不保证原子性
final         → 不可变，天然线程安全
              → final 字段在构造方法完成后对所有线程可见
```

---

## 总结

| 知识点 | 核心要点 |
|---|---|
| CopyOnWriteArrayList | 写时复制，读无锁，读多写少 |
| BlockingQueue | 满了阻塞写，空了阻塞读，生产消费利器 |
| 线程池工作流程 | 核心→队列→最大→拒绝 |
| 七大参数 | 核心数、最大数、存活时间、队列、工厂、拒绝策略 |
| 四种拒绝策略 | Abort、CallerRuns、Discard、DiscardOldest |
| 线程数配置 | CPU密集=核心数+1，IO密集=核心数×2 |
| 不推荐Executors | 无界队列/无界线程数，有OOM风险 |
| 断电恢复 | 数据库持久化 或 消息队列 |
| 并发容器 | ConcurrentHashMap最常用，按场景选择 |
