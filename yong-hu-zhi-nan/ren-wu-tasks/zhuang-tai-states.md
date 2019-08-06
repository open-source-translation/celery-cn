# 状态：States

Celery 可以跟踪任务当前的状态信息。状态信息包含成功任务的结果，或执行失败任务的异常信息。

有几个结果后端可供选择，它们都有不同的优点和缺点（参见结果后端）。

在任务的执行周期中，可能会有几种状态的变换，每次变换都会附加当前状态信息。当任务进入下一个状态时，上一个状态的信息会被移除，但可以进行推断任务的状态信息（例如，一个任务处于执行 `FAILED` 状态，则表示在某个时刻是处于 `STARTED` 状态的）。

还有一些任务状态信息，例如 `FAILURE_STATES`、`READY_STATES`。

客户端使用这些集合的成员关系来决定是否应该重新引发异常（PROPAGATE\_STATES），或者是否可以缓存状态\(如果任务准备好了，可以缓存状态\)。

也可以阅读[`自定义状态`](zhuang-tai-states.md#zi-ding-yi-zhuang-tai)。

## 结果后端

如果需要跟踪任务信息或返回值，需要提供一个 Celery 存储的结果后端，便于检索。有几个内置的结果后端可以考虑使用：SQLAlchemy/Django ORM、Memcached、RabbitMQ/QPid\(rpc\) 和 Redis，也可以自定义后端。

每一个后端适用于所有的情况，每一个后端都有优缺点，应该选择适合结果后端。

{% hint style="info" %}
## Warning： 

后端使用资源存储和传输结果。为确保资源被释放，需要对调用任务后返回的每一个 AsyncResult 实例调用 `get()` 或 `forget()`。
{% endhint %}

{% hint style="info" %}
## See also： 

`Task result backend settings`
{% endhint %}

### RPC 结果后端（RabbitMQ/QPid）

RPC 后端（`rpc://`）比较特殊，因为它实际上不存储状态信息，只是作为消息发送。这是一个重要区别，因为这意味着一个结果只能被检索一次，而且只能有启动任务的机器检索。俩个不同的进程获取的结果不同。

即使有这样的限制，如果需要实时状态更新信息，这是一个比较好的选择。使用消息传递这就意味着不用轮询获取状态信息。

默认情况下，消息暂时的（非持久的），如果重启，结果会消息。可以使用 `result_persistent` 配置将结果信息转换为持久消息。

### 数据库结果后端

对于多数人来说，使用数据库保存任务状态信息是比较方便的，特别是 web 应用程序使用的数据库这一类，但也有一些限制。

* 使用数据库轮询获取任务状态信息会导致数据库压力很大，应该设置轮询的间隔时间，例如 `result.get()`。
* 有些数据库使用默认的事务隔离级别，不适合轮询表以进行更改。

  在 MySQL 中，默认的事务隔离级别是 `REPEATABLE-READ`，在提交当前事务之前，事务不会看到其他事务所做的更改。

  建议将其更改为 `READ-COMMITTED` 级别。

## 内置状态

### PENDING

任务正在等待执行或未知。任何未知的任务 ID 都默认处于挂起状态。

### STARTED

任务已经开始。默认情况下不会记录，需要启用，请参阅 `app.Task.track_started.`。

meta-data：正在执行任务的职程（Worker） pid 和主机名。

### SUCCESS

任务执行成功。

meta-data：任务结果返回值 propagates：Yes ready: Yes

### FAILURE

任务执行失败。

meta-data：执行异常时的任务信息，其中 traceback 包含引发错误的堆栈信息。 propagates：Yes

### RETRY

任务处于重试状态。

meta-data：结果信息包含导致重试的异常信息，traceback 包含引发异常时堆栈的回溯。 propagates：No

### REVOKED

任务被撤销。

propagates：Yes

## 自定义状态

只需要设置一个位置的名称，就可以轻松的自定义状态，状态名通常是大写的字符串。例如，您可以查看定义自定义中止状态的可中止任务。

使用 `update_state()` 更新任务状态：

```python
@app.task(bind=True)
def upload_files(self, filenames):
    for i, file in enumerate(filenames):
        if not self.request.called_directly:
            self.update_state(state='PROGRESS',
                meta={'current': i, 'total': len(filenames)})
```

在这里，创建了一个名称为“ `PROGRESS`”的状态，通过 `current` 和 `total` 作为元数据的一部分，计算任务当前正在进行状态的任何应用程序以及任务在进程中位置。可以通过该方法来创建任务进度条。

## 创建可处理的异常

Python 的异常必须要符合一些简单规则，才能被 `pickle` 模块支持以及序列化。

使用 Pickle 作为序列化器时，引发不可拾取异常的任务将无法正常工作。

为确保异常是可被处理的，异常必须要在 `.args` 属性中提供初始化的参数。最简单的方法就是使用异常调用 `Exception.__init__`。

让我们来看一些有用的例子，还有一个不适用的例子：

```python
# OK:
class HttpError(Exception):
    pass

# BAD:
class HttpError(Exception):

    def __init__(self, status_code):
        self.status_code = status_code

# OK:
class HttpError(Exception):

    def __init__(self, status_code):
        self.status_code = status_code
        Exception.__init__(self, status_code)  # <-- REQUIRED
```

所以规则是：对于任何支持自定义参数 `*args` 的异常，都必须使用 `Exception.__init__(self, *args)`。

关键资产没有特殊支持，如果需要保存关键字参数，当异常被 unpickled 时，需要将它们作为普通的参数进行传递：

```python
class HttpError(Exception):

    def __init__(self, status_code, headers=None, body=None):
        self.status_code = status_code
        self.headers = headers
        self.body = body

        super(HttpError, self).__init__(status_code, headers, body)
```

