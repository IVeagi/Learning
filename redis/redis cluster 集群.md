## 1.为什么要实现Redis Cluster

```
1.主从复制不能实现高可用
2.随着公司发展，用户数量增多，并发越来越多，业务需要更高的QPS，而主从复制中单机的QPS可能无法满足业务需求
3.数据量的考虑，现有服务器内存不能满足业务数据的需要时，单纯向服务器添加内存不能达到要求，此时需要考虑分布式需求，把数据分布到不同服务器上
4.网络流量需求：业务的流量已经超过服务器的网卡的上限值，可以考虑使用分布式来进行分流
5.离线计算，需要中间环节缓冲等别的需求
```

## 2.数据分布

### 2.1 为什么要做数据分布

全量数据，单机Redis节点无法满足要求，按照分区规则把数据分到若干个子集当中

![image](https://upload-images.jianshu.io/upload_images/15294843-46104a2b20710bb0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2.2 常用数据分布方式之顺序分布

```
比如：1到100个数字，要保存在3个节点上，按照顺序分区，把数据平均分配三个节点上
1号到33号数据保存到节点1上，34号到66号数据保存到节点2上，67号到100号数据保存到节点3上
```

![image](https://upload-images.jianshu.io/upload_images/15294843-a59c6bd4d2c3035a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 顺序分区常用在关系型数据库的设计

### 2.3 常用数据分布方式之哈希分布

```
例如1到100个数字，对每个数字进行哈希运算，然后对每个数的哈希结果除以节点数进行取余，余数为1则保存在第1个节点上，余数为2则保存在第2个节点上，余数为0则保存在第3个节点，这样可以保证数据被打散，同时保证数据分布的比较均匀
```

![image](https://upload-images.jianshu.io/upload_images/15294843-83999ac341dd57af.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

哈希分布方式分为三个分区方式：

#### 2.3.1 节点取余分区

比如有100个数据，对每个数据进行hash运算之后，与节点数进行取余运算，根据余数不同保存在不同的节点上

![image](https://upload-images.jianshu.io/upload_images/15294843-014b109ac4db190e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

节点取余方式是非常简单的一种分区方式

节点取余分区方式有一个问题：即当增加或减少节点时，原来节点中的80%的数据会进行迁移操作，对所有数据重新进行分布

> 节点取余分区方式建议使用多倍扩容的方式，例如以前用3个节点保存数据，扩容为比以前多一倍的节点即6个节点来保存数据，这样只需要适移50%的数据。数据迁移之后，第一次无法从缓存中读取数据，必须先从数据库中读取数据，然后回写到缓存中，然后才能从缓存中读取迁移之后的数据

![image](https://upload-images.jianshu.io/upload_images/15294843-bb3c43b91e3d0c1c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

节点取余方式优点：

```
客户端分片
配置简单：对数据进行哈希，然后取余
```

节点取余方式缺点：

```
数据节点伸缩时，导致数据迁移
迁移数量和添加节点数据有关，建议翻倍扩容

```

#### 2.3.2 一致性哈希分区

一致性哈希原理：

```
将所有的数据当做一个token环，token环中的数据范围是0到2的32次方。然后为每一个数据节点分配一个token范围值，这个节点就负责保存这个范围内的数据。
```

![image](https://upload-images.jianshu.io/upload_images/15294843-e276c67ad7068dd3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
对每一个key进行hash运算，被哈希后的结果在哪个token的范围内，则按顺时针去找最近的节点，这个key将会被保存在这个节点上。
```

![image](https://upload-images.jianshu.io/upload_images/15294843-87bc56c54de267e3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image](https://upload-images.jianshu.io/upload_images/15294843-f75b985d4603c1ef.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
在上面的图中，有4个key被hash之后的值在在n1节点和n2节点之间，按照顺时针规则，这4个key都会被保存在n2节点上，
如果在n1节点和n2节点之间添加n5节点，当下次有key被hash之后的值在n1节点和n5节点之间，这些key就会被保存在n5节点上面了
在上面的例子里，添加n5节点之后，数据迁移会在n1节点和n2节点之间进行，n3节点和n4节点不受影响，数据迁移范围被缩小很多

同理，如果有1000个节点，此时添加一个节点，受影响的节点范围最多只有千分之2
一致性哈希一般用在节点比较多的时候
```

一致性哈希分区优点：

```
采用客户端分片方式：哈希 + 顺时针(优化取余)
节点伸缩时，只影响邻近节点，但是还是有数据迁移
```

一致性哈希分区缺点：

```
翻倍伸缩，保证最小迁移数据和负载均衡
```

#### 2.3.3 虚拟槽分区

虚拟槽分区是Redis Cluster采用的分区方式

预设虚拟槽，每个槽就相当于一个数字，有一定范围。每个槽映射一个数据子集，一般比节点数大

> Redis Cluster中预设虚拟槽的范围为0到16383

![image](https://upload-images.jianshu.io/upload_images/15294843-1d22af3936343d21.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

步骤：

```
1.把16384槽按照节点数量进行平均分配，由节点进行管理
2.对每个key按照CRC16规则进行hash运算
3.把hash结果对16383进行取余
4.把余数发送给Redis节点
5.节点接收到数据，验证是否在自己管理的槽编号的范围
    如果在自己管理的槽编号范围内，则把数据保存到数据槽中，然后返回执行结果
    如果在自己管理的槽编号范围外，则会把数据发送给正确的节点，由正确的节点来把数据保存在对应的槽中
```

> 需要注意的是：Redis Cluster的节点之间会共享消息，每个节点都会知道是哪个节点负责哪个范围内的数据槽

虚拟槽分布方式中，由于每个节点管理一部分数据槽，数据保存到数据槽中。当节点扩容或者缩容时，对数据槽进行重新分配迁移即可，数据不会丢失。
虚拟槽分区特点：

```
使用服务端管理节点，槽，数据：例如Redis Cluster
可以对数据打散，又可以保证数据分布均匀
```

#### 2.3 顺序分布与哈希分布的对比

![image](https://upload-images.jianshu.io/upload_images/15294843-6802e58ae1a78adf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 3.Redis Cluster基本架构

### 3.1 节点

Redis Cluster是分布式架构：即Redis Cluster中有多个节点，每个节点都负责进行数据读写操作

每个节点之间会进行通信。

### 3.2 meet操作

节点之间会相互通信

meet操作是节点之间完成相互通信的基础，meet操作有一定的频率和规则

![image](https://upload-images.jianshu.io/upload_images/15294843-80724f455dc37fab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 3.3 分配槽

把16384个槽平均分配给节点进行管理，每个节点只能对自己负责的槽进行读写操作

由于每个节点之间都彼此通信，每个节点都知道另外节点负责管理的槽范围

![image](https://upload-images.jianshu.io/upload_images/15294843-e8a18eaac2124958.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

客户端访问任意节点时，对数据key按照CRC16规则进行hash运算，然后对运算结果对16383进行取作，如果余数在当前访问的节点管理的槽范围内，则直接返回对应的数据
如果不在当前节点负责管理的槽范围内，则会告诉客户端去哪个节点获取数据，由客户端去正确的节点获取数据

![image](https://upload-images.jianshu.io/upload_images/15294843-4e008fb58ac608e6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image](https://upload-images.jianshu.io/upload_images/15294843-d6393d3552c2cc84.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 3.4 复制

保证高可用，每个主节点都有一个从节点，当主节点故障，Cluster会按照规则实现主备的高可用性

对于节点来说，有一个配置项：cluster-enabled，即是否以集群模式启动

### 3.5 客户端路由

#### 3.5.1 moved重定向

```
1.每个节点通过通信都会共享Redis Cluster中槽和集群中对应节点的关系
2.客户端向Redis Cluster的任意节点发送命令，接收命令的节点会根据CRC16规则进行hash运算与16383取余，计算自己的槽和对应节点
3.如果保存数据的槽被分配给当前节点，则去槽中执行命令，并把命令执行结果返回给客户端
4.如果保存数据的槽不在当前节点的管理范围内，则向客户端返回moved重定向异常
5.客户端接收到节点返回的结果，如果是moved异常，则从moved异常中获取目标节点的信息
6.客户端向目标节点发送命令，获取命令执行结果

```

![image](https://upload-images.jianshu.io/upload_images/15294843-94095bab2fecd671.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

需要注意的是：客户端不会自动找到目标节点执行命令

槽命中：直接返回

![image](https://upload-images.jianshu.io/upload_images/15294843-b6a4829b50699980.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
[root@mysql ~]# redis-cli -p 9002 cluster keyslot hello
(integer) 866
```

槽不命中：moved异常

```
[root@mysql ~]# redis-cli -p 9002 cluster keyslot php
(integer) 9244
```

![image](https://upload-images.jianshu.io/upload_images/15294843-546da1e1f6cb6a4f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
[root@mysql ~]# redis-cli -c -p 9002
127.0.0.1:9002> cluster keyslot hello
(integer) 866
127.0.0.1:9002> set hello world
-> Redirected to slot [866] located at 192.168.81.100:9003
OK
192.168.81.100:9003> cluster keyslot python
(integer) 7252
192.168.81.100:9003> set python best
-> Redirected to slot [7252] located at 192.168.81.101:9002
OK
192.168.81.101:9002> get python
"best"
192.168.81.101:9002> get hello
-> Redirected to slot [866] located at 192.168.81.100:9003
"world"
192.168.81.100:9003> exit
[root@mysql ~]# redis-cli -p 9002
127.0.0.1:9002> cluster keyslot python
(integer) 7252
127.0.0.1:9002> set python best
OK
127.0.0.1:9002> set hello world
(error) MOVED 866 192.168.81.100:9003
127.0.0.1:9002> exit
[root@mysql ~]# 
```

#### 3.5.2 ask重定向

![image](https://upload-images.jianshu.io/upload_images/15294843-a0a3f66b63390020.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在对集群进行扩容和缩容时，需要对槽及槽中数据进行迁移

当客户端向某个节点发送命令，节点向客户端返回moved异常，告诉客户端数据对应的槽的节点信息

如果此时正在进行集群扩展或者缩空操作，当客户端向正确的节点发送命令时，槽及槽中数据已经被迁移到别的节点了，就会返回ask，这就是ask重定向机制

![image](https://upload-images.jianshu.io/upload_images/15294843-fd15a0fcebfe0e52.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

步骤：

```
1.客户端向目标节点发送命令，目标节点中的槽已经迁移支别的节点上了，此时目标节点会返回ask转向给客户端
2.客户端向新的节点发送Asking命令给新的节点，然后再次向新节点发送命令
3.新节点执行命令，把命令执行结果返回给客户端
```

moved异常与ask异常的相同点和不同点

```
两者都是客户端重定向
moved异常：槽已经确定迁移，即槽已经不在当前节点
ask异常：槽还在迁移中
```

#### 3.5.3 smart智能客户端

使用智能客户端的首要目标：追求性能

从集群中选一个可运行节点，使用Cluster slots初始化槽和节点映射

将Cluster slots的结果映射在本地，为每个节点创建JedisPool，相当于为每个redis节点都设置一个JedisPool，然后就可以进行数据读写操作

读写数据时的注意事项：

```
每个JedisPool中缓存了slot和节点node的关系
key和slot的关系：对key进行CRC16规则进行hash后与16383取余得到的结果就是槽
JedisCluster启动时，已经知道key,slot和node之间的关系，可以找到目标节点
JedisCluster对目标节点发送命令，目标节点直接响应给JedisCluster
如果JedisCluster与目标节点连接出错，则JedisCluster会知道连接的节点是一个错误的节点
此时JedisCluster会随机节点发送命令，随机节点返回moved异常给JedisCluster
JedisCluster会重新初始化slot与node节点的缓存关系，然后向新的目标节点发送命令，目标命令执行命令并向JedisCluster响应
如果命令发送次数超过5次，则抛出异常"Too many cluster redirection!"

```

![image](https://upload-images.jianshu.io/upload_images/15294843-fd486efff4d627c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 3.6 多节点命令实现

> Redis Cluster不支持使用scan命令扫描所有节点
> 多节点命令就是在在所有节点上都执行一条命令
> 批量操作优化

#### 3.6.1 串行mget

定义for循环，遍历所有的key，分别去所有的Redis节点中获取值并进行汇总，简单，但是效率不高，需要n次网络时间

![image](https://upload-images.jianshu.io/upload_images/15294843-a80aeac6c70fec57.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 3.6.2 串行IO

对串行mget进行优化，在客户端本地做内聚，对每个key进行CRC16hash，然后与16383取余，就可以知道哪个key对应的是哪个槽

本地已经缓存了槽与节点的对应关系，然后对key按节点进行分组，成立子集，然后使用pipeline把命令发送到对应的node，需要nodes次网络时间，大大减少了网络时间开销

![image](https://upload-images.jianshu.io/upload_images/15294843-dc306576a65cc992.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 3.6.3 并行IO

并行IO是对串行IO的一个优化，把key分组之后，根据节点数量启动对应的线程数，根据多线程模式并行向node节点请求数据，只需要1次网络时间

![image](https://upload-images.jianshu.io/upload_images/15294843-f338ac061c8e508f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 3.6.4 hash_tag

将key进行hash_tag的包装，然后把tag用大括号括起来，保证所有的key只向一个node请求数据，这样执行类似mget命令只需要去一个节点获取数据即可，效率更高

![image](https://upload-images.jianshu.io/upload_images/15294843-20c659e95d61d9e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 3.6.5 四种优化方案优缺点分析

![image](https://upload-images.jianshu.io/upload_images/15294843-eeab78cbdc70c64a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 3.7 故障发现

Redis Cluster通过ping/pong消息实现故障发现：不需要sentinel

ping/pong不仅能传递节点与槽的对应消息，也能传递其他状态，比如：节点主从状态，节点故障等

故障发现就是通过这种模式来实现，分为主观下线和客观下线

#### 3.7.1 主观下线

某个节点认为另一个节点不可用，'偏见'，只代表一个节点对另一个节点的判断，不代表所有节点的认知

主观下线流程：

```
1.节点1定期发送ping消息给节点2
2.如果发送成功，代表节点2正常运行，节点2会响应PONG消息给节点1，节点1更新与节点2的最后通信时间
3.如果发送失败，则节点1与节点2之间的通信异常判断连接，在下一个定时任务周期时，仍然会与节点2发送ping消息
4.如果节点1发现与节点2最后通信时间超过node-timeout，则把节点2标识为pfail状态

```

![image](https://upload-images.jianshu.io/upload_images/15294843-2ff6c44a63ec68fb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 3.7.2 客观下线

当半数以上持有槽的主节点都标记某节点主观下线时，可以保证判断的公平性

集群模式下，只有主节点(master)才有读写权限和集群槽的维护权限，从节点(slave)只有复制的权限

客观下线流程：

```
1.某个节点接收到其他节点发送的ping消息，如果接收到的ping消息中包含了其他pfail节点，这个节点会将主观下线的消息内容添加到自身的故障列表中，故障列表中包含了当前节点接收到的每一个节点对其他节点的状态信息
2.当前节点把主观下线的消息内容添加到自身的故障列表之后，会尝试对故障节点进行客观下线操作
```

> 故障列表的周期为：集群的node-timeout * 2，保证以前的故障消息不会对周期内的故障消息造成影响，保证客观下线的公平性和有效性

![image](https://upload-images.jianshu.io/upload_images/15294843-dc717b66570c6328.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![image](https://upload-images.jianshu.io/upload_images/15294843-723a590537b728ba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 3.8 故障恢复

#### 3.8.1 资格检查

```
对从节点的资格进行检查，只有难过检查的从节点才可以开始进行故障恢复
每个从节点检查与故障主节点的断线时间
超过cluster-node-timeout * cluster-slave-validity-factor数字，则取消资格
cluster-node-timeout默认为15秒，cluster-slave-validity-factor默认值为10
如果这两个参数都使用默认值，则每个节点都检查与故障主节点的断线时间，如果超过150秒，则这个节点就没有成为替换主节点的可能性
```

#### 3.9.2 准备选举时间

```
使偏移量最大的从节点具备优先级成为主节点的条件
```

![image](https://upload-images.jianshu.io/upload_images/15294843-4c1bb692e6b80f4d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 3.8.3 选举投票

```
对选举出来的多个从节点进行投票，选出新的主节点
```

![image](https://upload-images.jianshu.io/upload_images/15294843-ba80d7fcca1069de.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 3.8.4 替换主节点

```
当前从节点取消复制变成离节点(slaveof no one)
执行cluster del slot撤销故障主节点负责的槽，并执行cluster add slot把这些槽分配给自己
向集群广播自己的pong消息，表明已经替换了故障从节点
```

#### 3.8.5 故障转移演练

```
对某一个主节点执行kill -9 {pid}来模拟宕机的情况
```

### 3.9 Redis Cluster的缺点

```
当节点数量很多时，性能不会很高
解决方式：使用智能客户端。智能客户端知道由哪个节点负责管理哪个槽，而且当节点与槽的映射关系发生改变时，客户端也会知道这个改变，这是一种非常高效的方式
```

## 4.搭建Redis Cluster

搭建Redis Cluster有两种安装方式

*   1.[原生命令安装](https://www.cnblogs.com/renpingsheng/p/9813959.html)
*   2.[官方工具安装](https://www.cnblogs.com/renpingsheng/p/9833740.html)

    ## 5.开发运维常见的问题

    ### 5.1 集群完整性

cluster-require-full-coverage默认为yes，即是否集群中的所有节点都是在线状态且16384个槽都处于服务状态时，集群才会提供服务

集群中16384个槽全部处于服务状态，保证集群完整性

当某个节点故障或者正在故障转移时获取数据会提示：(error)CLUSTERDOWN The cluster is down

> 建议把cluster-require-full-coverage设置为no

### 5.2 带宽消耗

Redis Cluster节点之间会定期交换Gossip消息，以及做一些心跳检测

官方建议Redis Cluster节点数量不要超过1000个,当集群中节点数量过多时，会产生不容忽视的带宽消耗

消息发送频率：节点发现与其他节点最后通信时间超过cluster-node-timeout /2时，会直接发送PING消息

消息数据量：slots槽数组(2kb空间)和整个集群1/10的状态数据(10个节点状态数据约为1kb)

节点部署的机器规模：集群分布的机器越多且每台机器划分的节点数越均匀，则集群内整体的可用带宽越高

带宽优化：

```
避免使用'大'集群：避免多业务使用一个集群，大业务可以多集群
cluster-node-timeout:带宽和故障转移速度的均衡
尽量均匀分配到多机器上：保证高可用和带宽
```

### 5.3 Pub/Sub广播

在任意一个cluster节点执行publish，则发布的消息会在集群中传播，集群中的其他节点都会订阅到消息，这样节点的带宽的开销会很大

publish在集群每个节点广播，加重带宽

解决办法：需要使用Pub/Sub时，为了保证高可用，可以单独开启一套Redis Sentinel

### 5.4 集群倾斜

对于分布式数据库来说，存在倾斜问题是比较常见的

集群倾斜也就是各个节点使用的内存不一致

#### 5.4.1 数据倾斜原因

1.节点和槽分配不均，如果使用redis-trib.rb工具构建集群，则出现这种情况的机会不多

```
redis-trib.rb info ip:port查看节点，槽，键值分布
redis-trib.rb rebalance ip:port进行均衡(谨慎使用)
```

2.不同槽对应键值数量差异比较大

```
CRC16算法正常情况下比较均匀
可能存在hash_tag
cluster countkeysinslot {slot}获取槽对应键值个数
```

3.包含bigkey：例如大字符串，几百万的元素的hash,set等

```
在从节点：redis-cli --bigkeys
优化：优化数据结构
```

4.内存相关配置不一致

```
hash-max-ziplist-value：满足一定条件情况下，hash可以使用ziplist
set-max-intset-entries：满足一定条件情况下，set可以使用intset
在一个集群内有若干个节点，当其中一些节点配置上面两项优化，另外一部分节点没有配置上面两项优化
当集群中保存hash或者set时，就会造成节点数据不均匀
优化：定期检查配置一致性
```

5.请求倾斜：热点key

```
重要的key或者bigkey
Redis Cluster某个节点有一个非常重要的key，就会存在热点问题
```

#### 5.4.2 集群倾斜优化：

```
避免bigkey
热键不要用hash_tag
当一致性不高时，可以用本地缓存+ MQ(消息队列)
```

### 5.5 集群读写分离

只读连接：集群模式下，从节点不接受任何读写请求

当向从节点执行读请求时，重定向到负责槽的主节点

readonly命令可以读：连接级别命令，当连接断开之后，需要再次执行readonly命令

读写分离：

```
同样的问题：复制延迟，读取过期数据，从节点故障
修改客户端：cluster slaves {nodeId}
```

### 5.6 数据迁移

官方迁移工具：redis-trib.rb和import

只能从单机迁移到集群

不支持在线迁移：source需要停写

不支持断点续传

单线程迁移：影响深度

在线迁移：

```
唯品会：redis-migrate-tool
豌豆荚：redis-port
```

### 5.7 集群VS单机

集群的限制:

```
key批量操作支持有限：例如mget,mset必须在一个slot
key事务和Lua支持有限：操作的key必须在一个节点
key是数据分区的最小粒度：不支持bigkey分区
不支持多个数据库：集群模式下只有一个db0
复制只支持一层：不支持树形复制结构
Redis Cluster满足容量和性能的扩展性，很多业务'不需要'
大多数时客户端性能会'降低'
命令无法跨节点使用：mget,keys,scan,flush,sinter等
Lua和事务无法跨节点使用
客户端维护更复杂：SDK和应用本身消耗(例如更多的连接池)
```

> 很多场景Redis Sentinel已经够用了

### 6.Redis Cluster总结：

```
1.Redis Cluster数据分区规则采用虚拟槽方式(16384个槽)，每个节点负责一部分槽和相关数据，实现数据和请求的负载均衡
2.搭建Redis Cluster划分四个步骤：准备节点，meet操作，分配槽，复制数据。
3.Redis官方推荐使用redis-trib.rb工具快速搭建Redis Cluster
4.集群伸缩通过在节点之间移动槽和相关数据实现
    扩容时根据槽迁移计划把槽从源节点迁移到新节点
    收缩时如果下线的节点有负责的槽需要迁移到其他节点，再通过cluster forget命令让集群内所有节点忘记被下线节点
5.使用smart客户端操作集群过到通信效率最大化，客户端内部负责计算维护键，槽以及节点的映射，用于快速定位到目标节点
6.集群自动故障转移过程分为故障发现和节点恢复。节点下线分为主观下线和客观下线，当超过半数节点认为故障节点为主观下线时，标记这个节点为客观下线状态。从节点负责对客观下线的主节点触发故障恢复流程，保证集群的可用性
7.开发运维常见问题包括：超大规模集群带席消耗，pub/sub广播问题，集群倾斜问题，单机和集群对比等
```

分类: [linux学习笔记](https://www.cnblogs.com/renpingsheng/category/977680.html),[Redis学习记录](https://www.cnblogs.com/renpingsheng/category/1317158.html)

标签: [redis](https://www.cnblogs.com/renpingsheng/tag/redis/), [cluster](https://www.cnblogs.com/renpingsheng/tag/cluster/), [集群](https://www.cnblogs.com/renpingsheng/tag/%E9%9B%86%E7%BE%A4/), [slot](https://www.cnblogs.com/renpingsheng/tag/slot/), [meet](https://www.cnblogs.com/renpingsheng/tag/meet/)

posted @ 2019-07-04 14:25  [割肉机](https://www.cnblogs.com/williamjie/)  阅读(29831)  评论(4)  [编辑](https://i.cnblogs.com/EditPosts.aspx?postid=11132211)  [收藏](javascript:void(0))

Post Comment

[#1楼](https://www.cnblogs.com/williamjie/p/11132211.html#4517518) 2020-03-10 20:17 | [hiyang](https://www.cnblogs.com/hiyang/)

文章写得真棒，看起来十分过瘾，请教一下你这图是用什么工具画的？

[支持(0) ](javascript:void(0);)[反对(0)](javascript:void(0);)

[#2楼](https://www.cnblogs.com/williamjie/p/11132211.html#4576967) 2020-05-14 10:36 | [湖南馒头](https://www.cnblogs.com/simendavid/)

干货满满。

[支持(0) ](javascript:void(0);)[反对(0)](javascript:void(0);)

[#3楼](https://www.cnblogs.com/williamjie/p/11132211.html#4652208) 2020-08-07 14:41 | [Zhuuuuu](https://home.cnblogs.com/u/1937360/)

太强了 哥

[支持(0) ](javascript:void(0);)[反对(0)](javascript:void(0);)

[#4楼](https://www.cnblogs.com/williamjie/p/11132211.html#4677553) 2020-09-07 17:32 | [木子北北](https://home.cnblogs.com/u/2135382/)

又干又硬，收藏了

[支持(0) ](javascript:void(0);)[反对(0)](javascript:void(0);)

[刷新评论](javascript:void(0);)[刷新页面](https://www.cnblogs.com/williamjie/p/11132211.html#)[返回顶部](https://www.cnblogs.com/williamjie/p/11132211.html#top)

登录后才能发表评论，立即 [登录](javascript:void(0);) 或 [注册](javascript:void(0);)， [访问](https://www.cnblogs.com/) 网站首页

[写给园友们的一封求助信](https://www.cnblogs.com/cmt/p/14003277.html)

[【推荐】News: 大型组态、工控、仿真、CADGIS 50万行VC++源码免费下载](http://www.softbam.com/index.htm)
[【推荐】有你助力，更好为你——博客园用户消费观调查，附带小惊喜！](https://www.wenjuan.com/s/UZBZJvjEKs/)
[【推荐】了不起的开发者，挡不住的华为，园子里的品牌专区](https://brands.cnblogs.com/huawei)
[【推荐】未知数的距离，毫秒间的传递，声网与你实时互动](https://brands.cnblogs.com/agora)
[【推荐】博客园x示说网，AI实战系列公开课第三期](https://www.slidestalk.com/m/334?__fuid=31832)
[【福利】AWS携手博客园为开发者送免费套餐](https://brands.cnblogs.com/aws/free?source=CH)

<iframe id="google_ads_iframe_/1090369/C1_0" title="3rd party ad content" name="google_ads_iframe_/1090369/C1_0" width="300" height="250" scrolling="no" marginwidth="0" marginheight="0" frameborder="0" srcdoc="" data-google-container-id="1" data-load-complete="true" style="margin: 0px; padding: 0px; border: 0px; vertical-align: bottom;"></iframe>

**相关博文：**
· [prometheus监控redis,redis-cluster](https://www.cnblogs.com/miaocbin/p/12028105.html "prometheus监控redis,redis-cluster")
· [RedisCluster原理](https://www.cnblogs.com/hello-shf/p/12080123.html "RedisCluster原理")
· [RedisClusterinUbuntu](https://www.cnblogs.com/luffystory/p/12081074.html "RedisClusterinUbuntu")
· [RedisClusterwithSpringBoot](https://www.cnblogs.com/luffystory/p/12083116.html "RedisClusterwithSpringBoot")
· [redis-sentinel-cluster-codis](https://www.cnblogs.com/cavaliersyb/p/12029017.html "redis-sentinel-cluster-codis")
» [更多推荐...](https://recomm.cnblogs.com/blogpost/11132211)

[![AWS免费套餐](https://upload-images.jianshu.io/upload_images/15294843-e46991dce42e19f0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)](https://brands.cnblogs.com/aws/free?source=C2) 

**最新 IT 新闻**:
· [曝张一鸣在游戏群批员工上班时聊游戏！遭回怼：那你退群啊](https://news.cnblogs.com/n/681999/)
· [诺基亚新机宣布：预装轻量化系统Android Go 12月15日登场](https://news.cnblogs.com/n/681998/)
· [三星发布110寸MicroLED面板电视：不烧屏、99.99%屏占比](https://news.cnblogs.com/n/681997/)
· [股价大跌！摩根大通警告特斯拉已被“严重高估” 只值90美元](https://news.cnblogs.com/n/681996/)
· [苹果用户问卷暗示：未来的iPhone包装内恐怕连数据线也不送了](https://news.cnblogs.com/n/681995/)
» [更多新闻...](https://news.cnblogs.com/ "IT 新闻")

**历史上的今天：**
2019-07-04 [sqrt()平方根计算函数的实现1——二分法](https://www.cnblogs.com/williamjie/p/11131314.html)
2019-07-04 [已知 sqrt (2)约等于 1.414，要求不用数学库，求 sqrt (2)精确到小数点后 10 位](https://www.cnblogs.com/williamjie/p/11131093.html)
2019-07-04 [如何实现一个高效的单向链表逆序输出？](https://www.cnblogs.com/williamjie/p/11131087.html)

昵称： [割肉机](https://home.cnblogs.com/u/williamjie/)
园龄： [5年5个月](https://home.cnblogs.com/u/williamjie/ "入园时间：2015-06-14")
粉丝： [492](https://home.cnblogs.com/u/williamjie/followers/)
关注： [3](https://home.cnblogs.com/u/williamjie/followees/)

[+加关注](javascript:void(0))

| 

| [<](javascript:void(0);) | 2020年12月 | [>](javascript:void(0);) |

 |
| 日 | 一 | 二 | 三 | 四 | 五 | 六 |
| 29 | 30 | 1 | 2 | 3 | 4 | 5 |
| 6 | 7 | 8 | 9 | 10 | 11 | 12 |
| 13 | 14 | 15 | 16 | 17 | 18 | 19 |
| 20 | 21 | 22 | 23 | 24 | 25 | 26 |
| 27 | 28 | 29 | 30 | 31 | 1 | 2 |
| 3 | 4 | 5 | 6 | 7 | 8 | 9 |

### 搜索

<input type="text" id="q" onkeydown="return zzk_go_enter(event);" class="input_my_zzk" style="margin: 0px; padding: 0px; width: 140px; vertical-align: middle; height: 20px;"> <input onclick="zzk_go()" type="button" value="找找看" id="btnZzk" class="btn_my_zzk" style="margin: 0px; padding: 0px 5px; appearance: button; vertical-align: middle; height: 22px; font-size: 12px;">

<input type="text" name="google_q" id="google_q" onkeydown="return google_go_enter(event);" class="input_my_zzk" style="margin: 0px; padding: 0px; width: 140px; vertical-align: middle; height: 20px;"> <input onclick="google_go()" type="button" value="谷歌搜索" class="btn_my_zzk" style="margin: 0px; padding: 0px 5px; appearance: button; vertical-align: middle; height: 22px; font-size: 12px;">

### 随笔分类

*   [Ainterview面试(19)](https://www.cnblogs.com/williamjie/category/1485042.html)
*   [Binterview每天(11)](https://www.cnblogs.com/williamjie/category/1496033.html)
*   [c++(1)](https://www.cnblogs.com/williamjie/category/1505192.html)
*   [docker(12)](https://www.cnblogs.com/williamjie/category/1225432.html)
*   [dubble(1)](https://www.cnblogs.com/williamjie/category/1262440.html)
*   [git(4)](https://www.cnblogs.com/williamjie/category/1242310.html)
*   [golang(41)](https://www.cnblogs.com/williamjie/category/1241689.html)
*   [Hibernate(6)](https://www.cnblogs.com/williamjie/category/1238093.html)
*   [Java(183)](https://www.cnblogs.com/williamjie/category/1224801.html)
*   [JVM(24)](https://www.cnblogs.com/williamjie/category/1265650.html)
*   [k8s(7)](https://www.cnblogs.com/williamjie/category/1242437.html)
*   [kafka(6)](https://www.cnblogs.com/williamjie/category/1310155.html)
*   [linux(54)](https://www.cnblogs.com/williamjie/category/1242975.html)
*   [mgo(43)](https://www.cnblogs.com/williamjie/category/1242506.html)
*   [MyBatis(3)](https://www.cnblogs.com/williamjie/category/1503915.html)
*   [更多](javascript:void(0))

### 随笔档案

*   [2020年3月(1)](https://www.cnblogs.com/williamjie/archive/2020/03.html)
*   [2020年1月(1)](https://www.cnblogs.com/williamjie/archive/2020/01.html)
*   [2019年12月(4)](https://www.cnblogs.com/williamjie/archive/2019/12.html)
*   [2019年9月(18)](https://www.cnblogs.com/williamjie/archive/2019/09.html)
*   [2019年8月(24)](https://www.cnblogs.com/williamjie/archive/2019/08.html)
*   [2019年7月(142)](https://www.cnblogs.com/williamjie/archive/2019/07.html)
*   [2019年6月(50)](https://www.cnblogs.com/williamjie/archive/2019/06.html)
*   [2019年5月(6)](https://www.cnblogs.com/williamjie/archive/2019/05.html)
*   [2019年4月(17)](https://www.cnblogs.com/williamjie/archive/2019/04.html)
*   [2019年3月(17)](https://www.cnblogs.com/williamjie/archive/2019/03.html)
*   [2019年2月(6)](https://www.cnblogs.com/williamjie/archive/2019/02.html)
*   [2019年1月(37)](https://www.cnblogs.com/williamjie/archive/2019/01.html)
*   [2018年12月(30)](https://www.cnblogs.com/williamjie/archive/2018/12.html)
*   [2018年11月(28)](https://www.cnblogs.com/williamjie/archive/2018/11.html)
*   [2018年10月(19)](https://www.cnblogs.com/williamjie/archive/2018/10.html)
*   [更多](javascript:void(0))

### Recent Comments

*   [1\. Re:Redis分布式锁的正确实现方式](https://www.cnblogs.com/williamjie/p/9395659.html)
*   @我是小菜啊1 解锁时判断id，但总感觉这去问个过期时间有些问题...
*   --极速开发社区
*   [2\. Re:java NIO面试题剖析](https://www.cnblogs.com/williamjie/p/11194561.html)
*   写入数据到Buffer，调用flip()方法，从Buffer中读取数据，调用clear()方法或者compact()方法。 对于这一段的描述写错了吧 新建的Buffer是可以直接写入的 写入完成之后想...
*   --我在东北玩泥巴丶
*   [3\. Re:Redis分布式锁的实现原理](https://www.cnblogs.com/williamjie/p/11250679.html)
*   如果客户端1因为watchdog一直持有锁，不释放，请问怎么去释放这个锁，应该还有一个最大超时时间阈值吧

*   --nan征
*   [4\. Re:一个用消息队列 的人，不知道为啥用 MQ，这就有点尴尬](https://www.cnblogs.com/williamjie/p/9481780.html)
*   1、解耦
    你们服务间的调用都通过mq吗？那服务降级，熔断怎么做。
    2、异步
    异步不是能使用线程池吗？

*   --寂静的春天
*   [5\. Re:Mysql索引面试题](https://www.cnblogs.com/williamjie/p/11187470.html)
*   感谢分享

*   --溢性循环

### Top Posts

*   [1\. HashTable和HashMap的区别详解(203612)](https://www.cnblogs.com/williamjie/p/9099141.html)
*   [2\. HashMap的扩容机制---resize()(156052)](https://www.cnblogs.com/williamjie/p/9358291.html)
*   [3\. redis的三种启动方式(140102)](https://www.cnblogs.com/williamjie/p/9370196.html)
*   [4\. TCP和UDP的最完整的区别(128275)](https://www.cnblogs.com/williamjie/p/9390164.html)
*   [5\. java获取当前时间戳的方法(114274)](https://www.cnblogs.com/williamjie/p/9259591.html)

### 推荐排行榜

*   [1\. HashTable和HashMap的区别详解(28)](https://www.cnblogs.com/williamjie/p/9099141.html)
*   [2\. 一个用消息队列 的人，不知道为啥用 MQ，这就有点尴尬(23)](https://www.cnblogs.com/williamjie/p/9481780.html)
*   [3\. 高可用Redis：Redis Cluster(16)](https://www.cnblogs.com/williamjie/p/11132211.html)
*   [4\. Redis分布式锁的正确实现方式(16)](https://www.cnblogs.com/williamjie/p/9395659.html)
*   [5\. 浅谈HTTP中GET、POST用法以及它们的区别(16)](https://www.cnblogs.com/williamjie/p/9099940.html)
