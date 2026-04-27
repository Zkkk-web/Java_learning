# Day 22 - MySQL 基础与架构

> 系统梳理 MySQL 基础知识与底层架构,涵盖 SQL 基础、数据类型、常用函数、查询执行流程等核心内容。本篇偏向面试与日常开发,可作为长期参考手册。

---

## 📚 目录

- [一、MySQL 基础概念](#一mysql-基础概念)
- [二、表连接与关联查询](#二表连接与关联查询)
- [三、数据库设计规范](#三数据库设计规范)
- [四、数据类型选型](#四数据类型选型)
- [五、SQL 关键字与语法](#五sql-关键字与语法)
- [六、SQL 执行顺序与解析](#六sql-执行顺序与解析)
- [七、MySQL 常用命令与函数](#七mysql-常用命令与函数)
- [八、MySQL 基础架构](#八mysql-基础架构)
- [九、查询语句执行流程](#九查询语句执行流程)
- [十、更新语句执行流程](#十更新语句执行流程)
- [十一、核心知识点总结](#十一核心知识点总结)

---

## 一、MySQL 基础概念

### 1.1 什么是 MySQL?

**MySQL 是一款开源的关系型数据库管理系统 (RDBMS)**,由瑞典 MySQL AB 公司开发,2008 年被 Sun 收购,2010 年随 Sun 被 Oracle 收购。

**核心特点**:
- **关系型**:数据以二维表形式存储,表与表之间通过外键关联
- **开源免费**:社区版完全免费,企业版收费
- **跨平台**:支持 Windows、Linux、macOS 等
- **支持多种存储引擎**:InnoDB(默认)、MyISAM、Memory、Archive 等
- **支持事务、ACID**(InnoDB)、行级锁、MVCC
- **生态成熟**:文档完善,社区活跃,工具链丰富

**MySQL 与其他数据库的对比**:

| 数据库 | 类型 | 特点 | 适用场景 |
|--------|------|------|---------|
| **MySQL** | 关系型 | 开源、轻量、生态好 | Web 应用、中小型项目 |
| **Oracle** | 关系型 | 企业级、功能强、收费贵 | 大型企业、金融 |
| **PostgreSQL** | 关系型 | 功能丰富、SQL 标准支持好 | 复杂查询、地理信息 |
| **SQL Server** | 关系型 | 微软生态、Windows 友好 | .NET 项目 |
| **MongoDB** | 文档型 NoSQL | 灵活 schema、横向扩展 | 海量非结构化数据 |
| **Redis** | KV 型 NoSQL | 内存级、超高性能 | 缓存、计数器 |

**MySQL 版本演进关键节点**:
- 5.5:InnoDB 成为默认存储引擎
- 5.6:在线 DDL、GTID 复制
- 5.7:JSON 类型、原生 JSON 支持、性能大幅提升
- **8.0**:CTE、窗口函数、隐藏索引、降序索引、原子 DDL(主流推荐版本)

---

## 二、表连接与关联查询

### 2.1 两张表怎么进行连接?

通过 **JOIN** 关键字 + **ON 条件** 把两张表关联起来。本质是基于某个字段(通常是主键/外键)做笛卡尔积过滤。

```sql
SELECT u.name, o.order_no
FROM user u
JOIN orders o ON u.id = o.user_id;
```

### 2.2 内连接、左连接、右连接的区别

| 连接类型 | 关键字 | 含义 | 结果 |
|---------|--------|------|------|
| **内连接** | `INNER JOIN` | 只返回两表都匹配的行 | 交集 |
| **左连接** | `LEFT JOIN` | 返回左表所有行,右表无匹配填 NULL | 左表全集 |
| **右连接** | `RIGHT JOIN` | 返回右表所有行,左表无匹配填 NULL | 右表全集 |
| **全连接** | `FULL JOIN` | 两表所有行(MySQL 不支持,需 UNION 模拟) | 并集 |
| **交叉连接** | `CROSS JOIN` | 笛卡尔积(无 ON 条件) | M × N |

**图示**:

```
表 A          表 B
[1,2,3]       [2,3,4]

INNER JOIN  →  [2,3]
LEFT  JOIN  →  [1,2,3]
RIGHT JOIN  →  [2,3,4]
FULL  JOIN  →  [1,2,3,4]
```

**实战示例**:

```sql
-- 内连接:有订单的用户
SELECT u.name, o.amount FROM user u INNER JOIN orders o ON u.id = o.user_id;

-- 左连接:所有用户(包括没下过单的)
SELECT u.name, o.amount FROM user u LEFT JOIN orders o ON u.id = o.user_id;

-- 找出从未下单的用户(常见面试题)
SELECT u.name FROM user u LEFT JOIN orders o ON u.id = o.user_id
WHERE o.id IS NULL;
```

---

## 三、数据库设计规范

### 3.1 数据库三大范式

**第一范式 (1NF)**:**字段不可再分**(原子性)
```
❌ 错误:user 表的 address 字段存 "北京市朝阳区xx路"
✅ 正确:拆成 province / city / district / detail
```

**第二范式 (2NF)**:**消除部分依赖**(在 1NF 基础上,非主键字段必须完全依赖整个主键)
```
❌ 错误:订单表 (order_id, product_id, product_name, qty)
       product_name 只依赖 product_id,不依赖整个联合主键
✅ 正确:拆成订单表和商品表
```

**第三范式 (3NF)**:**消除传递依赖**(非主键字段不能依赖其他非主键字段)
```
❌ 错误:user 表 (user_id, dept_id, dept_name)
       dept_name 依赖 dept_id,而 dept_id 又依赖 user_id → 传递依赖
✅ 正确:拆成 user 表和 dept 表
```

**反范式化**:实际开发中为了提高查询性能,会主动违反范式,**冗余字段** 减少 JOIN。例如订单表冗余 user_name、product_name,避免每次查询都关联用户表/商品表。

> 💡 **设计哲学**:范式追求严谨,反范式追求性能,**没有银弹,看业务权衡**。

---

## 四、数据类型选型

### 4.1 varchar 与 char 的区别

| 维度 | char | varchar |
|------|------|---------|
| **长度** | 固定长度 | 可变长度 |
| **存储** | 不足补空格,多了截断 | 实际长度 + 1~2B 长度前缀 |
| **最大长度** | 255 字符 | 65535 字节(行长度限制) |
| **性能** | 略快(无需计算长度) | 略慢但省空间 |
| **使用场景** | 长度固定字段:身份证、手机号、MD5 | 长度变化大字段:用户名、地址 |

**经验**:
- 长度 ≤ 5 用 char,差异不大但简单
- 长度变化大用 varchar,省空间
- 频繁更新用 char(varchar 更新可能导致行迁移)

### 4.2 blob 和 text 的区别

| 维度 | blob | text |
|------|------|------|
| **存储内容** | 二进制(图片、音频) | 字符(文章、JSON) |
| **字符集** | 无字符集 | 有字符集和排序规则 |
| **比较** | 按字节比较 | 按字符集排序规则比较 |
| **类型** | TINY/BLOB/MEDIUM/LONG | TINY/TEXT/MEDIUM/LONG |

**最大长度对比**:
- TINYTEXT/TINYBLOB:255 字节
- TEXT/BLOB:64 KB
- MEDIUMTEXT/MEDIUMBLOB:16 MB
- LONGTEXT/LONGBLOB:4 GB

> ⚠️ **生产建议**:大字段尽量用对象存储(OSS/S3),数据库只存 URL,避免拖慢主库。

### 4.3 DATETIME 和 TIMESTAMP 的区别

| 维度 | DATETIME | TIMESTAMP |
|------|----------|-----------|
| **存储空间** | 8 字节 | 4 字节 |
| **范围** | 1000-01-01 ~ 9999-12-31 | 1970-01-01 ~ 2038-01-19 |
| **时区** | 与时区无关 | **跟随时区自动转换** |
| **默认值** | 不能用函数(MySQL 5.6+ 可) | 支持 CURRENT_TIMESTAMP |
| **NULL** | 默认 NULL | 默认非 NULL |

**经典 2038 问题**:TIMESTAMP 用 32 位无符号整数存储 Unix 时间戳,2038-01-19 03:14:07 UTC 后会溢出。

**经验**:
- 跨时区应用用 TIMESTAMP(自动转换)
- 历史数据/未来日期用 DATETIME(范围大)
- MySQL 8 推荐 **DATETIME(3)** 带毫秒精度

### 4.4 记录货币用什么类型比较好?

✅ **DECIMAL**(精确小数,定点数)
- `DECIMAL(M, D)`:M 总位数,D 小数位数
- 例:`DECIMAL(10, 2)` 表示总 10 位,小数 2 位,最大 99999999.99
- **不存在精度丢失问题**

❌ 不要用 FLOAT/DOUBLE
- 浮点数有精度问题:`0.1 + 0.2 ≠ 0.3`
- 涉及金额会出现 1 分钱的误差,引发对账事故

**替代方案**:用 BIGINT 存"分"(扩大 100 倍),程序层处理。性能更好,但可读性差。

### 4.5 怎么存储 emoji?

**问题根源**:emoji 是 4 字节 UTF-8 字符,而 MySQL 默认的 `utf8` 字符集只支持 1-3 字节(其实是个"假 utf8")。

**解决方案**:用 **utf8mb4** 字符集(mb4 = most bytes 4)。

```sql
-- 建库
CREATE DATABASE mydb CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- 建表
CREATE TABLE post (
    content VARCHAR(500) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci
) DEFAULT CHARSET=utf8mb4;

-- 修改已有表
ALTER TABLE post CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- 配置文件 my.cnf
[mysqld]
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
[client]
default-character-set = utf8mb4
```

**排序规则选择**:
- `utf8mb4_general_ci`:速度快,精度低
- `utf8mb4_unicode_ci`:符合 Unicode 标准,推荐
- `utf8mb4_0900_ai_ci`:MySQL 8 默认,最新 Unicode 9.0

> 💡 **MySQL 8 默认就是 utf8mb4**,新项目无需特殊配置。

---

## 五、SQL 关键字与语法

### 5.1 drop、delete 与 truncate 的区别

| 维度 | DROP | DELETE | TRUNCATE |
|------|------|--------|----------|
| **类型** | DDL | DML | DDL |
| **作用** | 删除整个表(结构+数据) | 删除部分/全部数据 | 清空表数据(保留结构) |
| **WHERE** | 不支持 | 支持 | 不支持 |
| **触发器** | 不触发 | 触发 | 不触发 |
| **事务** | 不能回滚 | **可以回滚** | 不能回滚 |
| **速度** | 快 | 慢(逐行删) | 极快(直接释放数据页) |
| **自增 ID** | 删除 | 保留当前值 | 重置为 1 |
| **空间** | 立即释放 | 不释放(标记删除) | 立即释放 |

**使用场景**:
- 不要这张表了 → DROP
- 删几行数据,可能反悔 → DELETE
- 清空表数据,要快 → TRUNCATE

### 5.2 UNION 与 UNION ALL 的区别

| 维度 | UNION | UNION ALL |
|------|-------|-----------|
| **去重** | **自动去重**(类似 DISTINCT) | 不去重 |
| **排序** | 默认排序 | 不排序 |
| **性能** | 慢(去重+排序) | 快 |

```sql
-- UNION:1,2,3,4 (去重)
SELECT id FROM a -- [1,2,3]
UNION
SELECT id FROM b -- [2,3,4]

-- UNION ALL:1,2,3,2,3,4 (保留重复)
SELECT id FROM a UNION ALL SELECT id FROM b
```

> 💡 **经验**:能用 UNION ALL 就别用 UNION,性能差一个数量级。

### 5.3 count(1)、count(*) 与 count(列名) 的区别

| 写法 | 含义 | NULL 处理 | 性能 |
|------|------|----------|------|
| **count(*)** | 统计行数 | 包含 NULL 行 | InnoDB 优化,推荐 |
| **count(1)** | 统计行数 | 包含 NULL 行 | 与 count(*) 几乎一样 |
| **count(列名)** | 统计该列非 NULL 的行数 | **不包含 NULL** | 最慢(要判断 NULL) |
| **count(distinct 列)** | 该列去重后的非 NULL 行数 | 不包含 NULL | 慢 |

**MySQL 官方建议**:
- 使用 **count(\*)**,优化器有专门优化
- count(1) 和 count(*) 在 InnoDB 下性能基本一致
- count(主键) 比 count(普通列) 快(走主键索引)

> ⚠️ **InnoDB 不维护行数**:count 操作需要全表扫描;MyISAM 维护行数,count(*) 直接返回。所以 InnoDB 表的 count 比较慢。

### 5.4 in 和 exists 的区别

| 维度 | IN | EXISTS |
|------|----|----|
| **执行逻辑** | 先执行子查询,得到结果集再匹配 | 外表逐行判断子查询是否有结果 |
| **驱动表** | 子查询表(小) | 外表(小) |
| **NULL 处理** | NULL 会有问题 | 正常 |
| **适用场景** | 子查询结果集小 | 外表小,子查询大 |

**优化原则**:**小表驱动大表**
- 外表大、子查询表小 → 用 **IN**
- 外表小、子查询表大 → 用 **EXISTS**
- `not in` 永远不要用(NULL 问题 + 不走索引),改用 `not exists` 或 `left join...where is null`

```sql
-- IN 适合
SELECT * FROM big_table WHERE id IN (SELECT id FROM small_table);

-- EXISTS 适合
SELECT * FROM small_table s WHERE EXISTS (SELECT 1 FROM big_table b WHERE b.id = s.id);
```

> 💡 **MySQL 5.6+ 优化**:子查询会被优化器自动改写,IN 和 EXISTS 性能差距已经不大,但理解原理仍很重要。

---

## 六、SQL 执行顺序与解析

### 6.1 SQL 查询语句的执行顺序

**编写顺序**(我们怎么写 SQL):

```sql
SELECT  字段
FROM    表
JOIN    表
ON      条件
WHERE   条件
GROUP BY 字段
HAVING  条件
ORDER BY 字段
LIMIT   数量
```

**执行顺序**(MySQL 怎么执行):

```
1. FROM       → 选定表
2. ON         → 关联条件
3. JOIN       → 关联类型
4. WHERE      → 行级过滤
5. GROUP BY   → 分组
6. HAVING     → 组级过滤
7. SELECT     → 选字段
8. DISTINCT   → 去重
9. ORDER BY   → 排序
10. LIMIT     → 分页
```

**记忆口诀**:**先 FROM 再 ON,JOIN 完再过滤,分组聚合后,投影排序末**。

**关键点**:
- **WHERE 在 GROUP BY 之前**:WHERE 不能用聚合函数,HAVING 可以
- **SELECT 别名不能在 WHERE 用,但能在 ORDER BY 用**(SELECT 在 ORDER BY 之前)
- **LIMIT 最后执行**:深分页 `LIMIT 1000000, 10` 慢就是因为前面要扫 100 万行

```sql
-- 错误示例
SELECT name AS n FROM user WHERE n = 'tom';  -- ❌ WHERE 时还没有别名 n
SELECT name AS n FROM user ORDER BY n;       -- ✅ ORDER BY 时有别名 n
```

### 6.2 SQL 的语法树解析

MySQL 收到 SQL 后,会经过 **词法分析 → 语法分析 → 语义分析 → 优化器 → 执行** 几步:

```
SQL 字符串
   │ 词法分析(Lexical)
   ▼
Token 流 [SELECT, name, FROM, user, WHERE, id, =, 1]
   │ 语法分析(Syntax,基于 BNF 文法)
   ▼
抽象语法树 AST
   │ 语义分析
   ▼
解析树(检查表/列存在、类型匹配、权限)
   │ 优化器
   ▼
执行计划(Execution Plan)
   │ 执行器
   ▼
查询结果
```

**抽象语法树示例**:

```
SELECT name FROM user WHERE id = 1;

         SELECT
        /      \
   columns    FROM
      |        |
     name    user
              |
            WHERE
              |
             '='
            /   \
           id    1
```

> 💡 实战中可以用 `EXPLAIN` 查看执行计划,`EXPLAIN FORMAT=TREE`(MySQL 8)直接看树形结构。

### 6.3 SQL 的隐式数据类型转换

当 MySQL 比较不同类型的值时,会自动做类型转换,可能 **导致索引失效**!

**常见隐式转换坑**:

```sql
-- 假设 phone 是 varchar 类型并加了索引

-- ❌ 索引失效:字符串列与数字比较,字符串列被转为数字
SELECT * FROM user WHERE phone = 13800138000;

-- ✅ 索引生效
SELECT * FROM user WHERE phone = '13800138000';

-- ❌ 索引失效:列被函数处理,转为字符串
SELECT * FROM user WHERE DATE(create_time) = '2024-01-01';

-- ✅ 索引生效
SELECT * FROM user WHERE create_time >= '2024-01-01' AND create_time < '2024-01-02';
```

**类型转换规则**(简化版):
1. 两边都是数字 → 不转换
2. 一方是字符串、一方是数字 → **字符串转数字**(关键!)
3. 一方是日期、一方是字符串 → 字符串转日期
4. 涉及 NULL → 结果为 NULL

**性能口诀**:**列别动,值随便**(不要在列上做转换/函数,值可以随便处理)。

---

## 七、MySQL 常用命令与函数

### 7.1 MySQL 常用命令

**连接相关**:
```bash
mysql -h host -u user -p           # 连接
mysql -h host -P 3306 -u user -p database  # 指定端口和库
mysql -e "SHOW DATABASES;"         # 不进入交互模式直接执行
exit / quit / \q                   # 退出
```

**库表操作**:
```sql
SHOW DATABASES;                    -- 查看所有库
USE dbname;                        -- 切换库
SHOW TABLES;                       -- 查看所有表
DESC tablename;                    -- 查看表结构
SHOW CREATE TABLE tablename;       -- 查看建表语句
SHOW INDEX FROM tablename;         -- 查看索引
SHOW PROCESSLIST;                  -- 查看当前连接(排查慢查询)
SHOW STATUS LIKE '%lock%';         -- 查看锁状态
SHOW VARIABLES LIKE '%timeout%';   -- 查看系统变量
```

**性能分析**:
```sql
EXPLAIN SELECT ...;                -- 查看执行计划
EXPLAIN ANALYZE SELECT ...;        -- MySQL 8,实际执行后的分析
SHOW PROFILE;                      -- 查看查询性能
SHOW ENGINE INNODB STATUS;         -- InnoDB 详细状态
```

### 7.2 MySQL bin 目录下的可执行文件

| 命令 | 用途 |
|------|------|
| `mysql` | 客户端,连接服务器 |
| `mysqld` | 服务端守护进程 |
| `mysqldump` | **逻辑备份**(导出 SQL) |
| `mysqlimport` | 导入数据 |
| `mysqladmin` | 管理工具(创建库、查看状态、关闭服务) |
| `mysqlbinlog` | 解析 binlog 日志 |
| `mysqlcheck` | 检查、修复、优化、分析表 |
| `mysqlslap` | 压力测试工具 |
| `mysql_secure_installation` | 安全配置脚本 |
| `mysql_upgrade` | 版本升级 |

**最常用 mysqldump**:
```bash
# 备份整个库
mysqldump -u root -p mydb > backup.sql

# 备份多个库
mysqldump -u root -p --databases db1 db2 > backup.sql

# 备份单表
mysqldump -u root -p mydb users > users.sql

# 只导结构
mysqldump -u root -p -d mydb > schema.sql

# 只导数据
mysqldump -u root -p -t mydb > data.sql
```

### 7.3 MySQL 第 3-10 条记录怎么查询?

经典分页 SQL:`LIMIT offset, count`

```sql
-- 查第 3-10 条(共 8 条)
SELECT * FROM table ORDER BY id LIMIT 2, 8;
-- 含义:跳过前 2 条,取 8 条
```

**深分页优化**:`LIMIT 1000000, 10` 会慢,因为要扫描 100 万行再丢弃。

```sql
-- ❌ 慢:深分页
SELECT * FROM orders ORDER BY id LIMIT 1000000, 10;

-- ✅ 优化方案 1:子查询(只扫主键)
SELECT * FROM orders WHERE id >= (
    SELECT id FROM orders ORDER BY id LIMIT 1000000, 1
) LIMIT 10;

-- ✅ 优化方案 2:游标分页(记住上次最大 id)
SELECT * FROM orders WHERE id > 1000000 ORDER BY id LIMIT 10;
```

### 7.4 用过哪些 MySQL 函数?

**字符串函数**:
| 函数 | 说明 |
|------|------|
| `CONCAT(s1, s2, ...)` | 拼接字符串 |
| `CONCAT_WS(sep, s1, s2)` | 带分隔符拼接 |
| `LENGTH(s)` | 字节长度 |
| `CHAR_LENGTH(s)` | 字符长度 |
| `UPPER/LOWER` | 大小写转换 |
| `SUBSTRING(s, start, len)` | 截取子串 |
| `REPLACE(s, old, new)` | 替换 |
| `TRIM(s)` | 去首尾空格 |
| `LPAD/RPAD` | 左/右填充 |

**数值函数**:`ABS、CEIL、FLOOR、ROUND、MOD、RAND`

**日期函数**:
| 函数 | 说明 |
|------|------|
| `NOW() / CURRENT_TIMESTAMP` | 当前时间 |
| `CURDATE() / CURRENT_DATE` | 当前日期 |
| `DATE_ADD(date, INTERVAL 1 DAY)` | 日期加减 |
| `DATEDIFF(d1, d2)` | 日期差(天数) |
| `DATE_FORMAT(date, '%Y-%m-%d')` | 格式化 |
| `UNIX_TIMESTAMP() / FROM_UNIXTIME()` | 时间戳与日期互转 |

**聚合函数**:`COUNT、SUM、AVG、MAX、MIN、GROUP_CONCAT`

**条件函数**:
| 函数 | 说明 |
|------|------|
| `IF(条件, 真值, 假值)` | 三元表达式 |
| `IFNULL(v, default)` | NULL 替换默认值 |
| `CASE WHEN ... THEN ... ELSE ... END` | 多分支判断 |
| `COALESCE(v1, v2, ...)` | 返回第一个非 NULL 值 |
| `NULLIF(a, b)` | a=b 返回 NULL,否则返回 a |

**JSON 函数**(MySQL 5.7+):`JSON_EXTRACT、JSON_OBJECT、JSON_ARRAY、->、->>`

**窗口函数**(MySQL 8+):`ROW_NUMBER、RANK、DENSE_RANK、LAG、LEAD、SUM() OVER()`

**实战:每个部门工资前 3 名**(MySQL 8 窗口函数):
```sql
SELECT * FROM (
    SELECT *, RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rk
    FROM employee
) t WHERE rk <= 3;
```

---

## 八、MySQL 基础架构

### 8.1 MySQL 整体架构图

MySQL 采用 **分层架构**,经典的"客户端-服务器"模式:

```
┌──────────────────────────────────────────────┐
│           客户端 (mysql/JDBC/ORM)             │
└──────────────────┬───────────────────────────┘
                   │ 网络连接(TCP)
┌──────────────────▼───────────────────────────┐
│            连接器 (Connector)                 │
│   验证用户名密码、权限、维护连接、线程管理       │
└──────────────────┬───────────────────────────┘
                   ▼
┌──────────────────────────────────────────────┐
│         查询缓存 (Query Cache)                │
│   命中直接返回结果(MySQL 8.0 已删除!)         │
└──────────────────┬───────────────────────────┘
                   ▼
┌──────────────────────────────────────────────┐
│             分析器 (Parser)                   │
│   词法分析 + 语法分析 → AST                   │
└──────────────────┬───────────────────────────┘
                   ▼
┌──────────────────────────────────────────────┐
│           优化器 (Optimizer)                  │
│   选索引、决定 JOIN 顺序、改写 SQL             │
└──────────────────┬───────────────────────────┘
                   ▼
┌──────────────────────────────────────────────┐
│            执行器 (Executor)                  │
│   权限校验、调用存储引擎接口                   │
└──────────────────┬───────────────────────────┘
                   ▼
┌──────────────────────────────────────────────┐
│        存储引擎 (Storage Engine)              │
│  ┌────────┬────────┬────────┬────────┐       │
│  │ InnoDB │ MyISAM │ Memory │  ...   │       │
│  └────────┴────────┴────────┴────────┘       │
└──────────────────┬───────────────────────────┘
                   ▼
┌──────────────────────────────────────────────┐
│         物理存储 (磁盘文件)                    │
│   .ibd 数据文件、.frm 表结构、binlog、redo log│
└──────────────────────────────────────────────┘
```

### 8.2 各模块职责详解

**1. 连接器 (Connector)**
- 三次握手建立 TCP 连接
- 验证用户名密码
- 查询权限表,分配权限给该连接
- 维护连接(默认 8 小时空闲断开,`wait_timeout`)
- 长连接 vs 短连接:长连接性能好但占内存

```sql
SHOW PROCESSLIST;            -- 查看连接
KILL <connection_id>;        -- 强制断开
```

**2. 查询缓存 (Query Cache)**
- MySQL 8.0 **彻底删除**(更新表会清空缓存,失效率高,鸡肋)
- 8.0 之前默认关闭

**3. 分析器 (Parser)**
- 词法分析:把 SQL 字符串切分为 token
- 语法分析:按 SQL 语法生成 AST,语法错就报 "You have an error in your SQL syntax"

**4. 优化器 (Optimizer)**
- 多个索引选哪个
- 多表 JOIN 顺序
- 子查询改写为 JOIN
- 基于 **CBO(Cost-Based Optimizer)** 成本模型选最优执行计划

**5. 执行器 (Executor)**
- 检查表/字段权限
- 调用存储引擎 API,逐行返回数据
- 把结果集返回客户端

**6. 存储引擎 (Storage Engine)**
- **可插拔架构**(MySQL 一大亮点)
- InnoDB:默认,支持事务、行锁、外键、MVCC
- MyISAM:不支持事务,表锁,读多写少场景(已被 InnoDB 取代)
- Memory:数据放内存,极快但断电丢失
- Archive:仅支持 INSERT 和 SELECT,高压缩,归档场景

### 8.3 InnoDB vs MyISAM

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| **事务** | ✅ 支持(ACID) | ❌ 不支持 |
| **锁粒度** | 行锁 + 表锁 | 表锁 |
| **外键** | ✅ 支持 | ❌ 不支持 |
| **MVCC** | ✅ 支持 | ❌ 不支持 |
| **崩溃恢复** | ✅ 通过 redo log | ❌ 容易损坏 |
| **全文索引** | ✅(5.6+) | ✅ |
| **count(*)** | 慢(需扫表) | 快(维护行数) |
| **存储** | 数据+索引一起(.ibd) | 数据(.MYD)+索引(.MYI)分开 |
| **适用场景** | OLTP、并发写、需事务 | 只读、统计、归档 |

> 💡 **现代选择**:99% 场景用 InnoDB,MyISAM 已经基本被淘汰。

---

## 九、查询语句执行流程

### 9.1 一条 SELECT 是如何执行的?

以 `SELECT * FROM user WHERE id = 1;` 为例:

```
1. 客户端 → 连接器
   建立连接、验证身份、分配权限

2. (MySQL 8 跳过)查询缓存
   key=SQL,value=结果。命中直接返回

3. 分析器
   - 词法分析:识别 SELECT、user、id 等 token
   - 语法分析:生成 AST,检查语法

4. 优化器
   - 检查表 user 是否存在
   - 检查列 id 是否存在
   - 选择索引:发现 id 是主键 → 走主键索引
   - 决定执行计划

5. 执行器
   - 检查用户对 user 表的 SELECT 权限
   - 调用 InnoDB 引擎接口:
     · 第一次调用:取主键 id=1 的第一行
     · 判断是否符合 where(此处 where 已通过索引,无需再判断)
     · 加入结果集
   - 返回结果集给客户端

6. 存储引擎层
   - InnoDB Buffer Pool 中查找数据页
   - 没有就从磁盘加载(.ibd 文件)
   - 通过 B+Tree 索引定位记录
```

**整体流程图**:

```
SELECT * FROM user WHERE id=1
         ↓
    [连接器] → 验证权限
         ↓
    [分析器] → 生成 AST
         ↓
    [优化器] → 选择主键索引
         ↓
    [执行器] → 调用 InnoDB API
         ↓
    [InnoDB] → Buffer Pool / 磁盘
         ↓
       结果返回
```

---

## 十、更新语句执行流程

### 10.1 一条 UPDATE 是如何执行的?

以 `UPDATE user SET name='tom' WHERE id=1;` 为例。

**核心区别**:UPDATE 不仅要走查询的全流程,**还要写两个日志:redo log 和 binlog**,并且采用 **两阶段提交**。

```
1-5 步同 SELECT:连接器 → 分析器 → 优化器 → 执行器 → 引擎层

6. 执行器调用 InnoDB API,读 id=1 的行
   - 数据页在 Buffer Pool 中?是 → 直接读
   - 不在?从磁盘读到 Buffer Pool

7. 执行器拿到数据,修改 name='tom',调用引擎写入

8. InnoDB 写 redo log(物理日志)
   - 状态:prepare
   - 记录:在第 X 页第 Y 行,name 改为 'tom'
   - 写到 redo log buffer,后台刷盘

9. 执行器写 binlog(逻辑日志)
   - 记录:UPDATE user SET name='tom' WHERE id=1
   - 写到 binlog cache,事务提交时刷盘

10. InnoDB 提交 redo log
    - 状态:prepare → commit

11. 返回客户端:更新成功
```

### 10.2 redo log 与 binlog

| 维度 | redo log | binlog |
|------|----------|--------|
| **所属层** | InnoDB 引擎层 | Server 层 |
| **格式** | 物理日志(数据页改动) | 逻辑日志(SQL 语句/行变更) |
| **写入方式** | **循环写**(覆盖旧记录) | **追加写**(写满换文件) |
| **作用** | **崩溃恢复** | **主从复制、数据备份** |
| **大小** | 固定(4 个文件,每个 1G) | 不限 |

**redo log 的作用**:
- **崩溃恢复 (crash-safe)**:数据库异常重启后,根据 redo log 把已提交但未刷盘的数据恢复
- **WAL (Write-Ahead Logging)**:先写日志再写磁盘,顺序写远快于随机写

**binlog 的作用**:
- 主从复制:从库读主库的 binlog 重放
- 数据备份与恢复:误删数据时,通过 binlog 回放恢复
- 数据审计

### 10.3 两阶段提交 (2PC)

为什么要两阶段?**保证 redo log 和 binlog 的一致性**,防止主从数据不一致。

```
                    时刻 T
            ┌────────────────────┐
            │  写 redo log(prepare)│
            └─────────┬──────────┘
                      │
                      ▼
            ┌────────────────────┐
            │     写 binlog       │
            └─────────┬──────────┘
                      │
                      ▼
            ┌────────────────────┐
            │  写 redo log(commit)│
            └────────────────────┘
```

**崩溃恢复规则**:
- 在 prepare 阶段崩溃 → redo log 无 commit 标记 → 回滚
- 写完 binlog,但 redo log 未 commit → 检查 binlog 是否完整 → 完整就提交,不完整就回滚
- 全部完成 → 数据已落盘

**为什么不一阶段?** 假设只有一个日志:
- 先 redo 后 binlog:redo 写完,binlog 没写,崩溃后主库恢复有这条数据,从库没有 → 不一致
- 先 binlog 后 redo:binlog 写完,redo 没写,崩溃后主库丢失这条数据,从库有 → 不一致

### 10.4 update 流程图

```
UPDATE user SET name='tom' WHERE id=1
         ↓
    [Server 层]
    连接器 → 分析器 → 优化器 → 执行器
         ↓
    [InnoDB 层]
    Buffer Pool 读取 id=1 的行
         ↓
    [Server 层]
    执行器修改:name='tom'
         ↓
    [InnoDB 层]
    写入新值,生成 redo log (prepare)
         ↓
    [Server 层]
    写 binlog
         ↓
    [InnoDB 层]
    redo log 状态改为 commit
         ↓
       事务完成
```

---

## 十一、核心知识点总结

### 11.1 一图流(全景图)

```
┌──────────────────────────────────────────────┐
│              MySQL 知识体系                    │
├──────────────────────────────────────────────┤
│  基础概念                                      │
│   ├── 关系型 DB、ACID、SQL 标准                 │
│   └── 主流版本:5.7 / 8.0                      │
├──────────────────────────────────────────────┤
│  数据类型                                      │
│   ├── 字符串:CHAR / VARCHAR / TEXT / BLOB     │
│   ├── 数字:INT / BIGINT / DECIMAL             │
│   ├── 时间:DATETIME / TIMESTAMP               │
│   └── 字符集:utf8mb4(支持 emoji)             │
├──────────────────────────────────────────────┤
│  SQL 语法                                      │
│   ├── DML / DDL / DCL                         │
│   ├── JOIN / UNION / 子查询                    │
│   ├── 聚合 / 分组 / 排序                       │
│   └── 执行顺序:FROM→WHERE→GROUP→SELECT→ORDER  │
├──────────────────────────────────────────────┤
│  架构(分层)                                  │
│   ├── Server 层:连接器/分析器/优化器/执行器    │
│   └── 引擎层:InnoDB(主流)、MyISAM           │
├──────────────────────────────────────────────┤
│  执行流程                                      │
│   ├── SELECT:5 步走                           │
│   └── UPDATE:redo log + binlog 两阶段提交     │
└──────────────────────────────────────────────┘
```

### 11.2 高频面试题(自查清单)

- [ ] 什么是 MySQL?为什么用它而不是 Oracle?
- [ ] InnoDB 和 MyISAM 的区别?
- [ ] 数据库三大范式?为什么要反范式?
- [ ] varchar 和 char 的区别?什么时候用 char?
- [ ] DATETIME 和 TIMESTAMP 的区别?
- [ ] 为什么货币不能用 float?
- [ ] 怎么存储 emoji?utf8 和 utf8mb4 区别?
- [ ] drop、delete、truncate 区别?可以回滚吗?
- [ ] UNION 和 UNION ALL 的区别?
- [ ] count(\*)、count(1)、count(列) 区别?哪个最快?
- [ ] in 和 exists 区别?什么时候用哪个?
- [ ] SQL 的执行顺序是什么?
- [ ] 隐式类型转换会导致什么问题?
- [ ] MySQL 整体架构是怎样的?
- [ ] 一条 SELECT 是如何执行的?
- [ ] 一条 UPDATE 是如何执行的?
- [ ] redo log 和 binlog 的区别?
- [ ] 为什么要两阶段提交?

### 11.3 实战经验

**1. 字段设计**
- 能用 NOT NULL 就别用 NULL(NULL 影响索引、聚合函数)
- 字符串字段都给默认值(空串 `''`),不要 NULL
- 时间字段统一用 `DATETIME(3)` 或 `TIMESTAMP`
- 金额一律 `DECIMAL(20, 2)` 或扩大 100 倍存 `BIGINT`
- 状态用 `TINYINT`,加注释说明每个值含义

**2. SQL 编写**
- 永远不要 `SELECT *`,需要什么字段写什么
- WHERE 条件尽量走索引(联合索引最左前缀)
- 不要在索引列上做函数/计算
- LIMIT 深分页用 id > xxx 的游标方式
- JOIN 不超过 3 张表,大表关联走索引列
- 小表驱动大表

**3. 字符集与排序**
- 全局 utf8mb4
- 排序规则:utf8mb4_0900_ai_ci(MySQL 8 默认)或 utf8mb4_unicode_ci

**4. 备份策略**
- 物理备份:Percona XtraBackup(在线热备)
- 逻辑备份:mysqldump(小库)
- binlog 增量备份
- 主从 + 定期备份 + 异地容灾

### 11.4 推荐资源

📚 **必读书籍**
- 《MySQL 必知必会》—— 入门
- 《高性能 MySQL》(第 4 版) ⭐⭐⭐⭐⭐
- 《MySQL 技术内幕:InnoDB 存储引擎》—— 姜承尧
- 《MySQL 是怎样运行的》—— 小孩子 4919

🌐 **在线资料**
- [MySQL 8.0 官方文档](https://dev.mysql.com/doc/refman/8.0/en/)
- [极客时间 - MySQL 实战 45 讲](https://time.geekbang.org/column/intro/100020801)(强烈推荐)
- [Use The Index, Luke!](https://use-the-index-luke.com/)

🛠 **常用工具**
- 客户端:Navicat、DataGrip、DBeaver、HeidiSQL
- 监控:Prometheus + mysqld_exporter + Grafana
- 慢查询分析:pt-query-digest
- 在线 DDL:gh-ost、pt-online-schema-change

---

## 📌 Day 22 总结

| 维度 | 收获 |
|------|------|
| **基础扎实** | 理解 MySQL 是什么、各种数据类型如何选型 |
| **SQL 熟练** | 掌握连接、范式、UNION、count、in/exists 等高频用法 |
| **架构理解** | 看懂 Server 层 + 引擎层的分层架构 |
| **流程闭环** | 理解 SELECT 和 UPDATE 的完整执行链路 |
| **日志机制** | 区分 redo log 和 binlog,理解两阶段提交的必要性 |
| **工程实践** | 字段设计、SQL 优化、字符集、备份等实战经验 |

> MySQL 是后端工程师的必修课,**深度** 决定了你能走多远。下一篇将继续深入索引、事务、锁、MVCC 等核心机制。

**Day 22 ✅ Done — 持续更新中,欢迎 Star ⭐**
