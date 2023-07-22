列举下需要安装的软件及版本
- CentOS 7
- Docker 20.10.7  
- MySQL 8.0.23
- Elasticsearch 7.10.2
- Kibana 7.10.2
- JDK 1.8
- Canal 1.1.5

本文的重点是 Canal，为了方便直接使用 Docker 安装 MySQL 和 Elasticsearch，如果你已经安装了 MySQL 和 Elasticsearch 可以直接跳过

1.安装 MySQL

在宿主机创建存放 MySQL 文件的目录
```shell
mkdir -p /usr/local/mysql
cd /usr/local/mysql
mkdir data conf log
```
增加 mysql 配置文件
```shell
cd /usr/local/mysql/conf
vi my.cnf
# 配置如下
[client]
default_character_set=utf8mb4
[mysql]
default_character_set=utf8mb4
[mysqld]
collation_server=utf8mb4_general_ci
character_set_server=utf8mb4
```

使用 Docker 安装 MySQL
```shell
mkdir -p /opt/mysql
cd /opt/mysql
vi docker-compose.yml   
```
内容如下
```yaml
version: '3'
services:
  mysql:
    restart: always
    privileged: true
    image: mysql:8.0.23
    container_name: mysql
    volumes:
      - /usr/local/mysql/data:/var/lib/mysql
      - /usr/local/mysql/conf:/etc/mysql/conf.d
      - /usr/local/mysql/log:/var/log/mysql
    environment:
      MYSQL_ROOT_PASSWORD: "123456"
    ports:
      - 3306:3306
```
启动 MySQL 
```shell
docker compose up -d
# 查看 MySQL
docker ps | grep myql
```
进入到 MySQL 容器内
```shell
docker exec -it mysql /bin/bash
# 登录 MySQL
mysql -u root -p
# 输入密码 123456

mysql> use mysql;
mysql> select user, host from user;
+------------------+-----------+
| user             | host      |
+------------------+-----------+
| root             | %         |
| mysql.infoschema | localhost |
| mysql.session    | localhost |
| mysql.sys        | localhost |
| root             | localhost |
+------------------+-----------+
5 rows in set (0.00 sec)
```
可以看到 root 用户的 `host` 是 `%`，那么我们可以通过远程访问我们刚刚安装好的 MySQL 了

2.安装 Elasticsearch 和 Kibana
这里我们使用 docker compose 安装
```shell
mkdir -p /opt/es
cd /opt/es
vi docker-compose.yml   
```
内容如下
```yaml
version: "3.7"
services:
  elasticsearch:
    image: "docker.elastic.co/elasticsearch/elasticsearch:7.10.2"
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
启动 Elasticsearch 和 Kibana
```shell
docker compose up -d
```
通过宿主机访问 Elasticsearch
http://192.168.56.102:9200
```md
{
  "name" : "es01",
  "cluster_name" : "mycluster",
  "cluster_uuid" : "b4NU0aJhSLqPEpqROyvy1w",
  "version" : {
    "number" : "7.10.2",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "747e1cc71def077253878a59143c1f785afa92b9",
    "build_date" : "2021-01-13T00:42:12.435326Z",
    "build_snapshot" : false,
    "lucene_version" : "8.7.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```
访问 Kibana http://192.168.56.102:5601
通过 Kibana 的 Dev Tools 可以操作 Elasticsearch

3.安装 Canal

