---
title: ELK+Filebeat实现日志收集
date: 2021-09-24 20:39:00
tags: 运维
---
## 为什么用ELK
当前公司项目部署在十几台服务器上。 通过tail、grep等命令直接去每台服务器查日志很不方便。给开发团队每个成员维护日志查询账号也很麻烦。 希望能有一套集中的日志系统，方便开发人员查询日志，定位问题，也方便运维对异常日志进行监控。

<!--more-->

## ELK简介
ELK是三个开源软件的缩写，Elasticsearch , Logstash, Kibana， 除此之外也用到了Filebeat。  
1. Elasticsearch 搜索和分析引擎，用来存储日志  
2. Logstash 服务器端出具处理管道，用来转换数据存到Elasticsearch中  
3. Kibana 可视化界面，展示Elasticsearch中的数据  
4. Filebeat 日志采集器，部署在每台应用服务器上  

整体结构如下：

{% asset_img  elk.png elk image %}



## 安装与部署

### docker安装ELK镜像
#### ELK镜像安装启动
elk单独安装都比较复杂，所以直接用docker pull了一个别人安装好的镜像  
```shell
# 获取最新镜像
docker pull sebp/elk
# 启动镜像 d5e175b50ad8为镜像id, 服务器32G内存这里为es指定了16G堆内存
docker run -d -e ES_JAVA_OPTS="-Xms16G -Xmx16G" -p 5601:5601 -p 5044:5044 -p 9200:9200 -p 9300:9300 -it --restart=always --name elk d5e175b50ad8
```
如果出现报错,说明是es启动时要求max_map_count最低为262144
```shell
max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
```
修改此项配置，并写入配置文件
```shell
## 执行
sysctl -w vm.max_map_count=262144
```
编辑/etc/sysctl.conf，加入：vm.max_map_count=262144
#### 修改配置文件
elk容器运行后，进入容器修改logstash配置文件
```shell
## 进入容器
docker exec -it elk /bin/bash
## 编辑配置文件
vim /etc/logstash/conf.d/02-beats-input.conf 
```
注意/etc/logstash/conf.d/下还有其他配置文件，如果不用删掉即可。  
配置内容如下，主要是从filebeat中接收日志，格式化后输出到es中
```shell
# 监听5044端口接收filebeat的日志信息
input {  
  beats {
    port => "5044"
  }
}
# 日志过滤处理
filter {
  # 将message中信息的各个字段格式化出来，这个配置与应用服务器上输出的日志格式相匹配
  grok {
    match => ["message", "%{TIMESTAMP_ISO8601:timestamp}%{SPACE}\[%{NOTSPACE:thread}\]%{SPACE}\[%{NOTSPACE:file}\]%{SPACE}%{LOGLEVEL:level}%{SPACE}%{NOTSPACE:logger}%{SPACE}-%{SPACE}%{GREEDYDATA:msg}"]
  }
  # 将日志中timestamp转换成@timestamp，方便es按照时间保存
  date {
    match => ["timestamp", "yyyy-MM-dd HH:mm:ss.SSS"]
    target => "@timestamp"
    timezone => "Asia/Shanghai"
    remove_field => ["timestamp"]
  }
}
# 日志输出到ES，根据日志信息中的log_source不同环境输出到不同index中,这里分为test、prod
output {
  if [log_source] == "test" {
    elasticsearch {
      hosts => ["localhost:9200"]
      index => "xxx-test-log-%{+YYYY.MM.dd}"
    }
  }
  if [log_source] == "prod" {
    elasticsearch {
      hosts => ["localhost:9200"]
      index => "xxx-prod-log-%{+YYYY.MM.dd}"
    }
  }
}
```

### 应用服务器安装Filebeat

服务器系统为CentOS7，[参考官网yum方式安装](https://www.elastic.co/guide/en/beats/filebeat/current/setup-repositories.html),很简单  

修改filebeat.yml

```shell
filebeat.inputs:

- type: log
  enabled: true
  paths:
    - /opt/path1/xxx*.log # 监控的日志文件
  fields:
    log_source: test # 用于logstash处理时不同环境输出es不同index
    app: xxx
  fields_under_root: true
  multiline.pattern: ^(\d{4}|\d{2})\-(\d{2}|[a-zA-Z]{3})\-(\d{2}|\d{4})# 多行日志识别日期开头为一条日志
  multiline.negate: true
  multiline.match: after
- type: log
  enabled: true
  paths:
    - /opt/xxx/xxx*.log
  fields:
    log_source: test
    app: xxx
  fields_under_root: true
  multiline.pattern: ^(\d{4}|\d{2})\-(\d{2}|[a-zA-Z]{3})\-(\d{2}|\d{4})
  multiline.negate: true
  multiline.match: after

output.logstash:
  hosts: ["0.0.0.0:5044","0.0.0.0:5044"]# 输出到两台logstash
```

## 遇到的问题

### 选型时走的弯路

1. logback->logstash->elasticsearch：使用springboot+logback直接将日志通过tcp输出到logstash。这种方式会实时将日志通过tcp传输到logstash，依赖于logstash服务器的吞吐量、可用性。也会拖慢应用的执行
2. logback->rabbitmq->logstash->elasticsearch: 加了一层rabbitmqmq，让logstash可以异步处理日志。仍然会拖慢应用的执行速度，日志量大的时候发现JVM中输出日志到MQ的BlockingQueue中挤压了许多日志。造成堆内存过大

最开始不用filebeat的原因是认为每台应用服务器部署filebeat太麻烦，但是最终还是使用了filebeat->logstash->elasticsearch的方式。后面会考虑加一层kafka（filebeat不支持rabbitmq）,应用部署用docker（维护一套开发环境配置，部署弹性扩容都方便）。

### 遇到的问题

1. filebeat启动后不久就报错：too many open files.。 原因是第一次同步filebeat同时打开的日志文件太多，linux默认一个进程最多打开1024个文件，参考[ELK笔记](https://wiki.eryajf.net/pages/5016.html)改下配置即可
2. elasticsearch、logstash默认配置处理慢，GC频繁，每天100G+日志压不住。修改配置、JVM参数优化即可

