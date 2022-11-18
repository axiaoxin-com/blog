---
title: "搭建 Filebeat Elasticsearch Kibana V6.3.0 日志监控"
author: "axiaoxin"
date: 2018-07-05T16:48:56+08:00
subtitle: ""
image: ""
tags: ["elasticsearch"]
slug: ""
---

## Filebeat+Elasticsearch+Kibana 环境搭建

上次搭建 ELK 还是两年前，现在使用当时的方法发现不行了，不得不说 ELK 的版本更新速度真的很快。

本文记录使用 ELK Stack 最新的 6.3.0 版本，使用 filebeat 进行日志收集，统一存储到 elasticsearch 并用 kibana 展示的搭建过程。

```text
filebeat \

filebeat -  elasticsearch - kibana - browser

filebeat /
```


在统一日志存放的机器上安装 elasticsearch 和 kibana

kibana可以使用rpm包直接安装：

```sh
rpm -ivh kibana-6.3.0-x86_64.rpm
```

默认配置文件在 `/etc/kibana/` 下，修改以下配置，其余配置按实际情况修改

```yaml
server.port: 5601
server.host: "YOUR_HOST_IP"
elasticsearch.url: "http://YOUR_ELASTICSEARCH_IP:9200"
```

重启:

```sh
service kibana restart
```

访问:

```sh
curl http://10.120.93.150:5601/api/status
```

日志：

```sh
tail -f /var/log/kibana/kibana.std*
```

elasticsearch 不能使用 root 运行，所以使用 tar 包安装，切换为非 root 用户


```sh
tar xzf elasticsearch-6.3.0.tar.gz
```

配置文件在 `elasticsearch-6.3.0/config/` 下，修改一下配置，其余按实际情况修改

```yaml
path.data: /path/of/data
path.logs: /path/of/save/logs

bootstrap.system_call_filter: false

network.host: YOUR_HOST_IP
http.port: 9200
```

启动服务：

```sh
(./bin/elasticsearch &)
```


在需要收集日志的机器上可以直接使用rpm包直接安装 filebeat

```sh
rpm -ivh filebeat-6.3.0-x86_64.rpm
```

默认配置文件在 `/etc/filebeat/` 下，修改以下配置，其余配置按实际情况修改

```yaml
filebeat.inputs:
- type: log

    # Change to true to enable this input configuration.
    enabled: true

    paths:
    - /your/log/*.log


setup.kibana:
    host: "your.kibana.host:5601"

output.elasticsearch:
    hosts: ["your.elasticsearch.host:9200"]
```


重启服务：

```sh
service filebeat restart
```

可以查看日志：

```sh
tail -f /var/log/filebeat/filebeat
```

至此，整个环境已经搭建完成。


## Filebeat 使用 Elasticsearch 的 Pipeline 处理日志内容

以前使用 Logstash 时，都是通过 logstash 来对日志内容做过滤解析等操作，
现在6.3.0版本中，可以通过 filebeat 直接写数据到 es 中，要对日志内容做处理的话设置对应的 pipeline 就可以。

以gunicorn的access日志内容为例：

```text
[05/Jul/2018:11:13:55 +0800] 10.242.169.166 "-" "Jakarta Commons-HttpClient/3.0" "POST /message/entry HTTP/1.0" 200 <13968> 0.061638
```

有以上内容的日志，记录请求发生的时间，发起请求的 ip，referer，useragent，status_line， status_code, 进程id， 请求执行时间。

在不使用 pipeline 的情况下，在 kibana 中可以看到日志内容在 message 字段中，如果想要获取每个请求的执行时间是拿不到的。

使用 pipeline，需要现在 es 中增加对应的 pipeline，可以在 kibana 的 devtools 界面的 console 中写入，然后点击执行即可

```sh
PUT _ingest/pipeline/gunicorn-access
{
    "description" : "my gunicorn access log pipeline",
    "processors": [
        {
            "grok": {
                "field": "message",
                "patterns": ["\\[%{HTTPDATE:timestamp}\\] %{IP:remote} \"%{DATA:referer}\" \"%{DATA:ua}\" \"%{DATA:status_line}\" %{NUMBER:status_code:int} <%{NUMBER:process_id:int}> %{NUMBER:use_time:float}"]
            }
        }
    ]
}
```

processor 使用的 grok，主要是 patterns 的编写，es 的[默认正则 pattern](https://github.com/elastic/elasticsearch/blob/6.3/libs/grok/src/main/resources/patterns/grok-patterns) 可以直接使用。注意 JSON 转义符号。

NUMBER 类型最好指定是 int 还是 float，不然默认是 string，搜索的时候可能会有问题。

在写 patterns 的时候，可以借助 devtools 界面的 grokdebugger 工具检测是否正确。测试无误即可执行 put 操作。完成后修改 filebeat 配置

inputs 中设置 type 字段用于判断

```yaml
- type: log
  enabled: true
  paths:
    - /path/to/gunicorn-access.log
  fields:
    type: gunicorn-access
  multiline.pattern: ^\[
  multiline.negate: true
  multiline.match: after
```

es output 中添加

```yaml
pipelines:
  - pipeline: gunicorn-access
    when.equals:
      fields.type: gunicorn-access
```

重启。


在开启自带的 nginx 日志监控时，nginx 的错误日志时间会比当前时间快8小时，需要对它的 pipeline 设置时区，设置方法为：

通过 `GET _ingest/pipeline/` 先找到 nginx error log 的 pipeline 名字为：`filebeat-6.3.0-nginx-error-pipeline`

复制他的 pipeline 配置，在 date 字段下添加 timezone 配置

```json
{
    "date": {
        "field": "nginx.error.time",
        "target_field": "@timestamp",
        "formats": [
        "YYYY/MM/dd H:m:s"
        ],
        "timezone": "Asia/Shanghai"
    }
}
```

然后将新的完整 pipeline put 到 es 中 `PUT _ingest/pipeline/filebeat-6.3.0-nginx-error-pipeline` 。注意，需重启es才能生效。
