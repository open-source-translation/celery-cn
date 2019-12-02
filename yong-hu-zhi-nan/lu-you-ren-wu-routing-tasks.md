# 路由任务：Routing Tasks

{% hint style="info" %}
并不是所有的传输中间件\(Broker\)都存在`topic`和`fanout` 等代理路由概念，请参阅 [transport comparison table](https://kombu.readthedocs.io/en/master/introduction.html#transport-comparison).
{% endhint %}

## 基本用法

### **自动路由**

最简单的路由方式是使用选项 `task_create_missing_queues` 进行设置\(默认情况下，此设置为打开状态\)。

如果启用了该参数，将会自动创建没有在 `task_queues` 选项中定义的命名队列。这样可以更加容易的执行简单的路由任务。

假如你有两个处理常规任务的服务器 `x` 和 `y` ，以及一个只处理与 `feed` 相关的任务的服务器 `z` 。那么你可以使用这样的配置：

```python
task_routes = {'feed.tasks.import_feed': {'queue': 'feeds'}}
```

启用这样的路由设置后，`import_feed` 的任务将会被路由到 `feeds` 队列中，而其他的任务将会被路由到默认的队列\(因为历史原因被命名为`celery`\)。

另外，你还可以使用通配符，甚至正则表达式来匹配所有在 `feed.tasks` 命名空间内的所有任务:

```python
app.conf.task_routes = {'feed.tasks.*': {'queue': 'feeds'}}
```

如果匹配模式的顺序很重要，你应该使用列表的方式指定路由的次序：

```python
task_routes = ([
    ('feed.tasks.*', {'queue': 'feeds'}),
    ('web.tasks.*', {'queue': 'web'}),
    (re.compile(r'(video|image)\.tasks\..*'), {'queue': 'media'}),
],)
```

{% hint style="info" %}
`task_routes` 配置可以使用字典或者一个路由对象的列表，在这种情况下，我们需要使用固定元组的一个列表来配置路由。
{% endhint %}

安装完路由之后，你可以按照如下方式启动服务器 `z` 来只处理 `feeds` 队列的消息:

```bash
user@z:/$  celery -A proj worker -Q feeds
```

你可以指定任意数量的队列，所以你也可以让这个服务器去处理来自默认队列的消息：

```bash
user@z:/$  celery -A proj worker -Q feeds,celery
```

#### **修改默认队列的名称**

你可以使用如下的配置来修改默认队列的名称:

```python
app.conf.task_default_queue = 'default'
```

#### **队列是如何定义的**

这部分的特性主要是隐藏复杂的 `AMPQ` 协议实现，只对用户暴露出需要的基础用法。但是，你可能仍然对队列是如何被声明的原理感兴趣。

使用如下的配置将会创建一个名为 `video` 的队列:

```python
{
    'exchange': 'video',
  'exchange_type': 'direct',
  'routing_key': 'video'
}
```

对于那些非 `AMPQ` 的后端组件如 `Redis` 或者 `SQS` 并不支持交换机，所以他们要求交换机的名称与队列的名称一致。 使用这种设计可以确保正常的处理不同的情况。

### **手动路由**

假设你有两台处理常规任务的服务器，`x` 和 `y`，以及另一台只处理与 `feed` 相关的任务，你可以使用如下的配置：

```python
from kombu import Queue

app.conf.task_default_queue = 'default'
app.conf.task_queues = (
    Queue('default',    routing_key='task.#'),
    Queue('feed_tasks', routing_key='feed.#'),
)
task_default_exchange = 'tasks'
task_default_exchange_type = 'topic'
task_default_routing_key = 'task.default'
```

`task_queues` 是一个包含 `Queue` 实例的列表。如果你不想指定 `exchange` 和 `exchange_type` 的值。这些变量将会被 `task_default_exchange` 和 `task_default_exchange_type` 来设置。

要将一个任务路由到 `feed_tasks` 队列中，你可以在`task_routes`配置中添加一个入口:

```python
task_routes = {
        'feeds.tasks.import_feed': {
            'queue': 'feed_tasks',
            'routing_key': 'feed.import',
        },
}
```

你还可以使用 `Task.apply_async()` 或者 `send_task()` 中的 `routing_key` 参数来重载这些设置:

```python
>>> from feeds.tasks import import_feed
>>> import_feed.apply_async(args=['http://cnn.com/rss'],
...                         queue='feed_tasks',
...                         routing_key='feed.import')
```

要使服务器 `z` 只处理来自 `feed` 队列的消息，你可以使用 [`celery worker -Q`](http://docs.celeryproject.org/en/latest/reference/celery.bin.worker.html#cmdoption-celery-worker-q) 来启动服务：

```bash
user@z:/$ celery -A proj worker -Q feed_tasks --hostname=z@%h
```

服务器 `x` 和 `y` 需要配置为从默认的队列中消费消息:

```bash
user@x:/$ celery -A proj worker -Q default --hostname=x@%h
user@y:/$ celery -A proj worker -Q default --hostname=y@%h
```

如果你想，你也可以让 `feed` 消息的处理职程去处理常规消息，比如在某个时间出现很多任务需要去做:

```bash
user@z:/$ celery -A proj worker -Q feed_tasks,default --hostname=z@%h
```

如果你想添加配置了另一个交换机的队列，只需要指定自定义的 `exchange` 和 `exchange_type` :

```python
from kombu import Exchange, Queue

app.conf.task_queues = (
    Queue('feed_tasks',    routing_key='feed.#'),
    Queue('regular_tasks', routing_key='task.#'),
    Queue('image_tasks',   exchange=Exchange('mediatasks', type='direct'),
                           routing_key='image.compress'),
)
```

如果你对这些术语感到迷惑，你应该阅读一下 `AMPQ`.

> See also:
>
> 除了如下的 [`Redis Message Priorities`，还有](http://docs.celeryproject.org/en/latest/userguide/routing.html#amqp-primer) [`Rabbits and Warrens` \(译者注: 此文地址已不可用, 找到此文的](http://blogs.digitar.com/jjww/2009/01/rabbits-and-warrens/)[备份地址](https://web.archive.org/web/20160303092138/http://blogs.digitar.com/jjww/2009/01/rabbits-and-warrens/)\)，一篇描述队列和交换机的优秀博客。除此之外还有 `CloudAMQP` 的教程，对于 `RabbitMQ` 的用户来说， \[RabbitMQ FAQ\]\(https://www.rabbitmq.com/faq.html\) 也可以作为一个有用的信息源。

## 特殊的路由选项

### **RabbitMQ 消息优先级**

**支持的中间人\(Broker\)：**

RabbitMQ

从 `4.0` 版本开始引入。

队列可以通过设置 `x-max-priority` 参数来支持优先级：

```python
from kombu import Exchange, Queue

app.conf.task_queues = [
    Queue('tasks', Exchange('tasks'), routing_key='tasks',
          queue_arguments={'x-max-priority': 10}),
]
```

可以通过指定参数 [`task_default_priority`](http://docs.celeryproject.org/en/latest/userguide/configuration.html#std:setting-task_queue_max_priority) 来设置所有队列的默认最大优先级

```python
app.conf.task_queue_max_priority = 10
```

可以通过指定参数 [`task_default_priority`](http://docs.celeryproject.org/en/latest/userguide/configuration.html#std:setting-task_default_priority) 来设置所有任务的默认优先级

```python
app.conf.task_default_priority = 5
```

### **Redis 消息优先级**

**支持的中间人\(Broker\)：**

Redis

虽然 `Celery` 的 `Redis` 中间人\(Broker\) 支持了优先级的字段，但是 `Redis` 本身并没有优先级的概念。所以在尝试使用 `Redis` 来实现优先级之前，请阅读下方的说明，因为你可能遇到一些意想不到的行为。

优先级的支持是通过为每个队列创建 `n` 个列表来实现的。也就是说即使存在 10\(0-9\) 个优先级别，在默认情况下也会被合并成 4 个级别来节省资源。也就是说一个名为 `celery` 的队列将会分成 4 个队列:

```python
['celery0', 'celery3', 'celery6', 'celery9']
```

如果你想要更多的优先级别，你可以通过设置中间人\(Broker\)参数 `priority_steps` 来实现：

```python
app.conf.broker_transport_options = {
    'priority_steps': list(range(10)),
}
```

这就是说，要注意到这样的实现永远不如在服务器端实现优先级别，只能近似说是最佳的实现。但是这对于你的应用来说也足够好了。

## AMQP 入门

### **消息**

一个消息由消息头和消息体组成。`Celery` 使用消息头来存储消息的内容类型以及内容的编码。内容类型通常是用来序列化消息的序列化格式，消息体包含要执行的任务的名称，任务ID\(UUID\)，执行任务的参数以及一些额外的元数据\(比如重试次数和ETA\(下次执行任务的时间\)\)。

这是通过一个 `Python` 的字典来表示的示例任务消息：

```python
{'task': 'myapp.tasks.add',
 'id': '54086c5e-6193-4575-8308-dbab76798756',
 'args': [4, 4],
 'kwargs': {}}
```

### **生产者，消费者和中间人**

发送消息的客户端通常被称为一个发布者，或一个生产者，而接收消息的实体被称为消费者。

中间人是将消息从生产者路由到消费者的消息服务器。

你可能在与AMQP相关的材料中看到这些术语被大量使用。

### 交换机，队列和路由键

1. 消息将被发送到交换机
2. 交换机将消息路由到一个或者多个队列。存在多种交换机类型来提供不同的消息路由方式，或实现不同的消息发送方案。
3. 消息将在队列中等待直到有人消费它。
4. 一旦消息被确认消费，将会从队列中删除。

发送和接收消息所需要的步骤如下：

1. 创建一个交换机
2. 创建一个队列
3. 绑定队列到交换机。

为了使 [`task_queues`](http://docs.celeryproject.org/en/latest/userguide/configuration.html#std:setting-task_queues) 中的队列工作， `Celery` 将会自动创建所需要的实体\(除非队列的 _auto\_declare_ 选项被设置为 `False` \)。

下方是一个包含三个队列的示例队列配置; 一个用于视频，一个用于图片，另一个用于其他消息的默认队列：

```python
from kombu import Exchange, Queue

app.conf.task_queues = (
    Queue('default', Exchange('default'), routing_key='default'),
    Queue('videos',  Exchange('media'),   routing_key='media.video'),
    Queue('images',  Exchange('media'),   routing_key='media.image'),
)
app.conf.task_default_queue = 'default'
app.conf.task_default_exchange_type = 'direct'
app.conf.task_default_routing_key = 'default'
```

### **交换机类型**

交换机的类型定义了交换机将会如何路由消息。在正常情况下交换机被定义为 `direct`, `topic`, `fanout`。此外也可以通过 `RabbitMQ` 的插件来使用非标准的交换机类型，比如由 `Michael Bridgen` 实现的 [last-value-cache plug-in](https://github.com/squaremo/rabbitmq-lvc-plugin) 。

#### 直连交换机

直连交换机通过精确的路由键来进行匹配，所以被路由键 `video` 绑定的队列只接收具有该路由键的消息。

#### 主题交换机

主题交换机通过以点分隔的单词和通配符来匹配路由键：`*`\(匹配单个单词\) `#`\(匹配零或多个单词\)。

假如有如下路由键 `usa.news`，`usa.weather`，`norway.news` 和 `norway.weather`，可以通过绑定 `*.news` 来接收所有的新闻，绑定 `usa.#` 来接收与 `USA` 有关的所有消息，或绑定 `usa.weather` 来接收所有与 `USA` 天气有关的消息。

### 相关的API命令

**exchange.declare\(exchange\_name, type, passive, durable, auto\_delete, internal\)**

通过名称声明交换机

参阅 [`amqp:Channel.exchange_declare`](https://amqp.readthedocs.io/en/latest/reference/amqp.channel.html#amqp.channel.Channel.exchange_declare)

**关键字参数**

* **passive** 被动意味着不会创建交换机，但是你可以通过这个参数来检查交换机是否被创建。
* **durable** 交换机将被持久化\(也就是说，中间人\(Broker\)重启后，交换机仍然存在\)
* **auto\_delete** 指定该参数意味着如果没有队列使用该交换机，那么交换机将被中间人\(Broker\)删除。

**queue.declare\(queue\_name, passive, durable, exclusive, auto\_delete\)**

通过名称声明一个队列

参阅 [`amqp:Channel.queue_declare`](https://amqp.readthedocs.io/en/latest/reference/amqp.channel.html#amqp.channel.Channel.queue_declare)

**关键字参数**

* **exclusive** 专有队列只能通过当前的连接进行消费，专有队列也同时是自动删除的

**queue.bind\(queue\_name, exchange\_name, routing\_key\)**

通过路由键\(routing\_key\)将队列绑定到交换机

队列如果没有被绑定将不会接收消息，因为绑定是必须的

参阅 [`amqp:Channel.queue_bind`](https://amqp.readthedocs.io/en/latest/reference/amqp.channel.html#amqp.channel.Channel.queue_bind)

**queue.delete\(name. If\_unused=False, if\_empty=False\)**

删除队列及其绑定

参阅 [`amqp:Channel.queue_delete`](https://amqp.readthedocs.io/en/latest/reference/amqp.channel.html#amqp.channel.Channel.queue_delete)

**exchange.delete\(name. If\_unused=False\)**

删除交换机

参阅 [`amqp:Channel.exchange_delete`](https://amqp.readthedocs.io/en/latest/reference/amqp.channel.html#amqp.channel.Channel.exchange_delete)

{% hint style="info" %}
声明并不代表创建，在你声明的时候，你可以断定这个实体已经存在，并且是可操作的。这里并没有规定消费者或生产者中的哪一方需要最先创建交换机/队列/绑定。通常来说，最先需要它的哪一方就会创建它。
{% endhint %}

### 使用 API

`Celery` 自带了一个名为 `celery amqp` 的工具，用于通过命令行来操作 `AMQP API` 去管理任务，比如说创建或者删除队列或交换机，清理队列或发送消息。该工具也可以用于 非 `AMQP` 的中间人，但是不一定实现了所有的命令操作。

你可以直接在```celery amqp`` 的命令里写参数，或者无参数启动命令模式：

```bash
$ celery -A proj amqp
-> connecting to amqp://guest@localhost:5672/.
-> connected.
1>
```

这里的 `1>` 是命令提示。数字 `1` 表示到目前为止指定的命令数。 输入 `help` 可以得到所有可用的命令列表。工具还支持自动补全，所以你可以输入一个命令然后按 `tab` 键来显示可能匹配到的命令列表。

让我们创建一个你可以发送消息的队列:

```bash
1> exchange.declare testexchange direct
ok.
2> queue.declare testqueue
ok. queue:testqueue messages:0 consumers:0.
3> queue.bind testqueue testexchange testkey
ok.
```

上方的命令创建了一个直连交换机 `testexchange` 和一个名为 `testqueue` 的队列。该队列通过路由键 `testkey` 绑定到直连交换机。

从现在开始，所有发送到 `testexchange` 交换机的带有路由键`testkey` 的消息将被移动到队列 `testqueue` 中。你可以通过 `basic.publish` 命令发送一条消息:

```bash
4> basic.publish 'This is a message!' testexchange testkey
ok.
```

现在消息已经发送出去，你可以去获取消息了。你可以在这里使用 `basic.get` 命令，该命令将会以同步轮询的方式去获取队列中的新消息\(这种方式对于维护任务来说是还可以的，但是对于服务来说，你需要使用 `basic.consume`命令来代替它\)

从队列中弹出一条消息:

```bash
{'body': 'This is a message!',
 'delivery_info': {'delivery_tag': 1,
                   'exchange': u'testexchange',
                   'message_count': 0,
                   'redelivered': False,
                   'routing_key': u'testkey'},
 'properties': {}}
```

`AMQP` 使用确认来表明一条消息已经被接收并且成功处理。如果消息没有被确认并且消费者的通道关闭了，消息将被传递给另一个消费者。

请注意上方结构中列出来的传递标记 `delivery_tag` ; 再每个连接通道中，每个接收到的消息都有一个唯一的传递标记，这个标记用来确认消息。但是要注意，传递标记并不是跨连接唯一的，所以在另一个客户端中，传递标记为 1 的消息可能与当前连接中的消息是不一致的。

你可以通过 `basic.ack` 命令来确认你收到的消息：

```bash
6> basic.ack 1
ok.
```

在我们的测试回话结束后，你应该清除你创建的实体：

```bash
7> queue.delete testqueue
ok. 0 messages deleted.
8> exchange.delete testexchange
ok.
```

## 路由任务

### **队列声明**

在 `Celery` 存在的队列可以通过 [`task_queues`](http://docs.celeryproject.org/en/latest/userguide/configuration.html#std:setting-task_queues) 来设置。

下方是一个包含三个队列的示例队列配置; 一个用于视频，一个用于图片，另一个用于其他消息的默认队列：

```python
default_exchange = Exchange('default', type='direct')
media_exchange = Exchange('media', type='direct')

app.conf.task_queues = (
    Queue('default', default_exchange, routing_key='default'),
    Queue('videos', media_exchange, routing_key='media.video'),
    Queue('images', media_exchange, routing_key='media.image')
)
app.conf.task_default_queue = 'default'
app.conf.task_default_exchange = 'default'
app.conf.task_default_routing_key = 'default'
```

在这里 `task_default_queue` 指定队列将被用于路由那些没有显示指定队列的任务。

`task_default_exchange`，`exchange type` 以及`routing key` 将被用作于任务的默认值，并且也被用作于 `task_queues` 中的实体的默认值。

对单个队列的多个绑定也是被支持的。如下一个两个路由键同时绑定于同一个队列的例子:

```python
from kombu import Exchange, Queue, binding

media_exchange = Exchange('media', type='direct')

CELERY_QUEUES = (
    Queue('media', [
        binding(media_exchange, routing_key='media.video'),
        binding(media_exchange, routing_key='media.image'),
    ]),
)
```

### **指定任务目标**

任务的目标是通过如下的\(按顺序\)的方式决定的:

1. `Task.apply_async` 的路由参数
2. 在任务本身定义的路由参数
3. 在 [`task_routes`](http://docs.celeryproject.org/en/latest/userguide/configuration.html#std:setting-task_routes) 中定义的 [Routers](http://docs.celeryproject.org/en/latest/userguide/routing.html#routers)

最好的做法是不在配置中进行硬编码，而是将其作为 [Routers](http://docs.celeryproject.org/en/latest/userguide/routing.html#routers) 的配置。这是最灵活的，并且合理的默认值仍然可以设置为任务的属性。

### **路由器**

路由器是决定任务的路由选项的函数。

定义一个新的路由器，你所需要做的是通过签名 `(name, args, kwargs, options, task=None, **kw)` 定义一个函数:

```python
def route_task(name, args, kwargs, options, task=None, **kw):
        if name == 'myapp.tasks.compress_video':
            return {'exchange': 'video',
                    'exchange_type': 'topic',
                    'routing_key': 'video.compress'}
```

如果你返回队列的键值，它将会带着在[`task_queues`](http://docs.celeryproject.org/en/latest/userguide/configuration.html#std:setting-task_queues) 中定义的配置展开：

```python
{'queue': 'video', 'routing_key': 'video.compress'}
```

变成 -&gt;

```python
{'queue': 'video',
 'exchange': 'video',
 'exchange_type': 'topic',
 'routing_key': 'video.compress'}
```

你可以通过将路由器的类添加到 [`task_routes`](http://docs.celeryproject.org/en/latest/userguide/configuration.html#std:setting-task_routes) 的配置中来安装路由器:

```python
task_routes = (route_task,)
```

路由器方法也可以通过名称添加

```python
task_routes = ('myapp.routers.route_task',)
```

对于类似上方示例的简单的任务名称-&gt;路由映射，你可以简单地将字典放置在 [`task_routes`](http://docs.celeryproject.org/en/latest/userguide/configuration.html#std:setting-task_routes) 中来过的同样的行为效果:

```python
task_routes = {
    'myapp.tasks.compress_video': {
        'queue': 'video',
        'routing_key': 'video.compress',
    },
}
```

将会按照顺序遍历路由器，直到在第一个返回真的路由器处停止，并将该路由器用作为任务的最终路由器。

你也可以将多个路由器定义在一个序列中：

```text
task_routes = [
    route_task,
    {
        'myapp.tasks.compress_video': {
            'queue': 'video',
            'routing_key': 'video.compress',
    },
]
```

路由器将会被按顺序访问，并选择第一个返回的值。

如果你使用的是 `Redis` 或 `RabbitMQ` ，你也可以在路由器中指定队列的默认优先级。

```python
task_routes = {
    'myapp.tasks.compress_video': {
        'queue': 'video',
        'routing_key': 'video.compress',
        'priority': 10,
    },
}
```

类似的，对任务使用 `apply_async` 调用时，传递的参数将会覆盖默认的优先级。

```python
task.apply_async(priority=0)
```

#### **优先级顺序和集群响应**

需要重视的是，因为职程\(worker\) 的预取机制，如果同一时间提交了一堆任务，那么它们的优先级顺序可能发生混乱。禁用职程的预取可以防止该问题，但是对于小而快的任务，这么做会导致达不到理想的性能。在大多数情况下，简单的将 _worker\_prefetch\_multiplier_ 参数减少到 1，是一个简单而清晰的方式来提升系统的灵敏性，并且不会存在禁用预取带来的成本。

要注意的是优先级的顺序是按照值的反序来排列的：0 是最高优先级。

### **广播**

`Celery` 也支持广播路由。下面是一个 _broadcast\_tasks_ 交换机的示例, 它将任务分发给所有连接到它的职程：

```python
from kombu.common import Broadcast

app.conf.task_queues = (Broadcast('broadcast_tasks'),)
app.conf.task_routes = {
    'tasks.reload_cache': {
        'queue': 'broadcast_tasks',
        'exchange': 'broadcast_tasks'
    }
}
```

现在任务 `tasks.reload_cache` 将会被被发送给从当前队列中消费的所有职程。

如下是另一个关于广播路由的任务，这次使用了 `celery beat` 定时器:

```python
from kombu.common import Broadcast
from celery.schedules import crontab

app.conf.task_queues = (Broadcast('broadcast_tasks'),)

app.conf.beat_schedule = {
    'test-task': {
        'task': 'tasks.reload_cache',
        'schedule': crontab(minute=0, hour='*/3'),
        'options': {'exchange': 'broadcast_tasks'}
    },
}
```

#### **广播和结果**

注意 `Celery` 结果并没有定义如果有两个任务使用同一个任务 ID 时会发生什么。如果同一个人任务被派发到多于一个职程，该任务的状态历史将不被保留。

在这种情况下设置 `task.ignore_result` 属性忽略任务结果将会是个好主意。



