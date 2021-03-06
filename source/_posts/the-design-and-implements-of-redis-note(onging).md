---
title: 《Redis设计与思想》读书笔记
date: 2021-02-23 11:40:20
categories:
- Redis
tags:
- Redis
- 读书笔记
---

# 第1部分 Redis数据结构与对象

## 字符串（String）

### 基本使用和性质

```
SET key value
GET key
GETRANGE key start end //返回 key 字符串的一个子串
GETSET key value //设置新值返回旧值
GETBIT key offset //对 key 存储的字符串值获取指定偏移量
STRLEN key //获取 key 的字符串长度
INCR key //将 key 存储的数字值增一
APPEND key value // 对已存在的字符串 key 追加值到末尾
```

- 二进制安全，可以包含任意类型数据（包括jpeg图像，序列化对象等）
- 是 Redis 的键的类型

- 最大长度 512MB
- 应用方式：
  - 适用INCR DECR命令族作为原子计数器
  - 使用APPEND追加字符串
  - 使用GETRANGE和SETRANGE命令，使得字符串作为随机访问变量
  - 编码大量数据到最小空间，创建一个布隆过滤器

### 实现：SDS（Simple Dynamic String）

SDS定义：

```c++
struct sdshdr {
    // 记录 buf 数组已经使用的字节的数量
    // 等于 SDS 所保存字符串的长度
    int len;
    
    // 记录 buf 数组中未使用字节的数量
    int free;
    
    // 字节数组，用于保存字符串
    char buff[];
```

可以看出底层使用 char 数组存储

为了和C操作兼容，SDS遵循了C字符串以空字符(\0)结尾的惯例

SDS存储并自己维护了字符串长度，所以可以常数时间内获取

SDS避免了**缓存区溢出**，修改时会检查剩余空间，如果不满足则按照一定策略扩容

此外，空间分配时还存在 **空间预分配** 和 **惰性空间释放** 两个机制，即 SDS 增长时会额外分配未使用空间；缩短时不马上回收，用 free 记录

SDS 的 len 长度记录保证了字符串的二进制安全性

## 列表（List）

### 基本使用和性质

```
LPUSH key value1 [value2] //将一个或多个值插入到key列表头部，不存在该列表则会建立
LPOP key //移出并获取列表头部第一个元素
LINDEX key index // 通过索引获得key列表中的元素
LLEN key // 获得列表长度
LTRIM key start stop // 对一个列表进行修剪
BLPOP key1 [key2] timeout //弹出头部第一个元素，如果没有元素会阻塞到有元素或超时为止
```

- 和基本双端队列类似，支持添加元素到列表头部（LPUSH）或者尾部（RPUSH）。
- 列表的最大长度是 2^23-1 个元素 (4294967295，超过 40 亿个元素)。
- 一些应用方式：
  - 为社交网络时间轴 (timeline) 建模，使用 LPUSH 命令往用户时间轴插入元素，使用 LRANGE 命令获得最近事项。
  - 使用 LPUSH 和 LTRIM 命令创建一个不会超出给定数量元素的列表，只存储最近的 N 个元素。
  -  列表可以用作消息传递原语，例如，众所周知的用于创建后台任务的 Ruby 库 Resque。
  - 你可以用列表做更多的事情，这种数据类型支持很多的命令，包括阻塞命令，如 BLPOP。

### 实现：双向链表

链表是一个双端无环（表头尾指向null）的链表

Redis链表的节点值是一个 void* 类型，这使得 Redis 链表可以保存不同类型值（多态），还可以通过（dup free match）三个属性设置不同的类型特定函数

## 哈希/哈希键（Hashes）

### 基本使用和性质

哈希键和 HashMap 类似，可以存储多个字符串字段(field)和字符串值之间的映射，所以是表示**对象**的理想数据类型

### 实现：字典/关联数组（dict）

字典是哈希键的**底层实现之一**，同时也是 Redis 数据库**本身的底层实现**

字典通过哈希表实现，哈希表中又包含多个哈希表节点，每个节点代表一个键值对

```c++
// 哈希表实现
typedef struct dictht {
    dictEntry **table;  // 哈希表数组
    unsigned long size;
    
    unsigned long sizemark; // 哈希表大小掩码，用于计算索引值，总是等于 size-1
    
    unsigned long used; // 已有节点数量
}
// 哈希节点实现
typedef struct dictEntry {
    void *key;
    // 各类值的可能形式
    union{
        void *val;
        uint_64_tu64;
        int_64_ts64;
    } v;
    struct dicEntry *next; //发生键冲突时形成链表用
} dictEntry;
// 字典实现
typedef struct dict {
    dictType *type; //类型特定函数
    void *privatedata; //私有数据，用于向类型特定函数传递参数
    
    dictht ht[2]; //哈希表本身，正常存储使用dict[0], dict[1]在 rehash 时使用
    
    in trehashidex; // rehash 索引，如果没有在进行 rehash，为-1
}
```

- **哈希算法**

Redis 字典会先通过哈希算法计算 key 的哈希值，然后再根据**掩码(size-1)进行按位与计算**获取下标值

Redis 使用 MurmurHash2 算法计算键的哈希值

- **rehash**

字典会根据 dictht.used 大小来选择新哈希表的目标大小

**rehash过程会用到两个哈希表，通过创建新表再交换的方式进行扩容**，新哈希表建立在 dict[1] 上，然后将所有元素 rehash 到 dict[0]上

复制完成后，释放 dict[0]，将 ht[1] 设置为新的 ht[0]，并在 ht[1]上创建新哈希表以等待下次 rehash

触发条件：（服务器没有在 `BGWRITE` / `BGREWRITEOF` 时）负载因子>1；（服务器 `BGWRITE` / `BGREWRITEOF` 时）负载因子>5；负载因子 < 0.1（收缩）；负载因子是 used / size

**渐进式rehash**：不立刻将 HashMap 所有节点复制到另一张表上，而是先建空表，每次对字典进行CURD操作时，附带一次复制操作。此时查找和更新会在两个表上进行，并保证ht[0]不存放新值

## 有序集合（Sorted sets）

### 基本使用和性质

Redis有序集合和Redis集合（Sets）类似，是**非重复**字符串集合（collection）。

不同的是，每一个有序集合的成员都有一个关联的分数 (score)，用于按照分数高低排序。尽管成员是唯一的，但是分数是可以重复的。

对有序集合我们可以：

- 通过很快速的方式添加，删除和更新元素 (在**和元素数量的对数成正比**的时间内)
- 由于元素 是有序的而无需事后排序，你可以通过分数或者排名 (位置) 很快地来获取一个范围内的元素
- 很快地访问有序集合的中间元素

有序集合的应用：

- 例如多人在线游戏排行榜，每次提交一个新的分数，你就使用 ZADD 命令更新。你可以很容易地使用 ZRANGE 命令获取前几名用户，你也可以用 ZRANK 命令，通过给定用户名返回其排行。同时使用 ZRANK 和 ZRANGE 命令可以展示与给定用户相似的用户及其分数。以上这些操作都非常的快。
- 有序集合常用来**索引**存储在 Redis 内的数据。例如，假设你有很多表示用户的哈希，你可以使用有序集合，用年龄作为元素的分数，用用户 ID 作为元素值，于是你就可以使用 ZRANGEBYSCORE 命令很快且轻而易举地检 索出给定年龄区间的所有用户了。

### 实现：跳跃表（skiplist）

**【有序集合本身需要同时通过跳跃表和字典实现】**

是**有序集合的实现之一**，在集合**数量包含较多**或者成员字符**串普遍较长**时，有序集合会转为跳跃表实现（但是有序集合需要同时使用跳跃表和字典，**在满足 O(1) 的节点查找速度的同时，保持有序性**）

跳跃表是一种有序数据结构，它通过在每个节点中**维持多个指向其他节点的指针**，从而达到快速访问节点的目的

跳跃表支持平均 O(logN)，最坏 O(N) 复杂度的节点查找，还可以通过顺序性操作来批量处理节点，效率甚至可以和平衡树媲美

结构：基本来说就是每个节点增加了多层指针（仅单向变多）的双向链表，这些层级可以加速访问其他节点的速度

![](https://cdn.jsdelivr.net/gh/syameimarukibou/imagebox/img/image-20210321210057153.png)

## 集合（Sets）

### 基本使用和性质

```

```



Redis集合时没有顺序的字符串集合。可以在 O(1) 的时间复杂度添加、删除和测试元素存在与否 (不管集合中有多少元素都是常量时间)。

与HashSet类似，多次加入同一元素只会保存一个拷贝

Redis 集合非常有意思的是，支持很多服务器端的命令，可以在很短的时间内和已经存在的集合一起计算**并集，交集和差集**。

应用方式包括：

- 你可以使用 Redis 集合追踪唯一性的事情。你想知道访问某篇博客文章的所有唯一IP吗？只要每次页面访问时使用 SADD 命令就可以了。你可以放心，重复的 IP 是不会被插入进来的。
- Redis 集合可以表示关系。你可以通过使用集合来表示每个标签，来创建一个标签系统。然后你可以把所有 拥有此标签的对象的 ID 通过 SADD 命令，加入到表示这个标签的集合中。你想获得同时拥有三个不同标签 的对象的全部 ID 吗？用 SINTER 就可以了。
- 你可以使用 SPOP 或 SRANDMEMBER 命令来从集合中随机抽取元素。

## 其他实现：整数集合（intsets）

仅当一个**集合**只包含整数元素且集合元素不多时，使用该实现

底层数据结构为数组

## 其他实现：压缩列表（ziplist）

压缩列表（ziplist）是**列表键和哈希键**的底层实现之一，当列表键或哈希键只包含少量列表项时，使用压缩列表作为实现

压缩列表时以节约内存为目的的，以**一系列特殊编码连续内存块组成的顺序型数据结构**

每个节点支持一个字节数组或整数值，每个节点包含三个部分：**previous_entry_length（上一个节点的长度）、encoding（该节点编码方式）、content（内容）**

- 连续更新

因为压缩列表的内容连续存储，同时不同内容的长度可能不同，更新某个元素，如果加入新元素后触发下一个节点 `previous_entry_length`值更新并且超过原本长度，会导致该节点扩展，同时可能再触发下一个节点扩展。但是这个情况发生概率较小

## 对象

### 基本结构

通过基本的几种数据结构， Redis创建了一个对象系统，包括以上提到的 **字符串对象、列表对象、哈希对象、集合对象、有序集合对象**

对象的基本定义：

```c
typedef struct redisObject {
    // 类型
    unsigned type:4;
    // 编码
    unsigned encoding:4;
    // 指向底层数据结构的指针
    void *ptr;
} robj;
```

对于 Redis 保存的键值对来说，键总是一个字符串对象，而值可以是上面提到的五个对象之中的一个

一个对象类型和编码的对应关系：

| 类型(REDIS_) | 编码(REDIS_ENCODING_) | 对象                                  |
| ------------ | --------------------- | ------------------------------------- |
| STRING       | INT                   | 整数值实现的字符串对象                |
| STRING       | EMBSTR                | 使用 embstr 编码的SDS实现的字符串对象 |
| STRING       | RAW                   | 使用SDS实现的字符串对象               |
| LIST         | ZIPLIST               | 使用压缩列表的列表对象                |
| LIST         | LINKEDLIST            | 使用双端链表实现的列表对象            |
| HASH         | ZIPLIST               | 使用压缩列表实现的哈希对象            |
| HASH         | HT                    | 使用字典实现的哈希对象                |
| SET          | INTSET                | 使用整数集合实现的集合对象            |
| SET          | HT                    | 使用字典实现的集合对象                |
| ZSET         | ZIPLIST               | 使用压缩列表实现的有序集合对象        |
| ZSET         | SKIPLIST              | 使用跳跃表和字典实现的有序集合对象    |

### 对象内存回收

Redis 自行构建了一个引用计数技术实现的内存回收机制，每个对象包含一个引用计数成员变量

对象的引用计数在这些情况下变化：

- 创建新对象时，引用计数值被初始化为1
- 对象被新程序使用时，引用计数值+1
- 对象不再被一个程序使用时，引用计数-1
- 对象引用计数值变为0时，对象所占用内存被释放

### 对象共享

```c
typedef struct redisObject {
    // ...
    
    // 对象引用
    int refcount;
    
    // ...
    
} robj;
```



对象的 refcount 属性除了实现引用计数内存回收机制之外，还带有对象共享的作用

假设键A先创建了一个包含整数值 100 的字符串对象，那么当键B也创建一个 100 的字符串对象时，服务器自然会考虑让键B共享这个字符串对象

当另一个键去引用一个已经存在的对象的时候，会使得那个对象的引用+1

除此之外，目前来说，Redis 会在初始化服务器时，创建一万个字符串对象， **这些对象包含了从0到9999的所有整数值**，当服务器需要用到值为0到9999的字符串对象时，服务器就会使
用这些共享对象，而不是新创建对象。（也就是说，当我们用 `OBJECT REFCOUNT` 命令去检查一个刚刚建立的10000以内常数键时，其引用已经为 2）

> 创建共享字符串对象的数量可以通过修改redis. h/REDIS_ SHARED_ INTEGERS 常量
> 来修改。

共享键也可以被其他数据结构嵌套下的字符串对象所引用

### 对象空转时长

redisObject 包含的最后一个属性为 lru 属性，该属性记录对象最后一次被命令程序访问的时间

```c
typedef struct redisObject {
    // ...
    
    // 对象引用
    unsigned lru:22;
    
    // ...
    
} robj;
```

通过 `OBJECT IDLETIME` 可以打印给定键的空转时长，通过当前时间减去键的 lru 时间得到

如果服务器打开了 maxmemory 选项，服务器用于回收内存的算法为 volatile-lru 或者 allkey-lru，那么当服务器占用内存超过时，空转时间较高的那部分键会被优先释放

# 第2部分 单机数据库实现

## 第9章 数据库

### 结构

所有数据库被保存在服务器状态 **redis.h/redisServer** 结构的 db 数组中，db 数组的每一个项都是一个 redis.h/**redisDb** 结构，每个 redisDb 结构代表一个数据库

```c
struct redisServer {
	//...
	redisDb *Db
}
```

#### 数据库切换

每个 **Redis 客户端** （在服务器内部通过一个 **redisClient** 结构保存）都保存了一个指向 redisDb 的指针 *Db，也就是客户端自己的目标数据库，可以通过 SELECT 命令来切换数据库

默认的目标数据库为 0 号数据库

> 由于 Redis 到目前为止都还没有可以返回目标数据库的命令，所以处理多数据库程序时需要注意

#### 数据库键空间

每个数据库由 redis.h/redisDb 结构表示， redisDb 结构的 **dict 字典**保存了数据库中所有键值对，我们将这个字典称为**键空间（key space）**

```c
typedef struct redisDb {
    // ...
    // 数据库键空间，保存数据库中所有的键值对
    dict *dict;
    // ...
} redisDb;
```

键空间和用户所见的数据库直接对应，也就是说键空间（*dict）的每个键都是数据库的键，键空间的每个值都是数据库的值

![image-20210325211424274](https://cdn.jsdelivr.net/gh/syameimarukibou/imagebox/img/image-20210325211424274.png)

**键空间维护：**使用命令读写键之后，服务器会在完成指定读写操作的基础上执行一些额外的维护操作

- 读取一个键后，根据键是否存在，更新键空间命中和不命中次数，通过 *INFO states* 命令的 `keyspace_hits` 属性和 `keyspace_misses` 属性中查看
- 读取一个键之后，更新这个键的 lru 时间
- 如果读取键时发现该键过期，那么服务器会删除这个键，根据键管理策略决定
- 如果有客户端使用 *WATCH* 命令监视了一个键，那么服务器在修改键之后会将这个键标记位**脏（dirty）**，使得事务程序能够注意到（见 19章）
- 每次修改键后会把脏键**计数器+1**，会触发服务器的持久化以及复制操作（见10章 11章 15章）
- 如果服务器开启了数据库通知功能，那么修改键之后，会按配置发送响应的数据库通知

### 键过期机制

设置方式：

- *EXPIRE key ttl*（设置秒级别生存时间）
- *PEXPIRE key ttl*  （设置毫秒级别生存时间）
- *EXPIREAT key timestamp* （设置秒级别过期时间戳）
- *PEXPIREAT key timestamp*  （设置毫秒级别过期时间戳）

最终都是通过 PEXPIREAT 命令实现

键的过期时间在 redisDb 结构中的 **expires 字典**中保存，称为过期字典，字典的键为指向键空间中某个键对象的指针，值是一个 long long 类型的整数

> 由于键空间和过期字典指向同一个键对象，所以不会出现任何重复对象，也不会浪费任何区间

过期时间的删除通过 *PERSIST* 操作完成

通过 *TTL* 返回键的生存时间

#### 过期键删除策略

Redis 使用**惰性删除和定期删除两种策略**进行过期键的管理

#### 键过期机制下 AOF、RDB、复制机制的表现

**RDB：**

- **RDB生成：**使用 *SAVE* *BGSAVE* 命令创建一个新的 RDB 文件的时候，**已经过期的键不会 被保存到新的 RDB 文件中**
- **RDB载入：**
  - **主服务器载入：**过期的键不会被载入
  - **从服务器载入：**载入所有键。因为在进行主服务器数据同步时会被清空，所以实际也没有影响

**AOF：**

- **AOF 写入：**由于AOF的机制，如果数据库某个键已经过期但还没有被删除，那么AOF不会受任何影响（无感知）；当过期键被删除之后，**程序会向 AOF 文件追加（append）一条 DEL 命令**，显式记录该键的删除
- **AOF 重写：** 过期的键不会被载入

**复制：**

- 服务器运行在复制模式下时，从服务器的过期键删除动作会由主服务器控制

- 主服务器删除键时，**向所有从服务器发送 *DEL* 命令**
- 从服务器**不会主动删除**在自己空间内发现的过期键，如果过期键被访问，那么仍然会正常访问这个键

## 第10 11章 持久化

### RDB持久化

#### 基本

RDB 是对**保存 Redis 整个内存数据库状态**的的副本文件，是一个经过压缩的二进制文件

RDB 持久化可以手动执行，也可以根据服务器配置定期执行

通过两个 Redis 命令生成 RDB 文件：*SAVE*，*BGSAVE*，区别在于是否阻塞 Redis 进程

Redis 没有显式载入 RDB 文件的命令，Redis 服务器会在**启动时检测 RDB** 文件的存在，然后自动载入 RDB 文件

因为 AOF 更新频率高于 RDB ，所以服务器在开启 AOF 持久化时优先使用AOF文件还原数据库状态，除非关闭 AOF ，否则不会用 RDB 还原数据库状态

使用 *BGSAVE* 命令期间，服务器无法进行 SAVE 命令，同时 *BGREWRITEAOF* 会被延迟到 *BGSAVE* 完成后进行

#### 自动保存

可以通过服务器配置中的 `save` 选项来设置服务器执行 *BGSAVE* 的机制，可以提供多个。只要任意一个条件被满足就会触发 BGSAVE

```
save 900 1  // 当服务器900秒内对服务器进行了1次以上修改时，进行 BGSAVE
save 300 10 // 当服务器300秒内对服务器进行了10次以上修改时，进行 BGSAVE
save 60 10000 // 当服务器10000秒内对服务器进行了60次以上修改时，进行 BGSAVE
```

 自动保存配置被保存在**服务器状态 redisServer 结构的 saveparam 属性**中

除了 saveparam 之外，服务器还维持一个 dirty 计数器和一个 lastsave 属性，分别记录”据上一次 SAVE/BGSAVE 命令之后服务器进行了多少次修改“ 以及 ”上一次成功 SAVE/BGSAVE 的时间戳“

判断是否触发自动保存由 Redis 的周期操作函数 **serverCron** 检查（默认100毫秒执行一次）。

#### RDB文件结构

完整的 RDB 文件包含这些部分

![image-20210325231202853](C:\Users\Kibou\AppData\Roaming\Typora\typora-user-images\image-20210325231202853.png)

REDIS：保存”REDIS“5个字符，用于快速检查是否载入的时 RDB 文件

db_version：代表 RDB 文件的版本号（这里只介绍第6版(0006) RDB 文件的结构）

databases：包含零个或者任意多个数据库，以及库中的键值对数据

EOF：1字节的正文结束标志位；check_sum：8字节长的数字校验和

**database部分**：

![image-20210325232118047](https://cdn.jsdelivr.net/gh/syameimarukibou/imagebox/img/image-20210325232118047.png)

![image-20210325232220222](https://cdn.jsdelivr.net/gh/syameimarukibou/imagebox/img/image-20210325232220222.png)

key 总是一个字符串对象。

根据 TYPE 的不同，RDB 会以不同的方式保存 value 

### AOF持久化

#### 基本

和 RDB 通过保存数据库的键值对记录数据库状态不同，AOF 持久化通过**保存 Redis 服务器所执行的写命令**来记录数据库状态

#### 过程

AOF 持久化的实现可以分为 命令追加、文件写入、文件同步（sync）三个步骤

**命令追加：**

每次执行完写命令，将该命令**追加到服务器状态的 aof_buf 缓冲区**结尾（数据类型为 sds）

**文件写入与同步：**

服务器每次**结束事件循环前**，调用 *flushAppendOnlyFile* 函数，根据策略（服务器配置的 **appendfsync** 选项）决定是否将缓冲区写入 AOF 文件中。

appendfsync 选项有这样一些值：

**always：**将缓冲区所有内容写入，并**马上同步**

**everysec：**将缓冲区所有内容写入，如果**上次同步距离现在超过一秒**，那么再次同步（通过一个线程专门执行）

**no：**将缓冲区所有内容写入，但不同步，将同步**交给操作系统决定**

> 需要决定同步(sync)策略的原因是，现代操作系统在调用 write 函数写文件时，通常会将数据保存在一个内存缓冲区中，不一定会立刻写入磁盘文件。

#### AOF文件载入还原

Redis 命令只能在客户端上下文进行，所以读取 AOF 文件进行数据还原时，需要创建一个伪客户端（fake client），通过这个 fake client 顺序执行指令，行为和带有网络连接的客户端执行指令的效果基本一样

### AOF 重写

由于 AOF 持久化保存所有执行的指令，和 RDF 有增有减的情况不同，AOF 文件会随着服务器运行持续变大。

为了解决这个问题，Redis 提供 AOF 文件重写 （rewrite）功能，从而通过创建一个新的 AOF 文件来代替现有的 AOF 文件

#### 实现原理

简单来说，Redis 通过 **读取服务器当前各个键的状态，用一条构造命令代替AOF文件中这些键相关的所有指令** 来达到重写 AOF 文件的目的

通过 *BGREWRITEAOF* 命令启动

#### 后台重写

AOF 重写的过程会被放到子进程中进行

但是这种方式存在的问题是，如果AOF 重写过程中服务器发生读写，可能会导致重写完成的 AOF 文件和原本服务器中不一致的情况

解决方法是，设置一个 AOF 重写缓冲区，执行写指令时同时向两个缓冲区进行追加

AOF 重写完成后，将 AOF 重写缓冲区的所有内容写入新 AOF 文件，改名，并且原子性地覆盖掉现有的 AOF 文件

![image-20210326000933979](C:\Users\Kibou\AppData\Roaming\Typora\typora-user-images\image-20210326000933979.png)

## 第12章 事件

Redis 服务器是一个事件驱动程序，服务器主要处理以下两类事件

- **文件事件：** Redis 服务通过**套接字**与客户端（或其他 Redis 服务器）进行连接，而文件事件就是服务器对套接字的抽象。服务器与客户端（或者其他服务器）的通信会产生相应的文件事件，服务器通过**监听并处理这些事件**来完成一系列的网络通信操作。
- **时间事件**：Redis 服务器中的一些操作（比如 serverCron 函数）需要在指定时间点执行，而时间事件就是服务器对这类定时操作的抽象。

### 文件事件

Redis 基于 **Reactor 模式**开发了自己的网络事件驱动器，称为**文件事件处理器（file event handler）**

- file event handler 使用 I/O 多路复用程序来监听多个套接字，并根据套接字目前执行的任务来为套接字关联不同的事件处理器
- 当被监听的套接字准备好执行连接应答（accept），读取（read），写入（write），关闭（close）等操作时，与**操作相对应的文件事件就会产生**，这时候 file event handler 就会调用套接字关联好的事件处理器来处理这些事件

虽然 **file event handler** 以单线程方式运行，但是通过 IO 复用程序监听多个套接字，使得 file event handler 本身**既实现了高性能的网络通讯模型，又可以很好地与 Redis 服务器中其他同样单线程运行的模块进行对接**，这样保证了 Redis 内部单线程设计的简单性

#### 文件事件处理器（File Event Hanler）的构成

![image-20210322002505811](https://cdn.jsdelivr.net/gh/syameimarukibou/imagebox/img/image-20210322002505811.png)

文件事件处理器包括四个部分：**套接字，多路复用程序，文件事件派发器（dispatcher），以及事件处理器**

**文件事件**是对**套接字**操作的抽象（下文的套接字=文件事件，每当一个套接字准备好执行连接应答（accept），读取（read），写入（write），关闭（close）等操作时，就会产生相应的文件事件，这些事件可能并发出现

多路复用程序负责监听多个套接字，并向 dispatcher 传送那些产生了事件的套接字

尽管多个文件事件可能并发出现，但是多路复用程序总是会将所有产生d的套接字放到一个**队列**里，通过这个队列，以有序（sequentially）、同步（sychronously）、每次一个套接字的方式向 dispatcher 传送套接字。当上一个套接字被响应处理器处理完毕之后，复用程序才会继续向 dispatcher 传送下一个套接字

dispatcher 接收复用程序传来的套接字，会根据套接字产生事件的类型，调用相应的事件处理器，服务器会为不同任务关联不同事件处理器，也就是一个个函数

#### I/O 多路复用程序的实现

Redis 的 I/O 多路复用程序的所有功能都是通过包装常见的 **select epoll evport 和 kqueue** 这些 I/O 多路复用函数库来实现的，每个 I/O 多路复用函数库在 Redis 源码中都对应一个单独的文件，比如 ae_select.c、ae_epoll.c ae_kqueue.c 等

Redis 为每个多路复用函数库都实现了相同的 API ，所以 多路复用程序的底层实现（select epoll等）可以互换，这使得程序在编译时可以自动选择性能最高的多路复用函数库来作为 Redis 多路复用程序（优先级为：**evport -> epoll -> kqueue -> select** ）

#### 事件类型

当一个套接字同时产生读写事件时，Dispatcher 会优先处理一个套接字的读事件，然后再是写事件

#### API

#### 文件事件处理器

Redis 为文件事件编写了多个处理器，用于实现不同网络通讯需求，比如

- 应答各个客户端 -> 为**监听套接字**关联**连接应答**处理器
- 接收客户端传来的命令请求 -> 为**客户端套接字**关联**命令请求**服务器
- 为了向客户端返回命令执行结果 -> 为**客户端套接字**关联**命令回复**处理器

- 主服务器和从服务器进行复制操作 -> 为**主从服务器**都关联特别为复制功能编写的复制处理器

服务器最常用的是与客户端通信的连接应答处理器、命令请求处理器和命令回复处理器

**连接应答处理器**

位置：networking.c/acceptTcpHandler ，具体实现为 sys/socket.h/accept 函数的包装

Redis 初始化时，程序会将这个连接应答处理器和服务器监听套接字的 `AE_READABLE` 事件关联，当有用户调用 sys/socket.h/connect 函数连接服务监听套接字时，套接字产生 `AE_READABLE` 事件，引发连接应答处理器执行，并执行相应应答操作

**命令请求处理器**

位置：networking.c/readQueryFromClient，具体实现为 unistd.h/read 函数的包装

一个客户端成功连接到服务器之后，服务器会将客户端套接字的  `AE_READABLE` 事件和命令请求处理器关联，客户端发送命令请求就会触发套接字 `AE_READABLE` 事件，引发命令请求处理器执行，并执行相应的套接字读入操作

在连接服务器的整个过程中，服务器都会为客户端套接字的  `AE_READABLE` 事件关联命令请求处理器

**命令回复处理器**

位置：networking.c/snedReplyToClient，具体实现为 unistd.h/write 函数的包装

当服务器有命令回复需要传送给客户端时，服务器会将客户端套接字的 `AE_WRITABLE` 事件和命令回复处理器关联，当客户端准备好接收服务器回传的命令回复时（当客户端**尝试读取命令回复**时），产生 `AE_WRITABLE` 事件，引发命令回复处理器执行，并执行相应套接字写入操作

### 时间事件

正常模式下的 Redis 服务器只使用 serverCron 一个时间事件

#### 时间事件应用实例：serverCron 函数

主要实现 Redis 服务器对自身资源和状态的检查和调整，由 redis.c/serverCron 函数负责执行，主要工作包括：

- 更新服务器的各类**统计信息**，事件、内存占用、数据库占用情况等
- 清理库中**过期键值对**
- 关闭和清理**失效客户端**
- 尝试 AOF RDB **持久化操作**
- 如果服务器是**主服务器**，那么对**从服务器定期同步**
- 如果处于**集群**模式，则对集群进行**定期同步和连接测试**

在Redis2.6版本，服务器默认规定 serverCron 每秒运行10次，平均每间隔**100毫秒运行一次。**

从Redis2.8开始，用户可以通过修改 hz 选项来调整 serverCron 的每秒执行次数，具体信息请参考示例配置文件 redis.conf 关于 hz 选项的说明。

### 不同类型事件调度

![image-20210322013506493](https://cdn.jsdelivr.net/gh/syameimarukibou/imagebox/img/image-20210322013506493.png)

为了减少资源消耗，（当没有文件时事件时）Redis 服务器不会一直轮询时间事件，而是参考最接近当前时间的时间事件，决定 aeApiPoll 调用的阻塞时间（**如果有文件事件到达会停止阻塞返回**）

由于时间事件在文件事件之后执行且不会抢占，时间事件的处理时间通常会比时间事件设定的到达时间稍晚一些

# 第3部分 多机数据库实现

## 第15章 复制

### 基本机制

Redis 中，通过执行 *SLAVEOF* 命令或者设置 `slaveof` 选项，可以让一个服务器成为另一个服务器的从服务器

发送 *SLAVEOF* 命令之后，复制通过两个过程完成，同步（**sync**） 和 命令传播（**command propagate**）

- 同步：将从服务器的状态初始化到主服务器当前的数据库状态，这个过程通过主服务器执行 *BGSAVE* 生成 RDB 文件并发送给从服务器进行
- 命令传播：在同步完成之后，主服务器上发生的所有命令需要传输给从服务器

### 复制的改进

当出现【主从服务器正常运行一段时间后因为网络原因中断复制，随后再连接上】的情况时，同步（SYNC）命令需要对原本主服务器的全部数据生成一次 RDB 文件，这种机制相当低效。为了改变这种情况，引入了 ***PSYNC*** 命令代替，可以**在网络暂时断连的情况下，仅仅只发送断连期间未同步的内容**

这个机制通过主从双方服务器记录的 **复制偏移量** 和 **复制积压缓冲区** 实现

**复制偏移量：**

- 对于主服务器，记录自己已经向从服务器发送过数据的**字节数**
- 对于从服务器，记录自己已经从主服务器收到数据的**字节数**

**复制积压缓冲区**：主服务器进行命令传播时，同时会向具有最大长度的复制积压缓冲区中保存写命令，

当从服务器重连之后，**如果从服务器的 offset 值之后的数据仍然存在于缓冲区中，那么可以进行部分重同步**，否则，将进行完整从同步

复制积压缓冲区的大小可以通过 `repl-backlog-size` 选项进行修改

### 主从心跳

进入命令传播阶段之后，从服务器将以默认每秒一次的频率，向主服务器发送心跳命令，格式为：

```
RELCONF ACK <replication_offset>
```

这个心跳命令包括 检测主从网络连接状态，辅助实现 min-slaves 选项，检测命令丢失 三个作用

**检测主从网络连接状态：**

如果主服务器**超过一秒钟**没有收到从服务器发来的 ACK 命令，那么主服务器将得知从服务器连接出现问题

通过主服务器的 *INFO replication* 选项可以查看服务器最后一次发送 ACK 距现在过了多少秒，即 **lag** 值

**辅助实现 min-slaves 选项：** 

Redis 通过 `min-slaves-to-write` 和 `min-slaves-max-lag` 两个选项防止主服务器在不安全的情况下执行写命令

比如在 `min-slaves-to-write 3  ` 和 `min-slaves-max-lag 10`  的情况下，在从服务器数量少于3个或三个从服务器的 lag 值均大于等于 10 秒的时候，主服务器将拒绝执行写命令 

**检测命令丢失：**

如果从服务器向主服务器发送 ACK 命令的偏移量少于自身偏移量，那么主服务器将根据差距，从缓冲区中找到缺少的数据，发给从服务器

和 部分重同步 的操作很类似，区别在于补发数据操作会在主从服务器未断线的情况下执行

> 检测命令丢失机制仅在 Redis 2.8 以上才支持，在 2.8 以前，如果命令丢失，那么主从服务器都无法感知到

## 第16章 Sentinel

### 基本机制

Sentinel（哨兵）是 Redis 高可用性解决方案

一个 Sentinel 系统由一个或者多个 Sentinel 实例组成，这个系统可以监视任意多个主服务器，当监视的主服务器下线超过设置的时长下限时，将进行这些操作：

- 挑选原主服务器下属的某个从服务器**升级**为新的主服务器
- 向原主服务器下属的所有从服务器发送复制指令，让它们成为新主服务器的从服务器，开始**复制新主服务器**
- 监视已经下线的原主服务器，当其上线后，将它**降级**设置为新主服务器的从服务器

启动一个 Sentinel 可以使用命令：

```
$ redis-sentiel /path/to/your/sentinel.conf
或者
$ redis-server /path/to/your/sentinel.conf --sentinel
```

Sentinel 本质上一个运行在特殊模式下的 redis 服务器 

Sentinel 和主服务器建立的网络连接包括两个异步网络连接，一个是**命令连接**，另一个是**订阅连接**

Sentinel 运行中会通过实例数据结构维护发现的其他个体的信息，这些个体包括 **监视的主服务器，监视的主服务器包括的从服务器，同时监视主服务器的其他 Sentinel** 

### **与主从服务器与其他 Sentinel 通讯**

**INFO 命令**：

Sentinel **每10秒1次**会向主服务器发送 *INFO* 命令，通过分析 *INFO* 命令的回复来获取主服务器的当前信息，包括主服务器本身的信息和下属所有从服务器的信息

Sentinel 除了和主服务器进行双连接之外，还会根据 INFO 命令结果，和主服务器的每个**从服务器建立双连接**，也会向从服务器发送 INFO 命令

**_sentinel___:hello 频道**：

Sentinel 会以**每2秒1次**的频率，向所有被监视的主从服务器的 **_sentinel___:hello 频道**发送关于主从服务器的各类消息，格式如下：

```
PUBLISH __sentinel__:hello "<s_ip>,<s_port>,<s_runid>,<s_epoch>,<m_name>,<m_ip>,<m_port>,<m_epoch>"
```

除了通过 _sentinel___:hello 频道向主从服务器发送消息外，也通过这个订阅频道从服务器接受消息

对于监视同一个服务器的多个 Sentinel 来说，**一个 Sentinel 发送的信息可以被其他 Sentinel 接收到**，这些信息会被用于更新其他 Sentinel 对发送消息的 Sentinel 的认知，也会被用于更新其他 Sentinel 对被监视的服务器的认知

**和其他 Sentinel 的通讯**：

Sentinel 通过频道消息发现新的 Sentinel 时，不仅会为新的 Sentinel 创建实例结构，还会**创建一个连向新 Sentinel 的命令连接**。

最终，监视同一个主服务器的多个 Sentinel 可以形成互相连接的网络，互相可以通过发送命令请求来进行信息交换

### 下线检测

Sentinel 会以**每秒1次**的频率向和它创建了命令连接的**所有实例（主、从、Sentinel）**发送 *PING* 命令

实例对 *PING* 的回复分为以下几种情况：

- 有效回复：返回 ***+PONG*、 *-LOADING* 、*-MASTERDOWN*** 三种回复中的一种
- 无效回复：返回其他回复或者时限内没有任何回复

#### 主观下线

Sentinel 配置文件中的 `down-after-milliseconds` 选项会指定 Sentinel 判断实例进入主观下线状态所需的时间长度：如果在该时限内连续返回无效回复，那么Sentinel 判定实例下线，并在对应实例结构的 flag 属性下设置主观下线标识

> 用户设置的 `down-after-milliseconds` 也会被作为判定主服务器所属所有从服务器是否下线的标准

#### 客观下线

在判定主观下线之后，为了判断一个主服务器是否真的下线，还需要像**同样监视该服务器的其他Sentinel** 发送询问，看它们是否也认为主服务器已经下线（可以是主观/客观下线），收集到足够数量（大于Sentinel 配置中设置的 `quorum` 参数的值）的下线判断后，Sentinel  将其判断为客观下线，并进行故障转移操作

#### 领头 Sentinel 选举

当一个主服务器被判断为客观下线时，监视这个下线主服务器的各个 Sentinel 会进行协商，选举出一个领头 Sentinel ，并由领头 Sentinel 对下线主服务器执行故障转移操作。

这个选举过程是对 **Raft** 算法的领头选举方法的实现

## 第17章 集群

### 基本

Redis 集群通过

# 第4部分 独立功能实现

## 第19章 事务

