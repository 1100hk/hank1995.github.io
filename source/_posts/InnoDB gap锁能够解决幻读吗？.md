---
title: InnoDB Gap锁如何解决幻读？
date: 2013-12-28 22:04:53
tags:
---

## 1. Gap锁是什么？

ANSI SQL-92根据现象定义了隔离界别，这三种phenomena分别是脏读，不可重复读，和幻读。四种隔离级别分别是1）READ UNCOMMITTED，2）READ COMMITTED，3）REPEATABLE READ，4）SERIALIZABLE。在现代数据库设计中通过2PL和MVCC的结合可以很容易解决脏读和不重复读的问题。实际上在PG中幻读也是通过2PL和MVCC解决的，但是否能够完全解决了，本文不做过多探讨。

回到InnoDB，他在设计上并没有严格遵照隔离级别的要求来设计。InnoDB的只读操作采用快照读，读写操作采用的当前读，当前读可以理解为读已提交的数据。因此要实现RR的隔离级别，仅采用行锁是无法解决幻读的。例如下面这样简单的带有主键表的插入场景，如果只有行锁，T1事务在执行update时会lock record 15，此时进来T2插入record 11和16成功插入并提交，紧接着T1继续执行相同条件的update，不应该被看到的record 11和16也会被更新掉！这种情况数据一致性就无法保障。

```sql
create table tt(a int primary key, b int);
insert into tt values(3,300),(4,400),(5,500),(10,1000),(15,1500);
```

| T1                                | T2                                  |
| --------------------------------- | ----------------------------------- |
| begin;                            |                                     |
| update tt set b=b-100 where a>10; |                                     |
|                                   | begin;                              |
|                                   | insert into tt values(11,1100)；ok  |
|                                   | insert into tt values(16,1600);  ok |
|                                   | commit;                             |
| update tt set b=b-200 where a>10; |                                     |

InnoDB给出的解决方案就是在RR隔离级别下引入一种锁协议——Gap锁。简单的说就是所有的记录上的显式加锁（无论手动还是自动，无论是s锁还是x锁）都自带了buff，会锁定当前记录以及其前面的间隙，在InnoDB种也称作next-lock。

再次重申，Gap协议是为了解决幻读的，不会出现幻读的场景当然也就没必要引入Gap语义，例如如果是唯一键点查，那只需要锁定对应的一条记录即可。在InnoDB RR隔离级别下，这种锁称为REC_NOT_GAP锁。但是如果是一个二级索引点查，那可能就需要锁定某些gap但是不包括记录本身，在InnoDB RR隔离级别下这种锁为gap锁。下一节通过几个具体案例简要说明。

## 2. RR隔离界别锁场景

### 场景1：主表无索引删除锁全表

```sql
create table t (a int);
insert into t values (21),(25),(25),(30);
```

| T1                          | T2                                 |
| --------------------------- | ---------------------------------- |
| begin;                      |                                    |
| select * from t; (快照读)   |                                    |
| delete from t where a = 25; | begin;                             |
|                             | insert into t values (25);（阻塞） |
|                             | insert into t values (10);（阻塞） |
|                             | insert into t values (50);（阻塞） |
|                             | delete from t where a=21;（阻塞）  |
|                             | delete from t where a=30; （阻塞） |
| commit;                     |                                    |
|                             | commit;                            |

![image-1.png](https://github.com/hanke1995/image_store/blob/main/gap_locks/image-1.png?raw=true)

无索引表任何查询都会遍历全表，并对所有记录加显示锁，即上图中的每个x锁定记录本身和记录之前的gap，例如记录25上的x锁锁定(21,25]区间。导致T2的所有插入操作被阻塞。

### 场景2： 非唯一索引

```sql
create table inx(a int, b int, index norinx(a));
insert into inx values(1,100), (1,200),(2,300),(2,400),(10,500);
```

| T1                                                           | T2                                      |
| ------------------------------------------------------------ | --------------------------------------- |
| begin;                                                       |                                         |
| update inx force index(norinx) set b=b+10 where a=2; (二级索引表上，a=2 key加 next-lock锁，10加gap锁) |                                         |
|                                                              | begin;                                  |
|                                                              | insert into inx values(3,600); (阻塞)   |
|                                                              | update inx set b=b-10 where a=10;       |
|                                                              | update inx set b=b-10 where a=1;        |
| commit;                                                      | insert into inx values(1,600); （阻塞） |
|                                                              | delete from inx where a=1;              |
|                                                              | insert into inx values(10,600);         |
|                                                              | commit;                                 |

![image-2.png](https://github.com/hanke1995/image_store/blob/main/gap_locks/image-2.png?raw=true)

非唯一索引因为索引非唯一，所以记录的前后都有可能插入数据，因此除了在记录本身上加显示锁外，还需要在下个记录上设置间隙位，表明只锁定记录之前的gap，不锁定记录本身，如上索引key 10。

### 场景3： 唯一索引

```sql
create table uni(a int, b int);
create unique index unidx on uni(a);
insert into uni values(1,100), (2,200),(3,300),(4,400),(10,500);
```

| T1                                                  | T2                             |
| --------------------------------------------------- | ------------------------------ |
| begin;                                              |                                |
| update uni force index(unidx) set b=b+10 where a=2; |                                |
| 唯一索引点查只锁记录                                |                                |
|                                                     | begin;                         |
|                                                     | insert into uni values(5,600); |
| commit;                                             |                                |

![image-3.png](https://github.com/hanke1995/image_store/blob/main/gap_locks/image-3.png?raw=true)

### 场景4： 主键范围锁

```sql
create table tt(a int primary key, b int);
insert into tt values(1,100), (2,200);
insert into tt values(3,300),(4,400),(5,500),(10,1000),(15,2000);
```

| T1                                | T2                                  |
| --------------------------------- | ----------------------------------- |
| begin;                            |                                     |
| update tt set b=b-100 where a>10; |                                     |
|                                   | begin;                              |
|                                   | insert into tt values(6,600)；ok    |
| commit                            | insert into tt values(11,600); 阻塞 |
|                                   | commit;                             |

![image-4.png](https://github.com/hanke1995/image_store/blob/main/gap_locks/image-4.png?raw=true)

### 场景5： 主键索引+非唯一索引

```sql
create table pri_nor(a int primary key, b int, c int, index norinx(b));
insert into pri_nor values(1, 1, 100), (2 ,1, 200), (3, 5 , 300), (4, 5, 400), (5, 15, 500);
```

| T1                                                       | T2                                                     |
| -------------------------------------------------------- | ------------------------------------------------------ |
| begin;                                                   |                                                        |
| update pri_nor force index(norinx) set c=c+10 where b=1; |                                                        |
|                                                          | begin;                                                 |
|                                                          | insert into pri_nor values(6, 6, 600); (无阻塞)        |
|                                                          | insert into pri_nor values(6, 2, 600); //阻塞区间[1,5) |

![image-5.png](https://github.com/hanke1995/image_store/blob/main/gap_locks/image-5.png?raw=true)

### 场景6: 主键索引+唯一索引

```sql
create table pri_uni(a int primary key, b int, c int);
create unique index unidx on pri_uni(b);
insert into pri_uni values(1, 1, 100), (2 ,2, 200), (5, 5 , 300), (10, 10, 400), (15, 15, 500);
```

| T1                                                      | T2                                   |
| ------------------------------------------------------- | ------------------------------------ |
| begin;                                                  |                                      |
| update pri_uni force index(unidx) set c=c+10 where b=2; |                                      |
| update pri_uni force index(unidx) set c=c+10 where b=5; | begin;                               |
|                                                         | insert into pri_uni values(3,3,300); |
|                                                         |                                      |

![image-6.png](https://github.com/hanke1995/image_store/blob/main/gap_locks/image-6.png?raw=true)

如果只是走主键索引，不会对二级索引加锁。

```sql
update pri_uni force index(PRI) set c=c+10 where a=2;
```

![image-7.png](https://github.com/hanke1995/image_store/blob/main/gap_locks/image-7.png?raw=true)

## 3. GAP锁能够完全避免幻读吗？

### RR隔离级别仍然幻读？

```sql
create table t (a int);
insert into t values (21),(25),(25),(30);
```

| T1                                 | T2                         |
| ---------------------------------- | -------------------------- |
| begin;                             |                            |
| select * from t; (快照读)          |                            |
|                                    | begin;                     |
|                                    | insert into t values (25); |
| delete from t where a = 25; (阻塞) |                            |
|                                    | commit;                    |
| commit;                            |                            |

![image-8.png](https://github.com/hanke1995/image_store/blob/main/gap_locks/image-8.png?raw=true)

![image-9.png](https://github.com/hanke1995/image_store/blob/main/gap_locks/image-9.png?raw=true)

上面一个例子结果看T1最后删除了3条记录，但是理论上T1应该只删除2条记录。同样的案例在PG中只会删除两条记录，pg的读写事务也使用快照读的方式，保证了语义的正确性。因此innodb的repeatable read并没有真正意义上实现可重复读，仅仅是只读操作使用快照读方式解决non-repeatable read问题。而写操作则采用的read commit的隔离界别，没有避免幻读的问题。

在上例中innodb认为T1事务的真正开始时间是从只读事务转读写事务开始。hmm，其实这么解释也有点牵强，毕竟innodb_trx统计的事务开始时间也包括只读事务。

![image-10.png](https://github.com/hanke1995/image_store/blob/main/gap_locks/image-10.png?raw=true)

### innodb RR可序列化更新，但是违背了可重复读语义

对同一行数据更新的场景肯定不能使用多版本，否则会更新丢失。事务开始时间不同，按照可重复读的语义，两个事务的修改对方是看不到的，所以对同一行数据的更新是不可序列化的。不可序列化应该失败掉。

```sql
create table tt(a int primary key, int b);
insert into tt values(1,100), (2,200);
```

| T1                                | T2                                       |
| --------------------------------- | ---------------------------------------- |
| begin;                            |                                          |
| update tt set b=b-10 where a = 1; |                                          |
|                                   | begin;                                   |
|                                   | update tt set b=b+20 where a=1; （阻塞） |
| commit                            |                                          |
|                                   | commit;                                  |

如果此时更新使用快照读，像场景一一样，势必会出现更新丢失的问题，t1和t2读到同一时刻的b，但是结果却只体现了一个事务的操作。因此PG中将这种行为定义为不可串行化的语义，报错。

```
ERROR: could not serialize access due to concurrent update
```

| T1                                | T2                                                           |
| --------------------------------- | ------------------------------------------------------------ |
| begin;                            |                                                              |
| update tt set b=b-10 where a = 1; |                                                              |
|                                   | begin;                                                       |
|                                   | update tt set b=b+20 where a=1; （阻塞）                     |
| commit；                          | ERROR: could not serialize access due to concurrent update   |
|                                   | update tt set b=b+20 where a=1;  （重试）                    |
|                                   | ERROR:  current transaction is aborted, commands ignored until end of transaction block |
|                                   | rollback；此时必须回滚事务                                   |

而innodb则使用的是锁等待方式，以保证串行化执行，当然也违背了可重复读的语义。