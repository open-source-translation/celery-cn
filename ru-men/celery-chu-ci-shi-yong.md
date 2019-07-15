# Celery 初次使用

Celery是一个包含一系列的消息任务队列。您可以不用了解内部的原理直接使用，它的使用时非常简单的。此外Celery可以快速与您的产品扩展与集成，以及Celery提供了一系列Celery可能会用到的工具和技术支持方案。

在本教程中，您将学习Celery的基础支持。

学习如下：

* 选择并且安装一个消息中间件（Broker）
* 安装Celery并且创建第一个任务
* 运行职程（Worker）以及调用任务
* 跟踪任务的情况以及返回值

**译者：**我觉得Celery上手是非常简单的，看完本次教程。建议看看下一节的 Celery 的介绍，你可能会发现很多有趣的功能。

## 选择中间人（Broker）

Celery需要一个中间件来进行接收和发送消息，通常以独立的服务形式出现，成为 消息中间人（Broker）

以下有几种选择：

### RabbitMQ

[RabbitMQ](https://www.rabbitmq.com/)的功能比较齐全、稳定、便于安装。在生产环境来说是首选的，有关 Celery 中使用 RabbitMQ 的详细信息：

[使用RabbitMQ](zhong-jian-ren-brokers/shi-yong-rabbitmq.md)

如果您使用的是Ubuntu或Debian，可以通过以下命令进行安装RabbitMQ：

```bash
$ sudo apt-get install rabbitmq-server
```

如果在Docker中运行RabbitMQ，可以使用以下命令：

```bash
$ docker run -d -p 5462:5462 rabbitmq
```

命令执行完毕之后，中间人（Broker）会在后台继续运行，准备输出一条 _Starting rabbitmq-server: SUCCESS_ 的消息。

如果您没有Ubuntu或Debian，你可以访问官方网站查看其他操作系统（如：Windows）的安装方式：

[http://www.rabbitmq.com/download.html](http://www.rabbitmq.com/download.html)

### Redis

[Redis](https://redis.io/)功能比较全，但是如果突然停止运行或断电会造成数据丢失，有关 Celery 中使用 Redis 的详细信息：

[使用Redis](zhong-jian-ren-brokers/shi-yong-redis.md)

在Docker中运行Redis，可以通过以下命令实现：

```bash
$ docker run -d -p 6379:6379 redis
```

### 其他中间人（Broker）

除以上提到的中间人（Broker）之外，还有处于实验阶段的中间人（Broker），其中包含[ Amazon SQS](zhong-jian-ren-brokers/shi-yong-amazon-sqs.md)。

相关完整的中间人（Broker）列表，请查阅[中间人（Broker）概况](zhong-jian-ren-brokers/#zhong-jian-ren-broker-gai-kuang)。

## 安装 Celery

Celery 在python的PyPI中管理，可以使用 pip 或 easy\_install 来进行安装：

```bash
$ pip install celery
```

## 应用

创建第一个Celery实例程序，我们把创建Celery程序成为Celery 应用或直接简称 为 app，创建的第一个实例程序可能需要包含Celery中执行操作的所有入口点，例如创建任务、管理职程（Worker）等，所以必须要导入Celery模块。

在本教程中将所有的内容，保存为一个app文件中。针对大型的项目，可能需要创建 独立的模块。

首先创建 tasks.py：

{% code-tabs %}
{% code-tabs-item title="tasks.py" %}
```python
from celery import Celery
app = Celery('tasks', broker='amqp://guest@localhost//')
@app.task
def add(x, y):
    return x + y
```
{% endcode-tabs-item %}
{% endcode-tabs %}

第一个参数为当前模块的名称，只有在 \_\_main\_\_ 模块中定义任务时才会生产名称。

第二个参数为中间人（Broker）的链接URL，实例中使用的RabbitMQ（Celery默认使用的也是RabbitMQ）。

更多相关的Celery中间人（Broker）的选择方案，可查阅上面的中间人（Broker）。例如，对于 RabbitMQ 可以写为 amqp://localhost ，使用 Redis 可以写为 redis://localhost。

创建了一个名称为add的任务，返回的俩个数字的和。

**译者：**我比较喜欢使用Redis作为中间人（Broker），对于新上手的建议使用Redis作为中间人（Broker），因为我觉得Redis比RabbitMQ好用一点。

## 运行 Celery 职程（Worker）服务

现在可以使用 worker 参数进行执行我们刚刚创建职程（Worker）：

```text
$ celery -A tasks worker --loglevel=info
```

{% hint style="warning" %}
## 注意：

如果职程（Worker）没有正常启动，请查阅 “故障排除”相关分布。
{% endhint %}

在生产环境中，如果需要将职程（Worker）作为守护进程在后台运行，可以使用平台提供的工具来进行实现，或使用类似 supervisord 这样的工具来进行管理（详情：[守护进程（Daemonization）](../yong-hu-zhi-nan/shou-hu-jin-cheng-daemonization.md)部分）

关于 Celery 可用的命令完整列表，可以通过以下命令进行查看：

```bash
$  celery worker --help
```

也可以通过以下命令查看一些 Celery 帮助选项：

```bash
$ celery help
```

## 调用任务

需要调用我们创建的实例任务，可以通过 delay\(\) 进行调用。

delay\(\) 是 apply\_async\(\) 的快捷方法，可以更好的控制任务的执行（详情：[调用任务（Calling Tasks）](../yong-hu-zhi-nan/tiao-yong-ren-wu-calling-tasks.md)）：

```bash
>>> from tasks import add
>>> add.delay(4, 4)
```

该任务已经有职程（Worker）开始处理，可以通过控制台输出的日志进行查看执行情况。

调用任务会返回一个 AsyncResult 的实例，用于检测任务的状态，等待任务完成获取返回值（如果任务执行失败，会抛出异常）。默认这个功能是不开启的，如果开启则需要配置 Celery 的结果后端，下一小节会详细说明。

## 保存结果

如果您需要跟踪任务的状态，Celery 需要在某处存储任务的状态信息。Celery 内置了一些后端结果：[SQLAlchemy/Django](https://www.sqlalchemy.org/) ORM、[Memcached](http://memcached.org/)、[Redis](https://redis.io/)、 RPC \([RabbitMQ](https://www.rabbitmq.com/)/AMQP\)以及自定义的后端结果存储中间件。

针对本次实例，我们使用RPC作为结果后端，将状态信息作为临时消息回传。后端通过 backend 参数指定给 Celery（或者通过配置模块中的 result\_backend 选项设定）：

```text
app = Celery('tasks', backend='rpc://', broker='pyamqp://')
```

可以使用Redis作为Celery结果后端，使用RabbitMQ作为中间人（Broker）可以使用以下配置（这种组合比较流行）：

```text
app = Celery('tasks', backend='redis://localhost', broker='pyamqp://')
```

更多关于后端结果配置，请查阅[任务结果后端](../yong-hu-zhi-nan/ren-wu-tasks.md)。

现在已经配置结果后端，重新调用执行任务。会得到调用任务后返回的一个 AsyncResult 实例：

```bash
>>> result = add.delay(4, 4)
```

ready\(\) 可以检测是否已经处理完毕：

```bash
>>> result.ready()
False
```

整个任务执行过程为异步的，如果一直等待任务完成，会将异步调用转换为同步调用：

```bash
>>> result.get(timeout=1)
8
```

如果任务出现异常，get\(\) 会再次引发异常，可以通过 propagate 参数进行覆盖：

```bash
>>> result.get(propagate=False)
```

如果任务出现异常，可以通过以下命令进行回溯：

```bash
>>> result.traceback
```

{% hint style="warning" %}
## 警告：

如果后端使用资源进行存储结果，必须要针对调用任务后返回每一个 AsyncResult 实例调用 get\(\) 或 forget\(\) ，进行资源释放。
{% endhint %}

完整的结果对象，请参阅：celery.result。

## 配置

## 接下来干什么

## 故障处理

### 职程（Worker）无法正常启动：权限错误

### 任务总处于 PENDING （待处理）状态

