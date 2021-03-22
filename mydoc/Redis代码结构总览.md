# Redis 代码结构总览

参考：<https://github.com/redis/redis>

Redis根目录仅包含README、Makefile（实际调用src目录中的Makefile）以及Redis和Sentinel的示例配置。您可以找到一些用于执行Redis、Redis Cluster和Redis Sentinel单元测试的shell脚本，这些脚本在`tests`目录中实现。

根目录内有以下重要目录：

- `src`：包含用C编写的Redis实现。
- `tests`：包含用Tcl实现的单元测试。
- `deps`：包含Redis使用的库。编译Redis所需的所有文件都在此目录中；您的系统只需要提供`libc`、一个POSIX兼容接口和一个C编译器。值得注意的是，`deps`包含`jemalloc`的副本，该副本是Linux下Redis的默认分配器。请注意，在deps下，还有一些东西是从Redis项目开始的，但是主仓库不是`redis/redis`，即<https://github.com/redis>下还有一些Redis用到的仓库。

还有更多目录，但是对于我们这里的目标而言，它们并不是很重要。我们将主要关注包含Redis实现的`src`，探讨每个文件中包含的内容。
公开的文件顺序是合乎逻辑的顺序，以便逐步披露不同层次的复杂性。

注意：Redis最近被大量重构。函数名称和文件名已更改，因此您可能会发现本文档更紧密地反映了`unstable`分支。
例如，在Redis 3.0中，`server.c`和`server.h`文件分别命名为`redis.c`和`redis.h`。但是，总体结构是相同的。
请记住，应针对`unstable`分支执行所有新的开发和pull requests。

## server.h

理解程序工作方式的最简单方法是了解程序使用的**数据结构**。因此，我们将从Redis的主头文件（即`server.h`）开始。

所有服务器配置以及通常所有共享状态都在名为`server`的全局结构`struct redisServer`类型中定义。
此结构中的几个重要字段是：

- `server.db`：Redis数据库的数组，用于存储数据。
- `server.commands`：是命令表。
- `server.clients`：连接到服务器的客户端的链接列表。
- `server.master`：是一个特殊的客户端，如果实例是副本，则是master。

还有其他许多字段。大多数字段都直接在结构定义中注释。

Redis的另一个重要数据结构是定义**客户端的数据结构**。在过去，它被称为`redisClient`，现在只是`client`。
该结构有很多字段，在这里我们仅显示主要字段：

```c
struct client {
    int fd;
    sds querybuf;
    int argc;
    robj **argv;
    redisDb *db;
    int flags;
    list *reply;
    char buf[PROTO_REPLY_CHUNK_BYTES];
    // ... many other fields ...
}
```

client结构体定义了一个已连接的客户端：

- fd：客户端socket文件描述符
- argc和argv填充客户端正在执行命令所需的参数
- querybuf积累来自客户端的请求，Redis服务器根据Redis协议对它进行解析，然后调用客户端命令对应的函数实现。
- reply和buf是动态和静态缓冲区，用于累积服务器发送给客户端的答复。一旦文件描述符可写，这些缓冲区就会按照递增的顺序写入套接字。

如您在上面的客户端结构中所见，命令中的参数被描述为`robj`结构体。
以下是完整的robj结构，其中定义了Redis对象：

(...)

## server.c

这是Redis服务器的入口点，其中定义了`main()`函数。
以下是启动Redis服务器的最重要步骤。

(...)

## networking.c

该文件定义了客户端、主服务器和副本（在Redis中只是特殊的客户端）中的所有I / O功能：

(...)

## aof.c and rdb.c

从名称中可以猜到，这些文件实现了Redis的`RDB`和`AOF`持久性。Redis使用基于`fork()`系统调用的持久性模型，以创建具有与Redis主线程相同（共享）内存内容的线程。
该辅助线程将内存中的内容转储到磁盘上。rdb.c使用此机制在磁盘上创建快照，而aof.c使用此机制在仅附加文件太大时执行AOF重写。

(...)

## db.c

某些Redis命令可对特定数据类型进行操作；
其他都是一般的。通用命令的示例是`DEL`和`EXPIRE`。
它们对键进行操作，而不是针对其值进行操作。
所有这些通用命令都在db.c中定义。

(...)

## object.c

定义Redis对象的robj结构已经描述了。
在object.c内部，有所有在基本级别上与Redis对象一起操作的函数，例如分配新对象，处理引用计数等功能。
该文件中的重要功能有：

(...)

## replication.c

这是Redis中最复杂的文件之一，建议仅在对其余代码库有所了解后再进行处理。在此文件中，实现了Redis的主角色和副本角色。

(...)

## Other C files

- t_hash.c，t_list.c，t_set.c，t_string.c，t_zset.c和t_stream.c包含Redis数据类型的实现。它们既实现了用于访问给定数据类型的API，又实现了针对这些数据类型的客户端命令实现。
- ae.c实现了Redis事件循环，它是一个自包含的库，易于阅读和理解。
- sds.c是Redis字符串库，有关更多信息，请访问http://github.com/antirez/sds。
- 与内核公开的原始接口相比，anet.c是一个以更简单的方式使用POSIX网络的库。
- dict.c是非阻塞哈希表的实现，该哈希表会逐步进行哈希刷新。
- `scripting.c`实现了Lua脚本编制。它是完全独立的，并且与其余的Redis实现隔离，并且足够简单，如果您熟悉Lua API。
- `cluster.c`实现Redis集群。仅在非常熟悉Redis代码库的其余部分之后，才建议阅读。如果要阅读cluster.c，请确保阅读[Redis Cluster规范](https://redis.io/topics/cluster-spec)。
