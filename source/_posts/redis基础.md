---
title: redis基础
date: 2019-10-05 14:26:00
tags: 
  - redis
categories: 
  - 计算机
---

##  基本介绍

Redis 是一个开源的内存中的数据结构存储系统，它可以用作数据库、缓存和消息中间件，支持多种类型的数据结构,是单进程，单实例。

memcache和redis对比：

memcache <key,value> value没有类型的概念

redis <key,value> value支持多种类型的多种方法。

例：“person,{name:'小明',age:18}”当客户端需要获取value中的age时，memcache会返回所有value值，让客户端去遍历所需要的值 ，而redis会直接返回value相对应的age。本质上解耦，体现计算向数据移动。

数据库排名网站 ：<https://db-engines.com/en/>

redis中文网站：<http:redis.cn>

redis官方网站: <https://redis.io/>

## 基本语法

### string

读取(范围/批量):get k1 / getrange k1 3 -1 / mget k1 k2

赋值(范围/批量):set k1 hello / getrange k1 2 -1 / mset k2 222 k3 333

 							set key nx 不存在时赋值，在分步式锁时使用

​     						set key xx 存在时赋值

append,strlength等等字符串基本语法

数值：递增)incr k1  / incrby k1 22 递减)decr k1 / decrby k1 2

redis是二进制安全的，需要和客户端沟通好编码类型.

bitmap(位图):setbit,getbit ,bitcount,bitop 

| 0(第0个字节)    | 1(第1个字节)  |
| --------------- | ------------- |
| 00000000（8位） | 00000000(8位) |

```
127.0.0.1:6379> setbit k1 1 1 //第0个字节第2位设置为1 即01000000
127.0.0.1:6379> get k1
"@"
127.0.0.1:6379> setbit k1 10 1  //第2个字节第3位设置为1 即01000000
127.0.0.1:6379> get k1
"@ "
127.0.0.1:6379> bitcount k1 0 0 //k1第0个字节有几个1
(integer) 1
127.0.0.1:6379> bitcount k1 0 1 //k1第0至1个字节有几个1
(integer) 2
127.0.0.1:6379> setbit k2 1 1
127.0.0.1:6379> setbit k2 7 1
127.0.0.1:6379> get k2 //01000001
"A"
127.0.0.1:6379> setbit k3 1 1
127.0.0.1:6379> setbit k3 6 1
127.0.0.1:6379> get k3 //01000010
"B"
127.0.0.1:6379> bitop and destkey k2 k3 //按位与操作结果为01000000
127.0.0.1:6379> get destkey
"@"
```

位图场景 1)不定期统计用户登录天数

key为当前用户id,value为bitmap，每一位相当于天数，登录则为1，反之为0

```
setbit zs 1 1 //第2天登录设为1
setbit zs 90 1 //第91天登录设为1
setbit zs 364 1 //第365天登录设为1
bitcount zs -7 -1 //统计最后一周登录天数
```

2）不定期统计活跃用户数量

key为时间，value为bitmap,每一位相当于一个用户id，

```
setbit 20191001 0 1 //2019年10月1号 张三登录
setbit 20191002 0 1 //2019年10月2号 张三登录
setbit 20191002 1 1 //2019年10月2号 李四登录
bitop or destkey 20191001  20191002  //2019年10月1号至10月2号登录人数
bitcount destkey 0 -1 //统计2019年10月1号至10月2号登录人数
```

### list

常用命令:lpush、lrange、lpop、lindex、lset、lrem、linsert、llen、ltrim

 				rpush、range、rpop、blpop

list是重复有序的，包含了栈、队列、数组、单播订阅的概念

### hash

常用命令:hset、hmset、hget、hmget、hvals、hgetall

应用场景:点赞、收藏、详情页

### set

常用命令：sadd、smembers、sinter、sinterstore、sunion、sdiff、srandmember、spop

```
sadd k9 1 2 3 4 5 
srandmember k9 3 //随机取三个，不会重复
srandmember k9 -3  //随机取三个，会重复
srandmember k9 10 //只能取出5个
srandmember k9 -10  //随机取10个，会重复
spop k9 //随机取1个，并把这个值去除 可用于年会抽奖，一人一份的场景
```

  set是去重无序的.应用场景：随机抽奖

### sorted_set

常用命令:zadd、zrange、zrevrange、zscore、zrank、zunionscore

应用场景：排行榜

排序之所以快是因为使用skip_list(跳跃表)实现的。skip_list是一个分层结构多级链表，最下层是原始链表，每个层级都是下一个层级的“高速跑道".跳表具有如下性质：

1)由很多层结构组成

2)每一层都是一个有序链表

3)最底层的链表包含所有元素

4)如果一个元素出现在 i层，则它在i层之下的链表也都会出现

5)每个节点包含两个指针，一个指向同一链表中的下一个元素，一个指向下面一层的元素。

{% asset_img skip_list.png %}

## IO模式及零拷贝

bio 阻塞 

nio 非阻塞 实现方式：select,poll,epoll 多路复用，同步非阻塞

aio 异步非阻塞

|        | 进程连接数         | io效率                               | 消息传递           |
| ------ | ------------------ | ------------------------------------ | ------------------ |
| select | 有限               | 采用轮询方式，遍历所有fd判断是否就绪 | 用户态、内核态切换 |
| poll   | 基于链表，没有限制 | 采用轮询方式，遍历所有fd判断是否就绪 | 用户态、内核态切换 |
| epoll  | 有限，但是上限很大 | 采用回调机制，返回就绪fd             | mmap内存共享空间   |

epoll详解：

epoll的通俗解释是一种当文件描述符的内核缓冲区非空的时候，发出可读信号进行通知，当写缓冲区不满的时候，发出可写信号通知的机制。

核心API:epoll_create,epoll_ctl,epoll_wait

epoll_create生成epoll实例并返回一个文件描述符即epoll句柄，epoll_ctl,epoll_wait均以此为核心。

epoll_ctl 将被监听的描述符添加到红黑树或从红黑树中删除或者对监听事件进行修改

epoll_wait 向用户进程返回处于ready状态下的文件描述符列表

采用回调机制，在执行add操作时，将文件描述符放到红黑树上，同时注册回调函数，内核在检测到某fd可读/写时会调用此回调函数，该回调函数将文件描述符放在就绪链表中。

零拷贝sendfile

零拷贝是指计算机在网络上发送文件时，不需要将文件内容拷贝到用户空间而直接在内核空间中传输到网络的方式，减少cpu的占用和内存带宽。

{% asset_img send01.jpg %}

1、发出sendfile系统调用，用户空间和内核空间上下文切换（第1次上下文切换）

2、通过DMA将磁盘文件中的内容拷贝到内核空间缓冲区（第1次拷贝：hard driver->kernel buffer）

3、DMA发出中断，CPU处理中断，将数据从内核缓冲区拷贝到socket缓冲区(第2次拷贝：kener buffer -> socket buffer)

4、sendfile系统调用返回，内核空间到用户空间上下文切换(第2次上下文切换)

5、通过DMA将socket缓冲区中的数据传递到网卡(第3次拷贝：socket buffer ->网卡)

3次上下文切换，3次拷贝。

{% asset_img send02.jpg %}

1、调用read()，上下文切换到内核，DMA把磁盘数据复制到内核缓冲区

2、read()返回，上下文切换到用户空间，CPU把数据复制到用户的缓存空间

3、write()上下文切换到内核，CPU把数据复制到socket缓冲区

4、write()返回，上下文切换到用户进程

5、DMA把socket缓冲区数据复制到网卡

4次上下文切换，4次拷贝。

