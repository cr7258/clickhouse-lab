# Elasticsearch VS ClickHouse

Clickhouse 是俄罗斯搜索巨头 Yandex 开发的完全列式存储计算的分析型数据库。ClickHouse 在这两年的 OLAP 领域中一直非常热门，国内互联网大厂都有大规模使用。

Elasticsearch 是一个近实时的分布式搜索分析引擎，它的底层存储完全构建在 Lucene 之上。简单来说是通过扩展 Lucene 的单机搜索能力，使其具有分布式的搜索和分析能力。 Elasticsearch 通常会和其它两个开源组件 Logstash（日志采集）和 Kibana（仪表盘）一起提供端到端的日志/搜索分析的功能，常常被简称为 ELK。

Elasticsearch 最擅长的主要是完全搜索场景（where 过滤后的记录数较少），在内存富裕运行环境下可以展现出非常出色的并发查询能力。但是在大规模数据的分析场景下（where 过滤后的记录数较多），ClickHouse 凭借极致的列存和向量化计算会有更加出色的并发表现，并且查询支持完备度也更好。

本次实验将测试 Elasticsearch 和 ClickHouse 对基本查询的性能差异。

## 测试架构

测试用到的所有组件都通过 Docker 容器的方式部署在 192.168.1.41 这台虚拟机上。
其中 Vector 负责产生数据并写入 Elasticsearch 和 ClickHouse，Kibana 和 TabixUI 提供了可视化的操作界面，Juypter 用于运行 Python 测试代码。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210722224237.png)

## 克隆代码

```sh
https://github.com/cr7258/clickhouse-lab
```
## 创建容器网络

创建一个 Docker 网络，本实验所有的容器都连接到该网络，容器之间可以通过容器名访问，Docker Embedded DNS 会负责容器名到 IP 地址的 DNS 解析。

```sh
docker network create esvsch
```
## 部署 Elasticsearch

通过 docker-compose 部署 Elasticsearch，为了方便操作同时部署了 Kibana。

```yaml
version: '3.7'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.4.0
    container_name: elasticsearch
    environment:
      - xpack.security.enabled=false
      - discovery.type=single-node
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    cap_add:
      - IPC_LOCK
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
      - 9300:9300
    deploy:
      resources:
        limits:
          cpus: '4'
          memory: 4096M
        reservations:
          memory: 4096M

  kibana:
    container_name: kibana
    image: docker.elastic.co/kibana/kibana:7.4.0
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - 5601:5601
    depends_on:
      - elasticsearch

volumes:
  elasticsearch-data:
    driver: local
networks:
  default:
    external:
      name: esvsch
```

启动 Elasticsearch 和 Kibana：

```sh
cd elastic
docker-compose up -d
```

浏览器输入 http://192.168.1.41:5601 访问 Kibana 界面，默认没有设置用户名密码，之后可以通过 Dev Tools 界面操作 Elasticsearch：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210722213351.png)



## 部署 ClickHouse

通过 docker-compsoe 部署 ClickHouse，为了方便操作同时部署了 TabixUI。

```sh
version: "3.7"
services:
  clickhouse:
    container_name: clickhouse
    image: yandex/clickhouse-server
    volumes:
      - ./data/config:/var/lib/clickhouse
    ports:
      - "8123:8123"
      - "9000:9000"
      - "9009:9009"
      - "9004:9004"
    ulimits:
      nproc: 65535
      nofile:
        soft: 262144
        hard: 262144
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "localhost:8123/ping"]
      interval: 30s
      timeout: 5s
      retries: 3
    deploy:
      resources:
        limits:
          cpus: '4'
          memory: 4096M
        reservations:
          memory: 4096M
  
  tabixui:
    container_name: tabixui
    image: spoonest/clickhouse-tabix-web-client
    ports:
      - "18080:80"
    depends_on:
      - clickhouse
    deploy:
      resources:
        limits:
          cpus: '0.1'
          memory: 128M
        reservations:
          memory: 128M
networks:
  default:
    external:
      name: esvsch
```

启动 ClickHouse 和 TabixUI：

```sh
cd clickhouse
docker-compose up -d
```

浏览器输入 http://192.168.1.41:18080 访问 TabixUI，创建一个新连接，自定义一个名字，连接 ClickHouse 的地址为 http://192.168.1.41:8123 ，用户名为 default，密码为空。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210722213535.png)


## 初始化表结构

Elasticsearch 有动态映射的功能，当遇到文档中以前未遇到的字段，Elasticsearch 可以通过动态映射确定字段的数据类型并自动把新的字段添加到类型映射。因此对于 Elasticseach 我们不需要事先创建索引。

在 ClickHouse 上我们需要事先创建好表结构：
```sql
CREATE TABLE default.syslog(
    application String,
    hostname String,
    message String,
    mid String,
    pid String,
    priority Int16,
    raw String,
    timestamp DateTime('UTC'),
    version Int16
) ENGINE = MergeTree()
    PARTITION BY toYYYYMMDD(timestamp)
    ORDER BY timestamp
    TTL timestamp + toIntervalMonth(1);

ALTER TABLE syslog DELETE where raw is not null
```

## 通过 Vector 将数导入 Elasticsearch 和 ClickHouse

 Vector 是一个轻量，超快和开源的可观察管道构建工具, 可以用于收集，转换和发送日志、指标、事件等内容。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210722214824.png)

我们使用 Vector 构建 10w 条 syslog 日志，然后分别输出到 Elasticsearch 和 ClickHouse。
Vector 的配置文件如下，我们启动 Vector 容器时会挂载该文件到容器的 /etc/vector/vector.toml。

```yaml
#生成 syslog 的模拟数据，生成 10w 条，生成间隔和 0.01 秒。
[sources.in]
  type = "generator"
  format = "syslog"
  interval = 0.01
  count = 100000

#把原始消息复制一份，这样抽取的信息同时可以保留原始消息。
[transforms.clone_message]
  type = "add_fields"
  inputs = ["in"]
  fields.raw = "{{ message }}"

#使用正则表达式，按照 syslog 的定义，抽取出 application，hostname，message，mid，pid，priority，timestamp，version 这几个字段。
[transforms.parser]
  # General
  type = "regex_parser"
  inputs = ["clone_message"]
  field = "message" # optional, default
  patterns = ['^<(?P<priority>\d*)>(?P<version>\d) (?P<timestamp>\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{3}Z) (?P<hostname>\w+\.\w+) (?P<application>\w+) (?P<pid>\d+) (?P<mid>ID\d+) - (?P<message>.*)$']

#数据类型转化
[transforms.coercer]
  type = "coercer"
  inputs = ["parser"]
  types.timestamp = "timestamp"
  types.version = "int"
  types.priority = "int"

#把生成的数据打印到控制台，供开发调试。
[sinks.out_console]
  # General
  type = "console"
  inputs = ["coercer"] 
  target = "stdout" 

  # Encoding
  encoding.codec = "json" 
  
#输入到 ClickHouse
[sinks.out_clickhouse]
  host = "http://clickhouse:8123"
  inputs = ["coercer"]
  table = "syslog"
  type = "clickhouse"
 
  encoding.only_fields = ["application", "hostname", "message", "mid", "pid", "priority", "raw", "timestamp", "version"]
  encoding.timestamp_format = "unix"

#输出到 Elasticsearch
[sinks.out_es]
  # General
  type = "elasticsearch"
  inputs = ["coercer"]
  compression = "none" 
  endpoint = "http://elasticsearch:9200" 
  index = "syslog-%F"

  # Encoding

  # Healthcheck
  healthcheck.enabled = true
```

启动 Vector：

```sh
cd vector
make start
```

## 启动 Jupyter Notebook

Jupyter Notebook 是一个开源的 Web 应用程序，允许用户创建和共享包含代码、方程式、可视化和文本的文档。

启动 Jupyter：

```sh
cd notebook
#构建镜像
make docker
#启动容器
make run
```
浏览器访问 http://192.168.1.41:8888 登录 Jupter。我事先准备在 Jupyter 上准备了 Python SDK 调用 Elaticsearch 和 ClickHouse 的代码，大家可以直接点击运行查看。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210722220557.png)

其中 Query Tester.ipynb 准备了 Elasticsearch 和 ClickHouse 性能对比的代码，Elasticsearch 使用 DSL 语言查询，ClickHouse 使用 SQL 语言查询。简单测试了一些常见的查询，每个查询语句分别在 Elasticsearch 和 ClickHouse 上运行 10 次。

查询所有记录：
```sh
#Elasticsearch
{
  "query":{
    "match_all":{}
  }
}

#ClickHouse 
SELECT * FROM syslog
```

匹配单个字段：
```sh
#Elasticsearch
{
  "query":{
    "match":{
      "hostname":"for.org"
    }
  }
}

#ClickHouse 
SELECT * FROM syslog WHERE hostname='for.org'
```

匹配多个字段：
```sh
#Elasticsearch
{
  "query":{
    "multi_match":{
      "query":"up.com ahmadajmi",
        "fields":[
          "hostname",
          "application"
        ]
    }
  }
}

#ClickHouse 
SELECT * FROM syslog WHERE hostname='for.org' OR application='ahmadajmi'
```

查找包含特定单词的字段：
```sh
#Elasticsearch
{
  "query":{
    "term":{
      "message":"pretty"
    }
  }
}

#ClickHouse 
SELECT * FROM syslog WHERE lowerUTF8(raw) LIKE '%pretty%'
```

范围查询，查找版本大于 2 的记录：
```sh
#Elasticsearch
{
  "query":{
    "range":{
      "version":{
        "gte":2
      }
    }
  }
}

#ClickHouse 
SELECT * FROM syslog WHERE version >= 2
```

查找到存在指定字段的记录：
```sh
#Elasticsearch
{
  "query":{
    "exists":{
      "field":"application"
    }
  }
}

#ClickHouse 
SELECT * FROM syslog WHERE application is not NULL
```

正则表达式查询，查询匹配某个正则表达式的数据：
```sh
#Elasticsearch
{
  "query":{
    "regexp":{
      "hostname":{
        "value":"up.*",
          "flags":"ALL",
            "max_determinized_states":10000,
              "rewrite":"constant_score"
      }
    }
  }
}

#ClickHouse 
SELECT * FROM syslog WHERE match(hostname, 'up.*')
```

统计某个字段出现的次数：
```sh
#Elasticsearch
{
  "aggs":{
    "version_count":{
      "value_count":{
        "field":"version"
      }
    }
  }
}

#ClickHouse 
SELECT count(version) FROM syslog
```

查找不重复的字段的个数：
```sh
#Elasticsearch
{
  "aggs":{
    "my-agg-name":{
      "cardinality":{
        "field":"priority"
      }
    }
  }
}

#ClickHouse 
SELECT count(distinct(priority)) FROM syslog
```

以上查询语句我们也可以在 Kibana 和 TabixUI 上运行一下看看效果：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210722222627.png)

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210722222609.png)

运行完 Query Tester.ipynb 脚本最后会输出统计图。

所有的查询的响应时间的分布：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210721165445.png)

总查询时间的对比：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210721165517.png)

通过测试数据我们可以看出 ClickHouse 在大部分的查询的性能上都明显要优于 Elasticsearch。在正则查询（Regex query）和单词查询（Term query）等搜索常见的场景下，也并不逊色。
在聚合场景下，ClickHouse 表现异常优秀，充分发挥了列存引擎的优势。


## 参考链接
* https://segmentfault.com/a/1190000039919389
* https://zhuanlan.zhihu.com/p/353296392
* https://github.com/gangtao/esvsch

## 欢迎关注
![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210306213609.png)
