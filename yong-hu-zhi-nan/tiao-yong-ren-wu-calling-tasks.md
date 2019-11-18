# 调用任务：Calling Tasks

## 基础入门

本文档介绍了任务实例和 [canvas](tiao-yong-ren-wu-calling-tasks.md) 对 Celery 统一的调用接口。

这些 API 定义了标准的执行选项集，也就是下面这三个方法：

* apply\_async\(args\[, kwargs\[, ...\]\]\)

  发送一个任务消息。

* delay\(\*args, \*\*kwargs\)

  直接发送一个任务消息,但是不支持运行参数。

* calling\(**\_\_call\_\_**\)

  应用一个支持调用接口（例如，add\(2,2\)）的对象,意味着任务不会被一个 worker 执行,但是会在当前线程中执行\(但是消息不会被发送\)。

### 速查表

* `T.delay(arg, kwarg=value)`

  调用 apply\_async 的快捷方式（.delay\(_args, \*_kwargs\)等价于调用 .apply\_async\(args, kwargs\)）。

* `T.apply_async((arg,), {'kwarg': value})`
* `T.apply_async(countdown=10)`

  从现在起, 十秒内执行。

* `T.apply_async(eta=now + timedelta(seconds=10))`

  从现在起十秒内执行，指明使用eta。

* `T.apply_async(countdown=60, expires=120)`

  从现在起一分钟执行，但在两分钟后过期。

* `T.apply_async(expires=now + timedelta(days=2))`

  两天内过期，使用datetime对象。

## 例子

`delay()` 方法就像一个很规则的函数，很方便去调用它：

```python
task.delay(arg1, arg2, kwarg1='x', kwarg2='y')
```

用 `apply_async()` 替代你写的：

```python
task.apply_async(args=[arg1, arg2], kwargs={'kwarg1': 'x', 'kwarg2': 'y'})
```

尽管运行十分方便，但是如果像设置额外的行参数，你必须用 `apply_async`。

## 小技巧

如果任务在当前线程没有注册,你可以通过名字替代的方法使用 send\_task\(\) 去调用这个任务。

接下来我们着重的任务参数详情（task excution options），所有的例子都基于一个任务add,返回两个参数之和：

```python
@app.task
def add(x, y):
    return x + y
```

### 还有其他的方式......

你也多了解下一章将会讲到的Canvas,签名的对象用来传递任务的签名（例如,通过网络发送），它们还支持API调用：

```python
task.s(arg1, arg2, kwarg1='x', kwargs2='y').apply_async()
```

## Linking\(callbacks/errbacks\)

Celery支持将任务链，一个任务在另一个任务之后。回调任务将用父任务的结果作为一部分参数：

```python
res = add.apply_async((2, 2), link=add.s(16))

# 译者注
# res.get() --> 4
# res.children[0].get() --> 20
```

第一个任务的结果（4）会被发送下一个新的任务的参数去加上16，可以这样表达 `(2+2)+16=20`。

如果task引发异常（errback），您还可以使异常的回调，但这与常规回调的行为不同，因为它将被传递父任务的ID，而不是结果。这是因为抛出序列化引发的异常，因此错误回调需要启用backend，并且任务必须检索任务的结果。

这是一个错误回调的例子：

```python
@app.task
def error_handler(uuid):
    result = AsyncResult(uuid)
    exc = result.get(propagate=False)
    print('Task {0} raised exception: {1!r}\n{2!r}'.format(
          uuid, exc, result.traceback))
```

可以使用 `link_error` 执行选项将其添加到任务中：

```python
add.apply_async((2, 2), link_error=error_handler.s())
```

此外，`link` 和 `link_error` 选项都可以是list：

```python
add.apply_async((2, 2), link=[add.s(16), other_task.s()])
```

然后将依次调用回调/错误返回，并且将使用父任务的返回值作为部分参数来调用所有回调。

## On Message

Celery 可以通过消息回调获取所有状态的改变。例如对于长时任务发送人任务进程，你可以这样做：

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

r = hello.apply_async(4, 6)
print(r.get(on_message=on_raw_message, propagate=False))
```

将生成如下输出：

```python
{'task_id': '5660d3a3-92b8-40df-8ccc-33a5d1d680d7',
 'result': {'progress': 50},
 'children': [],
 'status': 'PROGRESS',
 'traceback': None}
{'task_id': '5660d3a3-92b8-40df-8ccc-33a5d1d680d7',
 'result': {'progress': 90},
 'children': [],
 'status': 'PROGRESS',
 'traceback': None}
{'task_id': '5660d3a3-92b8-40df-8ccc-33a5d1d680d7',
 'result': 'hello world: 10',
 'children': [],
 'status': 'SUCCESS',
 'traceback': None}
hello world: 10
```

## ETA and Countdown

ETA（estimated time of arrival, 预计到底时间）让你设置一个日期和时间，在这个时间之前任务将被执行。countdown 是一种以秒为单位设置ETA的快捷方式。

```python
>>> result = add.apply_async((2, 2), countdown=3)
>>> result.get()    # this takes at least 3 seconds to return
20
```

确保任务在指定的日期和时间之后的某个时间执行，但不一定在该时间执行。可能原因可能包括许多项目在队列中等待，或者严重的网络延迟。为了确保您的任务及时执行，你应该监视队列中的拥塞情况。使用Munin或类似工具来接收警报，因此可以采取适当的措施来减轻负载。点击查看[Munin](https://docs.celeryproject.org/en/4.0/userguide/monitoring.html#monitoring-munin)。

尽管 `countdown` 是整数，但eta必须是一个 `datetime` 对象，并指定确切的日期和时间（包括毫秒精度和时区信息）：

```python
>>> from datetime import datetime, timedelta

>>> tomorrow = datetime.utcnow() + timedelta(days=1)
>>> add.apply_async((2, 2), eta=tomorrow)
```

## Expiration

`expries` 参数定义了一个可选的到期时间，既可以作为任务之后秒发布，或在特定日期和时间使用 `datetime`：

```python
>>> # Task expires after one minute from now.
>>> add.apply_async((10, 10), expires=60)

>>> # Also supports datetime
>>> from datetime import datetime, timedelta
>>> add.apply_async((10, 10), kwargs,
...                 expires=datetime.now() + timedelta(days=1)
```

当 `worker` 收到过期的任务时，它将任务标记为REVOKED（[TaskRevokedError](https://docs.celeryproject.org/en/4.0/reference/celery.exceptions.html#celery.exceptions.TaskRevokedError)）。

## 消息重发 \(Message Sending Retry\)

当连接失败时，Celery 会自动重试发送消息，并且可以配置重试行为（例如重试频率或最大重试次数）或全部禁用。

```python
add.apply_async((2, 2), retry=False)
```

相关设定

* [task\_publish\_retry](tiao-yong-ren-wu-calling-tasks.md)
* [task\_publish\_retry\_policy](tiao-yong-ren-wu-calling-tasks.md)

## 重试策略 \(Retry Plicy \)

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

重试的最长时间为0.4秒。默认情况下将其设置为相对较短，因为如果代理连接断开，连接失败可能导致重试堆效应–例如，许多 Web 服务器进程正在等待重试，从而阻止了其他传入请求。

## 连接错误处理\(Connection Error Handling\)

当您发送任务并且传输连接丢失或无法启动连接时，将引发 `OperationalError` 错误：

```python
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

```python
>>> from celery.utils.log import get_logger
>>> logger = get_logger(__name__)

>>> try:
...     add.delay(2, 2)
... except add.OperationalError as exc:
...     logger.exception('Sending task raised: %r', exc)
```

## 序列化 \(Serializers\)

在客户端和工作人员之间传输的数据需要进行序列化，因此 Celery 中的每条消息都有一个 content\_type 标头，该标头描述了用于对其进行编码的序列化方法。

默认的序列化器是JSON，但是您可以使用 [task\_serializer](https://docs.celeryproject.org/en/4.0/userguide/configuration.html#std:setting-task_serializer) 设置更改此设置，或者针对每个任务，甚至针对每条消息进行更改。

有内置的支持JSON，[pickle](https://docs.python.org/dev/library/pickle.html#module-pickle)，YAML 和msgpack，你也可以通过他们登记到 Kombu 注册表中添加自己的自定义序列化

### 安全

pickle 模块允许执行任意功能，请参阅安全指南。

Celery 还带有一个特殊的序列化程序，该序列化程序使用加密技术对您的消息进行签名。

### 也可以看看

Kombu 中的[消息序列化](http://kombu.readthedocs.io/en/master/userguide/serialization.html#guide-serialization)的用户指南。

每个序列化器都有其优点和缺点：

* json - JSON 被大多数的编程语言支持，并且现在是 Python 标准的一部分（自2.6开始），并且现代 Python 库（例如 [simplejson](https://pypi.python.org/pypi/simplejson/)）具有非常快的 json 解析速度。

  JSON 的缺点是会限制你使用如下的数据类型：字符串、Unicode、字典和列表。小数和日期明显缺失。

  二进制数据使用 Base64 编码进行传输，这与支持纯二进制传输的相比，数据传输量增长了34%。

  但如果你的数据满足上述限制，并且你需要跨语言支持，则 JSON 可能是你的最佳选择。

  有关更多信息，请参见 [http://json.org](http://json.org)。

* pickle - 如果你除了 Python 外，并不需要支持其他语言，那么使用 pickle 编码将让你获得对所有 Python 内建类型的支持（类实例除外）。相比 JSON，使用 pickle 序列化的二进制文件更小，传输速度更快。

  请参阅 [pickle](https://docs.python.org/dev/library/pickle.html#module-pickle) 获得更多信息。

* yaml - YAML 和 JSON 有许多相似的特征，yaml 支持包括日期、递归引用在内的更多数据类型。然而，Python 的 YMAL 库相比 JSON 库 要慢很多。

  如果你需要更具表现能力的数据集合，则 YMAL 比上面的序列化方式更适合。

  有关更多信息，请参见 [http://yaml.org/](http://yaml.org/)。

* msgpack - msgpack 是一种接近 JSON 的二进制序列化格式。但是，它还很年轻，因此此时应该将支持视为实验性的

  有关更多信息，请参见 [http://msgpack.org/](http://msgpack.org/)。

编码类型可以用作消息头，因此 workers 知道如何反序列化所有的任务。如果你使用自定义序列方案，则该序列化必须被 workers 支持。

发送任务时的序列化配置优先级如下（从高到低）：

* 1.`serializer` 执行选项。
* 2.[Task.serializer](tiao-yong-ren-wu-calling-tasks.md) 属性。
* 3.[task\_serializer](tiao-yong-ren-wu-calling-tasks.md) 属性。

为单个任务调用设置序列化方式：

```python
>>> add.apply_async((10, 10), serializer='json')
```

## 压缩 \(Compression\)

Celery 可以使用以下内建方案压缩消息。

* brotli

  brotli 针对 web 进行了优化，尤其是小型文档。该压缩对诸如字体、html页面等静态内容最有效。

  要使用 brotli，请用以下命令进行安装。

  ```text
  $ pip install celery[brotli]
  ```

* bzip2

  bzip2 创建的文件比 gzip 小，但是压缩和解压的速度明显慢于 gzip。

  要使用 bzip2，请确保 bzip2 已经编译到你的 Python 可执行文件中。

  如果你得到以下错误 [ImportError](https://docs.python.org/dev/library/exceptions.html#ImportError)

  ```text
  >>> import bz2
  Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
  ImportError: No module named 'bz2'
  ```

  这意味着你应该重新编译支持 bzip2 的 Python 版本。

* gzip

  gzip 适用于内存占用较小的系统，因此 gzip 非常适合内存有限的系统。该压缩常用语生成带有 “.tar.gz” 后缀的文件。

  ```text
  要使用 gzip，请确保 gzip 已经编译到你的 Python 可执行文件中。

  如果你得到以下错误[ImportError](https://docs.python.org/dev/library/exceptions.html#ImportError)
  >>> import gzip
  Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
  ImportError: No module named 'gzip'
  ```

  这意味着你应该重新编译支持 gzip 的 Python 版本。

* lzma

  lzma 具有较好的压缩效率以及压缩解压速度，但内存消耗更大。

  要使用 lzma，请确保 gzip 已经编译到你的 Python 可执行文件中，并且你的 Python 版本为3.3或更高版本。

  如果你得到以下错误 [ImportError](https://docs.python.org/dev/library/exceptions.html#ImportError)

  ```text
  >>> import lzma
  Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
  ImportError: No module named 'lzma'
  ```

  这意味着你应该重新编译支持 lzam 的 Python 版本。

  也可以通过以下的方式进行安装：

  ```text
  $ pip install celery[lzma]
  ```

* zlib

  zlib 是 Deflate 算法的抽象，它的 API 支持包括 gzip 格式和轻量级流格式文件的支持。zlib 是许多软件系统的重要组成部分，例如 Linux 内核以及 Git VCS。

  要使用 zlib，请确保 zlib 已经编译到你的 Python 可执行文件中。

  如果你得到以下错误 [ImportError](https://docs.python.org/dev/library/exceptions.html#ImportError)

  ```text
  >>> import zlib
  Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
  ImportError: No module named 'zlib'
  ```

  这意味着你应该重新编译支持 zlib 的 Python 版本。

* zstd

  zstd是一个针对 zlib 的实时压缩方案，且有着更好的压缩效率。zstd 由 Huff0 和 FSE 库提供快速算法。

  要使用zstd，请用以下命令进行安装。

  ```text
  $ pip install celery[zstd]
  ```

你还可以创建自己的压缩方式，并在[kumbo压缩注册](tiao-yong-ren-wu-calling-tasks.md)中注册它们。

发送任务时的压缩方案配置优先级如下（从高到低）：

* 1.`compression` 执行选项。
* 2.Task.compression 属性。
* 3.task\_compression 属性。

任务调用时指定压缩方法的示例：

```python
>>> add.apply_async((2, 2), compression='zlib')
```

## 连接\(Connections\)

### 自动池支持

* 从2.3版开始，支持自动连接池，因此您不必手动处理连接和发布者即可重用连接。
* 从2.5版开始，默认情况下启用连接池。
* 有关 [broker\_pool\_limit](https://docs.celeryproject.org/en/latest/userguide/configuration.html#std:setting-broker_pool_limit) 更多信息，请参见设置。

您可以通过创建发布者来手动处理连接：

```python
results = []
with add.app.pool.acquire(block=True) as connection:
    with add.get_publisher(connection) as publisher:
        try:
            for args in numbers:
                res = add.apply_async((2, 2), publisher=publisher)
                results.append(res)
print([res.get() for res in results])
```

尽管这是个特定示例，但是可以更好的展现一组：

```python
>>> from celery import group

>>> numbers = [(2, 2), (4, 4), (8, 8), (16, 16)]
>>> res = group(add.s(i, j) for i, j in numbers).apply_async()

>>> res.get()
[4, 8, 16, 32]
```

## 路由选择 \(Routing options\)

Celery 可以将任务路由到不同的队列。

使用以下 queue 可以完成简单的路由\(name &lt;-&gt; name\)：

```text
add.apply_async(queue='priority.high')
```

然后，您可以指派 workers 给 priority.high 的队列,使用 worker -Q 参数将分配给队列：

```text
$ celery -A proj worker -l info -Q celery,priority.high
```

## 也可以看看

不建议用代码对队列名称进行硬编码，最佳做法是使用配置路由器（[task\_routes](https://docs.celeryproject.org/en/4.0/userguide/configuration.html#std:setting-task_routes)）。

要了解有关路由的更多信息，请参阅“[路由任务\(Routing Tasks\)](https://docs.celeryproject.org/en/4.0/userguide/routing.html#guide-routing)”。

## 高级选项

这些选项适用于想要使用AMQP完整路由功能的高级用户。有兴趣的人士可以阅读[路由指南](https://docs.celeryproject.org/en/4.0/userguide/routing.html#guide-routing)。

* 交换\(exchange\)

  发送信息的 exchange\(或者 kombu.entity.Exchange\) 的名称。

* routing\_key

  用于确定路由的密钥。

* 优先\(priority\)

  0～255 之间的数字，其中255是最高优先级。

  支持：RabbitMQ，Redis（优先级颠倒，最高为0）。

