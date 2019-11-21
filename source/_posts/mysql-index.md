---
title: SQL调优实例与总结
date: 2018-11-07 12:00:00
top: false
cover: false
toc: true
summary: 这是一篇从实例出发的sql调优教程，文章开头有生成数据的语句。一键生成环境，快速上手调优。
categories: Mysql
tags:
  - Mysql
---
# SQL调优实例与总结

> Author：[Voluee](https://github.com/Voluee) 
> MySQL版本：5.5
>
> 这是一篇从实例出发的sql调优教程，文章开头有生成数据的语句。一键生成环境，快速上手调优。

## 生成数据

```sql
drop table if exists student;
 
create table if not exists student
(
    s_id int not null auto_increment primary key,
    s_name varchar(50) not null
) engine = innodb default charset = utf8;
 
 
drop table if exists course;
 
create table if not exists course
(
    c_id int not null auto_increment primary key,
    c_name varchar(10) not null
) engine = innodb default charset = utf8;
 
 
drop table if exists sc;

create table if not exists sc
(
    sc_id int not null auto_increment primary key,
    s_id int not null,
    c_id int not null,
    score int not null
) engine = innodb default charset = utf8;
 
-- 以下数据表对应的数据记录数
--      10  course
--  70,000  student
-- 700,000  sc
-- 根据以上要求准备测试数据
 
INSERT INTO course ( c_id, c_name ) 
	SELECT 
		NULL AS c_id,
		concat( 'course', ID ) AS c_name 
	FROM
		information_schema.`COLLATIONS` 
	ORDER BY ID ASC 
	LIMIT 0, 10;

-- 创建7w条空数据
INSERT INTO student ( s_id, s_name ) 
	SELECT 
		NULL AS s_id,
		'' AS s_name 
	FROM
		( SELECT 1 AS column_order_id FROM information_schema.`COLUMNS` LIMIT 0, 3500 ) AS t
		CROSS JOIN 
			( SELECT 1 AS collation_order_id FROM information_schema.`COLUMNS` LIMIT 0, 20 ) AS t2;
     
-- 给7w条数据赋值
UPDATE student 
SET s_name = concat( 'student', s_id ) 
WHERE
	s_name = '';
 
-- 创建70w条数据，10 * 7w 
INSERT INTO sc ( sc_id, s_id, c_id, score ) 
	SELECT 
		NULL AS sc_id,
		t2.s_id,
		t.c_id,
		ceiling( rand( ) * 100 ) AS score 
	FROM
		course AS t
		CROSS JOIN student AS t2;
```

## 单索引调优过程

查询目的：查找语文考100分的考生(c_id = 1)

### in查询

查询语句如下：

```sql
SELECT
	s.* 
FROM
	student s 
WHERE
	s.s_id IN ( SELECT sc.s_id FROM sc sc WHERE sc.c_id = 1 AND sc.score = 100 )
```

执行时间：`30118.381s`，没有实际测过（参考来的），反正很慢很慢很慢就是了。

看一下执行计划（不懂的[点击详解](http://132.232.92.124/2019/06/06/explain/)）：

```sql
EXPLAIN SELECT s.* FROM student s WHERE s.s_id IN ( SELECT sc.s_id FROM sc sc WHERE sc.c_id = 1 AND sc.score = 100 )
```

<img src="https://i.postimg.cc/3Nf0yVdL/image.jpg" width="800"/>

查看结果发现没有用到索引，type全是ALL，那么首先想到的就是建立一个索引，建立索引的字段当然是在where条件的字段。

先给sc表的c_id和score建个索引：

```sql
CREATE index sc_c_id_index on sc(c_id);
CREATE index sc_score_index on sc(score);
CREATE index sc_s_id_index on sc(s_id);
```

再次执行上述查询语句，时间为：`0.983s`，明显变快很多倍

看一下执行计划：

<img src="https://i.postimg.cc/NfLHhszZ/in.jpg" width="800"/>

再查看引擎优化后的sql（EXPLAIN EXTENDED SQL; SHOW WARNINGS;）：

```sql
SELECT
	`test`.`s`.`s_id` AS `s_id`,
	`test`.`s`.`s_name` AS `s_name` 
FROM
	`test`.`student` `s` 
WHERE
	< in_optimizer > (
	`test`.`s`.`s_id`,< EXISTS > (
SELECT
	1 
FROM
	`test`.`sc` 
WHERE
	(
	( `test`.`sc`.`score` = 100 ) 
	AND ( `test`.`sc`.`c_id` = 2 ) 
	AND ( < CACHE > ( `test`.`s`.`s_id` ) = `test`.`sc`.`s_id` ) 
	) 
	) 
```

从执行计划可以看到MySQL将sql优化成了EXISTS子句，即先执行外层查询，再执行里层查询，这样久要循环`69987 * 4`次，从执行计划可以看到只走了sc的s_id的索引，耗时相对较长，所以下面改用连接查询测试。

### 连接查询

查询语句：

```sql
SELECT
	s.* 
FROM
	student s
	INNER JOIN sc sc ON sc.s_id = s.s_id 
WHERE
	sc.c_id = 1 
	AND sc.score = 100;
```

先删除之前的索引，执行上述查询语句，耗时：`0.213s`，效率非常快，看看执行计划，

<img src="https://i.postimg.cc/nhD9zdZk/image.jpg" width="800"/>

可以看到执行计划先走的student表的主键索引，再走sc全表，不加索引是in查询的`5倍`效率

我们给sc的c_id和score建立个索引试试

```sql
CREATE index sc_c_id_index on sc(c_id);
CREATE index sc_score_index on sc(score);
```

<img src="https://i.postimg.cc/bv0G0sZS/image.jpg" width="800"/>

执行上述查询语句，耗时：`0.054s`，从执行计划可以看到，走了c_id和score两个索引一共只有`1396 * 1`的查询次数，查询速度提升了4倍。相比in查询提高了`20倍`效率。

附图SQL语句执行顺序：

`FROM > ON > JOIN > WHERE > GROUP BY> WITH > HAVING > SELECT > DISTINCT > ORDER BY> LIMIT`

<img src="https://i.postimg.cc/DZ2SxnGj/image.png" width="300"/>

## 联合索引的调优

<img src="https://i.postimg.cc/fLBJFpjw/image.jpg" width="800"/>

- 其实从执行计划的type就能看出来sc的type是`index_merge`，这里用到了`intersect`并集操作，即两个索引同时检索的结果再求并集，再看字段`score`和`c_id`的区分度，单从一个字段看从SC表检索，c_id = 2检索的结果是`142158`，score = 84的结果是`6851`，而`c_id = 2 and score = 84` 的结果是`1390`，即这两个字段联合起来的区分度是比较高的，因此建立联合索引查询效率将会更高。
- 另外一个角度看，该表的数据是70w，以后会更多，就索引存储而言，都是不小的数目，随着数据量的增加，索引就不能全部加载到内存，而是要从磁盘去读取，这样索引的个数越多，读磁盘的开销就越大，因此根据具体业务情况建立`多列的联合索引`是必要的，那么我们来试试吧。

```sql
alter table SC drop index sc_c_id_index;
alter table SC drop index sc_score_index;
create index sc_c_id_score_index on SC(c_id,score);
```

<img src="https://i.postimg.cc/qqqzdwj2/image.jpg" width="800"/>

执行上述查询语句，消耗时间为：`0.029s`，这个速度还是可以接受的，快了`一倍`，随着数据量越大效果越明显。

### 覆盖索引

就是查询的列都建立了索引`至少索引要包含查询的列`，这样在获取结果集的时候不用再去磁盘获取其它列的数据，直接返回索引数据即可

简洁说就是`select`尽量不返回`*`而是返回需要的有索引的列，如下面所示，第二条效率大于第一条

```sql
SELECT * FROM employees WHERE name= '小明' AND age = 22 AND POSITION ='Java';
SELECT NAME, age, POSITION FROM employees WHERE NAME= '小明' AND age = 22 AND POSITION ='Java';
```

### 最左前缀

假设 a、b、c 为联合索引，即`create index index_a_b_c on table(a,b,c);`

| WHERE 语句                                                   | 索引使用情况                          |
| ------------------------------------------------------------ | ------------------------------------- |
| WHERE a = '小明'                                             | 使用到 a                              |
| WHERE a = '小明' AND b = '李磊'                              | 使用到 a 、b                          |
| WHERE a = '小明' AND b = '李磊' AND c = '韩梅梅'             | 使用到 a、b、c                        |
| WHERE b = '李磊' 或者 WHERE b = '李磊' AND c = '韩梅梅' 或者 WHERE c = '韩梅梅' | 没有用到                              |
| WHERE a = '小明' AND c = '韩梅梅'                            | a 用到了，c 没有用到，因为 b 中间断了 |
| WHERE a = '小明' AND b > '李磊' AND c = '韩梅梅'             | a、b 用到了，c 不能用在范围后         |
| WHERE a = '小明' AND b = '李磊%' AND c = '韩梅梅'            | 使用到 a、b、c                        |
| WHERE a = '小明' AND b = '%李磊' AND c = '韩梅梅'            | 只用到 a                              |
| WHERE a = '小明' AND b = '%李磊%' AND c = '韩梅梅'           | 只用到 a                              |
| WHERE a = '小明' AND b = '李%磊%' AND c = '韩梅梅'           | 使用到 a、b、c                        |

## 总结

### 索引优化相关

- 表的主键、外键必须有索引
- 数据量超过300的表应该有索引
- 索引应该建在选择性高的字段上
- 排序字段上需要建立索引（`ORDER BY`）
- 分组字段上需要建立索引（`GROUP BY`）
- 索引应该建在小字段上，对于大的文本字段甚至超长字段，不要建索引
- WHERE条件字段上需要建立索引（`WHERE`）
- 多表连接的字段上需要建立索引，这样可以极大的提高表连接的效率（`LEFT JOIN`、`RIGHT JOIN`、`INNER JOIN`）
- 嵌套子查询效率比较低，可以将其优化成连接查询（少用`IN`，多用`JOIN`）
- 频繁进行数据操作的表，不要建立太多的索引
- 连接表时，可以先用where条件对表进行过滤，然后做表连接（虽然MySQL会对连表语句做优化）
- 复合索引的建立需要进行仔细分析；尽量考虑用单字段索引代替
  - 正确选择复合索引中的主列字段，一般是选择性较好的字段
  - 如果复合索引中包含的字段经常单独出现在Where子句中，则分解为多个单字段索引
  - 如果复合索引所包含的字段超过3个，那么仔细考虑其必要性，考虑减少复合的字段
  - 果既有单字段索引，又有这几个字段上的复合索引，一般可以删除复合索引
  - 复合索引的几个字段是否经常同时以AND方式出现在Where子句中？单字段查询是否极少甚至没有？如果是，则可以建立复合索引；否则考虑单字段索引
- 学会分析sql执行计划，mysql会对sql进行优化，所以分析执行计划很重要
- 尽量使用覆盖索引，能增加查询效率
- **`删除数据，修改索引字段，新增操作都会对索引进行维护，维护开销大，同时需要更大的磁盘空间，需要综合平衡，取最优点`**

### 调优建议相关

- 对查询进行优化，应尽量避免全表扫描，首先应考虑在 where 及 order by 涉及的列上建立索引。
- 应尽量避免在 where 子句中对字段进行 null 值判断，否则将导致引擎放弃使用索引而进行全表扫描，

```
select id from t where num is null;
```

- 可以在 num 上设置默认值 0,确保表中 num 列没有 null 值，然后这样查询：

```sql
select id from t where num=0;
```

- 应尽量避免在 where 子句中使用!=或<>操作符，否则将引擎放弃使用索引而进行全表扫描。

- 应尽量避免在 where 子句中使用 or 来连接条件，否则将导致引擎放弃使用索引而进行全表扫描，

```sql
select id from t where num=10 or num=20;
```

​		可以这样查询：

```sql
select id from t where num=10 union all select id from t where num=20;
```

- in 和 not in 也要慎用，否则会导致全表扫描，如：

```sql
select id from t where num in(1,2,3);
```

- 对于连续的数值，能用 between 就不要用 in 了：

```sql
select id from t where num between 1 and 3;
```

- 下面的查询也将导致全表扫描：

```sql
select id from t where name like '%c%';
```

若要提高效率，可以考虑全文检索。

- 如果在 where 子句中使用参数，也会导致全表扫描。因为 SQL 只有在运行时才会解析局部变量，但优 化程序不能将访问计划的选择推迟到运行时;它必须在编译时进行选择。然 而，如果在编译时建立访问计 划，变量的值还是未知的，因而无法作为索引选择的输入项。如下面语句将进行全表扫描：

```sql
select id from t where num=@num ;
```

- 可以改为强制查询使用索引：

```sql
select id from t with(index(索引名)) where num=@num ;
```

- 应尽量避免在 where 子句中对字段进行表达式操作， 这将导致引擎放弃使用索引而进行全表扫描。

```sql
select id from t where num/2=100;
```

可以这样查询：

```sql
select id from t where num=100*2;
```

- 应尽量避免在 where 子句中对字段进行函数操作，这将导致引擎放弃使用索引而进行全表扫描。如：

```sql
select id from t where substring(name,1,3)='abc';#name 以 abc 开头的 id
```

​		应改为：

```sql
select id from t where name like 'abc%';
```

- 不要在 where 子句中的“=”左边进行函数、算术运算或其他表达式运算，否则系统将可能无法正确使用 索引。

- 在使用索引字段作为条件时，如果该索引是复合索引，那么必须使用到该索引中的第一个字段作为条件 时才能保证系统使用该索引， 否则该索引将不会 被使用， 并且应尽可能的让字段顺序与索引顺序相一致。

  不要写一些没有意义的查询，如需要生成一个空表结构：

```sql
select col1,col2 into #t from t where 1=0;
```

- 这类代码不会返回任何结果集，但是会消耗系统资源的，应改成这样：

```sql
create table #t(…);
```

- 很多时候用 exists 代替 in 是一个好的选择：

```sql
select num from a where num in(select num from b);
```

​			用下面的语句替换：

```sql
select num from a where exists(select 1 from b where num=a.num);
```

- 并不是所有索引对查询都有效，SQL 是根据表中数据来进行查询优化的，当索引列有大量数据重复时， SQL 查询可能不会去利用索引，如一表中有字段 ***,male、female 几乎各一半，那么即使在 *** 上建 了索引也对查询效率起不了作用。

- 索引并不是越多越好，索引固然可以提高相应的 select 的效率，但同时也降低了 insert 及 update 的效率，因为 insert 或 update 时有可能会重建索引，所以怎样建索引需要慎重考虑，视具体情况而定。一个表的索引数最好不要超过 6 个，若太多则应考虑一些不常使用到的列上建的索引是否有必要。

- 应尽可能的避免更新 clustered 索引数据列， 因为 clustered 索引数据列的顺序就是表记录的物理存储顺序，一旦该列值改变将导致整个表记录的顺序的调整，会耗费相当大的资源。若应用系统需要频繁更新 clustered 索引数据列，那么需要考虑是否应将该索引建为 clustered 索引。

- 尽量使用数字型字段，若只含数值信息的字段尽量不要设计为字符型，这会降低查询和连接的性能，并 会增加存储开销。这是因为引擎在处理查询和连接时会逐个比较字符串中每一个字符，而对于数字型而言 只需要比较一次就够了。

- 尽可能的使用 varchar/nvarchar 代替 char/nchar , 因为首先变长字段存储空间小， 可以节省存储空间， 其次对于查询来说，在一个相对较小的字段内搜索效率显然要高些。

- 任何地方都不要使用 select * from t ,用具体的字段列表代替“*”,不要返回用不到的任何字段。

- 尽量使用表变量来代替临时表。如果表变量包含大量数据，请注意索引非常有限(只有主键索引)。

- 避免频繁创建和删除临时表，以减少系统表资源的消耗。

- 临时表并不是不可使用，适当地使用它们可以使某些例程更有效，例如，当需要重复引用大型表或常用 表中的某个数据集时。但是，对于一次性事件， 最好使用导出表。

- 在新建临时表时，如果一次性插入数据量很大，那么可以使用 select into 代替 create table,避免造成大量 log ,以提高速度;如果数据量不大，为了缓和系统表的资源，应先 create table,然后 insert.

- 如果使用到了临时表， 在存储过程的最后务必将所有的临时表显式删除， 先 truncate table ,然后 drop table ,这样可以避免系统表的较长时间锁定。

- 尽量避免使用游标，因为游标的效率较差，如果游标操作的数据超过 1 万行，那么就应该考虑改写。

- 使用基于游标的方法或临时表方法之前，应先寻找基于集的解决方案来解决问题，基于集的方法通常更 有效。

- 与临时表一样，游标并不是不可使用。对小型数据集使用 FAST_FORWARD 游标通常要优于其他逐行处理方法，尤其是在必引用几个表才能获得所需的数据时。在结果集中包括“合计”的例程通常要比使用游标执行的速度快。如果开发时间允许，基于游标的方法和基于集的方法都可以尝试一下，看哪一种方法的效果更好。

- 在所有的存储过程和触发器的开始处设置 SET NOCOUNT ON ,在结束时设置 SET NOCOUNT OFF .无需在执行存储过程和触发器的每个语句后向客户端发送 DONE_IN_PROC 消息。

- 尽量避免大事务操作，提高系统并发能力。 sql 优化方法使用索引来更快地遍历表。 缺省情况下建立的索引是非群集索引，但有时它并不是最佳的。在非群集索引下，数据在物理上随机存放在数据页上。合理的索引设计要建立在对各种查询的分析和预测上。一般来说：

  a.有大量重复值、且经常有范围查询( > ,< ,> =,< =)和 order by、group by 发生的列，可考虑建立集群索引;

  b.经常同时存取多列，且每列都含有重复值可考虑建立组合索引;

  c.组合索引要尽量使关键查询形成索引覆盖，其前导列一定是使用最频繁的列。索引虽有助于提高性能但 不是索引越多越好，恰好相反过多的索引会导致系统低效。用户在表中每加进一个索引，维护索引集合就 要做相应的更新工作。

- 定期分析表和检查表。

```
分析表的语法：ANALYZE [LOCAL | NO_WRITE_TO_BINLOG] TABLE tb1_name[, tbl_name]...
```

以上语句用于分析和存储表的关键字分布，分析的结果将可以使得系统得到准确的统计信息，使得SQL能够生成正确的执行计划。如果用户感觉实际执行计划并不是预期的执行计划，执行一次分析表可能会解决问题。在分析期间，使用一个读取锁定对表进行锁定。这对于MyISAM，DBD和InnoDB表有作用。

```
例如分析一个数据表：analyze table table_name
检查表的语法：CHECK TABLE tb1_name[,tbl_name]...[option]...option = {QUICK | FAST | MEDIUM | EXTENDED | CHANGED}
```

检查表的作用是检查一个或多个表是否有错误，CHECK TABLE 对MyISAM 和 InnoDB表有作用，对于MyISAM表，关键字统计数据被更新

CHECK TABLE 也可以检查视图是否有错误，比如在视图定义中被引用的表不存在。

- 定期优化表。

```
优化表的语法：OPTIMIZE [LOCAL | NO_WRITE_TO_BINLOG] TABLE tb1_name [,tbl_name]...
```

如果删除了表的一大部分，或者如果已经对含有可变长度行的表(含有 VARCHAR、BLOB或TEXT列的表)进行更多更改，则应使用OPTIMIZE TABLE命令来进行表优化。这个命令可以将表中的空间碎片进行合并，并且可以消除由于删除或者更新造成的空间浪费，但OPTIMIZE TABLE 命令只对MyISAM、 BDB 和InnoDB表起作用。

```
例如： optimize table table_name
```

注意： analyze、check、optimize执行期间将对表进行锁定，因此一定注意要在MySQL数据库不繁忙的时候执行相关的操作。

## 参考文章

> [一次非常有意思的sql优化经历](https://www.cnblogs.com/tangyanbo/p/4462734.html)
> [MySQL 性能调优专题二（Explain执行计划使用详解）](https://juejin.im/post/5ce61072f265da1b5e72cb20)
> [sql查询调优之where条件排序字段以及limit使用索引的奥秘](https://www.cnblogs.com/tangyanbo/p/6378741.html)

