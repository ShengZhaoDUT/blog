title: Distributed_memcache
date: 2014-05-18 21:31:15
categories: [nosql, Distributed_System]
tags: [memcache, DHT, Distributed System]
---


## Distributed Memory Cache
- memcached

- redis

本文主要总结了memcached和redis两种分布式缓存，两者的实现和Consistent Hash方面的实现。最终给出一个
memcached和redis之间的区别。

## Memcached
* Free & open source, high-performance, distributed memory object caching system.但是其实memcached
并没有实现分布式的集群设置，在server端没有实现key值的分布，需要在客户端自己设计逻辑实现。最简单的key值
分布是通过hash函数。采用hash函数进行key值分布，当有新节点加入或者已经存在的actived节点需要退出
整个集群时，整个hash函数会发生改变，导致整个key值在集群中重新分布，网络开销大。
* 97年有大神（论文
以后我再找）提出了Consistent Hash. 在客户端采用Consistent Hash算法进行key值的分发，计算出当前key
在哪个机器上存取后，直接和该及其通信，完成key-value的读写。
* 现在主流的memcached的Java客户端是:
  
  - memcached-java-client
  
  - spymemcached
  
  - xmemcached
  
  三者的比较在网上比比皆是。总之memcached-java-client比较稳定，但是后两者采用nio的IO方式，效率更高。


## Consistent Hash

* Consistent Hash中路由表的算法
* 虚拟节点

## Redis

* Redis用作缓存也非常常见，相比于memcached主要优势在于：
* 有更加丰富的数据结构，memcached的value虽然支持序列化，但是每次修改时都要将所有的value取回然后更改之后再put回去，网络开销大。
* Redis可以同步到硬盘，数据可靠性更强，这也是它能称之为内存数据库的一个原因吧。
* Redis在数据量比较小的情况下，performance要比memcached好。
* 当然redis的single-thread的模型确实有一些问题，但是内存的性能非常高，而且从一个帖子上看到，单线程也减少了对锁的考虑
							  
  综合以上原因，redis越来越受到人们的欢迎。
* 同样，Redis来作Distributed memcache时，也需要做一些优化。虽然redis现在已经开始支持redis sharding cluster，但是看了设计之后，个人感觉现在server端的设计还很不完善，自发现的机制在可靠性，一致性方面都有一些问题。而且key值的分布也是通过广播的方式告诉其他节点。客户端无论发请求到哪个节点，都可以访问到一个key的range和机器的对应，当然客户端也可以通过缓存这部分数据来实现更好的性能。
* 面对并不是很完善的设计，还是从客户端来实现更优秀，Jedis提供了一个consistent hash的实现，ShardedJedisPool, ShardedJedis可以方便在客户端进行数据的分发。

---

* 邮件(zszhaoshengzs@163.com)

---
