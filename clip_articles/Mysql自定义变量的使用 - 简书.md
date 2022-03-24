# Mysql自定义变量的使用 - 简书
[![](https://cdn2.jianshu.io/assets/default_avatar/9-cceda3cf5072bcdd77e8ca4f21c40998.jpg)
](https://www.jianshu.com/u/717bc9ae3e2b)

0.2722016.12.13 00:11:51 字数 1,891 阅读 23,481

> 用户自定义变量是一个容易被遗忘的 MySQL 特性，但是如果能用的好，发挥其潜力，在某些场景可以写出非常高效的查询语句。在查询中混合使用过程化和关系化逻辑的时候，自定义变量可能会非常有用。单纯的关系查询将所有的东西都当成无序的数据集合，并且一次性操作它们。MySQL 则采用了更加程序化的处理方式。MySQL 的这种方式有它的弱点，但如果能够熟练地掌握，则会发现其强大之处，而用户自定义变量也可以给这种方式带来很大的帮助

用户自定义变量是一个用来存储内容的临时容器，在连接 MySQL 的整个过程中都存在，可以使用下面的 SET 和 SELECT 语句来定义它们：

    mysql> SET @one := 1;
    mysql> SET @min_actor := (SELECT MIN(actor_id) FROM sakila.actor);
    mysql> SET @last_week := CURRENT_DATE - INTERVAL 1 WEEK; 

然后可以在任何可以使用表达式的地方使用这些自定义变量：

`SELECT ... WHERE col <= @last_week;`

在了解自定义变量的强大之前，我们先来看看它自身的一些属性和限制，看看在哪些场景下我们不能使用用户自定义变量：

-   使用自定义变量的查询，无法使用查询缓存
-   不能再使用常量或者标识符的地方使用自定义变量，例如表名、列名和 LIMIT 子句中。
-   用户自定义变量的生命周期是在**一个连接**中有效，所以不能用它们来做连接间的通信。
-   如果使用连接池或者持久化连接，自定义变量可能让看起来毫无关系的代码发生交互。
-   自定义变量的类型是一个动态类型。
-   MySQL 优化器在某些场景下可能会将这些变量优化掉，这可能导致代码不按预想的方式运行。
-   赋值的顺序和赋值的时间点并不总是固定的，这依赖于优化器的决定。
-   赋值符号 := 的优先级非常低，所以需要注意，赋值表达式应该使用明确的括号。
-   使用未定义变量不会产生任何语法错误，如果没有意识到这一点，非常容易犯错。

### 优化排名语句

使用自定义变量的一个特性是你可以在给一个变量赋值的同时使用这个变量，即 “左值” 特性。例如：

    mysql> SET @rownum := 0;
    mysql> SELECT actor_id, @rownum := @rownum + 1 AS rownum
    FROM actor order by actor_id LIMIT 3;
        +----------+--------+
        | actor_id | rownum |
        +----------+--------+
        |        1 |      1 |
        |        2 |      2 |
        |        3 |      3 |
        +----------+--------+ 

这个例子的实际意义并不大，它只是实现了一个和该表主键一样的列。不过，我们可以把这当作一个排名。现在我们来看一个更复杂的用法。我们先编写一个查询获取演过最多电影的前 10 位演员，然后根据他们的出演电影次数做一个排名，如果出演的电影数量一样，则排名相同。我们先编写一个查询，返回每个演员参演电影的数量。

     mysql> SET @curr_cnt := 0, @prev_cnt := 0, @rank := 0;
    mysql> SELECT actor_id, COUNT(*) as cnt
        -> FROM film_actor
        -> GROUP BY actor_id
        -> ORDER BY cnt DESC
        -> LIMIT 10;
        +----------+-----+
        | actor_id | cnt |
        +----------+-----+
        |      107 |  42 |
        |      102 |  41 |
        |      198 |  40 |
        |      181 |  39 |
        |       23 |  37 |
        |       81 |  36 |
        |       37 |  35 |
        |      106 |  35 |
        |       60 |  35 |
        |       13 |  35 |
        +----------+-----+ 

现在我们再把排名加上去，这里看到有四个演员都参演了 35 部电影，所以他们的排名应该是相同的。我们使用三个变量来实现：一个用来记录当前的排名，一个用来记录前一个演员的排名，还有一个用来记录当前演员参演的电影数量。只有当前演员参演的电影的数量和前一个演员不同时，排名才变化。我们试试下面的写法：

     mysql> SELECT actor_id,
        -> @curr_cnt := COUNT(*) AS cnt,
        -> @rank     := IF(@prev_cnt <> @curr_cnt, @rank + 1, @rank) AS rank,
        -> @prev_cnt := @curr_cnt AS dummy
        -> FROM film_actor
        -> GROUP BY actor_id
        -> ORDER BY cnt DESC
        -> LIMIT 10;
        +----------+-----+------+-------+
        | actor_id | cnt | rank | dummy |
        +----------+-----+------+-------+
        |      107 |  42 |    0 |     0 |
        |      102 |  41 |    0 |     0 |
        |      198 |  40 |    0 |     0 |
        |      181 |  39 |    0 |     0 |
        |       23 |  37 |    0 |     0 |
        |       81 |  36 |    0 |     0 |
        |      106 |  35 |    0 |     0 |
        |       60 |  35 |    0 |     0 |
        |       13 |  35 |    0 |     0 |
        |       37 |  35 |    0 |     0 |
        +----------+-----+------+-------+ 

我们发现跟我们设想的不太一样。这里，通过 EXPLAIN 我们看到将会使用临时表和文件排序，所以可能是由于变量赋值的时间和我们预料的不同。  
使用 SQL 语句生成排名值通常需要做两次计算，例如，需要额外计算一次出演过相同数量电影的演员有哪些。使用变量则可一次完成 --- 这对性能是一个很大的提升。  
针对这个案例，另一个简单的方案是在 FROM 子句中使用子查询生成的一个中间的临时表：

     mysql> SELECT actor_id,
        -> @curr_cnt := cnt AS cnt,
        -> @rank     := IF(@prev_cnt <> @curr_cnt, @rank + 1, @rank) AS rank,
        -> @prev_cnt := @curr_cnt AS dummy
        -> FROM (
        -> SELECT actor_id, COUNT(*) AS cnt
        -> FROM film_actor
        -> GROUP BY actor_id
        -> ORDER BY cnt DESC
        -> LIMIT 10
        -> ) as der;
    +----------+-----+------+-------+
    | actor_id | cnt | rank | dummy |
    +----------+-----+------+-------+
    |      107 |  42 |    1 |    42 |
    |      102 |  41 |    2 |    41 |
    |      198 |  40 |    3 |    40 |
    |      181 |  39 |    4 |    39 |
    |       23 |  37 |    5 |    37 |
    |       81 |  36 |    6 |    36 |
    |       37 |  35 |    7 |    35 |
    |      106 |  35 |    7 |    35 |
    |       60 |  35 |    7 |    35 |
    |       13 |  35 |    7 |    35 |
    +----------+-----+------+-------+ 

### 避免重复查询刚刚更新的数据

如果在更新行的同学又希望获得该行的信息，避免重复查询，可以用变量巧妙的实现。例如，我们的一个客户希望能够更高效地更新一条记录的时间戳，同时希望查询当前记录中存放的时间戳是什么。简单地，可以用下面的代码来实现：

     UPDATE t1 SET lastUpdated = NOW() WHERE id = 1;
    SELECT lastUpdated FROM t1 WHERE id = 1; 

使用变量，我们可以按如下方式重写查询:

     UPDATE t1 SET lastUpdated = NOW() WHERE id = 1 AND @now := NOW();
    SELECT @now; 

上面看起来仍然需要两个查询，需要两次网络来回，但是这里第二个查询无需访问数据表，所以会快很多。

### 统计更新和插入的数量

     INSERT INTO t1(c1, c2) VALUES(4, 4), (2, 1), (3, 1)
    ON DUPLICATE KEY UPDATE
        c1 = VALUES(c1) + (0 * (@x := @x + 1)); 

当每次由于冲突导致更新时对变量 @x 自增一次，然后表达式乘以 0 让其不影响更新的内容，另外，MySQL 的协议会返回被更改的总行数，所以不需要单独统计。

### 确定取值的顺序

使用用户自定义变量的一个最常见的问题就是没有注意到在赋值和读取变量的时候可能是在查询的不同阶段。例如，在 SELECT 子句中进行赋值然后再 WHERE 子句中读取变量，则可能变量取值并不如你所想：

     mysql> SET @rownum := 0;
    mysql> SELECT actor_id, @rownum := @rownum + 1 AS cnt
        -> FROM actor
        -> WHERE @rownum <= 1;
    +----------+------+
    | actor_id | cnt  |
    +----------+------+
    |       58 |    1 |
    |       92 |    2 |
    +----------+------+ 

因为 WHERE 和 SELECT 是在查询执行的不同阶段被执行的。如果在查询中再加入 ORDER BY 的话，结果可能会更不同；

     mysql> SET @rownum := 0;
    mysql> SELECT actor_id, @rownum := @rownum + 1 AS cnt
        -> FROM actor
        -> WHERE @rownum <= 1
        -> ORDER BY first_name; 

这是因为 ORDER BY 引入了文件排序，而 WHERE 条件是在文件排序操作之前取值的，所以这条查询会返回表中的全部记录。解决这个问题的办法是让变量的赋值和取值发生在执行查询的同一阶段：

     mysql> SET @rownum := 0;
    mysql> SELECT actor_id, @rownum AS rownum
        -> FROM actor
        -> WHERE (@rownum := @rownum + 1) <= 1;
    +----------+--------+
    | actor_id | rownum |
    +----------+--------+
    |       58 |      1 |
    +----------+--------+ 

### 编写偷懒的 UNION

假设需要编写一个 UNION 查询，其第一个子查询作为分支条件先执行，如果找到了匹配的行，则跳过第二个分支。例如先在一个频繁访问的表查找热数据，找不到再去另外一个较少访问的表查找冷数据。

     SELECT id FROM users WHERE id = 123;
    UNION ALL
    SELECT id FROM users_archived WHERE id = 123; 

上面的查询可以工作，但是无论第一个表找没找到，都会在第二个表再找一次，如果使用变量的话可以很好地规避这个问题。

     SELECT GREATEST(@found := -1, id) AS id, 'users' AS which_tbl
    FROM users WHERE id = 1
    UNION ALL
        SELECT id, 'users_archived'
        FROM users_archived WHERE id = 1 AND @found IS NULL
    UNION ALL   
        SELECT 1, 'reset' FROM DUAL WHERE (@found := NULL) IS NOT NULL; 

### 用户自定义变量的其他用处

通过一些实践，可以了解所有用户自定义变量能够做的有趣的事情，例如下面这些用法：

-   查询运行时计算总数和平均值
-   模拟 GROUP 语句中的函数 FIRST() 和 LAST()
-   S 对大量数据做一些数据计算
-   计算一个大表的 MD5 散列值
-   编写一个样本处理函数
-   模拟读 / 写游标
-   在 SHOW 语句的 WHERE 子句中加入变量值

更多精彩内容，就在简书 APP

"小礼物走一走，来简书关注我"

还没有人赞赏，支持一下

[![](https://cdn2.jianshu.io/assets/default_avatar/9-cceda3cf5072bcdd77e8ca4f21c40998.jpg)
](https://www.jianshu.com/u/717bc9ae3e2b)

### 被以下专题收入，发现更多相似内容

### 推荐阅读[更多精彩内容](https://www.jianshu.com/)

-   1. SQL 简介 SQL 的目标 理想情况下，数据库语言应允许用户： 建立数据库和关系结构 完成基本数据管理任务...
-   Android 自定义 View 的各种姿势 1 Activity 的显示之 ViewRootImpl 详解 Activity...


-   “看来今年的桔子要欠收了!” 这是我看到桔园时跟君说的第一句话。 那个时候，我们正沿着乡间水泥路慢慢的往前走。往年这...

    [![](https://upload-images.jianshu.io/upload_images/3902913-ccaa17f4b11a6aec.jpg?imageMogr2/auto-orient/strip|imageView2/1/w/300/h/240/format/webp)
    ](https://www.jianshu.com/p/eec31153fc43)
-   鸟鸣: 为春天守一份爱 □山田耕夫 有一种朦胧的爱 鸟...
-   1\.\[导入第三方包]\[1] 将第三方包添加到 lib 目录下面 -> 右键找到 BuildPath->Add to Buil... 
    [https://www.jianshu.com/p/357a02fb2d64](https://www.jianshu.com/p/357a02fb2d64)
