# Java 集合框架详解 | Day 7

\---

## 一、List、Set、Queue、Map 总览

### 继承关系

```
Collection
├── List（有序，可重复）
│   ├── ArrayList
│   ├── LinkedList
│   └── Vector（已淘汰）
├── Set（无序，不可重复）
│   ├── HashSet
│   ├── LinkedHashSet（有序）
│   └── TreeSet（排序）
└── Queue（队列，先进先出）
    ├── LinkedList
    └── PriorityQueue（优先级队列）

Map（键值对，单独体系）
├── HashMap
├── LinkedHashMap
├── TreeMap
└── Hashtable（已淘汰）
```

### 怎么选？

```
需要有序 + 可重复     → List
需要去重              → Set
需要按顺序处理任务    → Queue
需要键值对快速查找    → Map
```

\---

## 二、ArrayList 详解

### 底层结构

```java
// 底层就是一个数组
Object\[] elementData;
```

### 初始化

```java
// 方式1：默认，初始容量是0，第一次add才变成10
ArrayList<String> list = new ArrayList<>();

// 方式2：指定初始容量（推荐，减少扩容次数）
ArrayList<String> list = new ArrayList<>(100);
```

### 扩容机制

```
第一次 add → 容量变成 10
满了之后   → 扩容为原来的 1.5 倍

10 → 15 → 22 → 33 → 49...

扩容步骤：
1. 创建新数组（原来的1.5倍）
2. 把旧数组数据复制到新数组
3. 旧数组丢掉
```

### 常用方法

```java
list.add("A");          // 尾部添加
list.add(0, "A");       // 指定位置添加（后面要移位，慢）
list.get(0);            // 获取第0个元素
list.set(0, "B");       // 修改第0个元素
list.remove(0);         // 删除第0个元素
list.size();            // 元素个数
list.contains("A");     // 是否包含
list.indexOf("A");      // 第一次出现的位置
list.clear();           // 清空
```

### 时间复杂度

```
查找（get）      → O(1)  直接下标，很快
尾部插入（add）  → O(1)  直接加在后面
中间插入/删除   → O(n)  后面的元素都要移位，很慢
```

### 线程不安全

```java
// 多线程用这个代替
CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();
```

\---

## 三、LinkedList 详解

### 底层结构

```
双向链表，每个节点有三个部分：

\[prev | data | next]
  ↑             ↑
上一个节点    下一个节点

第一个节点的 prev = null
最后一个节点的 next = null
```

### 同时实现了 List 和 Deque

```java
// 当 List 用
LinkedList<String> list = new LinkedList<>();
list.add("A");
list.get(0);

// 当队列（Queue）用
list.offer("A");   // 入队（加到尾部）
list.poll();       // 出队（从头部取出）
list.peek();       // 看队头，不取出

// 当双端队列（Deque）用
list.addFirst("A");  // 加到头部
list.addLast("A");   // 加到尾部
list.removeFirst();  // 删除头部
list.removeLast();   // 删除尾部
```

### 时间复杂度

```
查找（get）         → O(n)  要从头一个个找，慢
头部/尾部插入删除   → O(1)  直接改指针，很快
中间插入/删除      → O(n)  先要找到位置，再改指针
```

### ArrayList vs LinkedList 对比

||ArrayList|LinkedList|
|-|-|-|
|底层|数组|双向链表|
|查找|O(1) 快|O(n) 慢|
|头部增删|O(n) 慢|O(1) 快|
|尾部增删|O(1) 快|O(1) 快|
|内存|连续，节省|不连续，多存指针|
|适合|查多改少|头部频繁增删|

> 日常开发 99% 用 ArrayList，LinkedList 用于频繁头部增删

\---

## 四、栈（Stack）详解

### 什么是栈？

```
后进先出（LIFO）
就像叠盘子：
  最后放上去的盘子，最先被拿走

push → 压栈（放盘子）
pop  → 弹栈（取盘子）
peek → 看栈顶（看最上面的盘子，不取走）
```

### Java 里的栈

```java
// 方式1：Stack 类（老旧，不推荐）
Stack<Integer> stack = new Stack<>();
stack.push(1);   // 入栈
stack.pop();     // 出栈
stack.peek();    // 看栈顶

// 方式2：用 Deque 代替（推荐）
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1);   // 入栈
stack.pop();     // 出栈
stack.peek();    // 看栈顶
```

### 为什么不推荐 Stack 类？

```
Stack 继承自 Vector
Vector 所有方法都加了锁
单线程也要加锁解锁，浪费性能

推荐用 ArrayDeque 代替
没有锁，性能更好
```

### 栈的常见应用

```
① 浏览器的前进后退
   后退 = 弹出当前页，回到上一页

② 括号匹配
   遇到左括号 → 压栈
   遇到右括号 → 弹出匹配

③ 方法调用栈
   每调用一个方法 → 压栈
   方法执行完    → 弹栈
   递归太深      → 栈溢出（StackOverflowError）
```

### 括号匹配代码示例

```java
boolean isValid(String s) {
    Deque<Character> stack = new ArrayDeque<>();
    for (char c : s.toCharArray()) {
        if (c == '(') stack.push(c);
        else if (c == ')') {
            if (stack.isEmpty()) return false;
            stack.pop();
        }
    }
    return stack.isEmpty();
}
```

\---

## 五、总结对比

|集合|底层|有序|重复|线程安全|
|-|-|-|-|-|
|ArrayList|数组|✅|✅|❌|
|LinkedList|双向链表|✅|✅|❌|
|Stack|数组|✅|✅|✅（但慢）|
|ArrayDeque|数组|✅|✅|❌|
|HashSet|HashMap|❌|❌|❌|
|HashMap|数组+链表+红黑树|❌|key唯一|❌|

\---

## 六、高频面试题

**Q：ArrayList 和数组的区别？**

```
数组：长度固定，不能自动扩容
ArrayList：长度可变，自动扩容，功能更多
```

**Q：ArrayList 默认容量是多少？**

```
创建时是 0
第一次 add 才变成 10
之后每次扩容 1.5 倍
```

**Q：为什么用 ArrayDeque 代替 Stack？**

```
Stack 继承 Vector，所有方法加锁，性能差
ArrayDeque 没有锁，性能更好
功能一样，但更快
```

**Q：LinkedList 为什么查找慢？**

```
链表不连续，没有下标
要从头节点一个个往后找
最坏情况要找 n 次，O(n)
```

