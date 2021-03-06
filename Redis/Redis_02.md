# Redis-集群

### 一、简介

Redis 在3.0版本前只支持单实例模式，虽然支持主从模式部署来解决单点故障，但是现在互联网企业动辄大几百G的数据，完全无法满足业务的需求，所以，Redis 在 3.0 版本以后就推出了集群模式。 

将多台redis服务器组成集群，分担负载。相对于主从架构，是进一步的扩展和升级。集群中的多台主服务器同时对外提供读写功能，并分担整体的负载压力。而且每台主服务器都还会有自己的从服务器，作为数据副本，也作为主服务器的候补，当主服务器意外崩溃，则从服务器自动成为主服务器，保证了整个集群的高可用性。

在负载压力分担，和系统的高可用性上，集群是很好的解决方案 。



### 二、集群搭建

#### 1、配置集群服务器

- 启动至少6个redis服务器（3主3从），每台服务器在配置中要增加： 

  ```
  cluster-enabled yes   #开启集群支持
  
  cluster-config-file nodes.conf   #记录节点信息
  ```

- 以各自的配置开启所有服务器 

  ```
  redis-server 7001/redis.conf
  redis-server 7002/redis.conf
  redis-server 7003/redis.conf
  redis-server 7004/redis.conf
  redis-server 7005/redis.conf
  redis-server 7006/redis.conf
  ```

  

#### 2、创建redis集群

##### 2.1 安装依赖

- 安装ruby
  - tar 解压 ruby-2.4.2.tar.gz
  - cd到解压目录
  - ./configure --prefix=/usr/local/ruby   指定安装位置
  - make && make install
  - ln -s /usr/local/ruby/bin/ruby /usr/bin/ruby  链接
- 安装gem（gem是ruby的一个工具包 ） 
  - yum install rubygems
  - ln -s /usr/local/ruby/bin/gem /usr/bin/gem
- gem install redis（安装redis接口 ）



##### 2.2 创建Redis集群

在redis的src目录下`/usr/local/redis-3.0.7/src`执行：

```
					 #复制      1：1  ip1:port1 ip2:port2 ... ipn:portn   
./redis-trib.rb create --replicas 1 192.168.134.124:7001 192.168.134.124:7002 192.168.134.124:7003 192.168.134.124:7004 192.168.134.124:7005 192.168.134.124:7006 
```

![集群搭建过程](https://github.com/DeerKing007/Data-Base/blob/master/Redis/Redis-pic/集群搭建过程.gif)

`--replicas 1` 表示主从复制比例为 1:1，即一个主节点对应一个从节点；然后，默认给我们分配好了每个主节点和对应从节点服务，以及 slot 的大小，因为在 Redis 集群中有且仅有 16383 个 slot ，默认情况会给我们平均分配，当然你可以指定，后续的增减节点也可以重新分配。 

**查看集群信息**：`./redis-trib.rb check 192.168.134.124:7000 `   

至此，集群搭建完毕，6个节点，3主，3从，只有主节点才拥有槽，并对外提供读写数据。注意至少有3个主节点才可以搭建集群，为每个主分配至少1个从，所以至少需要6个redis节点才可以形成集群。



##### 2.3 槽

- redis cluster 默认分配了 16384 个slot，所有的主redis服务器，大概均分所有的槽 

- 存/取值时 ，redis会根据key，计算一个介于  0 – 16383之间的数字，此数字即为当前数据的槽位置，通过槽位置，决定哪个redis主服务器来负责本次访问

![1534788385982](https://github.com/DeerKing007/Data-Base/blob/master/Redis/Redis-pic/1534788385982.png)

Redis 集群会把数据存在一个master节点，然后在这个master和其对应的slave之间进行数据同步。当读取数据时，也根据一致性哈希算法到对应的master节点获取数据。只有当一个master 挂掉之后，才会启动一个对应的salve节点，充当master。

需要注意的是：必须要`3个或以上`的主节点，否则在创建集群时会失败，并且当存活的主节点数小于总节点数的一半时，整个集群就无法提供服务了。



##### 2.4 集群说明

- 集群搭建后：
  - 性能的进一步提升，可以在单位时间内，吞吐更多的请求
  - 数据的存储节点，具有高可用性(集群有好的容错机制)
- 集群容错：
  - 每个主机都会有自己的从机，则主机宕机后，对应的从机上位成为主机，所以即使有主机发生意外，则整个集群依然可以顺利运行
  - 所有槽可以被覆盖时，则集群正常运行
  - 如果16384个槽，不能被所有主机完整覆盖了，则集群宕机



### 三、集群节点操作

#### 1、从节点操作

##### 1.1 添加从节点

```shell
										 # 主节点id
./redis-trib.rb add-node --slave --master-id 4032891b648e76b2975e5bc701e9c8a52ad6f3dd 192.168.134.124:7006 192.168.134.124:7000 
#新节点的ip:port      #已存在的节点的ip:port
```

##### 1.2 删除从节点

```shell
					#集群中已存在的节点ip:port  #要删除的节点的id
./redis-trib.rb del-node 192.168.134.124:7003  875e192f647ccf06ce6b1d92e15df31a2286dbbf 
```



#### 2、主节点操作

##### 2.1 添加主节点

```shell
./redis-trib.rb add-node 192.168.134.124:7006 192.168.134.124:7000
                         #新增加的主节点         #一个已存在的节点
```

将7006添加到7001所在集群中，成为一个主节点，但此时它不持有任何槽，需要重新分片

##### 2.2 重新分片

```
 ./redis-trib.rb reshard 192.168.134.124:7006
```

分出去的槽，从之前拥有操作的所有主节点中获取。
