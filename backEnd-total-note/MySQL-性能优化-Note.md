

# MySQL优化笔记


## 参考资料
* 3万字总结，Mysql优化之精髓
    * 原文链接：https://blog.csdn.net/xinzhifu1/article/details/104228470
* 100道MySQL数据库经典面试题解析（收藏版）
    * https://juejin.im/post/5ec15ab9f265da7bc60e1910


## 优化原因
* 系统的吞吐量瓶颈往往出现在数据库的访问速度上
* 随着应用程序的运行，数据库的中的数据会越来越多，处理时间会相应变慢
* 数据是存放在磁盘上的，读写速度无法和内存相比

## 优化方法及原则
* 设计数据库时：数据库表、字段的设计，存储引擎
* 利用好MySQL自身提供的功能，如索引等
* 横向扩展：MySQL集群、负载均衡、读写分离
* SQL语句的优化（收效甚微）





### 原则：尽量使用整型表示字符串存储IP
     INET_ATON(str)，address to number
     INET_NTOA(number)，number to address

* decimal不会损失精度，存储空间会随数据的增大而增大。double占用固定空间，较大数的存储会损失精度。非定长的还有varchar、text

### 原则：尽可能选择小的数据类型和指定短的长度
### 原则：尽可能使用 not null
    非null字段的处理要比null字段的处理高效些！且不需要判断是否为null。
    null在MySQL中，不好处理，存储需要额外空间，运算也需要特殊的运算符。如select null = null和select null <> null（<>为不等号）有着同样的结果，只能通过is null和is not null来判断字段是否为null。
    如何存储？MySQL中每条记录都需要额外的存储空间，表示每个字段是否为null。因此通常使用特殊的数据进行占位，比如int not null default 0、string not null default ‘’

### 原则：单表字段不宜过多
    二三十个就极限了  why？

### 三大范式
* 第一范式1NF：字段原子性  关系型数据库，默认满足第一范式
* 第二范式：消除对主键的部分依赖  即在表中加上一个与业务逻辑无关的字段作为主键
* 第三范式：消除对主键的传递依赖

### 索引检索为什么快？
    关键字相对于数据本身，数据量小
    关键字是有序的，二分查找可快速确定位置

### MySQL中索引类型
    普通索引（key），唯一索引（unique key），主键索引（primary key），全文索引（fulltext key）
    三种索引的索引方式是一样的，只不过对索引的关键字有不同的限制：
    普通索引：对关键字没有限制
    唯一索引：要求记录提供的关键字不能重复
    主键索引：要求关键字唯一且不为null

* 我们可以通过 explain selelct 来分析SQL语句执行前的执行计划


### order by
    当我们使用order by将查询结果按照某个字段排序时，如果该字段没有建立索引，那么执行计划会将
    查询出的所有数据使用外部排序（将数据从硬盘分批读取到内存使用内部排序，最后合并排序结果），
    这个操作是很影响性能的，因为需要将查询涉及到的所有数据从磁盘中读到内存（如果单条数据过大
    或者数据量过多都会降低效率），更无论读到内存之后的排序了。
    
    但是如果我们对该字段建立索引alter table 表名 add index(字段名)，那么由于索引本身是有序
    的，因此直接按照索引的顺序和映射关系逐条取出数据即可。而且如果分页的，那么只用取出索引表
    某个范围内的索引对应的数据，而不用像上述那取出所有数据进行排序再返回某个范围内的数据

### 索引覆盖
如果要查询的字段都建立过索引，那么引擎会直接在索引表中查询而不会访问原始数据（否则只要有一个字段没有
建立索引就会做全表扫描），这叫索引覆盖。因此我们需要尽可能的在select后只写必要的查询字段，以增加索引覆盖的几率。

### like查询，不能以通配符开头
    比如搜索标题包含mysql的文章：
    select * from article where title like '%mysql%';
    这种SQL的执行计划用不了索引（like语句匹配表达式以通配符开头），因此只能做全表扫描，效率极低，在实际工程中几乎不被采用。而一般会使用第三方提供的支持中文的全文索引来做。
    但是 关键字查询 热搜提醒功能还是可以做的，比如键入mysql之后提醒mysql 教程、mysql 下载、mysql 安装步骤等。用到的语句是：
    select * from article where title like 'mysql%';
    这种like是可以利用索引的（当然前提是title字段建立过索引）。

* or，两边条件都有索引可用  一但有一边无索引可用就会导致整个SQL语句的全表扫描

### 如何创建索引
    建立基础索引：在where、order by、join字段上建立索引。
    优化，组合索引：基于业务逻辑
    如果条件经常性出现在一起，那么可以考虑将多字段索引升级为复合索引
    如果通过增加个别字段的索引，就可以出现索引覆盖，那么可以考虑为该字段建立索引
    查询时，不常用到的索引，应该删除掉

### B+Tree聚簇结构
聚簇结构（也是在BTree上升级改造的）中，关键字和记录是存放在一起的。
在MySQL中，仅仅只有Innodb的主键索引为聚簇结构，其它的索引包括Innodb的非主键索引都是典型的BTree结构。

### 读写分离
读写分离是依赖于主从复制，而主从复制又是为读写分离服务的。因为主从复制要求slave不能写只能读（如果对slave执行写操作，那么show slave status将会呈现Slave_SQL_Running=NO，此时你需要按照前面提到的手动同步一下slave）



### binlog
      当有数据写入到数据库时，还会同时把更新的SQL语句写入到对应的binlog文件里，这个文件就是上文说的binlog文件。使用mysqldump备份时，只是对一段时间的数据进行全备，但是如果备份后突然发现数据库服务器故障，这个时候就要用到binlog的日志了。

主要作用是用于数据库的主从复制及数据的增量恢复。

1.啥是binlog? 记录数据库增删改,不记录查询的二进制日志.
2.作用:用于数据恢复.





LIMIT OFFSET, ROW_COUNT 实现分页
存在性能问题的方式
SELECT * FROM myTable ORDER BY `id` LIMIT 1000000, 30
1
写出这样SQL语句的人肯定心里是这样想的：MySQL数据库会直接定位到符合条件的第1000000位，然后再取30条数据。

然而，实际上MySQL不是这样工作的。

LIMIT 1000000, 30 的意思是：扫描满足条件的1000030行，扔掉前面的1000000行，然后返回最后的30行。

可以利用子查询先定位出要查询的语句ID，然后读取数据，减少磁盘的I/O



### SQL注入

    * #{}能够很大程度上防止sql注入;
    - 1、用${}传入数据直接显示在生成的sql中，如上面的语句，用role_id = ${roleId,jdbcType=INTEGER},那么sql在解析的时候值为role_id= roleid，执行时会报错;
    - 2、${}方式无法防止sql注入;
    - 3、$一般用入传入数据库对象，比如数据库表名;
    - 4、能用#{}时尽量用#{}