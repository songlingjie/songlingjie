---
title: 关于MySQL锁的一些思考
date: 2022-03-28 21:28:08
tags: MySQL,Lock
categories: 
  - 技术笔记
  - MySQL
banner_img: /img/mysql_lock.jpg
index_img: /img/mysql_lock.jpg
---

### 锁是什么以及为什么要加锁？

##### 1.1**锁**是什么

​		所谓的**锁**其实是一个内存中的结构，如果有锁等待的话在 information_schema.INNODB_LOCKS 看到 大致的结构是：

```json
{
  "lock_id":"锁id",
  "trx_id":"事务id，表示这是那个事物生成的",
  "lock_type":"锁的类型",
  "lock_data":"加锁的记录"
}
```



##### 1.2锁的作用

> 锁是用来解决事物并发问题带来的一些问题，如果没有并发那么也就不需要锁了

事物并发分为3种情况，锁是解决并发带来的问题一种方案

- 读-读并发

  这种情况不会对数据造成影响，是允许出现的

- 写-写并发

  这种情况可能会出现脏写，任何一种隔离级别都不允许这种情况出现，解决这个问题的办法就是<font color=#FF000>**加锁**</font> ， 对同一条记录修改时需要排队执行

  > 脏写：事务修改了别的事物未提交的数据。

- 读-写，写-读并发

  这些情况下带来的问题有带来的问题有<font color=#FF000>**脏读、不可重复读、幻读**</font>。

  > 关于脏读、不可重复读、幻读的定义
  >
  > - 脏读：可以读取到未提交事物的数据
  >
  >   | 时间 | session1                                   | session2                                                     |
  >   | ---- | ------------------------------------------ | ------------------------------------------------------------ |
  >   | t1   | begin;                                     | SET TRANSACTION ISOLATION LEVEL   READ UNCOMMITTED;<br/>begin; |
  >   | t2   | INSERT INTO `t4` (`id`, `a`) VALUES(2, 3); |                                                              |
  >   | t3   |                                            | select * from t4;#是可以看到session1 未提交的写入数据        |
  >
  >   
  >
  > - 不可重复读：前后两次查询到记录的值不一样
  >
  >   | 时间 | session1                                                     | session2                        |
  >   | ---- | ------------------------------------------------------------ | ------------------------------- |
  >   | t1   | SET TRANSACTION ISOLATION LEVEL   READ COMMITTED;<br />begin; |                                 |
  >   | t2   | select * from t4 where id =1; # id=1 a= 1                    |                                 |
  >   | t3   |                                                              | update t4 set a=11 where id =1; |
  >   | t4   | select * from t4 where id =1; # id=1 a= 11                   |                                 |
  >
  >   
  >
  > - 幻读：出现在查询结果集中但不在先前查询结果集中的行，就是一个事务在前后两次查询同一个范围的时候，后一次查询看到了前一次查询没有记录。 
  >
  >   [MySQL文档对幻读的解释]: https://dev.mysql.com/doc/refman/5.7/en/glossary.html#phantom
  >   [幻读和写偏斜的理解]: https://github.com/Vonng/ddia/blob/master/ch7.md#写入偏斜与幻读
  >
  >   
  >
  >   | 时间 | session1                                                     | session2                                       |
  >   | ---- | ------------------------------------------------------------ | ---------------------------------------------- |
  >   | t1   | SET TRANSACTION ISOLATION LEVEL   READ COMMITTED;<br />begin; |                                                |
  >   | t2   | select * from t4 where id >=1; #一条记录 id=1 a= 11          |                                                |
  >   | t3   |                                                              | INSERT INTO `t4` (`id`, `a`)<br/>VALUES(2, 2); |
  >   | t4   | select * from t4 where id >=1; #两条记录 id=1 a= 11，id=12a= 2 |                                                |
  >
  >   

​		解决<font color=#FF000>**脏读、不可重复读、幻读**</font>方法

- 读操作利用 多版本并发控制（MVCC，Multi Version Concurrency Control ），写操作加锁。RC 和RR 在不加锁读取记录时用了MVCC，也就解决了脏读的问题。其中RC 每次查询是生成一个视图，也就导致了RC 不支持重复读和没有解决幻读，RR 是在事物启动时生成一个视图，也同时解决了可重复读和幻读

  >所谓的MVCC 是基于undo log的版本链来表示每一行都有多个版本。
  >
  >生成视图时（生成视图的时机和隔离级别有关系）记录当前活跃的事物id、已提交事务的最大id、未开始的事物id。
  >
  >在事务中取一行记录取出记录中的事务id 判断是否 <= 已提交事务的最大id，如果不符合要求，根据记录中的undo log id 取之前的版本。实际情况比这些要复杂，比如果  <= 已提交事务的最大id 但是这个事物也在当前活跃的事物id里。这些大家下来可以讨论一下，这里不做过多解释了

- 读写都加锁，如果某些场景我们的读操作必须读取最新值就采用这种方法，但是会阻塞写操作，性能低。脏读是因为读取别的事务未提交的记录，如果在读的时候加锁，那么别的事物无法修改更新这条记录，也就解决了脏读和不可重复读。对于幻读是指当前事物读取了一个范围内的记录，别的事物又在这个范围内写入记录，这种情况解决起来需要加 间隙锁，下边会介绍。

​		

------



### **锁的类型**

##### 2.1共享锁和排他锁

​		InnoDB有两种类型的锁， 共享（S）锁和排他（X）锁。记录锁和表锁都有这两种类型

- 共享 ( S) 锁允许持有该锁的事务读取一行 。加锁语句 SELECT ... LOCK IN SHARE MODE ，这是针对记录锁的加锁方式

- 独占 ( X) 锁允许持有该锁的事务更新或删除一行 。加锁语句 SELECT ... FOR UPDATE

  | 兼容性 | S锁  | X锁  |
  | ------ | ---- | ---- |
  | S锁    | ✅    | ❎    |
  | X锁    | ❎    | ❎    |


##### 2.2意向锁

​		在某一行上加共享/排他锁，可以称为记录锁（下边会介绍记录锁），意向锁是表级别的锁，当我们再记录上加锁时，会在表级别上加一个意向锁，意向锁之间不会冲突

>意向锁的主要用在 加表锁时可以快速判断这个表中有没有针对记录的共享锁或者排他锁，这不是重点，简单了解一下吧。
>
>lock table t write 给表t手动加一个 排他锁
>
>lock table t read  给表t手动加一个 共享锁

​		有两种类型的意图锁：

- 意向共享锁 IS 表示事务打算在表中的各个行上设置 共享 锁 *。*

- 意向排他锁 IX表示事务打算对表中的各个行设置排他锁。

  | 兼容性 | S    | X    | IS   | IX   |
  | ------ | ---- | ---- | ---- | ---- |
  | S      | ✅    | ❎    | ✅    | ❎    |
  | X      | ❎    | ❎    | ❎    | ❎    |
  | IS     | ✅    | ❎    | ✅    | ✅    |
  | IX     | ❎    | ❎    | ✅    | ✅    |



##### 2.3AUTO-INC Locks

​		自增锁是一种特殊的表级锁，比入说我们可以为表的某个列添加`AUTO_INCREMENT`属性，之后在插入记录时，可以不指定该列的值，系统会自动为它赋上递增的值。采用`AUTO-INC`锁，也就是在执行插入语句时就在表级别加一个`AUTO-INC`锁，然后为每条待插入记录的`AUTO_INCREMENT`修饰的列分配递增的值，<font color=#ff000>在该语句执行结束后，再把`AUTO-INC`锁释放掉。不会等到事物提交才释放锁。减少别的事务锁等待时间。</font>这样一个事务在持有`AUTO-INC`锁的过程中，其他事务的插入语句都要被阻塞，可以保证一个语句中分配的递增值是连续的。

##### 2.4记录锁

​		2.2 和 2.3 是针对于表的，接下来介绍的锁是针对于记录的锁

​		记录锁官方的类型是：LOCK_REC_NOT_GAP，记录锁是有X 和 S 两种类型 分别用 FOR UPDATE 和 LOCK IN SHARE MODE 加锁

##### 2.5间隙锁

​		上边我们说过MySQL 在可重复读下可以解决幻读的问题，解决方案有两种，可以使用`MVCC`方案解决，也可以采用加锁方案解决。但是在使用`加锁`方案解决时有个大问题，就是事务在第一次执行读取操作时，那些幻影记录尚不存在，我们无法给这些幻影记录加上记录锁，所以有了间隙锁，官方的类型名称为：<font color=#ff000>LOCK_GAP</font>，比如我们要给c=5 加 间隙锁，锁定的范围就是 （0，5）这个区间，不包含id=0 和 id=5 这样两行。下边这个表t中总共有 （-∞，0），（0，5），（5，10），（10，15），（15，+∞） 五个区间。

​		这个gap锁的仅仅是为了防止插入幻影记录而提出的，虽然有共享gap锁和独占gap锁这样的说法，但是它们起到的作用都是相同的。而且如果你对一条记录加了gap锁（不论是共享gap锁还是独占gap锁），并不会限制其他事务对这条记录加记录锁或者继续加gap锁。

| Id   | c    | d    |
| ---- | ---- | ---- |
| 0    | 0    | 0    |
| 5    | 5    | 5    |
| 10   | 10   | 10   |
| 15   | 15   | 15   |

##### 2.6 Next Key锁

​		Next Key 本质就是一个记录锁和一个gap锁的合体，官方类型是：LOCK_ORDINARY 它既能保护该条记录，又能阻止别的事务将新记录插入被保护记录前边的间隙。比如给id=10这条记录加 Next Key锁 锁定的范围是 （5，10）这个区间加Id=10这条记录，但不包含Id=5 这条记录。next key 锁是前开后闭的。这也是加锁的基本单位。

##### 2.7插入意向锁

​		我们说一个事务在插入一条记录时需要判断一下插入位置是不是被别的事务加了所谓的gap锁（next-key锁也包含gap锁），如果有的话，插入操作需要等待，直到拥有gap锁的那个事务提交。InnoDB规定事务在等待的时候也需要生成一个锁，表明有事务想在某个间隙中插入新记录，但是现在在等待。这种类型的锁命名为Insert Intention Locks，官方的类型名称为：LOCK_INSERT_INTENTION，我们也可以称为插入意向锁



[MySQL文档对于锁的介绍]: https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html#innodb-intention-locks



------



### 可重复读隔离级别下的加锁

准备 表t 

```sql
CREATE TABLE `t` (
  `id` int(11) NOT NULL,
  `c` int(11) DEFAULT NULL,
  `d` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `c` (`c`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

INSERT INTO `t` (`id`, `c`, `d`)
VALUES
	(0, 0, 0),
	(5, 5, 5),
	(10, 10, 10),
	(15, 15, 15),
	(20, 20, 20),
	(25, 25, 25);

```



##### **3.1加锁的基本单位是 next key**

|      | session1                                 | session2                                                     |
| ---- | ---------------------------------------- | ------------------------------------------------------------ |
| t1   | begin;                                   |                                                              |
| t2   | select * from t where c = 10 for update; | begin;                                                       |
| t3   |                                          | INSERT INTO `t` (`id`, `c`, `d`) VALUES	(8, 8, 8); <font color=#ff000>blocked</font> |

```sql
select * from information_schema.INNODB_TRX;	 
```

![image-20220415172906715](https://tva1.sinaimg.cn/large/e6c9d24ely1h1ajbo3xwuj20ty02nt92.jpg)

​	

```sql
select * from information_schema.INNODB_LOCK_WAITS;
```

​	![image-20220415172942467](https://tva1.sinaimg.cn/large/e6c9d24ely1h1ajc85fluj20bn02zgll.jpg)



```sql
select * from information_schema.INNODB_LOCKS;
```

​	![image-20220415173023393](https://tva1.sinaimg.cn/large/e6c9d24ely1h1ajcxxlyvj20k20353yp.jpg)



##### **3.2访问到的数据会加锁**

​	  如下表中这个例子，会话一 给c=5加了共享锁，但是只给二级索引加了锁，主键索引并没有加锁因为只查询了字段c不需要查主键索引，走了索引覆盖 。

​		会话二可以修改成功是因为只更新了主键索引没有更新 索引c。

​		会话三写入失败是因为会话一c=5 这一行在索引c上 有Next key Lock。锁住了 （0,5）的区间，导致写入失败。

|      | session1                                                     | session2                         | session3                                                     |
| ---- | ------------------------------------------------------------ | -------------------------------- | ------------------------------------------------------------ |
| t1   | begin;                                                       |                                  |                                                              |
| t2   | select c from t where c=5 lock in share mode; #这里如果换成for update 也会对主键索引加锁，相应的会话二也会阻塞 | begin;                           | begin;                                                       |
| t3   |                                                              | update t set d = 6 where id=5; ✅ | INSERT INTO `t` (`id`, `c`, `d`) VALUES (3, 3, 3); <font color=#ff000>blocked</font> |

##### **3.3唯一索引加锁**

- 等值查询 ：next key 退化为 记录锁

|      | session1                                       | session2                                                     |
| ---- | ---------------------------------------------- | ------------------------------------------------------------ |
| t1   | begin;                                         |                                                              |
| t2   | select * from t where id=5 lock in share mode; | begin;                                                       |
| t3   |                                                | select * from t where id=10 for update;✅<br/>INSERT INTO `t` (`id`, `c`, `d`) VALUES (7, 7, 7);✅ |



- 范围查询：满足条件的记录都会加next key  会访问到第一个不满足条件的数据位置

|      | session1                                          | session2                                                     |
| ---- | ------------------------------------------------- | ------------------------------------------------------------ |
| t1   | begin;                                            |                                                              |
| t2   | select * from t where id >= 5 lock in share mode; | begin;                                                       |
| t3   |                                                   | select * from t where id = 10 for update;<br/><font color=#ff000>blocked</font> |



- 唯一二级索引列无记录查询：会对 大于查询条件的第一个值 加 next key，比如 查出c=3 记录不存在 会对 id=5 这一行加上next key

```sq
ALTER TABLE t DROP INDEX c, ADD UNIQUE KEY uniq_c (c);
```

|      | session3                                         | session2                                                     |
| ---- | ------------------------------------------------ | ------------------------------------------------------------ |
| t1   | begin;                                           |                                                              |
| t2   | select * from t where c <= 3 lock in share mode; | begin;                                                       |
| t3   |                                                  | select * from t where c = 5 for update;<br/><font color=#ff000>blocked</font> |

```sql
ALTER TABLE t DROP INDEX uniq_c, ADD KEY c (c);
```



##### **3.4普通索引加锁**

- 等值查询：向右遍历到第一个不满足条件的值 next key 退化成 gap 

|      | session1                                      | session2                                                     |
| ---- | --------------------------------------------- | ------------------------------------------------------------ |
| t1   | begin;                                        |                                                              |
| t2   | select * from t where c=5 lock in share mode; | begin;                                                       |
| t3   |                                               | select * from t where c=10 for update;    ✅<br/>INSERT INTO `t` (`id`, `c`, `d`) VALUES (7, 7, 7);    <font color=#ff000>blocked</font> |

- 范围查询：唯一索引一样

- 无记录时： 唯一索引一样



##### **3.5 加锁时使用limit**

​		还是针对于表t，再写入一条记录 

```sql
INSERT INTO `t` (`id`, `c`, `d`) VALUES (10, 10, 10);
```

​		此时索引c 的数据是

| c    | 0    | 5    | 10   | 10   | 15   | 20   | 25   |
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| id   | 0    | 5    | 10   | 30   | 15   | 20   | 25   |

​		会话二不会阻塞的原因是会话一 在语句中明确的使用limit 2，索引只对c=10 两行加了锁。同时已印证了只对访问到的记录加锁

|      | session1                                        | session2                                                 |
| ---- | ----------------------------------------------- | -------------------------------------------------------- |
| t1   | begin;                                          |                                                          |
| t2   | select * from t where c=10 for update  limit 2; | begin;                                                   |
| t3   |                                                 | INSERT INTO `t` (`id`, `c`, `d`) VALUES (13, 13, 13);  ✅ |

##### 3.6加锁规则总结

- [ ] 加锁的基本单位是 next key
- [ ] 访问到的数据会加锁
- [ ] 唯一索引等值查询 next key 退化为 记录锁
- [ ] 普通索引等值查询 向右遍历到第一个不满足条件的值 next key 退化成 gap 
- [ ] 范围查询时满足条件的记录都会加next key  会访问到第一个不满足条件的数据位置
- [ ] 无记录的范围查询 会对 大于查询条件的第一个值 加 next key
