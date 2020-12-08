### mysql建立索引的方法论为：

*   多数查询经常使用的列；

*   很少进行修改操作的列；

*   索引需要建立在数据差异化大的列上

###for update的用法

~~~mysql
#会等待行锁释放之后，返回查询结果。
SELECT * FROM TABLE FOR UPDATE
#不等待行锁释放，提示锁冲突，不返回结果
SELECT * FROM TABLE FOR UPDATE NOWAIT 
#等待5秒，若行锁仍未释放，则提示锁冲突，不返回结果
SELECT * FROM TABLE FOR UPDATE WAIT 5
#查询返回查询结果，但忽略有行锁的记录
SELECT * FROM TABLE FOR UPDATE SKIP LOCKED
~~~

###优化及注意事项

#####1、应尽量避免在 where 子句中使用!=或<>操作符，否则将引擎放弃使用索引而进行全表扫描。

~~~mysql
SELECT * FROM TABLE WHERE id<>'3' FOR UPDATE;
~~~

但是有种情况还是会用到索引如下：

~~~mysql
SELECT * FROM TABLE WHERE id<>’3' AND name=‘name’;
~~~

id不会用到索引，如果name加了索引，name字段还是会用到索引。

#####2、应尽量避免在 where 子句中对字段进行 null 值判断，否则将导致引擎放弃使用索引而进行全表扫描。

~~~mysql
SELECT name FROM TABLE WHERE name IS NULL;
~~~

这个查询在高版本里面已经经过优化，对null进行判断，效率也是很高的，我的版本是5.7支持对null的判断。低版本的还是要注意。

#####3、尽量避免在where子句中使用or来连接条件，否则将导致引擎放弃使用索引而进行全表扫描

~~~mysql
SELECT * FROM TABLE WHERE name=‘name’ OR name=’name1’;
~~~

这种查询效率在数据量很大的时候查询效率很低，在MySQL高版本中两边都有索引也会用索引，如果有一个不用，MySQL就放弃索引。

我们经常用下面的方式代替：

~~~
SELECT * FROM TABLE WHERE name='name' UNION ALL SELECT * FROM TABLE WHERE name='name1';
~~~

这种方式就是分开查询，然后合并，效率非常高。

#####4、在模糊查询语句中百分号不能前置

~~~mysql
SELECT * FROM TABLE WHERE name like '%am%'; 
~~~

这样会放弃索引，全表扫描

#####5、in 和 not in 也要慎用

连续的数字使用

~~~mysql
SELECT * FROM TABLE WHERE num IN(1,2,3);
~~~

这样看似很好，但是如果连续数字很多，这样写就很麻烦，所以代替以上写法

~~~mysql
SELECT * FROM TABLE WHERE num BETWEEN 1 AND 3;
~~~

我们来比较一下 EXISTS 和 IN，什么时候用 IN 和 EXISTS

1.  子表数据量比外表数据量少，使用IN。
2.  子表数据量比外表数据量大，使用EXISTS。
3.  子表与外表数据量大小差不多，用 IN 与 EXISTS 的效率相差不大。

原因分析

1.  IN 语句：使用hash将外表与内表连接。

    ~~~mysql
    SELECT * FROM TABLE WHERE id IN (SELECT id FROM TABLE1)
    ~~~

    会用到TABLE表的id索引。以上查询使用了IN语句,IN只执行一次,它查出TABLE1表中的所有id字段并缓存起来。之后,检查TABLE表的id是否与TABLE1表中的id相等,如果相等则将TABLE表的记录加入结果集中,直到遍历完TABLE表的所有记录。如:TABLE表有10000条记录,TABLE1表有1000000条记录,那么最多有可能遍历100001000000次,效率很差。如果TABLE表有10000条记录,TABLE1表有100条记录,那么最多有可能遍历10000100次,遍历次数大大减少,效率大大提升.

2.  EXISTS 是对外表做loop循环，每次loop循环再对子表进行访问。

    ~~~mysql
    SELECT t.* FROM TABLE AS t WHERE EXISTS(SELECT 1 FROM TABLE1 as t1 WHERE t.id=t1.id)
    ~~~

    会用TABLE1表的id索引。以上查询使用了EXISTS 语句,EXISTS会执行TABLE.length次,它并不缓存exists结果集,因为 EXISTS 结果集的内容并不重要,重要的是结果集中是否有记录,如果有则返回true,没有则返回false。当TABLE1表比TABLE表数据大时适合使用exists,因为它没有那么遍历操作,只需要再执行一次查询就行.

    *   TABLE表有10000条记录,TABLE1表有1000000条记录,那么 EXISTS 会执行10000次去判断TABLE表中的id是否与TABLE1表中的id相等.
    *   TABLE表有10000条记录,TABLE1表有100000000条记录,那么 EXISTS 还是执行10000次,因为它只执行TABLE.length次,可见TABLE1表数据越多,越适合 EXISTS 发挥效果.
    *   TABLE表有10000条记录,TABLE1表有100条记录,那么 EXISTS 还是执行10000次,还不如使用 IN 遍历10000*100次,因为 IN 是在内存里遍历比较,而 EXISTS 需要查询数据库,我们都知道查询数据库所消耗的性能更高,而内存比较很快.

3.  not in 和 not exists

    *   使用 NOT IN 会内外表都全表扫描；使用 NOT EXISTS 还能用上内表（子表）的索引。所以，一定建议使用 NOT EXISTS。

    *   如果在 WHERE 子句中使用参数，也会导致全表扫描。因为SQL只有在运行时才会解析局部变量，但优化程序不能将访问计划的选择推迟到运行时；它必须在编译时进行选择。如果在编译时建立访问计划，变量的值还是未知的，因而无法作为索引选择的输入项。

        ~~~mysql
        SELECT * FROM TABLE WHERE name=@name;
        ~~~

    *   应尽量避免在 WHERE 子句中对字段进行表达式操作，这将导致引擎放弃使用索引而进行全表扫描。

        ~~~mysql
        SELECT * FROM TABLE WHERE id/2=4
        ~~~

        这样语句不会有人会写，很傻。

    *   应尽量避免在 WHERE 子句中对字段进行函数操作，这将导致引擎放弃使用索引而进行全表扫描。

        ~~~mysql
        SELECT * FROM TABLE WHERE substring(name,1,3)='hek';
        ~~~

    *   在使用索引字段作为条件时，如果该索引是复合索引，那么必须使用到该索引中的第一个字段作为条件时才能保证系统使用该索引，否则该索引将不会被使用，并且应尽可能的让字段顺序与索引顺序相一致。(如果name和password是复合索引)

        ~~~mysql
        SELECT * FROM TABLE WHERE name='name' AND password='123456';
        ~~~

        在低版本中所以顺序是不能变的，但是在高版本中，顺序可以是任何组合

        ~~~mysql
        SELECT * FROM TABLE WHERE password='123456' AND name='name';
        ~~~

        这条语句在低版本中是不用索引，但是在5.7版本中是用到索引。

        ~~~mysql
        SELECT * FROM TABLE WHERE name='name';
        ~~~

        这个毫无疑问用到索引

        ~~~msyql
        SELECT * FROM TABLE WHERE password='123456';
        ~~~

        这个是无法引用索引的

    *   并不是所有索引对查询都有效，SQL是根据表中数据来进行查询优化的，当索引列有大量数据重复时，SQL查询可能不会去利用索引，如一表中有字段 sex，male、female几乎各一半，那么即使在sex上建了索引也对查询效率起不了作用。(索引要建立在差异化大的字段上面)
    *   索引并不是越多越好，索引固然可以提高相应的 SELECT 的效率，但同时也降低了 INSERT 及 UPDATE 的效率，因为 INSERT 或 UPDATE 时有可能会重建索引，所以怎样建索引需要慎重考虑，视具体情况而定。一个表的索引数最好不要超过6个，如果太多就要考虑有些字段上面的索引是否必要。
    *   应尽可能的避免更新 clustered 索引数据列，因为 clustered 索引数据列的顺序就是表记录的物理存储顺序，一旦该列值改变将导致整个表记录的顺序的调整，会耗费相当大的资源。若应用系统需要频繁更新 clustered 索引数据列，那么需要考虑是否应将该索引建为 clustered 索引。
    *   尽量使用数字型字段，若只含数值信息的字段尽量不要设计为字符型，这会降低查询和连接的性能，并会增加存储开销。这是因为引擎在处理查询和连接时会 逐个比较字符串中每一个字符，而对于数字型而言只需要比较一次就够了。
    *   任何地方都不要使用 SELECT \* FROM TABLE ，用具体的字段列表代替“\*”，不要返回用不到的任何字段。
    *   如果使用到了临时表，在存储过程的最后务必将所有的临时表显式删除，先 truncate table ，然后 drop table ，这样可以避免系统表的较长时间锁定。

#####6、limit分页优化

当偏移量特别大时，limit效率会非常低

~~~mysql
SELECT id FROM A LIMIT 1000,10   很快
SELECT id FROM A LIMIT 90000,10  很慢
~~~

优化方法：

~~~mysql
SELECT id FROM A order by id LIMIT 90000,10;
~~~

很快(因为用到索引id)

#####7、尽量用 UNION ALL 替换 UNION

UNION 和 UNION ALL 的差异主要是前者需要将两个（或者多个）结果集合并后再进行唯一性过滤操作，这就会涉及到排序，增加大量的cpu运算，加大资源消耗及延迟。所以当我们可以确认不可能出现重复结果集或者不在乎重复结果集的时候，尽量使用 UNION ALL 而不是UNION。

#####8、INNER JOIN 和 LEFT JOIN、RIGHT JOIN、子查询

1.  INNER JOIN内连接也叫等值连接是，LEFT / RIGHT JOIN是外连接。

    ~~~mysql
    SELECT A.id,A.name,B.id,B.name FROM A LEFT JOIN B ON A.id =B.id;    左连接
    SELECT A.id,A.name,B.id,B.name FROM A RIGHT JOIN B ON B A.id= B.id; 右连接
    SELECT A.id,A.name,B.id,B.name FROM A INNER JOIN B ON A.id =B.id;   内连接
    ~~~

    INNER JOIN 效率高，因为 INNER JOIN 是等值连接，返回的行数比较少。

    ~~~mysql
    SELECT A.id,A.name,B.id,B.name FROM A,B WHERE A.id = B.id;
    ~~~

    推荐：能用 INNER JOIN 连接尽量使用 INNER JOIN 连接。

2.  子查询的性能又比外连接性能慢，尽量用外连接来替换子查询。

    ~~~mysql
    SELECT * FROM A WHERE EXIXTX (SELECT * FROM B WHERE id>=3000 AND A.id=B.id);
    ~~~

    一种简单的优化就是用 INNER JOIN 的方法来代替子查询，查询语句改为

    ~~~mysql
    SELECT * FROM A INNER JOIN B ON A.id=B.id where b.id>=3000;
    ~~~

3.  使用JOIN时候，应该用小的结果驱动大的结果（LEFT JOIN左边表结果尽量小，如果有条件应该放到左边先处理，right join同理反向），同时尽量把牵涉到多表联合的查询拆分多个query(多个表查询效率低，容易锁表和阻塞)。

    ~~~mysql
    SELECT * FROM A LEFT JOIN B ON A.id=B.id WHEREA.id>10;
    ~~~

    替换

    ~~~mysql
    SELECT * FROM (SELECT * FROM A WHERE id >10) T1 LEFT JOIN B on T1.id=B.id;
    ~~~

    LEFT / RIGHT JOIN 时系统做的逻辑运算量大于INNER JOIN，是因为INNER JOIN 只需选出能匹配的记录，LEFT JOIN 不仅需要选出能匹配的，而且还要返回左表不能匹配的，所以多出了这一部分逻辑运算。

#####9、InnoDB基于索引的行锁，什么时候行锁呢？

InnoDB行锁是通过索引上的索引项来实现的，这一点ＭySQL与Oracle不同，后者是通过在数据中对相应数据行加锁来实现的。InnoDB这种行锁实现特点意味者：只有通过索引条件检索数据，InnoDB才会使用行级锁，否则，InnoDB将使用表锁

在MySQL中，行级锁并不是直接锁记录，而是锁索引。索引分为主键索引和非主键索引两种，如果一条sql语句操作了主键索引，MySQL就会锁定这条主键索引；如果一条语句操作了非主键索引，MySQL会先锁定该非主键索引，再锁定相关的主键索引。 在UPDATE、DELETE操作时，MySQL不仅锁定WHERE条件扫描过的所有索引记录，而且会锁定相邻的键值，即所谓的next-key locking。

由于InnoDB预设是Row-Level Lock，所以只有明确的指定主键，MySQL才会执行Row lock (只锁住被选取的资料例) ，否则MySQL将会执行Table Lock (将整个资料表单给锁住)

(明确指定主键，并且有此数据，row lock)

~~~mysql
SELECT * FROM TABLE WHERE id='3';
~~~

(明确指定主键，若查无此数据，无lock)

~~~mysql
SELECT * FROM TABLE WHERE id='-1';
~~~

(无主键，table lock)

~~~msyql
SELECT * FROM TABLE WHERE name='name';
~~~

(主键不明确，table lock)

~~~mysql
SELECT * FROM TABLE WHERE id<>'3';
~~~

(主键不明确，table lock)

~~~mysql
SELECT * FROM TABLE WHERE id LIKE '3';
~~~

#####10、索引类型对锁类型的影响

主键：众所周知，自带最高效的索引属性

唯一索引：属性值重复率为0，可以作为业务主键

普通索引：属性值重复率大于0，不能作为唯一指定条件

注意：对于普通索引，当“重复率”低时，甚至接近主键或者唯一索引的效果时，依然是行锁；但是如果“重复率”高时，Mysql不会把这个普通索引当做索引，即会造成一个没有索引的SQL，从而形成表锁。

#####11、explain sql优化器

1.  表的读取顺序
2.  数据读取操作的操作类型
3.  哪些索引可以使用
4.  哪些索引被实际使用
5.  表之间的引用
6.  每张表有多少行被优化器查询

*   type 字段：它提供了判断查询是否高效的重要依据依据. 通过 type 字段, 我们判断此次查询是全表扫描还是索引扫描等.

*   const: 针对主键或唯一索引的等值查询扫描, 最多只返回一行数据. const 查询速度非常快, 因为它仅仅读取一次即可

*   eq_ref: 此类型通常出现在多表的 join 查询, 表示对于前表的每一个结果, 都只能匹配到后表的一行结果. 并且查询的比较操作通常是 =, 查询效率较高.

*   ref: 此类型通常出现在多表的 join 查询, 针对于非唯一或非主键索引, 或者是使用了最左前缀规则索引的查询.

*   range: 表示使用索引范围查询, 通过索引字段范围获取表中部分数据记录. 这个类型通常出现在 =, <>, >, >=, <, <=, IS NULL, <=>, BETWEEN, IN() 操作中. 当 type 是 range 时, 那么 EXPLAIN 输出的 ref 字段为 NULL, 并且 key_len 字段是此次查询中使用到的索引的最长的那个.

*   index: 表示全索引扫描(full index scan), 和 ALL 类型类似, 只不过 ALL 类型是全表扫描, 而 index 类型则仅仅扫描所有的索引, 而不扫描数据. index 类型通常出现在: 所要查询的数据直接在索引树中就可以获取到, 而不需要扫描数据. 当是这种情况时, Extra 字段会显示 Using index.

*   ALL: 表示全表扫描, 这个类型的查询是性能最差的查询之一. 通常来说, 我们的查询不应该出现 ALL 类型的查询, 因为这样的查询在数据量大的情况下, 对数据库的性能是巨大的灾难. 如一个查询是 ALL 类型查询, 那么一般来说可以对相应的字段添加索引来避免.

*   查询效率由低到高：ALL < index < range ~ index_merge < ref < eq_ref < const < system

*   rows 也是一个重要的字段. MySQL 查询优化器根据统计信息, 估算 SQL 要查找到结果集需要扫描读取的数据行数. 这个值非常直观显示 SQL 的效率好坏, 原则上 rows 越少越好.

*   Extra

    *   Using filesort

        当 Extra 中有 Using filesort 时, 表示 MySQL 需额外的排序操作, 不能通过索引顺序达到排序效果. 一般有 Using filesort, 都建议优化去掉, 因为这样的查询 CPU 资源消耗大.

    *   Using index

        “覆盖索引扫描”, 表示查询在索引树中就可查找所需数据, 不用扫描表数据文件, 往往说明性能不错

    *   Using temporary

        查询有使用临时表, 一般出现于排序, 分组和多表 join 的情况, 查询效率不高, 建议优化.

