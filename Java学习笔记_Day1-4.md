# Java 学习笔记 | Day 1-4

> 从零开始学 Java，记录学习内容和踩过的坑。

---

## Day 1-2

### 基础概念

#### 指针 vs 普通变量（C++ 对比）
- 普通变量存值，指针存地址（门牌号）
- C++ 需要手动 `delete` 释放内存，Java 有 GC 自动回收
- 现代 C++ 推荐用智能指针（`unique_ptr`、`shared_ptr`）

```cpp
int x = 42;
int* ptr = &x;   // ptr 存储 x 的地址
*ptr = 100;      // 通过指针修改 x 的值
```

---

#### 方法（Method）
- 把一段重复的步骤打包起来，给它起个名字
- 定义 = 第一次写出来；调用 = 之后使用它
- 方法是最基础的封装体现

```java
// 定义方法
int add(int a, int b) {
    return a + b;  // return = 把结果交出去
}

// 调用方法
int result = add(3, 5);  // result = 8
```

---

#### 类（Class）
- 模具/模板，不能直接用，要 `new` 出对象才能用
- 属性 = 这个东西有什么；方法 = 这个东西能做什么

```java
class Dog {
    String name;   // 属性
    int age;

    void bark() {  // 方法
        System.out.println(name + "：汪汪！");
    }
}

// 使用
Dog dog = new Dog();
dog.name = "小黑";
dog.bark();  // 输出：小黑：汪汪！
```

---

#### 面向对象三大特性

**封装**：把细节藏起来，只暴露必要接口

```java
class ATM {
    private int money = 10000;  // private = 外面看不到

    public void withdraw(int amount) {  // public = 外面能用
        money -= amount;
    }
}
```

**继承**：子类继承父类的属性和方法（`extends`）

```java
class Animal {
    void breathe() {
        System.out.println("呼吸");
    }
}

class Dog extends Animal {
    void bark() {
        System.out.println("汪汪");
    }
}
// Dog 既能 bark() 也能 breathe()
```

**多态**：同一个命令，不同对象自动执行不同的方法

```java
Animal[] animals = { new Dog(), new Cat() };
for (Animal a : animals) {
    a.makeSound();  // 狗汪汪，猫喵喵，自动区分
}
```

---

### 数据类型

**常用类型只需记这几个：**

| 类型 | 用途 |
|---|---|
| `int` | 整数，99% 情况用这个 |
| `double` | 小数，99% 情况用这个 |
| `boolean` | true 或 false |
| `String` | 字符串 |
| `char` | 单个字符 |

> 其他类型（byte、short、long、float）用到再查

**类型转换陷阱：**

```java
short s1 = 1;
s1 = s1 + 1;   // ❌ 报错！结果是 int，不能赋值给 short
s1 += 1;       // ✅ 正确！+=  自动做了类型转换
```

---

### 二进制基础

**整数转二进制**（除2取余）：
```
25 ÷ 2 = 12 余 1
12 ÷ 2 = 6  余 0
6  ÷ 2 = 3  余 0
3  ÷ 2 = 1  余 1
1  ÷ 2 = 0  余 1
从下往上读：11001
```

**小数转二进制**（乘2取整）：
```
0.125 × 2 = 0.25  → 记 0
0.25  × 2 = 0.5   → 记 0
0.5   × 2 = 1.0   → 记 1，结束
结果：0.001
```

> 0.1 的二进制是无限循环的，这就是为什么 float/double 不精确！

**浮点数存储（32位）：**
```
符号位S(1位) + 指数E(10位) + 尾数M(21位)

25.125 = 11001.001 = 1.1001001 × 2⁴
S=0（正数）  E=4   M=1001001
```

---

### 位运算

```java
2 << 3   // 左移3位 = 2 × 2³ = 16，最快的乘法
```

| 符号 | 含义 |
|---|---|
| `<` | 小于（比较大小） |
| `<<` | 左移（位运算，乘以2的n次方） |
| `>` | 大于 |
| `>>` | 右移（除以2的n次方） |

---

### 运算符

```java
// 短路运算（推荐）
&&   // 左边false，直接结束
||   // 左边true，直接结束

// 非短路（不推荐）
&    // 两边都算
|    // 两边都算
```

**正确写法示例：**
```java
// 必须先判断 null，再判断内容
username != null && !username.equals("")
// 顺序不能反！否则 NullPointerException
```

---

### i++ 陷阱

```java
int i = 1;
i = i++;  // 结果是 1，不是 2！

// JVM 内部：
int temp = i;   // temp = 1
i++;            // i = 2
i = temp;       // i 被打回 1

// 正确写法
i++;            // 直接写，结果是 2 ✅
```

---

## Day 3

### 抽象类

- 不能直接 `new`，必须让子类继承并补全
- 可以有抽象方法（没有内容），强制子类实现

```java
abstract class Animal {
    abstract void makeSound();  // 抽象方法，没有内容

    void breathe() {            // 普通方法，子类直接用
        System.out.println("呼吸");
    }
}

class Dog extends Animal {
    @Override
    void makeSound() {
        System.out.println("汪汪");  // 必须实现
    }
}
```

---

### 接口（Interface）

- 纯规定，所有方法都没有内容
- 用 `implements` 实现，可以实现多个

```java
interface Flyable {
    void fly();
}

interface Swimmable {
    void swim();
}

class Duck implements Flyable, Swimmable {
    void fly()  { System.out.println("鸭子飞"); }
    void swim() { System.out.println("鸭子游"); }
}
```

**抽象类 vs 接口：**

| | 抽象类 | 接口 |
|---|---|---|
| 方法 | 可以有普通方法 | 全部没有内容 |
| 继承/实现 | 只能继承一个 | 可以实现多个 |
| 关键字 | `extends` | `implements` |

---

### 重写 vs 重载

**重写（Override）**：子类替换父类的方法

```java
class Animal {
    void makeSound() { System.out.println("动物叫"); }
}

class Dog extends Animal {
    @Override
    void makeSound() { System.out.println("汪汪"); }  // 替换了
}
```

**重载（Overload）**：同一个类里，同名方法参数不同

```java
class Calculator {
    int add(int a, int b) { return a + b; }
    double add(double a, double b) { return a + b; }  // 参数不同
}
```

---

### SOLID 五大原则

| 原则 | 全称 | 一句话 |
|---|---|---|
| SRP | 单一职责 | 一个类只做一件事 |
| OCP | 开闭原则 | 加功能新增代码，不改旧代码 |
| LSP | 里氏替换 | 子类能完全替换父类 |
| ISP | 接口隔离 | 接口要小而精 |
| DIP | 依赖倒置 | 依赖抽象，不依赖具体实现 |

---

### 重要关键字

**this**：代表当前对象自己

```java
class Dog {
    String name;

    void setName(String name) {
        this.name = name;  // this.name = 我自己的属性
                           // name = 传进来的参数
    }
}
```

**static**：属于整个类，不属于某个对象

```java
class Student {
    String name;              // 每个学生自己的
    static String school = "清华";  // 所有学生共享

    static void study() { }   // 不需要 new，直接 Student.study()
}
```

**final**：不能改变

```java
final int x = 10;
x = 20;  // ❌ 报错

final StringBuilder sb = new StringBuilder("abc");
sb.append("d");              // ✅ 内容可以改
sb = new StringBuilder();   // ❌ 不能换对象
```

---

### 深拷贝 vs 浅拷贝

```
浅拷贝：复制了外壳，内部对象还是共享的
        改了 person1 的地址 → person2 也变了！

深拷贝：完全独立，互不影响
        改了 person1 的地址 → person2 不受影响
```

---

### == vs equals()

```java
String a = new String("abc");
String b = new String("abc");

a == b          // false，不是同一个对象（地址不同）
a.equals(b)     // true，内容相同

// 记住：比较字符串永远用 equals()
```

---

### 值传递 vs 引用传递

Java 只有值传递，传对象时传的是**地址的复印件**：

```java
void change(int x) {
    x = 100;  // 改的是复印件，外面的 a 不变
}

void changeName(Person p) {
    p.name = "李四";  // 通过地址改内容，外面的 person 变了
}
```

---

## Day 4

### 集合框架

#### List、Set、Queue 对比

| | List | Set | Queue |
|---|---|---|---|
| 有没有顺序 | ✅ 按放入顺序 | ❌ 无序 | ✅ 先进先出 |
| 允许重复 | ✅ | ❌ | ✅ |
| 能随意取 | ✅ `get(0)` | ❌ | ❌ 只能取队头 |
| 适合场景 | 有序数据 | 去重 | 按顺序处理任务 |

```java
// List
List<String> list = new ArrayList<>();
list.add("A");
list.get(0);  // 直接取

// Set
Set<String> set = new HashSet<>();
set.add("A");
set.add("A");  // 第二个被忽略

// Queue
Queue<String> queue = new LinkedList<>();
queue.offer("A");  // 入队
queue.poll();      // 出队（先进先出）
```

---

#### ArrayList vs LinkedList

| | ArrayList | LinkedList |
|---|---|---|
| 查找 | ✅ 快，直接跳 | ❌ 慢，一个个找 |
| 增删 | ❌ 慢，要移位 | ✅ 快，改指针 |
| 适合 | 经常查找 | 经常增删 |

> 日常开发用 ArrayList，LinkedList 用于频繁增删

---

#### HashMap

- 通讯录：用 key（名字）快速找 value（内容）

```java
HashMap<String, Integer> map = new HashMap<>();
map.put("apple", 3);      // 存入
map.get("apple");          // 取出 → 3
map.containsKey("apple");  // 是否存在 → true
map.remove("apple");       // 删除
```

**HashMap 底层原理：**
```
初始容量 = 16
负载因子 = 0.75
触发扩容 = 16 × 0.75 = 12个元素时扩容
扩容后 = 原来的 2 倍
```

---

#### 线程安全

**快速失败（Fail-Fast）：**
- `ArrayList` 遍历时被修改 → 直接报错
- 宁可报错，不给错误结果

**CopyOnWriteArrayList（线程安全）：**
```
写操作：
1. 复制一份新数组
2. 在新数组上修改（加锁）
3. 修改完替换原数组

读操作：
直接读旧数组，不加锁，不受影响
```

> 适合读多写少的场景

---

### 时间复杂度

| 复杂度 | 意思 | 例子 |
|---|---|---|
| O(1) | 一步搞定 | `arr[0]` 直接取 |
| O(log n) | 每次缩小一半 | 二分查找 |
| O(n) | 遍历一遍 | 单层循环 |
| O(n²) | 两层循环 | 暴力双循环 |

```
速度：O(1) > O(log n) > O(n) > O(n²)
```

**实际例子：**
```java
// 两数之和 - 暴力 O(n²)
for (int i = 0; i < n; i++)
    for (int j = i+1; j < n; j++)  // 两层循环

// 两数之和 - HashMap O(n)
for (int i = 0; i < n; i++) {
    if (map.containsKey(target - nums[i]))  // 一层循环
        return ...
    map.put(nums[i], i);
}
```

---

### 金融计算精度

```java
// ❌ 不能用 float/double（精度丢失）
System.out.println(0.1 + 0.2);  // 0.30000000000000004

// ✅ 方法1：BigDecimal
BigDecimal a = new BigDecimal("0.1");
BigDecimal b = new BigDecimal("0.2");
a.add(b);  // 精确的 0.3

// ✅ 方法2：转成整数（元→分）
int price = 199;   // 1.99元存成199分
int total = 199 * 3;  // = 597分，精确
```

---

### 其他重要概念

**hashCode 和 equals 的关系：**
```
equals 相等  →  hashCode 必须相等
hashCode 相等 →  equals 不一定相等（可能哈希冲突）
```

**modCount（修改计数器）：**
- 记录集合被修改了几次
- 遍历时发现 modCount 变了 → 立刻报错
- 防止遍历途中数据被修改

**transient：**
```java
class User implements Serializable {
    String name;
    transient String password;  // 序列化时跳过，不存储
}
```

**null vs 空字符串：**
```java
String a = "";    // 有盒子，盒子是空的
String b = null;  // 连盒子都没有

// 正确判断顺序：先判断null，再判断内容
if (b != null && !b.equals("")) { ... }
```

---

## 总结

```
Day 1-2：Java 基础 → 类、方法、封装、继承、多态、数据类型、二进制
Day 3：  进阶概念 → 抽象类、接口、重写重载、SOLID、深浅拷贝
Day 4：  集合框架 → List/Set/Queue、HashMap、线程安全、时间复杂度
```

> 看得懂但写不出来是正常的，合上笔记自己写才是真正的练习。
