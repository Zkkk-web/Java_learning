# Java 集合框架笔记 | Day 5-6

> 重难点总结，包含底层原理

---

## 一、集合框架全景

```
Collection
├── List（有序，可重复）
│   ├── ArrayList
│   └── LinkedList
├── Set（无序，不可重复）
│   └── HashSet
└── Queue（先进先出）
    └── LinkedList（同时实现了 List 和 Queue）

Map（键值对）
└── HashMap
```

---

## 二、List

### ArrayList vs LinkedList

| | ArrayList | LinkedList |
|---|---|---|
| 底层结构 | 动态数组 | 双向链表 |
| 查找 | ✅ O(1)，直接下标 | ❌ O(n)，从头找 |
| 增删（中间） | ❌ O(n)，要移位 | ✅ O(1)，改指针 |
| 内存 | 连续，节省 | 不连续，多存指针 |
| 适合场景 | 查多改少 | 改多查少 |

> 日常开发用 ArrayList，LinkedList 用于频繁增删

### ArrayList 扩容机制

```
初始容量：10
触发扩容：放满了
扩容大小：原来的 1.5 倍

10 → 15 → 22 → 33...
```

---

## 三、HashMap ⭐（重点）

### 基本使用

```java
HashMap<String, Integer> map = new HashMap<>();
map.put("apple", 3);       // 存入
map.get("apple");          // 取出 → 3
map.containsKey("apple");  // 是否存在 → true
map.remove("apple");       // 删除
map.size();                // 大小
```

### 底层结构

```
JDK7：数组 + 链表
JDK8：数组 + 链表 + 红黑树（链表超过8个转红黑树）

为什么转红黑树？
链表查找 O(n)，红黑树查找 O(log n)，更快
```

### put 流程

```
1. 计算 hashCode（高低16位异或混合）
2. 数组为空 → 先初始化（16个格子）
3. 找到对应格子
   ├── 格子为空 → 直接放
   ├── key 相同 → 覆盖 value
   ├── 是红黑树 → 树方式插入
   └── 是链表 → 插到尾部
              链表 ≥ 8 → 转红黑树
4. 超过阈值 → 扩容
```

### 容量和负载因子

```
初始容量：16
负载因子：0.75
触发扩容：16 × 0.75 = 12 个元素时
扩容大小：原来的 2 倍

为什么是 0.75？
太大（接近1）→ 冲突多，查找慢
太小（接近0）→ 浪费内存
0.75 是最佳平衡点（数学证明）
```

### 为什么容量必须是 2 的幂次方？

```
定位格子：hash & (length - 1)
等价于：  hash % length（但位运算更快）

这个等价关系成立的前提：length 必须是 2 的幂次方

例：length = 16，length - 1 = 15 = 0000 1111
hash & 15 → 只保留低4位 → 就是 hash % 16
```

### hash 函数为什么做高低位异或？

```
问题：hash & (length-1) 只用了 hashCode 的低位
      高位信息全部丢失，容易冲突

解决：h ^ (h >>> 16)
      把高16位混入低16位
      这样低位里也包含了高位信息
      两个原来低位相同的 key，混合后就不同了

效果：元素分布更均匀，哈希冲突减少
```

### 传入初始容量不是 2 的幂次方怎么办？

```
传入 17 → 自动调整为 32（最近的2的幂次方）
传入 30 → 自动调整为 32

算法：把最高位的1往右扩散，最后+1
10000 → 11000 → 11100 → 11110 → 11111 → 100000 = 32
```

### 哈希冲突解决：拉链法

```
同一个格子有多个元素 → 用链表串起来

第5格 → [apple] → [grape] → null

查找时：
1. 先比较 hashCode（快速定位格子）
2. 再用 equals 确认是哪个元素
```

### JDK7 vs JDK8 对比

| | JDK7 | JDK8 |
|---|---|---|
| 数据结构 | 数组+链表 | 数组+链表+红黑树 |
| 插入方式 | 头插法 | 尾插法 |
| hash函数 | 多次移位异或（复杂） | 一次高低位异或（简单） |
| 多线程问题 | 死循环 | 不会死循环（但仍不安全） |

---

## 四、HashSet

### 底层原理

```java
// HashSet 内部就是一个 HashMap
private HashMap<E, Object> map;
private static final Object PRESENT = new Object(); // 假值占位

// add 操作就是 put
set.add("A")
// 底层：map.put("A", PRESENT)
```

### 为什么能去重？

```
HashMap 的 key 天生唯一
第二次 add 相同元素 → 覆盖了同一个 key
map 里还是只有一个 → 自动去重
```

---

## 五、线程安全问题

### HashMap 不安全的三个问题

```
① 死循环（JDK7）
  多线程扩容时头插法导致环形链表
  JDK8 用尾插法修复

② 元素丢失
  两个线程同时 put，后者覆盖前者
  数据丢了

③ get 到 null
  线程1扩容，线程2同时 get
  元素还没搬到新位置，get 不到
```

### ConcurrentHashMap 怎么解决？

```
JDK7：分段锁
  把数组分成16段，每段单独加锁
  A操作第1段，B操作第2段，互不影响

JDK8：CAS + synchronized（更细）
  格子为空 → CAS（不加锁，更快）
  格子有元素 → synchronized 锁单个格子

规则：
  读 vs 读 → 不冲突，无锁
  读 vs 写 → 不冲突，读旧的
  写 vs 写 → 冲突！同一时间只能一个写
```

### CopyOnWriteArrayList

```
写操作时：
1. 复制一份新数组
2. 在新数组上修改（加锁，防止多个写冲突）
3. 修改完替换原数组

读操作时：
直接读旧数组，不加锁，不受影响

适合：读多写少的场景
```

### 快速失败（Fail-Fast）

```
ArrayList 遍历时被修改 → 立刻抛出 ConcurrentModificationException

原理：
开始遍历记录 modCount
每次修改 modCount + 1
遍历中发现 modCount 变了 → 报错

宁可报错，不给错误结果
```

---

## 六、时间复杂度

| 操作 | ArrayList | LinkedList | HashMap |
|---|---|---|---|
| 查找 | O(1) | O(n) | O(1) |
| 插入头部 | O(n) | O(1) | - |
| 插入尾部 | O(1) | O(1) | - |
| 删除 | O(n) | O(1) | O(1) |

---

## 七、选集合的原则

```
需要有序 + 可重复           → List
  查多改少                  → ArrayList
  改多查少                  → LinkedList

需要去重                    → Set → HashSet

需要键值对，快速查找         → Map → HashMap

多线程环境
  读多写少                  → CopyOnWriteArrayList
  键值对                    → ConcurrentHashMap
```

---

## 八、高频面试题

**Q：HashMap 和 Hashtable 的区别？**
```
HashMap：线程不安全，key 允许 null，JDK1.2
Hashtable：线程安全（全部加锁），key 不允许 null，老旧不推荐
```

**Q：HashMap 的 key 可以是任何对象吗？**
```
可以，但必须重写 hashCode() 和 equals()
否则相同内容的两个对象会被当成不同的 key
```

**Q：为什么重写 equals 必须重写 hashCode？**
```
equals 相等 → hashCode 必须相等（规定）
不然 HashMap 和 HashSet 会出问题：
  同一个 key 放了两次，但 map 认为是不同的 key
```

**Q：链表什么时候转红黑树？**
```
链表长度 ≥ 8 且 数组长度 ≥ 64 → 转红黑树
链表长度 < 6 → 红黑树退化回链表
```

---

## 总结

```
重点掌握：
  HashMap 的 put 流程
  HashMap 的扩容机制
  哈希冲突的解决方式（拉链法）
  线程安全的解决方案

理解原理：
  为什么容量是 2 的幂次方
  为什么负载因子是 0.75
  为什么高低16位做异或
```
