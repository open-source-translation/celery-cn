# 调用任务：Calling Tasks

### 基础入门

这章主要介绍用于Celery任务初始化和"canvas"统一的"调用接口"

### 这些API定义了标准的运行参数集,也就是下面这三个方法

* apply\_async \(args\[,kwargs\[,...\]\]\)

  发送一个任务消息

* delay\(_args,\*_kwargs\)

  直接发送一个任务消息,但是不支持运行参数

* calling\(**call**\)

  应用一个支持调用接口\(例如,add\(2,2\)\)的对象,意味着任务不会被一个worker执行,但是会在当前线程中"执行",\(但是消息不会被发送\)

#### 速查表

* `T.delay(arg, kwarg=value)`

  Star arguments shortcut to `.apply_async.(.delay(*args, **kwargs) calls .apply_async(args, kwargs)).`

* `T.apply_async((arg,), {'kwarg': value})`
* `T.apply_async(countdown=10)`

  executes 10 seconds from now.

* `T.apply_async(eta=now + timedelta(seconds=10))`

  executes 10 seconds from now, specified using eta

* `T.apply_async(countdown=60, expires=120)`

    executes in one minute from now, but expires after 2 minutes.

* `T.apply_async(expires=now + timedelta(days=2))`

    expires in 2 days, set using datetime.

### 例子

`delay()` 方法就像一个很规则的函数,很方便去调用它:

```python
task.delay(arg1, arg2, kwarg1='x', kwarg2='y')
```

用 `apply_async()`替代你写的:

```python
task.apply_async(args=[arg1, arg2], kwargs={'kwarg1': 'x', 'kwarg2': 'y'})
```

尽管运行十分方便,但是如果像设置额外的行参数,你必须用 `apply_async`。

### 小技巧

如果任务在当前线程没有注册,你可以通过名字替代的方法使用send\_task\(\)去调用这个任务.

接下来我们着重的任务参数详情\(task excution options\),所有的例子都基于一个任务add,返回两个参数之和:

```text
@app.task
def add(x, y):
    return x + y
```

#### 还有其他的方式...

你也多了解下一章将会讲到的Canvas,签名的对象用来传递任务的签名\(例如,通过网络发送\),它们还支持API调用:

```python
task.s(arg1, arg2, kwarg1='x', kwargs2='y').apply_async()
```

### Linking\(callbacks/errbacks\)

Celery支持将任务链，一个任务在另一个任务之后。回调任务将用父任务的结果作为一部分参数:

```python
  add.apply_async((2, 2), link=add.s(16))
```

第一个任务的结果\(4\)会被发送下一个新的任务的参数去加上16,可以这样表达 `(2+2)+16=20`

如果task引发异常（errback），您还可以使异常的回调，但这与常规回调的行为不同，因为它将被传递父任务的ID，而不是结果。这是因为抛出序列化引发的异常，因此错误回调需要启用backend，并且任务必须检索任务的结果。

这是一个错误回调的例子:

```python
@app.task
def error_handler(uuid):
    result = AsyncResult(uuid)
    exc = result.get(propagate=False)
    print('Task {0} raised exception: {1!r}\n{2!r}'.format(
          uuid, exc, result.traceback))
```

可以使用 `link_error` 执行选项将其添加到任务中：

```text
add.apply_async((2, 2), link_error=error_handler.s())
```

此外，`link` 和 `link_error` 选项都可以是list：

```python
add.apply_async((2, 2), link=[add.s(16), other_task.s()])
```

然后将依次调用回调/错误返回，并且将使用父任务的返回值作为部分参数来调用所有回调。

### On Message

Celery 可以通过消息回调获取所有状态的改变. 例如对于长时任务发送人任务进程,你可以这样做

```python
@app.task(bind=True)
def hello(self, a, b):
    time.sleep(1)
    self.update_state(state="PROGRESS", meta={'progress': 50})
    time.sleep(1)
    self.update_state(state="PROGRESS", meta={'progress': 90})
    time.sleep(1)
    return 'hello world: %i' % (a+b)
```

```python
def on_raw_message(body):
    print(body)

r = hello.apply_async()
print(r.get(on_message=on_raw_message, propagate=False))
```

将生成如下输出：

```python
{'task_id'：'5660d3a3-92b8-40df-8ccc-33a5d1d680d7'，
 'result'：{'progress'：50}，
 'children'：[]，
 'status'：'PROGRESS'，
 'traceback'：None} 
{'task_id'：'5660d3a3-92b8-40df-8ccc-33a5d1d680d7'，
 'result'：{'progress'：90}，
 'children'：[]，
 'status'：'PROGRESS'，
 'traceback'：None} 
{'task_id'：'5660d3a3-92b8-40df-8ccc-33a5d1d680d7'，
 'result'：'hello world：10'，
 'children'：[]，
 'status'：'SUCCESS'，
 'traceback'：None} 
你好全世界：10
```

### ETA and Countdown

ETA\(\(estimated time of arrival\) 让你设置一个日期和时间,在这个时间之前任务将被执行.countdown 是一种以秒为单位设置ETA的快捷方式.

```python
>>> result = add.apply_async((2, 2), countdown=3)
>>> result.get()    # this takes at least 3 seconds to return
20
```

确保任务在指定的日期和时间之后的某个时间执行，但不一定在该时间执行。可能原因可能包括许多项目在队列中等待，或者严重的网络延迟。为了确保您的任务及时执行，你应该监视队列中的拥塞情况。使用Munin或类似工具来接收警报，因此可以采取适当的措施来减轻负载。点击查看[Munin](https://docs.celeryproject.org/en/4.0/userguide/monitoring.html#monitoring-munin).

尽管 `countdown` 是整数，但eta必须是一个 `datetime` 对象，并指定确切的日期和时间（包括毫秒精度和时区信息）：

```text
>>> from datetime import datetime, timedelta

>>> tomorrow = datetime.utcnow() + timedelta(days=1)
>>> add.apply_async((2, 2), eta=tomorrow)
```

### Expiration

`expries` 参数定义了一个可选的到期时间，既可以作为任务之后秒发布，或在特定日期和时间使用 `datetime`：

```text
>>> # 任务在1分钟后生效
>>> add.apply_async((10, 10), expires=60)

>>> # 也支持 datetime
>>> from datetime import datetime, timedelta
>>> add.apply_async((10, 10), kwargs,
...                 expires=datetime.now() + timedelta(days=1)
```

当 `worker` 收到过期的任务时，它将任务标记为REVOKED（[TaskRevokedError](https://docs.celeryproject.org/en/4.0/reference/celery.exceptions.html#celery.exceptions.TaskRevokedError)）。

### 消息重发 \(Message Sending Retry\)

当连接失败时，Celery会自动重试发送消息，并且可以配置重试行为（例如重试频率或最大重试次数）或全部禁用。

```text
add.apply_async((2, 2), retry=False)
```

相关设定

* task\_publish\_retry
* task\_publish\_retry\_policy

### 重试策略 \(Retry Plicy \)

重试策略是一种控制重试行为的映射，可以包含以下键：

* max\_retries

最大重试次数，在这种情况下，将抛出重试失败的异常。

值为None意味着它将永远重试。

默认值为重试3次。

* interval\_start

定义两次重试之间要等待的秒数（浮点数或整数）。默认值为0（第一次重试是瞬时的）。

* interval\_step

在每次连续重试时，此数字将被添加到重试延迟中（浮点数或整数）。默认值为0.2。

* interval\_max

重试之间等待的最大秒数（浮点数或整数）。默认值为0.2。

例如，默认策略与以下内容相关：

```text
add.apply_async((2, 2), retry=True, retry_policy={
    'max_retries': 3,
    'interval_start': 0,
    'interval_step': 0.2,
    'interval_max': 0.2,
})
```

重试的最长时间为 0.4 秒。默认情况下将其设置为相对较短，因为如果代理连接断开，连接失败可能导致重试堆效应–例如，许多 Web 服务器进程正在等待重试，从而阻止了其他传入请求。

### 连接错误处理\(Connection Error Handling\)

当您发送任务并且传输连接丢失或无法启动连接时，将引发 `OperationalError` 错误：

```text
>>> from proj.tasks import add
>>> add.delay(2, 2)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "celery/app/task.py", line 388, in delay
        return self.apply_async(args, kwargs)
  File "celery/app/task.py", line 503, in apply_async
    **options
  File "celery/app/base.py", line 662, in send_task
    amqp.send_task_message(P, name, message, **options)
  File "celery/backends/rpc.py", line 275, in on_task_call
    maybe_declare(self.binding(producer.channel), retry=True)
  File "/opt/celery/kombu/kombu/messaging.py", line 204, in _get_channel
    channel = self._channel = channel()
  File "/opt/celery/py-amqp/amqp/connection.py", line 272, in connect
    self.transport.connect()
  File "/opt/celery/py-amqp/amqp/transport.py", line 100, in connect
    self._connect(self.host, self.port, self.connect_timeout)
  File "/opt/celery/py-amqp/amqp/transport.py", line 141, in _connect
    self.sock.connect(sa)
  kombu.exceptions.OperationalError: [Errno 61] Connection refused
```

如果启用了[重试](https://docs.celeryproject.org/en/4.0/userguide/calling.html#calling-retry)，则只有在重试用尽后或立即禁用后才发生。

您也可以处理此错误：

```text
>>> from celery.utils.log import get_logger
>>> logger = get_logger(__name__)

>>> try:
...     add.delay(2, 2)
... except add.OperationalError as exc:
...     logger.exception('Sending task raised: %r', exc)
```

### 序列化 \(Serializers\)

在客户端和工作人员之间传输的数据需要进行序列化，因此Celery中的每条消息都有一个content\_type标头，该标头描述了用于对其进行编码的序列化方法。

默认的序列化器是JSON，但是您可以使用[task\_serializer](https://docs.celeryproject.org/en/4.0/userguide/configuration.html#std:setting-task_serializer)设置更改此设置，或者针对每个任务，甚至针对每条消息进行更改。

有内置的支持JSON，[pickle](https://docs.python.org/dev/library/pickle.html#module-pickle)，YAML 和msgpack，你也可以通过他们登记到 Kombu 注册表中添加自己的自定义序列化

#### 安全

pickle模块允许执行任意功能，请参阅安全指南。

Celery还带有一个特殊的序列化程序，该序列化程序使用加密技术对您的消息进行签名。

#### 也可以看看

Kombu中的[消息序列化](http://kombu.readthedocs.io/en/master/userguide/serialization.html#guide-serialization)的用户指南.

每个序列化器都有其优点和缺点。

* json –许多编程语言都支持JSON，现在 这是Python的标准部分（自2.6版本开始），并且使用现代Python库（例如[simplejson](https://pypi.python.org/pypi/simplejson/)）进行解码相当快。 JSON的主要缺点是它将您限制为以下数据类型：strings, Unicode, floats, Boolean, dictionaries, and lists. Decimals and dates 明显缺失。 二进制数据将使用Base64编码进行传输，与支持本地二进制类型的编码格式相比，传输的数据大小增加了34％。 但是，如果您的数据符合上述限制，并且需要跨语言支持，则JSON的默认设置可能是您的最佳选择。 有关更多信息，请参见[http://json.org](http://json.org)。
* pickle–如果不希望支持其他语言除了Python，然后使用pickle编码，将为您提供所有内置Python数据类型（类实例除外）的支持，发送二进制文件时和较小的消息比JSON处理的速度快些。 请参阅[pickle](https://docs.python.org/dev/library/pickle.html#module-pickle)以获取更多信息。
* yaml – YAML与json具有许多相同的特征， 除了它本身数据类型之外,支持更多数据类型（including dates, recursive references, etc.）。但是，YAML的Python库比JSON的库慢很多。 如果您需要一组更具表现力的数据类型，并且需要维护跨语言兼容性，那么YAML可能比上面的更合适。 有关更多信息，请参见[http://yaml.org/](http://yaml.org/)。
* msgpack – msgpack是一种更接近JSON的二进制序列化格式 在功能上。但是，它还很年轻，因此此时应该将支持视为实验性的。 有关更多信息，请参见[http://msgpack.org/](http://msgpack.org/)。

所使用的编码可用作消息头，因此工作人员知道如何反序列化任何任务。如果使用自定义序列化程序，则该序列化程序必须对工作程序可用。

以下顺序用于确定发送任务时使用的序列化程序：

1. 该 serializer 执行选项。
2. 该 Task.serializer 属性
3. 该 task\_serializer 设置。
4. 为单个任务调用设置自定义序列化程序的示例：

```text
>>> add.apply_async((10, 10), serializer='json')
```

### 压缩 \(Compression\)

Celery可以使用gzip或bzip2压缩消息。您还可以创建自己的压缩方案并将其注册到中[kombu compression registry](http://kombu.readthedocs.io/en/master/reference/kombu.compression.html#kombu.compression.register).

以下顺序用于确定发送任务时使用的压缩方案：

1. 该压缩执行选项。
2. 该 Task.compression 属性。
3. 该 task\_compression 属性。

指定调用任务时使用的压缩的示例：

```text
>>> add.apply_async((2, 2), compression='zlib')
```

### 连接\(Connections\)

#### 自动池支持

* 从2.3版开始，支持自动连接池，因此您不必手动处理连接和发布者即可重用连接。
* 从2.5版开始，默认情况下启用连接池。
* 有关broker\_pool\_limit更多信息，请参见设置。

您可以通过创建发布者来手动处理连接：

```text
results = []
with add.app.pool.acquire(block=True) as connection:
    with add.get_publisher(connection) as publisher:
        try:
            for args in numbers:
                res = add.apply_async((2, 2), publisher=publisher)
                results.append(res)
print([res.get() for res in results])
```

尽管这是个特定示例,但是可以更好的展现一组：

```text
>>> from celery import group

>>> numbers = [(2, 2), (4, 4), (8, 8), (16, 16)]
>>> res = group(add.s(i, j) for i, j in numbers).apply_async()

>>> res.get()
[4, 8, 16, 32]
```

### 路由选择 \(Routing options\)

Celery可以将任务路由到不同的队列。

使用以下queue可以完成简单的路由\(name &lt;-&gt; name\)：

```text
add.apply_async(queue='priority.high')
```

然后，您可以指派 workers 给 priority.high 的队列,使用 worker -Q 参数将分配给队列：

```text
$ celery -A proj worker -l info -Q celery,priority.high
```

### 也可以看看

不建议用代码对队列名称进行硬编码，最佳做法是使用配置路由器（[task\_routes](https://docs.celeryproject.org/en/4.0/userguide/configuration.html#std:setting-task_routes)）。

要了解有关路由的更多信息，请参阅“[路由任务\(Routing Tasks\)](https://docs.celeryproject.org/en/4.0/userguide/routing.html#guide-routing)”。

### 高级选项

这些选项适用于想要使用AMQP完整路由功能的高级用户。有兴趣的人士可以阅读[路由指南](https://docs.celeryproject.org/en/4.0/userguide/routing.html#guide-routing)。

* 交换\(exchange\)

发送信息的 exchange\(或者 kombu.entity.Exchange\) 的名称

* routing\_key

用于确定路由的密钥。

* 优先\(priority\)

0到255之间的数字，其中255是最高优先级。

支持：RabbitMQvvvvvv，Redis（优先级颠倒，最高为0）。

