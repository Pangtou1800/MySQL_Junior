# MySQL_Junior - MySQL高级

## 内容简介

    1.MySQL架构介绍

        ·MySQL简介
        ·MySQL Linux版安装
        ·MySQL配置文件
        ·MySQL逻辑架构介绍
        ·MySQL存储引擎

    2.索引优化分析

        ·性能下降、执行时间长、等待时间长
        ·常见通用的Join查询
        ·索引简介
        ·性能分析
        ·索引优化

    3.查询截取分析

        ·查询优化
        ·慢查询日志
        ·批量数据脚本
        ·Show Profile
        ·全局查询日志

    4.MySQL锁机制

        ·行锁
        ·表锁
        ·页锁

    5.主从复制

        ·基本原则
        ·常见问题
        ·一主一从常见配置

## 第一节 MySQL简介

### 1.1 概述

### 1.2 高级MySQL

    完整的MySQL优化需要很深的功底，大公司都设有专门的DBA

    1.MySQL内核

    2.SQL优化工程师

    3.MySQL服务器的优化

    4.各种参数常量设定

    5.查询语句优化

    6.主从复制

    7.软硬件升级

    8.容灾设备

    9.SQL编程

### 1.3 MySQL Linux版的安装配置

    ·生产环境使用GA - General Available 稳定发布版本
    ·检查当前系统是否安装了MySQL
        rpm -qa | grep -i mysql
    ·rpm安装mysql-server
        rpm -ivh mysql-server-...rpm
            install verbose hash(进度条)
    ·安装mysql-client
    ·查看进程
        ps -ef | grep mysql
    ·查看安装MySQL时创建的用户和用户组
        cat /etc/passed | grep mysql
        cat /etc/group | grep mysql
        也可以查看版本
        mysqladmin --version 查看版本
    ·启动MySQL服务
        service mysql start
        -> systemctl start mysqld
    ·关闭MySQL服务
        service mysql stop
        -> systemctl stop mysqld
    ·为root用户设置密码
        初次：/usr/bin/mysqladmin -u root password 123456
        修改密码： /usr/bin/mysqladmin -u root -p password 123456

    ·配置系统启动
        chkconfig mysqld on
        -> systemctl enable mysqld.service
        或者用ntsysv用界面设置

    ·查看datadir的位置
        进入mysql后，show variable like '%data%';

    ·查看各种mysql相关目录
        whereis mysql -> 暂时看起来没啥用

    ·查看mysql配置文件的位置
        whereis my.cnf

        -> /etc/my.cnf
        -> 与此同时还有一个目录叫my.cnf.d，里面有msql-server.cnf，里面有点东西

        [mysqld]
        datadir=/var/lib/mysql
        socket=/var/lib/mysql/mysql.sock
        log-error=/var/log/mysql/mysqld.log
        pid-file=/run/mysqld/mysqld.pid

    ·查看启动时加载文件顺序等
        which mysql - 显示启动脚本
        /usr/bin/mysql --verbose --help | grep -A 1 'Default options' - 显示脚本启动时的配置

    ·查看、修改字符集
        show variables like '%character%';

### 1.4 MySQL主要配置文件

    1.二进制日志 log-bin -> 主从复制

    2.错误日志 log-error - 默认关闭

    3.查询日志 log

    4.数据文件

    5.配置文件

### 1.5 MySQL逻辑架构简介

    Connectors
        ||
    MySQL Server
        -Management Services & Utilities
        1.连接层
            -Connection Pool
        2.服务层
            -SQL Interface
            -Parser
            -Optimizer
            -Caches & Buffers
        3.引擎层
            -Pluggable Storage Engines
                -MyISAM
                -InnoDB
                -NDB
                -Archive
                -Federated
                -Memory
                -Merge
                -Partner
                -Community
                -Custom
                ...
        4.存储层
            -FileSystem
            -Files & Logs

    和其他数据库产品相比，MySQL有点与众不同。
    它插件式的存储引擎架构将查询处理和其他的系统任务数据的存储提取相分离。
    这种架构可以使我们根据业务需求选择合适的引擎。

#### 1.5.1 存储引擎

    ·查看存储引擎
        show engines; - 查看当前产品对各个引擎的支持情况。

    ·查看默认引擎
        show variables like '%engine%'; 

||MyISAM|InnoDB|
|-|-|-|
|主外鍵|不支持|支持|
|事務|不支持|支持|
|鎖|表鎖|行鎖|
|緩存|只缓存索引，不缓存真实数据|缓存索引和真是数据，对内存要求高|
|表空间|小|大|
|关注点|性能|事务|
|默认安装|Y|Y|

#### 1.5.2 阿里巴巴实际在使用的数据库产品

    1.Percona公司为MySQL数据库服务器进行了改进，提供了XtraDB可以替代InnoDB
    2.阿里巴巴在Percona的原型上加以修改，提供了AliSQL+AliRedis

## 第二节 MySQL索引优化分析

### 2.1 性能下降：SQL慢

    ·查询语句写的烂
    ·索引失效
        -单值
        -符合
    ·关联查询太多join
    ·服务器调优及各个参数设置

        show index from user;
        show keys from user;

        单值索引 - 以某一个字段为关键字建索引

            create index idx_user_name on user(name);

        复合索引 - 以多个字段为关键字建索引

            create index idx_user_nameEmail on user(name, email);

### 2.2 关于Join语句

    1.SQL执行顺序
        -手写
        -机读
            FROM - ON - JOIN - WHERE - GROUP BY - HAVING - 
                SELECT - DISTINCT - ORDER BY - LIMIT
        -总结
            SQL解析
                -FROM 笛卡尔积
                -ON 主表保留
                -JOIN 不符合ON也添加
                -WHERE 非聚合，非select列名
                -GROUP BY 改变对表引用
                -HAVING 只作用于分组后
                -SELECT DISTINCT
                -ORDER BY 只可用SELECT别名
                -LIMIT rows offset

### 2.3 常用Join查询

    |A |+|B | - FROM A INNER JOIN B ON A.K=B.K
    |A+|+|B | - FROM A LEFT JOIN B ON A.K=B.K
    |A |+|B+| - FROM A RIGHT JOIN B ON A.K=B.K
    |A+|+|B+| - FROM A OUTER JOIN B ON A.K=B.K
    |A+| |B | - FROM A LEFT JOIN B ON A.K=B.K WHERE B.K IS NULL
    |A | |B+| - FROM A RIGHT JOIN B ON A.K=B.K WHERE A.K IS NULL
    |A+| |B+| - FROM A OUTER JOIN B ON A.K=B.K WHERE A.K IS NULL OR B.K IS NULL

    然而MySQL不支持FULL OUTER JOIN，可以用UNION实现。
    UNION默认去重，UNION ALL不去重。

## 第三节 索引

### 3.1 什么是索引

    MySQL官方：
        索引（Index）是帮助MySQL高效获取数据的数据结构。

        简单理解为排好序的快速查找数据结构。

    在数据之外，数据库系统还维护着满足特定查找算法的数据结构，
    这些数据结构以某种方式引用（指向）数据结构，
    这样就可以在这些数据结构的基础上实现高级的查找算法。
    这种数据结构就是索引。

    一般来说索引本身也很大，不可能全部存储在内存中，因此索引往往以索引文件的形式存储在磁盘上。

    我们平常说的索引一般都是B树。
    聚集索引、次要索引、复合索引、前缀索引、唯一索引默认都是使用B+树。
    此外，还有哈希索引。

### 3.2 索引优势、劣势

    优势：
        ·提高数据检索效率，降低数据库IO成本
        ·索引如果是有序的，那么也可以降低排序的成本，降低CPU消耗

    劣势：
        ·索引本身要占用空间
        ·降低数据修改效率，因为更新数据时还要同时更新索引
        ·索引本身也需要良好的设计和维护

### 3.3 索引分类

    1.单值索引
        一个索引只包含单个列，一个表可以有多个单值索引

    2.唯一索引
        索引列的值必须唯一 ※不要求非空

    3.复合索引
        一个索引包含多个列

    4.基本语法

### 3.4 基本语法

    1.创建

        create [unique] index indexName on tableName(col1(length),...);

    2.删除

        drop index [indexName] on tableName;

    3.查看

        show index from tableName;

    4.使用ALTER的语法

        alter table tbl1 add primary key (col1,...);
        alter table tbl1 add unique idx1 (col1,...);
        alter table tbl1 add index idx1 (col1,...);
        alter table tbl1 add fulltext idx1 (col1,...); //全文检索

### 3.5 MySQL中的索引结构

    1.BTree索引
    2.Hash索引
    3.Full-text索引
    4.R-Tree索引

### 3.6 索引创建场景

    1.主键自动建立唯一索引
    2.频繁作为查询条件的字段应该创建索引
    3.外键关系自动建立索引
    4.频繁更新的字段不宜作为索引
    5.不作为查询条件的字段不创建索引
    6.高并发下倾向创建组合索引
    7.查询中排序的字段可以适用索引
    8.查询中统计或者分组的字段可以适用索引

    不推荐索引：

        1.表记录少
        2.更新频度高
        3.数据重复且平均分布

        ※索引选择性：不同值/总记录数 -> 越接近1创建索引效果越好

## 第四节 性能优化

### 4.1 性能分析前提知识

    1.MySQL Query Optimizer 查询优化器

        MySQL有专门负责优化SELECT语句的优化器模块，主要功能：
        通过计算收集到的统计信息，为客户端请求的Query提供他认为的执行计划。

        客户端向MySQL发出一条Query，命令解析器区别出是SELECT语句之后，转发给Optimizer。
        Optimizer会：
            ·进行优化，计算出常量表达式
            ·进行简化和转换，去掉无用条件
            ·分析Hint信息，以及确定Hint是否可以完全确定执行计划
            ·没有Hint或足够的Hint信息时，根据Query得出相应的执行计划

    2.MySQL常见瓶颈

        CPU：CPU在饱和的时候一般发生在数据装入内存或从磁盘上读取数据的时候
        IO：磁盘I/O瓶颈发生在装入数据远大于内存容量的时候
        服务器硬件：top，free，iostat和vmstat来查看系统的性能状态

    3.Explain

        ·是什么（查看执行计划）

            使用EXPLAIN关键字可以模拟优化器执行SQL查询语句，
            从而知道MySQL是如何执行查询语句的。

        ·能干嘛

            -表的读取顺序
            -数据读取操作的操作类型
            -哪些索引可以使用
            -哪些索引被实际使用
            -表之间的引用
            -每张表有多少行被优化器查询

        ·怎么用

            -expalin + sql语句
            -分析执行计划包含的信息
                ·id
                ·select_type
                ·table
                ·possible_keys
                ·key
                ·key_len
                ·ref
                ·rows
                ·filtered
                ·Extra

        ·各字段解释

        ·热身Case

    4.Explain各字段解释

        ·id - select查询的序列号，包含一组数字，表示查询中执行select子句或操作表的顺序

            三种情况：
                -id相同时，执行顺序由上到下

                    explain select t2.*
                    from t1, t2, t3
                    where t1.id=t2.id and t1.id=t3.id
                    and t1.col='';

| id  | select_type | table... |
|-----|-------------|----------|
| 1   | SIMPLE      | t1...    |
| 1   | SIMPLE      | t3...    |
| 1   | SIMPLE      | t2...    |

                -id不同，如果是子查询，id的序号会递增，id值越大优先级越高

                    explain select t2.*
                    from t2
                    where id=(
                        select id
                        from t1
                        where id=(
                            select t3.id
                            from t3
                            where t3.col='')
                        )
                    );

|id|select_type|table|type|possible_keys|key|key_len|ref|rows|Extra|
|-|-|-|-|-|-|-|-|-|-|
|1|PRIMARY|t2|const|PRIMARY|PRIMARY|4|const|1||
|2|SUBQUERY|t1|const|PRIMARY|PRIMARY|4||1|Using index|
|3|SUBQUERY|t3|ALL|NULL|NULL|NULL|NULL|1|Using where|

                -id相同不同，同时存在；同级从上向下，不同级由高到低

                    explain select t2.*
                    from (
                        select t3.id
                        from t3
                        where t3.col=''
                    ) s1, t2
                    where s1.id = t2.id;

|id|select_type|table...
|-|-|-|
|1|PRIMARY|\<derived2\>...
|1|PRIMARY|t2...
|2|DERIVED|t3...

        DERIVED:衍生

        ·select_type - 数据读取操作的操作类型
            -SIMPLE
                简单查询（不包含子查询或者UNION操作）
            -PRIMARY
                若查询中包含任何复杂的子部分，则最外层被标记为PRIMARY
            -SUBQUERY
                在SELECT或WHERE列表中包含的子查询
            -DERIVED
                在FROM中包含的子查询被标记为DERIVED
                MySQL会递归执行这些子查询，把结果放在临时表里（<derived + [id]>）
            -UNION
                第二个SELECT出现在UNION之后，则被标记为UNION。
                若UNION包含在FROM子句的子查询中，外层SELECT将被标记为DERIVED
            -UNION RESULT
                从UNION表获取结果的

mysql> explain select * from tbl_emp a left join tbl_dept b on a.deptId=b.id union select * from tbl_emp a right join tbl_dept b on a.deptId=b.id; 

| id | select_type  | table      | partitions | type   | possible_keys | key        | key_len | ref           | rows | filtered | Extra           |
|-|-|-|-|-|-|-|-|-|-|-|-|
|  1 | PRIMARY      | a          | NULL       | ALL    | NULL          | NULL       | NULL    | NULL          |    8 |   100.00 | NULL            |
|  1 | PRIMARY      | b          | NULL       | eq_ref | PRIMARY       | PRIMARY    | 4       | test.a.deptId |    1 |   100.00 | NULL            |
|  2 | UNION        | b          | NULL       | ALL    | NULL          | NULL       | NULL    | NULL          |    5 |   100.00 | NULL            |
|  2 | UNION        | a          | NULL       | ref    | fk_dept_id    | fk_dept_id | 5       | test.b.id     |    2 |   100.00 | NULL            |
| NULL | UNION RESULT | <union1, 2> | NULL       | ALL    | NULL          | NULL       | NULL    | NULL          | NULL |     NULL | Using temporary |

        ·table - 数据来源表
        ·type - 访问类型
            -ALL
            -index
            -range
            -ref
            -eq_ref
            -const,system
            -NULL
            ...

            常见type效率排列：
                system > const > eq_ref > ref > range > index > ALL

            ※百万数据量 + ALL 直接考虑优化
            一般来说保证到达range或者ref级别

        

            -system
                表只有一行记录（等于系统表），是const类型的特例，平时不会出现
            -const
                通过索引一次就找到了，比如等值比较PK或者Unique索引。
                将主键置于where列表中，MySQL就将该查询转换为const级别
            -eq_ref
                唯一性索引扫描。对于每个索引键，表中只有一条记录与之匹配。
                常见于主键或者唯一索引扫描

 explain select * from tbl_emp, tbl_dept where tbl_emp.deptId=tbl_dept.id; 

| id  | table    | type   |
|-----|----------|--------|
| 1   | tbl_emp  | ALL    |
| 1   | tbl_dept | eq_ref |

            -ref
                非唯一性索引扫描。对于每个索引键，表中有不止一条记录与之匹配。
            -range
                在索引上检索给定范围。因为有开始和结束点，所以还是优于全表扫描的。
            -index
                FULL INDEX SCAN，遍历索引树。
                因为只遍历索引树即可，所以优于全表读。
            -ALL
                主键索引全表扫描。

        ·possible_keys - 可能应用在查询中的索引，一个或者多个。
            ※但不一定被实际使用

        ·key - 实际被应用的索引，一个。如果为NULL，则没有使用索引。
            若查询中使用了覆盖索引，则该索引仅出现在key中。

            explain select col2,col3 from t1;

        

        ·key_len - 索引中使用的字节数，显示为字段的定义长度，而非实际的数据长度
            ※联合索引可以只使用一部分

        ·ref - 显示哪一列被用于和这张表的索引列相比较了，可能是一个常数
            多个索引字段被使用的话会显示多个值
            where t1.id=t2.id and t1.id=t3.id and t1.col1=""
            id - 1 - 1 - 1
            ref - const - test.t1.id - test.t1.id
            ： t1.col1和常数相比较，t2.id和t1.id相比较，t3.id和t1.id相比较

        

        ·rows - 根据表统计信息和索引选用情况，大致统计出需要查询多少行数据

        ·Extra - 额外信息
            -using filesort
                MySQL会使用一个外部的索引排序（无法利用索引完成的排序称为filesort）
            -using temporary
                使用临时表保存中间结果。常见于order by和分组查询group by
            -using index
                查询操作使用了覆盖索引，如果同时出现了using where，表名索引用来查找
            -using where
                索引用来查找
            -using join buffer
                使用了join缓存
            -impossible where
                where值总为false
            -select tables optimized away
                在没有group by的情况下，基于索引优化MIN、MAX操作
                或者MyISAM引擎优化COUNT(*)操作，也就是在执行计划阶段就完成了查询
            -distinct
                优化distinct操作，找到第一个匹配后就停止

### 4.2 索引建立和优化

    1.单表

        create table if not exists article (
            id int primary key auto_increment,
            author_id int not null,
            category_id int not null,
            views int not null,
            comments int not null,
            title varbinary(255) not null,
            content text not null
        );

        insert into article values(1,1,1,1,1,'1','1'),(2,2,2,2,2,'2','2'),
        (3,3,3,3,3,'3','3');

    

        案例1：查询category_id=1且comments>1的情况下，views最多的article_id

        select id,author_id from article
        where category_id =1 and comments>1
        order by views desc limit 1;

        explain之后，type是All，Extra中有filesort。

        尝试优化1：建立索引

        create index idx_article_ccv on article(category_id,comments,views);

        使用了索引。
        type变为range，Extra变为where，但是依然有filesort。 => 因为范围导致后续索引失效了

        drop index idx_article_ccv on article;

        尝试优化2：建立索引

        create index idx_article_cvc on article(category_id,views,comments);

        使用了索引，Extra中filesort消失，出现了Backward index scan。

        drop index idx_article_ccv on article;

        尝试优化3：

        

        create index idx_article_cv on article(category_id,views);

        和2效果相同

        尝试优化4：

        create index idx_article_cvca on article(category_id,views,comments,author_id);

        Extra中出现了Using index，起飞

    

    2.双表

        create table class (
            id int primary key auto_increment,
            card int
        );

        create table book (
            bookId int primary key auto_increment,
            card int
        );

        insert into class(card) values(floor(1 + (rand()*20)));
        insert into book(card) values(floor(1 + (rand()*20)));

        案例：

            select * from class left join book on class.card=book.card;

        没有索引：

            id - 1 - 1
            table - class - book
            type - all - all
            extra - null - using where; using join buffer

        

        尝试优化1：

            左连接加在右表上
            create index idx_book_card on book(card);

            id - 1 - 1
            table - class - book
            type - all - ref
            extra - null - using index

            drop index idx_book_card on book;

        

        尝试优化2：

            左连接加在左表上
            create index idx_class_card on class(card);

            id - 1 - 1
            table - class - book
            type - index - all
            extra - using index - using where; using join buffer

            drop index idx_class_card on book;

        结论：join的话被驱动表要建索引。

    

    3.三表

        create table if not exists phone (
                phoneid int primary key auto_increment,
                card int
        );

        insert into phone(card) values(floor(1 + rand()*20));

        案例：

            select * from class 
            left join book on class.card=book.card
            left join phone on book.card=phone.card;

        

        未优化：

            id - 1 - 1 - 1
            table - class - book - phone
            type - All - All - All
            Extra - NULL - Using where; Using join buffer - ...

        

        尝试优化1：

            create index idx_book_card on book(card);
            create index idx_phone_card on phone(card);

            id - 1 - 1 - 1
            table - class - book - phone
            type - All - ref - ref
            Extra - NULL - Using index - Using index

    4.Join语句的优化
        ·小表驱动大表，因为驱动表是全表扫描
        ·优先优化内层循环
        ·被驱动表上Join字段建立索引
        ·无法保证被驱动表的索引时，调节JoinBuffer

### 4.3 索引失效

    create table staffs (
        id int primary key auto_increment,
        name varchar(24) not null default '' comment '姓名',
        age int not null default 0 comment '年龄',
        pos varchar(20) not null default '' comment '职位',
        add_time timestamp not null default current_timestamp comment '入职时间'
    ) comment '员工记录表';

    insert into staffs(name, age, pos, add_time)
    values('z3',22,'manager',now()),
    ('July',23,'dev',now()),('2000',23,'dev',now());

    create index idx_staffs_nameAgePos on staffs(name,age,pos);

    避免索引失效：
        ·使用全值匹配 - 索引中的每个列都在查询条件中
        ·遵循最佳左前缀法则 - 索引中的列从左到右开始适用
        ·不在索引列上做任何操作（计算、函数、自动·手动类型转换）
        ·范围条件列的右边的索引列无法使用（不等号之后的条件无法使用索引） - 范围之后全失效
        ·尽量使用覆盖引擎
        ·不等于条件也很难使用索引 （8.1测试结果type-range）
        ·IS NULL， IS NOT NULL也无法使用索引
        ·like 以%开头无法使用索引
            -如何解决两头%索引失效的问题？
            -推荐使用覆盖索引来解决
             -type - index; Extra - Using where; Using index
        ·字符串不加单引号索引失效
        ·用OR连接会索引失效

### 4.4 题目分析

    create table test03(
        id int primary key auto_increment,
        c1 char(10),
        c2 char(10),
        c3 char(10),
        c4 char(10),
        c5 char(10)
    );

    insert into test03(c1,c2,c3,c4,c5)
    values('a1','a2','a3','a4','a5'),('b1','b2','b3','b4','b5'),
    ('c1','c2','c3','c4','c5'),('d1','d2','d3','d4','d5'),
    ('e1','e2','e3','e4','e5');

    create index idx_test03_c1234 on test03(c1,c2,c3,c4);

    0.explain select * from test03 where c1='a1';
      explain select * from test03 where c1='a1' and c2='a2';

        ->普通的使用索引

    1.explain select * from test03 where c1='a1' and c2='a2' and c4='a4' and c3='a3';

        ·可以使用索引？ 可以
        ·用了几列？ 4
            -> 等值条件顺序无关
        ·type ref
        ·Extra Using index condition

    2.explain select * from test03 where c1='a1' and c2='a2' and c3>'a3' and c4='a4'; 

        ·可以使用吗？ 可以
        ·用了几列？ 3
        ·type range
        ·Extra Using index condition

    3.explain select * from test03 where c1='a1' and c2='a2' and c4>'a4' and c3='a3'; 

        ·可以使用吗？ 可以
        ·用了几列？ 4
        ·type range
        ·Extra Using index condition

    4.explain select * from test03 where c1='a1' and c2='a2' and c4='a4' order by c3;

        ·可以使用吗？ 可以
        ·用了几列？ 2
        ·type ref
        ·Extra Using index condition -> 如果是非索引后续列的话会出现Using filesort

    5.explain select * from test03 where c1='a1' and c2='a2' order by c3;

        ·同上

    6.explain select * from test03 where c1='a1' and c2='a2' order by c4; 

        ·可以使用吗？ 可以
        ·用了几列？ 2
        ·type ref
        ·Extra Using index condition; Using filesort

    7.explain select * from test03 where c1='a1' and c5='a5' order by c2,c3;

        ·可以使用吗？ 可以
        ·用了几列？ 1
        ·type ref
        ·Extra Using index condition; Using where

    8.explain select * from test03 where c1='a1' and c5='a5' order by c3, c2; 

        ·可以使用吗？ 可以
        ·用了几列？ 1
        ·type ref
        ·Extra Using index condition; Using where; Using filesort

    9.explain select * from test03 where c1='a1' and c2='a2' order by c2,c3;

        ·可以使用吗？ 可以
        ·用了几列？ 2
        ·type ref
        ·Extra Using index condition

    10.explain select * from test03 where c1='a1' and c2='a2' and c5='a5' order by c2,c3;

        ·可以使用吗？ 可以
        ·用了几列？ 2
        ·type ref
        ·Extra Using index condition; Using where

    11.explain select * from test03 where c1='a1' and c2='a2' and c5='a5' order by c3, c2; 

        ·可以使用吗？ 可以
        ·用了几列？ 2
        ·type ref
        ·Extra Using index condition; Using where -> 不会出现filesort，c2的排序条件被优化了

    12.explain select c1, c2, c3, c4 from test03 where c1='a1' and c4='a4' group by c2, c3; 

        ·可以使用吗？ 可以
        ·用了几列？ 1
        ·type ref
        ·Extra Using where; Using index

    13.explain select c1, c2, c3, c4 from test03 where c1='a1' and c4='a4' group by c3, c2; 

        ·可以使用吗？ 可以
        ·用了几列？ 1
        ·type ref
        ·Extra Using where; Using index; Using temporary

        ※分组之前必排序，所以会产生临时表

### 4.5 一般性建议

    ·对于单键索引，尽量选择针对当前Query过滤性更好的索引
    ·选择组合索引时，过滤性好的字段应当尽量靠左
    ·选择组合索引时，尽量包含更多where条件中的字段
    ·多分析统计信息，调整Query写法来选择合适索引

## 第五节 查询截取分析

    MySQL性能分析流程：

        1. 开启慢查询日志，设置阈值，比如超过5秒就是慢SQL，将其抓取
        2. 观察，取一个期间的慢SQL执行情况

        3.explain + 慢SQL分析
        4.show profile - 更加细致的SQL执行情况分析

        5. 运维经理或DBA，进行SQL服务器的参数调优

### 5.1 查询优化

    1.永远小表驱动大表

        select * from A where id int (select id from B);

            for (id : B.ids) {
                select * from A where A.id = B.id;
            }

        B表的数据集小于A表的数据集时，用in优于exists。

        select * from A where exists (select 1 from B where B.id=A.id);

            for (* : A.*) {
                select * from B where B.id = A.id;
            }

        A表的数据集小于B表的数据集时，用exists优于in。

        ※当然，A与B表的id字段都应该建立索引

        ·select * from tbl where exists(subquery)
            -exists(subquery)返回true或false，查询内容会被无视

    2.order by关键字优化

        ·order by子句，尽量使用index方式来排序，避免使用filesort。
        ·两种情况可以使用index方式：
            -order by使用索引最左前列
            -使用where子句与order by子句条件组合满足索引最左前列
              ※全部降序也可以用索引 - Backward index scan
        ·如果没有使用index，MySQL有两种排序方式：
            -单路排序
                全部读入后在buffer中排序；
                只读一次，但是需要的空间更多
            -双路排序
                MySQL 4.1之前使用双路排序，扫两次磁盘；
                先读order by列和指针，排好序后再出数据

            单路排序的问题：
                如果内存放不下，就要依赖外部存储来排序，IO次数可能会多于双路。

            优化策略：
                ·增大sort_buffer_size
                ·增大max_length_for_sort_data
                    -Query字段长度大于这个参数且排序字段不包括TEXT，BLOB时就会回到多路排序

    3.group by关键字优化

        ·group by之前必须排序，优化方式基本与order by相同
        ·where高于having，所以条件优先写在where里

### 5.2 慢查询日志

    默认阈值10秒。
    自动记录查询超过阈值的SQL。

    默认不开启慢查询日志，需要手动开启。

        ·show variables like 'slow%'; 
        ·set global slow_query_log=1; 
        ·show variables like 'long_query_time'; 
        ·set global long_query_time=3; 

    执行超过long_query_time的查询语句，就会被记录在slow_query_log_file中。

    可以使用select sleep(1)来睡眠，不过如果在表上执行的话，
    MySQL会对满足条件的行每行sleep指定的时间。

    在生产环境中，如果手工分析日志，查找、分析SQL，显然比较费力，
    MySQL提供了日志分析工具mysqldumpslow。

        mysqldumpslow --help

            s - 排序方式
            c - 访问次数
            l - 锁定时间
            r - 返回记录
            t - 查询时间
            al - 平均锁定时间
            ar - 平均返回记录数
            at - 平均查询时间
            t - 返回前面多少条数据
            g - 正则表达式，大小写不敏感

        ·得到返回记录集最多的10条SQL
            mysqldumpslow -s r -t 10 /var/lib/mysql/localhost-slow.log
        ·得到访问次数最多的10条SQL
            mysqldumpslow -s c -t 10 /var/lib/mysql/localhost-slow.log
        ·得到访问时间最长的的10条有左连接的SQL
            mysqldumpslow -s t -t 10 -g 'left join' /var/lib/mysql/localhost-slow.log

            ※建议配合more使用，否则容易爆屏

### 5.3 批量数据脚本

    目标：插入1000W数据

    1.建表

        create table dept(
            id int primary key auto_increment,
            deptNo int not null default 0,
            dName varchar(20) not null default "",
            loc varchar(13) not null default ""
        );

        create table emp(
            id int primary key auto_increment,
            empNo int not null default 0,
            eName varchar(20) not null default "",
            job varchar(9) not null default "",
            mgr int not null default 0,
            hiredate date not null,
            sal decimal(7,2) not null,
            comm decimal(7,2) not null,
            deptNo int not null default 0
        );

    2.设置参数log_bin_trust_function_creators

        show variables like 'log_bin_trust_function_creators'; 
        set global log_bin_trust_function_creators=1; 

    3. 编写函数插入不同的数据

        ·随机字符串     
            delimiter ; ; 
            create function rand_string(n int) returns varchar(255)
            begin
                declare chars_str varchar(100) default 'abcdefghijklmnopqrstuvwxyz'; 
                declare return_str varchar(255) default ''; 
                declare i int default 0; 
                
                while i < n do
                    set return_str=concat(return_str, substring(chars_str, floor(1+rand()*26), 1)); 
                    set i = i + 1; 
                end while; 

                return return_str; 
            end; ; 

        ·随机编号
            delimiter ; ; 
            create function rand_num() returns int(5)
            begin
                declare i int default 0; 
                set i = floor(100+rand()*10); 
                return i; 
            end; ; 

    4. 创建存储过程

        ·往emp表中插入数据的存储过程
            delimiter ; ; 
            create procedure insert_emp(in start int, in max_num int)
            begin
                declare i int default 0; 
                set autocommit = 0; 
                repeat
                    set i = i + 1; 
                    insert into emp(empNo, eName, job, mgr, hiredate, sal, comm, deptNo)
                    values((start+i), rand_string(6), 'SALEMAN', 0001, curdate(), 2000, 400, rand_num()); 
                until i = max_num
                end repeat; 
                commit; 
            end; ; 
        ·往dept表中插入数据的存储过程
            delimiter ; ; 
            create procedure insert_dept(in start int, in max_num int)
            begin
            declare i int default 0; 
                set autocommit = 0; 
                repeat
                    set i = i + 1; 
                    insert into dept(deptNo, dName, loc)
                    values((start+i), rand_string(10), rand_string(8)); 
                until i = max_num
                end repeat; 
                commit; 
            end; ; 

    5. 调用存储过程

        call insert_emp(1,500000);
        call insert_dept(1,500000);

### 5.4 show profile

    测试环境中重现故障

    1.收到报警
    2.开启慢查询，抓取慢SQL
    3.explain观察
    4.show profile

    ·是什么？
        MySQL提供的可以用来分析当前会话中语句执行的资源消耗情况
        可以拥有SQL调优的测量
        默认情况下，参数处于关闭状态
        默认保存最近15次的运行结果

    ·分析步骤

        1. 查看当前版本是否支持

            show vairables like '%profiling%'; 

        2. 开启

            set global profiling=1; 

        3. 运行SQL

            select * from tbl_emp; 

        4. 查看结果

            show profiles; 

        5. 诊断SQL结果

            show profile cpu, block io for query (Query_ID); 
            ※可以看到一条SQL的完整生命周期

                starting
                check permissions
                Opening tables
                init
                System lock
                optimizing
                statistics
                preparing
                executing
                end
                query end
                waiting for handler commit
                closing tables
                freeing items
                cleaning up
            
            ※参数可选

                ALL,BLOCK IO,CONTEXT SWITCHES,CPU,IPC,MEMORY,PAGE FAULTS,
                SOURCE,SWAPS

            主要关注四个生命周期阶段：
                ·converting heap to MyISAM - 查询结果从内存移动到磁盘
                ·Creating tmp table - 创建临时表
                ·Copying to tmp table on disk - 内存临时表复制到磁盘
                ·locked - 锁等待
            
            ※这些步骤出现时往往就代表着效率低下

### 5.5 全局查询日志

    1.配置启用

        set global general_log=1;

    2.编码启用

        set global log_output='TABLE';

        此后，SQL语句将会记录到MySQL库里的general_log表，可以用下面的命令查看

        select * from mysql.general_log;

    3.永远不要在生产环境开启这个功能

## 第六节 MySQL锁机制

### 6.1 概述

    ·定义

        锁是计算机协调多个进程或线程并发访问某一资源的机制。
        在数据库中，除传统的计算资源（如CPU、RAM、IO等）的争用以外，
        数据也是一种供许多用户共享的资源。如何保证数据并发访问的一致性、有效性，
        也是数据库必须解决的一个问题。同时，锁冲突也是影响数据库并发访问性能的一个重要因素。
        从这个角度来说，锁对数据库而言显得尤其重要，也更加复杂。

    ·分类

        -从对数据操作的粒度分：
            表锁、行锁
        -从对数据操作的类型分：
            读锁（共享锁）、写锁（排它锁）

    ·三锁

        表锁 - 偏读
            MyISAM引擎。
            开销小，加锁快；无死锁；锁定粒度大，发生锁冲突概率最高，并发度最低
        行锁 - 偏写
            InnoDB引擎。
            开销大，加锁慢；有死锁；锁定粒度小，发生锁冲突概率低，并发度高
        页锁

### 6.2 读锁·写锁示例

    建表

        create table mylock(
            id int primary key auto_increment,
            name varchar(20)
        )engine=myisam;

        insert into mylock(name) values('a'), ('b'), ('c'), ('d'), ('e'); 

    手动锁表

        lock table tbl1 read/write[, tbl2 read/write]; 

    查看上锁状况

        show open tables;

    释放表锁

        unlock tables;

    ※惊呆
        自己加了读·写锁不能读没有锁的表。

        自己加了读锁自己不能写，直接出错；
        自己加了读锁别人写会进入等待。

        自己加了写锁自己随便搞；别人读和写都是进入等待。

        读锁可以重复加，写锁重复加会等待。

    观察表锁定

        +----------------------------+-------+
        | Variable_name              | Value |
        +----------------------------+-------+
        | Table_locks_immediate      | 84    | 立即获取了表锁的查询次数
        | Table_locks_waited         | 0     | 获取表锁时经过了等待的查询次数
        +----------------------------+-------+

        
