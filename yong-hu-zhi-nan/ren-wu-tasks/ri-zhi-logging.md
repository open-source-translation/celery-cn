# 日志：Logging

职程（Worker）会自动记录日志信息，也可以手动配置日志记录信息。

Celery 中有一个 `celery.task`，可以通过继承的方式获取日志的一部分任务名称以及唯一ID。

最好的做法是在模块顶部为所有任务创建一个共有的日志记录器：

```python
from celery.utils.log import get_task_logger

logger = get_task_logger(__name__)

@app.task
def add(x, y):
    logger.info('Adding {0} + {1}'.format(x, y))
    return x + y
```

Celery 使用的是 Python 标准的日志库，详情文档可以在这里找到。 也可以使用 print() ,所有标准 out/-err 都会被重定向到写入到日志系统中（可以关闭该功能，详情参阅 worker_redirect_stdouts）。

{% hint style="info" %}
## Note：

如果在任务或者任务模块中的某个位置创建了日志记录器实例，职程（Worker）不会更新重定向。

如果需要将 `sys.stdout` 和 `sys.stderr`重定向到自定义日志记录器中，必须手动启用该功能，例如：

```python
import sys

logger = get_task_logger(__name__)

@app.task(bind=True)
def add(self, x, y):
    old_outs = sys.stdout, sys.stderr
    rlevel = self.app.conf.worker_redirect_stdouts_level
    try:
        self.app.log.redirect_stdouts_to_logger(logger, rlevel)
        print('Adding {0} + {1}'.format(x, y))
        return x + y
    finally:
        sys.stdout, sys.stderr = old_outs
```
{% endhint %}

{% hint style="info" %}
## Note：

如果需要将 Celery 日志记录器不打印日志，应该检查记录器值是否正确传播。在此案例中，启用 `celery.app.trace`，以便发出 "succeeded in" 日志信息：

```python
import celery
import logging

@celery.signals.after_setup_logger.connect
def on_after_setup_logger(**kwargs):
    logger = logging.getLogger('celery')
    logger.propagate = True
    logger = logging.getLogger('celery.app.trace')
    logger.propagate = True
```
{% endhint %}

## 参数校验

_4.0 新版本功能_

Celery 会校验调用任务时传递的参数信息，就像 Python 调用普通函数时一样：

```python
>>> @app.task
... def add(x, y):
...     return x + y

# Calling the task with two arguments works:
>>> add.delay(8, 8)
<AsyncResult: f59d71ca-1549-43e0-be41-4e8821a83c0c>

# Calling the task with only one argument fails:
>>> add.delay(8)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "celery/app/task.py", line 376, in delay
    return self.apply_async(args, kwargs)
  File "celery/app/task.py", line 485, in apply_async
    check_arguments(*(args or ()), **(kwargs or {}))
TypeError: add() takes exactly 2 arguments (1 given)
```

可以通过设置任务的 `typing`参数设置为 **`False`**来关闭参数校验：

```python
>>> @app.task(typing=False)
... def add(x, y):
...     return x + y

# Works locally, but the worker receiving the task will raise an error.
>>> add.delay(8)
<AsyncResult: f59d71ca-1549-43e0-be41-4e8821a83c0c>
```

## 隐藏参数中的敏感信息

_4.0 新版本功能_

使用 task_protocol 2 或更高版本的时（自 4.0 开始为默认值），可以重写位置参数和关键字参数在日志中的表现方式，并且可以使用 `argsrepr` 和 `kwargsrepr` 调用参数监控事件：

```python
>>> add.apply_async((2, 3), argsrepr='(<secret-x>, <secret-y>)')

>>> charge.s(account, card='1234 5678 1234 5678').set(
...     kwargsrepr=repr({'card': '**** **** **** 5678'})
... ).delay()
```

{% hint style="warning" %}
## 警告

任何能够从中间人（Broker）中读取任务消息或以其他方式拦截任务消息的人都可以访问敏感信息。 如果消息中包含敏感信息，应该针对该敏感信息进行加密，在本案例中使用性用卡号对实际的号码进行加密，并且进行安全存储，然后在任务中进行检索解密。
{% endhint %}
