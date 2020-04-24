# binlog

MySQL整体来看，由Server层和存储引擎层组成。`redo log`是InnoDB引擎特有的日志，而Server层也有自己的日志，称为binlog。

> 之所以会有两份日志，是因为最开始MySQL没用InnoDB引擎，MySQL自带的引擎是MyISAM，而MyISAM没有crash-safe的能力，binlog日志也只能用于归档。
>
> InnoDB是另一个公司以插件形式引入MySQL的，以实现crash-safe能力，即通过redo log实现。

`sync_binlog`这个参数设置成1的时候，表示每次事务的`binlog`都持久化到磁盘，设置成1可以保证MySQL异常重启之后`binlog`不丢失。

## 不支持崩溃恢复

![&#x53EA;&#x7528;binlog&#x652F;&#x6301;&#x5D29;&#x6E83;&#x6062;&#x590D;](https://github.com/wsfy15/gitbook/tree/0588f684e4ae072323ae076dad8931aee02e2f7e/.gitbook/assets/1587215472557.png)

假设只用binlog通过两阶段提交来实现崩溃恢复，这是不可行的，因为binlog没有能力恢复“数据页”。

如果在图中标的位置，也就是binlog2写完了，但是整个事务还没有commit的时候，MySQL发生了crash。

重启后，引擎内部事务2会回滚，然后应用binlog2可以补回来；但是对于事务1来说，系统已经认为提交完成了，不会再应用一次binlog1。

因为InnoDB引擎使用的是WAL技术，执行事务的时候，写完内存和日志，可能还没写磁盘，事务就算完成了。如果之后崩溃，要依赖于日志来恢复数据页。

也就是说在图中这个位置发生崩溃的话，事务1也是可能丢失了的，而且是数据页级的丢失。此时，binlog里面并没有记录数据页的更新细节，是补不回来的。而redo log会有checkpoint和write pos，两者间的操作都是未写入磁盘的。

如果优化一下binlog的内容，让它记录数据页的更改，其实这就是又做了一个redo log出来。

## 存在的必要性

如果只从崩溃恢复的角度来讲是可以把binlog关掉的，这样就没有两阶段提交了，但系统依然是crash-safe的。

不过，binlog有着redo log无法替代的功能。

* 归档。redo log是循环写，写到末尾是要回到开头继续写的。这样的历史日志没法保留，redo log也就起不到归档的作用。
* MySQL系统依赖于binlog。binlog作为MySQL一开始就有的功能，被用在了很多地方。其中，MySQL系统高可用的基础，就是binlog复制。
* 一些公司有异构系统（比如一些数据分析系统），这些系统就靠消费MySQL的binlog来更新自己的数据。关掉binlog的话，这些下游系统就没法输入了。

## 与redo log的区别

* `redo log`是InnoDB引擎特有的；
* `binlog`是MySQL的Server层实现的，所有引擎都可以使用。
* `redo log`是物理日志，记录的是“在某个**数据页**上做了什么修改”
* `binlog`是逻辑日志，记录的是这个语句的原始逻辑，比如“给`ID=2`这一行的c字段加1 ”
* `redo log`是循环写的，空间固定会用完
* `binlog`是可以追加写入的。“追加写”是指`binlog`文件写到一定大小后会切换到下一个，并不会覆盖以前的日志。

在执行更新语句`update T set c=c+1 where ID=1;`时，流程如下：

1. 执行器先找引擎取`ID=1`这一行。ID是主键，引擎直接用树搜索找到这一行。如果`ID=1`这一行所在的数据页本来就在内存中，就直接返回给执行器；否则，需要先从磁盘读入内存，然后再返回。
2. 执行器拿到引擎给的行数据，把这个值加上1，比如原来是N，现在就是N+1，得到新的一行数据，再调用引擎接口写入这行新数据。
3. 引擎将这行新数据更新到内存中，同时将这个更新操作记录到`redo log`里面，此时`redo log`处于prepare状态。然后告知执行器执行完成了，随时可以提交事务。
4. 执行器生成这个操作的`binlog`，并把`binlog`写入磁盘。
5. 执行器调用引擎的提交事务接口，引擎把刚刚写入的`redo log`改成提交（commit）状态，更新完成。

![1586843121296](https://github.com/wsfy15/gitbook/tree/48e0a955057b1c3dc9b2c5f445ace4015c27780c/.gitbook/assets/1586843121296.png)

图中浅色框表示是在InnoDB内部执行的，深色框表示是在执行器中执行的。

最后三步将`redo log`的写入拆成了两个步骤：prepare和commit，这就是"两阶段提交"。

> `commit`语句是MySQL语法中，用于提交一个事务的命令。一般跟`begin/start transaction`配对使用。
>
> 图中最后一步的`commit`步骤，指的是事务提交过程中的一个小步骤，也是最后一步。当这个步骤执行完成后，这个事务就提交完成了。
>
> 因此，`commit`语句执行的时候，会包含这个`commit`步骤。
>
> 这个例子里，没有显式地开启事务，因此这个`update`语句自己就是一个事务，在执行完成后提交事务时，就会用到这个“commit步骤“。

## 写入机制

binlog的写入逻辑比较简单：事务**执行过程中**，先把日志写到binlog cache，事务**提交的时候**，再把binlog cache写到binlog文件中。

**一个事务的binlog是不能被拆开的**，因此不论这个事务多大，也要确保一次性写入。这就涉及到了binlog cache的保存问题。

系统给binlog cache分配了一片内存，**每个线程一个**，参数 `binlog_cache_size`用于控制单个线程内binlog cache所占内存的大小。如果超过了这个参数规定的大小，就要暂存到磁盘。

事务提交的时候，执行器把binlog cache里的完整事务写入到binlog中，并清空binlog cache。

![binlog&#x5199;&#x76D8;&#x72B6;&#x6001;](https://github.com/wsfy15/gitbook/tree/9ee3721bf49692ec3d96660cdee86bb7a9b400d3/.gitbook/assets/9ed86644d5f39efb0efec595abb92e3e.png)

每个线程有自己binlog cache，但是共用同一份binlog文件。

* 图中的`write`，是指把日志写入到文件系统的page cache，并没有把数据持久化到磁盘，所以速度比较快。
* 图中的`fsync`，才是将数据持久化到磁盘的操作。一般情况下，我们认为`fsync`才占磁盘的IOPS。

`write`和`fsync`的时机，是由参数`sync_binlog`控制的：

1. `sync_binlog=0`的时候，表示每次提交事务都只`write`，不`fsync`；
2. `sync_binlog=1`的时候，表示每次提交事务都会执行`fsync`；
3. `sync_binlog=N(N>1)`的时候，表示每次提交事务都`write`，但累积N个事务后才`fsync`。

在出现IO瓶颈的场景里，将`sync_binlog`设置成一个比较大的值，可以提升性能。在实际的业务场景中，考虑到丢失日志量的可控性，一般不建议将这个参数设成0，比较常见的是将其设置为100~1000中的某个数值。

将`sync_binlog`设置为N的风险是：如果主机发生异常重启，会丢失最近N个事务的binlog日志。

