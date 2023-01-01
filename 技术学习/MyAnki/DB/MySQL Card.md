#MySQLCard 
#八股文
#Anki
#Linux

#myanki

```ActivityHistory
/
```
```toc 
	style: bullet | number | inline (default: bullet) 
	min_depth: number (default: 2) max_depth: number (default: 6) 
	title: string (default: undefined) 
	allow_inconsistent_headings: boolean (default: false) 
	delimiter: string (default: |) 
	varied_style: boolean (default: false) 
```


# MySQL
## MySQL 基础
###  说一下 MySQL 执行一条查询语句的内部执行过程？
==连接器==、==缓存==、==分析器==、==优化器==、==执行器== <!--SR:!2022-08-31,3,150!2022-09-02,7,231!2022-08-30,4,211!2022-09-02,7,231!2022-09-02,7,231-->


* 连接器：::客户端先通过连接器连接到 MySQL 服务器。 <!--SR:!2022-08-31,7,210-->
* 缓存：::连接器权限验证通过之后，先查询是否有查询缓存，如果有缓存（之前执行过此语句）则直接返回缓存数据，如果没有缓存则进入分析器。 <!--SR:!2022-09-05,16,230-->
* 分析器：::分析器会对查询语句进行语法分析和词法分析，判断 SQL 语法是否正确，如果查询语法错误会直接返回给客户端错误信息，如果语法正确则进入优化器。 <!--SR:!2022-08-30,5,150-->
* 优化器：::优化器是对查询语句进行优化处理，例如一个表里面有多个索引，优化器会判别哪个索引性能更好。 <!--SR:!2022-09-01,13,230-->
* 执行器：::优化器执行完就进入执行器，执行器就开始执行语句进行查询比对了，直到查询到满足条件的所有数据，然后进行返回。 <!--SR:!2022-10-05,38,230-->

#TODO 手绘该过程

###  MySQL怎么建立索引，怎么建立主键索引，怎么删除索引？
MySQL 建立索引的主要方法5
用`alter table`或者`create index`。
```MySQL
alter table table_name add primary key(column_list) #添加一个主键索引
alter table table_name add index (column_list)      #添加一个普通索引
alter table table_name add unique (column_list)     #添加一个唯一索引
```
```MySQL
create index index_name on table_name (column_list)   #创建一个普通索引
create unique index_name on table_name (column_list)  #创建一个唯一索引
```

MySQL 删除索引的主要方法
用==alter table==或者==drop index==。
```MySQL
alter table table_name drop index index_name    #删除一个普通索引
alter table table_name drop primary key         #删除一个主键索引
```
```MySQL
drop index index_name on table table_name
```
<!--SR:!2022-09-04,7,148!2022-08-30,2,217-->

### Mysql的优化（高频，索引优化，性能优化）
#### 高频

#### 高频访问：
==分表分库==、==增加缓存==、==增加数据库的索引== <!--SR:!2022-09-09,11,168!2022-09-08,13,237!2022-09-11,13,217-->

* 分表分库：::将数据库表进行水平拆分，减少表的长度 <!--SR:!2022-09-01,8,168-->
* 增加缓存： ::在web和DB之间加上一层缓存层 <!--SR:!2022-10-14,49,268-->
* 增加数据库的索引：::在合适的字段加上索引，解决高频访问的问题 <!--SR:!2022-09-19,24,208-->

#### 并发优化：
==主从读写分离==、==负载均衡集群== <!--SR:!2022-09-01,6,188!2022-09-12,14,232-->

* 主从读写分离：::只在主服务器上写，从服务器上读 <!--SR:!2022-09-12,17,248-->
* 负载均衡集群：::通过集群或者分布式的方式解决并发压力 <!--SR:!2022-09-11,16,188-->


  
