# Day 23 - MySQL 存储引擎、日志、SQL 优化与索引

> 接续 Day 22 的 MySQL 基础与架构,本篇深入 **段区页行存储结构、存储引擎对比、Buffer Pool、日志系统、慢 SQL 优化、索引原理与 B+Tree**,是 MySQL 底层原理与性能调优的核心篇章。

---

## 📚 目录

- [一、MySQL 存储结构:段区页行](#一mysql-存储结构段区页行)
- [二、存储引擎](#二存储引擎)
- [三、InnoDB 的 Buffer Pool](#三innodb-的-buffer-pool)
- [四、MySQL 日志系统](#四mysql-日志系统)
- [五、binlog 与 redo log](#五binlog-与-redo-log)
- [六、两阶段提交](#六两阶段提交)
- [七、redo log 写入过程](#七redo-log-写入过程)
- [八、慢 SQL 与优化](#八慢-sql-与优化)
- [九、explain 执行计划](#九explain-执行计划)
- [十、索引原理与分类](#十索引原理与分类)
- [十一、索引创建与失效](#十一索引创建与失效)
- [十二、B+Tree 深入](#十二btree-深入)
- [十三、核心知识点总结](#十三核心知识点总结)

---

## 一、MySQL 存储结构:段区页行

### 1.1 InnoDB 存储结构层级

InnoDB 把数据存储划分为五个层级,从大到小:**表空间 → 段 → 区 → 页 → 行**。

```
┌──────────────────────────────────────────┐
│   Tablespace (表空间) — .ibd 文件          │
│  ┌────────────────────────────────────┐  │
│  │   Segment (段) — 数据段/索引段/回滚段 │  │
│  │  ┌──────────────────────────────┐  │  │
│  │  │   Extent (区) — 1 MB,64 个页  │  │  │
│  │  │  ┌────────────────────────┐  │  │  │
│  │  │  │   Page (页) — 16 KB     │  │  │  │
│  │  │  │  ┌──────────────────┐  │  │  │  │
│  │  │  │  │   Row (行) — 记录 │  │  │  │  │
│  │  │  │  └──────────────────┘  │  │  │  │
│  │  │  └────────────────────────┘  │  │  │
│  │  └──────────────────────────────┘  │  │
│  └────────────────────────────────────┘  │
└──────────────────────────────────────────┘
```

### 1.2 各层详解

**1. 表空间 (Tablespace)**
- InnoDB 数据存储的最高层逻辑结构
- **系统表空间**:`ibdata1`,共享给所有库表
- **独立表空间**:每张表一个 `.ibd` 文件(`innodb_file_per_table=ON`,推荐)
- **Undo 表空间**:存放回滚日志(MySQL 8 独立)
- **临时表空间**:存临时表

**2. 段 (Segment)**
- 由若干个区组成,逻辑概念
- **数据段**:存放 B+Tree 叶子节点(实际数据)
- **索引段**:存放 B+Tree 非叶子节点
- **回滚段**:存 undo log

**3. 区 (Extent)**
- 物理上连续的 64 个页 = 1 MB
- 为什么有区?**保证数据在磁盘上连续**,提升顺序 IO 性能
- 表空间小的时候用 **碎片区**(零散分配),数据多了用 **完整区**(整区分配)

**4. 页 (Page)**
- **InnoDB 磁盘 IO 的最小单位,默认 16 KB**
- 通过 `innodb_page_size` 配置,可设为 4K/8K/16K/32K/64K
- 类型:数据页、索引页、Undo 页、系统页等
- **关键**:即使只读 1 行,也会把整页 16KB 加载到 Buffer Pool

**5. 行 (Row)**
- 数据按行存储在页内
- 每页约能存 2 行到上千行(视行大小)
- **行格式**:Compact、Redundant、Dynamic(默认)、Compressed

### 1.3 行格式 (Row Format)

**Compact 格式结构**:

```
┌────────────────┬──────────────┬──────────┬────────────┐
│ 变长字段长度列表 │  NULL 值列表  │ 记录头信息 │ 实际数据列  │
└────────────────┴──────────────┴──────────┴────────────┘
        ↑               ↑              ↑           ↑
    倒序存储         位图标识NULL    5字节        含3隐藏列
```

**3 个隐藏列**:
- `DB_ROW_ID`:6 字节,行 ID(无主键时作为聚簇索引)
- `DB_TRX_ID`:6 字节,事务 ID(MVCC 用)
- `DB_ROLL_PTR`:7 字节,回滚指针(指向 undo log)

**Dynamic vs Compact**:
- 大字段(VARCHAR/TEXT/BLOB)在 Compact 下会存 768 字节前缀 + 溢出页指针
- Dynamic 下 **完全溢出**,行内只存 20 字节指针,**节省页空间**

> 💡 **页满了会怎样?** 触发 **页分裂**,把数据拆到新页,代价高,所以推荐 **自增主键**(顺序写入)。

---

## 二、存储引擎

### 2.1 MySQL 常见存储引擎

| 引擎 | 特点 | 锁 | 事务 | 适用场景 |
|------|------|----|----|---------|
| **InnoDB** ⭐ | 默认,支持事务、行锁、外键、MVCC | 行锁 | ✅ | 99% 场景 |
| **MyISAM** | 不支持事务,表锁,有 count(*) 优化 | 表锁 | ❌ | 只读统计(已过时) |
| **Memory** | 数据在内存,极快但断电丢失 | 表锁 | ❌ | 临时表、缓存 |
| **Archive** | 高压缩比,只支持 INSERT/SELECT | 行锁 | ❌ | 日志归档 |
| **CSV** | 数据存为 CSV 文本文件 | 表锁 | ❌ | 数据交换 |
| **Blackhole** | 黑洞,写入丢失,但写 binlog | - | - | 复制中转 |
| **NDB** | MySQL Cluster 集群引擎 | 行锁 | ✅ | 分布式高可用 |
| **TokuDB** | 高压缩、写优化 | 行锁 | ✅ | 海量写入(Percona) |

### 2.2 存储引擎应该怎么选择?

**默认选 InnoDB**,理由:
1. 事务支持(ACID)
2. 行级锁,并发好
3. 崩溃恢复(crash-safe)
4. 外键支持
5. MVCC,读不加锁

**特殊场景才考虑其他**:
- 临时计算结果 → Memory
- 历史日志归档 → Archive
- 主从复制中转 → Blackhole

```sql
-- 查看支持的引擎
SHOW ENGINES;

-- 查看某表用的引擎
SHOW CREATE TABLE user;

-- 修改引擎
ALTER TABLE user ENGINE = InnoDB;
```

### 2.3 InnoDB 和 MyISAM 主要区别

| 维度 | InnoDB | MyISAM |
|------|--------|--------|
| **事务** | ✅ 支持 ACID | ❌ |
| **锁粒度** | 行锁 + 表锁 | 表锁 |
| **外键** | ✅ | ❌ |
| **MVCC** | ✅ | ❌ |
| **崩溃恢复** | ✅(redo log) | ❌(易损坏) |
| **聚簇索引** | ✅(主键 = 数据) | ❌(索引和数据分开) |
| **count(\*)** | 慢(实时计算) | 快(维护行数) |
| **全文索引** | ✅(5.6+) | ✅ |
| **存储文件** | `.ibd`(数据+索引) | `.MYD`(数据)+ `.MYI`(索引) |
| **缓存** | 缓存数据+索引 | 只缓存索引 |
| **自增锁** | 表级 AUTO-INC 锁(可调) | 表锁 |

**关键差异:聚簇索引 vs 非聚簇索引**

```
InnoDB(聚簇):
  主键索引 ─→ 叶子节点直接存数据行
  二级索引 ─→ 叶子节点存主键值,需"回表"

MyISAM(非聚簇):
  所有索引 ─→ 叶子节点存数据行的物理地址
  没有"回表"的概念
```

---

## 三、InnoDB 的 Buffer Pool

### 3.1 什么是 Buffer Pool?

**Buffer Pool 是 InnoDB 在内存中的数据缓冲区**,用于缓存磁盘页,减少磁盘 IO。它是 InnoDB 性能的核心。

```bash
# 查看 Buffer Pool 大小
SHOW VARIABLES LIKE 'innodb_buffer_pool_size';

# 推荐配置:物理内存的 50%~80%
innodb_buffer_pool_size = 8G
```

**生产配置经验**:
- 8G 内存机器 → Buffer Pool 4~5G
- 32G 内存机器 → Buffer Pool 24G
- 专用数据库服务器 → 70~80% 内存

### 3.2 Buffer Pool 结构

Buffer Pool 由若干 **Page** 组成,每个 Page 16KB,通过三大链表管理:

```
┌─────────────────────────────────────────────┐
│              Buffer Pool                     │
│  ┌──────┬──────┬──────┬──────┬──────┐       │
│  │ Page │ Page │ Page │ Page │ Page │  ...  │
│  └──────┴──────┴──────┴──────┴──────┘       │
└─────────────────────────────────────────────┘
        │              │              │
        ▼              ▼              ▼
   Free List       LRU List        Flush List
   (空闲页)      (使用中页)        (脏页待刷盘)
```

**Free List**:空闲页链表,新读入的页从这里取空间。

**LRU List**:管理"使用中的页",InnoDB 的 **改良版 LRU** 是关键。

**Flush List**:被修改过但还没刷盘的"脏页"链表,后台线程定期刷盘。

### 3.3 改良版 LRU(分代 LRU)

**为什么不用普通 LRU?**
- 全表扫描会把热点数据淘汰(扫一次,大量页进 LRU 头)
- 预读机制可能加载用不到的页

**InnoDB 解决方案**:把 LRU 链表分成两段

```
LRU 链表(总长度的 5/8 是 young,3/8 是 old)
┌─────────────────────────┬──────────────────┐
│      Young (热数据)      │    Old (冷数据)   │
│  最近高频访问            │  新加载的页先放这   │
└─────────────────────────┴──────────────────┘
            ↑                       ↑
         热点头部                 冷数据头部
```

**核心机制**:
1. **新页先进 old 区头部**(冷数据)
2. **页进入 old 后超过 1 秒再被访问**,才晋升到 young 头部(防止预读和全表扫描污染)
3. young 区内部访问也不会立刻移到头部(每 1/4 距离才移动一次,减少链表操作)

参数:
```bash
innodb_old_blocks_pct = 37          # old 占比 37%
innodb_old_blocks_time = 1000       # 进入 old 1 秒后才能升 young (毫秒)
```

### 3.4 脏页刷盘

**脏页**:Buffer Pool 中被修改过、还没写到磁盘的页。

**刷盘时机**:
1. **redo log 写满** → 必须刷脏页腾空间
2. **Buffer Pool 内存不足** → LRU 尾部的脏页要被淘汰,先刷盘
3. **MySQL 空闲时** → 后台线程主动刷
4. **MySQL 正常关闭** → 全部刷完

**刷脏页过慢会导致 SQL 突然变慢**(俗称"抖动"),因为前台线程被迫等待刷盘。

```bash
innodb_io_capacity = 2000           # IO 能力,SSD 建议 2000+,机械盘 200
innodb_max_dirty_pages_pct = 75     # 脏页比例上限
```

---

## 四、MySQL 日志系统

### 4.1 MySQL 日志文件有哪些?

MySQL 一共有 **8 种核心日志**:

| 日志 | 所属层 | 作用 | 默认开启 |
|------|-------|------|---------|
| **redo log** | InnoDB | 崩溃恢复(WAL) | ✅ |
| **undo log** | InnoDB | 事务回滚、MVCC | ✅ |
| **binlog** | Server | 主从复制、数据恢复 | ❌(需开启) |
| **错误日志 error log** | Server | 启动/运行错误 | ✅ |
| **慢查询日志 slow log** | Server | 记录慢 SQL | ❌ |
| **通用查询日志 general log** | Server | 记录所有 SQL(性能差,慎用) | ❌ |
| **中继日志 relay log** | Server | 从库复制中转 | 从库自动 |
| **DDL log** | Server | DDL 元数据操作 | ✅ |

### 4.2 三大核心日志对比

| 日志 | 层级 | 内容 | 写入方式 | 作用 |
|------|------|------|---------|------|
| **redo log** | InnoDB | 物理日志(数据页改动) | 循环写 | 崩溃恢复 |
| **undo log** | InnoDB | 逻辑日志(反向 SQL) | 段管理 | 回滚 + MVCC |
| **binlog** | Server | 逻辑日志(SQL/行) | 追加写 | 复制 + 备份 |

### 4.3 慢查询日志配置

```sql
-- 查看慢日志状态
SHOW VARIABLES LIKE 'slow_query_log%';

-- 开启慢日志
SET GLOBAL slow_query_log = 'ON';

-- 设置阈值(秒),默认 10s,生产建议 1s 或更低
SET GLOBAL long_query_time = 1;

-- 未走索引的 SQL 也记录
SET GLOBAL log_queries_not_using_indexes = 'ON';
```

**分析慢日志神器**:`mysqldumpslow` 和 `pt-query-digest`(更强)

```bash
# 按平均时间排序,显示前 10 条
mysqldumpslow -s at -t 10 /var/log/mysql/slow.log

# Percona 工具,更详细
pt-query-digest /var/log/mysql/slow.log
```

---

## 五、binlog 与 redo log

### 5.1 详细对比

| 维度 | redo log | binlog |
|------|----------|--------|
| **层级** | InnoDB 引擎层 | Server 层 |
| **是否所有引擎都有** | ❌ 仅 InnoDB | ✅ 所有引擎 |
| **格式** | 物理日志:第 X 页第 Y 行改成什么 | 逻辑日志:SQL/行变更 |
| **写入方式** | **循环写**,固定大小,旧的被覆盖 | **追加写**,写满换文件不覆盖 |
| **作用** | 崩溃恢复 | 主从复制、备份恢复 |
| **写入时机** | 事务执行过程中持续写 | 事务提交时写 |
| **大小** | 固定(默认 4 个文件 × 1G) | 不限,按文件名递增 |

### 5.2 binlog 的三种格式

**1. STATEMENT(SQL 语句)**
```
INSERT INTO user(name) VALUES('tom');
```
- ✅ 占用空间小
- ❌ 主从可能不一致(`UUID()`、`NOW()` 等函数)

**2. ROW(行变更)**
```
[user 表第 1 行,变更前 (1, 'jack'),变更后 (1, 'tom')]
```
- ✅ 主从一致性好
- ❌ 占用空间大(`UPDATE 100w 行` 会写 100 万条)

**3. MIXED(混合)**
- 默认用 STATEMENT,涉及不确定函数自动切 ROW
- MySQL 5.7 之前默认,**MySQL 8 默认 ROW**

```sql
-- 查看格式
SHOW VARIABLES LIKE 'binlog_format';

-- 设置格式
SET GLOBAL binlog_format = 'ROW';
```

### 5.3 redo log 的循环写

```
┌──────────────────────────────────────┐
│   redo log 文件组(默认 4 个 × 1GB)    │
│  ┌──────┬──────┬──────┬──────┐       │
│  │file 0│file 1│file 2│file 3│       │
│  └──────┴──────┴──────┴──────┘       │
│         ↑                ↑            │
│      checkpoint       write_pos       │
│      (擦除位置)        (写入位置)       │
└──────────────────────────────────────┘

write_pos 追上 checkpoint → 必须刷脏页推进 checkpoint → 期间所有更新阻塞
```

### 5.4 为什么需要两个日志?

历史原因 + 各司其职:
- **binlog 先有**(MySQL 自带),原本只支持 STATEMENT,不能保证崩溃恢复
- **redo log 是 InnoDB 后来加的**,InnoDB 是后来引入的引擎

如果只有 binlog:
- binlog 不能用于崩溃恢复(无法判断哪些事务已提交)
- 即使重做,效率低(逻辑日志要重新执行 SQL)

如果只有 redo log:
- 不能做主从复制(其他引擎没 redo)
- 不能做时点恢复

---

## 六、两阶段提交

### 6.1 为什么要两阶段提交?

**核心目的:保证 redo log 和 binlog 的一致性**,防止主从数据不一致或数据丢失。

### 6.2 流程图

```
                客户端 commit
                     ↓
    ┌────────────────────────────────┐
    │  阶段 1:写 redo log (prepare) │
    │  状态 = prepare, 含 XID         │
    └────────────────┬───────────────┘
                     ↓
    ┌────────────────────────────────┐
    │      阶段 2a:写 binlog          │
    │  事件序列含 XID                 │
    └────────────────┬───────────────┘
                     ↓
    ┌────────────────────────────────┐
    │  阶段 2b:写 redo log (commit) │
    │  状态 prepare → commit          │
    └────────────────┬───────────────┘
                     ↓
                 提交完成
```

### 6.3 崩溃恢复的判断逻辑

重启时,扫描 redo log:

```
对每条 redo log 记录:
  if (redo log 状态 = commit)
      → 数据已落盘,跳过
  else if (redo log 状态 = prepare)
      → 通过 XID 找到对应的 binlog
      if (binlog 完整且包含该 XID)
          → 提交事务(写 commit 标记)
      else
          → 回滚事务
```

**这样做的两个保证**:
1. binlog 写入成功 → 事务一定能恢复(主从一致)
2. binlog 写入失败 → 事务一定回滚(无残留)

### 6.4 反例分析

**假设没有两阶段提交,先写 redo log 后写 binlog**:
```
T1: redo log 写完
T2: 崩溃!binlog 还没写
T3: 重启后主库恢复:有这条数据
T4: 从库基于 binlog 复制:没有这条数据
   → 主从不一致 ❌
```

**先写 binlog 后写 redo log**:
```
T1: binlog 写完
T2: 崩溃!redo log 还没写
T3: 重启后主库恢复:无法恢复(没 redo)
T4: 从库基于 binlog 复制:有这条数据
   → 主从不一致 ❌
```

**两阶段提交完美解决**:无论崩溃在哪一步,都能通过 prepare/commit 状态 + binlog 完整性判断,保证一致。

---

## 七、redo log 写入过程

### 7.1 redo log 的整体流程

```
事务执行
    ↓
修改 Buffer Pool 中的数据页(产生脏页)
    ↓
生成 redo log record,写入 redo log buffer (内存)
    ↓
按策略刷到 OS Page Cache (操作系统缓存)
    ↓
fsync 落盘到 redo log 文件
```

### 7.2 redo log 的三层缓存

```
┌─────────────────────────────────────────┐
│    redo log buffer (MySQL 内存)          │
│  innodb_log_buffer_size,默认 16MB        │
└────────────────┬────────────────────────┘
                 ↓ write()
┌─────────────────────────────────────────┐
│    OS Page Cache (操作系统缓存)           │
└────────────────┬────────────────────────┘
                 ↓ fsync()
┌─────────────────────────────────────────┐
│    redo log 文件(磁盘)                  │
└─────────────────────────────────────────┘
```

### 7.3 三种刷盘策略

通过参数 `innodb_flush_log_at_trx_commit` 控制:

| 值 | 行为 | 安全 | 性能 | 说明 |
|----|------|------|------|------|
| **0** | 每秒由后台线程刷 | ❌ 最差 | ✅ 最好 | 崩溃丢 1 秒数据 |
| **1**(默认) | 每次提交都 fsync | ✅ 最好 | ❌ 最差 | 真正 ACID,不丢数据 |
| **2** | 每次提交写 OS,每秒 fsync | 🟡 中等 | 🟡 中等 | MySQL 崩溃不丢,OS 崩溃丢 1 秒 |

**生产建议**:
- 金融、支付场景 → **1**(数据安全第一)
- 内部系统、日志型 → **2**(平衡)
- 临时数据、可恢复 → **0**(性能第一)

类似的,binlog 也有 `sync_binlog`:
- 0:由 OS 控制
- 1:每次提交都刷盘(默认,推荐)
- N:积累 N 次再刷

**双 1 配置**:`innodb_flush_log_at_trx_commit=1` + `sync_binlog=1` = 最高数据安全。

### 7.4 redo log 的特性

**1. 顺序写**:redo log 是连续追加,顺序 IO 比随机 IO 快几个数量级。

**2. WAL (Write-Ahead Logging)**:**先写日志,再改磁盘数据**。这样即使数据没刷盘,通过日志也能恢复。

**3. Crash-Safe**:即使突然断电,只要 redo log 落盘了,数据就不丢。

**4. 有限大小**:循环写,redo log 满了必须强制刷脏页。

### 7.5 LSN(日志序列号)

**LSN (Log Sequence Number)**:redo log 的全局递增编号,标识每条日志的位置。

每个数据页头部都有一个 `FIL_PAGE_LSN`,表示该页最后一次被修改对应的 LSN。

**崩溃恢复用 LSN 判断**:
- 数据页 LSN >= redo log LSN → 数据已刷盘,跳过
- 数据页 LSN < redo log LSN → 应用 redo log

---

## 八、慢 SQL 与优化

### 8.1 什么是慢 SQL?

**定义**:执行时间超过阈值的 SQL,默认阈值 10 秒,**生产建议 1 秒甚至 200ms**。

**慢 SQL 的常见原因**:
1. 没走索引(全表扫描)
2. 索引失效(隐式转换、函数操作)
3. 数据量大(扫描行数多)
4. JOIN 太多张表
5. 子查询嵌套过深
6. ORDER BY / GROUP BY 没走索引(filesort)
7. 深分页(`LIMIT 1000000, 10`)
8. 锁等待(被其他事务堵住)
9. Buffer Pool 命中率低(频繁磁盘 IO)
10. 网络问题(返回数据量过大)

### 8.2 优化 SQL 的方法

**🔍 第一步:定位**
```sql
-- 1. 开启慢日志
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;

-- 2. 看 EXPLAIN
EXPLAIN SELECT * FROM orders WHERE user_id = 100;

-- 3. 看 profile
SET profiling = 1;
SELECT ...;
SHOW PROFILES;
SHOW PROFILE FOR QUERY 1;
```

**🛠 第二步:优化策略**

**1. 索引优化**(最常见)
```sql
-- ❌ 全表扫描
SELECT * FROM user WHERE name = 'tom';

-- ✅ 加索引
ALTER TABLE user ADD INDEX idx_name(name);

-- ✅ 复合索引(覆盖更多场景)
ALTER TABLE user ADD INDEX idx_name_age(name, age);
```

**2. SQL 改写**
```sql
-- ❌ 函数导致索引失效
SELECT * FROM user WHERE DATE(create_time) = '2024-01-01';

-- ✅ 改写
SELECT * FROM user WHERE create_time >= '2024-01-01' AND create_time < '2024-01-02';

-- ❌ NOT IN 不走索引
SELECT * FROM user WHERE id NOT IN (1,2,3);

-- ✅ LEFT JOIN 替代
SELECT u.* FROM user u LEFT JOIN exclude e ON u.id = e.id WHERE e.id IS NULL;
```

**3. 深分页优化**
```sql
-- ❌ 慢
SELECT * FROM orders ORDER BY id LIMIT 1000000, 10;

-- ✅ 子查询走索引
SELECT * FROM orders WHERE id >= (
    SELECT id FROM orders ORDER BY id LIMIT 1000000, 1
) LIMIT 10;

-- ✅ 游标分页(最优)
SELECT * FROM orders WHERE id > 1000000 ORDER BY id LIMIT 10;
```

**4. 字段优化**
```sql
-- ❌ SELECT *,数据传输大
SELECT * FROM order_detail;

-- ✅ 只取需要的列
SELECT id, order_no, amount FROM order_detail;
```

**5. JOIN 优化**
- 用小表驱动大表
- JOIN 字段必须有索引
- JOIN 表数量控制在 3 张以内
- 复杂查询拆成多次简单查询

**6. 分库分表**
- 单表超过 1000 万行考虑分表
- 垂直拆分:把大字段分到副表
- 水平拆分:按 user_id 哈希分表

**7. 读写分离 / 缓存**
- 主写从读
- 热点数据进 Redis
- 减少数据库压力

### 8.3 优化心法

**优化 SQL 的金字塔**:

```
        ┌────────────┐
        │  架构优化   │  ← 分库分表、读写分离、缓存
        ├────────────┤
        │  SQL 改写   │  ← 改写不合理 SQL
        ├────────────┤
        │  索引优化   │  ← 加索引、覆盖索引
        ├────────────┤
        │ 表结构设计  │  ← 字段类型、范式
        ├────────────┤
        │  参数调优   │  ← Buffer Pool、innodb_io_capacity
        └────────────┘
```

**永远从下往上优化:小代价优先,架构变更最后。**

---

## 九、explain 执行计划

### 9.1 explain 平常有用过吗?

**EXPLAIN 是分析 SQL 性能的核心工具**,不执行 SQL,只看执行计划。

```sql
EXPLAIN SELECT * FROM user WHERE name = 'tom';
```

输出示例:
```
+----+-------------+-------+------+---------------+----------+---------+-------+------+-----------+
| id | select_type | table | type | possible_keys | key      | key_len | ref   | rows | Extra     |
+----+-------------+-------+------+---------------+----------+---------+-------+------+-----------+
| 1  | SIMPLE      | user  | ref  | idx_name      | idx_name | 102     | const | 1    | Using ... |
+----+-------------+-------+------+---------------+----------+---------+-------+------+-----------+
```

### 9.2 字段含义详解

| 字段 | 含义 | 重点关注 |
|------|------|---------|
| **id** | 查询序号,越大越先执行 | 多表 JOIN 看顺序 |
| **select_type** | 查询类型 | SIMPLE/PRIMARY/SUBQUERY... |
| **table** | 涉及的表 | - |
| **partitions** | 命中的分区 | 分区表 |
| **type** | **访问类型(最重要)** | 性能由好到差 |
| **possible_keys** | 可能用到的索引 | - |
| **key** | **实际用的索引** | NULL 说明没走索引 |
| **key_len** | 用到的索引长度(字节) | 联合索引判断用了几列 |
| **ref** | 索引比较的值来源 | const/字段名/func |
| **rows** | **预估扫描行数** | 越小越好 |
| **filtered** | 过滤百分比 | 越大越好 |
| **Extra** | **额外信息(关键)** | Using index/Using where/... |

### 9.3 type 字段(性能等级)

**从好到坏**:

```
system > const > eq_ref > ref > range > index > ALL
   ↑                                              ↑
最好                                            最差
```

| type | 说明 | 示例 |
|------|------|------|
| **system** | 表只有一行(系统表) | 极少见 |
| **const** | 主键/唯一索引等值查询 | `WHERE id = 1` |
| **eq_ref** | JOIN 时用主键/唯一索引匹配 | `JOIN ON a.id = b.user_id` |
| **ref** | 普通索引等值查询 | `WHERE name = 'tom'` |
| **range** | 索引范围扫描 | `WHERE id > 100` |
| **index** | 全索引扫描(只扫索引树) | 索引覆盖 + 无 where |
| **ALL** | **全表扫描(必须优化)** | 没索引或索引失效 |

> 💡 **目标**:至少要达到 **range** 级别,生产代码不允许 **ALL**(除非小表)。

### 9.4 Extra 字段(关键提示)

| Extra 值 | 含义 | 好坏 |
|----------|------|------|
| **Using index** | **覆盖索引**,只查索引就够 | ⭐ 最好 |
| **Using where** | 用 WHERE 过滤(但已经走索引了) | ✅ 正常 |
| **Using index condition** | 索引下推 ICP(MySQL 5.6+) | ✅ 好 |
| **Using temporary** | 用了临时表(GROUP BY/DISTINCT) | ⚠️ 优化 |
| **Using filesort** | 文件排序(无法用索引排序) | ⚠️ 优化 |
| **Using join buffer** | JOIN 没走索引,用了缓冲区 | ❌ 必须优化 |
| **Impossible WHERE** | WHERE 永远不成立 | - |
| **Select tables optimized away** | 直接从索引取得结果 | ⭐ 最好 |

### 9.5 实战分析

```sql
-- 例 1:type=ALL,rows 很大
EXPLAIN SELECT * FROM user WHERE name = 'tom';
-- 优化:加索引 ALTER TABLE user ADD INDEX idx_name(name);

-- 例 2:Using filesort
EXPLAIN SELECT * FROM user ORDER BY age;
-- 优化:age 加索引,且 ORDER BY 与索引顺序一致

-- 例 3:Using temporary
EXPLAIN SELECT name, COUNT(*) FROM user GROUP BY name;
-- 优化:GROUP BY 字段加索引

-- 例 4:索引覆盖
EXPLAIN SELECT id, name FROM user WHERE name = 'tom';
-- 在 idx_name(name) 索引下,Extra=Using index(因为查询字段都在索引里)
```

**MySQL 8 增强**:
```sql
-- 树形格式,更直观
EXPLAIN FORMAT=TREE SELECT ...;

-- 实际执行,带真实统计
EXPLAIN ANALYZE SELECT ...;
```

---

## 十、索引原理与分类

### 10.1 索引为什么能提高查询效率?

**核心**:把 **O(N) 全表扫描** 变成 **O(log N) 树形查找**。

**类比**:
- 没索引 = 在没有目录的字典里找一个字 → 一页一页翻
- 有索引 = 通过部首/拼音目录快速定位 → 几次跳转就找到

**索引本质**:**用空间换时间**,通过额外的数据结构(B+Tree)加速查找。

**代价**:
- 占用磁盘空间
- 减慢写入(INSERT/UPDATE/DELETE 要维护索引)
- 索引太多反而拖慢查询(优化器选择困难)

### 10.2 索引分类

**按数据结构分**:
- B+Tree 索引(默认,最常用)
- Hash 索引(Memory 引擎、自适应哈希)
- 全文索引 (Full-text)
- 空间索引 (R-Tree,GIS 数据)

**按物理存储分**:
- **聚簇索引** (Clustered):叶子节点存完整数据(InnoDB 的主键)
- **非聚簇索引** (Secondary):叶子节点存主键值,需回表

**按字段特性分**:
- **主键索引** (Primary Key):非空 + 唯一,InnoDB 自动建为聚簇索引
- **唯一索引** (Unique):值唯一,可为 NULL
- **普通索引** (Normal):无任何约束
- **联合索引** (Composite):多列组合
- **前缀索引** (Prefix):字符串前几个字符
- **覆盖索引**:查询字段全部在索引里,无需回表
- **降序索引** (MySQL 8+):索引按降序存
- **隐藏索引** (MySQL 8+):优化器忽略,用于测试

### 10.3 聚簇索引 vs 非聚簇索引

```
聚簇索引(主键):                  非聚簇索引(普通):
┌────────────────┐                 ┌────────────────┐
│  非叶节点       │                 │  非叶节点       │
│  [10, 20, 30]   │                 │  [a, m, x]      │
└──────┬─────────┘                 └──────┬─────────┘
       ↓                                  ↓
┌──────────────────┐               ┌──────────────────┐
│   叶子节点        │               │   叶子节点        │
│ id=10 [完整行]    │               │ name=a, id=10    │
│ id=20 [完整行]    │               │ name=b, id=20    │
└──────────────────┘               └──────────────────┘
                                          ↓ 回表
                                    主键索引获取完整行
```

**回表 (Back to Table)**:
```sql
-- name 是普通索引
SELECT id, name FROM user WHERE name = 'tom';  -- 不回表(覆盖索引)
SELECT * FROM user WHERE name = 'tom';         -- 回表(需要其他字段)
```

**优化技巧:覆盖索引**
```sql
-- 假设常用查询是 SELECT id, name, age FROM user WHERE name = ?
-- 建联合索引避免回表
ALTER TABLE user ADD INDEX idx_name_age(name, age);
```

---

## 十一、索引创建与失效

### 11.1 创建索引有哪些注意点?

**✅ 适合建索引的场景**:
1. WHERE / ORDER BY / GROUP BY / JOIN 频繁用到的字段
2. 区分度高的字段(如手机号、邮箱)
3. 长字符串考虑前缀索引
4. 联合索引代替单列索引(覆盖更多场景)

**❌ 不适合建索引的场景**:
1. 数据量很小的表(<1000 行)
2. 频繁更新的字段(维护成本高)
3. 区分度低的字段(性别、状态)
4. 大字段(TEXT/BLOB,只能前缀索引)

**📝 建索引的最佳实践**:

```sql
-- 1. 联合索引遵循"最左前缀原则",高区分度字段在前
ALTER TABLE user ADD INDEX idx_name_age_city(name, age, city);
-- 能命中:WHERE name=?  / WHERE name=? AND age=? / WHERE name=? AND age=? AND city=?
-- 不命中:WHERE age=?  / WHERE city=?

-- 2. 字符串用前缀索引(节省空间)
ALTER TABLE user ADD INDEX idx_email(email(10));
-- 取前 10 个字符建索引

-- 3. NOT NULL,加默认值
ALTER TABLE user MODIFY COLUMN name VARCHAR(50) NOT NULL DEFAULT '';

-- 4. 自增主键
CREATE TABLE user (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    ...
);
```

### 11.2 索引哪些情况下会失效?

**经典 11 种失效场景**:

**1. 不满足最左前缀**
```sql
-- 索引 idx(a, b, c)
WHERE b = 1;              -- ❌ 跳过 a
WHERE a = 1 AND c = 1;    -- 部分:只用了 a
```

**2. 索引列上做计算/函数**
```sql
WHERE id + 1 = 10;        -- ❌
WHERE YEAR(create_time) = 2024;  -- ❌
WHERE SUBSTRING(name, 1, 3) = 'tom';  -- ❌
```

**3. 隐式类型转换**
```sql
-- phone 是 varchar
WHERE phone = 13800138000;  -- ❌ 字符串转数字
WHERE phone = '13800138000';  -- ✅
```

**4. LIKE 以通配符开头**
```sql
WHERE name LIKE '%tom';     -- ❌
WHERE name LIKE '%tom%';    -- ❌
WHERE name LIKE 'tom%';     -- ✅
```

**5. 用 OR 连接非索引字段**
```sql
-- a 有索引,b 没索引
WHERE a = 1 OR b = 1;     -- ❌
-- 优化:b 也加索引,或者改 UNION
```

**6. NOT / != / <> / NOT IN**
```sql
WHERE id != 1;            -- ❌
WHERE id NOT IN (1, 2, 3);  -- ❌ (个别情况)
```

**7. IS NULL / IS NOT NULL**
- 视字段定义和数据分布,可能失效
- 字段尽量 NOT NULL DEFAULT ''

**8. 数据分布不均**
```sql
-- 表 1000 行,name='tom' 占 800 行
WHERE name = 'tom';  -- 优化器认为还不如全表扫描
```

**9. 范围条件后的列失效**
```sql
-- 索引 idx(a, b, c)
WHERE a = 1 AND b > 10 AND c = 5;
-- a 用了索引,b 用了范围,c 不会走索引
```

**10. ORDER BY 与索引顺序不一致**
```sql
-- 索引 idx(a, b)
ORDER BY a ASC, b DESC;   -- ❌(MySQL 8 之前)
```

**11. 优化器决定不走索引**
- 全表扫描比索引快(小表)
- 强制索引:`SELECT * FROM t FORCE INDEX(idx) WHERE ...`

### 11.3 索引不适合哪些场景?

1. **数据量小**:全表扫一下还更快
2. **频繁写入**:每次写都要维护索引
3. **频繁更新的字段**:索引重排成本高
4. **低区分度字段**:性别、是否删除(boolean)

### 11.4 索引是不是建的越多越好?

❌ **不是!**

**索引的代价**:
1. **占用磁盘**:每个索引是一棵 B+Tree
2. **拖慢写入**:每次 INSERT/UPDATE/DELETE 都要维护所有索引
3. **优化器开销**:索引多了,选择执行计划耗时
4. **可能反而变慢**:优化器选错索引

**经验法则**:
- 单表索引数量 **不超过 5-7 个**
- 联合索引代替多个单列索引
- 定期检查 **未使用的索引**(`sys.schema_unused_indexes`)
- 不要重复索引(如 `idx(a)` 和 `idx(a, b)` 重复)

```sql
-- 查看未使用的索引(MySQL 5.7+)
SELECT * FROM sys.schema_unused_indexes;

-- 查看重复索引
SELECT * FROM sys.schema_redundant_indexes;
```

---

## 十二、B+Tree 深入

### 12.1 为什么 InnoDB 要使用 B+Tree?

**数据库索引的核心需求**:
1. **快速查找**:O(log N)
2. **范围查询**:支持 `BETWEEN`、`>`、`<`
3. **磁盘 IO 少**:树要矮,一次 IO 加载更多数据
4. **稳定**:无论查什么,IO 次数相近

**候选数据结构对比**:

| 数据结构 | 等值查询 | 范围查询 | 磁盘友好 | 用于? |
|---------|---------|---------|---------|-------|
| **Hash** | O(1) | ❌ | ✅ | Memory 引擎 |
| **二叉搜索树** | O(log N) | ✅ | ❌(易退化) | - |
| **AVL/红黑树** | O(log N) | ✅ | ❌(树太高) | - |
| **B 树** | O(log N) | ✅(中等) | 🟡 | MongoDB |
| **B+ 树** | O(log N) | ✅(优秀) | ✅(树矮) | **MySQL** |
| **LSM 树** | O(log N) | ✅ | ✅(写优化) | RocksDB |

### 12.2 B 树 vs B+ 树

```
B 树:
                  [30, 60]
                /    |    \
          [10,20]  [40,50] [70,80,90]
          (每个节点都存数据)

B+ 树:
                  [30, 60]
                /    |    \
          [10,20]  [40,50] [70,80]
              ↓        ↓       ↓
        [10|20|25→40|50|55→60|70|80] (叶子节点链表)
        (只有叶子节点存数据,叶子之间双向链表)
```

**B+ 树的核心优势**:

| 特性 | B 树 | B+ 树 |
|------|------|-------|
| **数据存储** | 所有节点都存数据 | **只有叶子节点存数据** |
| **非叶节点** | 占用空间大 | 只存索引键,**能存更多** → 树更矮 |
| **范围查询** | 中序遍历,慢 | **叶子节点链表,顺序扫描快** |
| **查询稳定性** | 命中非叶节点提前返回 | 必须到叶子节点,**IO 次数稳定** |
| **磁盘 IO** | 多 | 少(树矮) |

### 12.3 一棵 B+ 树能存多少条数据?

**经典面试题:三层 B+ 树能存多少行?**

**前提假设**:
- 一个数据页 16 KB
- 索引行(主键 BIGINT 8B + 指针 6B)= 14B,实际按 16B 算
- 数据行平均 1 KB(很多业务行就这量级)

**计算**:

```
非叶节点:每个页能放多少索引行?
   16 KB / 16 B = 1024 个索引项

叶子节点:每个页能放多少数据行?
   16 KB / 1 KB = 16 行数据

三层 B+ 树容量:
   第一层(根)   = 1 个页 → 1024 个指针
   第二层       = 1024 个页 → 1024 × 1024 个指针
   第三层(叶子)= 1024 × 1024 个页 → 每页 16 行

   总数据 = 1024 × 1024 × 16 ≈ 1677 万行
```

**结论**:**3 层 B+ 树就能存 2000 万行数据,只需 3 次磁盘 IO!**

这就是为什么单表 2000 万行内查询性能仍然很好,**3 次 IO 加上 Buffer Pool 缓存,绝大部分根节点和非叶节点常驻内存**,实际可能只需 1 次 IO。

### 12.4 索引为什么用 B+ 树不用普通二叉树/红黑树?

**1. 树高问题**

```
红黑树存 100 万行:
  log₂(1,000,000) ≈ 20 层 → 20 次磁盘 IO

B+ 树存 2000 万行:
  3 层 → 3 次磁盘 IO
```

红黑树是 **二叉**,每个节点只有 2 个分支,**树高 = log₂N**。
B+ 树是 **多叉**(每个节点 1000+ 分支),**树高 = log₁₀₀₀N**,差距悬殊。

**2. 磁盘 IO 模式**

数据库读磁盘以 **页(16KB)** 为单位,而二叉树每个节点只有一个键,**16KB 加载进来只用了一个键,极度浪费**。

B+ 树一个节点就是一个 **页**,16KB 内塞下 1000 多个键,IO 利用率高。

**3. 范围查询**

红黑树范围查询要中序遍历,效率低。
B+ 树叶子节点是 **双向链表**,范围查询直接顺着链表走,极快。

**4. 节点磁盘友好性**

B+ 树非叶节点不存数据,**节点更小**,**单页能容纳更多键**,树更矮,IO 更少。

### 12.5 索引为什么用 B+ 树不用 Hash?

| 场景 | Hash | B+ Tree |
|------|------|---------|
| **等值查询** | O(1),最快 | O(log N) |
| **范围查询** | ❌ 不支持 | ✅ 优秀(链表) |
| **排序** | ❌ | ✅(已排序) |
| **模糊查询(LIKE)** | ❌ | ✅(前缀) |
| **多列联合索引** | ❌ | ✅(最左前缀) |
| **索引覆盖** | ❌ | ✅ |
| **哈希冲突** | 有 | 无 |

**结论**:Hash 只适合 **简单的等值查询**,而业务中 95% 的查询都涉及范围、排序、模糊匹配,所以 B+ Tree 完胜。

> 💡 **InnoDB 的妥协**:有个 **自适应哈希索引 (Adaptive Hash Index, AHI)**,InnoDB 监控到某些索引页被频繁等值访问时,自动建一个内存哈希索引来加速。这是 B+ Tree + Hash 的混合优化。

### 12.6 B+ 树查询过程示例

```sql
-- 查询 SELECT * FROM user WHERE id = 25;
```

```
                  [50, 100]            ← 根节点(常驻内存)
                  /    |    \
            [20,40] [70,90] [120]       ← 非叶节点(常驻内存)
            /  |  \
       [10|15|20→25|30|40]              ← 叶子节点(磁盘读)
                ↑
              找到 id=25
```

**步骤**:
1. 内存读根节点:25 < 50,走左
2. 内存读非叶:25 在 [20, 40] 之间,走中间
3. 磁盘读叶子页:链表遍历,找到 25 → 返回数据

**总 IO 次数:1 次**(前 2 步在 Buffer Pool 里)。

### 12.7 范围查询过程

```sql
-- SELECT * FROM user WHERE id BETWEEN 25 AND 80;
```

```
            找到 25 所在叶子页
                  ↓
            顺着叶子节点的双向链表向后遍历
                  ↓
            读到 80 停止
```

**关键**:叶子节点的双向链表是 B+ 树的杀手锏。

---

## 十三、核心知识点总结

### 13.1 一图流(全景图)

```
┌────────────────────────────────────────────┐
│         MySQL 深度知识体系                   │
├────────────────────────────────────────────┤
│  存储结构                                    │
│   表空间 → 段 → 区 → 页(16K) → 行            │
├────────────────────────────────────────────┤
│  存储引擎                                    │
│   InnoDB(主流) / MyISAM(过时) / Memory     │
│   Buffer Pool + 改良 LRU(young/old)        │
├────────────────────────────────────────────┤
│  日志                                        │
│   redo log:循环写、物理日志、crash-safe     │
│   binlog:追加写、逻辑日志、复制备份         │
│   undo log:回滚 + MVCC                      │
│   两阶段提交:prepare → binlog → commit     │
├────────────────────────────────────────────┤
│  SQL 优化                                   │
│   慢日志 → EXPLAIN → 索引/SQL 改写          │
│   type:ALL→index→range→ref→eq_ref→const    │
├────────────────────────────────────────────┤
│  索引                                        │
│   B+ Tree:多叉、矮、叶子链表                 │
│   聚簇 vs 非聚簇,回表,覆盖索引              │
│   最左前缀,11 种失效场景                    │
│   3 层存 2000 万行                           │
└────────────────────────────────────────────┘
```

### 13.2 高频面试题(自查清单)

**存储引擎**
- [ ] InnoDB 和 MyISAM 区别?为什么 InnoDB 成为默认?
- [ ] 一张表对应几个文件?
- [ ] 段、区、页、行分别是什么?
- [ ] 行格式有几种?Compact 行结构?

**Buffer Pool**
- [ ] Buffer Pool 是什么?为什么需要?
- [ ] 为什么改良 LRU?如何防止全表扫描污染?
- [ ] 什么是脏页?何时刷盘?
- [ ] 抖动是怎么来的?

**日志**
- [ ] MySQL 有几种日志?各自作用?
- [ ] redo log 和 binlog 的区别?
- [ ] 为什么要两阶段提交?
- [ ] redo log 怎么循环写?
- [ ] innodb_flush_log_at_trx_commit 三个值的区别?
- [ ] WAL 是什么?

**SQL 优化**
- [ ] 什么是慢 SQL?如何定位?
- [ ] 慢 SQL 优化的方法?
- [ ] EXPLAIN 各字段含义?type 等级?
- [ ] Extra 中的 Using filesort/Using temporary 怎么优化?
- [ ] 深分页怎么优化?

**索引**
- [ ] 索引为什么快?代价是什么?
- [ ] 索引分类?聚簇 vs 非聚簇?
- [ ] 什么是回表?如何避免?
- [ ] 什么是覆盖索引?
- [ ] 联合索引最左前缀原则?
- [ ] 索引失效的场景?(11 种以上)
- [ ] 索引是不是越多越好?
- [ ] 为什么 InnoDB 用 B+ Tree 而不是 Hash/红黑树?
- [ ] B 树和 B+ 树的区别?
- [ ] 一棵 B+ 树能存多少行?三层为什么是 2000 万?
- [ ] 主键为什么推荐自增?

### 13.3 实战经验汇总

**索引设计原则**
1. 主键自增 BIGINT,避免页分裂
2. 高频查询字段建联合索引,**注意顺序(高区分度在前)**
3. 字符串用前缀索引节省空间
4. 单表索引 ≤ 5-7 个
5. 字段 NOT NULL DEFAULT
6. 用覆盖索引避免回表

**SQL 编写禁忌**
1. ❌ `SELECT *`
2. ❌ 索引列做计算/函数
3. ❌ `WHERE phone = 13800138000`(隐式转换)
4. ❌ `LIKE '%xxx%'`
5. ❌ `LIMIT 1000000, 10`(深分页)
6. ❌ 大事务(锁持有时间长)
7. ❌ JOIN 超过 3 张表

**生产配置参考**
```ini
[mysqld]
# 必备
default-storage-engine = InnoDB
innodb_file_per_table = 1
character-set-server = utf8mb4
collation-server = utf8mb4_0900_ai_ci

# Buffer Pool(物理内存的 50-75%)
innodb_buffer_pool_size = 8G
innodb_buffer_pool_instances = 8

# redo log
innodb_log_file_size = 1G
innodb_log_files_in_group = 4
innodb_flush_log_at_trx_commit = 1

# binlog
log_bin = mysql-bin
binlog_format = ROW
sync_binlog = 1
expire_logs_days = 7

# 慢日志
slow_query_log = 1
long_query_time = 1
log_queries_not_using_indexes = 1

# 连接
max_connections = 1000
wait_timeout = 600
```

### 13.4 推荐资源

📚 **深度阅读**
- 《MySQL 是怎样运行的》—— 小孩子 4919 ⭐⭐⭐⭐⭐
- 《MySQL 技术内幕:InnoDB 存储引擎》—— 姜承尧
- 《高性能 MySQL》(第 4 版)
- 极客时间《MySQL 实战 45 讲》

🛠 **实战工具**
- 慢日志分析:pt-query-digest
- 在线 DDL:gh-ost、pt-online-schema-change
- 监控:Prometheus + mysqld_exporter
- 压测:sysbench

---

## 📌 Day 23 总结

| 维度 | 收获 |
|------|------|
| **存储结构** | 理解段区页行的层级,知道为何页大小是 16KB |
| **引擎对比** | 深入掌握 InnoDB 与 MyISAM 的本质区别 |
| **Buffer Pool** | 学会改良 LRU 的设计哲学,避免污染 |
| **日志体系** | 三大日志各司其职,两阶段提交保证一致性 |
| **SQL 优化** | 慢日志 → EXPLAIN → 改写/索引,有完整方法论 |
| **B+ Tree** | 从根本上理解为什么 MySQL 选择它,3 层存 2000 万行 |

> 索引是 MySQL 的灵魂,B+ Tree 是索引的基石。理解了这一篇,面试时关于"InnoDB 存储原理"和"为什么用 B+ Tree"这两道送分题,你能聊上 30 分钟。

**Day 23 ✅ Done — 持续更新中,欢迎 Star ⭐**
