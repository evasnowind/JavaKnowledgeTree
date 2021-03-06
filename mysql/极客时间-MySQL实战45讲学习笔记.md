# 极客时间-MySQL实战45讲学习笔记

[TOC]

## 01 | 基础架构：一条SQL查询语句是如何执行的？

SQL执行示例
```
mysql> select * from T where ID=10；
```

MySQL分为：
- Server层
  - 包括连接器、查询缓存、分析器、优化器、执行器等，涵盖 MySQL 的大多数核心服务功能，以及所有的内置函数（如日期、时间、数学和加密函数等），所有跨存储引擎的功能都在这一层实现，比如存储过程、触发器、视图等。
- 存储引擎
  - 插件式设计，常见：InnoDB, MyISAM, Memory
  - 从5.5 开始默认为InnoDB
  - create table时可以指定存储引擎


### 连接器
查看当前已创建的连接：
```
show processlist
```
客户端如果太长时间没动静，连接器就会自动将它断开。这个时间是由参数 wait_timeout 控制的，默认值是 8 小时。

数据库里面，长连接是指连接成功后，如果客户端持续有请求，则一直使用同一个连接。短连接则是指每次执行完很少的几次查询就断开连接，下次查询再重新建立一个。
建立连接的过程通常是比较复杂的，所以我建议你在使用中要尽量减少建立连接的动作，也就是尽量使用长连接。

但全部使用长连接后，你可能会发现，有些时候 MySQL 占用内存涨得特别快
——MySQL 在执行过程中临时使用的内存是管理在连接对象里面的，连接断开时才会释放。如果长连接累积下来，可能导致内存占用太大，被系统强行杀掉（OOM），从现象看就是 MySQL 异常重启了。

解决长连接导致OOM、继而导致MySQL异常重启的方法：
- 定期断开长连接。
- 如果你用的是 MySQL 5.7 或更新版本，可以在每次执行一个比较大的操作后，通过执行 mysql_reset_connection 来重新初始化连接资源。这个过程不需要重连和重新做权限验证，但是会将连接恢复到刚刚创建完时的状态。


### 查询缓存
大多数情况下我会建议你不要使用查询缓存，因为查询缓存往往弊大于利。
查询缓存的失效非常频繁，只要有对一个表的更新，这个表上所有的查询缓存都会被清空。除非你的业务就是有一张静态表，很长时间才会更新一次。比如，一个系统配置表，那这张表上的查询才适合使用查询缓存。
MySQL 也提供了这种“按需使用”的方式。你可以将参数 query_cache_type 设置成 DEMAND，这样对于默认的 SQL 语句都不使用查询缓存。而对于你确定要使用查询缓存的语句，可以用 SQL_CACHE 显式指定，像下面这个语句一样：

```
mysql> select SQL_CACHE * from T where ID=10；
```
8.0 开始彻底没有查询缓存功能。

### 分析器
### 优化器
### 执行器
在数据库的慢查询日志中看到一个 rows_examined 的字段，表示这个语句执行过程中扫描了多少行。这个值就是在执行器每次调用引擎获取数据行的时候累加的。在有些场景下，执行器调用一次，在引擎内部则扫描了多行，因此引擎扫描行数跟 rows_examined 并不是完全相同的。


## 02 | 日志系统：一条SQL更新语句是如何执行的？
在一个表上有更新的时候，跟这个表有关的查询缓存会失效，所以这条语句就会把表 T 上所有缓存结果都清空。

### redo log 重做日志
**WAL 技术**：Write-Ahead Logging，先写日志，再写磁盘
InnoDB 引擎就会先把记录写到 redo log，更新内存，然后在适当的时候（系统较为空闲时）更新磁盘记录。
InnoDB 的 redo log 是固定大小的，结构如下图：
![02-redo-log.png](images/02-redo-log.png)
redo log保证了InnoDB存储引擎在发生异常、重启时，之前提交的记录不会丢失——crash-safe能力

### binlog 归档日志
MySQL的结构:
- server层
  - binlog，只用于归档
- 存储引擎层
  - MySQL 自带的引擎是 MyISAM没有redo log，InnoDB是以插件形式集成进来，InnoDB自己为保证crash-safe才实现了redo log

binlog与redo log区别：
- redo log是InnoDB特有，binlog在MySQL server层，所有引擎可用
- redo log是物理日志，binlog是逻辑日志
- redo log循环写入，binlog追加写入

Redo log不是记录数据页“更新之后的状态”，而是记录这个页 “做了什么改动”。
Binlog有两种模式，statement 格式的话是记sql语句， row格式会记录行的内容，记两条，更新前和更新后都有。

### binlog不能去掉
- redo log只有InnoDB有，别的引擎没有。
- redolog是循环写的，不持久保存，binlog的“归档”这个功能，redolog是不具备的。

其核心就是， redo log 记录的，即使异常重启，都会刷新到磁盘，而 bin log 记录的， 则主要用于备份。

### 为什么日志需要“两阶段提交”
redo log 和 binlog 都可以用于表示事务的提交状态，而两阶段提交就是让这两个状态保持逻辑上的一致。

## 03 | 事务隔离：为什么你改了我还看不见？

### 隔离性与隔离级别
ACID（Atomicity、Consistency、Isolation、Durability，即原子性、一致性、隔离性、持久性）

多个事务同时执行，可能出现：
脏读（dirty read）、不可重复读（non-repeatable read）、幻读（phantom read）的问题
——隔离级别

SQL 标准的事务隔离级别包括：
- 读未提交（read uncommitted）
- 读提交（read committed）
- 可重复读（repeatable read）
  - 事务在执行期间看到的数据前后必须是一致的
- 串行化（serializable ）
隔离级别越高，效率越低，需要权衡。

Oracle 数据库的默认隔离级别是“读提交”
MySQL默认隔离级别是“可重复读”
——从Oracle迁移到MySQL，为保证数据库隔离级别的一致，你一定要记得将 MySQL 的隔离级别设置为“读提交”，即将启动参数 transaction-isolation 的值设置成 READ-COMMITTED
查看方式：
```
show variables like 'transaction_isolation';
```
### 事务隔离的实现

为何建议尽量不使用长事务？
- 对回滚段的影响
- 占用锁资源
- 可能拖垮整个库

参考文章：[MySQL-长事务详解](https://www.cnblogs.com/kunjian/p/11552646.html)


### 事务的启动方式
MySQL 的事务启动方式有以下几种：
1. 显式启动：begin 或 start transaction。配套的提交语句是 commit，回滚语句是 rollback。
2. set autocommit=0，这个命令会将这个线程的自动提交关掉。意味着如果你只执行一个 select 语句，这个事务就启动了，而且并不会自动提交。这个事务持续存在直到你主动执行 commit 或 rollback 语句，或者断开连接。

建议总是使用 set autocommit=1, 通过显式语句的方式来启动事务。

若纠结“多一次交互”的问题，希望减少交互次数，则建议使用**commit work and chain 语法**

可以在 information_schema 库的 innodb_trx 这个表中查询长事务
如下面这个语句，用于查找持续时间超过 60s 的事务：
```
select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started))>60
```

### 参考资料
- [MySQL-长事务详解](https://www.cnblogs.com/kunjian/p/11552646.html)


## 04 | 深入浅出索引（上）
### 索引的常见模型：
- 哈希表：适用于只有等值查询的场景，比如 Memcached 及其他一些 NoSQL 引擎。
- 有序数组：在等值查询和范围查询场景中的性能就都非常优秀，只适用于静态存储引擎
- N 叉树
- 其他：跳表、LSM 树等

### InnoDB 的索引模型
B+ 树
根据叶子节点的内容，索引类型分为：主键索引和非主键索引
主键索引的叶子节点存的是整行数据。在 InnoDB 里，主键索引也被称为聚簇索引（clustered index）。
非主键索引的叶子节点内容是主键的值。在 InnoDB 里，非主键索引也被称为二级索引（secondary index）。
基于主键索引和普通索引的查询有什么区别？
——基于非主键索引的查询需要多扫描一棵索引树。因此，我们在应用中应该尽量使用主键查询。

### 索引维护
选择自增字段作为主键，还是采用业务字段作为主键？
主键长度越小，普通索引的叶子节点就越小，普通索引占用的空间也就越小。从性能和存储空间方面考量，自增主键往往是更合理的选择。

有些业务的场景需求是这样的：只有一个索引，且该索引必须是唯一索引（KV场景）
——没有其他索引，不用考虑其他索引叶子节点大小问题，直接用业务字段即可。



## 05 | 深入浅出索引（下）
回到主键索引树搜索的过程，我们称为回表

### 覆盖索引
```
select * from T where k between 3 and 5
```
需要回表。

```
select ID from T where k between 3 and 5
```
覆盖索引
**由于覆盖索引可以减少树的搜索次数，显著提升查询性能，所以使用覆盖索引是一个常用的性能优化手段。**

应用：
市民信息表中有姓名、身份证号字段：
针对利用身份证号查询市民信息：索引仅建在身份证号即可
若有利用身份证号查姓名的高频需求：额外创建（身份证号、姓名）的联合索引，避免回表，直接覆盖索引

### 最左前缀原则
**在建立联合索引的时候，如何安排索引内的字段顺序。**
评估标准是，索引的复用能力
- **第一原则是，如果通过调整顺序，可以少维护一个索引，那么这个顺序往往就是需要优先考虑采用的**
  - 当已经有了 (a,b) 这个联合索引后，一般就不需要单独在 a 上建立索引了
- 索引占用空间

### 索引下推
MySQL 5.6 引入索引下推优化（index condition pushdown)， 可以在索引遍历过程中，对索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数。


## 06 | 全局锁和表锁 ：给表加个字段怎么有这么多阻碍？
根据加锁的范围，MySQL 里面的锁大致可以分成全局锁、表级锁和行锁三类。

### 全局锁
全局锁就是对整个数据库实例加锁。MySQL 提供了一个加全局读锁的方法，命令是 Flush tables with read lock (FTWRL)。

**全局锁的典型使用场景是，做全库逻辑备份。**也就是把整库每个表都 select 出来存成文本。
——另一种思路：如果是InnoDB这种支持事务的引擎，可以在可重复读隔离级别下开启一个事务，以便保证备份过程中看到的是一致性视图。MyISAM不支持事务，只能用FTWRL了。

官方自带的逻辑备份工具是 mysqldump。当 **mysqldump** 使用参数**–single-transaction** 的时候，导数据之前就会启动一个事务，来确保拿到一致性视图。
——single-transaction 方法只适用于所有的表使用事务引擎的库。

既然要全库只读，为什么不使用 set global readonly=true 的方式呢？
- 在有些系统中，readonly 的值会被用来做其他逻辑，比如用来判断一个库是主库还是备库。
- 在异常处理机制上有差异。

### 表级锁
MySQL 里面表级别的锁有两种：一种是表锁，一种是元数据锁（meta data lock，MDL)。
表锁的语法是 lock tables … read/write。
需要注意，lock tables 语法除了会限制别的线程的读写外，也限定了本线程接下来的操作对象。

另一类表级的锁是 MDL（metadata lock)。
MDL 不需要显式使用，在访问一个表的时候会被自动加上：读锁之间不互斥，读写锁之间、写锁之间是互斥的，用来保证变更表结构操作的安全性。

事务中的 MDL 锁，在语句执行开始时申请，但是语句结束后并不会马上释放，而会等到整个事务提交后再释放。

**问题：如何安全地给小表加字段？**

## 07 | 行锁功过：怎么减少行锁对性能的影响？
MySQL 的行锁是在引擎层由各个引擎自己实现的。但并不是所有的引擎都支持行锁
**在 InnoDB 事务中，行锁是在需要的时候才加上的，但并不是不需要了就立刻释放，而是要等到事务结束时才释放。这个就是两阶段锁协议。**
——如果你的事务中需要锁多个行，要把最可能造成锁冲突、最可能影响并发度的锁尽量往后放。

### 死锁和死锁检测
数据库死锁时，有两种策略：
- 直接进入等待，直至超过超时时间。超时时间配置innodb_lock_wait_timeout。默认值为50s。
- 发起死锁检测，发现死锁后，主动回滚死锁链条中的某一个事务.参数 innodb_deadlock_detect 设置为 on 则表示开启死锁检测。

怎么解决由这种热点行更新导致的性能问题呢？
——死锁检测要耗费大量的 CPU 资源

## 08 | 事务到底是隔离的还是不隔离的？
事务的启动时机：begin/start transaction 命令并不是一个事务的起点，在执行到它们之后的第一个操作 InnoDB 表的语句，事务才真正启动。如果你想要马上启动一个事务，可以使用 start transaction with consistent snapshot 这个命令。

InnoDB 利用了“所有数据都有多个版本”的这个特性，实现了“秒级创建快照”的能力。

一个数据版本，对于一个事务视图来说，除了自己的更新总是可见以外，有三种情况：
- 版本未提交，不可见；
- 版本已提交，但是是在视图创建后提交的，不可见；
- 版本已提交，而且是在视图创建前提交的，可见。

**更新数据都是先读后写的，而这个读，只能读当前的值，称为“当前读”（current read）。**

除了 update 语句外，select 语句如果加锁，也是当前读。如：查询语句 select * from t where id=1 加上 lock in share mode 或 for update，下面这两个 select 语句，就是分别加了读锁（S 锁，共享锁）和写锁（X 锁，排他锁）。
```
mysql> select k from t where id=1 lock in share mode;
mysql> select k from t where id=1 for update;
```

### 事务的可重复读的能力是怎么实现的？
可重复读的核心就是一致性读（consistent read）；而事务更新数据的时候，只能用当前读。如果当前的记录的行锁被其他事务占用的话，就需要进入锁等待。

InnoDB 的行数据有多个版本，每个数据版本有自己的 row trx_id，每个事务或者语句有自己的一致性视图。普通查询语句是一致性读，一致性读会根据 row trx_id 和一致性视图确定数据版本的可见性。对于可重复读，查询只承认在事务启动前就已经提交完成的数据；对于读提交，查询只承认在语句启动前就已经提交完成的数据；而当前读，总是读取已经提交完成的最新版本。

## 09 | 普通索引和唯一索引，应该怎么选择？
### 查询过程
- 对于普通索引来说，查找到满足条件的第一个记录 (5,500) 后，需要查找下一个记录，直到碰到第一个不满足 k=5 条件的记录。
- 对于唯一索引来说，由于索引定义了唯一性，查找到第一个满足条件的记录后，就会停止继续检索。
上述区别带来的性能差距微乎其微。

### 更新过程
**change buffer**：当需要更新一个数据页时，如果数据页在内存中就直接更新，而如果这个数据页还没有在内存中的话，在不影响数据一致性的前提下，InooDB 会将这些更新操作缓存在 change buffer 中，这样就不需要从磁盘中读入这个数据页了。

change buffer 在内存中有拷贝，也会被写入到磁盘上。

将 change buffer 中的操作应用到原数据页，得到最新结果的过程称为 merge。除了访问这个数据页会触发 merge 外，系统有后台线程会定期 merge。在数据库正常关闭（shutdown）的过程中，也会执行 merge 操作。

**什么条件下可以使用 change buffer 呢？**
对于唯一索引来说，所有的更新操作都要先判断这个操作是否违反唯一性约束。
唯一索引的更新就不能使用 change buffer，实际上也只有普通索引可以使用。

change buffer 用的是 buffer pool 里的内存，因此不能无限增大。change buffer 的大小，可以通过参数 innodb_change_buffer_max_size 来动态设置。这个参数设置为 50 的时候，表示 change buffer 的大小最多只能占用 buffer pool 的 50%。

例子：在表中插入一个新记录，InnoDB处理流程：
- 这个记录要更新的目标页在内存中：唯一索引查询到之后，需要判断是否唯一，多一次判断，其他操作和普通索引一致，因而性能差异很小。
- 这个记录要更新的目标页不在内存中：唯一索引需要将数据页写入内存，判断是否冲突，然后写入；普通索引将更新记录在 change buffer，语句执行就结束了。
将数据从磁盘读入内存涉及随机 IO 的访问，是数据库里面成本最高的操作之一。change buffer 因为减少了随机磁盘访问，所以对更新性能的提升是会很明显的。

**change buffer 的使用场景**
change buffer 只限于用在普通索引的场景下，而不适用于唯一索引。
因为 merge 的时候是真正进行数据更新的时刻，而 change buffer 的主要目的就是将记录的变更动作缓存下来，所以在一个数据页做 merge 之前，change buffer 记录的变更越多（也就是这个页面上要更新的次数越多），收益就越大。

因此，对于写多读少的业务来说，change buffer 的使用效果最好。这种业务模型常见的就是账单类、日志类的系统。
反过来，假设一个业务的更新模式是写入之后马上会做查询，那么即使满足了条件，将更新先记录在 change buffer，但之后由于马上要访问这个数据页，会立即触发 merge 过程。这样随机访问 IO 的次数不会减少，反而增加了 change buffer 的维护代价。所以，对于这种业务模式来说，change buffer 反而起到了副作用。

### 索引选择和实践
这两类索引在查询能力上是没差别的，主要考虑的是对更新性能的影响。所以，我建议你尽量选择普通索引。
如果所有的更新后面，都马上伴随着对这个记录的查询，那么你应该关闭 change buffer。而在其他情况下，change buffer 都能提升更新性能。

如果要简单地对比这两个机制在提升更新性能上的收益的话，**redo log 主要节省的是随机写磁盘的 IO 消耗（转成顺序写），而 change buffer 主要节省的则是随机读磁盘的 IO 消耗。**


## 10 | MySQL为什么有时候会选错索引？
### 优化器的逻辑
优化器选择索引的目的，是找到一个最优的执行方案，并用最小的代价去执行语句。
考虑因素：扫描行数，还会结合是否使用临时表、是否排序等因素进行综合判断。

**扫描行数是怎么判断的？**
一个索引上不同的值越多，这个索引的区分度就越好。而一个索引上不同的值的个数，我们称之为“基数”（cardinality）。也就是说，这个基数越大，索引的区分度越好。
```
show index from 表名;
```
该命令可以查看区分度（cardinality）

**MySQL 是怎样得到索引的基数的呢？**
采样统计，因而所计算基数并不准确
如：InnoDB 默认会选择 N 个数据页，统计这些页面上的不同值，得到一个平均值，然后乘以这个索引的页面数，就得到了这个索引的基数。而数据表是会持续更新的，索引统计信息也不会固定不变。所以，当变更的数据行数超过 1/M 的时候，会自动触发重新做一次索引统计。

存储索引统计的方式：持久化存储，或是只存在内存中。innodb_stats_persistent参数用于配置该方式。

如果是普通索引，在索引上查询到之后，还需要回到主键索引查出整行数据。
——这个代价优化器也要算进去，即：比较直接用主键索引进行查询的代价，与使用普通索引、然后回表的代价，两者相比较。

如果是统计基数有问题，可以用如下命令重新计算基数，然后再用explain看看是否有改善：
```
analyze table 表名
```
在实践中，如果你发现 explain 的结果预估的 rows 值跟实际情况差距比较大，可以采用这个方法来处理。

### 索引选择异常和处理
大多数时候优化器都能找到正确的索引，但偶尔你还是会碰到我们上面举例的这两种情况：原本可以执行得很快的 SQL 语句，执行速度却比你预期的慢很多，你应该怎么办呢？
- 方法1：采用 force index 强行选择一个索引
  - 需要改写代码，开发、测试、发布，代价较大
- 方法2：可以考虑修改语句，引导 MySQL 使用我们期望的索引
  - 例子：`select * from t where (a between 1 and 1000) and (b between 50000 and 100000) order by b limit 1;` 
  - 改写方法1：把“order by b limit 1” 改成 “order by b,a limit 1”
  - 改写方法2：`select * from  (select * from t where (a between 1 and 1000)  and (b between 50000 and 100000) order by b limit 100)alias limit 1;`
- 方法3：在有些场景下，我们可以新建一个更合适的索引，来提供给优化器做选择，或删掉误用的索引。


## 11 | 怎么给字符串字段加索引？
问题：如何在邮箱这样的字段上加索引？

### 思路1：给邮箱整个字段加索引
### 思路2：使用邮箱前缀加索引

区别：
- 存储上，思路1占用空间大，但可以减少扫描次数；思路2，前缀占用空间少，可能重复，因而可能会导致更多的扫描次数
- 如果查询的是只是"物理id,邮箱地址"，那么思路1不用回表；思路2还是需要回表查询。
  - 此即前缀索引对覆盖索引的影响

**使用前缀索引，定义好长度，就可以做到既节省空间，又不用额外增加太多的查询成本。**
问题：如何确定前缀索引使用多长的前缀？
根据字段区分度，统计索引上有多少个不同的值来判断。

```
mysql> select 
  count(distinct left(email,4)）as L4,
  count(distinct left(email,5)）as L5,
  count(distinct left(email,6)）as L6,
  count(distinct left(email,7)）as L7,
from SUser;
```

当然，使用前缀索引很可能会损失区分度，所以你需要预先设定一个可以接受的损失比例，比如 5%。然后，在返回的 L4~L7 中，找出不小于 L * 95% 的值，假设这里 L6、L7 都满足，你就可以选择前缀长度为 6。


其他方式:
比如类似身份证号的字段，前缀有大量重复，可能是中间、或是后缀有区分度的情况

### 思路3：使用倒序存储
存储身份证号的时候把它倒过来存，查询时，这么查询：
```
mysql> select field_list from t where id_card = reverse('input_id_card_string');
```
注意，也要用count(distinct)的方式验证倒序后，采用前缀索引的区分度。

### 思路4：使用hash字段
可以在表上再创建一个整数字段，来保存身份证的校验码，同时在这个字段上创建索引。
```
mysql> alter table t add id_card_crc int unsigned, add index(id_card_crc);
```

每次插入新记录的时候，都同时用 crc32() 这个函数得到校验码填到这个新字段。
由于校验码可能存在冲突，查询语句 where 部分要判断 id_card 的值是否精确相同。即：
```
mysql> select field_list from t where id_card_crc=crc32('input_id_card_string') and id_card='input_id_card_string'
```

思路3、4都不支持范围索引，只支持等值查询；查询效率上，思路4使用hash的查询性能更稳定，因为hash冲突概率很小


## 12 | 为什么我的MySQL会“抖”一下？
当内存数据页跟磁盘数据页内容不一致的时候，我们称这个内存页为“脏页”。

**WAL技术**
Write-Ahead Logging，它的关键点就是先写日志，再写磁盘。InnoDB 引擎就会先把记录写到 redo log里面，并更新内存，这个时候更新就算完成了。
平时执行很快的更新操作，其实就是在写内存和日志，而 MySQL 偶尔“抖”一下的那个瞬间，可能就是在刷脏页（flush）。

### 什么情况会引发数据库的 flush 过程呢？
- 情况1：InnoDB 的 redo log 写满了。这时候系统会停止所有更新操作，把 checkpoint 往前推进，redo log 留出空间可以继续写。
- 情况2：系统内存不足。当需要新的内存页，而内存不够用的时候，就要淘汰一些数据页，空出内存给别的数据页使用。如果淘汰的是“脏页”，就要先将脏页写到磁盘。
- 情况3：MySQL 认为系统“空闲”的时候。
- 情况4：MySQL 正常关闭的情况。

情况3是空闲时候，不会影响业务，不用管；
情况4即将关闭肯定没有什么业务请求，不用管；
情况1会堵塞更新，需要尽量避免；
情况2是常态，InnoDB 用缓冲池（buffer pool）管理内存，缓冲池中的内存页有三种状态：
- 还没有使用
- 已使用，是干净页
- 已使用，是脏页
InnoDB 的策略是尽量使用内存，因此对于一个长时间运行的库来说，未被使用的页面很少。
出现以下这两种情况，都是会明显影响性能的：
1. 一个查询要淘汰的脏页个数太多，会导致查询的响应时间明显变长；
2. 日志写满，更新全部堵住，写性能跌为 0，这种情况对敏感业务来说，是不能接受的。

所以，InnoDB 需要有控制脏页比例的机制，来尽量避免上面的这两种情况。

### InnoDB 刷脏页的控制策略
1. 首先需要告知InnoDB所在主机的IO能力
   配置innodb_io_capacity参数，建议设置为磁盘的IOPS。磁盘的IOPS可以用fio来测试。比如作者用以下语句测试随机读写性能：
   ```  
    fio -filename=$filename -direct=1 -iodepth 1 -thread -rw=randrw -ioengine=psync -bs=16k -size=500M -numjobs=10 -runtime=10 -group_reporting -name=mytest 
   ```
   innodb_io_capacity 参数配置错误，可能导致上不去。
2. 控制InnoDB刷脏页速度
   如果刷太慢，将导致：内存脏页太多，redo log写满
   要避免该情况，因而刷盘速度需要考虑：一个是脏页比例，一个是 redo log 写盘速度。
   参数 innodb_max_dirty_pages_pct 是脏页比例上限，默认值是 75%。
   ![InnoDB刷脏页速度策略](images/InnoDB刷脏页速度策略.png)

   无论是你的查询语句在需要内存的时候可能要求淘汰一个脏页，还是由于刷脏页的逻辑会占用 IO 资源并可能影响到了你的更新语句，都可能是造成你从业务端感知到 MySQL“抖”了一下的原因。

要尽量避免这种情况，你就要合理地设置 innodb_io_capacity 的值，并且平时要多关注脏页比例，不要让它经常接近 75%。
其中，脏页比例是通过 Innodb_buffer_pool_pages_dirty/Innodb_buffer_pool_pages_total 得到的，具体的命令参考下面的代码：
```

mysql> select VARIABLE_VALUE into @a from global_status where VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty';
select VARIABLE_VALUE into @b from global_status where VARIABLE_NAME = 'Innodb_buffer_pool_pages_total';
select @a/@b;
```

MySQL 中有一个机制：在准备刷一个脏页的时候，如果这个数据页旁边的数据页刚好是脏页，就会把这个“邻居”也带着一起刷掉；而且这个把“邻居”拖下水的逻辑还可以继续蔓延。
——可能导致刷脏页时查询更慢。
在 InnoDB 中，innodb_flush_neighbors 参数就是用来控制这个行为的，值为 1 的时候会有上述的“连坐”机制，值为 0 时表示不找邻居，自己刷自己的。
机械硬盘下这种机制可以减少随机IO，提高性能，建议设置为1；SSD则没必要，建议设置为0.MySQL 8中已经默认置为0.


## 13 | 为什么表数据删掉一半，表文件大小不变？
### 直接删除表数据无法减少占用空间的原因
表数据既可以存在共享表空间里，也可以是单独的文件。这个行为是由参数 innodb_file_per_table 控制的：
- OFF 表示的是，表的数据放在系统共享表空间，也就是跟数据字典放在一起；
- ON表示每个 InnoDB 表数据存储在一个以 .ibd 为后缀的文件中。

建议不论使用 MySQL 的哪个版本，都将这个值设置为 ON。
——一个表单独存一个文件更方便管理，并且drop table命令删除表时，ON的情况可以直接删除数据、回收空间，OFF则不会回收空间。

将 innodb_file_per_table 设置为 ON，是推荐做法，以下都是基于该设置展开。
在删除整个表的时候，可以使用 drop table 命令回收表空间。但有些时候并不是删除所有、只删除某些行，此时会遇到：
**表中的数据被删除了，但是表空间却没有被回收。**

原因1：删除（delete）操作造成数据“空洞”。以InnoDB为例，存储采用B+树，删除时只是标记为可复用、数据并未真正被删除。
InnoDB按页存储数据，若删除某页上的所有数据，则整个数据页可以复用、但同样没有被真正删除。
原因2：插入操作同样会造成数据“空洞”。B+树，新增数据，可能造成叶子分裂，分裂后可能有空洞。
原因3：更新操作同样会造成数据“空洞”。更新相当于删除一个旧值、插入一个新值。同原因1、2所解释。

### 收缩表空间的操作：重建表
重建表可以减少减少空洞、回收空间。
可以应用`alter table A engine=InnoDB`命令来重建表。在** MySQL 5.5 版本**之前，这个命令会自动完成转存数据、交换表名、删除旧表的操作。
注意：花时间最多的步骤是往临时表插入数据的过程，该过程中原始表不能有更新，否则可能就存在数据丢失。即DDL操作是离线的。

**MySQL 5.6 版本开始引入的 Online DDL，对这个操作流程做了优化。**：`alter table A engine=InnoDB`命令，对于一个大表来说，Online DDL 最耗时的过程就是拷贝数据到临时表的过程，这个步骤的执行期间可以接受增删改操作。因而算做Online的。
——由于扫描原表、构建临时文件很消耗CPU和IO，需要控制操作时间。推荐采用使用 GitHub 开源的 gh-ost 来做。

### Online和inplace
Online DDL做法，等价于：
```
alter table t engine=innodb,ALGORITHM=inplace;
```
即** MySQL 5.5 版本**之后的版本`alter table A engine=InnoDB`命令在InnoDB内部创建临时文件，在mysql server层没有感知。

另一种则是：
```
alter table t engine=innodb,ALGORITHM=copy;
```
copy即强制拷贝表，即** MySQL 5.5 版本**之前`alter table A engine=InnoDB`命令的实现，即在mysql server层创建表。


## 14 | count(*)这么慢，我该怎么办？
### count(*) 的实现方式
在不同的 MySQL 引擎中，count(*) 有不同的实现方式：
- MyISAM：把一个表的总行数存在了磁盘上，count(*)直接返回这个总行数，效率高
- InnoDB：执行 count(*)时，一行行扫描，累积计数
——上述说的是没有过滤条件的count(*)。加where后MyISAM也要扫描很多行。

**为什么 InnoDB 不跟 MyISAM 一样，也把数字存起来呢？**
这和 InnoDB 的事务设计有关系，可重复读是它默认的隔离级别，在代码上就是通过多版本并发控制，也就是 MVCC 来实现的。每一行记录都要判断自己是否对这个会话可见，因此对于 count(*) 请求来说，InnoDB 只好把数据一行一行地读出依次判断，可见的行才能够用于计算“基于这个查询”的表的总行数。

**在保证逻辑正确的前提下，尽量减少扫描的数据量，是数据库系统设计的通用法则之一。**

- MyISAM 表虽然 count(*) 很快，但是不支持事务；
- show table status 命令虽然返回很快，但是不准确；
- InnoDB 表直接 count(*) 会遍历全表，虽然结果准确，但会导致性能问题。

### 用缓存系统保存计数
将计数保存在缓存系统中的方式，还不只是丢失更新的问题。即使 Redis 正常工作，这个值还是逻辑上不精确的。因为：这两个不同的存储构成的系统，不支持分布式事务，无法拿到精确一致的视图。
因此，需要将计数保存到数据库里，利用数据库事务保证多个操作数据的一致性。

### 不同的 count 用法
按照效率排序的话，`count(字段)<count(主键 id)<count(1)≈count(*)`，所以建议尽量使用 count(*)。





## 15 | 答疑文章（一）：日志和索引相关问题
### 日志相关问题

**在两阶段提交的不同时刻，MySQL 异常重启会出现什么现象。**

如果 redo log 里面的事务是完整的，也就是已经有了 commit 标识，则直接提交；
如果 redo log 里面的事务只有完整的 prepare，则判断对应的事务 binlog 是否存在并完整：
  a. 如果是，则提交事务；
  b. 否则，回滚事务。

- 追问 1：MySQL 怎么知道 binlog 是完整的?
  - 一个事务的 binlog 是有完整格式的：
    - statement 格式的 binlog，最后会有 COMMIT
    - row 格式的 binlog，最后会有一个 XID event
    - 在 MySQL 5.6.2 版本以后，还引入了 binlog-checksum 参数
- 追问 2：redo log 和 binlog 是怎么关联起来的?
  - 有一个共同的数据字段，叫 XID
- 追问 3：处于 prepare 阶段的 redo log 加上完整 binlog，重启就能恢复，MySQL 为什么要这么设计?
  - 在binlog 写完以后 MySQL 发生崩溃，这时候 binlog 已经写入了，之后就会被从库（或者用这个 binlog 恢复出来的库）使用
- 追问 4：如果这样的话，为什么还要两阶段提交呢？干脆先 redo log 写完，再写 binlog。崩溃恢复的时候，必须得两个日志都完整才可以。是不是一样的逻辑？
  - 两阶段提交就是为了给所有人一个机会，当每个人都说“我 ok”的时候，再一起提交。
- 追问 5：不引入两个日志，也就没有两阶段提交的必要了。只用 binlog 来支持崩溃恢复，又能支持归档，不就可以了？
  - 至少现在的 binlog 能力，还不能支持崩溃恢复。
- 追问 6：那能不能反过来，只用 redo log，不要 binlog？
  - 如果只从崩溃恢复的角度来讲是可以的。你可以把 binlog 关掉，这样就没有两阶段提交了，但系统依然是 crash-safe 的。界各个公司的使用场景的话，就会发现在正式的生产库上，binlog 都是开着的。一个是归档。redo log 是循环写，写到末尾是要回到开头继续写的。这样历史日志没法保留，redo log 也就起不到归档的作用。一个就是 MySQL 系统依赖于 binlog。
- 追问 7：redo log 一般设置多大？
  - 如果是现在常见的几个 TB 的磁盘的话，就不要太小气了，直接将 redo log 设置为 4 个文件、每个文件 1GB 吧。
- 追问 8：正常运行中的实例，数据写入后的最终落盘，是从 redo log 更新过来的还是从 buffer pool 更新过来的呢？
- 追问 9：redo log buffer 是什么？是先修改内存，还是先写 redo log 文件？ 



## 16 | “order by”是怎么工作的？

### 全字段排序
sort_buffer_size，就是 MySQL 为排序开辟的内存（sort_buffer）的大小。如果要排序的数据量小于 sort_buffer_size，排序就在内存中完成。但如果排序数据量太大，内存放不下，则不得不利用磁盘临时文件辅助排序。


### rowid 排序
全字段排序算法，如果查询要返回的字段很多的话，那么 sort_buffer 里面要放的字段数太多，这样内存里能够同时放下的行数很少，要分成很多个临时文件，排序的性能会很差。

max_length_for_sort_data，是 MySQL 中控制用于排序的行数据的长度的一个参数。如果单行的长度超过这个值，MySQL 就认为单行太大，要换一个算法。

### 全字段排序 VS rowid 排序

MySQL 的一个设计思想:如果内存够，就要多利用内存，尽量减少磁盘访问。

并不是所有的 order by 语句，都需要排序操作的。从上面分析的执行过程，MySQL 之所以需要生成临时表，并且在临时表上做排序操作，其原因是原来的数据都是无序的。

可以利用创建合适的索引，避免查询过程中生成临时表——只要索引的查询结果直接符合排序要求。

进一步优化：**覆盖索引**
覆盖索引是指，索引上的信息足够满足查询请求，不需要再回到主键索引上去取数据。
——注意权衡！！！：创建索引、覆盖索引所带来的成本。


## 17 | 如何正确地显示随机消息？
英语学习 App 首页有一个随机显示单词的功能，也就是根据每个用户的级别有一个单词表，然后这个用户每次访问首页的时候，都会随机滚动显示三个单词。
如何设计SQL语句？

### 内存临时表
对于内存表，回表过程只是简单地根据数据行的位置，直接访问内存得到数据，根本不会导致多访问磁盘。
order by rand() 使用了内存临时表，内存临时表排序的时候使用了 rowid 排序方法。

### 磁盘临时表
tmp_table_size 这个配置限制了内存临时表的大小，默认值是 16M。如果临时表大小超过了 tmp_table_size，那么内存临时表就会转成磁盘临时表。磁盘临时表使用的引擎默认是 InnoDB，是由参数 internal_tmp_disk_storage_engine 控制的。

### 随机排序方法
如果只随机选择 1 个 word 值，可以怎么做呢？思路上是这样的：
1. 取得这个表的主键 id 的最大值 M 和最小值 N;
2. 用随机函数生成一个最大值到最小值之间的数 X = (M-N)*rand() + N;
3. 取不小于 X 的第一个 ID 的行。

## 18 | 为什么这些SQL语句逻辑相同，性能却差异巨大？
假设表结构如下：
```
mysql> CREATE TABLE `tradelog` (
  `id` int(11) NOT NULL,
  `tradeid` varchar(32) DEFAULT NULL,
  `operator` int(11) DEFAULT NULL,
  `t_modified` datetime DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `tradeid` (`tradeid`),
  KEY `t_modified` (`t_modified`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 案例一：条件字段函数操作

对索引字段做函数操作，可能会破坏索引值的有序性，因此优化器就决定放弃走树搜索功能。
例子：
`select count(*) from tradelog where month(t_modified)=7;`查询效率极低，但用`where t_modified='2018-7-1’`则可以利用到索引、很快返回结果

### 案例二：隐式类型转换

在 MySQL 中，字符串和数字做比较的话，是将字符串转换成数字。
查询语句`select * from tradelog where tradeid=110717;`相当于执行`select * from tradelog where  CAST(tradid AS signed int) = 110717;`
由于对索引字段做函数操作，优化器会放弃走树搜索功能。

### 案例三：隐式字符编码转换
`select * from trade_detail where tradeid=$L2.tradeid.value; `
实际上等价于`select * from trade_detail  where CONVERT(traideid USING utf8mb4)=$L2.tradeid.value; `
连接过程中要求在被驱动表的索引字段上加函数操作，是直接导致对被驱动表做全表扫描的原因。

3个案例，实际上都是说一个事情：对索引字段做函数操作，可能会破坏索引值的有序性，因此优化器就决定放弃走树搜索功能。


## 19 | 为什么我只查一行的语句，也执行这么慢？
### 第一类：查询长时间不返回
- 等 MDL 锁
- 等 flush
- 等行锁

### 第二类：查询慢
**坏查询不一定是慢查询。**


## 20 | 幻读是什么，幻读有什么问题？
### 幻读是什么？

### 幻读有什么问题？
1.  语义被破坏
2.  数据不一致

### 如何解决幻读？
间隙锁和 next-key lock 的引入，帮我们解决了幻读的问题，但同时也带来了一些“困扰”。
间隙锁的引入，可能会导致同样的语句锁住更大的范围，这其实是影响了并发度的。


## 21 | 为什么我只改一行的语句，锁这么多？
两条前提:
1. MySQL 后面的版本可能会改变加锁策略，所以这个规则只限于截止到现在的最新版本，即 5.x 系列 <=5.7.24，8.0 系列 <=8.0.13。
2. 如果大家在验证中有发现 bad case 的话，请提出来，我会再补充进这篇文章，使得一起学习本专栏的所有同学都能受益。

丁奇总结的加锁规则：
- 原则 1：加锁的基本单位是 next-key lock。希望你还记得，next-key lock 是前开后闭区间。
- 原则 2：查找过程中访问到的对象才会加锁。
- 优化 1：索引上的等值查询，给唯一索引加锁的时候，next-key lock 退化为行锁。
- 优化 2：索引上的等值查询，向右遍历时且最后一个值不满足等值条件的时候，next-key lock 退化为间隙锁。
- 一个 bug：唯一索引上的范围查询会访问到不满足条件的第一个值为止。



## 22 | MySQL有哪些“饮鸩止渴”提高性能的方法？
### 短连接风暴
max_connections 参数，用来控制一个 MySQL 实例同时存在的连接数的上限，超过这个值，系统就会拒绝接下来的连接请求，并报错提示“Too many connections”
调高 max_connections参数值？
——有风险，让更多的连接都可以进来，那么系统的负载可能会进一步加大，大量的资源耗费在权限验证等逻辑上，结果可能是适得其反，已经连接的线程拿不到 CPU 资源去执行业务的 SQL 请求。
解决：
1. 第一种方法：先处理掉那些占着连接但是不工作的线程。
kill connection，这个行为跟事先设置 wait_timeout 的效果是一样的。
在 show processlist 的结果里，踢掉显示为 sleep 的线程，可能是有损的。——比如断开的是正在事务中的线程
要看事务具体状态的话，你可以查 information_schema 库的 innodb_trx 表。
从服务端断开连接使用的是 kill connection + id 的命令

2. 第二种方法：减少连接过程的消耗。
如果现在数据库确认是被连接行为打挂了，那么一种可能的做法，是让数据库跳过权限验证阶段。
跳过权限验证的方法是：重启数据库，并使用–skip-grant-tables 参数启动。
——风险极高


### 慢查询性能问题
在 MySQL 中，会引发性能问题的慢查询，大体有以下三种可能：
1. 索引没有设计好；
    重建索引。
2. SQL 语句没写好；
   查询重写
   `select * from t where id + 1 = 10000`
   利用
   `
insert into query_rewrite.rewrite_rules(pattern, replacement, pattern_database) values ("select * from t where id + 1 = ?", "select * from t where id = ? - 1", "db1");
call query_rewrite.flush_rewrite_rules();
   `
3. MySQL 选错了索引。 
  应急方案就是给这个语句加上 force index。
  同样地，使用查询重写功能，给原来的语句加上 force index，也可以解决这个问题。
- 上线前，在测试环境，把慢查询日志（slow log）打开，并且把 long_query_time 设置成 0，确保每个语句都会被记录入慢查询日志；
- 在测试表里插入模拟线上的数据，做一遍回归测试；
- 观察慢查询日志里每类语句的输出，特别留意 Rows_examined 字段是否与预期一致。

### QPS 突增问题



## 23 | MySQL是怎么保证数据不丢的？



## 24 | MySQL是怎么保证主备一致的？
### MySQL 主备的基本原理

备库设置为readonly的原因：
1. 有时候一些运营类的查询语句会被放到备库上去查，设置为只读可以防止误操作；
2. 防止切换逻辑有 bug，比如切换过程中出现双写，造成主备不一致；
3. 可以用 readonly 状态，来判断节点的角色。
——主库同步时用的超级用户权限，不会受readonly影响

### binlog 的三种格式对比
1. statement
2. row
3. mixed：前两种的混合

### 为什么会有 mixed 格式的 binlog？
- 有些 statement 格式的 binlog 可能会导致主备不一致，所以要使用 row 格式。
- row 格式的缺点是，很占空间。
- mysql 采用折中方案：MySQL 自己会判断这条 SQL 语句是否可能引起主备不一致，如果有可能，就用 row 格式，否则就用 statement 格式。


现在越来越多的场景要求把 MySQL 的 binlog 格式设置成 row。
——主要是为了保证恢复数据

在重放 binlog 数据的时候，如果：用 mysqlbinlog 解析出日志，然后把里面的 statement 语句直接拷贝出来执行，是错误的做法！！！
——语句依赖于上下文。

**用 binlog 来恢复数据的标准做法是，用 mysqlbinlog 工具解析出来，然后把解析结果整个发给 MySQL 执行。**

### 循环复制问题
生产环境目前主要设置双主结构、互为主备。需要解决循环复制问题。


## 25 | MySQL是怎么保证高可用的？
### 主备延迟
- 主库 A 执行完成一个事务，写入 binlog，我们把这个时刻记为 T1;
- 之后传给备库 B，我们把备库 B 接收完这个 binlog 的时刻记为 T2;
- 备库 B 执行完成这个事务，我们把这个时刻记为 T3。

主备延迟时间=T3-T1
备库上执行`show slave status`，返回结果里面会显示 `seconds_behind_master`，用于表示当前备库延迟了多少秒。

### 主备延迟的来源
- 有些部署条件下，备库所在机器的性能要比主库所在的机器性能差。
  - 主备可能发生切换，备库随时可能变成主库，所以主备库选用相同规格的机器，并且做对称部署，是现在比较常见的情况。
- 备库的压力大
  - 一般的想法是，主库既然提供了写能力，那么备库可以提供一些读能力。或者一些运营后台需要的分析语句，不能影响正常业务，所以只能在备库上跑。
  - 解决：
    - （1）一主多从。除了备库外，可以多接几个从库，让这些从库来分担读的压力。
    - （2）通过 binlog 输出到外部系统，比如 Hadoop 这类系统，让外部系统提供统计类查询的能力。
- 大事务
  - 主库上必须等事务执行完成才会写入 binlog，再传给备库。
  - 典型的大事务场景。
    - 一次性地用 delete 语句删除太多数据。
      - 不要这样做，应该分批删除
    - 大表 DDL
      - 计划内的 DDL，建议使用 gh-ost 方案
- 备库的并行复制能力


### 主备切换策略
- 可靠性优先策略
  - 切换流程中是有不可用时间的
- 可用性优先策略
  - 直接切换，不判断`seconds_behind_master`
  - 可能导致数据不一致

大多数情况下，建议使用可靠性优先策略。——因为数据可靠更重要。
看业务，比如记录日志的，也可以可用性优先。

## 26 | 备库为什么会延迟好几个小时？
如果备库执行日志的速度持续低于主库生成日志的速度，那这个延迟就有可能成了小时级别。而且对于一个压力持续比较高的主库来说，备库很可能永远都追不上主库的节奏。

### 备库并行复制能力
- MySQL 5.5 版本的并行复制策略
  - 丁奇自己实现的策略：
    - 按表分发策略，按行分发策略
    - 相比于按表并行分发策略，按行并行策略在决定线程分发的时候，需要消耗更多的计算资源。
- MySQL 5.6 版本的并行复制策略
  - 官方实现的版本
  - 支持的粒度是按库并行
- MySQL 5.7 的并行复制策略
  - 同时处于 prepare 状态的事务，在备库执行时是可以并行的；
  - 处于 prepare 状态的事务，与处于 commit 状态的事务之间，在备库执行时也是可以并行的。
- MySQL 5.7.22 的并行复制策略
  - 新增：基于 WRITESET 的并行复制

## 27 | 主库出问题了，从库怎么办？
一个切换系统会怎么完成一主多从的主备切换过程??

### 基于位点的主备切换
考虑到切换过程中不能丢数据，所以我们找位点的时候，总是要找一个“稍微往前”的，然后再通过判断跳过那些在从库 B 上已经执行过的事务。
常情况下，我们在切换任务的时候，要先主动跳过这些错误，有两种常用的方法。
1. 主动跳过一个事务
```
set global sql_slave_skip_counter=1;
start slave;
```
2. 通过设置 slave_skip_errors 参数，直接设置跳过指定的错误
一般会遇到：
- 1062 错误是插入数据时唯一键冲突；
- 1032 错误是删除数据时找不到行。

### 基于 GTID 的主备切换
MySQL 5.6 版本引入了 GTID

## 28 | 读写分离有哪些坑？
一主多从架构的应用场景：读写分离，以及怎么处理主备延迟导致的读写分离问题。

### 强制走主库方案

将查询请求做分类：
- 对于必须要拿到最新结果的请求，强制将其发到主库上。
- 对于可以读到旧数据的请求，才将其发到从库上
  
### Sleep 方案
主库更新后，读从库之前先 sleep 一下。具体的方案就是，类似于执行一条 select sleep(1) 命令。


### 判断主备无延迟方案
- 第一种确保主备无延迟的方法是：每次从库执行查询请求前，先判断 seconds_behind_master 是否已经等于 0。如果还不等于 0 ，那就必须等到这个参数变为 0 才能执行查询请求。
- 第二种方法，对比位点确保主备无延迟
- 第三种方法，对比 GTID 集合确保主备无延迟：

### 配合 semi-sync 方案
### 等主库位点方案
### GTID 方案


## 29 | 如何判断一个数据库是不是出问题了？
### select 1 判断
`innodb_thread_concurrency`参数，控制 InnoDB 的并发线程上限。
在 InnoDB 中，innodb_thread_concurrency 这个参数的默认值是 0，表示不限制并发线程数量。但是，不限制并发线程数肯定是不行的。通常情况下，我们建议把 innodb_thread_concurrency 设置为 64~128 之间的值。
**并发连接和并发查询**:在 show processlist 的结果里，看到的几千个连接，指的就是并发连接。而“当前正在执行”的语句，才是我们所说的并发查询。并发连接数达到几千个影响并不大，就是多占一些内存而已。我们应该关注的是并发查询，因为并发查询太高才是 CPU 杀手。

**在线程进入锁等待以后，并发线程的计数会减一。**

### 查表判断
在系统库（mysql 库）里创建一个表，比如命名为 health_check，里面只放一行数据，然后定期执行`select * from mysql.health_check;`
可以检测出由于并发线程过多导致的数据库不可用的情况，但空间满了以后，这种方法又会变得不好使。

### 更新判断
更新判断是一个相对比较常用的方案了，不过依然存在一些问题。其中，“判定慢”一直是让 DBA 头疼的问题。

### 内部统计
MySQL 5.6 版本以后提供的 performance_schema 库，就在 file_summary_by_event_name 表里统计了每次 IO 请求的时间。

### 小结
丁奇比较倾向的方案，是优先考虑 update 系统表，然后再配合增加检测 performance_schema 的信息。



## 31 | 误删数据后除了跑路，还能怎么办？

误删数据分类：
- 使用 delete 语句误删数据行；
- 使用 drop table 或者 truncate table 语句误删数据表；
- 使用 drop database 语句误删数据库；
- 使用 rm 命令误删整个 MySQL 实例。

### 误删行
用 Flashback 工具通过闪回把数据恢复回来。 
不建议直接在主库上执行这些操作。

使用 delete 命令删除的数据，你还可以用 Flashback 来恢复。而使用 truncate /drop table 和 drop database 命令删除的数据，就没办法通过 Flashback 来恢复了。

### 误删库 / 表
需要使用全量备份，加增量日志.这个方案要求线上有定期的全量备份，并且实时备份 binlog。

### 延迟复制备库
如果有非常核心的业务，不允许太长的恢复时间，我们可以考虑搭建延迟复制的备库。这个功能是 MySQL 5.6 版本引入的。

### 预防误删库 / 表的方法
第一条建议是，账号分离。这样做的目的是，避免写错命令。
第二条建议是，制定操作规范。这样做的目的，是避免写错要删除的表名。

### rm 删除数据



## 32 | 为什么还有kill不掉的语句？

在 MySQL 中有两个 kill 命令：
- kill query + 线程 id，表示终止这个线程中正在执行的语句；
- kill connection + 线程 id，这里 connection 可缺省，表示断开这个线程的连接，当然如果这个线程有语句正在执行，也是要先停止正在执行的语句的。

### 收到 kill 以后，线程做什么？
当用户执行 kill query thread_id_B 时，MySQL 里处理 kill 命令的线程做了两件事：
1. 把 session B 的运行状态改成 THD::KILL_QUERY(将变量 killed 赋值为 THD::KILL_QUERY)；
2. 给 session B 的执行线程发一个信号。

kill 无效的情况：
- 第一类情况，即：线程没有执行到判断线程状态的逻辑。
- 另一类情况是，终止逻辑耗时较长

### 另外两个关于客户端的误解
#### 第一个误解是：如果库里面的表特别多，连接就会很慢。
当使用默认参数连接的时候，MySQL 客户端会提供一个本地库名和表名补全的功能。为了实现这个功能，客户端在连接成功后，需要多做一些操作：
1. 执行 show databases；
2. 切到 db1 库，执行 show tables；
3. 把这两个命令的结果用于构建一个本地的哈希表。

第3步最花时间。

**我们感知到的连接过程慢，其实并不是连接慢，也不是服务端慢，而是客户端慢。**

#### –quick 是一个更容易引起误会的参数，也是关于客户端常见的一个误解。


## 33 | 我查这么多数据，会不会把数据库内存打爆？
### 全表扫描对 server 层的影响
假设要执行：
```
mysql -h$host -P$port -u$user -p$pwd -e "select * from db1.t" > $target_file
```

MySQL 是“边读边发的”
在服务端 show processlist 看到State 的值一直处于“Sending to client”，就表示服务器端的网络栈写满了。
**对于正常的线上业务来说，如果一个查询的返回结果不会很多的话，我都建议你使用 mysql_store_result 这个接口，直接把查询结果保存到本地内存。**

### 全表扫描对 InnoDB 的影响
内存的数据页是在 Buffer Pool (BP) 中管理的，在 WAL 里 Buffer Pool 起到了加速更新的作用。而实际上，Buffer Pool 还有一个更重要的作用，就是加速查询。

内存命中率:可以在show engine innodb status中可以看到“Buffer pool hit rate”字样，显示的就是当前的命中率。

InnoDB Buffer Pool 的大小是由参数 innodb_buffer_pool_size 确定的，一般建议设置成可用物理内存的 60%~80%。
InnoDB 内存管理用的是最近最少使用 (Least Recently Used, LRU) 算法，但有改进。在 InnoDB 实现上，按照 5:3 的比例把整个 LRU 链表分成了 young 区域和 old 区域。为了处理类似全表扫描的操作量身定制的。


## 34 | 到底可不可以使用join？
### Index Nested-Loop Join
可以使用被驱动表的索引时，mysql中进行join查询时，采用的是Index Nested-Loop Join算法查询结果，即：
读入驱动表数据，然后利用索引去读被驱动表

### Simple Nested-Loop Join
被驱动表用不上索引时， 读入驱动表数据，然后驱动表读一条记录，就全表扫描被驱动表一次
——mysql没用到该算法，而是用下面的Block Nested-Loop Join算法

### Block Nested-Loop Join
把表 t1 的数据读入线程内存 join_buffer 中，扫描表 t2，把表 t2 中的每一行取出来。join_buffer 的大小是由参数 join_buffer_size 设定的，默认值是 256k。如果放不下表 t1 的所有数据话，策略很简单，就是分段放。

**决定哪个表做驱动表的时候，应该是两个表按照各自的条件过滤，过滤完成之后，计算参与 join 的各个字段的总数据量，数据量小的那个表，就是“小表”，应该作为驱动表。**

### 结论
- 如果可以使用被驱动表的索引，join 语句还是有其优势的；
- 不能使用被驱动表的索引，只能使用 Block Nested-Loop Join 算法，这样的语句就尽量不要使用；
- 在使用 join 的时候，应该让小表做驱动表。


## 35 | join语句怎么优化？
### Multi-Range Read 优化
Multi-Range Read 优化 (MRR)注意目的是尽量使用顺序读盘。

因为大多数的数据都是按照主键递增顺序插入得到的，所以我们可以认为，如果按照主键的递增顺序查询的话，对磁盘的读比较接近顺序读，能够提升读性能。
explain 结果中，我们可以看到 Extra 字段多了 Using MRR，表示的是用上了 MRR 优化。
MRR 能够提升性能的核心在于，这条查询语句在索引 a 上做的是一个范围查询（也就是说，这是一个多值查询），可以得到足够多的主键 id。这样通过排序以后，再去主键索引查数据，才能体现出“顺序性”的优势。

### Batched Key Access
Batched Key Access(BKA) 算法是对 Index Nested-Loop Join(NLJ)算法的优化。

NLJ 算法执行的逻辑是：从驱动表 t1，一行行地取出 a 的值，再到被驱动表 t2 去做 join。

把表 t1 的数据取出来一部分，先放到一个临时内存——join_buffer。

### BNL 算法的性能问题

**大表 join 操作虽然对 IO 有影响，但是在语句执行结束后，对 IO 的影响也就结束了。但是，对 Buffer Pool 的影响就是持续性的，需要依靠后续的查询请求慢慢恢复内存命中率。**
为了减少这种影响，你可以考虑增大 join_buffer_size 的值，减少对被驱动表的扫描次数。
**执行语句之前，需要通过理论分析和查看 explain 结果的方式，确认是否要使用 BNL 算法。如果确认优化器会使用 BNL 算法，就需要做优化。优化的常见做法是，给被驱动表的 join 字段加上索引，把 BNL 算法转成 BKA 算法。**
——不能只是考虑业务如何实现，实现业务后要做优化！！！

### BNL 转 BKA
一些情况下，我们可以直接在被驱动表上建索引，这时就可以直接转成 BKA 算法了。
遇到低频字段，不适合创建索引的情况，可以考虑使用临时表。

**总体来看，不论是在原表上加索引，还是用有索引的临时表，我们的思路都是让 join 语句能够用上被驱动表上的索引，来触发 BKA 算法，提升查询性能。**

### 扩展 -hash join
mysql目前没实现，但我们可以在业务端自己实现。


## 36 | 为什么临时表可以重名？
临时表和内存表的区别：
- 内存表:使用 Memory 引擎的表，建表语法是 create table … engine=memory。
- 临时表，可以使用各种引擎类型 。

### 临时表的特性
- 建表语句 `show tables 命令不显示临时表`
- 只能被创建它的session访问
- 可以与普通表重名
- session内有同名的临时表和普通表的时候，show create 语句，以及增删改查语句访问的是临时表
- show tables 命令不显示临时表

适合join优化场景，原因：
- 可以重名，多个session通知执行join优化时不会冲突
- 不需要担心数据删除问题。

### 临时表的应用
由于不用担心线程之间的重名冲突，临时表经常会被用在复杂查询的优化过程中。其中，分库分表系统的跨库查询就是一个典型的使用场景。

```
select v from ht where f=N;
```
在ht分库分表的情况下，分区key用f做区分。若有另一个索引k，查询时类似这种
```
select v from ht where k >= M order by t_modified desc limit 100;
```
一般情况下，分库分表系统都有一个中间层 proxy。对于上述查询，有两种实现思路：
- 1. 在 proxy 层的进程代码中实现排序。
- 2. 把各个分库拿到的数据，汇总到一个 MySQL 实例的一个表中，然后在这个汇总实例上做逻辑操作。

**在实践中，我们往往会发现每个分库的计算量都不饱和，所以会直接把临时表 temp_ht 放到 32 个分库中的某一个上。**

### 为什么临时表可以重名？
创建临时表语句执行时`create temporary table ...`，MySQL 要给这个 InnoDB 表创建一个 frm 文件保存表结构定义，另外还需要找地方存数据。frm 文件放在临时文件目录下，文件名的后缀是.frm，前缀是“#sql{进程 id}_{线程 id}_ 序列号”。
内存中，table_def_key：
- 普通表 库名 + 表名
- 临时表 库名 + 表名 + server_id + thread_id

### 临时表和主备复制
在 binlog_format='row’的时候，临时表的操作不记录到 binlog 中，也省去了不少麻烦，这也可以成为你选择 binlog_format 时的一个考虑因素。



## 37 | 什么时候会使用内部临时表？
### union 执行流程
`union`会创建内部临时表，将数据合并；`union all`则不会创建临时表。

### group by 执行流程
`group by`
注意，**内存临时表的大小是有限制的，参数 tmp_table_size 就是控制这个内存大小的，默认是 16M。**内存临时表的大小超过该变量规定大小，那么，这时候就会把内存临时表转成磁盘临时表，磁盘临时表默认使用的引擎是 InnoDB。
```
set tmp_table_size=1024; # 把内存临时表的大小限制为最大 1024 字节
select id%100 as m, count(*) as c from t1 group by m order by null limit 10;
```

### group by 优化方法 -- 索引
不论是使用内存临时表还是磁盘临时表，group by 逻辑都需要构造一个带唯一索引的表，执行代价都是比较高的。
优化思路：group by的输入数据有序时，可以避免使用临时表。
——做法：在 MySQL 5.7 版本支持了 generated column 机制，用来实现列数据的关联更新。你可以用下面的方法创建一个列 z，然后在 z 列上创建一个索引（如果是 MySQL 5.6 及之前的版本，你也可以创建普通列和索引，来解决这个问题）。
```
alter table t1 add column z int generated always as(id % 100), add index(z);
select z, count(*) as c from t1 group by z;
```

### group by 优化方法 -- 直接排序
在 group by 语句中加入 SQL_BIG_RESULT 这个提示（hint），就可以告诉优化器：这个语句涉及的数据量很大，请直接用磁盘临时表。
```
select SQL_BIG_RESULT id%100 as m, count(*) as c from t1 group by m;
```

丁奇总结的指导原则：
- 如果对 group by 语句的结果没有排序要求，要在语句后面加 order by null；
- 尽量让 group by 过程用上表的索引，确认方法是 explain 结果里没有 Using temporary 和 Using filesort；
- 如果 group by 需要统计的数据量不大，尽量只使用内存临时表；
- 也可以通过适当调大 tmp_table_size 参数，来避免用到磁盘临时表；
- 如果数据量实在太大，使用 SQL_BIG_RESULT 这个提示，来告诉优化器直接使用排序算法得到 group by 的结果。


## 38 | 都说InnoDB好，那还要不要使用Memory引擎？
### 内存表的数据组织结构
InnoDB是一棵B+树。
Memory 引擎的数据和索引是分开的。

- InnoDB 引擎把数据放在主键索引上，其他索引上保存的是主键 id。这种方式，我们称之为索引组织表（Index Organizied Table）。
  - 有序
- Memory 引擎采用的是把数据单独存放，索引上保存数据位置的数据组织形式，我们称之为堆组织表（Heap Organizied Table）。
  - 无序，hash索引

### 不建议在生产环境上使用内存表
- 锁粒度问题
  - 内存表不支持行锁，只支持表锁。因此，一张表只要有更新，就会堵住其他所有在这个表上的读写操作。
- 数据持久化问题
  - 数据在内存中，容易出现数据不一致

**基于内存表的特性，我们还分析了它的一个适用场景，就是内存临时表。内存表支持 hash 索引，这个特性利用起来，对复杂查询的加速效果还是很不错的。**

## 39 | 自增主键为什么不是连续的？

### 自增值保存在哪儿？
表的结构定义存放在后缀名为.frm 的文件中，但是并不会保存自增值。
- MyISAM 引擎的自增值保存在数据文件中
- InnoDB 引擎的自增值，其实是保存在了内存里。MySQL 直到 8.0 版本，才给 InnoDB 表的自增值加上了持久化的能力，确保重启前后一个表的自增值不变。

### 自增值修改机制
从 auto_increment_offset 开始，以 auto_increment_increment 为步长
这两个参数系统默认是1，但某些情况使用的不是默认值。比如：双 M 的主备结构里要求双写的时候，就可能会设置成 auto_increment_increment=2，让一个库的自增id是奇数，另一个是偶数，避免两个库发生冲突。

### 自增值的修改时机
自增主键 id 不连续的原因：
- 1. 唯一键冲突
- 2. 事务回滚也会产生类似的现象
  - 为了提高性能
- 3. 对于批量插入数据的语句，MySQL 有一个批量申请自增 id 的策略

### 自增锁的优化

在生产上，尤其是有 insert … select 这种批量插入数据的场景时，从并发插入数据性能的角度考虑，我建议你这样设置：innodb_autoinc_lock_mode=2 ，并且 binlog_format=row. 这样做，既能提升并发性，又不会出现数据一致性问题。
——批量插入数据，包含的语句类型是 insert … select、replace … select 和 load data 语句。

MySQL 5.1.22 版本开始引入的参数 innodb_autoinc_lock_mode，控制了自增值申请时的锁范围。从并发性能的角度考虑，我建议你将其设置为 2，同时将 binlog_format 设置为 row。我在前面的文章中其实多次提到，binlog_format 设置为 row，是很有必要的。


## 40 | insert语句的锁为什么这么多？
### insert … select 语句
```
CREATE TABLE `t` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `c` (`c`)
) ENGINE=InnoDB;

insert into t values(null, 1,1);
insert into t values(null, 2,2);
insert into t values(null, 3,3);
insert into t values(null, 4,4);

create table t2 like t
```

在可重复读隔离级别下，binlog_format=statement 时执行：
```
insert into t2(c,d) select c,d from t;
```
时，需要对表 t 的所有行和间隙加锁.

原因：可能出现主备不一致。

insert … select 是很常见的在两个表之间拷贝数据的方法。需要注意，在可重复读隔离级别下，这个语句会给 select 的表里扫描到的记录和间隙加读锁。

### insert 循环写入
```
insert into t(c,d)  (select c+1, d from t force index(c) order by c desc limit 1);
```

要学会用 explain 的结果来“脑补”整条语句的执行过程。

如果 insert 和 select 的对象是同一个表，则有可能会造成循环写入。这种情况下，我们需要引入用户临时表来做优化。

### insert 唯一键冲突

### insert into … on duplicate key update
insert into … on duplicate key update 这个语义的逻辑是，插入一行数据，如果碰到唯一键约束，就执行后面的更新语句。
注意，如果有多个列违反了唯一性约束，就会按照索引的顺序，修改跟第一个索引冲突的行。


## 42 | grant之后要跟着flush privileges吗？
grant 语句会同时修改数据表和内存，判断权限的时候使用的是内存数据。因此，规范地使用 grant 和 revoke 语句，是不需要随后加上 flush privileges 语句的。

### flush privileges 使用场景
flush privileges 语句本身会用数据表的数据重建一份内存权限数据，所以在权限数据可能存在不一致的情况下再使用。而这种不一致往往是由于直接用 DML 语句操作系统权限表导致的，所以我们尽量不要使用这类语句。




## 43 | 要不要使用分区表？

### 分区表是什么？  




## 44 | 答疑文章（三）：说一说这些好问题
### join 的写法

在 MySQL 里，NULL 跟任何值执行等值判断和不等值判断的结果，都是 NULL。这里包括， select NULL = NULL 的结果，也是返回 NULL。

即使我们在 SQL 语句中写成 left join，执行过程还是有可能不是从左到右连接的。也就是说，**使用 left join 时，左边的表不一定是驱动表。**
**如果需要 left join 的语义，就不能把被驱动表的字段放在 where 条件里面做等值判断或不等值判断，必须都写在 on 里面。**

### Simple Nested Loop Join 的性能问题
虽然 BNL 算法和 Simple Nested Loop Join 算法都是要判断 M*N 次（M 和 N 分别是 join 的两个表的行数），但是 Simple Nested Loop Join 算法的每轮判断都要走全表扫描，因此性能上 BNL 算法执行起来会快很多。

### distinct 和 group by 的性能
如果只需要去重，不需要执行聚合函数，distinct 和 group by 哪种效率高一些呢？
即如下场景：
```
select a from t group by a order by null;
select distinct a from t;
```
答案：执行流程一样，性能是相同的。

### 备库自增主键问题
自增 id 的生成顺序，和 binlog 的写入顺序可能是不同的
SET INSERT_ID 语句是固定跟在 insert 语句


## 45 | 自增id用完怎么办？
### 表定义自增值 id
表定义的自增值达到上限后的逻辑是：再申请下一个 id 时，得到的值保持不变。

### InnoDB 系统自增 row_id
如果你创建的 InnoDB 表没有指定主键，那么 InnoDB 会给你创建一个不可见的，长度为 6 个字节的 row_id。row_id 达到上限后，则会归 0 再重新递增，如果出现相同的 row_id，后写的数据会覆盖之前的数据。
我们应该在 InnoDB 表中主动创建自增主键。因为，表自增 id 到达上限后，再插入数据时报主键冲突错误，是更能被接受的。

### Xid
MySQL 内部维护了一个全局变量 global_query_id，每次执行语句的时候将它赋值给 Query_id，然后给这个变量加 1。如果当前语句是这个事务执行的第一条语句，那么 MySQL 还会同时把 Query_id 赋值给这个事务的 Xid。
而 global_query_id 是一个纯内存变量，重启之后就清零了。在同一个数据库实例中，不同事务的 Xid 也是有可能相同的。但是 MySQL 重启之后会重新生成新的 binlog 文件，这就保证了，同一个 binlog 文件里，Xid 一定是惟一的。

### Innodb trx_id
Xid 是由 server 层维护的。InnoDB 内部使用 Xid，就是为了能够在 InnoDB 事务和 server 之间做关联。但是，InnoDB 自己的 trx_id，是另外维护的。
InnoDB 内部维护了一个 max_trx_id 全局变量，每次需要申请一个新的 trx_id 时，就获得 max_trx_id 的当前值，然后并将 max_trx_id 加 1。
InnoDB 的 max_trx_id 递增值每次 MySQL 重启都会被保存起来

只读事务不分配 trx_id。

### thread_id
thread_id：系统保存了一个全局变量 thread_id_counter，每新建一个连接，就将 thread_id_counter 赋值给这个新连接的线程变量。
