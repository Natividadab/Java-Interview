
## MySQL
### 引擎对比
1. InnoDB支持事务
2. InnoDB支持外键
3. InnoDB有行级锁，MyISAM是表级锁

MyISAM相对简单所以在效率上要优于InnoDB。如果系统插入和查询操作多，不需要事务外键。选择MyISAM  
如果需要频繁的更新、删除操作，或者需要事务、外键、行级锁的时候。选择InnoDB。

### [数据库性能优化](https://www.zhihu.com/question/19719997)
1. 优化SQL语句和索引，在where/group by/order by中用到的字段建立索引，索引字段越小越好，复合索引建立的顺序
2. 加缓存，Memcached, Redis
3. 主从复制，读写分离
4. 垂直拆分，其实就是根据你模块的耦合度，将一个大的系统分为多个小的系统，也就是分布式系统
5. 水平切分，针对数据量大的表，这一步最麻烦，最能考验技术水平，要选择一个合理的sharding key,为了有好的查询效率，表结构也要改动，做一定的冗余，应用也要改，sql中尽量带sharding key，将数据定位到限定的表上去查，而不是扫描全部的表；

### [SQL优化](https://www.imooc.com/video/3711)：
优化步骤：

1. 根据慢日志定位慢查询SQL
2. 用explain分析SQL（type和extra字段分析）
3. 修改SQL或加索引（如下）

* 对经常查询的列建立索引，但索引建多了当数据改变时修改索引会增大开销
* 使用精确列名查询而不是*，特别是当数据量大的时候
* 减少[子查询](http://www.cnblogs.com/zhengyun_ustc/p/slowquery3.html)，使用Join替代
* 不用NOT IN，因为会使用全表扫描而不是索引；不用IS NULL，NOT IS NULL，因为会使索引、索引统计和值更加复杂，并且需要额外一个字节的存储空间。

问：max(xxx)如何用索引优化？
答：在xxx列上建立索引，因为索引是B+树顺序排列的，在下次查询的时候就会使用索引来查询到最大的值是哪个

问：有个表特别大，字段是姓名、年龄、班级，如果调用`select * from table where name = xxx and age = xxx`该如何通过建立索引的方式优化查询速度？  
答：由于mysql查询每次只能使用一个索引，如果在name、age两列上创建联合索引的话将带来更高的效率。如果我们创建了(name, age)的联合索引，那么其实相当于创建了(name, age)、(name)2个索引。因此我们在创建复合索引时应该将最常用作限制条件的列放在最左边，依次递减。其次还要考虑该列的数据离散程度，如果有很多不同的值的话建议放在左边，name的离散程度也大于age。

#### 最左前缀匹配原则
定义：在检索数据时从联合索引的最左边开始匹配。一直向右匹配直到遇到范围查询（>/</between/like）就停止后面的匹配。比如查询a = 3, b = 4 and c > 5 and d = 6，如果建立(a,b,c,d)索引，d是用不到索引的，如果建立(a,b,d,c)索引，则都可以用到，此外a,b,d的顺序可以任意调整

联合索引原理：联合索引是将第一个字段作为非叶子节点形成B+树结构的，在查询索引的时候先根据该字段一步步查询到具体的叶子节点，叶子节点上还会存在第二个字段甚至第三个字段的已排序结果。所以对于(a,b,c,d)这个联合索引会存在(a)，(a,b)，(a,b,c)，(a,b,c,d)四个索引

## 事务隔离级别
1. 原子性（Atomicity）：事务作为一个整体被执行 ，要么全部执行，要么全部不执行；
2. 一致性（Consistency）：在事务开始之前和事务结束以后，数据库的完整性没有被破坏。这表示写入的资料必须完全符合所有的预设规则，这包含资料的精确度、串联性以及后续数据库可以自发性地完成预定的工作；
3. 隔离性（Isolation）：多个事务并发执行时，一个事务的执行不应影响其他事务的执行，然后你可以扯到隔离级别；
4. 持久性（Durability）：一个事务一旦提交，对数据库的修改应该永久保存。

不同事务隔离级别的问题：  
[https://www.jianshu.com/p/4e3edbedb9a8](https://www.jianshu.com/p/4e3edbedb9a8)

## 锁表、锁行
1. InnoDB 支持表锁和行锁，使用索引作为检索条件修改数据时采用行锁，否则采用表锁
2. InnoDB 自动给修改操作加锁，给查询操作不自动加锁
3. 行锁相对于表锁来说，优势在于高并发场景下表现更突出，毕竟锁的粒度小
4. 当表的大部分数据需要被修改，或者是多表复杂关联查询时，建议使用表锁优于行锁

[https://segmentfault.com/a/1190000012773157](https://segmentfault.com/a/1190000012773157)

### 悲观锁乐观锁、如何写对应的SQL
悲观锁：select for update  
乐观锁：先查询一次数据，然后使用查询出来的数据+1进行更新数据，如果失败则循环  

[https://www.jianshu.com/p/f5ff017db62a](https://www.jianshu.com/p/f5ff017db62a)

### MVVC
https://tech.meituan.com/2014/08/20/innodb-lock.html

### 索引
#### 原理
聚集索引：数据行的物理顺序与列值（一般是主键的那一列）的逻辑顺序相同，所以一个表中只能拥有一个聚集索引。叶子结点即存储了真实的数据行，不再有另外单独的数据页  
非聚集索引：该索引中索引的逻辑顺序与磁盘上行的物理存储顺序不同，所以一个表中可以拥有多个非聚集索引。叶子结点包含索引字段值及指向数据页数据行的逻辑指针  

#### 索引原理
使用B+树来构建索引，为什么不用二叉树？因为红黑树在磁盘上的查询性能远不如B+树

红黑树最多只有两个子节点，所以高度会非常高，导致遍历查询的次数会多，又因为红黑树在数组中存储的方式，导致逻辑上很近的父节点与子节点可能在物理上很远，导致无法使用磁盘预读的局部性原理，需要很多次IO才能找到磁盘上的数据

但B+树一个节点中可以存储很多个索引的key，且将大小设置为一个页，一次磁盘IO就能读取很多个key，且叶子节点之间还加上了下个叶子节点的指针，遍历索引也会很快。

B+树的高度如何计算？
1. 单个叶子节点（页）中的记录数 =16K/1K=16。（这里假设一行记录的数据大小为1k）
2. 假设主键 ID 为 bigint 类型，长度为 8 字节，而指针大小在 InnoDB 源码中设置为 6 字节，这样一共 14 字节
3. 我们一个页中能存放多少这样的单元，其实就代表有多少指针，即 16384/14=1170。
4. 那么可以算出一棵高度为 2 的 B+ 树，能存放 1170*16=18720 条这样的数据记录。

B与B+区别:
1. b+树的中间节点不保存数据，所以磁盘页能容纳更多节点元素；
2. b+树查询必须查找到叶子节点，b树只要匹配到即可不用管元素位置，因此b+树查找更稳定
3. 对于范围查找来说，b+树只需遍历叶子节点链表即可，b树却需要重复地中序遍历

#### 优化
如何选择合适的列建立索引？

1. WHERE / GROUP BY / ORDER BY / ON 的列
2. 离散度大（不同的数据多）的列使用索引才有查询效率提升
3. 索引字段越小越好，因为数据库按页存储的，如果每次查询IO读取的页越少查询效率越高

## 分区分库分表
分区：把一张表的数据分成N个区块，在逻辑上看最终只是一张表，但底层是由N个物理区块组成的，通过将不同数据按一定规则放到不同的区块中提升表的查询效率。

水平分表：为了解决单标数据量过大（数据量达到千万级别）问题。所以将固定的ID hash之后mod，取若0~N个值，然后将数据划分到不同表中，需要在写入与查询的时候进行ID的路由与统计

垂直分表：为了解决表的宽度问题，同时还能分别优化每张单表的处理能力。所以将表结构根据数据的活跃度拆分成多个表，把不常用的字段单独放到一个表、把大字段单独放到一个表、把经常使用的字段放到一个表

分库：面对高并发的读写访问，当数据库无法承载写操作压力时，不管如何扩展slave服务器，此时都没有意义了。因此数据库进行拆分，从而提高数据库写入能力，这就是分库。

问题：

1. 事务问题。在执行分库之后，由于数据存储到了不同的库上，数据库事务管理出现了困难。如果依赖数据库本身的分布式事务管理功能去执行事务，将付出高昂的性能代价；如果由应用程序去协助控制，形成程序逻辑上的事务，又会造成编程方面的负担。
2. 跨库跨表的join问题。在执行了分库分表之后，难以避免会将原本逻辑关联性很强的数据划分到不同的表、不同的库上，我们无法join位于不同分库的表，也无法join分表粒度不同的表，结果原本一次查询能够完成的业务，可能需要多次查询才能完成。
3. 额外的数据管理负担和数据运算压力。额外的数据管理负担，最显而易见的就是数据的定位问题和数据的增删改查的重复执行问题，这些都可以通过应用程序解决，但必然引起额外的逻辑运算。

欢迎光临[我的博客](http://www.wangtianyi.top/?utm_source=github&utm_medium=github)，发现更多技术资源~
