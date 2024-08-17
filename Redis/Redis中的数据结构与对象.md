---
title: Redis中的数据结构与对象
date: 2021-07-11
categories:
 -  分布式系统
---

## 简单动态字符串（simple dynamic string，SDS）

Redis没有直接使用C语言传统的字符串表示（以空字符串结尾的字符数组，以下简称C字符串），而是自己构建了一种名为简单动态字符串的抽象类型，并将SDS用作Redis的默认字符串表示。

SDS的定义

```c
struct sdshdr{
    // 记录buf数组中已使用字节的数量
    // 等于SDS所保存字符串的长度
    int len;
    
    // 记录buf数组中未使用字节的数量
    int free;
    
    // 字节数组，用于保存字符串
    char buf[];
}
```

SDS相较于C字符串的好处

一、常数复杂度获取字符串长度

因为C字符串并不记录自身的长度信息，所以为了获取一个C字符串的长度，必须遍历整个字符串，直到遇到代表字符串结尾的空字符'\0'为止，这个操作的时间复杂度是O(N)。

而SDS在len属性中记录了SDS本身的长度，所以获取一个SDS长度的复杂度仅为O(1)。

二、减少修改字符串时带来的内存重分配次数

因为C字符串并不记录自身的长度，所以对于一个包含了N个字符的C字符串来说，这个C字符串的底层实现总是一个N+1个字符长的数组（额外一个字符空间用于保存空字符），C语言以末尾'\0'来隐式表示数组长度的做法导致每次增长或者缩短一个C字符串，程序都总要对保存这个C字符串的数组进行一次内存重分配的操作：

- 如果要增长字符串，程序需要先通过内存重分配来扩展底层数组的空间大小，如果没有这步就会导致缓冲区溢出
- 如果要缩短字符串操作，程序需要通过内存重分配来释放字符串不再使用的那部分空间，如果没有则会产生内存泄露

内存重分配会涉及到系统调用，它是一个比较耗时的操作。

Redis作为数据库，经常被用于速度要求严苛、数据被频繁修改的场合，为了避免C字符串的这种缺陷，SDS通过未使用空间解除了字符串长度和底层数组长度之间的关联：在SDS中，buf数组可以包含未使用的字节，这个字节数由free记录。通过未使用空间，SDS实现了空间预分配和惰性空间释放两种优化策略。

空间预分配

空间预分配用于优化SDS的字符串增长操作：当SDS的API对一个SDS进行修改，并且需要对SDS进行空间扩展的时候，程序不仅会对SDS分配修改所必要的空间，还会为SDS分配额外的未使用的空间，其空间预分配的策略如下：

- 如果对SDS进行修改之后，SDS的长度小于1MB，那么程序分配和len属性同样大小的未使用空间，即free=len
- 如果对SDS进行修改之后，SDS的长度将大于等于1MB，那么程序只会分配1MB的未使用空间

在扩展SDS空间之前，SDS API会先检查未使用空间是否足够，如果足够，API就会直接使用未使用空间，而无需执行内存重分配。通过这种预分配策略，SDS将连续增长N次字符串所需的内存重分配次数从必定N次降低为最多N次。

惰性空间释放

惰性空间释放用于优化SDS的字符串缩短操作：当SDS的API需要缩短SDS保存的字符串时，程序并不立即使用内存重分配来回收缩短后多出来的字节，而是使用free属性将这些字节的数量记录起来，并等待将来使用。

与此同时，SDS也提供了相应的API，让我们可以在有需要的时候，真正地释放SDS的未使用空间，所以不用担心惰性空间释放策略会操作内存浪费。

三、二进制安全

C字符串中的字符必须符合某种编码（比如ASCII），并且除了字符串的末尾之外，字符串里面不能包含空字符，否则最先被程序读入的空字符将被误认为是字符串结尾，这些现在使得C字符串只能保存文本数据，而不能保存像图片、音频、视频、压缩文件这样的二进制数据，因为这些数据中可能会出现空字符。

SDS不仅可以保存文本数据，还可以保存任意格式的二进制数据，因为SDS使用len属性的值而不是空字符来判断字符串是否结束。

四、兼容部分 C 字符串函数

虽然 SDS 的 API 都是二进制安全的，但它们一样遵循 C 字符串以空字符结尾的惯例：这些 API 总会将 SDS 保存的数据的末尾设置为空字符，并且总会在为 buf 数组分配空间时多分配一个字节来容纳这个空字符，这是为了让那些保存文本数据的 SDS 可以重用一部分<string.h> 库定义的函数。

总结

C字符串和SDS之间的区别总结

| C字符串                                    | SDS                                        |
| ------------------------------------------ | ------------------------------------------ |
| 获取字符串长度的复杂度为O(N)               | 获取字符串长度的复杂度为O(1)               |
| API是不安全的，可能会造成缓冲区溢出        | API是安全的，不会造成缓冲区溢出            |
| 修改字符串长度N次必然需要执行N次内存重分配 | 修改字符串长度N次最多需要执行N次内存重分配 |
| 只能保存文本数据                           | 可以保存文本或者二进制数据                 |
| 可以使用所有<string.h>库中的函数           | 可以使用一部分<string.h>库中的函数         |

## 链表

列表键的底层实现就是一个链表。

每个链表结点使用一个adlist.h/listNode结构来表示：

```c
typedef struct listNode{
    // 前置结点
    struct listNode *prev;
    
    // 后置结点
    struct listNode *next;
    
    // 结点的值
    void *value;
}listNode;
```

Redis中还使用adlist.h/list来持有一条链表：

```c
typedef struct list{
    // 表头结点
    listNode *head;
    
    // 表尾结点
    listNode *tail;
    
    // 链表所包含的结点数量
    unsigned long len;
    
    // 结点值复制函数
    void *(*dup) (void *ptr);
    
    // 结点值释放函数
    void (*free)(void *ptr);
    
    // 结点值对比函数
    int (*match)(void *ptr, void *key);
}list;
```

dup、free和match成员则是用于实现多态链表所需的类型特定函数

- dup函数用于复制链表结点所保存的值
- free函数用于释放链表结点锁保存的值
- match函数则用于对比链表结点所保存的值和另一个输入值是否相等

下图是由一个list结构和三个listNode结构组成的链表

<img src="https://gitee.com/Krains/FigureBed/raw/master/img/redis%E5%88%97%E8%A1%A8%E9%94%AE.png" alt="redis列表键" style="zoom:80%;" />

## 字典

Redis的数据库的底层实现是字典，对数据库的增、删、查、改操作也是构建在对字典的操作之上的。

例如，当我们执行命令

```cmd
redis> set msg "hello world"
ok
```

在数据库中创建一个键为"msg"，值为"hello world"的键值对时，这个键值对就是保存在代表数据库的字典里面的。

除了用来表示数据库之外，字典还是哈希键的底层实现之一。

### 字典的实现

redis的字典使用哈希表作为底层实现，一个哈希表里面可以有多个哈希表节点，而每个哈希表节点就保存了字典中的一个键值对。

#### 哈希表

Redis字典所使用的哈希表由dict.h/dictht结构定义

```c
typedef struct dictht{
    // 哈希表数组，数组中每个元素都是一个指向dictEntry结构的指针
    dictEntry **table;
    // 哈希表大小
    unsigned long size;
    // 哈希表大小掩码，用于计算索引值，总是等于size-1
    unsigned long sizemask;
    // 该哈希表已有节点数量
    unsigned long used;
}dictht;
```

### 哈希表节点

哈希表节点使用dictEntry结构表示，每个dictEntry结构都保存着一个键值对：

```c++
typedef struct dictEntry{
    // 键
    void *key;
    // 值
    union{
        void *val;
        uint64_tu64;
        int64_ts64;
    }v;
    // 指向下个哈希表节点，形成链表
    struct dictEntry *next;
}dictEntry;
```

### 字典

Redis中的字典由dict.h/dict结构表示：

```c
typedef struct dict{
    // 类型特定函数
    dictType *type;
    // 私有数据
    void *privdate;
    // 哈希表
    dictht ht[2];
    // rehash索引
    // 当rehash不在进行时，值为-1
    int rehashindex;
}dict;
```

type属性和privdata属性是针对不同类型的键值对，为创建多态字典而设置的：

- type属性是一个指向dictType结构的指针，每个dictType结构保存了一簇用于操作特定类型键值对的函数，Redis会为用途不同的字典设置不同类型特定函数
- 而privdata属性则保存了需要传给那些类型特定函数的可选参数

```c
typedef struct dictType{
    // 计算哈希值的函数
    unsigned int (*hashFunction)(const void *key);
    // 复制键的函数
    void *(*keyDup)(void *privadata, const void *key);
    // 复制值的函数
    void *(*valDup)(void *privdata, const void *obj);
    // 对比键的函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    // 销毁键的函数
    void (*keyDestructor)(void *privdata, void *key);
    // 销毁值的函数
    void (*valDestructor)(void *privdata, void *obj);
}dictType;
```

字典结构如下

![redis字典](https://gitee.com/Krains/FigureBed/raw/master/img/redis%E5%AD%97%E5%85%B8.png)

### 哈希算法

当要将一个新的键值对添加到字典里面时，程序需要先根据键值对的键计算出哈希值和索引值，然后再根据索引值，将包含新键值对的哈希表结点放到哈希表数组的指定索引上面

```c
hash = dict->type->hashFunction(key);
index = hash & dict->ht[x].sizemask;
```

### 解决键冲突

当有两个或以上数量的键被分配到了哈希表数组的同一个索引上面时，我们称这些键发生了冲突。

Redis使用链地址法来解决键冲突，被分配到同一个索引上的多个结点可以用单向链表连接起来。为了速度考虑，程序总是将新结点添加到链表的表头位置。

### rehash

Redis对字典的哈希表执行rehash的步骤如下：

- 为字典的ht[1]哈希表分配空间
  - 如果执行的是扩展操作，那么ht[1]的大小为第一个大于等于ht[0].used*2的2^n（2的n次方幂）
  - 如果执行的是收缩操作，那么ht[1]的大小为第一个大于等于ht[0].used的2^n
- 将保存在ht[0]中的所有键值对rehash到ht[1]上面
- 当ht[0]包含的所有键值对都迁移到了ht[1]之后（ht[0]变为空表），释放ht[0]，将ht[1]设置为ht[0]，并在ht[1]新创建一个空白哈希表，为下一次rehash做准备

哈希表扩展与收缩的时机

扩展时机

- 服务器目前没有在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于1
- 服务器目前正在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于5

收缩

- 当哈希表的负载因子小于0.1时，程序自动开始对哈希表执行收缩操作

其中哈希表的负载因子可以通过公式得出：

$$ load\_factor= ht[0].used/ht[0].size$$

根据BGSAVE命令或BGREWRITEAOF命令是否正在执行，服务器执行扩展操作所需的负载因子并不相同，这是因为在执行BGSAVE命令或BGREWRITEAOF命令过程中，Redis需要创建当前服务器进程的子进程，而大多数操作系统都采用写时复制（copy-on-write）技术来优化子进程的使用效率，所以在子进程存在期间，服务器会提高执行扩展操作所需的负载因子，从而尽可能地避免在子进程存在期间进行哈希表扩展操作，这可以避免不必要的内存写入操作，最大限度地节约内存。

**渐进式rehash**

详细步骤

- 为ht[1]分配空间，让字典同时持有ht[0]和ht[1]两个哈希表
- 在字典中维持一个索引计数器变量rehashidx，并将它的值设置为0，表示rehash工作正是开始
- 在rehash进行期间，每次对字典执行添加、删除、查找或者更新操作时，程序除了执行指定的操作以外，还会顺带将ht[0]哈希表在rehashidx索引上的所有键值对rehash到ht[1]，但rehash工作完成之后，程序将rehashidx属性的值增一
- 随着字典操作的不断执行，最终在某个时间点上，ht[0]的所有键值对都会被rehash至ht[1]，这时程序将rehashidx属性的值设为-1，表示rehash操作已完成

渐进式rehash的好处在于它采取分而治之的方式，将rehash键值对所需的计算工作均摊到对字典的每个添加、删除、查找和更新操作上，从而避免了集中式rehash而带来的庞大计算量。

在渐进式rehash的过程中，字典会同时使用ht[0]和ht[1]两个哈希表，所以在渐进式rehash进行期间，字典的删除、查找、更新等操作会在两个哈希表上进行，例如要在字典上查找一个键的话，程序会现在ht[0]里面进行查找，如果没找到的话，就会继续到ht[1]里面进行查找。此外，新添加到字典的键值对一律会被保存到ht[1]里面。

## 跳跃表

Redis使用跳跃表作为有序集合键(zset)的底层实现，大部分情况下，跳跃表的效率可以和平衡树相媲美，并且因为跳跃表的实现比平衡树要来得更为简单，所以有不少程序都使用跳跃表来代替平衡树。

跳跃表支持平均O(logN)、最坏O(N)复杂度的结点查找，还可以通过顺序性操作来批量处理结点。

跳跃表的实现

redis.h/zskiplist结构用于保存跳跃表节点的相关信息

```c
typedef struct zskiplist{
    // 表头节点和表尾节点
    struct zskiplistNode *header, *tail;
    // 表中节点的数量
    unsigned long length;
    // 表中层数最大的结点层数
    int level;
}zskiplist;
```

redis.h/zskiplistNode结构表示跳跃表节点

```c
typedef struct zskiplistNode {
    // 层
    struct zskiplistLevel {
        // 前进指针
        struct skiplistNode *forward;
        // 跨度，记录前进指针指向的结点与当前结点的距离
        unsigned int span;
    } level[];
    // 后退指针
    struct zskiplistNode *backward;
    // 分值，结点按照分值从小到大排序
    double score;
    // 成员对象
    robj *obj;
} zskiplistNode;
```

层

跳跃表节点的level数组可以包含多个元素，每个元素都包含一个指向其他节点的指针，程序可以通过这些层来加快访问其他节点的速度。每次创建一个新跳跃表节点的时候，程序都根据幂次定律（power law，越大的数出现的概率越小）随机生成一个介于1和32之间的值作为level数组的大小，这个大小就是层的“高度”。

分值和成员

节点的分值是一个double类型的浮点数，跳跃表中的所有节点都按分值从小到大来排序

节点的成员对象是一个指针，它指向一个字符串对象，而字符串对象则保存着一个SDS值。同一个跳跃表中，各个节点保存的成员对象必须是唯一的（如何保证唯一？用set？），但是多个节点保存的分值可以是相同的：分值相同的节点按照成员对象在字典序中的大小来进行排序。

跳跃表图示：

![redis跳跃表](https://gitee.com/Krains/FigureBed/raw/master/img/redis%E8%B7%B3%E8%B7%83%E8%A1%A8.png)

跳跃表API

| 函数                  | 作用                                                   | 时间复杂度            |
| --------------------- | ------------------------------------------------------ | --------------------- |
| zslInsert             | 将包含给定成员和分值的新结点添加到跳跃表               | 平均O(logN)，最坏O(N) |
| zslDelete             | 删除跳跃表中包含给定成员和分值的结点                   | 平均O(logN)，最坏O(N) |
| zslGetRank            | 返回包含给定成员和分值的结点在跳跃表中的排位           | 平均O(logN)，最坏O(N) |
| zslDeleteRangeByScore | 给定一个分值范围，删除跳跃表中所有在这个范围之内的结点 | O(N)                  |
| zslDeleteRangeByRank  | 给定一个排位范围，删除跳跃表中所有在这个范围之内的结点 | O(N)                  |

## 整数集合

整数集合（intset）是集合键（set）的底层实现之一，当一个集合只包含整数值元素，并且这个集合的元素数量不多时，Redis就会使用整数集合作为集合键的底层实现。

整数集合的实现

每个intset.h/intset结构表示一个整数集合

```c
typedef struct intset{
    // 编码方式
    uint32_t encoding;
    // 集合包含的元素数量
    uint32_t length;
    // 保存元素的数组
    int8_t contents[];
} intset;
```

contents数组是整数集合的底层实现：整数集合的每个元素都是contents数组的一个数组项（item），各个项在数组中按值的大小从小到大有序地排列，并且数组中不包含任何重复项。

示例：encoding属性的值为INTSET_ENC_INT16，表示整数集合的底层实现为int16_t类型的数组，数组中每个整数的类型都是int16_t

![Redis整数集合](https://gitee.com/Krains/FigureBed/raw/master/img/Redis%E6%95%B4%E6%95%B0%E9%9B%86%E5%90%88.png)

升级

每当我们要将一个新元素添加到整数集合里面，并且新元素的类型比整数集合现有元素的类型都要长时，整数集合需要先进行升级，然后才能将新元素添加到整数集合里面。

## 压缩列表

压缩列表是列表键和哈希键的底层实现之一，当一个列表键只包含少量列表键，并且每个列表项要么就是小整数值，要么就是长度比较短的字符串，那么Redis就会使用压缩列表来做列表键的底层实现。

另外，当一个哈希键只包含少量键值对，并且每个键值对的键和值要么就是小整数值，要么就是长度比较短的字符串，那么Redis就会使用压缩列表来做哈希键的底层实现。

## 对象

Redis使用对象来表示数据库中的键和值，每当我们在Redis的数据库中新建一个键值对时，我们至少会创建两个对象，一个对象用作键值对的键（键对象），另一个对象用于键值对的值（值对象）。对于Redis数据库保存的键值对来说，键总是一个字符串对象。

Redis中每个对象都由一个redisObject结构表示

```c
typedef struct redisObject{
    // 类型
    unsigned type;
    // 编码
    unsigned encoding;
    // 指向底层实现数据结构的指针
    void *ptr;
    // ...
}robj;
```

type

| 类型常量     | 对象的名称   |
| ------------ | ------------ |
| REDIS_STRING | 字符串对象   |
| REDIS_LIST   | 列表对象     |
| REDIS_HASH   | 哈希对象     |
| REDIS_SET    | 集合对象     |
| REDIS_ZSET   | 有序集合对象 |

可以使用type命令查看数据库键对应的值对象的类型。

编码和底层实现

对象的ptr指针指向对象的底层实现数据结构，而这些数据结构由对象的encoding属性决定。

encoding记录了对象使用了什么数据结构作为对象的底层实现

| 编码常量                  | 编码所对应的底层数据结构   |
| ------------------------- | -------------------------- |
| REDIS_ENCODING_INT        | long类型的整数             |
| REDIS_ENCODING_EMBSTR     | embstr编码的简单动态字符串 |
| REDIS_ENCODING_RAW        | 简单动态字符串             |
| REDIS_ENCODING_HT         | 字典                       |
| REDIS_ENCODING_LINKEDLIST | 双端链表                   |
| REDIS_ENCODING_ZIPLIST    | 压缩列表                   |
| REDIS_ENCODING_INTSET     | 整数集合                   |
| REDIS_ENCODING_SKIPLIST   | 跳跃表和字典               |

为了节约内存，每种类型的对象都至少使用了两种不同的编码，注意不是同时使用，而是每次只能用一个编码。

![Redis不同类型和编码的对象](https://gitee.com/Krains/FigureBed/raw/master/img/Redis%E4%B8%8D%E5%90%8C%E7%B1%BB%E5%9E%8B%E5%92%8C%E7%BC%96%E7%A0%81%E7%9A%84%E5%AF%B9%E8%B1%A1.png)

使用OBJECT ENCODING命令可以查看一个数据库键的值对象编码。

### 字符串对象

字符串对象的编码可以是int、raw或者embstr。

如果一个字符串对象保存的是整数值，并且这个整数值可以用long类型来表示，那么字符串对象会将整数值保存在字符串对象结构的ptr属性里面（将void*转换成long），并将字符串对象的编码设置为int。

![Redis_int编码的字符串对象](https://gitee.com/Krains/FigureBed/raw/master/img/Redis_int%E7%BC%96%E7%A0%81%E7%9A%84%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%AF%B9%E8%B1%A1.png)

如果字符串对象保存的是一个字符串值，并且这个字符串值的长度小于32字节，那么字符串对象将使用embstr编码的方式来保存这个字符串值。

![redis_embstr编码的字符串对象](https://gitee.com/Krains/FigureBed/raw/master/img/redis_embstr%E7%BC%96%E7%A0%81%E7%9A%84%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%AF%B9%E8%B1%A1.png)

如果字符串对象保存的是一个字符串值，并且这个字符串值的长度大于32字节，那么字符串对象将使用一个简单动态字符串（SDS）来保存这个字符串值，并将对象的编码设置为raw。

![redis_raw编码的字符串对象](https://gitee.com/Krains/FigureBed/raw/master/img/redis_raw%E7%BC%96%E7%A0%81%E7%9A%84%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%AF%B9%E8%B1%A1.png)

embstr编码与raw编码的区别

raw编码会调用两次内存分配函数来分别创建redisObject结构和sdshdr结构，而emstr编码则通过调用一次内存分配函数来分配一块连续的空间，空间中一次包含redisObject和sdshdr两个结构

embstr编码对比raw的优势

- embstr编码将创建字符串对象所需的内存分配次数从raw编码的两次降低为一次。
- 释放embstr编码的字符串对象只需调用一次内存释放函数，而释放raw需要两次。
- embstr编码的字符串对象的所有数据都保存在一块连续的内存里面，所以这种编码的字符串对象比起raw编码的字符串对象能够更好地利用缓存带来的优势（局部性原理）。

编码的转换

对于int编码的字符串对象来说，如果我们向对象执行了一些命令，使得这个对象保存的不再是整数值，而是一个字符串值，那么字符串对象的编码将从int变为raw，例如append操作，不能转为embstr是因为它需要在redisObject尾部有空闲的空间，而实际上可能不存在。

Redis没有为embstr编码的字符串对象编写任何相应的修改程序，所以embstr编码的字符串对象实际上是只读的。当我们对embstr编码的字符串对象执行任何修改命令时，程序会先将对象的编码从embstr转换成raw，然后再执行修改命令。

参考连接：

- [Redis五种数据类型及应用场景](https://www.cnblogs.com/agilestyle/p/11532375.html)

- Redis设计与实现