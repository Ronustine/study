[TOC]

## 例子
```sql
CREATE TABLE `employees` ( 
    `id` int(11) NOT NULL AUTO_INCREMENT, 
    `name` varchar(24) NOT NULL DEFAULT '' COMMENT '姓名', 
    `age` int(11) NOT NULL DEFAULT '0' COMMENT '年龄', 
    `position` varchar(20) NOT NULL DEFAULT '' COMMENT '职位', 
    `hire_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '入职时间',
PRIMARY KEY (`id`), 
KEY `idx_name_age_position` (`name`,`age`,`position`) USING BTREE 
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8 COMMENT='员工记录表'; 

INSERT INTO employees(name,age,position,hire_time) VALUES('LiLei',22,'mana ger',NOW()); 
INSERT INTO employees(name,age,position,hire_time) VALUES('HanMeimei', 23,'dev',NOW()); 
INSERT INTO employees(name,age,position,hire_time) VALUES('Lucy',23,'dev',NOW());
```
## 优化法则
#### 全值匹配
`EXPLAIN SELECT * FROM employees WHERE name= 'LiLei';`
ref是const

#### 最左前缀法则
指的是查询从索引的最左前列开始并且不跳过索引中的列；

这个不会走索引
EXPLAIN SELECT * FROM employees WHERE age = 22 AND position ='manager';

#### <span id="jump1">不要在索引列上做任何操作</span>
计算、函数、自动or手动类型转换，MySQL在优化之前只要有这些，就不会使用索引，会导致索引失效，转向全表扫描；
```sql
-- (1)
EXPLAIN SELECT * FROM employees WHERE left(name,3) = 'LiLei';

-- (2) 转化为日期范围查询，会走索引：
EXPLAIN select * from employees where date(hire_time) ='2018-09-30';
-> 
EXPLAIN select * from employees where hire_time >='2018-09-30 00:00:00' and hire_time <='2018-09-30 23:59:59';
```

#### 存储引擎不能使用索引中条件范围条件右边的列
EXPLAIN SELECT * FROM employees WHERE name= 'LiLei' AND age > 22 AND position ='manager';
这个只走到name、age

#### 尽量使用覆盖索引（只访问索引的查询）
即索引包括查询列，减少select* 语句
`EXPLAIN SELECT name,age FROM employees WHERE name= 'LiLei' AND age = 23 AND position ='manager';`
name,age都在索引内，可直接返回了

#### MySQL不要使用不等于（!= 、<、>）
无法使用索引，导致全表扫描
`EXPLAIN SELECT * FROM employees WHERE name != 'LiLei';`
可以改成index，把*改成在索引内的值

#### is null，is not null 也无法使用索引
null很特殊，没法和索引比对。尽量把字段设计成NOT NULL

#### like以通配符开头（'%Lei'）索引会失效变成全表
如果要使用索引
- 使用覆盖索引；
- 要么用专门的搜索引擎；

#### 字符串不加单引号索引失效；
MySQL会自动转换，转换就会加操作，参考[不要在索引列上做任何操作](#jump1)

#### 少用or/in，不一定会使用索引
mysql内部优化器会根据检索比例、表大小等多个因素评估是否使用索引；

#### 范围查询优化
给年龄添加单值索引`ALTER TABLE `employees` ADD INDEX `idx_age` (`age`) USING BTREE ;`
`explain select * from employees where age >=1 and age <=2000;`
MySQL内部优化器会根据检索比例，表大小等多个因素整体评估是否使用索引。
优化方法：可以将大的范围拆分成多个小范围

## MySQL如何选择合适的索引
上面法则介绍的会走，但走还是不走索引，使用Trace分析工具，注意开启这个会影响MySQL性能，只能临时分析sql使用：
```sql
set session optimizer_trace="enabled=on",end_markers_in_json=on; ‐‐开启trace

select * from employees where name > 'a' order by position;
SELECT * FROM information_schema.OPTIMIZER_TRACE;
```
看rows_estimation -> table_scan（全表扫描成本）、potential_range_indexs（可能使用的索引）-> analyzing_range_alternatives（各个索引使用成本）

注意：不同sql之间的cost值没有可比较的意义
`set session optimizer_trace="enabled=off"; ‐‐关闭trace`

## order by与group by
`EXPLAIN select * from employees where name='LiLei' and position='dev' order by age`
利用最左前缀法则：中间字段不能断，因此查询用到了name索引，从key_len=74也能看出，age索引列用在排序过程中，因为Extra字段里没有using filesort

`EXPLAIN select * from employees where name='LiLei' order by position`
从explain的执行结果来看：key_len=74，查询使用了name索引，由于用了position进行排序，跳过了 age，出现了Using filesort。

**结论**
1. 两种排序是filesort和index，Using index是MySQL扫描索引本身完成排序。index效率高，filesort低；
2. order by满足两种情况会使用Using index：order by语句使用索引最左前列；使用where子句和order by子句条件条件列符合最左前缀法则；
3. 尽量在索引列上完成排序，遵循索引建立的最左前缀法则；
4. order by的条件不在索引列上，就会产生；
5. 能用覆盖索引尽量用；
6. group by和order by类似，先排序后分组。遵照索引创建顺序的最左前缀法则。如果group by优化不要排序那么加上order by null禁止排序。where高于having，尽量写在where中；

## filesort文件排序原理
- 单路排序：一次性去除满足条件行的所有字段，然后在sort buffer中进行排序，trace工具可以看到sort_mode显示sort_key, additional_fields或者sort_key,packed_additional_fields；
- 双路排序（回表排序模式）：根据相应条件取出相应的排序字段，可以直接定位行数据的行ID，然后在sort buffer中进行排序，排序后再次取回其它需要的字段，tance工具可以看到sort_mode信息里显示sort_key, rowid；

MySQL通过比较系统变量 max_length_for_sort_data(默认1024字节) 的大小和需要查询的字段总大小来 判断使用哪种排序模式。 
- 如果 max_length_for_sort_data 比查询字段的总长度大，那么使用 单路排序模式； 
- 如果 max_length_for_sort_data 比查询字段的总长度小，那么使用 双路排序模式；

tace中注意这个："number_of_tmp_files": 3, ‐‐使用临时文件的个数，这个值如果为0代表全部使用的sort_buffer内存排序，否则使用的是磁盘排序。

## 分页查询的优化

#### 根据自增且连续的主键排序分页（这种情况很少）
常见：`select * from employees limit 90000, 5`
上面的是全表扫描读出90000条数据后再取其的后5条，数据量越多查询越慢

这分页没有另外的order by，可以通过主键索引。而且主键是自增且连续，所以可以这样
优化：`select * from employees where id > 90000 limit 5`

但实际上表中的数据会经常删除，主键有空缺，导致结果不一致，第一条的语法规则是：`SELECT * FROM table  LIMIT [offset,] rows | rows OFFSET offset`，90000是位移量。

应用范围很小：主键自增且连续、结果是按照主键排序的

#### 根据非主键字段排序的分页查询
常见：`select * from employees ORDER BY name limit 90000,5;`
这里有可能会用name的索引，也可能不会用，这看MySQL的最终优化结果。

关键点是让排序返回的字段尽可能少，可以在排序和分页操作先查出主键，再用主键查询
优化：`select * from employees e inner join (select id from employees order by name limit 90000,5) ed on e.id = ed.id;`

## Join关联查询优化
少用，在报表上可以用。关联常见两个算法：Nested-Loop Join(NLJ 嵌套循环连接)、Block Nested-Loop Join(BNL 基于块的嵌套循环)
关键点是大小表的判断，用小表驱动大表.
🌰
```sql
CREATE TABLE `t1` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `a` int(11) DEFAULT NULL,
    `b` int(11) DEFAULT NULL,
    PRIMARY KEY (`id`),
    KEY `idx_a` (`a`)
    ) ENGINE=InnoDB AUTO_INCREMENT=10001 DEFAULT CHARSET=utf8;

create table t2 like t1;
-- 往t1表插入1万行记录，往t2表插入100行记录
```
#### Nested-Loop Join
> 一次一行循环地从第一张表（称为驱动表）中读取行，在这行数据中取到关联字段，根据关联字段在另一张表（被驱动表）里取出满足条件的行，然后取出两张表的结果合集。

`EXPLAIN select * from t1 inner join t2 on t1.a= t2.a;`
| id | select_type | table | type | possible_keys | key | key_len | ref | rows | filtered | Extra |
| - | - | - | - | - | - | - | - | - | - | - |
| 1 | SIMPLE | t2 | ALL | idx_a |  |  |  | 100 | 100 | Using where |
| 1 | SIMPLE | t1 | ref | idx_a | idx_a | 5 | test.t2.a | 1 | 100 | |
从上面可以得出：
- 驱动表是t2，被驱动表是t1。先执行的是驱动表（执行计划结果的id如果一样则按从上到下顺序执行sql）；优化器一般会优先选择小表做驱动表。注意：inner join中，排在前面的表并不一定是驱动表
- 使用了NLJ算法。一般join语句中，如果执行计划Extra中未出现Using join buffer则表示使用的join算法是NLJ

sql执行的大致流程为：
1. 从t2读一行数据；
2. 从第一步的数据取出关联字段a，到t1中查找；
3. 取出表t1中满足条件的行，跟t2中获取到的结果合并，作为返回结果给客户端；
4. 重复上面3步；

整个过程会读取t2表的所有数据（100行），然后遍历这每行数据中字段a，根据t2表中a的值索引扫描t1表中的对应行（扫描100次t1的索引，1次扫描可认为最终只扫描t1表一行完整数据，也就是总共t1表扫描了100行）。整个过程扫描了200行。

如果驱动表的关联字段没索引，使用NLJ算法性能会比较低，mysql会选择BNL。

#### Block Nested-Loop Join
> 把驱动表的数据读入join_buffer，然后扫描被驱动表，把被驱动表每一行取出来与join_buffer中的数据做对比.

`EXPLAIN select * from t1 inner join t2 on t1.b= t2.b;`
| id | select_type | table | type | possible_keys | key | key_len | ref | rows | filtered | Extra |
| - | - | - | - | - | - | - | - | - | - | - |
| 1 | SIMPLE | t2 | ALL | | | | | 100 | 100 | |
| 1 | SIMPLE | t1 | ALL | | | | | 10337 | 10 | Using where; Using join buffer(Block Nested Loop)  |
Extra中的Using join buffer(Block Nested Loop)说明该关联查询使用的是BNL算法

sql执行的大致流程为：
1. 把t2的所有数据放入到join_buffer中；
2. 把t1中每一行取出来，跟join_buffer中的数据做对比；
3. 返回满足join条件的数据；

整个过程对t1、t2都做了一次全表扫描，扫描总行数为10000+100（t1 + t2的数据总量）=10100。并且join_buffer里的数据是无序的，因此对t1每一行都要做100次判断，所以内存中的判断次数是100 * 10000=100万次

说明：
为什么没有索引要选择BNL算法而不是NLJ？
如果第二条采用NLJ，那么扫描行数是100*10000=100万次，这是磁盘扫描。显然，BNL磁盘扫描次数少很多，相比于磁盘扫描，BNL的内存会计算会快很多。

**结论**
MySQL对于被驱动表的关联字段没索引的关联查询，一般会使用BNL算法。如果有索引一般选择NLJ算法，有索引的情况下NLJ算法比BNL算法性能更高。
优化建议：
- 关联字段加索引，尽量让MySQL选择NLJ；
- 小表驱动大表，多表连接时如果明确知道小表是哪张，可以用straight_join写法固定连接驱动方式，省去MySQL优化器自己判断的时间；
- 要是不确定，那就优化器判断，一般都是靠谱的，慎重使用straight_join；

straight_join：可以让左边表驱动右边的，能改变优化器对于联表查询的执行顺序。不过只是inner join，不能当作left、right join

## IN、EXSISTS优化
与join一样，小表驱动大表
#### IN
当B表小于A表，优先用IN
```sql
select * from A where id in(select id from B)

-- 等价于
for(select id from B) {
    select * from A where A.id = B.id
}
```

#### EXISTS
当A表的数据集小于B表数据集，优先用EXISTS。
语义是将A的数据放到子查询B中做条件验证，根据结果（true false）来决定主查询的数据是否保留
```sql
select * from A where exists(select 1 from B where B.id = A.id)
-- 等价于
for(select * from A) {
    select * from B where B.id = A.id
}
```

- EXISTS (subquery)只返回TRUE或FALSE,因此子查询中的SELECT * 也可以用SELECT替换,官方说法是实际执行时会 忽略SELECT清单,因此没有区别；
- EXISTS子查询的实际执行过程可能经过了优化而不是我们理解上的逐条对比；
- EXISTS子查询往往也可以用JOIN来代替，何种最优需要具体问题具体分析；

## count(*)查询优化
测试前关闭MySQL查询缓存：
`set global query_cache_size=0;`
`set global query_cache_type=0;`

这几个做比较
```sql
EXPLAIN select count(1) from employees;
EXPLAIN select count(id) from employees;
EXPLAIN select count(name) from employees;
EXPLAIN select count(*) from employees;
```
| id | select_type | table | type | possible_keys | key | key_len | ref | rows | filtered | Extra |
| - | - | - | - | - | - | - | - | - | - | - |
| 1 | SIMPLE | employees | index | | idx_name_age_position | 140 | | 100140 | 100 | Using index |
执行计划一样，说明四个sql执行效率应该差不多，区别在于某个字段count不会统计字段为null值的数据行。MySQL为什么选择辅助索引不是主键索引，二级索引相对主键索引存储数据更少，检索性能更高。

优化：
- Myisam存储引擎的表做不带where条件的count查询性能高，因为总行数会被MySQL存储在磁盘上，不需要计算；
- 如果只需要估计值，则用`show table status like 'employees' `；
- 总行数维护到Redis;
- 增加计数表，插入或删除表数据的时候同时维护技术表，让他们在同一个事务操作；