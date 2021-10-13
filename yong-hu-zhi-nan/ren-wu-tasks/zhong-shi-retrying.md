# 重试：Retrying

当任务出现错误时，可以通过 `app.Task.retry()` 重新执行。 

当调用 `retry` 时，会发送与原始任务相同的ID发送一条消息，将该消息发送到原始任务的对列中。

当任务被重试时，也会被记录为一个任务状态，便于通过 result 实例来跟踪任务。 

这是一个 `retry`的例子：

```python
@app.task(bind=True)
def send_twitter_status(self, oauth, tweet):
    try:
        twitter = Twitter(oauth)
        twitter.update_status(tweet)
    except (Twitter.FailWhaleError, Twitter.LoginError) as exc:
        raise self.retry(exc=exc)
```

{% hint style="info" %}
## None：

`app.Task.retry()` 调用时会引发异常，在重试之后将无法访问任何代码。重试异常，并不是作为错误处理的，表示职程（Worker）需要重试任务，便于在启用结果后端时存储正确的任务状态。

重试异常是正常的，除非将 `throw` 设置为 `False` 可以避免该情况发生。
{% endhint %}

任务装饰器的 `bind` 参数将允许访问 `self`（任务类型实例）。

 `exc` 参数主要用传递日志和存储任务结果时的使用的异常信息。`exception` 和 `traceback` 都将在任务状态中可用(如果启用了结果后端)。

任务如果有一个 `max_retries` 值，超出了重试的最大次数，则会重新引发当前的异常信息，但如果：

*   `exc` 参数没有设置

      该情况会引发 `MaxRetriesExceededError` 异常
*   没有异常

      如果没有初始异常来重新引发exc参数，可以使用：

    ```python
      self.retry(exc=Twitter.LoginError())
    ```

      设置 `exc` 参数值

## 自定义重试延迟

当任务需要重试时，可以等待一段时间来进行重试，默认延迟可以通过 `default_retry_delay` 来进行设置的。默认时间为 3 分钟，延迟时间的单位是秒（类型可以为 int 或 float）。

也可以通过 `retry()` 中的 `countdown` 参数来覆盖默认值。

```python
@app.task(bind=True, default_retry_delay=30 * 60)  # retry in 30 minutes.
def add(self, x, y):
    try:
        something_raising()
    except Exception as exc:
        # overrides the default delay to retry after 1 minute
        raise self.retry(exc=exc, countdown=60)
```

## 自动重试已知异常

_4.0 版本的新功能_

有时，您只想在引发特定异常时重试任务。 可也通过 Celery 中 `task()` 装饰器中的 `autoretry_for` 参数进行自动重试任务：

```python
from twitter.exceptions import FailWhaleError

@app.task(autoretry_for=(FailWhaleError,))
def refresh_timeline(user):
    return twitter.refresh_timeline(user)
```

可以通过 `task()` 中的 `retry_kwargs` 参数来指定 `retry()` 内部调用参数：

```python
@app.task(autoretry_for=(FailWhaleError,),
          retry_kwargs={'max_retries': 5})
def refresh_timeline(user):
    return twitter.refresh_timeline(user)
```

这是手动处理异常的替代方法，上面的示例与在 `try` ... `except`语句中包含的代码块一样：

```python
@app.task
def refresh_timeline(user):
    try:
        twitter.refresh_timeline(user)
    except FailWhaleError as exc:
        raise div.retry(exc=exc, max_retries=5)
```

如果你想自动重试任何错误，只需使用：

```python
@app.task(autoretry_for=(Exception,))
def x():
    ...
```

_4.2 版本新功能_

如果任务依赖于另外一个服务，例如向 API 发起请求，可以使用 `exponential backoff`，避免请求淹没服务。 Celery 的自动重试可以通过 `retry_backoff` 参数进行兼容：

```python
from requests.exceptions import RequestException

@app.task(autoretry_for=(RequestException,), retry_backoff=True)
def x():
    ...
```

默认情况下，`exponential backoff` 会随机引入 jitter，最大延迟限制在10分钟，可以通过下面的选项进行自定义配置。

*   Task.autoretry_for

      异常类的列表或元组，如果任务在执行的过程中引发异常，任务将自动重试。默认情况下不会自动重试任何异常。
*   Task.retry_kwargs

      字典类型，自定义配置自动重试参数。注意，如果使用下面的 `exponential backoff` 选项是，`countdown` 任务选项将由 Celery 的自动重试系统决定，字典中包含 `countdown` 会被忽略。
*   Task.retry_backoff

      一个布尔值或一个数字。如果将此选项设置为True，则自动重试将按照 `exponential backoff` 规则延迟。第一次重试延迟 1 秒，第二次重试延迟 2 秒，第三次延迟 4 秒，第四次延迟 8 秒，以此类推。（如果启用了 `retry_jitter` 会修改延迟值）。如果该选项设置为数字，则作为延迟因子，例如，该选项设置为 3，那么第一次重试将延迟 3 秒，第二次将延迟 6 秒，第三次延迟 12 秒，第四次延迟 24秒，以此类推。默认情况下，该选项设置为 False，自动重试不会延迟。
*   Task.retry_backoff_max

      一个数字，如果启动了 `retry_backoff`，该选项在任务自动重试之间设置以秒为单位的最大延迟。默认情况，该选项默认值为 600，即 10分钟。
*   Task.retry_jitter

      一个布尔值，Jitter 用于随机性引入指数回退延迟，防止队列中所有任务同时执行，如果该选项设置为 True，则将 `retry_backoff` 计算的延迟作为最大值，实际的延迟值为一个介于 0 和最大值之间的一个随机数。默认情况下，该选项为 True。
