# Filebeat——读取json行文件的方式收集日志

## Filebeat安装
### 下载页面
https://www.elastic.co/cn/downloads/past-releases/filebeat-6-4-2
go语言写的
### 启动命令
前台启动  
```shell
filebeat -e -c filebeat.yml
```
后台启动
```bash
nohup ./filebeat -c filebeat.yml >> filebeat.out 2>&1 &
```

### 配置示例
```yaml
filebeat.inputs:       #日志文件输入设置
  - type: log          #文件的输入类型
    enabled: true      #开启服务
    paths:             #日志文件路径
      - "/home/avatar/luna/logs/**/*filebeat*.log"
    json.keys_under_root: true    # 默认情况下，解码的 JSON 位于“json”键下 在输出文档中。如果启用此设置，则密钥将复制到顶部 输出文档中的级别。默认值为 false
    json.overwrite_keys: true     # 如果启用了 和 此设置，则 解码的 JSON 对象中的值会覆盖 Filebeat 的字段 通常在发生冲突时添加（类型、源、偏移量等）  https://www.5axxw.com/questions/simple/vxbvrj
setup.ilm.enabled: false          # 索引生命周期 ilm 功能默认开启，开启情况下索引名称只能为 filebeat-*
setup.template.name: "log"        # 定义模板名称
setup.template.pattern: "log-*"   # 定义模板的匹配索引名称
setup.template.append_fields:     #  要添加到模板和kibana索引模式的字段列表，此设置添加新字段。它不会覆盖或更改现有字段  
  - name: stack_trace
    type: text
  - name: thread_name
    type: text
  - name: logger_name
    type: text
  - name: level
    type: text
  - name: appname
    type: text
  - name: TraceId
    type: text
  - name: LUNA_TID
    type: text
setup.kibana:
  host: "10.60.44.51:5601"
output.elasticsearch:
  hosts: ["10.60.44.51:9200"]
  username: "elastic"
  password: "5PdOsAFf7GdFnzYsIrgu"
  index: "log-fg-%{+yyyy.MM.dd}" #索引名称
xpack.monitoring.enabled: true   # 启动 xpack,基础安全免费
```

说明：
>- filebeat.inputs paths 配置filebeat打印格式的日志位置，可以用通配符
>- setup.* 里面的配置无所谓，用不到，但没这些会启动报错
>- output.elasticsearch 里面配置es的地址、要建立的索引名通配符


## java端logback的配置
1. [这个版本的github地址，java版本要求1.7以上](https://github.com/logstash/logstash-logback-encoder/tree/logstash-logback-encoder-5.3)

### `pom.xml`
添加`LogstashEncoder`的引用
```xml
<dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
    <version>5.3</version>
</dependency>
```

### `logback.xml`
配置文件打印的encoder为`net.logstash.logback.encoder.LogstashEncoder`

#### logback 在 1.2及以上的版本
比如，新的用java8的tfp、luna、rh2项目。
```xml
<appender name="ROLLINGFILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
        <level>TRACE</level>
    </filter>
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <fileNamePattern>${logback.path}/%d{yyyy-MM-dd}/taskplatform.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
        <!--日志大小-->
        <maxFileSize>100MB</maxFileSize>
        <!--日志文件保留天数-->
        <maxHistory>90</maxHistory>
    </rollingPolicy>
    <encoder charset="UTF-8" class="net.logstash.logback.encoder.LogstashEncoder"/>
</appender>
<!-- 异步输出 -->
<appender name="ASYNC_ROLLINGFILE" class="ch.qos.logback.classic.AsyncAppender">
    <!-- 不丢失日志.默认的,如果队列的80%已满,则会丢弃TRACT、DEBUG、INFO级别的日志 -->
    <discardingThreshold>0</discardingThreshold>
    <!-- 更改默认的队列的深度,该值会影响性能.默认值为256 -->
    <queueSize>2048</queueSize>
    <includeCallerData>false</includeCallerData>
    <!-- 添加附加的appender,最多只能添加一个 -->
    <appender-ref ref="ROLLINGFILE"/>
</appender>
```
#### logback 在 1.1 版本的项目
比如使用java7的ats项目。
在`logback.xml`添加这个两个appender以后，在`<root></root>`标签添加appender `async_logstash`。
```xml
<appender name="logstash"
          class="ch.qos.logback.core.rolling.RollingFileAppender">

    <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
        <level>TRACE</level>
    </filter>

    <!-- 可让每天产生一个日志文件，最多 30 个，自动回滚 -->
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
        <fileNamePattern>../log/filebeat-%d{yyyy-MM-dd}.json.log</fileNamePattern>
        <maxHistory>30</maxHistory>
    </rollingPolicy>

    <encoder charset="UTF-8" class="net.logstash.logback.encoder.LogstashEncoder"/>
</appender>
<!-- 异步输出 -->
<appender name="async_logstash" class="ch.qos.logback.classic.AsyncAppender">
    <!-- 不丢失日志.默认的,如果队列的80%已满,则会丢弃TRACT、DEBUG、INFO级别的日志 -->
    <discardingThreshold>0</discardingThreshold>
    <!-- 更改默认的队列的深度,该值会影响性能.默认值为256 -->
    <queueSize>2048</queueSize>
    <includeCallerData>false</includeCallerData>
    <!-- 添加附加的appender,最多只能添加一个 -->
    <appender-ref ref="logstash"/>
</appender>
```
### 打印出来的日志内容示例
每行是标准的`elasticsearch`能够识别的json格式
```
{"@timestamp":"2019-04-19T20:01:19.447+08:00","@version":"1","message":"using  logger:  com.alibaba.dubbo.common.logger.slf4j.Slf4jLoggerAdapter","logger_name":"com.alibaba.dubbo.common.logger.LoggerFactory","thread_name":"main","level":"INFO","level_value":20000,"application":"essen-prod-user-management-platform","TraceId":"N/A"}
{"@timestamp":"2019-04-19T20:01:19.656+08:00","@version":"1","message":"  [DUBBO]  Use  container  type([spring])  to  run  dubbo  serivce.,  dubbo  version:  2.6.5,  current  host:  10.244.4.57","logger_name":"com.alibaba.dubbo.container.Main","thread_name":"main","level":"INFO","level_value":20000,"application":"essen-prod-user-management-platform","TraceId":"N/A"}
{"@timestamp":"2019-04-19T20:01:26.921+08:00","@version":"1","message":"{dataSource-1}  inited","logger_name":"com.alibaba.druid.pool.DruidDataSource","thread_name":"main","level":"INFO","level_value":20000,"application":"essen-prod-user-management-platform","TraceId":"N/A"}
{"@timestamp":"2019-04-19T20:01:28.720+08:00","@version":"1","message":"Redisson  3.10.3","logger_name":"org.redisson.Version","thread_name":"main","level":"INFO","level_value":20000,"application":"essen-prod-user-management-platform","TraceId":"N/A"}
{"@timestamp":"2019-04-19T20:01:29.639+08:00","@version":"1","message":"Redis  cluster  nodes  configuration  got  from  redis-app-0.redis-service/10.244.1.2:6379:\n25a7add7323cbc629c9dda00f10d3a323ed208b3  10.244.1.8:6379@16379  slave  1ef25440d3ba9bc978ec13a4b9bcc36c54561c25  0  1555675289538  7  connected\na7ef2299b1d4d3510924a3de4cc0cd5f3323cff9  10.244.1.2:6379@16379  myself,slave  4093269f2e8bad0e5a501737b10f6736750248fa  0  1555675288000  1  connected\n4093269f2e8bad0e5a501737b10f6736750248fa  10.244.0.47:6379@16379  master  -  0  1555675289539  6  connected  0-5460\n1ef25440d3ba9bc978ec13a4b9bcc36c54561c25  10.244.0.48:6379@16379  master  -  0  1555675289000  7  connected  10922-16383\n16ac3f87550360d2bf0a2e312a72d63085c41b17  10.244.0.46:6379@16379  master  -  0  1555675288036  3  connected  5461-10921\ne7f0e179c01c4113e3547a70620030797733ff54  10.244.1.225:6379@16379  slave  16ac3f87550360d2bf0a2e312a72d63085c41b17  0  1555675289000  3  connected\n","logger_name":"org.redisson.cluster.ClusterConnectionManager","thread_name":"main","level":"INFO","level_value":20000,"application":"essen-prod-user-management-platform","TraceId":"N/A"}
{"@timestamp":"2019-04-19T20:01:29.770+08:00","@version":"1","message":"1  connections  initialized  for  10.244.0.46/10.244.0.46:6379","logger_name":"org.redisson.connection.pool.MasterPubSubConnectionPool","thread_name":"redisson-netty-1-8","level":"INFO","level_value":20000,"application":"essen-prod-user-management-platform","TraceId":"N/A"}
{"@timestamp":"2019-04-19T20:01:29.777+08:00","@version":"1","message":"1  connections  initialized  for  10.244.0.48/10.244.0.48:6379","logger_name":"org.redisson.connection.pool.MasterPubSubConnectionPool","thread_name":"redisson-netty-1-30","level":"INFO","level_value":20000,"application":"essen-prod-user-management-platform","TraceI d":"N/A"}
{"@timestamp":"2019-04-19T20:01:29.784+08:00","@version":"1","message":"1 connections initialized for 10.244.0.47/10.244.0.47:6379","logger_name":"org.redisson.connection.pool.MasterPubSubConnectionPool","thread_name":"redisson-netty-1-6","level":"INFO","level_value":20000,"application":"essen-prod-user-management-platform","TraceId":"N/A"}
{"@timestamp":"2019-04-19T20:01:29.797+08:00","@version":"1","message":"master: redis://10.244.0.46:6379 added for slot ranges: [[5461-10921]]","logger_name":"org.redisson.cluster.ClusterConnectionManager","thread_name":"redisson-netty-1-22","level":"INFO","level_value":20000,"application":"essen-prod-user-management-platform","TraceId":"N/A"}
{"@timestamp":"2019-04-19T20:01:29.798+08:00","@version":"1","message":"32 connections initialized for 10.244.0.46/10.244.0.46:6379","logger_name":"org.redisson.connection.pool.MasterConnectionPool","thread_name":"redisson-netty-1-22","level":"INFO","level_value":20000,"application":"essen-prod-user-management-platform","TraceId":"N/A"}
{"@timestamp":"2019-04-19T20:01:29.801+08:00","@version":"1","message":"master: redis://10.244.0.47:6379 added for slot ranges: [[0-5460]]","logger_name":"org.redisson.cluster.ClusterConnectionManager","thread_name":"redisson-netty-1-25","level":"INFO","level_value":20000,"application":"essen-prod-user-management-platform","TraceId":"N/A"}
{"@timestamp":"2019-04-19T20:01:29.802+08:00","@version":"1","message":"master: redis://10.244.0.48:6379 added for slot ranges: [[10922-16383]]","logger_name":"org.redisson.cluster.ClusterConnectionManager","thread_name":"redisson-netty-1-23","level":"INFO","level_value":20000,"application":"essen-prod-user-management-platform","TraceId":"N/A"}
{"@timestamp":"2019-04-19T20:01:29.802+08:00","@version":"1","message":"32 connections initialized for 10.244.0.47/10.244.0.47:6379","logger_name":"org.redisson.connection.pool.MasterConnectionPool","thread_name":"redisson-netty-1-25","level":"INFO","level_value":20000,"application":"essen-prod-user-management-platform","TraceId":"N/A"}
{"@timestamp":"2019-04-19T20:01:29.803+08:00","@version":"1","message":"32 connections initialized for 10.244.0.48/10.244.0.48:6379","logger_name":"org.redisson.connection.pool.MasterConnectionPool","thread_name":"redisson-netty-1-23","level":"INFO","level_value":20000,"application":"essen-prod-user-management-platform","TraceId":"N/A"}
{"@timestamp":"2019-04-19T20:01:30.502+08:00","@version":"1","message":" [DUBBO] No spring extension (bean) named:level, try to find an extension (bean) of type com.alibaba.dubbo.common.logger.Level, dubbo version: 2.6.5, current host: 10.244.4.57","logger_name":"com.alibaba.dubbo.config.spring.extension.SpringExtensionFactory","thread_name":"main","level":"WARN","level_value":30000,"application":"essen-prod-user-management-platform","TraceId":"N/A"}
```

## 使用`Filebeat`替代`Logstash`的原因

### 融汇二代遇到的问题
logback的logstash插件是异步通过网络发送的，用的是一个[环状队列](http://ifeve.com/disruptor/)做缓存还没发送的日志。
其本意是为了提升并发性能，但在融汇二代在做压测的时候，出现了内存泄漏问题，这个插件占用大量的内存不能释放，导致整个应用的性能急剧下降，所以改用其他方案。

![img](./images/Disruptor-300x144.png)

### 一份压测结果
内容源自博文：[从ELK到EFK演进](https://www.cnblogs.com/tylercao/p/7803520.html)

#### 压测环境

- 虚拟机 8 cores 64G内存 540G SATA盘
- Logstash 版本 2.3.1
- Filebeat 版本 5.5.0

#### 压测方案

Logstash / Filebeat 读取 350W 条日志 到 console，单行数据 580B，8个进程写入采集文件

#### 压测结果

| 项目     | workers | cpu usr | 总共耗时 | 收集速度    |
| -------- | ------- | ------- | -------- | ----------- |
| Logstash | 8       | 53.7%   | 210s     | 1.6w line/s |
| Filebeat | 8       | 38.0%   | 30s      | 11w line/s  |

Filebeat 所消耗的CPU只有 Logstash 的70%，但收集速度为 Logstash 的7倍。从我们的应用实践来看，Filebeat 确实用较低的成本和稳定的服务质量，解决了 Logstash 的资源消耗问题。

#### 与k8s集成

由于filebeat对资源的低消耗，可以作为容器的sidebar方式，与容器绑定部署（这个是我最早的部署方式）。


## 官方详细配置参考
- [filebeat-input-log 官方文档](https://www.elastic.co/guide/en/beats/filebeat/6.4/filebeat-input-log.html)
- [elasticsearch-output 官方文档](https://www.elastic.co/guide/en/beats/filebeat/6.4/elasticsearch-output.html)
- [开启filebeat监控配置 官方文档](https://www.elastic.co/guide/en/beats/filebeat/6.4/monitoring.html)

[`filebeat.reference.yml`](https://www.elastic.co/guide/en/beats/filebeat/6.4/filebeat-reference-yml.html)文件内容：
```yaml
######################## Filebeat Configuration ############################

# This file is a full configuration example documenting all non-deprecated
# options in comments. For a shorter configuration example, that contains only
# the most common options, please see filebeat.yml in the same directory.
#
# You can find the full configuration reference here:
# https://www.elastic.co/guide/en/beats/filebeat/index.html


#==========================  Modules configuration ============================
filebeat.modules:

#------------------------------- System Module -------------------------------
#- module: system
  # Syslog
  #syslog:
    #enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

    # Convert the timestamp to UTC. Requires Elasticsearch >= 6.1.
    #var.convert_timezone: false

    # Input configuration (advanced). Any input configuration option
    # can be added under this section.
    #input:

  # Authorization logs
  #auth:
    #enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

    # Convert the timestamp to UTC. Requires Elasticsearch >= 6.1.
    #var.convert_timezone: false

    # Input configuration (advanced). Any input configuration option
    # can be added under this section.
    #input:

#------------------------------- Apache2 Module ------------------------------
#- module: apache2
  # Access logs
  #access:
    #enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

    # Input configuration (advanced). Any input configuration option
    # can be added under this section.
    #input:

  # Error logs
  #error:
    #enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

    # Input configuration (advanced). Any input configuration option
    # can be added under this section.
    #input:

#------------------------------- Auditd Module -------------------------------
#- module: auditd
  #log:
    #enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

    # Input configuration (advanced). Any input configuration option
    # can be added under this section.
    #input:

#---------------------------- elasticsearch Module ---------------------------
- module: elasticsearch
  # Server log
  server:
    enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

  gc:
    enabled: true
    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

  audit:
    enabled: true
    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

  slowlog:
    enabled: true
    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

  deprecation:
    enabled: true
    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

#------------------------------- Icinga Module -------------------------------
#- module: icinga
  # Main logs
  #main:
    #enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

    # Input configuration (advanced). Any input configuration option
    # can be added under this section.
    #input:

  # Debug logs
  #debug:
    #enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

    # Input configuration (advanced). Any input configuration option
    # can be added under this section.
    #input:

  # Startup logs
  #startup:
    #enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

    # Input configuration (advanced). Any input configuration option
    # can be added under this section.
    #input:

#--------------------------------- IIS Module --------------------------------
#- module: iis
  # Access logs
  #access:
    #enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

    # Input configuration (advanced). Any input configuration option
    # can be added under this section.
    #input:

  # Error logs
  #error:
    #enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

    # Input configuration (advanced). Any input configuration option
    # can be added under this section.
    #input:

#-------------------------------- Kafka Module -------------------------------
- module: kafka
  # All logs
  log:
    enabled: true

    # Set custom paths for Kafka. If left empty,
    # Filebeat will look under /opt.
    #var.kafka_home:

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

    # Convert the timestamp to UTC. Requires Elasticsearch >= 6.1.
    #var.convert_timezone: false

#------------------------------- kibana Module -------------------------------
- module: kibana
  # All logs
  log:
    enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

#------------------------------ logstash Module ------------------------------
#- module: logstash
  # logs
  #log:
    #enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    # var.paths:

  # Slow logs
  #slowlog:
   #enabled: true
    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

#------------------------------- mongodb Module ------------------------------
#- module: mongodb
  # Logs
  #log:
    #enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

    # Input configuration (advanced). Any input configuration option
    # can be added under this section.
    #input:

#-------------------------------- MySQL Module -------------------------------
#- module: mysql
  # Error logs
  #error:
    #enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

    # Input configuration (advanced). Any input configuration option
    # can be added under this section.
    #input:

  # Slow logs
  #slowlog:
    #enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

    # Input configuration (advanced). Any input configuration option
    # can be added under this section.
    #input:

#-------------------------------- Nginx Module -------------------------------
#- module: nginx
  # Access logs
  #access:
    #enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

    # Input configuration (advanced). Any input configuration option
    # can be added under this section.
    #input:

  # Error logs
  #error:
    #enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

    # Input configuration (advanced). Any input configuration option
    # can be added under this section.
    #input:

#------------------------------- Osquery Module ------------------------------
- module: osquery
  result:
    enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

    # If true, all fields created by this module are prefixed with
    # `osquery.result`. Set to false to copy the fields in the root
    # of the document. The default is true.
    #var.use_namespace: true

#----------------------------- PostgreSQL Module -----------------------------
#- module: postgresql
  # Logs
  #log:
    #enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

    # Input configuration (advanced). Any input configuration option
    # can be added under this section.
    #input:

#-------------------------------- Redis Module -------------------------------
#- module: redis
  # Main logs
  #log:
    #enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths: ["/var/log/redis/redis-server.log*"]

  # Slow logs, retrieved via the Redis API (SLOWLOG)
  #slowlog:
    #enabled: true

    # The Redis hosts to connect to.
    #var.hosts: ["localhost:6379"]

    # Optional, the password to use when connecting to Redis.
    #var.password:

#------------------------------- Traefik Module ------------------------------
#- module: traefik
  # Access logs
  #access:
    #enabled: true

    # Set custom paths for the log files. If left empty,
    # Filebeat will choose the paths depending on your OS.
    #var.paths:

    # Input configuration (advanced). Any input configuration option
    # can be added under this section.
    #input:


#=========================== Filebeat inputs =============================

# List of inputs to fetch data.
filebeat.inputs:
# Each - is an input. Most options can be set at the input level, so
# you can use different inputs for various configurations.
# Below are the input specific configurations.

# Type of the files. Based on this the way the file is read is decided.
# The different types cannot be mixed in one input
#
# Possible options are:
# * log: Reads every line of the log file (default)
# * stdin: Reads the standard in

#------------------------------ Log input --------------------------------
- type: log

  # Change to true to enable this input configuration.
  enabled: false

  # Paths that should be crawled and fetched. Glob based paths.
  # To fetch all ".log" files from a specific level of subdirectories
  # /var/log/*/*.log can be used.
  # For each file found under this path, a harvester is started.
  # Make sure not file is defined twice as this can lead to unexpected behaviour.
  paths:
    - /var/log/*.log
    #- c:\programdata\elasticsearch\logs\*

  # Configure the file encoding for reading files with international characters
  # following the W3C recommendation for HTML5 (http://www.w3.org/TR/encoding).
  # Some sample encodings:
  #   plain, utf-8, utf-16be-bom, utf-16be, utf-16le, big5, gb18030, gbk,
  #    hz-gb-2312, euc-kr, euc-jp, iso-2022-jp, shift-jis, ...
  #encoding: plain


  # Exclude lines. A list of regular expressions to match. It drops the lines that are
  # matching any regular expression from the list. The include_lines is called before
  # exclude_lines. By default, no lines are dropped.
  #exclude_lines: ['^DBG']

  # Include lines. A list of regular expressions to match. It exports the lines that are
  # matching any regular expression from the list. The include_lines is called before
  # exclude_lines. By default, all the lines are exported.
  #include_lines: ['^ERR', '^WARN']

  # Exclude files. A list of regular expressions to match. Filebeat drops the files that
  # are matching any regular expression from the list. By default, no files are dropped.
  #exclude_files: ['.gz$']

  # Optional additional fields. These fields can be freely picked
  # to add additional information to the crawled log files for filtering
  #fields:
  #  level: debug
  #  review: 1

  # Set to true to store the additional fields as top level fields instead
  # of under the "fields" sub-dictionary. In case of name conflicts with the
  # fields added by Filebeat itself, the custom fields overwrite the default
  # fields.
  #fields_under_root: false

  # Ignore files which were modified more then the defined timespan in the past.
  # ignore_older is disabled by default, so no files are ignored by setting it to 0.
  # Time strings like 2h (2 hours), 5m (5 minutes) can be used.
  #ignore_older: 0

  # How often the input checks for new files in the paths that are specified
  # for harvesting. Specify 1s to scan the directory as frequently as possible
  # without causing Filebeat to scan too frequently. Default: 10s.
  #scan_frequency: 10s

  # Defines the buffer size every harvester uses when fetching the file
  #harvester_buffer_size: 16384

  # Maximum number of bytes a single log event can have
  # All bytes after max_bytes are discarded and not sent. The default is 10MB.
  # This is especially useful for multiline log messages which can get large.
  #max_bytes: 10485760

  ### Recursive glob configuration

  # Expand "**" patterns into regular glob patterns.
  #recursive_glob.enabled: true

  ### JSON configuration

  # Decode JSON options. Enable this if your logs are structured in JSON.
  # JSON key on which to apply the line filtering and multiline settings. This key
  # must be top level and its value must be string, otherwise it is ignored. If
  # no text key is defined, the line filtering and multiline features cannot be used.
  #json.message_key:

  # By default, the decoded JSON is placed under a "json" key in the output document.
  # If you enable this setting, the keys are copied top level in the output document.
  #json.keys_under_root: false

  # If keys_under_root and this setting are enabled, then the values from the decoded
  # JSON object overwrite the fields that Filebeat normally adds (type, source, offset, etc.)
  # in case of conflicts.
  #json.overwrite_keys: false

  # If this setting is enabled, Filebeat adds a "error.message" and "error.key: json" key in case of JSON
  # unmarshaling errors or when a text key is defined in the configuration but cannot
  # be used.
  #json.add_error_key: false

  ### Multiline options

  # Multiline can be used for log messages spanning multiple lines. This is common
  # for Java Stack Traces or C-Line Continuation

  # The regexp Pattern that has to be matched. The example pattern matches all lines starting with [
  #multiline.pattern: ^\[

  # Defines if the pattern set under pattern should be negated or not. Default is false.
  #multiline.negate: false

  # Match can be set to "after" or "before". It is used to define if lines should be append to a pattern
  # that was (not) matched before or after or as long as a pattern is not matched based on negate.
  # Note: After is the equivalent to previous and before is the equivalent to to next in Logstash
  #multiline.match: after

  # The maximum number of lines that are combined to one event.
  # In case there are more the max_lines the additional lines are discarded.
  # Default is 500
  #multiline.max_lines: 500

  # After the defined timeout, an multiline event is sent even if no new pattern was found to start a new event
  # Default is 5s.
  #multiline.timeout: 5s

  # Setting tail_files to true means filebeat starts reading new files at the end
  # instead of the beginning. If this is used in combination with log rotation
  # this can mean that the first entries of a new file are skipped.
  #tail_files: false

  # The Ingest Node pipeline ID associated with this input. If this is set, it
  # overwrites the pipeline option from the Elasticsearch output.
  #pipeline:

  # If symlinks is enabled, symlinks are opened and harvested. The harvester is opening the
  # original for harvesting but will report the symlink name as source.
  #symlinks: false

  # Backoff values define how aggressively filebeat crawls new files for updates
  # The default values can be used in most cases. Backoff defines how long it is waited
  # to check a file again after EOF is reached. Default is 1s which means the file
  # is checked every second if new lines were added. This leads to a near real time crawling.
  # Every time a new line appears, backoff is reset to the initial value.
  #backoff: 1s

  # Max backoff defines what the maximum backoff time is. After having backed off multiple times
  # from checking the files, the waiting time will never exceed max_backoff independent of the
  # backoff factor. Having it set to 10s means in the worst case a new line can be added to a log
  # file after having backed off multiple times, it takes a maximum of 10s to read the new line
  #max_backoff: 10s

  # The backoff factor defines how fast the algorithm backs off. The bigger the backoff factor,
  # the faster the max_backoff value is reached. If this value is set to 1, no backoff will happen.
  # The backoff value will be multiplied each time with the backoff_factor until max_backoff is reached
  #backoff_factor: 2

  # Max number of harvesters that are started in parallel.
  # Default is 0 which means unlimited
  #harvester_limit: 0

  ### Harvester closing options

  # Close inactive closes the file handler after the predefined period.
  # The period starts when the last line of the file was, not the file ModTime.
  # Time strings like 2h (2 hours), 5m (5 minutes) can be used.
  #close_inactive: 5m

  # Close renamed closes a file handler when the file is renamed or rotated.
  # Note: Potential data loss. Make sure to read and understand the docs for this option.
  #close_renamed: false

  # When enabling this option, a file handler is closed immediately in case a file can't be found
  # any more. In case the file shows up again later, harvesting will continue at the last known position
  # after scan_frequency.
  #close_removed: true

  # Closes the file handler as soon as the harvesters reaches the end of the file.
  # By default this option is disabled.
  # Note: Potential data loss. Make sure to read and understand the docs for this option.
  #close_eof: false

  ### State options

  # Files for the modification data is older then clean_inactive the state from the registry is removed
  # By default this is disabled.
  #clean_inactive: 0

  # Removes the state for file which cannot be found on disk anymore immediately
  #clean_removed: true

  # Close timeout closes the harvester after the predefined time.
  # This is independent if the harvester did finish reading the file or not.
  # By default this option is disabled.
  # Note: Potential data loss. Make sure to read and understand the docs for this option.
  #close_timeout: 0

  # Defines if inputs is enabled
  #enabled: true

#----------------------------- Stdin input -------------------------------
# Configuration to use stdin input
#- type: stdin

#------------------------- Redis slowlog input ---------------------------
# Experimental: Config options for the redis slow log input
#- type: redis
  #enabled: false

  # List of hosts to pool to retrieve the slow log information.
  #hosts: ["localhost:6379"]

  # How often the input checks for redis slow log.
  #scan_frequency: 10s

  # Timeout after which time the input should return an error
  #timeout: 1s

  # Network type to be used for redis connection. Default: tcp
  #network: tcp

  # Max number of concurrent connections. Default: 10
  #maxconn: 10

  # Redis AUTH password. Empty by default.
  #password: foobared

#------------------------------ Udp input --------------------------------
# Experimental: Config options for the udp input
#- type: udp
  #enabled: false

  # Maximum size of the message received over UDP
  #max_message_size: 10KiB

#------------------------------ TCP input --------------------------------
# Experimental: Config options for the TCP input
#- type: tcp
  #enabled: false

  # The host and port to receive the new event
  #host: "localhost:9000"

  # Character used to split new message
  #line_delimiter: "\n"

  # Maximum size in bytes of the message received over TCP
  #max_message_size: 20MiB

  # The number of seconds of inactivity before a remote connection is closed.
  #timeout: 300s

  # Use SSL settings for TCP.
  #ssl.enabled: true

  # List of supported/valid TLS versions. By default all TLS versions 1.0 up to
  # 1.2 are enabled.
  #ssl.supported_protocols: [TLSv1.0, TLSv1.1, TLSv1.2]

  # SSL configuration. By default is off.
  # List of root certificates for client verifications
  #ssl.certificate_authorities: ["/etc/pki/root/ca.pem"]

  # Certificate for SSL server authentication.
  #ssl.certificate: "/etc/pki/client/cert.pem"

  # Server Certificate Key,
  #ssl.key: "/etc/pki/client/cert.key"

  # Optional passphrase for decrypting the Certificate Key.
  #ssl.key_passphrase: ''

  # Configure cipher suites to be used for SSL connections.
  #ssl.cipher_suites: []

  # Configure curve types for ECDHE based cipher suites.
  #ssl.curve_types: []

  # Configure what types of client authentication are supported. Valid options
  # are `none`, `optional`, and `required`. Default is required.
  #ssl.client_authentication: "required"

#------------------------------ Syslog input --------------------------------
# Experimental: Config options for the Syslog input
# Accept RFC3164 formatted syslog event via UDP.
#- type: syslog
  #enabled: false
  #protocol.udp:
    # The host and port to receive the new event
    #host: "localhost:9000"

    # Maximum size of the message received over UDP
    #max_message_size: 10KiB

# Accept RFC3164 formatted syslog event via TCP.
#- type: syslog
  #enabled: false

  #protocol.tcp:
    # The host and port to receive the new event
    #host: "localhost:9000"

    # Character used to split new message
    #line_delimiter: "\n"

    # Maximum size in bytes of the message received over TCP
    #max_message_size: 20MiB

    # The number of seconds of inactivity before a remote connection is closed.
    #timeout: 300s

    # Use SSL settings for TCP.
    #ssl.enabled: true

    # List of supported/valid TLS versions. By default all TLS versions 1.0 up to
    # 1.2 are enabled.
    #ssl.supported_protocols: [TLSv1.0, TLSv1.1, TLSv1.2]

    # SSL configuration. By default is off.
    # List of root certificates for client verifications
    #ssl.certificate_authorities: ["/etc/pki/root/ca.pem"]

    # Certificate for SSL server authentication.
    #ssl.certificate: "/etc/pki/client/cert.pem"

    # Server Certificate Key,
    #ssl.key: "/etc/pki/client/cert.key"

    # Optional passphrase for decrypting the Certificate Key.
    #ssl.key_passphrase: ''

    # Configure cipher suites to be used for SSL connections.
    #ssl.cipher_suites: []

    # Configure curve types for ECDHE based cipher suites.
    #ssl.curve_types: []

    # Configure what types of client authentication are supported. Valid options
    # are `none`, `optional`, and `required`. Default is required.
    #ssl.client_authentication: "required"

#------------------------------ Docker input --------------------------------
# Experimental: Docker input reads and parses `json-file` logs from Docker
#- type: docker
  #enabled: false

  # Combine partial lines flagged by `json-file` format
  #combine_partials: true

  # Use this to read from all containers, replace * with a container id to read from one:
  #containers:
  #  stream: all # can be all, stdout or stderr
  #  ids:
  #    - '*'

#========================== Filebeat autodiscover ==============================

# Autodiscover allows you to detect changes in the system and spawn new modules
# or inputs as they happen.

#filebeat.autodiscover:
  # List of enabled autodiscover providers
#  providers:
#    - type: docker
#      templates:
#        - condition:
#            equals.docker.container.image: busybox
#          config:
#            - type: log
#              paths:
#                - /var/lib/docker/containers/${data.docker.container.id}/*.log

#========================= Filebeat global options ============================

# Name of the registry file. If a relative path is used, it is considered relative to the
# data path.
#filebeat.registry_file: ${path.data}/registry

# The permissions mask to apply on registry file. The default value is 0600.
# Must be a valid Unix-style file permissions mask expressed in octal notation.
# This option is not supported on Windows.
#filebeat.registry_file_permissions: 0600

# By default Ingest pipelines are not updated if a pipeline with the same ID
# already exists. If this option is enabled Filebeat overwrites pipelines
# everytime a new Elasticsearch connection is established.
#filebeat.overwrite_pipelines: false

# How long filebeat waits on shutdown for the publisher to finish.
# Default is 0, not waiting.
#filebeat.shutdown_timeout: 0

# Enable filebeat config reloading
#filebeat.config:
  #inputs:
    #enabled: false
    #path: inputs.d/*.yml
    #reload.enabled: true
    #reload.period: 10s
  #modules:
    #enabled: false
    #path: modules.d/*.yml
    #reload.enabled: true
    #reload.period: 10s

#================================ General ======================================

# The name of the shipper that publishes the network data. It can be used to group
# all the transactions sent by a single shipper in the web interface.
# If this options is not defined, the hostname is used.
#name:

# The tags of the shipper are included in their own field with each
# transaction published. Tags make it easy to group servers by different
# logical properties.
#tags: ["service-X", "web-tier"]

# Optional fields that you can specify to add additional information to the
# output. Fields can be scalar values, arrays, dictionaries, or any nested
# combination of these.
#fields:
#  env: staging

# If this option is set to true, the custom fields are stored as top-level
# fields in the output document instead of being grouped under a fields
# sub-dictionary. Default is false.
#fields_under_root: false

# Internal queue configuration for buffering events to be published.
#queue:
  # Queue type by name (default 'mem')
  # The memory queue will present all available events (up to the outputs
  # bulk_max_size) to the output, the moment the output is ready to server
  # another batch of events.
  #mem:
    # Max number of events the queue can buffer.
    #events: 4096

    # Hints the minimum number of events stored in the queue,
    # before providing a batch of events to the outputs.
    # The default value is set to 2048.
    # A value of 0 ensures events are immediately available
    # to be sent to the outputs.
    #flush.min_events: 2048

    # Maximum duration after which events are available to the outputs,
    # if the number of events stored in the queue is < min_flush_events.
    #flush.timeout: 1s

  # The spool queue will store events in a local spool file, before
  # forwarding the events to the outputs.
  #
  # Beta: spooling to disk is currently a beta feature. Use with care.
  #
  # The spool file is a circular buffer, which blocks once the file/buffer is full.
  # Events are put into a write buffer and flushed once the write buffer
  # is full or the flush_timeout is triggered.
  # Once ACKed by the output, events are removed immediately from the queue,
  # making space for new events to be persisted.
  #spool:
    # The file namespace configures the file path and the file creation settings.
    # Once the file exists, the `size`, `page_size` and `prealloc` settings
    # will have no more effect.
    #file:
      # Location of spool file. The default value is ${path.data}/spool.dat.
      #path: "${path.data}/spool.dat"

      # Configure file permissions if file is created. The default value is 0600.
      #permissions: 0600

      # File size hint. The spool blocks, once this limit is reached. The default value is 100 MiB.
      #size: 100MiB

      # The files page size. A file is split into multiple pages of the same size. The default value is 4KiB.
      #page_size: 4KiB

      # If prealloc is set, the required space for the file is reserved using
      # truncate. The default value is true.
      #prealloc: true

    # Spool writer settings
    # Events are serialized into a write buffer. The write buffer is flushed if:
    # - The buffer limit has been reached.
    # - The configured limit of buffered events is reached.
    # - The flush timeout is triggered.
    #write:
      # Sets the write buffer size.
      #buffer_size: 1MiB

      # Maximum duration after which events are flushed, if the write buffer
      # is not full yet. The default value is 1s.
      #flush.timeout: 1s

      # Number of maximum buffered events. The write buffer is flushed once the
      # limit is reached.
      #flush.events: 16384

      # Configure the on-disk event encoding. The encoding can be changed
      # between restarts.
      # Valid encodings are: json, ubjson, and cbor.
      #codec: cbor
    #read:
      # Reader flush timeout, waiting for more events to become available, so
      # to fill a complete batch, as required by the outputs.
      # If flush_timeout is 0, all available events are forwarded to the
      # outputs immediately.
      # The default value is 0s.
      #flush.timeout: 0s

# Sets the maximum number of CPUs that can be executing simultaneously. The
# default is the number of logical CPUs available in the system.
#max_procs:

#================================ Processors ===================================

# Processors are used to reduce the number of fields in the exported event or to
# enhance the event with external metadata. This section defines a list of
# processors that are applied one by one and the first one receives the initial
# event:
#
#   event -> filter1 -> event1 -> filter2 ->event2 ...
#
# The supported processors are drop_fields, drop_event, include_fields, and
# add_cloud_metadata.
#
# For example, you can use the following processors to keep the fields that
# contain CPU load percentages, but remove the fields that contain CPU ticks
# values:
#
#processors:
#- include_fields:
#    fields: ["cpu"]
#- drop_fields:
#    fields: ["cpu.user", "cpu.system"]
#
# The following example drops the events that have the HTTP response code 200:
#
#processors:
#- drop_event:
#    when:
#       equals:
#           http.code: 200
#
# The following example renames the field a to b:
#
#processors:
#- rename:
#    fields:
#       - from: "a"
#         to: "b"
#
# The following example tokenizes the string into fields:
#
#processors:
#- dissect:
#    tokenizer: "%{key1} - %{key2}"
#    field: "message"
#    target_prefix: "dissect"
#
# The following example enriches each event with metadata from the cloud
# provider about the host machine. It works on EC2, GCE, DigitalOcean,
# Tencent Cloud, and Alibaba Cloud.
#
#processors:
#- add_cloud_metadata: ~
#
# The following example enriches each event with the machine's local time zone
# offset from UTC.
#
#processors:
#- add_locale:
#    format: offset
#
# The following example enriches each event with docker metadata, it matches
# given fields to an existing container id and adds info from that container:
#
#processors:
#- add_docker_metadata:
#    host: "unix:///var/run/docker.sock"
#    match_fields: ["system.process.cgroup.id"]
#    match_pids: ["process.pid", "process.ppid"]
#    match_source: true
#    match_source_index: 4
#    match_short_id: false
#    cleanup_timeout: 60
#    # To connect to Docker over TLS you must specify a client and CA certificate.
#    #ssl:
#    #  certificate_authority: "/etc/pki/root/ca.pem"
#    #  certificate:           "/etc/pki/client/cert.pem"
#    #  key:                   "/etc/pki/client/cert.key"
#
# The following example enriches each event with docker metadata, it matches
# container id from log path available in `source` field (by default it expects
# it to be /var/lib/docker/containers/*/*.log).
#
#processors:
#- add_docker_metadata: ~
#
# The following example enriches each event with host metadata.
#
#processors:
#- add_host_metadata:
#   netinfo.enabled: false
#

#============================= Elastic Cloud ==================================

# These settings simplify using filebeat with the Elastic Cloud (https://cloud.elastic.co/).

# The cloud.id setting overwrites the `output.elasticsearch.hosts` and
# `setup.kibana.host` options.
# You can find the `cloud.id` in the Elastic Cloud web UI.
#cloud.id:

# The cloud.auth setting overwrites the `output.elasticsearch.username` and
# `output.elasticsearch.password` settings. The format is `<user>:<pass>`.
#cloud.auth:

#================================ Outputs ======================================

# Configure what output to use when sending the data collected by the beat.

#-------------------------- Elasticsearch output -------------------------------
output.elasticsearch:
  # Boolean flag to enable or disable the output module.
  #enabled: true

  # Array of hosts to connect to.
  # Scheme and port can be left out and will be set to the default (http and 9200)
  # In case you specify and additional path, the scheme is required: http://localhost:9200/path
  # IPv6 addresses should always be defined as: https://[2001:db8::1]:9200
  hosts: ["localhost:9200"]

  # Set gzip compression level.
  #compression_level: 0

  # Configure escaping html symbols in strings.
  #escape_html: true

  # Optional protocol and basic auth credentials.
  #protocol: "https"
  #username: "elastic"
  #password: "changeme"

  # Dictionary of HTTP parameters to pass within the url with index operations.
  #parameters:
    #param1: value1
    #param2: value2

  # Number of workers per Elasticsearch host.
  #worker: 1

  # Optional index name. The default is "filebeat" plus date
  # and generates [filebeat-]YYYY.MM.DD keys.
  # In case you modify this pattern you must update setup.template.name and setup.template.pattern accordingly.
  #index: "filebeat-%{[beat.version]}-%{+yyyy.MM.dd}"

  # Optional ingest node pipeline. By default no pipeline will be used.
  #pipeline: ""

  # Optional HTTP Path
  #path: "/elasticsearch"

  # Custom HTTP headers to add to each request
  #headers:
  #  X-My-Header: Contents of the header

  # Proxy server url
  #proxy_url: http://proxy:3128

  # The number of times a particular Elasticsearch index operation is attempted. If
  # the indexing operation doesn't succeed after this many retries, the events are
  # dropped. The default is 3.
  #max_retries: 3

  # The maximum number of events to bulk in a single Elasticsearch bulk API index request.
  # The default is 50.
  #bulk_max_size: 50

  # The number of seconds to wait before trying to reconnect to Elasticsearch
  # after a network error. After waiting backoff.init seconds, the Beat
  # tries to reconnect. If the attempt fails, the backoff timer is increased
  # exponentially up to backoff.max. After a successful connection, the backoff
  # timer is reset. The default is 1s.
  #backoff.init: 1s

  # The maximum number of seconds to wait before attempting to connect to
  # Elasticsearch after a network error. The default is 60s.
  #backoff.max: 60s

  # Configure http request timeout before failing a request to Elasticsearch.
  #timeout: 90

  # Use SSL settings for HTTPS.
  #ssl.enabled: true

  # Configure SSL verification mode. If `none` is configured, all server hosts
  # and certificates will be accepted. In this mode, SSL based connections are
  # susceptible to man-in-the-middle attacks. Use only for testing. Default is
  # `full`.
  #ssl.verification_mode: full

  # List of supported/valid TLS versions. By default all TLS versions 1.0 up to
  # 1.2 are enabled.
  #ssl.supported_protocols: [TLSv1.0, TLSv1.1, TLSv1.2]

  # SSL configuration. By default is off.
  # List of root certificates for HTTPS server verifications
  #ssl.certificate_authorities: ["/etc/pki/root/ca.pem"]

  # Certificate for SSL client authentication
  #ssl.certificate: "/etc/pki/client/cert.pem"

  # Client Certificate Key
  #ssl.key: "/etc/pki/client/cert.key"

  # Optional passphrase for decrypting the Certificate Key.
  #ssl.key_passphrase: ''

  # Configure cipher suites to be used for SSL connections
  #ssl.cipher_suites: []

  # Configure curve types for ECDHE based cipher suites
  #ssl.curve_types: []

  # Configure what types of renegotiation are supported. Valid options are
  # never, once, and freely. Default is never.
  #ssl.renegotiation: never


#----------------------------- Logstash output ---------------------------------
#output.logstash:
  # Boolean flag to enable or disable the output module.
  #enabled: true

  # The Logstash hosts
  #hosts: ["localhost:5044"]

  # Number of workers per Logstash host.
  #worker: 1

  # Set gzip compression level.
  #compression_level: 3

  # Configure escaping html symbols in strings.
  #escape_html: true

  # Optional maximum time to live for a connection to Logstash, after which the
  # connection will be re-established.  A value of `0s` (the default) will
  # disable this feature.
  #
  # Not yet supported for async connections (i.e. with the "pipelining" option set)
  #ttl: 30s

  # Optional load balance the events between the Logstash hosts. Default is false.
  #loadbalance: false

  # Number of batches to be sent asynchronously to Logstash while processing
  # new batches.
  #pipelining: 2

  # If enabled only a subset of events in a batch of events is transferred per
  # transaction.  The number of events to be sent increases up to `bulk_max_size`
  # if no error is encountered.
  #slow_start: false

  # The number of seconds to wait before trying to reconnect to Logstash
  # after a network error. After waiting backoff.init seconds, the Beat
  # tries to reconnect. If the attempt fails, the backoff timer is increased
  # exponentially up to backoff.max. After a successful connection, the backoff
  # timer is reset. The default is 1s.
  #backoff.init: 1s

  # The maximum number of seconds to wait before attempting to connect to
  # Logstash after a network error. The default is 60s.
  #backoff.max: 60s

  # Optional index name. The default index name is set to filebeat
  # in all lowercase.
  #index: 'filebeat'

  # SOCKS5 proxy server URL
  #proxy_url: socks5://user:password@socks5-server:2233

  # Resolve names locally when using a proxy server. Defaults to false.
  #proxy_use_local_resolver: false

  # Enable SSL support. SSL is automatically enabled, if any SSL setting is set.
  #ssl.enabled: true

  # Configure SSL verification mode. If `none` is configured, all server hosts
  # and certificates will be accepted. In this mode, SSL based connections are
  # susceptible to man-in-the-middle attacks. Use only for testing. Default is
  # `full`.
  #ssl.verification_mode: full

  # List of supported/valid TLS versions. By default all TLS versions 1.0 up to
  # 1.2 are enabled.
  #ssl.supported_protocols: [TLSv1.0, TLSv1.1, TLSv1.2]

  # Optional SSL configuration options. SSL is off by default.
  # List of root certificates for HTTPS server verifications
  #ssl.certificate_authorities: ["/etc/pki/root/ca.pem"]

  # Certificate for SSL client authentication
  #ssl.certificate: "/etc/pki/client/cert.pem"

  # Client Certificate Key
  #ssl.key: "/etc/pki/client/cert.key"

  # Optional passphrase for decrypting the Certificate Key.
  #ssl.key_passphrase: ''

  # Configure cipher suites to be used for SSL connections
  #ssl.cipher_suites: []

  # Configure curve types for ECDHE based cipher suites
  #ssl.curve_types: []

  # Configure what types of renegotiation are supported. Valid options are
  # never, once, and freely. Default is never.
  #ssl.renegotiation: never

  # The number of times to retry publishing an event after a publishing failure.
  # After the specified number of retries, the events are typically dropped.
  # Some Beats, such as Filebeat and Winlogbeat, ignore the max_retries setting
  # and retry until all events are published.  Set max_retries to a value less
  # than 0 to retry until all events are published. The default is 3.
  #max_retries: 3
  
  # The maximum number of events to bulk in a single Logstash request. The
  # default is 2048.
  #bulk_max_size: 2048
  
  # The number of seconds to wait for responses from the Logstash server before
  # timing out. The default is 30s.
  #timeout: 30s

#------------------------------- Kafka output ----------------------------------
#output.kafka:
  # Boolean flag to enable or disable the output module.
  #enabled: true

  # The list of Kafka broker addresses from where to fetch the cluster metadata.
  # The cluster metadata contain the actual Kafka brokers events are published
  # to.
  #hosts: ["localhost:9092"]

  # The Kafka topic used for produced events. The setting can be a format string
  # using any event field. To set the topic from document type use `%{[type]}`.
  #topic: beats

  # The Kafka event key setting. Use format string to create unique event key.
  # By default no event key will be generated.
  #key: ''

  # The Kafka event partitioning strategy. Default hashing strategy is `hash`
  # using the `output.kafka.key` setting or randomly distributes events if
  # `output.kafka.key` is not configured.
  #partition.hash:
    # If enabled, events will only be published to partitions with reachable
    # leaders. Default is false.
    #reachable_only: false

    # Configure alternative event field names used to compute the hash value.
    # If empty `output.kafka.key` setting will be used.
    # Default value is empty list.
    #hash: []

  # Authentication details. Password is required if username is set.
  #username: ''
  #password: ''

  # Kafka version filebeat is assumed to run against. Defaults to the "1.0.0".
  #version: '1.0.0'

  # Configure JSON encoding
  #codec.json:
    # Pretty print json event
    #pretty: false

    # Configure escaping html symbols in strings.
    #escape_html: true

  # Metadata update configuration. Metadata do contain leader information
  # deciding which broker to use when publishing.
  #metadata:
    # Max metadata request retry attempts when cluster is in middle of leader
    # election. Defaults to 3 retries.
    #retry.max: 3

    # Waiting time between retries during leader elections. Default is 250ms.
    #retry.backoff: 250ms

    # Refresh metadata interval. Defaults to every 10 minutes.
    #refresh_frequency: 10m

  # The number of concurrent load-balanced Kafka output workers.
  #worker: 1

  # The number of times to retry publishing an event after a publishing failure.
  # After the specified number of retries, the events are typically dropped.
  # Some Beats, such as Filebeat, ignore the max_retries setting and retry until
  # all events are published.  Set max_retries to a value less than 0 to retry
  # until all events are published. The default is 3.
  #max_retries: 3

  # The maximum number of events to bulk in a single Kafka request. The default
  # is 2048.
  #bulk_max_size: 2048

  # The number of seconds to wait for responses from the Kafka brokers before
  # timing out. The default is 30s.
  #timeout: 30s

  # The maximum duration a broker will wait for number of required ACKs. The
  # default is 10s.
  #broker_timeout: 10s

  # The number of messages buffered for each Kafka broker. The default is 256.
  #channel_buffer_size: 256

  # The keep-alive period for an active network connection. If 0s, keep-alives
  # are disabled. The default is 0 seconds.
  #keep_alive: 0

  # Sets the output compression codec. Must be one of none, snappy and gzip. The
  # default is gzip.
  #compression: gzip

  # Set the compression level. Currently only gzip provides a compression level
  # between 0 and 9. The default value is chosen by the compression algorithm.
  #compression_level: 4

  # The maximum permitted size of JSON-encoded messages. Bigger messages will be
  # dropped. The default value is 1000000 (bytes). This value should be equal to
  # or less than the broker's message.max.bytes.
  #max_message_bytes: 1000000

  # The ACK reliability level required from broker. 0=no response, 1=wait for
  # local commit, -1=wait for all replicas to commit. The default is 1.  Note:
  # If set to 0, no ACKs are returned by Kafka. Messages might be lost silently
  # on error.
  #required_acks: 1

  # The configurable ClientID used for logging, debugging, and auditing
  # purposes.  The default is "beats".
  #client_id: beats

  # Enable SSL support. SSL is automatically enabled, if any SSL setting is set.
  #ssl.enabled: true

  # Optional SSL configuration options. SSL is off by default.
  # List of root certificates for HTTPS server verifications
  #ssl.certificate_authorities: ["/etc/pki/root/ca.pem"]

  # Configure SSL verification mode. If `none` is configured, all server hosts
  # and certificates will be accepted. In this mode, SSL based connections are
  # susceptible to man-in-the-middle attacks. Use only for testing. Default is
  # `full`.
  #ssl.verification_mode: full

  # List of supported/valid TLS versions. By default all TLS versions 1.0 up to
  # 1.2 are enabled.
  #ssl.supported_protocols: [TLSv1.0, TLSv1.1, TLSv1.2]

  # Certificate for SSL client authentication
  #ssl.certificate: "/etc/pki/client/cert.pem"

  # Client Certificate Key
  #ssl.key: "/etc/pki/client/cert.key"

  # Optional passphrase for decrypting the Certificate Key.
  #ssl.key_passphrase: ''

  # Configure cipher suites to be used for SSL connections
  #ssl.cipher_suites: []

  # Configure curve types for ECDHE based cipher suites
  #ssl.curve_types: []

  # Configure what types of renegotiation are supported. Valid options are
  # never, once, and freely. Default is never.
  #ssl.renegotiation: never

#------------------------------- Redis output ----------------------------------
#output.redis:
  # Boolean flag to enable or disable the output module.
  #enabled: true

  # Configure JSON encoding
  #codec.json:
    # Pretty print json event
    #pretty: false

    # Configure escaping html symbols in strings.
    #escape_html: true

  # The list of Redis servers to connect to. If load balancing is enabled, the
  # events are distributed to the servers in the list. If one server becomes
  # unreachable, the events are distributed to the reachable servers only.
  #hosts: ["localhost:6379"]

  # The Redis port to use if hosts does not contain a port number. The default
  # is 6379.
  #port: 6379

  # The name of the Redis list or channel the events are published to. The
  # default is filebeat.
  #key: filebeat

  # The password to authenticate with. The default is no authentication.
  #password:

  # The Redis database number where the events are published. The default is 0.
  #db: 0

  # The Redis data type to use for publishing events. If the data type is list,
  # the Redis RPUSH command is used. If the data type is channel, the Redis
  # PUBLISH command is used. The default value is list.
  #datatype: list

  # The number of workers to use for each host configured to publish events to
  # Redis. Use this setting along with the loadbalance option. For example, if
  # you have 2 hosts and 3 workers, in total 6 workers are started (3 for each
  # host).
  #worker: 1

  # If set to true and multiple hosts or workers are configured, the output
  # plugin load balances published events onto all Redis hosts. If set to false,
  # the output plugin sends all events to only one host (determined at random)
  # and will switch to another host if the currently selected one becomes
  # unreachable. The default value is true.
  #loadbalance: true

  # The Redis connection timeout in seconds. The default is 5 seconds.
  #timeout: 5s

  # The number of times to retry publishing an event after a publishing failure.
  # After the specified number of retries, the events are typically dropped.
  # Some Beats, such as Filebeat, ignore the max_retries setting and retry until
  # all events are published. Set max_retries to a value less than 0 to retry
  # until all events are published. The default is 3.
  #max_retries: 3

  # The number of seconds to wait before trying to reconnect to Redis
  # after a network error. After waiting backoff.init seconds, the Beat
  # tries to reconnect. If the attempt fails, the backoff timer is increased
  # exponentially up to backoff.max. After a successful connection, the backoff
  # timer is reset. The default is 1s.
  #backoff.init: 1s

  # The maximum number of seconds to wait before attempting to connect to
  # Redis after a network error. The default is 60s.
  #backoff.max: 60s

  # The maximum number of events to bulk in a single Redis request or pipeline.
  # The default is 2048.
  #bulk_max_size: 2048

  # The URL of the SOCKS5 proxy to use when connecting to the Redis servers. The
  # value must be a URL with a scheme of socks5://.
  #proxy_url:

  # This option determines whether Redis hostnames are resolved locally when
  # using a proxy. The default value is false, which means that name resolution
  # occurs on the proxy server.
  #proxy_use_local_resolver: false

  # Enable SSL support. SSL is automatically enabled, if any SSL setting is set.
  #ssl.enabled: true

  # Configure SSL verification mode. If `none` is configured, all server hosts
  # and certificates will be accepted. In this mode, SSL based connections are
  # susceptible to man-in-the-middle attacks. Use only for testing. Default is
  # `full`.
  #ssl.verification_mode: full

  # List of supported/valid TLS versions. By default all TLS versions 1.0 up to
  # 1.2 are enabled.
  #ssl.supported_protocols: [TLSv1.0, TLSv1.1, TLSv1.2]

  # Optional SSL configuration options. SSL is off by default.
  # List of root certificates for HTTPS server verifications
  #ssl.certificate_authorities: ["/etc/pki/root/ca.pem"]

  # Certificate for SSL client authentication
  #ssl.certificate: "/etc/pki/client/cert.pem"

  # Client Certificate Key
  #ssl.key: "/etc/pki/client/cert.key"

  # Optional passphrase for decrypting the Certificate Key.
  #ssl.key_passphrase: ''

  # Configure cipher suites to be used for SSL connections
  #ssl.cipher_suites: []

  # Configure curve types for ECDHE based cipher suites
  #ssl.curve_types: []

  # Configure what types of renegotiation are supported. Valid options are
  # never, once, and freely. Default is never.
  #ssl.renegotiation: never

#------------------------------- File output -----------------------------------
#output.file:
  # Boolean flag to enable or disable the output module.
  #enabled: true

  # Configure JSON encoding
  #codec.json:
    # Pretty print json event
    #pretty: false

    # Configure escaping html symbols in strings.
    #escape_html: true

  # Path to the directory where to save the generated files. The option is
  # mandatory.
  #path: "/tmp/filebeat"

  # Name of the generated files. The default is `filebeat` and it generates
  # files: `filebeat`, `filebeat.1`, `filebeat.2`, etc.
  #filename: filebeat

  # Maximum size in kilobytes of each file. When this size is reached, and on
  # every filebeat restart, the files are rotated. The default value is 10240
  # kB.
  #rotate_every_kb: 10000

  # Maximum number of files under path. When this number of files is reached,
  # the oldest file is deleted and the rest are shifted from last to first. The
  # default is 7 files.
  #number_of_files: 7

  # Permissions to use for file creation. The default is 0600.
  #permissions: 0600


#----------------------------- Console output ---------------------------------
#output.console:
  # Boolean flag to enable or disable the output module.
  #enabled: true

  # Configure JSON encoding
  #codec.json:
    # Pretty print json event
    #pretty: false

    # Configure escaping html symbols in strings.
    #escape_html: true

#================================= Paths ======================================

# The home path for the filebeat installation. This is the default base path
# for all other path settings and for miscellaneous files that come with the
# distribution (for example, the sample dashboards).
# If not set by a CLI flag or in the configuration file, the default for the
# home path is the location of the binary.
#path.home:

# The configuration path for the filebeat installation. This is the default
# base path for configuration files, including the main YAML configuration file
# and the Elasticsearch template file. If not set by a CLI flag or in the
# configuration file, the default for the configuration path is the home path.
#path.config: ${path.home}

# The data path for the filebeat installation. This is the default base path
# for all the files in which filebeat needs to store its data. If not set by a
# CLI flag or in the configuration file, the default for the data path is a data
# subdirectory inside the home path.
#path.data: ${path.home}/data

# The logs path for a filebeat installation. This is the default location for
# the Beat's log files. If not set by a CLI flag or in the configuration file,
# the default for the logs path is a logs subdirectory inside the home path.
#path.logs: ${path.home}/logs

#================================ Keystore ==========================================
# Location of the Keystore containing the keys and their sensitive values.
#keystore.path: "${path.config}/beats.keystore"

#============================== Dashboards =====================================
# These settings control loading the sample dashboards to the Kibana index. Loading
# the dashboards are disabled by default and can be enabled either by setting the
# options here, or by using the `-setup` CLI flag or the `setup` command.
#setup.dashboards.enabled: false

# The directory from where to read the dashboards. The default is the `kibana`
# folder in the home path.
#setup.dashboards.directory: ${path.home}/kibana

# The URL from where to download the dashboards archive. It is used instead of
# the directory if it has a value.
#setup.dashboards.url:

# The file archive (zip file) from where to read the dashboards. It is used instead
# of the directory when it has a value.
#setup.dashboards.file:

# In case the archive contains the dashboards from multiple Beats, this lets you
# select which one to load. You can load all the dashboards in the archive by
# setting this to the empty string.
#setup.dashboards.beat: filebeat

# The name of the Kibana index to use for setting the configuration. Default is ".kibana"
#setup.dashboards.kibana_index: .kibana

# The Elasticsearch index name. This overwrites the index name defined in the
# dashboards and index pattern. Example: testbeat-*
#setup.dashboards.index:

# Always use the Kibana API for loading the dashboards instead of autodetecting
# how to install the dashboards by first querying Elasticsearch.
#setup.dashboards.always_kibana: false

# If true and Kibana is not reachable at the time when dashboards are loaded,
# it will retry to reconnect to Kibana instead of exiting with an error.
#setup.dashboards.retry.enabled: false

# Duration interval between Kibana connection retries.
#setup.dashboards.retry.interval: 1s

# Maximum number of retries before exiting with an error, 0 for unlimited retrying.
#setup.dashboards.retry.maximum: 0


#============================== Template =====================================

# A template is used to set the mapping in Elasticsearch
# By default template loading is enabled and the template is loaded.
# These settings can be adjusted to load your own template or overwrite existing ones.

# Set to false to disable template loading.
#setup.template.enabled: true

# Template name. By default the template name is "filebeat-%{[beat.version]}"
# The template name and pattern has to be set in case the elasticsearch index pattern is modified.
#setup.template.name: "filebeat-%{[beat.version]}"

# Template pattern. By default the template pattern is "-%{[beat.version]}-*" to apply to the default index settings.
# The first part is the version of the beat and then -* is used to match all daily indices.
# The template name and pattern has to be set in case the elasticsearch index pattern is modified.
#setup.template.pattern: "filebeat-%{[beat.version]}-*"

# Path to fields.yml file to generate the template
#setup.template.fields: "${path.config}/fields.yml"

# A list of fields to be added to the template and Kibana index pattern. Also
# specify setup.template.overwrite: true to overwrite the existing template.
# This setting is experimental.
#setup.template.append_fields:
#- name: field_name
#  type: field_type

# Enable json template loading. If this is enabled, the fields.yml is ignored.
#setup.template.json.enabled: false

# Path to the json template file
#setup.template.json.path: "${path.config}/template.json"

# Name under which the template is stored in Elasticsearch
#setup.template.json.name: ""

# Overwrite existing template
#setup.template.overwrite: false

# Elasticsearch template settings
setup.template.settings:

  # A dictionary of settings to place into the settings.index dictionary
  # of the Elasticsearch template. For more details, please check
  # https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping.html
  #index:
    #number_of_shards: 1
    #codec: best_compression
    #number_of_routing_shards: 30

  # A dictionary of settings for the _source field. For more details, please check
  # https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-source-field.html
  #_source:
    #enabled: false

#============================== Kibana =====================================

# Starting with Beats version 6.0.0, the dashboards are loaded via the Kibana API.
# This requires a Kibana endpoint configuration.
setup.kibana:

  # Kibana Host
  # Scheme and port can be left out and will be set to the default (http and 5601)
  # In case you specify and additional path, the scheme is required: http://localhost:5601/path
  # IPv6 addresses should always be defined as: https://[2001:db8::1]:5601
  #host: "localhost:5601"

  # Optional protocol and basic auth credentials.
  #protocol: "https"
  #username: "elastic"
  #password: "changeme"

  # Optional HTTP Path
  #path: ""

  # Use SSL settings for HTTPS. Default is true.
  #ssl.enabled: true

  # Configure SSL verification mode. If `none` is configured, all server hosts
  # and certificates will be accepted. In this mode, SSL based connections are
  # susceptible to man-in-the-middle attacks. Use only for testing. Default is
  # `full`.
  #ssl.verification_mode: full

  # List of supported/valid TLS versions. By default all TLS versions 1.0 up to
  # 1.2 are enabled.
  #ssl.supported_protocols: [TLSv1.0, TLSv1.1, TLSv1.2]

  # SSL configuration. By default is off.
  # List of root certificates for HTTPS server verifications
  #ssl.certificate_authorities: ["/etc/pki/root/ca.pem"]

  # Certificate for SSL client authentication
  #ssl.certificate: "/etc/pki/client/cert.pem"

  # Client Certificate Key
  #ssl.key: "/etc/pki/client/cert.key"

  # Optional passphrase for decrypting the Certificate Key.
  #ssl.key_passphrase: ''

  # Configure cipher suites to be used for SSL connections
  #ssl.cipher_suites: []

  # Configure curve types for ECDHE based cipher suites
  #ssl.curve_types: []



#================================ Logging ======================================
# There are four options for the log output: file, stderr, syslog, eventlog
# The file output is the default.

# Sets log level. The default log level is info.
# Available log levels are: error, warning, info, debug
#logging.level: info

# Enable debug output for selected components. To enable all selectors use ["*"]
# Other available selectors are "beat", "publish", "service"
# Multiple selectors can be chained.
#logging.selectors: [ ]

# Send all logging output to syslog. The default is false.
#logging.to_syslog: false

# Send all logging output to Windows Event Logs. The default is false.
#logging.to_eventlog: false

# If enabled, filebeat periodically logs its internal metrics that have changed
# in the last period. For each metric that changed, the delta from the value at
# the beginning of the period is logged. Also, the total values for
# all non-zero internal metrics are logged on shutdown. The default is true.
#logging.metrics.enabled: true

# The period after which to log the internal metrics. The default is 30s.
#logging.metrics.period: 30s

# Logging to rotating files. Set logging.to_files to false to disable logging to
# files.
logging.to_files: true
logging.files:
  # Configure the path where the logs are written. The default is the logs directory
  # under the home path (the binary location).
  #path: /var/log/filebeat

  # The name of the files where the logs are written to.
  #name: filebeat

  # Configure log file size limit. If limit is reached, log file will be
  # automatically rotated
  #rotateeverybytes: 10485760 # = 10MB

  # Number of rotated log files to keep. Oldest files will be deleted first.
  #keepfiles: 7

  # The permissions mask to apply when rotating log files. The default value is 0600.
  # Must be a valid Unix-style file permissions mask expressed in octal notation.
  #permissions: 0600

# Set to true to log messages in json format.
#logging.json: false


#============================== Xpack Monitoring =====================================
# filebeat can export internal metrics to a central Elasticsearch monitoring cluster.
# This requires xpack monitoring to be enabled in Elasticsearch.
# The reporting is disabled by default.

# Set to true to enable the monitoring reporter.
#xpack.monitoring.enabled: false

# Uncomment to send the metrics to Elasticsearch. Most settings from the
# Elasticsearch output are accepted here as well. Any setting that is not set is
# automatically inherited from the Elasticsearch output configuration, so if you
# have the Elasticsearch output configured, you can simply uncomment the
# following line, and leave the rest commented out.
#xpack.monitoring.elasticsearch:

  # Array of hosts to connect to.
  # Scheme and port can be left out and will be set to the default (http and 9200)
  # In case you specify and additional path, the scheme is required: http://localhost:9200/path
  # IPv6 addresses should always be defined as: https://[2001:db8::1]:9200
  #hosts: ["localhost:9200"]

  # Set gzip compression level.
  #compression_level: 0

  # Optional protocol and basic auth credentials.
  #protocol: "https"
  #username: "beats_system"
  #password: "changeme"

  # Dictionary of HTTP parameters to pass within the url with index operations.
  #parameters:
    #param1: value1
    #param2: value2

  # Custom HTTP headers to add to each request
  #headers:
  #  X-My-Header: Contents of the header

  # Proxy server url
  #proxy_url: http://proxy:3128

  # The number of times a particular Elasticsearch index operation is attempted. If
  # the indexing operation doesn't succeed after this many retries, the events are
  # dropped. The default is 3.
  #max_retries: 3

  # The maximum number of events to bulk in a single Elasticsearch bulk API index request.
  # The default is 50.
  #bulk_max_size: 50

  # The number of seconds to wait before trying to reconnect to Elasticsearch
  # after a network error. After waiting backoff.init seconds, the Beat
  # tries to reconnect. If the attempt fails, the backoff timer is increased
  # exponentially up to backoff.max. After a successful connection, the backoff
  # timer is reset. The default is 1s.
  #backoff.init: 1s

  # The maximum number of seconds to wait before attempting to connect to
  # Elasticsearch after a network error. The default is 60s.
  #backoff.max: 60s

  # Configure http request timeout before failing an request to Elasticsearch.
  #timeout: 90

  # Use SSL settings for HTTPS.
  #ssl.enabled: true

  # Configure SSL verification mode. If `none` is configured, all server hosts
  # and certificates will be accepted. In this mode, SSL based connections are
  # susceptible to man-in-the-middle attacks. Use only for testing. Default is
  # `full`.
  #ssl.verification_mode: full

  # List of supported/valid TLS versions. By default all TLS versions 1.0 up to
  # 1.2 are enabled.
  #ssl.supported_protocols: [TLSv1.0, TLSv1.1, TLSv1.2]

  # SSL configuration. By default is off.
  # List of root certificates for HTTPS server verifications
  #ssl.certificate_authorities: ["/etc/pki/root/ca.pem"]

  # Certificate for SSL client authentication
  #ssl.certificate: "/etc/pki/client/cert.pem"

  # Client Certificate Key
  #ssl.key: "/etc/pki/client/cert.key"

  # Optional passphrase for decrypting the Certificate Key.
  #ssl.key_passphrase: ''

  # Configure cipher suites to be used for SSL connections
  #ssl.cipher_suites: []

  # Configure curve types for ECDHE based cipher suites
  #ssl.curve_types: []

  # Configure what types of renegotiation are supported. Valid options are
  # never, once, and freely. Default is never.
  #ssl.renegotiation: never

  #metrics.period: 10s
  #state.period: 1m

#================================ HTTP Endpoint ======================================
# Each beat can expose internal metrics through a HTTP endpoint. For security
# reasons the endpoint is disabled by default. This feature is currently experimental.
# Stats can be access through http://localhost:5066/stats . For pretty JSON output
# append ?pretty to the URL.

# Defines if the HTTP endpoint is enabled.
#http.enabled: false

# The HTTP endpoint will bind to this hostname or IP address. It is recommended to use only localhost.
#http.host: localhost

# Port on which the HTTP endpoint will bind. Default is 5066.
#http.port: 5066

#============================= Process Security ================================

# Enable or disable seccomp system call filtering on Linux. Default is enabled.
#seccomp.enabled: true

```
