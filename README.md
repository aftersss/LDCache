# LDCache
LDCache（Local and Distributed Cache）Java开发的本地和分布式两级缓存框架。构思类似于OSChina的J2Cache（http://git.oschina.net/ld/J2Cache）
，经过对J2Cache代码的研读，发现其存在很多架构上和代码上的缺陷，对一些异常情况没有做处理，有并发bug。所以自己写了个两级缓存框架。先占位，代码不久后提交。

该框架使用java语言开发，采用gradle构建。

该框架特别适合用于缓存变化频次相对较低（因为每次广播让L1缓存失效有一定开销）、访问频次较高的数据（如果单条缓存的数据量较大，则性能提升更明显）。

该框架可以解决如下问题：
1.热点Key问题：https://yq.aliyun.com/articles/8547?spm=5176.2020520107.105.1.nFSNti
2.降低延迟。（本地有缓存以后，不再需要去远程取，免去了网络开销，同时免去了反序列化的开销）
3.避免完全使用独立缓存系统所带来的网络IO开销问题



第一级缓存使用Ehcache等本地缓存框架（L1），第二级缓存使用 Redis或memcached等分布式缓存框架（L2）。由于大量的缓存读取会导致 L2 的网络成为整个系统的瓶颈，因此 L1 的目标是降低对 L2 的读取次数。该缓存框架主要用于集群环境中。单机也可使用，用于避免应用重启导致的 L1 缓存数据丢失。

LDCache支持 JGroups 和 Redis Subscribe、kafka三种方式进行缓存失效的通知（可以自行扩展）。在某些云平台上可能无法使用 JGroups 组播方式，可以采用 Redis 发布订阅的方式。详情请看 j2cache.properties 配置文件的说明。

数据读取顺序： -> L1 -> L2 -> Database
数据更新流程：
1 依次更新 L1 -> L2 ，广播清除某个缓存信息（不会再更新本机的L1）
2 其他机器接收到广播后从 L1中清除指定的缓存信息

J2Cache 配置
配置文件位于 core/resources 目录下，包含三个文件：

ehcache.xml Ehcache 的配置文件，配置说明请参考 Ehcache 文档
j2cache.properties J2Cache 核心配置文件，可配置 Redis 服务器、连接池以及缓存广播的方式
network.xml JGroups 网络配置，如果使用 JGroups 组播的话需要这个文件，一般无需修改
实际使用过程需要将这三个文件复制到应用类路径中，如 WEB-INF/classes 目录。

构建方法
使用 Ant 构建

安装 Redis
修改 core/resource/j2cache.properties 配置使用已安装的 Redis 服务器
执行 build.sh 进行项目编译
运行多个 runtest.sh
直接在 runtest 输入多个命令进行测试
使用 Maven 构建

$ mvn install

项目直接导入 Eclipse 自动编译

Maven 支持
<dependency>
  <groupId>net.oschina.j2cache</groupId>  
  <artifactId>j2cache-core</artifactId>  
  <version>1.2.0</version>  
</dependency>
示例代码
请看 core/Java/net/oschina/j2cache/CacheTester.java
