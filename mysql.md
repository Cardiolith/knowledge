# MySQL 实战45讲



### 基础架构

![img](https://static001.geekbang.org/resource/image/0d/d9/0d2070e8f84c4801adbfa03bda1f98d9.png)



MySQL 可以分为 Server 层与存储引擎层。

Server 层包括连接器，分析器，优化器，执行器等，所有跨存储引擎的功能都在这一层实现。

而存储引擎层负责数据的存储与提取。其架构是插件式的。其中最常用的是InnoDB，从MySQL 5.5.5 版本成为默认存储引擎。

#### 连接器

第一步，连接到数据库。连接器负责跟客户端建立连接，获取权限，维持连接和管理连接。

![img](https://static001.geekbang.org/resource/image/f2/ed/f2da4aa3a672d48ec05df97b9f992fed.png)

数据库里面，**长连接**是指连接成功后，如果客户端持续有请求，则一直使用同一个连接。短连接则是指每次执行完很少的几次查询就断开连接，下次查询再重新建立一个。

但是全部使用长连接后，你可能会发现，有些时候 MySQL 占用内存涨得特别快，这是因为 **MySQL 在执行过程中临时使用的内存是管理在连接对象里面的**。这些资源会在连接断开的时候才释放。所以如果长连接累积下来，可能导致内存占用太大，被系统强行杀掉（OOM），从现象看就是 MySQL 异常重启了。

怎么解决这个问题呢？你可以考虑以下两种方案。

- 定期断开长连接。使用一段时间，或者程序里面判断执行过一个占用内存的大查询后，断开连接，之后要查询再重连。
- 如果你用的是 MySQL 5.7 或更新版本，可以在每次执行一个比较大的操作后，通过执行 mysql_reset_connection 来重新初始化连接资源。这个过程不需要重连和重新做权限验证，但是会将连接恢复到刚刚创建完时的状态。

#### 查询缓存

MySQL 拿到一个查询请求后，会先查看缓存。之前执行过的语句及其结果可能会以key-value的形式，缓存在内存中。如果存在，那么会直接返回给客户端。

如果表更新了，那么这个表上所有的查询缓存都会被清空。

MySQL 8.0 版本将查询缓存的功能都删除了。

#### 分析器

如果没有命中缓存，那么就开始真正执行语句了。

分析器还会先做词法分析。识别输入的语句中每个字符串的含义。

做完这些识别以后，就要做“语法分析”。根据词法分析的结果，语法分析器会根据语法规则，判断你输入的这个语句是否满足MySQL语法。

#### 优化器

优化器实在表里面有多个索引的时候，决定使用哪个索引；或者在一个语句有多表关联(join)时，决定各个表的连接顺序。例如：

```mysql
mysql> select * from t1 join t2 using(ID)  where t1.c=10 and t2.d=20;
```

- 既可以先从表 t1 里面取出 c=10 的记录的 ID 值，再根据 ID 值关联到表 t2，再判断 t2 里面 d 的值是否等于 20。
- 也可以先从表 t2 里面取出 d=20 的记录的 ID 值，再根据 ID 值关联到 t1，再判断 t1 里面 c 的值是否等于 10。

优化器的作用就是决定使用哪一个方案。

#### 执行器

开始执行时，首先判断你对这个表是否有没有查询的权限，如果没有，就会返回没有权限的错误（在工程实现上，如果命中查询缓存，会在查询缓存返回结果的时候，做权限验证。查询也会在优化器之前调用 precheck 验证权限)。

如果有权限，就打开表继续执行。打开表的时候，执行器就会根据表的引擎定义，去使用这个引擎提供的接口。

比如我们这个例子中的表 T 中，ID 字段没有索引，那么执行器的执行流程是这样的：

1. 调用 InnoDB 引擎接口取这个表的第一行，判断 ID 值是不是 10，如果不是则跳过，如果是则将这行存在结果集中；
2. 调用引擎接口取“下一行”，重复相同的判断逻辑，直到取到这个表的最后一行。
3. 执行器将上述遍历过程中所有满足条件的行组成的记录集作为结果集返回给客户端。

对于有索引的表，执行的逻辑也差不多。第一次调用的是“取满足条件的第一行”这个接口，之后循环取“满足条件的下一行”这个接口，这些接口都是引擎中已经定义好的。

你会在数据库的慢查询日志中看到一个 **rows_examined** 的字段，表示这个语句执行过程中扫描了多少行。这个值就是在执行器每次调用引擎获取数据行的时候累加的。

在有些场景下，执行器调用一次，在引擎内部则扫描了多行，因此**引擎扫描行数跟 rows_examined** 并不是完全相同的。



### 日志系统

更新语句与查询语句执行链路相同。

不同点在，在一个表上有更新时，和这张表有关的查询缓存都会失效。更新流程还涉及两个重要的日志模块：redo log(重做日志) 和 binlog(归档日志)。

#### redo log

如果每一次更新操作都需要写进磁盘，然后磁盘找到对应的那条记录，然后再更新，整个过程的IO成本，查找的成本都很高。

其实就是 MySQL 里经常说到的 **WAL** 技术，WAL 的全称是 **Write-Ahead Logging**，它的关键点就是先写日志，再写磁盘，也就是先写粉板，等不忙的时候再写账本。

具体来说，当有一条记录需要更新的时候，InnoDB 引擎就会先把记录写到 **redo log**（粉板）里面，并更新内存，这个时候更新就算完成了。同时，InnoDB 引擎会在适当的时候，将这个操作记录更新到磁盘里面，而这个更新往往是在系统比较空闲的时候做，这就像打烊以后掌柜做的事。

InnoDB 的 redo log 是固定大小的，比如可以配置为一组 4 个文件，每个文件的大小是 1GB，那么这块“粉板”总共就可以记录 4GB 的操作。从头开始写，写到末尾就又回到开头循环写，如下面这个图所示。

![img](https://static001.geekbang.org/resource/image/16/a7/16a7950217b3f0f4ed02db5db59562a7.png)

**write pos** 是当前记录的位置，一边写一边后移，写到第 3 号文件末尾后就回到 0 号文件开头。**checkpoint** 是当前要擦除的位置，也是往后推移并且循环的，擦除记录前要把记录更新到数据文件。write pos 和 checkpoint 之间的是“粉板”上还空着的部分，可以用来记录新的操作。如果 write pos 追上 checkpoint，表示“粉板”满了，这时候不能再执行新的更新，得停下来先擦掉一些记录，把 checkpoint 推进一下。



#### binlog

redo log 是 InnoDB 引擎特有的日志，而 Server 层也有自己的日志，称为 binlog（归档日志）

这两种日志有以下三点不同:

1. redo log 是 InnoDB 引擎特有的；binlog 是 MySQL 的 Server 层实现的，所有引擎都可以使用。
2. redo log 是物理日志，记录的是“在某个数据页上做了什么修改”；binlog 是逻辑日志，记录的是这个语句的原始逻辑，比如“给 ID=2 这一行的 c 字段加 1 ”。
3. redo log 是循环写的，空间固定会用完；binlog 是可以追加写入的。“追加写”是指 binlog 文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。

执行器和 InnoDB 引擎在执行这个简单的 update 语句时的内部流程。

1. 执行器先找引擎取 ID=2 这一行。ID 是主键，引擎直接用树搜索找到这一行。如果 ID=2 这一行所在的数据页本来就在内存中，就直接返回给执行器；否则，需要先从磁盘读入内存，然后再返回。
2. 执行器拿到引擎给的行数据，把这个值加上 1，比如原来是 N，现在就是 N+1，得到新的一行数据，再调用引擎接口写入这行新数据。
3. 引擎将这行新数据更新到内存中，同时将这个更新操作记录到 redo log 里面，此时 redo log 处于 prepare 状态。然后告知执行器执行完成了，随时可以提交事务。
4. 执行器生成这个操作的 binlog，并把 binlog 写入磁盘。
5. 执行器调用引擎的提交事务接口，引擎把刚刚写入的 redo log 改成提交（commit）状态，更新完成。

![img](https://static001.geekbang.org/resource/image/2e/be/2e5bff4910ec189fe1ee6e2ecc7b4bbe.png)

你可能注意到了，最后三步看上去有点“绕”，将 redo log 的写入拆成了两个步骤：prepare 和 commit，这就是"两阶段提交"。两阶段提交。



### 事务隔离

事务的隔离级别：

- 读未提交：一个事务可以看到其他事务尚未提交的变更。
- 读提交：事务提交之后，其他事务可以看到它做的变更。
- 可重复读：一个事务执行过程中看到的数据，总是和这个事务启动时看到的数据是一致的。
- 可串行化：对同一行记录，读会加读锁，写会加写锁，如果发生读写锁冲突，后访问的事务必须等到前一个事务执行完成，才能继续执行。

![img](https://static001.geekbang.org/resource/image/7d/f8/7dea45932a6b722eb069d2264d0066f8.png)

- 若隔离级别是“读未提交”， 则 V1 的值就是 2。这时候事务 B 虽然还没有提交，但是结果已经被 A 看到了。因此，V2、V3 也都是 2。
- 若隔离级别是“读提交”，则 V1 是 1，V2 的值是 2。事务 B 的更新在提交后才能被 A 看到。所以， V3 的值也是 2。
- 若隔离级别是“可重复读”，则 V1、V2 是 1，V3 是 2。之所以 V2 还是 1，遵循的就是这个要求：事务在执行期间看到的数据前后必须是一致的。
- 若隔离级别是“串行化”，则在事务 B 执行“将 1 改成 2”的时候，会被锁住。直到事务 A 提交后，事务 B 才可以继续执行。所以从 A 的角度看， V1、V2 值是 1，V3 的值是 2。



#### 事务隔离的实现



### 索引

三种常用的索引数据结构：哈希表、有序数组和二叉树。

hash表查找速度快，区间查找效率低。

有序数据支持区间查找，插入删除效率低。

