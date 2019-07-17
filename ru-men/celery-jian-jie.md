---
description: Celery简介
---

# Celery 简介

## 什么是任务队列

任务队列一般用于线程或计算机之间分配工作的一种机制。

任务队列的输入是一个称为任务的工作单元，有专门的工作进行不断的监视任务队列，进行执行新的任务工作。

Celery 通过消息机制进行通信，通常使用中间人（Broker）作为客户端和职程（Worker）调节。启动一个任务，客户端向消息队列发送一条消息，然后中间人（Broker）将消息传递给一个职程（Worker），最后由职程（Worker）进行执行中间人（Broker）分配的任务。

Celery 可以有多个职程（Worker）和中间人（Broker），用来提高Celery的高可用性以及横向扩展能力。

Celery 是用Python编写的，但是协议可以通过任何语言进行实现。迄今，以及有Ruby实现的

Celery 是用 Python 编写的，但协议可以用任何语言实现。出了 Python 语言实现之外，还有Node.js的[node-celery](https://github.com/mher/node-celery)和php的[celery-php](https://github.com/gjedeer/celery-php)。

可以通过暴露 HTTP 的方式进行，任务交互以及其它语言的集成开发。

## 我们需要什么？

Celery 需要消息中间件来进行发送和接收消息。 RabbitMQ 和 Redis 中间人的功能比较齐全，但也支持其它的实验性的解决方案，其中包括 SQLite 进行本地开发。

Celery 可以在一台机器上运行，也可以在多台机器上运行，甚至可以跨数据中心运行。

### 版本要求

Celery 4.0 运行：

* Python ❨2.7,3.4,3.5❩
* PyPy ❨5.4,5.5❩

这是支持 Python2.7 的最后一个版本，从下一个版本Celery5.x开始，需要Python3.5或更高的版本。

如果您的 Python 运行环境比较老，则需要使用旧版本的Celery：

* Python 2.6：Celery 3.1 或更早版本。
* Python 2.5：Celery 3.0 或更早版本。
* Python 2.4：Celery 2.2 或更早版本。

Celery 是一个资金最少的项目，因此我们不支持 Microsoft Windows。请不要打开与该平台相关的任何问题。

## 开始

如果您是第一次使用 Celery，或您使用的是 3.1 之前的版本，建议您阅读入门教程：

* [第一次使用Clery](celery-chu-ci-shi-yong.md)
* [下一步](celery-jin-jie-shi-yong.md)

## Celery 是...

* 简单

Celery 上手比较简单，不需要配置文件就可以直接运行。

它拥有一个庞大的社区，您可以在社区中进行交流问题，也可以通过 IRC 频道或[邮件列表](https://groups.google.com/group/celery-users)进行交流。

这是一个简单的 Demo：

```python
from celery import Celery
app = Celery('hello', broker='amqp://guest@localhost//')
@app.task
def hello():
    return 'hello world'
```

* 高可用

如果出现丢失连接或连接失败，职程（Worker）和客户端会自动重试，并且中间人通过 主/主 主/从 的方式来进行提高可用性。

* 快速

单个 Celery 进行每分钟可以处理数以百万的任务，而且延迟仅为亚毫秒（使用 RabbitMQ、 librabbitmq 在优化过后）。

* 灵活

Celery 的每个部分几乎都可以自定义扩展和单独使用，例如自定义连接池、序列化方式、压缩方式、日志记录方式、任务调度、生产者、消费者、中间人（Broker）等。

### 它支持

* 中间人
  * [RabbitMQ](zhong-jian-ren-brokers/shi-yong-rabbitmq.md)
  * [Redis](zhong-jian-ren-brokers/shi-yong-redis.md)
  * [Amazon SQS](zhong-jian-ren-brokers/shi-yong-amazon-sqs.md)
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

## 功能

* 监控

可以针对整个流程进行监控，内置的工具或可以实时说明当前集群的概况。更多......

* 调度

可以通过调度功能在一段时间内指定任务的执行时间 datetime，也可以根据简单每隔一段时间进行执行重复的任务，支持分钟、小时、星期几，也支持某一天或某一年的Crontab表达式。更多......

* 工作流

可以通过“canvas“进行组成工作流，其中包含分组、链接、分块等等。

简单和复杂的工作流程可以使用一组“canvas“组成，其中包含分组、链接、分块等。更多......

* 资源（内存）泄漏保护

--max-tasks-per-child 参数适用于可能会出现资源泄漏（例如：内存泄漏）的任务。更多......

* 时间和速率的限制

您可以控制每秒/分钟/小时执行任务的次数，或者任务执行的最长时间，也将这些设置为默认值，针对特定的任务或程序进行定制化配置。更多......

* 自定义组件

开发者可以定制化每一个职程（Worker）以及额外的组件。职程（Worker）是用 “bootsteps” 构建的-一个依赖关系图，可以对职程（Worker）的内部进行细粒度控制。

## 框架集成

Celery可以快速的集成一些常用的Web框架，详细如下：

| Web框架 | 集成包 |
| :--- | :--- |
| [Pyramid](http://docs.pylonsproject.org/en/latest/docs/pyramid.html) | [pyramid\_celery](https://pypi.org/project/pyramid_celery/) |
| [Pylons](http://pylonshq.com/) | [celery-pylons](https://pypi.python.org/pypi/celery-pylons/) |
| [Flask](http://flask.pocoo.org/) | 不需要 |
| [web2py](http://web2py.com/) | [web2py-celery](https://pypi.python.org/pypi/web2py-celery/) |
| [Tornado](http://www.tornadoweb.org/) | [tornado-celery](https://pypi.python.org/pypi/tornado-celery/) |
| [Tryton](http://www.tryton.org/) | [celery\_tryton](https://pypi.python.org/pypi/celery_tryton/) |

针对 [Django](https://djangoproject.com/) ，请参考 Django 的初次使用。

集成包并不是必须安全的，但使用它们可以更加快速和方便的开发，有时它们会在 fork\(2\) 中添加例如数据库关闭连接的回调。

## 快速跳转

### 我想要 --&gt; 

* 获取任务执行返回值
* 查看任务存放的队列
* 在任务中使用日志
* 查看一个列表中运行的职程（Worker）
* 最佳的学习实战
* 清空所有任务消息
* 创建一个自定义任务的基类
* 查看职程（Worker）执行的任务
* 为一组任务添加回调
* 将任务迁移到新的中间人（Broker）
* 分割任务为若干份
* 查看事件类型消息
* 优化职程（Worker）
* 给Celery提交贡献
* 查看内置任务状态列表
* 了解Celery的配置参数
* 创建自定义任务状态
* 获取使用Celery的人员和公司
* 自定义设置任务名称
* 跟踪开始任务
* 编写自己的远程控制命令
* 重试失败的任务
* 运行时修改职程（Worker）的队列
* 获取当前执行的任务ID

### 跳转 --&gt;

* [中间人：Broker](zhong-jian-ren-brokers/)
* [职程：Worker](../yong-hu-zhi-nan/zhi-cheng-worker-wen-dang-workers-guide.md)
* [安全：Security](../yong-hu-zhi-nan/an-quan-security.md)
* [贡献：Contributing](../fu-lu/gong-xian-contributing.md)
* [应用：Applications](../yong-hu-zhi-nan/ying-yong-application.md)
* [守护进程：Daemonizing](../yong-hu-zhi-nan/shou-hu-jin-cheng-daemonization.md)
* [路由：Routing](../yong-hu-zhi-nan/lu-you-ren-wu-routing-tasks.md)
* [信号：Signals](../yong-hu-zhi-nan/xin-hao-signals.md)
* [任务：Tasks](../yong-hu-zhi-nan/ren-wu-tasks.md)
* [监控：Monitoring](../yong-hu-zhi-nan/jian-kong-he-guan-li-shou-ce-monitoring-and-management-guide.md)
* [配置：Configuration](../yong-hu-zhi-nan/pei-zhi-he-mo-ren-pei-zhi-configuration-and-defaults.md)
* [FAQ：FAQ](../fu-lu/chang-jian-wen-ti-faqfrequently-asked-questions.md)
* [调用：Calling](../yong-hu-zhi-nan/tiao-yong-ren-wu-calling-tasks.md)
* [优化：Optimizing](../yong-hu-zhi-nan/you-hua-optimizing.md)
* [Django：Django](../fu-lu/django.md)
* [API接口：API Reference](../fu-lu/api-api-reference.md)

## 安装

您可以通过 python 的 pip 安装或通过源代码进行安装 Celery 。

使用pip进行安装

```text
$ pip install -U Celery
```

### 捆绑

Celery 自定义了一组用于安装 Celery 和特定功能的依赖。

您可以在\[\]中加入您需要依赖，可以通过,进行安装多个依赖包。

```bash
$ pip install "celery[librabbitmq]"
$ pip install "celery[librabbitmq,redis,auth,msgpack]"
```

目前发行的如下：

#### 序列化

* celery\[auth\]：使用auth保证程序的安全
* celery\[msgpack\]：使用msgpack序列化
* celery\[yaml\]：使用yaml序列化

**并发**

* celery\[eventlet\]：基于 [eventle](https://pypi.python.org/pypi/eventlet/\) 的并发池
* celery\[gevent\]：基于 [gevent](https://pypi.python.org/pypi/gevent/) 的并发池

**传输和后端**

* celery\[librabbitmq\]：使用librabbitmq库
* celery\[redis\]：使用Redis进行消息传输或后端结果存储
* celery\[sqs\]：使用Amazon SQS进行消息传输（实验阶段）
* celery\[tblib\]：使用 task\_remote\_tracebacks 的功能
* celery\[memcache\]：使用Memcached作为后端结果存储（使用的是[pylibmc](https://pypi.python.org/pypi/pylibmc/)）
* celery\[pymemcache\]：使用Memcached作为后端结果存储（纯Python实现）
* celery\[cassandra\]：使用Apache Cassandra作为后端结果存储
* celery\[couchbase\]：使用CouchBase作为后端结果存储
* celery\[arangodb\]：使用ArangoDB作为后端结果存储
* celery\[elasticsearch\]：使用ElasticSearch作为后端结果存储
* celery\[riak\]：使用Riak作为后端结果存储
* celery\[dynamodb\]：使用AWS DynamoDB作为后端结果存储
* celery\[zookeeper\]：使用Zookeeper进行消息传输
* celery\[sqlalchemy\]：使用SQLlchemy作为后端结果存储（支持）
* celery\[pyro\]：使用Pyro4进行消息传输（实验阶段）
* celery\[slmq\]：使用 SoftLayer Message Queue进行消息传输（实验阶段）
* celery\[consul\]：使用Consul.io Key/Value进行存储传输消息或后端结果存储（实验阶段）
* celery\[django\]：支持比较低的Django版本，不建议您在项目中使用它，它仅供参考

### **下载源代码进行安装**

从 pypi 下载最新版本的 Celery ：

{% embed url="https://pypi.org/project/celery/" caption="PyPI Celery" %}

您可以通过执行以下命令来进行安装

```bash
$ tar xvfz celery-0.0.0.tar.gz
$ cd celery-0.0.0
$ python setup.py build
# python setup.py install
```

如果您没有安装 virtualenv ，最后安装的命令必须使用管理员权限进行安装。

### **使用开发版本**

#### **pip**

使用 Celery 的开发版本需要开发版本的 [kombu](https://pypi.python.org/pypi/kombu/)、[amqp](https://pypi.python.org/pypi/amqp/)、[billiard](https://pypi.python.org/pypi/billiard/) 和 [vine](https://pypi.python.org/pypi/vine/)。

您可以通过 pip 来进行安装：

```bash
$ pip install https://github.com/celery/celery/zipball/master#egg=celery
$ pip install https://github.com/celery/billiard/zipball/master#egg=billiard
$ pip install https://github.com/celery/py-amqp/zipball/master#egg=amqp
$ pip install https://github.com/celery/kombu/zipball/master#egg=kombu
$ pip install https://github.com/celery/vine/zipball/master#egg=vine
```

#### **git**

请查阅“[贡献：Contributing](../fu-lu/gong-xian-contributing.md)”部分。

