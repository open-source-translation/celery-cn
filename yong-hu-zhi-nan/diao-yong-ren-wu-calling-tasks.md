# 调用任务：Calling Tasks

## 基础

本文档介绍了任务实例和 [canvas]() 对 Celery 统一的 `Calling API`。

该 API 定义了三种方法，以及一系列的标准执行选项：

* `apply_async(args[, kwargs[, …]])`

  发送任务消息

* `delay(*args, **kwargs)`

  发送任务消息的快捷方式, 但是不支持执行选项

* `calling (__call__)`

  支持直接调用 API（例如 add(2, 2)）， 意味着任务将不会由 worker 执行，而是在当前进程中执行(不会发送消息)。

备忘单:

* `T.delay(arg, kwarg=value)`

   调用 apply_async 的快捷方式（.delay(*args, **kwargs)等价于调用 .apply_async(args, kwargs)）。

* `T.apply_async((arg,), {'kwarg': value})`

* `T.apply_async(countdown=10)`

  从现在起, 十秒内执行。

* `T.apply_async(eta=now + timedelta(seconds=10))`

  从现在起十秒内执行，指明使用eta。

* `T.apply_async(countdown=60, expires=120)`

  从现在起一分钟执行，但在两分钟后过期。

* `T.apply_async(expires=now + timedelta(days=2))`

  两天内过期，使用datetime对象。

例子：

`delay()` 方法很方便，因为它看起来像调用常规函数。
```py
task.delay(arg1, arg2, kwarg1='x', kwarg2='y')
```

使用 `apply_async()` 相反，你必须写：
```py
task.apply_async(args=[arg1, arg2], kwargs={'kwarg1': 'x', 'kwarg2': 'y'})
```

显然 `delay()` 更方便，但是如果你需要添加额外的执行选项，你必须使用 `apply_async()`

```
技巧

如果任务没有在当前进程中注册，你可以使用 send_task() 指定名称调用任务。
```

本文剩下的部分将会详细介绍任务执行选项，所有的用例都使用一个名为 add 的任务，并返回两个参数的和。
```py
@app.task
def add(x, y):
    return x + y
```

还有另一种方法：

你将在稍后阅读 [Canvas]() 进一步了解相关内容，`signature`'s 对象用于传递任务调用的签名（例如通过网络发送），并且它们都支持 `Calling API`。
```py
task.s(arg1, arg2, kwarg1='x', kwargs2='y').apply_async()
```

## 连接(回调/错误回调)
Celery 支持将任务连接在一起，以便一个任务紧随另一个任务。回调任务将使用父任务的结果作为部分的调用参数。
```py
res = add.apply_async((2, 2), link=add.s(16))

# 译者注
# res.get() --> 4
# res.children[0].get() --> 20
```

第一个任务的结果（ 4 ），将会发送给新任务，新任务将会将先前的结果与16相加，形成的表达式为: `(2 + 2) + 16 = 20`

`add.s`调用被称为签名，如果你还不了解这是什么，你应该在[canvas指南]()中阅读它们。在这里，你还可以了解到[任务链]()：将任务连接在一起的更简单的方法。

在实践中`link`执行选项被任务是内部原语，你可能不会直接使用它，而是使用`任务链`

如果任务抛出异常，你也可以使用回调，但是这与常规的回调行为不同，因为它将传递父任务的 ID（而不是结果）给回调任务。这是因为并不是总能将异常序列化，因此错误回调需要启用 backend。

这是一个错误回调示例：
```py
@app.task
def error_handler(uuid):
    result = AsyncResult(uuid)
    exc = result.get(propagate=False)
    print('Task {0} raised exception: {1!r}\n{2!r}'.format(
          uuid, exc, result.traceback))
```

可以使用 `link_error` 执行选项，给任务添加错误回调。
```py
add.apply_async((2, 2), link_error=error_handler.s())
```

此外，`link` 和 `link_error` 都可以作为一个列表：
```py
add.apply_async((2, 2), link=[add.s(16), other_task.s()])
```

父任务的结果将作为部分参数来分别调用所有的回调。

## On Message
Celery通过 `on_message` 回调可以捕获所有的状态变更。

例如，长时间运行的任务发送任务进度，可以这样做：
```py
@app.task(bind=True)
def hello(self, a, b):
    time.sleep(1)
    self.update_state(state="PROGRESS", meta={'progress': 50})
    time.sleep(1)
    self.update_state(state="PROGRESS", meta={'progress': 90})
    time.sleep(1)
    return 'hello world: %i' % (a+b)
```

```py
def on_raw_message(body):
    print(body)

r = hello.apply_async(args=[5, 5])
print(r.get(on_message=on_raw_message, propagate=False))
```

将阻塞，并生成如下的输出：
```py
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

## ETA和倒计时
通过 ETA（estimated time of arrival, 预计到底时间）你可以设定一个特定的时间，这是最早执行任务的时间。`倒计时`是一种以秒为单位设置ETA的快捷方式。
```py
>>> result = add.apply_async((2, 2), countdown=3)
>>> result.get()    # this takes at least 3 seconds to return
20
```

承诺任务在指定的日期时间后执行，并不保证在指定的时间一定执行。在截止时间后运行的原因可能是许多任务在队列中等待，或者严重的网络延时。为了确保你的任务能够及时执行，你应该监视队列中的任务拥塞情况。使用Munin或者其他类似工具来接收报警，因此可以采用合适的行为来减轻负担。见[Munin]()

虽然`倒计时`是一个整数，但是ETA必须是一个 `datetime` 对象，指定确切的日期和时间（包括毫秒精度和时区信息）：
```py
>>> from datetime import datetime, timedelta

>>> tomorrow = datetime.utcnow() + timedelta(days=1)
>>> add.apply_async((2, 2), eta=tomorrow)
```

## 过期
`expires` 参数定义了一个可选的过期时间，即可以通过整数指定倒计时（秒），也可以通过datetime指定日期时间。

```py
>>> # Task expires after one minute from now.
>>> add.apply_async((10, 10), expires=60)

>>> # Also supports datetime
>>> from datetime import datetime, timedelta
>>> add.apply_async((10, 10), kwargs,
...                 expires=datetime.now() + timedelta(days=1)
```

当工作进程收到一个过期的任务，会将任务标记为 [REVOKED]()（[TaskRevokedError]()）

## 消息发送重试
当连接失败时，Celery将会自动重试发送消息，并且可以配置重试行为（例如重试频率、最大重试次数），或者禁用。

要想禁用重试，可以配置retry执行参数为False。
```py
add.apply_async((2, 2), retry=False)
```

相关设置:
* [task_publish_retry]()
* [task_publish_retry_policy]()

### 重试策略
重试策略是一种控制重试行为的方式，可以包含以下的关键参数：
* max_retries

  在放弃前的最大重试次数，在这种场景下，将抛出重试失败的异常。
  
  值为 None，表示永远进行重试。
  
  默认值为重试3次。

* interval_start

  定义两次重试之间的间隔（秒）。默认值是0（首次重试是瞬的）。

* interval_step

  在每次连续重试时，该值将会添加到重试延时中（整数和浮点数）。默认值为0.2。

* interval_max

  两次重试之间的最大等待时间（秒）。默认值为0.2秒。

例如，默认的重试策略配置如下：
```py
add.apply_async((2, 2), retry=True, retry_policy={
    'max_retries': 3,
    'interval_start': 0,
    'interval_step': 0.2,
    'interval_max': 0.2,
})
```

重试的最长时间是0.4秒。默认情况下，应该把时间设置的相对较短，如果 broker 连接断开，将会导致一系列的因为连接失败的重试。例如，大量的WebServer进程等待重试，阻塞其他的输入请求。

**译者：**重试将会阻塞任务发送函数（delay/apply_async）。

## 连接错误处理
When you send a task and the message transport connection is lost, or the connection cannot be initiated, an OperationalError error will be raised:

当你发送一个任务时，消息传输丢失或者连接初始化失败，将会抛出 `OperationalError` 错误：
```py
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

如果启用了重试，这将会在重试用尽后，或者禁用后才发生。

你也能捕获这个失败：

```py
>>> from celery.utils.log import get_logger
>>> logger = get_logger(__name__)

>>> try:
...     add.delay(2, 2)
... except add.OperationalError as exc:
...     logger.exception('Sending task raised: %r', exc)
```

## 序列化
客户端和工作进程之间数据传输需要序列化，因此每条信息在Celery中都有一个 `content_type` 头，用于描述消息编码的序列化方法。

默认的序列化工具是 `JSON`，但是你可以通过使用配置 `task_serializer` 变更序列化方式，或者针对每个任务，甚至每条消息进行更改。

Celery内置支持 *JSON*, [pickle](), *YAML*以及*msgpack*，并且你也可以通过Kombu序列化注册表，添加自定义序列化方式。

Kombu用户指南中的[消息序列化](https://kombu.readthedocs.io/en/master/userguide/serialization.html#guide-serialization)

每个选项都有其优点和缺点：
* json - JSON 被大多数的编程语言支持，并且现在是 Python 标准的一部分（自2.6开始），并且现代 Python 库（例如 [simplejson](https://pypi.python.org/pypi/simplejson/)）具有非常快的 json 解析速度。

  JSON 的缺点是会限制你使用如下的数据类型：字符串、Unicode、字典和列表。小数和日期明显缺失。

  二进制数据使用 Base64 编码进行传输，这与支持纯二进制传输的相比，数据传输量增长了34%。

  但如果你的数据满足上述限制，并且你需要跨语言支持，则 JSON 可能是你的最佳选择。

* pickle - 如果你除了 Python 外，并不需要支持其他语言，那么使用 pickle 编码将让你获得对所有 Python 内建类型的支持（类实例除外）。相比 JSON，使用 pickle 序列化的二进制文件更小，传输速度更快。

  请参阅 [pickle](https://docs.python.org/dev/library/pickle.html#module-pickle) 获得更多信息。

* yaml - YAML 和 JSON 有许多相似的特征，yaml 支持包括日期、递归引用在内的更多数据类型。然而，Python 的 YMAL 库相比 JSON 库 要慢很多。

  如果你需要更具表现能力的数据集合，则 YMAL 比上面的序列化方式更适合。

  有关更多信息，请参见http://yaml.org/。

* msgpack - msgpack 是一种接近 JSON 的二进制序列化格式。它还不够稳定，因此应该将其视为实验性的。

  有关更多信息，请参见http://msgpack.org/。

编码类型可以用作消息头，因此 workers 知道如何反序列化所有的任务。如果你使用自定义序列方案，则该序列化必须被 workers 支持。

发送任务时的序列化配置优先级如下（从高到低）：
* 1.`serializer` 执行选项。
* 2.[Task.serializer]() 属性。
* 3.[task_serializer]() 属性。

为单个任务调用设置序列化方式：
```py
>>> add.apply_async((10, 10), serializer='json')
```

## 压缩
Celery 可以使用以下内建方案压缩消息。

* brotli

  brotli 针对 web 进行了优化，尤其是小型文档。该压缩对诸如字体、html页面等静态内容最有效。
  
  要使用 brotli，请用以下命令进行安装。
  ```
  $ pip install celery[brotli]
  ```

* bzip2

  bzip2 创建的文件比 gzip 小，但是压缩和解压的速度明显慢于 gzip。
  
  要使用 bzip2，请确保 bzip2 已经编译到你的 Python 可执行文件中。
  
  如果你得到以下错误 [ImportError](https://docs.python.org/dev/library/exceptions.html#ImportError)
  ```
  >>> import bz2
  Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
  ImportError: No module named 'bz2'
  ```
  这意味着你应该重新编译支持 bzip2 的 Python 版本。

* gzip

  gzip 适用于内存占用较小的系统，因此 gzip 非常适合内存有限的系统。该压缩常用语生成带有 “.tar.gz” 后缀的文件。
  ```
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
  ```
  >>> import lzma
  Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
  ImportError: No module named 'lzma'
  ```
  
  这意味着你应该重新编译支持 lzam 的 Python 版本。
  
  也可以通过以下的方式进行安装：
  ```
  $ pip install celery[lzma]
  ```

* zlib

  zlib 是 Deflate 算法的抽象，它的 API 支持包括 gzip 格式和轻量级流格式文件的支持。zlib 是许多软件系统的重要组成部分，例如 Linux 内核以及 Git VCS。

  要使用 zlib，请确保 zlib 已经编译到你的 Python 可执行文件中。

  如果你得到以下错误 [ImportError](https://docs.python.org/dev/library/exceptions.html#ImportError)
  ```
  >>> import zlib
  Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
  ImportError: No module named 'zlib'
  ```
  这意味着你应该重新编译支持 zlib 的 Python 版本。

* zstd

  zstd是一个针对 zlib 的实时压缩方案，且有着更好的压缩效率。zstd 由 Huff0 和 FSE 库提供快速算法。

  要使用zstd，请用以下命令进行安装。
  ```
  $ pip install celery[zstd]
  ```

你还可以创建自己的压缩方式，并在[kumbo压缩注册]()中注册它们。


发送任务时的压缩方案配置优先级如下（从高到低）：
* 1.`compression` 执行选项。
* 2.Task.compression 属性。
* 3.task_compression 属性。

任务调用时指定压缩方法的示例：
```py
>>> add.apply_async((2, 2), compression='zlib')
```

## 连接
你可以通过创建一个发布者来手动处理连接：

```py
results = []
with add.app.pool.acquire(block=True) as connection:
    with add.get_publisher(connection) as publisher:
        try:
            for args in numbers:
                res = add.apply_async((2, 2), publisher=publisher)
                results.append(res)
print([res.get() for res in results])
```

**自动池支持** 从2.3版开始，支持自动连接池，因此你不必手动处理连接和发布者即可重用连接。从2.5版开始，默认情况下启用连接池。有关[broker_pool_limit]() 更多信息，请参见设置。

这个特定的示例可以通过 group 更好的表示：

```py
>>> from celery import group

>>> numbers = [(2, 2), (4, 4), (8, 8), (16, 16)]
>>> res = group(add.s(i, j) for i, j in numbers).apply_async()

>>> res.get()
[4, 8, 16, 32]
```

## 路由选项
Celery 可以路由任务到不同的队列。

使用 `queue` 执行选项，可以完成简单的路由。

```py
add.apply_async(queue='priority.high')
```

你可以通过使用 workers 的 `-Q` 参数，将 workers 分配给队列 `priority.high`。

```
$ celery -A proj worker -l info -Q celery,priority.high
```

不建议对队列名称使用硬编码，最好的方式是使用 routers 配置（[task_routes]()）。要了解有关路由的更多信息，请看[路由任务]()

## 结果选项
你可以通过使用 [task_ignore_result ]() 设置或 `ignore_result` 执行选项，开启或禁用对任务结果的存储。

```py
>>> result = add.apply_async(1, 2, ignore_result=True)
>>> result.get()
None

>>> # Do not ignore result (default)
...
>>> result = add.apply_async(1, 2, ignore_result=False)
>>> result.get()
3
```

如果你希望在 backend 中存储关于任务的额外元数据，可以将`result_extended`设置为True。

更多关于任务的信息，请参见[任务](ren-wu-tasks/README.md)

### 高级选项
这些选项适用于想要使用AMQP完整路由功能的高级用户。有兴趣的人士可以阅读[路由指南]()。

* exchange

  名称交换

* routing_key

  用于确定 Routing Key。

* priority

  0～255之间的数字，255是最高优先级。支持 RabbitMQ，Redis（优先级颠倒，0是最高优先级）。
