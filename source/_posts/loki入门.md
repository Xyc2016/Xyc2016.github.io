---
title: loki入门
date: 2023-02-04 23:00:00
---
日志对于排查问题很重要。对于小型，低成本项目，可以考虑使用loki，grafana生态中的一个开源的日志聚合系统。本次我们来学习loki。
## 了解loki
[Loki overview](https://grafana.com/docs/loki/latest/fundamentals/overview/)
Loki 是一个高效储存log数据的储存。但和其它日志系统不一样的是，Loki的index是从label构建的，log信息没有index。
### Loki日志收集的原理
一个客户端获取日志，把日志变成流，然后把流通过HTTP发送到Loki。
### Loki的特点
1. 索引日志时的高效内存利用
2. 多租户。多个客户端可以利用一个Loki实例。
3. LogQL。Loki的查询语言。
4. 扩展性。Loki可以单binary运行，所有组件在一个进程中。
5. 灵活性。很多客户端支持插件，这使得它们可以新增Loki作为他们的日志聚合工具。
6. Grafana。Loki和Grafana无缝集成。
## 安装和部署loki
它可以部署到多种环境：k8s，docker等。
我们本次学习docker安装，学习两种使用方式：收集本地的log和收集docker的log。
### 收集本地的log
运行以下命令，即可搭建本地的日志收集服务。
```
wget https://raw.githubusercontent.com/grafana/loki/v2.7.2/production/docker-compose.yaml -O docker-compose.yaml
docker-compose -f docker-compose.yaml up
```
我们打开上面的链接，可以看出此compose有3个组件。loki，promtail，grafana。promtail把日志发送到loki，grafana展示loki的数据。
另外，grafana端口是3000。loki端口是3100。
打开后，账号密码都输入admin，登录。
然后点击Configuration，选择Data Sources，点击Add data source，选择Loki，输入URL，http://loki:3100，点击Save and test。这样就添加了loki数据源。
然后点击Explore就可以看日志了。在loki数据源里选择任意文件名，例如/var/log/docker.log，点击Run query，就可以看到docker的日志了。

（docker loki driver有个大坑需要注意一下，在此plugin为激活时，无法停止loki container。需要先disable plugin 才能停止loki container）

然而，收集和查看本地日志，意义不大。我们学习一下如何收集docker的日志。这样可以根据image等信息进行搜索。
### 收集docker的log
这种方式需要给docker设置相应的log driver。
先在docker中安装loki-docker-driver
```
docker plugin install grafana/loki-docker-driver:latest --alias loki --grant-all-permissions
```
配置log driver
```json
{
  "log-driver": "loki",
  "debug": true,
  "log-opts": {
        "loki-url": "http://localhost:3100/loki/api/v1/push",
        "loki-batch-size": "400"
  }
}
```
这样docker的默认log driver就是loki了。另外我们再以上文件中指定了loki的和端口为localhost:3100。
修改docker-compose.yaml，把promtail去掉，另外增加一个专门产生log的服务。
```yaml
version: "3"

networks:
  loki:

services:
  loki:
    image: grafana/loki:latest
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - loki

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    networks:
      - loki

  flog:
    image: mingrammer/flog
    command: -f json -d 1s -l
    networks:
      - loki
```
更新此compose
```sh
docker compose -f docker-compose.yaml -d
```
就可以打开grafana看日志了
## 日志查询
grafana的日志查询又是一个大坑。同样一段query，从其它地方复制过来可以，人工输入再点击查询就可能报错。
