# 一、建表sql

```sql
DROP TABLE IF EXISTS actor;
CREATE TABLE actor (
  id int(11) NOT NULL,
  name varchar(45) DEFAULT NULL,
  update_time datetime DEFAULT NULL,
  PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

INSERT INTO actor VALUES ('1', '徐凤年', '2021-12-28 13:46:10');
INSERT INTO actor VALUES ('2', '李淳罡', '2021-12-28 13:46:10');
INSERT INTO actor VALUES ('3', '李当心', '2021-12-28 13:46:10');
INSERT INTO actor VALUES ('4', '拓跋菩萨', '2021-12-28 13:46:10');
INSERT INTO actor VALUES ('5', '段誉', '2021-12-28 13:49:59');



DROP TABLE IF EXISTS film;
CREATE TABLE film (
  id int(11) NOT NULL AUTO_INCREMENT,
  name varchar(10) DEFAULT NULL,
  PRIMARY KEY (id),
  KEY idx_name (name)
) ENGINE=InnoDB AUTO_INCREMENT=3 DEFAULT CHARSET=utf8;

INSERT INTO film VALUES ('2', '天龙八部');
INSERT INTO film VALUES ('1', '雪中悍刀行');



DROP TABLE IF EXISTS film_actor;
CREATE TABLE film_actor (
  id int(11) NOT NULL,
  film_id int(11) NOT NULL,
  actor_id int(11) NOT NULL,
  remark varchar(255) DEFAULT NULL,
  PRIMARY KEY (id),
  KEY idx_film_actor_id (film_id,actor_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

INSERT INTO film_actor VALUES ('1', '1', '1', 'hello');
INSERT INTO film_actor VALUES ('2', '1', '2', 'hello');
INSERT INTO film_actor VALUES ('3', '1', '3', 'hello');
INSERT INTO film_actor VALUES ('4', '1', '4', 'hello');
INSERT INTO film_actor VALUES ('5', '2', '5', 'hello');
```

# 二、explain执行计划

## （一）explain两个变种

###   1、**explain extended** 

explain 5.7以上版本有partitions和filtered字段，如果以下版本想显示这两个字段需要加上 extended 如下图

![1640920107768](../image/1640920107768.png)

filtered：就是当前的查询结果，占总查询记录的百分比

### 2、 **explain partitions** 

相比 explain 多了个 partitions 字段，如果查询是基于分区表的话，会显示查询将访问的分

区。

不过mysql5.7及以后 废弃了上述写法explain partitions 和explain extended 直接 explain就自带了上述的分区信息 和过滤信息



## （二） explain中的列 

```
set session optimizer_switch='derived_merge=off';       #关闭mysql5.7新特性对衍生表的合并优化
show warnings;                                          #查看mysql帮助优化的结果
```

### 1.id列

id列的编号是 select 的序列号，有几个 select 就有几个id，并且id的顺序是按 select 出现的顺序增长的。

id列越大执行优先级越高，id相同则从上往下执行，id为NULL最后执行。

### 2.select_type 

select_type 表示对应行是简单还是复杂的查询。

1）simple：简单查询。查询不包含子查询和union

2）primary：复杂查询中最外层的 select

3）subquery：包含在 select 中的子查询（不在 from 子句中）

4）derived：包含在 from 子句中的子查询。MySQL会将结果存放在一个临时表中，也称为派生表（derived的英文含义）

### 3.table列

这一列表示 explain 的一行正在访问哪个表。

当 from 子句中有子查询时，table列是  格式，表示当前查询依赖 id=N 的查询，于是先执行 id=N 的查

询。 如下图所示先执行的就是id为3的查询

![1640944183709](../image/1640944183709.png)

### 4.type列

这一列表示关联类型或访问类型，即MySQL决定如何查找表中的行，查找数据行记录的大概范围。

依次从最优到最差分别为：

```
system > const > eq_ref > ref > range > index > ALL
```

#### **NULL**：

mysql能够在优化阶段分解查询语句，在执行阶段用不着再访问表或索引。例如：在索引列中选取最小值，可以单独查找索引来完成，不需要在执行时访问表

因为 索引是排好序的 那么就直接取了

例如：id是主键索引，并且是排好序的，不用再去访问表或索引，比如取id的最小值这种。

```sql
 explain select min(id) from film; 
```

**const, system：**mysql能对查询的某部分进行优化并将其转化成一个常量（可以看show warnings 的结果）。用于**primary key** 或 **unique key** 的所有列与**常数**比较时，所以表最多有一个匹配行，读取1次，速度比较快。system是const的特例，表里**只**有**一条数据**匹配时为system 

例如： 

```sql
explain extended select * from (select * from film where id = 1) tmp;
show warnings;
```

![1640944801503](../image/1640944801503.png)

#### **eq_ref**：

**primary key** 或 **unique key** 索引的所有部分被连接使用 ，最多只会返回一条符合条件的记录。这可能是在const 之外最好的联接类型了，简单的 select 查询不会出现这种 type。主键或唯一键关联，eq_ref效率仅次于const。

```sql
explain select * from film_actor left join film on film_actor.film_id = film.id;
```

![1640945181008](../image/1640945181008.png)

#### **ref：**

相比 **eq_ref**，不使用唯一索引，而是使用**普通索引**或者**唯一性索引的部分前缀**，索引要和某个值相比较，可能会找到多个符合条件的行。

1.简单 select 查询，name是普通索引（非唯一索引） 

```sql
explain select * from film where name = 'film1';     
```

 ![img](../image/2021012810532335.png) 

2.关联表查询，idx_film_actor_id是film_id和actor_id的联合索引，这里使用到了film_actor的左边前缀film_id部分。

```sql
 explain select film_id from film left join film_actor on film.id = film_actor.film_id; 
```

   ![img](../image/20210128105414977.png)            



#### **range：**

范围扫描通常出现在 in(), between ,> ,<, >= 等操作中。使用一个索引来检索给定范围的行。

```sql
 explain select * from actor where id > 1; 
```

![img](../image/20210128105450780.png) 



#### **index：**

扫描全索引就能拿到结果，一般是扫描某个二级索引，这种扫描不会从索引树根节点开始快速查找，而是直接对二级索引的叶子节点遍历和扫描，速度还是比较慢的，这种查询一般为使用覆盖索引，二级索引一般比较小，所以这种通常比ALL快一些。要查询的字段在索引中就能查找出来，比如一张表里就只有name和id两个字段，同时做了name的索引，这时候通过二级索引就可以直接查出所要的结果。

```sql
explain select  * from  film;
```

![1641534379011](../image/1641534379011.png)

二级索引的叶子节点数据量只有主键索引的值 而主键索引叶子节点存放的是这一行所有的数据



#### **ALL：**

即全表扫描，扫描你的聚簇索引的所有叶子节点。通常情况下这需要增加索引来进行优化了。

```sql
 explain select * from actor; 
```

 ![img](../image/20210128105658479.png) 



### 5.possible_keys列

这一列显示查询可能使用哪些索引来查找。

explain 时可能出现 possible_keys 有列，而 key 显示 NULL 的情况，这种情况是因为表中数据不多，mysql认为索引

对此查询帮助不大，选择了全表查询。

如果该列是NULL，则没有相关的索引。在这种情况下，可以通过检查 where 子句看是否可以创造一个适当的索引来提

高查询性能，然后用 explain 查看效果。



### 6.key列

这一列显示mysql实际采用哪个索引来优化对该表的访问。

如果没有使用索引，则该列是 NULL。如果想强制mysql使用或忽视possible_keys列中的索引，在查询中使用 force

index、ignore index。