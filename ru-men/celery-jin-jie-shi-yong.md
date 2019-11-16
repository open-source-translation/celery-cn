# Celery 进阶使用

[`Celery 初次使用`](celery-chu-ci-shi-yong.md)章节简单的说明了一下 Celery 的基本使用，本章节将更详细的介绍和使用 Celery，其中包含在自己的应用程序中和库中使用 Celery。

本章节结尾记录有 Celery 的所有功能和最佳实战，建议继续阅读“用户指南”。

##  在应用程序使用

### 我们的项目

项目的结构：

```bash
proj/__init__.py
    /celery.py
    /tasks.py
```

proj/celery.py

{% code title="proj/celery.py" %}
```python
from __future__ import absolute_import, unicode_literals
from celery import Celery

app = Celery('proj',
             broker='amqp://',
             backend='amqp://',
             include=['proj.tasks'])

# Optional configuration, see the application user guide.
app.conf.update(
    result_expires=3600,
)

if __name__ == '__main__':
    app.start()
```
{% endcode %}

在此程序中，创建了 Celery 实例（也称 `app`），如果需要使用 Celery，导入即可。

* 该 broker 参数为指定的中间人（Broker）URL。

  有关更多信息，请查阅[`选择中间人`](celery-chu-ci-shi-yong.md#xuan-ze-zhong-jian-ren-broker) 。

* 该 backend 参数为指定的结果后端 URL。

  它主要用于跟踪任务的状态的信息，默认情况下禁用结果后端，在此实例中已经开启了该功能，主要便于后续检索，可能在会在程序中使用不同的结果后端。每一个后端都有不同的优点和缺点，如果不需要结果，最好禁用。也可以通过设置 `@task(ignore_result=True)` 参数，针对单个任务禁用。

  有关更多详细材料，请参阅[`保存结果`](celery-chu-ci-shi-yong.md#bao-cun-jie-guo) 。

* 该 include 参数是程序启动时倒入的模块列表，可以该处添加任务模块，便于职程能够对应相应的任务。

proj/tasks.py

{% code title="proj/tasks.py" %}
```python
from __future__ import absolute_import, unicode_literals
from .celery import app

@app.task
def add(x, y):
    return x + y

@app.task
def mul(x, y):
    return x * y

@app.task
def xsum(numbers):
    return sum(numbers)
```
{% endcode %}

### 运行职程（Worker）

Celery 程序可以用于启动职程（Worker）：

```bash
$ celery -A proj worker -l info
```

当职程（Worker）开始运行时，可以看到一部分日志消息：

```bash
-------------- celery@halcyon.local v4.0 (latentcall)
---- **** -----
--- * ***  * -- [Configuration]
-- * - **** --- . broker:      amqp://guest@localhost:5672//
- ** ---------- . app:         __main__:0x1012d8590
- ** ---------- . concurrency: 8 (processes)
- ** ---------- . events:      OFF (enable -E to monitor this worker)
- ** ----------
- *** --- * --- [Queues]
-- ******* ---- . celery:      exchange:celery(direct) binding:celery
--- ***** -----

[2012-06-08 16:23:51,078: WARNING/MainProcess] celery@halcyon.local has started.
```

* _broker_ 为 Celery 程序中指定的中间人（Broker）的连接URL，也可以通过 `-b` 选项在命令行进行设置其他的中间人（Broker）。
* _concurrency_ 为同时处理任务的工作进程数量，所有的进程都被占满时，新的任务需要进行等待其中的一个进程完成任务才能执行进行任务。

默认的并发数为当前计算机的 CPU 数，可以通过设置 `celery worker-c` 项进行自定义设置并发数。没有推荐的并发数，因为最佳的并发数取决于很多因素，如果任务主要是 I/O 限制，可以进行增加并发数，经过测试，设置超过两倍的 CPU 数量效果不是很好，很有可能会降低性能。

包括默认的 prefork 池，Celery 也支持在单个线程中使用 Eventlet、Gevent。（详情参阅：[`并发：Concurrency`](../yong-hu-zhi-nan/bing-fa-concurrency.md)）

* Events 选项设置为启用状态时， Celery 会开启监控事件来进行监视 职程（Worker）。一般情况用于监控程序，如 Flower 和 实时 Celery 监控，详情参阅 [`监控和管理手册：Monitoring and Management Guide`](../yong-hu-zhi-nan/jian-kong-he-guan-li-shou-ce-monitoring-and-management-guide.md)。
* Queues 为职程（Worker）任务队列，可以告诉职程（Worker）同时从多个任务队列中进行消费。通常用于将任务消息路由到特定的职程（Worker）、提升服务质量、关注点分离、优先级排序的常用手段。详情参阅 [`路由任务：Routing Tasks`](../yong-hu-zhi-nan/lu-you-ren-wu-routing-tasks.md)。

可以通过 --help 参数进行查看完整的使用列表：

```bash
$ celery worker --help
```

[`职程（Worker）文档：Workers Guide`](../yong-hu-zhi-nan/zhi-cheng-worker-wen-dang-workers-guide.md) 章节中详细描述了配置项的使用。

### 停止职程（Worker）

使用 `Control + c` 就可以停止职程（Worker），`职程（Worker）文档：Workers Guide` 章节详细的描述了职程（Worker）支持的信号列表。

### 后台运行

在生产环境中，如果需要后台运行职程（Worker），可以参阅 [`守护进程：Daemonization`](../yong-hu-zhi-nan/shou-hu-jin-cheng-daemonization.md)。

可以使用 `celery multi` 命令在后台启动一个或多个职程（Worker）：

```bash
$ celery multi start w1 -A proj -l info
celery multi v4.0.0 (latentcall)
> Starting nodes...
    > w1.halcyon.local: OK
```

也可以进行重启：

```bash
$ celery  multi restart w1 -A proj -l info
celery multi v4.0.0 (latentcall)
> Stopping nodes...
    > w1.halcyon.local: TERM -> 64024
> Waiting for 1 node.....
    > w1.halcyon.local: OK
> Restarting node w1.halcyon.local: OK
celery multi v4.0.0 (latentcall)
> Stopping nodes...
    > w1.halcyon.local: TERM -> 64052
```

停止运行：

```bash
$ celery multi stop w1 -A proj -l info
```

`stop` 命令是异步的，所以不会等待职程（Worker）关闭。可以通过 `stopwait` 命令进行停止运行，可以保证在退出之前完成当前正在执行的任务：

```bash
$ celery multi stopwait w1 -A proj -l info
```

{% hint style="warning" %}
## 注意：

`celery multi` 不存储有关职程（Worker）的信息，所以在重新启动时需要使用相同的命令参数，停止运行时只能通过`pidfile`和`logfile`参数来进行停止运行。
{% endhint %}

默认情况下会在当前目录中创建pid文件和日志文件，为防止多个职程（Worker）干扰，建议将这些文件存放在专门的目录中：

```bash
$ mkdir -p /var/run/celery
$ mkdir -p /var/log/celery
$ celery multi start w1 -A proj -l info --pidfile=/var/run/celery/%n.pid \
                                        --logfile=/var/log/celery/%n%I.log
```

也可以使用 `multi` 命令启动多个职程（Worker），有一个强大的语法为不同职程（Worker）设置不同的参数：

```bash
$ celery multi start 10 -A proj -l info -Q:1-3 images,video -Q:4,5 data \
    -Q default -L:4,5 debug
```

更多实例，可也参阅 `multi` API 模块。

### 关于 --app  参数

使用 --app 参数可也指定运行的 Celery 应用程序实例，格式必须为 `module.path:attribute`

但如果只设置包名，它将进行搜索app实例，顺序如下：

用 --app=proj：

1. 名为 `proj.app` 的属性
2. 名为 `proj.celery` 的属性
3. 模块 `proj` 中值为 Celery 应用程序的任何属性，如果还没有找到，将尝试检索名为 `proj.celery`的子模块
4. 名为 `proj.celery.app` 的属性
5. 名为 `proj.celery.celery` 的属性
6. 模块 `proj.celery` 中值为 Celery 应用程序的任何属性
7. 在此方案模仿文档中使用的实例，即 针对单个模块包含的`proj:app` ，以及 大型项目的 `proj.celery:app`

## 程序调用

你可以使用 `delay()` 方法进行调用：

```bash
>>> add.delay(2, 2)
```

`delay()` 实际上为 `apply_async()` 的快捷使用：

```bash
>>> add.apply_async((2, 2))
```

`apply_async()` 可以指定调用时执行的参数，例如运行的时间，使用的任务队列等：

```bash
>>> add.apply_async((2, 2), queue='lopri', countdown=10)
```

上面的实例中，任务被下发到 `lopri` 队列中，任务下发之后会在最早10秒内执行。 直接调用任务函数进行执行任务，不会发送任何任务消息：

```bash
>>> add(2, 2)
4
```

`delay()` `apply_async()` 以及 `apply(__call__)` 为 Celery 调用的API，也可以用于签名。 

[`调用任务：Calling Tasks`](../yong-hu-zhi-nan/tiao-yong-ren-wu-calling-tasks.md) 章节中详细的描述了 Calling API 使用。 

每一个任务被调用时会赋值一个的任务ID（UUIID）。

 `delay()` 和 `apply_async()` 方法会返回一个 AsyncResult 实例，可以用于进行跟踪任务状况。如果进行跟踪任务状态，需要设置一个结果后端，以便于存储。

 默认情况下禁用结果，因为没有一个适合所有应用程序的结果后端，对于大量的任务来说，保存返回内容不是非常有用的，所以该默认值是一个比较合理的。另外，结果后端不是用于监控任务以及职程（Worker），Celery 有专用的事物消息来进行监控（详情请参阅：[`监控和管理手册：Monitoring and Management Guide`](../yong-hu-zhi-nan/jian-kong-he-guan-li-shou-ce-monitoring-and-management-guide.md)）。

 如果配置了结果后端，可以获取任务的返回值：

```bash
>>> res = add.delay(2, 2)
>>> res.get(timeout=1)
4
```

也可以通过 `id` 属性进行获取任务的ID：

```bash
>>> res.id
d6b3aea2-fb9b-4ebc-8da4-848818db9114
```

如果任务执行引发异常，可以进行检查异常以及溯源，默认情况下 `result.get()` 会抛出异常：

```bash
>>> res = add.delay(2)
>>> res.get(timeout=1)
```

```bash
Traceback (most recent call last):
File "<stdin>", line 1, in <module>
File "/opt/devel/celery/celery/result.py", line 113, in get
    interval=interval)
File "/opt/devel/celery/celery/backends/rpc.py", line 138, in wait_for
    raise meta['result']
TypeError: add() takes exactly 2 arguments (1 given)
```

如果不希望 Celery 抛出异常，可以通过设置 `propagate` 来进行禁用：

```bash
>>> res.get(propagate=False)
TypeError('add() takes exactly 2 arguments (1 given)',)
```

在这种情况下，他可以返回引发错误的实例，需要检查任务是否执行成功还是失败，可以通过在结果实例中使用对应的方法：

```bash
>>> res.failed()
True

>>> res.successful()
False
```

如何知道任务是否执行失败？可以通过查看任务的 state 进行查看：

```bash
>>> res.state
'FAILURE'
```

一个任务只能有当前只能有一个状态，但他的执行过程可以为多个状态，一个典型的阶段是：

```bash
PENDING -> STARTED -> SUCCESS
```

启动状态是一种比较特殊的状态，仅在 `task_track_started` 启用设置或 `@task(track_started=True)`的情况下才会进行记录。 挂起状态实际上不是记录状态，而是未知任务ID的默认状态，可以从此实例中看到：

```bash
>>> from proj.celery import app

>>> res = app.AsyncResult('this-id-does-not-exist')
>>> res.state
'PENDING'
```

重试任务比较复杂，为了证明，一个任务会重试两次，任务的阶段为：

```bash
PENDING -> STARTED -> RETRY -> STARTED -> RETRY -> STARTED -> SUCCESS
```

更多任务状态信息可以查阅用户指南中的[`任务：Tasks`](../yong-hu-zhi-nan/ren-wu-tasks/)章节的`State`部分内容。

更多调用任务信息可以参阅[`调用任务：Calling Tasks`](../yong-hu-zhi-nan/tiao-yong-ren-wu-calling-tasks.md)。

## Canvas：设计工作流程

通过上面的实例学了使用 `delay` 方法进行调用任务，有时候可能希望将任务调用的签名传递给另外一个进程或其他函数的参数，Celery 提供了一共交签名的东西。

签名通过一种方式进行封装任务调用的参数以及执行选项，便于传递给他的函数，甚至通过序列化通过网络传送。



可以将 add 使用的参数作为任务创建的签名，倒计时为 10 秒，如下所示（2,2）：

```bash
>>> add.signature((2, 2), countdown=10)
tasks.add(2, 2)
```

也可以通过一个快捷的方式进行操作：

```bash
>>> add.s(2, 2)
tasks.add(2, 2)
```

### 再次调用API ...

签名实例支持调用API：这就意味着可以使用 `delay` 和 `apply_async` 方法。 但区别就在于签名实例已经指定了参数签名，该 add 任务有两个参数，需要指定两个参数的签名才能够成一个完整的签名实例：

```bash
>>> s1 = add.s(2, 2)
>>> res = s1.delay()
>>> res.get()
4
```

也可以创建不完整的签名来进行创建，我称之为 `partials` 的内容：

```bash
# incomplete partial: add(?, 2)
>>> s2 = add.s(2)
```

s2 为一个不完整的签名，需要另外一个参数，可以通过调用签名解决：

```bash
# resolves the partial: add(8, 2)
>>> res = s2.delay(8)
>>> res.get()
10
```

在这里，设置了设置了参数值为 8，它位于参数值为 2 的签名，形成了完整的 add\(8,2\) 签名。 也可以设置新的参值，新设置的参数会覆盖原有的参数值：

```bash
>>> s3 = add.s(2, 2, debug=True)
>>> s3.delay(debug=False)   # debug is now False.
```

如上所述，签名支持调用API：

* sig.apply\_async\(args=\(\), kwargs={}, \*\*options\)

    使用可选的部分参数和部分关键字参数调用签名以及支持部分执行选项。

* sig.delay\(_args, \*_kwargs\)

    快捷版本的 `apply_async` ，任何参数都将作为签名中的参数，关键字参数将与任何现有键合并。

  这些看起来比较有用，但是可以用来做什么？为了解决这个问题，就需要介绍 canvas 原语.....

### 原语

这些原语本身就是签名对象，可以通过任何进行组合，形成复杂的工作流。

{% hint style="warning" %}
## Note

在下面的这些实例中，如果要运行，需要配置一个结果后端。
{% endhint %}

让我们一起来看一些例子：

#### 

#### 组：Groups

一个 group 并行调用任务列表，返回一个特殊的结果实例，可以将结果作为一个列表进行查看，并且通过索引进去获取返回值。

```bash
>>> from celery import group
>>> from proj.tasks import add

>>> group(add.s(i, i) for i in xrange(10))().get()
[0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
```

* Partial group

```bash
>>> g = group(add.s(i) for i in xrange(10))
>>> g(10).get()
[10, 11, 12, 13, 14, 15, 16, 17, 18, 19]
```



#### 链：Chains

可以将任务链接在一起，在一个人返回后进行调用另外一个任务：

```bash
>>> from celery import chain
>>> from proj.tasks import add, mul

# (4 + 4) * 8
>>> chain(add.s(4, 4) | mul.s(8))().get()
64
```

或 partial chain

```bash
>>> # (? + 4) * 8
>>> g = chain(add.s(4) | mul.s(8))
>>> g(4).get()
64
```

链也可以这样写：

```bash
>>> (add.s(4, 4) | mul.s(8))().get()
64
```

#### 和弦：Chords

和弦是一个带有回调的组：

```bash
>>> from celery import chain
>>> from proj.tasks import add, mul

# (4 + 4) * 8
>>> chain(add.s(4, 4) | mul.s(8))().get()
64
```

或 partial chain

```bash
>>> # (? + 4) * 8
>>> g = chain(add.s(4) | mul.s(8))
>>> g(4).get()
64
```

链接到其他任务的组将自动转换为和弦：

```bash
>>> (group(add.s(i, i) for i in xrange(10)) | xsum.s())().get()
90
```

这些原语都是签名的类型，可以根据需要进行组合，例如：

```bash
>>> upload_document.s(file) | group(apply_filter.s() for filter in filters)
```

有关更多工作流信息，请参阅用户指南中[ Canvas](../yong-hu-zhi-nan/canvas-she-ji-gong-zuo-liu-cheng-canvas-designing-workflows.md) 章节。

## 路由

Celery 支持 AMQP 中提供的所有路由，可以将消息发送到指定的任务队列路由。

通过 `task_routes` 可以设置一个按名称分配的路由任务队列，将所有的内容集中存放在一个位置：

```bash
app.conf.update(
    task_routes = {
        'proj.tasks.add': {'queue': 'hipri'},
    },
)
```

可以在程序是使用 queue 参数进行指定队列：

```bash
>>> from proj.tasks import add
>>> add.apply_async((2, 2), queue='hipri')
```

可以通过设置运行职程（Worker）时指定职程（Worker）从某个队列中进行消费（`celery worker -Q`）：

```bash
$ celery -A proj worker -Q hipri
```

也可以通过“,”作为分割符进行设置多个队列，例如，可以将默认队列和 `hipri` 队列一起通过职程（Worker）进行消费，其中默认队列 `celery` 由于历史原因被命名：

```bash
$ celery -A proj worker -Q hipri,celery
```

队列名称的顺序不分前后，职程（Worker）给予队列分配的权重是相同的。 相关路由的信息以及使用 AMQP 路由的全部功能，详情请参考[`路由任务：Routing Tasks`](../yong-hu-zhi-nan/lu-you-ren-wu-routing-tasks.md)。

## 远程控制

使用 RabbitMQ（AMQP）、Redis 或 Qpid 作为中间人（Broker），可以在运行时控制和检查职程（Worker）。 例如，当前职程（Worker）正在处理的任务：

```bash
$ celery -A proj inspect active
```

这是通过广播消息实现的，集群中所有职程（Worker）都会所有收到远程控制发出的指令。 也可以通过 `--destination` 选项指定一个或多个职程（Worker）进行操作，使用“,”进行分割职程（Worker）主机列表：

```bash
$ celery -A proj inspect active --destination=celery@example.com
```

如果没有提供目的地，那么每个工作人员都将采取行动并回复请求。

celery inspect 命令不能修改程序，只能进行查看职程（Worker）概况以及统计信息，可以通过 `help` 进行查看：

```bash
$ celery -A proj inspect --help
```

celery control 命令可以查看实际上改变了工作在运行时的状况：

```bash
$ celery -A proj control --help
```

例如，可以强制职程（Worker）启用事件消息（用于监控任务以及职程（Worker））：

```bash
$ celery -A proj control enable_events
```

启动事件后，可以启动事件转储程序，进行查看职程（Worker）目前执行的状况：

```bash
$ celery -A proj events --dump
```

或者可以启动 `curses` 接口：

```bash
$ celery -A proj events
```

当监控完毕之后，可以禁用事件：

```bash
$ celery -A proj control disable_events
```

celery status 命令可以远程控制并且显示集群中职程（Worker）的列表：

```bash
$ celery -A proj status
```

可以通过查阅 [`监控和管理手册：Monitoring and Management Guide`](../yong-hu-zhi-nan/jian-kong-he-guan-li-shou-ce-monitoring-and-management-guide.md) ，查看 Celery 有关命令以及监控信息。

## 时区

内部和消息中的所有的时间和日期使用的都是 UTC 时区。 当职程（Worker）收到消息时，例如倒计时设置，会将 UTC 时间转换为本地时间。如果需要使用与系统不同的时区，可以通过 `timezone`进行配置：

```bash
app.conf.timezone = 'Europe/London'
```

## 优化

默认情况下，默认的配置项没有针对吞吐量进行优化，默认的配置比较合适大量短任务和比较少的长任务。 如果需要优化吞吐量，请参考[`优化：Optimizing`](../yong-hu-zhi-nan/you-hua-optimizing.md)。 如果使用的中间人是 RabbitMQ，可以将换成 librabbitmq 模块（通过 C 语言实现的AMQP客户端）：

```bash
$ pip install librabbitmq
```

## 现在做什么？

现在您已经阅读完毕本文档，您已经继续阅读 `用户指南`。

 如果您愿意，还有一个 [`API参考`](../fu-lu/api-api-reference.md)。

