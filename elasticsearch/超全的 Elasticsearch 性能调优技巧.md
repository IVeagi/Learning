因为总是看到很多同学在说elasticsearch性能不够好，集群不够稳定，询问关于elasticsearch的调优，但是每次都是一个个点的单独讲，很多时候都是case by case的解答，今天简单梳理下日常的elasticsearch使用调优，以下仅为自己日常经验之谈，如有疏漏，还请大家帮忙指正。
# 一、配置文件调优
elasticsearch.yml
## 内存锁定
bootstrap.memory_lock：true 允许 JVM 锁住内存，禁止操作系统交换出去。
zen.discovery
Elasticsearch 默认被配置为使用单播发现，以防止节点无意中加入集群。组播发现应该永远不被使用在生产环境了，否则你得到的结果就是一个节点意外的加入到了你的生产环境，仅仅是因为他们收到了一个错误的组播信号。 ES是一个P2P类型的分布式系统，使用gossip协议，集群的任意请求都可以发送到集群的任一节点，然后es内部会找到需要转发的节点，并且与之进行通信。 在es1.x的版本，es默认是开启组播，启动es之后，可以快速将局域网内集群名称，默认端口的相同实例加入到一个大的集群，后续再es2.x之后，都调整成了单播，避免安全问题和网络风暴；
单播discovery.zen.ping.unicast.hosts，建议写入集群内所有的节点及端口，如果新实例加入集群，新实例只需要写入当前集群的实例，即可自动加入到当前集群，之后再处理原实例的配置即可，新实例加入集群，不需要重启原有实例； 节点zen相关配置： discovery.zen.ping_timeout：判断master选举过程中，发现其他node存活的超时设置，主要影响选举的耗时，参数仅在加入或者选举 master 主节点的时候才起作用discovery.zen.join_timeout：节点确定加入到集群中，向主节点发送加入请求的超时时间，默认为3sdiscovery.zen.minimum_master_nodes：参与master选举的最小节点数，当集群能够被选为master的节点数量小于最小数量时，集群将无法正常选举。
故障检测（ fault detection ）
两种情况下回进行故障检测，第一种是由master向集群的所有其他节点发起ping，验证节点是否处于活动状态；第二种是：集群每个节点向master发起ping，判断master是否存活，是否需要发起选举。 故障检测需要配置以下设置使用 形如： discovery.zen.fd.ping_interval 节点被ping的频率，默认为1s。 discovery.zen.fd.ping_timeout 等待ping响应的时间，默认为 30s，运行的集群中，master 检测所有节点，以及节点检测 master 是否正常。 discovery.zen.fd.ping_retries ping失败/超时多少导致节点被视为失败，默认为3。

www.elastic.co/guide/en/el…

## 队列数量
不建议盲目加大es的队列数量，如果是偶发的因为数据突增，导致队列阻塞，加大队列size可以使用内存来缓存数据，如果是持续性的数据阻塞在队列，加大队列size除了加大内存占用，并不能有效提高数据写入速率，反而可能加大es宕机时候，在内存中可能丢失的上数据量。 哪些情况下，加大队列size呢？GET /_cat/thread_pool，观察api中返回的queue和rejected，如果确实存在队列拒绝或者是持续的queue，可以酌情调整队列size。

www.elastic.co/guide/en/el…

## 内存使用
设置indices的内存熔断相关参数，根据实际情况进行调整，防止写入或查询压力过高导致OOM， indices.breaker.total.limit: 50%，集群级别的断路器，默认为jvm堆的70%； indices.breaker.request.limit: 10%，单个request的断路器限制，默认为jvm堆的60%； indices.breaker.fielddata.limit: 10%，fielddata breaker限制，默认为jvm堆的60%。

www.elastic.co/guide/en/el…

根据实际情况调整查询占用cache，避免查询cache占用过多的jvm内存，参数为静态的，需要在每个数据节点配置。 indices.queries.cache.size: 5%，控制过滤器缓存的内存大小，默认为10%。接受百分比值，5%或者精确值，例如512mb。

www.elastic.co/guide/en/el…

## 创建shard
如果集群规模较大，可以阻止新建shard时扫描集群内全部shard的元数据，提升shard分配速度。 cluster.routing.allocation.disk.include_relocations: false，默认为true。

www.elastic.co/guide/en/el…

# 二、系统层面调优
## jdk版本
当前根据官方建议，选择匹配的jdk版本；
## jdk内存配置
首先，-Xms和-Xmx设置为相同的值，避免在运行过程中再进行内存分配，同时，如果系统内存小于64G，建议设置略小于机器内存的一半，剩余留给系统使用。 同时，jvm heap建议不要超过32G（不同jdk版本具体的值会略有不同），否则jvm会因为内存指针压缩导致内存浪费，详见：

www.elastic.co/guide/cn/el…

## 交换分区
关闭交换分区，防止内存发生交换导致性能下降（部分情况下，宁死勿慢） swapoff -a
## 文件句柄
Lucene 使用了 大量的 文件。 同时，Elasticsearch 在节点和 HTTP 客户端之间进行通信也使用了大量的套接字，所有这一切都需要足够的文件描述符，默认情况下，linux默认运行单个进程打开1024个文件句柄，这显然是不够的，故需要加大文件句柄数 ulimit -n 65536

www.elastic.co/guide/en/el…

## mmap
Elasticsearch 对各种文件混合使用了 NioFs（ 注：非阻塞文件系统）和 MMapFs （ 注：内存映射文件系统）。请确保你配置的最大映射数量，以便有足够的虚拟内存可用于 mmapped 文件。这可以暂时设置： sysctl -w vm.max_map_count=262144 或者你可以在 /etc/sysctl.conf 通过修改 vm.max_map_count 永久设置它。

www.elastic.co/guide/cn/el…

## 磁盘
如果你正在使用 SSDs，确保你的系统 I/O 调度程序是配置正确的。 当你向硬盘写数据，I/O 调度程序决定何时把数据实际发送到硬盘。 大多数默认 nix 发行版下的调度程序都叫做 cfq（完全公平队列）。但它是为旋转介质优化的： 机械硬盘的固有特性意味着它写入数据到基于物理布局的硬盘会更高效。 这对 SSD 来说是低效的，尽管这里没有涉及到机械硬盘。但是，deadline 或者 noop 应该被使用。deadline 调度程序基于写入等待时间进行优化， noop 只是一个简单的 FIFO 队列。 echo noop > /sys/block/sd/queue/scheduler
## 磁盘挂载
mount -o noatime,data=writeback,barrier=0,nobh /dev/sd* /esdata* 其中，noatime，禁止记录访问时间戳；data=writeback，不记录journal；barrier=0，因为关闭了journal，所以同步关闭barrier； nobh，关闭buffer_head，防止内核影响数据IO
磁盘其他注意事项
使用 RAID 0。条带化 RAID 会提高磁盘I/O，代价显然就是当一块硬盘故障时整个就故障了，不要使用镜像或者奇偶校验 RAID 因为副本已经提供了这个功能。 另外，使用多块硬盘，并允许 Elasticsearch 通过多个 path.data 目录配置把数据条带化分配到它们上面。 不要使用远程挂载的存储，比如 NFS 或者 SMB/CIFS。这个引入的延迟对性能来说完全是背道而驰的。

# 三、elasticsearch使用方式调优

当elasticsearch本身的配置没有明显的问题之后，发现es使用还是非常慢，这个时候，就需要我们去定位es本身的问题了，首先祭出定位问题的第一个命令：
## hot_threads
GET /_nodes/hot_threads&interval=30s
抓取30s的节点上占用资源的热线程，并通过排查占用资源最多的TOP线程来判断对应的资源消耗是否正常，一般情况下，bulk，search类的线程占用资源都可能是业务造成的，但是如果是merge线程占用了大量的资源，就应该考虑是不是创建index或者刷磁盘间隔太小，批量写入size太小造成的。

www.elastic.co/guide/en/el…

## pending_tasks
GET /_cluster/pending_tasks
有一些任务只能由主节点去处理，比如创建一个新的 索引或者在集群中移动分片，由于一个集群中只能有一个主节点，所以只有这一master节点可以处理集群级别的元数据变动。在99.9999%的时间里，这不会有什么问题，元数据变动的队列基本上保持为零。在一些罕见的集群里，元数据变动的次数比主节点能处理的还快，这会导致等待中的操作会累积成队列。这个时候可以通过pending_tasks api分析当前什么操作阻塞了es的队列，比如，集群异常时，会有大量的shard在recovery，如果集群在大量创建新字段，会出现大量的put_mappings的操作，所以正常情况下，需要禁用动态mapping。

www.elastic.co/guide/en/el…

## 字段存储
当前es主要有doc_values，fielddata，storefield三种类型，大部分情况下，并不需要三种类型都存储，可根据实际场景进行调整： 当前用得最多的就是doc_values，列存储，对于不需要进行分词的字段，都可以开启doc_values来进行存储（且只保留keyword字段），节约内存，当然，开启doc_values会对查询性能有一定的影响，但是，这个性能损耗是比较小的，而且是值得的；
fielddata构建和管理 100% 在内存中，常驻于 JVM 内存堆，所以可用于快速查询，但是这也意味着它本质上是不可扩展的，有很多边缘情况下要提防，如果对于字段没有分析需求，可以关闭fielddata；
storefield主要用于_source字段，默认情况下，数据在写入es的时候，es会将doc数据存储为_source字段，查询时可以通过_source字段快速获取doc的原始结构，如果没有update，reindex等需求，可以将_source字段disable；
_all，ES在6.x以前的版本，默认将写入的字段拼接成一个大的字符串，并对该字段进行分词，用于支持整个doc的全文检索，在知道doc字段名称的情况下，建议关闭掉该字段，节约存储空间，也避免不带字段key的全文检索；
norms：搜索时进行评分，日志场景一般不需要评分，建议关闭；
## tranlog
Elasticsearch 2.0之后为了保证不丢数据，每次 index、bulk、delete、update 完成的时候，一定触发刷新 translog 到磁盘上，才给请求返回 200 OK。这个改变在提高数据安全性的同时当然也降低了一点性能。 如果你不在意这点可能性，还是希望性能优先，可以在 index template 里设置如下参数：
{
    "index.translog.durability": "async"
}
复制代码
index.translog.sync_interval：对于一些大容量的偶尔丢失几秒数据问题也并不严重的集群，使用异步的 fsync 还是比较有益的。比如，写入的数据被缓存到内存中，再每5秒执行一次 fsync ，默认为5s。小于的值100ms是不允许的。 index.translog.flush_threshold_size：translog存储尚未安全保存在Lucene中的所有操作。虽然这些操作可用于读取，但如果要关闭并且必须恢复，则需要重新编制索引。此设置控制这些操作的最大总大小，以防止恢复时间过长。达到设置的最大size后，将发生刷新，生成新的Lucene提交点，默认为512mb。
refresh_interval
执行刷新操作的频率，这会使索引的最近更改对搜索可见，默认为1s，可以设置-1为禁用刷新，对于写入速率要求较高的场景，可以适当的加大对应的时长，减小磁盘io和segment的生成；
## 禁止动态mapping
动态mapping的坏处：

造成集群元数据一直变更，导致集群不稳定；
可能造成数据类型与实际类型不一致；
对于一些异常字段或者是扫描类的字段，也会频繁的修改mapping，导致业务不可控。

动态mapping配置的可选值及含义如下： true：支持动态扩展，新增数据有新的字段属性时，自动添加对于的mapping，数据写入成功 false：不支持动态扩展，新增数据有新的字段属性时，直接忽略，数据写入成功 strict：不支持动态扩展，新增数据有新的字段时，报错，数据写入失败
## 批量写入
批量请求显然会大大提升写入速率，且这个速率是可以量化的，官方建议每次批量的数据物理字节数5-15MB是一个比较不错的起点，注意这里说的是物理字节数大小。文档计数对批量大小来说不是一个好指标。比如说，如果你每次批量索引 1000 个文档，记住下面的事实： 1000 个 1 KB 大小的文档加起来是 1 MB 大。 1000 个 100 KB 大小的文档加起来是 100 MB 大。 这可是完完全全不一样的批量大小了。批量请求需要在协调节点上加载进内存，所以批量请求的物理大小比文档计数重要得多。 从 5–15 MB 开始测试批量请求大小，缓慢增加这个数字，直到你看不到性能提升为止。然后开始增加你的批量写入的并发度（多线程等等办法）。 用iostat 、 top 和 ps 等工具监控你的节点，观察资源什么时候达到瓶颈。如果你开始收到 EsRejectedExecutionException ，你的集群没办法再继续了：至少有一种资源到瓶颈了。或者减少并发数，或者提供更多的受限资源（比如从机械磁盘换成 SSD），或者添加更多节点。
## 索引和shard
es的索引，shard都会有对应的元数据，且因为es的元数据都是保存在master节点，且元数据的更新是要hang住集群向所有节点同步的，当es的新建字段或者新建索引的时候，都会要获取集群元数据，并对元数据进行变更及同步，此时会影响集群的响应，所以需要关注集群的index和shard数量，建议如下： 1.使用shrink和rollover api，相对生成合适的数据shard数； 2.根据数据量级及对应的性能需求，选择创建index的名称，形如：按月生成索引：test-YYYYMM，按天生成索引：test-YYYYMMDD； 3.控制单个shard的size，正常情况下，日志场景，建议单个shard不大于50GB，线上业务场景，建议单个shard不超过20GB；
## segment merge
段合并的计算量庞大， 而且还要吃掉大量磁盘 I/O。合并在后台定期操作，因为他们可能要很长时间才能完成，尤其是比较大的段。这个通常来说都没问题，因为大规模段合并的概率是很小的。 如果发现merge占用了大量的资源，可以设置： index.merge.scheduler.max_thread_count: 1 特别是机械磁盘在并发 I/O 支持方面比较差，所以我们需要降低每个索引并发访问磁盘的线程数。这个设置允许 max_thread_count + 2 个线程同时进行磁盘操作，也就是设置为 1 允许三个线程。 对于 SSD，你可以忽略这个设置，默认是 Math.min(3, Runtime.getRuntime().availableProcessors() / 2) ，对 SSD 来说运行的很好。 业务低峰期通过force_merge强制合并segment，降低segment的数量，减小内存消耗； 关闭冷索引，业务需要的时候再进行开启，如果一直不使用的索引，可以定期删除，或者备份到hadoop集群；
## 自动生成_id
当写入端使用特定的id将数据写入es时，es会去检查对应的index下是否存在相同的id，这个操作会随着文档数量的增加而消耗越来越大，所以如果业务上没有强需求，建议使用es自动生成的id，加快写入速率。
## routing
对于数据量较大的业务查询场景，es侧一般会创建多个shard，并将shard分配到集群中的多个实例来分摊压力，正常情况下，一个查询会遍历查询所有的shard，然后将查询到的结果进行merge之后，再返回给查询端。此时，写入的时候设置routing，可以避免每次查询都遍历全量shard，而是查询的时候也指定对应的routingkey，这种情况下，es会只去查询对应的shard，可以大幅度降低合并数据和调度全量shard的开销。
## 使用alias
生产提供服务的索引，切记使用别名提供服务，而不是直接暴露索引名称，避免后续因为业务变更或者索引数据需要reindex等情况造成业务中断。
## 避免宽表
在索引中定义太多字段是一种可能导致映射爆炸的情况，这可能导致内存不足错误和难以恢复的情况，这个问题可能比预期更常见，index.mapping.total_fields.limit ，默认值是1000
## 避免稀疏索引
因为索引稀疏之后，对应的相邻文档id的delta值会很大，lucene基于文档id做delta编码压缩导致压缩率降低，从而导致索引文件增大，同时，es的keyword，数组类型采用doc_values结构，每个文档都会占用一定的空间，即使字段是空值，所以稀疏索引会造成磁盘size增大，导致查询和写入效率降低。
