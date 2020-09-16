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