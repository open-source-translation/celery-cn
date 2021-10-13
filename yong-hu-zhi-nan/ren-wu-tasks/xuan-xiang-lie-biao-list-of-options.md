# 选项列表：List of Options

任务装饰器有很多配置选项，例如可以使用 `rate_limit` 选项配置任务的频率。

传递给任务装饰器的任何关键字参数都为任务结果类的属性，这是一个内置的属性列表。

## 常规配置

### Task.name

任务注册的名称。 可以手动设置任务名称，也可以通过模块名和类名进行自动生成。

 详情查阅 [`名字：Names`](ming-zi-names.md) 章节。

### Task.request

如果该任务正处于执行状态，该信息包含该任务的请求信息。使用多线程本地存储。

详情查阅 [`任务请求章节：Task Request`](ren-wu-qing-qiu-task-request.md) 章节。

### Task.max_retries

当前任务调用 `self` 或使用 `autoretry_for` 参数时才会启用。

如果重试的次数超过最大限制，会引发 `MaxRetriesExceededError` 异常。

{% hint style="info" %}
## Note：

在异常时不会自动重试，所以必须手动调用 `retry()`。
{% endhint %}

默认值重试次数为3次，如果设置为 `None` 会关闭重试限制，直到任务执行成功为止。

### Task.throws

预期内的异常，如果在元组中含有该异常类，将不会被视为异常。

列表中的错误将错误信息记录到结果后端中，但职程（Worker）不会将该事件记录为错误信息，也没有回溯。

例如：

```python
@task(throws=(KeyError, HttpNotFound)):
def get_foo():
    something()
```

错误类型：

*   预期错误（在 Task.throws）

      记录 `INFO` 信息，但回溯排除。
*   非预期错误

      记录 `ERROR` 信息，可回溯。

### Task.default_retry_delay

默认重试任务休眠事件（以秒为单位）。可以为 `int`，也可以为 `float` 类型，默认情况为 3 分钟。

### Task.rate_limit

限制指定任务类型的速率（限制在指定时间内运行的任务数量）。当速率限制生效时，任务仍然会完成，但是可能需要一些时间才能开始。

如果限制速率为 `None`，表示速率限制无效。速率可以为 `int` 也可以为 `float` 类型，则被表示为“每秒任务数”。

速率限制也可以在数值后面添加 "/s"、"/m" 或 "/h"，以秒、分钟或以小时为单位。任务将在指定的时间内平均分配。

例如："\`00/m" （每分钟100个任务）。则强制会在同一个职程（Worker）实例上启动俩个任务之间至少 600ms 的延迟。

默认值通过 `task_default_rate_limit` 进行设定：如果未指定，表示默认情况禁用任务的速率限制。

注意，该速率限制为每一个职程（Worker）实例的限制，并非全局速率限制。配置全局速率限制（例如，API每秒最多请求的次数），必须制定队列。

### Task.time_limit

该任务的硬时间限制（以秒为单位），如果没有设置职程（Worker）时，使用默认值。

### Task.soft_time_limit

该任务的软时间限制（以秒为单位），如果没有设置职程（Worker）时，使用默认值。

### Task.ignore_result

不存储任务状态信息，如果配置该选项 `AsyncResult` 将失效，无法进行检测任务情况以及返回内容。

### Task.store_errors_even_if_ignored

如果设置为 `True` ，即使任务被忽略，也会存储错误信息。

### Task.serializer

标识需要使用默认序列化的字符串。默认为 `task_serializer`，也可以为 `pickle`、`json`、`yaml` 或者通过 `kombu.serialization.registry`注册的自定义序列化方法。

详情查阅 `Serializers` 章节。

### Task.compression

标识需要使用默认压缩方法的字符串。

默认为 `task_compression`，可以设置为 `gzip`、`bzip2`或通过 `kombu.compression` 注册的自定义压缩方案。

详情查阅 `Compression` 章节。

### Task.backend

结果后端的实例，用于任务结果后端，默认为 `app.backend`，可以通过 `result_backend` 进行配置。

### Task.acks_late

如果设置为 `True`，任务执行后（而不是执行前，默认为执行前）才会确认该任务的消息。

注意：如果职程（Worker）执行过程中崩溃，任务可能会执行多次。

可以通过 `task_acks_late` 参数来进行全局配置。

### Task.track_started

如果设置为 `True`，当职程（Worker）执行任务时，任务状态为 `stared`。默认为 `False`，因为在正常情况下是不需要该颗粒度级别的。任务要么挂起、完成要么等待重试。如果有长时间运行的任务吗，并且需要报告任当任务状态时，`stared`比较有用。

任务执行的职程（Worker）和进程 id 可以通过状态的元数据中进行查看（例如：result.info\['pid']）。

可以通过 `task_track_started` 进行全局配置。

{% hint style="info" %}
## See one see （可以看一看）

任务的 API 接口
{% endhint %}

