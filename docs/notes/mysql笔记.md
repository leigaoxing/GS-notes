# 1. 存储过程

**存储过程：procedure**

概念类似于函数。就是把一段代码封装起来。
当要执行这一段代码的时候，可以通过调用该存储过程来实现。
在封装的语句体里面，可以使用if/else,case,while等控制结构。
可以经行sql编程

**查看现有的存储过程**。

```sql
show procedure status
```

**删除存储过程**

```sql
drop procedure 存储过程的名字
```

**调用存储过程**

```sql
call 存储过程的名字()
```

第一个存储过程，体会'封装‘sql的过程

```sql
create procedure p1()
begin 
  select * from g;
end$
```

第二个存储过程。体会“参数与控制结构”

```sql
create procedure p2(n int)
begin
  select * from g where num > n; 
end$
```

第三个存储过程。体会“控制结构”

```sql
create procedure p3(n int,j char(1))
begin
    if j = 'h' then
        select * from g where num > n; 
    else
        select * from g where num < n; 
    end if;
end$
```

```sql
#计算1->n的和
create procedure p4(n smallint)
begin 
  declare i int;
  declare s int;
  set i = 1;
  set s = 0;
  while i <= n do 
    set s = s+i;
    set i = i+1;
  end while;
  select s;
end$
```

在 mysql 中，存储过程和函数的区别：

- 一个是名称不同。
- 一个就是存储过程没有返回值。

# 2.触发器

对于insert来说，新增的行，用new来表示
行中的每一列的值，用new.列名表示

```sql
create trigger tg3 
after delete on o
for each row 
begin 
update g set num = num - old.num where id = old.gid;
end$
```


对于delete来说，原来有一行后来删除这一行。
想要表示这一行，old来表示，old.列名就可以引用被删除行中的值。

对于update来说，被修改的行。修改前的数据，用old来表示，old.列名引用被修改之前行中的值。
修改之后的数据，用new来表示。new.列名引用修改之后行中的值。

```sql
create trigger tg3 
after update on o
for each row 
begin 
update g set num = num + old.num - new.num where id = old.gid;
end$
```

触发器四要素：

- 监听地点  table
- 监听事件  insert/update/delete
- 触发事件  insert/update/delete
- 触发时间  after/before

定义触发器格式：

```sql
create trigger 名称
before/after  insert/update/delete on 表
(for each row -- mysql写死的）
begin
  sql 语句
end
```

# 3. 索引

1.什么是索引

建立索引的方式有两种：二叉树索引log2N、哈希（散列）索引

散列算法弊端：
1.算出来的值，不连续，空间浪费；
2.两个不同的数据 可能会出现散列值重复的情况；

碰撞性：即出现相同散列值
最大空间受限：硬盘空间限制，不能无限大

**索引加快了查询的速度，降低了增删改的速度；**

**先去掉索引，在导入数据，最后统一加索引；**

使用原则：

- 不要过度索引；
- 索引条件列（where后面最频繁的条件比较适宜索引）
- 索引散列值，多余集中的值不要索引。例如。给性别“男”，“女”加索引，意义不大。

索引创建语法：

1. 普通索引（index):仅仅是加快查询速度
2. 主键索引(primary key index)：主键不能重复
3. 唯一索引(unique index)：行上的值不能重复
4. 全文索引(Fulltext index):

主键必唯一，但是唯一不一定是主键；
一张表上只能有一个主键，但是可以用一个或多个唯一索引。

**查看一张表上的所有索引：**

```sql
show index from 表名
```

**建立索引**

```sql
Alter table 表名 add index/unique/fulltext [索引名] （列名）
```

primary key

```sql
Alter table 表名 add primary key（列名）//不要加索引名 因为主键只有一个
```

**删除索引**

```sql
Alter table 表名 drop index 索引名
```

主键索引删除

```sql
Alter table 表名 drop  primary key
```

索引：是针对数据所建立的目录；
作用：可以加快查询速度；
负面影响：降低了增删改的速度；

案例：设有新闻表15列，10列上有索引，共500行数据，如何快速导入？
1.把空表的索引全部删除；
2.导入数据；
3.数据导入完毕后集中建立索引；

# 重点可以看下以下文章

[MySQL索引背后的数据结构及算法原理](http://blog.codinglabs.org/articles/theory-of-mysql-index.html)

[重新学习Mysql数据库4：Mysql索引实现原理](https://yq.aliyun.com/articles/640002?spm=a2c4e.11153940.0.0.48e77fb8SmwKm9)

[面试题:MySQL性能调优——索引详解与索引的优化 没用](https://www.cnblogs.com/shan1393/p/8999622.html)



















