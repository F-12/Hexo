title: 微服务监控告警——使用Grafana Webhook通知Webex Teams
toc: true
thumbnail: /images/banner.jpg
tags:
  - 微服务
  - ALert
  - 告警
nanoid: vV8IHvoG4kY713wL_xRqI
date: 2019-04-26 18:06:11
---

# 背景

Grafana支持Alert，但是Notification Channel不支持Webex Teams


# Alert方案

1. 使用Grafana Webhook
2. 实现一个Adapter，接受Grafana通知
3. Apdater接受通知，再发送Webex Teams通知


# Alert反馈方案

1. 在Webex Teams里回复消息，指定回复模式触发Webex Teams Hook
2. Webex Teams Adpater负责将消息翻译为对特定alert的暂停、重置等功能
3. Adpater可以提供Alert管理功能，比如记录谁处理哪个时间的哪个alert，以及处理结果，和root cause分析