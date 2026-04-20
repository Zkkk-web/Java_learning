# ThreadLocalMap 源码、JMM 与锁深入 | Day 16

> ThreadLocalMap 底层、Java内存模型、volatile、synchronized 深入

---

## 一、ThreadLocalMap 源码分析

### 基本结构

```java
static class ThreadLocalMap {
    // Entry 继承弱引用
    static class Entry extends WeakReference<ThreadLocal<?>> {
        Object value;  // 强引用

        Entry(ThreadLocal<?> k, Object v) {
            super(k);   // key 是弱引用
            value = v;  // value 是强引用
        }
    }

    private Entry[] table;        // 存储数据的数组
    private int size = 0;         // 元素数量
    private int threshold;        // 扩容阈值
}
```

### 为什么 key 用弱引用？

```
如果 key 用强引用：
  ThreadLocal 对象没有外部引用了
  但 ThreadLocalMap 里的 key 还强引用着它
  → ThreadLocal 无法被 GC 回收 → 内存泄漏

如果 key 用弱引用：
  ThreadLocal 没有外部引用时
  弱引用的 key 会被 GC 回收，变成 null
  → ThreadLocal 对象可以被回收

但 value 仍然是强引用！
key = null 但 value 还在 → 还是可能内存泄漏
→ 所以用完必须 remove()
```

---

## 二、ThreadLocalMap 怎么解决哈希冲突？

### 开放地址法（线性探测）

```
不像 HashMap 用链表
ThreadLocalMap 用的是线性探测

冲突了：往后找下一个空位置

比如 key 算出来应该在第5格，但第5格有人了：
  → 找第6格
  → 第6格也有人 → 找第7格
  → 找到空格放进去

查找时：
  从计算的位置往后找
  找到 key 相同的 → 命中
  找到空格 → 不存在
```

### set 流程

```
① 计算 key 的哈希值，找到位置 i
② table[i] 为空 → 直接放
③ table[i] 的 key == 当前 key → 更新 value
④ table[i] 的 key == null（被 GC 回收了）→ 替换这个位置（顺便清理）
⑤ 以上都不是 → i+1，继续找
```

---

## 三、ThreadLocalMap 扩容机制

```
初始容量：16
扩容阈值：容量 × 2/3（比 HashMap 的 0.75 更严格）

扩容步骤：
① 先清理 key=null 的脏数据（expungeStaleEntries）
② 清理后元素数量还超过阈值的 3/4 → 扩容
③ 新容量 = 旧容量 × 2
④ 重新哈希，把所有元素放到新数组

特点：
  扩容前先清理垃圾，减少不必要的扩容
```

---

## 四、父线程能用 ThreadLocal 传值给子线程吗？

```
普通 ThreadLocal：不能
  子线程有自己的 ThreadLocalMap，父线程的数据不会自动复制

InheritableThreadLocal：可以
  创建子线程时，会把父线程的 inheritableThreadLocals 复制一份

注意：
  是复制，不是共享
  子线程拿到的是父线程数据的副本
  后续父线程修改，子线程感知不到
```

```java
// 使用 InheritableThreadLocal
InheritableThreadLocal<String> itl = new InheritableThreadLocal<>();
itl.set("父线程数据");

new Thread(() -> {
    System.out.println(itl.get());  // 父线程数据 ✅
}).start();
```

### 线程池里的问题

```
线程池的线程是复用的
父线程设置了 InheritableThreadLocal
线程池里的线程第一次创建时复制了父线程的值
但之后复用时，不会再次复制

解决：使用阿里的 TransmittableThreadLocal（TTL）
  专门解决线程池场景下的 ThreadLocal 传递问题
```

---

## 五、Java 内存模型（JMM）深入

### 核心概念

```
主内存：所有线程共享（堆、方法区）
工作内存：每个线程私有（寄存器、缓存）

线程只能操作工作内存里的变量副本
不能直接操作主内存

读：主内存 → 工作内存（load）
写：工作内存 → 主内存（store）
```

### 八种内存操作

```
lock     → 把主内存变量标记为线程独占
unlock   → 解除独占
read     → 从主内存读到工作内存
load     → 把 read 的值放入工作内存变量副本
use      → 工作内存变量传给执行引擎
assign   → 执行引擎结果赋值给工作内存变量
store    → 工作内存变量传到主内存
write    → 把 store 的值放入主内存变量
```

### 三大特性

```
① 原子性
  基本读写是原子的（int、boolean 等）
  long、double 在32位JVM里不是原子的（64位操作分两次）
  i++ 不是原子的（读-改-写三步）

② 可见性
  一个线程修改了共享变量
  其他线程不一定立刻看到（缓存导致）
  volatile、synchronized、final 能保证可见性

③ 有序性
  编译器和CPU会重排指令
  volatile 和 happens-before 规则保证有序性
```

---

## 六、i++ 是原子操作吗？

```
不是！分三步：
① 从工作内存读取 i 的值
② 执行 i + 1
③ 把结果写回工作内存
④ 刷新到主内存

多线程下两个线程同时执行 i++：
  线程A读取 i=0
  线程B读取 i=0
  线程A写回 i=1
  线程B写回 i=1（覆盖了A！）
  结果：i=1，期望是2

解决方案：
  AtomicInteger.incrementAndGet()  → CAS
  synchronized(this) { i++; }      → 加锁
```

---

## 七、指令重排详解

### 三种重排

```
① 编译器重排
  javac 或 JIT 编译时，调整代码顺序

② 指令级并行重排
  CPU 把多条指令并行执行

③ 内存系统重排
  CPU 有读写缓冲区，看起来像重排
```

### 重排的限制

```
as-if-serial：
  单线程内，不管怎么排，结果不能变
  有数据依赖的指令不能重排

  int a = 1;  ①
  int b = 2;  ②
  int c = a + b; ③

  ① ② 可以重排（互不依赖）
  ③ 不能排到 ① ② 前面（依赖它们）
```

### 多线程下的重排问题

```java
// 线程A
a = 1;        // ①
flag = true;  // ②  可能被重排到①前面

// 线程B
if (flag) {
    int b = a;  // 可能读到 a=0！
}

// 解决：volatile 禁止重排
volatile boolean flag = false;
volatile int a = 0;
```

---

## 八、happens-before 规则

```
如果 A happens-before B，则：
  A 的操作对 B 可见
  A 的操作在 B 之前执行

六大规则：

① 程序顺序规则
  同一线程，前面的操作 happens-before 后面的

② volatile 规则
  写 volatile happens-before 读这个 volatile

③ 监视器锁规则
  解锁 happens-before 之后的加锁

④ 线程启动规则
  Thread.start() happens-before 线程内的任何操作

⑤ 线程终止规则
  线程内所有操作 happens-before Thread.join() 返回

⑥ 传递性
  A hb B，B hb C → A hb C
```

---

## 九、as-if-serial 规则

```
无论怎么重排
单线程的执行结果不能改变

编译器、处理器都要遵守这个规则
这是重排的底线

区别：
  as-if-serial：保证单线程结果正确
  happens-before：保证多线程可见性和有序性
```

---

## 十、volatile 深入 ⭐

### 两个作用

```
① 保证可见性
  写 volatile → 立刻刷到主内存
  读 volatile → 从主内存重新读

② 禁止指令重排
  通过内存屏障实现
```

### 内存屏障

```
volatile 写：
  写之前加 StoreStore 屏障（前面所有写先完成）
  写之后加 StoreLoad  屏障（写完立刻刷主内存）

volatile 读：
  读之后加 LoadLoad  屏障（后续读先完成）
  读之后加 LoadStore 屏障（后续写等读完成）

效果：
  volatile 写 happens-before 后续的 volatile 读
  保证了顺序
```

### 不能保证原子性

```java
volatile int i = 0;

// 多线程执行 i++ 还是不安全
// 因为 i++ 是三步操作
// volatile 只保证每次读写都是最新值
// 但三步之间还是可能被打断
```

### 经典使用：双重检查锁单例

```java
public class Singleton {
    // 必须加 volatile！
    private volatile static Singleton instance;

    public static Singleton getInstance() {
        if (instance == null) {           // 第一次检查
            synchronized (Singleton.class) {
                if (instance == null) {   // 第二次检查
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}

// 为什么需要 volatile？
// new Singleton() 分三步：
//   ① 分配内存
//   ② 初始化对象
//   ③ 将 instance 指向内存
// 重排后可能是 ①③②
// 另一个线程看到 instance != null，但对象还没初始化完
// 会拿到半初始化的对象！
// volatile 禁止重排，保证 ③ 一定在 ② 之后
```

---

## 十一、synchronized 深入

### 三种使用方式

```java
// ① 实例方法（锁当前对象）
public synchronized void method() { }

// ② 静态方法（锁 Class 对象）
public static synchronized void method() { }

// ③ 代码块（锁指定对象）
synchronized (this) { }
synchronized (Singleton.class) { }
```

### 底层实现

```
字节码层面：
  进入 synchronized → monitorenter 指令
  退出 synchronized → monitorexit  指令

每个对象都有一个 monitor（监视器）
monitor 里有：
  _owner：当前持有锁的线程
  _count：重入次数
  _EntryList：等待获取锁的线程队列
  _WaitSet：wait() 后等待的线程集合
```

### 可重入性

```java
synchronized (this) {
    // 进来，count = 1
    synchronized (this) {
        // 再进来，count = 2（可重入）
    }
    // 出来，count = 1
}
// 出来，count = 0，锁释放
```

---

## 十二、synchronized 怎么保证可见性？

```
加锁时：清空工作内存，从主内存重新读
解锁时：把工作内存的修改刷回主内存

所以：
  A 解锁 → 数据刷回主内存
  B 加锁 → 从主内存重新读
  B 一定能看到 A 的修改

happens-before 监视器锁规则：
  解锁 happens-before 加锁
  所以 B 一定能看到 A 解锁前的所有操作
```

---

## 十三、synchronized 锁升级 ⭐

```
无锁 → 偏向锁 → 轻量级锁 → 重量级锁

偏向锁：
  第一个线程来，把线程ID写入 Mark Word
  下次同一线程来，检查线程ID，直接进
  几乎零开销

轻量级锁：
  第二个线程来竞争，撤销偏向锁
  CAS 自旋竞争，不挂起线程
  适合竞争不激烈、锁持有时间短的情况

重量级锁：
  自旋多次还没拿到锁
  线程挂起，进入 EntryList 等待
  需要 OS 介入，开销大

升级过程不可逆（偏向锁可以撤销）
```

### 自适应自旋

```
轻量级锁自旋的次数：
  不是固定的，JVM 自适应调整

  上次自旋成功 → 这次多自旋几次
  上次自旋失败 → 这次少自旋或直接升级重量级锁
```

---

## 十四、synchronized 和 ReentrantLock 的区别

| 特性 | synchronized | ReentrantLock |
|---|---|---|
| 实现层面 | JVM 内置（C++） | Java 代码（AQS） |
| 锁释放 | 自动 | 手动 unlock() |
| 可中断 | ❌ | ✅ lockInterruptibly() |
| 公平锁 | ❌ 只有非公平 | ✅ 可选 |
| 多条件 | 一个（wait/notify） | 多个 Condition |
| 超时获取 | ❌ | ✅ tryLock(timeout) |
| 锁状态查询 | ❌ | ✅ isLocked() |
| 性能 | JDK6后差不多 | JDK6后差不多 |

### 怎么选？

```
简单同步场景     → synchronized（简单，不容易出错）
需要高级功能     → ReentrantLock
  - 可中断等待
  - 公平锁
  - 多个等待条件
  - 超时获取锁
```

---

## 总结

| 知识点 | 核心要点 |
|---|---|
| ThreadLocalMap | 数组+线性探测，key弱引用，扩容2/3阈值 |
| 内存泄漏 | key被GC后value仍存在，用完必须remove() |
| 父子线程传值 | InheritableThreadLocal，线程池用TTL |
| JMM三大特性 | 原子性、可见性、有序性 |
| i++ | 不是原子操作，用AtomicInteger |
| 指令重排 | 编译器+CPU优化，volatile禁止重排 |
| happens-before | 六大规则，保证多线程可见性 |
| volatile | 可见性+禁止重排，不保证原子性 |
| synchronized | 底层monitor，可重入，保证三大特性 |
| 锁升级 | 偏向→轻量→重量，单向不可逆 |
| ReentrantLock | 比synchronized更灵活，手动释放 |
