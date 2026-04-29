# Day 24 - MySQL 索引深入、锁机制与事务

> 接续 Day 23,本篇深入 **B+ 树细节、索引下推、各种锁、事务 ACID、隔离级别、MVCC**。这是 MySQL 进阶的硬核内容,也是中高级面试的重灾区。

---

## 📚 目录

- [一、B+ 树补充与对比](#一b-树补充与对比)
- [二、索引核心概念](#二索引核心概念)
- [三、最左前缀与索引下推](#三最左前缀与索引下推)
- [四、如何判断索引是否生效](#四如何判断索引是否生效)
- [五、MySQL 锁分类](#五mysql-锁分类)
- [六、全局锁与表级锁](#六全局锁与表级锁)
- [七、行锁、间隙锁、临键锁](#七行锁间隙锁临键锁)
- [八、意向锁](#八意向锁)
- [九、乐观锁与悲观锁](#九乐观锁与悲观锁)
- [十、死锁排查](#十死锁排查)
- [十一、事务的四大特性 ACID](#十一事务的四大特性-acid)
- [十二、事务隔离级别](#十二事务隔离级别)
- [十三、幻读详解](#十三幻读详解)
- [十四、MVCC 多版本并发控制](#十四mvcc-多版本并发控制)
- [十五、核心知识点总结](#十五核心知识点总结)

---

## 一、B+ 树补充与对比

### 1.1 为什么用 B+ 树而不用 B 树?

虽然 Day 23 已经讲过,这里再聚焦三个核心理由:

**1. B+ 树更"矮",IO 更少**

```
B 树:每个节点存 [键 + 数据 + 子指针]
   → 16KB 页能存的键数少 → 树更高

B+ 树:非叶节点只存 [键 + 子指针],数据全在叶子
   → 16KB 页能存上千个键 → 树更矮
```

**示例对比**:
- 主键 BIGINT(8B) + 指针(6B)= 14B,16KB 页可存约 **1170 个键**
- 加上数据(假设行 1KB),B 树非叶节点只能存约 16 个键

**2. B+ 树范围查询更快**

```
B 树范围查询:中序遍历(回溯),复杂
B+ 树范围查询:叶子节点双向链表,顺序扫描即可
```

```sql
-- 这种范围查询 B+ 树有天生优势
SELECT * FROM orders WHERE create_time BETWEEN '2024-01-01' AND '2024-12-31';
```

**3. B+ 树查询性能稳定**

B 树可能在非叶节点就命中,**查询路径长度不一致**。
B+ 树必须走到叶子节点,**每次查询 IO 次数相同**,SQL 性能可预测。

### 1.2 B+ 树索引和 Hash 索引的区别

| 维度 | B+ 树 | Hash |
|------|-------|------|
| **等值查询** | O(log N) | **O(1) 最快** |
| **范围查询** | ✅ 优秀 | ❌ 不支持 |
| **排序** | ✅(已排序) | ❌ |
| **模糊匹配** | ✅(前缀) | ❌ |
| **联合索引最左前缀** | ✅ | ❌(整体哈希) |
| **覆盖索引** | ✅ | ❌(还需回查) |
| **哈希冲突** | 无 | 有(链表/再散列) |
| **磁盘友好** | ✅(节点对应页) | ⚠️(分散存储) |
| **内存友好** | ✅ | ✅ |

**为什么 InnoDB 选 B+ 树?**

业务查询 95% 涉及:
- 范围查询(`>`、`<`、`BETWEEN`)
- 排序(`ORDER BY`)
- 模糊匹配(`LIKE 'tom%'`)
- 联合索引

这些 Hash **全部不支持**,即使等值查询快到 O(1) 也救不了。

**Hash 索引的应用场景**:
- Memory 引擎默认用 Hash
- InnoDB 的 **自适应哈希索引 (AHI)**:监控热点等值查询,自动建内存哈希,加速访问
- Redis(KV 存储,本质就是哈希)

```sql
-- 查看 AHI 状态
SHOW ENGINE INNODB STATUS;
-- 关注:Hash table size、hash searches/s、non-hash searches/s
```

---

## 二、索引核心概念

### 2.1 聚簇索引和非聚簇索引

**聚簇索引 (Clustered Index)**:**索引和数据存在一起**,叶子节点就是完整的数据行。

**非聚簇索引 (Secondary Index)**:也叫二级索引,**叶子节点存主键值**,需要"回表"。

```
聚簇索引(主键 id):
       [10, 20, 30]              ← 非叶节点
        /    |    \
   [1|2|...10] [11|...20] [21|...30]   ← 叶子 = 完整数据行
   每条记录:[id, name, age, ...所有列]


非聚簇索引(name):
       [Alice, Mike, Tom]         ← 非叶节点
        /     |       \
   [Alice→1][Bob→5]  [Tom→10]    ← 叶子 = 索引列 + 主键值
   每条记录:[name, id]
                  ↓ 回表
            根据 id 去聚簇索引查完整数据
```

**InnoDB**:
- 主键 → 聚簇索引(自动)
- 没主键 → 选第一个唯一非空索引
- 都没有 → InnoDB 自动生成 6 字节的 ROW_ID

**MyISAM**:
- **没有聚簇索引**,所有索引都是非聚簇的
- 索引和数据分开存储(`.MYI` 索引、`.MYD` 数据)
- 叶子节点存数据的物理地址

### 2.2 回表了解吗?

**回表 (Back to Table)**:通过非聚簇索引查到主键值,再用主键去聚簇索引查完整数据的过程。

```sql
-- 表 user(id PK, name idx, age, city)
SELECT * FROM user WHERE name = 'tom';
```

**执行过程**:
```
1. 用 name='tom' 在 idx_name 索引里查,找到主键 id=100
2. 拿 id=100 在聚簇索引中查,得到完整行数据 [100, 'tom', 25, 'BJ']
   ↑ 这一步就是回表
```

**回表的代价**:多一次 B+ 树查询 IO。

**避免回表 = 覆盖索引**(下面讲)。

### 2.3 联合索引了解吗?

**联合索引 (Composite Index)**:多个字段组合成一个索引。

```sql
ALTER TABLE user ADD INDEX idx_name_age_city(name, age, city);
```

**联合索引的 B+ 树结构**:

```
按 (name, age, city) 三元组排序
[Alice|20|BJ] [Alice|25|SH] [Bob|30|GZ] [Tom|18|BJ] [Tom|22|SZ]
   ← 先比 name,name 相同再比 age,age 相同再比 city
```

**联合索引能命中的查询**:

| WHERE 条件 | 是否命中 | 用了哪些列 |
|-----------|---------|-----------|
| `name = 'tom'` | ✅ | name |
| `name = 'tom' AND age = 25` | ✅ | name, age |
| `name = 'tom' AND age = 25 AND city = 'BJ'` | ✅ | 全部 |
| `name = 'tom' AND city = 'BJ'` | 部分 | 只用 name(中间断了) |
| `age = 25` | ❌ | 跳过 name |
| `city = 'BJ'` | ❌ | 跳过 name |
| `age = 25 AND city = 'BJ'` | ❌ | 跳过 name |
| `name LIKE 'tom%'` | ✅ | name |
| `name LIKE '%tom'` | ❌ | 通配符在前 |

**口诀**:**最左前缀,断在范围**。

### 2.4 覆盖索引了解吗?

**覆盖索引 (Covering Index)**:**SELECT 的所有字段都包含在索引里,无需回表**。

```sql
-- 索引:idx_name_age(name, age)

-- ❌ 需要回表(city 不在索引里)
SELECT name, age, city FROM user WHERE name = 'tom';

-- ✅ 覆盖索引(name 和 age 都在索引里)
SELECT name, age FROM user WHERE name = 'tom';

-- ✅ 覆盖索引(主键 id 自带,叶子节点存了)
SELECT id, name, age FROM user WHERE name = 'tom';
```

**EXPLAIN 中的标识**:`Extra: Using index`(看到这个就是覆盖索引)。

**实战优化技巧**:把高频查询的字段都放进联合索引,实现覆盖。

```sql
-- 业务高频查询
SELECT user_id, status, created_at FROM orders WHERE user_id = ?;

-- 建联合索引,实现覆盖
ALTER TABLE orders ADD INDEX idx_user(user_id, status, created_at);
```

> ⚠️ **注意**:覆盖索引虽好,但不要把太多字段塞进索引,**索引膨胀** 会反向拖慢查询和写入。

---

## 三、最左前缀与索引下推

### 3.1 什么是最左前缀原则?

**最左前缀 (Leftmost Prefix)**:**联合索引的查询必须从最左列开始,且不能跳过中间的列**。

```sql
-- 索引:idx(a, b, c)
WHERE a = 1                    ✅ 用 a
WHERE a = 1 AND b = 2          ✅ 用 a, b
WHERE a = 1 AND b = 2 AND c = 3 ✅ 全用
WHERE a = 1 AND c = 3          ⚠️ 只用 a,c 用不到
WHERE b = 2 AND c = 3          ❌ 不用
```

**为什么?** 因为联合索引按 **(a, b, c) 元组顺序排列**,a 没确定的话,b 和 c 整体是无序的。

```
按 (a, b, c) 排序:
[1,1,1] [1,1,2] [1,2,1] [1,2,3] [2,1,1] [2,2,1]

只看 b 列:1, 1, 2, 2, 1, 2  ← 不是有序的!
所以单独按 b 查,索引派不上用场
```

**特殊情况:范围查询会"截断"后续索引**

```sql
-- 索引 idx(a, b, c)
WHERE a = 1 AND b > 2 AND c = 3
                ↑ 范围
-- 只用了 a 和 b,c 用不到索引
```

**MySQL 8.0+ 的优化**:范围查询后的列也可能用到索引(取决于优化器)。

### 3.2 什么是索引下推?

**索引下推 (Index Condition Pushdown, ICP)**:MySQL 5.6 引入的优化,**在存储引擎层就用索引过滤**,减少回表次数。

**没有 ICP 时**:

```sql
-- 索引 idx_name_age(name, age)
SELECT * FROM user WHERE name LIKE 'tom%' AND age = 25;
```

```
1. 引擎层用 name LIKE 'tom%' 找到主键集合(假设 1000 个)
2. 全部回表 1000 次,取出完整数据
3. Server 层用 age = 25 过滤(假设最终 50 行)
   → 浪费了 950 次回表
```

**有 ICP 时**(MySQL 5.6+ 默认开启):

```
1. 引擎层用 name LIKE 'tom%' 找到候选(1000 个)
2. **直接在索引里**用 age = 25 过滤(50 个)
3. 只回表 50 次
   → 节省 95% 的回表
```

**EXPLAIN 标识**:`Extra: Using index condition`

```sql
-- 查看 ICP 是否开启
SHOW VARIABLES LIKE 'optimizer_switch';
-- 找到 index_condition_pushdown=on
```

**ICP 适用条件**:
1. 必须用辅助索引(二级索引)
2. WHERE 条件中部分字段在索引里
3. 表访问方式是 range / ref / eq_ref / ref_or_null

---

## 四、如何判断索引是否生效

### 4.1 用 EXPLAIN

```sql
EXPLAIN SELECT * FROM user WHERE name = 'tom';
```

**关键字段**:
- `key`:**实际使用的索引**(NULL 表示没用)
- `key_len`:用到的索引长度,可判断联合索引用了几列
- `type`:访问类型,至少要 `range`,生产禁止 `ALL`
- `rows`:预估扫描行数,越小越好
- `Extra`:`Using index`(覆盖)、`Using index condition`(ICP)、`Using filesort`(警告)

### 4.2 key_len 计算

```
索引列字节数 = 列定义字节数 + 是否 NULL(1B) + 是否变长(2B) + 字符集
```

**示例**:`VARCHAR(50) NOT NULL utf8mb4`
- 字符集 utf8mb4 每字符 4 字节 → 50 × 4 = 200
- 变长 + 2
- 不为 NULL,不加 1
- **key_len = 202**

**判断联合索引用了几列**:
```sql
-- 索引 idx(name VARCHAR(50), age INT, city VARCHAR(20)) 
-- 全部 NOT NULL,utf8mb4

WHERE name = 'tom';
-- key_len = 202(只用了 name)

WHERE name = 'tom' AND age = 25;
-- key_len = 202 + 4 = 206(用了 name + age)

WHERE name = 'tom' AND age = 25 AND city = 'BJ';
-- key_len = 202 + 4 + 82 = 288(全用)
```

### 4.3 用 SHOW INDEX

```sql
SHOW INDEX FROM user;
```

输出:
```
Table | Non_unique | Key_name | Seq_in_index | Column_name | Cardinality | ...
user  | 0          | PRIMARY  | 1            | id          | 100000      |
user  | 1          | idx_name | 1            | name        | 99000       |
```

**Cardinality**:基数,该列不同值的估计数量。
- 越接近行总数 → 区分度越高 → 索引越有效

### 4.4 强制索引 / 忽略索引

```sql
-- 强制使用某索引
SELECT * FROM user FORCE INDEX(idx_name) WHERE name = 'tom';

-- 忽略某索引
SELECT * FROM user IGNORE INDEX(idx_name) WHERE name = 'tom';

-- 建议使用某索引(不强制)
SELECT * FROM user USE INDEX(idx_name) WHERE name = 'tom';
```

> ⚠️ **慎用 FORCE**:优化器选错索引可能是统计信息过期,先 `ANALYZE TABLE user;` 更新一下,再不行才考虑强制。

### 4.5 实战:索引到底走没走?

```sql
-- 1. EXPLAIN 看 key 字段
EXPLAIN SELECT * FROM user WHERE name = 'tom'\G

-- 2. 看实际执行情况(MySQL 8.0)
EXPLAIN ANALYZE SELECT * FROM user WHERE name = 'tom'\G

-- 3. 开 profiling 看耗时分解
SET profiling = 1;
SELECT * FROM user WHERE name = 'tom';
SHOW PROFILE FOR QUERY 1;

-- 4. 慢日志记录未走索引的 SQL
SET GLOBAL log_queries_not_using_indexes = 'ON';
```

---

## 五、MySQL 锁分类

### 5.1 锁的全景图

```
                       MySQL 锁
                         │
         ┌───────────────┼─────────────────┐
         ▼               ▼                 ▼
      按粒度         按性质            按算法 (InnoDB)
         │               │                 │
   ┌─────┼─────┐    ┌────┼────┐      ┌────┼────┐
   ▼     ▼     ▼    ▼    ▼    ▼      ▼    ▼    ▼
 全局锁 表锁  行锁  共享 排他 意向    Record Gap Next-Key
                    (S)  (X) (IS/IX)  锁    锁    锁

         按思想               按使用场景
         │                       │
   ┌─────┴─────┐          ┌──────┴──────┐
   ▼           ▼          ▼             ▼
 乐观锁     悲观锁      自增锁        元数据锁(MDL)
```

### 5.2 锁的分类速查

| 维度 | 类型 | 说明 |
|------|------|------|
| **粒度** | 全局锁 | 锁整个数据库实例 |
| | 表级锁 | 锁整张表(包括 MDL) |
| | 行级锁 | 锁某行(InnoDB 才有) |
| **性质** | 共享锁 (S) | 读锁,可共享 |
| | 排他锁 (X) | 写锁,独占 |
| | 意向锁 (IS/IX) | 表级,声明意图 |
| **算法** | Record Lock | 记录锁(锁行) |
| | Gap Lock | 间隙锁(锁间隙) |
| | Next-Key Lock | 临键锁(行+间隙) |
| **思想** | 乐观锁 | 不加锁,提交时校验 |
| | 悲观锁 | 提前加锁 |

---

## 六、全局锁与表级锁

### 6.1 全局锁了解吗?

**全局锁**:对整个数据库实例加锁,**所有写操作被阻塞**。

```sql
-- 加全局读锁
FLUSH TABLES WITH READ LOCK;  -- 简称 FTWRL

-- 释放
UNLOCK TABLES;
```

**典型用途:全库逻辑备份**

```bash
mysqldump --single-transaction -uroot -p mydb > backup.sql
# --single-transaction:不加 FTWRL,而是开启可重复读事务备份(InnoDB 推荐)

# 不加 --single-transaction 的话,mysqldump 会自动加 FTWRL,期间整库不可写
```

**FTWRL 的代价**:
- 主库不能写 → 业务停摆
- 从库不能执行 binlog → 主从延迟

**生产建议**:
- InnoDB 表用 `mysqldump --single-transaction`(基于 MVCC,不加锁)
- MyISAM 表只能用 FTWRL(不支持事务)
- 大库优先用 **物理备份**:Percona XtraBackup(在线热备)

### 6.2 表级锁

**MySQL 中的表级锁分两种**:

**1. 表锁**

```sql
-- 加锁
LOCK TABLES user READ;       -- 读锁
LOCK TABLES user WRITE;      -- 写锁
LOCK TABLES user READ, orders WRITE;  -- 多表

-- 释放
UNLOCK TABLES;
```

**InnoDB 中很少用表锁**(行锁更细),主要 MyISAM 用。

**2. 元数据锁 (MDL, Metadata Lock)**

MySQL 5.5 引入,**自动加,无需用户操作**。

- 增删改查 → MDL 读锁
- DDL(改表结构)→ MDL 写锁

**MDL 读读不冲突,读写/写写冲突**。

**经典坑:DDL 阻塞业务**

```
T1: 开启长事务,SELECT 一直没提交     → 持有 MDL 读锁
T2: ALTER TABLE user ADD COLUMN ...   → 等待 MDL 写锁
T3: SELECT * FROM user                → 等待 MDL 读锁(被 T2 排到队尾)
                                      → 业务全部阻塞!
```

**应对**:
- 长事务必查:`SELECT * FROM information_schema.innodb_trx;`
- DDL 设超时:`ALTER TABLE ... ALGORITHM=INPLACE, LOCK=NONE`
- 用工具做 Online DDL:gh-ost、pt-online-schema-change

---

## 七、行锁、间隙锁、临键锁

### 7.1 MySQL 的行锁

**InnoDB 的三种行锁算法**:

| 锁类型 | 锁定对象 | 示例 |
|-------|---------|------|
| **Record Lock(记录锁)** | 索引上的某一行 | `WHERE id = 5`(等值,id 是索引) |
| **Gap Lock(间隙锁)** | 索引上某个开区间 | `WHERE id > 5 AND id < 10` |
| **Next-Key Lock(临键锁)** | 记录 + 前面的间隙 | RR 隔离级别下范围查询的默认锁 |

**示例数据**:`id` 列有索引,值为 [5, 10, 15, 20]

```
间隙:(-∞, 5)、(5, 10)、(10, 15)、(15, 20)、(20, +∞)
```

### 7.2 Record Lock(记录锁)

**锁住具体的索引记录**。

```sql
-- 事务 A
BEGIN;
SELECT * FROM t WHERE id = 10 FOR UPDATE;
-- 锁住 id=10 这一行

-- 事务 B
UPDATE t SET name='x' WHERE id = 10;  -- 阻塞
UPDATE t SET name='x' WHERE id = 15;  -- 不阻塞(锁不到)
```

### 7.3 Gap Lock(间隙锁)

**锁住索引记录之间的"空隙",防止插入**。

```sql
-- 事务 A(REPEATABLE READ 隔离级别)
BEGIN;
SELECT * FROM t WHERE id BETWEEN 10 AND 15 FOR UPDATE;
-- 锁住 (10, 15) 这个间隙

-- 事务 B
INSERT INTO t VALUES(12, 'tom');  -- 阻塞!12 在间隙中
INSERT INTO t VALUES(7, 'jack');  -- 不阻塞
```

**间隙锁的特点**:
- **只在 RR 隔离级别下有**
- **目的:防止幻读**(后面 MVCC 章节会讲)
- **多个事务可以同时持有同一个间隙锁**(不冲突)

### 7.4 临键锁了解吗?

**Next-Key Lock = Record Lock + Gap Lock**(锁住记录 + 它左边的间隙)。

**InnoDB 在 RR 隔离级别下,行锁默认就是临键锁**。

**示例**:索引 [5, 10, 15, 20]

```
临键锁的范围(左开右闭):
(-∞, 5]、(5, 10]、(10, 15]、(15, 20]、(20, +∞)
```

```sql
BEGIN;
SELECT * FROM t WHERE id > 10 AND id <= 15 FOR UPDATE;
-- 锁住 (10, 15] 临键锁

-- 阻止其他事务:
INSERT id=11/12/13/14 → 阻塞(在间隙)
UPDATE id=15 → 阻塞(在记录)
DELETE id=15 → 阻塞
```

**优化**:在某些情况下,临键锁会 **退化**:
- 唯一索引等值查询命中 → 退化为 Record Lock
- 唯一索引等值未命中 → 退化为 Gap Lock

### 7.5 锁住索引 vs 锁住表

**关键认知**:**InnoDB 的行锁是加在索引上的!**

```sql
-- name 没索引
UPDATE user SET age = 25 WHERE name = 'tom';
-- → 全表扫描,锁住所有行(变成了表锁)

-- name 有索引
UPDATE user SET age = 25 WHERE name = 'tom';
-- → 只锁满足条件的行
```

**生产警示**:更新/删除务必走索引,否则锁全表,业务停摆。

---

## 八、意向锁

### 8.1 意向锁是什么?

**意向锁 (Intention Lock)**:**表级锁**,声明"我要在这个表的某些行加锁"。

**两种意向锁**:
- **意向共享锁 (IS)**:表示要在某些行加 S 锁
- **意向排他锁 (IX)**:表示要在某些行加 X 锁

### 8.2 为什么需要意向锁?

**场景**:事务 A 要给整张表加表锁(写锁),需要先确认 **没有其他事务在表的某些行加了行锁**。

**无意向锁时**:必须遍历整张表的所有行,逐行检查 → **太慢!**

**有意向锁时**:
```
事务 B 在某行加 X 锁前,先在表上加 IX 锁(自动)
事务 A 想加表锁,只需检查表上有没有 IX/IS 锁(O(1))
```

**意向锁的兼容性**:

| | IS | IX | S | X |
|---|----|----|---|---|
| **IS** | ✅ | ✅ | ✅ | ❌ |
| **IX** | ✅ | ✅ | ❌ | ❌ |
| **S** | ✅ | ❌ | ✅ | ❌ |
| **X** | ❌ | ❌ | ❌ | ❌ |

**关键**:
- IS / IX 互相兼容(都只是"意向",真正加锁还得抢行锁)
- IX 与 S 冲突(因为 IX 表示要写,S 是读锁)

### 8.3 意向锁是自动加的

```sql
-- 事务 A
BEGIN;
SELECT * FROM t WHERE id = 1 LOCK IN SHARE MODE;
-- 自动:表加 IS 锁,行加 S 锁

UPDATE t SET name='x' WHERE id = 1;
-- 自动:表加 IX 锁,行加 X 锁
```

> 💡 **小结**:意向锁是 InnoDB 的设计巧思,**用空间换时间**,加表锁时不用扫全表检查。

---

## 九、乐观锁与悲观锁

### 9.1 悲观锁 (Pessimistic Lock)

**思想**:**先抢锁,再操作**。假定别人会冲突,所以一开始就锁住。

```sql
-- 悲观锁示例:扣库存
BEGIN;
SELECT stock FROM goods WHERE id = 1 FOR UPDATE;  -- 加 X 锁
-- 程序判断库存够用
UPDATE goods SET stock = stock - 1 WHERE id = 1;
COMMIT;
```

**特点**:
- 简单粗暴,逻辑直接
- **并发低**,锁等待严重
- 适合**高竞争**场景(冲突概率大)

### 9.2 乐观锁 (Optimistic Lock)

**思想**:**先操作,提交时校验**。假定不会冲突,失败再重试。

**实现方式 1:版本号**

```sql
-- 表结构
CREATE TABLE goods (
    id INT,
    stock INT,
    version INT
);

-- 业务代码
old_stock, old_version = SELECT stock, version FROM goods WHERE id = 1;
new_stock = old_stock - 1;

-- 提交时带版本号校验
UPDATE goods SET stock = new_stock, version = version + 1
WHERE id = 1 AND version = old_version;

-- 影响行数 = 1 → 成功
-- 影响行数 = 0 → 别人改过了,需重试
```

**实现方式 2:CAS(基于 stock 本身)**

```sql
UPDATE goods SET stock = stock - 1
WHERE id = 1 AND stock >= 1;

-- 影响行数 = 0 → 库存不够或被改了
```

**特点**:
- **不加数据库锁,并发高**
- 失败需重试
- 适合**低竞争 + 读多写少** 场景

### 9.3 怎么选?

| 场景 | 选择 |
|------|------|
| 秒杀(高并发抢) | 乐观锁 + Redis 预扣 |
| 普通扣款转账 | 悲观锁(简单可靠) |
| 评论点赞 | 乐观锁 |
| 工单分配(避免重复) | 悲观锁 |
| 配置项更新 | 乐观锁(冲突极少) |

**经验**:**冲突频率高用悲观,冲突频率低用乐观**。

---

## 十、死锁排查

### 10.1 什么是死锁?

**死锁 (Deadlock)**:多个事务互相持有对方需要的锁,无限等待。

**经典案例**:

```
事务 A:
  BEGIN;
  UPDATE t SET v=1 WHERE id=1;  -- 锁 id=1
  -- 等 id=2
  UPDATE t SET v=2 WHERE id=2;

事务 B(同时):
  BEGIN;
  UPDATE t SET v=1 WHERE id=2;  -- 锁 id=2
  -- 等 id=1
  UPDATE t SET v=2 WHERE id=1;
```

A 等 B 释放 id=2,B 等 A 释放 id=1 → **死锁**。

### 10.2 InnoDB 的死锁处理

**InnoDB 内置死锁检测**:

```bash
innodb_deadlock_detect = ON      # 默认开启
innodb_lock_wait_timeout = 50    # 锁等待超时(秒)
```

**机制**:
1. 检测到死锁 → 自动 **回滚一个代价较小的事务**(undo log 较少的)
2. 被回滚事务收到 `ERROR 1213 (40001): Deadlock found when trying to get lock`

**生产建议**:
- 应用层捕获死锁错误,**自动重试**
- 高并发系统可关闭检测(`innodb_deadlock_detect=OFF`),靠 timeout 避免死锁

### 10.3 死锁排查 SOP

**1. 看错误日志**

```sql
SHOW ENGINE INNODB STATUS\G
-- 重点看 LATEST DETECTED DEADLOCK 部分
```

输出内容:
```
LATEST DETECTED DEADLOCK
*** (1) TRANSACTION:
TRANSACTION 1234, ACTIVE 5 sec
... (事务 1 信息,持有什么锁,等什么锁)

*** (2) TRANSACTION:
TRANSACTION 1235, ACTIVE 4 sec
... (事务 2 信息)

*** WE ROLL BACK TRANSACTION (2)
```

**2. 查实时锁等待**

```sql
-- MySQL 5.7
SELECT * FROM information_schema.INNODB_TRX;        -- 当前事务
SELECT * FROM information_schema.INNODB_LOCKS;      -- 当前锁
SELECT * FROM information_schema.INNODB_LOCK_WAITS; -- 锁等待

-- MySQL 8.0
SELECT * FROM performance_schema.data_locks;
SELECT * FROM performance_schema.data_lock_waits;
```

**3. 找出阻塞链**

```sql
-- MySQL 8.0,直接看谁阻塞了谁
SELECT
  r.trx_id AS waiting_trx_id,
  r.trx_mysql_thread_id AS waiting_thread,
  r.trx_query AS waiting_query,
  b.trx_id AS blocking_trx_id,
  b.trx_mysql_thread_id AS blocking_thread,
  b.trx_query AS blocking_query
FROM performance_schema.data_lock_waits w
JOIN information_schema.innodb_trx b ON b.trx_id = w.blocking_engine_transaction_id
JOIN information_schema.innodb_trx r ON r.trx_id = w.requesting_engine_transaction_id;
```

### 10.4 避免死锁的实战经验

1. **加锁顺序一致**:多个事务按相同顺序加锁
   ```
   永远先锁 id 小的,再锁 id 大的
   ```
2. **缩短事务**:事务尽可能短,锁持有时间短
3. **降低隔离级别**:RC 隔离级别下没间隙锁,死锁概率降低
4. **使用 SELECT ... FOR UPDATE NOWAIT** / `SKIP LOCKED`(MySQL 8)
5. **避免大事务**:批量操作分批
6. **合理使用索引**:走索引锁少,全表扫描锁多

---

## 十一、事务的四大特性 ACID

### 11.1 ACID 四大特性

| 特性 | 含义 | 类比 |
|------|------|------|
| **A 原子性 (Atomicity)** | 事务是不可分割的最小单位,要么全成功,要么全失败 | 转账 A 扣 100,B 收 100,要么都成功要么都失败 |
| **C 一致性 (Consistency)** | 事务前后,数据库的完整性约束没被破坏 | 转账后总金额不变 |
| **I 隔离性 (Isolation)** | 多个事务并发执行,互不干扰 | A 转账时,B 看到的是中间还是最终状态? |
| **D 持久性 (Durability)** | 事务一旦提交,数据永久保存 | 即使断电,提交的数据也不会丢 |

### 11.2 ACID 靠什么保证的?

**这是最经典的高频面试题。**

| 特性 | 实现机制 |
|------|---------|
| **原子性 A** | **undo log**(回滚日志)|
| **一致性 C** | **由 A、I、D 共同保证**(本质是其他三个的目的) |
| **隔离性 I** | **锁 + MVCC** |
| **持久性 D** | **redo log**(重做日志)|

**详细解析**:

**1. 原子性 ← undo log**
- 事务执行时,每条修改都对应一条 undo log(反向 SQL)
- 回滚时,逐条执行 undo log 即可还原
- 例:`INSERT` 的 undo 是 `DELETE`,`UPDATE` 的 undo 是反向 `UPDATE`

**2. 持久性 ← redo log**
- 事务提交时,redo log 先落盘(WAL)
- 即使数据页没刷盘,崩溃后也能用 redo log 恢复
- 这就是为什么 `innodb_flush_log_at_trx_commit=1` 能保证不丢数据

**3. 隔离性 ← 锁 + MVCC**
- 锁:写写互斥
- MVCC:读不加锁,通过版本链实现"读已提交快照"

**4. 一致性 ← 业务 + 引擎共同保证**
- 引擎层:A、I、D 三大保证
- 业务层:CHECK 约束、外键、UNIQUE 等
- 一致性是 **目的**,其他三个是 **手段**

### 11.3 事务用法

```sql
-- 显式事务
BEGIN;  -- 或 START TRANSACTION;
INSERT INTO ...;
UPDATE ...;
COMMIT;       -- 提交
-- 或 ROLLBACK;   -- 回滚

-- 自动提交
SHOW VARIABLES LIKE 'autocommit';  -- 默认 ON
SET autocommit = 0;  -- 关闭,所有 SQL 都需手动 COMMIT

-- 保存点
BEGIN;
INSERT ...;
SAVEPOINT sp1;
UPDATE ...;
ROLLBACK TO sp1;  -- 只回滚到保存点
COMMIT;
```

---

## 十二、事务隔离级别

### 12.1 四大隔离级别

| 级别 | 脏读 | 不可重复读 | 幻读 | 性能 |
|------|------|-----------|------|------|
| **READ UNCOMMITTED (RU)** | ✅ 会 | ✅ 会 | ✅ 会 | 最快 |
| **READ COMMITTED (RC)** | ❌ | ✅ 会 | ✅ 会 | 快 |
| **REPEATABLE READ (RR)** ⭐ | ❌ | ❌ | InnoDB 解决 | 中 |
| **SERIALIZABLE** | ❌ | ❌ | ❌ | 最慢 |

**MySQL 默认隔离级别:REPEATABLE READ (RR)**
**Oracle / PostgreSQL 默认:READ COMMITTED (RC)**

### 12.2 三个并发问题

**1. 脏读 (Dirty Read)**:读到其他事务 **未提交** 的数据。

```
A: BEGIN; UPDATE balance SET v=200 WHERE id=1;
B: SELECT v FROM balance WHERE id=1;  -- 读到 200(脏)
A: ROLLBACK;  -- 回滚后 v 还是 100
B: 之前读到的 200 是错误的!
```

**2. 不可重复读 (Non-Repeatable Read)**:同一事务中,**两次读同一行,结果不同**(因为别人 UPDATE 了)。

```
A: BEGIN;
A: SELECT v FROM t WHERE id=1;  -- 读到 100
B: UPDATE t SET v=200 WHERE id=1; COMMIT;
A: SELECT v FROM t WHERE id=1;  -- 读到 200(同一事务读到了不同值)
```

**3. 幻读 (Phantom Read)**:同一事务中,**两次范围查询,行数不同**(因为别人 INSERT/DELETE 了)。

```
A: BEGIN;
A: SELECT * FROM t WHERE age > 18;  -- 5 行
B: INSERT INTO t VALUES(...,20); COMMIT;
A: SELECT * FROM t WHERE age > 18;  -- 6 行(凭空多了一行!)
```

### 12.3 事务的隔离级别是如何实现的?

| 隔离级别 | 实现机制 |
|---------|---------|
| **RU** | 不加锁,直接读最新数据 |
| **RC** | **MVCC**:每次 SELECT 生成 ReadView |
| **RR** | **MVCC**:事务开始时生成 ReadView,全程用 |
| **SERIALIZABLE** | **加锁**:所有 SELECT 加 S 锁,串行执行 |

**关键**:RC 和 RR 的区别在 **ReadView 的生成时机**。
- RC:每次 SELECT 都生成新的(看到最新已提交)
- RR:整个事务一个 ReadView(看到事务开始时的快照)

### 12.4 怎么设置隔离级别

```sql
-- 查看当前
SELECT @@transaction_isolation;

-- 全局设置
SET GLOBAL TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 当前会话
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 单事务
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN;
...
COMMIT;
```

---

## 十三、幻读详解

### 13.1 幻读的定义

**幻读**:同一事务中,**两次相同查询,得到不同的行集合**(凭空多/少了行)。

**注意**:
- 不可重复读 = 同一行的值变了
- 幻读 = 行的数量变了

### 13.2 InnoDB 怎么解决幻读?

**两种读取方式,两种解决方案**:

**1. 快照读 (Snapshot Read) → MVCC 解决**

普通 SELECT 是快照读:
```sql
SELECT * FROM t WHERE age > 18;
```

RR 隔离级别下,事务期间所有快照读都基于 **第一次读时的 ReadView**,看到的是同一个版本快照,自然不会有幻读。

**2. 当前读 (Current Read) → 临键锁 (Next-Key Lock) 解决**

当前读会读最新数据并加锁:
```sql
SELECT * FROM t WHERE age > 18 FOR UPDATE;     -- 加 X 锁
SELECT * FROM t WHERE age > 18 LOCK IN SHARE MODE;  -- 加 S 锁
INSERT / UPDATE / DELETE                        -- 隐式当前读
```

InnoDB 在 RR 下,当前读会加 **临键锁**(Record + Gap),**锁住范围内的所有间隙**,其他事务无法 INSERT 进来,从而避免幻读。

```sql
-- 索引 [10, 20, 30]
SELECT * FROM t WHERE id > 15 FOR UPDATE;
-- 加临键锁:(15, 20], (20, 30], (30, +∞)

-- 其他事务:
INSERT id=18 → 阻塞(在间隙)
INSERT id=25 → 阻塞
```

### 13.3 RR 下的幻读"漏洞"

**经典的"幻读未完全解决"案例**:

```
表 t 初始数据:[(1, 'a'), (2, 'b')]

事务 A(RR):
  BEGIN;
  SELECT * FROM t;  -- 快照读,得到 [(1,'a'), (2,'b')]

事务 B:
  BEGIN;
  INSERT INTO t VALUES(3, 'c');
  COMMIT;

事务 A:
  SELECT * FROM t;  -- 快照读,仍然是 [(1,'a'), (2,'b')](MVCC 保护)
  
  UPDATE t SET name='x' WHERE id=3;  -- 当前读!命中 id=3 并更新
  
  SELECT * FROM t;  -- 快照读
  -- 结果:[(1,'a'), (2,'b'), (3,'x')]  ← 幻读!
```

**原因**:UPDATE 是当前读,把 id=3 这行"拉进了"事务 A 的视野。

**结论**:**RR 在快照读层面解决了幻读,但混用快照读和当前读时仍可能出现幻读**。

### 13.4 完全避免幻读

1. 使用 SERIALIZABLE 隔离级别(性能差)
2. 关键场景显式 `SELECT ... FOR UPDATE`(当前读 + 临键锁)
3. 业务层加分布式锁

---

## 十四、MVCC 多版本并发控制

### 14.1 MVCC 是什么?

**MVCC (Multi-Version Concurrency Control)**:多版本并发控制,**让读操作不加锁也能看到一致性视图**。

**核心思想**:每行数据保留多个版本,事务读时根据规则选合适的版本。

**好处**:
- 读不加锁 → 高并发
- 读写不冲突 → 性能高
- 实现 RC 和 RR 隔离级别

### 14.2 MVCC 的三大组件

**1. 隐藏字段**

每行数据除了用户字段,还有 3 个隐藏列:

| 隐藏列 | 大小 | 含义 |
|-------|------|------|
| `DB_TRX_ID` | 6 字节 | 最后修改该行的事务 ID |
| `DB_ROLL_PTR` | 7 字节 | 回滚指针,指向 undo log 中的旧版本 |
| `DB_ROW_ID` | 6 字节 | 行 ID(无主键时作为聚簇索引) |

**2. undo log 版本链**

每次更新数据时,旧版本被写入 undo log,通过 `DB_ROLL_PTR` 链起来:

```
                                  当前数据
                                     │
          [v3, trx_id=300] ←─ DB_ROLL_PTR
                ↑
          [v2, trx_id=200] ←  ← undo log
                ↑
          [v1, trx_id=100] ←
                ↑
         (最早版本,可能是 INSERT 的初始值)
```

**3. ReadView(读视图)**

事务在执行快照读时,生成一个 ReadView,决定能看到哪些版本。

**ReadView 的关键字段**:

| 字段 | 含义 |
|------|------|
| `m_ids` | 当前活跃(未提交)的事务 ID 列表 |
| `min_trx_id` | m_ids 中最小的事务 ID |
| `max_trx_id` | 系统下一个要分配的事务 ID(即"未来" ID) |
| `creator_trx_id` | 创建该 ReadView 的事务 ID |

### 14.3 MVCC 可见性判断

事务读取一行数据时,通过版本链 + ReadView,逐版本判断:

**用 `DB_TRX_ID` 与 ReadView 比较**:

```
1. trx_id == creator_trx_id
   → 自己改的,可见 ✅

2. trx_id < min_trx_id
   → 在视图生成前已提交,可见 ✅

3. trx_id >= max_trx_id
   → 视图生成后才开启的事务,不可见 ❌
   → 沿 undo log 找老版本

4. min_trx_id <= trx_id < max_trx_id
   → 在 m_ids 中?活跃中,不可见 ❌(找老版本)
   → 不在 m_ids 中?已提交,可见 ✅
```

### 14.4 RC 与 RR 的区别在 MVCC 中的体现

**关键差异**:**ReadView 的生成时机**

**RC (Read Committed)**:
- **每次 SELECT 都生成新的 ReadView**
- 能看到查询时刻已提交的所有数据
- 因此会出现"不可重复读"

**RR (Repeatable Read)**:
- **事务的第一次 SELECT 生成 ReadView,整个事务期间复用**
- 看到的始终是事务开始时的快照
- 因此能"重复读"

**示例对比**:

```
表 t:id=1, v=100

事务 A 开启,trx_id=100
事务 B 开启,trx_id=200

A: SELECT v FROM t WHERE id=1;  -- 读到 100

B: BEGIN;
B: UPDATE t SET v=200 WHERE id=1;
B: COMMIT;

A: SELECT v FROM t WHERE id=1;
   ├─ RC:重新生成 ReadView,B 已提交,看到 200
   └─ RR:沿用初始 ReadView,B 在 m_ids 中(开始时未提交),沿 undo 找老版本,看到 100
```

### 14.5 MVCC 与锁的协同

| 操作 | 类型 | 机制 |
|------|------|------|
| `SELECT ...` | 快照读 | MVCC,无锁 |
| `SELECT ... FOR UPDATE` | 当前读 | 加 X 锁 |
| `SELECT ... LOCK IN SHARE MODE` | 当前读 | 加 S 锁 |
| `INSERT / UPDATE / DELETE` | 当前读 + 写 | 加 X 锁 + 写 undo |

**当前读永远读最新版本**(不走 MVCC),并加锁防止其他事务并发改动。

### 14.6 MVCC 的局限

1. **只能解决读问题**,写写冲突还是要靠锁
2. **不能完全解决幻读**(混用快照读和当前读时,见 13.3)
3. **undo log 占空间**:长事务会让 undo 链很长,系统负担重
4. **purge 线程压力**:需要后台清理过期 undo

---

## 十五、核心知识点总结

### 15.1 一图流(全景图)

```
┌─────────────────────────────────────────────────┐
│           MySQL 索引/锁/事务体系                  │
├─────────────────────────────────────────────────┤
│  索引                                             │
│   B+树:多叉、矮、叶子双向链表                       │
│   聚簇/非聚簇,回表,覆盖索引                         │
│   最左前缀,索引下推 ICP                            │
│   EXPLAIN:type / key / Extra                    │
├─────────────────────────────────────────────────┤
│  锁                                              │
│   全局锁 → 表锁(MDL) → 行锁                       │
│   行锁:Record / Gap / Next-Key                  │
│   意向锁(IS/IX):优化加表锁的性能                    │
│   乐观锁(版本号) vs 悲观锁(FOR UPDATE)              │
├─────────────────────────────────────────────────┤
│  事务                                             │
│   ACID:                                         │
│     A ← undo log                                │
│     C ← 业务 + AID                                │
│     I ← 锁 + MVCC                                │
│     D ← redo log                                │
├─────────────────────────────────────────────────┤
│  隔离级别                                          │
│   RU → RC → RR(默认) → SERIALIZABLE              │
│   幻读:RR 用 MVCC + 临键锁解决                     │
├─────────────────────────────────────────────────┤
│  MVCC                                            │
│   隐藏字段 + undo 版本链 + ReadView                 │
│   RC:每次 SELECT 生成 ReadView                    │
│   RR:事务级 ReadView,只生成一次                    │
└─────────────────────────────────────────────────┘
```

### 15.2 高频面试题(自查清单)

**索引深入**
- [ ] 为什么用 B+ 树不用 B 树/Hash/红黑树?
- [ ] 聚簇索引和非聚簇索引的区别?
- [ ] 什么是回表?如何避免?
- [ ] 什么是覆盖索引?
- [ ] 联合索引最左前缀是什么?
- [ ] 什么是索引下推 ICP?
- [ ] 怎么判断索引有没有用?

**锁**
- [ ] MySQL 有几种锁?
- [ ] 全局锁、表锁、行锁分别什么场景?
- [ ] 行锁的三种算法?
- [ ] 临键锁是什么?如何防止幻读?
- [ ] 意向锁的作用?为什么需要?
- [ ] 乐观锁和悲观锁怎么选?
- [ ] 死锁怎么排查?如何避免?

**事务**
- [ ] ACID 分别是什么?各自靠什么保证?
- [ ] 四大隔离级别?MySQL 默认是哪个?
- [ ] 脏读、不可重复读、幻读的区别?
- [ ] 隔离级别是怎么实现的?
- [ ] InnoDB 怎么解决幻读?

**MVCC**
- [ ] MVCC 是什么?为什么要有它?
- [ ] MVCC 的三大组件?
- [ ] ReadView 是什么?可见性怎么判断?
- [ ] RC 和 RR 的 MVCC 实现差异?
- [ ] MVCC 能完全解决幻读吗?

### 15.3 实战经验汇总

**索引优化**
- 高频联合查询 → 联合索引 + 覆盖索引
- 避免函数/计算/隐式转换
- 字符串前缀索引节省空间
- ANALYZE TABLE 定期更新统计

**锁的实战**
- UPDATE/DELETE 必须走索引,否则锁全表
- 长事务是 MDL 阻塞元凶,定期排查
- 死锁加捕获和重试逻辑
- 高并发场景考虑 RC 隔离级别(死锁少)

**事务边界**
- 事务尽量短,锁持有时间短
- 不在事务中做 RPC、邮件等长耗时操作
- 大事务拆成多个小事务
- 注意 Spring 的 `@Transactional` 默认传播行为

**MVCC 注意**
- 长事务会让 undo log 链堆积,占用大量空间
- 排查长事务:`SELECT * FROM information_schema.innodb_trx WHERE TIME_TO_SEC(TIMEDIFF(NOW(), trx_started)) > 60;`
- 设置 `innodb_undo_log_truncate=ON`,自动清理

### 15.4 推荐资源

📚 **必读书籍**
- 《MySQL 是怎样运行的》—— 小孩子 4919 ⭐⭐⭐⭐⭐
- 《MySQL 技术内幕:InnoDB 存储引擎》—— 姜承尧
- 《高性能 MySQL》(第 4 版)
- 极客时间《MySQL 实战 45 讲》—— 林晓斌

🌐 **进阶**
- 《数据库系统概念》—— Silberschatz
- 《数据库内核月报》—— 阿里数据库内核团队
- InnoDB 源码导读

---

## 📌 Day 24 总结

| 维度 | 收获 |
|------|------|
| **索引深入** | 理解回表/覆盖/最左前缀/ICP,知道索引怎么用最高效 |
| **锁机制** | 掌握 8 种锁的分类,知道行锁的三种算法及兼容性 |
| **事务原理** | ACID 不再是背诵,能讲清每个特性的底层实现 |
| **隔离级别** | 理解 4 个级别的差异,知道 MySQL 默认 RR 的原因 |
| **幻读** | 区分快照读和当前读,理解 RR 解决幻读的两条路径 |
| **MVCC** | 弄懂版本链 + ReadView,能解释 RC/RR 的本质区别 |

> 索引、锁、事务是 MySQL 的"三座大山",也是后端面试的硬核高地。**讲透 MVCC 与隔离级别**,基本上 MySQL 这关就稳了。

**Day 24 ✅ Done — 持续更新中,欢迎 Star ⭐**
