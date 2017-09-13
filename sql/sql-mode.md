---
title: SQL Mode
category: user guide
---

# SQL Mode

TiDB服务器支持不同的 SQL Mode ，并且根据 `sql_mode` 系统变量的不同，客户端可以使用不同的 SQL Mode。数据库管理员可以设置全局的 SQL Mode ，每个应用可以设置它自己的 SQL Mode 来满足自己的需求。

不同的模式会影响 SQL 语句的记法以及数据操作的有效性检查。

## 设置 SQL Mode

TiDB的 SQL Mode 的默认值包括 `STRICT_TRANS_TABLES` 和 `NO_ENGINE_SUBSTITUTION` 。

通过设置全局的或者session级别的 `sql_mode` 系统变量可以在运行时改变 SQL Mode ：

    SET GLOBAL sql_mode = 'modes';
    SET SESSION sql_mode = 'modes';
    
设置全局变量需要 SUPER 权限，并且能影响所有连接的客户端。设置 session 级别的变量只影响当前的客户端。每个客户端都可以在任意时刻修改 `sql_mode` 的值。

使用下面表达式可以查询当前的全局或 session 的 `sql_mode` 值：

    SELECT @@GLOBAL.sql_mode;
    SELECT @@SESSION.sql_mode;
    
## 比较重要的 SQL Mode

最重要的一些 `sql_mode` 值可能是下面这些：

* ANSI

这个模式会改变 SQL 的词法表式，行为更加接近标准的 SQL 。

* STRICT\_TRANS\_TABLES

如果一个值不能够被插入到事务表中，则 abort 掉该 statement 。对于非事务表，如果值出现在单行 statement 中，或者多行 statement 的第一行中，则 abort 掉该 statement 。

* TRADITIONAL

这个模式的简单描述就是在插入一个非法值到某列的时候，“返回错误而不是警告”。

## 完整的 SQL Mode 列表

* ANSI\_QUOTES

将 \" 作为 identifier 的引用字符（跟 \` 字符类似）而不是字符串的引用字符。在这个模式下，你仍然可以使用 \` 引用 identifier 。当 `ANSI_QUOTES` 启用时，你不能使用双引号标识字符串字面量，因为它被会解释成 identifier 。

* STRICT\_ALL\_TABLES

启用严格模式。无效数据值将被拒绝。

* STRICT\_TRANS\_TABLES

启用严格模式。无效数据值将被拒绝。

## 严格 SQL Mode

严格 SQL Mode 控制 TiDB 在处理修改操作，比如 INSERT 或 UPDATE 时，遇到无效的值或者缺失值的行为。一个值可能因为各种原因是无效值。例如，可能对于某列是一个错误的数据类型，或者可能数值越界了。缺失值可能是插入一行，没有提供某一列的值，而这一列定义声明了非空并且没有提供默认值。

如果没有开启严格模式， TiDB 会自动处理无效或者缺失值，并生成警告。如果希望在严格模式下也是这种行为，可以使用 `INSERT IGNORE` 或者 `UPDATE IGNORE` 。

由于 `SELECT` 不会修改数据，在严格模式下，遇到无效值时只会生成警告，而不是错误。

当 SQL Mode 设置了 `STRICT_ALL_TABLES` 或者 `STRICT_TRANS_TABLES` 的时候严格模式生效。

严格模式会影响 除以零，日期为零，日期里面包含零 等行为。

+ 严格模式影响 除以零 的处理，包含 `MOD(N, 0)` ：
  - 如果严格模式未开启，除以零生成一个 NULL 并且不产生警告。
  - 如果严格模式开启，只要没有 IGNORE， 除以零将生成一个错误。如果是 `INSERT IGNORE` 或者 `UPDATE IGNORE` ，除以零将插入 NULL 并且生成一条警告。

+ 严格模式影响服务器是否将 `'0000-00-00'` 作为有效日期：
  - 如果严格模式未开启， `'0000-00-00'` 是有效的值并且插入它不会生成警告。
  - 如果严格模式开启， `'0000-00-00'` 不是有效值并且插入将产生一个错误。如果是 `INSERT IGNORE` 或者 `UPDATE IGNORE` ，插入成功并生成一条警告。

- 严格模式影响服务器是否允许日期数据里面出现0月或者0日（比如 `2010-00-01` 或 `2010-01-00` ）。


## IGNORE 对语句执行的影响

一些 SQL 语句支持 IGNORE 关键字。这个关键字将使服务器将部分返回错误的行为改成生成警告。对于多行的 statement ， IGNORE 会跳过错误继续下一行而不是 abort 。

例如，如果表 t 包含一个主键列 i ，插入多行重复的 i 会产生 duplicate-key 错误：

    mysql> INSERT INTO t (i) VALUES(1),(1);
    ERROR 1062 (23000): Duplicate entry '1' for key 'PRIMARY'
    
使用 IGNORE 之后，包含重复 key 的行依然可以插入，但是会报一个警告而不是错误：

    mysql> INSERT IGNORE INTO t (i) VALUES(1),(1);
    Query OK, 1 row affected, 1 warning (0.01 sec)
    Records: 2  Duplicates: 1  Warnings: 1
    mysql> SHOW WARNINGS;
    +---------|------|---------------------------------------+
    | Level   | Code | Message                               |
    +---------|------|---------------------------------------+
    | Warning | 1062 | Duplicate entry '1' for key 'PRIMARY' |
    +---------|------|---------------------------------------+
    1 row in set (0.00 sec)

## 严格模式对语句执行的影响

TiDB 可以在不同的模式下操作，根据 `sql_mode` 系统变量的不同，客户端可以在不同模式下执行。在严格模式下，特定的警告会被视为错误。

例如，非严格模式下，插入字符串 'abc' 到一个整数的列里面，结果会被自动转为0并且生成一条警告：

    mysql> SET sql_mode = '';
    Query OK, 0 rows affected (0.00 sec)
    mysql> INSERT INTO t (i) VALUES('abc');
    Query OK, 1 row affected, 1 warning (0.01 sec)
    mysql> SHOW WARNINGS;
    +---------|------|--------------------------------------------------------+
    | Level   | Code | Message                                                |
    +---------|------|--------------------------------------------------------+
    | Warning | 1366 | Incorrect integer value: 'abc' for column 'i' at row 1 |
    +---------|------|--------------------------------------------------------+
    1 row in set (0.00 sec)


在严格模式下，无效的值会被拒绝并产生一个错误：

    mysql> SET sql_mode = 'STRICT_ALL_TABLES';
    Query OK, 0 rows affected (0.00 sec)
    mysql> INSERT INTO t (i) VALUES('abc');
    ERROR 1366 (HY000): Incorrect integer value: 'abc' for column 'i' at row 1

