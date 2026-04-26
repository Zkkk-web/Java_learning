# Day 21 - JVM 与编译内幕

> 系统梳理 JVM 从入门到实战的核心知识体系。本篇覆盖类加载机制、字节码原理、运行时内存模型、垃圾回收、JIT 编译以及线上性能监控与排查实战,内容偏面试与工程实践,可作为长期参考手册。

---

## 📚 目录

- [一、JVM 基础认知](#一jvm-基础认知)
- [二、类加载机制](#二类加载机制)
- [三、字节码与类文件](#三字节码与类文件)
- [四、运行时内存结构](#四运行时内存结构)
- [五、对象的创建与分配](#五对象的创建与分配)
- [六、垃圾回收机制](#六垃圾回收机制)
- [七、垃圾收集器全家桶](#七垃圾收集器全家桶)
- [八、JIT 即时编译](#八jit-即时编译)
- [九、性能监控三板斧](#九性能监控三板斧)
- [十、实战:内存泄露排查](#十实战内存泄露排查)
- [十一、实战:CPU 100% 排查](#十一实战cpu-100-排查)
- [十二、核心知识点总结](#十二核心知识点总结)

---

## 一、JVM 基础认知

### 1.1 大白话带你认识 JVM

**JVM 到底是什么?**
JVM(Java Virtual Machine)本质上是运行在操作系统之上的一个 **进程**,它做三件事:

1. **翻译官**:把 `.class` 字节码翻译成 CPU 能执行的机器指令
2. **内存管家**:自动管理对象的内存分配与回收(GC)
3. **性能优化器**:运行期监控热点代码,通过 JIT 编译优化执行效率

**为什么需要 JVM?**
屏蔽底层操作系统差异 → 实现 **"Write Once, Run Anywhere"**。Java 代码不直接面向 CPU,而是面向 JVM 这个"虚拟 CPU"。

**JVM ≠ JRE ≠ JDK**

```
JDK = JRE + 开发工具(javac, javap, jdb...)
JRE = JVM + 核心类库(rt.jar / java.base...)
JVM = 类加载器 + 执行引擎 + 运行时数据区
```

**主流 JVM 实现**:HotSpot(最常用,Oracle/OpenJDK)、GraalVM(支持多语言、AOT 编译)、OpenJ9(IBM,内存占用低)、Dragonwell(阿里)。

### 1.2 JVM 如何运行 Java 代码?

完整生命周期:

```
.java 源文件
   │ javac 前端编译
   ▼
.class 字节码文件
   │ ClassLoader 加载
   ▼
方法区/元空间(类元数据)+ 堆(Class 对象)
   │
   ▼
执行引擎(解释器逐条解释 / JIT 编译为机器码)
   │
   ▼
CPU 执行机器指令
```

**关键点**:JVM 默认采用 **解释执行 + JIT 混合模式**。冷代码解释执行(启动快),热点代码 JIT 编译(执行快),兼顾启动速度与运行性能。

---

## 二、类加载机制

### 2.1 类加载的生命周期

一个类从被加载到 JVM 内存,到从内存卸载,经历 7 个阶段:

```
加载 → [验证 → 准备 → 解析] 链接 → 初始化 → 使用 → 卸载
```

| 阶段 | 做什么 |
|------|--------|
| **加载** | 通过类全限定名获取二进制字节流,生成 `Class` 对象 |
| **验证** | 文件格式、元数据、字节码、符号引用四重校验,防止恶意字节码 |
| **准备** | 为静态变量分配内存并赋 **零值**(`static int a = 5` 此时 a=0) |
| **解析** | 将常量池的符号引用转为直接引用(可静态可动态) |
| **初始化** | 执行 `<clinit>()`,真正给静态变量赋值,执行静态代码块 |

### 2.2 双亲委派模型

```
        Bootstrap ClassLoader (C++ 实现, 加载 rt.jar/java.base)
                  ▲
                  │ parent
        Extension ClassLoader (加载 jre/lib/ext)
                  ▲
                  │ parent
        Application ClassLoader (加载 classpath)
                  ▲
                  │ parent
        自定义 ClassLoader
```

**工作流程**:子加载器收到加载请求 → 先委派给父加载器 → 父加载器找不到才自己加载。

**为什么这样设计?**
- 避免类的重复加载(同名类只加载一次)
- 保证核心类库安全(自己写 `java.lang.String` 也不会被加载)

**何时打破双亲委派?**
- JDBC SPI(`Thread.setContextClassLoader`)
- OSGi、Tomcat(每个 Webapp 独立 ClassLoader,实现隔离)
- 热部署、模块化(JDK9+ 的 Layer)

### 2.3 类加载的时机

主动引用(必触发初始化):
- `new` 实例化、读写静态字段、调用静态方法
- 反射 `Class.forName()`
- 初始化子类时,父类先初始化
- JVM 启动主类(含 `main` 方法的类)

被动引用(不触发初始化):
- 子类引用父类静态字段(只初始化父类)
- 通过数组定义类(`Foo[] arr = new Foo[10]` 不初始化 Foo)
- 引用类的 `static final` 常量(编译期已放入调用方常量池)

---

## 三、字节码与类文件

### 3.1 Java 类文件结构

`.class` 文件是一组以 8 字节为单位的二进制流,严格按以下顺序排列:

```
ClassFile {
    u4             magic;              // 魔数 0xCAFEBABE
    u2             minor_version;      // 副版本号
    u2             major_version;      // 主版本号 (JDK 8 = 52, JDK 17 = 61)
    u2             constant_pool_count;
    cp_info        constant_pool[];    // 常量池(字面量+符号引用)
    u2             access_flags;       // 访问标志(public/final/abstract)
    u2             this_class;
    u2             super_class;
    u2             interfaces_count;
    u2             interfaces[];
    u2             fields_count;
    field_info     fields[];           // 字段表
    u2             methods_count;
    method_info    methods[];          // 方法表
    u2             attributes_count;
    attribute_info attributes[];       // 属性表(Code、LineNumberTable...)
}
```

### 3.2 javap 与字节码

`javap` 是分析字节码的瑞士军刀:

```bash
javap -c Foo.class           # 反编译查看字节码
javap -v Foo.class           # 详细信息(常量池、栈深度、局部变量表)
javap -p Foo.class           # 显示 private 成员
```

**示例**:`int a = 1 + 2;` 编译后只有一条指令 `iconst_3`(常量折叠)。

通过 javap 可以看穿 Java 的语法糖:
- 自动装箱 → `Integer.valueOf()`
- 增强 for 循环 → 数组用下标 / 集合用 Iterator
- 字符串拼接 → `StringBuilder.append()`(JDK 9+ 用 `invokedynamic`)
- 泛型 → 类型擦除 + 桥接方法

### 3.3 栈虚拟机 vs 寄存器虚拟机

| 维度 | 栈虚拟机(JVM) | 寄存器虚拟机(Dalvik/ART/Lua) |
|------|----------------|------------------------------|
| 操作数来源 | 操作数栈 | 虚拟寄存器 |
| 指令长度 | 短(无需指定操作数地址) | 长(需指定寄存器号) |
| 指令数量 | 多(频繁压栈出栈) | 少 |
| 可移植性 | 强(无寄存器假设) | 弱(依赖寄存器数量) |
| 执行效率 | 略低 | 略高 |

**示例:计算 `1 + 2`**

```
# JVM 栈式 (4 条指令)
iconst_1     // 1 入栈
iconst_2     // 2 入栈
iadd         // 弹出两数相加,结果入栈
istore_0    // 结果存入局部变量 0

# Dalvik 寄存器式 (2 条指令)
const/4 v0, 1
add-int/lit8 v0, v0, 2
```

### 3.4 字节码指令分类

JVM 共约 200 条指令,按功能分类:

| 类别 | 代表指令 | 说明 |
|------|----------|------|
| **加载存储** | `iload` `istore` `aload` `astore` | 局部变量与操作数栈互传 |
| **运算** | `iadd` `isub` `imul` `idiv` `irem` | 算术运算 |
| **类型转换** | `i2l` `f2d` `i2b` | 基本类型互转 |
| **对象操作** | `new` `getfield` `putfield` `instanceof` | 对象创建与字段访问 |
| **方法调用** | `invokevirtual` `invokestatic` `invokespecial` `invokeinterface` `invokedynamic` | 5 种调用 |
| **控制转移** | `goto` `if_icmpeq` `tableswitch` `lookupswitch` | 流程控制 |
| **异常** | `athrow` | 抛异常 |
| **同步** | `monitorenter` `monitorexit` | synchronized 实现 |

**5 种方法调用指令的差异**(高频面试题):

| 指令 | 用途 |
|------|------|
| `invokestatic` | 调用静态方法 |
| `invokespecial` | 调用构造方法、私有方法、父类方法(super) |
| `invokevirtual` | 调用普通实例方法(虚方法,运行期动态分派) |
| `invokeinterface` | 调用接口方法 |
| `invokedynamic` | JDK 7 引入,Lambda、字符串拼接(JDK 9+)的底层 |

---

## 四、运行时内存结构

### 4.1 整体内存布局(JDK 8+)

```
┌─────────────────────────────────────────────────┐
│              JVM Memory Layout                  │
├─────────────────────────────────────────────────┤
│  Thread Private (线程私有,生命周期同线程)       │
│   ┌───────────┐ ┌───────────┐ ┌───────────┐    │
│   │ PC Reg    │ │ JVM Stack │ │ Native    │    │
│   │ (程序计数器)│ │ (虚拟机栈)│ │ Stack     │    │
│   └───────────┘ └───────────┘ └───────────┘    │
├─────────────────────────────────────────────────┤
│  Thread Shared (线程共享)                        │
│   ┌───────────────────┐ ┌─────────────────┐    │
│   │      Heap         │ │  Metaspace      │    │
│   │  (对象实例,GC主战场)│ │  (类元数据,直接内存)│   │
│   └───────────────────┘ └─────────────────┘    │
├─────────────────────────────────────────────────┤
│  Direct Memory (堆外内存,NIO 使用)               │
└─────────────────────────────────────────────────┘
```

### 4.2 各区域详解

**程序计数器 (PC Register)**
- 存储当前线程执行字节码的行号指示器
- 唯一不会发生 OOM 的区域
- 执行 native 方法时值为 undefined

**虚拟机栈 (JVM Stack)**
- 每个方法对应一个栈帧
- 异常:`StackOverflowError`(栈深度超限)、`OutOfMemoryError`(动态扩展申请不到内存)
- 通过 `-Xss` 设置栈大小(默认 512K~1M)

**本地方法栈 (Native Stack)**
- 服务于 native 方法(JNI)
- HotSpot 中与虚拟机栈合二为一

**堆 (Heap)**
- **GC 主战场**,对象实例的"大本营"
- 通过 `-Xms`(初始)、`-Xmx`(最大)设置大小
- 分代结构:Young(Eden + S0 + S1) + Old

**方法区 / 元空间 (Metaspace)**
- 存储类元数据、运行时常量池、静态变量
- JDK 7 及以前叫"永久代",在堆中
- **JDK 8 改名元空间,放在直接内存**(避免 PermGen OOM)
- 通过 `-XX:MetaspaceSize` 和 `-XX:MaxMetaspaceSize` 控制

### 4.3 深入理解栈帧结构

栈帧(Stack Frame)是方法执行的基本单元:

```
┌────────────────────────┐
│   局部变量表             │ ← 存方法参数和局部变量(基本类型/对象引用)
│ Local Variable Table    │   按 Slot 划分(32位 1 Slot, long/double 2 Slot)
├────────────────────────┤
│   操作数栈              │ ← 字节码指令的"工作台",压栈出栈
│ Operand Stack           │   方法调用时作为参数传递
├────────────────────────┤
│   动态链接              │ ← 指向方法区中该方法的引用
│ Dynamic Linking         │   支持运行期符号引用→直接引用的转换
├────────────────────────┤
│   方法返回地址           │ ← 正常返回:调用者 PC;异常返回:由异常表决定
│ Return Address          │
└────────────────────────┘
```

**关键点**:
- 局部变量表大小在编译期就确定,写在方法的 Code 属性里(`max_locals`)
- 操作数栈最大深度同样编译期确定(`max_stack`)
- `this` 是局部变量表 Slot 0(实例方法)

### 4.4 常见 JVM 内存参数速查

```bash
-Xms2g                      # 堆初始大小
-Xmx2g                      # 堆最大大小(建议与 Xms 相等,避免动态扩展)
-Xmn1g                      # 新生代大小
-Xss512k                    # 每个线程栈大小
-XX:MetaspaceSize=256m      # 元空间初始大小
-XX:MaxMetaspaceSize=512m   # 元空间最大大小
-XX:SurvivorRatio=8         # Eden:S0:S1 = 8:1:1
-XX:NewRatio=2              # Old:Young = 2:1
-XX:+UseG1GC                # 使用 G1 收集器
-XX:MaxGCPauseMillis=200    # G1 期望最大停顿时间
-XX:+HeapDumpOnOutOfMemoryError  # OOM 时自动 dump
-XX:HeapDumpPath=/path/heap.hprof
```

---

## 五、对象的创建与分配

### 5.1 对象创建的 6 步流程

```
1. 类加载检查 → 是否已加载、解析、初始化
2. 分配内存   → 指针碰撞 / 空闲列表
3. 内存清零   → 实例字段赋默认值(0/null/false)
4. 设置对象头 → Mark Word + Klass Pointer + 数组长度
5. 执行 <init> → 构造方法,赋程序员指定的初始值
6. 返回引用   → 把对象地址给变量
```

### 5.2 对象内存布局(64位 HotSpot)

```
┌─────────────────────────────┐
│   Header (对象头)            │
│   ├── Mark Word (8B)         │  哈希码、GC分代年龄、锁状态、偏向锁线程ID
│   ├── Klass Pointer (4/8B)   │  类型指针(开启压缩指针后 4B)
│   └── Array Length (4B)      │  仅数组对象有
├─────────────────────────────┤
│   Instance Data (实例数据)    │  字段值,按 long/double > int > short > byte 排列
├─────────────────────────────┤
│   Padding (对齐填充)          │  保证对象大小是 8B 的倍数
└─────────────────────────────┘
```

**普通空对象 = 16B**(头 12B + 填充 4B);开启压缩指针时。

### 5.3 Java 创建的对象到底放在哪?

**默认答案**:堆。
**深入答案**:**不一定**。HotSpot 通过 **逃逸分析** 优化分配策略:

```
                ┌──────────────┐
对象创建 ──→  逃逸分析判断
                │
        ┌───────┼────────┐
        ▼       ▼        ▼
    全局逃逸  方法逃逸   未逃逸
        │       │        │
        ▼       ▼        ▼
       堆      堆      栈/标量替换
```

**三种优化**:
1. **栈上分配**:对象不逃逸出方法,直接在栈帧上分配,随栈帧销毁
2. **标量替换**:把对象拆成基本类型,直接放寄存器/栈,连对象都不创建
3. **同步消除**:确认只有当前线程访问,自动去掉 synchronized

**TLAB(Thread Local Allocation Buffer)**:
为减少多线程分配竞争,每个线程在 Eden 中预分配一小块私有空间(默认占 Eden 1%)。线程创建对象时优先在自己的 TLAB 中"指针碰撞"分配,无需加锁。

```bash
-XX:+UseTLAB                # 默认开启
-XX:TLABSize=512k           # 设置 TLAB 大小
-XX:+PrintTLAB              # 打印 TLAB 使用情况
```

### 5.4 对象分配的完整路径

```
new Object()
    │
    ▼
逃逸分析? ──是──→ 栈上分配 / 标量替换
    │
    否
    ▼
TLAB 够用? ──是──→ TLAB 内指针碰撞
    │
    否
    ▼
对象大? ──是──→ 直接进老年代 (-XX:PretenureSizeThreshold)
    │
    否
    ▼
Eden 区分配 ──→ Eden 满 ──→ Minor GC ──→ 存活对象进 Survivor
                                          │
                                          ▼
                                    年龄达阈值/动态年龄判定 ──→ 晋升老年代
```

---

## 六、垃圾回收机制

### 6.1 如何判断对象已死?

**引用计数法**(Java 不用):
- 对象有计数器,被引用 +1,引用失效 -1,为 0 即可回收
- ❌ 致命缺陷:无法解决循环引用

**可达性分析**(Java 实际使用):
- 从 **GC Roots** 出发向下搜索,不可达的对象判定为可回收
- **GC Roots 包括**:
  - 虚拟机栈中引用的对象(局部变量)
  - 方法区中类静态属性引用的对象
  - 方法区中常量引用的对象
  - 本地方法栈中 JNI 引用的对象
  - 活动线程、Synchronized 持有的对象

### 6.2 四种引用强度

| 引用类型 | 回收时机 | 典型场景 |
|---------|---------|---------|
| **强引用** | 永不回收(OOM 也不回收) | 普通对象 `Object o = new Object()` |
| **软引用** | 内存不足时回收 | 缓存(`SoftReference`) |
| **弱引用** | 下次 GC 时回收 | `WeakHashMap`、ThreadLocal 的 Entry key |
| **虚引用** | 任何时候都可能回收 | 跟踪对象被回收的状态(配合 ReferenceQueue) |

### 6.3 三种基础回收算法

**标记-清除 (Mark-Sweep)**
```
[A][B][ ][C][ ][D][E][ ]   ← 标记前
[A][ ][ ][C][ ][ ][E][ ]   ← 清除后(碎片化)
```
- 优点:简单
- 缺点:**内存碎片**,大对象分配可能触发 Full GC

**标记-复制 (Mark-Copy)**
```
From: [A][B][ ][C][ ][D]
To:   [ ][ ][ ][ ][ ][ ]
   ↓ 复制存活对象
From: [ ][ ][ ][ ][ ][ ]
To:   [A][C][D][ ][ ][ ]
```
- 优点:无碎片,效率高
- 缺点:浪费一半空间(Survivor 区改为 8:1:1 优化)

**标记-整理 (Mark-Compact)**
```
[A][B][ ][C][ ][D][E][ ]   ← 标记
[A][C][D][E][ ][ ][ ][ ]   ← 整理(存活对象向一端移动)
```
- 优点:无碎片,无空间浪费
- 缺点:移动对象成本高,需 STW

### 6.4 分代收集理论

**三大经验假说**:
1. **弱分代假说**:绝大多数对象都是朝生夕灭的
2. **强分代假说**:熬过越多次 GC 的对象越难消亡
3. **跨代引用假说**:跨代引用相对同代引用是少数

**分代结构**:

```
┌─────────────────────────────────┬─────────────┐
│        新生代 (1/3 堆)           │  老年代(2/3) │
├──────────┬──────────┬───────────┤             │
│  Eden    │   S0     │    S1     │             │
│  (8/10)  │  (1/10)  │   (1/10)  │             │
└──────────┴──────────┴───────────┴─────────────┘
       Minor GC 主战场                 Major GC
```

**对象晋升老年代的 4 种方式**:
1. 年龄达到阈值(默认 15,`-XX:MaxTenuringThreshold`)
2. 大对象直接进老年代(`-XX:PretenureSizeThreshold`)
3. **动态年龄判定**:Survivor 中相同年龄对象总和 > S 区一半,该年龄及以上全部晋升
4. **空间分配担保**:Minor GC 时 Survivor 放不下,直接进老年代

### 6.5 GC 类型

| 名称 | 范围 | 触发条件 |
|------|------|---------|
| **Minor GC / Young GC** | 新生代 | Eden 满 |
| **Major GC / Old GC** | 老年代 | 老年代满(部分收集器有) |
| **Mixed GC** | 新生代 + 部分老年代 | G1 特有 |
| **Full GC** | 整个堆 + 方法区 | 老年代满、System.gc()、元空间满、空间分配担保失败 |

**Stop The World (STW)**:GC 时所有用户线程暂停,只跑 GC 线程。优化目标:**减少 STW 时间**(尤其老年代 GC)。

---

## 七、垃圾收集器全家桶

### 7.1 收集器对比表

| 收集器 | 区域 | 算法 | 模式 | 特点 | 适用场景 |
|--------|------|------|------|------|---------|
| **Serial** | 新生代 | 复制 | 串行 STW | 简单高效(单核) | 客户端、小内存 |
| **Serial Old** | 老年代 | 标记-整理 | 串行 STW | Serial 老年代版 | 客户端 |
| **ParNew** | 新生代 | 复制 | 并行 STW | Serial 多线程版 | 配合 CMS |
| **Parallel Scavenge** | 新生代 | 复制 | 并行 STW | **吞吐量优先** | 后台计算任务 |
| **Parallel Old** | 老年代 | 标记-整理 | 并行 STW | PS 老年代版 | 后台计算 |
| **CMS** | 老年代 | 标记-清除 | **并发** | **低延迟**(JDK 14 移除) | Web 服务(老) |
| **G1** | 全堆 | 标记-整理 + 复制 | 并发 + 并行 | **可预测停顿** | 大堆、平衡型(JDK 9+ 默认) |
| **ZGC** | 全堆 | 染色指针 + 读屏障 | 并发 | **亚毫秒级停顿**,支持 TB 级堆 | 超低延迟 |
| **Shenandoah** | 全堆 | Brooks 指针 | 并发 | 与 ZGC 类似 | RedHat 主推 |

### 7.2 G1 收集器深入

**核心思想**:把堆划分为多个大小相等的 **Region**(默认 2048 个,大小 1~32MB),每个 Region 可以是 Eden / Survivor / Old / Humongous(大对象,跨多个 Region)。

```
┌──┬──┬──┬──┬──┬──┬──┬──┐
│E │O │S │E │H │O │E │O │   ← Region 不固定身份
├──┼──┼──┼──┼──┼──┼──┼──┤
│O │E │E │S │O │E │H │H │
└──┴──┴──┴──┴──┴──┴──┴──┘
```

**收集流程**:
1. **初始标记**(STW,极快):标记 GC Roots 直接关联的对象
2. **并发标记**:从 GC Roots 遍历对象图(与用户线程并行)
3. **最终标记**(STW):处理 SATB 残留
4. **筛选回收**(STW):按 Region 回收价值排序,在停顿目标内回收最多的

**关键参数**:
```bash
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200       # 期望最大停顿(默认 200ms)
-XX:G1HeapRegionSize=4m        # Region 大小
-XX:InitiatingHeapOccupancyPercent=45   # 触发并发标记的堆占用阈值
```

### 7.3 ZGC 极简介绍

**3 大杀手锏**:
- **染色指针 (Colored Pointer)**:把 GC 信息存在指针的 4 位 metadata bit 中
- **读屏障 (Load Barrier)**:读对象时检查指针,实现并发转移
- **NUMA 感知**:针对多 CPU 架构优化

**性能**:停顿时间 < 10ms(JDK 17 ZGC),不随堆大小增加。支持 8MB ~ 16TB 堆。

---

## 八、JIT 即时编译

### 8.1 解释器 vs 编译器

| 维度 | 解释器 | JIT 编译器 |
|------|--------|-----------|
| 启动 | 快(无需编译) | 慢(编译耗时) |
| 执行 | 慢(逐条解释) | 快(直接机器码) |
| 内存 | 省 | 占用编译缓存 |

**HotSpot 策略**:混合模式 + 分层编译

### 8.2 热点探测

**两个计数器**:
- **方法调用计数器**:统计方法被调用的次数
- **回边计数器**:统计循环回跳的次数(OSR On-Stack Replacement)

任意一个超过阈值即触发 JIT 编译。Client 模式默认 1500 次,Server 模式 10000 次。

### 8.3 分层编译 (Tiered Compilation)

JDK 7+ 默认开启,共 5 层:

```
Level 0: 解释执行
Level 1: C1 编译,无 profiling
Level 2: C1 编译,带方法/回边计数 profiling
Level 3: C1 编译,带完整 profiling
Level 4: C2 编译,激进优化(基于 Profile-Guided Optimization)
```

执行路径:`L0 → L3 → L4`(常见)。

### 8.4 经典 JIT 优化

**方法内联 (Method Inlining)**:把被调方法体直接嵌入调用处,消除调用开销。
```java
// 内联前
int sum(int a, int b) { return a + b; }
int x = sum(1, 2);

// 内联后
int x = 1 + 2;  // 进而被常量折叠为 x = 3
```

**逃逸分析 (Escape Analysis)**:见 5.3 节。

**锁消除 (Lock Elimination)**:确认无竞争时消除锁。
```java
// StringBuffer 是同步的,但局部变量无逃逸,锁会被消除
public String concat(String a, String b) {
    StringBuffer sb = new StringBuffer();
    sb.append(a).append(b);
    return sb.toString();
}
```

**锁粗化 (Lock Coarsening)**:连续加同一把锁时合并。

**公共子表达式消除 / 循环展开 / 死代码消除 / 常量折叠**:同 C++ 编译器优化。

### 8.5 AOT 与 GraalVM

**AOT (Ahead-Of-Time)**:JDK 9+ 引入 `jaotc`,提前编译为机器码。
**GraalVM Native Image**:把 Java 程序编译为独立可执行文件,启动毫秒级,内存占用低,适合 Serverless / 微服务冷启动场景。代价:动态特性受限(反射需配置)。

---

## 九、性能监控三板斧

### 9.1 命令行篇

JDK 自带的 6 大神器:

| 命令 | 用途 | 常用示例 |
|------|------|---------|
| `jps` | 查看 Java 进程 | `jps -lv` |
| `jstat` | 监控 GC、类加载 | `jstat -gcutil <pid> 1000` 每秒打印 GC |
| `jinfo` | 查看/修改 JVM 参数 | `jinfo -flag MaxHeapSize <pid>` |
| `jmap` | 堆 dump、堆统计 | `jmap -dump:format=b,file=heap.hprof <pid>` |
| `jstack` | 线程栈 dump | `jstack -l <pid> > stack.txt` |
| `jcmd` | 综合诊断(推荐) | `jcmd <pid> GC.heap_dump heap.hprof` |

**jstat 输出解读示例**:
```
S0    S1    E     O     M    YGC YGCT FGC FGCT GCT
0.00  85.0  60.5  20.3  98.2 100  1.5  2   0.5  2.0
```
- `S0/S1`:Survivor 0/1 使用率
- `E`:Eden 使用率
- `O`:老年代使用率
- `M`:Metaspace 使用率
- `YGC/YGCT`:Young GC 次数/总耗时
- `FGC/FGCT`:Full GC 次数/总耗时

### 9.2 可视化篇

| 工具 | 来源 | 特点 |
|------|------|------|
| **JConsole** | JDK 自带 | 入门级,功能简单 |
| **VisualVM** | JDK 自带(JDK 9+ 单独下载) | 插件丰富,支持 dump 分析 |
| **JMC (Java Mission Control)** | Oracle | 强大的 JFR 分析 |
| **JFR (Java Flight Recorder)** | JDK 11+ 完全开源 | 低开销飞行记录 |
| **MAT (Memory Analyzer Tool)** | Eclipse | 堆 dump 分析神器 |
| **JProfiler / YourKit** | 商业 | 全方位 profiling |

**JFR 使用**:
```bash
# 启动应用时开启
java -XX:StartFlightRecording=duration=60s,filename=record.jfr MyApp

# 运行中触发
jcmd <pid> JFR.start duration=60s filename=record.jfr
```

### 9.3 Arthas 篇(阿里开源神器)

**安装运行**:
```bash
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar
```

**核心命令清单**:

| 命令 | 用途 |
|------|------|
| `dashboard` | 实时监控面板(线程、内存、GC) |
| `thread` | 线程信息,`thread -b` 找死锁 |
| `thread -n 3` | CPU 占用 Top 3 线程 |
| `jvm` | 查看 JVM 运行参数 |
| `sysprop` / `sysenv` | 查看系统属性/环境变量 |
| `sc` | 搜索已加载的类(Search Class) |
| `sm` | 搜索类的方法(Search Method) |
| `jad` | **反编译运行中的类** |
| `watch` | 观察方法的参数、返回值、异常 |
| `trace` | 跟踪方法调用链路与耗时 |
| `stack` | 输出当前方法被调用的路径 |
| `tt` | 时光隧道,记录方法执行历史 |
| `monitor` | 周期性监控方法调用次数、耗时 |
| `redefine` | **热更新类**(应急修复) |
| `profiler` | 火焰图生成(集成 async-profiler) |

**实战示例**:线上接口变慢
```bash
# 1. trace 找到耗时方法
$ trace com.xxx.Service queryUser

# 2. watch 观察参数返回值
$ watch com.xxx.Service queryUser '{params, returnObj}' -x 2

# 3. profiler 生成火焰图
$ profiler start
$ profiler stop --file /tmp/flame.html
```

---

## 十、实战:内存泄露排查

### 10.1 现象与定位

**现象**:
- 老年代占用率持续上涨
- Full GC 频繁,但回收效果差
- 最终 `OutOfMemoryError: Java heap space`

**核心思路**:**dump → 分析 → 定位 GC Roots 引用链**

### 10.2 完整排查流程

```bash
# 1. 确认问题:观察 GC 趋势
jstat -gcutil <pid> 1000 30

# 2. dump 堆(应用还活着时)
jmap -dump:live,format=b,file=heap.hprof <pid>
# 或 OOM 时自动 dump
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/path/heap.hprof

# 3. 用 MAT 打开分析
# 关注三个视图:
#   - Histogram: 按类型统计对象数和占用
#   - Dominator Tree: 支配树,找内存大户
#   - Leak Suspects: MAT 自动诊断的可疑泄露
```

### 10.3 常见内存泄露场景

**1. 静态集合长期持有**
```java
public class Cache {
    private static Map<String, Object> CACHE = new HashMap<>();
    public static void put(String k, Object v) {
        CACHE.put(k, v);  // 永不清理 → 泄露
    }
}
// ✅ 修复:换用 WeakHashMap / 定期清理 / Guava Cache
```

**2. 资源未关闭**
```java
// ❌ 流没关
InputStream in = new FileInputStream(file);
// ✅ try-with-resources
try (InputStream in = new FileInputStream(file)) { ... }
```

**3. ThreadLocal 未 remove**
```java
// 线程池场景下,ThreadLocal 存的对象会随线程长期存活
threadLocal.set(bigObject);
// ✅ 用完必须
threadLocal.remove();
```

**4. 监听器未注销**
```java
eventBus.register(this);   // 长生命周期对象持有短生命周期的 this
// ✅ 在合适时机
eventBus.unregister(this);
```

**5. 内部类隐式持有外部类**
```java
public class Outer {
    private byte[] big = new byte[100_000_000];
    public Runnable getTask() {
        return new Runnable() {  // 非静态内部类隐式持有 Outer.this
            @Override public void run() { ... }
        };
    }
}
// ✅ 改为静态内部类 + 弱引用
```

### 10.4 MAT 分析技巧

- **Path to GC Roots**:右键对象 → exclude weak/soft references → 找到强引用链
- **Histogram → Group by Package**:快速定位是哪个业务模块的问题
- **OQL (Object Query Language)**:`SELECT * FROM java.util.HashMap WHERE size > 1000`

---

## 十一、实战:CPU 100% 排查

### 11.1 经典四步法

```bash
# 1. 找到占用高的 Java 进程
$ top
PID  USER  %CPU  %MEM  COMMAND
8888 app   780   30.5  java

# 2. 找到该进程内占用高的线程
$ top -Hp 8888
PID  USER  %CPU  %MEM  COMMAND
8901 app   95.2  ...   java
8902 app   94.8  ...   java

# 3. 线程 ID 转十六进制
$ printf "%x\n" 8901
22c5

# 4. jstack 查看该线程在做什么
$ jstack 8888 | grep -A 30 "0x22c5"
"http-nio-8080-exec-3" #45 daemon prio=5 tid=0x... nid=0x22c5 runnable
   at com.xxx.MyClass.busyLoop(MyClass.java:42)
   ...
```

### 11.2 常见 CPU 飙高元凶

| 元凶 | 特征 | 解决方案 |
|------|------|---------|
| **死循环 / 无限递归** | 单线程 100% | 修代码逻辑 |
| **频繁 Full GC** | GC 线程占用高,业务线程大量 BLOCKED | 调堆大小、优化对象创建 |
| **线程上下文切换** | sy 高(`vmstat 1`) | 减少线程数、优化锁 |
| **正则回溯爆炸** | 单个正则匹配卡死 | 检查正则贪婪/回溯 |
| **序列化爆炸** | JSON/Hessian 大对象 | 控制对象大小、改用 Protobuf |
| **锁竞争** | 大量 BLOCKED 线程,等同一把锁 | 降低锁粒度、用 CAS / 读写锁 |
| **大量定时任务并发** | ScheduledExecutor 线程繁忙 | 错峰、改异步队列 |

### 11.3 一键排查脚本

```bash
#!/bin/bash
# cpu_top.sh - 自动定位 CPU 高的 Java 线程
PID=$1
TOP_TID=$(ps -mp $PID -o THREAD,tid | sort -rn | head -2 | tail -1 | awk '{print $8}')
HEX_TID=$(printf "%x" $TOP_TID)
jstack $PID | grep -A 30 "nid=0x$HEX_TID"
```

### 11.4 进阶:用 Arthas 一键定位

```bash
# 在 Arthas 中
$ thread -n 3                    # 直接显示 CPU Top 3 线程及栈
$ profiler start                 # 启动火焰图采样
# ... 等 30 秒 ...
$ profiler stop --file flame.html
# 火焰图中越宽的方法 = 占用 CPU 越多
```

---

## 十二、核心知识点总结

### 12.1 一图流(全景图)

```
                ┌──────────────────────────┐
                │      Java 源代码 (.java)   │
                └────────────┬─────────────┘
                             │ javac
                             ▼
                ┌──────────────────────────┐
                │      字节码 (.class)       │
                └────────────┬─────────────┘
                             │ ClassLoader (双亲委派)
                             ▼
        ┌────────────────────┴──────────────────────┐
        │              JVM 运行时数据区              │
        ├─────────────┬──────────────┬──────────────┤
        │  线程私有    │    线程共享    │   堆外内存    │
        │ PC/Stack    │ Heap+Metaspace│ Direct Memory │
        └──────┬──────┴───────┬──────┴──────────────┘
               │              │
               ▼              ▼
        ┌─────────────┐  ┌──────────────┐
        │  执行引擎    │  │   GC 系统     │
        │ 解释 + JIT   │  │ 分代+G1/ZGC   │
        └─────────────┘  └──────────────┘
               │              │
               └──────┬───────┘
                      ▼
        ┌──────────────────────────┐
        │    监控工具 + 实战排查      │
        │ jstat/jmap/Arthas/MAT    │
        └──────────────────────────┘
```

### 12.2 高频面试题(自查清单)

- [ ] JVM 内存结构有哪些?哪些线程私有?
- [ ] 元空间和永久代的区别?为什么改成元空间?
- [ ] 双亲委派模型?为什么要破坏?怎么破坏?
- [ ] 类加载的过程?静态变量在哪个阶段赋值?
- [ ] 5 种方法调用指令的区别?
- [ ] 对象的内存布局?如何计算对象大小?
- [ ] 如何判断对象可回收?GC Roots 有哪些?
- [ ] 强软弱虚四种引用?各自使用场景?
- [ ] CMS 与 G1 的区别?G1 的 Region 设计?
- [ ] ZGC 为什么这么快?染色指针是什么?
- [ ] 什么是 Stop The World?如何减少 STW?
- [ ] 逃逸分析是什么?有哪些优化?
- [ ] TLAB 是什么?为什么需要它?
- [ ] 什么情况会触发 Full GC?如何避免?
- [ ] 线上 OOM 如何排查?CPU 100% 如何排查?
- [ ] 介绍下 JIT 编译?C1 和 C2 的区别?
- [ ] String 在 JVM 中的存储?intern 方法?

### 12.3 调优黄金法则

1. **不要过早优化**:先压测找瓶颈,再针对性调优
2. **监控先行**:没有数据就没有调优,先把 JFR/Prometheus 加上
3. **Xms = Xmx**:避免运行期堆动态扩展抖动
4. **OOM 必 dump**:加 `-XX:+HeapDumpOnOutOfMemoryError`
5. **GC 日志必开**:`-Xlog:gc*:file=gc.log:time,uptime:filecount=10,filesize=10m`
6. **选对收集器**:延迟敏感选 ZGC / G1,吞吐优先选 Parallel
7. **关注业务指标**:GC 是手段,业务 SLA(P99 延迟、TPS)才是目的

### 12.4 推荐资源

📚 **必读书籍**
- 《深入理解 Java 虚拟机》(第 3 版) —— 周志明 ⭐⭐⭐⭐⭐
- 《Java 性能权威指南》—— Scott Oaks
- 《实战 Java 虚拟机》—— 葛一鸣

🌐 **在线资料**
- [OpenJDK 官方 Wiki](https://wiki.openjdk.org/)
- [HotSpot 源码 GitHub](https://github.com/openjdk/jdk)
- [Arthas 官方文档](https://arthas.aliyun.com/)
- [GCeasy](https://gceasy.io/)(在线 GC 日志分析)
- [HeapHero](https://heaphero.io/)(在线堆分析)

🛠 **工具栈**
- 监控:Prometheus + Grafana + Micrometer
- 链路:SkyWalking / Pinpoint
- 诊断:Arthas + JFR + MAT

---

## 📌 Day 21 总结

| 维度 | 收获 |
|------|------|
| **理论闭环** | 从源码 → 字节码 → 加载 → 执行 → GC,完整理解 Java 程序生命周期 |
| **内存模型** | 清晰掌握 JVM 各内存区域职责、对象分配路径、栈帧结构 |
| **GC 演进** | 理解从 Serial 到 ZGC 的设计权衡(吞吐 vs 延迟 vs 内存) |
| **JIT 优化** | 知道方法内联、逃逸分析、锁消除等优化背后的原理 |
| **实战能力** | 掌握 OOM、CPU 100%、Full GC 频繁的标准排查 SOP |
| **工具链** | 命令行 + 可视化 + Arthas 三件套,覆盖线上诊断全场景 |

> JVM 是 Java 工程师的内功,值得每隔一段时间回炉重造。下次面试或线上故障,你会感谢今天努力的自己。

**Day 21 ✅ Done — 持续更新中,欢迎 Star ⭐**
