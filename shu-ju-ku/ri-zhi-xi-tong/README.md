# 日志系统

通过更新语句的执行过程介绍日志系统。首先创建表`T`：

```text
mysql> create table T(ID int primary key, c int);
```

执行更新语句`update T set c=c+1 where ID=1;`时，根据语句的执行流程，在连接数据库后，会将与表`T`相关的所有查询缓存失效，然后通过词法分析和语法分析知道这是一条合法的更新语句，优化器决定使用`ID`索引，由执行器负责具体执行，找到该行并更新。

与查询流程不同的是，更新流程还涉及`redo log`（重做日志）和 `binlog`（归档日志）。

## redo log

当执行更新语句时，如果要同时写入磁盘，那么磁盘需要找到对应的记录，然后更新，对于单个更新操作，这个过程IO成本、查找成本比较高，降低了MySQL的运行效率。

因此，可以先将更新语句写入日志（追加方式比较快，且不需要打开日志文件的开销，日志文件句柄应该已经由某些线程控制了），（不忙的时候）再写磁盘，即**WAL技术**（Write-Ahead Logging）。

> 可能有一个线程负责日志IO，当执行语句的线程执行到更新处时，只需要往全局的日志接口添加该更新的日志，负责日志的线程定时读取内存中的批量日志，追加方式写入redo log，这样效率比较高，不会阻塞执行线程。
>
> 不过，对于redo log，应该是每有一个更新操作，就会写入磁盘上的日志文件，这样才能保证所有执行过的操作都被记录。

具体来说，当有一条记录需要更新的时候，InnoDB引擎就会先把记录写到`redo log`里面，并更新内存，这个时候更新就算完成了。同时，InnoDB引擎会在适当的时候，将这个操作记录更新到磁盘里面，而这个更新往往是在系统比较空闲的时候做。

不过，`redo log`的大小是有限制的，例如，可以配置为一组4个文件，每个文件大小为1GB，那么最多可以记录4GB的操作。写的方式是**从头开始写，写到末尾就又回到开头循环写**。

![](../../.gitbook/assets/image%20%284%29.png)

* `write pos`是当前记录的位置，一边写一边后移，写到第3号文件末尾后就回到0号文件开头。
* `checkpoint`是还未写入磁盘的操作的起点，也是往后推移并且循环的，后移时需要将该记录更新到磁盘。

`write pos`和`checkpoint`之间的空间可以用来记录新的操作，如果`write po`追上了 `checkpoint`，即当MySQL负载一直很高时，没空后移`checkpoint`，导致`redo log`写满了，那么这时就只能先停止对外的服务，先将`redo log`中的一部分更新到磁盘上，推进`checkpoint`，为接下来的更新腾出空间。

有了redo log，InnoDB就可以保证即使数据库发生异常重启，之前提交的记录都不会丢失，这个能力称为**crash-safe**。

`innodb_flush_log_at_trx_commit`这个参数设置成1的时候，表示每次事务的`redo log`都直接持久化到磁盘，设置成1可以保证MySQL异常重启之后数据不丢失。

### 大小设置

redo log太小的话，会导致很快就被写满，然后不得不强行刷redo log，这样WAL机制的能力就发挥不出来了。

所以，如果是现在常见的几个TB的磁盘的话，那就直接将redo log设置为4个文件、每个文件1GB。

### redo log buffer

在一个事务的更新过程中，日志是要写多次的。比如下面这个事务：

```text
mysql> begin;
mysql> insert into t1 ...
mysql> insert into t2 ...
mysql> commit;
```

这个事务要往两个表中插入记录，插入数据的过程中，生成的日志都得先保存起来，但又不能在还没commit的时候就直接写到redo log文件里。

因此这个阶段的日志都是写到redo log buffer，它是一块内存，用来先存redo日志。也就是说，在执行第一个`insert`的时候，数据的内存被修改了\(change buffer\)，相应的日志写入到了redo log buffer。

但是，真正把日志写到redo log磁盘文件（文件名是 ib\_logfile+数字），是在执行`commit`语句的时候做的。就算在`commit`执行前crash了，也没有关系，因为之前修改的数据页都是内存中的，redo log也没有写入到redo log磁盘文件，崩溃了就没了，磁盘的数据并没有被修改。

> 这里说的事务执行过程中不会“主动去刷盘”，以减少不必要的IO消耗。
>
> 但是可能会出现“被动写入磁盘”，比如内存不够、其他事务提交等情况。这种情况下是否会将包含未提交事务的数据变更的数据页写入磁盘？

单独执行一个更新语句的时候，InnoDB会自己启动一个事务，在语句执行完成的时候提交。过程跟上面是一样的，只不过是“压缩”到了一个语句里面完成。

### 写入机制

事务在执行过程中，生成的redo log是要先写到redo log buffer的，等到事务提交时才持久化到磁盘。所有线程共享同一个redo log buffer。

redo log buffer里的内容，不需要每次在生成时就持久化，因为此时事务还未提交，如果这时MySQL异常重启，这部分log丢失并不会带来影响。

但是在事务还没提交的时候，redo log buffer中的部分日志有可能被持久化到磁盘。

> **redo log的三种状态**
>
> ![MySQL redo log&#x5B58;&#x50A8;&#x72B6;&#x6001;](../../.gitbook/assets/1587701630038.png)
>
> * 红色部分：存在redo log buffer中，物理上是在MySQL进程内存中
> * 黄色部分：写到磁盘\(`write`\)，但是没有持久化（`fsync`\)，物理上是在文件系统的page cache里面
> * 绿色部分：持久化到磁盘，对应的是hard disk

日志写到redo log buffer是很快的，`wirte`到page cache也差不多，但是持久化到磁盘的速度就慢多了。

为了控制redo log的写入策略，InnoDB提供了`innodb_flush_log_at_trx_commit`参数，它有三种可能取值：

* `0`：每次事务提交时都只是把redo log留在redo log buffer中;
* `1`：表示每次事务提交时都将redo log直接持久化到磁盘；
* `2`：表示每次事务提交时都只是把redo log写到page cache。

InnoDB有一个后台线程，每隔1秒，就会把redo log buffer中的日志，调用`write`写到文件系统的page cache，然后调用`fsync`持久化到磁盘。

事务执行中间过程的redo log也是直接写在redo log buffer中的，这些redo log也会被后台线程一起持久化到磁盘。也就是说，一个没有提交的事务的redo log，也是可能已经持久化到磁盘的。

除了后台线程每秒一次的轮询操作外，还有两种场景会让一个没有提交的事务的redo log写入到磁盘中。

1. **redo log buffer占用的空间即将达到 `innodb_log_buffer_size`一半的时候，后台线程会主动写盘。**注意，由于这个事务并没有提交，所以这个写盘动作只是`write`，而没有调用`fsync`，也就是只留在了文件系统的page cache。
2. **并行的事务提交的时候，顺带将这个事务的redo log buffer持久化到磁盘。**假设一个事务A执行到一半，已经写了一些redo log到buffer中，这时候有另外一个线程的事务B提交，如果`innodb_flush_log_at_trx_commit`设置的是1，那么按照这个参数的逻辑，事务B要把redo log buffer里的日志全部持久化到磁盘。这时候，就会带上事务A在redo log buffer里的日志一起持久化到磁盘。

两阶段提交的时候，时序上redo log先prepare， 再写binlog，最后再把redo log commit。

如果`innodb_flush_log_at_trx_commit`设置成1，那么redo log在prepare阶段就要持久化一次，因为有一个崩溃恢复逻辑是要依赖于prepare 的redo log，再加上binlog来恢复的。

每秒一次后台轮询刷盘，再加上崩溃恢复这个逻辑，InnoDB就认为redo log在commit的时候就不需要`fsync`了，只会write到文件系统的page cache中就够了。

## binlog

MySQL整体来看，由Server层和存储引擎层组成。上面介绍的的`redo log`是InnoDB引擎特有的日志，而Server层也有自己的日志，称为binlog。

> 之所以会有两份日志，是因为最开始MySQL没用InnoDB引擎，MySQL自带的引擎是MyISAM，而MyISAM没有crash-safe的能力，binlog日志也只能用于归档。
>
> InnoDB是另一个公司以插件形式引入MySQL的，以实现crash-safe能力，即通过redo log实现。

`sync_binlog`这个参数设置成1的时候，表示每次事务的`binlog`都持久化到磁盘，设置成1可以保证MySQL异常重启之后`binlog`不丢失。

### 不支持崩溃恢复

![&#x53EA;&#x7528;binlog&#x652F;&#x6301;&#x5D29;&#x6E83;&#x6062;&#x590D;](https://github.com/wsfy15/gitbook/tree/0588f684e4ae072323ae076dad8931aee02e2f7e/.gitbook/assets/1587215472557.png)

假设只用binlog通过两阶段提交来实现崩溃恢复，这是不可行的，因为binlog没有能力恢复“数据页”。

如果在图中标的位置，也就是binlog2写完了，但是整个事务还没有commit的时候，MySQL发生了crash。

重启后，引擎内部事务2会回滚，然后应用binlog2可以补回来；但是对于事务1来说，系统已经认为提交完成了，不会再应用一次binlog1。

因为InnoDB引擎使用的是WAL技术，执行事务的时候，写完内存和日志，可能还没写磁盘，事务就算完成了。如果之后崩溃，要依赖于日志来恢复数据页。

也就是说在图中这个位置发生崩溃的话，事务1也是可能丢失了的，而且是数据页级的丢失。此时，binlog里面并没有记录数据页的更新细节，是补不回来的。而redo log会有checkpoint和write pos，两者间的操作都是未写入磁盘的。

如果优化一下binlog的内容，让它记录数据页的更改，其实这就是又做了一个redo log出来。

### binlog存在的必要性

如果只从崩溃恢复的角度来讲是可以把binlog关掉的，这样就没有两阶段提交了，但系统依然是crash-safe的。

不过，binlog有着redo log无法替代的功能。

* 归档。redo log是循环写，写到末尾是要回到开头继续写的。这样的历史日志没法保留，redo log也就起不到归档的作用。
* MySQL系统依赖于binlog。binlog作为MySQL一开始就有的功能，被用在了很多地方。其中，MySQL系统高可用的基础，就是binlog复制。
* 一些公司有异构系统（比如一些数据分析系统），这些系统就靠消费MySQL的binlog来更新自己的数据。关掉binlog的话，这些下游系统就没法输入了。

### 与redo log的区别

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

### 写入机制

binlog的写入逻辑比较简单：事务**执行过程中**，先把日志写到binlog cache，事务**提交的时候**，再把binlog cache写到binlog文件中。

**一个事务的binlog是不能被拆开的**，因此不论这个事务多大，也要确保一次性写入。这就涉及到了binlog cache的保存问题。

系统给binlog cache分配了一片内存，**每个线程一个**，参数 `binlog_cache_size`用于控制单个线程内binlog cache所占内存的大小。如果超过了这个参数规定的大小，就要暂存到磁盘。

事务提交的时候，执行器把binlog cache里的完整事务写入到binlog中，并清空binlog cache。

![binlog&#x5199;&#x76D8;&#x72B6;&#x6001;](../../.gitbook/assets/9ed86644d5f39efb0efec595abb92e3e.png)

每个线程有自己binlog cache，但是共用同一份binlog文件。

* 图中的`write`，是指把日志写入到文件系统的page cache，并没有把数据持久化到磁盘，所以速度比较快。
* 图中的`fsync`，才是将数据持久化到磁盘的操作。一般情况下，我们认为`fsync`才占磁盘的IOPS。

`write`和`fsync`的时机，是由参数`sync_binlog`控制的：

1. `sync_binlog=0`的时候，表示每次提交事务都只`write`，不`fsync`；
2. `sync_binlog=1`的时候，表示每次提交事务都会执行`fsync`；
3. `sync_binlog=N(N>1)`的时候，表示每次提交事务都`write`，但累积N个事务后才`fsync`。

在出现IO瓶颈的场景里，将`sync_binlog`设置成一个比较大的值，可以提升性能。在实际的业务场景中，考虑到丢失日志量的可控性，一般不建议将这个参数设成0，比较常见的是将其设置为100~1000中的某个数值。

将`sync_binlog`设置为N的风险是：如果主机发生异常重启，会丢失最近N个事务的binlog日志。

## 两阶段提交

为什么要采用两阶段提交？

对数据库备份和恢复是通过`binlog`实现的，`binlog`会记录所有的逻辑操作，并且是采用“追加写”的形式。备份系统中会保存最近的`binlog`，同时系统会定期做整库备份。这里的“定期”取决于系统的重要性，可以是一天一备，也可以是一周一备。

当需要恢复到指定的某一秒时，比如某天下午两点发现中午十二点有一次误删表，需要找回数据，那你可以这么做：

* 首先，找到最近的一次全量备份，如果运气好，可能就是昨天晚上的一个备份，从这个备份恢复到**临时库**
* 然后，从备份的时间点开始，将备份的`binlog`依次取出来，重放到中午误删表之前的那个时刻

这样临时库就跟误删之前的线上库一样了，然后就可以把表数据从临时库取出来，按需要恢复到线上库去。

如果日志不采用两阶段提交，由于`redo log`和`binlog`是两个独立的逻辑，一个是Server层，一个是存储引擎层。考虑先写`redo log`再写`binlog`，或者相反的顺序，假设当前有一条更新语句，将`ID=1`的行的字段`c`加一（0-&gt;1），如果写完第一个日志后，第二个日志还没写完就发生了crash，这两种顺序的写法会产生什么后果呢？

* **先写redo log后写binlog**。假设在`redo log`写完，`binlog`还没有写完的时候，MySQL进程异常重启。由于`redo log`提供了crash-safe的能力，系统即使崩溃，仍然能够把数据恢复回来，所以恢复后这一行`c`的值是1。

  但是由于`binlog`没写完就crash了，这时候`binlog`里面就没有记录这个语句。因此，之后备份日志的时候，存起来的`binlog`里面就没有这条语句。

  然后你会发现，如果需要用这个`binlog`来恢复临时库的话，由于备份恢复只与`binlog`相关，而这个语句的`binlog`丢失，这个临时库就会少了这一次更新，恢复出来的这一行`c`的值就是0，与原库的值不同。

* **先写binlog后写redo log**。如果在`binlog`写完之后crash，由于`redo log`还没写，而崩溃恢复读的是`redo log`，而不是`binlog`，导致崩溃恢复后这个事务无效，所以这一行`c`的值是0。但是`binlog`里面已经记录了“把c从0改成1”这个日志。所以，在之后用`binlog`来恢复的时候就多了一个事务出来，恢复出来的这一行`c`的值就是1，与原库的值不同。

因此，如果不使用“两阶段提交”，那么数据库的状态就有可能和用它的日志恢复出来的库的状态不一致。

### 异常重启的现象

崩溃恢复时的判断规则：

1. 如果redo log里面的事务是完整的，也就是已经有了prepare和commit**标识**，则直接提交；
2. 如果redo log里面的事务只有完整的prepare，则判断对应的事务binlog是否存在并完整：

   a. 如果是，则提交事务。因为已经存在binlog里了，会被从库使用，所以必须提交以保证主备数据一致性。

   b. 否则，回滚事务。

> **redo log与binlog如何建立关联？**
>
> 它们有一个共同的数据字段，叫XID。崩溃恢复的时候，会按顺序扫描redo log，如果碰到只有prepare、而没有commit的redo log，就拿着XID去binlog找对应的事务。

![ &#x4E24;&#x9636;&#x6BB5;&#x63D0;&#x4EA4;&#x793A;&#x610F;&#x56FE;](https://github.com/wsfy15/gitbook/tree/0588f684e4ae072323ae076dad8931aee02e2f7e/.gitbook/assets/ee9af616e05e4b853eba27048351f62a.jpg)

* 在时刻A，即写入redo log 处于prepare阶段之后、写binlog之前，发生了崩溃\(crash\)，由于此时binlog还没写，redo log也还没提交，所以崩溃恢复的时候，这个事务会回滚。这时候，binlog还没写，所以也不会传到备库。
* 在时刻B，即binlog写完，redo log还没commit前发生crash。根据恢复规则的2\(a\)，该事务会被提交。

## 备份周期的选择

一天一备与一周一备的对比：

* 在一天一备的模式里，最坏情况下只需要应用一天的`binlog`。
* 一周一备最坏情况就要应用一周的`binlog`了。

对应的系统指标是RTO（恢复目标时间）。

不过，更频繁全量备份需要消耗更多存储空间，所以这个RTO是成本换来的，需要根据业务重要性来评估了。

## 日志完整性判断

一个事务的binlog是有完整格式的：

* statement格式的binlog，最后会有COMMIT；
* row格式的binlog，最后会有一个XID event。

在MySQL 5.6.2版本以后，还引入了`binlog-checksum`参数，用来验证binlog内容的正确性。对于binlog日志由于磁盘原因，可能会在日志中间出错的情况，MySQL可以通过校验checksum的结果来发现。所以，MySQL还是有办法验证事务binlog的完整性的。

## 数据更新到磁盘

正常运行中的实例，数据写入后的最终落盘，是从redo log更新过来的还是从buffer pool更新过来的呢？

1. 如果是正常运行的实例的话，数据页被修改以后，跟磁盘的数据页不一致，称为脏页。最终数据落盘，就是把内存中的数据页写盘。这个过程，甚至与redo log毫无关系。
2. 在崩溃恢复场景中，InnoDB如果判断到一个数据页可能在崩溃恢复的时候丢失了更新，就会将它读到内存，然后让redo log更新内存内容。更新完成后，内存页变成脏页，就回到了第一种情况的状态。

## 日志更新到磁盘

通常说的MySQL“双1”配置，指的就是`sync_binlog`和`innodb_flush_log_at_trx_commit`都设置成 1。也就是说，一个事务完整提交前，需要等待两次刷盘，一次是redo log（prepare 阶段），一次是binlog。

那么，如果从MySQL看到的TPS是每秒两万的话，每秒就会写四万次磁盘。但是磁盘能力也就两万左右，怎么能实现两万的TPS？

### 日志逻辑序列号

log sequence number，LSN是单调递增的，用来对应redo log的一个个写入点。每次写入长度为length的redo log， LSN的值就会加上length。

LSN也会写到InnoDB的数据页中，来确保数据页不会被多次执行重复的redo log。

### 组提交

下图是三个并发事务`(trx1, trx2, trx3)`在prepare 阶段，都写完redo log buffer，持久化到磁盘的过程，对应的LSN分别是50、120 和160。

![redo log &#x7EC4;&#x63D0;&#x4EA4;](../../.gitbook/assets/933fdc052c6339de2aa3bf3f65b188cc.png)

1. trx1是第一个到达的，会被选为这组的 leader；
2. 等trx1要开始写盘的时候，这个组里面已经有了三个事务，这时候LSN也变成了160；
3. trx1去写盘的时候，带的就是`LSN=160`，因此等trx1返回时，所有LSN小于等于160的redo log，都已经被持久化到磁盘；
4. 这时候trx2和trx3就可以直接返回了。

一次组提交里面，组员越多，节约磁盘IOPS的效果越好。但如果只有单线程压测，那就只能老老实实地一个事务对应一次持久化操作了。

在并发更新场景下，第一个事务写完redo log buffer以后，接下来这个`fsync`越晚调用，组员可能越多，节约IOPS的效果就越好。

为了让一次`fsync`带的组员更多，MySQL有一个很有趣的优化：拖时间。

![&#x4E24;&#x9636;&#x6BB5;&#x63D0;&#x4EA4;](../../.gitbook/assets/1587711819317.png)

两阶段提交中，”写binlog“其实是分成两步的：

1. 先把binlog从binlog cache中`write`到磁盘上的binlog文件；
2. 调用fsync持久化。

MySQL为了让组提交的效果更好，把redo log做`fsync`的时间拖到了步骤1之后：

![&#x4E24;&#x9636;&#x6BB5;&#x63D0;&#x4EA4;&#x7EC6;&#x5316;](../../.gitbook/assets/1587711878621.png)

这么一来，binlog也可以组提交了。在执行第4步把binlog `fsync`到磁盘时，如果有多个事务的binlog已经写完了，也是一起持久化的，这样也可以减少IOPS的消耗。

通常情况下第3步执行得会很快，所以binlog的`write`和`fsync`间的间隔时间短，导致能集合到一起持久化的binlog比较少，因此binlog的组提交的效果通常不如redo log的效果那么好。

如果想提升binlog组提交的效果，可以通过设置 `binlog_group_commit_sync_delay`和 `binlog_group_commit_sync_no_delay_count`来实现。

1. `binlog_group_commit_sync_delay`：延迟多少微秒后才调用`fsync`
2. `binlog_group_commit_sync_no_delay_count`：累积多少次以后才调用`fsync`

这两个条件是或的关系，只要有一个满足条件就会调用`fsync`。所以当`binlog_group_commit_sync_delay`设置为0的时候，`binlog_group_commit_sync_no_delay_count`也无效了。

> WAL机制主要得益于两个方面：
>
> 1. redo log 和 binlog都是**顺序写**，磁盘的顺序写比随机写速度要快；
> 2. 组提交机制，可以大幅度降低磁盘的IOPS消耗。

**如果MySQL出现了性能瓶颈，而且瓶颈在IO上，可以考虑下面的方法：**

* 设置 `binlog_group_commit_sync_delay`和 `binlog_group_commit_sync_no_delay_count`参数，减少binlog的写盘次数。这个方法是基于“额外的故意等待”来实现的，因此可能会增加语句的响应时间，但没有丢失数据的风险。
* 将`sync_binlog`设置为大于1的值（比较常见是100~1000）。这样做的风险是，主机掉电时会丢binlog日志。
* 将`innodb_flush_log_at_trx_commit`设置为2。redo log写到文件系统的page cache的速度也是很快的，将这个参数设置成2跟设置成0其实性能差不多。这样做的风险是，主机掉电的时候会丢数据。

### 常见问题

Q：执行一个`update`语句以后，再去执行`hexdump`命令直接查看ibd文件内容，为什么没有看到数据有改变呢？

A：可能是因为WAL机制的原因。`update`语句执行完成后，InnoDB只保证写完了redo log、内存，可能还没来得及将数据写到磁盘。

