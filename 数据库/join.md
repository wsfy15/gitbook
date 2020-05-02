# JOIN

创建两个相同的表，有主键索引和字段`a`上的索引，`t2`有1000行，`t1`有100行，用它们进行join的实验。

```
CREATE TABLE `t2` (
  `id` int(11) NOT NULL,
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `a` (`a`)
) ENGINE=InnoDB;

drop procedure idata;
delimiter ;;
create procedure idata()
begin
  declare i int;
  set i=1;
  while(i<=1000)do
    insert into t2 values(i, i, i);
    set i=i+1;
  end while;
end;;
delimiter ;
call idata();

create table t1 like t2;
insert into t1 (select * from t2 where id<=100);
```

## Index Nested-Loop Join

```
mysql> select * from t1 straight_join t2 on (t1.a=t2.a);
```

用`straight_join`可以让MySQL使用固定的连接方式执行查询，即`t1` 是驱动表，`t2`是被驱动表。如果直接使用`join`语句，MySQL优化器可能会选择表`t1`或`t2`作为驱动表，这样会影响分析SQL语句的执行过程。

![使用索引字段join的 explain结果](join.assets/1588385693078.png)

从explain结果里可以看到，使用到了表`t2`的索引`a`，其执行流程如下：

1. 从表`t1`中读入一行数据 R；
2. 从数据行R中，取出`a`字段到表`t2`里去查找；
3. 取出表`t2`中满足条件的行，跟R组成一行，作为结果集的一部分；
4. 重复执行步骤1到3，直到表`t1`的末尾循环结束。

![Index Nested-Loop Join算法的执行流程](join.assets/1588386028283.png)

这个过程就跟写程序时的嵌套查询类似，并且可以用上被驱动表的索引，所以称之为“Index Nested-Loop Join”，简称NLJ。

在这个流程里，对驱动表`t1`做了全表扫描，扫描行数为100行。对于`t1`表的每一行，根据`a`字段去表`t2`查找，走的是树搜索过程。由于我们构造的数据都是一一对应的，因此每次的搜索过程都只扫描一行，也是总共扫描100行。

所以，这个流程扫描行数为200行。

如果不使用`join`，实现相同目的的流程如下：

1. 执行`select * from t1`，查出表`t1`的所有数据，这里有100行
2. 循环遍历这100行数据：
   - 从每一行R取出字段a的值`$R.a`；
   - 执行`select * from t2 where a=$R.a`；
   - 把返回的结果和R构成结果集的一行。

这个查询过程的扫描行数也是200行，但却执行了101条语句，比直接`join`多了100次交互。除此之外，客户端还要拼接SQL语句和结果。

### 驱动表选择

在上面的Index NLJ语句执行过程中，驱动表是走全表扫描，而被驱动表是走树搜索。

假设驱动表的行数是N，执行过程就要扫描驱动表N行，然后对于每一行，到被驱动表上匹配一次。

假设被驱动表的行数是M。每次在被驱动表查一行数据，要先搜索索引a，再搜索主键索引。每次搜索一棵树近似复杂度是$log_2M$，所以在被驱动表上查一行的时间复杂度是 $2*log_2M$。

因此整个执行过程，近似复杂度是 $N + N*2*log_2M$。显然，N对扫描行数的影响更大，因此应该**让小表来做驱动表**。



## Simple Nested-Loop Join

Simple NLJ相比Index NLJ，就是被驱动表上没有索引可以使用，例如下面的查询：

```
mysql> select * from t1 straight_join t2 on (t1.a=t2.b);
```

由于表`t2`的字段`b`上没有索引，因此每次到`t2`去匹配的时候，就要做一次全表扫描。因此，这条语句会对表`t2`做多达100次的全表扫描，总共扫描$100*1000=10万$行。

因此，MySQL没有使用这么笨重的算法，而是用了Block Nested-Loop Join，简称BNL。



## Block Nested-Loop Join

由于被驱动表上没有索引，BNL使用一片叫做join_buffer的内存存储驱动表的部分数据，然后取出被驱动表的每一行，看看是否与join_buffer中的数据满足join条件：

1. 把表`t1`的数据读入线程内存join_buffer中，由于这个语句用的是`select *`，因此是把整个表`t1`放入了内存
2. 扫描表`t2`，**把表`t2`中的每一行取出来，跟join_buffer中的数据做对比**，满足join条件的，作为结果集的一部分返回

![Block Nested-Loop Join 算法的执行流程](join.assets/1588387224421.png)

![不使用索引字段join的 explain结果](join.assets/1588388274454.png)

在这个过程中，对表`t1`和`t2`都做了一次全表扫描，因此总的扫描行数是1100。由于join_buffer是以无序数组的方式组织的，因此对表`t2`中的每一行，都要做100次判断，总共需要在内存中做的**判断次数**是：$100*1000=10万$次。

如果使用Simple Nested-Loop Join算法进行查询，**扫描行数**也是10万行。从时间复杂度上来说，这两个算法是一样的。但是，Block Nested-Loop Join算法的这**10万次判断**是内存操作，速度上会快很多，性能也更好。

### 驱动表选择

假设小表的行数是N，大表的行数是M，那么在这个算法里：

1. 两个表都做一次全表扫描，所以总的扫描行数是M+N；
2. 内存中的判断次数是M*N。

可以看到，这两个算式中的M和N对效率没有影响，因此选择大表还是小表做驱动表，执行耗时是一样的。



### join_buffer_size

join_buffer的大小是由参数`join_buffer_size`控制的，默认是256k，如果驱动表的大小比这个值大，那就**分段放**。

把`join_buffer_size`改成1200，再执行：

```
select * from t1 straight_join t2 on (t1.a=t2.b);
```

执行过程就变成了：

1. 扫描表`t1`，顺序读取数据行放入join_buffer中，放完第88行join_buffer满了，继续第2步；
2. 扫描表`t2`，把`t2`中的每一行取出来，跟join_buffer中的数据做对比，满足join条件的，作为结果集的一部分返回；
3. 清空join_buffer；
4. 继续扫描表`t1`，顺序读取最后的12行数据放入join_buffer中，继续执行第2步。

![Block Nested-Loop Join -- 两段](join.assets/695adf810fcdb07e393467bcfd2f6ac4.jpg)

这个流程才体现出了这个算法名字中“Block”的由来，表示“分块去join”。

由于表`t1`被分成了两次放入join_buffer中，**导致表`t2`会被扫描两次**。虽然分成两次放入join_buffer，但是判断等值条件的次数还是不变的，依然是$(88+12)*1000=10万$次。

#### 驱动表选择

假设，驱动表的数据行数是N，需要分K段才能完成算法流程，被驱动表的数据行数是M。这里的K与N成正比关系，因此把K表示为$λ*N$，显然$λ$的取值范围是$(0,1)$。

在这个算法的执行过程中：

1. 扫描行数是$ N+λ*N*M$；
2. 内存判断 $N*M$次。

内存判断次数不受选择哪个表作为驱动表影响，只需要考虑扫描行数，在M和N大小确定的情况下，N小一些，整个算式的结果会更小。所以还是**应该让小表当驱动表**。

N越大，分段数K越大。而N固定的时候，参数`join_buffer_size`会影响K的大小。`join_buffer_size`越大，一次可以放入的行越多，分成的段数也就越少，对被驱动表的全表扫描次数就越少。

因此，某些情况下如果`join`语句很慢，就把`join_buffer_size`改大。



## 能不能使用join？

在判断要不要使用`join`语句时，先看explain结果里面，Extra字段里面有没有出现“Block Nested Loop”字样。

- 如果可以使用Index Nested-Loop Join算法，也就是可以用上被驱动表上的索引，就没问题。
- 如果使用Block Nested-Loop Join算法，扫描行数就会过多。尤其是在大表上的join操作，这样可能要扫描被驱动表很多次，会占用大量的系统资源。所以这种join尽量不要用。

如果要使用join，总是应该使用小表做驱动表。

### 小表

```
select * from t1 straight_join t2 on (t1.b=t2.b) where t2.id<=50;
select * from t2 straight_join t1 on (t1.b=t2.b) where t2.id<=50;
```

上面两条语句都没有使用索引，第一个语句的join_buffer需要放`t1`的所有行，而第二个语句的join_buffer只需要放入`t2`的前50行。所以这里，“t2的前50行”是那个相对小的表，也就是“小表”。



```
select t1.b,t2.* from  t1  straight_join t2 on (t1.b=t2.b) where t2.id<=100;
select t1.b,t2.* from  t2  straight_join t1 on (t1.b=t2.b) where t2.id<=100;
```

表`t1 `和 `t2`都是只有100行参加join。但是，这两条语句每次查询**放入join_buffer中的数据是不一样的**：

- 表`t1`只查字段b，如果把`t1`放到join_buffer中，则join_buffer中只需要放入b的值
- 表`t2`需要查所有的字段，如果把表`t2`放到join_buffer中的话，就需要放入三个字段id、a和b。

这里“只需要一列参与`join`的表`t1`”是那个相对小的表。

**在决定哪个表做驱动表的时候，应该是两个表按照各自的条件过滤，过滤完成之后，计算参与join的各个字段的总数据量，数据量小的那个表，就是“小表”，应该作为驱动表。**



## join语句优化

如果被驱动表是一个大表，并且是一个**冷数据表**，除了查询过程中可能会导致IO压力大以外，还会导致BP内存数据页的淘汰，内存命中率降低等问题。那么，这种语句应该怎么优化呢？















