## 1 不返回大型结果集
## 2避免使用大型文档
   默认 http.max_content_length = 100MB   ES将拒绝索引任何大于次大小的文档
     可以调整，但是lucene仍有1GB大小的限制
## 3文档的分数不可重现
   可能转到同一分片的不通副本上，需要查看index.number_of_replicas 是否大于0
     解决方法是使用一个字符串，将登录用户（例如用户ID或者会话ID）标识为首选项，确保指定用户所有查询命中相同分片，因此多次查询分数一致
![image.png](https://upload-images.jianshu.io/upload_images/15294843-4eea7bebcb65e6d8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
## 4.相关性看起来不对
具有相同内容的两个文档获得不同的分数，或者完全匹配的文档未排在第一位，则问题可能与分片有关
## 5.调整索引速度（写入速度）

   1)使用批量请求 
   
   2)多线程返送数据到ES 
   
   3)取消或者增加刷新间隔 
   
   4)禁用初始加载的，副本设置副本数为0 
   
   5)禁用交换分区swap 
   
   6)为文件系统缓存提供内存 ，主机的一般内存为文件缓存 
   
   7)使用自动生成的ID
   
   8)使用更快的硬件
   
   9)索引缓冲区大小，确保indices.memort.index_buffer_size 足够大，以便每个分片最多提供512MB索引缓冲区来执行索引编制
   
   10)使用跨集群复制可以防止搜索窃取索引中的资源，就是读写分离
    
   11)调整磁盘使用情况：
    
          ①禁用不需要的功能
          
          ②不适用默认动态字符串映射
          
          ③注意分片大小
          
          ④ 禁用_source
          
          ⑤用best_compression 压缩
          
          ⑥强制合并：ES的索引存储在一个或者多个分片中。每个分片都是一个lucene索引，由一个多个段（segment）组成 --磁盘上的实际文件，较大的短在存储数据时效率更高  max_num_segment=1
          
          ⑦ 收缩指数：允许减少索引中的分片数，结合强制合并，有效减少shards 和segment
         
          ⑧使用索引排序来共置类似文档
        
          ⑨ 在文档中使用相同顺序放置字段
        
          ⑩ 汇总历史数据
        
## 6 调整搜索速度
   1) 为文件系统缓存提供内存 ，确保一半的可用内存进入文件系统缓存，这个是头号性能因素
   
   2)使用更快的硬件
   
   3)文档建模：避免连接，嵌套会使查询慢几倍，父子关系会慢数百倍，如果可以通过非规范化文档来回答形同问题而无需连接，会显著加速
   
   4)搜索尽可能少的字段：query_string 和multi_match 查询越多，速度越慢，提高多字段的搜索速度的常用技术是将值复制到单个字段中，然后搜索是使用此字段  可以通过mapping中的 copy_to 执行无需更改文档的源
 ![image.png](https://upload-images.jianshu.io/upload_images/15294843-68d5cf9d24ac52c7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 
   5)索引前数据
   
   6)将映射标识符视为keyword
   
![image.png](https://upload-images.jianshu.io/upload_images/15294843-a1dbe52f1bdb00bc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

   7)避免使用脚本
   
   8)搜索输入日期（比如精确到分钟或者小时）
   
   
   9)强制合并只读索引
   
   10)预热系统缓存 index.store.preload  重启会将文件系统缓存清空
   
   11) 索引排序加快连词速度
   
   12) 用于优化缓存利用率 preference，参考3
   
   13) 适当调整副本数
   
   14)  使用更快的短语查询index_phrases适用于文本字段mapping
   
   15) 使用更快对的'前缀查询'  index_prefixes 文本字段
   
   16) 用于加速过滤 contant_keyword
   
## 7.调整分片大小
分片是为了防止硬件故障并且增加容量，单分片最多20亿文档

[|调整分片大小Elasticsearch Guide [7.15] |弹性的](https://www.elastic.co/guide/en/elasticsearch/reference/7.15/size-your-shards.html)
