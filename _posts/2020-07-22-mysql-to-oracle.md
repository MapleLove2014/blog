---
title: 手动迁移Mysql到Oracle的各种坑
author: ZhangMapler
date: 2020-07-22 18:19:00 +0800
categories: [Database]
tags: [mysql, oracle]
math: false
---

# 手动迁移mysql到oracle遇到的坑和解决方法

> 本次迁移的oracle数据库版本为 11g release2，一个比较老的版本。有可能在新版本的oracle上不适用喔

## 本地oracle安装

迁移的第一步首先是在本地安装个oracle环境，本地迁移验证没有问题后，再应用到测试环境和生产环境上。本来以为现在这种成熟的软件都是一键式傻瓜安装了，结果马上就劈里啪啦打脸。

1. 本地win10环境安装打不开，换个express版本安装也不行

2. 通过win10 v2004发布的沙箱功能安装以下，也不行。这就很奇怪了，因为沙箱是一个很纯净的win10系统，为什么也不行呢？

3. windows上不行，干脆上linux。先使用win10的windows subsystem for linux，ubuntu 16 按着官方的安装文档装一下，果不其然，还是不行

4. 难道是wsl有问题，直接上linux虚拟机试一下 centos7，还是不行，即将崩溃

5. 第二天上班，思来想去，突然想到还有docker。抱着试一试的心态去搜索了下，结果还真有镜像。就这一瞬间我感觉他就是我的救命稻草了，顺利启动，终于是安装成功了。

跟着[镜像官网](https://hub.docker.com/r/oracleinanutshell/oracle-xe-11g)上的命令执行即可，它提供了多种运行选择，我使用的是第二个，接下来会说连接串

```
docker run -d -p 49161:1521 -e ORACLE_ALLOW_REMOTE=true oracleinanutshell/oracle-xe-11g
```

启动后可以通过命令 `docker ps` 查看是否运行成功

> 如果是在wsl2上面使用docker，有可能会把电脑的内存跑满。如果是这样的话，参考这个临时的[解决方案](https://github.com/microsoft/WSL/issues/4166#issuecomment-526725261)

![图1]({{ "/assets/img/mysql2oracle/wsl-rc.png"}})

在当前用户主目录下创建一个.wslconf文件，文件内容：
```
[wsl2]
memory=2GB
swap=4GB
localhostForwarding=true
```
其中 `localhostForwarding` 表示win10上能不能通过`localhost`的方式访问到wsl2

折腾了一天成功安装后，静下心想想，感觉自己还是变了。放在几年前，大学时期，对于为什么安装失败，为什么打不开这种问题，我会一直陷入其中。想尽各种办法去解决这个问题。我记得当时自己为了安装visual studio，重装了好几次系统，换了好几次版本。整个过程说实话，我会变得非常暴躁，焦急，抓狂，欲哭无泪，郁闷。好像是一个心病，忽略它吧，心里总是痒痒的。就算是解决问题了，也还是会很沮丧，会一直想：搞了几天，就只安装了一个软件，多浪费时间，啥也没干。现在我是想通了，甚至感觉以前自己很傻X。其实它是一个主次的问题，是一个任务优先级的问题。自己主要的任务是要用软件完成业务，而不是花费大量的时间只是搞懂怎么能成功安装上。而且安装本身我认为并没有啥技术含量，安装过程中出现的问题本质上是软件本身创造出的问题（糟糕的设计，对环境考虑不周到），花费时间在这个上面没有任何意义，当然了，如果你是专门搞这个的，就另当别论了。既然当前环境安装失败，那就换个环境再试一下，不必头铁，一直陷在其中，这样真的超级浪费时间，还搞人心态(因为，大部分的这种问题，特别是和个人计算机环境相关的问题，很难解决掉，大概率解决不了)。最后用docker顺利运行，然后抓紧去搞迁移的任务。BTW，docker真的强呀，真的傻瓜式安装，这些和环境相关的问题，别人都已经给做好了，简直不要太舒服。

## 使用sql developer连接

既然安装好了，怎么连接呢。既然是oracle，这里还是推荐使用oracle官方的sql developer吧。虽然navicat也可以，但是个人感觉体验差了一点。

用户名，密码啥的镜像的网站上有，我贴一下自己的连接属性：

![图2]({{ "/assets/img/mysql2oracle/local-oracle-connection.png"}})

密码是：oracle

## Java Datasource配置

如果是MySQL数据库，连接串我们就很熟悉了，大概长这样：jdbc:mysql://localhost:3306/some-db?useUnicode=true&characterEncoding=UTF-8

Oracle也差不多，只不过把协议scheme变一下，首先要查看下当前的数据库，方法如下，命令来自[这里](https://stackoverflow.com/questions/8978047/how-to-query-database-name-in-oracle-sql-developer)

```
select * from v$database;

or

select ora_database_name from dual;
```

本人的docker环境上，数据库为 `xe`，大小写无所谓，oracle最后会统一转为大写

然后再参考下[这篇文章](https://www.iteye.com/blog/pgwcumt-1873415)，最终的连接串长这样：

```
jdbc:oracle:thin:@localhost:49161/XE
```
> 至于SID，service name和database有啥区别，我又不晓得了

由于项目使用的是`sharding-jdbc`，最终的配置如下：

```
spring.shardingsphere.datasource.master.driver-class-name = oracle.jdbc.OracleDriver
spring.shardingsphere.datasource.master.url = jdbc:oracle:thin:@localhost:49161/XE
spring.shardingsphere.datasource.master.username = system
spring.shardingsphere.datasource.master.password = oracle
```

> 其他Datasource的实现框架配置大同小异

## mysql和oracle字段类型对照

环境准备好后，要准备迁移的sql脚本了。首先在navicat上对mysql上的表进行 `dump structure and data`，然后对脚本进行编辑。

第一步要做的就是表的基本字段的修改，先看下两种数据库的类型[对照表](https://docs.oracle.com/cd/E12151_01/doc.150/e12155/oracle_mysql_compared.htm#BABHHAJC)，还有sql标准和oracle字段类型[对照表](https://www.oracletutorial.com/oracle-basics/oracle-data-types/)

针对常用的数据类型，总结下来大概有以下几点：

1. mysql的varchar(n)是可变长度的字符串类型，其中`n`表示最大可以存储的字符数。从[官网文档](https://dev.mysql.com/doc/refman/8.0/en/char.html)摘一段

    > The CHAR and VARCHAR types are declared with a length that indicates the maximum number of characters you want to store. For example, CHAR(30) can hold up to 30 characters.

    mysql的varchar对应于oracle varchar2类型，其中varchar2类型的长度在下一节再提

2. 在mysql中常用于作为自增主键的类型bigint，在oracle中对应于number(19,0)

3. 在mysql中存储长文本的longtext类型，在oracle中对应于clob

4. 在mysql中存储枚举值、boolean、小整数的tinyint，在oracle中对应于number

5. 在mysql中表示时间的datetime和timestamp类型，在oracle中对应于date

根据此对应关系，相应的修改sql脚本即可，举个栗子：

mysql的表结构：
```sql
CREATE TABLE user (
  id bigint(20) NOT NULL,
  user_id char(32) NOT NULL,
  username varchar(32) NOT NULL,
  is_responsible tinyint(1) NOT NULL DEFAULT 1 ,
  state tinyint(4) NOT NULL DEFAULT 1,
  create_time datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  update_time datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
);
```
变换成oracle后为：
```sql
CREATE TABLE user (
  id number(19,0) NOT NULL,
  user_id char(32) NOT NULL,
  username varchar2(32) NOT NULL,
  is_responsible number(1,0) DEFAULT 1 NOT NULL  ,
  state number(4,0) DEFAULT 1 NOT NULL ,
  create_time date DEFAULT CURRENT_DATE NOT NULL ,
  update_time date DEFAULT CURRENT_DATE NOT NULL ,
  PRIMARY KEY (id),
);
```

## varchar2(16char)或者char(16char)和varchar2(16)或者char(16)的区别

16char表示可以存储16个字符，单个16呢表示的是字节的单位。我们从这个[文档](https://www.oracletutorial.com/oracle-basics/oracle-varchar2/)上摘抄一段

```
When you create a table with a VARCHAR2 column, you must specify the maximum string length,   either in bytes:

VARCHAR2(max_size BYTE)

or in characters

VARCHAR2(max_size CHAR)

By default, Oracle uses BYTE if you don’t explicitly specify BYTE or CHAR after the max_size. In other words, a VARCHAR2(N) column can hold up to N bytes of characters.
```

## varchar2会将mysql中的空字符串('')当成NULL处理

往oracle中varchar2类型 `NOT NULL` 字段中插入空字符串 `''` 会报错。原因是oracle把空字符串当成了 `null` 处理，参考这篇[回答](https://stackoverflow.com/questions/49678159/insert-empty-string-in-oracle)的解决方法为：

1. 定义时使用 `DEFAULT NULL`

2. 定义后通过 `ALTER` 语句修改

```sql
alter table user modify name varchar2(32char) DEFAULT NULL
```

## id怎么用触发器自动自增

自增主键id在mysql中只需要把 `AUTO_INCREMENT` 加上即可，但是在oracle中就没这么方便了，要么程序手动设置，或者触发器自动更新。当然oracle新版本增加了自增的属性，具体实现大家可以参考这篇[回答](https://stackoverflow.com/a/11296469/6147582)

## 时间字段的处理

时间的类型在oracle建议使用date类型，在mysql中通常会给表创建两个通用字段：`create_time` 和 `update_time`，分别表示记录的创建时间和更新时间。mysql旧版本中支持对时间设为当前时间，新版本支持 `ON UPDATE` 来更新时间。但在oracle中就没这么简答了，那么具体在oracle怎么实现时间的自动设置呢，我们先来看 `create_time` 

### create_time默认值

对于`create_time`而言，只需要设为默认值即可，oracle的语法为：`DEFAULT CURRENT_DATE`，当然也可以使用其他的日期函数，比如 `SYS_DATE`等

### update_time怎么用触发器自动更新

对于 `update_time` 首先要设置和 `create_time` 相同的默认值，语法相同。时间更新可以用两种方式来实现：

1. 记录 `update` 时，手动设置 `update_time` 字段

    ```java
    UserDO user = new UserDO();
    user.setUpdateTime(LocalDateTime.now());
    sqlSession.update(user);
    ```

2. 使用oracle的触发器

    当执行 `update` 操作时，触发器监听 `update` 事件，对记录进行修改：
    
    ```sql
    CREATE OR REPLACE TRIGGER user_update_time_trigger
    BEFORE UPDATE ON user
    FOR EACH ROW
    BEGIN
      if :new.update_time is NULL OR :new.update_time = :old.update_time then
        :new.update_time := CURRENT_DATE;
      end if;
    END;
    ```

    如果对于 `:new.update_time` 和 `:old.update_time` 的值不清楚的话，可以看下这个[简单的调试方法](https://stackoverflow.com/a/3732385/6147582)

    > 另外要说一点的是，不建议使用数据库的触发器、存储过程等在sql上执行的逻辑程序。原因大概有以下几点：

    1. 难开发，难调试：每个数据库的编程语言的语法特性往往都不同，增大开发难度。并且数据库中的代码调试不方便，错误定位分析麻烦

    2. 管理：业务逻辑代码往往能通过git，maven，jenkins，workflow等工具组件，很方便的进行版本管理，自动构建发布等。相反，数据库上的程序代码就很难做到，即使可以通过版本工具进行托管，往往在发布的时候也需要手动将代码编译到数据库中

    3. 数据库依赖：数据库上的程序强绑定数据库，使得数据库迁移麻烦

    4. 数据库性能瓶颈：在数据库上执行太多的业务逻辑代码将会占用数据库宝贵的资源，甚至会级联影响其他服务。另外，数据库的伸缩扩容往往难于应用程序

    5. 单一责任原则：数据库应该只作为最简单的数据存储服务来使用，不应该侵入业务

## oracle identifier length限制

创建触发器，索引，序列时，要注意名字不能超过 `30` 个字符，具体可以看这篇[文章](https://www.tekstream.com/resource-center/ora-00972-identifier-is-too-long/) 。不过[oracle新版本](https://oracle-base.com/articles/12c/long-identifiers-12cr2)已经可以支持 `128` 个字符，关于长度限制就大可不必担心了。

## oracle索引，序列和触发器名字不能重复

上面说的是长度限制，另外要注意的是名字也不能重复，注意是全局不能重复。比如索引的名字是 `A`，创建名字为 `A` 的序列时就会失败，再另一张表上创建名称为 `A`的索引也会失败。我真的搞不明白，oracle为什么要这么反人类。在mysql中就没有这种担心，大胆创建即可，保证单表下名称唯一就行了。

## 连续创建触发器需要分割符

在一个sql脚本中连续创建触发器是会报错的，[解决方法](https://blog.csdn.net/zp357252539/article/details/102620877)为：

```sql
END;   /* trigger end */
/
alter trigger trigger enable;
```

## 建表语句DEFAULT VALUE要在 NOT NULL后

这个就是纯语法特性了，没什么好说的

## 不能直接使用COMMENT

mysql中可以很方便的定义列时在最后加上CMMENT属性，但在oracle中是不支持的，做法是通过单独的COMMENT语句：

```sql
COMMENT ON COLUMN user.user_id IS '用户id';
```

## oracle字段名千万不要使用保留字

首先看下oracle的[保留字](https://docs.oracle.com/cd/B19306_01/em.102/b40103/app_oracle_reserved_words.htm)，业务中可能常用到的有 `LEVEL` , `ACCOUNT`, `COST` , `UID` 等。在mysql可以使用 `backtick(``)` 来转意，但oracle只能使用双引号，但会影响查询的语句。

比如将LEVEL改为 "LEVEL"，那么查询的语句也要带上双引号。所以如果出现这种情况，还是建议修改字段名吧。

## 程序中减少原生的sql

数据库的迁移只是开始，程序往往免不了进行修改。就以这次迁移为例，程序中对mapper修改比较多。也正是通过这次修改，让我深深的意识到使用框架的重要性。本人在程序中基本不写原生sql，不写表字段名常量，查询时一律采用lambda的方式进行增删改查，对表字段的访问是通过方法引用来完成的，并且本人也在团队中不止一次的强调推广使用这种方式的好处。在本次的迁移过程中就直接体现出来了，基本不需要修改，框架已经做了兼容。当然了对于复杂的多表联合分页查询还是免不了写sql的，这些地方需要修改。

这里说一下为什么不建议写原生sql：

1. 数据库依赖：每个数据库的语法特性都不太相同，数据迁移麻烦

2. 易出错：敲字段名时，往往一不注意就会敲错

3. 对重构不友好：举一个简单的场景，当数据表的字段修改时。如果是原生的sql，那么需要全局搜索字段进行替换，非常麻烦且容易出错。如果使用lambda+字段方法引用的方式，只需要对数据库实体对象名进行refactor `rename` 即可。方便，且不会出错。

```java
User getUserByName(String name){
  return mapper.select(Wrappers.<User>lambdaQuery().eq(User::getName, name));
}
```

另外推荐下mybatis的分页插件 [Pagehelper](https://github.com/pagehelper/Mybatis-PageHelper)，分页的超级兼容框架，非常好用

## 总结

这次数据迁移一度搞得我心力交瘁，主要是之前工作中也没用过oracle，不清楚它和mysql的不同。光安装就浪费了很长时间。不过迁移完还是能够学到oracle的不少东西，以及工程实践上的经验。




