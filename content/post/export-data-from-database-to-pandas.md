---
title: "从数据库中导出数据到 pandas"
date: 2021-01-17T00:00:00+08:00
---

## 目标

从 Oracle 数据库中读取数据，然后进行数据预处理．首选的编程语言是 Python，毕竟是搞数据科学的嘛．

实际上，[上一篇文章](../brief-intro-to-pandas-data-structure)，我们刚刚介绍了 pandas 中的 `DataFrame` 数据结构．实际上，它与关系型数据库中的表，是可以对应起来的．那么，更加熟悉 pandas 那一套的数据科学工程师，从数据库获取数据，然后丢给 pandas 处理，也是自然而然的事情了．

## 如何解决

1. 确认如何使用 Python 连接 Oracle 数据库．这个不难，可以用 [cx_Oracle](https://oracle.github.io/python-cx_Oracle/)．使用 cx_Oracle 还需要安装 Oracle Instant Client，这个直接去 Oracle 官网下载安装即可．
2. 为了方便在 Python 中处理数据库，我们可以考虑使用 ORM，即 Object Relational Mapping．这样我们可以将数据库中的表的各种关系映射到 Python 中的一个对象，数据表中的各个字段和值，可以用对象中的属性和属性的值来表示，方便我们处理．这个，我们可以用 [SQLAlchemy](https://www.sqlalchemy.org/) 来解决．
3. 在数据科学中，我们常用 pandas 来处理数据．正好，pandas 也可以直接从 SQL 数据库中读取数据（使用 `pd.read_sql_table` ）．

此外，考虑到 Oracle 数据库是一种关系型数据库，数据表实际上是一个二维的数组（表格），而 pandas 中的 `DataFrame` 也是二维的数组（表格）表示．那么，可以考虑直接用 pandas 读取 Oracle 数据库中的数据．

## 稍微进阶

实际上，这个方法并不仅限于 Oracle 数据库，完全可以推广到关系型数据库，PostgreSQL，MySQL，Microsoft SQL Server 等等，都可以使用．只是这里我们用的 pandas 中的 `pd.read_sql_table` 实际上是用 SQLAlchemy 来连接数据库，实际上能用哪些数据库，得看 SQLAlchemy 的支持了．

## 一些小坑

最后，安装 Oracle Instant Client 有点麻烦，想要省事的，可以考虑去用 Oracle 的 Docker 镜像，不过需要稍微修改一下．参考 [OracleLinuxInstantClient](https://github.com/oracle/docker-images/blob/master/OracleInstantClient/oraclelinux7/21/Dockerfile) 和 [OracleLinuxDevelopers](https://github.com/oracle/docker-images/blob/master/OracleLinuxDevelopers/oraclelinux7/python/3.6-oracledb/Dockerfile) 的 Dockerfile，把 Oracle Instant Client 和 cx_Oracle 都装一起，最后再顺便把 pandas，SQLAlchemy 也装进一个镜像里即可．

SQLAlchemy 用小写字母来表示大小写不敏感的标识符，而在 Oracle 数据库中用大写字母来表示大小写不敏感的标识符，SQLAlchemy 会自动地做两者之间的转换．因此，我们在代码中应该总是使用小写字母（毕竟我们直接用的是 SQLAlchemy），除非这个标识符是大小写敏感的．具体请看 SQLAlchemy 的[这个文档](https://docs.sqlalchemy.org/en/14/dialects/oracle.html#identifier-casing)．

考虑到数据量的大小，读取数据的时候可能需要分块读取．不过这个也是 pandas 读写数据的基本操作了，应该不是太大的问题．

## 也许更好的解决方案

如果你不一定要用代码去连接读取数据库，那么实际上我们可以有更好或者说更简单的解决方案．个人推荐使用 [DBeaver](https://dbeaver.io/)，这是个开源的数据库工具，可以用来连接各种数据库．下载安装之后，添加数据库连接，按照提示填写数据库连接需要的 IP、端口、用户名、密码等信息即可连接．全部都有友好的操作界面，非常简单明了．略微坑一点的是，DBeaver 连接各种数据库需要对应的数据库驱动，这些都是在使用的时候自动下载安装的，受限于国内的网络环境，有的时候不太顺利．连接到数据库之后，用户可以很方便地查看到数据库中有哪些表、视图等等，鼠标点一点就可以按照需要将其导出为文件，比如 csv 文件．后续再使用 pandas 来做进一步的处理．当然，你也可以在 DBeaver 中写 SQL 语句去进行各种操作．但是我们主要的目的是从数据库获取数据，数据的处理并不用 SQL 语言来实现，而是丢给了 pandas．这样，也省去了学习 SQL 语言的负担．换上熟悉的 pandas 多方便呀．
