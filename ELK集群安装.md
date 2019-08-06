# ELK集群搭建
## ELK是什么？
**ELK**是三个开源软件的缩写，分别表示：Elasticsearch , Logstash, Kibana，也可以指elk技术栈，包含一系列的组件。

**Elasticsearch**是一个分布式、高扩展、高实时的搜索与数据分析引擎。它能很方便的使大量数据具有搜索、分析和探索的能力。充分利用ElasticSearch的水平伸缩性，能使数据在生产环境变得更有价值。ElasticSearch 的实现原理主要分为以下几个步骤，首先用户将数据提交到Elastic Search 数据库中，再通过分词控制器去将对应的语句分词，将其权重和分词结果一并存入数据，当用户搜索数据时候，再根据权重将结果排名，打分，再将返回结果呈现给用户。它的特点有：分布式，零配置，自动发现，索引自动分片，索引副本机制，restful风格接口，多数据源，自动搜索负载等。

**Logstash**是开源的服务器端数据处理管道，能够同时从多个来源采集数据，转换数据，然后将数据发送到您最喜欢的“存储库”中。一般用在日志的搜集、分析、过滤，支持大量的数据获取方式。

**Kibana** 可以对 Elasticsearch 进行可视化，还可以在 Elastic Stack 中进行导航，这样便可以进行各种操作了，从跟踪查询负载，到理解请求如何流经您的整个应用，都能轻松完成。权限管理依赖收费授权的x-pack组件，若无权限管理，则整个es数据内容对素有用户可见可操作，存在安全风险，若不想购买许可证可以考虑功能强大的[Grafana](https://grafana.com/)替代。

**Filebeat**隶属于**Beats**。目前Beats包含四种工具：
1. Packetbeat（收集网络流量数据）
2. Topbeat（收集系统、进程和文件系统级别的 CPU 和内存使用情况等数据）
3. Filebeat（收集文件数据）
4. Winlogbeat（搜集 Windows 事件日志数据）
5. heartbeat（用于系统或者应用监控）


**官方文档地址**[https://www.elastic.co/guide/index.html](https://www.elastic.co/guide/index.html)
**官方下载地址**[https://www.elastic.co/cn/downloads/](https://www.elastic.co/cn/downloads/)

## 集群设计
本文集群基于elasticsearch 7.2.0 组件实现，并作为笔者工作所设计系统的的一个组成部分，包括了[elasticsearch](https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.3.0-linux-x86_64.tar.gz)、[logstash](https://artifacts.elastic.co/downloads/logstash/logstash-7.3.0.tar.gz)、[kibama](https://artifacts.elastic.co/downloads/kibana/kibana-7.3.0-linux-x86_64.tar.gz)、[filebeat](https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.3.0-linux-x86_64.tar.gz)、[elasticsearch-head插件](https://github.com/mobz/elasticsearch-head)、[中文分词插件IK](https://github.com/medcl/elasticsearch-analysis-ik)以及[kafka](https://kafka.apache.org/downloads)，ELK7版本较之前版本主要配置有些变化，为避免版本不一致踩坑付出不必要学习成本，请尽量保持版本一致性，熟悉后可查询官方文档使用最新版。本文档只做集群安装配置说明，组件相关更多功能和配置后期有空会增加系列文章，有兴趣同学可以先自行查阅官方文档说明。

### 总体架构
系统总体数据流如下图，其中agent使用了filebeat，用来搜集处理nginx反向代理服务的日志以及WEB应用日志，数据搜集后统一发送给kafka集群，其他组件可以消费原始数据，也可以走logstash->elasticwearch进行简单的日志归集与统计分析
![](https://i.imgur.com/c3h10Or.png)


## Nginx
### 格式化nginx access日志
为方便处理数据，将相关[Nginx](http://nginx.org/en/docs/)日志格式化为json格式，减少后期转换开销，比这nginx使用的淘宝[Tegine](http://tengine.taobao.org/index_cn.html)版本，可能部分字段没有，没有的字段值若未加引号，会导致logstash json过滤器处理异常，请注意。nginx日志字段及格式语法可参见官方文档[http://nginx.org/en/docs/http/ngx_http_log_module.html](http://nginx.org/en/docs/http/ngx_http_log_module.html)。另外filebeat提供了nginx等众多组件的[官方模块](https://www.elastic.co/guide/en/beats/filebeat/7.2/configuration-filebeat-modules.html)，启用后可以快速配置nginx的模块处理，本文档未使用官方模块，为自定义处理方式。
```
    log_format main  '{"bytes_sent":$bytes_sent,'
#      '"content_length": $content_length,'
#      '"content_type": "$content_type",'
      '"http_x_forwarded_for": "$http_x_forwarded_for",'
      '"http_referer": "$http_referer",'
      '"http_user_agent": "$http_user_agent",'
#      '"document_root": "$document_root",'
      '"document_uri": "$document_uri",'
      '"host": "$host",'
#      '"hostname": "$hostname",'
      '"pid": $pid,'
      '"proxy_protocol_addr": "$proxy_protocol_addr",'
#      '"proxy_protocol_port": $proxy_protocol_port,'
#      '"realpath_root": "$realpath_root",'
      '"remote_addr": "$remote_addr",'
      '"remote_port": "$remote_port",'
      '"remote_user": "$remote_user",'
      '"request": "$request",'
      '"request_filename": "$request_filename",'
#      '"request_id": "$request_id",'
      '"request_length": $request_length,'
      '"request_method": "$request_method",'
      '"request_time": "$request_time",'
      '"request_uri": "$request_uri",'
      '"scheme": "$scheme",'
      '"sent_http_name": "$sent_http_name",'
      '"server_addr": "$server_addr",'
      '"server_name": "$server_name",'
      '"server_port": $server_port,'
      '"server_protocol": "$server_protocol",'
      '"status": "$status",'
      '"time_iso8601": "$time_iso8601",'
      '"time_local": "$time_local",'
      '"upstream_addr": "$upstream_addr",'
#      '"upstream_bytes_received": $upstream_bytes_received,'
      '"upstream_cache_status": "$upstream_cache_status",'
      '"upstream_http_name": "$upstream_http_name",'
      '"upstream_response_length": "$upstream_response_length",'
      '"upstream_response_time": "$upstream_response_time",'
      '"upstream_status": "$upstream_status"}';
```


## filebeat
### 安装与部署
1. 下载并解压到指定目录
2. 创建data目录
3. 编辑配置文件
4. 启动filebeat
5. 停止filebeat

```bash
#下载并解压到指定目录
tar -zxvf filebeat-7.2.0-linux-x86_64.tar.gz -C /usr/local/elk/
#创建data目录
cd /usr/local/elk/filebeat-7.2.0-linux-x86_64
mkdir data
#编辑配置文件
vim filebeat.yml
#启动filebeat
nohup filebeat -e >>/dev/null 2>&1 &
#停止filebeat
ps -ef | grep "filebeat"
kill -QUIT 进程号
 
```
### 配置
以下展示出的默认文件中增加的配置，这里output输出使用的kafka，请注释掉输出到其他组件的配置
```yml
#=========================== Filebeat inputs =============================

filebeat.inputs:

# Each - is an input. Most options can be set at the input level, so
# you can use different inputs for various configurations.
# Below are the input specific configurations.

- type: log

  # Change to true to enable this input configuration.
  enabled: true

  # Paths that should be crawled and fetched. Glob based paths.
  # 可配置多个路径
  paths:
    - /home/elk/logs/nginx/access*.log
  
  # 以下是filebeat中自定义字段，方便后期区分数据进行进一步处理  
  fields:
    ServerIp: 10.11.48.160
    ApplicationId: elk-global-nginx
    ApplicationDescribe: elk-global-nginx
    LogType: "access"
    LogLabel: "access"
    
- type: log

  # Change to true to enable this input configuration.
  enabled: true

  # Paths that should be crawled and fetched. Glob based paths.
  paths:
    - /home/elk/logs/nginx/error*.log
    
  fields:
    ServerIp: 10.11.48.160
    ApplicationId: elk-global-nginx
    ApplicationDescribe: elk-global-nginx
    LogType: "error"
    LogLabel: "error"


- type: log

  # Change to true to enable this input configuration.
  enabled: true

  # Paths that should be crawled and fetched. Glob based paths.
  paths:
    - /home/elk/logs/ConsoleGlobal.*
    
  fields:
    ServerIp: 10.11.48.160
    ApplicationId: elk-global-console
    ApplicationDescribe: elk-global-console
    LogType: "server"
    LogLabel: "server"

  # filebeat读取日志内容是按行读取的，一般日志都是按行打印，但是可能存在类似java异常信息多行情况就需要对多行特殊处理    
  multiline:
      pattern: '^[[:space:]]+(at|\.{3})\b|^Caused by:'
      negate: false
      match: after

- type: log

  # Change to true to enable this input configuration.
  enabled: true

  # Paths that should be crawled and fetched. Glob based paths.
  paths:
    - /home/elk/logs/ELKLog.*
    
  fields:
    ServerIp: 10.11.48.160
    ApplicationId: ELK-global-main
    ApplicationDescribe: ELK-global-main
    LogType: "server"
    LogLabel: "server"
    
  multiline:
      pattern: '^[[:space:]]+(at|\.{3})\b|^Caused by:'
      negate: false
      match: after

- type: log

  # Change to true to enable this input configuration.
  enabled: true

  # Paths that should be crawled and fetched. Glob based paths.
  paths:
    - /home/ELK/logs/test/*.log
    
  fields:
    ServerIp: 10.11.48.160
    ApplicationId: ELK-test
    ApplicationDescribe: ELK-test
    LogType: "server"
    LogLabel: "server"


#----------------------------- kafka output -----------------------------------
output.kafka:
  enabled: true
  # initial brokers for reading cluster metadata
  hosts: ["10.11.48.160:9092", "10.11.48.161:9092", "10.11.48.165:9092"]

  # message topic selection + partitioning
  topic: '%{[fields][ApplicationId]}-%{[fields][LogType]}'
  partition.round_robin:
    reachable_only: false

  compression: gzip
```


## zookeeper
kafka依赖zookeeper，安装kafka前需先安装配置zookeeper集群
1. 下载zookeeper：[https://zookeeper.apache.org/releases.html](https://zookeeper.apache.org/releases.html)
2. 配置zookeeper
```
#zk存放数据的目录，zk 需要有一个叫做myid的文件也是放到（必须）这个目录下,集群环境不得重复

dataDir=/usr/local/elk/apache-zookeeper-3.5.5/data 

dataLogDir=/usr/local/elk/apache-zookeeper-3.5.5/logs

clientPort=2181

#最大客户端连接数

maxClientCnxns=20

#是作为Zookeeper服务器之间或客户端与服务器之间维持心跳的时间间隔

tickTime=2000

#此配置表示，允许follower(相对于Leaderer言的“客户端”)连接并同步到Leader的初始化连接时间，以tickTime为单位。当初始化连接时间超过该值，则表示连接失败。

initLimit=10

#此配置项表示Leader与Follower之间发送消息时，请求和应答时间长度。如果follower在设置时间内不能与leader通信，那么此follower将会被丢弃。

syncLimit=5

#server.myid=ip:followers_connect to the leader:leader_election # server 是固定的，myid 是需要手动分配，第一个端口是follower是链接到leader的端口，第二个是用来选举leader 用的port

server.1=10.11.48.160:2888:3888

server.2=10.11.48.161:2888:3888

server.3=10.11.48.165:2888:3888
```
3. 配置环境变量(~/.bash_profile)
```
ZOOKEEPER_HOME=/usr/local/elk/apache-zookeeper-3.5.5
PATH=$HOME/.local/bin:$HOME/bin:$ZOOKEEPER_HOME/bin:$PATH
export  ZOOKEEPER_HOME PATH
```

### 操作步骤
```bash
tar -zxvf apache-zookeeper-3.5.5.tar.gz -C /usr/local/elk/
mkdir data
mkdir logs
cd data
#创建myid文件变设置编号，集群中myid不得重复
touch myid
vim conf/zoo.cfg
./zkServer.sh start
./zkServer.sh status
./zkServer.sh stop
```








