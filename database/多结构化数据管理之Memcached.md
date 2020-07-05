## 多结构化数据管理之Memcached

## 1. Memcached简介

- Memcached是一个高性能的`分布式内存对象`缓存系统，用于动态web应用减轻数据库负载。
- 基本特征：在内存中缓存数据和对象，为动态、数据库驱动网站提供更快的运行速度，以至于减少了读取数据库的次数，减少磁盘开销。
- 分布式缓存，不同主机上的多个用户可同时访问， 解决了单机应用的局限。
- 使用自己的页块分配器
- 使用基于存储“键-值”对的`hashmap哈希表`
- 不提供冗余（如复制hashmap条目），当某个服务器S停止运行或崩溃了，所有存放在S上的键-值对都将丢失。

## 2. Memcached应用规则

- 经常访问的表：user、user_details
- 生存周期：变量在memcached的有效期限
- 活跃用户的信息：预先导入到memcached 
- memcached服务部署：多台机器上启动 
- 监控memcached服务：编写相应监控脚本



## 3.  Memcached运行原理

![图片1](https://github.com/LeonAllen/notes/blob/master/database/images/%E5%9B%BE%E7%89%871.png)

尽管是分布式的缓存服务器，但是！！！

服务器端：没有分布式功能

各个memcached之间：不互相通信以共享信息

如何进行分布式：取决于客户端的实现

使用`libevent`作为底层的网络处理组件

![libevent](https://github.com/LeonAllen/notes/blob/master/database/images/libevent.png)

### 3.1 libevent

libevent 学习大门：https://blog.csdn.net/Lemon_tea666/article/details/92637297

libevent github大门：https://github.com/libevent/libevent

libevent：一个异步事件处理程序库，将Linux的epoll、BSD类操作系统的kqueue等事件处理功能封装成统一接口。

使用双向链表保存所有注册的I/O和Signal事件，采用min_heap来管理timeout事件。

主循环函数不断检测注册事件，如果有事件发生，则将其放入就绪链表，并调用事件的回调函数，完成业务逻辑处理。

Libevent接口针对三种事件进行了统一的封装：

- 文件描述符上的特定事件、

- 定时事件、

- 信号

  事件发生时执行回调函数，而不是事件驱动的网络服务器中的event loop。用户只需调用event_dispatch()函数，然后动态的增加或者删除事件。

## 4. Memcached内存分配

早期的Memcached内存分配通过对所有记录进行malloc和free来进行。

1. 容易产生内存碎片；
2. 加重了对操作系统内存管理器的负担。
3. 改进措施：默认采用`Slab Allocator`机制分配、管理内存

`Slab Allocator`基本原理：

`Chunk`——按照预先规定的大小，将分配的内存分割成各种特定长度的块。

`slab class`——尺寸相同的块分成组（chunk的集合）。

分配到的内存不会释放，重复使用已分配的内存。

`Slab Allocator`解决了当初的内存碎片问题，但也产生了新问题：由于分配特定长度内存，可能无法有效利用分配的内存。(说白了就是字节浪费，将100字节的数据缓存到128字节的chunk中，浪费了剩余的28字节。)

## 5.Memcached分布式存储处理

Memcached通过将不同的键保存到不同的服务器上实现了分布式。服务器增多后键会分散，即使一台memcached服务器发生故障，也不影响其他缓存节点，系统依然能继续运行。

Memcached的标准的分布式方法（对键的存储根据服务器台数的余数进行分散）：

1）求得键的整数哈希值；

2）除以服务器台数，根据其余数来选择服务器。

3）当选择的服务器无法连接时，rehash——将连接次数添加到键之后再次计算哈希值并尝试连接。

优点：方法简单，数据的分散性一般较好。

缺点：当添加或移除服务器时，缓存重组的代价大。



改进的分布式方法——`Consistent Hashing`：

1）求出服务器节点的哈希值， 将其配置到0～232的圆上；

2）用同样的方法求出存储数据的键的哈希值并映射到圆上；

3）从数据映射到的位置开始顺时针查找，将数据保存到找到的第一台服务器上；

4）如果超过232仍然找不到服务器，就保存到第一台服务器上。

![Consistent Hashing](https://github.com/LeonAllen/notes/blob/master/database/images/Consistent%20Hashing.png)

  使用一般的hash函数，服务器的映射地点的分布可能出现不均匀的情况。

 解决方案：

虚拟节点：

  为每个物理节点（服务器）在圆环上分配100～200个点，从而抑制分布不均匀，最大限度地减小服务器增减时的缓存重新分布。

## 6. Memcached架构举例

![memcached架构举例](https://github.com/LeonAllen/notes/blob/master/database/images/memcached%E6%9E%B6%E6%9E%84%E4%B8%BE%E4%BE%8B.png)

200台左右的memcached服务器, 每台服务器的容量为3GB，则系统就有了将近600GB的巨大的内存数据库。 
