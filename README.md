先占位，代码不久后提交。

# LDCache

##简介
LDCache（Local and Distributed Cache）Java开发的本地和分布式两级缓存框架。

##主流缓存系统的痛点
目前缓存主要分为两种，本地缓存（EhCache等）、分布式缓存（Memcached、Redis等）。两种缓存方式各有优缺点：
本地缓存适合缓存长期甚至永久不会改变的数据，由于集成在本地jvm中，访问速度快、性能高，不需要网络开销，但是需要占用jvm的内存，jvm重启缓存会丢失（除非缓存会持久化到硬盘，但这是以性能损失为代价的），不适合缓存需要经常改变的数据（除非你是单机）。
集中式缓存集中存储数据，缓存独立于应用服务器，应用服务器重启缓存不会丢失，但是每次获取缓存都需要网络开销、反序列化开销，增加延时，高并发下容易出现热点key问题。
目前部分本地缓存已经支持分布式，比如ehcache，他有两种同步方式，copy/invalidate, copy方式就是某个key放入了新的缓存时，把该缓存复制到集群中的其他机器,这会加重网络负担（缓存复制到其他机器后，并不一定会被访问到）。invalidate方式就是让其他机器的缓存失效，此方式只需要传播key等简短信息，网络负载较低，但是其他机器需要重新从数据库加载，增大数据库负担，这种方式不够完美。

##LDCache设计思路
为了综合本地缓存和集中式缓存的优点，有了本项目的诞生。

本项目使用Ehcache等本地缓存框架作为第一级缓存（L1）， 使用Redis或memcached等集中式缓存框架作为第二级缓存（L2）。读取缓存时会先尝试用L1取，取不到再去L2取，L2取到后会把缓存也放进L1，L2还取不到再去数据库读取。缓存失效时会把L1、L2缓存都清除，同时广播消息，让集群中其他机器的L1缓存都失效。

LDCache支持 JGroups 和 Redis Subscribe、kafka三种方式进行缓存失效的通知（可以自行扩展）。在某些云平台上可能无法使用 JGroups 组播方式，可以采用 Redis 发布订阅的方式。详情请看 ldcache.properties 配置文件的说明。

该框架使用java语言开发，采用gradle构建。

##LDCache适用范围
该框架特别适合用于缓存变化频次相对较低（因为每次广播让L1缓存失效有一定开销）、访问频次较高的数据（如果单条缓存的数据量较大，则性能提升更明显）。对于变化频次较高、访问频次较低的缓存，建议直接走Redis、memcached等集中式缓存。

该框架可以解决如下问题：
1.解决热点Key问题，参考这篇文章：https://yq.aliyun.com/articles/8547?spm=5176.2020520107.105.1.nFSNti
2.降低延迟。（本地有缓存以后，不再需要去远程取，免去了网络开销，同时免去了反序列化的开销）
3.避免完全使用独立缓存系统所带来的网络IO开销问题（高并发下大量缓存的存取操作会造成网络拥塞，使网络成为瓶颈）

##使用限制
由于本项目的主要目的是管理本地缓存和集中式缓存，如果使用incr、decr等操作会带来一致性问题，所以本框架只有简单的get、set、remove操作。

J2Cache 配置
配置文件位于 core/resources 目录下，包含三个文件：

ehcache.xml Ehcache 的配置文件，配置说明请参考 Ehcache 文档
j2cache.properties J2Cache 核心配置文件，可配置 Redis 服务器、连接池以及缓存广播的方式
network.xml JGroups 网络配置，如果使用 JGroups 组播的话需要这个文件，一般无需修改
实际使用过程需要将这三个文件复制到应用类路径中，如 WEB-INF/classes 目录。

构建方法

安装 Redis
修改 core/resource/j2cache.properties 配置使用已安装的 Redis 服务器
执行 build.sh 进行项目编译
运行多个 runtest.sh
直接在 runtest 输入多个命令进行测试

项目直接导入 Eclipse 自动编译

Maven 支持
<dependency>
  <groupId>ldcache</groupId>  
  <artifactId>ldcache</artifactId>  
  <version>1.1.2</version>  
</dependency>

示例代码
请看 core/Java/net/oschina/j2cache/CacheTester.java


本项目构思类似于OSChina的J2Cache  http://git.oschina.net/ld/J2Cache
经过对J2Cache代码的研读，发现其存在很多架构上和代码上的缺陷，对一些异常情况没有做处理，有并发bug。所以自己写了这个两级缓存框架。
