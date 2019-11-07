# Semipredicates

职程（Worker）将任务包装在一个函数中，该函数记录任务的最终状态。有许多异常向该函数发出信号，用于更改处理任务返回的方式。

## Ignore

任务可能忽略，职程（Worker）会忽略该任务，并且不会记录该任务的任何状态信息，但会确认该消息（从任务队列中删除）。

如果需要自定义实现类似撤销的功能，或手动存储任务结果，可以使用该功能。

在 Redis 中保存已经撤销任务的例子：

```python
from celery.exceptions import Ignore@app.task(bind=True)def some_task(self):    if redis.ismember('tasks.revoked', self.request.id):        raise Ignore()
```

手动存储结果的例子：

```python
from celery import statesfrom celery.exceptions import Ignore@app.task(bind=True)def get_tweets(self, user):    timeline = twitter.get_timeline(user)    if not self.request.called_directly:        self.update_state(state=states.SUCCESS, meta=timeline)    raise Ignore()
```

## Reject

任务可以使用 AMQPs `basic_reject`方法来引发拒绝来拒绝任务消息。正常情况下是无效的，除非 `Task.acks_late` 启用。

拒绝消息和压缩消息效果相同，但一些中间人（Broker）可能实现可以使用的附加功能。例如，RabbitMQ 支持 `[死信交换机](http://www.rabbitmq.com/dlx.html)` 概念，可以将队列配置为死信交换，将被拒绝的消息重新交付给死信交换机。

Reject 也可以用于重新排队消息，但使用时需要非常小心的，很有可能会造成无限消息循环。

当任务导致内存不足时，使用 `Reject` 的例子：

```python
import errnofrom celery.exceptions import Reject@app.task(bind=True, acks_late=True)def render_scene(self, path):    file = get_file(path)    try:        renderer.render_scene(file)    # if the file is too big to fit in memory    # we reject it so that it's redelivered to the dead letter exchange    # and we can manually inspect the situation.    except MemoryError as exc:        raise Reject(exc, requeue=False)    except OSError as exc:        if exc.errno == errno.ENOMEM:            raise Reject(exc, requeue=False)    # For any other error we retry after 10 seconds.    except Exception as exc:        raise self.retry(exc, countdown=10)
```

重新排队的例子：

```python
from celery.exceptions import Reject@app.task(bind=True, acks_late=True)def requeues(self):    if not self.request.delivery_info['redelivered']:        raise Reject('no reason', requeue=True)    print('received two times')
```

有关 `basic_reject` 方法更多细节，请查阅中间人（Broker）章节。

## Retry

重试异常由任务引发。方法来告诉职程（Worker）任务正在重试。

