---
title: ELK日志收集部署
date: 2021-09-24 20:39:00
tags: 运维
---
## 为什么用ELK
当前公司项目部署在十几台服务器上。 通过tail、grep等命令直接去每台服务器查日志很不方便。给开发团队每个成员维护日志查询账号也很麻烦。 希望能有一套几种的日志查询系统，方便开发人员查询日志，定位问题，也方便运维对异常日志进行监控。

## ELK简介
ELK是三个开源软件的缩写，Elasticsearch , Logstash, Kibana， 除此之外也用到了Filebeat。  
1. Elasticsearch 搜索和分析引擎，用来存储日志  
2. Logstash 服务器端出具处理管道，用来转换数据存到Elasticsearch中  
3. Kibana 可视化界面，展示Elasticsearch中的数据  
4. Filebeat 日志采集器，部署在每台应用服务器上  
整体架构图如下



## 安装与部署

### docker安装ELK镜像
#### ELK镜像安装启动
elk单独安装都比较复杂，所以直接用docker pull了一个别人安装好的镜像  
```shell
# 获取最新镜像
docker pull sebp/elk
# 启动镜像 d5e175b50ad8为镜像id, 服务器32G内存这里为es指定了16G堆内存和G1垃圾收集器
docker run -d -e ES_JAVA_OPTS="-Xms16G -Xmx16G -XX:+UseG1GC" -p 5601:5601 -p 5044:5044 -p 9200:9200 -p 9300:9300 -it --restart=always --name elk d5e175b50ad8
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



## 踩过的坑
