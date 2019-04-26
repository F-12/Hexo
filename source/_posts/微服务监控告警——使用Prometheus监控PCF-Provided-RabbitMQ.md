---
title: 微服务监控告警——使用Prometheus监控PCF Provided RabbitMQ
date: 2019-04-25 14:25:58
tags:
    - Monitoring
    - 监控
    - 微服务
    - Prometheus
    - RabbitMQ
---

# 背景

Cloud Foundry中的RabbitMQ是以Service的形式提供，我们需要监控关于RabbitMQ运行状况的一些指标。  
监控系统选择使用Prometheus作为存储引擎，但是Prometheus无法直接访问部署在云环境上的PCF里的RabbitMQ。

# 方案
1. 在PCF上创建RabbitMQ的service-key，提供一个可以访问RabbitMQ HTTP API的账号
```
cf create-service-key My-RabbitMQ-Service rabbitmq_exporter
```

2. 使用如下命令获取RabbitMQ各个节点的元数据
```
cf service-key My-RabbitMQ-Service rabbitmq_exporter
```
数据如下形式：
```json
{
 "dashboard_url": "https://rmq-d9a9c1ab-2383-400b-b5e8-3b84dabe221a.sys.example.cf.company.com/#/login/HOSTNAME/PASSWORD",
 "hostname": "10.227.200.200",
 "hostnames": [
  "10.227.200.200"
 ],
 "http_api_uri": "https://HOSTNAME:PASSWORD@rmq-d9a9c1ab-2383-400b-b5e8-3b84dabe221a.sys.example.cf.company.com/api/",
 "http_api_uris": [
  "https://HOSTNAME:PASSWORD@rmq-d9a9c1ab-2383-400b-b5e8-3b84dabe221a.sys.example.cf.company.com/api/"
 ],
 "password": "PASSWORD",
 "protocols": {
  "amqp": {
   "host": "10.227.200.200",
   "hosts": [
    "10.227.200.200"
   ],
   "password": "PASSWORD",
   "port": 5672,
   "ssl": false,
   "uri": "amqp://HOSTNAME:PASSWORD@10.227.200.200/d9a9c1ab-2383-400b-b5e8-3b84dabe221a",
   "uris": [
    "amqp://HOSTNAME:PASSWORD@10.227.200.200/d9a9c1ab-2383-400b-b5e8-3b84dabe221a"
   ],
   "username": "HOSTNAME",
   "vhost": "d9a9c1ab-2383-400b-b5e8-3b84dabe221a"
  }
 },
 "ssl": false,
 "uri": "amqp://HOSTNAME:PASSWORD@10.227.200.200/d9a9c1ab-2383-400b-b5e8-3b84dabe221a",
 "uris": [
  "amqp://HOSTNAME:PASSWORD@10.227.200.200/d9a9c1ab-2383-400b-b5e8-3b84dabe221a"
 ],
 "username": "HOSTNAME",
 "vhost": "d9a9c1ab-2383-400b-b5e8-3b84dabe221a"
}
```

3. 在Host `192.168.8.101`上对每一个RabbitMQ节点部署一个rabbitmq_exporter应用，提供prometheus访问的metrics endpoint

```
 RABBIT_URL=https://HOSTNAME:PASSWORD@rmq-d9a9c1ab-2383-400b-b5e8-3b84dabe221a.sys.example.cf.company.com RABBIT_USER=HOSTNAME RABBIT_PASSWORD=PASSWORD PUBLISH_PORT=9419 ./rabbitmq_exporter
```

4. 在Prometheus Server的配置中增加rabbitmq_exporter的抓取配置
```
scrape_configs:
  - job_name: 'rabbitmq_exporter'
    metrics_path: '/metrics'
    scrape_interval: 15s
    static_configs:
    - targets: ['192.168.8.101:9419']
```

# 监控指标
结合RabbitMQ自身Metrics，制定如下监控项：

- 存活状态
- 存活节点数
- Exchange总数
- Queues总数
- 连接数
- 每个queue中待处理的Message数目
- 每个queue中处理中的message数目
- RabbitMQ Server的内存使用情况
- 每个Exchange中的rabbitmq_exchange_messages_published_in_total变化趋势
- 每个Exchange中的rabbitmq_exchange_messages_published_out_total变化趋势
- 特定Exchange中每个queue的rabbitmq_queue_consumers数目变化趋势
- 特定Exchange中每个queue的rabbitmq_queue_disk_reads_total数目变化趋势
- 特定Exchange中每个queue的rabbitmq_queue_disk_writes_total数目变化趋势
- 特定Exchange中每个queue分配的内存数目
- 特定Exchange中每个queue中message占用的内存数目

更多指标可以参见rabbitmq_exporter的metrics列表。
