# Java 并发编程：JMM 与锁 | Day 10

> 原子操作、指令重排、volatile、synchronized 详解

---

## 一、i++ 是原子操作吗？

### 答案：不是！

```
i++ 分三步：
1. 读取 i 的值
2. i + 1
3. 把结果写回 i

三步不是一个整体，可以被其他线程打断
```

### 多线程下的问题

```
线程A：读取 i = 0
线程B：读取 i = 0
线程A：i + 1 = 1，写回
线程B：i + 1 = 1，写回

结果：i = 1
期望：i = 2

数据丢失了！
```

### 怎么保证原子性？

```java
// 方法1：synchronized 加锁
synchronized (this) {
    i++;
}

// 方法2：AtomicInteger（推荐）
AtomicInteger i = new AtomicInteger(0);
i.incrementAndGet();  // 原子操作，线程安全
```

---

## 二、什么是指令重排？

### 是什么？

```
编译器和 CPU 为了提高性能
会对代码的执行顺序进行调整

你写的顺序：
int a = 1;
int b = 2;
int c = a + b;

实际执行可能是：
int b = 2;
int a = 1;
int c = a + b;

结果一样，但顺序不同
```

### 为什么要重排？

```
CPU 执行指令时，某些指令需要等待（比如读内存很慢）
重排可以让 CPU 在等待期间执行其他指令
提高整体效率
```

### 单线程没问题，多线程有问题

```java
// 线程A
a = 1;
flag = true;  // 可能被重排到 a=1 前面

// 线程B
if (flag) {
    int b = a;  // 可能读到 a=0，因为 a=1 还没执行
}
```

### 怎么禁止重排？

```java
// 用 volatile 禁止重排
volatile boolean flag = false;
volatile int a = 0;
```

---

## 三、happens-before 了解吗？

### 是什么？

```
happens-before 是 JMM 的规则
保证：如果 A happens-before B
那么 A 的操作结果对 B 可见
```

### 六大规则

```
① 程序顺序规则
  同一个线程里，前面的操作 happens-before 后面的操作

② volatile 规则
  写 volatile 变量 happens-before 读这个变量

③ 锁规则
  解锁 happens-before 后续的加锁

④ 线程启动规则
  start() happens-before 线程里的所有操作

⑤ 线程终止规则
  线程所有操作 happens-before join() 返回

⑥ 传递性规则
  A happens-before B，B happens-before C
  → A happens-before C
```

### 一句话

> happens-before 不是说 A 一定在 B 之前执行，
> 而是说 A 的结果 B 一定能看到。

---

## 四、as-if-serial 了解吗？

### 是什么？

```
as-if-serial = 好像是串行执行的

不管编译器和 CPU 怎么重排
单线程的执行结果不能改变

换句话说：
重排可以，但不能影响单线程的结果
```

### 举例

```java
int a = 1;    // ①
int b = 2;    // ②
int c = a + b; // ③

① 和 ② 没有依赖关系，可以重排
但 ③ 依赖 ① 和 ②，③ 必须在 ① ② 之后

所以不管怎么排，结果都是 c = 3
```

### as-if-serial vs happens-before

```
as-if-serial：保证单线程结果正确
happens-before：保证多线程可见性
```

---

## 五、volatile 详解 ⭐

### 两个作用

```
① 保证可见性
  一个线程修改了值，其他线程立刻能看到

② 禁止指令重排
  写 volatile 前面的代码不能排到后面
  读 volatile 后面的代码不能排到前面
```

### 不能保证原子性

```java
volatile int i = 0;
i++;  // 仍然不是原子操作！
```

### 底层原理：内存屏障

```
volatile 写操作：
  写之前加 StoreStore 屏障（前面写操作都完成才能写）
  写之后加 StoreLoad  屏障（写完立刻刷到主内存）

volatile 读操作：
  读之后加 LoadLoad  屏障（读完才能读后面的）
  读之后加 LoadStore 屏障（读完才能写后面的）
```

### 经典使用场景

**① 状态标志位**
```java
volatile boolean stop = false;

// 线程A：停止线程B
stop = true;

// 线程B：检查标志位
while (!stop) {
    doSomething();
}
```

**② 单例模式（双重检查锁）**
```java
public class Singleton {
    private volatile static Singleton instance;  // 必须 volatile！

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();  // 如果没有 volatile，可能看到半初始化的对象
                }
            }
        }
        return instance;
    }
}
```

> 为什么需要 volatile？
> `new Singleton()` 分三步：分配内存、初始化、赋值给 instance
> 重排后可能先赋值再初始化，其他线程拿到的是半初始化的对象

---

## 六、synchronized 用过吗？

### 三种用法

```java
// ① 修饰方法（锁当前对象）
public synchronized void method() { }

// ② 修饰静态方法（锁 Class 对象）
public static synchronized void method() { }

// ③ 修饰代码块（锁指定对象）
synchronized (this) { }
synchronized (Xxx.class) { }
```

### 特点

```
① 可重入：同一个线程可以重复获取同一把锁
② 自动释放：出了 synchronized 块自动释放锁
③ 保证原子性、可见性、有序性
```

---

## 七、synchronized 实现原理

### 底层靠 monitor（监视器锁）

```
每个对象都有一个 monitor

进入 synchronized → monitorenter（计数器+1）
退出 synchronized → monitorexit （计数器-1）
计数器为0 → 锁被释放
```

### 对象头

```
Java 对象在内存里分三部分：
┌─────────────┐
│  对象头      │  ← 存锁信息、hashCode、GC年龄
├─────────────┤
│  实例数据    │  ← 存字段值
├─────────────┤
│  对齐填充    │  ← 凑整用
└─────────────┘

对象头里的 Mark Word 存锁状态：
无锁 → 偏向锁 → 轻量级锁 → 重量级锁
```

---

## 八、synchronized 怎么保证可见性？

```
线程加锁时：
  把工作内存清空，重新从主内存读

线程解锁时：
  把工作内存的数据刷回主内存

所以：
  A线程解锁 → 数据刷回主内存
  B线程加锁 → 从主内存重新读
  B 一定能看到 A 的修改 ✅
```

---

## 九、synchronized 锁升级 ⭐

### 为什么需要锁升级？

```
重量级锁（操作系统级别）太重了
大部分情况下，同一时间只有一个线程访问
没必要每次都用重量级锁

所以 JDK6 引入了锁升级机制
从轻到重，按需升级
```

### 四个状态

```
无锁
  ↓ 第一个线程来了
偏向锁（只记录线程ID，不加锁）
  ↓ 第二个线程来竞争
轻量级锁（CAS 自旋，不挂起线程）
  ↓ 竞争激烈，自旋多次还没拿到锁
重量级锁（操作系统级别，线程挂起等待）
```

### 各阶段详解

```
偏向锁：
  只有一个线程用，记住线程ID
  下次同一个线程来，不需要任何操作直接进
  成本：几乎为0

轻量级锁：
  有竞争，但竞争不激烈
  用 CAS 自旋等待（不停尝试）
  不挂起线程，响应快
  成本：消耗 CPU

重量级锁：
  竞争激烈
  线程挂起，等待唤醒
  成本：线程上下文切换，很重
```

> 锁只能升级，不能降级（偏向锁可以撤销）

---

## 十、synchronized 和 ReentrantLock 的区别

| | synchronized | ReentrantLock |
|---|---|---|
| 实现方式 | JVM 内置 | Java 代码实现 |
| 锁的释放 | 自动释放 | 必须手动 unlock() |
| 可中断 | ❌ 不能 | ✅ 可以中断等待 |
| 公平锁 | ❌ 非公平 | ✅ 可以选择公平/非公平 |
| 条件变量 | wait/notify | Condition（更灵活） |
| 性能 | JDK6后差不多 | JDK6后差不多 |

### 怎么选？

```
简单场景     → synchronized（简单，不容易出错）
需要高级功能 → ReentrantLock
  比如：需要可中断、需要公平锁、需要多个条件变量
```

### ReentrantLock 使用模板

```java
ReentrantLock lock = new ReentrantLock();

lock.lock();  // 加锁
try {
    // 业务逻辑
} finally {
    lock.unlock();  // 必须手动释放！放在 finally 里
}
```

---

## 总结

| 概念 | 核心要点 |
|---|---|
| i++ | 不是原子操作，用 AtomicInteger 代替 |
| 指令重排 | 编译器优化，单线程没问题，多线程要注意 |
| happens-before | A 的结果对 B 可见的规则 |
| as-if-serial | 重排不影响单线程结果 |
| volatile | 可见性+禁止重排，但不保证原子性 |
| synchronized | 原子性+可见性+有序性，自动释放锁 |
| 锁升级 | 无锁→偏向锁→轻量级锁→重量级锁 |
| ReentrantLock | 比 synchronized 更灵活，但要手动释放 |
