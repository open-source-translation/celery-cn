---
description: "Release:\t4.3"
---

# 中间人：Brokers

Celery 支持多种消息传输的方式。

## 中间人（Broker）使用指南

* 使用 [RabbitMQ](shi-yong-rabbitmq.md)
* 使用 [Redis](shi-yong-redis.md)
* 使用 [Amazon SQS](shi-yong-amazon-sqs.md)

## 中间人（Broker）概况

这是不同的中间件比对情况，更多的信息可以在每个中间件的文档中找到（见众人指南）。

| 名称         | 状态   | 监控 | 远程控制 |
| ---------- | ---- | -- | ---- |
| RabbitMQ   | 稳定   | 是  | 是    |
| Redis      | 稳定   | 是  | 是    |
| Amazon SQS | 稳定   | 否  | 否    |
| Zookeeper  | 实验阶段 | 否  | 否    |

目前实验阶段的中间人（Broker）只是功能性的，但是没有专门的维护人员。

缺少监控就意味着这个监控已经失效，因此相关的 Flower、Celery events、celerymon 和其他基于此功能的监控工具全部失效。

远程管理控制是指可以通过 celery inspect 和 celery control（以及使用远程控制API的工具）在程序运行时检查和管理职程（Worker）的能力。
