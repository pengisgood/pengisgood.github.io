---
title: 在MySQL中使用LAST_INSERT_ID获取唯一自增序列
date: 2019-07-15 20:03:02
tags: [MySQL]
categories: [Database]
---

一般如果遇到生成全局唯一的自增ID的需求时，往往第一反应都是直接利用数据的Sequence对象，简单，直接了当。但是MySQL偏偏不支持Sequence对象，那我们该如何是好呢？

## 什么是Sequence

Sequence也叫做序列，一般用做表的主键，或者一些项目的编号等。一般具有以下几个特点：

* 全表唯一
* 自增
* 不一定严格连续（中间由于事务的回滚可能会出现`洞`，比如1，2，3，5，6）

<!-- more -->

在常见的几种数据库中，Oracle、SQL Server都内置有Sequence对象，具体用法就不在此赘述了。在本文中我们来讨论一下如何在原生不支持Sequence的MySQL（目前最新的大版本为8.0）中模拟出Sequence的效果。

## 如何在MySQL中模拟Sequence

MySQL中的`auto_increment`一般是用来生成表的主键，本身能够生成自增的唯一ID，但是一张表只能有一个列带有`auto_increment`属性。在实际项目中，我们可能需要不止一种序列号，比如项目编号（PROJ-001，PROJ-002...）、发票编号（INV-0001，INV-0002...），订单编号（ORD-0001，ORD-0002...）等等，下面将通过`auto_increment`和`LAST_INSERT_ID`相结合实现该功能。

###  LAST_INSERT_ID函数

该函数有两种形式：[`LAST_INSERT_ID()`](https://dev.mysql.com/doc/refman/8.0/en/information-functions.html#function_last-insert-id), [`LAST_INSERT_ID(expr)`](https://dev.mysql.com/doc/refman/8.0/en/information-functions.html#function_last-insert-id)。无参的形式会返回最近一次执行`INSERT`语句时`auto_increment`的值；带`expr`的形式会返回表达式的值，并且该值会被记住，在下一次调用`LAST_INSERT_ID()`时也返回该值。下面我们来看一个例子。

首先创建一张表：

```mysql
CREATE TABLE user (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL
);
```

然后插入两条数据：

```mysql
INSERT INTO user(name) VALUES('张三');
INSERT INTO user(name) VALUES('李四');

SELECT LAST_INSERT_ID();
```

此时得到的结果是2。

**注意：**如果是一条语句插入多条值，则返回的是插入第一条时自动生成的ID，而不是最后一条的。

比如我们再插入三条数据，不过换个写法：

```mysql
INSERT INTO user(name) 
VALUES('王五'),
      ('赵六'),
      ('郑七');

SELECT LAST_INSERT_ID();
```

此时得到的结果是3，而**不是**5。

### 获取自增序列

在实际的项目中我们完全可以换个方式，避免上面👆的情况，作为一个程序员，何必没有困难，制造困难为难自己呢？接着看下一个更加通用的例子。

创建另一张表并初始化数据：

```mysql
CREATE TABLE sequence (
    id INT AUTO_INCREMENT PRIMARY KEY,
    seq_type VARCHAR(50) NOT NULL,
    year INT NOT NULL,
    current_val BIGINT NOT NULL
);

INSERT INTO sequence(seq_type, year, current_val) VALUES('INVOICE', 2019, 0);
```

每次在获取`current_val`之前，先通过`LAST_INSERT_ID(current_val + 1)`更新：

```mysql
UPDATE sequence 
SET current_val = LAST_INSERT_ID(current_val + 1)
WHERE seq_type = 'INVOICE' AND year = 2019;

SELECT LAST_INSERT_ID();
```

这样每次都能获取自增之后的值了，但是也有例外的情况。比如两个人同时在获取新的值，A先做了update操作，然后B也做了update操作，然后A的操作由于某种原因回滚了，B的操作成功了，此时序列中间就会出现一个`洞`。虽然不是严格连续的，但是在大多数业务场景中，已经满足要求了。

**还需要注意的是**，如果`seq_type`或者`year`条件不满足，那么这里的`SELECT LAST_INSERT_ID();`就会始终返回上一次的值，可能会导致意想不到的的错误。

## LAST_INSERT_ID() vs. MAX()

`LAST_INSERT_ID()`是以数据库连接为基础的，即使有多个人同时通过多个连接获取Sequence也不会有问题，每个客户端会获取到属于他自己的序列号，不用担心会受到其他客户端的影响，或者影响其他客户端。在这种情况下，`MAX()`恐怕就不能正常工作了。

## 参考链接

> * https://www.percona.com/community-blog/2018/10/12/generating-identifiers-auto_increment-sequence/
> * https://dev.mysql.com/doc/refman/8.0/en/information-functions.html#function_last-insert-id
> * http://www.mysqltutorial.org/mysql-last_insert_id.aspx