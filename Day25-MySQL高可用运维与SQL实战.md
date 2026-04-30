# Day 25 - MySQL 高可用、运维实战与 SQL 题

> 完结 MySQL 系列。本篇覆盖 **读写分离、主从复制、分库分表、运维实战、经典 SQL 面试题**,是从原理走向工程实践的关键一步,也是中高级面试的常见考点。

---

## 📚 目录

- [一、MySQL 读写分离](#一mysql-读写分离)
- [二、读写分离的实现方式](#二读写分离的实现方式)
- [三、主从复制原理](#三主从复制原理)
- [四、主从同步延迟](#四主从同步延迟)
- [五、分库的实践](#五分库的实践)
- [六、分表的实践](#六分表的实践)
- [七、水平分库分表的分片策略](#七水平分库分表的分片策略)
- [八、不停机扩容方案](#八不停机扩容方案)
- [九、常用分库分表中间件](#九常用分库分表中间件)
- [十、分库分表带来的问题](#十分库分表带来的问题)
- [十一、运维实战:百万级数据删除](#十一运维实战百万级数据删除)
- [十二、运维实战:千万级大表加字段](#十二运维实战千万级大表加字段)
- [十三、运维实战:CPU 飙升排查](#十三运维实战cpu-飙升排查)
- [十四、SQL 经典面试题](#十四sql-经典面试题)
- [十五、核心知识点总结](#十五核心知识点总结)

---

## 一、MySQL 读写分离

### 1.1 什么是读写分离?

**读写分离 (Read/Write Split)**:**写操作走主库,读操作走从库**,通过多台从库分担读压力。

```
                    ┌────────────────┐
                    │     应用层      │
                    └───────┬────────┘
                            │
                ┌───────────┴───────────┐
                ▼                       ▼
            写请求                    读请求
                │                       │
                ▼                       ▼
          ┌──────────┐         ┌────────────────┐
          │  Master  │ ──────→ │  Slave1 / Slave2│
          │  (主库)   │  复制    │   (从库,负载均衡)│
          └──────────┘         └────────────────┘
```

### 1.2 为什么需要读写分离?

**互联网业务特点**:**读多写少**(典型读写比 4:1 ~ 10:1,甚至 100:1)。

**单库的瓶颈**:
- 单台 MySQL 极限约 **5000 QPS**(SSD,简单查询)
- CPU、IO 容易成为瓶颈
- 备份、统计查询会拖慢线上业务

**读写分离的好处**:
1. **横向扩展**:加从库就能提升读能力
2. **隔离慢查询**:复杂报表走专门的从库
3. **高可用**:主库故障时从库可顶上(配合 MHA/MGR)
4. **备份不影响业务**:在从库上做备份

### 1.3 读写分离的代价

- **数据延迟**:从库有同步延迟,刚写入立即读可能读不到
- **架构复杂**:多了路由、监控、切换的复杂度
- **不能盲目用**:写后立即读、强一致需求场景需要走主库

---

## 二、读写分离的实现方式

### 2.1 三种主流实现

**1. 应用层(代码硬编码)**
```java
// 简单粗暴
if (isWriteOperation()) {
    masterDataSource.query(sql);
} else {
    slaveDataSource.query(sql);
}
```
- ✅ 简单,无中间件依赖
- ❌ 业务侵入大,每个项目都要写
- 🛠 工具:Spring `AbstractRoutingDataSource`、Sharding-JDBC

**2. 代理层(中间件)**

```
应用 → 中间件代理 → Master / Slaves
```

应用透明无感,中间件自动路由 SELECT 到从库,其他到主库。

- ✅ 业务零改动
- ❌ 多一跳网络,增加延迟
- 🛠 工具:**MyCat、ShardingSphere-Proxy、ProxySQL、Atlas**

**3. JDBC 层(嵌入式)**

如 **Sharding-JDBC**(现 ShardingSphere-JDBC),以 JAR 包形式集成到应用。

```
应用 ← Sharding-JDBC ← 直连 → Master / Slaves
```

- ✅ 性能好(无代理跳转)
- ✅ 配置灵活
- ❌ 仅 Java 生态

### 2.2 路由策略

**默认规则**:
- `INSERT / UPDATE / DELETE` → 主库
- `SELECT` → 从库
- 显式事务内 → 全部走主库(避免主从延迟问题)

**特殊场景强制走主库**:
```java
// Sharding-JDBC 提供的强制路由
HintManager hint = HintManager.getInstance();
hint.setMasterRouteOnly();  // 这次查询强制走主库
```

**应用场景**:
- 注册后立即读用户信息(写后立读)
- 资金核算等强一致场景
- 报表类长查询要避开主库

---

## 三、主从复制原理

### 3.1 主从复制流程

**MySQL 主从复制基于 binlog**,核心是三个线程:

```
┌──────────────────────────────────────────────────────┐
│                    Master                            │
│  ┌───────────────┐                                   │
│  │ 客户端写入      │                                   │
│  └──────┬────────┘                                   │
│         ▼                                            │
│  ┌───────────────┐    ┌──────────────┐               │
│  │  事务提交       │ ─→ │   binlog     │               │
│  └───────────────┘    └──────┬───────┘               │
│                              │                       │
│                       ┌──────┴──────────┐            │
│                       │ binlog dump 线程 │            │
│                       └──────┬──────────┘            │
└──────────────────────────────┼───────────────────────┘
                               │ 网络
┌──────────────────────────────┼───────────────────────┐
│                    Slave     ▼                       │
│                       ┌──────────────────┐           │
│                       │   IO 线程         │           │
│                       │   接收并写入      │           │
│                       └──────┬───────────┘           │
│                              ▼                       │
│                       ┌──────────────┐               │
│                       │  relay log   │  中继日志      │
│                       └──────┬───────┘               │
│                              ▼                       │
│                       ┌──────────────────┐           │
│                       │   SQL 线程        │           │
│                       │   重放 binlog     │           │
│                       └──────┬───────────┘           │
│                              ▼                       │
│                       ┌──────────────────┐           │
│                       │  从库数据         │           │
│                       └──────────────────┘           │
└──────────────────────────────────────────────────────┘
```

**详细流程**:
1. **主库**:事务提交时把变更写入 **binlog**
2. **主库 binlog dump 线程**:通知从库有新 binlog,把 binlog 内容发过去
3. **从库 IO 线程**:接收 binlog,写入本地 **relay log**
4. **从库 SQL 线程**:读 relay log,重放 SQL 应用到从库
5. 从库数据与主库一致

### 3.2 三种复制模式

**1. 异步复制(默认)**

主库写完 binlog 就返回客户端成功,不等从库。

- ✅ 性能最好
- ❌ 主库挂了,未同步的 binlog 丢失

**2. 半同步复制 (Semi-Sync)**

主库等 **至少一个从库** 收到 binlog 后才返回成功。

- ✅ 数据安全性提升
- ❌ 性能略降,主库要等网络往返

```sql
-- 主库
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET GLOBAL rpl_semi_sync_master_enabled = 1;

-- 从库
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
SET GLOBAL rpl_semi_sync_slave_enabled = 1;
```

**3. 全同步复制(组复制 MGR)**

所有从库都同步成功才返回。

- ✅ 数据强一致
- ❌ 性能最差,任一从库慢就拖累主库

**MySQL Group Replication (MGR)**:基于 Paxos 的多主同步,MySQL 5.7+ 推出。

### 3.3 binlog 的三种格式(回顾)

| 格式 | 内容 | 主从一致性 | 大小 |
|------|------|----------|------|
| **STATEMENT** | SQL 语句 | 弱(函数问题) | 小 |
| **ROW** | 行变更 | 强 | 大 |
| **MIXED** | 混合 | 较强 | 中 |

**MySQL 8 默认 ROW**,推荐生产用 ROW。

### 3.4 主从复制的拓扑

```
1. 一主一从        Master ──→ Slave

2. 一主多从        Master ──┬─→ Slave1
                            ├─→ Slave2
                            └─→ Slave3

3. 级联复制        Master ──→ Slave1 ──→ Slave2
                                  └────→ Slave3
                                  (Slave1 既做从又做主)

4. 双主复制        Master1 ⇄ Master2  (互为主从,适合多活)

5. MGR 组复制      Node1 ⇄ Node2 ⇄ Node3  (无主,Paxos 协议)
```

---

## 四、主从同步延迟

### 4.1 主从延迟的原因

**核心原因**:**SQL 重放是单线程的**(MySQL 5.6 之前完全单线程,5.7+ 改进)。

主库可以并发写,从库 SQL 线程串行重放,**写入压力大时从库追不上主库**。

**常见诱因**:
1. **大事务**:主库一个事务跑 1 分钟,从库也得 1 分钟,期间所有同步阻塞
2. **大表 DDL**:`ALTER TABLE` 大表,从库重放期间延迟急剧上升
3. **从库性能差**:硬件配置不如主库
4. **网络延迟**:跨机房复制
5. **从库压力大**:从库还要扛读流量,SQL 线程被挤压

### 4.2 如何监控延迟?

```sql
-- 在从库执行
SHOW SLAVE STATUS\G

-- 关键字段
Seconds_Behind_Master: 0     -- 延迟秒数(0 表示无延迟)
Slave_IO_Running: Yes        -- IO 线程状态
Slave_SQL_Running: Yes       -- SQL 线程状态
```

> ⚠️ `Seconds_Behind_Master` 不完全准确(基于 SQL 线程当前执行的事件时间戳与主库的差),建议配合 **pt-heartbeat** 工具精确测量。

### 4.3 主从延迟怎么处理?

**方案 1:并行复制(MySQL 5.7+)**

从库 SQL 线程改为多线程重放,大幅提升从库吞吐。

```sql
SET GLOBAL slave_parallel_type = 'LOGICAL_CLOCK';
SET GLOBAL slave_parallel_workers = 8;
```

**方案 2:业务层处理**
- **关键查询走主库**:写后立读、资金等场景强制路由主库
- **延时容忍设计**:UI 上提示"数据可能有 X 秒延迟"
- **本地缓存**:写完顺手缓存,N 秒内自己读自己的写入

**方案 3:架构优化**
- 减少大事务(拆分 DELETE/UPDATE)
- 大表 DDL 用 gh-ost(无锁在线变更)
- 从库高配硬件
- 跨机房用专线 / 物理就近

**方案 4:半同步复制 + 强一致需求**
- 主从切换后保证至少一个从库与主库一致
- 配合 MGR 实现强一致

### 4.4 写后立读的解决方案

**问题**:用户刚提交了订单,刷新页面查询订单列表,因主从延迟看不到新订单。

**方案对比**:

| 方案 | 说明 | 缺点 |
|------|------|------|
| **强制走主库** | 写后 N 秒内的查询路由主库 | 主库压力大 |
| **客户端缓存** | 提交后本地缓存,N 秒内优先用 | 复杂 |
| **半同步** | 至少一个从库已同步 | 性能略降 |
| **判断延迟** | 查从库前看 `Seconds_Behind_Master`,大于阈值切主库 | 需中间件支持 |
| **GTID 等位** | 写完拿到 GTID,从库等到该 GTID 再读 | 实现复杂 |

---

## 五、分库的实践

### 5.1 为什么需要分库?

单库的瓶颈:
- **连接数限制**:MySQL 默认 1000+,实际推荐不超 500
- **磁盘 IO 瓶颈**:大表查询拖慢全库
- **CPU 瓶颈**:复杂查询占满 CPU
- **可用性**:单库挂了全业务停摆

**分库的两种方向**:

```
垂直分库:按业务模块拆            水平分库:按数据切片
                              
   原库:                           原库:user(1000万行)
   user/order/product            
                                    分库后:
   分库后:                         user_db_0:id 哈希=0
   user_db / order_db /            user_db_1:id 哈希=1
   product_db                      user_db_2:id 哈希=2
                                   user_db_3:id 哈希=3
```

### 5.2 怎么分库?

**垂直分库**:按业务模块切分

```
user 业务 → user_db
   ├ user 表
   ├ user_profile 表
   └ user_account 表

order 业务 → order_db
   ├ order 表
   ├ order_item 表
   └ order_address 表

product 业务 → product_db
   ├ product 表
   └ category 表
```

**优点**:
- 业务解耦
- 单库压力下降
- 故障隔离(订单库挂了不影响商品)

**缺点**:
- 跨库 JOIN 不行,需应用层处理
- 跨库事务难

**水平分库**:同一类数据拆到多个库

```sql
-- 按 user_id 哈希
db_id = user_id % 4

user_id=100 → db_0
user_id=101 → db_1
user_id=102 → db_2
user_id=103 → db_3
```

**适用场景**:
- 单库数据量过大(50G+)
- 单业务并发量过高

---

## 六、分表的实践

### 6.1 什么时候分表?

经验阈值:
- **单表 1000 万行** 开始考虑分表
- **单表 5000 万行 / 50GB** 必须分表
- **B+ 树超过 4 层**(查询性能明显下降)

### 6.2 分表的两种方向

**垂直分表**:按字段拆分

```
原表 user(id, name, age, avatar, signature, biography_long_text)
     ↓
拆分:
  user_basic(id, name, age)              ← 高频访问
  user_profile(id, avatar, signature)    ← 中频
  user_extend(id, biography_long_text)   ← 低频大字段
```

**适用**:
- 表字段过多(几十个)
- 大字段(TEXT/BLOB)拖慢主查询
- 冷热数据分离

**水平分表**:按行拆分

```
原表 order(1 亿行)
     ↓ 按 user_id 哈希分 16 张表
  order_0, order_1, ..., order_15
```

**适用**:
- 单表行数过大
- 单条记录小但量级大

### 6.3 分库分表的常见组合

```
1. 只分库不分表:垂直分库,业务隔离
2. 只分表不分库:单业务数据量大,但 IO 压力可控
3. 既分库又分表:大型互联网应用标配

   例:订单系统
   ├ 4 个库:order_db_0 ~ order_db_3
   └ 每个库 16 张表:order_0 ~ order_15
   总共 64 张表,可承载几十亿订单
```

---

## 七、水平分库分表的分片策略

### 7.1 四大主流分片策略

**1. 范围分片 (Range)**

按字段范围切分。

```
user_id 1-1000万   → db_0
user_id 1001万-2000万 → db_1
user_id 2001万-3000万 → db_2
```

或按时间:
```
2024-01 ~ 2024-06 → db_0
2024-07 ~ 2024-12 → db_1
```

- ✅ 扩容简单(加新范围段)
- ✅ 范围查询友好
- ❌ **数据热点**:新数据全集中在最新段,老段空闲

**2. 哈希分片 (Hash)** ⭐ 最常用

```
db_id = user_id % 4
```

- ✅ 数据分布均匀
- ✅ 单点查询定位快
- ❌ **扩容痛**:加一个库,全部数据要重新哈希
- ❌ 范围查询要查所有分片

**3. 一致性哈希 (Consistent Hash)**

哈希环,扩容时只影响相邻节点。

```
        node_0
       /      \
  node_3      node_1
       \      /
        node_2

加 node_4 后,只有 node_4 旁边的数据需迁移
```

- ✅ 扩容只迁移少量数据
- ❌ 实现复杂,数据分布可能不均(虚拟节点改善)

**4. 查表/路由 (Lookup)**

维护一张映射表,根据业务字段查路由。

```
user_id → 路由表 → db_id

user_id=100 → 路由表查到 db_2
```

- ✅ 灵活,可任意调整路由
- ❌ 多一次查询
- ❌ 路由表本身可能成为瓶颈

### 7.2 分片键 (Sharding Key) 的选择

**好的分片键**:
- **业务高频查询字段**(避免广播查询)
- **数据分布均匀**(避免热点)
- **不可变**(改动会引发数据迁移)

**典型选择**:
- 用户系统 → user_id
- 订单系统 → user_id(优先)或 order_id
- IM 消息 → conversation_id
- 日志系统 → 时间(范围分片)

**反例**:
- 用 status(状态字段)→ 数据严重倾斜
- 用自增 ID(不带业务含义)→ 无法按用户聚合查询

### 7.3 复合分片(双维度)

**问题**:订单表用 user_id 分片好查 "我的订单",但商家查 "店铺所有订单" 怎么办?

**方案**:**双向冗余**

```
order_by_user(按 user_id 分片) → C 端用户查
order_by_shop(按 shop_id 分片) → B 端商家查

写入时双写,binlog 异步同步保证最终一致
```

---

## 八、不停机扩容方案

### 8.1 为什么扩容是个难题?

哈希分片下,**加一个库,几乎所有数据都要迁移**(取模值变了)。

```
原:user_id % 4  → 4 个库
扩容后:user_id % 5  → 5 个库

原 user_id=100 在 db_0 (100%4=0)
现 user_id=100 在 db_0 (100%5=0)  ← 巧合相等
原 user_id=101 在 db_1 (101%4=1)
现 user_id=101 在 db_1 (101%5=1)
原 user_id=104 在 db_0 (104%4=0)
现 user_id=104 在 db_4 (104%5=4)  ← 要迁移!
```

约 **80% 的数据需要迁移**。

### 8.2 经典方案:成倍扩容(2 倍法)

利用哈希性质:**扩容到原来的 2 倍,只需迁移一半数据**。

```
原:4 个库,user_id % 4
扩容:8 个库,user_id % 8

user_id=100:
  原:100 % 4 = 0 → db_0
  新:100 % 8 = 4 → db_4 (迁移)

user_id=104:
  原:104 % 4 = 0 → db_0
  新:104 % 8 = 0 → db_0 (不变)

规律:user_id % 8 的结果要么等于 user_id % 4,要么等于 user_id % 4 + 4
```

**操作流程**:
1. 准备 4 台新机器(从原 4 台同步数据)
2. 配置:每台原库 → 新库的复制关系
3. 等数据同步完成
4. 路由切换:`% 4` 改为 `% 8`
5. 删除每个库中"不属于自己"的数据

### 8.3 在线平滑扩容步骤

```
阶段 1:数据双写
  └ 应用同时写入旧库和新库,读旧库

阶段 2:历史数据迁移
  └ 按 user_id 范围批量迁移,工具:DataX、MySQL dump

阶段 3:数据校验
  └ 对比新旧库数据一致性

阶段 4:读切换
  └ 灰度切流到新库读,观察一段时间

阶段 5:停止双写
  └ 关闭旧库写入,数据完全切换

阶段 6:删除旧库
  └ 确认无问题后下线旧库
```

### 8.4 避免扩容的设计

**方案 1:一致性哈希**(扩容只迁部分数据)

**方案 2:预分片**(一次性分多,前期合并)

```
设计时:逻辑分 1024 张表
初期:4 个库,每个库 256 张表
扩容:8 个库,每个库 128 张表(只搬表,不迁数据)
```

**方案 3:基因法**(将分片信息编码到主键)

```
order_id = user_id 后 4 位 + 业务序号

user_id=12345
order_id = 2345_xxxxxx  ← 后 4 位含 user_id 信息

无论用 user_id 还是 order_id 查,都能定位到分片
```

---

## 九、常用分库分表中间件

### 9.1 主流中间件对比

| 中间件 | 类型 | 特点 | 推荐度 |
|--------|------|------|-------|
| **ShardingSphere-JDBC** | JDBC 嵌入式 | Apache 顶级,配置丰富 | ⭐⭐⭐⭐⭐ |
| **ShardingSphere-Proxy** | 代理 | 上面同款的代理版本 | ⭐⭐⭐⭐⭐ |
| **MyCat** | 代理 | 老牌国产,文档多 | ⭐⭐⭐ |
| **Vitess** | 代理 | YouTube 出品,K8s 友好 | ⭐⭐⭐⭐(国外) |
| **Cobar** | 代理 | 阿里早期开源,已停更 | ❌ |
| **TDDL** | JDBC | 阿里内部,半开源 | ⭐⭐ |

**ShardingSphere 主推理由**:
- Apache 顶级项目,活跃度高
- 支持分库、分表、读写分离、影子库
- JDBC + Proxy 双模式,场景全覆盖
- 数据加密、分布式事务、分布式治理

### 9.2 ShardingSphere-JDBC 简单示例

```yaml
# application.yml
spring:
  shardingsphere:
    datasource:
      names: ds0,ds1
      ds0:
        url: jdbc:mysql://host1:3306/db0
        username: root
        password: ***
      ds1:
        url: jdbc:mysql://host2:3306/db0
        username: root
        password: ***
    rules:
      sharding:
        tables:
          t_order:
            actual-data-nodes: ds$->{0..1}.t_order_$->{0..15}
            database-strategy:
              standard:
                sharding-column: user_id
                sharding-algorithm-name: db-mod
            table-strategy:
              standard:
                sharding-column: order_id
                sharding-algorithm-name: table-hash
```

### 9.3 选型建议

- **新项目、Java 单语言** → ShardingSphere-JDBC
- **多语言、统一管理** → ShardingSphere-Proxy / Vitess
- **学习成本敏感** → MyCat(教程多但更新慢)
- **大型互联网公司** → 自研(如阿里 TDDL、字节 ByteHouse)

---

## 十、分库分表带来的问题

分库分表 **不是银弹**,带来一系列复杂度。

### 10.1 跨库 JOIN 不行了

**原生 JOIN 失效**,需要应用层处理:

```sql
-- 之前
SELECT u.name, o.amount FROM user u JOIN orders o ON u.id = o.user_id;

-- 分库后(假设 user 在 db_user,orders 在 db_order)
-- 必须分两步:
List<Order> orders = queryOrders();
List<Long> userIds = extractUserIds(orders);
List<User> users = queryUsersByIds(userIds);  -- IN 查询批量
// 应用层关联
```

**优化思路**:
- **冗余字段**(订单表冗余 user_name)
- **绑定表**(JOIN 字段相同,分到一个库)
- **广播表**(小字典表全库都有一份)

### 10.2 分布式事务

跨库写入无法靠本地事务保证。

**方案对比**:

| 方案 | 强一致 | 性能 | 复杂度 | 场景 |
|------|-------|------|-------|------|
| **2PC / XA** | ✅ | 差 | 中 | 金融、强一致 |
| **TCC (Try-Confirm-Cancel)** | ✅ | 好 | 高 | 业务定制 |
| **Saga** | 最终 | 好 | 中 | 长事务 |
| **本地消息表** | 最终 | 好 | 低 | 简单场景 |
| **MQ 事务消息** | 最终 | 好 | 低 | 异步解耦 |

**实战经验**:**能不用强一致就别用,优先选择最终一致**(MQ + 本地消息表)。

### 10.3 全局唯一 ID

自增 ID 在多库下会冲突。

**主流方案**:

| 方案 | 优点 | 缺点 |
|------|------|------|
| **UUID** | 全局唯一,无中心 | 太长(36字节)、无序、不适合主键 |
| **数据库号段** | 趋势递增,性能好 | 依赖数据库 |
| **Redis incr** | 高性能 | 依赖 Redis |
| **Snowflake** ⭐ | 64bit,有序,本地生成 | 时钟回拨问题 |
| **美团 Leaf** | 工程化 Snowflake | - |
| **百度 UID-Generator** | 优化版 | - |

**Snowflake 结构**:

```
0 | 41 位时间戳 | 10 位机器 ID | 12 位序号
↑   毫秒级       1024 台机器     每毫秒 4096 个

总长度 64 位,正好一个 Long
有序、本地生成、无依赖
```

### 10.4 分页排序

```sql
-- 查"全部订单按时间倒序前 10 条"
-- 单库:SELECT * FROM orders ORDER BY create_time DESC LIMIT 10

-- 分库后,需要从每个库各取 10 条,再合并排序取前 10
-- 假设 8 个库,要查 80 条,在内存排序后取 10 条
```

**深分页问题更严重**:`LIMIT 1000000, 10` 在每个库都要扫,内存合并成本极高。

**优化**:
- **二次查询法**:第一次得到大概范围,第二次精确定位
- **基于游标分页**:`WHERE id > last_id LIMIT 10`(推荐)
- **业务上禁用深分页**(只允许查最近 N 页)

### 10.5 分布式锁

单库的 `SELECT ... FOR UPDATE` 不再适用,需要 Redis、ZooKeeper 等分布式锁。

### 10.6 数据迁移与扩容(已在第 8 节讲)

### 10.7 总结:分库分表的代价

```
┌──────────────────────────────────────┐
│   分库分表带来的复杂度                  │
├──────────────────────────────────────┤
│   ❌ 跨库 JOIN 失效                    │
│   ❌ 分布式事务                        │
│   ❌ 全局 ID 生成                      │
│   ❌ 分页排序复杂                       │
│   ❌ 跨库聚合 (count/sum)              │
│   ❌ 分布式锁                          │
│   ❌ 扩容迁移困难                       │
│   ❌ 运维成本激增                       │
└──────────────────────────────────────┘
```

> 💡 **金句**:**能不分库分表就不分**。在分库分表之前,先用读写分离、缓存、归档、SSD 升级、参数调优、分区表等手段。

---

## 十一、运维实战:百万级数据删除

### 11.1 直接 DELETE 的灾难

```sql
-- ❌ 灾难性写法
DELETE FROM orders WHERE create_time < '2020-01-01';
-- 假设要删 500 万行
```

**问题**:
1. **大事务**:undo log 巨大,回滚段爆炸
2. **锁表**:其他事务全部阻塞
3. **主从延迟**:从库重放期间延迟暴涨
4. **磁盘空间不释放**:DELETE 是逻辑删除
5. **可能崩溃**:undo log / binlog cache 撑爆

### 11.2 正确姿势:分批删除

```sql
-- ✅ 分批删除,每批 1000 行
DELETE FROM orders 
WHERE create_time < '2020-01-01' 
LIMIT 1000;

-- 循环执行,直到影响行数 = 0
-- 每批之间 sleep 0.5 秒,给主从同步喘息
```

**Shell 脚本示例**:

```bash
#!/bin/bash
while true; do
    rows=$(mysql -e "DELETE FROM orders WHERE create_time < '2020-01-01' LIMIT 1000;" -BN | wc -l)
    affected=$(mysql -e "SELECT ROW_COUNT();" -BN)
    if [ "$affected" -eq 0 ]; then
        break
    fi
    echo "Deleted $affected rows"
    sleep 0.5
done
```

### 11.3 终极方案:重建表

数据要 **删除 90% 以上**,直接重建更快:

```sql
-- 步骤 1:建新表(同结构)
CREATE TABLE orders_new LIKE orders;

-- 步骤 2:导入要保留的数据
INSERT INTO orders_new 
SELECT * FROM orders WHERE create_time >= '2020-01-01';

-- 步骤 3:重命名(原子操作)
RENAME TABLE orders TO orders_old, orders_new TO orders;

-- 步骤 4:验证后删除旧表
DROP TABLE orders_old;
```

**优点**:
- 速度极快
- 自动回收磁盘空间
- 顺便重建索引(碎片清零)

**注意**:RENAME 期间会有短暂表锁(毫秒级),但 INSERT 阶段会有延迟,建议在低峰期。

### 11.4 大批量 UPDATE 同理

```sql
-- ❌ 灾难
UPDATE user SET status = 0 WHERE last_login < '2023-01-01';

-- ✅ 分批
UPDATE user SET status = 0 
WHERE last_login < '2023-01-01' AND status != 0
LIMIT 1000;
```

---

## 十二、运维实战:千万级大表加字段

### 12.1 直接 ALTER 的问题

```sql
-- ❌ 千万级表直接执行
ALTER TABLE orders ADD COLUMN remark VARCHAR(200);
```

**问题**:
1. **MDL 写锁**:DDL 期间业务全阻塞
2. **耗时长**:可能几十分钟
3. **磁盘暴涨**:重建表需要一倍空间
4. **主从延迟**:从库重放也要这么久

### 12.2 MySQL 8 的 Instant DDL

MySQL 8 对部分 DDL 支持 **秒级完成**:

```sql
-- 加列(默认值不依赖现有数据)
ALTER TABLE orders ADD COLUMN remark VARCHAR(200) DEFAULT NULL, ALGORITHM=INSTANT;

-- 改列名
ALTER TABLE orders RENAME COLUMN old_name TO new_name, ALGORITHM=INSTANT;
```

**支持的操作**:
- 加列(末尾)
- 改默认值
- 重命名列(8.0.18+)
- 改索引可见性

**不支持的**:
- 加索引(用 INPLACE)
- 修改列类型
- 主键变更

### 12.3 Online DDL(MySQL 5.6+)

```sql
ALTER TABLE orders ADD INDEX idx_status(status), ALGORITHM=INPLACE, LOCK=NONE;
```

**ALGORITHM 选项**:
- `COPY`:旧版本,复制全表(锁表)
- `INPLACE`:就地修改,**不复制**
- `INSTANT`:仅元数据修改

**LOCK 选项**:
- `NONE`:不锁表(读写都允许)
- `SHARED`:允许读不允许写
- `EXCLUSIVE`:全锁

### 12.4 终极方案:gh-ost(GitHub 出品)

**原理**:基于 binlog 的无锁在线变更

```
1. 创建影子表(目标结构)
2. 拷贝原表数据到影子表(分批)
3. 应用原表的 binlog 增量到影子表
4. 切换:RENAME 原表和影子表
```

**特点**:
- ✅ **真正零锁**(只在 RENAME 时极短)
- ✅ 可暂停、可回退
- ✅ 不依赖触发器,影响小
- ✅ 主从延迟可控

**使用示例**:

```bash
gh-ost \
  --host=127.0.0.1 \
  --user=root \
  --password=*** \
  --database=mydb \
  --table=orders \
  --alter="ADD COLUMN remark VARCHAR(200)" \
  --execute
```

### 12.5 pt-online-schema-change(Percona)

类似 gh-ost,**基于触发器**实现。

```bash
pt-online-schema-change \
  --alter "ADD COLUMN remark VARCHAR(200)" \
  D=mydb,t=orders \
  --execute
```

**对比 gh-ost**:
- pt-osc 用触发器,对源表性能略有影响
- gh-ost 不用触发器,更轻量,主流推荐

---

## 十三、运维实战:CPU 飙升排查

### 13.1 排查 SOP

**第一步:确认是 MySQL 进程导致的**

```bash
top
# 看 mysqld 进程的 CPU%
```

**第二步:看当前正在执行的 SQL**

```sql
-- 看所有连接
SHOW PROCESSLIST;

-- 只看正在执行的(状态非 Sleep)
SELECT * FROM information_schema.PROCESSLIST 
WHERE COMMAND != 'Sleep' 
ORDER BY TIME DESC;
```

关注:
- **Time 字段**:执行时间长的 SQL
- **State 字段**:Sending data / Copying to tmp table / Locked 都是问题点

**第三步:杀掉异常 SQL(应急)**

```sql
KILL <connection_id>;
```

**第四步:看 InnoDB 状态**

```sql
SHOW ENGINE INNODB STATUS\G
```

关注:
- **TRANSACTIONS** 段:有没有长事务
- **ROW OPERATIONS** 段:行操作速率
- **LATEST DETECTED DEADLOCK**:最近死锁

**第五步:慢日志分析**

```bash
# 实时观察
tail -f /var/log/mysql/slow.log

# 汇总分析
pt-query-digest /var/log/mysql/slow.log
```

### 13.2 常见 CPU 飙升原因

**1. 索引失效或缺失**

```sql
-- 看慢日志,定位 EXPLAIN type=ALL 的 SQL
EXPLAIN SELECT * FROM orders WHERE description LIKE '%xxx%';
```

加索引或优化 SQL。

**2. 大事务长期持锁**

```sql
-- 找长事务
SELECT * FROM information_schema.innodb_trx 
WHERE TIME_TO_SEC(TIMEDIFF(NOW(), trx_started)) > 60;
```

KILL 或等其结束。

**3. 死锁频繁**

```sql
SHOW ENGINE INNODB STATUS\G
-- 看 LATEST DETECTED DEADLOCK
```

应用层加重试,或调整加锁顺序。

**4. 缓存失效雪崩**

Redis 挂了,流量全打到 MySQL → 应急加缓存或限流。

**5. 突发慢查询**

业务上线了新查询没走索引,或数据分布变化让原索引失效。

```sql
-- 强制重新统计
ANALYZE TABLE orders;
```

**6. 全表扫 + 高并发**

`SELECT * FROM big_table`(没 LIMIT)被高并发调用,瞬间撑爆 CPU。

### 13.3 应急处理流程

```
1. 立即降级:开启限流,挡掉部分流量
2. KILL 异常 SQL:快速恢复
3. 看 SHOW PROCESSLIST:定位罪魁祸首
4. 紧急优化:加索引 / 改 SQL / 加缓存
5. 复盘:慢日志 + EXPLAIN + 业务侧验证
```

### 13.4 实战:一段排查脚本

```bash
#!/bin/bash
# mysql_diag.sh

echo "=== Top SQL by Time ==="
mysql -e "SELECT id,user,host,db,command,time,state,LEFT(info,100) AS sql_snippet 
          FROM information_schema.PROCESSLIST 
          WHERE COMMAND != 'Sleep' 
          ORDER BY time DESC LIMIT 10;"

echo "=== Long Transactions ==="
mysql -e "SELECT trx_id, trx_state, trx_started, 
                 TIME_TO_SEC(TIMEDIFF(NOW(), trx_started)) AS duration_sec,
                 trx_mysql_thread_id, LEFT(trx_query, 100) AS query
          FROM information_schema.innodb_trx
          WHERE TIME_TO_SEC(TIMEDIFF(NOW(), trx_started)) > 30
          ORDER BY duration_sec DESC;"

echo "=== Lock Waits ==="
mysql -e "SELECT * FROM performance_schema.data_lock_waits LIMIT 5;"
```

---

## 十四、SQL 经典面试题

### 14.1 题 1:三列表的查询(部门-员工)

> 一张表 `employee(id, name, age, salary, dept_id)`,查询每个部门工资最高的员工

**方案 1:子查询(经典)**

```sql
SELECT e.*
FROM employee e
WHERE salary = (
    SELECT MAX(salary) FROM employee 
    WHERE dept_id = e.dept_id
);
```

**方案 2:窗口函数(MySQL 8 推荐)**

```sql
SELECT * FROM (
    SELECT *, RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rk
    FROM employee
) t
WHERE rk = 1;
```

**变种:每个部门工资前 3 名**

```sql
SELECT * FROM (
    SELECT *, DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rk
    FROM employee
) t
WHERE rk <= 3;
```

### 14.2 题 2:连续登录用户

> 查询连续登录 3 天及以上的用户

```sql
-- 表 login_log(user_id, login_date)

-- 思路:把"日期 - 行号"作为分组键,连续日期会得到相同的差值
SELECT user_id, MIN(login_date) AS start_date, MAX(login_date) AS end_date, COUNT(*) AS days
FROM (
    SELECT user_id, login_date,
           DATE_SUB(login_date, INTERVAL ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) DAY) AS grp
    FROM login_log
    GROUP BY user_id, login_date
) t
GROUP BY user_id, grp
HAVING COUNT(*) >= 3;
```

### 14.3 题 3:索引创建

> 表 `user(id, name, age, city, phone)`,以下查询如何建索引?
> 1. `WHERE name = ?`
> 2. `WHERE name = ? AND age = ?`
> 3. `WHERE city = ? ORDER BY age`
> 4. `WHERE phone = ?`(几乎唯一)

**建议方案**:

```sql
-- 联合索引,覆盖前 2 个查询
ALTER TABLE user ADD INDEX idx_name_age(name, age);

-- city + age 索引,加速排序
ALTER TABLE user ADD INDEX idx_city_age(city, age);

-- phone 唯一索引
ALTER TABLE user ADD UNIQUE INDEX uk_phone(phone);
```

**思路**:
- 高区分度字段在前
- 排序字段放联合索引尾部(避免 filesort)
- 唯一字段建唯一索引

### 14.4 题 4:深分页优化

> `SELECT * FROM orders ORDER BY id LIMIT 1000000, 10` 怎么优化?

**问题**:扫描 100 万行,丢弃,只保留 10 行 → 浪费严重。

**优化方案**:

```sql
-- 方案 1:子查询(走主键索引,只扫主键)
SELECT * FROM orders WHERE id >= (
    SELECT id FROM orders ORDER BY id LIMIT 1000000, 1
) LIMIT 10;

-- 方案 2:游标分页(最优,适合连续翻页)
SELECT * FROM orders WHERE id > <last_id> ORDER BY id LIMIT 10;
-- 客户端记住每次的 last_id,翻下一页传过来

-- 方案 3:延迟关联
SELECT o.* FROM orders o
INNER JOIN (
    SELECT id FROM orders ORDER BY id LIMIT 1000000, 10
) t ON o.id = t.id;
```

**为什么方案 1/3 快?** 子查询里只扫主键索引(覆盖索引),不回表,扫描成本低 10 倍。

**业务建议**:**禁止深分页**,前端翻页限制在 100 页内,超出引导用 "搜索 + 筛选" 替代。

### 14.5 题 5:学生成绩表

> 表 `score(student_id, subject, score)`,查:
> 1. 平均分大于 80 的学生
> 2. 每科都及格(>=60)的学生
> 3. 至少有一科不及格的学生

**1. 平均分大于 80**

```sql
SELECT student_id, AVG(score) AS avg_score
FROM score
GROUP BY student_id
HAVING AVG(score) > 80;
```

**2. 每科都及格**

```sql
-- 方案 1:HAVING + MIN
SELECT student_id
FROM score
GROUP BY student_id
HAVING MIN(score) >= 60;

-- 方案 2:NOT EXISTS
SELECT DISTINCT student_id 
FROM score s1
WHERE NOT EXISTS (
    SELECT 1 FROM score s2 
    WHERE s2.student_id = s1.student_id AND s2.score < 60
);
```

**3. 至少一科不及格**

```sql
SELECT DISTINCT student_id
FROM score
WHERE score < 60;
```

### 14.6 题 6:行转列(Pivot)

> 把 `score(student_id, subject, score)` 变成 `student_id, math, english, chinese`

```sql
SELECT 
    student_id,
    MAX(CASE WHEN subject = 'math' THEN score END) AS math,
    MAX(CASE WHEN subject = 'english' THEN score END) AS english,
    MAX(CASE WHEN subject = 'chinese' THEN score END) AS chinese
FROM score
GROUP BY student_id;
```

**MySQL 8 也可以用 JSON 聚合**:

```sql
SELECT 
    student_id, 
    JSON_OBJECTAGG(subject, score) AS scores
FROM score
GROUP BY student_id;
```

### 14.7 题 7:查询第二高的薪水

```sql
-- 方案 1:LIMIT + OFFSET
SELECT DISTINCT salary FROM employee 
ORDER BY salary DESC LIMIT 1 OFFSET 1;

-- 方案 2:子查询(没有第二高时返回 NULL)
SELECT (
    SELECT DISTINCT salary FROM employee 
    ORDER BY salary DESC LIMIT 1 OFFSET 1
) AS second_highest;

-- 方案 3:子查询排除最大
SELECT MAX(salary) FROM employee 
WHERE salary < (SELECT MAX(salary) FROM employee);
```

---

## 十五、核心知识点总结

### 15.1 一图流(全景图)

```
┌──────────────────────────────────────────────────┐
│           MySQL 高可用与运维体系                    │
├──────────────────────────────────────────────────┤
│  读写分离                                          │
│   主库写 + 多从库读                                 │
│   实现:JDBC 嵌入 / 中间件代理 / 应用层               │
│   主从复制:binlog → IO 线程 → relay log → SQL 线程  │
│   延迟处理:并行复制 + 关键查询走主库                  │
├──────────────────────────────────────────────────┤
│  分库分表                                           │
│   垂直:按业务/字段拆 │ 水平:按行/数据切片            │
│   分片策略:Range / Hash / 一致性哈希 / 路由表        │
│   扩容方案:成倍扩容 / 一致性哈希 / 预分片 / 基因法     │
│   中间件:ShardingSphere / MyCat / Vitess           │
│   代价:跨库 JOIN / 分布式事务 / 全局 ID / 分页        │
├──────────────────────────────────────────────────┤
│  运维实战                                           │
│   大批量删除:分批 + 重建表                           │
│   大表加字段:Online DDL / gh-ost / pt-osc          │
│   CPU 飙升:PROCESSLIST → KILL → 慢日志 → 优化       │
├──────────────────────────────────────────────────┤
│  SQL 实战                                          │
│   分组取 TopN:窗口函数                              │
│   连续日期:行号 + 日期差                             │
│   深分页:延迟关联 / 游标分页                          │
│   行转列:CASE WHEN + GROUP BY                      │
└──────────────────────────────────────────────────┘
```

### 15.2 高频面试题(自查清单)

**读写分离与主从**
- [ ] 什么是读写分离?为什么需要?
- [ ] 读写分离的实现方式有哪几种?
- [ ] 主从复制原理?三个线程分别做什么?
- [ ] binlog 三种格式的区别?
- [ ] 主从延迟怎么排查?怎么解决?
- [ ] 半同步复制是什么?

**分库分表**
- [ ] 什么时候需要分库分表?
- [ ] 垂直拆分和水平拆分的区别?
- [ ] 分片策略有哪些?各自优缺点?
- [ ] 怎么选分片键?
- [ ] 不停机扩容怎么实现?
- [ ] 常用的分库分表中间件?
- [ ] 分库分表会带来哪些问题?

**运维**
- [ ] 大批量数据删除怎么做?
- [ ] 千万级表加字段怎么操作?
- [ ] gh-ost 和 pt-osc 区别?
- [ ] CPU 飙升怎么排查?
- [ ] 怎么定位慢 SQL?

**SQL 题**
- [ ] 分组取 TopN
- [ ] 连续登录天数
- [ ] 深分页优化
- [ ] 行转列 / 列转行
- [ ] 第 N 高薪水

### 15.3 实战经验汇总

**架构演进路径**

```
单库 → 主从读写分离 → 缓存层 → 垂直分库 → 水平分库分表 → 多活
 ↑                                                      ↑
 创业期                                              亿级用户
```

**演进原则**:
1. 加机器(纵向扩容)永远是最快的
2. 加缓存比加机器有效
3. 分库分表是最后的手段
4. 不要过早优化

**生产高可用方案对比**

| 方案 | 一致性 | 切换时间 | 复杂度 | 推荐度 |
|------|-------|---------|-------|-------|
| **主从 + Keepalived** | 弱 | 30s+ | 低 | ⭐⭐⭐ |
| **MHA** | 弱-中 | 10-30s | 中 | ⭐⭐⭐⭐(经典) |
| **MGR** (组复制) | 强 | 秒级 | 高 | ⭐⭐⭐⭐ |
| **Orchestrator** | 中 | 秒级 | 中 | ⭐⭐⭐⭐⭐(GitHub) |
| **PXC** (Percona) | 强 | 秒级 | 高 | ⭐⭐⭐⭐ |

**容量评估经验**

| 指标 | 单库阈值 |
|------|---------|
| QPS | 5000 |
| TPS | 1000 |
| 单表行数 | 1000-2000 万 |
| 数据量 | 50 GB |
| 连接数 | 500 |

超过即考虑读写分离 / 分库分表。

### 15.4 推荐资源

📚 **必读**
- 《高性能 MySQL》(第 4 版) ⭐⭐⭐⭐⭐
- 《MySQL 是怎样运行的》—— 小孩子 4919
- 极客时间《MySQL 实战 45 讲》—— 林晓斌
- 《MySQL 运维内参》—— 周彦伟

🌐 **进阶**
- ShardingSphere 官方文档
- gh-ost 官方文档
- 阿里数据库内核团队博客

🛠 **常用工具**
- pt-toolkit:Percona 工具集
- gh-ost:GitHub 在线 DDL
- ShardingSphere:分库分表中间件
- MHA / Orchestrator:主从故障切换
- DataX:数据迁移
- pt-heartbeat:精确测主从延迟

---

## 📌 Day 25 总结(MySQL 系列完结)

| 维度 | 收获 |
|------|------|
| **读写分离** | 看透主从复制三线程,知道延迟成因与处理思路 |
| **分库分表** | 掌握分片策略选型 + 在线扩容方案 |
| **架构设计** | 理解分库分表的代价,不滥用,**该上才上** |
| **运维实战** | 大数据删除、加字段、CPU 飙升,**生产 SOP** |
| **SQL 实战** | 窗口函数、深分页、行转列等高频题型 |

> 至此,MySQL 五连发(Day 21 → Day 25)完结。从基础语法、底层架构、索引锁事务,到高可用与运维实战,**完整覆盖了 80% 以上的 MySQL 面试与生产场景**。

---

## 🎯 MySQL 系列回顾

| Day | 主题 | 关键词 |
|-----|------|-------|
| **Day 22** | MySQL 基础与架构 | 数据类型、SQL 执行顺序、Server 层架构 |
| **Day 23** | 存储引擎、日志、SQL 优化、索引 | InnoDB、Buffer Pool、redo/binlog、B+ Tree |
| **Day 24** | 索引深入、锁、事务、MVCC | 临键锁、ACID、隔离级别、ReadView |
| **Day 25** | 读写分离、分库分表、运维实战 | 主从复制、分片策略、扩容、Online DDL |

> **下一站**:Redis / 消息队列 / 分布式系统 / 微服务... 八股之路,不止于此。

**Day 25 ✅ Done — MySQL 系列完结!欢迎 Star ⭐**
