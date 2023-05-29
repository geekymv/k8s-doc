#### Docker 部署 es 单机版
- mkdir es
- cd es
- vi docker-compose.yml
```yaml
version: "3.7"
services:
  elasticsearch:
    image: "docker.elastic.co/elasticsearch/elasticsearch-oss:7.10.2"
    container_name: elasticsearch
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      node.name: es01
      discovery.type: single-node
      cluster.name: mycluster
      ES_JAVA_OPTS: -Xms512m -Xmx512m
    volumes:
      - "es-data-es01:/usr/share/elasticsearch/data"
    ulimits:
      memlock:
        soft: -1
        hard: -1
  kibana:
    image: docker.elastic.co/kibana/kibana-oss:7.10.2
    container_name: kibana
    depends_on:
      - elasticsearch
    ports:
      - "5601:5601"
      - "9600:9600"
    environment:
      SERVERNAME: kibana
      ELASTICSEARCH_HOSTS: http://elasticsearch:9200
      ES_JAVA_OPTS: -Xmx512m -Xms512m
volumes:
  es-data-es01: {}
```
启动 es 和 kibana
docker compose up

- 访问 es http://192.168.56.101:9200
- 访问 kibana http://192.168.56.101:5601

在 Kibana Dev Tools 中查看 es 集群健康状态
```shell
GET /_cluster/health
结果如下
{
  "cluster_name" : "mycluster",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 1,
  "active_shards" : 1,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```

查看节点状态
```shell
GET /_cat/nodes?v
结果如下
ip         heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
172.19.0.2           26          96  13    0.77    0.95     1.20 dimr      *      es01
```

查看索引情况
```shell
GET /_cat/indices?v
结果如下
health status index     uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   .kibana_1 5U-mzZQUSsCNT6SNjhSdnQ   1   0         15            4       47kb           47kb
```

### Elasticsearch 基本概念
- 一个ES集群由若干个节点（Node）组成
- ES 中的数据存放在节点上
- 数据是已文档（Document）的形式存放的，文档就是JSON对象
- 索引（Index）存放一组相关文档

#### Sharding
- 将一个物理大索引拆分成若干个小索引
- 每个小索引称为一个 Shard, 一个 Shard 本质上是一个 Lucene Index
- Index 是逻辑概念，底层是若干个 Shards, 分布在若干个节点上
- 一个 Shard 最多可存放20亿个文档

#### Sharding 主要作用
- 支持数据量的水平扩展
- 将一个大物理索引拆分成若干个小索引，让每个小索引都能存放在一个单独的节点上
- 提升查询性能，并行查询

#### Sharding 配置
- 一个Index缺省配置一个 Shard（ES7.x）, Shards 并非越多越好，存储开销/管理复杂性
- 增加或减少 Shards，成本比较高
    - 增加 Shards -> Split API
    - 减少 Shards -> Shrink API
- 配置多少个 Shards 合适
    - 视情况而定，节点数量，每个节点的容量，索引数量/大小，查询模式
    - 预期大索引，适当多分配一些 Shards（比如说上百G的索引可以从5个 Shards 开始）
    - 中小索引，从1～2个 Shards 开始
    - 监控+调整
  
#### 副本 Replication
- 副本在索引上配置，确保索引高可用
- 副本是分片的完全拷贝，也称为 Replica shards
- 被复制的源 Shard 称为 Primary shard
- Primary Shard + Replica Shard 统称为 Replication Group
- Replica Shards 也支持查询请求，提升查询性能
- Replica Shards 数量可配置，缺省为1
- Primary Shard 和 Replica Shard(s) 永远不会在同一个节点上
- 单节点集群也可配置 Replica Shard(s)，Replica Shard 的状态是 Unassigned
- 副本数量建议，关键业务数据 >= 2副本，非关键业务1个副本

添加一个索引
```shell
PUT /books
结果如下
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "books"
}
```
查看索引情况
```shell
GET /_cat/indices?v
结果如下
health status index     uuid                   pri rep docs.count docs.deleted store.size pri.store.size
yellow open   books     4bq7A8R8SU27Zg2p1OMnjg   1   1          0            0       208b           208b
green  open   .kibana_1 5U-mzZQUSsCNT6SNjhSdnQ   1   0         17            1     27.7kb         27.7kb
```
可以看到 `books` 索引的健康状态是 yellow, Primary Shard 数量是1, Replica Shards 数量也是1

查看 shards 情况
```shell
GET /_cat/shards?v
结果如下
index     shard prirep state      docs  store ip         node
.kibana_1 0     p      STARTED      19 34.2kb 172.19.0.2 es01
books     0     p      STARTED       0   208b 172.19.0.2 es01
books     0     r      UNASSIGNED                        
```
可以看到 `books` 索引的 Replica Shard 的状态是 UNASSIGNED，因为目前集群只有一个节点。











视频资料 https://www.bilibili.com/video/BV1FL411D7Xp
