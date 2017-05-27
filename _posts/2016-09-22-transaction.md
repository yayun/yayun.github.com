---
layout: post
title: "数据库事务学习笔记"
---

### 1. 数据库事务的基本概念 

当事务被提交给了DBMS（数据库管理系统），则DBMS需要确保该事务中的所有操作都成功完成且其结果被永久保存在数据库中, 
如果事务中有的操作没有成功完成，则事务中的所有操作都需要被回滚，回到事务执行前的状态;
同时，该事务对数据库或者其他事务的执行无影响，所有的事务都好像在独立的运行。
（A向B汇钱：1：A的钱减少  2: B的钱增加 如果扣掉A的钱之后程序挂掉？事务可以保证这两个操作要么全发生要么就像没有发生一样）

*四个基本特性（ACID）*

- Atomicity: 原子性，事务作为一个整体被执行，包含在其中的对数据的操作要么全部执行要么都不执行
- Consistency：在事务开始之前和事务结束以后，数据库的完整性没有被破坏
- Isolation： 多个事务并发执行时，一个事务的执行不应影响其他事务的执行
- Durability：已被提交的事务对数据库的修改应该永久保存在数据库中

### 2. 数据库隔离级别(Isolation)

ISO 根据症状定义了四种不同的隔离级别:

- Serializable：两个事务完全是串行的，一个执行完另一个才能执行（读select也是会上锁的）
- Repeatable Read：可以保证两个事物重复读到的数据是完全一样的（可重复读）
- Read Committed：只会读到commit的数据
- Read Uncommitted： 在事务没有提交之前，（数据库操作还没有commit的时候），其他的事务也可以读到

| 数据库隔离级别| 脏读 | 不可重复读 | 幻读 |
| :------| :------| :------| :------| 
| Serializable(可串行化) | 不可能 | 不可能 | 不可能 |
| Repeated Read（可重复读） | 不可能 | 不可能 | 可能|
| Read committed （提交读）| 不可能 | 可能 | 可能 |
| Read uncommitted (未提交读) | 可能 | 可能 | 可能 |
{:.table}

可重复读: 

在一个事务中读到的同一份数据， 数据的内容相同。 RC模式下不可重复读，也就是一个事务中的两次读
有可能第二次读到的和第一次读到的不相同。
因为RC模式读的都是别人commit的数据。RR 模式是可以重复读的，也就是
说一个事务的两次读都是相同的，是通过打快照(快照读)和next-key lock （针对当前读，gap lock + raw lock）的方式实现的

比如 A B 两个事务，A 事务执行了两次 query 同一条数据 ，在 A 执行第二次 query 之前执行第一次 query 之后 B update 了一下这条数据，
这时候 A 两次读到的数据不相同，这就叫做不可重复读

幻读:

同一个事务，两次同样的范围查询返回的结果集合不一致

比如 A B 两个事务， A 事务会执行两次同样的返回查询，B 事务会向 A 指定的范围内插入数据，A 事务的两次范围查询返回
的结果应保持一致，而不受 B 事务的影响.

其中 RR 和 RC 是比较常用的隔离级别, 
理论上数据库的隔离级别越高，并发性能越低，隔离性越好。
我们想要在保证业务正确性的前提下尽可能的提高并发，
就要小心的使用数据库的隔离级别和select for update 等锁操作.

### 3. MVCC

但是 ISO 的定义是面向基于锁的并发控制，现代数据库为了更好的性能一般都会基于 MVCC 来实现事务了。
而隔离级别的行为也与前面的定义存在一定的出入。

MVCC 是多版本并发控制的缩写，相对于传统数据库的锁并发控制。它会在事务开始时打快照，即使在事务执行
期间数据被其他事务修改，旧版本的数据仍会保留，并在本事务中一致的展现.

基于 MVCC 实现的 InnoDB ，已经不存在幻读现象了。
但是 RR 仍不如 Serializable 强，是因为会出现写覆盖现象.

### 4. RR 隔离级别下 write skew 现象

| id | uid | amount |
| :------| :------| :------| :------| 
| 1 | 1 | 20 |
{:.table}

id是主键 uid是非唯一索引
如果 A B 两个事务同时给 uid 为 1 的用户转账。

{% highlight javascript %}
A 事务：
amount = select amount from acount where uid = 1;
update acount set amount = amount + 20;

B 事务：
amount = select amount from acount where uid = 1;
update acount set amount = amount + 30;
{% endhighlight %}

这种情况就有可能出现 A B 两个事务完成之后，uid 为 1 的用户的amount为 50 而不是 70 的情况, 这种情况就叫做写覆盖。

为什么会出现这种现象呢？A B 事务两个事务同时读取到当前 amount 是 20 ，又因为 select 操作是快照读 读取之后不能保证其他
并发事务不能修改当前记录。就有可能出现后update的那个事务覆盖前一个事务操作结果的情况。


怎样避免这种现象呢？解决办法就是在 select 的时候上锁，
防止其它事务并发的修改当前事务访问的数据

```
select amount from acount where uid = 1 for update;
update acount set amount = amount + 20;
```

针对这种情况也可以尝试将转账变为一次原子操作

```
update account set amount = amount + 20 where uid = 1
```

针对 update 操作，mysql 会根据 where 条件，读取第一条满足条件的记录，然后 InnoDB 引擎会将第一条记录返回并加锁（ current read ），
待 Mysql Server 收到这条加锁记录之后，会再发起一个 Update 请求，更新这条记录。一条记录操作完成，再读取下一条记录，直至没有满足
条件的记录为止。update 返回操作的行数。

### 5. RC 模式下的 Phantom Read 现象

之前在线上有遇到过一个类似的并发问题, 不过原因来自于 InnoDB 中 RC 模式的 phantom read 现象。

| id | uid | date |
| :------| :------| :------| 
| 1 | 1 | 2016-08-04 |
{:.table}

```
checkin_record  = select id from checkin_records where uid = 1 and date = "today" for update;
if not checkin_record:
    insert into checkin_records(uid, date) values(1, today)
```

过去我们的数据库隔离级别是 RR 模式，这段代码一直安然无恙，但是迁移数据库上云之后，
表里面出现了重复签到的现象。经过调查发现云数据库的隔离级别是 RC 。因为 InnoDB在 RR 模式中
select for update 对不存在的数据行会上 gap lock，因而不会受到其它事务并发插入的干扰，而 RC 
模式在这一场景下不会上gap lock，会允许两个事务中的select for update 查询同时返回空, 也就是
说存在 phantom read 现象。

当然类似的问题应该使用数据库的唯一性约束来解决更为妥当。
