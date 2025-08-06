# 图解：Redis主从同步与同步流程

> 原创 已于 2025-06-15 09:54:06 修改 · 公开 · 947 阅读 · 23 · 14 · CC 4.0 BY-SA版权 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
> 文章链接：https://blog.csdn.net/lyh2004_08/article/details/148666417

**目录**

[TOC]



## <span style="color:black">一、Redis的主从同步</span>

> <span style="color:#262626">单节点</span><span style="color:#262626">Redis</span><span style="color:#262626">的并发能力是有上限的，要进一步提高</span><span style="color:#262626">Redis</span><span style="color:#262626">的并发能力，就需要搭建主从集群，实现读写分离。</span>
> 
> <span style="color:#262626">一般都是一主多从，主节点负责写数据，从节点负责读数据。</span>

---

## <span style="color:black">二、主从同步数据的流程</span>

###  **1.全量同步** 

> <span style="color:black">1. 从节点请求主节点同步数据（</span><span style="color:black">replication id</span><span style="color:black">、</span><span style="color:black">offset</span><span style="color:black">）</span>
> 
> 2. 主节点判断是否是第一次请求， **是第一次** 就与从节点同步版本信息（replication id<span style="color:black">和</span><span style="color:black">offset</span><span style="color:black">）</span>
> 
> 3. 主节点执行 **bgsave** <span style="color:black">，生成</span> **<span style="color:black">rdb</span>** <span style="color:black">文件后，发送给从节点去执行</span>
> 
> <span style="color:black">4.</span><span style="color:black">在</span><span style="color:black">rdb</span><span style="color:black">生成执行期间，主节点会以命令的方式记录到缓冲区（一个日志文件）</span>
> 
> <span style="color:black">5.</span><span style="color:black">把生成之后的命令日志文件发送给从节点进行同步</span>

#### <span style="color:black">图解</span>

 <img src="./assets/008_1.png" alt="" style="max-height:489px; box-sizing:content-box;" />

其中：

> <span style="color:#c00000">Replication Id</span><span style="color:#262626">：简称</span><span style="color:#262626">replid</span><span style="color:#262626">，是数据集的标记，</span><span style="color:#262626">id</span><span style="color:#262626">一致则说明是同一数据集。每一个</span><span style="color:#262626">master</span><span style="color:#262626">都有唯一的</span><span style="color:#262626">replid</span><span style="color:#262626">，</span><span style="color:#262626">slave</span><span style="color:#262626">则会继承</span><span style="color:#262626">master</span><span style="color:#262626">节点的</span><span style="color:#262626">replid</span>
> 
> <span style="color:#c00000">offset</span><span style="color:#262626">：偏移量，随着记录在</span><span style="color:#262626">repl_baklog</span><span style="color:#262626">中的数据增多而逐渐增大。</span><span style="color:#262626">slave</span><span style="color:#262626">完成同步时也会记录当前同步的</span><span style="color:#262626">offset</span><span style="color:#262626">。如果</span><span style="color:#262626">slave</span><span style="color:#262626">的</span><span style="color:#262626">offset</span><span style="color:#262626">小于</span><span style="color:#262626">master</span><span style="color:#262626">的</span><span style="color:#262626">offset</span><span style="color:#262626">，说明</span><span style="color:#262626">slave</span><span style="color:#262626">数据落后于</span><span style="color:#262626">master</span><span style="color:#262626">，需要更新</span>

###  **2.增量同步** 

> 1. 从节点请求主节点同步数据，主节点判断 **不是第一次请求** ，就获取从节点的offset<span style="color:black">值</span>
> 
> <span style="color:black">2.</span><span style="color:black">主节点从命令日志中获取</span><span style="color:black">offset</span><span style="color:black">值之后的数据，发送给从节点进行数据同步</span>

#### <span style="color:black">图解</span>

 <img src="./assets/008_2.png" alt="主从增量同步(slave重启或后期数据变化)" style="max-height:583px; box-sizing:content-box;" />

---

## 三、完整口述

Q： **能说一下，主从同步数据的流程吗？** 

> 主从同步分为了两个阶段，一个是 **全量同步** ，一个是 **增量同步** 。
> 
> 

1. 全量同步是指从节点第一次与主节点建立连接的时候使用全量同步，流程是这样的：
> 
>    1. 第一：从节点请求主节点同步数据，其中从节点会携带自己的replication id和offset偏移量。
> 
>    2. 第二：主节点判断是否是第一次请求，主要判断的依据就是，主节点与从节点是否是同一个replication id，如果不是，就说明是第一次同步，那主节点就会把自己的replication id和offset发送给从节点，让从节点与主节点的信息保持一致。
> 
>    3. 第三：在同时主节点会执行 **`BGSAVE`** ，生成 **RDB** 文件后，发送给从节点去执行，从节点先把自己的数据清空，然后执行主节点发送过来的 **RDB** 文件，这样就保持了一致。
> 
>    4. 当然，如果在 **RDB** 生成执行期间，依然有请求到了主节点，而主节点会以命令的方式记录到缓冲区，缓冲区是一个日志文件，最后把这个日志文件发送给从节点，这样就能保证主节点与从节点完全一致了，后期再同步数据的时候，都是依赖于这个日志文件，这个就是全量同步。
> 
> 

2. 增量同步指的是，当从节点服务重启之后，数据就不一致了，所以这个时候，从节点会请求主节点同步数据，主节点还是判断不是第一次请求，不是第一次就获取从节点的offset值，然后主节点从命令日志中获取offset值之后的数据，发送给从节点进行数据同步。

参考：有关 **RDB** 、 **BGSAVE** ： [Redis的数据持久化](https://blog.csdn.net/lyh2004_08/article/details/148606461) 