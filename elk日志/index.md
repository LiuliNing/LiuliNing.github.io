# ELK日志框架集成




<!--more-->
# ELK日志框架集成

```
createBy jdzt.lln
createTime 2022-01-01
```

```
业务场景及需求说明
公司现在有内网虚拟机多台，分别部署了多个不同环境（测试环境、试用环境、演示环境等）的项目（地灾、岩土、地勘等），同时对接了多个外部服务（致远OA，在线预览、python算法等），在开发阶段处理不同环境出现的问题时，基于日志来定位问题十分关键，但现阶段对于日志管理还处于十分混乱的情况，没有进行统一的收集和处理，导致开发人员在定位和处理不同项目的不同环境的不同数据的相同问题时，定位和分析十分困难。
ELK即作为新框架EasyCloud的架构的重要组成部分，也为解决以上问题，具备了实际需求和场景。
```

```
ELK是由elasticsearch+Logstash+kibana三款开源软件组合而成的日志收集处理套件，其中Logstash负责日志收集，elasticsearch负责日志的搜索、统计，而kibana则是ES的展示。
使用版本信息如下：
Elasticsearch version 7.16.2
Logstash version 7.16.2
kibana version 7.16.2
```

```
参考文档
官网文档
https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html
https://www.elastic.co/guide/en/logstash/current/docker.html
博客
https://blog.csdn.net/shykevin/article/details/108251996
https://www.cnblogs.com/chinda/p/13125625.html
https://www.cnblogs.com/xiao987334176/p/13565468.html
https://blog.csdn.net/soulteary/article/details/105921729
```



## 一  Elasticsearch安装

## 1.1 Elasticsearch安装

### 1.1.1 docker

```
docker pull docker.elastic.co/elasticsearch/elasticsearch:7.16.2
```

```
docker run \
--name elk_es \
--privileged=true \
--restart=always \
-d \
-e ES_JAVA_OPTS="-Xms512m -Xmx512m" \
-e "discovery.type=single-node" \
-v /opt/elk/es/:/usr/elasticsearch/data \
-v /opt/elk/es/conf/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
-v /opt/elk/es/plugins:/usr/share/elasticsearch/plugins \
-p 9200:9200 \
-p 9300:9300 \
66c29cde15ce


docker run \
--name elk_es \
--privileged=true \
--restart=always \
-d \
-e ES_JAVA_OPTS="-Xms512m -Xmx512m" \
-e "discovery.type=single-node" \
-v /opt/elk/elasticsearch:/usr/share/elasticsearch \
-p 9200:9200 \
-p 9300:9300 \
elasticsearch:7.16.2

```

### 1.1.2 docker-compose

```
version: '3.7'
services:
  elasticsearch:
    container_name: es
    image: elasticsearch:7.16.2
    ports:
      - "9200:9200"
      - "9300:9300"
    volumes:
      - /opt/elk/es/conf/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - /opt/elk/es/data:/usr/share/elasticsearch/data
      - /opt/elk/es/plugins:/usr/share/elasticsearch/plugins
    environment:
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - "discovery.type=single-node"
      - "COMPOSE_PROJECT_NAME=es-server"
    restart: always
```

## 1.2 Elasticsearch-Head安装

### 1.2.1 docker

```
docker pull mobz/elasticsearch-head:5-alpine
```

```
docker run -d \
  --name=elasticsearch-head \
  --restart=always \
  -p 9100:9100 \
  docker.io/mobz/elasticsearch-head:5-alpine
```

### 1.2.2  docker-compose

```
version: '3.7'
services:
  elasticsearch-head:
    image: mobz/elasticsearch-head:5-alpine
    container_name: elk_es_head
    restart: always
    ports:
      - "9100:9100"
    networks:
      - elk
networks:
  elk:
    driver: bridge
```



## 1.3配置文件 

### elasticsearch.yml

```

## Default Elasticsearch configuration from elasticsearch-docker.
## from https://github.com/elastic/elasticsearch-docker/blob/master/build/elasticsearch/elasticsearch.yml
#
cluster.name: "docker-cluster"
network.host: 0.0.0.0

# minimum_master_nodes need to be explicitly set when bound on a public IP
# set to 1 to allow single node clusters
# Details: https://github.com/elastic/elasticsearch/pull/17288
discovery.zen.minimum_master_nodes: 1

## Use single node discovery in order to disable production mode and avoid bootstrap checks
## see https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html
#
discovery.type: single-node

## Disable X-Pack
## see https://www.elastic.co/guide/en/x-pack/current/xpack-settings.html
##     https://www.elastic.co/guide/en/x-pack/current/installing-xpack.html#xpack-enabling
#
xpack.security.enabled: false
xpack.monitoring.enabled: false
xpack.ml.enabled: false
xpack.graph.enabled: false
xpack.watcher.enabled: false

http.cors.enabled : true
http.cors.allow-origin : "*"
http.cors.allow-methods : OPTIONS, HEAD, GET, POST
http.cors.allow-headers : X-Requested-With, Content-Type, Content-Length

```

```
访问
curl http://127.0.0.1:9200/
返回
{
  "name" : "4a8f92502f14",
  "cluster_name" : "docker-cluster",
  "cluster_uuid" : "cRd8xKAcRyCEa5eO9kuntw",
  "version" : {
    "number" : "7.16.2",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "2b937c44140b6559905130a8650c64dbd0879cfb",
    "build_date" : "2021-12-18T19:42:46.604893745Z",
    "build_snapshot" : false,
    "lucene_version" : "8.10.1",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
安装成功
```

## 二 Logstash安装

### 2.1 docker

```
docker pull logstash:7.16.2

docker run -d \
--name=elk_logstash \
-p 5045:5045 \
--restart=always \
-v /opt/elk/logstash:/usr/share/logstash \
-v /www/wwwroot/plmlogs:/www/wwwroot/plmlogs \
logstash:7.16.2


docker run -d \
--name=elk_logstash \
-p 5045:5045 \
--restart=always \
-v /opt/elk/logstash:/usr/share/logstash \
-v /www/wwwroot/plmLogs:/www/wwwroot/plmLogs \
logstash:7.16.2
```

### 2.2  docker-compos 

```
version: '3.7'
services:
  logstash:
    image: logstash:7.16.2
    container_name: elk_logstash
    restart: always
    volumes:
      - /opt/elk/logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml \
      - /opt/elk/logstash/config/conf.d/:/opt/elk/logstash/config/conf.d/ \
      - /opt/elk/logstash/data:/usr/share/logstash/data \
      - /opt/elk/logstash/pipeline:/usr/share/logstash/pipeline \
      - /opt/elk/logstash/logs:/opt/elk/logstash/logs \
      - /www/wwwroot/plmlogs:/www/wwwroot/plmlogs \
    environment:
      LS_JAVA_OPTS: "-Xmx512m -Xms512m"
    networks:
      - elk
networks:
  elk:
    driver: bridge

```

### 2.3 配置文件

#### 主配置文件

```
http.host: "0.0.0.0"
xpack.monitoring.elasticsearch.hosts: [ "http://192.168.23.128:9200" ]
path.config: /opt/elk/logstash/config/conf.d/*.conf
path.logs: /opt/elk/logstash/logs
```

#### 日志conf文件配置（示例）

```
input {
  file {
    #标签
    type => "systemlog-localhost"
    #采集点
    path => "/opt/plmLogs/"
    #开始收集点
    start_position => "beginning"
    #扫描间隔时间，默认是1s，建议5s
    stat_interval => "5"
  }
}

output {
  elasticsearch {
    hosts => ["127.0.0.1:9200"]
    index => "logstash-system-localhost-%{+YYYY.MM.dd}"
 }
}
```

## 三 kibana安装

### 3.1 docker 

```
docker pull kibana:7.16.2

docker run -d \
--name=elk_kibana \
--privileged=true \
-p 5601:5601 \
--restart=always \
-v /opt/elk/kibana/config/kibana.yml:/usr/share/kibana/config/kibana.yml \
kibana:7.16.2
```

### 3.2 docker-compose

```
version: '3.7'
services:
  kibana:
    image: kibana:7.16.2
    container_name: elk_kibana
    restart: always
    volumes:
      - /opt/elk/kibana/config/kibana.yml:/usr/share/kibana/config/kibana.yml
    ports:
      - "5601:5601"
    networks:
      - elk
networks:
  elk:
    driver: bridge

```

### 3.3  配置文件

```

```



## 四 合并docker-compose文件

```
version: '3.7'
services:
  elasticsearch:
    image: elasticsearch:7.16.2
    container_name: elk_es
    restart: always
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      ES_JAVA_OPTS: "-Xmx512m -Xms512m"
      xpack.security.enabled: "false"
      xpack.monitoring.enabled: "false"
      xpack.graph.enabled: "false"
      xpack.watcher.enabled: "false"
    networks:
      - elk
    volumes:
      - /opt/elk/es/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml
      - /opt/elk/es/data:/usr/share/elasticsearch/data
      - /opt/elk/es/plugins:/usr/share/elasticsearch/plugins
  elasticsearch-head:
    image: mobz/elasticsearch-head:5-alpine
    container_name: elk_es_head
    restart: always
    ports:
      - "9100:9100"
    networks:
      - elk
  logstash:
    image: logstash:7.16.2
    container_name: elk_logstash
    restart: always
    volumes:
      - /opt/elk/logstash/config/logstash.yml:/usr/share/logstash/config/logstash.yml \
      - /opt/elk/logstash/config/conf.d/:/opt/elk/logstash/config/conf.d/ \
      - /opt/elk/logstash/data:/usr/share/logstash/data \
      - /opt/elk/logstash/pipeline:/usr/share/logstash/pipeline \
      - /opt/elk/logstash/logs:/opt/elk/logstash/logs \
      - /www/wwwroot/plmlogs:/www/wwwroot/plmlogs \
    environment:
      LS_JAVA_OPTS: "-Xmx512m -Xms512m"
    networks:
      - elk
    depends_on:
      - elasticsearch
  kibana:
    image: kibana:7.16.2
    container_name: elk_kibana
    restart: always
    volumes:
      - /opt/elk/kibana/config/kibana.yml:/usr/share/kibana/config/kibana.yml
    ports:
      - "5601:5601"
    networks:
      - elk
    depends_on:
      - elasticsearch
networks:
  elk:
    driver: bridge

```



## 五 安装时异常处理记录

```
# 启动报错1
bootstrap checks failed. You must address the points described in the following [1] lines before starting Elasticsearch.
areas vm.max_map_count [65530] is too low, increase to at least [262144]
解决方案
https://blog.csdn.net/weixin_44186547/article/details/111474430
启动用户内存设置问题
sysctl -w vm.max_map_count=262144
sysctl -a|grep vm.max_map_count

#启动报错2
{"@timestamp":"2022-02-24T06:13:29.287Z", "log.level": "WARN", "message":"received plaintext http traffic on an https channel, closing connection Netty4HttpChannel{localAddress=/172.21.0.2:9200, remoteAddress=/192.168.23.1:52180}", "ecs.version": "1.2.0","service.name":"ES_ECS","event.dataset":"elasticsearch.server","process.thread.name":"elasticsearch[6c35a2a32c9e][transport_worker][T#7]","log.logger":"org.elasticsearch.xpack.security.transport.netty4.SecurityNetty4HttpServerTransport","elasticsearch.cluster.uuid":"VUG2U5RzQBisYw2XewbopA","elasticsearch.node.id":"Z-IvopcWQfGf-05cDPdlLA","elasticsearch.node.name":"6c35a2a32c9e","elasticsearch.cluster.name":"docker-cluster"}

```



```
# kibana启动异常报错
FATAL  Error: TaskManager is unable to start as Kibana has no valid UUID assigned to it.
Unable to write to UUID file at /usr/share/kibana/data/uuid. Ensure Kibana has sufficient permissions to read / write to this file.  Error was: ENOSPC
解决方案
权限不足，修改映射宿主机目录权限为 777

# 启动找不到es地址
Unable to retrieve version information from Elasticsearch nodes.
解决方案
配置elasticsearch.yml和kibana.yml，修改es的ip为docker0 的ip，不是宿主机ip

# 无法启动
server is not ready yet
解决方案
映射配置文件位置异常，重新核对解决。

# logstash 启动报错
/usr/local/bin/docker-entrypoint: line 12: exec: logstash: not found
解决方案
权限不足，修改映射宿主机目录权限为 777


```


