## 大数据量时 limit 的优化技巧

### limit 查询原理
我们都知道 MySQL 的分页是通过 limit 来实现的。

而 MySQL `limit[offset][rows]` 的工作原理是先扫描前 `offset + rows` 条数据，并返回最后 `rows` 条数据，其余数据则全部丢弃。

假设我们有一张用户表：
```mysql
CREATE TABLE `t_user` (
  `id` int NOT NULL auto_increment,
  `name` varchar(20) NOT NULL,
  `phone` text,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```
这张用户表里面有几个亿的数据量。

当我们使用 `select * from t_user limit m,n` 这样的方式去做分页查询时，就会导致随着偏移量 `m` 的增加，SQL 的查询时间将会越来越慢。

通过 `explain` 函数分析该查询的结果为：
```text
mysql> explain select * from t_user limit 1000000,10;
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------+
| id | select_type | table  | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------+
|  1 | SIMPLE      | t_user | NULL       | ALL  | NULL          | NULL | NULL    | NULL |    1 |   100.00 | NULL  |
+----+-------------+--------+------------+------+---------------+------+---------+------+------+----------+-------+
```
### 优化方案

通过对上面的 SQL 语句进行分析后，我们发现该查询并未使用到索引。用户表中，`id` 字段为唯一的主键索引，所以我们可以通过 `id` 作为查询条件来进行优化：

```mysql
select * from t_user t1 
join (select id from t_user limit 1000000,10) t2
on t1.id = t2.id
```
通过 `explain` 函数分析：
```text
mysql> explain select * from t_user t1
    -> join (select id from t_user limit 1000000,10) t2
    -> on t1.id = t2.id;
+----+-------------+------------+------------+-------+---------------+-------------+---------+------------+------+----------+-------------+
| id | select_type | table      | partitions | type  | possible_keys | key         | key_len | ref        | rows | filtered | Extra       |
+----+-------------+------------+------------+-------+---------------+-------------+---------+------------+------+----------+-------------+
|  1 | PRIMARY     | t1         | NULL       | ALL   | PRIMARY       | NULL        | NULL    | NULL       |    1 |   100.00 | NULL        |
|  1 | PRIMARY     | <derived2> | NULL       | ref   | <auto_key0>   | <auto_key0> | 4       | test.t1.id |    2 |   100.00 | Using index |
|  2 | DERIVED     | t_user     | NULL       | index | NULL          | PRIMARY     | 4       | NULL       |    1 |   100.00 | Using index |
+----+-------------+------------+------------+-------+---------------+-------------+---------+------------+------+----------+-------------+
```
因为使用到了主键索引，所以该查询的速度大幅得到了提升。

使用普通索引字段做子查询也是可以的，虽然普通索引会导致回表，性能不如主键索引，但是也要比不使用索引的速度快很多。前提是，普通索引的字段必须具备唯一性，如果字段会有重复，则不可以使用该方法来进行优化。