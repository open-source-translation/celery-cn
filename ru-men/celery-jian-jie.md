---
description: Celery简介
---

# Celery简介

### 什么是任务队列

任务队列一般用于线程或计算机之间分配工作的一种机制。

任务队列的输入是一个称为任务的工作单元，有专门的工作进行不断的监视任务队列，进行执行新的任务工作。

Celery通过消息机制进行通信，通常使用中间人（Broker）作为客户端和职称（Worker）调节。启动一个任务，客户端向消息队列发送一条消息，然后中间人（Broker）将消息传递给一个职称（Worker），最后由职称（Worker）进行执行中间人（Broker）分配的任务。

Celery可以有多个职称（Worker）和中间人（Broker），用来提高Celery的高可用性以及横向扩展能力。

Celery是用Python编写的，但是协议可以通过任何语言进行实现。迄今，以及有Ruby实现的

Celery 是用 Python 编写的，但协议可以用任何语言实现。出了Python语言实现之外，还有Node.js的[node-celery](https://github.com/mher/node-celery)和php的[celery-php](https://github.com/gjedeer/celery-php)。

 还有通过其它的开发语言来通过，暴露的HTTP端口来请求任务信息进行实现。

### 我们需要什么？

Celery需要消息中间件来进行发送和接收消息。RabbitMQ和Redis中间人的功能比较齐全，但也支持其它的实验性的解决方案，其中包括SQLite进行本地开发。

Celery可以在一台机器上运行，也可以在多台机器上运行，甚至可以跨数据中心运行。

#### 版本要求

Celery4.0运行：

* Python❨2.7,3.4,3.5❩
* PyPy❨5.4,5.5❩

这是支持Python2.7的最后一个版本，从下一个版本Celery5.x开始，需要Python3.5或更高的版本。

如果您的Python运行环境比较老，则需要使用旧版本的Celery：

* Python 2.6：Celery3.1或更早版本。
* Python 2.5：Celery 3.0或更早版本。
* Python 2.4：Celery2.2或更早版本。

Celery是一个资金最少的项目，因此我们不支持Microsoft Windows。请不要打开与该平台相关的任何问题。

### 开始

如果您是第一次使用Celery，或您使用的是3.1之前的版本，建议您阅读入门教程：

* 第一次使用Clery
* 下一步

### Celery 是...

* 简单

Celery比较容易使用和维护，不需要配置文件就可以直接运行。

它拥有一个庞大的社区，您可以在社区中进行交流问题，也可以通过IRC频道或[邮件列表](https://groups.google.com/group/celery-users)进行交流。

这是一个简单的Demo：

```python
from celery import Celery
app = Celery('hello', broker='amqp://guest@localhost//')
@app.task
def hello():
    return 'hello world'
```

* 高可用

如果出现丢失连接或连接失败，职称（Worker）和客户端会自动重试，并且中间人通过 主/主 主/从 的方式来进行提高可用性。

* 快速

单个Celery进行每分钟可以处理数以百万的任务，而且延迟仅为亚毫秒（使用RabbitMQ、librabbitmq在优化过后）。

* 灵活

Celery的每个部分几乎都可以自定义扩展和单独使用，例如自定义连接池、序列化方式、压缩方式、日志记录方式、任务调度、生产者、消费者、中间人（Broker）等。

#### 它支持

* 中间人
  * RabbitMQ
  * Redis
  * Amazon SQS
* 结果存储
  * AMQP、 Redis
  * Memcached
  * SQLAlchemy、Django ORM
  * Apache Cassandra、Elasticsearch
* 并发
  * prefork \(multiprocessing\)
  * [Eventlet](http://eventlet.net/)、[gevent](http://www.gevent.org/)
  * solo \(single threaded\)
* 序列化
  * pickle、json、yaml、msgpack
  * zlib、bzip2 compression
  * Cryptographic message signing

### 功能

### 框架集成

### 快速跳转

### 安装

