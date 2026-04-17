# Java 线程池详解与并发容器 | Day 13

> 线程池参数、拒绝策略、阻塞队列、并发容器、Fork/Join

---

## 一、线程池的主要参数 ⭐

```java
new ThreadPoolExecutor(
    int corePoolSize,         // ① 核心线程数
    int maximumPoolSize,      // ② 最大线程数
    long keepAliveTime,       // ③ 非核心线程存活时间
    TimeUnit unit,            // ④ 时间单位
    BlockingQueue workQueue,  // ⑤ 任务队列
    ThreadFactory threadFactory,      // ⑥ 线程工厂
    RejectedExecutionHandler handler  // ⑦ 拒绝策略
)
```

### 每个参数的意思

```
① 核心线程数：一直存活，即使没有任务也不销毁
② 最大线程数：队列满了才创建，不能超过这个数
③ 存活时间：非核心线程空闲超过这个时间就销毁
④ 时间单位：秒、毫秒等
⑤ 任务队列：核心线程都忙时，任务放这里等待
⑥ 线程工厂：创建线程的方式（可以自定义线程名）
⑦ 拒绝策略：队列满了、线程也满了，怎么处理新任务
```

---

## 二、线程池的拒绝策略 ⭐

### 四种内置策略

```
① AbortPolicy（默认）
   直接抛出 RejectedExecutionException 异常
   调用者知道任务被拒绝了

② CallerRunsPolicy
   让提交任务的线程自己执行这个任务
   不丢任务，但会降低提交速度（起到限流效果）

③ DiscardPolicy
   直接丢弃任务，不报错不通知
   适合允许丢失的场景

④ DiscardOldestPolicy
   丢弃队列里最老的任务（等最久的）
   把新任务放进去
```

### 自定义拒绝策略

```java
// 实现 RejectedExecutionHandler 接口
new RejectedExecutionHandler() {
    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
        // 可以记录日志、存数据库、发告警等
        log.warn("任务被拒绝：" + r.toString());
    }
}
```

---

## 三、线程池有哪几种阻塞队列？

```
① ArrayBlockingQueue
   有界队列，数组实现
   必须指定容量
   公平/非公平可选

② LinkedBlockingQueue
   可有界可无界，链表实现
   不指定容量 = 无界（Integer.MAX_VALUE）
   ⚠️ 危险：任务无限堆积，OOM（内存溢出）

③ SynchronousQueue
   没有容量，不存储任务
   put 必须等 take，一对一传递
   CachedThreadPool 用它

④ PriorityBlockingQueue
   无界，按优先级出队
   优先级高的先执行

⑤ DelayQueue
   延迟队列，到期才能取出
   适合定时任务
```

---

## 四、execute 和 submit 的区别

```java
// execute：提交 Runnable，没有返回值
executor.execute(() -> {
    doSomething();
});

// submit：提交 Callable，有返回值
Future<String> future = executor.submit(() -> {
    return "结果";
});
String result = future.get();  // 获取结果，会阻塞
```

| | execute | submit |
|---|---|---|
| 参数 | Runnable | Runnable 或 Callable |
| 返回值 | 无 | Future |
| 异常处理 | 直接抛出 | 封装在 Future 里 |

---

## 五、线程池怎么关闭？

```java
// 方式1：shutdown（推荐）
executor.shutdown();
// 不再接受新任务
// 等待队列里的任务执行完再关闭
// 温柔关闭

// 方式2：shutdownNow
executor.shutdownNow();
// 不再接受新任务
// 尝试中断正在执行的任务
// 返回队列里未执行的任务列表
// 强制关闭

// 等待关闭完成
executor.awaitTermination(60, TimeUnit.SECONDS);
```

---

## 六、线程池的线程数怎么配置？

### 核心公式

```
CPU 密集型（大量计算，很少等待）：
  核心线程数 = CPU核心数 + 1
  原因：CPU一直在跑，多一个线程防止偶尔停顿

IO 密集型（网络请求、数据库、文件读写）：
  核心线程数 = CPU核心数 × 2
  原因：IO期间CPU空闲，可以让更多线程利用CPU

混合型：
  核心线程数 = CPU核心数 × (1 + 等待时间/计算时间)
```

### 查看CPU核心数

```java
int cpuCores = Runtime.getRuntime().availableProcessors();
```

---

## 七、四种常见线程池

```java
// ① FixedThreadPool：固定数量的线程池
ExecutorService pool = Executors.newFixedThreadPool(10);
// 核心线程=最大线程=10
// 队列：LinkedBlockingQueue（无界）⚠️ 可能OOM

// ② CachedThreadPool：可缓存线程池
ExecutorService pool = Executors.newCachedThreadPool();
// 核心线程=0，最大线程=Integer.MAX_VALUE ⚠️ 可能创建太多线程
// 队列：SynchronousQueue
// 空闲60秒销毁线程

// ③ SingleThreadExecutor：单线程线程池
ExecutorService pool = Executors.newSingleThreadExecutor();
// 只有1个线程，保证顺序执行
// 队列：LinkedBlockingQueue（无界）⚠️ 可能OOM

// ④ ScheduledThreadPool：定时线程池
ScheduledExecutorService pool = Executors.newScheduledThreadPool(5);
// 支持定时和周期性任务
pool.scheduleAtFixedRate(task, 0, 1, TimeUnit.SECONDS);
```

### 为什么不推荐用 Executors 创建？

```
FixedThreadPool / SingleThreadExecutor
  → 队列无界，任务堆积可能 OOM

CachedThreadPool
  → 最大线程数无限，可能创建大量线程 OOM

推荐：手动创建 ThreadPoolExecutor，明确每个参数
```

---

## 八、线程池异常怎么处理？

```java
// 方式1：execute + try-catch
executor.execute(() -> {
    try {
        doSomething();
    } catch (Exception e) {
        log.error("任务异常", e);
    }
});

// 方式2：submit + Future.get()
Future<?> future = executor.submit(() -> doSomething());
try {
    future.get();
} catch (ExecutionException e) {
    log.error("任务异常", e.getCause());
}

// 方式3：自定义 ThreadFactory，设置异常处理器
ThreadFactory factory = r -> {
    Thread t = new Thread(r);
    t.setUncaughtExceptionHandler((thread, e) -> {
        log.error("线程异常", e);
    });
    return t;
};
```

---

## 九、线程池有几种状态？

```
RUNNING    → 正常运行，接受新任务
SHUTDOWN   → 调用 shutdown()，不接受新任务，处理完队列里的任务
STOP       → 调用 shutdownNow()，不接受新任务，中断所有任务
TIDYING    → 所有任务终止，线程数为0，即将调用 terminated()
TERMINATED → terminated() 执行完，彻底结束

状态转换：
RUNNING → SHUTDOWN（shutdown()）
RUNNING → STOP（shutdownNow()）
SHUTDOWN → TIDYING（队列和线程都空了）
STOP → TIDYING（线程都空了）
TIDYING → TERMINATED（terminated()执行完）
```

---

## 十、线程池怎么实现参数动态修改？

```java
ThreadPoolExecutor executor = new ThreadPoolExecutor(...);

// 动态修改核心线程数
executor.setCorePoolSize(20);

// 动态修改最大线程数
executor.setMaximumPoolSize(50);

// 动态修改存活时间
executor.setKeepAliveTime(30, TimeUnit.SECONDS);
```

> 实际项目中可以把参数存数据库，通过配置中心动态下发

---

## 十一、线程池调优了解吗？

```
① 监控指标
  活跃线程数：executor.getActiveCount()
  队列大小：executor.getQueue().size()
  完成任务数：executor.getCompletedTaskCount()
  拒绝次数：自定义拒绝策略时统计

② 调优方向
  队列积压 → 增大线程数或队列容量
  线程数太多 → 减少线程数，降低上下文切换
  频繁拒绝 → 增大队列或线程数

③ 压测
  先压测，再根据数据调整参数
  不要凭感觉设置
```

---

## 十二、线程池使用注意事项

```
① 不要用 Executors 创建，手动创建 ThreadPoolExecutor
② 线程池不用了要关闭（shutdown）
③ 任务里要有异常处理，不能让异常吞掉
④ 核心线程数和最大线程数根据业务类型设置
⑤ 队列用有界队列，避免 OOM
⑥ 给线程起有意义的名字，方便排查问题
   threadFactory = new ThreadFactoryBuilder().setNameFormat("order-pool-%d").build()
```

---

## 十三、线程池执行中断电了怎么办？

```
任务丢失了！线程池不持久化任务

解决方案：
① 把任务存数据库
  提交任务时，先存库（状态=待执行）
  执行完更新状态（状态=已完成）
  重启时，捞出待执行的任务重新提交

② 用消息队列（MQ）
  任务发到 MQ，MQ 持久化
  线程池从 MQ 消费任务
  断电重启后，未消费的任务还在 MQ 里

③ 幂等设计
  任务可以重复执行，结果一样
  不怕重试
```

---

## 十四、并发容器有哪些？

```
List：
  CopyOnWriteArrayList    读多写少

Map：
  ConcurrentHashMap       最常用，线程安全HashMap
  ConcurrentSkipListMap   有序，跳表实现

Set：
  CopyOnWriteArraySet     基于CopyOnWriteArrayList
  ConcurrentSkipListSet   有序

Queue：
  ArrayBlockingQueue      有界阻塞队列
  LinkedBlockingQueue     链表阻塞队列
  ConcurrentLinkedQueue   无界非阻塞队列
  PriorityBlockingQueue   优先级队列
  DelayQueue              延迟队列
  SynchronousQueue        无容量队列
```

---

## 十五、Java 的并发关键字

```
synchronized  → 加锁，保证原子性、可见性、有序性
volatile      → 保证可见性、禁止重排，不保证原子性
final         → 不可变，天然线程安全
```

---

## 十六、Fork/Join 框架了解吗？

### 是什么？

```
分而治之的并行计算框架

把大任务拆成小任务（Fork）
小任务并行执行
把结果合并（Join）

适合：可以递归拆分的计算密集型任务
```

### 代码示例（求1到100的和）

```java
class SumTask extends RecursiveTask<Long> {
    private long start, end;

    SumTask(long start, long end) {
        this.start = start;
        this.end = end;
    }

    @Override
    protected Long compute() {
        if (end - start <= 10) {
            // 任务足够小，直接计算
            long sum = 0;
            for (long i = start; i <= end; i++) sum += i;
            return sum;
        }

        // 任务太大，拆成两半
        long mid = (start + end) / 2;
        SumTask left = new SumTask(start, mid);
        SumTask right = new SumTask(mid + 1, end);

        left.fork();   // 异步执行左半部分
        long rightResult = right.compute();  // 同步执行右半部分
        long leftResult = left.join();       // 等左半部分完成

        return leftResult + rightResult;
    }
}

// 使用
ForkJoinPool pool = new ForkJoinPool();
Long result = pool.invoke(new SumTask(1, 100));
```

### Fork/Join vs 普通线程池

```
普通线程池：任务不能再拆分，一个线程执行一个任务
Fork/Join：任务可以递归拆分，工作窃取提高效率

工作窃取：
  每个线程有自己的双端队列
  自己的任务做完了，去偷别人队列尾部的任务
  减少线程空闲，提高利用率
```

---

## 总结

| 概念 | 核心要点 |
|---|---|
| 线程池7大参数 | 核心数、最大数、存活时间、队列、拒绝策略 |
| 拒绝策略 | Abort抛异常、CallerRuns调用者执行、Discard丢弃、DiscardOldest丢最老 |
| 四种线程池 | Fixed、Cached、Single、Scheduled，都不推荐用Executors创建 |
| 线程数配置 | CPU密集=核心数+1，IO密集=核心数×2 |
| 并发容器 | ConcurrentHashMap、CopyOnWriteArrayList、BlockingQueue |
| Fork/Join | 分治+工作窃取，适合递归计算 |
