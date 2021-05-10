## Linux 的 3 种“拷贝”命令

[原文链接](https://mp.weixin.qq.com/s/e-G7o8MtTup8ievdzY501Q)

1. **ln 创建链接文件，软链接可以跨文件系统，硬链接跨文件系统会报错，为什么？；**
2. **mv 好像有时候快，有时候非常慢，有些时候还会残留垃圾，为什么？；**
3. **cp 拷贝数据有时快，有时候非常慢，源文件和目标文件所占物理空间竟然不一致？**



## 文件和目录

1. 文件系统内有 3 个关键区域：超级块区域，inode 区域，数据 block 区域
2. 其中一个 inode 和一个文件对应，包含了文件的元数据信息
3. 一个 inode 有唯一的编号，可以理解成就是单调递增的整数。比如 1，2，3，4，5，6……

文件有两种类型：普通文件和目录文件，可以通过 `inode->i_mode` 字段，使用 `S_ISREG`，`S_ISDIR` 这两个宏来判断是哪个类型。

对于普通文件，inode 里面存储元数据，inode 可以索引到 block，block 里面存储用户的数据。

目录文件 inode 存储元数据，block 里面存储的是**目录条目`dirent`**。

> 假设在当前 `testdir `目录下，有 dir1，dir2，dir3 这三个文件。假设 dir1 的 inode 编号是 1024，dir2 是 1025，dir3 是 1026。
>
> 1. testdir 这个目录首先会对应有一个 inode，`inode->i_mode` 的类型是目录，并且还会有 block 块，通过 `inode->i_blocks` 能索引到这些 block
> 2. block 里面存储的内容很简单，是一个个目录条目 `dirent`，每一个 `dirent` 本质就是一个 **文件名字** 到 **inode 编号**的映射，所以，testdir 这个目录文件的 block 里存了 3 条记录 [dir1, 1024]，[dir2, 1025]，[dir3, 1026]



Linux的文件系统是一个树形结构，这棵树的每个节点是一个`dentry`，每个`dentry`绑定到唯一一个 inode 结构体，并且有父、子、兄弟的索引路径：

```
struct dentry {
   // ...
   struct dentry  *d_parent;   /* 父节点 */
   struct qstr    d_name;      // 名字
   struct inode   *d_inode;    // inode 结构体

   struct list_head d_child;     /* 兄弟节点 */
   struct list_head d_subdirs;   /* 子节点 */ 
};
```

文件树的结构在内存中以 `dentry` 结构体体现，对于每个目录文件，它的每个表项则称为`dirent` 。



## ln

`ln [OPTION]... TARGET LINK_NAME`，创建链接文件，链接文件有两种：

- 软链接文件：带上`-s` 参数，会新建一个全新的文件，有独立的inode和block，**存储源文件的路径**，所以它可以跨文件系统创建。

  在 coreutils 库里，调用栈如下：

  ```
  main -> do_link -> force_symlinkat -> symlinkat
  ```

  最终通过`symlinkat`这个系统调用完成创建，这个系统调用的实现与具体文件系统相关。如果是 minix 文件系统，就会先创建一个inode，然后在`LINK_NAME`所在的目录文件中添加一个`dirent`，将该inode加进去。

- 硬链接文件：没有创建新的inode和block，也就没有创建新文件，而是直接在`LINK_NAME`所在的目录文件添加一个`dirent`，这个 `dirent` 的名字是`LINK_NAME`，对应的inode号是`TARGET`的inode号。

  由于新旧两个 dirent 都是指向同一个 inode，就导致了一个限制：**不能跨文件系统。因为，不同文件系统的 inode 管理都是独立的。**

  在 coreutils 库里，调用栈如下：

  ```
  main -> do_link -> force_linkat -> linkat
  ```

  最终通过`linkat`这个系统调用完成创建，这个系统调用的实现与具体文件系统相关。如果是 minix 文件系统，就会把一个`dentry`和一个`inode`进行关联，也就是往`dentry`加一个`dirent`项。



## mv

### 源文件和目的文件 在同一个文件系统

只需要`rename`系统调用，只会涉及到元数据的操作，从 源文件所在目录的目录文件 删掉一个`dirent`，在 目的文件所在目录的目录文件 新增一个`dirent`。文件的inode不会发生变化，不会有数据拷贝。调用栈如下：

```
main -> renameat2
```

```
~$ stat a.txt
  File: a.txt
  Size: 0               Blocks: 0          IO Block: 4096   regular empty file
Device: 830h/2096d      Inode: 169968      Links: 1
Access: (0644/-rw-r--r--)  Uid: ( 1000/      sf)   Gid: ( 1000/      sf)
Access: 2021-05-08 12:35:11.491847800 +0800
Modify: 2021-05-08 12:35:11.491847800 +0800
Change: 2021-05-08 12:35:11.491847800 +0800
 Birth: -
~$ mv a.txt b.txt 
~$ stat b.txt
  File: b.txt
  Size: 0               Blocks: 0          IO Block: 4096   regular empty file
Device: 830h/2096d      Inode: 169968  （！！！没有发生变化）    Links: 1
Access: (0644/-rw-r--r--)  Uid: ( 1000/      sf)   Gid: ( 1000/      sf)
Access: 2021-05-08 12:35:11.491847800 +0800
Modify: 2021-05-08 12:35:11.491847800 +0800
Change: 2021-05-08 12:35:47.111847800 +0800
 Birth: -
```



### 源文件和目的文件 在不同的文件系统

当系统调用 `rename` 的时候，如果**源**和**目的**不在同一文件系统时，会报告 `EXDEV` 的错误码，提示该调用不能跨文件系统。因为`rename`不能用于跨文件系统。

```
#define EXDEV           18      /* Cross-device link */
```

现在需要先copy，再remove。调用栈如下：

```
main -> movefile -> do_move -> copy -> copy_internal -> renameat2
main -> remove
```

**mv 跨文件系统的时候，如果第一步成功了，第二步失败了（比如没有删除权限）会怎么样？**

会导致垃圾。也就是说，目标处创建了一个新文件，源文件并没有删除。



## cp

稀疏文件：**后分配空间**，用时才分配，最大效率利用磁盘空间。例如创建了一个 1GB 的文件，但实际上只使用了8KB 的空间，那么就只会分配这8KB的空间（1个inode 4KB的话，就只分配2个inode）。

```
# 创建一个稀疏文件
$ truncate -s 100G  test.txt
$ ls -lh ./test.txt
-rw-r--r-- 1 sf sf 100G May  8 12:46 ./test.txt
$ du -sh ./test.txt
0       ./test.txt
$ stat ./test.txt
  File: ./test.txt
  Size: 107374182400    Blocks: 0          IO Block: 4096   regular file
Device: 830h/2096d      Inode: 169980      Links: 1
# Size 是逻辑大小，实际物理空间占用还得看Blocks
```

由于稀疏文件只有部分block存了用户数据，其他的block就相当于空洞，在拷贝的时候可以针对这些空洞进行优化。`cp`有三种模式：

- `auto`：拷贝的时候会跳过空洞
- `always`：除了跳过空洞，还会跳过全0数据的block（即使这些全0数据是真实的数据）
- `never`：不管是空洞还是全 0 数据，都会在目标文件写入







