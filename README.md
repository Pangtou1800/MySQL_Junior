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


    
