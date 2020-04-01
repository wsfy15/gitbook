# 短网址系统

以[新浪提供的短网址系统](https://www.sina.lt/)为例，它可以把原始的长网址转化成短网址，当用户点击短网址的时候，短网址服务会将浏览器重定向为原始网址。

![1584152646956](../../.gitbook/assets/1584152646956.png)

## 通过哈希算法生成短网址

哈希算法可以将一个不管多长的字符串，转化成一个长度固定的哈希值。

在生成短网址这个问题上，不需要考虑反向解密的难度，只需要关心哈希算法的计算速度和冲突概率。能够满足这样要求的哈希算法有很多，MurmurHash算法就是其中一种应用比较广泛的。

MurmurHash算法提供了两种长度的哈希值，一种是32bits，一种是128bits。为了让最终生成的短网址尽可能短，我们可以选择32bits的哈希值。对于开头那个GitHub网址，经过MurmurHash计算后，得到的哈希值就是181338494。我们再拼上短网址服务的域名，就变成了最终的短网址[http://t.cn/181338494（其中，http://t.cn](http://t.cn/181338494（其中，http://t.cn) 是短网址服务的域名）。

### 让短网址更短

将10进制的哈希值，转化成更高进制的哈希值，这样哈希值就变短了。

16进制中用A～E，来表示10～15。在网址URL中，常用的合法字符有0～9、a～z、A～Z这样62个字符。为了让哈希值表示起来尽可能短，我们可以将10进制的哈希值转化成62进制。上述网址最终用62进制表示的短网址就是[http://t.cn/cgSqq。](http://t.cn/cgSqq。)

![1584152947908](../../.gitbook/assets/1584152947908.png)

### 解决哈希冲突

尽管MurmurHash算法，冲突的概率非常低。但是，**一旦冲突，就会导致两个原始网址被转化成同一个短网址**。当用户访问短网址的时候，我们就无从判断，用户想要访问的是哪一个原始网址了。

一般情况下，我们会在数据库（例如MySQL）中保存短网址跟原始网址之间的对应关系，以便后续用户在访问短网址的时候，可以根据对应关系，查找到原始网址。

当有一个新的原始网址需要生成短网址的时候，先利用MurmurHash算法，生成短网址。然后，拿这个新生成的短网址，在MySQL数据库中查找。

* 如果没有找到相同的短网址，这也就表明，这个新生成的短网址没有冲突。于是我们就将这个短网址返回给用户（请求生成短网址的用户），然后将这个短网址与原始网址之间的对应关系，存储到MySQL数据库中。
* 如果我们在数据库中，找到了相同的短网址，那也并不一定说明就冲突了。我们从数据库中，将这个短网址对应的原始网址也取出来。
  * 如果原始网址跟正在处理的原始网址是一样的，这就说明已经有人请求过这个原始网址的短网址了。我们就可以直接用这个短网址。
  * 如果原始网址跟正在处理的原始网址不一样，那就说明哈希算法发生了冲突。不同的原始网址，经过计算，得到的短网址重复了。

    我们可以给正在处理的原始网址拼接一串特殊字符，比如“\[DUPLICATED\]”，然后再重新计算哈希值，两次哈希计算都冲突的概率，显然是非常低的。就算又发生冲突了，我们可以再换一个拼接字符串，如“\[OHMYGOD\]”，再计算哈希值。然后把计算得到的哈希值，跟原始网址拼接了特殊字符串之后的文本，一并存储在MySQL数据库中。 当用户访问短网址的时候，短网址服务先通过短网址，在数据库中查找到对应的原始网址。如果原始网址有拼接特殊字符（这个很容易通过字符串匹配算法找到），我们就先将特殊字符去掉，然后再将不包含特殊字符的原始网址返回给浏览器。

### 优化生成短网址的性能

因为要判断生成的短网址是否冲突，我们需要拿生成的短网址，在数据库中查找。如果数据库中存储的数据非常多，那查找起来就会非常慢，势必影响短网址服务的性能。

* 可以**给短网址字段添加B+树索引**。这样通过短网址查询原始网址的速度就提高了很多。

在短网址生成的过程中，我们会跟数据库打两次交道，也就是会执行两条SQL语句。第一个SQL语句是查重，第二个SQL语句是存储新生成的短网址和原始网址。

> 一般情况下，数据库和应用服务（只做计算不存储数据的业务逻辑部分）会部署在两个独立的服务器或者虚拟服务器上。那两条SQL语句的执行就需要两次网络通信。这种IO通信耗时以及SQL语句的执行，才是整个短网址服务的性能瓶颈所在。所以，为了提高性能，我们需要尽量减少SQL语句。

* 可以**给数据库中的短网址字段，添加一个唯一索引**（不止是索引，还要求表中不能有重复的数据）。当有新的原始网址需要生成短网址的时候，我们不用先拿生成的短网址，在数据库中查找判重，而是直接将生成的短网址与对应的原始网址，尝试存储到数据库中。如果数据库能够将数据正常写入，那说明并没有违反唯一索引，也就是说，这个新生成的短网址并没有冲突。

  当然，如果数据库反馈违反唯一性索引异常，那就得重新执行刚刚讲过的“查询、写入”过程，SQL语句执行的次数不减反增。但是，在大部分情况下，我们把新生成的短网址和对应的原始网址，插入到数据库的时候，并不会出现冲突。所以，大部分情况下，我们只需要执行一条写入的SQL语句就可以了。所以，从整体上看，总的SQL语句执行次数会大大减少。

* **借助布隆过滤器查重**。把已经生成的短网址，构建成布隆过滤器。

  当有新的短网址生成的时候，我们先拿这个新生成的短网址，在布隆过滤器中查找。如果查找的结果是不存在，那就说明这个新生成的短网址并没有冲突。这个时候，我们只需要再执行写入短网址和对应原始网页的SQL语句就可以了。通过先查询布隆过滤器，总的SQL语句的执行次数减少了。

## 通过自增ID生成短网址

维护一个ID自增生成器，它可以生成1、2、3…这样自增的整数ID。当短网址服务接收到一个原始网址转化成短网址的请求之后，它先从ID生成器中取一个号码，然后将其转化成62进制表示法，拼接到短网址服务的域名后面，就形成了最终的短网址。最后，还是要把生成的短网址和对应的原始网址存储到数据库中。

每次新来一个原始网址，我们就生成一个新的短网址，这种做法就会导致**两个相同的原始网址生成了不同的短网址**。这种情况既可以处理，也可以不处理。

不做处理是因为，即使相同的原始网址对应不同的短网址，对用户影响不大。因为在大部分短网址的应用场景里，用户只关心短网址能否正确地跳转到原始网址。至于短网址长什么样子，他其实根本就不关心。

处理的话，可以借助哈希算法生成短网址的处理思想，当要给一个原始网址生成短网址的时候，先拿原始网址在数据库中查找，看数据库中是否已经存在相同的原始网址了。如果数据库中存在，那我们就取出对应的短网址，直接返回给用户。

不过，这种处理思路，我们需要给数据库中的短网址和原始网址这两个字段，都添加索引。短网址上加索引是为了提高用户查询短网址对应的原始网页的速度，原始网址上加索引是为了加快通过原始网址查询短网址的速度。这种解决思路虽然能满足“相同原始网址对应相同短网址”这样一个需求，但是是有代价的：一方面两个索引会占用更多的存储空间，另一方面索引还会导致插入、删除等操作性能的下降。

### 性能优化

实现ID生成器的方法有很多，比如利用数据库自增字段。当然我们也可以自己维护一个计数器，不停地加一加一。但是，一个计数器来应对频繁的短网址生成请求，显然是有点吃力的（因为计数器必须保证生成的ID不重复，笼统概念上讲，就是需要加锁）。

* 类似高性能队列Disruptor，**给ID生成器装多个前置发号器**。批量地给每个前置发号器发送ID号码。当接受到短网址生成请求的时候，就选择一个前置发号器来取号码。这样通过多个前置发号器，明显提高了并发发号的能力。

  ![1584157102786](../../.gitbook/assets/1584157102786.png)

* **直接实现多个ID生成器同时服务**。为了保证每个ID生成器生成的ID不重复，要求每个ID生成器按照一定的规则，来生成ID号码。比如，第一个ID生成器只能生成尾号为0的，第二个只能生成尾号为1的，以此类推。这样通过多个ID生成器同时工作，也提高了ID生成的效率。

  ![1584157116410](../../.gitbook/assets/1584157116410.png)
