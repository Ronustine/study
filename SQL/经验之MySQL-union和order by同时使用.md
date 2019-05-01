##### 问题： 如果直接用如下sql语句会报错：Incorrect usage of UNION and ORDER BY
```
SELECT * FROM t1 WHERE username LIKE 'l%' ORDER BY score ASC
UNION
SELECT * FROM t1 WHERE username LIKE '%m%' ORDER BY score ASC
```

因为union在没有括号的情况下只能使用一个order by，所以报错，这个语句有2种修改方法。如下：
- 可以将前面一个order by去掉，改成如下：
```
SELECT * FROM t1 WHERE username LIKE 'l%'
UNION
SELECT * FROM t1 WHERE username LIKE '%m%' ORDER BY score ASC
```
该sql的意思就是先union，然后对整个结果集进行order by。
- 可以通过两个查询分别加括号的方式，改成如下：
```
(SELECT * FROM t1 WHERE username LIKE 'l%' ORDER BY sroce ASC)
UNION
(SELECT * FROM t1 WHERE username LIKE '%m%' ORDER BY score ASC)
```
这种方式的目的是为了让两个结果集先分别order by，然后再对两个结果集进行union。
**但是，** 会发现这种方式虽然不报错了，但是两个order by并没有效果，所以应该改成如下：
```
SELECT * FROM
    (SELECT * FROM t1 WHERE username LIKE 'l%' ORDER BY score ASC) t3
UNION
SELECT * FROM
    (SELECT * FROM t1 WHERE username LIKE '%m%' ORDER BY score ASC) t4
```
也就是说，order by不能直接出现在union的子句中，但是可以出现在子句的子句中。

- 顺便提一句，union和union all 的区别。union会过滤掉两个结果集中重复的行，而union all不会过滤掉重复行。