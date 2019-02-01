---
title: MVCC
date: 2017-11-30 17:01:43
categories: DataBase
tags: MVCC
description: MVCC
---
 
## 前言

Multiversion concurrency control (MVCC) 出现的原因是为了**提升数据库的并发性能**。

假设执行环境为 MySQL InnoDB RC 或者 RR 隔离级别，在一个事务中执行以下SQL：

```
update mvcc set val = 'v1' where id = 1;
```

**如果**没有MVCC ，那么这条语句会锁住id 为1的记录，其它事务无法进行读取。直接事务提交，其他事务才可以对这条记录进行读取，并发性能显然低下。

实际上，MySQL 实现了MVCC , 在这种情况下其他事务是可以读取到该行数据的，只不过是该事务开始时该条记录的**快照**。

这样一来，并发读写的性能显然得到了很大的提升。

## MVCC 实现

> MVCC 的实现，是通过保存数据在**某个时间点**的快照来实现的。也就是说，不管需要执行多长时间，每个事务看到的数据都是一致的。根据**事务开始的时间**不同，每个事务对同一张表，同一时刻看到的数据可能是不一样的。

假设有以下数据：

```
mysql> select * from mvcc where id = 1;
+----+------+
| id | val  |
+----+------+
|  1 | v1   |
+----+------+

```

有三个事务对该条数据进行操作：

|时间|事务1|事务2|事务3|
|:----:|:----:|:----:|:----:|
|T1|start transaction;<br>select * from mvcc where id = 1;|||
|T2||start transaction;<br>update mvcc set val = 'v2' where id  = 1;<br>commit;||
|T3|||start transaction;|
|T4|select * from mvcc where id = 1;|select * from mvcc where id = 1;|select * from mvcc where id = 1;|

在MySQL InnoDB RR 环境中按时序执行以上语句，T4 时间三个事务的返回的val 分别是 v1, v2, v2。

我们只看事务1和事务3，他们事务开始的时间不同，在T4这个同一时间看到的结果是不一样的。因为MVCC的存在，事务1看到的是事务1发生时的快照，事务3看到的是事务3发生时的快照，期间数据被事务2修改过，因此同一时间T4看到的结果不一样。

<!-- more -->

## RC 与 RR

上面的例子强调的是RR 隔离级别，是因为MVCC 只能在RR 或者 RC 两个隔离级别下工作。其他两个隔离级别都和MVCC 不兼容，因为READ UNCOMMITED 总是读取最新的数据行，而不是符合当前事务版本的数据行。而SERIALLIZABLE 则会对所有读取的行都加锁。

在理解了MVCC 之后，理解RC 与 RR 的区别就相当容易了。

### RC
 
READ COMMITED，读已提交。也就是在事务中可以读到别的事务已经提交的数据，假设上面的例子中是RC 隔离级别，那么T4 时间事务1读到的val 值将会是v2。即出现了事务1在T1时读到的是v1，到时间T4时读到的是v2的情况，两次读取的结果不一样，所以RC 也称为不可重复读。

### RR

REPEATABLE READ，可重复读。上面的例子就是RR 级别，T1与T4时间读到的val 值都是v1，两次读取的值是一样的，所以称为可重复读。

## 疑惑

现在，可能你就会有疑惑了。因为MVCC 的存在，我们读到的是快照的数据，在我们平时的库存扣减的业务中不会出现问题吗？

假设有一下数据：

```
mysql> select * from good where id = 1;
+----+-------+
| id | total |
+----+-------+
|  1 |     8 |
+----+-------+
```

现在id 为1 的商品库存剩下 8件;

现有两个事务执行以下操作：

|时间|事务1|事务2|
|:----:|:----:|:----:|
|T1|start transaction;|start transaction;|
|T2|update good set total = total - 1 where id = 1;<br>commit;||
|T3||update good set total = total - 1 where id = 1;<br>commit;|
|T4|select * from good where id = 1;|select * from good where id = 1;|

既然读的是快照，那么T2，T3时间应该读到的都是8啊，然后大家都是8 - 7，两个事务都回写7，这样库存不就不对了吗？

然而，数据库并没有你想的那么笨。update语句的执行逻辑是，先进行**当前读**，然后返回数据并加锁，最后对数据进行更新。所以update的时候读到的数据都是最新的，并不是快照，所以上面的例子执行不会出错，库存最终会变成6。

在一个支持MVCC并发控制的系统中，哪些读操作是快照读？哪些操作又是当前读呢？以MySQL InnoDB为例:

### 快照读

简单的select操作，属于快照读，不加锁。

 - select * from table where ?;

### 当前读

特殊的读操作，插入/更新/删除操作，属于当前读，需要加锁。

- select * from table where ? lock in share mode;
- select * from table where ? for update;
- insert into table values (…);
- update table set ? where ?;
- delete from table where ?;

## MVCC 与 乐观并发控制

多版本并发控制（MVCC）用于解决**读-写冲突**的无锁并发控制，读操作只读该事务开始前的数据库的快照。 这样在读操作不用阻塞写操作，写操作不用阻塞读操作。

乐观并发控制（OCC）用于解决**写-写冲突**的无锁并发控制，认为事务间争用没有那么多，所以先进行修改，在提交事务前，检查一下事务开始后，有没有新提交改变，如果没有就提交，如果有就放弃并重试。适用于低数据争用，写冲突比较少的环境。

假设有以下数据：

```
mysql> select * from t_order where id = 1;
+----+--------+
| id | status |
+----+--------+
|  1 |      1 |
+----+--------+

```

status = 1 代表订单未支付，status = 2 为已支付。

假设RR 隔离级别下有两个事务执行以下操作：

|时间|事务1|事务2|
|:----:|:----:|:----:|
|T1|start transaction;|start transaction;|
|T2|select * from t_order where id = 1;||
|T3|发现未支付;||
|T4|update t_order set status = 2 where id = 1;||
|T5|执行支付逻辑;||
|T6|commit;||
|T7||select * from t_order where id = 1;|
|T8||因为快照读，status还是1，认为还没支付|
|T9||update t_order set status = 2 where id = 1;|
|T10||再次执行支付逻辑;|
|T11||commit;||

出问题了，执行了两次支付逻辑。这种情况就可以利用乐观并发控制来解决，将T4和T9的语句修改为`update t_order set status = 2 where id = 1 and status = 1;`，然后判断影响的数据条数，如果为0，说明没有更新成功，即可能是其它事务已经更新了status为2，这时就不应该执行支付逻辑了。这里的**and status = 1**条件就相当于简单的乐观并发控制。

## 提示

自行修改MySQL隔离级别，然后通过两个session观察数据的变化，就很容易明白MVCC 在这两个隔离级别下的区别了。






