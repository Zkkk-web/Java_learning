# JVM 调优与类加载机制 | Day 20

> JVM 监控、性能调优、GC 问题排查、类加载机制、双亲委派模型

---

## 一、可视化性能监控工具有哪些？

### JDK 自带工具

```
jps      → 查看 Java 进程列表
jstack   → 查看线程堆栈（排查死锁、线程卡住）
jmap     → 查看内存信息（堆转储、对象统计）
jstat    → 监控 GC 统计信息
jinfo    → 查看/修改 JVM 参数
jcmd     → 多功能命令行工具（替代上面几个）
```

### 可视化工具

```
VisualVM（推荐）
  JDK 自带，图形化界面
  查看堆、线程、GC、CPU 等
  支持远程监控

JConsole
  JDK 自带，轻量级
  查看内存、线程、类、MBean

Arthas（阿里开源，生产环境推荐）
  无需重启，线上诊断
  热修复、方法追踪、性能分析
  命令：watch、trace、dashboard 等

JProfiler / YourKit（商业）
  专业性能分析
```

### Arthas 常用命令

```bash
# 启动
java -jar arthas-boot.jar

# 查看 JVM 整体状态
dashboard

# 查看某个类的方法调用
watch com.example.UserService getUserById "{params, returnObj}" -x 3

# 查看方法调用链路耗时
trace com.example.UserService getUserById

# 反编译某个类（查看是否热部署成功）
jad com.example.UserService

# 查看类加载信息
classloader
```

---

## 二、JVM 的常见参数配置

### 堆内存参数

```bash
-Xms512m          # 初始堆大小（建议和Xmx一样，避免动态扩容）
-Xmx2g            # 最大堆大小
-Xmn512m          # 新生代大小
-XX:NewRatio=2    # 老年代:新生代 = 2:1
-XX:SurvivorRatio=8  # Eden:S0:S1 = 8:1:1
```

### 元空间参数

```bash
-XX:MetaspaceSize=256m     # 元空间初始大小
-XX:MaxMetaspaceSize=512m  # 元空间最大大小
```

### GC 相关参数

```bash
-XX:+UseG1GC              # 使用 G1 垃圾回收器（推荐）
-XX:MaxGCPauseMillis=200  # G1 最大停顿时间目标（毫秒）
-XX:+PrintGCDetails       # 打印 GC 详情
-XX:+PrintGCDateStamps    # 打印 GC 时间戳
-Xloggc:/logs/gc.log      # GC 日志文件
```

### OOM 相关参数

```bash
-XX:+HeapDumpOnOutOfMemoryError   # OOM时自动转储堆快照
-XX:HeapDumpPath=/dump/heap.hprof # 堆快照路径
-XX:+ExitOnOutOfMemoryError       # OOM后自动退出（容器环境推荐）
```

---

## 三、做过 JVM 调优吗？

### 调优思路

```
① 确认问题
  GC 频繁？Full GC？OOM？CPU高？响应慢？

② 收集数据
  jstat -gcutil <pid> 1000  每隔1秒打印GC统计
  jmap -dump:format=b,file=heap.hprof <pid>  堆快照

③ 分析问题
  用 VisualVM 或 MAT 分析堆快照
  找到占内存最多的对象
  找到 GC 频繁的原因

④ 调整参数
  根据分析结果调整堆大小、GC策略

⑤ 验证效果
  观察 GC 频率、停顿时间、响应时间是否改善
```

### 常见调优案例

```
案例1：老年代占用过高 → Full GC 频繁
  分析：可能有内存泄漏，或者对象晋升太快
  解决：扩大新生代，检查内存泄漏，用 MAT 分析

案例2：接口响应偶尔很慢
  分析：可能是 GC 停顿（STW）
  解决：换 G1 或 ZGC，减少停顿时间

案例3：内存占用高但 GC 后回收不了
  分析：内存泄漏
  解决：堆快照分析，找到泄漏的对象
```

---

## 四、CPU 占用过高怎么排查？⭐

### 步骤

```bash
# 第1步：找到 CPU 高的 Java 进程
top

# 第2步：找到进程里 CPU 高的线程
top -Hp <pid>
# 找到 CPU 高的线程ID（十进制）

# 第3步：把线程ID转成十六进制
printf "%x\n" <线程ID>

# 第4步：查看线程堆栈
jstack <pid> | grep <十六进制线程ID> -A 30

# 第5步：定位到代码
# 堆栈里能看到线程在执行哪个方法
```

### 常见原因

```
死循环：线程一直跑，CPU 飙高
频繁 GC：GC 线程占用大量 CPU
正则表达式回溯：复杂正则消耗大量 CPU
序列化/反序列化过多
```

---

## 五、内存飙高问题怎么排查？⭐

### 步骤

```bash
# 第1步：查看堆内存使用情况
jmap -heap <pid>

# 第2步：查看对象统计（哪些对象最多）
jmap -histo <pid> | head -20

# 第3步：导出堆快照
jmap -dump:format=b,file=heap.hprof <pid>

# 第4步：用 MAT 或 VisualVM 分析
# 找到最占内存的对象和引用链
```

### 常见原因

```
内存泄漏：对象被持有，无法 GC 回收
  → ThreadLocal 没有 remove
  → 静态集合一直添加元素
  → 监听器/回调没有移除

内存溢出：数据量太大，超过堆大小
  → 查询数据库没有分页
  → 大文件一次性读入内存
  → 缓存没有设置上限
```

---

## 六、频繁 Minor GC 怎么办？

### 原因分析

```
Minor GC 触发：Eden 区满了

频繁 Minor GC 说明：
  ① 新生代太小，Eden 很快装满
  ② 对象创建太快，产生大量短命对象
  ③ Survivor 太小，存活对象放不下，过早晋升老年代
```

### 解决方案

```
① 增大新生代
  -Xmn 调大新生代大小
  -XX:NewRatio 调整比例

② 增大 Eden 和 Survivor
  -XX:SurvivorRatio=8（默认）可以调整

③ 检查代码
  减少无效对象创建
  字符串拼接用 StringBuilder
  复用对象（对象池）

④ 调整晋升阈值
  -XX:MaxTenuringThreshold=15（默认15）
  调大，让对象在新生代多待一会儿
```

---

## 七、频繁 Full GC 怎么办？⭐

### 触发原因

```
① 老年代空间不足
  Minor GC 后存活对象太多，放不进老年代
  大对象直接分配到老年代

② 元空间不足
  类加载太多，元空间满了

③ 显式调用 System.gc()
  代码里手动触发

④ 空间分配担保失败
  Minor GC 前检查老年代空间不足
```

### 排查步骤

```bash
# 查看 GC 情况
jstat -gcutil <pid> 1000

输出含义：
S0    S1    E     O     M     CCS   YGC   YGCT  FGC   FGCT  GCT
0.00  0.00  73.2  28.4  95.2  93.4  10    0.318  2     0.342  0.661

S0/S1 = Survivor
E = Eden
O = Old（老年代）
M = Metaspace
YGC = Minor GC 次数
FGC = Full GC 次数
```

### 解决方案

```
① 老年代不足 → 扩大堆或优化代码（减少长生命周期对象）
② 元空间不足 → 增大 MaxMetaspaceSize
③ 内存泄漏  → 堆快照分析，找到泄漏点
④ 大对象太多 → 分批处理，避免大对象
⑤ 换 G1 GC  → 更好的 GC 策略，减少 Full GC
```

---

## 八、了解类的加载机制吗？⭐

### 是什么？

```
把 .class 字节码文件加载到 JVM 内存
并创建对应的 Class 对象
```

### 加载的触发时机

```
① new 了一个类的实例
② 调用类的静态方法或访问静态字段
③ 使用反射 Class.forName()
④ 子类被加载时，父类先加载
⑤ JVM 启动时，加载主类（含 main 方法的类）
```

---

## 九、类加载器有哪些？⭐

```
① Bootstrap ClassLoader（启动类加载器）
  加载 JDK 核心类库（rt.jar）
  C++ 实现，Java 里无法直接引用（返回null）
  加载路径：$JAVA_HOME/jre/lib

② Extension ClassLoader（扩展类加载器）
  加载扩展类库
  加载路径：$JAVA_HOME/jre/lib/ext
  JDK9 之后改名为 Platform ClassLoader

③ Application ClassLoader（应用类加载器）
  加载 classpath 下的类（自己写的类）
  默认的类加载器

④ 自定义类加载器
  继承 ClassLoader，重写 findClass()
  可以从网络、数据库加载类
  实现类隔离、热部署等功能
```

---

## 十、类的生命周期

```
① 加载（Loading）
  读取 .class 文件到内存
  创建 Class 对象

② 验证（Verification）
  检查字节码格式是否正确
  安全性检查

③ 准备（Preparation）
  为静态变量分配内存
  赋默认值（int=0，引用=null）
  注意：不是赋初始值！

④ 解析（Resolution）
  把符号引用替换为直接引用
  把类名字符串替换为内存地址

⑤ 初始化（Initialization）
  执行静态代码块和静态变量赋值
  真正执行 Java 代码的阶段

⑥ 使用（Using）

⑦ 卸载（Unloading）
  类不再被使用，GC 回收 Class 对象
```

---

## 十一、类装载的过程（详细）⭐

```
加载阶段：
  ① 通过类的全限定名找到 .class 文件
  ② 读取字节码到内存
  ③ 在方法区创建类的运行时数据结构
  ④ 在堆里创建 Class 对象（作为访问入口）

验证阶段：
  文件格式验证（魔数 0xCAFEBABE）
  元数据验证（语义分析）
  字节码验证（代码逻辑）
  符号引用验证

准备阶段：
  static int i = 100;  → 此阶段 i = 0（默认值）
  static final int i = 100;  → 此阶段 i = 100（常量，直接赋值）

初始化阶段：
  static int i = 100;  → 此阶段 i = 100（真正赋值）
  执行 static {} 代码块
```

---

## 十二、什么是双亲委派模型？⭐

### 是什么？

```
类加载器在加载类时，先委托父类加载器加载
父类加载器找不到，才自己加载

流程：
  自定义ClassLoader
    ↓ 委托
  Application ClassLoader
    ↓ 委托
  Extension ClassLoader
    ↓ 委托
  Bootstrap ClassLoader
    ↓ 找不到才往下找
  Extension ClassLoader 自己加载
    ↓ 找不到
  Application ClassLoader 自己加载
    ↓ 找不到
  自定义ClassLoader 自己加载
    ↓ 找不到
  ClassNotFoundException
```

---

## 十三、为什么要用双亲委派模型？

```
① 安全性
  防止用户自定义 java.lang.String 替换核心类
  核心类由 Bootstrap ClassLoader 加载，不会被覆盖

② 避免重复加载
  父类加载过的类，子类不会再加载
  同一个类只有一份 Class 对象

③ 保证核心类的唯一性
  不同的类加载器加载同一个类，得到不同的 Class 对象
  双亲委派保证核心类只有一个实例
```

---

## 十四、如何破坏双亲委派机制？

```java
// 继承 ClassLoader，重写 loadClass 方法
// loadClass 包含双亲委派逻辑
// 重写它就可以破坏双亲委派

public class MyClassLoader extends ClassLoader {
    @Override
    public Class<?> loadClass(String name) throws ClassNotFoundException {
        // 不委托父类，直接自己加载
        // 这就破坏了双亲委派
        return findClass(name);
    }

    @Override
    protected Class<?> findClass(String name) {
        // 自己加载字节码
        byte[] bytes = loadClassBytes(name);
        return defineClass(name, bytes, 0, bytes.length);
    }
}
```

---

## 十五、有哪些破坏双亲委派的例子？

```
① JDBC
  java.sql.Driver 是 JDK 的接口（Bootstrap加载）
  但实现类（mysql-connector）在 classpath（Application加载）
  父类加载器加载接口，子类加载器加载实现
  → 用线程上下文类加载器打破

② Tomcat
  不同 Web 应用可能有相同的类名但不同的版本
  Tomcat 为每个 Web 应用创建独立的 WebAppClassLoader
  实现应用间的类隔离
  打破双亲委派，各加载各的

③ OSGi
  模块化框架
  每个模块（Bundle）有自己的类加载器
  模块间通过导入导出机制共享类

④ 热部署
  修改代码后不重启
  创建新的类加载器重新加载修改的类
  旧的类加载器和类被 GC 回收
```

---

## 十六、Tomcat 的类加载机制

```
Tomcat 类加载器层次：
  Bootstrap ClassLoader（JDK核心）
    ↓
  Extension ClassLoader
    ↓
  Application ClassLoader（Tomcat自身）
    ↓
  Common ClassLoader（Tomcat公共库）
    ↓
  Catalina ClassLoader     WebApp ClassLoader1
  （Tomcat私有）            （Web应用1）
                           WebApp ClassLoader2
                           （Web应用2）

特点：
  每个 Web 应用有独立的 WebAppClassLoader
  不同应用的同名类互不影响
  违反双亲委派：先自己加载，父加载器找不到才用子
```

---

## 十七、解释执行和编译执行

```
解释执行：
  JVM 逐行解释字节码，翻译成机器码执行
  优点：启动快
  缺点：每次执行都要翻译，慢

编译执行（JIT）：
  JIT = Just In Time Compiler（即时编译器）
  把热点代码编译成本地机器码缓存起来
  下次执行直接用机器码，不需要翻译
  优点：执行快
  缺点：编译需要时间，启动慢

HotSpot 两种编译器：
  C1：轻量级，编译快，优化少，适合启动阶段
  C2：重量级，编译慢，优化多，适合长期运行

分层编译（JDK8默认）：
  先 C1 编译，运行一段时间后 C2 重新编译优化
  兼顾启动速度和运行性能

热点代码判断：
  方法调用次数超过阈值（默认10000次）→ JIT 编译
  循环体执行次数超过阈值 → JIT 编译
```

---

## 总结

| 知识点 | 核心要点 |
|---|---|
| 监控工具 | jstack/jmap/jstat，VisualVM，Arthas（生产推荐） |
| CPU高排查 | top→top -Hp→jstack，找热点线程 |
| 内存高排查 | jmap -dump导出堆，MAT分析泄漏 |
| 频繁MinorGC | 新生代太小，扩大Xmn |
| 频繁FullGC | 老年代不足/内存泄漏/元空间满，分析原因针对处理 |
| 类加载器 | Bootstrap→Extension→Application→自定义 |
| 类生命周期 | 加载→验证→准备→解析→初始化→使用→卸载 |
| 双亲委派 | 先委托父类，父类找不到才自己加载 |
| 破坏双亲委派 | 重写loadClass，JDBC/Tomcat/热部署都破坏了 |
| JIT | 热点代码编译成机器码，C1快启动，C2优化运行 |
