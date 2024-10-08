---
title: MySQL索引
date: 2020-08-09
categories:
 -  MySQL
---

## B+Tree

MySQL的基本存储结构是页，记录都存在页里面，下图以聚簇索引为例，**页与页之间构成一个双向链表**，**每个页中的记录又组成一个单向链表**，**页里边将记录分组，将每组第一个记录的主键提取出来构成一个目录项，目录项是一个数组，叶子结点记录了实际的记录，而非叶子结点并不记录实际记录，只是记录了其孩子结点第一个记录的主键以及所在页号。**

若想在B+Tree中查找一个记录，需从根结点出发，在目录项中用二分查找找到对应的记录所在组，如果当前是叶子结点，在组内遍历链表查找记录，如果是非叶子结点，继续往下找。

![B+Tree](https://gitee.com/Krains/FigureBed/raw/master/img/B+Tree.jpg)

对于范围查询来说，使用B+树查询也十分高效。B+树的叶子结点构成了一个双向链表，如果要查>=1的数据就直接先定位到1这个记录，然后遍历链表将后面所有记录取出。

各种树的比较

二叉查找树（BST）：解决了排序的问题，但是由于无法保证树的平衡，可能退化成链表

平衡二叉树（AVL）：左右子树的高度差不超过1，通过旋转解决平衡问题，插入一个数据时需要一次旋转，但是删除一个结点旋转的时间复杂度为O(logn)，旋转太耗时

红黑树：与AVL相比，并不追求严格的平衡，而是大致的平衡：确保从根到叶子的最长的可能路径不多于最短的可能路径的两倍长。因此其插入删除需要O(1)的旋转就能保证基本的平衡。但是红黑树太高，如果在磁盘中维护一个红黑树，在红黑树查找某一个值时，则需要很多磁盘IO次数，会严重影响性能。

B-树：是为磁盘等辅存设备设计的多路平衡查找树，与二叉树相比，B树的每个非叶结点可以有多个子树，因此当总结点数量相同时，B树的高度远远小于AVL树，因此磁盘的IO次数大大减少。B树的结点可以存放多个数据，因此可以充分利用局部性原理，所谓局部性原理，是指当一个数据被访问使用时，其附近的数据很大概率在短时间内被使用，读取数据时将整个结点读入缓存，缓存的命中率更高，访问效率越好。

B+树

与B树相比，B树的结点都存储数据，但是B+树的非叶子结点只存储键，叶子结点才存储完整的记录，这就导致B+树更加矮，因为其非叶子结点能够存储更多的键，那么其出度就越多，所能连接的儿子结点越多。此外其叶子结点也用双向链表连接起来，更加适合做范围查询。

总结

- 二叉查找树：解决了排序的基本问题，但是无法保证平衡，可能退化成链表，查找效率低
- 平衡二叉树：通过旋转解决了平衡的问题，但是旋转操作效率太低
- 红黑树：舍弃了严格平衡，解决了AVL旋转效率过低的问题，但是在磁盘等场景下，树仍然太高，IO次数太多
- B树：将二叉树改为多路平衡查找树，解决了树过高的问题
- B+树：在B树的基础上，将非叶子结点改造成不存储数据的纯索引结点，进一步降低了树的高度（使得非叶子结点能够存储更多的键），此外，将叶子结点使用指针连接成链表，范围查询更加高效

对比Hash索引的优势

- Hash索引不能够进行排序
- 只支持精确查找，无法用于范围查找

BTree+树高度计算

假设主键数据类型是INT，占用4bytes，一行数据总共占用是1KB，指针6bytes，一个页大小是16kb

高度为2时：对于非叶子结点来说，没有存数据，只存了主键，一个页能存16384/（4+6）个行数据，也就是能指向1638个页，对于叶子结点来说，每个页能够存储16/1=16行数据，那么16*1638=26208行数据。

高度为3时：多了一个目录的目录，所以1638 \*1638 \* 16= 42928704 行数据

## 索引

聚簇索引

以主键构建的索引，其叶子结点存储了真实的记录，InnoDb默认为主键建立了索引。

建立普通索引

```mssql
create index idx_t1_bcd on t1(b, c, d)  --使用t1表中的b，c，d列创建名为idx_t1_bcd的索引
```

![联合索引](https://gitee.com/Krains/FigureBed/raw/master/img/%E8%81%94%E5%90%88%E7%B4%A2%E5%BC%95.png)

bcd如何排序？

跟字符串排序一样，先比较a，不同则可以区分大小，相同在比较b，然后c。当然对于不同的字符集有不同的比较规则，MySql中collation就定义了每个字符集的比较规则。

普通索引叶子结点不存完整的数据，只存索引项和主键，查找数据的时候先通过普通索引找到对应的主键，在用这个主键去主键索引去找，这个操作叫回表。

如果bcd有重复如何？

将主键也存起来，以区分不同的bcd。

如何查看sql语句是否使用了索引？

```mysql
explain select * from t1 where b = 1 and c = 1 and d = 1;  -- 查看索引的使用情况
```

![索引字段](https://gitee.com/Krains/FigureBed/raw/master/img/%E7%B4%A2%E5%BC%95%E5%AD%97%E6%AE%B5.png)

type：如果为'all'时查询数据时用的是全盘扫描的方法

possible_keys: 可能使用的索引，对于要查询的数据来说，使用的查询条件可能可以使用多个索引，这里列出了所有可能可以使用的索引。

key：最终选择的索引，查询优化器选择它认为最优的索引，用它选择的索引做查询。

在建立了辅助索引的情况下，什么时候查询优化器可能会选择全盘扫描而不是使用辅助索引？

如果使用辅助索引找到的主键很多时（全表主键的80%-90%？），这个时候如果使用辅助索引效率会比较低，查询优化器会选择用全表扫描的方法查询。

### 辅助索引的最左匹配原则

全值匹配

```mysql
select * from t1 where b = 1 and c = 1 and d = 1;
```

最左匹配

```mysql
select * from t1 where b = 1;
select * from t1 where b = 1 and c = 1;
```

能够使用索引，因为对于给出b=1，可以使用它'1**'去比较'235', '322'的大小。

```mysql
select * from t1 where c = 1;
```

不能够使用索引，对于c=1，"\*1\*"无法与"215"，"312"比较大小，从而使用索引的时候无法排除掉一些不在其范围的值。B+树先是按照b列的值排序的，在b列的值相同的情况下才使用c列进行排序，也就是说b列的值不同的记录中c的值可能是无序的。而现在跳过b列直接根据c的值去查找，这是做不到的。

```mysql
select * from t1 where b like '%101%';
```

这样也是用不到索引的，前缀没有确定，无法比较索引项与条件的大小关系。

```mysql
select * from t1 where b > 1 and b < 8;
```

能够使用到索引，对于这种范围查询来说，上边的查询过程其实是这样的：

- 先找到b值为1的记录
- 找到b值为8的记录
- 由于所有的记录都是由链表连起来的（记录之间用单链表，数据页之间用双链表），只需要遍历链表就能够取出记录
- 找到这些记录的主键值，再到聚簇索引中回表查找完整的记录

在联合索引中使用范围查询的时候时，如果对多个列同时进行范围查找的话，只有对索引最左边的那个列进行范围查询的时候才能用到B+树索引，比如；

```mysql
select * from t1 where b > 3 and c > 1;
```

这个查询分为两个部分

- 通过条件b > 3 使用联合索引查找多条记录，该条件使用了索引（不一定，当查询记录过多时，查询优化器会使用全盘扫描的方式）
- 对这些记录使用c > 1进行过滤，该条件没有使用索引
- 最后将记录的主键取出去聚簇索引查找数据

但是这里还有前面说到的查询优化器的一个优化，比如

```mysql
select * from t1 where b > 1 and c > 1;
```

如果通过条件 b > 1 找出的记录过多的话，查询优化器会选择全盘扫描而不是使用索引。

```mysql
select * from t1 where b = 1 and c > 1;
```

两个索引列b、c都用到了，固定b时c也是排好序的。

```mysql
select * from t1 order by b, c, d;
```

因为索引本身就是对b、c、d进行排序的，所以能够使用索引提取主键，然后回表取数据。

创建索引时的技巧

- 根据最左匹配原则，建立索引的时候尽量将使用查询次数最多的项放到最前面。
- 索引列的类型尽量小，占用空间少，一个就可以多放几条记录，甚至可以降低B+Tree的高度，使得查找的效率变高
- 索引的字段要尽量分布比较均匀，如果像性别只有两个值0和1，那么就没必要做索引

参考链接

[1]. [MySQL的索引结构为什么使用B+树，而不是其他树形结构？](https://www.bilibili.com/read/cv5985933/)