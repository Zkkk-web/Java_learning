# Java 并发编程 | Day 9

> ThreadLocal 和 Java 内存模型

---

## 一、怎么保证线程安全？

### 三种方法

```
① 加锁（synchronized / Lock）
  同一时间只有一个线程能进来操作

② 使用线程安全的类
  ConcurrentHashMap、CopyOnWriteArrayList 等

③ ThreadLocal
  每个线程存自己的一份数据，互不影响
```

---

## 二、ThreadLocal 是什么？

### 一句话

> 每个线程有自己独立的变量副本，线程之间互不影响。

### 生活举例

```
普通变量 = 办公室公用的打印机
  所有人共用，容易冲突

ThreadLocal = 每人桌上放一台打印机
  各用各的，互不干扰
```

### 代码示例

```java
// 创建 ThreadLocal
ThreadLocal<String> threadLocal = new ThreadLocal<>();

// 线程A
new Thread(() -> {
    threadLocal.set("线程A的数据");
    System.out.println(threadLocal.get());  // 线程A的数据
}).start();

// 线程B
new Thread(() -> {
    threadLocal.set("线程B的数据");
    System.out.println(threadLocal.get());  // 线程B的数据
}).start();

// 两个线程互不影响 ✅
```

---

## 三、ThreadLocal 工作中用在哪？

### 最常见的场景

**① 存储用户登录信息**
```java
// 用户登录后，把用户信息存到 ThreadLocal
ThreadLocal<User> currentUser = new ThreadLocal<>();

// 请求进来时存入
currentUser.set(loginUser);

// 后续任何地方直接取，不用传参
User user = currentUser.get();

// 请求结束时清除！
currentUser.remove();
```

**② 数据库连接**
```java
// 每个线程持有自己的数据库连接
// 保证事务在同一个连接里执行
ThreadLocal<Connection> connectionHolder = new ThreadLocal<>();
```

**③ 日期格式化**
```java
// SimpleDateFormat 线程不安全
// 每个线程用自己的实例
ThreadLocal<SimpleDateFormat> dateFormat =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
```

---

## 四、ThreadLocal 底层怎么实现的？

### 核心结构

```
每个 Thread 对象里有一个 ThreadLocalMap

Thread
  └── ThreadLocalMap
        ├── Entry(ThreadLocal1, value1)
        ├── Entry(ThreadLocal2, value2)
        └── ...
```

### set / get 原理

```java
// set 的时候
threadLocal.set("数据");
// 实际上：
// 1. 获取当前线程
// 2. 获取当前线程的 ThreadLocalMap
// 3. 把 (ThreadLocal, 数据) 存进去

// get 的时候
threadLocal.get();
// 实际上：
// 1. 获取当前线程
// 2. 获取当前线程的 ThreadLocalMap
// 3. 以 ThreadLocal 为 key，取出数据
```

### 关键点

```
数据存在线程自己身上（ThreadLocalMap）
不是存在 ThreadLocal 对象里

所以线程之间的数据天然隔离
```

---

## 五、ThreadLocal 内存泄漏问题 ⭐

### 为什么会内存泄漏？

```
ThreadLocalMap 的 key 是弱引用
ThreadLocalMap 的 value 是强引用

弱引用：GC 时会被回收
强引用：GC 时不会被回收

当 ThreadLocal 对象没有外部引用时：
key（弱引用）→ 被 GC 回收，变成 null
value（强引用）→ 不会被回收！

结果：
key = null，但 value 还占着内存
这块内存永远访问不到，也不会被释放
→ 内存泄漏！
```

### 怎么解决？

```java
try {
    threadLocal.set("数据");
    // ... 使用数据
} finally {
    threadLocal.remove();  // ← 用完必须 remove！
}
```

> 记住：**用完必须 remove()，这是铁规则！**

---

## 六、ThreadLocalMap 源码要点

### 存储结构

```
ThreadLocalMap 是一个数组（Entry[]）
Entry 继承了 WeakReference（弱引用）

Entry {
    WeakReference<ThreadLocal> key;  // 弱引用
    Object value;                     // 强引用
}
```

### 哈希冲突怎么解决？

```
ThreadLocalMap 用的是开放地址法（线性探测）
不像 HashMap 用链表

冲突了 → 往后找下一个空位置放

查找时 → 从计算的位置往后找，直到找到或空位置
```

---

## 七、ThreadLocalMap 扩容机制

```
初始容量：16
负载因子：2/3（比 HashMap 的 0.75 更严格）
触发扩容：元素数量超过 容量 × 2/3

扩容大小：原来的 2 倍

扩容时顺便清理 key = null 的脏 Entry
（就是那些内存泄漏的 value）
```

---

## 八、父线程能用 ThreadLocal 传值给子线程吗？

### 普通 ThreadLocal 不行

```java
ThreadLocal<String> tl = new ThreadLocal<>();
tl.set("父线程的数据");

new Thread(() -> {
    System.out.println(tl.get());  // null！子线程拿不到
}).start();
```

### 用 InheritableThreadLocal 可以

```java
InheritableThreadLocal<String> itl = new InheritableThreadLocal<>();
itl.set("父线程的数据");

new Thread(() -> {
    System.out.println(itl.get());  // 父线程的数据 ✅
}).start();
```

### 原理

```
创建子线程时，会把父线程的 ThreadLocalMap 复制一份给子线程
所以子线程能拿到父线程设置的值

注意：是复制，后续父线程修改，子线程感知不到
```

---

## 九、Java 内存模型（JMM）

### 是什么？

```
JMM = Java Memory Model

规定了：
① 变量存在哪里（主内存 / 工作内存）
② 线程怎么读写变量
③ 什么时候把数据同步回主内存
```

### 核心概念

```
主内存：所有线程共享的内存（堆、方法区）
工作内存：每个线程私有的内存（寄存器、缓存）

线程操作变量的步骤：
1. 从主内存读取变量到工作内存
2. 在工作内存里操作
3. 把结果写回主内存
```

### 示意图

```
线程A的工作内存          主内存          线程B的工作内存
  [变量副本A]    ←→    [共享变量]    ←→    [变量副本B]
```

### 三大特性

```
① 原子性：操作不可分割
  int i = 1;     ← 原子操作
  i++;           ← 不是原子操作！（读→改→写三步）

② 可见性：一个线程修改，其他线程立刻看到
  用 volatile 保证可见性

③ 有序性：代码按顺序执行
  实际上编译器可能重排序，用 volatile 禁止重排序
```

### volatile 关键字

```java
// 保证可见性 + 禁止重排序
volatile boolean flag = false;

// 线程A
flag = true;

// 线程B
while (!flag) {  // 能立刻看到线程A的修改
    // ...
}
```

> volatile 不保证原子性！i++ 用 volatile 还是不安全

---

## 十、总结

| 概念 | 一句话 |
|---|---|
| ThreadLocal | 每个线程存自己的数据，互不影响 |
| 内存泄漏 | key是弱引用会被回收，value是强引用不会，用完必须remove() |
| InheritableThreadLocal | 子线程可以继承父线程的ThreadLocal值 |
| JMM | 规定线程怎么读写共享变量 |
| volatile | 保证可见性和有序性，但不保证原子性 |

### 使用 ThreadLocal 的铁规则

```java
try {
    threadLocal.set(value);
    // 业务逻辑
} finally {
    threadLocal.remove();  // 必须！防止内存泄漏
}
```
